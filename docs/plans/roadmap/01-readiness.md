# Readiness

| Unit | State | Start condition |
|---|---|---|
| 1.1 repo-scaffold | done 2026-10-21 | — |
| 1.2 recorder-import | draft | owner approved the spec |
| 1.3 pipeline-transcribe | draft | 1.2 merged |
| 1.4 asr-acceptance | draft | 1.3 merged; closes ⛔ `asr-quality-verdict` |
| 1.5 claude-summary | draft | ⛔ `asr-quality-verdict` closed; ⛔ `anthropic-api-key` closed |
| 1.6 shadow-run | draft | 1.5 merged |
| 1.7 cutover | draft | 1.6 receipt + owner "switch" |
| 1.8 public-readme | draft | 1.7 done |
| 2.1 channel-diarization | deferred | 1.8 done |

Human gates awaiting the owner: `asr-quality-verdict`, `anthropic-api-key`
(details in `docs/plans/2026-10-21-local-pipeline-migration.md` §Units).
