# Статус

Оновлено: 2026-10-21

## Наступні дії
- Виконати план `docs/plans/2026-10-21-local-pipeline-migration.md` у свіжій
  сесії (execution prompt наприкінці плану).

## Активні фронти
- Планування завершене 2026-10-21; код ще не написаний. Поточна робоча система
  (Hammerspoon → meetrec → MacWhisper → n8n на VPS) далі працює без змін до
  моменту перемикання.

## Блокери
- `ANTHROPIC_API_KEY` не заданий на машині — потрібен перед юнітом саммарі.

## Людські й зовнішні гейти
- Див. ⛔ у плані `docs/plans/2026-10-21-local-pipeline-migration.md` §Decisions.

## Відкладено й прийнятий борг
- CRM-інтеграція (Directus) і Telegram-тріаж — свідомо не в v1.
- Мертві launch agents `com.admin.movetranscriptions`, `com.admin.whisperautomove`
  — прибрати при перемиканні.
