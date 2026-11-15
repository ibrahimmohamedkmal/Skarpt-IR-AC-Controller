# ESP32 IR AC Controller + Config Writer

An ESP32-based **infrared air-conditioner controller** that is driven over **MQTT**, can act as a **Wi-Fi repeater**, can **learn** IR codes from any remote, and is configured by a **desktop GUI** (or remotely over MQTT) without ever re-compiling the firmware.

This repository is **documentation only**. It describes how the system is designed, how it behaves, and how it is configured. It intentionally contains **no source code and no compiled binaries**.

---

## Contents

1. [What it does](#1-what-it-does)
2. [System overview](#2-system-overview)
3. [Hardware](#3-hardware)
4. [Operating modes](#4-operating-modes)
5. [Configuration storage (NVS)](#5-configuration-storage-nvs)
6. [MQTT interface](#6-mqtt-interface)
7. [Remote configuration over MQTT](#7-remote-configuration-over-mqtt)
8. [Dual-broker mode](#8-dual-broker-mode)
9. [Wi-Fi repeater mode](#9-wi-fi-repeater-mode)
10. [IR learning mode](#10-ir-learning-mode)
11. [Reliability and self-healing](#11-reliability-and-self-healing)
12. [Boot behavior and state restore](#12-boot-behavior-and-state-restore)
13. [The Config Writer desktop tool](#13-the-config-writer-desktop-tool)
14. [Firmware build system](#14-firmware-build-system)
15. [Security notes](#15-security-notes)
16. [Known limitations](#16-known-limitations)

---

## 1. What it does

| Capability | Summary |
|---|---|
| IR control of an AC | Sends power, temperature, mode, fan, swing and turbo commands as IR frames |
| MQTT control | Every function is a plain MQTT topic, so it plugs into any home-automation or custom backend |
| Two ways to talk IR | Built-in library protocols (about 70 AC brands/families) **or** raw codes learned from your own remote |
| Wi-Fi repeater | Optionally broadcasts its own access point and routes clients through its uplink using NAT |
| Dual MQTT brokers | Can publish to and listen on two brokers at once |
| No-reflash configuration | Wi-Fi, MQTT, repeater and protocol settings live in flash storage and are written by a desktop tool or over MQTT |
| State persistence | Remembers the AC's last state across reboots and can re-send it after a power loss |
| Self-healing | Watchdog, heap monitor, bounded network timeouts and automatic reboot on prolonged disconnection |

Each physical unit has a fixed **Device ID** (an 8-digit string such as `00000009`) that is compiled into the firmware. It is used in every MQTT topic, in the MQTT client ID, in the default repeater network name, and to derive the repeater's subnet. It is deliberately **not** editable from the configuration tool.

---

## 2. System overview

```
                       ┌──────────────────────────────┐
  Home automation /    │         MQTT broker(s)       │
  app / script  ─────► │  (one, or two in dual mode)  │
                       └──────────────┬───────────────┘
                                      │  ac/<id>/...
                       ┌──────────────▼───────────────┐
   Config Writer       │            ESP32             │      IR LED  ───►  Air conditioner
   (desktop, USB) ───► │  Wi-Fi STA (+ optional AP)   │
   writes settings     │  MQTT client(s)              │      IR receiver ◄─ Original remote
   to flash            │  IR sender / learner         │      (learning only)
                       └──────────────┬───────────────┘
                                      │ optional AP + NAT
                                      ▼
                              Downstream Wi-Fi clients
                              (other devices / repeaters)
```

There are two deliverables:

1. **Firmware**: runs on the ESP32. Connects to Wi-Fi, subscribes to MQTT topics and turns commands into IR transmissions.
2. **Config Writer**: a Windows desktop program (Python/Tkinter, packaged as a single portable `.exe`) that writes settings straight into the ESP32's flash over USB and drives IR learning.

---

## 3. Hardware

| Item | Detail |
|---|---|
| Microcontroller | Classic ESP32 (the `esp32dev` board profile) |
| IR transmit pin | **GPIO 13** (drives the IR LED stage) |
| IR receive pin | **GPIO 15** (IR receiver module, used only while learning) |
| Carrier frequency | 38 kHz for learned/raw playback |
| Serial | 115200 baud over USB (logging, learning commands, configuration tool) |
| Flash size | Configured for **2 MB** in the build. Verify your board before flashing; many ESP32 dev boards have 4 MB. The build-system flash size and the board setting must agree or the partition layout will be wrong. |

IR transmission is **software generated** (timed pin toggling), not hardware-assisted. This matters for the reliability design described in [section 11](#11-reliability-and-self-healing).

---

## 4. Operating modes

The setting `ac_proto` selects how the unit speaks to the air conditioner.

### 4.1 Protocol mode (library-driven)

The unit holds a complete logical AC state (power, temperature, mode, fan, swing, turbo) and asks an IR library to encode the entire state into the correct frame for the chosen brand. This is how most real AC remotes work: every button press sends the full state.

- Default protocol: `COOLIX`.
- Supported protocol names include major families such as Daikin (many variants), Mitsubishi (several), Panasonic, Samsung, LG, Gree, Midea, Haier, Hitachi, Fujitsu, Toshiba, Carrier, Sharp, Whirlpool, Electra, Kelvinator, TCL, Vestel, Voltas, York and others.
- If the stored name is unknown or unsupported, the firmware logs a warning and falls back to `COOLIX`.
- Valid temperature range: **20 to 30**.
- Turbo is available in this mode only.

### 4.2 LEARNED mode

For ACs not covered by any built-in protocol, record the real remote's buttons and play them back as raw timing data. `ac_proto` is set to `LEARNED` automatically when learned codes are saved.

- There are **14 button slots**: power on, power off, temp up, temp down, mode cool/heat/dry/fan/auto, fan auto/min/med/max, and swing toggle.
- Because remotes like these only have "up" and "down" temperature buttons, a requested temperature is reached by **stepping**: the unit presses up or down once every 400 ms until the tracked setpoint matches. This runs in the background so MQTT handling is never blocked.
- Swing is a single toggle button. The unit remembers whether swing is on and only presses the button when the requested state differs, so `on` and `off` behave as absolute commands.
- Turbo is not available in this mode.
- Missing slots are skipped safely; the tracked setpoint does not advance if a frame was never sent.

See [section 10](#10-ir-learning-mode) for the learning workflow.

---

## 5. Configuration storage (NVS)

Everything configurable is stored in the ESP32's non-volatile storage (NVS) partition, in separate namespaces. The default partition layout puts NVS at offset `0x9000` with a size of `0x5000`.

### 5.1 Namespace `cfg`: connection and mode settings

| Key | Type | Meaning |
|---|---|---|
| `ssid` | string (max 32) | Wi-Fi network the unit joins (required) |
| `pass` | string (max 64) | Wi-Fi password |
| `mqtt_host1` | string (max 64) | Broker 1 hostname or IP (required) |
| `mqtt_port1` | 16-bit int | Broker 1 port (default 1883) |
| `mqtt_user1` / `mqtt_pass1` | string | Broker 1 credentials (optional; empty means anonymous) |
| `dual_mode` | 8-bit int | `1` enables the second broker |
| `mqtt_host2`, `mqtt_port2`, `mqtt_user2`, `mqtt_pass2` | as above | Broker 2 settings |
| `repeater_en` | 8-bit int | `1` enables the repeater access point (default off) |
| `ap_ssid` | string (max 32) | Repeater network name (default `AC-<device id>`) |
| `ap_pass` | string (max 64) | Repeater password (minimum 8 characters for WPA2) |
| `ac_proto` | string | Protocol name or `LEARNED` |

### 5.2 Other namespaces (managed by the firmware)

| Namespace | Holds |
|---|---|
| `acstate` | Last AC state in protocol mode: power, temperature, mode, fan, vertical swing, turbo |
| `irlearn` | Learned raw IR codes (one entry per button slot), plus tracked setpoint, power and swing state |
| `bootcfg` | The boot-resync on/off switch |

### 5.3 Sanity checks at boot

The firmware prints warnings (and degrades safely) when it finds:

- No Wi-Fi SSID configured.
- Dual mode enabled but no second broker host.
- Repeater enabled with a password shorter than 8 characters (the access point will refuse to start; it never falls back to an open network).
- An MQTT password with no username (the broker connection would be anonymous).
- A restored temperature outside 20 to 30 (clamped to 24).

---

## 6. MQTT interface

All topics are prefixed `ac/<id>/`, where `<id>` is the Device ID. Subscriptions use QoS 1. Payloads are case-insensitive for commands.

### 6.1 Command topics (you publish)

| Topic | Accepted payloads |
|---|---|
| `ac/<id>/set/power` | `on`, `off` |
| `ac/<id>/set/temp` | integer `20` to `30` |
| `ac/<id>/set/mode` | `cool`, `heat`, `dry`, `fan`, `auto` |
| `ac/<id>/set/fan` | `auto`, `min`, `med`, `max` |
| `ac/<id>/set/swing` | `on`, `off`, `toggle` |
| `ac/<id>/set/turbo` | `on`, `off` (protocol mode only) |
| `ac/<id>/set/bootresync` | `on`/`off` (also `1`/`0`, `true`/`false`) |

Invalid payloads are ignored. Commands are also ignored while an IR-learning capture is armed, so the unit cannot record its own transmission.

### 6.2 Feedback topics (the unit publishes)

| Topic | Retained | Content |
|---|---|---|
| `ac/<id>/state` | Yes | JSON snapshot of the real AC state (see below) |
| `ac/<id>/ack` | No | JSON acknowledgement of the last accepted command |
| `ac/<id>/status` | Yes | `online` (refreshed every 60 s); the broker publishes `offline` automatically as the last-will message if the unit drops |
| `ac/<id>/bootresync` | Yes | Current boot-resync setting, `on` or `off` |
| `ac/<id>/cfgresult` | No | Result of remote configuration commands |
| `ac/<id>/cfgstate/<key>` | No | Current configuration values, on request |

**State snapshot fields**

- Protocol mode: `device`, `power`, `degrees`, `mode`, `fan`, `swing`, `turbo`.
- LEARNED mode: `device`, `power`, `degrees`, `swing`.

The snapshot always has the same shape regardless of which command caused the change, and it is re-published on every broker (re)connection.

**Acknowledgement fields:** `device`, `topic`, `value`, `power`, `uptime_ms`.

---

## 7. Remote configuration over MQTT

Settings can be changed with no USB cable. Changes are **staged in RAM first, then committed together**, which prevents half-applied configurations.

| Topic (publish) | Payload | Effect |
|---|---|---|
| `ac/<id>/cfg/set/<key>` | the value | Validates and stages one setting |
| `ac/<id>/cfg/commit` | anything | Writes every staged setting to flash, then **reboots** |
| `ac/<id>/cfg/cancel` | anything | Discards staged settings |
| `ac/<id>/cfg/get` | anything | Publishes current settings to `cfgstate/<key>` (passwords are reported only as set or unset) |

Valid keys are the same as the `cfg` namespace above. Details worth knowing:

- Every command receives an `ok:...` or `err:...` reply on `cfgresult`.
- To set a text value to empty, send the literal text `<empty>` (an empty MQTT payload is how retained messages are cleared, so it cannot be used).
- Config commands are **cleared from the broker's retained storage** as soon as they are received, so a retained `commit` can never cause a reboot loop.
- Values longer than the field limit are rejected.
- Case is preserved for config values (SSIDs and passwords are case-sensitive).

---

## 8. Dual-broker mode

With `dual_mode` on, the unit keeps two independent MQTT connections.

- Commands from **either** broker are accepted.
- Acknowledgements, state and config replies are published to the broker the command came from **and** the other one, so both stay in sync.
- Each connection has its own retry timer with exponential back-off.
- If dual mode is enabled but broker 2 has no host, the unit falls back to single-broker operation.
- The boot re-sync fires as soon as **either** broker is connected, so an outage of one does not block it.

---

## 9. Wi-Fi repeater mode

When `repeater_en` is `1`, the unit runs **access point plus station** simultaneously:

- The **station** side joins your existing Wi-Fi.
- The **access point** side broadcasts its own network (up to 8 clients, WPA2, channel follows the uplink).
- **NAT** routes the AP clients' traffic out through the station link, and the upstream **DNS server is passed on** to AP clients via DHCP so name resolution works.

Design points:

- **Off by default.** When disabled the AP interface is never started, which avoids a stray default-named network appearing.
- **Unique subnet per device.** The AP subnet is `192.168.X.0/24` where `X` is derived directly from the Device ID (ID mod 250, plus 2). Distinct IDs always give distinct subnets, so chains of repeaters never collide with each other or the upstream router.
- **NAT is re-established after every uplink reconnection**, including re-propagating DNS in case the router handed out a different server.
- Reconnection attempts never power the radio off, so downstream clients are not kicked.
- The firmware needs specific network-stack options to make this work (IP forwarding, NAPT, packet copy at the L2/L3 boundary, up to 8 DHCP clients). See [section 14](#14-firmware-build-system).

---

## 10. IR learning mode

Learning is driven from the Config Writer's "learning" window over the USB serial link; the firmware side is a small line-based protocol.

### 10.1 Workflow

1. **Probe (optional):** point the original remote at the receiver and press any button. The unit reports whether the signal matches a supported built-in protocol (`MATCH:<name>`) or not (`UNKNOWN`). If it matches, you can simply use protocol mode.
2. **Capture each button:** choose a slot, the unit arms its receiver for **15 seconds**, and you press that button on the original remote. The capture is **staged in RAM**.
3. **Commit:** all staged slots are written to flash, `ac_proto` becomes `LEARNED`, and the unit reboots.

A commit is a **full replace**: slots not re-captured in the same session are cleared, so the saved set always reflects exactly what you staged. `ABORT` throws the staged captures away; `CANCEL` stops an armed capture.

### 10.2 Serial protocol

Commands are lines sent to the unit. The unit answers with lines beginning `RSP:LEARN:`.

| Command | Purpose |
|---|---|
| `CMD:LEARN:PROBE` | Identify a remote's protocol |
| `CMD:LEARN:SLOT:<key>` | Arm capture for one button slot |
| `CMD:LEARN:CANCEL` | Stop the armed capture |
| `CMD:LEARN:COMMIT` | Save everything staged and reboot |
| `CMD:LEARN:ABORT` | Discard everything staged |
| `CMD:LEARN:STATUS` | Report staged and saved counts |
| `CMD:LEARN:LIST` | List slot keys and labels |

Responses include `ARMED`, `MATCH`, `UNKNOWN`, `CAPTURED`, `FAILED`, `TIMEOUT`, `STATUS`, `STAGED`, `COMMITTING`, `SAVED`, `CANCELLED`, `ABORTED` and `ERROR`.

Slot keys: `pw1`, `pw0`, `tup`, `tdn`, `mcl`, `mht`, `mdr`, `mfn`, `mau`, `fau`, `fmn`, `fmd`, `fmx`, `swt`.

### 10.3 IR profiles

The Config Writer can save a unit's captured codes as a named **profile** and write that profile onto other units that have no IR receiver. This lets one unit with a receiver "teach" a whole fleet. See [section 13](#13-the-config-writer-desktop-tool).

---

## 11. Reliability and self-healing

| Mechanism | What it does |
|---|---|
| **Task watchdog (60 s)** | Watches only the main application loop (idle tasks are excluded to avoid false resets under Wi-Fi/NAT load); a hang triggers a clean reboot |
| **Heap monitor** | Logs free and fragmented heap every 30 s; reboots if free heap drops below 20 KB |
| **Disconnect timeout** | If Wi-Fi or an MQTT broker stays down for more than 5 minutes, the unit reboots |
| **Back-off retries** | Wi-Fi and MQTT reconnection intervals double from 1 s up to 30 s |
| **Bounded TCP connect** | Broker connections use an explicit 3 s connect timeout. Without this, an unreachable host could block longer than the watchdog and force a panic reboot |
| **Non-blocking temperature ramp** | Learned-mode temperature stepping happens a step per loop pass instead of blocking inside the MQTT handler |
| **IR blast window** | During an IR transmit the loop task's priority is raised temporarily so network packet processing (which is busiest in repeater mode) cannot interrupt the timed pulses and corrupt the waveform; it is restored immediately afterwards |
| **Learning guard** | Incoming commands are ignored while a capture is armed |
| **Wi-Fi tuning** | Power-save disabled, maximum transmit power for range |

---

## 12. Boot behavior and state restore

1. Load configuration, learned IR codes and the boot-resync setting from flash.
2. Build topic names, start the watchdog, initialise the IR pins.
3. Restore the last AC state (protocol mode: from `acstate`; learned mode: power, setpoint and swing from `irlearn`).
4. Start Wi-Fi (and the repeater AP if enabled), then connect to the broker(s).
5. **Boot re-sync:** once a broker is connected, the restored state is **re-sent over IR**. This covers the case where the air conditioner lost power and reset to defaults while the controller silently "remembered" the old state. In learned mode this replays the power button for the remembered power state.

The boot re-sync can be switched **only via MQTT** (`set/bootresync`); it is deliberately not exposed in the desktop tool. It is on by default and the choice is stored in flash so it applies from the very next boot.

---

## 13. The Config Writer desktop tool

A Tkinter GUI that writes the `cfg` namespace into the ESP32 over USB, with no need to rebuild or reflash the firmware.

### 13.1 Features

- **Serial port detection** with a hint for likely ESP32 adapters, plus automatic refresh.
- **Form-based configuration** of Wi-Fi, one or two MQTT brokers (with optional authentication), repeater enable/SSID/password, and the AC protocol (dropdown of all supported names, including `LEARNED`).
- **Input validation** for hostnames and ports as you type, and a check for missing required fields before writing.
- **Password visibility toggles** on every secret field.
- **Write config:** builds an NVS image, flashes it at the NVS offset, and resets the board.
- **Read config from device:** reads the NVS partition back, decodes the binary format (strings, integers, blobs) and fills the form.
- **Live serial monitor and log** with colour-coded lines, zoom, copy and clear. The device ID is picked up automatically from the boot message.
- **IR learning window** (probe, per-button capture, commit, abort) using the serial protocol above.
- **IR profiles:** save a device's captured codes as a named profile and apply it to other devices when writing their configuration.
- Responsive layout (side-by-side or stacked depending on window shape) with a draggable divider.

### 13.2 How a write works

1. The serial monitor is released so the port is free.
2. The tool **reads the existing NVS partition first** and keeps every other namespace (learned IR codes, remembered AC state). A plain "change the SSID" write therefore never wipes learned data.
3. Only the `cfg` namespace is replaced with the values from the form. If a profile is selected, its codes replace `irlearn` and `ac_proto` is forced to `LEARNED`.
4. An NVS image is generated and flashed with the embedded flashing library (460800 baud), then the board is reset.
5. Optional repeater fields left blank are not written, so the firmware's own defaults apply.

Only one flashing operation can run at a time; extra clicks are rejected while one is in progress.

### 13.3 Building the Windows executable

The tool is packaged into one portable `.exe` with PyInstaller using a small build script.

- **Needs on the build machine:** Python 3 on the PATH.
- **Dependencies:** a serial library, the ESP flashing library, PyInstaller, and the ESP NVS image generator.
- **What the script does:** checks the required files sit together in one folder, creates a virtual environment, installs the dependencies, verifies PyInstaller, removes any locked previous build, and produces a single windowed executable with the flashing and NVS-generation libraries bundled in.
- **Result:** the `.exe` runs on any Windows PC with **no Python, no pip, no PATH setup and no Arduino tooling**.
- Windows SmartScreen will warn because the executable is unsigned ("More info", then "Run anyway").
- IR profiles are saved in an `ir_profiles` folder **next to the executable**.

---

## 14. Firmware build system

| Aspect | Detail |
|---|---|
| Toolchain | PlatformIO |
| Platform / board | Espressif 32, `esp32dev` |
| Framework | **ESP-IDF and Arduino together** (hybrid). ESP-IDF is required because the repeater's NAT support is not available in the prebuilt Arduino core |
| Libraries | An MQTT client library and an IR send/receive/AC library |
| Monitor | 115200 baud with exception decoding |
| Logging | Library debug output off; default log level none |

### Notable build-time options

| Option | Why |
|---|---|
| IP forwarding, NAPT, L2-to-L3 packet copy | Make the repeater actually route (NAT rewrites packets, so zero-copy is incompatible) |
| Up to 8 DHCP clients | Headroom for the repeater's DHCP server and NAT tables |
| 1000 Hz FreeRTOS tick | **Required** by the Arduino component; its timing assumes millisecond ticks |
| Auto-start Arduino | **Required** or the Arduino entry point is never linked and the build fails with an "undefined reference to app_main" error |
| Task watchdog 60 s | Matches the watchdog timeout used by the application |
| Flash size 2 MB | Must match **both** the build-system board setting and the ESP-IDF setting, or partition offsets are wrong. Change all three to 4 MB if your board has 4 MB |

The component build file restricts source discovery to real source extensions so it does not try to compile non-source files placed in the source folder.

---

## 15. Security notes

- **Default repeater password.** If no repeater password is stored, the firmware uses a well-known built-in default. Always set your own before enabling the repeater.
- **MQTT is unencrypted.** Connections are plain TCP on the configured port: no TLS. Credentials and commands cross the network in clear text, so keep brokers on a trusted network or behind a VPN.
- **Credentials are stored in plain text** in the ESP32's flash. Enabling flash encryption would be needed to protect them against physical access.
- **Remote configuration is powerful.** Anyone who can publish to `ac/<id>/cfg/#` on your broker can change Wi-Fi settings and reboot the unit. Restrict that topic with broker access-control rules.
- **No firmware over-the-air updates** are implemented; updates require USB.
- **Device IDs are sequential and guessable.** Do not rely on the topic name as a secret.

---

## 16. Known limitations

- The device ID is fixed at compile time, so each unit needs its own firmware build.
- Learned mode cannot control turbo, and temperature changes are relative (stepped), so a mismatch between the tracked setpoint and the AC's real setpoint can occur if someone also uses the physical remote.
- Because IR is one-way, the controller **cannot read** the AC's actual state; it reports what it last sent.
- IR timing is software generated, so extreme network load can still affect it despite the priority boost.
- The repeater is NAT-based (a routed second network), not a transparent Layer-2 bridge.
- Only 14 button slots are learnable, and only one profile can be applied per write.
