# ChirpStack Gateway Mesh - Complete Protocol Deep Dive

**Analysis Date:** January 6, 2026  
**Author:** Deep Dive Analysis  
**Status:** Complete Comprehensive Analysis

---

## Table of Contents

1. [Introduction](#introduction)
2. [Network Architecture](#network-architecture)
3. [Global State Variables](#global-state-variables)
4. [Uplink Handling](#uplink-handling)
   - [4.1 handle_uplink() - Entry Point](#41-handle_uplink---entry-point)
   - [4.2 Border Gateway: proxy_uplink_lora_packet()](#42-border-gateway-proxy_uplink_lora_packet)
   - [4.3 Relay Gateway: relay_uplink_lora_packet()](#43-relay-gateway-relay_uplink_lora_packet)
   - [4.4 Packet Structure and Overhead Analysis](#44-packet-structure-and-overhead-analysis)
   - [4.5 Metadata Structure Deep Dive](#45-metadata-structure-deep-dive)
   - [4.6 Helper Functions](#46-helper-functions)
5. [Mesh Packet Handling](#mesh-packet-handling)
   - [5.1 handle_mesh() - Main Router](#51-handle_mesh---main-router)
   - [5.2 Border Gateway: proxy_uplink_mesh_packet()](#52-border-gateway-proxy_uplink_mesh_packet)
   - [5.3 Border Gateway: proxy_event_mesh_packet()](#53-border-gateway-proxy_event_mesh_packet)
   - [5.4 Relay Gateway: relay_mesh_packet() - The Flooding Engine](#54-relay-gateway-relay_mesh_packet---the-flooding-engine)
6. [Downlink Handling](#downlink-handling)
   - [6.1 handle_downlink() - Routing Decision](#61-handle_downlink---routing-decision)
   - [6.2 Border Gateway: proxy_downlink_lora_packet()](#62-border-gateway-proxy_downlink_lora_packet)
   - [6.3 Border Gateway: relay_downlink_lora_packet()](#63-border-gateway-relay_downlink_lora_packet)
7. [Command System](#command-system)
   - [7.1 send_mesh_command()](#71-send_mesh_command)
8. [Critical Analysis](#critical-analysis)
9. [Open Questions and Future Work](#open-questions-and-future-work)

---

## 1. Introduction

ChirpStack and RAK recently implemented the mesh protocol. Since the documentation doesn't show explicitly the implemented protocol of the mesh, we decided to deep dive into the codebase.

This document provides a comprehensive analysis of the mesh.rs implementation, covering every function, data structure, and protocol decision in detail.

### Network Components

In this protocol, mainly three components are present:

1. **Border Gateway** - Directly connected to the network server
2. **Relay Gateway** - In the mesh hopping path with no direct connection to the network server
3. **End Devices** - Transmit to either Relay or Border gateway

---

## 2. Network Architecture

### 2.1 Network Topology

```
End Device → Relay Gateway(s) → Border Gateway → Network Server
           ←  (mesh flooding) ←                ←   (internet)
```

### 2.2 Communication Paths

**Uplink Path:**
```
End Device --[LoRa]--> Relay Gateway --[Mesh]--> Border Gateway --[ZMQ]--> Network Server
```

**Downlink Path:**
```
Network Server --[ZMQ]--> Border Gateway --[Mesh]--> Relay Gateway --[LoRa]--> End Device
```

---

## 3. Global State Variables

At the start of mesh.rs, constant global variables are defined:

```rust
static CTX_PREFIX: [u8; 3] = [1, 2, 3];
static MESH_CHANNEL: Mutex<usize> = Mutex::new(0);
static UPLINK_ID: Mutex<u16> = Mutex::new(0);
static UPLINK_CONTEXT: LazyLock<Mutex<HashMap<u16, Vec<u8>>>> =
    LazyLock::new(|| Mutex::new(HashMap::new()));
static PAYLOAD_CACHE: LazyLock<Mutex<Cache<PayloadCache>>> =
    LazyLock::new(|| Mutex::new(Cache::new(64)));
```

These global variables are used throughout the mesh protocol implementation to maintain state and coordinate operations.

### 3.1 CTX_PREFIX - Context Identification

**Purpose:** Magic bytes for identifying mesh-routed downlinks

**Value:** `[1, 2, 3]`

**Usage:** Used to detect whether a downlink should be routed through the mesh or transmitted directly.

### 3.2 MESH_CHANNEL - Frequency Selection

**Purpose:** Round-robin channel selection counter

**Type:** `Mutex<usize>` for thread-safe access

**Usage:** Cycles through configured mesh frequencies to distribute transmission load.

### 3.3 UPLINK_ID - Correlation Counter

**Purpose:** Counter for uplink correlation

**Range:** 0-4095 (12-bit value, wraps around)

**Usage:** Links uplinks with their corresponding downlinks for timing purposes.

### 3.4 UPLINK_CONTEXT - Timing Storage

**Purpose:** Maps uplink_id → context bytes

**Type:** `LazyLock<Mutex<HashMap<u16, Vec<u8>>>>`

**Usage:** Stores routing information and timing data for downlinks. This HashMap is created the first time the static is accessed (LazyLock), and lives in a fixed memory location. Since this isn't a compile-time constant, it cannot be defined with just `static`.

The matching of data is: the key is uplink ID and the value is the context bytes from the concentrator. This will be created the first time static is accessed (LazyLock), and this lives in a fixed memory location.

### 3.5 PAYLOAD_CACHE - Duplicate Detection

**Purpose:** LRU cache for duplicate packet detection

**Capacity:** 64 entries

**Type:** `LazyLock<Mutex<Cache<PayloadCache>>>`

**Usage:** The ONLY anti-flooding mechanism besides hop count. Every gateway has a local payload cache as a global variable. This has thread safe initialization, protects cache from concurrent access by using mutex, and it is an LRU (Least Recently Used) cache holding packets with a maximum cache capacity of 64.

---

## 4. Uplink Handling

### 4.1 handle_uplink() - Entry Point

This is the first function defined here which handles uplinks. This takes one of the main decisions at the border gateway.

```rust
pub async fn handle_uplink(border_gateway: bool, pl: &gw::UplinkFrame) -> Result<()> {
    match border_gateway {
        true => proxy_uplink_lora_packet(pl).await,
        false => relay_uplink_lora_packet(pl).await,
    }
}
```

**Decision Logic:**
- If border gateway receives a LoRa packet, it will call the `proxy_uplink_lora_packet` function
- If it is a relay gateway, it will call the `relay_uplink_lora_packet` function

---

### 4.2 Border Gateway: proxy_uplink_lora_packet()

```rust
async fn proxy_uplink_lora_packet(pl: &gw::UplinkFrame) -> Result<()> {
    info!(
        "Proxying LoRaWAN uplink, uplink: {}",
        helpers::format_uplink(pl)?
    );

    let pl = gw::Event {
        event: Some(gw::event::Event::UplinkFrame(pl.clone())),
    };

    proxy::send_event(pl).await
}
```

#### Step 1: Log the Uplink

They first log the uplink for debugging/monitoring. The `format_uplink` helper function from helpers converts the UplinkFrame into the following format:

```
"[uplink_id: {}, freq: {}, rssi: {}, snr: {}, mod: {}]"
```

#### Step 2: Wrap in Event Structure

Then from the event struct they specify the outer Protocol buffer message as the Uplink Frame.

```rust
pub struct Event {
    #[prost(oneof = "event::Event", tags = "1, 2, 3")]
    pub event: ::core::option::Option<event::Event>,
}

/// Nested message and enum types in `Event`.
pub mod event {
    #[derive(Clone, PartialEq, ::prost::Oneof)]
    pub enum Event {
        /// Uplink frame.
        #[prost(message, tag = "1")]
        UplinkFrame(super::UplinkFrame),
        /// Gateway stats.
        #[prost(message, tag = "2")]
        GatewayStats(super::GatewayStats),
        /// Gateway Mesh Event.
        #[prost(message, tag = "3")]
        Mesh(super::MeshEvent),
    }
}
```

This is from the gw.rs file.

#### Step 3: Send to Network Server

Then it will be sent through the ZMQ socket to the ChirpStack network server by calling the `send_event` function from proxy.

---

### 4.3 Relay Gateway: relay_uplink_lora_packet()

The other uplink type will be handled by the `relay_uplink_lora_packet`.

```rust
async fn relay_uplink_lora_packet(pl: &gw::UplinkFrame) -> Result<()> {
    let conf = config::get();

    let rx_info = pl
        .rx_info
        .as_ref()
        .ok_or_else(|| anyhow!("rx_info is None"))?;
    let tx_info = pl
        .tx_info
        .as_ref()
        .ok_or_else(|| anyhow!("tx_info is None"))?;
    let modulation = tx_info
        .modulation
        .as_ref()
        .ok_or_else(|| anyhow!("modulation is None"))?;
```

#### Step 1: Extract Required Fields

First, extract the required fields and get references to the reception/transmission metadata.

#### Step 2: Create Mesh Packet

Then in the next step, a mesh packet will be created for the uplink.

```rust
let mut packet = MeshPacket {
    mhdr: MHDR {
        payload_type: PayloadType::Uplink,
        hop_count: 1,
    },
    payload: Payload::Uplink(UplinkPayload {
        metadata: UplinkMetadata {
            uplink_id: store_uplink_context(&rx_info.context),
            dr: helpers::modulation_to_dr(modulation)?,
            channel: helpers::frequency_to_chan(tx_info.frequency)?,
            rssi: rx_info.rssi as i16,
            snr: rx_info.snr as i8,
        },
        relay_id: backend::get_relay_id().await?,
        phy_payload: pl.phy_payload.clone(),
    }),
    mic: None,
};
```

**MHDR (Mesh Header):**
- Payload type: `Uplink`
- Hop count: `1` (since this is the first hop)

**Uplink Payload Structure:**

The packet structure uses a struct for Uplink Payload:

```rust
pub struct UplinkPayload {
    pub metadata: UplinkMetadata,
    pub relay_id: [u8; 4],
    pub phy_payload: Vec<u8>,
}

impl UplinkPayload {
    pub fn from_slice(b: &[u8]) -> Result<UplinkPayload> {
        if b.len() < 9 {
            return Err(anyhow!("At least 9 bytes are expected"));
        }

        let mut md = [0; 5];
        let mut gw_id = [0; 4];
        md.copy_from_slice(&b[0..5]);
        gw_id.copy_from_slice(&b[5..9]);

        Ok(UplinkPayload {
            metadata: UplinkMetadata::from_bytes(md),
            relay_id: gw_id,
            phy_payload: b[9..].to_vec(),
        })
    }
}
```

---

### 4.4 Packet Structure and Overhead Analysis

#### Overhead Calculation

- **Metadata:** 5 bytes
- **Relay ID:** 4 bytes
- **MHDR (Mesh header):** 1 byte (111 for mesh, 2 bits for event, 3 bits for hop count)
- **MIC:** 4 bytes
- **Total Overhead:** 14 bytes

#### Maximum Payload Constraint

To use the highest spreading factor (SF12 at 125kHz), the maximum payload is 51 bytes.

**Available for PHYPayload:**
```
51 bytes (max) - 14 bytes (overhead) = 37 bytes
```

#### Critical Question

This gives us the problem: **"Do the devices have to be aware of the amount of data to be transmitted?"** or the mesh should be able to handle the data rates in order to allow the whole payload to be transmitted.

---

### 4.5 Metadata Structure Deep Dive

We should also have an idea about how the Metadata structure works:

```rust
pub struct UplinkMetadata {
    pub uplink_id: u16,
    pub dr: u8,
    pub rssi: i16,
    pub snr: i8,
    pub channel: u8,
}

impl UplinkMetadata {
    pub fn from_bytes(b: [u8; 5]) -> Self {
        let snr = b[3] & 0x3f;
        let snr = if snr > 31 {
            (snr as i8) - 64
        } else {
            snr as i8
        };

        UplinkMetadata {
            uplink_id: u16::from_be_bytes([b[0], b[1]]) >> 4,
            dr: b[1] & 0x0f,
            rssi: -(b[2] as i16),
            snr,
            channel: b[4],
        }
    }

    pub fn to_bytes(&self) -> Result<[u8; 5]> {
        if self.uplink_id > 4095 {
            return Err(anyhow!("Max uplink_id value is 4095"));
        }

        if self.dr > 15 {
            return Err(anyhow!("Max dr value is 15"));
        }

        if self.rssi > 0 {
            return Err(anyhow!("Max rssi value is 0"));
        }

        if self.rssi < -255 {
            return Err(anyhow!("Min rssi value is -255"));
        }

        if self.snr < -32 {
            return Err(anyhow!("Min snr value is -32"));
        }
        // ... rest of implementation
    }
}
```

#### Bit-Level Structure

Rust has some specific data sizes to store the data. That is the reason for the results to be stored as the closest higher ceiling value of any returning value. For example, 4 bits are saved as u8 which is unsigned 8 bits or 1 byte.

**Byte Allocation:**

| Byte | Bits | Field | Range/Notes |
|------|------|-------|-------------|
| 0 | 8 bits | Uplink ID (high) | First 8 bits of 12-bit ID |
| 1 | 4 bits (high) | Uplink ID (low) | Last 4 bits of 12-bit ID |
| 1 | 4 bits (low) | Data Rate | 0-15 |
| 2 | 8 bits | RSSI | Stored as positive, converted to negative |
| 3 | 6 bits | SNR | Uses two's complement for negatives |
| 3 | 2 bits | Reserved | I don't exactly understand the reason for leaving the extra 2 bits |
| 4 | 8 bits | Channel | Index mapping to frequency |

#### Field Details

**Uplink ID (12 bits):**
- Takes first 8 bits completely
- Plus the next 4 bits from the 2nd byte
- Range: 0-4095
- Used for correlation with downlinks

**Data Rate (4 bits):**
- Last 4 bits of the 2nd byte
- Range: 0-15
- Maps to modulation parameters

**RSSI (8 bits):**
- Stored in 3rd byte as a positive value
- Converted to negative value when being used
- Range: 0 to -255 dBm

**SNR (6 bits):**
- Uses only 6 bits to store values
- After the value 32, they subtract 64 to store it as a negative value within -1 and -31
- The extra 2 bits might be reserved for future use or alignment

**Channel (8 bits):**
- Last byte used to store the channel index
- Maps to a frequency channel from configuration

---

### 4.6 Helper Functions

In compressing the metadata, several helper functions are being called:

#### 4.6.1 get_uplink_id()

```rust
fn get_uplink_id() -> u16 {
    let mut uplink_id = UPLINK_ID.lock().unwrap();
    *uplink_id += 1;

    if *uplink_id > 4095 {
        *uplink_id = 0;
    }

    *uplink_id
}
```

**Purpose:** When the relay gateway receives an uplink from an end device, the concentrator provides the context field with the timing information. When sending a downlink response, this context must be returned.

In a mesh network, the downlink may be delayed and the relay needs to remember the relevant uplink. As I understood, this is the reason for storing the uplink ID. The memory allows storage of uplink context up to 2^12, which means 4096 entries.

This is one of the global variables being called:

```rust
static UPLINK_ID: Mutex<u16> = Mutex::new(0);
```

Mutex is used as a thread-safe counter so the variable will not be used elsewhere. If the number exceeds 4095, it will be wrapped around.

#### 4.6.2 store_uplink_context()

```rust
pub fn store_uplink_context(ctx: &[u8]) -> u16 {
    let uplink_id = get_uplink_id();
    let mut uplink_ctx = UPLINK_CONTEXT.lock().unwrap();
    uplink_ctx.insert(uplink_id, ctx.to_vec());
    uplink_id
}
```

Then the `store_uplink_context` function is used. Here they first call the `get_uplink_id` function to get a unique ID for the uplink, and then it will be inserted into the next global variable:

```rust
static UPLINK_CONTEXT: LazyLock<Mutex<HashMap<u16, Vec<u8>>>> =
    LazyLock::new(|| Mutex::new(HashMap::new()));
```

This is a HashMap wrapped in multiple layers for thread safety. The data storage uses the key of uplink ID and the value is the context bytes from the concentrator.

#### 4.6.3 modulation_to_dr()

The matching of data rate uses the `modulation_to_dr` helper function. They take the configuration file defined mappings and then map the parameters to one of the mappings and return the index.

**Example from EU868 config:**

```toml
[[mappings.data_rates]]
modulation = "LORA"
spreading_factor = 12
bandwidth = 125000
code_rate = "4/5"

[[mappings.data_rates]]
modulation = "LORA"
spreading_factor = 11
bandwidth = 125000
code_rate = "4/5"

[[mappings.data_rates]]
modulation = "LORA"
spreading_factor = 10
bandwidth = 125000
code_rate = "4/5"
```

#### 4.6.4 frequency_to_chan()

The next helper function is `frequency_to_chan`, which maps the frequency to a defined channel in the configuration file. As mentioned earlier, it returns an 8-bit value.

**Future Optimization:** We can maybe work with this part in the future. If the number of configured channels are low, this will use a low matching index and maybe push this to be a 4-byte header.

#### Step 3: Get Relay ID and Clone Payload

After this, they get the relay ID from the backend and clone the physical payload to complete the mesh packet structure.

#### Step 4: Calculate MIC

```rust
packet.set_mic(if conf.mesh.signing_key != Aes128Key::null() {
    conf.mesh.signing_key
} else {
    get_signing_key(conf.mesh.root_key)
})?;
```

After constructing the packet, the MIC (Message Integrity Code) is calculated and added.

#### Step 5: Prepare Mesh Transmission

```rust
let pl = gw::DownlinkFrame {
    downlink_id: getrandom::u32()?,
    items: vec![gw::DownlinkFrameItem {
        phy_payload: packet.to_vec()?,
        tx_info: Some(gw::DownlinkTxInfo {
            frequency: get_mesh_frequency(&conf)?,
            power: conf.mesh.tx_power,
            modulation: Some(helpers::data_rate_to_gw_modulation(
                &conf.mesh.data_rate,
                false,
            )),
            timing: Some(gw::Timing {
                parameters: Some(gw::timing::Parameters::Immediately(
                    gw::ImmediatelyTimingInfo {},
                )),
            }),
            ..Default::default()
        }),
        ..Default::default()
    }],
    ..Default::default()
};
```

This prepares the mesh packet for radio transmission:

- **Random ID:** Generated for this transmission
- **Mesh packet:** Serialized to bytes
- **Transmission parameters:** Set according to configuration

##### 4.6.5 get_mesh_frequency()

```rust
pub fn get_mesh_frequency(conf: &Configuration) -> Result<u32> {
    if conf.mesh.frequencies.is_empty() {
        return Err(anyhow!("No mesh frequencies are configured"));
    }

    let mut mesh_channel = MESH_CHANNEL.lock().unwrap();
    *mesh_channel += 1;

    if *mesh_channel >= conf.mesh.frequencies.len() {
        *mesh_channel = 0;
    }

    Ok(conf.mesh.frequencies[*mesh_channel])
}
```

`get_mesh_frequency` is the function where they check the configured channels from the mesh configuration and then choose the channel to transmit based on a round-robin format. Here, to keep track of the current number being used, another global variable is called:

```rust
static MESH_CHANNEL: Mutex<usize> = Mutex::new(0);
```

If the maximum number is the current number, it will be wrapped around and start from the beginning.

##### 4.6.6 data_rate_to_gw_modulation()

The next helper function is `data_rate_to_gw_modulation`, which converts the data rate index to the modulation parameters so the concentrator can use these parameters for the transmission.

#### Step 6: Set Timing and Transmit

Then the timing is set to be **immediate**, meaning the mesh packet will be sent immediately. After that, the transmission is logged and finished.

```rust
info!(
    "Relaying uplink LoRa frame, uplink_id: {}, downlink_id: {}, mesh_packet: {}",
    rx_info.uplink_id, pl.downlink_id, packet,
);
```

That explains how an uplink from the end device is being handled.

---

## 5. Mesh Packet Handling

### 5.1 handle_mesh() - Main Router

Then we will look at how to handle the mesh packets at Border or Relay gateways. For this, the `handle_mesh` function is implemented.

```rust
pub async fn handle_mesh(border_gateway: bool, pl: &gw::UplinkFrame) -> Result<()> {
    let conf = config::get();
    let mut packet = MeshPacket::from_slice(&pl.phy_payload)?;
    
    if !packet.validate_mic(if conf.mesh.signing_key != Aes128Key::null() {
        conf.mesh.signing_key
    } else {
        get_signing_key(conf.mesh.root_key)
    })? {
        warn!("Dropping packet, invalid MIC, mesh_packet: {}", packet);
        return Ok(());
    }

    if !PAYLOAD_CACHE.lock().unwrap().add((&packet).into()) {
        trace!(
            "Dropping packet as it has already been seen, mesh_packet: {}",
            packet
        );
        return Ok(());
    };
```

#### Step 1: Parse and Validate

First, load the gateway configuration file. Then clone the physical payload as a mutable variable, deserialize it into a mesh packet struct and extract the MHDR, payload, and check whether the MIC is valid. For this, either the configured signing key or the derived key from the root key is used.

#### Step 2: Duplicate Detection

To prevent the processing of the same packet, cache is being used here.

```rust
static PAYLOAD_CACHE: LazyLock<Mutex<Cache<PayloadCache>>> =
    LazyLock::new(|| Mutex::new(Cache::new(64)));
```

Every gateway has a local payload cache as a global variable. This has thread-safe initialization, protects cache from concurrent access by using mutex, and it is an LRU (Least Recently Used) cache holding packets with a maximum cache capacity of 64.

#### Step 3: Decrypt (If Needed)

After this, if the packet is an encrypted payload, it will be decrypted. The MIC is calculated on the encrypted payload, so we must validate it, then decrypt to access the actual payload data.

#### Step 4: Route Based on Gateway Type

```rust
match border_gateway {
    true => match packet.mhdr.payload_type {
        PayloadType::Uplink => proxy_uplink_mesh_packet(pl, packet).await,
        PayloadType::Event => proxy_event_mesh_packet(pl, packet).await,
        _ => Ok(()),
    },
    false => relay_mesh_packet(pl, packet).await,
}
```

This is the decision-making logic in this function:

1. First, we check whether the gateway is a border or relay gateway
2. If it is a **border gateway**, we check whether the received payload is an uplink mesh packet or an event mesh packet
3. If it is a **relay gateway**, the `relay_mesh_packet` function is called

---

### 5.2 Border Gateway: proxy_uplink_mesh_packet()

First, let's look at how the border gateway handles uplink mesh packets.

```rust
async fn proxy_uplink_mesh_packet(pl: &gw::UplinkFrame, packet: MeshPacket) -> Result<()> {
    let mesh_pl = match &packet.payload {
        Payload::Uplink(v) => v,
        _ => {
            return Err(anyhow!("Expected Uplink payload"));
        }
    };

    info!(
        "Unwrapping relayed uplink, uplink_id: {}, mesh_packet: {}",
        pl.rx_info.as_ref().map(|v| v.uplink_id).unwrap_or_default(),
        packet
    );
```

#### Step 1: Extract Payload

Extracts the uplink payload from the mesh packet and returns an error if not an uplink. Then logs an info-level statement that records when the border gateway unwraps a mesh-encapsulated uplink.

#### Step 2: Clone and Modify Frame

```rust
let mut pl = pl.clone();
```

Clones the mesh frame to modify it. The original `pl` is the LoRaWAN packet received from the mesh relay.

#### Step 3: Update RX Info

```rust
if let Some(rx_info) = &mut pl.rx_info {
    rx_info.gateway_id = hex::encode(backend::get_gateway_id().await?);

    rx_info
        .metadata
        .insert("hop_count".to_string(), (packet.mhdr.hop_count).to_string());
    rx_info
        .metadata
        .insert("relay_id".to_string(), hex::encode(mesh_pl.relay_id));

    rx_info.snr = mesh_pl.metadata.snr.into();
    rx_info.rssi = mesh_pl.metadata.rssi.into();
    rx_info.context = {
        let mut ctx = Vec::with_capacity(CTX_PREFIX.len() + 6);
        ctx.extend_from_slice(&CTX_PREFIX);
        ctx.extend_from_slice(&mesh_pl.relay_id);
        ctx.extend_from_slice(&mesh_pl.metadata.uplink_id.to_be_bytes());
        ctx
    };
}
```

Then RX info is modified:

- **Change gateway ID** to border gateway because the Network Server sees the border as the receiver
- **Add hop count and relay ID** to metadata
- **Replace RSSI and SNR** from the original values of the end device transmission to the relay
- **Add 9-byte identifier** of context for routing downlinks back

**Context Structure:**
```
[CTX_PREFIX][relay_id (4 bytes)][uplink_id (2 bytes)]
[1, 2, 3]   [aa, bb, cc, dd]    [0x04, 0x2A]
```

#### Step 4: Update TX Info

```rust
if let Some(tx_info) = &mut pl.tx_info {
    tx_info.frequency = helpers::chan_to_frequency(mesh_pl.metadata.channel)?;
    tx_info.modulation = Some(helpers::dr_to_modulation(mesh_pl.metadata.dr, false)?);
}
```

Restores the original transmission parameters.

#### Step 5: Restore PHYPayload

```rust
pl.phy_payload.clone_from(&mesh_pl.phy_payload);

let pl = gw::Event {
    event: Some(gw::event::Event::UplinkFrame(pl)),
};

proxy::send_event(pl).await
}
```

Restores the proprietary mesh packet bytes with the original LoRaWAN PHYPayload from the end device. Then wraps the reconstructed uplink in an Event and sends to the Network Server via ZMQ.

---

### 5.3 Border Gateway: proxy_event_mesh_packet()

The `proxy_event_mesh_packet` implements how the border gateway handles the sending of an event mesh packet to the network server. This unwraps mesh event packets at the border.

```rust
async fn proxy_event_mesh_packet(pl: &gw::UplinkFrame, packet: MeshPacket) -> Result<()> {
    let mesh_pl = match &packet.payload {
        Payload::Event(v) => v,
        _ => {
            return Err(anyhow!("Expected Heartbeat payload"));
        }
    };

    info!(
        "Unwrapping relay event packet, uplink_id: {}, mesh_packet: {}",
        pl.rx_info.as_ref().map(|v| v.uplink_id).unwrap_or_default(),
        packet
    );
```

#### Step 1: Extract Event Payload

Extracts the EventPayload from the mesh packet. Even though the error message says expected "Heartbeat", this code actually handles both heartbeat and proprietary events. Then records the event unwrapping as a log.

#### Step 2: Create Mesh Event

```rust
let event = gw::Event {
    event: Some(gw::event::Event::Mesh(gw::MeshEvent {
        gateway_id: hex::encode(backend::get_gateway_id().await?),
        relay_id: hex::encode(mesh_pl.relay_id),
        time: Some(mesh_pl.timestamp.into()),
        events: mesh_pl
            .events
            .iter()
            .map
```

After that, creates a MeshEvent containing metadata about the relay gateway and the events.

#### Step 3: Transform Events

```rust
.map(|e| gw::MeshEventItem {
    event: Some(match e {
        Event::Heartbeat(v) => {
            gw::mesh_event_item::Event::Heartbeat(gw::MeshEventHeartbeat {
                relay_path: v
                    .relay_path
                    .iter()
                    .map(|v| gw::MeshEventHeartbeatRelayPath {
                        relay_id: hex::encode(v.relay_id),
                        rssi: v.rssi.into(),
                        snr: v.snr.into(),
                    })
                    .collect(),
            })
        }
        Event::Proprietary(v) => {
            gw::mesh_event_item::Event::Proprietary(gw::MeshEventProprietary {
                event_type: v.0.into(),
                payload: v.1.clone(),
            })
        }
        Event::Encrypted(_) => panic!("Events must be decrypted first"),
    }),
})
.collect(),
    })),
};
```

This transforms internal event types into Protocol Buffer types:

**Heartbeat Events:**
- Includes relay path showing each hop with relay_id, RSSI, and SNR

**Proprietary Events:**
- Custom vendor-specific events with the type ID and payload

**Encrypted Events:**
- If an Encrypted event is encountered, it will panic because it should never occur here and should be decrypted earlier

#### Step 4: Send Event

```rust
proxy::send_event(event).await?;

Ok(())
}
```

Then send the event via ZMQ to the ChirpStack Network Server.

---

### 5.4 Relay Gateway: relay_mesh_packet() - The Flooding Engine

That is the procedure for how the Border gateway handles the received packet. Now let's look at how the Relay gateway handles this procedure through the `relay_mesh_packet` function.

```rust
async fn relay_mesh_packet(pl: &gw::UplinkFrame, mut packet: MeshPacket) -> Result<()> {
    let conf = config::get();
    let relay_id = backend::get_relay_id().await?;
    let rx_info = pl
        .rx_info
        .as_ref()
        .ok_or_else(|| anyhow!("rx_info is None"))?;
```

#### Initial Setup

First, reference the radio frame that contained the mesh packet. This function returns success or error. Then receive the global configuration which contains mesh frequencies, TX power, data rate, max hop count, and encryption keys.

After that, make an async call to get this gateway's relay identifier. Then extract the RX metadata from the received frame. This contains RSSI, SNR, timestamp, and gateway ID.

#### Match Payload Type

```rust
match &mut packet.payload
```

Call the received payload as a mutable reference in case we have to modify heartbeat relay paths. After that, we match with the four payload types and make decisions accordingly.

The four types are:
1. **Uplink** - The uplink from end device
2. **Downlink** - LoRaWAN downlink to end device
3. **Event** - Heartbeat/proprietary telemetry
4. **Command** - Remote management commands

#### Case 1: Uplink Payload

```rust
packets::Payload::Uplink(pl) => {
    if pl.relay_id == relay_id {
        trace!("Dropping packet as this relay was the sender");
        // Drop the packet, as we are the original sender.
        return Ok(());
    }
}
```

This is for the case when a gateway received an uplink, wrapped it in a mesh packet, and broadcast it. When this relay gateway hears it, first the gateway checks the relay ID. If they are identical, that means this relay was the original sender. So it drops to prevent processing its own packet. If not, it falls through to flooding logic.

#### Case 2: Downlink Payload

```rust
packets::Payload::Downlink(pl) => {
    if pl.relay_id == relay_id {
        let pl = gw::DownlinkFrame {
            downlink_id: getrandom::u32()?,
```

When the received payload is downlink, first the gateway checks the relay ID. If it matches, this is the target relay, so it can unwrap and send to the end device. If no match, it will fall through to flooding logic.

**When This Gateway is the Target:**

##### Step 1: Create DownlinkFrame

First, it creates a new DownlinkFrame to send to the concentrator and generates a random downlink ID for tracking/logging.

##### Step 2: Set Transmission Items

```rust
items: vec![gw::DownlinkFrameItem {
    phy_payload: pl.phy_payload.clone(),
    tx_info: Some(gw::DownlinkTxInfo {
        frequency: pl.metadata.frequency,
        power: helpers::index_to_tx_power(pl.metadata.tx_power)?,
        timing: Some(gw::Timing {
            parameters: Some(gw::timing::Parameters::Delay(
                gw::DelayTimingInfo {
                    delay: Some(prost_types::Duration {
                        seconds: pl.metadata.delay.into(),
                        ..Default::default()
                    }),
                },
            )),
        }),
        modulation: Some(helpers::dr_to_modulation(pl.metadata.dr, true)?),
        context: get_uplink_context(pl.metadata.uplink_id)?,
        ..Default::default()
    }),
    ..Default::default()
}],
```

Downlink can have multiple items (for retries), but mesh uses only one. Then for the item:

- **Extract PHYPayload:** The original LoRaWAN packet destined for the end device
- **Set frequency:** The end device's RX window frequency
- **Convert power:** Use helper function to convert power index back to dBm value
- **Set timing:** Delay timing for RX1 or RX2 windows
- **Set modulation:** Use helper function to get full modulation parameters
- **Retrieve context:** Use uplink_id to get the original uplink's radio context

The uplink_id is the key to uplink_context in the HashMap. By using the uplink ID, the gateway retrieves the original uplink's radio context stored when the uplink was first received. This contains the radio timestamp and concentrator internal state, allowing the radio to calculate precise delay from that original uplink timestamp.

##### Step 3: Set Gateway ID and Transmit

```rust
gateway_id: hex::encode(backend::get_gateway_id().await?),
..Default::default()
};

info!(
    "Unwrapping relayed downlink, downlink_id: {}, mesh_packet: {}",
    pl.downlink_id, packet
);
return helpers::tx_ack_to_err(&backend::send_downlink(pl).await?);
```

Set the gateway ID to this relay's ID. Then log the downlink and send it to the radio concentrator, which returns DownlinkTxAck. The `tx_ack_to_err` converts any non-OK statuses to Rust errors. This returns immediately without falling through to flooding logic.

#### Case 3: Event Payload

```rust
packets::Payload::Event(pl) => {
    if pl.relay_id == relay_id {
        trace!("Dropping packet as this relay was the sender");
        // Drop the packet, as we are the sender.
        return Ok(());
    }

    for event in &mut pl.events {
        if let Event::Heartbeat(v) = event {
            v.relay_path.push(packets::RelayPath {
                relay_id,
                rssi: rx_info.rssi as i16,
                snr: rx_info.snr as i8,
            });
        }
    }
}
```

First, it follows the same logic as uplink: check the relay ID and if they match, drop the packet. This prevents the relay from forwarding its own events.

Event payloads can contain multiple events. Then we go through the heartbeat event first. We modify the heartbeat event by appending this relay to the hop path. This creates a trail showing the mesh topology, with each hop recording relay ID, RSSI, and SNR.

**Question:** Where is this data being used? The heartbeat relay path data is sent to the ChirpStack Network Server, but this codebase doesn't process or visualize it. The actual data usage would be:
- NS would receive it via ZMQ
- UI would display mesh topology and signal quality
- There's no decision-making happening based on this relay path

#### Case 4: Command Payload

```rust
packets::Payload::Command(pl) => {
    if pl.relay_id == relay_id {
        let resp = commands::execute_commands(pl).await?;

        if !resp.is_empty() {
            events::send_events(resp).await?;
        }

        return Ok(());
    }
}
```

Commands are addressed to specific relays. First, we check whether this command is intended for this relay or not. If the command is intended for this relay:

1. Execute it (commands.rs script handles command execution)
2. If the commands produced responses, send them back as events (events.rs script handles this)
3. Return immediately without flooding

#### The Flooding Logic

Then we enter the flooding logic (reached when packet is not for this relay):

```rust
packet.mhdr.hop_count += 1;

packet.encrypt(get_encryption_key(conf.mesh.root_key))?;

packet.set_mic(if conf.mesh.signing_key != Aes128Key::null() {
    conf.mesh.signing_key
} else {
    get_signing_key(conf.mesh.root_key)
})?;

if packet.mhdr.hop_count > conf.mesh.max_hop_count {
    return Err(anyhow!("Max hop count exceeded"));
}
```

We reach here when:
- Uplink/Event packet is not from this relay
- Downlink/Command is not addressed to this relay

**Steps:**

1. **Increment hop count** to track propagation depth through mesh
2. **Re-encrypt** because the packet was decrypted earlier
3. **Recalculate MIC** because hop count changed and heartbeat path may be modified
4. **Check max hop count** to prevent infinite flooding (typically 3-5)

If max hop count is exceeded, the packet is dropped and returns an error.

#### Prepare Retransmission

```rust
let pl = gw::DownlinkFrame {
    downlink_id: getrandom::u32()?,
    items: vec![gw::DownlinkFrameItem {
        phy_payload: packet.to_vec()?,
        tx_info: Some(gw::DownlinkTxInfo {
            frequency: get_mesh_frequency(&conf)?,
            modulation: Some(helpers::data_rate_to_gw_modulation(
                &conf.mesh.data_rate,
                false,
            )),
            power: conf.mesh.tx_power,
            timing: Some(gw::Timing {
                parameters: Some(gw::timing::Parameters::Immediately(
                    gw::ImmediatelyTimingInfo {},
                )),
            }),
            ..Default::default()
        }),
        ..Default::default()
    }],
    ..Default::default()
};

info!(
    "Re-relaying mesh packet, downlink_id: {}, mesh_packet: {}",
    pl.downlink_id, packet
);
backend::mesh(pl).await
}
```

Then we build the retransmission frame:

- **Shadow earlier variable:** By using `pl` again, the earlier parameter is getting shadowed
- **Generate random ID** for this mesh transmission
- **Serialize mesh packet** to bytes
- **Set transmission data:**
  - Get mesh frequency (round-robin)
  - Use configured mesh data rate
  - Set transmission power
  - Transmit immediately without delay

**Reason for immediate transmission:**
- Flooding requires fast propagation
- No need for timing precision

Then log this operation and send to the backend, which sends to the radio concentrator.

---

## 6. Downlink Handling

### 6.1 handle_downlink() - Routing Decision

```rust
pub async fn handle_downlink(pl: gw::DownlinkFrame) -> Result<gw::DownlinkTxAck> {
    if let Some(first_item) = pl.items.first() {
        let tx_info = first_item
            .tx_info
            .as_ref()
            .ok_or_else(|| anyhow!("tx_info is None"))?;
        if tx_info.context.len() != CTX_PREFIX.len() + 6
            || !tx_info.context[0..CTX_PREFIX.len()].eq(&CTX_PREFIX)
        {
            return proxy_downlink_lora_packet(pl).await;
        }
    }

    relay_downlink_lora_packet(&pl).await
}
```

This function routes downlink packets from the network server, determining whether to proxy directly using the border gateway or relay via mesh.

**Input:** Downlink frame from ChirpStack Network Server containing:
- End device address
- Payload
- TX parameters
- Context

**Output:** Downlink TX acknowledgement indicating transmission success

#### Decision Process

The downlink may contain multiple items for retry scenarios. The script:

1. Takes the first downlink item
2. Extracts the TX info
3. Determines whether this downlink is for mesh-relayed or direct communication

**Context Formats:**
- **Regular context:** For direct communication (Border)
- **Mesh context:** `[CTX_PREFIX][Relay ID][uplink ID]`

**Routing Decision:**
- If context doesn't match mesh format → Call `proxy_downlink_lora_packet` (direct)
- If context matches mesh format → Call `relay_downlink_lora_packet` (mesh)

---

### 6.2 Border Gateway: proxy_downlink_lora_packet()

```rust
async fn proxy_downlink_lora_packet(pl: gw::DownlinkFrame) -> Result<gw::DownlinkTxAck> {
    info!(
        "Proxying LoRaWAN downlink, downlink: {}",
        helpers::format_downlink(&pl)?
    );
    backend::send_downlink(pl).await
}
```

**Purpose:** Border gateway sends downlink directly to end device (no mesh involved)

**Steps:**
1. Call `format_downlink` helper to format the packet
2. Log it
3. Send to backend, which sends to the concentrator

---

### 6.3 Border Gateway: relay_downlink_lora_packet()

```rust
async fn relay_downlink_lora_packet(pl: &gw::DownlinkFrame) -> Result<gw::DownlinkTxAck> {
    let conf = config::get();
```

By calling `relay_downlink_lora_packet`, the border gateway wraps Network Server downlinks in mesh packets and broadcasts them to relay gateways.

**Input:** Reference to downlink from Network Server with LoRaWAN payload and context

**Output:** Acknowledgement with status for each downlink item

This function is called by the border gateway when `handle_downlink` detects mesh context.

#### Step 1: Initialize Acknowledgements

```rust
let mut tx_ack_items: Vec<gw::DownlinkTxAckItem> = pl
    .items
    .iter()
    .map(|_| gw::DownlinkTxAckItem {
        status: gw::TxAckStatus::Ignored.into(),
    })
    .collect();
```

Gets the configuration and initializes acknowledgement by pre-allocating the acknowledgement vector. The Network Server may send multiple items. Only the first successful item will be transmitted; remaining items stay ignored.

#### Step 2: Process Each Item

```rust
for (i, downlink_item) in pl.items.iter().enumerate() {
```

Proceeds through each downlink item sequentially until one succeeds.

#### Step 3: Extract Information

For each item, extract transmission, modulation, timing, and delay information:

```rust
let tx_info = downlink_item
    .tx_info
    .as_ref()
    .ok_or_else(|| anyhow!("tx_info is None"))?;
let modulation = tx_info
    .modulation
    .as_ref()
    .ok_or_else(|| anyhow!("modulation is None"))?;
let timing = tx_info
    .timing
    .as_ref()
    .ok_or_else(|| anyhow!("timing is None"))?;
let delay = match &timing.parameters {
    Some(gw::timing::Parameters::Delay(v)) => v
        .delay
        .as_ref()
        .map(|v| v.seconds as u8)
        .unwrap_or_default(),
    _ => {
        return Err(anyhow!("Only Delay timing is supported"));
    }
};
```

Pattern match on timing type and extract the delay field. If NOT delay type, return an error.

**Assumption:** The gateway may or may not be configured for GPS timing and cannot send the downlink immediately because the relay needs to calculate correct RX window timing relative to the original uplink.

#### Step 4: Extract Context

```rust
let ctx = tx_info
    .context
    .get(CTX_PREFIX.len()..CTX_PREFIX.len() + 6)
    .ok_or_else(|| anyhow!("context does not contain enough bytes"))?;
```

Extract routing information from the context field (6 bytes after prefix).

#### Step 5: Create Mesh Packet

```rust
let mut packet = packets::MeshPacket {
    mhdr: packets::MHDR {
        payload_type: packets::PayloadType::Downlink,
        hop_count: 1,
    },
    payload: packets::Payload::Downlink(packets::DownlinkPayload {
        phy_payload: downlink_item.phy_payload.clone(),
        relay_id: {
            let mut b: [u8; 4] = [0; 4];
            b.copy_from_slice(&ctx[0..4]);
            b
        },
        metadata: DownlinkMetadata {
            uplink_id: {
                let mut b: [u8; 2] = [0; 2];
                b.copy_from_slice(&ctx[4..6]);
                u16::from_be_bytes(b)
            },
            dr: helpers::modulation_to_dr(modulation)?,
            frequency: tx_info.frequency,
            tx_power: helpers::tx_power_to_index(tx_info.power)?,
            delay,
        },
    }),
    mic: None,
};
```

Create a new mesh packet structure:

- **Mesh header:** Payload type as downlink, hop count set to 1 (first hop)
- **Payload:** Entire LoRaWAN packet embedded inside
- **Target relay ID:** Set from context
- **Metadata:**
  - Uplink ID (key to retrieve original uplink context)
  - Convert modulation parameters, frequency, and transmission power to indexes
  - Add delay parameter

#### Step 6: Calculate MIC

```rust
packet.set_mic(if conf.mesh.signing_key != Aes128Key::null() {
    conf.mesh.signing_key
} else {
    get_signing_key(conf.mesh.root_key)
})?;
```

Calculate the MIC and sign the mesh packet.

#### Step 7: Prepare Frame

```rust
let pl = gw::DownlinkFrame {
    downlink_id: pl.downlink_id,
    items: vec![gw::DownlinkFrameItem {
        phy_payload: packet.to_vec()?,
        tx_info: Some(gw::DownlinkTxInfo {
            frequency: get_mesh_frequency(&conf)?,
            power: conf.mesh.tx_power,
            modulation: Some(helpers::data_rate_to_gw_modulation(
                &conf.mesh.data_rate,
                false,
            )),
            timing: Some(gw::Timing {
                parameters: Some(gw::timing::Parameters::Immediately(
                    gw::ImmediatelyTimingInfo {},
                )),
            }),
            ..Default::default()
        }),
        ..Default::default()
    }],
    ..Default::default()
};
```

Create frame for the radio concentrator:
- Use downlink ID from Network Server
- Convert mesh packet struct to bytes
- Set transmission parameters for mesh broadcast
- Get mesh frequency, transmission power, and data rate
- Set timing to immediately transmit to mesh

#### Step 8: Log and Transmit

```rust
info!(
    "Sending downlink frame as relayed downlink, downlink_id: {}, mesh_packet: {}",
    pl.downlink_id, packet
);
```

Log the mesh broadcast with downlink ID and mesh packet details.

#### Step 9: Handle Result

```rust
match backend::mesh(pl).await {
    Ok(_) => {
        tx_ack_items[i].status = gw::TxAckStatus::Ok.into();
        break;
    }
    Err(e) => {
        warn!("Relay downlink failed, error: {}", e);
        tx_ack_items[i].status = gw::TxAckStatus::InternalError.into();
    }
}
```

On successful transmission, mark the item's status as OK and stop processing remaining items. For error case, log warning with error details and loop continues to try the next item.

#### Step 10: Return Acknowledgement

```rust
Ok(gw::DownlinkTxAck {
    gateway_id: pl.gateway_id.clone(),
    downlink_id: pl.downlink_id,
    items: tx_ack_items,
    ..Default::default()
})
```

Return acknowledgement to Network Server with status for each item.

---

## 7. Command System

### 7.1 send_mesh_command()

The Border gateway sends remote management commands to specific relay gateways via mesh network using the `send_mesh_command` function.

```rust
pub async fn send_mesh_command(pl: gw::MeshCommand) -> Result<()> {
    let conf = config::get();

    let mut packet = packets::MeshPacket {
        mhdr: packets::MHDR {
            payload_type: packets::PayloadType::Command,
            hop_count: 1,
        },
        payload: packets::Payload::Command(packets::CommandPayload {
            timestamp: SystemTime::now(),
            relay_id: {
                let mut relay_id: [u8; 4] = [0; 4];
                hex::decode_to_slice(&pl.relay_id, &mut relay_id)?;
                relay_id
            },
            commands: pl
                .commands
                .iter()
                .filter_map(|v| {
                    v.command
                        .as_ref()
                        .map(|gw::mesh_command_item::Command::Proprietary(v)| {
                            packets::Command::Proprietary((v.command_type as u8, v.payload.clone()))
                        })
                })
                .collect(),
        }),
        mic: None,
    };
    
    packet.encrypt(get_encryption_key(conf.mesh.root_key))?;
    
    packet.set_mic(if conf.mesh.signing_key != Aes128Key::null() {
        conf.mesh.signing_key
    } else {
        get_signing_key(conf.mesh.root_key)
    })?;
```

This follows the same procedure as the previous downlink packet. Additionally, in this code:

- **Set timestamp** as the current system time when the command was created
- **Set parameters** and create the command payload
- **Encrypt** the command payload
- **Calculate MIC** to sign the packet

After that, create the downlink payload and transmit immediately to the mesh, then log.

The complete flow involves flooding through the mesh network until the target relay (matching relay_id) receives and executes the command.

---

## 8. Critical Analysis

### 8.1 Key Design Decisions

#### Flooding Protocol
- **Simplicity:** No routing tables, no neighbor discovery
- **Trade-off:** Network efficiency vs. implementation complexity
- **Suitable for:** Sparse deployments with few hops

#### No Acknowledgements
- **Fire-and-forget:** No confirmation at mesh layer
- **Implication:** Relies entirely on LoRa physical layer reliability
- **Risk:** Silent failures with no notification

#### Cache-Based Deduplication
- **64-entry LRU cache:** May be insufficient for busy networks
- **Vulnerability:** Old packets can loop if cache overflows
- **No time-based expiry:** Relies solely on LRU eviction

### 8.2 Overhead Analysis

**Mesh Packet Overhead:**
- MHDR: 1 byte
- Metadata: 5 bytes
- Relay ID: 4 bytes
- MIC: 4 bytes
- **Total: 14 bytes**

**Maximum Payload at SF12:**
- Total available: 51 bytes
- Overhead: 14 bytes
- **Available for LoRaWAN PHYPayload: 37 bytes**

**Question:** Do end devices need to be aware of this constraint?

### 8.3 Timing Constraints

**LoRaWAN Class A Windows:**
- RX1: Opens 1 second after uplink
- RX2: Opens 2 seconds after uplink
- **Total budget: ~2 seconds for complete round-trip**

**Mesh Latency Budget:**
```
Uplink processing: ~50ms
Mesh flooding (1-2 hops): ~100-200ms
Border processing: ~50ms
Network Server: ~200-400ms
Downlink mesh flooding: ~100-200ms
Relay unwrapping: ~50ms
---
Total: ~550-950ms (leaves margin for RX1)
```

If total latency exceeds 2 seconds, downlink is lost forever.

### 8.4 Memory Considerations

**UPLINK_CONTEXT HashMap:**
- **No cleanup mechanism** - entries persist indefinitely
- Relies on uplink_id wrap (4096 entries max)
- **Should implement:** Time-based expiry after RX2 window

**PAYLOAD_CACHE:**
- 64 entries may be insufficient for dense networks
- Cache eviction can allow packet reprocessing
- **Trade-off:** Memory usage vs. loop prevention

### 8.5 Collision Problem

**Mesh Frequency Selection:**
- Each gateway has independent counter
- Round-robin selection not synchronized
- Immediate transmission with no backoff

**Collision Scenario:**
```
Multiple relays receive same packet
→ All select frequency independently
→ All transmit immediately
→ High probability of same frequency
→ RF collision → Packet loss
```

**Mitigation Factors:**
- Multiple configured frequencies
- Natural timing variance
- Sparse network deployment
- LoRa collision tolerance

### 8.6 Security Considerations

**Strengths:**
- MIC prevents tampering
- Encryption for sensitive payloads (Events/Commands)
- Silent drops prevent DoS feedback

**Weaknesses:**
- No sender authentication beyond shared key
- Replay attacks possible if cache evicts early
- No timestamp validation visible in code

### 8.7 Data Usage Questions

**Heartbeat Relay Path:**
- Collected and sent to Network Server
- Shows topology: relay_id, RSSI, SNR for each hop
- **Question:** How is this data used?
- **Missing:** Network visualization, route quality assessment

### 8.8 Scalability Analysis

**Current Design Indicators:**
- 64-entry cache
- Typical max hops: 3-5
- No route optimization
- Immediate flooding

**Conclusion:** Designed for small, sparse networks (rural/remote deployments)

**Not suitable for:**
- Dense urban deployments
- Large-scale networks (>10 gateways in range)
- High-traffic scenarios

---

## 9. Open Questions and Future Work

### 9.1 Cache Implementation
- **Q:** What exactly is hashed for PayloadCache? Full packet or specific fields?
- **Q:** Why 64 entries specifically? Empirical testing?
- **Q:** Is there hidden time-based expiry in Cache implementation?

### 9.2 Context Management
- **Q:** How long should context be retained?
- **Q:** What happens if downlink arrives after context is lost?
- **A:** uplink_id wraps at 4095 → potential HashMap collision
- **A:** No cleanup mechanism confirmed → memory leak

### 9.3 Collision Handling
- **Q:** Multiple relays receive same packet - race condition on cache?
- **Q:** Collision probability at physical layer?
- **Q:** Any CSMA/CA at mesh layer?
- **A:** No collision avoidance implemented

### 9.4 Scalability
- **Q:** Maximum supported mesh size?
- **Q:** Packet loss statistics in production?
- **Q:** Real-world hop count distribution?

### 9.5 Reliability
- **Q:** Why no retries or ACKs at mesh layer?
- **Q:** How does this integrate with LoRaWAN confirmed uplinks?
- **Q:** Silent failure handling strategy?

### 9.6 Frequency Management
- **Q:** How many mesh frequencies typically configured?
- **Q:** Does channel hopping help or hurt flooding?
- **Q:** Collision probability with single vs. multiple frequencies?

### 9.7 Command System
- **Q:** How do relay event responses get routed back?
- **Q:** Correlation between commands and response events?
- **Q:** Command timeout handling?

### 9.8 Payload Size Constraints
- **Q:** Are end devices aware of 37-byte PHYPayload limit?
- **Q:** Should mesh support adaptive data rate?
- **Q:** Fragmentation/reassembly consideration?

### 9.9 Future Optimizations
- **Potential improvements:**
  - Add time-based context cleanup
  - Increase cache size or implement hierarchical caching
  - Add random backoff to reduce collisions
  - Implement route quality metrics
  - Add mesh-layer acknowledgements for critical packets
  - Visualize heartbeat topology data

---

## Conclusion

This comprehensive analysis reveals a simple, elegant mesh protocol optimized for sparse, rural deployments. The flooding-based approach trades network efficiency for implementation simplicity, making it suitable for scenarios with limited relay density.

Key strengths include simplicity, minimal state management, and effective integration with LoRaWAN. However, scalability concerns, collision risks, and lack of acknowledgements limit its applicability to dense or mission-critical deployments.

The protocol demonstrates thoughtful compression of metadata and clever use of context for routing, but would benefit from addressing memory leaks, implementing collision avoidance, and providing better observability of mesh performance.

---

**End of Document**
