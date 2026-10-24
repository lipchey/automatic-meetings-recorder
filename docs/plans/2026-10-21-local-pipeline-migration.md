Status: active (Gate P 2026-10-21; owner decisions 2026-10-24 folded in; execution pending)
ID namespace: none
Roadmap rows: `docs/plans/roadmap/01-readiness.md`

# Local pipeline migration: Parakeet and Claude API replace MacWhisper and the VPS

## 0 Why / scope
Executes `docs/specs/2026-10-21-local-meeting-pipeline-design.md`. Scope: new repo
content (recorder import, one pipeline script, launcher, install, verify, README),
shadow validation against real meetings, a single cut-over of the Hammerspoon
launcher, retirement of the MacWhisper/VPS path for this flow. Out of scope: CRM,
triage, video script, edits inside `personal-n8n` (follow-up note only),
diarization (deferred unit 2.1).

## Baseline (verified 2026-10-21)
- Current flow and landmarks: `docs/research/2026-10-21-current-system-baseline.md`.
  Facts the plan rests on, read in the live files: `hs.task.new(POSTER, cb, args)` —
  POSTER is an executable path, five args passed separately; `sweepLeftovers()` only
  NOTIFIES (lists recent `.mix.wav`/outbox, suggests the re-run command); binaries
  and prune are launched from a hardcoded `~/Scripts`; `build-meetrec.sh` signs
  ad-hoc (its own header warns TCC may re-ask); `prune-recordings.sh` deletes by age
  only; the screen mux overwrites `.screen.mp4` beside the recording.
- `parakeet_mlx` Python API defaults `chunk_duration=None` (whole file in one pass);
  the CLI used in the spike passes 120 s / 15 s overlap.
- Tooling present: `uv`, `ffmpeg`, `parakeet-mlx 0.5.3` + cached v3 model, `gh`
  logged in as `lipchey`; `ANTHROPIC_API_KEY` absent.
- Spike (`docs/research/2026-10-21-parakeet-spike.md`): 80-min uk/en call → 65 s,
  words on par with MacWhisper; 2.5-min casual ru/uk call → −40 % words.
- Obsidian inbox is a local folder (`…/ObsidianVault/00-Inbox/transcriptions`).

## Units

### 1.1 `repo-scaffold` — done 2026-10-21
Docs skeleton, conventions, memory, hooks, research baseline.

### 1.2 `recorder-import` — draft
Copy into `recorder/`: `~/.hammerspoon/init.lua` → `recorder/hammerspoon/init.lua`,
`~/Scripts/{micusers.swift,meetrec.swift,meetrec-Info.plist,build-meetrec.sh,prune-recordings.sh}`.
Edits allowed in the copy, nothing installed live:
- every path the Lua/scripts resolve (`micusers`, `meetrec.app`, prune, POSTER,
  RECDIR) comes from ONE root `REPO` + `RECORDINGS_DIR`; no `~/Scripts` literal remains;
- `build-meetrec.sh` builds into `recorder/bin/`; `install.sh --build` builds
  there and pre-warms `uv run --script pipeline/process.py --selftest` (deps + model
  download) — the Hammerspoon activation step is a SEPARATE flag used only in 1.7;
- sweep stale references to `transcribe-and-post.sh`/MacWhisper in comments and
  notification strings; comments follow the repo comment rule;
- UI strings (dialogs, notifications, menu titles) move into one Lua table with
  `uk` and `en` entries, selected by `UI_LANG` from `.env` (owner 2026-10-24:
  default `uk`); code and comments in English;
- grep the copy for secrets/absolute personal paths; `.env.example` lists every knob.
Acceptance: `install.sh --build` on this Mac produces `recorder/bin/{micusers,meetrec.app}`;
`codesign -dv` shows bundle id `com.lipchey.meetrec`; `diff` of the imported Lua vs
live shows ONLY the path-root and string changes (receipt in `docs/acceptance/`).
Live `~/.hammerspoon/init.lua` untouched (checksum before/after).

### 1.3 `pipeline-transcribe` — draft
`pipeline/process.py` (uv script, PEP 723 deps + committed `uv` lock, `requires-python`
pinned) with argv contract `<base> <title> <startedISO> <endedISO> <mode>`:
- `mode=mic` → mic track only; `mode=mix` → amix+dynaudnorm of mic+sys (as today);
- `parakeet_mlx` with explicit model `mlx-community/parakeet-tdt-0.6b-v3`,
  `chunk_duration=120, overlap_duration=15`;
- durable stage artifacts beside the recording: `<base>.transcript.json` after ASR,
  `<base>.done` ONLY after note + mux both succeeded; re-running the same base skips
  stages whose artifact exists;
