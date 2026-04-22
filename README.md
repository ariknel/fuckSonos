# ESP32-S3 Bluetooth → Sonos Bridge

> Stream **any** Android audio — YouTube, Instagram, radio apps, anything — to your Sonos speakers. No app required. No "Cast" button needed. Controlled via a web UI from any browser on your network.

Android users shouldn't need a specific "Sonos" button in every app. This project turns an ESP32-S3 into a Bluetooth A2DP receiver that re-broadcasts audio as an HTTP stream your WiFi speakers can pull from — a DIY "virtual Line-In." Speaker selection, volume, and grouping are managed through a responsive web interface served directly from the device.

---

## The problem this solves

Every major multiroom speaker platform — Sonos, Denon HEOS, Yamaha MusicCast, Bluesound BluOS, Bose SoundTouch, Samsung SmartThings Audio — shares the same fundamental design philosophy: **they decide what you can play, not you.**

Each platform maintains its own curated list of supported streaming services and its own cast protocol. If your app isn't on their approved list, you're out of luck. As an Android user, your options typically boil down to:

- **Spotify → Sonos**: works, via Spotify Connect
- **YouTube Music → Sonos**: works, via Chromecast (on compatible models only)
- **Random internet radio app → Sonos**: doesn't work
- **Instagram Reels audio → Sonos**: doesn't work
- **A podcast app that hasn't partnered with Sonos → Sonos**: doesn't work
- **System audio, game sounds, anything else → Sonos**: doesn't work

Apple users get a partial escape hatch via AirPlay 2, which acts as a system-level cast protocol accepted by most of these platforms. **Android has no equivalent.** Google Cast/Chromecast is supported by some platforms but not all, and it still requires the specific app to implement a Cast button. There is no Android mechanism that says "send *all* phone audio to these speakers."

The platforms are aware of this gap. They've chosen not to fill it because a walled garden protects their service integrations, keeps users inside their own app, and justifies the premium hardware price. Sonos has faced years of community complaints about arbitrary app support. HEOS, MusicCast, and Bluesound have the same limitation with slightly different boundaries. Bose discontinued its SoundTouch line entirely, migrating to a model that leans on AirPlay 2 and Alexa — which still doesn't help Android users wanting arbitrary audio routing. Yamaha's MusicCast is locked to Yamaha hardware only. Samsung's emerging Music Studio line is yet another closed ecosystem.

This bridge sidesteps the entire problem. It pairs with your phone as a Bluetooth speaker — receiving *all* audio from any app at the OS level — then re-streams it over the same UPnP/HTTP protocol those speakers already use for everything else. The speakers don't know or care where the stream originates.

---

## How it works

```
Android Phone  ──[BT A2DP]──▶  BM83 Module  ──[I2S]──▶  ESP32-S3  ──[HTTP + UPnP]──▶  Sonos Speaker(s)
  any app audio              Classic BT sink             stream server                    plays in sync
```

The ESP32-S3 does three things at once:

1. **Receives audio via I2S from the BM83** — the BM83 acts as the Bluetooth A2DP sink; the ESP32-S3 reads the decoded PCM stream over I2S
2. **Runs a tiny HTTP stream server** — wraps incoming PCM audio into a stream Sonos can pull from a URL
3. **Serves a web control UI** — speaker selection, grouping, and per-room volume; accessible from any browser on your LAN
4. **Sends UPnP/SOAP commands** — tells target speakers to play that URL, controls volume and grouping

---

## Why ESP32-S3?

The S3 was chosen deliberately:

- **More RAM** — 512 KB SRAM vs 320 KB on the classic ESP32. Enough to run AsyncWebServer comfortably alongside the audio buffer and UPnP stack
- **Faster CPU** — Xtensa LX7 cores at 240 MHz vs LX6, handling the web UI and audio pipeline without contention
- **USB OTG** — easier firmware updates and serial monitoring
- **No built-in Bluetooth Classic** — this is solved with an external BM83 module (see below), which is actually a cleaner architecture: the BT radio and audio decoding are handled by dedicated hardware, and the S3 focuses entirely on networking and control

