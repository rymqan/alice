# alice — Yandex Station Mini 3 → self-hosted Claude assistant

State lives in files, not in the chat:

- `docs/research-report.md` — findings, architecture (thin terminal: Pulsar on the speaker, brain on the M4 Pro).
- `docs/research/*.md` — detailed, sourced research.
- `tasks/todo.md` — the plan; tick items only after their check passes. Gate G1–G5 answered 2026-11-08.
- `tasks/lessons.md` — corrections from the owner.

Secrets, device identifiers, firmware dumps: `~/alice-secrets/`, `~/alice-backup/` — never in this repo.

## Setup on any Mac

```bash
git clone https://github.com/rymqan/alice.git ~/Projects/alice && cd ~/Projects/alice
uv venv --python 3.12 .venv && uv pip install --python .venv/bin/python -r requirements.txt
.venv/bin/python -c "import anthropic, aioesphomeapi, mlx_whisper"   # must exit 0
```

## Hand off to another developer

The project state is in the repo files (`README.md`, `docs/`, `tasks/todo.md`). The conversation itself is a local JSONL transcript on the owner's Mac. Two levels of hand-off:

**Level 1 — repo only (always do this).** The repo is at https://github.com/rymqan/alice (public). The other developer clones it and starts a fresh session:

```bash
git clone https://github.com/rymqan/alice.git ~/Projects/alice && cd ~/Projects/alice
uv venv --python 3.12 .venv && uv pip install --python .venv/bin/python -r requirements.txt
claude
```
First prompt: "Read README.md, docs/research-report.md and tasks/todo.md. Gate G1–G5 is answered. Continue from the first unticked item in Phase 2a." Everything decided so far is in those files; the research transcripts add nothing the docs do not already say.

**Level 2 — continue the exact conversation (optional).** Claude Code resumes any transcript by absolute path (`claude --resume <transcript-path>`, https://code.claude.com/docs/en/sessions#resume-a-session). Steps:

1. Owner sends the file `~/alice-handoff/8f1d7eaa-a80a-47bc-a6a6-88440325aacf.jsonl` (2.8 MB) out of band (AirDrop, Signal, shared drive). It was scanned on 2026-11-08: no API keys or tokens; it does contain the owner's hostname and email twice (from Claude Code's own system context) and local paths under `/Users/alisherrymkan/`.
2. Developer puts it anywhere outside the repo, e.g. `~/alice-handoff/`, clones the repo (Level 1), then from the repo dir:
   ```bash
   claude --resume ~/alice-handoff/8f1d7eaa-a80a-47bc-a6a6-88440325aacf.jsonl
   ```
3. Expect: the session resumes in the developer's current directory with the full history. It continues on the model it was using (Claude Fable 5.1); pass `--model <alias>` if that model is not on their plan. Absolute paths quoted in the history point at the owner's Mac; the same files exist relative to the repo root. Keep a single copy of the file per machine (the docs warn that duplicates make `--resume <id>` lookups fail). A "Resume this conversation?" usage-estimate dialog may appear; accept it.

Not available for cross-account hand-off (checked in the docs on 2026-11-08): Remote Control and cloud-session sharing are tied to the owner's claude.ai account. Cloud-session sharing gives a teammate a read-only view, and `--teleport` requires the same account; a terminal session cannot be pushed to the cloud.

## Attribution

Device facts condensed from the *Yandex Station Extended* wiki (hackalice.me, github.com/Marker284/yandex-station-extended), CC BY 4.0, and the Pulsar project (github.com/Morvex885/pulsar, AGPL-3.0).
