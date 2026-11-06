# 3tris (عتريس) SaaS — Handoff for Implementing Agent

> Read this whole file before writing any code. You will be given ONE task at a time
> (see "Current task" at the bottom). Do only that task.

---

## 1. What we are building

3tris is a Telegram bot that helps Arabic freelancers find jobs. Today it works for one
user only. We are turning it into a multi-user service.

Phase 1 behavior:
1. Watch freelance sources ONCE for everyone (Telegram channels `KhamsatRequests`,
   `MostaqlDevelopment`, plus polling Mostaql directly).
2. Save each new post as a normalized **lead** (deduplicated).
3. Classify the lead into one category (cheap LLM call, once per lead).
4. Match the lead to every user whose chosen fields include that category.
5. Send each matched user a Telegram message with buttons.
6. When the user taps "write proposal", generate a personalized proposal with an LLM.
7. The user applies manually (copy button + link to the job). **There is NO auto-apply.
   Never automate logging in or submitting on Khamsat/Mostaql.**

---

## 2. Rules (follow strictly)

1. Do ONLY the current task. Do not implement future tasks or refactor unrelated code.
2. Do not change the schema, env var names, job signatures, queue names, or callback
   formats defined here. If you believe a change is needed, stop and explain why.
3. Secrets live only in `.env`. Never hardcode tokens/keys. Keep `.env.example` updated
   (with placeholder values only).
4. Only use dependencies listed in section 3. Ask before adding any other.
5. Handle errors: a single bad lead/user must never crash a service or worker. Log and continue.
6. Use type hints. Keep functions small. Async in the bot; sync is fine in RQ jobs.
7. Finish every task with the report described in section 10.

---

## 3. Stack (fixed)

- Python 3.12
- `aiogram` 3.x — the bot
- `telethon` — reading Telegram channels (user session, read-only)
- `sqlalchemy` 2.x + `alembic` + `psycopg[binary]` — PostgreSQL
- `redis` + `rq` — job queue
- `openai` (OpenAI-compatible client, base URL from env) — LLM calls
- `pydantic-settings` — config
- `httpx` + `beautifulsoup4` — Mostaql polling
- `pytest` — tests
- Docker + Docker Compose

---

## 4. Repo layout

```
3tris/
  app/
    __init__.py
    config.py              # Settings (pydantic-settings), single source of env config
    db/
      models.py            # SQLAlchemy models (section 6)
      session.py           # engine + SessionLocal
    ingest/
      main.py              # entrypoint for the `ingest` service
      telegram_watch.py    # Telethon channel listener
      mostaql_watch.py     # Mostaql poller
      normalize.py         # raw post -> lead dict
    jobs/
      queue.py             # RQ queues
      classify.py          # classify_lead(lead_id)
      match.py             # match_lead(lead_id)
      notify.py            # notify_match(match_id)
      propose.py           # generate_proposal(match_id, regenerate=False)
    bot/
      main.py              # entrypoint for the `bot` service
      onboarding.py        # /start, /settings, /pause, /resume
      leads.py             # callback handlers for lead buttons
      admin.py             # /stats + error alerts
    llm/
      client.py            # thin wrapper over OpenAI-compatible client
      prompts.py           # prompt templates (owner writes the proposal prompt)
  migrations/              # alembic
  tests/
  docker-compose.yml
  Dockerfile
  requirements.txt
  .env.example
  README.md
```

---

## 5. Environment variables (`.env.example`)

```
DATABASE_URL=postgresql+psycopg://tris:tris@db:5432/tris
REDIS_URL=redis://redis:6379/0

BOT_TOKEN=
ADMIN_CHAT_ID=

TG_API_ID=
TG_API_HASH=
TG_SESSION_PATH=/data/telethon.session
WATCH_CHANNELS=KhamsatRequests,MostaqlDevelopment
MOSTAQL_POLL_SECONDS=180

LLM_BASE_URL=
LLM_API_KEY=
LLM_MODEL_CHEAP=
LLM_MODEL_PROPOSAL=

FREE_DAILY_PROPOSALS=3
```

