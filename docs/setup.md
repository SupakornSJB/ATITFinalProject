# Setup Guide — Secrets, Certificates & Configuration

This guide covers every value you must supply yourself to run the full system:
certificates, broker config, device configs, and server secrets. **None of these are
committed to git** — each repo only contains example/placeholder values.

> Do the steps in order. Certificates are created once (step 1) and then copied into
> each component.

---

## Overview — what lives where

| Value | Used by | Where it goes | Committed? |
|---|---|---|---|
| Root CA (`ca.crt` / `ca.key`) | Everything | `atit_micropython/certs/` | No — `ca.key` especially **never** |
| Broker cert (CN = broker host) | Mosquitto | Mosquitto `certs/` folder | No |
| Device certs (`esp32-01`, …) | ESP32 firmware | ESP32 filesystem (flashed) | No |
| Relay cert (`gameserver`) | Relay server → MQTT and → Unity | `ATITInfraNode/GameServer/Certs/` | No |
| Unity server cert (`unity-server`) | Unity gRPC server | `ATITProject/Assets/Certs/` | No |
| WiFi SSID / password, broker host | ESP32 firmware | `atit_micropython/game/config.py` (copied from an example) | No |
| `Dashboard:JwtSecret` | Relay server dashboard login | `dotnet user-secrets` / env var | No |
| `IoTGame:AdminApiKey` | Relay server admin endpoints | `dotnet user-secrets` / env var | No |
| Dashboard admin password | Dashboard login | SQLite `dashboard.db` via `seed_admin.py` | No |

All certificates are signed by the same project CA, and every connection is mutual TLS:

```
ESP32 (esp32-xx cert) ──MQTT/TLS :8883──▶ Mosquitto (broker cert) ◀──MQTT/TLS── Relay (gameserver cert)
                                                                                    │
                                                                       gRPC/mTLS :9090
                                                                                    ▼
                                                                  Unity server (unity-server cert)
```

### Repository layout used below

Clone this repository with its submodules — the components end up side by side, and every
command below runs from the `ATITFinalProject/` root unless it says `cd`:

```bash
git clone --recurse-submodules https://github.com/SupakornSJB/ATITFinalProject
```

```
ATITFinalProject/
├── atit_micropython/   # firmware   — github.com/SupakornSJB/atit_micropython
├── ATITInfraNode/      # relay      — github.com/SupakornSJB/ATITInfraNode
├── ATITProject/        # Unity      — github.com/SupakornSJB/ATITProject
└── atit-dashboard/     # dashboard  — github.com/SupakornSJB/atit-dashboard
```

Cloning the four repos separately into one folder works the same way.

Scripts are bash; on Windows run them from **Git Bash** (they already set
`MSYS_NO_PATHCONV=1` so OpenSSL paths work there). Requires `openssl`, Python 3,
`mpremote`, the .NET 8 SDK, and Unity 6 LTS.

---

## 1. Create the certificates (once)

Run everything from the firmware repo root:

```bash
cd atit_micropython

# Root CA — certs/ca.key (KEEP SECRET) and certs/ca.crt (public)
bash scripts/gen_ca.sh

# One cert per ESP32 — the CN becomes the device's MQTT username and must equal
# DEVICE_ID in its config.py and the device id in the relay's DeviceRegistry
bash scripts/gen_device_cert.sh esp32-01
bash scripts/gen_device_cert.sh esp32-02

# Relay server — CN "gameserver" is referenced by the Mosquitto ACL below
bash scripts/gen_device_cert.sh gameserver

# Unity dedicated server
bash scripts/gen_device_cert.sh unity-server

# Broker — the CN must be the exact host/IP the ESP32s put in MQTT_HOST,
# because the firmware verifies the broker's hostname
bash scripts/gen_device_cert.sh 192.168.1.50
```

Each call writes `certs/<name>/device.crt` and `certs/<name>/device.key`.
Everything under `certs/` is git-ignored, including `ca.crt` — each deployment
creates its own CA, so a committed one would be useless to anyone else.
Because `ca.crt` is not in any repo, the copy steps below (2, 3.2, 4.1, 5.1) are the
only way each component gets it — every component must trust this same CA.

---

## 2. Mosquitto broker

Copy `certs/ca.crt`, `certs/<broker-host>/device.crt` and `certs/<broker-host>/device.key`
into the broker's certs folder, then configure `mosquitto.conf`:

