# RuView Project — Complete Context Document
*Everything discussed, built, fixed, and verified across the entire conversation*

---

## 1. PROJECT OVERVIEW

### What is RuView?
- **GitHub:** https://github.com/ruvnet/RuView
- **Description:** WiFi DensePose — turns commodity WiFi signals into real-time human pose estimation, vital sign monitoring, and presence detection without cameras
- **Stars:** 32,900+ (suspected inflated)
- **License:** MIT
- **Author:** ruvnet (not us — we are building on top of it)
- **Based on:** Carnegie Mellon University's "DensePose From WiFi" research (arXiv:2301.00250)

### What RuView Claims
- Pose estimation (17 keypoints) from WiFi CSI
- Breathing rate detection (6-30 BPM)
- Heart rate detection (40-120 BPM)
- Presence detection
- Through-wall sensing
- Multi-person tracking
- 54,000 frames/sec (Rust pipeline)
- 810x faster than Python

---

## 2. CODE AUTHENTICITY AUDIT

We conducted a thorough audit using manual inspection and GitHub Copilot (GPT-5.3-Codex).

### RSSI Collection (v1/src/sensing/rssi_collector.py) — REAL ✅
- `LinuxWifiCollector` reads from real `/proc/net/wireless`
- `WindowsWifiCollector` uses real `netsh wlan show interfaces`
- `MacosWifiCollector` compiles and runs real Swift CoreWLAN binary
- `SimulatedCollector` clearly labeled as simulation/testing only
- `create_collector()` factory tries real hardware first, falls back to simulation
- **Verdict: 100% legitimate code**

### CSI Extraction (v1/src/sensing/csi_extractor.py) — MIXED ⚠️
- `ESP32BinaryParser` — REAL: correctly parses binary I/Q data
- Binary format: magic `0xC5110001`, I/Q pairs, amplitude/phase
- UDP listener `_read_udp_data()` — REAL: genuine datagram endpoint
- `_establish_hardware_connection()` — FAKE: placeholder returns `True`
- `_read_raw_data()` default path — FAKE: returns hardcoded bytes
- Atheros parser — NOT IMPLEMENTED: raises `CSIExtractionError` honestly
- **Verdict: CSI parsing works with real ESP32 in binary mode**

### Neural Network (v1/src/models/densepose_head.py) — ARCHITECTURE REAL, WEIGHTS MISSING ❌
- PyTorch architecture is legitimate (Conv2d, BatchNorm, FPN, etc.)
- No pretrained weight files (.pth, .onnx) in v1 path
- Models folder only has `.pyc` bytecode files
- `.rvf` files exist but weight loading code is COMMENTED OUT in `pose_service.py`
- Constructor called without required config (bug)
- Output format mismatch: returns dict, service expects tensor
- **Verdict: Pose estimation CANNOT work — 3 separate bugs blocking it**

### End-to-End Pipeline — BROKEN ❌
- `CSIExtractor` is never called anywhere in the pose pipeline
- No training script connecting CSI data to DensePose model
- `np.random.rand` NOT found in production code (only in tests)
- **Verdict: No working end-to-end CSI → Pose pipeline**

### Summary Table
| Component | Verdict |
|---|---|
| RSSI collection | ✅ Real |
| ESP32 binary CSI parser | ✅ Real |
| UDP listener | ✅ Real |
| Hardware handshake | ❌ Placeholder |
| DensePose architecture | ✅ Real code |
| Pretrained weights | ❌ Missing/commented out |
| End-to-end pose pipeline | ❌ Broken |
| Breathing/heart rate (Rust server) | ✅ Real and working |

---

## 3. FKCCI MANTHAN CONTEXT

### What is Manthan?
- FKCCI (Federation of Karnataka Chambers of Commerce and Industry)
- Annual business plan competition for UG/PG/Diploma students across Karnataka
- 18th edition in 2026
- Evaluates: original, commercially viable, implementable business proposals
- Website: https://manthan.fkcci.org

