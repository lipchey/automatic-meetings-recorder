Status: draft (Gate P + Astra pending)
ID namespace: none
Roadmap rows: `docs/plans/roadmap/01-readiness.md`

# Local pipeline migration: Parakeet, Senko and Claude API replace MacWhisper and the VPS

## 0 Why / scope
Executes `docs/specs/2026-10-21-local-meeting-pipeline-design.md`. Scope: new repo
content (recorder import, one pipeline script, install, README), cut-over of the
Hammerspoon hand-off, retirement of the MacWhisper/VPS path for this flow. Out of
scope: CRM, triage, video script, changes inside the `personal-n8n` repo (only a
follow-up note).

## Baseline (verified 2026-10-21)
- Current flow and its file landmarks: `docs/research/2026-10-21-current-system-baseline.md`.
- Tooling present: `uv`, `ffmpeg`, `parakeet-mlx 0.5.3` + cached v3 model, `gh` logged
  in as `lipchey`; `ANTHROPIC_API_KEY` absent; ollama not running (not needed).
- Spike: 80-min call → 65 s ASR, word count on par with MacWhisper
  (`docs/research/2026-10-21-parakeet-spike.md`).
- Obsidian inbox is a local folder (`…/ObsidianVault/00-Inbox/transcriptions`).

## Units

### 1.1 `repo-scaffold` — done 2026-10-21
Docs skeleton, conventions, memory, hooks, research baseline. This commit.

### 1.2 `recorder-import` — draft
Copy as-is into `recorder/`: `~/.hammerspoon/init.lua` → `recorder/hammerspoon/init.lua`,
`~/Scripts/{micusers.swift,meetrec.swift,meetrec-Info.plist,build-meetrec.sh,prune-recordings.sh}`.
Add `install.sh`: builds `micusers` and `meetrec.app` (same bundle id, so existing
TCC grants survive), symlinks `~/.hammerspoon/init.lua` to the repo copy (back up
the original), copies `.env.example` → `.env` if absent. Grep the imported files
for secrets/absolute personal paths before commit; parametrize via `.env` where found.
Acceptance: `./install.sh` on this Mac; Hammerspoon reloads (`hammerspoon-mic.log`
shows the config banner); manual tray start/stop produces `<base>.{mic,sys}.m4a`.

### 1.3 `pipeline-transcribe` — draft
`pipeline/process.py` (uv script, PEP 723 deps): argv contract of
`transcribe-and-post.sh`; mix → parakeet_mlx → write `<INBOX>/<title> <date>.md`
with frontmatter + `## Transcript` (`[hh:mm:ss] text`), `summary: pending`; touch
`<base>.done`; screen mux as today; `caffeinate -i` wrapper. No summary, no
diarization yet. Delete nothing from the old script yet.
Acceptance: run on the two spike recordings → two `.md` in a temp inbox dir
(`OBSIDIAN_INBOX` pointed at scratch), timestamps monotonic, wall time ≤ 2 min
for the 80-min file. One `assert`-based self-check (`python process.py --selftest`)
covering the md writer and the timestamp formatter.

### 1.4 `claude-summary` — draft (⛔ `anthropic-api-key`)
One `anthropic` call, structured output `{title, summary_md}`, model from
`SUMMARY_MODEL`, prompt = today's contract (see spec §5). On any API error: keep
`summary: pending`, log, exit 0 so the md still lands. Log `usage` tokens + est.
cost per run to `~/Library/Logs/meeting-pipeline/<base>.log`.
Acceptance: both spike recordings summarized; Ukrainian output for the uk call;
title ≤ 7 words, filename-safe; measured cost per 80-min call recorded in
`docs/acceptance/`.
⛔ `anthropic-api-key`: невідомо, чи заведено ключ; БЛОКУЄ 1.4; ЗАКРИЄ `ANTHROPIC_API_KEY` у `.env` на цьому Mac; відкрито 2026-10-21.

### 1.5 `diarization` — draft
`senko` behind `DIARIZE=1`; speaker per sentence by greatest overlap; transcript
lines become `[hh:mm:ss] Speaker N: text`. If the merge needs > ~40 LOC, switch to
Trail of Bits Scribe and say so in `decisions.md`.
Acceptance: 80-min call diarized in ≤ 60 s extra; owner spot-checks 5 speaker
turns against audio (receipt in `docs/acceptance/`).

### 1.6 `asr-acceptance` — draft (⛔ `asr-quality-verdict`)
Run the new pipeline on the 5 most recent `~/Recordings` bases that still have a
MacWhisper `.txt`; produce a side-by-side (word counts, 3 random 30-s windows
each, read by the owner). Owner verdict closes the gate.
⛔ `asr-quality-verdict`: невідомо, чи Parakeet-only прийнятний на реальних
uk/en і ru/uk дзвінках; БЛОКУЄ 1.7 (cutover); ЗАКРИЄ рішення власника після
side-by-side; якщо «ні» → активувати deferred `whisper-fallback`; відкрито 2026-10-21.

### 1.7 `cutover` — draft
Hammerspoon `POSTER` → `uv run … pipeline/process.py`; remove webhook/outbox code
from the repo copy; unload dead launch agents `com.admin.movetranscriptions`,
`com.admin.whisperautomove`; add a follow-up note for `personal-n8n` to deactivate
`macwhisper_transcript_triage`. MacWhisper stays installed until the owner
removes it. Acceptance: first real meeting end-to-end → `.md` in the real inbox,
`hammerspoon-mic.log` shows ✅, nothing hit the VPS (n8n executions list empty for
that hour).

### 1.8 `public-readme` — draft
Root `README.md`: what it does, 1 diagram, requirements, `install.sh`, `.env`,
cost table, how to re-run a recording, limitations (code-switching, lid close).
Acceptance: a fresh reader follows it on this Mac without the docs folder.

### Deferred
- `whisper-fallback` — `ASR_BACKEND=whisper` via `mlx_whisper` large-v3; only if 1.6 fails.
- `crm-directus`, `telegram-triage` — owner decided out of v1.

## Order and start conditions
1.2 → 1.3 → 1.4 (needs gate closed) ∥ 1.5 → 1.6 → gate → 1.7 → 1.8.
1.3 and 1.5 are file-disjoint from 1.2 only after 1.2 lands (shared `.env.example`).
Branch: `local-pipeline-migration` in a worktree; one PR per unit or one PR for
1.2–1.5 if the owner prefers fewer reviews.

## Risks
- Code-switching quality (spike −40 % words on casual ru/uk) — covered by 1.6 + deferred fallback.
- TCC: rebuilt `meetrec.app` must keep bundle id and signing identity or the
  Screen Recording / Microphone grants are re-asked — 1.2 verifies.
- Google Drive for desktop sync lag or conflict on the `.md` — write once, atomic rename.
- Lid closed right after the call → job pauses until wake (accepted).

## Decisions / gates / open questions
- Gates: `anthropic-api-key` (1.4), `asr-quality-verdict` (1.7). Formats per
  `docs/CONVENTIONS.md` §6.
- Default `SUMMARY_MODEL=claude-sonnet-5-5` is the author's assumption
  (`decisions.md` 2026-10-21 — Саммарі); cost table in the spec.