```conf
listener 8883 0.0.0.0
protocol mqtt

cafile   C:\Program Files (x86)\Mosquitto\certs\ca.crt
certfile C:\Program Files (x86)\Mosquitto\certs\device.crt
keyfile  C:\Program Files (x86)\Mosquitto\certs\device.key

# No client cert = no connection; the cert CN becomes the MQTT username
require_certificate true
use_identity_as_username true

acl_file C:\Program Files (x86)\Mosquitto\acl.conf
```

`acl.conf` — each device may only publish under its own id; the relay may read all:

```conf
pattern write game/devices/%u/#

user gameserver
topic read game/devices/#
```

> **Port:** 8883 is the conventional port for MQTT over TLS, but any port works as long as
> every client uses the listener's port. The firmware switches TLS on whenever cert paths are
> set in `config.py` (`iotgame/mqtt.py`), and the relay whenever `Mqtt:UseTls` is `true`.

Restart Mosquitto after changing the config.

---

## 3. ESP32 firmware (`atit_micropython`)

**3.1 Create your config from the example — never edit the example itself:**

```bash
cp game/config.example.py game/config.py     # one file per device you flash
# or, for the 4-input player-2 layout:
cp game/config_device2.example.py game/config.py
```

Fill in:

| Key | Value |
|---|---|
| `WIFI_SSID` / `WIFI_PASSWORD` | Your 2.4 GHz network (ESP32 cannot join 5 GHz) |
| `DEVICE_ID` | Same name you used in `gen_device_cert.sh`, e.g. `esp32-01` |
| `MQTT_HOST` | Broker host/IP — must match the broker cert CN from step 1 |
| `MQTT_PORT` | The broker's TLS listener port (`8883` above) |
| `MQTT_USER` / `MQTT_PASSWORD` | `None` (identity comes from the cert) |
| `CA_CERT_PATH` / `CERT_PATH` / `KEY_PATH` | Leave as `ca.crt` / `device.crt` / `device.key` |
| `INPUTS` | Your wiring — see the examples in `game/config.example.py` |

`config.py` and `config_*.py` are git-ignored; only `*.example.py` files are committed.

**3.2 Flash the certificates** (ESP32 connected over USB):

```bash
python scripts/provision_device.py esp32-01 --port COM3
```

**3.3 Upload the firmware:**

```bash
mpremote connect COM3 cp -r iotgame :
mpremote connect COM3 cp game/config.py :config.py
mpremote connect COM3 cp game/main.py :main.py
mpremote connect COM3 repl        # watch for "[MQTT] Connected"
```

---

## 4. Relay server (`ATITInfraNode/GameServer`)

**4.1 Copy the certificates** (from the `ATITFinalProject/` root):

```bash
cp atit_micropython/certs/ca.crt                  ATITInfraNode/GameServer/Certs/ca.crt
cp atit_micropython/certs/gameserver/device.crt   ATITInfraNode/GameServer/Certs/device.crt
cp atit_micropython/certs/gameserver/device.key   ATITInfraNode/GameServer/Certs/device.key
```

Paths in `appsettings.json` (`Certs/ca.crt`, `Certs/device.crt`, `Certs/device.key`) are
relative to the `GameServer/` project folder, so this works from any directory.

**4.2 Review `GameServer/appsettings.json`** (committed, non-secret):

| Key | Value |
|---|---|
| `Mqtt:Host` / `Mqtt:Port` | Broker host and its TLS listener port |
| `Mqtt:UseTls` | `true` |
| `Unity:GrpcEndpoint` | `https://localhost:9090` — run Unity on the same machine as the relay (recommended; the relay skips the server-certificate check for `localhost`). Any other host is checked against the OS trust store and host name: install `ca.crt` as a trusted root on the relay machine, issue Unity's cert for that DNS name (`gen_device_cert.sh <host>`, which writes `certs/<host>/` — copy from there instead of `certs/unity-server/` in 5.1), and note the script sets only the CN, so an IP-address endpoint needs a certificate with a subjectAltName |
| `DeviceRegistry` | Maps each `DEVICE_ID` and its `slotId`s to players and actions |

**4.3 Set the secrets** — these must never go in `appsettings.json`:

```bash
cd ATITInfraNode/GameServer

# Required: signs dashboard login tokens. At least 32 bytes — the server refuses to start otherwise.
dotnet user-secrets set "Dashboard:JwtSecret" "$(openssl rand -base64 48)"

# Optional: protects /api/admin/* and /api/sessions/* (callers send it as the X-IoTGame-Key header).
# If left empty, those endpoints accept every request. Known limitation: the dashboard's
# Start/End round buttons send only the login JWT, so they stop working once this is set.
dotnet user-secrets set "IoTGame:AdminApiKey" "$(openssl rand -hex 32)"

# Only if your broker uses username/password instead of client certs:
dotnet user-secrets set "Mqtt:Username" "your-broker-user"
dotnet user-secrets set "Mqtt:Password" "your-broker-password"
```