### Our Approach
- We are NOT claiming to have invented the technology
- We are building a startup/business on top of open-source WiFi sensing
- Our innovation: business model, India localization, vertical focus, commercialization
- We disclose open-source foundation in synopsis

### Business Name: SenseWave
- Tagline: "See without watching"
- Team: 4+ members, no registered company
- Target verticals: Eldercare, Hospital Monitoring, Smart Building

---

## 4. HARDWARE SETUP

### ESP32-S3 Board Details
- **Chip:** ESP32-S3 (QFN56) revision v0.2
- **Flash:** 8MB (confirmed via `flash-id` command)
- **PSRAM:** 2MB embedded (AP_3v3)
- **MAC:** b4:3a:45:3f:6d:74
- **USB Mode:** USB-Serial/JTAG
- **COM Port:** COM7 (Windows)
- **Antenna:** External IPEX/U.FL with DTP9753-ANT2-1.6 PCB antenna (removable)
- **USB-C ports:** Two — top (Serial/data), bottom (JTAG/programming)
- **Use black cable (bottom JTAG port) for flashing**

### Key Hardware Facts
- ESP32-S3 fully supports WiFi CSI ✅
- Standard ESP32 (original) has limited CSI — not recommended
- ESP32-C3 Mini works but has first-word-invalid bug
- ESP32-S3 is recommended and tested board for RuView

---

## 5. FIRMWARE JOURNEY

### Version Used
- v0.5.0-esp32 from https://github.com/ruvnet/RuView/releases/tag/v0.5.0-esp32

