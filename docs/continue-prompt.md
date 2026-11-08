# Prompt for a new Claude Code session

Paste the block below as the first message in a fresh `claude` session started inside a clone of https://github.com/rymqan/alice .

```text
You are the lead engineer on a personal project: replace Alice on a Yandex Station Mini 3 ("Bergamot") with a self-hosted Claude voice assistant. Work in this repo (https://github.com/rymqan/alice). Plan first, then build. Track work in tasks/todo.md; log any correction from me in tasks/lessons.md. Use .venv/bin/python (create .venv with `uv venv --python 3.12 .venv` and install requirements.txt if missing).

READ FIRST, in this order:
1. README.md
2. docs/research-report.md (architecture decision and per-requirement feasibility)
3. tasks/todo.md (the plan; ticked items are done and verified)
4. tasks/lessons.md
5. docs/research/01..06-*.md only when a step needs the underlying detail.

STATE (as of 2026-11-08):
- Phase 1 research is complete. Phase 2 gate G1–G5 is answered and approved:
  G1 brain host = the always-on M4 Pro MacBook on the home LAN.
  G2 Telegram = Bot API (new bot via @BotFather), no user-account scraping.
  G3 weather = Open-Meteo with explicit models ecmwf_ifs,ukmo_seamless,icon_seamless,gfs_seamless; fallback Visual Crossing; Yandex Weather API rejected.
  G4 audio stays on the LAN; cloud STT fallback is allowed but off by default.
  G5 device work is approved.
- Architecture = B "thin terminal": Pulsar (Rust daemon, github.com/Morvex885/pulsar, AGPL) runs on the rooted speaker, reuses Yandex's own VQE and wake-word libs, streams post-wake 16 kHz PCM over the ESPHome Native API; a Python aioesphomeapi consumer on the Mac does STT (mlx-whisper), Claude (Messages API, claude-haiku-5-5, memory tool, web_search/web_fetch), TTS, and serves audio back by URL. Home Assistant is the fallback consumer. `ctl.sh disable` restores stock Alice without reboot.
- Device facts that gate everything: Mini 3 is on build 0.369.1.50 (Oct 21 2026 forced OTA). Root works only via "retail CA + markers from the factory image" (https://hackalice.me/start/retail-ca-mini3/) and ONLY if factory mode still reports a factory-N build; if it reports .50, stop. Bricking is unrecoverable (no USB burn, no UART login). The speaker has 244 MB RAM, 32-bit userspace, no Python.
- Done so far: research docs, plan, .venv deps verified (anthropic, aioesphomeapi, mlx-whisper), repo on GitHub.
- Next unticked item: tasks/todo.md Phase 2a item P1 (read the Mini 3 root guide and SSH page end to end, write docs/device-procedure.md with the exact commands and CA hashes for build .50). Continue down the list in order.

RULES:
- Follow the abort rule in tasks/todo.md Phase 2b at every device step. Never write to the speaker's system partition beyond what the guide specifies. Take the full backup (D4) before the first write (D5).
- Secrets, MAC addresses, serials, firmware dumps go to ~/alice-secrets/ and ~/alice-backup/, never into the repo. handoff/ is gitignored; keep it that way.
- Smallest working diff; stdlib before dependencies; each milestone ends with one runnable check and a short log of what was verified.
- Tick a todo item only after its check passed. Commit after each verified item with a short message; push to origin main.
- If something deviates from the guide or the plan, stop and ask me instead of improvising on the device.

Start by reading the files above, then tell me in five lines what P1 requires from me (for example, the router DNS method), and begin.
```