---

## Choosing a Bluetooth module

The ESP32-S3 has BLE only — it cannot receive A2DP audio from a phone without an external chip. This section is the most important part of the BOM to get right, because the module market is scattered, availability varies wildly, and the wrong choice will waste a weekend.

### The honest overview: scarcity is a real problem

There is no single "just buy this" answer in 2025. The module options fall into three tiers, and **every tier has a catch**:

- **Budget tier (€2–6):** CSR8645 and its derivatives are everywhere on Aliexpress. They will not work for this project — closed firmware, no usable UART host control.
- **Mid tier (€8–15):** The BM83 (Microchip) is the best-fit module for host-controlled A2DP, but it is genuinely hard to find as a hobbyist-friendly breakout. The bare chip is in stock at Mouser/LCSC, but requires a carrier PCB. Tindie sellers occasionally stock breakouts, but with "low stock due to chip shortage" notices.
- **Premium tier (€20–40):** Qualcomm QCC3083/3084-based modules are increasingly available on Aliexpress and from Chinese module houses. They support LDAC, aptX Lossless, LC3, and every modern codec — but MCU host control requires Qualcomm's proprietary VM/GAIA protocol over I2C or UART, which has almost no hobbyist documentation outside of NDA-gated developer access.

### Module comparison

| Module | Chip | BT ver | Codecs | I2S out | Host control | Availability | ~Price | Verdict |
|---|---|---|---|---|---|---|---|---|
| **BM83** | Microchip IS2083BM | 5.0 | SBC, AAC | ✅ | ✅ UART, full docs | ⚠️ Mouser/LCSC (chip), rare breakouts | €8–15 | **Recommended** |
| QCC3083 / QCC3084 | Qualcomm | 5.3–5.4 | SBC, AAC, aptX, aptX HD, aptX LL, LDAC, LC3 | ✅ | ⚠️ Proprietary GAIA/VM protocol, NDA docs | ✅ Aliexpress (MY-BT304B and others) | €20–40 | Overkill + hard to control |
| CSR8645 / CSRA64215 | Qualcomm (legacy) | 4.0–4.2 | SBC, AAC, aptX | ✅ (PCM header, tricky) | ❌ Closed firmware | ✅ Everywhere | €2–5 | **Avoid** — no host control |
| BC127 (SparkFun) | CSR/Qualcomm | 4.0 | SBC, AAC, aptX | ❌ analog only | ✅ UART "Melody" commands | ⚠️ SparkFun, occasional stock | €15–25 | Audio via analog only, no I2S |
| HC-05 / HC-06 | CSR | 2.0 | ❌ no audio | ❌ | ✅ AT commands | ✅ Everywhere | €1–3 | **Avoid** — data only |

### ⭐ Recommended: Microchip BM83

The BM83 is the recommended module for this project. It runs in **Host mode**: the ESP32-S3 controls pairing, connection events, and audio routing via UART commands defined in a fully public datasheet. Audio is delivered over I2S directly into the S3's DMA buffers. This gives clean separation between the BT radio layer and the networking/control layer.

**What makes it the right fit:**
- Complete UART command set, documented in a freely downloadable Microchip datasheet — no NDA, no SDK registration wall
- I2S master output, clock and frame sync provided by the BM83, just connect 3 wires to the S3
- Firmware configurable with Microchip's free GUI tool (Windows)
- Supports SBC and AAC — sufficient for music streaming to Sonos
- FCC/CE certified (important if you ever commercialise)

**The catch:** The bare BM83SM1 stamp module is available at Mouser (~€8) and LCSC, but you need to solder it to a breakout PCB or order a pre-made board. Pre-built breakout boards are available from Tindie sellers (search "BM83 breakout") but stock comes and goes. The official Microchip EVB (DM164152) is ~€80 and is fine for development but overkill for a finished build.