---

## 6. Database schema

All tables have `id` (int PK, autoincrement) unless noted, and `created_at` (timestamptz, default now()).

**users**
- `tg_chat_id` BIGINT UNIQUE NOT NULL
- `name` TEXT
- `plan` TEXT NOT NULL DEFAULT 'free' — one of `free`, `pro`
- `is_paused` BOOLEAN NOT NULL DEFAULT false
- `onboarded` BOOLEAN NOT NULL DEFAULT false

**profiles** (one per user)
- `user_id` FK users.id UNIQUE NOT NULL
- `fields` TEXT[] NOT NULL DEFAULT '{}' — values from the FIELDS list below
- `portfolio_url` TEXT NULL
- `tone_notes` TEXT NULL
- `sample_proposals` TEXT[] NOT NULL DEFAULT '{}' — max 3

**leads**
- `source` TEXT NOT NULL — `khamsat` or `mostaql`
- `external_id` TEXT NOT NULL
- `url` TEXT NOT NULL
- `title` TEXT NOT NULL
- `description` TEXT NOT NULL
- `budget_text` TEXT NULL
- `offers_count` INT NULL
- `category` TEXT NULL — set by classify
- UNIQUE (`source`, `external_id`)

**matches**
- `user_id` FK users.id NOT NULL
- `lead_id` FK leads.id NOT NULL
- `status` TEXT NOT NULL DEFAULT 'sent' — `sent`, `skipped`, `proposal_requested`, `applied`
- UNIQUE (`user_id`, `lead_id`)

**proposals**
- `match_id` FK matches.id NOT NULL
- `text` TEXT NOT NULL
- `model` TEXT NOT NULL
- `tokens_in` INT NOT NULL
- `tokens_out` INT NOT NULL

**usage_daily** (composite PK `user_id`, `date`; no `id`)
- `user_id` FK users.id
- `date` DATE (Cairo date)
- `proposals_count` INT NOT NULL DEFAULT 0

### FIELDS (fixed list)
| key | Arabic label |
|---|---|
| `video` | مونتاج فيديو |
| `web` | تطوير مواقع |
| `mobile` | تطبيقات موبايل |
| `ai_automation` | ذكاء اصطناعي وأتمتة |

Classifier may also return `other` (never matched to anyone).

---

## 7. Pipeline, queues and job signatures

```
ingest saves lead ──► classify_lead(lead_id)
                         └─► match_lead(lead_id)
                               └─► notify_match(match_id)  (one per matched user)
user taps ✍️ ───────────► generate_proposal(match_id, regenerate=False)
```

- RQ queues: `default` (classify, match, notify) and `high` (generate_proposal).
- Job functions take IDs only, open their own DB session, and are idempotent
  (running twice must not double-notify or double-insert).

---

## 8. Telegram contract

**Lead message** (notify_match):
```
📌 {source_label} | {title}
{description trimmed to 400 chars}…
💰 {budget_text or "غير محدد"}   👥 العروض: {offers_count or "?"}
```
If `offers_count > 10`, prepend `⚠️ عليه أكتر من 10 عروض\n`.

Buttons:
- `✍️ اكتب عرض` → callback `w:{match_id}`
- `تخطي` → callback `s:{match_id}`
- `🔗 افتح المشروع` → URL button to `lead.url`

**Proposal message** (generate_proposal):
- The proposal text.
- Buttons: copy-text button with the proposal (if the installed aiogram/Bot API supports
  it; otherwise send the proposal inside a monospace block instead), `🔗 افتح المشروع`
  (URL), `🔄 اكتب تاني` → callback `r:{match_id}`.

**Quota exceeded message:**
`خلصت عروضك النهارده ({limit} في اليوم في الباقة المجانية). جرّب بكرة 🙏`

Commands: `/start`, `/settings`, `/pause`, `/resume`, `/stats` (admin only).

---

## 9. Tasks (in order — do not skip ahead)

