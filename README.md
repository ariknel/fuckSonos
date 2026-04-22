# ESP32 Bluetooth → Sonos Bridge

> Stream **any** Android audio — YouTube, Instagram, radio apps, anything — to your Sonos speakers. No app required. No "Cast" button needed.

Android users shouldn't need a specific "Sonos" button in every app. This project turns a classic ESP32 into a Bluetooth A2DP receiver that re-broadcasts audio as an HTTP stream your Sonos speakers can pull from — essentially a DIY "virtual Line-In."

---

## How it works

```
Android Phone  ──[BT A2DP]──▶  ESP32 Bridge  ──[HTTP + UPnP]──▶  Sonos Speaker(s)
  any app audio                   one stream                        plays in sync
```

The ESP32 does three things at once:

1. **Acts as a Bluetooth speaker** — your Android pairs with it once, then all system audio streams over A2DP
2. **Runs a tiny HTTP stream server** — wraps incoming PCM audio into a stream Sonos can pull from a URL
3. **Sends UPnP/SOAP commands** — tells target speakers to play that URL, controls volume and grouping

---

## Board decision

> ⚠️ **Read this before buying anything.**

| Board | BT Classic (A2DP) | Dual-core | Verdict |
|---|---|---|---|
| **Classic ESP32 (WROOM-32E)** | ✅ Yes | ✅ Yes | **Use this** |
| ESP32-S3 | ❌ No (BLE only) | ✅ Yes | Won't receive A2DP without external BT chip |
| ESP32-S2 | ❌ No | ❌ No | Not suitable |

**Use the classic ESP32.** A2DP (the Bluetooth audio profile Android uses) requires Bluetooth Classic. The S3 has BLE only — it cannot receive audio from your phone without an external chip. The classic ESP32 is also dual-core (2× Xtensa LX6), so you get the same processing benefit. Use the one you have in your drawer.

---

## Bill of materials

| Component | Part | ~Cost | Notes |
|---|---|---|---|
| Microcontroller | ESP32-WROOM-32E dev board | €4–8 | Classic ESP32 only |
| Display | SSD1306 128×64 OLED (I2C) | €2–4 | `U8g2` library |
| Input | KY-040 rotary encoder | €1–2 | Navigation + volume |
| Audio out (optional) | PCM5102 I2S DAC | €3–6 | Adds real 3.5mm Line-Out jack |
| Power | USB-C breakout or 5V/1A supply | €1–3 | 500mA min |
| Enclosure | 3D printed or project box | — | STL in `/hardware` |

**Total: roughly €10–20.**

---

## Wiring

```
ESP32 pin   →   Component
─────────────────────────────────────
GPIO 21     →   OLED SDA
GPIO 22     →   OLED SCL
GPIO 18     →   Encoder CLK
GPIO 19     →   Encoder DT
GPIO 5      →   Encoder SW (push button)

# Optional I2S DAC (PCM5102)
GPIO 25     →   DAC BCK
GPIO 26     →   DAC LCK
GPIO 22     →   DAC DIN
3.3V / GND  →   DAC power
```

---

## Software architecture

```
firmware/
├── src/
│   ├── main.cpp              # Setup, core pinning, top-level state machine
│   ├── bt_sink.cpp           # A2DP sink (esp32-a2dp lib)
│   ├── audio_buffer.cpp      # Lock-free PCM ring buffer
│   ├── http_stream.cpp       # HTTP audio stream server
│   ├── sonos_upnp.cpp        # UPnP discovery + SOAP control
│   ├── sonos_groups.cpp      # Speaker selection + group management
│   ├── oled_ui.cpp           # U8g2 display + encoder handler
│   ├── nvm_store.cpp         # NVS persistence (pairing, last config)
│   └── config.h              # WiFi creds, stream port, etc.
├── platformio.ini
└── README.md
```

### Core pinning

```
Core 0  →  bt_sink task      (A2DP callbacks, I2S write, ring buffer)
Core 1  →  http_stream       (serve audio chunks over HTTP)
           sonos_upnp         (UPnP discovery, SOAP commands)
           sonos_groups       (group state machine)
           oled_ui            (display refresh, encoder polling)
```

### Key libraries

```ini
# platformio.ini
lib_deps =
    pschatzmann/ESP32-A2DP
    me-no-dev/ESPAsyncWebServer
    me-no-dev/AsyncTCP
    olikraus/U8g2
```

---

## User experience

This is the most important section. The Sonos app sets a high bar for speaker management UX — the bridge should feel like a natural extension of that, not a hacker tool you need to fight.

### First boot — setup flow

The goal is zero configuration beyond WiFi. WiFi credentials are the only thing you enter manually (via `config.h` before flashing, or a captive portal on first boot).