`user-secrets` are stored outside the repo (`%APPDATA%\Microsoft\UserSecrets\`) and are
only loaded when `ASPNETCORE_ENVIRONMENT=Development` (the default launch profile).
Outside Development, use environment variables — `:` becomes `__`:

```bash
export Dashboard__JwtSecret="..."
export Mqtt__Password="..."
```

**4.4 Run it and create the dashboard admin:**

```bash
dotnet run                                  # creates dashboard.db on first start
pip install bcrypt
python Scripts/seed_admin.py "<choose-an-admin-password>"   # run from GameServer/
```

Log in to the dashboard as `admin` with that password. `dashboard.db` is git-ignored.

---

## 5. Unity dedicated server (`ATITProject`)

**5.1 Copy the certificates** (from the `ATITFinalProject/` root):

```bash
cp atit_micropython/certs/ca.crt                    ATITProject/Assets/Certs/ca.crt
cp atit_micropython/certs/unity-server/device.crt   ATITProject/Assets/Certs/device.crt
cp atit_micropython/certs/unity-server/device.key   ATITProject/Assets/Certs/device.key
```

`Assets/Certs/.gitignore` keeps all of these out of git.

**5.2 Build and run:** File → Build Settings → Dedicated Server → Build (or the CLI build in the
Unity README, which writes `Builds/Server/ATITProject.exe` and copies the certs next to it), then:

```powershell
cd ATITProject
Builds/Server/ATITProject.exe -batchmode -nographics `
  -grpcPort 9090 -gamePort 7777 `
  -grpcCert Assets/Certs/device.crt `
  -grpcKey  Assets/Certs/device.key `
  -caCert   Assets/Certs/ca.crt `
  -logFile  server.log
```

The cert arguments default to those `Assets/Certs/` paths when omitted. Relative paths
resolve against the directory you start the server from — here the project root, so it uses
the files from 5.1. If they aren't found the server logs `starting INSECURE` and runs without
TLS, so check for `Starting with mTLS` in the log. Game clients connect with
`-serverAddress <host> -gamePort 7777`.

---

## 6. Admin dashboard (`atit-dashboard`)

With the relay running (step 4):

```bash
cd atit-dashboard
npm install
npm run dev          # http://localhost:5173/dashboard/
```

Vite proxies `/api` and `/hubs` to the relay at `https://localhost:5000` (see `vite.config.ts`),
so no extra configuration is needed. Log in as `admin` with the password from step 4.4.

To serve the dashboard from the relay itself (one server, no Node at demo time), build it into
the relay's `wwwroot` — the relay serves it at `https://<relay-host>:5000/dashboard/`:

```bash
npm run deploy     # = npm run build -- --outDir ../ATITInfraNode/GameServer/wwwroot/dashboard --emptyOutDir
```

---

## 7. Never commit

Before every commit, `git status` should show none of these:

- `*.key`, `*.csr`, `*.srl`, `*.pfx`, `*.p12`
- `config.py`, `game/config.py`, `config_*.py` (only `*.example.py` belongs in git)
- `secrets.json`, `.env*`, `dashboard.db*`
- Real values in `appsettings.json` — `JwtSecret`, `Password`, `AdminApiKey` stay empty

If a secret is ever committed, **rotate it** (generate a new one) — deleting the file
does not remove it from git history.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Dashboard:JwtSecret is missing or shorter than 32 bytes` on start | Step 4.3 not done, or not running in Development (user-secrets not loaded) — set `Dashboard__JwtSecret` instead |
| ESP32 never connects; broker log shows TLS errors | Cert paths missing in `config.py`, `MQTT_PORT` isn't the TLS listener's port, or `MQTT_HOST` doesn't match the broker cert CN |
| ESP32 can't see the network | 5 GHz SSID — ESP32 only supports 2.4 GHz |
| ESP32 connects but relay receives nothing | Mosquitto ACL: device cert CN ≠ topic `deviceId`, or relay cert CN ≠ `gameserver` |
| Relay can't reach Unity / TLS handshake failure on 9090 | Relay cert not signed by the same `ca.crt` Unity loads; for a non-localhost endpoint, `ca.crt` not trusted by the OS or the Unity cert's name doesn't match the host; wrong `-grpcCert`/`-caCert` paths |
| `FileNotFoundException` for a cert | File not copied in step 4.1 / 5.1 |
