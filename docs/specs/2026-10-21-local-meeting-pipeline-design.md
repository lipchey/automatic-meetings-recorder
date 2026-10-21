Status: draft (Gate P + Astra consult folded in 2026-10-21; owner approval pending)
ID namespace: none

# Local meeting pipeline: record → Parakeet → Claude API → Obsidian

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
         └─ on stop: hs.task → bin/meeting-pipeline <base> <title> <start> <end> <mode>
                       (executable; sets PATH, loads .env, exec caffeinate -i uv run --script pipeline/process.py …)
                 1. mode=mic → mic track; mode=mix → ffmpeg amix+dynaudnorm → 16 kHz mono wav   [as today]
                 2. parakeet_mlx (v3, 120 s chunks / 15 s overlap) → <base>.transcript.json      [replaces MacWhisper + language probe]
                 3. Claude API, one call, structured output {title, summary_md}
                 4. write <base>.md, place as <INBOX>/<YYYY-MM-DD HH-MM> <title>.md (one stable name per base)
                 5. screen .mov + audio → <base>.screen.mp4                                        [as today]
                 6. touch <base>.done only when 3–5 all succeeded; notify (Hammerspoon ✅/❌ as today)
   Re-run / recovery: `process.py --sweep` finishes every base without .done, skipping stages
   whose artifact exists. Not auto-invoked; Hammerspoon's sweep notifies + shows the command.
```

Owner decisions (2026-10-21, `.claude/memory/decisions.md`): Parakeet V3; no VPS;
Obsidian-only output; Claude API; public repo; diarization only if low-friction (→ deferred units 2.x).

Author's calls (mine; owner may override):
- **One Python file** `pipeline/process.py` with PEP 723 inline deps
  (`anthropic`, `parakeet-mlx`) + committed uv lock, run via `uv run --script`
  from the executable launcher `bin/meeting-pipeline` (Hammerspoon's `hs.task`
  takes an executable + argv, not a shell string). Same argv contract as today's
  `transcribe-and-post.sh`. Why Python, not bash: ASR is a Python library;
  in-process avoids re-loading the model and parsing CLI output.
- **Config via `.env`** (gitignored; `.env.example` committed): `ANTHROPIC_API_KEY`,
  `SUMMARY_MODEL` (default `claude-sonnet-5-5`), `OBSIDIAN_INBOX`, `RECORDINGS_DIR`.
  No other config surface.
- **Summary + title in one Claude call** with structured output; the prompt keeps
  today's contract (overview 2–4 sentences, `## Key points`, `## Decisions`,
  `## Action items` as `- [ ]`, transcript's own language, never invent).
- **Durable stages, truthful completion**: `<base>.transcript.json` persists the
  ASR result; `<base>.md` is written beside the recording before it is placed in
  the inbox; `.done` exists only when note, summary and mux all succeeded. API
  failure → `summary: pending`, non-zero exit, ❌ notification; `process.py --sweep`
  finishes pending bases idempotently. Hammerspoon's existing sweep keeps
  notifying with the command (verified: it never re-ran jobs). `prune` must not
  delete a base without `.done` before 90 days.
- **One stable note name per base**: `<YYYY-MM-DD HH-MM> <title>.md` — the time
  prefix prevents same-day same-title collisions; a re-run renames, never duplicates.
- **Diarization is not on the release path.** Deferred 2.1 = "Me / Others" from
  the separate mic/sys tracks (zero deps); 2.2 = Senko experiment on copies.
- **All paths from one root**: `install.sh` sets `REPO`; Lua, scripts and the
  launcher derive binaries, prune and launcher paths from it; `RECORDINGS_DIR`,
  `OBSIDIAN_INBOX` from `.env`. No `~/Scripts` literals.
- **Live activation happens once, at cutover**, with a backup and a tested
  `--rollback`; everything before runs on copies of recordings.
- **Sleep**: `caffeinate -i` against idle sleep; lid close pauses the job,
  `--sweep` finishes it after wake (no queue daemon).
- **Markdown layout**: YAML frontmatter (`date`, `app`, `started`, `ended`,
  `duration_min`, `model`, `summary: done|pending`, `recording: <base>`), then
  summary, then `## Transcript` as `[hh:mm:ss] text` lines (speaker labels only
  once 2.x lands). The
  ```tabs``` layout of the current n8n export is dropped (its summary tabs were
  always empty).

## 6 Left to the builder
- Structured-output schema and the anthropic SDK call shape (read the `claude-api`
  skill; do not write from memory); `stop_reason` refusal/max_tokens handling;
  `count_tokens` before the call.
- Explicit `chunk_duration=120, overlap_duration=15` on the Python API (its
  default is unchunked — the spike's numbers came from the CLI defaults).
- Whether `--local-attention` is needed on 24 GB for 2–3 h recordings (measure).
- `install.sh` details (symlink vs copy of `init.lua`, codesign of meetrec.app
  keeping bundle id `com.lipchey.meetrec` so TCC grants survive).

## 7 Acceptance for the whole design
A real meeting recorded by the usual Hammerspoon flow ends; within ~5 minutes the
`.md` with summary and timestamped transcript is in the inbox; nothing was sent to
the VPS; `git log -p` finds no secret or personal path; a fresh macOS user account
reproduces a manual recording → note from `README.md` with `install.sh` + `.env`;
the README states plainly what leaves the Mac (transcript text → Claude API;
notes → a Google-Drive-backed vault).