```
Power on
│
▼
OLED: "SonosBridge" + version
│
▼
Connect to WiFi
│
OLED: "Scanning for speakers..."
│
▼
UPnP M-SEARCH discovers all Sonos speakers on the LAN
│
OLED: "Pair your phone"  ← Bluetooth name "SonosBridge"
│
▼
Phone pairs over BT (saved permanently in NVS)
│
▼
OLED: speaker selection screen
```

After first boot, the bridge remembers both the phone pairing and the last-used speaker configuration. Subsequent boots reach the ready state in a few seconds.

---

### Speaker selection

This is the core UX interaction. Sonos rooms are named by the user ("Living Room", "Kitchen") — the bridge shows those exact names, not IP addresses, pulled from the UPnP description XML.

The encoder knob drives all navigation. **Rotate** to scroll. **Short press** to toggle a speaker on or off. **Hold** to confirm and start streaming.

```
┌─────────────────────────────┐
│  SELECT SPEAKERS            │
│                             │
│  ▸ Living Room       [ ]    │
│    Kitchen           [ ]    │
│    Bedroom           [ ]    │
│    Office            [ ]    │
│                             │
│  [hold to stream]           │
└─────────────────────────────┘
```

As you toggle speakers, selected ones fill their marker:

```
┌─────────────────────────────┐
│  SELECT SPEAKERS            │
│                             │
│  ▸ Living Room       [✓]    │
│    Kitchen           [✓]    │
│    Bedroom           [ ]    │
│    Office            [ ]    │
│                             │
│  2 selected  [hold →]       │
└─────────────────────────────┘
```

Hold confirms and starts streaming to the selected set.

---

### Single speaker vs grouped streaming

**Single speaker selected:** The bridge sends `SetAVTransportURI` + `Play` directly to that speaker. Simple, lowest latency.

**Multiple speakers selected:** This is where naive implementations fail. If you independently tell each speaker to pull `http://[esp-ip]:8080/stream`, they will drift out of sync within seconds — each Sonos speaker buffers differently. The result is an audible echo that gets worse over time.

The correct approach uses Sonos's own coordinator architecture:

```
Selected: Living Room + Kitchen
│
▼
1. Designate the first selected speaker as "coordinator"
2. Send SetAVTransportURI + Play to the coordinator only
3. Send the group-join SOAP command to each follower,
   pointing them at the coordinator's session — not at
   the ESP32 stream directly
4. Sonos handles hardware-level sync between them internally
```

Once a follower has joined a coordinator's session, Sonos synchronises audio between them at the firmware level over the LAN. The bridge just sets up the relationship — it doesn't need to solve sync itself. This is the same mechanism the Sonos app uses.

The coordinator/follower pattern means the ESP32 stream is pulled once, by the coordinator, and then distributed by Sonos to the followers in perfect sync.

---

### The streaming screen

Once audio is flowing:

```
┌─────────────────────────────┐
│  ♫  STREAMING               │
│                             │
│  Living Room                │
│  Kitchen                    │
│                             │
│  ████████░░░  Vol 75%       │
│                             │
│  BT●  WiFi●  Sync●          │
└─────────────────────────────┘
```

Three status dots at the bottom show Bluetooth connected, WiFi connected, and Sonos sync confirmed (the bridge polls `GetTransportInfo` every 30 seconds and verifies the coordinator is in `PLAYING` state).

Encoder rotation adjusts group volume. Short press enters per-speaker volume control. Long press stops playback and returns to speaker select.

---

### Per-speaker volume

Hold the button from the streaming screen to enter the per-speaker volume sub-menu:

```
┌─────────────────────────────┐
│  VOLUME                     │
│                             │
│  Living Room  ████████  80% │
│▸ Kitchen      ██████░░  60% │
│                             │
│  [press = back]             │
└─────────────────────────────┘
```

Each line sends `SetVolume` via the RenderingControl SOAP service to that speaker's IP independently. Rotate to change volume on the highlighted speaker, short press moves focus to the next speaker, long press goes back.

---

## Known UX pitfalls — and how the bridge handles them

These are real problems the Sonos app itself struggles with. The bridge should handle each one gracefully rather than silently failing.

### Speaker not found on initial scan

Sonos speakers sometimes take 10–15 seconds to respond to UPnP M-SEARCH, especially after a router restart or if they've been in standby for a while.

**Bridge behaviour:** Show "Scanning..." with an animated progress indicator. Retry M-SEARCH three times with 5-second gaps before showing "No speakers found — retry?" Don't show an empty list and leave the user guessing.

### Speaker is already playing something

Sending `SetAVTransportURI` to a speaker currently playing Spotify or a radio station will interrupt it without warning. The Sonos app asks for confirmation — the bridge should too.

