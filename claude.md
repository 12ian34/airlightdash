# airlightdash

## What this is
Hackathon quickstart: Supabase (hosted Postgres) + Lightdash (BI / semantic layer / AI agents) with no dbt.
README.md is the guide for hackathon participants at a NYC event.
Written by the Lightdash team — includes a "rough edges" section with internal product feedback.

## Architecture
- **Supabase** — hosted Postgres, used as both the app database and the analytics warehouse
- **Lightdash YAML** — semantic layer defined in `lightdash/models/*.yml`, no dbt required
- **Lightdash Cloud** — app.lightdash.cloud for BI UI, dashboards, AI agents
- **Lightdash CLI** — `lightdash lint`, `lightdash deploy`, `lightdash install-skills`

## Data (our test dataset)
- Source: SQLite DB from an air quality sensor (`airlab.db`, local only, not committed)
- Table: `airlab_readings` — 3,586 rows of CO2, temperature, humidity, pressure, VOC, NOx
- Seeded into hosted Supabase via `psql` with `supabase/seed.sql`
- Supabase project ref: `wlekztlmqhckvcxkxwbz` (eu-west-1)

## Key files
- `lightdash.config.yml` — declares warehouse type as postgres
- `lightdash/models/airlab_readings.yml` — Lightdash YAML model (dimensions, metrics, time intervals)
- `lightdash/charts/*.yml` — chart definitions (line, big_number, etc.)
- `lightdash/dashboards/airlab-overview.yml` — dashboard layout referencing charts
- `supabase/config.toml` — Supabase local config (project_id: airlightdash)
- `supabase/migrations/20260215000000_create_airlab_readings.sql` — Postgres table DDL
- `supabase/seed.sql` — 3,586 INSERT statements with NULL handling
- `README.md` — hackathon guide + rough edges section
- `.claude/skills/developing-in-lightdash/` — Lightdash reference docs installed via `lightdash install-skills`

## Commands
- `lightdash lint` — validate YAML models (fast, run constantly)
- `lightdash deploy --create --no-warehouse-credentials` — first deploy, creates project
- `lightdash deploy --no-warehouse-credentials` — subsequent model deploys (dimensions, metrics, explores)
- `lightdash upload --include-charts` — push charts and dashboards (use `--force` first time)
- `lightdash upload --force -c chart-slug-1 chart-slug-2 --include-charts` — upload specific charts only
- `lightdash install-skills` — install AI copilot rules for Lightdash YAML
- `supabase link` — link to hosted Supabase project
- `supabase db push` — push migrations to remote
- `/opt/homebrew/opt/libpq/bin/psql "CONNECTION_STRING" -f supabase/seed.sql` — seed data

**deploy vs upload:** `deploy` = model changes. `upload` = charts/dashboards. They're separate — you often need both after changes.

## Conventions
- No dbt — Lightdash YAML exclusively (type: model, sql_from, dimensions, metrics)
- YAML schema: https://raw.githubusercontent.com/lightdash/lightdash/refs/heads/main/packages/common/src/schemas/json/model-as-code-1.0.json
- Always `lightdash lint` after editing model YAML
- Don't commit secrets, .env, or the SQLite db file

## Connecting Supabase to Lightdash
- Use the **Shared Pooler** host from Supabase Connect, not the direct host
- User format: `postgres.PROJECT_REF` (e.g. `postgres.wlekztlmqhckvcxkxwbz`)
- SSL mode: **`no-verify`** (required — `require` and default both fail)
- Port: match what Supabase shows (usually 6543 transaction, 5432 session)
- dbt project section: select "CLI", ignore the rest

## Status
- [x] SQLite DB copied from remote server (ian@pian_local)
- [x] Supabase migration created and pushed
- [x] Seed SQL generated (3586 rows, NULLs handled)
- [x] Data seeded into hosted Supabase
- [x] Lightdash YAML model created and linted
- [x] Lightdash CLI skills installed
- [x] Hackathon guide written with rough edges section
- [x] Supabase project linked
- [x] Lightdash project deployed
- [x] Lightdash warehouse connection configured (shared pooler, no-verify SSL)
- [x] Charts and dashboard created and uploaded
- [x] Fixed blank pressure/humidity charts (area→line, removed areaStyle)

## Gotchas discovered
- Direct Supabase host doesn't resolve from Lightdash Cloud — use shared pooler
- SSL mode must be `no-verify` — even `require` fails with "self-signed certificate in certificate chain"
- Model-level `count` metrics need explicit `sql` — lint doesn't catch it, deploy fails
- `lightdash deploy --create` has interactive project name prompt, no CLI flag for it
- libpq needed for psql on Mac: `brew install libpq` (keg-only, /opt/homebrew/opt/libpq/bin/psql)
- SQLite export: empty CSV fields become `, ,` not `NULL` — must handle in seed generation
- Lightdash onboarding wizard assumes dbt — select "Create manually" → "I've defined them" → "CLI"
- "Save & Test" on warehouse connection doesn't actually test the connection
