# ATITFinalProject

> **Turn anything physical into a game controller.**

An IoT multiplayer game where the controllers are **ESP32 microcontrollers** — thumbsticks,
knobs and buttons wired to a board — and the room itself is part of the game: a humidity
sensor changes how fast enemies spawn, and a distance sensor changes the game speed.

The current game is a **two-player co-op space shooter** built in Unity. Every input travels
over a secured pipeline — **MQTT → relay server → gRPC → Unity dedicated server** — with
**mutual TLS on every network hop**, server-side intrusion detection, and an admin dashboard
that shows attacks as they happen.

This is a final project exploring IoT, real-time multiplayer networking and security. The goal
was not just a game but a clean, extensible architecture: the firmware, relay and Unity layers
are each split into a reusable, game-agnostic library and a thin game-specific layer.

---

## Repositories

This umbrella repository holds the architecture overview, the protocol specification and the
[setup guide](docs/setup.md). Each component lives in its own repository and is included here
as a **git submodule**, pinned to a set of versions that work together:

| Repository | Component | Stack |
|---|---|---|
| [atit_micropython](https://github.com/SupakornSJB/atit_micropython) | ESP32 controller firmware + device certificate scripts | MicroPython |
| [ATITInfraNode](https://github.com/SupakornSJB/ATITInfraNode) | Relay server: MQTT → gRPC, device registry, intrusion detection, dashboard API | C# / ASP.NET Core 8 |
| [ATITProject](https://github.com/SupakornSJB/ATITProject) | Unity dedicated server + spectator client | C# / Unity 6 LTS |
| [atit-dashboard](https://github.com/SupakornSJB/atit-dashboard) | Admin security dashboard: live alerts, quarantine, round control | React + TypeScript |

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Security](#security)
- [Why This Architecture?](#why-this-architecture)
- [Technology Choices](#technology-choices)
- [Protocol Specification](#protocol-specification)
- [Key Design Decisions](#key-design-decisions)
- [Project Phases](#project-phases)
- [Getting Started](#getting-started)
- [Project Status](#project-status)

---

## System Architecture

Each layer has one responsibility and talks to its neighbours through a documented protocol.

```
╔══════════════════════════════════════════════════════════════════╗
║  PHYSICAL LAYER — ESP32 controllers (MicroPython)                ║
║                                                                  ║
║   ESP32 #1                          ESP32 #2                     ║
║   thumbstick · round buttons        knob · up / down / fire      ║
║   DHT11 · HC-SR04 ultrasonic                                     ║
║                                                                  ║
║   Reads GPIO / ADC / sensors, publishes one event per change     ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  MQTT over mutual TLS — each board has its own client cert
                   │  topic: game/devices/{deviceId}/slot/{slotId}
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║  BROKER — Mosquitto (self-hosted)                                ║
║   • require_certificate: no CA-signed cert, no connection        ║
║   • cert CN = MQTT username; ACL: a device may only publish      ║
║     under game/devices/<its CN>/#                                ║
╚══════════════════╤═══════════════════════════════════════════════╝
                   │  MQTT over mutual TLS (relay cert, read-all ACL)
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║  RELAY LAYER — ASP.NET Core 8 (IoTGame.Core + GameServer)        ║
║   • Quarantine check, rate limiting, unknown-slot detection      ║
║   • DeviceRegistry: (deviceId, slotId) → player + action,        ║
║     or → environment signal (spawn_rate, ultrasonic_zone)        ║
║   • Ultrasonic distance → zones 1–4 with hysteresis              ║
║   • Intrusion alerts + device state → SignalR → dashboard        ║
╚══════════════════╤══════════════════════════════╤════════════════╝
                   │  gRPC over mutual TLS         │  REST + SignalR (JWT)
                   │  SendInputBatch ·             ▼
                   │  SendEnvironmentEvent ·    ┌──────────────────────┐
                   │  SendAdminCommand          │  ADMIN DASHBOARD      │
                   ▼                            │  React — live alerts, │
╔══════════════════════════════════════════╗   │  quarantine, rounds   │
║  GAME SERVER — Unity dedicated server    ║   └──────────────────────┘
║   • Grpc.Core server, client cert        ║
║     required and verified                ║
║   • ConcurrentQueue bridges gRPC threads ║
║     → Unity main thread                  ║
║   • Server-authoritative game state      ║
╚══════════════════╤═══════════════════════╝
                   │  Netcode for GameObjects / Unity Transport, port 7777
                   ▼
╔══════════════════════════════════════════╗
║  CLIENTS — Unity spectator clients       ║
║   • Render the replicated game           ║
║   • No connection to devices or relay    ║
╚══════════════════════════════════════════╝
```

---

## Security

Phase 6 hardened every hop. Each component has its own certificate signed by a project CA
(`scripts/gen_ca.sh`, `scripts/gen_device_cert.sh`):

| Hop | Protection |
|---|---|
| ESP32 → broker | Mutual TLS; the certificate CN is the device identity, and the ACL scopes each device to its own topics |
| Broker → relay | Mutual TLS with the relay's own certificate (`CN=gameserver`, read-only ACL) |
| Relay → Unity | gRPC over mutual TLS; Unity requires a client certificate signed by the project CA |
| Dashboard → relay | JWT login (bcrypt-hashed users in SQLite); the security API and SignalR hub require the token |

On top of transport security the relay runs a rule-based **intrusion detector**: devices that
flood events or fail authentication are **quarantined automatically**, unknown slots and
devices raise alerts, and an operator can quarantine or release a device from the dashboard.
`rogue_device.py` (relay repo) is a demo attacker for *no certificate*, *flood* and *spoofed
slot* scenarios; it has not been re-tested since Phase 6 (see the relay's known limitations).

Secrets never live in the repositories — see [docs/setup.md](docs/setup.md).

---

## Why This Architecture?

This section is intentionally detailed — a core goal of this project is to demonstrate
understanding of *why* each architectural choice was made, not just *what* was built.

### Why a relay server instead of ESP32 → Unity directly?

**Security boundary.** The Unity dedicated server is a private, trusted service. It should
never be internet-facing, nor responsible for IoT security. Device identity is enforced at
the broker by client certificates; the relay adds rate limiting, unknown-slot detection and
quarantine, and only then forwards fully resolved events. Unity accepts gRPC only from a
client presenting a certificate signed by the project CA.

**Protocol translation.** MQTT is a pub/sub broker protocol designed for constrained devices.
Unity's multiplayer stack (NGO/UTP) is a direct-connection transport. The relay is the
impedance-matching layer between the IoT world and the game world.

**Separation of concerns.** Which button belongs to which player, and what a knob or sensor
means in the game, is infrastructure config, not game logic. In the relay it is a
`DeviceRegistry` change — no firmware reflash, no Unity rebuild.

**Independent scaling.** The relay is stateless per event and can scale horizontally; the
Unity server is stateful and scales by running more instances per game session.

### Why MQTT for ESP32 → server?

A persistent TCP connection means no connection setup per button press, which keeps physical
input feeling immediate. MQTT is built for constrained devices: `umqtt.simple` ships with
MicroPython, per-message overhead is a few bytes instead of full HTTP headers, and the broker
gives fan-out, ACLs and certificate authentication for free. HTTP REST would have been simpler
but adds a request per event.

### Why gRPC for relay → Unity?

The `.proto` file is a machine-checked contract shared by the relay (client) and Unity
(server); both generate C# stubs from it, so serialization mismatches surface at compile time.
HTTP/2 multiplexing lets batched input and environment events share one connection. The cost
is setup complexity — stub generation and IL2CPP stripping exemptions for the Unity server
build — which is justified because this is the interface where two separate codebases must
agree on a data format.

### Why a dedicated server instead of a listen server?

The game is driven by IoT events, not by any player's client, so there is no natural host
player. A headless dedicated server can run anywhere, stays up regardless of which clients are
connected, and NGO's server-authoritative model fits input that arrives out-of-band over gRPC.

---

## Technology Choices

| Technology | Used for | Why chosen | Considered alternatives |
|---|---|---|---|
| **ESP32** | Controller hardware | WiFi built-in, MicroPython support, plenty of GPIO/ADC, low cost | Raspberry Pi Pico W, Arduino (no native WiFi) |
| **MicroPython** | Firmware | Python syntax; `umqtt.simple` and TLS built-in; fast iteration | C/C++ with ESP-IDF (faster, much more complex) |
| **MQTT** | ESP32 → broker | Designed for constrained devices; persistent connection; low per-message overhead | HTTP REST (request per event), WebSocket (heavier on MicroPython) |
| **Mosquitto** (self-hosted) | MQTT broker | Client-certificate auth, cert CN as username and per-device ACL patterns — the basis of the device security model | HiveMQ Cloud (used in early phases; managed, simpler setup), EMQX |
| **ASP.NET Core 8** | Relay server | First-class gRPC, mature MQTTnet library, SignalR, C# shared with Unity | Node.js (weaker gRPC story), Go |
| **gRPC / Protobuf** | Relay → Unity | Typed shared contract, HTTP/2 multiplexing, code generation on both sides | WebSocket (no schema), raw TCP, REST |
| **Unity 6 LTS** | Game engine | NGO is first-party; Dedicated Server build target; URP | Godot, Unreal (too heavy for a POC) |
| **Netcode for GameObjects** | Multiplayer | First-party, server-authoritative model fits IoT-driven input | Mirror, Fish-Net |
| **React + Vite** | Admin dashboard | Fast to build; `@microsoft/signalr` client for live alerts | Server-rendered Razor pages |

---

## Protocol Specification

### MQTT topic

```
game/devices/{deviceId}/slot/{slotId}
```

| Segment | Example | Description |
|---|---|---|
| `deviceId` | `esp32-01` | Unique per board — equal to its certificate CN |
| `slotId` | `joystick-x`, `btn-1`, `dht11` | Physical input slot on that board |

The topic carries **physical identity only**. Which player a slot belongs to and what it does
are resolved by the relay from its registry. The relay subscribes to `game/devices/+/slot/+`.

### MQTT payload

```json
{
  "deviceId": "esp32-01",
  "slotId": "dht11",
  "inputType": "dht11",
  "event": "change",
  "value": 27.0,
  "auxiliary": { "humidity": 61.0 },
  "timestamp": 1777777777,
  "sequenceNumber": 4821
}
```

| Field | Type | Description |
|---|---|---|
| `inputType` | string | `button`, `knob`, `dht11`, `ultrasonic` |
| `event` | string | `pressed` / `released` for buttons, `change` for everything else |
| `value` | float | Buttons `1.0`/`0.0`; knobs normalized `0.0–1.0`; DHT11 °C; ultrasonic cm |
| `auxiliary` | object | Optional extra readings from multi-value sensors (DHT11 humidity) |
| `timestamp` | int64 | Board clock in seconds (`utime.time()`), NTP-synced on WiFi connect |
| `sequenceNumber` | int64 | Per-device counter; resets on reboot |

### Device registry (relay config)

The registry is the **only place** where hardware identity and game meaning meet. Each slot
has a role:

| `inputRole` | Becomes | Example |
|---|---|---|
| `player` | Input for one player: `move-x`, `move-y`, `move-y-up`, `move-y-down`, `fire` | thumbstick, knob, buttons |
| `stage` | Round control: `admin-start`, `admin-end` | operator buttons on ESP32 #1 |
| `environment` | Environment signal via `envKey` / `auxiliaryKeys` | humidity → `spawn_rate`, distance → `ultrasonic_zone` |

```json
"esp32-02": {
  "defaultPlayerId": "player-2",
  "inputs": {
    "knob":  { "type": "knob",   "inputName": "move-x",      "inputRole": "player" },
    "btn-1": { "type": "button", "inputName": "move-y-up",   "inputRole": "player" },
    "btn-2": { "type": "button", "inputName": "move-y-down", "inputRole": "player" },
    "btn-3": { "type": "button", "inputName": "fire",        "inputRole": "player" }
  }
}
```

### gRPC contract

`game_controller_v2.proto` — the relay is the client, Unity the server:

```proto
service GameControllerService {
  rpc SendInputEvent       (InputEventRequest)       returns (InputEventResponse);
  rpc SendInputBatch       (InputBatchRequest)       returns (InputEventResponse);
  rpc SendEnvironmentEvent (EnvironmentEventRequest) returns (InputEventResponse);
  rpc SendAdminCommand     (AdminCommandRequest)     returns (AdminCommandResponse);
}

message InputEventRequest {
  string player_id       = 1;  // resolved by the relay
  string device_id       = 2;  // diagnostics only
  string input_name      = 3;  // semantic action, e.g. "move-x", "fire"
  string input_type      = 4;
  string event_type      = 5;
  float  value           = 6;
  int64  timestamp       = 7;
  int64  sequence_number = 8;
}

message EnvironmentEventRequest {
  string env_key   = 1;  // "spawn_rate", "ultrasonic_zone", "gravity"
  float  raw_value = 2;
  int64  timestamp = 3;
}
```

`SendAdminCommand` carries round control (`start_round`, `end_round`, `reset`,
`session_start`) with optional numeric parameters.

---

## Key Design Decisions

### Physical slot identity is separated from semantic meaning

Firmware only knows `slotId`, a hardware label. The game only knows `inputName` / `envKey`.
They are linked in the relay registry alone, so reassigning a control or giving a sensor a
new purpose is a config change. Introduced in Phase 2.1 after two players sharing one board
both needed a `jump` input — with physical slot ids they are simply `btn1` and `btn2`.

### Library / game split in every layer (Phase 4)

| Layer | Reusable library | Game-specific |
|---|---|---|
| Firmware | `iotgame/` — device loop, MQTT/TLS, input classes | `game/` — configs and entry point |
| Relay | `IoTGame.Core` — pipeline, rate limiting, zones, gRPC relay, extension interfaces | `GameServer` — registry mapping, intrusion detector, dashboard API |
| Unity | `Assets/IoTGame/` — gRPC host, queue, input / environment / admin dispatchers | `Assets/Scripts/` — ships, enemies, rounds, environment effects |

A new game reuses the three libraries and only writes its own game layer.

### The environment is an input

Sensors aren't attached to players; they map to environment signals. Humidity changes the
enemy spawn interval (3 s down to a 1 s minimum), and distance zones 1–4 set the game speed
(0.5×–1.5×).
The relay applies hysteresis to the distance reading so jitter at a zone boundary doesn't
flicker the game speed.

### Configurable input buffering

`IoTGame:BufferWindowMs = 0` forwards every event immediately; a value like `16` batches
events per frame for high-frequency analog input. Different games want different trade-offs,
so it is configuration, not code.

### Thread safety in Unity is a hard constraint

gRPC handlers run on background threads; Unity's API is main-thread only. The bridge is
structural: handlers only enqueue into a `ConcurrentQueue`, and `IoTInputManager` drains it
in `Update()` and dispatches on the main thread.

---

## Project Phases

| Phase | Summary |
|---|---|
| **1** | One button per device, one player per device — end-to-end ESP32 → MQTT → relay → gRPC → Unity |
| **2** | Multi-input devices (buttons, knobs, sensors), multi-device players, shared devices, input buffering and rate limiting |
| **2.1** | Protocol fix — physical `slotId` separated from semantic `inputName`; topic `game/devices/{deviceId}/slot/{slotId}` |
| **3** | Space shooter with a stage controller; DHT11 and HC-SR04 environment sensors; `SendEnvironmentEvent` |
| **3.1–3.3** | Humidity + stage panel, gravity and spawn-rate effects, gRPC keep-alive and reconnection |
| **4** | Library foundation — reusable library / game split in firmware, relay and Unity; session management; admin API |
| **5** | Game redesign — two-player co-op shooter, AI enemies that shoot back, admin-only round start, world-space UI |
| **6** | Security — mutual TLS on every hop, per-device certificates and broker ACLs, intrusion detection with quarantine, admin security dashboard |

Each repository keeps its phase plans in `docs/plans/` and session logs in `docs/logs/`.

---

## Getting Started

**[docs/setup.md](docs/setup.md)** walks through the full setup: creating the CA and
certificates, configuring Mosquitto, flashing the ESP32s, the relay's secrets, building the
Unity server, and running the dashboard.

Prerequisites: ESP32 boards with MicroPython, Mosquitto, the .NET 8 SDK, Unity 6000.3.11f1,
Python 3 with `mpremote`, OpenSSL (Git Bash on Windows), and Node.js for the dashboard.

```bash
# Everything, at the pinned versions
git clone --recurse-submodules https://github.com/SupakornSJB/ATITFinalProject

# Already cloned without submodules?
git submodule update --init

# Move every component to its latest main
git submodule update --remote
```

GitHub's *Download ZIP* does not include submodule contents — clone with
`--recurse-submodules` instead.

---

## Project Status

| Component | Status |
|---|---|
| ESP32 firmware | ✅ Phase 6 complete |
| Relay server | ✅ Phase 6 complete |
| Unity dedicated server + client | ✅ Phase 6 complete |
| Admin dashboard | 🟡 Implemented; not fully tested end to end |

The full pipeline — ESP32 controllers through to the Unity clients — works end to end.

Known limitations are listed in each component's README. The main ones:

- The admin round-control API accepts only an API key, so the dashboard's round buttons need it
  unset.
- The relay skips the Unity server-certificate check for `localhost` endpoints; for remote
  hosts the project CA must be installed as a trusted root.
- Unity accepts any client certificate signed by the project CA — a certificate copied from a
  device could call it directly and bypass the relay's intrusion detection.
- Unity falls back to plain gRPC (with a warning) if its certificate files are missing.

---

## License

MIT — see [LICENSE](./LICENSE).
