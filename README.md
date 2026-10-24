# SAMUDRA-(Off-Grid Climate Resilience Network)
SAMUDRA is a self-sufficient flood and weather early-warning system built by Team VEEKSHANA. Two ESP32 nodes exchange sensor readings over an encrypted, channel-hopping nRF24L01 radio link with no internet or mobile network, and a live Python dashboard shows the operator what is happening and raises alerts.

Hackathon project: Idea & Solution presentation, 
## Team VEEKSHANA

- B. Sai Tejaswi
- O M Ruthwikaa
- C Mahesh Babu
- G M Charan Kumar
- # COLLEGE NAME:-
-  CHADAWADA RAMANAMA ENGINEEING COLLEGE ,TIRUPATI

# Why this exists
Flash floods and rising water can leave communities with only minutes to respond.
Storms can knock out internet and mobile networks before local warning systems are ready.
The gap is the last mile: warning people locally when connectivity has failed.

SAMUDRA's answer is fewer points of failure: two local nodes, a protected radio link, and a display that runs from a USB connection alone.
# How it works
<img width="2720" height="1920" alt="Image" src="https://github.com/user-attachments/assets/61291591-0b6f-4930-9a85-7f2ff553e4ea" />
<img width="2720" height="2208" alt="Image" src="https://github.com/user-attachments/assets/a59fbde1-0561-4300-b11e-e63979f740d4" />

| Design choice | What it does |
|---|---|
| **Channel hopping** | Both nodes derive the radio channel from the current RTC second plus a shared secret (`HOP_KEY`), so the channel changes every second from matching clocks. 15 channels between 2.402 and 2.486 GHz. |
| **AES-128-CBC** | Each 32-byte reading is encrypted with the ESP32's built-in mbedTLS before it is transmitted. The IV is derived from the epoch second and `HOP_KEY`. |
| **USB display** | The receiver prints one JSON line per packet over USB, so the operator screen needs no network. |

The sender transmits near the **middle of each RTC second**, so small differences between the two clocks cannot push a packet onto the wrong channel.

## Features

- Rain, pressure, temperature and water-level monitoring
- Three alerts: **rain detected**, **water level high** (sensor-to-water distance at or below 15 cm), **pressure drop** (3 hPa over 60 s by default)
- Encrypted, frequency-hopping, auto-acknowledged radio link
- Self-healing firmware: nRF24, BMP180 and RTC problems are retried instead of hanging
- Live desktop dashboard: status banner, sensor cards, four live charts, link health (good / rejected / missed packets), event log
- Every packet saved to CSV in `dashboard/logs/` for later analysis
- One-click **Sync clocks** button that sets both boards' RTCs to the PC time in the same instant
- Demo mode: try the dashboard with simulated data and no hardware

---

## Hardware

| Part | Used for | Where |
|---|---|---|
| ESP32 DevKit V1 × 2 | Sender and receiver nodes | Both |
| nRF24L01+ × 2 | 2.4 GHz radio link | Both |
| DS3231 RTC × 2 | Shared clock for hopping and IV | Both |
| BMP180 | Pressure and temperature | Sender |
| Rain sensor (AO + DO) | Rain detection | Sender |
| HC-SR04 | Water distance | Sender |
| 10 µF capacitor × 2 | Across each nRF24's VCC/GND | Both |

### Wiring

| Signal | Sender | Receiver |
|---|---|---|
| nRF24 CE / CSN | GPIO 4 / GPIO 5 | GPIO 4 / GPIO 5 |
| nRF24 SCK / MOSI / MISO | GPIO 18 / 23 / 19 | GPIO 18 / 23 / 19 |
| I2C SDA / SCL (DS3231, BMP180) | GPIO 21 / 22 | GPIO 21 / 22 (DS3231) |
| Rain AO / DO | GPIO 34 / 35 | — |
| HC-SR04 TRIG / ECHO | GPIO 26 / 27 | — |
| Alert LED / buzzer | — | GPIO 2 / GPIO 25 |

**Notes**
- Power the nRF24L01 from **3.3 V only**, and add the 10 µF capacitor. Most "random" packet loss comes from nRF24 power noise.
- The HC-SR04 ECHO pin outputs 5 V. Use a divider to bring it to 3.3 V for the ESP32.
- Both nodes need a working DS3231: the radio channel and the encryption IV both depend on it.

## Repository layout

```
.
├── sender/                 PlatformIO project (ESP32 sender firmware)
│   ├── platformio.ini
│   └── src/main.cpp
├── receiver/               PlatformIO project (ESP32 receiver firmware)
│   ├── platformio.ini
│   └── src/main.cpp
├── dashboard/
│   ├── veekshana_dashboard.py
│   └── requirements.txt
└── README.md
```

## Getting started

### 1. Flash the firmware (VS Code + PlatformIO)

