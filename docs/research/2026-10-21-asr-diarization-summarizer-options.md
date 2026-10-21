Status: historical (research snapshot). Provenance: Codex gpt-6.1-sol @ xhigh with web search, 2026-10-21, brief by the planning session.

# Local meeting transcription and notes on Apple Silicon

Research cutoff: **2026-10-21**. Hardware and installed software are taken from the brief. Release dates are distinguished from repository snapshots; runtime estimates are explicitly labeled.

## 1. NVIDIA Parakeet TDT 0.6B v3

**Coverage and license.** Released **2025-08-14**, v3 supports 25 languages: Bulgarian, Croatian, Czech, Danish, Dutch, English, Estonian, Finnish, French, German, Greek, Hungarian, Italian, Latvian, Lithuanian, Maltese, Polish, Portuguese, Romanian, Russian, Slovak, Slovenian, Spanish, Swedish, and Ukrainian. It detects language automatically; weights are **CC-BY-4.0**. [NVIDIA model card](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3)

**Code switching is not guaranteed.** In an August 2025 English/Russian example, a user reported entirely English output; NVIDIA staff replied that v3 is not fully tuned for code switching. This is a relevant warning, rather than a direct Ukrainian/English evaluation. Supporting both languages does not establish reliable switching within the same sentence or file. [NVIDIA discussion, 2025-08-16](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3/discussions/1)

