# Project routing

Living cursor: `.claude/memory/status.md` — read it first, every session.
Documentation map: `docs/README.md`. Documentation rules: `docs/CONVENTIONS.md`.
Decisions: `.claude/memory/decisions.md`. Plans: `docs/plans/`. Specs: `docs/specs/`.

# Delivery

Solo repo. Feature work = linked worktree under `../worktrees/<repo>-<branch>` +
PR to `main`; small maintenance (docs, status, memory) commits directly to `main`.
Commit messages in English, no AI attribution trailers.

# Verification

`scripts/verify` (added with the first code unit) is the one entry point; until
it exists, "verified" means the acceptance evidence named in the plan unit.
