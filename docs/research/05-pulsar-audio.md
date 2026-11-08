# Thin-terminal options for Mini 3 (bergamot): Pulsar + raw audio (2026-11-08)

Research agent output. VERIFIED = source read; ASSUMED = inference.

## 1. Pulsar

**Upstream.** `Morvex885/pulsar` is the origin (created 2026-10-21, last push 2026-11-04, **AGPL-3.0**). `MyrikLD/pulsar` is a fork (2026-10-31). PR #30 and issue #50 live on Morvex885. VERIFIED.

**Language / size / install.** Daemon in Rust, installer in Go. Release v0.6.0 (2026-11-04): `pulsar-armhf.bin` ≈4.2 MB, `pulsar-aarch64.bin` 4.5 MB. Installer copies to `/data/pulsar`, writes `/etc/init.d/S99zpulsar` (needs `mount -o remount,rw /`), uses adb; `--ssh` installs dropbear in `/data/dropbear` on port 22 root/empty password. Uninstall: `/data/pulsar/ctl.sh disable; rm -rf /data/pulsar`, remove `S99zpulsar`. VERIFIED (README, releases).

**Mic capture.** Does **not** read `vqe_0` from the stock daemon; stops the stock stack and re-implements the pipeline: opens ALSA `hw:0,1` directly (Mini 3 profile: 48 kHz, 8 ch S16, `mic_channels [0,1,2]`, `spk_channels [6]`), feeds Yandex VQE via `dlopen` of `/system/vendor/quasar/libs/libyandex_evil.so` (preset `bergamot`, 768 frames in → 256 samples @16 kHz out, 16 ms), then runs stock spotter models from `/system/vendor/quasar/spotter_models` via the same library. Signatures reverse-engineered from `maind`. VERIFIED (`model.rs`, `frontend.rs`, `engine/ffi.rs`).

**Coexistence with aliced.** None: `ctl.sh enable` runs `/system/vendor/quasar/q stop`; `boot.sh` waits for `quasar_launcher2` then stops it. **`ctl.sh disable` restores Yandex (`q start`)** — reboot-free toggle. adbd/dropbear untouched. VERIFIED.

**Wake word.** On-device, Yandex native «Алиса»/«Яндекс» nnet models, plus local command spotter («стоп», «громче»…). 1.024 s pre-roll ring; Mini 3 pre-roll 6 chunks (alisa) / 8 (yandex), contributor measurements suggest the pre-roll value is a guess and may leak the end of the wake word into STT. After wake, 16 kHz mono PCM streamed to the client. VERIFIED.

**Protocol.** ESPHome Native API, TCP 6053, plaintext or Noise (key auto-provisioned by HA on first connect, stored `/data/pulsar/api_key`; manual via `ctl.sh api-key <base64>`). `VoiceAssistantFeature` = `VOICE_ASSISTANT | API_AUDIO | TIMERS | ANNOUNCE | START_CONVERSATION`; mic audio goes **over the API socket** (`VoiceAssistantAudio`), not UDP; `SPEAKER` not advertised, inbound audio ignored. Also mDNS, UDP 6054 wake-arbitration, Sendspin 8928, update check against GitHub Releases (auto-update off by default). VERIFIED.

**Playback.** TTS arrives as a URL in `TTS_START`/`TTS_END` event data; device fetches over HTTP and plays through the firmware's GStreamer (`Channel::Announce`) → ALSA `speakers`/dmix. Earcons MP3/WAV; TTS from HA is FLAC. VERIFIED (`voice.rs`, docs/mini3.md).

**Server side without HA.** ESPHome model: device = server, consumer = client. Plain Python `aioesphomeapi` (46.28.0) can be the "HA": `subscribe_voice_assistant(handle_start, handle_stop, handle_audio=...)` sets `API_AUDIO` and delivers 16 kHz PCM chunks; `send_voice_assistant_event(VoiceAssistantEventType.*, {"url": ...})` drives the pipeline; `send_voice_assistant_announcement_await_response(media_id, ...)` plays a URL. Pulsar logs "wake ignored: Home Assistant voice assistant not subscribed" if nobody is subscribed. Caveat: Pulsar's state machine mirrors HA quirks (`docs/voice.md`: expects RUN_START…TTS_END order, owes `AnnounceFinished`, 15 s STT watchdog). VERIFIED that API calls exist; **ASSUMED** Pulsar tolerates a minimal custom event sequence (untested by anyone found).