**Comparable multilingual WER (%; lower is better).** NVIDIA’s September 2025 paper gives these same-evaluation results; the averages cover 24 languages across FLEURS/CoVoST/MLS, rather than conversational Ukrainian meetings. [Paper §6.1.2 and Appendix B](https://arxiv.org/html/2509.14128v2)

| Evaluation | Parakeet v3 | Whisper large-v3 |
|---|---:|---:|
| Multilingual average | 9.7 | 9.9 |
| Ukrainian FLEURS | 6.79 | 6.51 |
| Ukrainian CoVoST2 | 5.10 | 12.52 |
| Russian FLEURS | 5.51 | 4.17 |
| Russian CoVoST2 | 3.00 | 4.04 |

The paper does not provide a matched multilingual large-v3-turbo comparison. A separate May 2026 M4 Pro personal-voice-memo benchmark reports Parakeet/large-v3/turbo WER of **21.0/24.3/19.4% Russian** and **17.9/17.1/15.5% English**. Its private audio and missing retained May raw results make it exploratory; it cannot rank Ukrainian meeting performance. [NVIDIA paper](https://arxiv.org/html/2509.14128v2), [benchmark author’s methodology/results](https://github.com/Smolevich/stt-benchmarks)

**Long audio and formatting.** NVIDIA documents automatic punctuation/capitalization, word/segment timestamps, approximately **24 minutes with full attention on an A100 80GB**, or **three hours with local attention**. These are implementation/hardware limits, not a 24GB Mac guarantee or the maximum file length of a chunking CLI. [Model card and inference examples](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3)

**Names, jargon, Ukrainian evidence.** One application author’s informal 40-phrase multilingual comparison reports better Latin brand names from Parakeet, but better question marks and some Ukrainian spellings from turbo; it used ONNX/INT8, not MLX, and reports no WER. The stock MLX CLI has no initial-prompt, language-force, or vocabulary-hint flag. Treat names and technical terms as acceptance-test material. [Author’s Ukrainian report](https://github.com/alex80674097219/offline-voice-typing), [CLI source](https://raw.githubusercontent.com/senstella/parakeet-mlx/master/parakeet_mlx/cli.py)

> **Verdict:** Parakeet v3 is a strong speed-oriented candidate, with Ukrainian support and competitive published multilingual WER. [Paper](https://arxiv.org/html/2509.14128v2)
> Retain a Whisper fallback; approve unattended use only after checking representative Ukrainian/English switches, names, and decisions. [Code-switch warning](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3/discussions/1)

## 2. senstella/parakeet-mlx CLI

**Version:** latest verified release at the cutoff is **0.5.3, 2026-10-01**. Defaults are v3, SRT output, bf16, full attention, **120-second chunks with 15-second overlap**. Setting chunk duration to zero disables chunking. [Versioned PyPI documentation](https://pypi.org/project/parakeet-mlx/0.5.3/)

Example stock-CLI invocation; these commands were not executed during this research:

```bash
parakeet-mlx meeting.wav \
  --model mlx-community/parakeet-tdt-0.6b-v3 \
  --chunk-duration 120 --overlap-duration 15 \
  --local-attention --local-attention-context-size 256 \
  --output-format all --output-dir transcripts
```

The chunk and local-attention controls above are documented flags. Local attention reduces intermediate memory. Other controls include `--decoding greedy|beam`, `--beam-size`, `--max-words`, `--silence-gap`, and `--max-duration`. [Versioned CLI reference](https://pypi.org/project/parakeet-mlx/0.5.3/)

**Formats:** TXT is plain transcript text; SRT/VTT carry subtitle timings; JSON preserves text, sentences, nested token timings/durations/confidence. `--output-format all` writes all four. SRT/VTT are normally sentence-based; `--highlight-words` produces successive word/token highlighting. JSON times are rounded to three decimal places, which does not establish millisecond accuracy. [CLI serialization source](https://raw.githubusercontent.com/senstella/parakeet-mlx/master/parakeet_mlx/cli.py)

**Published Mac speeds:** below, RTF = wall time/audio duration; conversions from “times realtime” are arithmetic, not new measurements.

| Hardware/source | Published result | Scope |
|---|---|---|
| M2 Air 16GB | 33–39× realtime; RTF ≈0.026–0.030 | May 2025, CLI 0.2.7, English-only **v2**, bf16/chunked |
| M3 Air 24GB | 25.33–28.64×; RTF ≈0.035–0.039 | Same older discussion; settings affect speed |
| M4 Pro | 0.18–0.22s per 10s clip; RTF 0.018–0.022 | May 2026 personal EN/RU benchmark, short clips |

Sources: [upstream M2/M3 benchmark discussion](https://github.com/senstella/parakeet-mlx/discussions/15), [M4 Pro benchmark author](https://github.com/Smolevich/stt-benchmarks). None is a controlled hour-long v3 benchmark on this exact Mac.

**Memory and issues:** the reviewed upstream Mac benchmarks do not establish stock Python v3 peak RAM; installed RAM is not process usage. Timestamp jumps/sentence loss prompted a maintainer suggestion to try beam decoding; other reports concern noise/alignment and MLX streaming compatibility. These are reports to regression-test, not proof every 0.5.3 installation fails. [Benchmark scope](https://github.com/senstella/parakeet-mlx/discussions/15), [timestamp issue #43](https://github.com/senstella/parakeet-mlx/issues/43), [noise issue #46](https://github.com/senstella/parakeet-mlx/issues/46), [streaming issue #49](https://github.com/senstella/parakeet-mlx/issues/49)

> **Verdict:** Pin 0.5.3, keep overlapping chunks, and retain JSON as the canonical ASR artifact. [Release/flags](https://pypi.org/project/parakeet-mlx/0.5.3/)
> Compare beam decoding on missing/jumping passages and measure peak RAM on one full meeting before unattended deployment. [Maintainer’s response](https://github.com/senstella/parakeet-mlx/issues/43)

## 3. Offline speaker diarization on Apple Silicon

**Least-friction integrated CLI: Trail of Bits Scribe**, combining Parakeet-MLX and Senko. Its repository snapshot at the cutoff documents Apple Silicon, Python 3.13+, uv, ffmpeg, and automatic model downloads without an account. After cloning and `uv sync`, run `uv run scribe meeting.wav --format json`. [Scribe README](https://github.com/trailofbits/scribe)

| Option | Offline Mac route and material limitation |
|---|---|
| **Senko / Scribe** | macOS 14+ Core ML segmentation/embeddings plus CPU clustering; author reports 1h diarized in 7.7s on M3. Outputs one active speaker at a time, suppressing overlapping speakers. [Senko](https://github.com/narcotic-sh/senko) |
| **FluidAudio** | Swift/Core ML offline Community-1 pipeline; macOS 14+, Swift 6.0; release **0.17.7, 2026-10-08**. Code Apache-2.0; converted Community-1 artifacts CC-BY-4.0. [Manifest](https://github.com/FluidInference/FluidAudio/blob/main/Package.swift), [releases](https://github.com/FluidInference/FluidAudio/releases), [code license](https://github.com/FluidInference/FluidAudio/blob/main/LICENSE), [model card](https://huggingface.co/FluidInference/speaker-diarization-coreml) |
| **pyannote.audio 3.x/4.x** | MIT code; 3.1.0 released 2023-11-16, 4.0.0 2025-09-29, 4.0.7 2026-06-30. CPU is an offline baseline; prefer 4.x/Community-1 for a new evaluation. [Changelog](https://github.com/pyannote/pyannote-audio/blob/main/CHANGELOG.md), [license](https://github.com/pyannote/pyannote-audio/blob/main/LICENSE) |
| **NeMo Sortformer / diar_streaming** | Older streaming Sortformer V2.1 is capped at four speakers. Newer official **NeMo-Speech.cpp** supports Metal and eight-speaker Nemotron 3. For hour-long files use its default stateful streaming pass; full-attention `--offline` is limited to ≈6.6min. [V2.1 card/license](https://huggingface.co/nvidia/diar_streaming_sortformer_4spk-v2.1), [native runtime](https://github.com/NVIDIA/NeMo-Speech.cpp), [CLI](https://github.com/NVIDIA/NeMo-Speech.cpp/blob/main/docs/cli.md) |
| **WhisperX** | Complete Whisper/alignment/diarization pipeline; documented Mac-safe ASR path is CPU/int8. CTranslate2 does not supply Apple Metal GPU ASR; prebuilt GPU support is NVIDIA. Requires pyannote model access. [WhisperX](https://github.com/m-bain/whisperX), [backend hardware](https://opennmt.net/CTranslate2/hardware_support.html) |
| **sherpa-onnx** | Standalone offline diarization binary/Swift APIs with ONNX segmentation and embeddings, fixed speaker counts or clustering thresholds. More model/CLI assembly; not the complete Community-1/VBx pipeline. Check each model’s license. [Official documentation](https://k2-fsa.github.io/sherpa/onnx/speaker-diarization/models.html) |

**HF token and licenses:** the legacy speaker-diarization-3.1 pipeline is MIT and requires accepting both its and segmentation-3.0’s HF conditions. Community-1 is **CC-BY-4.0**, also gated for initial download. Documented local-directory loading works fully offline after downloading artifacts: the token does not require uploading audio or using cloud inference. [3.1 model card](https://huggingface.co/pyannote/speaker-diarization-3.1), [Community-1 offline instructions](https://huggingface.co/pyannote/speaker-diarization-community-1)

Community-1 provides ordinary overlap-capable diarization and an **exclusive** timeline for easier ASR reconciliation; known/minimum/maximum speaker counts can be supplied. Disable pyannote telemetry with `PYANNOTE_METRICS_ENABLED=0` for a deliberately offline setup. [Community-1 outputs](https://huggingface.co/pyannote/speaker-diarization-community-1), [telemetry configuration](https://pypi.org/project/pyannote-audio/4.0.7/)

**Quality for 2–6 Ukrainian speakers:** none of these cited evaluations establishes Ukrainian meeting DER. Senko notes English/Mandarin embedding training; Scribe additionally assigns each whole ASR segment to its greatest-overlap speaker, potentially mislabeling rapid changes inside a sentence. FluidAudio’s conversion is not guaranteed identical to upstream pyannote: its benchmarks trade speed against DER. Generic speaker IDs still require naming/enrollment. [Senko limitations](https://github.com/narcotic-sh/senko), [Scribe assignment source](https://github.com/trailofbits/scribe/blob/main/scribe.py), [FluidAudio benchmarks](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Benchmarks.md), [speaker matching](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Diarization/GettingStarted.md)

FluidAudio’s component CLI is `swift run -c release fluidaudiocli process meeting.wav --mode offline --output meeting.json`; reconcile its intervals with ASR word times. NeMo’s route is `nemo-speech diarize meeting.wav --format rttm --output meeting.rttm`. [FluidAudio CLI](https://github.com/FluidInference/FluidAudio), [NeMo CLI](https://github.com/NVIDIA/NeMo-Speech.cpp/blob/main/docs/cli.md)

> **Verdict:** Start with Scribe/Senko for the simplest batch trial; inspect interruptions and rapid turns before trusting owner attribution. [Scribe](https://github.com/trailofbits/scribe)
> For important 5–6-person meetings, compare FluidAudio offline Community-1 on a manually labeled excerpt; Ukrainian accuracy remains unproven. [Evaluation scope](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Benchmarks.md)

## 4. Alternatives worth deploying or screening out

**Deployable complement: mlx_whisper with full large-v3.** Large-v3 was released 2023-11-06; turbo followed 2024-10-01, reducing the decoder from 32 layers to four with a speed/accuracy tradeoff. Use cached full large-v3 for difficult mixed-language passages; turbo is useful when turnaround matters. This is a fallback recommendation, not a measured Ukrainian/English win. MLX Whisper supports word timestamps and local model paths. [Large-v3 release](https://github.com/openai/whisper/discussions/1762), [turbo release](https://github.com/openai/whisper/discussions/2363), [MLX implementation](https://github.com/ml-explore/mlx-examples/tree/main/whisper)

The following lack evidence sufficient to displace the cached pair for this workflow, so they receive screening notes rather than installation recommendations.

| Candidate | Reason not to adopt as primary here |
|---|---|
| **WhisperKit** | Native Apple implementation of Whisper, not a new acoustic model; Swift/Core ML integration may help an app but does not itself improve language accuracy using the same weights. [Official project](https://github.com/argmaxinc/argmax-oss-swift) |
| **Voxtral** | Open-weight Realtime-2602 supports EN/RU but not Ukrainian. **Voxtral Mini Transcribe 3 launched 2026-10-20**, adds Ukrainian/file-transcription coverage, but is documented as Premier public preview/API and self-deployment; this does not establish a public downloadable Mac open-weight package. [2602 card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602), [3.0 version/date](https://docs.mistral.ai/models/voxtral-mini-transcribe-3-0), [announcement](https://mistral.ai/news/voxtral-3/) |
| **Canary-1B-v2 / Canary-Qwen** | 1B-v2 (2025-08-14) supports Ukrainian/English/Russian, but its card does not demonstrate a mixed-meeting/Mac advantage. Canary-Qwen-2.5B (2025-07-17) is English-only. [1B-v2](https://huggingface.co/nvidia/canary-1b-v2), [Qwen variant](https://huggingface.co/nvidia/canary-qwen-2.5b) |
| **Qwen3-ASR** | January 2026 0.6B/1.7B family lists 30 languages including EN/RU, but omits Ukrainian. [Official card](https://huggingface.co/Qwen/Qwen3-ASR-1.7B) |
| **Kyutai** | Released STT models are English/French 1B and English-only 2.6B, leaving Ukrainian unsupported. [Developer model card](https://huggingface.co/kyutai/stt-2.6b-en) |
| **Moonshine** | Ukrainian-specific base-uk exists; its sparse official card supplies no evaluation establishing an advantage for mixed meetings. [Official Ukrainian model](https://huggingface.co/moonshine-ai/moonshine-base-uk) |
| **Ukrainian-specialized Whisper** | KSE’s dido-yvanchyk-v2, trained 2025-12-26, reports 13.2% WER on its own test data without a matched Parakeet baseline. Ukrainian calibration of quantization also does not establish a superior language fine-tune. [KSE card](https://huggingface.co/KSE-RESEARCH-Group/whisper-large-v3-dido-yvanchyk-v2), [quantized Ukrainian card](https://huggingface.co/Yehor/whisper-large-v3-turbo-quantized-uk) |

> **Verdict:** My one primary is **Parakeet v3** for fast local batches; my one fallback is **full Whisper large-v3 through mlx_whisper** for questionable mixed-language passages. [Comparison](https://arxiv.org/html/2509.14128v2), [MLX implementation](https://github.com/ml-explore/mlx-examples/tree/main/whisper)
> If acceptance tests show repeated language drift, promote the fallback to default rather than attempting to repair transcript errors in the summarizer.

## 5. Local Ukrainian summarization on M4 Pro / 24GB

**Recommendation: Qwen3.5 9B, Q4_K_M through Ollama.** The 9B release dates to **2026-03-02**; Qwen lists Ukrainian among 201 languages/dialects and a 262,144-token native context. Ollama’s explicit `qwen3.5:9b-q4_K_M` tag is approximately **6.6GB**. These support a practical trial, not a Ukrainian-summary quality guarantee. [Official chronology](https://github.com/QwenLM/Qwen3.8#news), [language coverage](https://qwen.ai/blog?id=qwen3.5), [9B card](https://huggingface.co/Qwen/Qwen3.5-9B), [Ollama tags](https://ollama.com/library/qwen3.5/tags)

**Other realistic sizes:** Qwen3 14B supports Ukrainian and has a 9.3GB Ollama download; its FP16 KV cache at 24,576 tokens is approximately **3.75GiB**, calculated from its configuration. Qwen3 30B-A3B’s roughly 19GB weights plus cache/OS make it a poor default on 24GB despite only ≈3B active parameters. Download size is not total working memory. Run ASR and LLM stages sequentially. [Ollama sizes](https://ollama.com/library/qwen3), [Qwen3 languages](https://qwenlm.github.io/blog/qwen3/), [14B architecture](https://huggingface.co/Qwen/Qwen3-14B/raw/main/config.json), [30B architecture](https://huggingface.co/Qwen/Qwen3-30B-A3B/raw/main/config.json)

Set context explicitly and disable thinking. Example `/api/chat` body:

```json
{"model":"qwen3.5:9b-q4_K_M","think":false,"stream":false,
 "messages":[{"role":"user","content":"...instructions and transcript..."}],
 "options":{"num_ctx":24576,"num_predict":2000}}
```

Here `think` is top-level and `num_ctx` is an option. A 24,576-token budget accommodates the brief’s 10–15k input estimate plus instructions/output; count actual tokens and reject truncation. Ollama defaults vary with available GPU memory, so set context explicitly. [Thinking API](https://docs.ollama.com/capabilities/thinking), [context configuration](https://docs.ollama.com/context-length)

**Expected runtime—estimate:** budget **2–5 minutes** for a non-thinking summary of about 1,500 output tokens, including input processing/loading. A July 2026 MLX benchmark reports M4 Pro 24GB Qwen3-14B int4 at 224 prompt tokens/s and 29 generated tokens/s: arithmetic gives ≈97–119s for 10–15k input plus 1,500 output. Its M4 Pro 64GB Qwen3.5-9B int4 result is 39 generated tokens/s. These short-prompt MLX results are not full-transcript Ollama measurements; a busy Mac or longer context may exceed the planning range. [Benchmark methodology/hardware tables](https://github.com/weklund/mlx-coding-bench)

**Quality versus Claude API:** no matched Ukrainian meeting-summary evaluation was established by the cited evidence. The available local/API benchmark covers coding/reasoning/writing tasks and cannot establish parity for action-item extraction. My judgement is to use local 9B for reviewable first drafts and retain existing Claude output for A/B comparison during migration, without treating either as ground truth. [Benchmark scope](https://github.com/weklund/mlx-coding-bench)

Recommended prompt contract: Ukrainian notes; separate decisions, proposals, unresolved questions, and actions; every action includes an evidence timestamp, explicit owner or null, explicit deadline or null; preserve technical names; never infer a commitment from a suggestion. For difficult transcripts, extract evidence in chunks and synthesize those extracts.

> **Verdict:** Start with **qwen3.5:9b-q4_K_M, 24,576 context, thinking off**; reserve 2–5 minutes as an estimate, then record actual timings. [Model](https://ollama.com/library/qwen3.5/tags), [timing evidence](https://github.com/weklund/mlx-coding-bench)
> Evaluate missed/invented actions and Ukrainian readability on the same meetings used to assess Claude; equivalent quality remains unproven.

## 6. macOS triggers, sleep, wake, and permissions

**Recorder completion is the strongest trigger.** Zoom normally stores recordings under `~/Documents/Zoom` and converts them after recording ends; closing the laptop during conversion can stop it. Watch finalized media rather than treating a new folder as proof audio is ready. [Zoom locations/conversion](https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0063423), [conversion failures](https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0061761)

OBS WebSocket **v5** provides `RecordStateChanged` with output state, active flag, and saved `outputPath` when stopped. Audio Hijack **4.6.0, 2026-08-27** supports session-event scripts/recording context; shell/Shortcut calls block its main thread, so enqueue work instead of running ASR inside the callback. [OBS protocol](https://raw.githubusercontent.com/obsproject/obs-websocket/master/docs/generated/protocol.md), [Audio Hijack version](https://rogueamoeba.com/support/knowledgebase/?showCategory=Audio+Hijack), [automation](https://rogueamoeba.com/support/manuals/audiohijack/?page=scripting), [API](https://rogueamoeba.com/support/knowledgebase/?showArticle=AH-Scripting-API)

For Screenshot/QuickTime recordings, use the saved movie as the handoff. Apple documents microphone selection for recording audio; test that remote participants are actually captured rather than assuming the movie includes their audio. [Apple screen-recording guide](https://support.apple.com/en-gb/102618)

**Microphone logging is only a heuristic.** A 2026-09-01 reverse-engineering report of Zoom 7.1.5 shows this detector: `log stream --type log --level default --style compact --predicate 'subsystem == "com.apple.controlcenter" AND category == "sensor-indicators"'`. It parses microphone attributions, not meeting identity; Chrome mic use need not mean Google Meet. The format is undocumented, not a stable API. Validate it on macOS 15.7 and use it only as a secondary hint. [Primary analysis/tested command](https://xusheng.dev/posts/reversing/zoom_my_notes_detector/main/)

**WatchPaths versus Folder Actions:** launchd’s manual warns WatchPaths can miss changes or fire while files are inconsistent. Automator Folder Actions run when items are added, not when a recorder has completed. Recommendation: a user LaunchAgent with a persistent queue, recursive rescan, completion/probe checks, deduplication, single-worker lock, and atomic result publication. File-size stability is only a fallback heuristic; paused recordings can also be stable. [Apple-authored launchd manual, mirror](https://keith.github.io/xcode-man-pages/launchd.plist.5.html), [Apple Folder Actions documentation](https://support.apple.com/guide/automator/create-a-workflow-aut7cac58839/2.10/mac/15.0)

**Sleep and missed runs:** `StartInterval` firings during sleep, or while the job runs, are missed. `StartCalendarInterval` coalesces calendar firings missed during sleep into one run after wake. Neither is a persistent job queue; add `RunAtLoad` for login/reload recovery and calendar rescanning rather than assuming missed intervals are replayed. A 30-second awake interval is reasonable for a short queue scan. [Current Apple-authored manual, mirror](https://keith.github.io/xcode-man-pages/launchd.plist.5.html)

**Preventing sleep:** wrap processing in `/usr/bin/caffeinate -i /absolute/path/to/worker` to prevent idle sleep while it runs; `-s` is effective only on AC power. Do not rely on it to defeat lid closure. In February 2026 Apple DTS reiterated that StartCalendarInterval guarantees no particular system power state; a reported DarkWake may start a job and then suspend it. Persist the queue and resume/retry after full wake. [Apple caffeinate manual source](https://raw.githubusercontent.com/apple-oss-distributions/PowerManagement/main/caffeinate/caffeinate.8), [Apple discussion/DTS response](https://developer.apple.com/forums/thread/815034)

**Scheduled wake:** pmset can request wake schedules; Apple documents `pmset -g sched` and, for example, `sudo pmset repeat wake M 8:00:00`. This is separate from launchd scheduling. For a closed-lid laptop, design for processing after reopening/full wake; scheduling wake does not establish that an unattended GPU batch will finish. Startup also raises login/FileVault constraints. [Apple scheduling guide](https://support.apple.com/en-gb/guide/mac-help/-mchl40376151/mac), [power-state caveats](https://developer.apple.com/forums/thread/815034)

**TCC/background gotchas:** microphone capture requires the responsible app’s grant and usage-description configuration; screen/system-audio capture has separate controls. File-only ffmpeg conversion does not acquire the microphone—distinguish file input from AVFoundation device capture. Protected Documents folders and process attribution can still block a LaunchAgent when a Terminal test works. Test the actual background path; prefer a stable app/helper identity for capture, absolute executables, and explicit log/cache paths. [Apple capture authorization](https://developer.apple.com/documentation/bundleresources/requesting-authorization-for-media-capture-on-macos), [screen/audio controls](https://support.apple.com/en-gb/guide/mac-help/mchld6aa7d23/mac), [FFmpeg devices](https://ffmpeg.org/ffmpeg-devices.html), [Apple DTS permissions](https://developer.apple.com/forums/thread/678819)

> **Verdict:** Use recorder-finalization events plus a durable queue and periodic rescans; WatchPaths and microphone logs should only help discover work. [OBS event](https://raw.githubusercontent.com/obsproject/obs-websocket/master/docs/generated/protocol.md), [launchd](https://keith.github.io/xcode-man-pages/launchd.plist.5.html)
> Keep the lid open during conversion/processing, use caffeinate for idle sleep, and make wake recovery idempotent. [Zoom](https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0061761), [Apple power discussion](https://developer.apple.com/forums/thread/815034)

## Recommended stack

| Stage | Recommendation |
|---|---|
| ASR | **Primary: Parakeet TDT 0.6B v3**, parakeet-mlx 0.5.3, chunked; **fallback: full large-v3 via mlx_whisper**. [CLI](https://pypi.org/project/parakeet-mlx/0.5.3/), [fallback](https://github.com/ml-explore/mlx-examples/tree/main/whisper) |
| Diarization | **Scribe/Senko** for the quickest integrated workflow; audit rapid turns. Compare **FluidAudio offline Community-1** if speaker attribution is unacceptable. [Scribe](https://github.com/trailofbits/scribe), [FluidAudio](https://github.com/FluidInference/FluidAudio) |
| Summarizer | **Ollama qwen3.5:9b-q4_K_M**, 24,576 context, thinking off; require timestamped evidence and nullable owners/deadlines. [Model](https://ollama.com/library/qwen3.5/tags), [API](https://docs.ollama.com/capabilities/thinking) |
| Trigger/orchestration | **User LaunchAgent + local queue worker**, OBS stop/finalized Zoom audio, 30-second awake rescans and calendar/login recovery, caffeinate around work. Existing workflows can instead use local n8n. [launchd](https://keith.github.io/xcode-man-pages/launchd.plist.5.html), [local n8n](https://docs.n8n.io/hosting/installation/npm/) |

Before removing MacWhisper/VPS from routine use, manually check Ukrainian/English switches, Russian passages, technical names, silence/noise, overlapping speech, and 2–6 speakers. Include a full hour-long recording; score transcript errors, missing/invented actions, speaker attribution, runtime, and peak memory.

## Risks

- **Language drift/proper nouns:** multilingual support does not remove the documented code-switch limitation. [NVIDIA warning](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3/discussions/1), [Ukrainian anecdote](https://github.com/alex80674097219/offline-voice-typing)
- **Wrong action owners:** Senko suppresses overlaps; Scribe assigns complete ASR segments to one speaker. [Senko](https://github.com/narcotic-sh/senko), [assignment source](https://github.com/trailofbits/scribe/blob/main/scribe.py)
- **Unproven combined accuracy/runtime:** cited tests do not jointly benchmark this Ukrainian pipeline. [ASR scope](https://arxiv.org/html/2509.14128v2), [diarization scope](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Benchmarks.md), [LLM scope](https://github.com/weklund/mlx-coding-bench)
- **Sleep, partial recordings, TCC:** preserve originals and durable job state; conversion, permissions, and DarkWake can interrupt processing. [Zoom](https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0061761), [Apple power](https://developer.apple.com/forums/thread/815034), [permissions](https://developer.apple.com/forums/thread/678819)
- **Reproducibility/offline setup:** pin versions/commits, retain attribution, pre-download artifacts, and disable optional telemetry. Converted models have separate license/provenance considerations. [Release](https://pypi.org/project/parakeet-mlx/0.5.3/), [offline/license](https://huggingface.co/pyannote/speaker-diarization-community-1), [telemetry](https://pypi.org/project/pyannote-audio/4.0.7/), [conversion provenance](https://huggingface.co/FluidInference/speaker-diarization-coreml)

