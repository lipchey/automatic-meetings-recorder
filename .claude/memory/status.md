# Статус

Оновлено: 2026-10-24

## Наступні дії
- Свіжа сесія виконує `docs/plans/2026-10-21-local-pipeline-migration.md` з юніта
  1.2 (execution prompt — у плані §Execution prompt). Спочатку прочитати spec.

## Активні фронти
- Планування закрите 2026-10-24 (Gate P + Astra + рішення власника). Код ще не
  написаний. Репо публічний на GitHub, лише документація.
- Поточна робоча система (Hammerspoon → meetrec → MacWhisper → n8n на VPS) далі
  працює без змін до юніта 1.7 cutover.
- Ресерч локальних LLM для саммарі (16 GB Mac / Windows) завершено — висновок у
  `docs/research/2026-10-24-local-llm-summarizer-options.md` §Synthesis → deferred юніт 2.3.

## Блокери
- ⛔ `anthropic-api-key` — ключа на машині нема; блокує 1.5.

## Людські й зовнішні гейти
- ⛔ `asr-quality-verdict` (після 1.4) і ⛔ `anthropic-api-key` (1.5) — текст у плані §Units.

## Відкладено й прийнятий борг
- 2.1 channel-diarization після cutover; CRM/Directus і Telegram-тріаж — не в v1.
- Мертві launch agents і prune без перевірки `.done` — закриваються в 1.7.