1. Install [VS Code](https://code.visualstudio.com/) and the **PlatformIO IDE** extension.
2. Open the `sender/` folder as a PlatformIO project, connect the sender ESP32, and click **Upload**.
3. Repeat with `receiver/` and the receiver ESP32.
4. If upload stalls at "Connecting...", hold the **BOOT** button on the board.

Libraries are installed automatically from `platformio.ini`: RF24, RTClib, Adafruit BMP085 Library, Adafruit BusIO.

### 2. Run the dashboard

```bash
cd dashboard
python -m venv .venv
# Windows:      .venv\Scripts\activate
# Linux / Mac:  source .venv/bin/activate
pip install -r requirements.txt
python veekshana_dashboard.py            # normal
python veekshana_dashboard.py --demo     # simulated data, no hardware needed
python veekshana_dashboard.py --rx COM5 --tx COM7   # Linux: --rx /dev/ttyUSB0
```

Tkinter ships with Python (on Debian/Ubuntu: `sudo apt install python3-tk`).

### 3. Sync the clocks

Connect **both** boards by USB (receiver port and sender port in the dashboard), then click **Sync clocks to PC time**. Both RTCs are set to the same UTC time at the same instant. If the two clocks differ by more than about half a second, packets will not line up.

Close the Arduino/PlatformIO Serial Monitor before starting the dashboard, because only one program can use a serial port at a time.

## Configuration

Settings live at the top of each `main.cpp`. Sender and receiver must match for the shared items.

| Setting | Default | Notes |
|---|---|---|
| `AES_KEY`, `HOP_KEY` | demo values | **Change both before any real deployment**, on both boards |
| `CHANNELS` | 15 channels, 2 to 86 | Must match on both |
| `WATER_LEVEL_ALERT_CM` | 15 | Sender: alert when distance to water is at or below this |
| `PRESSURE_DROP_ALERT` / `PRESSURE_WINDOW_S` | 3 hPa / 60 s | Sender: use e.g. 3 hPa over 10800 s for real storm trends |
| `RAIN_WET_THRESHOLD` | 2000 | Receiver: uncalibrated, lower raw value = wetter |
| `RF24_PA_LOW` | low power | Raise to `RF24_PA_HIGH` for range |

## Packet format

Each reading is a packed 32-byte struct (two AES blocks), encrypted before transmission:

| Field | Size | Meaning |
|---|---|---|
| nodeId | 1 | Sender ID |
| epoch | 4 | RTC Unix time |
| rainRaw | 4 | Rain sensor analog value (float) |
| pressure | 4 | hPa |
| humidity | 4 | Reserved (-1; the BMP180 has no humidity) |
| waterLevel | 4 | Distance sensor to water, cm (-1 = no echo) |
| alertFlags | 1 | bit0 rain, bit1 water high, bit2 pressure drop |
| seq | 1 | Sequence number (for missed-packet counting) |
| checksum | 1 | XOR checksum |
| reserved | 8 | First 4 bytes carry temperature (float) |

### Serial output (receiver, 115200 baud)

One JSON object per line, for example:

```json
{"src":"rx","type":"pkt","node":1,"seq":37,"epoch":1761200037,"ch":56,"rain":3000,"rain_state":"DRY","pressure":1008.00,"temp":28.50,"hum":null,"water":30.0,"flags":0,"missed":0,"good":38,"bad":0}
```

Other line types: `status` (once per second: link state, counters, RTC time) and `event` (warnings and info). The receiver accepts `T<unix_epoch>` on serial to set its RTC.

## Measuring it

The project is meant to be judged on logged evidence, not assumptions. The dashboard and CSV logs give you:

| Metric | Definition | Where to see it |
|---|---|---|
| Delivery ratio | received ÷ sent | Dashboard link card (good / rejected / missed) and CSV `seq` / `missed` |
| Data completeness | valid fields ÷ expected fields | CSV: empty cells mark missing sensor values |
| Latency | receive time − send time | Not yet automated. The sender's `epoch` and the CSV `pc_time` are the raw inputs |

## Known limitations

Being honest about what is and is not proven:

- **No message authentication.** AES-CBC hides the data but does not prove who sent it, and the XOR checksum is not cryptographic. An attacker could replay old packets. **AES-GCM** (or AES-CBC plus an HMAC) is the stronger option.
- **Keys are hard-coded** in the firmware. Anyone who can read the firmware can read the key.
- **Jamming resistance is untested.** Encryption and hopping have been exercised under ordinary conditions only.
- **Both nodes depend on a working DS3231.** Without a synced RTC the radio channel and the IV cannot match.
- **Rain and water thresholds are uncalibrated** and need tuning for the real site.
- Developed and tested as a prototype. It has not been field-tested outdoors or for power consumption.

## Roadmap
- **Switch to AES-GCM with an authentication tag and replay protection**
- **Deliberate jamming test and logged results**
- **Automated latency measurement**
- **Calibrate the rain and water-level thresholds on site**
- **Measure energy draw and add a low-power mode**
-  
