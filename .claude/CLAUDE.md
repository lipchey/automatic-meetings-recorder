# automatic-meetings-recorder — Claude preload

## Routing
- Start every session from `.claude/memory/status.md` (living cursor). When it
  disagrees with `docs/plans/roadmap/`, the roadmap wins.
- Map of all docs: `docs/README.md`. Rules for writing them: `docs/CONVENTIONS.md`.
- Settled decisions: `.claude/memory/decisions.md` — append, never rewrite.

## What this repo is
Local-only pipeline that records online meetings on the Mac, transcribes them
with an open-source ASR model (Parakeet TDT 0.6b v3 via `parakeet-mlx`),
summarizes with the Claude API and files the result as Markdown into the
Obsidian vault. No VPS, no n8n. Public repo: never commit secrets, vault
paths or recordings — config lives in `.env` (see `.env.example`).

## Memory discipline
- `status.md` is a snapshot of the active front, blockers and next action —
  state + pointer to the fact's owner, never a history.
- A decision goes to `decisions.md` as `### YYYY-MM-DD — <topic>` with the
  options that lost and why.
- An open question the user must answer is a ⛔ gate line (format in
  `docs/CONVENTIONS.md` §Gates); a session reaching gated work stops and asks.

## Code comments
Keep only comments that answer "which bug or wrong fix returns if this is
deleted?" — no header retellings, no dates, no plan/PR ids.

## Delivery rules
See `AGENTS.md` §Delivery. Non-trivial change → Gate C (codex-chain) before
commit; non-trivial plan → Gate P (codex-plan-review) before execution.
