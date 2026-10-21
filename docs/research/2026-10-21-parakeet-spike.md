Status: historical (measurement snapshot).

# parakeet-mlx spike on real recordings (M4 Pro 24 GB, macOS 15.7, 2026-10-21)

Setup: `parakeet-mlx 0.5.3`, model `mlx-community/parakeet-tdt-0.6b-v3`, defaults
(bf16, full attention, 120 s chunks / 15 s overlap), input = mic+sys tracks mixed
with the current `amix + dynaudnorm` filter to 16 kHz mono wav.

| Recording | Audio | Wall time | ×realtime | Words (Parakeet) | Words (MacWhisper, current) |
|---|---|---|---|---|---|
| 2026-09-12 client call, uk/en | 80.3 min | 64.8 s | ≈74× | 7523 | 7684 |
| 2026-09-05 casual call, ru/uk mix | 2.5 min | 5.0 s | ≈30× (incl. model load) | 194 | 314 |

Observations
- Hour-long audio in about a minute; no language probe needed (auto language).
- Outputs txt/srt/vtt/json; JSON carries sentence + token timestamps (needed for
  diarization merge).
- Casual ru/uk code-switching lost ≈40 % of words vs MacWhisper's whisper-large-v3
  path; business uk/en call is on par by word count. Quality on mixed casual speech
  is the acceptance risk the plan's `asr-acceptance` unit measures.
