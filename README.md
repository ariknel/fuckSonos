# ESP32 Bluetooth → Sonos Bridge

> Stream **any** Android audio (YouTube, Instagram, radio apps) to your Sonos speakers — no app required.

Android users shouldn't need a specific "Sonos" button in every app. This project turns a classic ESP32 into a Bluetooth A2DP receiver that re-broadcasts audio as an HTTP stream your Sonos speaker can pull from — essentially a DIY "virtual Line-In."

---

## How it works

```
Android Phone  ──[BT A2DP]──▶  ESP32  ──[HTTP stream + UPnP]──▶  Sonos Speaker
  any app audio                  bridge                             plays stream
```

The ESP32 does three things simultaneously:

1. **Acts as a Bluetooth speaker** — your Android pairs with it once and streams all audio via A2DP
2. **Runs a tiny Icecast-style HTTP server** — wraps incoming PCM audio into a stream Sonos can pull
3. **Sends UPnP/SOAP commands** — tells the Sonos speaker "play `http://192.168.x.x:8080/stream`"

---

## Board decision

> ⚠️ **This is the most important hardware choice in the project.**

| Board | BT Classic (A2DP) | Dual-core | Verdict |
|---|---|---|---|
| **Classic ESP32 (WROOM-32E)** | ✅ Yes | ✅ Yes (2 × Xtensa LX6) | **Recommended — use this** |
| ESP32-S3 | ❌ No (BLE only) | ✅ Yes | Won't work without external BT chip |
| ESP32-S2 | ❌ No | ❌ No | Not suitable |

**Use the classic ESP32.** The S3 is faster, but it lacks Bluetooth Classic — and A2DP (the audio streaming profile your phone uses) requires Bluetooth Classic. BLE cannot carry audio streams. Unless you add an external CSR8635 or similar chip to the S3, it cannot receive A2DP. Use what you have in the drawer.

---

## Bill of materials

| Component | Part | ~Cost | Notes |
|---|---|---|---|
| Microcontroller | ESP32-WROOM-32E dev board | €4–8 | Classic ESP32 only |
| Display | SSD1306 128×64 OLED (I2C) | €2–4 | `U8g2` library |
| Input | KY-040 rotary encoder | €1–2 | Volume + speaker select |
| Audio out (optional) | PCM5102 I2S DAC | €3–6 | Adds real Line-Out jack |
| Power | USB-C breakout or 5V/1A PSU | €1–3 | 500mA min under load |
| Enclosure | 3D printed or project box | — | STL in `/hardware` |

Total BOM (without enclosure): roughly **€10–20**.

---

## Wiring

```
ESP32 pin   →   Component
─────────────────────────────────────
GPIO 21     →   OLED SDA
GPIO 22     →   OLED SCL
GPIO 18     →   Encoder CLK
GPIO 19     →   Encoder DT
GPIO 5      →   Encoder SW (button)

# Optional I2S DAC (PCM5102)
GPIO 25     →   DAC BCK (bit clock)
GPIO 26     →   DAC LCK (word select)
GPIO 22     →   DAC DIN (data)
3.3V / GND  →   DAC power
```

---

## Software architecture

```
firmware/
├── src/
│   ├── main.cpp              # Setup, core pinning
│   ├── bt_sink.cpp           # A2DP sink (esp32-a2dp lib)
│   ├── audio_buffer.cpp      # Lock-free PCM ring buffer
│   ├── http_stream.cpp       # Icecast HTTP server (ESPAsyncWebServer)
│   ├── sonos_upnp.cpp        # UPnP discovery + SOAP play/stop
│   ├── oled_ui.cpp           # U8g2 display + encoder handler
│   └── config.h              # WiFi creds, stream port, etc.
├── platformio.ini
└── README.md
```

### Core pinning strategy

```
Core 0  →  bt_sink task    (A2DP callbacks, I2S write, ring buffer)
Core 1  →  http_stream     (web server, serve PCM chunks)
           sonos_upnp      (discovery, SOAP commands)
           oled_ui         (display refresh, encoder polling)
```

Keeping BT/audio isolated on Core 0 prevents WiFi interrupts from causing audio glitches.

### Key libraries

```ini
# platformio.ini
lib_deps =
    pschatzmann/ESP32-A2DP       ; BT A2DP sink
    me-no-dev/ESPAsyncWebServer  ; async HTTP stream server
    me-no-dev/AsyncTCP           ; async TCP layer
    olikraus/U8g2                ; OLED display
```

---

## User experience flow

```
Power on
    │
    ▼
OLED: "Pair phone"
    │
Android pairs over BT (saved to NVS after first time)
    │
    ▼
ESP32 scans WiFi for Sonos speakers (UPnP M-SEARCH)
    │
OLED lists found speakers → user selects with encoder
    │
    ▼
Audio starts playing on phone
    │
ESP32 auto-sends UPnP SOAP "play http://[esp-ip]:8080/stream"
    │
    ▼
Sonos plays stream — encoder knob controls volume via UPnP
```

---

## Latency reality check

| Use case | Expected delay | Acceptable? |
|---|---|---|
| Music, podcasts, radio | 1–2 seconds | ✅ Yes |
| YouTube (audio only) | 1–2 seconds | ✅ Mostly |
| Video with lip sync | 1–2 seconds | ❌ No — lips won't match |

Sonos buffers network streams by design. This latency is inherent to the architecture and cannot be fully eliminated in software. **This bridge is ideal for music streaming, not video.**

---

## Milestones

See [`BUILD_PLAN.md`](./BUILD_PLAN.md) for the full phased roadmap.

| Phase | Goal | Status |
|---|---|---|
| 0 | Repo + toolchain | 🔲 |
| 1 | BT A2DP sink → I2S out | 🔲 |
| 2 | HTTP audio stream server | 🔲 |
| 3 | Sonos UPnP discovery + SOAP | 🔲 |
| 4 | OLED UI + rotary encoder | 🔲 |
| 5 | Integration + latency tuning | 🔲 |
| 6 | Enclosure + polish | 🔲 |

---

## Getting started

```bash
# Clone
git clone https://github.com/ariknel/fuckSonos
cd esp32-sonos-bridge

# Edit credentials
cp firmware/src/config.h.example firmware/src/config.h
nano firmware/src/config.h  # fill in WiFi SSID + password

# Flash (PlatformIO)
pio run --target upload --environment esp32dev

# Monitor serial
pio device monitor
```

---

## FAQ

**Can I use the ESP32-S3 I already have?**
Not directly. The S3 lacks Bluetooth Classic, which is required for A2DP audio streaming. You'd need to add an external Bluetooth audio module (e.g. CSR8635 breakout) and route I2S audio between them — significantly more complex. Use the classic ESP32.

**Does it work with Sonos Move / Roam?**
Yes. Those speakers support BT natively too, but this bridge lets you use *any* Sonos speaker including fixed speakers (Era, Five, Arc, older Play series).

**What audio quality does A2DP support?**
Standard SBC codec: 44.1 kHz, up to 328 kbps. The `esp32-a2dp` library also supports AAC if your phone negotiates it.

**Will this work on the same WiFi as Sonos?**
Yes — Sonos and the ESP32 must be on the same LAN subnet. UPnP discovery uses multicast which doesn't cross subnets.

---

## License

MIT — do whatever you want, attribution appreciated.