- note written to `<base>.md` beside the recording first, then placed into
  `OBSIDIAN_INBOX` under ONE stable name per base: `<YYYY-MM-DD HH-MM> <title>.md`
  (time prefix = uniqueness; title = app name until a summary exists). The chosen
  inbox path is recorded in `<base>.transcript.json`; a later run renames, never
  duplicates; inbox missing → keep the local `.md`, exit non-zero with a clear log line;
- `--sweep`: process every base in `RECORDINGS_DIR` with recordings but no `.done`
  (idempotent). Not auto-invoked (a permanently broken file must not loop) —
  Hammerspoon's sweep keeps today's behaviour: notify + show the command.
- screen mux as today, after the note;
- `bin/meeting-pipeline`: the executable Hammerspoon launches — absolute paths, sets
  PATH, loads `.env`, `exec caffeinate -i uv run --script …`.
- `scripts/verify`: `uv run pipeline/process.py --selftest` (asserts: timestamp
  formatter, md writer, stable-name/rename, sweep selection, argv parsing with
  `mode`), `bash -n` on shell files, `luac -p` on the Lua.
Acceptance: `scripts/verify` exit 0; run on COPIES of the two spike recording families
in a temp `RECORDINGS_DIR` with a temp `OBSIDIAN_INBOX` → two notes, timestamps
monotonic, 80-min file ≤ 2 min wall, originals byte-identical; kill the process
mid-ASR and re-run → completes without re-doing finished stages.

### 1.4 `asr-acceptance` — draft (⛔ `asr-quality-verdict`)
Frozen cohort (copies, never originals): `meet-20260912-100235` (uk/en business),
`meet-20260905-112702` (ru/uk casual, the known −40 % case), plus the three most
recent other bases with a MacWhisper `.txt` as of this date, listed in the receipt.
For each: word counts, 3 fixed 30-s windows chosen to contain names, numbers,
negations and a language switch; the owner LISTENS to the audio for those windows
and marks Parakeet / MacWhisper / both wrong. Missing coverage = gate stays open.
Threshold (owner 2026-10-24): business uk/en calls must be at least as usable as
the MacWhisper transcript in the listened windows; personal ru/uk casual calls are
best-effort and cannot fail the gate on their own.
⛔ `asr-quality-verdict`: невідомо, чи Parakeet-only проходить поріг (бізнес uk/en)
на cohort; БЛОКУЄ 1.5–1.7; ЗАКРИЄ рішення власника на receipt у `docs/acceptance/`;
«ні» → активувати deferred `whisper-fallback` перед 1.5; відкрито 2026-10-21.

### 1.5 `claude-summary` — draft (⛔ `anthropic-api-key`)
One `anthropic` call in `process.py`: structured output `{title, summary_md}`,
model from `SUMMARY_MODEL`, prompt = today's contract (spec §5) + explicit summary
language rule for mixed calls; check `stop_reason` (refusal / max_tokens) and treat
both as failure; `count_tokens` before the call, reserve output; on any failure keep
`summary: pending`, no `.done`, exit non-zero with the log line. Log `usage` + cost
per run. `scripts/verify` gains the API-failure path (mocked client).
Acceptance: both spike copies summarized; Ukrainian output for the uk call; title
≤ 7 words, filename-safe; measured cost per 80-min call in `docs/acceptance/`.
⛔ `anthropic-api-key`: невідомо, чи заведено ключ; БЛОКУЄ 1.5; ЗАКРИЄ успішний
автентифікований виклик обраної моделі зі схемою (не лише наявність змінної);
відкрито 2026-10-21.

### 1.6 `shadow-run` — draft
Old path stays authoritative and untouched. After each of 3–5 real meetings the
owner (or `process.py --sweep` with `RECORDINGS_DIR` = a copy dir) runs the new
pipeline into a scratch inbox; compare note vs the VPS summary for decisions, owners,
names. Receipt lists the meetings, wall time, cost, peak memory (`/usr/bin/time -l`)
and one ≥ 2 h recording if available.
Acceptance: receipt in `docs/acceptance/`; owner says "switch".

### 1.7 `cutover` — draft
Only when idle (no recording, no processing). Steps, each with its rollback:
1. `install.sh --activate`: backs up `~/.hammerspoon/init.lua` to a dated file,
   writes a 2-line stub (`REPO=…; dofile(REPO.."/recorder/hammerspoon/init.lua")`),
   reloads Hammerspoon. Rollback: `install.sh --rollback` restores the backup + reload.
2. TCC: if macOS re-asks Screen Recording / Microphone for the rebuilt app, grant;
   verify with a manual tray recording BEFORE the next real meeting.
