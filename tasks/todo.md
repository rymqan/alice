# Alice replacement — plan

Each item has a check. Tick only after the check passed. Device items are **blocked until the Phase 2 gate is approved**.
Secrets, MACs, serials, backups live outside the repo (`~/alice-secrets/`, `~/alice-backup/`), never in `docs/` or git.

## Phase 1 — Research (done)

- [x] Firmware/root status, stock pipeline → `docs/research/01-mini3-root.md`
- [x] iOS bridge limits → `docs/research/02-ios-bridge.md`
- [x] Claude Code resume/remote control (verified locally on CLI 2.1.296) → `docs/research/03-claude-code-control.md`
- [x] Server stack → `docs/research/04-server-stack.md`
- [x] Pulsar + raw audio on Mini 3 → `docs/research/05-pulsar-audio.md`
- [x] Synthesis + architecture recommendation → `docs/research-report.md`

## Phase 2 — Gate (needs your answers; nothing below starts before this)

- [ ] G1. Brain host: this MacBook (must stay awake, on home LAN) or a home server?
- [ ] G2. Telegram: new bot via @BotFather (assistant = bot you chat with) or also a user account via MTProto (assistant reads/sends as you, ToS risk)?
- [ ] G3. Weather provider: Open-Meteo (no key, default) or another?
- [ ] G4. Audio leaving the device: plan keeps all audio on LAN (local STT). OK, or do you allow a cloud STT (Yandex SpeechKit / OpenAI) as fallback?
- [ ] G5. Explicit approval to touch the speaker with the steps in Phase 2b–2c.

## Phase 2a — Mac prep (safe, no device)

- [ ] P1. Read the full Mini 3 guide once, end to end: https://hackalice.me/start/retail-ca-mini3/ and https://hackalice.me/start/ssh/ . Check: a `docs/device-procedure.md` exists quoting the exact commands for our build with the per-build CA hashes from the guide.
- [ ] P2. Install adb: `brew install android-platform-tools`. Check: `adb version` prints a version.
- [ ] P3. Set up the custom update server (eQuasar, Docker + nginx) per guide, with `aaa-certificate-services.pem` from `yandex-station-binaries`. Check: `curl -k https://<mac-ip>/check_updates` answers; cert chain ends at AAA.
- [ ] P4. Decide DNS redirect method (router override vs Mac hotspot). Check: from a phone on that network, `nslookup quasar.yandex.net` returns the Mac IP; note the exact rollback command in `docs/device-procedure.md`.
- [ ] P5. Download Pulsar v0.6.0 `pulsar-armhf.bin` + `satellite/deploy/{ctl.sh,boot.sh,supervise.sh}` from https://github.com/Morvex885/pulsar/releases into `~/alice-secrets/pulsar/`. Check: `shasum -a 256` matches the release asset digest; file is ~4.2 MB.
- [ ] P6. Python env: `.venv` (3.12) exists. Install `anthropic aioesphomeapi mlx-whisper`. Check: `.venv/bin/python -c "import anthropic, aioesphomeapi, mlx_whisper"` exits 0.
- [ ] P7. `ANTHROPIC_API_KEY` in `~/alice-secrets/env` (chmod 600), loaded by `source`, never in repo. Check: `.venv/bin/python -c "import anthropic; print(anthropic.Anthropic().models.list().data[0].id)"` prints a model id.

## Phase 2b — Device: verify, root, back up (first write to `system` happens at D4)

Abort rule at every step: any output that differs from the guide's expected value → stop, restore DNS, write the discrepancy to `tasks/lessons.md`, ask.

- [ ] D1. Record retail build from the Yandex app (device → About) into `~/alice-secrets/device.md`. Check: build string starts `0.369.1.50.`.
- [ ] D2. Boot factory mode (hold mic-off 10–15 s at power-on), get root shell via P3/P4 flow. Check: `adb shell id` → `uid=0(root)`; `cat /proc/cmdline` contains `rootfstype=ramfs`.
- [ ] D3. **Precondition check:** `cat /system/vendor/quasar/version_full` shows `factory-N`. If it shows `0.369.1.50…` → **STOP, project ends here for this unit** (restore DNS, reboot normally).
- [ ] D4. **Backup before any write.** From the factory root shell: `cat /proc/mtd` → for each mtdN: `adb shell "dd if=/dev/mtdblockN" > ~/alice-backup/mtdN.img` (if `dd` on a partition errors on bad blocks, record it and fall back to `tar` of the mounted filesystem). Also tar `/mnt/retail/system/vendor/quasar/` (after mounting ubi0:rootfs ro) and `/bkupgrade/data/quasar/`. Check: `~/alice-backup/SHA256SUMS` lists every image; `mtd3` image size = 176 MB; `tar tzf quasar.tgz | grep quasar.cfg` hits.
- [ ] D5. Root per guide: mount retail rw, back up CA to `*.orig`, append AAA cert, push back. Check: `grep -c BEGIN ca-certificates.crt` = 123; size = orig + 1517 bytes; sha256 matches the guide's value for our build.
- [ ] D6. ADB markers + Wi-Fi persistence per guide. Check: `activate_adb.sh check` reports both markers; `grep update_config /bkupgrade/data/wpa_supplicant.conf` → `update_config=0`.
- [ ] D7. Normal boot (`fw_setenv` per guide), reboot. Check: `adb connect <ip>:5555 && adb shell id` → root; Alice still answers a voice question.
- [ ] D8. Dropbear SSH (port 2222) via `S99zdropbear` **with a key**, not empty password. Check: `ssh -p 2222 -i ~/alice-secrets/speaker_key root@<ip> uname -a` works; password login fails.
- [ ] D9. **Update block:** `yandex-noupdate.sh`. Check: `yandex-noupdate.sh status` reports blocked; `grep updatesd /system/vendor/quasar/launch.json` empty; after reboot, Alice still works.
- [ ] D10. Insurance: copy CA + dropbear + `restore-system.sh` to `/data/opt`. Check: `ls /data/opt` lists them; script runs without error in dry mode.
- [ ] D11. Restore real DNS on the router. Check: `nslookup quasar.yandex.net` returns Yandex; SSH still works after reboot (ADB may stop — expected).
- [ ] D12. **Rollback rehearsal:** run `yandex-noupdate.sh restore`, confirm `status` says unblocked, then re-apply D9. Check: both states observed and logged in `docs/device-log.md` (no identifiers).
- [ ] D13. Write `docs/kill-switch.md`: (a) instant: `/data/pulsar/ctl.sh disable` (after D14); (b) full: `yandex-noupdate.sh restore`, remove `S99zpulsar`/`S99zdropbear`, restore `ca-certificates.crt.orig`, remove DNS override, optional factory reset. Check: file exists; every command copied from the guide verbatim.

