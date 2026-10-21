Status: draft (awaiting owner approval after Gate P + Astra consult)
ID namespace: none

# Local meeting pipeline: record → Parakeet → Senko → Claude API → Obsidian

## 0 Intent and why
Сьогоднішній пайплайн (`docs/research/2026-10-21-current-system-baseline.md`)
працює, але має дві зайві ланки: платний MacWhisper, яким ми керуємо через сокет
запущеного GUI-застосунку, і VPS з n8n/Directus/Telegram, який потрібен лише
тому, що колись процесинг писали «в хмарі». Mac наприкінці мітингу і так
увімкнений і має M4 Pro. Мета — один публічний репозиторій, який повністю
відтворює пайплайн локально, читається за 10 хвилин і стає основою YouTube-відео.

## 1 Goals
- Після кінця дзвінка, без участі людини, в Obsidian inbox з'являється
  `<Назва> <YYYY-MM-DD>.md` із саммарі (overview, key points, decisions, action
  items) і повним транскриптом з таймстемпами і (якщо увімкнено) мітками спікерів.
- Жодної платної десктоп-програми, жодного сервера. Єдина хмарна залежність —
  Claude API для саммарі (≈ $0.003–0.10 за мітинг залежно від моделі).
- Один `install.sh`, один `.env`, один скрипт пайплайну.

## 2 Non-goals (v1)
- CRM: запис у Directus, матчинг клієнт/проєкт через Google Calendar.
- Тріаж Work/Obsidian/Discard (Telegram або локальний діалог) — усе летить в inbox.
- Fallback ASR на whisper — лише якщо acceptance Parakeet провалиться (deferred unit).
- Переписування рекордера: Hammerspoon + meetrec.app переносяться як є.
- Сценарій відео — окреме завдання після коду.

## 3 Anti-goals
- Секрети, шляхи машини або записи в git (репо публічне).
- Два паралельні шляхи обробки (новий і старий) після cutover.
- Абстракції «на виріст»: плагіни бекендів, черги, демони.

## 4 Constraints (verified 2026-10-21)
- macOS 15.7, Apple M4 Pro, 24 GB. `uv`, `ffmpeg`, `parakeet-mlx 0.5.3`
  (модель `mlx-community/parakeet-tdt-0.6b-v3` у HF-кеші), `mlx_whisper` стоять.
- Obsidian vault змонтований локально через Google Drive for desktop:
  `…/My Drive/ObsidianVault/00-Inbox/transcriptions` — звичайна папка на диску.
- Parakeet v3: 25 мов включно з uk/ru/en, автодетект мови, але code-switching
  всередині файлу офіційно «не повністю натренований»; спайк показав −40 % слів
  на побутовій ru/uk розмові і паритет на бізнес-дзвінку uk/en
  (`docs/research/2026-10-21-parakeet-spike.md`).
- `ANTHROPIC_API_KEY` на машині не заданий.

## 5 Architecture (the parts the owner decided + the author's calls)

```
Hammerspoon (mic-usage poll, unchanged)
   └─ meetrec.app  →  ~/Recordings/<base>.{mic,sys}.m4a [+ .screen.mov]
         └─ on stop: uv run --env-file .env pipeline/process.py <base> <title> <start> <end> <mode>
                 1. ffmpeg amix+dynaudnorm → <base>.wav (16 kHz mono)        [as today]
                 2. parakeet_mlx → sentences with timestamps                  [replaces MacWhisper + language probe]
                 3. senko (if DIARIZE=1) → speaker turns; sentence ← max-overlap speaker
                 4. Claude API, one call, structured output {title, summary_md}
                 5. write <INBOX>/<title> <date>.md ; touch <base>.done
                 6. screen .mov + audio → <base>.screen.mp4                  [as today]
                 7. notify (Hammerspoon shows ✅/❌ as today)
```

Owner decisions (2026-10-21, `.claude/memory/decisions.md`): Parakeet V3; no VPS;
Obsidian-only output; Claude API; public repo; diarization if low-friction.

Author's calls (mine; owner may override):
- **One Python file** `pipeline/process.py` with PEP 723 inline deps
  (`anthropic`, `parakeet-mlx`, `senko`), run via `uv run --script`. Same argv
  contract as today's `transcribe-and-post.sh`, so the Hammerspoon change is one
  path. Why Python, not bash: ASR and diarization are Python libraries; calling
  them in-process avoids re-loading the model and parsing CLI output.
- **Config via `.env`** (gitignored; `.env.example` committed): `ANTHROPIC_API_KEY`,
  `SUMMARY_MODEL` (default `claude-sonnet-5-5`), `OBSIDIAN_INBOX`, `RECORDINGS_DIR`,
  `DIARIZE`. No other config surface.
- **Summary + title in one Claude call** with structured output; the prompt keeps
  today's contract (overview 2–4 sentences, `## Key points`, `## Decisions`,
  `## Action items` as `- [ ]`, transcript's own language, never invent).
- **Offline/API failure is not a loss**: the `.md` is still written with the
  transcript and frontmatter `summary: pending`; re-run is the same command.
  Hammerspoon's existing leftover sweep re-runs any `<base>` without `.done`.
- **Diarization = Senko** (Core ML, no HF token, ~8 s/hour), speaker per
  sentence by greatest overlap; generic `Speaker 1..N` labels, no enrollment.
  Scribe (Trail of Bits) is the fallback integration if the Senko merge exceeds
  ~40 LOC.
- **Sleep**: wrap the run in `caffeinate -i`; lid close pauses the job, it
  resumes on wake (no queue daemon).
- **Markdown layout**: YAML frontmatter (`date`, `app`, `started`, `ended`,
  `duration_min`, `model`, `summary: done|pending`, `recording: <base>`), then
  summary, then `## Transcript` as `[hh:mm:ss] Speaker N: text` lines. The
  ```tabs``` layout of the current n8n export is dropped (its summary tabs were
  always empty).

## 6 Left to the builder
- Exact senko API calls and the overlap merge.
- Structured-output schema and the anthropic SDK call shape (read the `claude-api`
  skill; do not write from memory).
- Whether `--local-attention` is needed on 24 GB for 2–3 h recordings (measure).
- `install.sh` details (symlink vs copy of `init.lua`, codesign of meetrec.app
  keeping bundle id `com.lipchey.meetrec` so TCC grants survive).

## 7 Acceptance for the whole design
A real meeting recorded by the usual Hammerspoon flow ends; within ~5 minutes the
`.md` with summary and labelled transcript is in the inbox; nothing was sent to
the VPS; `git grep` finds no secret; the YouTube viewer can reproduce it from
`README.md` with `install.sh` + `.env`.
