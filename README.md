# ATITFinalProject

> **Turn anything physical into a game controller.**

---

## What Is This?

ATITFinalProject is a **proof-of-concept IoT multiplayer gaming system**. Instead of
using a keyboard or gamepad, players control a game by pressing a **physical button
wired to a microcontroller**. The button press travels through a cloud pipeline and
moves a character on screen in real time, for every player connected to the session.

The current POC is a **multiplayer 3D Flappy Bird** — each player has a physical ESP32
button. Press it and your bird flaps. Every connected client sees every bird, in sync.

This is a **final project** exploring the intersection of IoT, cloud networking, and
real-time multiplayer game development. The goal is not just to make a game — it is
to design and implement a clean, extensible architecture that could support many
different games and many different types of physical input devices.

---

## Repositories

This umbrella repository contains architecture documentation and the protocol
specification. The three component repositories are:

| Repository | Description | Stack |
|---|---|---|
| [ATITFinalProject-firmware](https://github.com/SupakornSJB/atit_micropython) | MicroPython firmware for ESP32 controllers | MicroPython |
| [ATITFinalProject-server](https://github.com/SupakornSJB/ATITInfraNode) | ASP.NET Core relay server — MQTT to gRPC | C# / .NET 8 |
| [ATITFinalProject-unity](https://github.com/SupakornSJB/ATITProject) | Unity dedicated server + game client | C# / Unity 6 LTS |

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Why This Architecture?](#why-this-architecture)
- [Technology Choices Explained](#technology-choices-explained)
- [Protocol Specification](#protocol-specification)
- [Key Design Decisions](#key-design-decisions)
- [Protocol Version History](#protocol-version-history)
- [Getting Started](#getting-started)
- [Project Status](#project-status)

---

## System Architecture

The system has four layers. Each layer has a single, well-defined responsibility and
communicates with adjacent layers through a documented protocol.

```
╔══════════════════════════════════════════════════════════════════╗
║  PHYSICAL LAYER                                                  ║
║                                                                  ║
║   ESP32 #1            ESP32 #2            ESP32 #N               ║
║   [ btn1 ][ knob1 ]   [ btn1 ][ btn2 ]   [ btn1 ]               ║
║                                                                  ║
║   MicroPython firmware — reads GPIO/ADC, publishes MQTT events   ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  MQTT over TLS (port 8883)
                   │  topic: game/devices/{deviceId}/slot/{slotId}
                   │  broker: HiveMQ Cloud
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║  RELAY LAYER                                                     ║
║                                                                  ║
║   ASP.NET Core Server                                            ║
║   • Subscribes to all device slot topics (wildcard)              ║
║   • Resolves (deviceId, slotId) → (playerId, inputName)          ║
║   • Authenticates devices, validates sessions                    ║
║   • Normalizes analog values (ADC range → 0.0–1.0)               ║
║   • Buffers and rate-limits high-frequency sensor input          ║
║   • Forwards clean, resolved events to Unity over gRPC           ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  gRPC (HTTP/2 + Protobuf)
                   │  SendInputBatch(events[])
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║  GAME SERVER LAYER                                               ║
║                                                                  ║
║   Unity Dedicated Server (headless, no graphics)                 ║
║   • Embedded Kestrel gRPC server (background thread)             ║
║   • Thread-safe ConcurrentQueue bridges gRPC → main thread       ║
║   • Authoritative game state via Netcode for GameObjects (NGO)   ║
║   • Replicates game state to all clients                         ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  Netcode for GameObjects (NGO)
                   │  Unity Transport (UTP), port 7777
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║  CLIENT LAYER                                                    ║
║                                                                  ║
║   Unity Game Clients (one per player)                            ║
║   • Renders the game in real time                                ║
║   • Receives replicated state from the dedicated server          ║
║   • No direct connection to IoT devices or relay server          ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Why This Architecture?

This section is intentionally detailed — a core goal of this project is to demonstrate
understanding of *why* each architectural choice was made, not just *what* was built.

### Why a relay server instead of ESP32 → Unity directly?

The most natural question is: why not skip the ASP.NET server and have the ESP32
publish MQTT events that Unity subscribes to directly?

**Security boundary.** The Unity dedicated server is a private, trusted service. It
should never be directly internet-facing, nor should it be responsible for
authenticating IoT devices. The relay server is the only public-facing entry point —
it handles device authentication, session validation, and player mapping before Unity
sees a single event. Without it, Unity would need to become an IoT backend, which is
the wrong responsibility for a game server.

**Protocol translation.** MQTT is a pub/sub broker protocol designed for constrained
IoT devices. Unity's multiplayer stack (NGO/UTP) is a direct-connection transport.
These are fundamentally different network models. The relay server translates between
them — it is the impedance matching layer between the IoT world and the game world.

**Separation of concerns.** Device configuration — which button belongs to which
player, what action a knob represents — is infrastructure config, not game logic.
Putting it in Unity would mean rebuilding and redeploying the game server whenever
device assignments change. In the relay server it is a config file change with no
rebuild required on either side.

**Independent scaling.** The relay server is stateless and scales horizontally. The
Unity dedicated server is stateful and scales by spinning up new instances per game
session. Merging them means you cannot scale one without the other.

### Why MQTT for ESP32 → Server?

MQTT was chosen over HTTP REST for the IoT leg for several reasons. The persistent
TCP connection means there is no handshake overhead per event — latency is consistently
5–20ms compared to 50–200ms for a new HTTP connection on each button press. MQTT is
also specifically designed for constrained devices: the client library (`umqtt.simple`)
is built into MicroPython, the per-message header is ~2 bytes versus full HTTP headers,
and delivery guarantees (QoS levels) are built-in without manual retry logic in firmware.

HTTP REST would have been simpler to implement but would have added perceptible latency
to every button press, which fundamentally breaks the feel of a physical controller.

### Why gRPC for Server → Unity?

gRPC was chosen over WebSocket or raw TCP for the relay-to-Unity leg because of its
strongly typed contract model. The `.proto` file is a machine-verified contract shared
between the relay server (gRPC client) and Unity (gRPC server) — both generate C#
stubs from the same file, so serialization mismatches are caught at compile time rather
than at runtime. HTTP/2 multiplexing also allows multiple in-flight calls to share one
connection, which is efficient for batched input events.

The tradeoff is setup complexity: gRPC requires `protoc` for stub generation and
careful IL2CPP stripping configuration for Unity dedicated server builds. This
complexity is justified because the protocol boundary between the relay server and
Unity is the highest-stakes interface in the system — it is where two completely
different codebases must agree on a data format.

### Why a dedicated server instead of a listen server?

A dedicated server (headless Unity build, no rendering) was chosen over a listen server
(one player also hosts) because the game session is driven by IoT device events, not
by any player's client. There is no natural "host player" — the server is an
independent process. A dedicated server can also be cloud-hosted and always available
regardless of which players are connected, and NGO's server-authoritative model
(where the server owns all game state) is a natural fit for input that arrives
out-of-band via gRPC rather than from a connected client.

---

## Technology Choices Explained

| Technology | Used for | Why chosen | Considered alternatives |
|---|---|---|---|
| **ESP32** | Physical controller hardware | WiFi built-in, MicroPython support, wide GPIO availability, low cost | Raspberry Pi Pico W (less RAM efficient), Arduino (no native WiFi) |
| **MicroPython** | Firmware language | Python syntax; `umqtt.simple` and `ussl` built-in; fast iteration | C/C++ with ESP-IDF (more performant but significantly more complex) |
| **MQTT** | ESP32 → broker transport | Designed for constrained IoT devices; persistent connection; low overhead per message | HTTP REST (higher latency per event), WebSocket (more complex on MicroPython) |
| **HiveMQ Cloud** | MQTT broker | Free tier sufficient for POC; managed TLS; reliable uptime | Mosquitto self-hosted (more control, more ops overhead), EMQX Cloud |
| **ASP.NET Core (.NET 8)** | Relay server | First-class gRPC support; MQTTnet library mature; C# across server and Unity | Node.js (weaker gRPC story), Go (less familiar) |
| **gRPC (Protobuf)** | Server → Unity transport | Strongly typed shared contract; HTTP/2 multiplexing; code generation for both sides | WebSocket (no schema enforcement), raw TCP (no framing or typing), REST (latency) |
| **Unity 6 LTS** | Game engine | NGO is a first-party Unity package; Dedicated Server build target built-in; URP included | Godot (weaker multiplayer tooling), Unreal (too heavy for a POC) |
| **Netcode for GameObjects** | Unity multiplayer | First-party Unity package; server-authoritative model fits IoT-driven input well | Mirror (community-maintained), Fishnet (more features but more complexity for a POC) |
| **Unity Transport (UTP)** | NGO transport layer | Default NGO transport; supports Unity Relay for NAT traversal if needed | LLAPI (deprecated), custom TCP socket |

---

## Protocol Specification

### MQTT Topic

```
game/devices/{deviceId}/slot/{slotId}
```

| Segment | Example | Description |
|---|---|---|
| `deviceId` | `esp32-bedroom` | Unique identifier per physical ESP32 device |
| `slotId` | `btn1`, `knob1` | Physical input slot identifier on that device |

The topic carries **physical identity only**. `slotId` is a hardware label, not a game
concept. Semantic meaning (`inputName`) and player assignment (`playerId`) are resolved
server-side from the device registry and never appear in the topic or firmware payload.

Server wildcard subscription: `game/devices/+/slot/+`

### MQTT Payload

```json
{
  "deviceId": "esp32-bedroom",
  "slotId": "btn1",
  "inputType": "button",
  "event": "pressed",
  "value": 1.0,
  "timestamp": 1234567890,
  "sequenceNumber": 4821
}
```

| Field | Type | Values | Description |
|---|---|---|---|
| `deviceId` | string | | Matches the topic segment |
| `slotId` | string | | Physical slot identifier — matches topic segment |
| `inputType` | string | `"button"`, `"knob"`, `"sensor"` | Hardware type of the slot |
| `event` | string | `"pressed"`, `"released"`, `"change"` | What happened |
| `value` | float | `0.0–1.0` | `1.0`/`0.0` for buttons; normalized range for analog inputs |
| `timestamp` | int64 | | Seconds since epoch (or since boot if NTP unavailable) |
| `sequenceNumber` | int64 | | Monotonic per-device counter; resets only on device reboot |

### Device Registry (server-side config)

The registry is the **only place** in the entire system where physical hardware
identity and game semantic meaning are linked together:

```json
{
  "DeviceRegistry": {
    "devices": {
      "esp32-bedroom": {
        "defaultPlayerId": "player-1",
        "inputs": {
          "btn1":  { "type": "button", "inputName": "jump" },
          "knob1": { "type": "knob",   "inputName": "speed", "minRaw": 0, "maxRaw": 4095 }
        }
      },
      "esp32-shared": {
        "defaultPlayerId": null,
        "inputs": {
          "btn1": { "type": "button", "inputName": "jump", "playerId": "player-2" },
          "btn2": { "type": "button", "inputName": "jump", "playerId": "player-3" }
        }
      }
    }
  }
}
```

`btn1` on `esp32-shared` and `btn1` on `esp32-bedroom` are completely independent —
`slotId` is scoped per device, not globally unique across the system.

### gRPC Contract

The `.proto` file is the machine-verified contract between the relay server (gRPC
client) and the Unity dedicated server (gRPC server). Both sides generate C# stubs
from this single file. Any change requires coordinated regeneration on both sides.

```proto
syntax = "proto3";

option csharp_namespace = "GameController.V2";

package game_controller.v2;

service GameControllerService {
  rpc SendInputEvent (InputEventRequest)  returns (InputEventResponse);
  rpc SendInputBatch (InputBatchRequest)  returns (InputEventResponse);
}

message InputEventRequest {
  string player_id        = 1;  // resolved by relay server from registry
  string device_id        = 2;  // retained for diagnostics only
  string input_name       = 3;  // semantic label, resolved by relay server
  string input_type       = 4;
  string event_type       = 5;
  float  value            = 6;  // always normalized 0.0–1.0
  int64  timestamp        = 7;
  int64  sequence_number  = 8;
}

message InputBatchRequest {
  repeated InputEventRequest events = 1;
}

message InputEventResponse {
  bool   accepted = 1;
  string message  = 2;
}
```

---

## Key Design Decisions

### Physical slot identity is separated from semantic input name

The firmware uses `slotId` — a physical hardware label (`btn1`, `knob1`). The game
uses `inputName` — a semantic label (`"jump"`, `"speed"`). These are linked only in
the server registry. This means reassigning a button to a different player or renaming
a game action requires only a registry config change — no firmware reflash, no Unity
rebuild, no redeployment of either component.

This was introduced in Phase 2.1 to fix a naming collision that arose when two players
shared one device but both had a `"jump"` input. With semantic names in the firmware,
naming them both `"jump"` was impossible. With physical slot IDs, they are simply
`btn1` and `btn2` — the distinction is hardware topology, not game semantics.

### One device can serve multiple players

A single ESP32 can have different slots mapped to different players in the registry.
The firmware is completely unaware of this arrangement — it just publishes slot events.
The relay server resolves player ownership per `(deviceId, slotId)` pair. This enables
shared physical controllers without any special firmware support.

### The relay server is the authentication boundary

The Unity dedicated server only accepts gRPC connections from the relay server and is
not internet-facing. This means Unity receives only pre-validated, fully-resolved
events and has no authentication logic of its own. The relay server issues and
validates device credentials, checks session membership, and rejects unknown devices
before any event reaches the game layer.

### Configurable input buffering

The relay server supports a `BufferWindowMs` setting. At `0` (default) every event
is forwarded to Unity immediately with no added latency. At `16ms`, events accumulate
and flush as a batch once per frame — useful for high-frequency knob or sensor inputs
where frame-aligned delivery matters more than minimum latency. This tradeoff is
configurable rather than hardcoded because different games have different requirements.

### Thread safety in Unity is a hard constraint

gRPC service methods in Unity run on Kestrel background threads. Unity's API is
main-thread only. These two facts cannot be reconciled by careful coding — they require
a structural solution. The bridge is a `ConcurrentQueue<InputBatchEnvelope>`: gRPC
handlers only enqueue, Unity's `Update()` loop only dequeues. This constraint is
documented explicitly in the Unity codebase and must never be violated.

---

## Protocol Version History

| Version | Status | Summary of Changes |
|---|---|---|
| **Phase 1** | ✅ Complete | Single button per device. One player per device. Topic: `game/controllers/{deviceId}/button`. Minimal JSON payload. |
| **Phase 2** | 🟡 In progress | Multi-input per device (button, knob, sensor). Multi-device per player. Shared devices. Topic: `game/devices/{deviceId}/input/{inputName}`. Added `inputType`, `value`, `sequenceNumber`, input buffering, rate limiting. |
| **Phase 2.1** | 🟡 In progress | Protocol fix — separated `slotId` (physical) from `inputName` (semantic). Topic: `game/devices/{deviceId}/slot/{slotId}`. `inputName` moved exclusively to server-side registry. No Unity changes required. |

---

## Getting Started

### Prerequisites

- ESP32 microcontroller flashed with MicroPython (latest stable)
- [HiveMQ Cloud](https://www.hivemq.com/mqtt-cloud-broker/) account (free tier is sufficient)
- [.NET 8 SDK](https://dotnet.microsoft.com/download)
- [Unity 6 LTS](https://unity.com/releases/lts) (6000.3.11f1) with NGO 2.9.1

### 1. Firmware

```bash
pip install mpremote esptool

# Flash MicroPython to ESP32 (if not already done)
esptool.py --chip esp32 erase_flash
esptool.py --chip esp32 write_flash -z 0x1000 esp32-firmware.bin

# See ATITFinalProject-firmware for full configuration instructions
git clone https://github.com/ATITFinalProject/ATITFinalProject-firmware
```

### 2. Relay Server

```bash
git clone https://github.com/ATITFinalProject/ATITFinalProject-server
cd ATITFinalProject-server

dotnet user-secrets set "Mqtt:Username" "your-hivemq-user"
dotnet user-secrets set "Mqtt:Password" "your-hivemq-password"

dotnet run
```

### 3. Unity Dedicated Server

Open `ATITFinalProject-unity` in Unity 6. Build as a Dedicated Server:

```
File > Build Settings > Dedicated Server > Build
```

Run the server build:

```bash
./ATITProject.x86_64 -batchmode -nographics -grpcPort 9090 -gamePort 7777
```

### 4. Unity Game Client

Run Play Mode in the Unity Editor, or build a standalone client binary and connect
to the dedicated server on port `7777`.

---

## Project Status

This is an **early proof of concept** built as a university final project.

| Component | Phase | Status |
|---|---|---|
| ESP32 Firmware | Phase 2.1 | 🟡 In development |
| ASP.NET Relay Server | Phase 2.1 | 🟡 In development |
| Unity Dedicated Server + Client | Phase 1 ✅ POC complete, Phase 2 🟡 | In progress |
| Architecture & Protocol Documentation | All phases | 🟢 Up to date |

---

## License

MIT — see [LICENSE](./LICENSE)
