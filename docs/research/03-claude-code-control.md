# Controlling Claude Code sessions on a Mac from another process/device — Nov 2026

Research agent output + local verification on this Mac (Claude Code 2.1.296). Canonical docs host: `https://code.claude.com/docs/en/…` (old docs.claude.com URLs 301 there).

## 1. CLI: resume / headless / list — VERIFIED

Flags (`claude --help` 2.1.296 + https://code.claude.com/docs/en/cli-reference): `-p/--print`, `-c/--continue`, `-r/--resume [id|name|/abs/path.jsonl]`, `--session-id <uuid>`, `--fork-session`, `--output-format text|json|stream-json`, `--input-format text|stream-json`, `--permission-mode acceptEdits|auto|bypassPermissions|manual|dontAsk|plan`, `--permission-prompts host|none`, `--allowedTools`, `--disallowedTools`, `--tools`, `--restricted`, `--bare`, `--no-session-persistence`, `--max-turns`, `--max-budget-usd`, `-n/--name`, `--bg`, `--teleport`, `--cloud`, `--remote-control [name]`, `--desktop`.

**Headless resume works** (https://code.claude.com/docs/en/headless#continue-conversations):
```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
claude -p --resume <id> --output-format json "summarize what we changed" | jq -r .result
```
`--resume <id>` resolves from any directory (v2.1.223+). `-p` sessions are hidden from the interactive picker and `claude --continue`, but `claude -p --continue` includes them (https://code.claude.com/docs/en/sessions#loop--p-agent-sdk-and-background-sessions).

**Transcript storage** (https://code.claude.com/docs/en/sessions#where-transcripts-are-stored): `~/.claude/projects/<project>/<session-id>.jsonl`, `<project>` = cwd with non-alphanumerics → `-`. Docs warn the JSONL format "is internal … and changes between versions". Confirmed locally.

**Listing sessions — no `claude sessions` command. Official options:**
1. **Live:** `claude agents --json` (`--all` adds completed background ones, `--cwd <path>` scopes). JSON array of every live session incl. plain interactive terminals. Fields: `cwd`, `kind` (`interactive`|`background`), `startedAt`, `sessionId`, `name`, `pid`, `status` (`busy`|`waiting`|`idle`), `waitingFor`; background adds `id` (short), `state` (`working`|`blocked`|`done`|`failed`|`stopped`). VERIFIED https://code.claude.com/docs/en/agent-view + ran locally (returned this very session). Docs say this is the supported interface; don't parse `~/.claude/jobs/<id>/state.json`.
2. **Historical:** Agent SDK `list_sessions()` / `listSessions()`.

**Resuming a session open elsewhere** (https://code.claude.com/docs/en/sessions):
- *Background (`--bg`) session still running:* `claude --resume <id> "prompt"` **sends the prompt into the running session as its next turn** and attaches. With piped stdio / `--output-format json` / `--permission-mode` / `--max-turns`, Claude Code **sends nothing and exits 1**. Use `--fork-session` for a copy, or `claude stop <id>` first.
- *Plain interactive TUI in another terminal:* no lock; resuming the same session in two terminals interleaves messages into one transcript; the running TUI does **not** see the `-p` turn. v2.1.248+ agent view refuses to open such a row.

## 2. Agent SDK — VERIFIED

- `claude-agent-sdk` **0.2.179** (2026-11-06), Python ≥3.10 (https://pypi.org/project/claude-agent-sdk/). `@anthropic-ai/claude-agent-sdk` 0.3.319. Python 0.2.169+ bundles CLI ≥2.1.302 (https://code.claude.com/docs/en/agent-sdk/python).
- Options (https://code.claude.com/docs/en/agent-sdk/sessions): `resume=<id>`, `continue_conversation=True`, `fork_session=True`, `session_id=<uuid>` (new sessions only), `permission_mode`, `allowed_tools`, `disallowed_tools`, `cwd`, `can_use_tool`, `hooks`, `setting_sources`, `env`.
- Session id: `ResultMessage.session_id`; Python `SystemMessage.data`.
- Multi-turn: Python `ClaudeSDKClient` (`query()`, `receive_response()`, `interrupt()`, `set_permission_mode()`).
- Catalogue: `list_sessions(directory=None, limit, offset, include_worktrees=True) -> list[SDKSessionInfo]` (`session_id, summary, last_modified, file_size, custom_title, first_prompt, git_branch, cwd, tag, created_at`), `get_session_messages()`, `get_session_info()`, `rename_session()`, `tag_session()` (https://code.claude.com/docs/en/agent-sdk/python#list_sessions).
- Python hook events: `PreToolUse, PostToolUse, PostToolUseFailure, UserPromptSubmit, Stop, SubagentStop, PreCompact, Notification, SubagentStart, PermissionRequest` — **no SessionStart/SessionEnd in Python SDK**; use settings-file hooks.
- SDK/`-p` sessions bind an inbox socket and appear in `ListAgents`, except with `--bare` (https://code.claude.com/docs/en/cross-session-messaging#non-interactive-sessions).

## 3. Remote Control (official phone path) — VERIFIED

https://code.claude.com/docs/en/remote-control . Entry points:
- `claude remote-control [--name "X"] [--spawn same-dir|worktree|session] [--capacity N] [--permission-mode M] [--continue|--session-id <id>]` — server mode, no local TUI, serves up to 32 sessions in cwd; phone/web can spawn sessions. `--continue`/`--session-id` re-serve sessions ~4 h after the server stopped.
- `claude --remote-control ["name"]` / `--rc` — interactive TUI also remote-controllable.
- `/remote-control` / `/rc` inside a running session. Also `/config` → "Enable Remote Control for all sessions".

Auth/transport: requires claude.ai login (**Pro, Max, Ultra, Team, Enterprise; API keys and `setup-token` not supported**). "Your local Claude Code session makes outbound HTTPS requests only and never opens inbound ports … registers with the Anthropic API and polls for work." Phone/web: sessions appear in claude.ai/code with a computer icon; send messages/images/files, pick model, stop subagents, rename. Env `CLAUDE_CODE_BRIDGE_SESSION_ID` while connected. Optional **Trusted Devices** (passkey/biometric step-up).

Related (https://code.claude.com/docs/en/claude-code-on-the-web): `--cloud "task"` creates a cloud VM session; `claude -p "msg" --cloud <session_id>` queues a message into it; `--teleport <id>` pulls a cloud session into the terminal (one-way).

## 4. Hooks as session registry — VERIFIED

Common stdin for every event (https://code.claude.com/docs/en/hooks): `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name`, `prompt_id`, `scratchpad_dir`. Env: `CLAUDE_PROJECT_DIR`, `CLAUDE_CODE_MESSAGING_SOCKET`, `CLAUDE_CODE_MESSAGING_TOKEN`, `CLAUDE_CODE_BRIDGE_SESSION_ID`. Hook types: `command`, `http` (POSTs JSON to a URL), `mcp_tool`, `prompt`, `agent`.
- `SessionStart` matchers `startup|resume|clear|compact|fork`; `SessionEnd` adds `reason`, default timeout 1.5 s.
- `Stop` adds `stop_hook_active`, `last_assistant_message`, `background_tasks[]`, `session_crons[]`.
- `Notification` matchers `permission_prompt|idle_prompt|auth_success|elicitation_dialog|agent_needs_input|agent_completed`.
Hooks run under `-p` unless `--bare`. **Verdict:** `claude agents --json` already gives live sessions with zero setup; use hooks for `Stop`/`last_assistant_message` push-to-voice and `Notification(permission_prompt)`.

## 5. Cross-session messaging and channels — VERIFIED

Cross-session messaging (v2.1.224+, https://code.claude.com/docs/en/cross-session-messaging): every non-bare session binds a per-session Unix socket (`CLAUDE_CODE_MESSAGING_SOCKET`; locally `/tmp/cc-socks/<pid>.sock`, mode 0600). Delivery between tool calls or starts a new turn when idle; never via Anthropic for same-machine. Docs document newline-delimited JSON with optional first auth line `{"type":"auth","token":"<CLAUDE_CODE_MESSAGING_TOKEN>"}`, **but the message-line schema is not documented** (ASSUMED undocumented). Inbound control `crossSessionInbound: accept|hold|refuse`; default holds messages from bypass-permission senders. Messages can't approve anything. **No `claude send` CLI, no HTTP API.**

Practical official bridge: a tiny SDK "courier" session with `allowed_tools=["SendMessage","ListAgents"]` relaying text to a named session. ASSUMED pattern built from documented tools.

Channels (research preview, https://code.claude.com/docs/en/channels): MCP server with `claude/channel` capability pushes notifications into a session started with `claude --channels plugin:telegram@claude-plugins-official` (official Telegram/Discord/iMessage plugins; needs Bun; pairing + sender allowlist). Custom servers need `--dangerously-load-development-channels server:<name>` and are **ignored under `-p`/SDK**. Optional permission relay lets you approve tool calls from the phone.

## 6. `claude mcp serve` and driving an existing TUI — VERIFIED

`claude mcp serve` exposes Claude Code's tools as a stdio MCP server; does **not** expose session control (https://code.claude.com/docs/en/mcp#use-claude-code-as-an-mcp-server). No official way for an external process to type into a foreground TUI. Official substitutes: start sessions with `--bg` (then `claude --resume <id> "prompt"` / `claude attach` / `claude logs`), Remote Control, or SDK-headless. tmux `send-keys` remains an unofficial hack. `--bg` can't combine with `-p`.

## 7. Recommended architecture (agent's recommendation)

1. Assistant server on the Mac (Python, `claude-agent-sdk`, `.venv`) owns sessions via `ClaudeSDKClient`/`query(resume=…)`, explicit `permission_mode` + `allowed_tools`, `cwd` per project, `setting_sources=["project"]`. Bind HTTP only on `127.0.0.1`.
2. Listing: `claude agents --json` (live) + `list_sessions()` (history).
3. Sending: SDK-owned → `resume=`; `--bg` sessions → `claude --resume <id> "text"` (TTY, not piped); foreground TUI → `SendMessage` courier or ask user to run `/rc`.
4. Phone: Anthropic **Remote Control** relay (outbound-only, claude.ai OAuth, Trusted Devices) or the official Telegram channel plugin. Never expose a listening port. Tailscale is a sound third-party option for private reachability (not in Claude docs).
5. Security: `-p` skips the workspace-trust dialog and runs project hooks/MCP — prefer `--bare`/`setting_sources` for untrusted dirs; bypass-permissions mode documented only for sandboxes; cross-session messages never grant permissions.

Local paths: `~/.claude/projects/<encoded-cwd>/<id>.jsonl`, `~/.claude/sessions/<pid>.json` (live registry), `~/.claude/daemon/roster.json`, `~/.claude/jobs/<id>/`.
