# Server-side stack: rooted Mini 3 as mic/speaker, brain on a Mac (2026-11-08)

Research agent output. VERIFIED = source read; ASSUMED = inference.

**Critical device fact:** Mini 3 has **~244 MB RAM**, 4× Cortex-A53 @1.2 GHz, 64-bit kernel 4.9.68, **32-bit armhf Buildroot userspace, glibc 2.30**, ALSA only, `aplay` present, `arecord` not mentioned, community recommends **statically linked aarch64-musl binaries**. No Python. VERIFIED https://hackalice.me/devices/station-mini3/ . Any Python-on-device plan (openWakeWord, wyoming-satellite) is off the table.

Device audio (VERIFIED same page): ALSA card0 `AML-AUGESOUND`, `hw:0,0` playback (ACM8625 amp, dmix in `/etc/asound.conf`), `hw:0,1` 8-channel capture; 3 PDM mics; Yandex `audiod` does VQE (echo cancellation using channel 6 speaker-feed) and emits 16 kHz; `aliced` on 127.0.0.1:9875. Stock spotter closed-source, treat as not reusable (ASSUMED).

## 1. Wake word

| Option | Status 2026 | License | Fits Mini 3? |
|---|---|---|---|
| openWakeWord 0.6.0 (no release since Feb 2024) | Python + onnxruntime; English only | Apache-2.0 code, **CC BY-NC-SA 4.0** models | No on device; fine on Mac. VERIFIED https://github.com/dscripka/openWakeWord |
| microWakeWord `pymicro-wakeword` 2.5.0 (Sep 2026) | wheels need glibc ≥2.35; macOS 15+ universal2; ships `okay_nabu` | Apache-2.0 | No on device; yes on Mac. VERIFIED https://pypi.org/project/pymicro-wakeword/ |
| Porcupine | **Free tier ended 2026-06-30**, no non-commercial tier | proprietary | Avoid. https://community.home-assistant.io/t/fyi-picovoice-confirmed-free-tier-accesskeys-will-stop-working-after-june-30-2026/1012744 |
| Russian wake word | both engines English-trained; custom training via Piper ru voices possible | — | ASSUMED, quality unproven |

**Recommendation:** wake word **on the Mac** over continuous 16 kHz mono LAN stream (256 kbit/s). Privacy nil if stream stays on LAN. `pymicro-wakeword` (Apache) or openWakeWord on Mac; English phrase for now. Gate with Silero VAD.

## 2. Audio transport

- **wyoming-satellite archived 2026-01-27**, replaced by Linux Voice Assistant (ESPHome protocol, **requires HA**, Python 3.11+, ≥512 MB RAM). Neither runs on Mini 3. VERIFIED https://github.com/rhasspy/wyoming-satellite , https://github.com/OHF-Voice/linux-voice-assistant .
- **Wyoming protocol** alive: `wyoming` 1.11.0 (2026-11-02), MIT; JSON-line header + PCM payload; `AsyncServer.from_uri()` + `AsyncEventHandler`; does not need HA. VERIFIED https://github.com/rhasspy/wyoming .
- PulseAudio tunnel: N/A (ALSA only). snapcast: multiroom, ~1 s buffer, wrong tool.
- **Raw PCM over TCP**: BusyBox `nc` likely (ASSUMED); `aplay` confirmed; `arecord` unconfirmed → may need static alsa-utils `arecord` or tiny static Go/C capture program on `hw:0,1`. 8-channel capture may need `plug:` + channel pick; alternatively tap `audiod` VQE output (ASSUMED).

**Recommendation:** two plain TCP streams, no protocol lib on device:
- Uplink: `arecord -D plughw:0,1 -f S16_LE -r 16000 -c 1 -t raw | nc MAC 10700` (restart loop).
- Downlink: `nc -l -p 10701 | aplay -f S16_LE -r 22050 -c 1 -t raw` (dmix lets it coexist with stock stack).
- Mac: ~100-line asyncio TCP server → VAD → wake word → utterance buffer → STT. Wyoming framing only if HA interop wanted later.

## 3. ASR/STT on Mac