**Mini 3 status (PR #30, merged 2026-10-30; FW `0.369.1.50.4794652521.20261021.117`).** Owner confirmed full loop «Алиса» → HA Assist → STT/agent → TTS; capture 0 dropped over 65 s; Music Assistant FLAC 60 s 0 xruns; Noise key auto-issued; SSH/ADB preserved; raw `hw:0,1` channel map verified (0/1/2 mics, 6 speaker reference, 7 duplicates 6). **Not verified:** AEC quality, physical buttons/mute switch, ring zones, end-to-end `pulsar-install` and reboot autostart, long-running stability. PR disclaimer: "навайбкожено с помощью ИИ". VERIFIED.

**Telemetry/OTA.** Speaker still uses firmware DNS (`unbound`) + NTP pointed at Yandex and stays bound to the Yandex account; Pulsar neither blocks nor tracks OTA — an OTA can wipe `S99zpulsar` and the root method. With `maind` stopped, telemetry/fluent-bit/updatesd roles aren't running (ASSUMED consequence of `q stop`). VERIFIED (README "Ограничения и риски").

## 2. Raw audio on Mini 3 (hackalice.me)

- Card 0 `AML-AUGESOUND`: `hw:0,0` = `TDM-B-acm8625` playback; `hw:0,1` = `PDM-dummy` capture, 8 ch S16 @48 kHz. 3 PDM mics ch 0–2; ch 6 = HW feedback AEC reference. `pcm.!default` → `speakers` → `plug` → `dmix_speakers` (`ipc_key 1024`, `hw:0,0`, 2 ch 48 kHz). Volume: `amixer set Master N%`. VERIFIED.
- `vqe_0` is **internal** to `maind --platformd`, not an ALSA device. Only published tap: Lite 2 `LD_PRELOAD` `vqe_tap.c` recipe; Mini 3 "not verified". VERIFIED.
- Direct raw capture while Alice runs: `hw:0,1` held by `maind` (ASSUMED). `q stop` first, then `arecord -D hw:0,1 -f S16_LE -c 8 -r 48000`. Pulsar's `frontend_probe 20 --record /tmp/raw.wav 10` is a tested raw dump on Mini 3. VERIFIED (PR #30).
- Playback with Alice running works: `aplay /data/sounds/my.wav` listed as working; `default` is dmix. VERIFIED.
- Stop/restart Alice: `/system/vendor/quasar/q stop|start|restart` (wrapper over `quasar_launcher2 /system/vendor/quasar/launch.json`, started by `S99quasar`). Roles: `--platformd`, `--mediad`, `--telemetry`, `--fluent-bitd`, `--updatesd`. VERIFIED.
- SSH: hackalice recipe dropbear on 2222; Pulsar's own dropbear on 22. VERIFIED.

**Self-hosted uniproxy.** `v3rt3p/quasar-server` (TS/Docker, GPLv3) and `amirilivice/quasard` (Python, GPU ~8 GB). Both keep the whole stock stack and replace only the cloud. uniproxy has no cert pinning so `uniProxyUrl` in `quasar.cfg` can be `ws://`; Mini 3 listed on neither; bergamot FW 0.369 uses protobuf Alice API + zstd. Telemetry keeps running. Mini 3 compatibility ASSUMED unknown.

## 3. wyoming-satellite on this device class

No report of wyoming-satellite on any Yandex station (VERIFIED absence). wyoming-satellite deprecated → linux-voice-assistant (Python 3.11+, 512 MB). No Python on firmware (ASSUMED). Go alternatives are toys. Pulsar ≈ "linux-voice-assistant in Rust for Yandex hardware", reusing Yandex VQE + wake-word models — decisive advantage.

## 4. Ranking for "wake word → ask weather → hear answer"

| Option | On device | Talks to Yandex | Latency | Fragility | Effort |
|---|---|---|---|---|---|
| **(a) Pulsar + Python aioesphomeapi consumer, no HA** | Pulsar (VQE + wake word + GStreamer); `q stop` | Only DNS/NTP + account binding | Low | Medium: Mini 3 experimental, pre-roll untuned, must emulate HA event sequence | Low-medium: copy binary + scripts to `/data/pulsar`, write `S99zpulsar`; ~150-line Python |
| (b) Pulsar + HA + Assist pipeline | same | same | Low-medium | Low (maintainer-tested path) | Medium: HA is a big dependency |
| (c) Self-hosted uniproxy | entire stock stack | telemetry/OTA still run unless blocked | Low | High for Mini 3 (protocol unmatched) | High |
| (d) raw `arecord`/`aplay` over SSH | ALSA only; `q stop` to free `hw:0,1` | none while stopped | Medium-high (768 kB/s raw, own AEC/wake on Mac) | Medium | Low to prototype, high to make usable |

**Recommendation: (a).** Pulsar solves VQE/AEC, on-device wake word, playback mixing on exactly this firmware build; ESPHome API consumable by `aioesphomeapi` without HA. Alice fallback via `ctl.sh disable`. Risks: single-unit verification; pre-roll tuning; OTA can delete `S99zpulsar`; HA-event choreography may need log-driven iteration (`ctl.sh log 200`). (b) is documented fallback with identical device setup. (d) only as diagnostic; (c) only if Alice's own voice/music needed.

## Sources

https://github.com/Morvex885/pulsar (README, `docs/mini3.md`, `docs/voice.md`, `docs/architecture.md`, `satellite/deploy/ctl.sh`, `boot.sh`, `crates/pulsar-core/src/model.rs`, `pulsar-audio/src/frontend.rs`, `engine/ffi.rs`, `pulsar/src/controller/voice.rs`, `pulsar-esphome/src/server/{entities,commands}.rs`, releases v0.6.0); https://github.com/Morvex885/pulsar/pull/30 ; https://github.com/Morvex885/pulsar/issues/50 ; https://github.com/MyrikLD/pulsar ; https://hackalice.me/devices/station-mini3/ , /internals/voice/ , /internals/architecture/ , /internals/telemetry/ , /recipes/ (microphone, quasard, bekend-self-hosted, v3rt3p, home-assistant, llm, sounds, standalone); https://github.com/v3rt3p/quasar-server ; aioesphomeapi `client.py`/`model.py` ; https://pypi.org/pypi/aioesphomeapi/json ; https://github.com/rhasspy/wyoming-satellite ; https://github.com/OHF-Voice/linux-voice-assistant ; https://github.com/m50/wygoming-satellite ; https://go2rtc.org/internal/wyoming/
