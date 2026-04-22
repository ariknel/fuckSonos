# BUILD_PLAN.md — ESP32 Bluetooth → Sonos Bridge

Phased build plan. Each phase produces a testable milestone before moving on.  
Estimated total build time: **4–8 weekends** depending on prior ESP32 experience.

---

## Phase 0 — Repo & toolchain setup
**Goal:** Everything compiles and flashes on a blank ESP32.  
**Time estimate:** 1–2 hours

### Tasks
- [ ] Install PlatformIO (VS Code extension or CLI)
- [ ] Create `platformio.ini` with `board = esp32dev`, `framework = arduino`
- [ ] Add all library dependencies to `lib_deps`
- [ ] Flash a "Hello World" serial sketch, confirm monitor output
- [ ] Set up repo structure (`firmware/src/`, `hardware/`, `docs/`)
- [ ] Add `config.h.example` with WiFi and stream port placeholders
- [ ] Create `.gitignore` (exclude `config.h`, `.pio/build/`)

### Done when
`pio run` succeeds, board shows up on serial monitor.

---

## Phase 1 — Bluetooth A2DP sink → audio output
**Goal:** Android phone pairs with the ESP32 and you can hear audio through a speaker/DAC.  
**Time estimate:** 1 weekend

### Tasks
- [ ] Add `pschatzmann/ESP32-A2DP` library
- [ ] Implement `bt_sink.cpp`: init A2DP sink with device name `"SonosBridge"`
- [ ] Register `avrc_metadata_callback` (optional: capture track title for OLED)
- [ ] Register `audio_data_callback` — write PCM chunks to I2S or ring buffer
- [ ] Test with I2S DAC (PCM5102) or built-in DAC on GPIO25/26
- [ ] Confirm audio plays end-to-end (phone → ESP32 → headphones/speaker)
- [ ] Pin BT task to Core 0 explicitly: `xTaskCreatePinnedToCore(..., 0)`
- [ ] Tune I2S buffer size: start at 512 samples, reduce until no glitches

### Done when
YouTube audio from Android plays through the ESP32 with no stuttering.

### Known pitfall
`esp32-a2dp` and WiFi both use the RF radio. They share antenna time but can coexist — keep BT on Core 0, WiFi on Core 1. Do not enable WiFi until Phase 2.

---

## Phase 2 — HTTP audio stream server
**Goal:** The ESP32 serves a live MP3/PCM HTTP stream that any browser (or VLC) can play.  
**Time estimate:** 1 weekend

### Tasks
- [ ] Connect to WiFi in `setup()` — print IP to serial
- [ ] Implement lock-free ring buffer in `audio_buffer.cpp`
  - Writer: A2DP callback (Core 0)
  - Reader: HTTP response chunked sender (Core 1)
- [ ] Add `ESPAsyncWebServer` + `AsyncTCP`
- [ ] Implement `GET /stream` endpoint:
  - Content-Type: `audio/mpeg` or `audio/wav`
  - Chunked transfer encoding
  - Read from ring buffer, send as fast as Sonos pulls
