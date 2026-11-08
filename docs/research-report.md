# Research report: Yandex Station Mini 3 → self-hosted Claude assistant

Date: 2026-11-08. Status: Phase 1 complete, no device touched.
Detailed per-topic findings with sources: `docs/research/01-mini3-root.md`, `02-ios-bridge.md`, `03-claude-code-control.md`, `04-server-stack.md`, `05-pulsar-audio.md`.
Attribution: device facts derive from the *Yandex Station Extended* wiki (hackalice.me, `Marker284/yandex-station-extended`), CC BY 4.0, and from the Pulsar repo (AGPL-3.0). Changes/condensation are ours.

Legend: **V** = verified against a primary source on 2026-11-08; **A** = assumed/inferred, must be checked on the device or in a test.

---

## 1. Firmware and root status (Phase 1a)

| Claim | Status |
|---|---|
| Current retail Mini 3 build is `0.369.1.50.<serial>.20261021.117` ("`.50`"), pushed by the forced OTA on 21 Oct 2026 | V |
| The `.50` OTA removed the AAA Certificate Services root from the device CA bundle and closed ADB. Old DNS-only root is dead | V |
| Root on Mini 3 still works on `.50` via **"retail CA + markers from the factory image"**: boot factory/recovery image (hold mic-off at power-on), get root ADB through a custom HTTPS update server + DNS redirect, mount retail `system`, re-append the AAA root to `ca-certificates.crt`, drop ADB marker files in `/data`, set Wi-Fi persistence, reboot | V |
| **Hard precondition:** factory partition must still show a `factory-N` build. If factory mode shows `0.369.1.50…`, the method fails. This is the first thing to check and the abort rule | V |
| Our unit's factory partition is still `factory-N` | **A — unknown until checked** |
| A button factory reset wipes `/data` (markers, SSH, Wi-Fi) but does not reflash `system`; the CA patch survives a reset | V |
| Update blocking: `yandex-noupdate.sh` replaces the OTA signing pubkey, stubs `device_ota_update.sh`, removes `--updatesd` from `launch.json`; reversible with `restore` | V |
| Persistence: dropbear SSH on 2222 via `/etc/init.d/S99zdropbear`; `restore-system.sh` in `/data/opt` re-applies after an OTA | V |
| Backup: wiki documents **no clean partition-dump workflow** for Mini 3; `contrib/rollback.sh` is untested on Mini-class; USB Burning Tool and UART login are **not available** (`rh_unlock=0`). A device that no longer boots is effectively unrecoverable | V |
| Kill switch to stock Alice: `yandex-noupdate.sh restore`, remove `S99z*` scripts, restore CA from `*.orig`, remove DNS override, optional factory reset. With Pulsar installed, `ctl.sh disable` restarts the Yandex stack instantly without reboot | V |
| Hardware: Amlogic AXG, 4×A53, **~244 MB RAM**, 512 MB NAND, Linux 4.9 aarch64 kernel, **32-bit armhf Buildroot userspace, glibc 2.30, no Python**, ALSA only (no PulseAudio) | V |

Correction to the brief: there is no "512 MB RAM" headroom; nothing interpreted (Python, Node) can run on the speaker. Only static binaries.

### Stock voice pipeline (what leaves the device)

| Stage | Where | Detail |
|---|---|---|
| Mic capture + VQE (echo cancel, beamforming) | on-device | `maind --platformd` ("aliced"), 3 PDM mics on `hw:0,1` ch 0–2, speaker reference ch 6, output 16 kHz mono |
| Wake word («Алиса»/«Яндекс») + ~25 no-wake commands | on-device | `nnet` engine in `libyandex_evil.so`, models in `/system/vendor/quasar/spotter_models` |
| ASR | **Yandex cloud** | post-wake audio as Opus over WebSocket (uniproxy, `uni.ws`), server-side endpointing |
| NLU / dialog / LLM | **Yandex cloud** | VINS/Megamind |
| TTS | **Yandex cloud** | Opus back → GStreamer `audio_client` → ALSA dmix |
| Telemetry | **Yandex cloud** | AppMetrica (serial, UUID, model), clickdaemon (~185 event types), fluent-bit logs, monitord ping every 10 s, random sampled audio clips for beta groups |

All of ASR/NLU/TTS/telemetry go to Yandex. Only wake word and VQE are local. (V)

## 2. Architecture comparison (Phase 1b)

