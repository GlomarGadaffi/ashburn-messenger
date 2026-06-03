# pocketdial-voice-poc

A **spike** to prove the missing half of [ESPHome feature request #3112](https://github.com/esphome/feature-requests/issues/3112):
a SIP **endpoint with real audio media** on an ESP32-S3 — mic → SIP, SIP → speaker.

> **Status: dev spike, not a product.** Built to validate (or kill) the "all-in-one
> ESP32 SIP voice" idea before committing to a repo/license structure. Target board
> is the only integrated dev board on hand; all board specifics live in one header.

## Why this exists

- **pocket-dial** (MIT) is a great SIP *registrar/proxy* on ESP32 — but it deliberately
  never touches audio (RTP flows peer-to-peer).
- **chrta/sip_call** (AGPL) initiates SIP calls on a GPIO trigger — but has no media path.
- **#3112** is fundamentally an **audio** problem, present in neither. This spike builds
  exactly that gap, reusing pocket-dial's MIT SIP parser for the fiddly bits.

## Target hardware

**LilyGO T3-S3 MVSRBoard** (ESP32-S3, 4 MB flash) — board rev **V1.1**:
- Mic: **MP34DT05-A** PDM MEMS → **I2S0** PDM-RX (CLK 15, DATA 48, EN 35)
- Amp: **MAX98357A** I2S class-D → **I2S1** std-TX (BCLK 40, LRCLK 41, DATA 39, SD 38)
- Vibration motor (PWM, GPIO46) — simulates the doorbell "actuation"
- PTT = BOOT button (GPIO0)

> ESP32-S3 PDM RX only works on I2S0, so the mic owns I2S0 and the speaker moves to
> I2S1. The S3 hardware PDM→PCM filter yields 16-bit PCM; we capture at 16 kHz and
> decimate 2:1 to the 8 kHz G.711 rate. (V1.0 boards have an MSM261 *I2S* mic on
> BCLK 47/WS 15/DATA 48 needing `i2s_channel_init_std_mode` — see `board_mvsr.h`.)

## The design (deliberately half-duplex)

Push-to-talk gates direction, so mic and speaker are never live at once →
**no acoustic echo cancellation needed**, which removes the single hardest part of
ESP32 two-way voice. AEC is a later upgrade, not a blocker.

```
 BOOT held : mic --I2S--> [G.711 µ-law] --> [RTP] --UDP--> ext 102 phone (P2P)
 BOOT up   : ext 102 phone --UDP--> [RTP] --> [G.711] --I2S--> MAX98357A speaker
```

Codec: G.711 µ-law (PCMU, PT 0), 8 kHz mono, 20 ms / 160-sample frames.
Signaling: `REGISTER` ext **500** (no auth — pocket-dial is an open registrar) →
`INVITE` ext **102** *through* the server → `200 OK → ACK` → `BYE`. The pocket-dial
server proxies signaling only; the `200 OK` carries ext 102's own SDP, so **RTP audio
flows peer-to-peer** straight between the board and the phone.

## Build / flash / test

1. **Edit `main/poc_config.h`** — set `POC_WIFI_SSID`/`POC_WIFI_PASS`. Defaults already
   target the pocket-dial server at `192.168.12.2:5060`, register as ext `500`, call ext `102`.
2. **Have ext 102 registered** to the same pocket-dial server (a softphone or IP phone)
   and ready to answer.
3. Build & flash with ESP-IDF v5.1+ / v6.x:
   ```
   idf.py set-target esp32s3
   idf.py -p <PORT> flash monitor
   ```

### Definition of done (the credibility gate)

> Power on → board joins Wi-Fi → **REGISTERs as ext 500** with the pocket-dial server →
> motor buzzes → **INVITEs ext 102 through the server** → 102 answers → **hold BOOT and
> talk: you're heard on 102's phone; release BOOT: 102's audio plays out the board
> speaker.** Intelligible both ways, with RTP flowing board↔102 directly (P2P).

If that works, the "all-in-one" idea is real and worth a proper repo. If the audio is
garbage, we learn the hard truth cheaply.

## What's real vs. stubbed

| Part | State |
|---|---|
| Board pin map, dual-I2S bring-up | ✅ real |
| G.711 µ-law encode/decode | ✅ real (CCITT reference) |
| RTP framing (TX) | ✅ real |
| REGISTER (ext 500, no auth) + INVITE/ACK/BYE via server | ✅ real (parser vendored from pocket-dial) |
| Push-to-talk half-duplex media loop | ✅ real |
| **Jitter buffer** | ⚠️ none — direct play-out (fine on a quiet LAN; add later) |
| Mic capture (PDM → PCM) | ✅ real — PDM2PCM @ 16 kHz, 2:1 decimate to 8 kHz (V1.1 MP34DT05) |
| **Digest auth (MD5)** | ❌ not needed — pocket-dial is an open registrar |
| **Acoustic echo cancellation** | ❌ intentionally avoided via PTT |
| **Incoming calls / full duplex** | ❌ later milestone |

## License & attribution

**MIT** — see [LICENSE](LICENSE). Dual copyright:

- **Original work** (audio pipeline, G.711, RTP, the `sip_uac` SIP client, Wi-Fi/board
  integration, application code) — © 2026 GlomarGadaffi.
- **Vendored SIP message parser** in [`components/sip_core/`](components/sip_core/)
  (`SipMessage`, `SipSdpMessage`, `SipMessageFactory`, `SipMessageTypes`, `IDGen`) —
  © 2022 BarGabriel, derived via [pocket-dial](https://github.com/GlomarGadaffi/pocket-dial)
  from **[BarGabriel/SipServer](https://github.com/BarGabriel/SipServer)** (MIT).

Also referenced (no license obligation, credited anyway):
- **G.711 µ-law** — the canonical CCITT / Sun reference companding algorithm.
- **Pin map** — from LilyGO's [T3-S3-MVSRBoard](https://github.com/Xinyuan-LilyGO/T3-S3-MVSRBoard) `pin_config.h`.

Fully permissive: built on MIT code only — **no copyleft dependencies** (notably *not* the
AGPL `chrta/sip_call`).