| Engine | License / Apple Silicon | Russian |
|---|---|---|
| **whisper.cpp** | MIT; Metal GPU, Core ML encoder on ANE; built-in Silero VAD; `pywhispercpp` wheels CPU-only (build for Core ML). VERIFIED https://github.com/ggml-org/whisper.cpp | large-v3 RU ≈9.8 WER CV17; turbo close, more hallucination on short clips (secondary) |
| **mlx-whisper** 0.4.3 | MIT, pip-only, native MLX GPU, needs ffmpeg; `mlx-community/whisper-large-v3-turbo` 1.61 GB. VERIFIED https://pypi.org/project/mlx-whisper/ | same models; ≈ whisper.cpp speed on M4 |
| faster-whisper 1.2.1 | MIT; CPU int8 only on Mac | same |
| Apple SpeechAnalyzer (macOS 26) | on-device Swift; third-party tests: **no Russian model** in new engine (ASSUMED) | check `SpeechTranscriber.supportedLocales` |
| **Vosk** | Apache; `vosk-model-small-ru-0.22` 45 MB, `vosk-model-ru-0.42` 1.8 GB. VERIFIED https://alphacephei.com/vosk/models | streaming, worse than Whisper on hard audio |
| Yandex SpeechKit | ~$0.00133/15 s. VERIFIED https://aistudio.yandex.ru/docs/en/speechkit/pricing | best RU, but audio leaves house |
| OpenAI / Deepgram | cloud | ASSUMED |

**Recommendation:** `mlx-whisper` + `whisper-large-v3-turbo`; explicit `language` or auto-detect; Silero VAD gate. Fallback whisper.cpp (Core ML) or Vosk small-ru.

## 4. TTS on Mac

| Engine | License / status | Russian |
|---|---|---|
| **macOS `say`** | built-in; Milena, Milena (Enhanced), Katya (Enhanced), Yuri; `say -v "Milena (Enhanced)" -o out.aiff` | acceptable, not great (ASSUMED). Locally only plain "Milena" installed as of today |
| **Piper** `piper-tts` 1.8.0 (Sep 2026) | **GPL-3.0**, macOS arm64 wheels; `ru_RU` denis/dmitri/irina/ruslan. VERIFIED https://pypi.org/project/piper-tts/ | poor stress placement reported (ASSUMED) |
| **Silero TTS** `v5_6_ru` | models **CC BY-NC 4.0**, code MIT; PyTorch ≥2.0; auto stress, SSML; RTF ≈0.04 CPU. VERIFIED https://github.com/snakers4/silero-models | **best open Russian prosody** |
| Kokoro | no Russian base; community `kokoro-ru` | experimental |
| XTTS-v2 | non-commercial license, heavy | skip |
| ElevenLabs / OpenAI TTS | cloud | text leaves house |

**Recommendation:** Silero `v5_6_ru` for Russian + `say` for English; simplest fallback `say` for both. Cache common phrases.

## 5. LLM