| | A. Full takeover | **B. Thin terminal (recommended)** | C. Hybrid (self-hosted uniproxy) |
|---|---|---|---|
| What runs on speaker | Everything: wake, STT, LLM client, TTS | **Pulsar** daemon (4 MB static Rust binary in `/data/pulsar` + one init script): opens `hw:0,1`, runs Yandex's own VQE and wake-word libs from the firmware, streams 16 kHz PCM after wake over ESPHome Native API, plays TTS from a URL via firmware GStreamer | Whole stock Yandex stack; `uniProxyUrl` pointed at our server speaking Yandex's protobuf+zstd protocol |
| Feasible on 244 MB / no Python | **No** (no STT/TTS fits) | Yes (verified on Mini 3 `.50` by a contributor, PR #30) | Yes, but Mini 3 not listed by either project; FW 0.369 protocol must be reverse-matched |
| Root footprint on `system` | large | CA patch + 2 init scripts (`S99zdropbear`, `S99zpulsar`) | CA patch + `quasar.cfg` edit (may be overwritten by online config sync) |
| Alice fallback | lost | **`ctl.sh disable` → stock Alice back, no reboot** | Alice voice/music kept but telemetry keeps running |
| Where memory/secrets live | on a device with a weak update story | **Mac only** | Mac only |
| Latency | — | low: audio pushed as spoken; TTS fetched by URL | low |
| Effort | very high | **low-medium**: ~150-line Python consumer + known HA event choreography | high (protocol work) |
| Known risk | — | Mini 3 verified on one unit; wake pre-roll constants untuned; must emulate HA's `RUN_START…TTS_END` event order (A); OTA can delete `S99zpulsar` | high |

**Recommendation: B, with Pulsar as the terminal layer and a Python `aioesphomeapi` consumer on the Mac (no Home Assistant).** It is the only option that fits the hardware, keeps every byte of logic and memory off the speaker, has a reboot-free kill switch, and has an existing Mini 3 `.50` verification. Fallback if the custom consumer fights Pulsar's state machine: run Home Assistant as the consumer (the maintainer-tested path) with the same device-side setup.

Option D (raw `arecord | nc` to Mac, wake word on Mac) remains a one-evening diagnostic only: it needs our own AEC and loses the on-device Russian wake word.

### Server stack (Mac)

| Component | Choice | Why | Fallback |
|---|---|---|---|
| Speaker link | Pulsar ESPHome API over LAN (TCP 6053, Noise-encrypted) via `aioesphomeapi` | PCM arrives post-wake only; AEC + wake word already done | Home Assistant |
| STT | `mlx-whisper` + `whisper-large-v3-turbo`, local | pip-only, Metal, RU+EN | whisper.cpp; Vosk small-ru for streaming |
| LLM | Anthropic Messages API, `claude-haiku-5-5` low effort, streaming; `claude-sonnet-5-5` for tool-heavy turns | fastest/cheapest current models | — |
| Memory | Anthropic **memory tool** (`memory_20250818`, stable, no beta header) backed by a local directory via `BetaLocalFilesystemMemoryTool` | plain files, survives restarts, zero infra | sqlite notes injected into prompt |
| Web | Anthropic `web_search` + `web_fetch` server tools | no scraper to maintain | `urllib` + trafilatura |
| TTS | Silero `v5_6_ru` (RU, CC BY-NC, PyTorch CPU) + macOS `say` (EN) | best open Russian prosody | `say -v Milena` for both (only plain Milena installed locally today) |
| Weather | Open-Meteo with explicit models `ecmwf_ifs,ukmo_seamless,icon_seamless,gfs_seamless` (ICON-EU does not cover Kazakhstan; `best_match` = global blend). Yandex Weather API rejected: 7-day trial, business-only ToS. See `docs/research/06-weather-kz.md` | no key, no card, 10 000/day, previous-runs API for self-verification | Visual Crossing (key, no card, ~1 000/day) |
| Reminders / briefing | `sqlite3` + asyncio loop; launchd plist | stdlib | APScheduler 3 |
| Telegram | Bot API long polling with `urllib` | stdlib | python-telegram-bot |

## 3. Per-requirement feasibility (Phase 1c)

| Requirement | Feasible path | Limit |
|---|---|---|
| Voice in/out on the speaker | Pulsar → Mac | Mini 3 support is 2 weeks old; expect pre-roll tuning |
| Claude brain with persistent memory | Messages API + memory tool, files on Mac | memory text is sent to Anthropic when Claude views it |
| Web browse/scrape | Anthropic `web_search`/`web_fetch` | `web_fetch` has no JS rendering; add Playwright only if needed |
| Reminders / tasks / plans / morning briefing | sqlite + scheduler; briefing played on speaker via Pulsar `ANNOUNCE` (URL) | — |
| Weather | Open-Meteo | — |
| **iPhone: read notifications** | **iOS 27 Shortcuts "When I receive a notification from [app]" automation → POST to Mac** (runs locked, no tap). Backup: iPhone Mirroring → macOS Notification Center DB (`usernoted/db2/db`, needs Full Disk Access, not in EU) | one automation per source app; no history; Focus/Summary interaction untested (A) |
| **iPhone: Telegram send/receive** | Bot API for "I chat with the assistant"; Telethon (MTProto, acts as you) only if the assistant must read your personal chats | bot can't DM first; userbot has ToS/ban risk |
| **iPhone: launch apps** | Push with fixed message body → iOS 27 notification automation → `Open App` | per-app/per-phrase setup; foregrounding from a locked background automation is A |
| **iPhone: close apps** | **Not possible.** No official API force-quits another app. Nearest: `Lock Screen` action or `Set Focus` (blocks the app) chained after a push | app keeps running; Face ID may re-unlock |
| iPhone: phone state | Home Assistant companion sensors (battery, SSID, Focus bool, location, activity) or timed Shortcuts posting `Get Device Details`/`Get Current Focus`/`Find Alarm` | **current app and screen on/off are unreadable** on iOS |
| Push to phone | Telegram bot (simplest); ntfy (routes via ntfy.sh for APNs); HA local push | no fully self-hosted instant iOS push exists |
| **Mac: list / resume / message Claude Code sessions** | `claude agents --json` (live sessions incl. interactive, with cwd/status/name); Agent SDK `list_sessions()` (history); `claude --resume <id> "text"` injects into a running `--bg` session; SDK `query(resume=...)` for sessions the assistant owns | no official way to type into someone else's foreground TUI; `SendMessage` courier works but its socket schema is undocumented |
| Mac: remote control from phone | **Claude Code Remote Control** (`claude --rc` / `claude remote-control`): outbound HTTPS only, no inbound ports, claude.ai OAuth (this Mac is logged in via claude.ai, verified), optional Trusted Devices | needs Pro/Max/Team plan; API-key auth not supported |
| Mac: other remote control "where safe" | assistant HTTP bound to `127.0.0.1`; phone reaches it through Telegram bot or Tailscale, never a forwarded port | — |

## 4. Privacy: every external service that receives audio, text, or memory (recommended stack)

| Service | Receives | Notes |
|---|---|---|
| Anthropic API | transcribed text, system prompt, tool results, memory files when Claude reads them, web search queries, fetched page text | **no audio** |
| Anthropic web search backend + fetched sites | queries, URLs | |
| Open-Meteo | approximate lat/lon | per-IP, no key |
| Visual Crossing (fallback only) | approximate lat/lon, API key | only when Open-Meteo fails |
| aviationweather.gov | airport codes only | 14-day accuracy self-check |
| Cloud STT fallback (opt-in per gate G4; provider TBD when wired, e.g. OpenAI) | **audio** of the utterance | only when local mlx-whisper fails or is too slow; off by default |
| Telegram | bot chat messages | |
| Hugging Face / GitHub | one-time model + Pulsar downloads | |
| Yandex | **while Alice fallback is enabled**: everything in §1. While Pulsar is enabled: DNS/NTP queries and account binding only (A) | OTA blocked by `yandex-noupdate.sh` |
| Apple (iPhone path) | Shortcuts automations run locally; pushes via APNs carry only our fixed trigger strings | |

Audio never leaves the LAN: speaker → Mac over Noise-encrypted ESPHome API; STT, TTS, wake word all local. Cloud STT (Yandex SpeechKit, OpenAI) is **not** in the plan unless you opt in at the gate.

## 5. Risks that shape the plan

1. **Bricking is unrecoverable.** No USB-burn path, no UART login. Every `system` write (CA patch, two init scripts) is done exactly per guide with hash checks, nothing else touches `system`. Backups let us restore *contents* via root shell, not revive a dead boot.
2. **Factory partition may already be `.50`.** Then root is impossible on this unit; project stops before any write.
3. **OTA.** Blocked by signature-key swap; `restore-system.sh` insurance. Still, never leave the speaker on real DNS with ADB markers expected to persist (wiki: `chattr +i` untested on Mini 3).
4. **Pulsar Mini 3 maturity.** One-unit verification, pre-roll untuned, AEC quality unknown. Mitigation: `ctl.sh disable` fallback; HA as alternate consumer.
5. **Mac as brain** must stay awake and on LAN (`caffeinate`/pmset), and serve TTS files over LAN HTTP to the speaker. Bound to LAN interface only.

## 6. Open items for the user (gate questions)

See end of `tasks/todo.md`.