### Flash Offset Discovery (Critical)
- Wrong: `0x10000` (from Issue #34 tutorial — for 4MB non-OTA layout)
- Wrong: caused "No bootable app partitions" error
- Correct: `0x20000` with `ota_data_initial.bin` at `0xf000` (for 8MB OTA layout)

### Correct Flash Command (8MB board, custom firmware)
```powershell
python -m esptool --chip esp32s3 --port COM7 --baud 460800 --before default-reset --after hard-reset write-flash --flash-mode dio --flash-size 8MB --flash-freq 80m 0x0 bootloader-custom.bin 0x8000 partition-table-custom.bin 0xf000 ota_data_initial.bin 0x20000 esp32-csi-node-custom.bin
```

### Watchdog Bug (task_wdt: edge_dsp)
- Prebuilt v0.5.0 binary had watchdog crash bug
- `edge_dsp` task on CPU 1 never yields, starves IDLE1
- Error: `E (27818) task_wdt: CPU 1: edge_dsp`
- Backtrace: `0x4037890F:0x3FC9D6A0 0x4037746D:0x3FC9D6C0 0x4200D225:0x3FCC9C00`
- Fix: Build firmware from source — source has `vTaskDelay(1)` at line 842 of `edge_processing.c`
- Also increased watchdog timeout: `CONFIG_ESP_TASK_WDT_TIMEOUT_S=30` (was 5)

### Building Firmware From Source (WSL2)
```bash
cd ~/RuView/firmware/esp32-csi-node
docker run --rm -v "$(pwd):/project" -w /project \
  espressif/idf:v5.2 bash -c "idf.py build"
```
- Requires Docker Desktop with WSL2 integration enabled
- Takes 5-10 minutes first time (1299-1341 compilation steps)
- Output at: `build/esp32-csi-node.bin`, `build/bootloader/bootloader.bin`, etc.

### Parser Fix (Magic Number 0xC5110002)
- ESP32 sends vitals packets with magic `0xC5110002`
- Aggregator only knew `0xC5110001` — rejected vitals packets with "Invalid magic" error
- Fix in `esp32_parser.rs` line 73:
```rust
if magic != ESP32_CSI_MAGIC && magic != 0xC5110002 {
```

### sdkconfig Changes Made
```
CONFIG_ESP_TASK_WDT_TIMEOUT_S=30  (was 5)
```

---

## 6. PROVISIONING

### Command
```powershell
python C:\Users\Ritham\Desktop\RuView\firmware\esp32-csi-node\provision.py --port COM7 --ssid "YOUR_WIFI" --password "YOUR_PASSWORD" --target-ip YOUR_LAPTOP_IP
```

### Important Notes
- NO `--edge-tier` flag (caused issues when added)
- Re-provision whenever laptop IP changes
- Writes to NVS partition at `0x9000`
- No need to reflash firmware when re-provisioning
- Close Arduino Serial Monitor before provisioning (port conflict)
- Common error: provisioned with literal "YOUR_IP" text — always check

### Networks Used
- **Phone hotspot:** SSID `raowaifi`, password `netbeka123`
- **Home WiFi:** SSID `Lakshmi_5GEXT`, password `9448486523`
- Hotspot laptop IP: `172.31.23.238`, ESP32 IP: `172.31.23.83`
- Home WiFi laptop IP: `192.168.0.114`

---

## 7. SOFTWARE STACK

### Primary: Rust Sensing Server (WSL2)
```bash
cd ~/RuView/rust-port/wifi-densepose-rs
cargo run -p wifi-densepose-sensing-server -- --source esp32 --bind-addr 0.0.0.0 --udp-port 5005 --http-port 3000 --ws-port 3001 --ui-path ../../ui
```
- Port 3000: HTTP dashboard
- Port 3001: WebSocket
- Port 5005: UDP from ESP32

### Aggregator (for raw CSI debugging)
```bash
cargo run -p wifi-densepose-hardware --bin aggregator -- --bind 0.0.0.0:5005 --verbose
```
- Shows: `[node:1 seq:0] sc=192 rssi=-30 amp=20.2`
- Use to verify ESP32 is streaming before running full server

### Windows WDAC Issue (Error 4551)
- Windows Application Control policy blocks unsigned Rust executables
- Cannot run Rust binaries compiled on Windows
- Solution: Use WSL2 for ALL Rust compilation and execution
- esptool (Python) works fine on Windows

### WSL2 Setup
- Ubuntu installed via `wsl --install`
- Docker Desktop with WSL2 integration enabled
- Repo cloned at `~/RuView`
- Rust installed via rustup
- WSL2 shares localhost with Windows — same IP address

---

## 8. VERIFIED WORKING FEATURES

### ESP32 CSI Vital Signs ✅ (PRIMARY DEMO)
- Breathing rate: 9.375 BPM (real, verified)
- Heart rate: 59-74 BPM (real, verified)
- Breathing confidence: 85%
- Heart rate confidence: 41-50%
- Source field: `"esp32"` confirmed in API
- Smartwatch comparison: showed 75 BPM vs RuView 59 BPM (16 BPM gap at 2min)
- With 5+ minutes: gap reduces to ±5 BPM
- Needs: sitting still, 1-2 meters from ESP32, chest facing antenna

### Observatory Dashboard ✅
- Beautiful holographic Three.js visualization
- Pose skeleton, vital signs panels, presence heatmap
- URL: `http://localhost:3000/observatory.html`
- Real vitals when ESP32 connected, simulation fallback otherwise

### RSSI Presence Detection ✅ (Inconsistent)
- Works with home WiFi router at -42 dBm
- Detects: present_still 🧍 → active 🏃 → absent
- Intel AX203 driver randomly clamps RSSI to fixed value
- Settings: `presence_variance_threshold=0.1`, `motion_energy_threshold=0.02`, `WINDOW_SEC=6.0`
- Works when: RSSI above -50 dBm, direct line between laptop and router

---

## 9. API ENDPOINTS

All at `http://localhost:3000`:

| Endpoint | Description |
|---|---|
| `/health` | Server health |
| `/api/v1/vital-signs` | Breathing, heart rate, confidence |
| `/api/v1/sensing/latest` | Full CSI frame with amplitude data |
| `/api/v1/status` | Source (esp32/simulate), ready status |
| `/api/v1/pose/current` | Pose estimation |
| `/api/v1/info` | Server info |
| `/ui/index.html` | Main UI (8 tabs) |
| `/observatory.html` | Holographic Observatory |

### UI Tabs
1. Dashboard — system status, active persons, zone occupancy
2. Hardware — antenna array visualization
3. Demo — live signal analysis + pose canvas
4. Architecture — pipeline diagram
5. Performance — benchmark charts (static, from CMU paper)
6. Applications — use case cards
7. Sensing — live breathing/heart rate graphs (ESP32 data)
8. Training — model management
9. Pose Fusion (separate page)
10. Observatory (separate page) — best visual

---

## 10. ENVIRONMENT

### User Details
- **Name:** Ritham
- **Location:** Bengaluru, Karnataka
- **OS:** Windows (main), WSL2 Ubuntu (for Rust)
- **Laptop:** Intel Wi-Fi 6 AX203 adapter
- **Driver version:** 23.160.0.4

### Windows Paths
```
C:\Users\Ritham\Desktop\RuView\          — Main repo
C:\Users\Ritham\Desktop\firmware\        — Flash binaries
  bootloader-custom.bin                  — Custom built
  partition-table-custom.bin             — Custom built
  esp32-csi-node-custom.bin              — Custom built (watchdog fixed)
  ota_data_initial.bin                   — OTA data
C:\Users\Ritham\.cargo\bin\cargo.exe     — Rust (full path needed on Windows)
C:\Python314\                            — Python installation
```

### WSL2 Paths
```
~/RuView/                                — Main repo
~/RuView/firmware/esp32-csi-node/        — Firmware source
~/RuView/firmware/esp32-csi-node/build/  — Built firmware
~/RuView/rust-port/wifi-densepose-rs/    — Rust workspace
~/RuView/v1/                             — Python legacy
~/RuView/v1/tests/integration/live_sense_monitor.py
```

### Windows Issues Encountered
1. **WDAC error 4551** — blocks Rust → use WSL2
2. **netsh WiFi not found** — needs Location Services ON + Admin rights
3. **COM7 port busy** — close Arduino Serial Monitor before provisioning
4. **Intel AX203 RSSI clamping** — driver randomly reports fixed values
5. **winget not found** — not available on this Windows version
6. **`cd C:\path`** — doesn't work in WSL2, use `/mnt/c/path`

---

## 11. COMPLETE FLASH HISTORY

| Attempt | Command | Result |
|---|---|---|
| Flash 1 | `0x20000 esp32-csi-node.bin` with `ota_data_initial.bin` | Watchdog crash |
| Flash 2 | `0x10000 esp32-csi-node.bin` (tutorial offset) | "No bootable partitions" |
| Flash 3 | `0x20000` correct 8MB partition table | Watchdog crash |
| Flash 4 ✅ | Custom firmware `0x20000` with `0xf000 ota_data_initial.bin` | **WORKING** |

---

## 12. SERIAL MONITOR OUTPUT REFERENCE

### Good Boot (Working)
```
I (251) cpu_start: Pro cpu start user code
I (1690) esp_netif_handlers: sta ip: 172.31.23.83
I (1690) main: Got IP: 172.31.23.83
I (1690) main: Connected to WiFi
I (2310) main: CSI streaming active → 172.31.23.238:5005
I (2170) csi_collector: CSI cb #1: len=384 rssi=-28 ch=13
```

### Watchdog Crash (Bad)
```
E (27818) task_wdt: Task watchdog got triggered
E (27818) task_wdt: CPU 1: edge_dsp
Backtrace: 0x4037890F:0x3FC9D6A0...
```

### Wrong Flash Offset
```
E (26) flash_parts: partition 4 invalid - offset 0x220000 exceeds flash
E (27) boot: Failed to verify partition table
E (28) boot: load partition table error!
```

### Wrong Target IP
```
E (1690) stream_sender: Invalid target IP: YOUR_IP
E (1700) main: Failed to initialize UDP sender
```

### Server Not Running (errno 118)
```
W (30413) stream_sender: sendto failed: errno 118
```

### Memory Error (ENOMEM)
```
W (4520) stream_sender: sendto ENOMEM — backing off for 100 ms
```

---

## 13. TROUBLESHOOTING GUIDE

| Symptom | Cause | Fix |
|---|---|---|
| `tick: 0`, `samples: 0` | No ESP32 data | Check IP, server, provisioning |
| `rssi_dbm: 0.0` | Simulation mode | ESP32 not connected |
| `errno 118` | Server not running | Start server first, then RST |
| `ENOMEM` | Started ESP32 before server | Server first, then RST |
| `parse error: Invalid magic` | Firmware mismatch | Use custom built firmware |
| Watchdog crash | edge_dsp task | Use custom built firmware |
| "No bootable partitions" | Wrong flash offset | Use `0x20000` not `0x10000` |
| "Invalid target IP: YOUR_IP" | Bad provisioning | Re-provision with real IP |
| COM7 busy | Another app using port | Close Arduino Serial Monitor |
| WDAC error 4551 | Windows security policy | Use WSL2 for Rust |
| RSSI stuck at fixed value | Intel driver clamping | Can't reliably fix |

---

## 14. CORRECT STARTUP SEQUENCE (EVERY TIME)

```
1. Turn on WiFi (hotspot or home router)
2. Connect laptop to WiFi
3. ipconfig → note WiFi IPv4
4. If IP changed from last time → re-provision ESP32
5. Start sensing server in WSL2 (wait for "UDP listening on 0.0.0.0:5005")
6. Press RST on ESP32
7. Wait 15-20 seconds for ESP32 to connect
8. Open http://localhost:3000/observatory.html
9. Wait 3-4 minutes for breathing_samples to reach 300
10. Check http://localhost:3000/api/v1/vital-signs for real data
```

**CRITICAL: Always start server BEFORE pressing RST on ESP32**

---

## 15. MANTHAN DEMO PLAN

### Hardware
- ESP32-S3 (custom firmware, watchdog fixed)
- USB-C cable or power bank
- Laptop running WSL2
- Phone hotspot OR home WiFi

### What to Show
1. Observatory dashboard — real heart rate + breathing
2. Smartwatch comparison side by side
3. Ask judge to walk near ESP32 — presence detection
4. API endpoint — raw JSON data

### Demo Script (5 minutes)
```
0:00-0:30  Hold ESP32 — "₹800, this is our entire sensor. No camera."
0:30-1:30  Observatory UI — "My heart rate: 59 BPM. Breathing: 9/min. No wearable."
1:30-2:30  Smartwatch vs WiFi — "Two technologies, same measurement"
2:30-3:30  Judge walks near ESP32 — "You try it — it detects you"
3:30-4:30  Applications slide — eldercare, hospital, smart buildings
4:30-5:00  Business model — ₹800 hardware + ₹299/room/month
```

---

## 16. IMPORTANT COMMANDS REFERENCE

### Flash (Windows PowerShell)
```powershell
cd C:\Users\Ritham\Desktop\firmware
python -m esptool --chip esp32s3 --port COM7 --baud 460800 --before default-reset --after hard-reset write-flash --flash-mode dio --flash-size 8MB --flash-freq 80m 0x0 bootloader-custom.bin 0x8000 partition-table-custom.bin 0xf000 ota_data_initial.bin 0x20000 esp32-csi-node-custom.bin
```

### Provision (Windows PowerShell)
```powershell
python C:\Users\Ritham\Desktop\RuView\firmware\esp32-csi-node\provision.py --port COM7 --ssid "raowaifi" --password "netbeka123" --target-ip 172.31.23.238
```

### Sensing Server (WSL2)
```bash
cd ~/RuView/rust-port/wifi-densepose-rs
cargo run -p wifi-densepose-sensing-server -- --source esp32 --bind-addr 0.0.0.0 --udp-port 5005 --http-port 3000 --ws-port 3001 --ui-path ../../ui
```

### Aggregator Debug (WSL2)
```bash
cargo run -p wifi-densepose-hardware --bin aggregator -- --bind 0.0.0.0:5005 --verbose
```

### Build Custom Firmware (WSL2)
```bash
cd ~/RuView/firmware/esp32-csi-node
docker run --rm -v "$(pwd):/project" -w /project espressif/idf:v5.2 bash -c "idf.py build"
```

### Copy Firmware to Windows (WSL2)
```bash
cp ~/RuView/firmware/esp32-csi-node/build/esp32-csi-node.bin /mnt/c/Users/Ritham/Desktop/firmware/esp32-csi-node-custom.bin
cp ~/RuView/firmware/esp32-csi-node/build/bootloader/bootloader.bin /mnt/c/Users/Ritham/Desktop/firmware/bootloader-custom.bin
cp ~/RuView/firmware/esp32-csi-node/build/partition_table/partition-table.bin /mnt/c/Users/Ritham/Desktop/firmware/partition-table-custom.bin
cp ~/RuView/firmware/esp32-csi-node/build/ota_data_initial.bin /mnt/c/Users/Ritham/Desktop/firmware/ota_data_initial.bin
```

### RSSI Monitor (Windows PowerShell)
```powershell
cd C:\Users\Ritham\Desktop\RuView
$env:PYTHONPATH = "."
python v1/tests/integration/live_sense_monitor.py
```

### Check ESP32 on Network
```powershell
arp -a | findstr "b4-3a"
ping 172.31.23.83
```

### Firewall Rules (Windows Admin)
```powershell
netsh advfirewall firewall add rule name="RuView UDP 5005" protocol=UDP dir=in localport=5005 action=allow
netsh advfirewall firewall add rule name="RuView TCP 3000" protocol=TCP dir=in localport=3000 action=allow
netsh advfirewall firewall add rule name="RuView TCP 3001" protocol=TCP dir=in localport=3001 action=allow
```

---

## 17. WHAT DOESN'T WORK

| Feature | Status | Reason |
|---|---|---|
| Pose estimation (17 keypoints) | ❌ | No trained weights, 3 code bugs |
| RSSI detection (reliable) | ❌ | Intel AX203 driver clamps values |
| Atheros router CSI | ❌ | Not implemented |
| Python v1 server | ❌ | Missing dependencies, broken imports |
| Rust on Windows | ❌ | WDAC blocks unsigned executables |
| ESP32 prebuilt firmware | ❌ | Watchdog bug on revision v0.2 |

---

## 18. MANTHAN SYNOPSIS KEY NUMBERS

- Hardware cost: ₹800 per room
- vs Camera system: ₹15,000+ per room
- Breathing accuracy: 85% confidence
- Heart rate: ±5 BPM with 5min calibration, ±16 BPM at 2min
- Market (eldercare): ₹4,200 crore by 2027
- Market (hospital): ₹8,900 crore by 2026
- Market (smart building): ₹12,500 crore by 2028
- Revenue model: ₹2,500 setup + ₹299/room/month SaaS
- Year 1 target: 500 rooms, ₹12.5L revenue
- Year 2 target: 2,000 rooms, ₹71L revenue
- Year 3 target: 8,000 rooms, ₹2.8Cr revenue
- Funding ask: ₹15 lakh seed

---

## 19. NEXT STEPS

### Before Manthan
- [ ] Test full setup at venue conditions
- [ ] Practice 5-minute demo 3 times
- [ ] Have simulation mode as backup if ESP32 fails
- [ ] Charge power bank
- [ ] Fill team names + college in synopsis
- [ ] Print synopsis for submission

### Technical Roadmap
- [ ] Build activity classifier (sitting/standing/walking/fallen) using collected CSI data
- [ ] Fix 3 pose estimation bugs for future pose capability
- [ ] Add WhatsApp alert integration for eldercare
- [ ] Multi-room mesh networking
- [ ] Mobile app for caregiver alerts
- [ ] Clinical validation study

### Business Roadmap
- [ ] Register company (LLP or Pvt Ltd)
- [ ] Approach 3 retirement homes in Bengaluru for pilot
- [ ] Apply for Karnataka startup grants (KBITS, iStart)
- [ ] NASSCOM 10,000 Startups application
