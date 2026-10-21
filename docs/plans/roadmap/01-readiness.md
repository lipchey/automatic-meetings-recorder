# Readiness

| Unit | State | Start condition |
|---|---|---|
| 1.1 repo-scaffold | done 2026-10-21 | — |
| 1.2 recorder-import | draft | Gate P + Astra passed, owner approved the spec |
| 1.3 pipeline-transcribe | draft | 1.2 merged |
| 1.4 claude-summary | draft | 1.3 merged; ⛔ `anthropic-api-key` closed |
| 1.5 diarization | draft | 1.3 merged |
| 1.6 asr-acceptance | draft | 1.3 + 1.5 merged |
| 1.7 cutover | draft | ⛔ `asr-quality-verdict` closed |
| 1.8 public-readme | draft | 1.7 done |

Human gates awaiting the owner: `anthropic-api-key`, `asr-quality-verdict`
(details in `docs/plans/2026-10-21-local-pipeline-migration.md` §Units).