- [ ] Test in VLC: `http://[esp-ip]:8080/stream` — audio should play
- [ ] Handle client disconnect gracefully (don't crash if Sonos drops)

### Done when
VLC on a PC plays the Android audio stream from the ESP32 URL with acceptable latency.

### Ring buffer sizing guide
```
BT A2DP @ 44.1kHz stereo SBC ≈ 172 KB/s raw PCM
Target buffer hold: ~500ms → 86 KB
Use: 96 KB ring buffer (power of 2 preferred)
```

---

## Phase 3 — Sonos UPnP discovery & SOAP control
**Goal:** The ESP32 finds Sonos speakers on the LAN and sends them the play command automatically.  
**Time estimate:** 1–2 weekends (trickiest phase)

### Tasks
- [ ] Implement UPnP M-SEARCH in `sonos_upnp.cpp`
  - Send `M-SEARCH * HTTP/1.1` to `239.255.255.250:1900`
  - Parse responses for `urn:schemas-upnp-org:device:ZonePlayer:1`
- [ ] Extract speaker name, IP, and AVTransport endpoint from description XML
- [ ] Store discovered speakers list (vector of structs)
- [ ] Implement SOAP `SetAVTransportURI`:
  ```xml
  CurrentURI = http://[esp-ip]:8080/stream
  CurrentURIMetaData = (DIDL-Lite XML, see below)
  ```
- [ ] Implement SOAP `Play` action (TransportState = PLAYING)
- [ ] Implement SOAP `Stop` action
- [ ] Implement SOAP `SetVolume` via RenderingControl service

### DIDL-Lite metadata template
```xml
<DIDL-Lite xmlns="urn:schemas-upnp-org:metadata-1-0/DIDL-Lite/">
  <item id="1" parentID="0" restricted="1">
    <dc:title>Bluetooth Audio</dc:title>
    <res protocolInfo="http-get:*:audio/mpeg:*">
      http://[esp-ip]:8080/stream
    </res>
  </item>
</DIDL-Lite>
```

### Done when
Calling `sonos.play("192.168.1.x")` from serial monitor starts audio on the Sonos speaker.

### Debugging tip
Wireshark on your PC with filter `udp.port == 1900` will show UPnP discovery traffic. Use this to verify M-SEARCH and responses before writing parsing code.

---

## Phase 4 — OLED UI & rotary encoder
**Goal:** The device is fully standalone — no serial monitor needed.  
**Time estimate:** 1 weekend

### Tasks
- [ ] Wire SSD1306 to I2C (GPIO 21 SDA, 22 SCL)
- [ ] Init `U8g2` library, confirm display shows text
- [ ] Wire KY-040 rotary encoder (CLK, DT, SW)
- [ ] Implement encoder interrupt handler: track position delta and button press
- [ ] Design UI state machine:

```
State: PAIRING
  Display: "Waiting for phone..."
  Action: none, wait for A2DP connection

State: SELECTING
  Display: scrollable list of discovered Sonos speakers
  Encoder: scroll up/down, press to select

State: STREAMING
  Display: speaker name + volume bar + connection indicator
  Encoder: rotate = volume up/down, press = stop/start

State: ERROR
  Display: error message
  Action: auto-retry after 5s
```

- [ ] Implement `oled_ui.cpp` with `U8g2` draw calls for each state
- [ ] Integrate encoder events into state transitions
- [ ] Volume changes call `sonos.setVolume()` via UPnP

### Done when
The device operates fully without a PC connected — pair phone, select speaker, music plays, encoder controls volume.

---

## Phase 5 — Integration, stability & latency tuning
**Goal:** Long-running stability, no crashes after 1+ hours.  
**Time estimate:** 1 weekend + ongoing

### Tasks
- [ ] Run 2-hour soak test: stream music continuously, check for heap exhaustion
  - Monitor: `ESP.getFreeHeap()` printed every 30s
- [ ] Test WiFi reconnect: pull router power briefly, confirm ESP32 recovers
- [ ] Test BT reconnect: forget device on phone, re-pair, confirm stream resumes
- [ ] Tune stream buffer for latency:
  - Reduce ring buffer → lower latency, higher risk of underruns
  - Increase buffer → higher latency, more stable
  - Target: lowest buffer size with zero audible glitches on typical songs
- [ ] Implement Sonos watchdog: poll `GetTransportInfo` every 30s, re-issue play if stopped
- [ ] Store last-used Sonos speaker IP in NVS (survives power cycle)
- [ ] Store BT pairing in NVS (already handled by ESP-IDF BT stack automatically)

### Latency measurement method
Play a YouTube video showing a metronome or clapping. Observe the audio delay vs the visuals. Note the seconds — that's your total latency budget.

---

## Phase 6 — Enclosure & polish
**Goal:** A clean physical product you'd put on a shelf.  
**Time estimate:** 1 weekend

### Tasks
- [ ] Design or find enclosure (project box ~100×60×30mm works well)
- [ ] CAD cutouts for: USB power port, OLED window, encoder knob, optional 3.5mm jack
- [ ] Optional: 3D print enclosure (STL files in `/hardware/enclosure/`)
- [ ] Solder headers or direct-wire components onto perfboard
- [ ] Add status LED: solid = streaming, blinking = waiting, off = no power
- [ ] Write final `README.md` with photos, wiring diagram image, flash instructions
- [ ] Tag `v1.0.0` release on GitHub

---

## Not in scope (possible v2 features)

- **Multi-room sync** — grouping Sonos speakers via UPnP Group Management
- **AAC / aptX codec** — richer audio quality if phone supports it
- **Web config UI** — browser-based WiFi setup instead of editing `config.h`
- **OTA updates** — push firmware updates over WiFi
- **Home Assistant integration** — MQTT status/control

---

## Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| BT + WiFi RF contention causes dropouts | Medium | High | Pin tasks to separate cores; tune antenna timing in `menuconfig` |
| ESP32 heap exhaustion after hours | Medium | High | Soak test Phase 5; use static buffers not malloc in hot paths |
| Sonos firmware update breaks UPnP API | Low | High | Pin to known-good SOAP endpoint; add retry logic |
| Latency unacceptable for user's use case | High (for video) | Medium | Document clearly; this is a music-only solution |
| Ring buffer overrun causes audio artifacts | Medium | Low | Use `portMUX_TYPE` spinlock or lock-free index pair |