Current models (VERIFIED https://platform.claude.com/docs/en/about-claude/models/overview): `claude-fable-5-1` ($10/$50), `claude-opus-5-5` ($4/$20), `claude-sonnet-5-5` ($2/$10), **`claude-haiku-5-5`** ($0.10/$0.50, fastest). All 1M context, adaptive thinking.

- Voice turn: `claude-haiku-5-5`, `output_config.effort="low"`, streaming (start TTS on first sentence); escalate to `claude-sonnet-5-5` for tool-heavy turns.
- **Memory tool** `{"type": "memory_20250818", "name": "memory"}` — **stable, no beta header**; client-side files under `/memories`; Python SDK ships `BetaLocalFilesystemMemoryTool(base_path=...)` + `client.beta.messages.tool_runner(...)`. VERIFIED https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool . Simplest persistent memory: directory of text files.
- Context editing beta `context-management-2025-06-27`; compaction beta `compact-2026-01-12` (ASSUMED from skill docs).
- **Claude Agent SDK** runs Claude Code as subprocess; heavier; use only when the assistant must edit files / run shell (i.e. M6). VERIFIED https://code.claude.com/docs/en/agent-sdk/python.md .

**Recommendation:** Messages API + `tool_runner` with memory tool + own `weather`/`reminder`/`telegram` tools.

## 6. Weather

- **Open-Meteo**: no key, free non-commercial <10k/day, CC-BY 4.0, global models, geocoding API. VERIFIED https://open-meteo.com/en/terms .
- OpenWeatherMap One Call 3.0: 1000/day free, **card required**. VERIFIED https://openweathermap.org/price .
- Yandex Weather API: free test **50 req/day**; enterprise ~540k ₽/yr+. VERIFIED https://yandex.ru/dev/weather/doc/en/concepts/pricing .
- wttr.in: hobby reliability (ASSUMED).

**Recommendation:** Open-Meteo via `urllib`. Fallback wttr.in.

## 7. Scheduler + store

`sqlite3` + asyncio sleep-until-next-job loop; launchd `StartCalendarInterval` plist for morning briefing (absolute `.venv/bin/python`). APScheduler 4.x still alpha; unnecessary.

## 8. Web browse/scrape

Anthropic **server tools**: `web_search_20250305` / `_20260209` / `_20260318` — stable, no beta header, $10 per 1000 searches; `web_fetch_20250910` / `_20260209` / `_20260309` / `_20260318` — no extra charge, only URLs already in conversation, no JS, `max_content_tokens`, `allowed_domains`. VERIFIED https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-search-tool , https://platform.claude.com/docs/en/agents-and-tools/tool-use/web-fetch-tool . Local minimal: `urllib` + `html.parser`; `trafilatura` 2.1.0 for readability; Playwright for JS only.

**Recommendation:** `web_search` + `web_fetch` server tools with `max_uses`; trafilatura only if needed.

## 9. Telegram

Bot API 10.3 (2026-08-24); `getUpdates` long polling. VERIFIED https://core.telegram.org/bots/api . `python-telegram-bot` 22.8 (LGPL), `aiogram` 3.31 (MIT).

**Recommendation:** stdlib `urllib` long-poll loop + `sendMessage`. Fallback python-telegram-bot.

## Recommended minimal stack

| Component | Choice | Why | Fallback |
|---|---|---|---|
| Device capture/playback | static `arecord` / BusyBox `nc` → TCP; `aplay` downlink | ALSA + BusyBox only | tiny static C/Go daemon; tap `audiod` VQE |
| Transport | raw 16 kHz S16LE mono over TCP, asyncio server on Mac | simplest, LAN-only | Wyoming framing |
| VAD | Silero VAD | cuts CPU, avoids hallucination | energy threshold |
| Wake word | `pymicro-wakeword` on Mac | device can't host; Porcupine gone | openWakeWord |
| STT | `mlx-whisper` + large-v3-turbo | pip-only, Metal, RU+EN | whisper.cpp; Vosk |
| TTS | Silero v5_6_ru + `say` | best RU prosody; zero-dep EN | Piper; `say` Milena |
| LLM | Messages API, haiku-5-5 low effort, streaming | fastest/cheapest | sonnet-5-5 |
| Memory | memory tool + `BetaLocalFilesystemMemoryTool` | stable, plain files | sqlite notes |
| Web | `web_search_*` + `web_fetch_*` | no scraper to maintain | urllib + trafilatura |
| Weather | Open-Meteo | no key | wttr.in |
| Scheduler | sqlite3 + asyncio loop; launchd | stdlib | APScheduler 3 |
| Telegram | urllib long polling | stdlib | python-telegram-bot |

## External services receiving data under this stack

1. **Anthropic** — transcribed text of post-wake utterances, system prompt, tool results, memory file contents when viewed, web search queries / fetched text. No audio.
2. **Anthropic web search backend / fetched websites** — queries and URLs.
3. **Open-Meteo** — approximate lat/lon, per-IP.
4. **Telegram** — all bot chat messages.
5. **Hugging Face / GitHub** — one-time model downloads.
6. **Yandex** — only while `aliced`/`updatesd`/telemetry still run on the speaker (ASSUMED).
Wake word, VAD, STT, TTS, memory storage: local.

## Sources

See inline URLs. Device: https://hackalice.me/devices/station-mini3/ . Wake: openWakeWord, pymicro-wakeword, HA Picovoice thread. Transport: wyoming-satellite, wyoming, linux-voice-assistant, snapcast. STT: whisper.cpp, pywhispercpp, mlx-whisper, faster-whisper, Vosk, SpeechKit pricing. TTS: piper-tts, piper1-gpl, piper-voices, silero-models, kokoro-ru, XTTS-v2. LLM: models overview, memory-tool, web-search-tool, web-fetch-tool, agent-sdk/python. Weather: open-meteo terms, OWM price, Yandex Weather pricing. Scheduler: APScheduler 4.0.0a6, launchd guide. Telegram: bots/api, python-telegram-bot, aiogram.