**Where to buy:**
- Bare module: [Mouser](https://www.mouser.com) or [LCSC](https://www.lcsc.com) — search `BM83SM1-00AA`
- Breakout board: [Tindie — BurgessWorld](https://www.tindie.com/products/cburgess129/bm83-bluetooth-50-dual-mode-breakout-board/) (check stock)
- MikroElektronika "BT Audio 3 Click" is another pre-made option with I2S exposed

### About the Qualcomm QCC3083/3084 modules

If you find a QCC3084-based board on Aliexpress (search "MY-BT304B" or "QCC3084 I2S module"), it is technically capable hardware. It supports every modern codec — LDAC, aptX Lossless, LC3 — and outputs I2S cleanly. For a passive audio receiver where the module runs standalone (embedded mode, no MCU control), it works well.

**The problem for this project:** controlling a QCC module from an MCU requires Qualcomm's GAIA (Generic Application Interface Architecture) protocol, which runs over I2C or UART. Public documentation for GAIA is sparse and partial. Qualcomm's full developer resources are behind an NDA-gated portal. You can find community-reverse-engineered GAIA command tables, but they are incomplete and chip-revision-dependent. The QCC3008 datasheet (an older, more open Qualcomm part) gives you a taste of the SPI/UART config complexity. For QCC3083/3084, expect to spend significant time reverse-engineering command sequences before you get reliable pairing and event handling from an external MCU.

For this project's use case — stream audio to Sonos, control via web UI — aptX HD or LDAC gives you no benefit over AAC (Sonos buffers and resamples the stream anyway). The codec quality advantage of a €35 QCC module is completely wasted here. **Save the QCC3084 board for a high-end headphone amp project where the codec actually matters.**

### Supported host boards

Any board with I2S input and a spare UART works. The BM83's I2S interface makes it board-agnostic.

| Board | Notes |
|---|---|
| **ESP32-S3-DevKitC-1** | Reference board for this project |
| ESP32-S3-WROOM-1 variants | Any S3 variant with PSRAM |
| ESP32 (classic WROOM-32E) | Has native A2DP — BM83 not needed here |
| Raspberry Pi (any) | I2S + UART available, different software stack |
| STM32 + ESP8266/ESP-01 for WiFi | Possible but significantly more complex |

---

## Bill of materials

| Component | Part | ~Cost | Notes |
|---|---|---|---|
| Microcontroller | ESP32-S3-DevKitC-1 | €5–10 | 8MB flash, 2MB PSRAM recommended |
| BT module | Microchip BM83 (IS2083BM) | €6–12 | A2DP sink + I2S output |
| Power | USB-C 5V/1A | €0–2 | Dev board powered via USB |
| Enclosure | 3D printed or project box | — | STL in `/hardware` |

**Total: roughly €12–22.** No display, no encoder — the web UI replaces all of it.

---

## Wiring

```
ESP32-S3 pin   →   BM83 pin
────────────────────────────────────────
GPIO 4         →   BM83 BCLK  (I2S bit clock)
GPIO 5         →   BM83 LRCK  (I2S word select / LR clock)
GPIO 6         →   BM83 SDOUT (I2S data out from BM83 → S3)
GPIO 17 (TX)   →   BM83 UART RX  (host commands to BM83)
GPIO 18 (RX)   →   BM83 UART TX  (events from BM83)
3.3V / GND     →   BM83 VDD / GND
```

> The BM83 requires a brief reset sequence on boot and a few UART init commands to enter Host mode. See `bt_bm83.cpp` for the initialisation flow.

---

## Software architecture

```
firmware/
├── src/
│   ├── main.cpp              # Setup, core pinning, top-level state machine
│   ├── bt_bm83.cpp           # BM83 UART host driver (pairing, events, volume)
│   ├── audio_buffer.cpp      # Lock-free PCM ring buffer (fed from I2S DMA)
│   ├── http_stream.cpp       # HTTP audio stream server
│   ├── sonos_upnp.cpp        # UPnP discovery + SOAP control
│   ├── sonos_groups.cpp      # Speaker selection + group management
│   ├── web_ui.cpp            # AsyncWebServer: serves SPA + REST API
│   ├── nvm_store.cpp         # NVS persistence (pairing, last config)
│   └── config.h              # WiFi creds, stream port, etc.
├── data/                     # SPIFFS: web UI static files (HTML/CSS/JS)
│   └── index.html
├── platformio.ini
└── README.md
```

### Core pinning

```
Core 0  →  bt_bm83 task       (UART rx/tx with BM83, I2S DMA read, ring buffer)
Core 1  →  http_stream        (serve audio chunks over HTTP)
           sonos_upnp          (UPnP discovery, SOAP commands)
           sonos_groups        (group state machine)
           web_ui              (AsyncWebServer: SPA + REST)
```

### Key libraries

```ini
# platformio.ini
lib_deps =
    me-no-dev/ESPAsyncWebServer
    me-no-dev/AsyncTCP
    bblanchon/ArduinoJson
```

No A2DP library needed — the BM83 handles the full Bluetooth stack in hardware and delivers decoded PCM via I2S.

---

## Web UI

This replaces the OLED + encoder entirely. Any phone, tablet, or laptop on the same WiFi network can open `http://sonosbridge.local` (mDNS) and control the bridge.

### Speaker selection

The UI shows Sonos room names exactly as configured in the Sonos app — not IP addresses. Toggle rooms on/off, then hit Stream.

```
┌──────────────────────────────────────────┐
│  SonosBridge                      ●●●   │
│                                          │
│  SELECT ROOMS                            │
│                                          │
│  ○  Living Room                          │
│  ○  Kitchen                              │
│  ○  Bedroom                              │
│  ○  Office                               │
│                                          │
│              [ Start Streaming → ]       │
└──────────────────────────────────────────┘
```

### Streaming view

Once audio is flowing, the UI shows which rooms are active and provides group volume + per-room sliders. Status indicators show BT, WiFi, and Sonos sync state.

```
┌──────────────────────────────────────────┐
│  SonosBridge              BT● WiFi● ✓   │
│                                          │
│  ♫  STREAMING                            │
│                                          │
│  Living Room  ████████████░░  80%  [−][+]│
│  Kitchen      ████████░░░░░░  60%  [−][+]│
│                                          │
│  Group vol    ████████████░░  75%        │
│                                          │
│              [ Stop ]                    │
└──────────────────────────────────────────┘
```

The REST API (`/api/speakers`, `/api/stream/start`, `/api/volume`) is documented in `docs/API.md` and can be called from Home Assistant or any HTTP client.

---

## Multi-speaker sync: the coordinator pattern

Naive multi-room implementations fail badly. If you independently tell each Sonos speaker to pull `http://[esp-ip]:8080/stream`, they drift out of sync within seconds — each speaker buffers differently, producing an audible echo.

The bridge uses Sonos's own coordinator architecture:

```
Selected: Living Room + Kitchen
│
▼
1. Designate first selected speaker as "coordinator"
2. Send SetAVTransportURI + Play to the coordinator only
3. Send group-join SOAP command to each follower,
   pointing them at the coordinator's session — not
   the ESP32 stream directly
4. Sonos handles hardware-level sync internally
```

The coordinator/follower pattern means the ESP32-S3 stream is pulled once, by the coordinator, then distributed in perfect sync by Sonos to all followers. This is the same mechanism the Sonos app itself uses.

---

## Edge case handling

### Speaker already playing something

Before committing, the bridge polls `GetTransportInfo`. If a speaker is in `PLAYING` state, the UI shows a warning: *"Living Room is busy — replace?"* One tap confirms.

### Speaker is in an existing Sonos group

After discovery, the bridge polls `GetGroupAttributes` on each speaker. Followers are marked in the UI: *"Kitchen — in group."* Selecting one prompts: *"Break from group?"* Confirming sends `BecomeCoordinatorOfStandaloneGroup` before adding to the bridge session.

### WiFi dropout mid-stream

A WiFi watchdog on Core 1 checks connectivity every 10 seconds. On reconnect, the bridge re-sends `SetAVTransportURI` + `Play` to the last coordinator. The UI shows "Reconnecting..." during the gap. No user action needed for brief dropouts.

### Bluetooth phone disconnects

On BM83 disconnect event (via UART), the bridge immediately sends `Stop` to all active speakers. The UI shows "Phone disconnected." On reconnect, it auto-resumes to the same speaker configuration.

### Volume conflict with Sonos app

The bridge never caches volume internally. Every `SetVolume` call is preceded by a live `GetVolume` read. Slider adjustments apply a delta to the current hardware value, not a stale local one.

---

## Latency

| Use case | Expected delay | Works? |
|---|---|---|
| Music, podcasts, radio | 1–2 seconds | ✅ Yes |
| Video with lip sync | 1–2 seconds | ❌ No |

Sonos buffers network streams by design. This latency is structural and not fixable in software. **This bridge is for music streaming, not video.**

---

## Getting started

```bash
# Clone
git clone https://github.com/yourname/esp32s3-sonos-bridge
cd esp32s3-sonos-bridge

# Configure
cp firmware/src/config.h.example firmware/src/config.h
nano firmware/src/config.h   # add WiFi SSID + password

# Flash firmware + SPIFFS (web UI)
pio run --target uploadfs --environment esp32s3dev
pio run --target upload --environment esp32s3dev

# Monitor
pio device monitor
```

On first boot: open `http://sonosbridge.local` in any browser on your network. Pair your Android phone to "SonosBridge" via Bluetooth settings. Done.

---

## Milestones

| Phase | Goal |
|---|---|
| 0 | Toolchain, repo structure, BM83 UART init |
| 1 | BM83 A2DP sink → I2S audio into S3 ring buffer |
| 2 | HTTP stream server |
| 3 | Sonos UPnP discovery + single-speaker SOAP |
| 4 | Multi-speaker grouping via coordinator pattern |
| 5 | Web UI + REST API (AsyncWebServer + SPIFFS) |
| 6 | Edge case handling (busy speaker, groups, dropouts) |
| 7 | mDNS, Home Assistant integration, v1.0 release |

See [`BUILD_PLAN.md`](./BUILD_PLAN.md) for detailed tasks and time estimates per phase.

---

## Speaker compatibility

The bridge uses **UPnP AV / DLNA** (specifically UPnP SetAVTransportURI + SOAP control) to tell speakers to pull the HTTP audio stream. This is a well-established open standard supported across the WiFi speaker market — not just Sonos.

### Confirmed compatible ecosystems

| Brand / Platform | Protocol | Multi-room sync | Notes |
|---|---|---|---|
| **Sonos** (all models) | UPnP/SOAP | ✅ Full coordinator/follower | Era, Arc, Play:1–5, Beam, Move, Roam, all older units |
| **Denon HEOS** (Home 150/250/350, HEOS 1–7) | UPnP/DLNA | ⚠️ Single-room reliable; HEOS group sync proprietary | Also built into Denon/Marantz AV receivers |
| **Yamaha MusicCast** | UPnP/DLNA | ⚠️ Single-room only | MusicCast group sync is proprietary and closed |
| **Bluesound BluOS** (Node, Pulse, etc.) | UPnP/DLNA | ⚠️ Single-room only | BluOS group sync is proprietary |
| **WiiM** (Mini, Pro, Ultra) | UPnP/DLNA | ⚠️ Single-room only | Excellent UPnP implementation, fast startup |
| **Bose SoundTouch** (legacy hardware) | UPnP + SSDP | ❌ | Cloud discontinued 2026; local LAN still works |
| **Generic DLNA** (Denon/Marantz AVRs, Onkyo, Pioneer, Sony, LG/Samsung TVs) | UPnP/DLNA | ❌ | Single-room; quality varies by firmware |

### What "works" actually means

**Single-room playback** is simple: if a device appears as a UPnP/DLNA renderer on your network (it will show up in apps like BubbleUPnP), it will work with this bridge. The bridge tells it a URL, it fetches the stream and plays it. That covers every device in the table above.

**Synchronised multi-room** is where ecosystems diverge sharply. Sonos uses a coordinator/follower model that is documented and accessible via SOAP — the bridge implements this fully. HEOS, MusicCast, and Bluesound each have proprietary group-sync protocols that are not publicly documented at the SOAP level. For those platforms, the bridge plays to multiple speakers independently, which will drift out of sync. If you need sync on non-Sonos hardware, use a single bridge → single speaker and let the platform's own app handle grouping within its ecosystem separately.

> **HEOS users:** Denon began phasing out the HEOS standalone speaker line in 2025. Existing units keep working. The HEOS stack is still present in Denon and Marantz AV receivers — these work fine with the bridge for single-room playback.

> **Bose SoundTouch users:** Bose ended cloud services for SoundTouch in 2026, making the official app non-functional. Local LAN UPnP playback still works. This bridge uses only local UPnP and is unaffected by the cloud shutdown — ironically making it one of the last practical ways to get audio into SoundTouch hardware.

> **Yamaha MusicCast users:** MusicCast is locked to Yamaha hardware only and its group-sync protocol is closed. Single-room bridge → MusicCast speaker works fine. For multi-room, use the MusicCast app to group speakers first, then point the bridge at the MusicCast group master.

### Why the README focuses on Sonos

Sonos is used as the primary example because it has the most complete and consistent UPnP/SOAP implementation in the consumer market, making the coordinator-based multi-room sync reliable and testable. The core mechanism — ESP32-S3 serves an HTTP audio stream, UPnP tells the speaker to pull it — is ecosystem-agnostic. `http_stream.cpp` and the UPnP discovery module work with any UPnP AV renderer. Only `sonos_groups.cpp` is Sonos-specific. Contributions adding HEOS or MusicCast group-sync support are welcome — see [`CONTRIBUTING.md`](./CONTRIBUTING.md).

---

## FAQ

**Why not use the classic ESP32 with its built-in Bluetooth?**
You can — and it's simpler. The classic ESP32 (WROOM-32E) has native A2DP and works with the `esp32-a2dp` library out of the box. This project targets the S3 specifically for its extra RAM and CPU headroom, which is needed to comfortably run the web UI server alongside the audio pipeline. The trade-off is adding the BM83 module.

**Why the BM83 and not the CSR8645?**
The CSR8645 is cheap and everywhere, but its firmware is essentially closed. There's no reliable way to command it over UART from a host MCU — it functions as a standalone audio endpoint, not a controllable peripheral. The BM83 has a proper UART host command set (full datasheet + SDK from Microchip) and was designed for exactly this kind of embedded integration.

**Does it work with speakers other than Sonos?**
Yes — see the [Speaker compatibility](#speaker-compatibility) section above. Any UPnP/DLNA renderer works for single-room playback. Sonos is the only platform with fully supported multi-room sync; other ecosystems (HEOS, MusicCast, Bluesound, WiiM) work in single-room mode.

**What audio quality?**
A2DP SBC at 44.1 kHz stereo. The BM83 also supports AAC if your phone negotiates it. Fine for music.

**Same WiFi network required?**
Yes. The ESP32-S3 and all Sonos speakers must be on the same LAN subnet. UPnP multicast doesn't cross subnets or VLANs.

**Can I control it from Home Assistant?**
Yes — the REST API (`/api/`) is designed for this. A HA integration guide is in `docs/HOME_ASSISTANT.md`.

---

## License

MIT — do whatever you want, attribution appreciated.