## Phase 2c — Device: Pulsar terminal

- [ ] D14. Manual install (not the Go installer, to control every write): `/data/pulsar/{pulsar-armhf.bin,ctl.sh,boot.sh,supervise.sh}` + `/etc/init.d/S99zpulsar` (one `remount,rw`). Check: `ctl.sh status` shows version 0.6.0; Alice still works (Pulsar disabled by default).
- [ ] D15. `ctl.sh enable`; say «Алиса». Check: `ctl.sh log 50` shows a wake detection and "voice assistant not subscribed"; `ctl.sh disable` → Alice answers again within ~10 s.
- [ ] D16. Reboot with Pulsar enabled. Check: `ctl.sh status` running after boot; `q` stack stopped.

## Phase 3 — Build (after gate; each milestone ends with one runnable check)

### M1 — voice → Mac → Claude → voice, one scenario "what's the weather"
- [ ] M1.1 `server/pulsar_client.py`: `aioesphomeapi` connect (Noise key from `ctl.sh api-key`), `subscribe_voice_assistant`, buffer PCM until stop, write WAV. Check: `tests/check_m1_capture.py` → after saying «Алиса, какая погода», a WAV >1 s exists.
- [ ] M1.2 `server/stt.py`: mlx-whisper transcribe. Check: `.venv/bin/python server/stt.py sample.wav` prints Russian text containing «погода».
- [ ] M1.3 `server/weather.py`: Open-Meteo via `urllib`, lat/lon from env. Check: `python server/weather.py` prints today's temp.
- [ ] M1.4 `server/brain.py`: Messages API, `claude-haiku-5-5`, `weather` tool. Check: `python server/brain.py "какая погода"` prints an answer with a temperature.
- [ ] M1.5 `server/tts.py`: `say -v Milena` → WAV (Silero later). Check: file plays on the Mac.
- [ ] M1.6 LAN HTTP file server on the Mac (stdlib `http.server`, bound to LAN IP) + Pulsar events `RUN_START → STT_* → TTS_START/TTS_END(url) → RUN_END`. **Check (milestone gate):** `tests/check_m1.py` runs the loop end to end; speaker says the weather; `docs/m1-log.md` records what was verified.

### M2 — memory, reminders, morning briefing
- [ ] M2.1 Memory tool (`memory_20250818`) with `BetaLocalFilesystemMemoryTool(~/alice-data/memories)`. Check: tell it your name, restart server, ask "who am I" → correct.
- [ ] M2.2 `server/reminders.py`: sqlite table, asyncio due-loop, `ANNOUNCE` to speaker. Check: "remind me in 1 minute" → speaker announces at T+60 s ±5.
- [ ] M2.3 Morning briefing: launchd plist → `briefing.py` (weather + reminders + tasks) → announce. Check: `launchctl kickstart` triggers it now.

### M3 — web
- [ ] M3.1 Enable Anthropic `web_search` + `web_fetch` server tools with `max_uses`. Check: ask for today's top headline → answer cites a URL.

### M4 — Telegram
- [ ] M4.1 Bot token in `~/alice-secrets/env`; `urllib` long-poll loop; same brain, same memory. Check: send "what's the weather" from your phone → reply.
- [ ] M4.2 Allowlist your Telegram user id only. Check: another account gets no reply.

### M5 — iPhone bridge
- [ ] M5.1 Shortcut automation "When I receive a notification from Telegram" → POST to Mac (via Tailscale or LAN). Check: a Telegram notification shows up in server log.
- [ ] M5.2 Launch app: fixed-body push → automation → Open App. Check: "open Maps" from the speaker opens Maps on the locked phone.
- [ ] M5.3 Close app = `Lock Screen` action (documented limit). Check: "close it" locks the phone.
- [ ] M5.4 Phone state: timed Shortcut posts battery/Focus/next alarm every 15 min. Check: "what's my battery" answered from latest post.

### M6 — Claude Code control
- [ ] M6.1 `server/cc.py`: list = `claude agents --json`; history = `claude_agent_sdk.list_sessions()`. Check: "list my Claude sessions" names this project.
- [ ] M6.2 Send: `claude --resume <id> "text"` for `--bg` sessions; SDK `query(resume=…)` for assistant-owned ones. Check: message appears in the target session transcript.
- [ ] M6.3 Phone: `claude --rc` documented as the official path; no inbound ports opened on the Mac. Check: `lsof -iTCP -sTCP:LISTEN` shows only `127.0.0.1`/LAN binds we created.

## Review (filled at the end of each phase)

_(empty)_