3. Unload dead launch agents `com.admin.movetranscriptions`, `com.admin.whisperautomove`
   (`launchctl bootout`; plists kept for rollback).
4. `prune-recordings.sh`: skip bases without `.done` unless older than 90 days.
5. Follow-up note (not done here): deactivate `macwhisper_transcript_triage` in
   `personal-n8n`; remove the meeting files from `~/Scripts` and archive
   `~/global-tools/call-recording/MEMORY.md` with a pointer to this repo.
Acceptance: first real meeting end-to-end → note in the real inbox,
`hammerspoon-mic.log` shows the new launcher path and ✅; no webhook code in the
repo; rollback rehearsed once (restore → reload → restore back) and logged.

### 1.8 `public-readme` — draft
Root `README.md` + `LICENSE` (+ model/dep notices: Parakeet CC-BY-4.0): what it
does, 1 diagram, **what leaves the Mac** (transcript text → Claude API; notes →
a Google-Drive-backed vault), opt-in app list and the recording indicator,
requirements, `install.sh`, `.env`, cost table, re-run/`--sweep`, limitations
(code-switching, lid close pauses, ad-hoc signing). Pre-push: `git log -p` grep for
secrets/paths across history; `.gitignore` covers notes, transcripts, logs, editor state.
Acceptance: a fresh macOS user account on this Mac follows README only → a manual
recording is transcribed into its own inbox.

### Deferred
- `2.1 channel-diarization` — owner 2026-10-24: chosen. "Me / Others" labels from
  the separate mic/sys tracks (mix mode only), energy comparison per sentence, no
  new dependency; transcript lines become `[hh:mm:ss] Я|Співрозмовник: text`
  (labels follow `UI_LANG`). After 1.7.
- `2.2 senko-diarization` — not chosen; kept only as a note for multi-party calls.
- `whisper-fallback` — `ASR_BACKEND=whisper` via `mlx_whisper` large-v3; only if 1.4 fails.
- `crm-directus`, `telegram-triage` — owner decided out of v1.

## Order and start conditions
Strictly serial (all code units touch `process.py` or `recorder/`):
1.2 → 1.3 → 1.4 (gate) → 1.5 (gate) → 1.6 → 1.7 → 1.8 → 2.x.
Branch `local-pipeline-migration` in a worktree; one PR per unit. Nothing in 1.2–1.6
changes the live Mac configuration.

## Risks
- Code-switching quality — 1.4 cohort includes the known failing sample.
- TCC re-grant after ad-hoc re-sign — 1.7 step 2 verifies before a real meeting.
- Google Drive mount absent → local `.md` kept, non-zero exit, Hammerspoon shows ❌.
- Lid closed right after the call → job pauses until wake; `--sweep` finishes it.
- Same-day same-title meetings — time-prefixed stable names.

## Decisions / gates / open questions
- Gates: `asr-quality-verdict` (1.4→1.5), `anthropic-api-key` (1.5). Format
  `docs/CONVENTIONS.md` §6.
- Default `SUMMARY_MODEL=claude-sonnet-5-5` — author's assumption; cost table in spec.
- Owner 2026-10-24: ASR threshold = business uk/en calls (1.4); repo language =
  English code/README, UI strings uk/en via `UI_LANG`; diarization = 2.1 channel-based.
- Open (non-blocking for v1): a local offline summarizer option for viewers on
  16 GB Mac / Windows — research `docs/research/2026-10-24-local-llm-summarizer-options.md`.
  If adopted it becomes deferred unit `2.3 local-summarizer` (`SUMMARY_BACKEND=
  claude|ollama`, same prompt contract), documented in 1.8 as the offline path.
- Gate P 2026-10-21 (Astra, xhigh): 12 findings, all VALID, folded in above
  (launcher executable, stable names, chunking, recovery stages, install root,
  late activation + rollback, copies for acceptance, serial order, frozen cohort,
  no Scribe hatch, `scripts/verify`, comment sweep).

## Execution prompt (paste into a fresh session)

```
Execute docs/plans/2026-10-21-local-pipeline-migration.md in
/Users/admin/Projects/automatic-meetings-recorder, starting at unit 1.2.
Read first: .claude/memory/status.md, docs/specs/2026-10-21-local-meeting-pipeline-design.md,
the plan, docs/research/2026-10-21-current-system-baseline.md. Base: origin/main.
Worktree: /Users/admin/Projects/worktrees/automatic-meetings-recorder-local-pipeline-migration
(branch local-pipeline-migration). One PR per unit; Gate C before each commit.
Units 1.2–1.6 must not change the live Mac config (~/.hammerspoon, ~/Scripts) —
work on copies of recordings. Stop at ⛔ gates and ask the owner. Ponytail mode.
```