**Bridge behaviour:** Before committing to a speaker, poll `GetTransportInfo`. If the speaker is in `PLAYING` state, show a warning screen: `"Living Room is busy. Replace? [Y/N]"` The encoder confirms or cancels. Silently hijacking playback is bad UX.

### Speaker is in an existing Sonos group

A speaker that's already a follower in a Sonos group (set up via the Sonos app) will ignore direct play commands — only coordinators accept them. This is one of the most confusing failure modes.

**Bridge behaviour:** After discovery, poll `GetGroupAttributes` on each speaker. If a speaker is a follower, mark it in the list: `"Kitchen  [in group]"`. If the user selects it, show: `"Kitchen is in a group. Break it? [Y/N]"` Selecting yes sends `BecomeCoordinatorOfStandaloneGroup` to remove it from its current group before adding it to the bridge's group.

### WiFi dropout mid-stream

If the ESP32 loses WiFi, the Sonos speaker keeps playing for a few seconds (burning its buffer), then stops. The bridge should reconnect and resume automatically.

**Bridge behaviour:** WiFi watchdog on Core 1 checks connectivity every 10 seconds. On reconnect, re-send `SetAVTransportURI` + `Play` to the last coordinator. Display shows "Reconnecting..." during the gap. No user action required for brief dropouts (under ~30 seconds). After a longer outage, the speaker will have dropped the stream — the bridge re-establishes the whole session.

### Bluetooth phone disconnects

When the phone goes out of range or Bluetooth is toggled off, the A2DP stream drops. The Sonos speaker will play silence briefly, then stop as the HTTP stream goes quiet.

**Bridge behaviour:** On A2DP disconnect, immediately send `Stop` to all active speakers. Show "Phone disconnected" on OLED. When the phone reconnects, auto-resume to the same speaker configuration — no need to re-select.

### Volume state conflict with the Sonos app

If someone adjusts volume in the Sonos app while the bridge is also managing volume, the bridge's cached volume value will be wrong and applying a delta will jump to the wrong level.

**Bridge behaviour:** Never cache volume internally. Always call `GetVolume` immediately before `SetVolume`. Encoder rotation applies a delta to the live hardware value, not a stale local one.

### Speaker name shows as an IP address

If UPnP description XML doesn't return a friendly name (rare, but happens with very old Sonos firmware), falling back to the IP address is confusing.

**Bridge behaviour:** Fall back to a human-readable label: `"Speaker @ .1.42"` (last two octets only). Never show a raw IP as a primary label.

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
git clone https://github.com/yourname/esp32-sonos-bridge
cd esp32-sonos-bridge

# Configure
cp firmware/src/config.h.example firmware/src/config.h
nano firmware/src/config.h   # add WiFi SSID + password

# Flash
pio run --target upload --environment esp32dev

# Monitor
pio device monitor
```

On first boot: pair your Android phone to "SonosBridge" in Bluetooth settings. That's it.

---

## Milestones

| Phase | Goal |
|---|---|
| 0 | Toolchain, repo structure |
| 1 | BT A2DP sink → audio out |
| 2 | HTTP stream server |
| 3 | Sonos UPnP discovery + single-speaker SOAP |
| 4 | Multi-speaker grouping via coordinator pattern |
| 5 | OLED UI + encoder navigation |
| 6 | Edge case handling (busy speaker, groups, dropouts) |
| 7 | Enclosure + v1.0 release |

See [`BUILD_PLAN.md`](./BUILD_PLAN.md) for detailed tasks and time estimates per phase.

---

## FAQ

**Can I use the ESP32-S3?**
Not without adding an external Bluetooth Classic module. The S3 has BLE only — it cannot receive A2DP audio from an Android phone. Use the classic ESP32.

**Does it work with all Sonos speakers?**
Yes — all Sonos speakers support UPnP/SOAP. This includes Era, Five, Arc, Play:1, Play:3, Play:5, Beam, Move, Roam, and older units. The Move and Roam have built-in Bluetooth too, but this bridge lets you group them with fixed speakers.

**What audio quality?**
A2DP SBC at 44.1 kHz stereo, up to 328 kbps. Fine for music. The `esp32-a2dp` library also supports AAC if your phone negotiates it.

**Will it interrupt my existing Sonos setup?**
Only when you actively stream to a speaker. When idle the bridge does nothing. When streaming, it interrupts whatever the target speaker is currently playing — same as any Sonos source. The bridge warns you before doing this.

**Same WiFi network required?**
Yes. The ESP32 and all Sonos speakers must be on the same LAN subnet. UPnP multicast doesn't cross subnets or VLANs.

---

## License

MIT — do whatever you want, attribution appreciated.
