# alice — Yandex Station Mini 3 → self-hosted Claude assistant

State lives in files, not in the chat:

- `docs/research-report.md` — findings, architecture (thin terminal: Pulsar on the speaker, brain on the M4 Pro).
- `docs/research/*.md` — detailed, sourced research.
- `tasks/todo.md` — the plan; tick items only after their check passes. Gate G1–G5 answered 2026-11-08.
- `tasks/lessons.md` — corrections from the owner.

Secrets, device identifiers, firmware dumps: `~/alice-secrets/`, `~/alice-backup/` — never in this repo.

## Setup on any Mac

```bash
git clone <your-private-remote> ~/Projects/alice && cd ~/Projects/alice
uv venv --python 3.12 .venv && uv pip install --python .venv/bin/python -r requirements.txt
.venv/bin/python -c "import anthropic, aioesphomeapi, mlx_whisper"   # must exit 0
```

## Continue on another device

Three ways, pick by device:

**1. Phone / browser, keep the session running on this Mac (official Remote Control).**
In this Claude Code session type `/remote-control` (alias `/rc`). Or from Terminal on this Mac:

```bash
cd ~/Projects/alice
claude --remote-control --resume 8f1d7eaa-a80a-47bc-a6a6-88440325aacf
```

Then open claude.ai/code or the Claude mobile app: the session appears with a computer icon. Outbound HTTPS only, no inbound ports, claude.ai login (this Mac is logged in that way). Needs this Mac awake.

**2. Another Mac, same transcript (resume the very same conversation).**
Push this repo to a private remote first (`git remote add origin <url> && git push -u origin main`). On the other Mac, with the same macOS username so the encoded project path matches:

```bash
git clone <url> ~/Projects/alice
mkdir -p ~/.claude/projects/-Users-alisherrymkan-Projects-alice
# copy from this Mac (AirDrop/scp): the transcript and the memory dir
scp 'this-mac:~/.claude/projects/-Users-alisherrymkan-Projects-alice/8f1d7eaa-a80a-47bc-a6a6-88440325aacf.jsonl' \
    ~/.claude/projects/-Users-alisherrymkan-Projects-alice/
scp -r 'this-mac:~/.claude/projects/-Users-alisherrymkan-Projects-alice/memory' \
    ~/.claude/projects/-Users-alisherrymkan-Projects-alice/
cd ~/Projects/alice && claude --resume 8f1d7eaa-a80a-47bc-a6a6-88440325aacf
```

If the username differs, `claude --resume /abs/path/to/8f1d7eaa-….jsonl` (resume by transcript path) per https://code.claude.com/docs/en/sessions . Keep exactly one copy of the transcript per Mac: the docs say a hand-copied duplicate in a second project dir makes `--resume <id>` report not-found.

Server-mode alternative for option 1 (no local TUI, phone-only): `claude remote-control --session-id 8f1d7eaa-a80a-47bc-a6a6-88440325aacf`.

**3. Fresh session anywhere (most robust).**
Clone the repo, run `claude`, and say:
"Read README.md, docs/research-report.md and tasks/todo.md. Gate G1–G5 is answered. Continue from the first unticked item in Phase 2a."
Everything decided so far is in those files.

## Attribution

Device facts condensed from the *Yandex Station Extended* wiki (hackalice.me, github.com/Marker284/yandex-station-extended), CC BY 4.0, and the Pulsar project (github.com/Morvex885/pulsar, AGPL-3.0).