**T0 — Skeleton**
Repo layout, `requirements.txt`, `Dockerfile`, `docker-compose.yml` with services
`db` (postgres:16), `redis` (redis:7), `ingest`, `worker` (`rq worker high default`),
`bot`. Volume for `/data` (Telethon session) and postgres data. `config.py` with all env
vars. Each app service logs "service X started" and stays alive. README with run steps.
*Done:* `docker compose up --build` starts all 5 services without errors.

**T1 — Database**
Models from section 6, `session.py`, Alembic initialized, first migration.
*Done:* migration applies on an empty DB; `tests/test_db.py` inserts and reads one row per table.

**T2 — Ingestion**
Port the owner's existing `watch.py` / `mostaql_watch.py` parsing (provided separately)
into `ingest/`. `normalize.py` returns a dict matching `leads`. Insert with
ON CONFLICT DO NOTHING on (source, external_id); enqueue `classify_lead` only for new rows.
Mostaql poll interval from `MOSTAQL_POLL_SECONDS`.
*Done:* new posts appear as leads within ~1 min; duplicates ignored; one bad post doesn't stop the watcher.

**T3 — Classify**
One call with `LLM_MODEL_CHEAP`, JSON output `{"category": "...", "offers_count": int|null}`.
Category must be a FIELDS key or `other`. Validate; retry once; fallback `other`.
Save, then enqueue `match_lead`.
*Done:* tests with ≥10 sample posts (mock the LLM); invalid JSON never crashes.

**T4 — Match**
Select users where `onboarded = true`, `is_paused = false`, `lead.category = ANY(profile.fields)`.
Insert match (ON CONFLICT DO NOTHING); enqueue `notify_match` only for new matches.
*Done:* test: a `web` lead matches only `web` users; running twice creates no duplicates.

**T5 — Onboarding**
`/start`: welcome → fields multi-select inline keyboard (toggle ✅, then "تمام") →
portfolio link (skippable) → tone notes (skippable) → up to 3 sample proposals
(skippable, "خلصت" to finish) → set `onboarded = true`. Use aiogram FSM.
`/settings` to edit any part. `/pause`, `/resume`.
*Done:* a new user finishes in under 2 minutes; profile saved; existing user `/start` doesn't duplicate.

**T6 — Notify + skip**
`notify_match` sends the lead message (section 8). Handler for `s:` sets status `skipped`
and edits the message to show it was skipped. Respect Telegram rate limits (retry on 429).
*Done:* buttons work; status changes in DB.

**T7 — Proposal**
Handler for `w:` and `r:` enqueues `generate_proposal` and replies "بكتب العرض… ✍️".
Job: check/increment `usage_daily` (Cairo date; `pro` = unlimited) → build prompt via
`prompts.build_proposal_prompt(lead, profile)` → call `LLM_MODEL_PROPOSAL` → save to
`proposals` with token counts → set status `proposal_requested` → send proposal message.
Regenerate counts toward quota. If `prompts.py` has no prompt yet, use a clearly marked placeholder.
*Done:* proposal arrives in ~15 s; quota enforced; tokens saved.

**T8 — Admin**
`/stats` (only `ADMIN_CHAT_ID`): users, onboarded users, leads today, proposals today,
tokens today, skip rate vs proposal rate. Worker/bot exceptions are sent to the admin chat
(rate-limited, max 1 per minute per error type).
*Done:* numbers match DB queries; a forced error reaches admin.

**T9 — Deploy**
Production compose (restart: unless-stopped), daily `pg_dump` backup with 7-day retention,
deploy notes in README.
*Done:* system recovers after VPS reboot with no manual steps.

---

## 10. Report format (end of every task)

```
TASK: T#
FILES CHANGED: list
WHAT I DID: 3–6 bullets
HOW TO TEST: exact commands
ASSUMPTIONS / OPEN QUESTIONS: list (or "none")
```

---

## Current task

**T0 — Skeleton.** (The owner will replace this line for each new session.)
