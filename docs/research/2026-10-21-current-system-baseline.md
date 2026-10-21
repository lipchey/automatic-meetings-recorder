Status: historical (baseline snapshot before migration). Verified 2026-10-21 by reading the live config repos; line refs point at those repos, not this one.

# Current meeting-recording system: digest (verified 2026-10-21 by Explore subagent)

Mac records + transcribes; VPS (n8n + Directus + DeepSeek + Telegram) matches calendar, summarizes, triages.
Hub doc: ~/global-tools/call-recording/MEMORY.md. Code notes: ~/Scripts/memory.md. VPS side: ~/Projects/n8n-projects/personal-n8n/.agent/memory.md §13.3–13.4.

## Pipeline
1. Trigger: Hammerspoon ~/.hammerspoon/init.lua:39 polls ~/Scripts/micusers (Swift, CoreAudio kAudioProcessPropertyIsRunningInput, micusers.swift:36) every 2 s. Call apps init.lua:48-53: Zoom, Slack, Telegram, WhatsApp; Chrome/Brave only when a tab starts with https://meet.google.com (AppleScript init.lua:361-384). Start on rising edge (init.lua:452), stop after 2 missed polls (STOP_CONFIRM). Manual: tray icon / ⌃⌥⌘S (init.lua:510).
2. Record: meetrec.app (ScreenCaptureKit, ~/Scripts/meetrec.swift:200-210 capturesAudio+captureMicrophone; bundle com.lipchey.meetrec owns TCC perms; build-meetrec.sh). Output ~/Recordings/meet-YYYYMMDD-HHMMSS.{sys,mic}.m4a (init.lua:241). Optional screen: dialog "Записувати екран разом із дзвінком?" (init.lua:211) → meetrec --screen-only → .screen.mov (meetrec.swift:91). MIN_RECORD_SECONDS=45 (init.lua:65). audioMode() init.lua:138: speakers→mic only, headphones→mix.
3. Transcribe: ~/Scripts/transcribe-and-post.sh via hs.task (init.lua:283), args {base,title,startedISO,endedISO,mode} (init.lua:298), title "<App> YYYY-MM-DD HH:MM" (init.lua:280). ffmpeg amix+dynaudnorm → 16 kHz mono wav (lines 70-72). `open -ga MacWhisper` (line 85); mw talks via ~/Library/Application Support/MacWhisper/cli.sock. Language routing (41-43, 93-98): first 180 s with whisperkit:openai_whisper-small, Perl Cyrillic/Latin count → uk: whisperkit:openai_whisper-large-v3-v20240930, en: parakeet-pro:nvidia_parakeet-v3. Line 103: TRANSCRIPT="$("$MW" transcribe "$MIX" --model "$CHOSEN")" → <base>.txt (108). No timestamps, no speakers.
4. Upload: WEBHOOK_URL=https://<n8n-host>/webhook/macwhisper-transcript (line 26); payload {title,transcript,recording_path,started_at,ended_at} (111-114); X-Webhook-Secret from ~/.config/transcribe-and-post/webhook-secret (mode 600). Failure → ~/Recordings/outbox/<base>.json (141), flushed on next success. Then screen .mov muxed with audio → <base>.screen.mp4 (158-205). Media never leaves the Mac.
5. VPS n8n `macwhisper_transcript_triage` (personal-n8n/n8n/workflows/macwhisper_transcript_triage.json): webhook path line 26, secret check $env.WEBHOOK_SHARED_SECRET (48; passes when empty). Google Calendar getAll (175) → event containing started_at with attendees, prefers video link. Sub-workflow resolve_client_project_by_email (kheOWRB36V2tKq03): Directus client by attendee email/name → open projects. Matched: DeepSeek deepseek-v4-flash summary (299, temp 0.3) → POST http://directus:8055/items/recordings status matched (342). Else pending row (371) → recording_orphan_handle.
6. Orphan: recording_orphan_handle.json — DeepSeek classify → {title, suggestion} (prompt line 95) → PATCH title → Telegram CRM_NotificationsBot  buttons [📝 Work][📥 Obsidian][🗑 Discard] (telegram_capture_callback.json). Work: pick client → project → DeepSeek summary (prompt 584) → status summarized. Obsidian: build .md → Google Drive folder <obsidian-inbox> (677) → delete row. Discard: delete row.
7. Outputs: Directus crm_db.recordings (title, transcript, summary, times, recording_path, status, client, project). Obsidian vault on Google Drive (inbox). Mac: ~/Recordings/*.txt + m4a + screen mp4, pruned after 40 days by prune-recordings.sh (run by Hammerspoon at each call start). Notifications "✅ Транскрипт запощено" / "❌ … НЕ запощено" (init.lua:289/295).

## Components
- Hammerspoon ~/.hammerspoon/init.lua (own git repo, "LOADED config v9"): menu-bar 🎙️/🔴/⏳, leftover sweep on load, reload guard while in flight. Log ~/Library/Logs/hammerspoon-mic.log. Old AX-clicking version: init.lua.macwhisper-ax.bak.
- ~/Scripts repo: micusers (Swift), meetrec.app (meetrec.swift, meetrec-Info.plist, build-meetrec.sh), transcribe-and-post.sh (ffmpeg/ffprobe, jq, curl, perl; logs ~/Library/Logs/transcribe-and-post/<base>.log, 36 runs), prune-recordings.sh.
- MacWhisper Pro: /Applications/MacWhisper.app/Contents/MacOS/mw; models ~/Library/Application Support/MacWhisper/models (3.5G: whisperkit-pro, parakeetkit-pro, speakerkit/sortformer); Database 3.9G. Its own "send after transcription" webhook must stay OFF (duplicates).
- VPS: one DigitalOcean box (.agent/memory.md:9), repo <personal-n8n repo>; docker: postgres, n8n (127.0.0.1:5678), directus (:8055), calcom, cloudflared; public only via Cloudflare Tunnel <n8n-host>, Access bypass for /webhook; secrets sops .env.sops.yaml.
- Secret names: WEBHOOK_SHARED_SECRET, CRM_NOTIFICATIONS_BOT_TELEGRAM_TOKEN, GOOGLE_API_CREDENTIALS, DIRECTUS_AGENT_TOKEN, N8N_VPS_IP. n8n creds: DeepSeek account, Google Calendar account, Google Drive account, Directus bookings (header auth), TelegramCRM_NotificationsBot.
- External: DeepSeek API, Google Calendar API, Google Drive, Telegram Bot API, Directus CRM.

## VPS-necessity facts
- Telegram buttons: telegramTrigger webhook must be publicly reachable → local replacement = long-polling getUpdates, or a local dialog.
- Ingest webhook: only the Mac calls it; outbox fallback exists → nothing remote needed.
- CRM Directus recordings/clients/projects live on the VPS, shared with invoices, agent bot, tax reminders; n8n writes via in-cluster http://directus:8055. Dropping VPS for this pipeline ⇒ open decision where recordings live / how client lookup works.
- Calendar match, DeepSeek, Drive upload = plain API calls; no server needed.
- No cron/night jobs in this pipeline; everything runs right after the call. Mac must be awake (15 s MacWhisper wait already assumes so).
- Unrelated but public-dependent: obsidian_voice_routing (Android Tasker → /webhook/voice-diary) stays on VPS.

## Languages, prompts, formats
- Logs: 18 uk / 18 en probes. Russian not detected separately (routes to uk model). UI strings Ukrainian.
- Matched prompt (triage:324): "You summarize a client meeting transcript for a CRM. Reply in the transcript's own language. Output raw Markdown… Overview paragraph (2–4 sentences). ## Key points / ## Decisions / ## Action items (- [ ] Task (owner if stated))… Faithful — do not invent." Work-button prompt (callback:584) same minus Decisions. Orphan classifier (orphan:95): JSON {title (3-7 words, transcript language, filename-safe, + date), suggestion}.
- Obsidian .md: ```tabs block with tab: Summary DeepSeek / tab: Summary Gemini / tab: Transcription — summary tabs left EMPTY, only transcript filled.

## Open questions / gaps
- Diarization: speaker models installed but mw transcribe called without diarization flag.
- Drive folder <obsidian-inbox> unnamed in files; "Obsidian inbox" (probably ObsidianVault/00-Inbox, unverified).
- Dead launch agents: com.admin.movetranscriptions (watches missing ~/My Drive/TranscriptionResults, exit 1), com.admin.whisperautomove (~/Data/WhisperAutomation → ~/My Drive/TranscriptionWatcher) — cleanup candidates.
- WEBHOOK_SHARED_SECRET on VPS not verified set; guard passes when empty.
- Not found: Hazel, Folder Actions, OBS/Audio Hijack setup, Notion. Tampermonkey Meet beacon + Cal.com MEETING_* webhooks documented as retired.
