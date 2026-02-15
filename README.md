# Supabase + Lightdash — Hackathon Quick Start

Get a semantic layer and BI on top of your Supabase data in ~15 minutes. No dbt. No data engineering background needed. Just your Supabase tables, some YAML, and [Lightdash YAML](https://docs.lightdash.com/guides/lightdash-yaml).

You'll end up with charts, dashboards, and an AI agent that can answer questions about your data in plain English.

---

## What you'll need

- A Mac with [Homebrew](https://brew.sh)
- A Supabase project with some data in it (or about to have data — if you're building an app that writes to Supabase, that counts)

---

## 1. Create a Supabase project

If you don't have one yet:

1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) and sign in (or create an account)
2. Click **New Project**
3. Pick an org, give it a name, and **save your database password** — you'll need it later for Lightdash
4. Enable the **Data API** if prompted
5. Wait for it to provision (~1–2 min)

---

## 2. Get some data in there

However your app works — Supabase API, a script, CSV import via Table Editor, raw SQL — just make sure you have at least one table with data before moving on.

Once you do, grab your table structure. You'll feed this to your AI copilot in step 5 to auto-generate your Lightdash model.

1. Go to **SQL Editor** in the Supabase dashboard
2. Run:

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;
```

3. **Copy the output** and keep it handy

---

## 3. Sign up for Lightdash

1. Go to [app.lightdash.cloud/register](https://app.lightdash.cloud/register)
2. Sign up and verify your email
3. You'll land on a project setup wizard — **stop here, don't click anything yet!**

Leave this tab open. We'll come back to it in step 6.

---

## 4. Install the Lightdash CLI

```bash
brew tap lightdash/lightdash
brew install lightdash
```

Log in:

```bash
lightdash login https://app.lightdash.cloud
```

This pops open your browser to authorize the CLI.

Install AI copilot skills (this loads Lightdash reference docs into Cursor, Claude Code, etc.):

```bash
lightdash install-skills
```

This is worth doing — it means your AI copilot already knows the Lightdash YAML format and can generate models for you.

---

## 5. Define your Lightdash YAML model

This is where you describe your data to Lightdash — what the columns mean, what metrics to compute, how to label things for humans. Full docs here: **[Lightdash YAML guide](https://docs.lightdash.com/guides/lightdash-yaml)**

### Set up the project files

```bash
mkdir -p lightdash/models
```

Create `lightdash.config.yml` in your project root:

```yaml
warehouse:
  type: postgres
```

### Generate your model with AI (the fast way)

If you ran `lightdash install-skills` in step 4, your AI copilot already knows the Lightdash YAML format. Just paste the table structure you grabbed from Supabase and ask for a model.

Example prompt:

> Here's my Supabase table structure:
>
> ```
> table_name  | column_name      | data_type
> ------------+------------------+---------------------------
> my_readings | id               | bigint
> my_readings | recorded_at      | timestamp with time zone
> my_readings | sensor_name      | text
> my_readings | value            | double precision
> ```
>
> Create a Lightdash YAML model file at `lightdash/models/my_readings.yml` for this table. Include useful metrics and set up time intervals on any timestamp columns. Run `lightdash lint` when done.

Your copilot will generate a valid model with dimensions, metrics, labels, and descriptions — and validate it.

### Or write it by hand

Create a YAML file at `lightdash/models/your_table.yml`:

```yaml
type: model
name: my_readings
label: My Readings
description: Sensor readings over time
sql_from: public.my_readings

metrics:
  total_readings:
    type: count
    sql: ${TABLE}.id
    description: Total number of readings

  avg_value:
    type: average
    sql: ${TABLE}.value
    label: Average Value
    round: 1

dimensions:
  - name: id
    type: number
    sql: ${TABLE}.id
    hidden: true

  - name: recorded_at
    type: timestamp
    sql: ${TABLE}.recorded_at
    label: Recorded At
    time_intervals:
      - RAW
      - HOUR
      - DAY
      - WEEK
      - MONTH

  - name: sensor_name
    type: string
    sql: ${TABLE}.sensor_name
    label: Sensor Name

  - name: value
    type: number
    sql: ${TABLE}.value
    label: Value
    round: 1
```

Key things:

- **`sql_from`** — your Supabase table: `public.your_table`
- **`dimensions`** — one per column. Types: `number`, `string`, `timestamp`, `date`, `boolean`
- **`metrics`** — aggregations: `count`, `average`, `sum`, `min`, `max`. **Every metric needs an explicit `sql` field** (even `count`)
- **`time_intervals`** — add on timestamp/date columns for hour/day/week/month grouping
- **`label`** and **`description`** — these power Lightdash's AI agent, so make them descriptive
- Full schema spec: [model-as-code-1.0.json](https://raw.githubusercontent.com/lightdash/lightdash/refs/heads/main/packages/common/src/schemas/json/model-as-code-1.0.json)

### Validate

```bash
lightdash lint
```

```
✓ All Lightdash Code files are valid!
```

If you see errors, fix them before deploying. Lint is fast — run it every time you edit YAML.

---

## 6. Connect Lightdash to Supabase

Go back to the Lightdash tab you left open in step 3 (the setup wizard).

1. Click **"Create manually"**
2. Click **"I've already defined them"**

This takes you to the connection form.

### Get your Supabase connection details

1. In the Supabase dashboard, click **Connect** at the top of your project
2. Select the **Shared Pooler** tab
3. Click **"View parameters"**

### Fill in the Lightdash connection form

- **Host** — the pooler host (e.g. `aws-0-us-east-1.pooler.supabase.com`)
- **Database** — `postgres`
- **User** — `postgres.xxxx` (the full string with the project ref after the dot)
- **Password** — the database password you saved in step 1

### The critical part: Advanced settings

Click **Advanced** — this section looks optional but it is not:

- **Port** — match what Supabase shows (usually `6543` for transaction mode, `5432` for session mode)
- **SSL mode** — set to **`no-verify`**. If you skip this you'll get "self-signed certificate in certificate chain" and nothing will work

For the **dbt project** section: select **"CLI"** and ignore everything else. This section is irrelevant for Lightdash YAML users, but you have to pick something.

Hit **Save & Test**.

> **Why the shared pooler?** Lightdash Cloud connects from their servers, not your laptop. The direct Supabase host (`db.xxxx.supabase.co`) may not resolve from Lightdash's infrastructure. The shared pooler always works.

---

## 7. Deploy your models

Now push your Lightdash YAML models to the project:

```bash
lightdash deploy --no-warehouse-credentials
```

If this is your first time and the wizard didn't create a project yet, use:

```bash
lightdash deploy --create --no-warehouse-credentials
```

It'll ask for a project name — hit Enter for the default.

Subsequent deploys (after editing YAML) are just:

```bash
lightdash deploy --no-warehouse-credentials
```

---

## 8. Explore your data

Go to your Lightdash project in the browser. You should see your model in the sidebar — click into it and start exploring:

- Pick dimensions and metrics, hit **Run query**
- Build charts and pin them to dashboards
- Try the **AI agent** — ask it questions about your data in plain English

That's it. You've got a full semantic layer and BI tool running on top of Supabase, with no dbt in sight.

---

## Tips

- **Let AI write your YAML** — after `lightdash install-skills`, just paste your table structure and ask. It's fast.
- **Run `lightdash lint` constantly** — it's instant and catches most errors before deploy
- **Good descriptions = good AI** — `label` and `description` fields directly improve what Lightdash's AI agent can do
- **Every metric needs `sql`** — even `count` types need `sql: ${TABLE}.column`
- **Supabase SQL Editor** is great for quick data checks and schema grabs
- **Subsequent deploys** don't need `--create`: just `lightdash deploy --no-warehouse-credentials`

---

## Project structure

```
your-project/
├── lightdash.config.yml          # Tells Lightdash you're using Postgres
├── lightdash/
│   └── models/
│       └── your_table.yml        # Your Lightdash YAML model
└── README.md
```

No dbt, no migrations, no build step. Just YAML and deploy.

---

## Rough edges we hit (Lightdash team notes)

We set this up end-to-end for a hackathon and hit some sharp edges. These are notes for ourselves and anyone else who runs into the same things.

### Onboarding assumes dbt

The setup wizard is built for dbt users. If you're using Lightdash YAML, you have to guess your way through:

1. **"Create manually"** — not obvious this is the right path
2. **"I've already defined them"** — the metrics prompt wording is dbt-centric
3. **The dbt project section is irrelevant** — you have to pick "CLI" and ignore the rest, which feels wrong
4. **No Supabase preset** — just "Postgres". A Supabase option could auto-fill the pooler host, suggest `no-verify` SSL, and skip the dbt section
5. **Advanced settings look optional but are critical** — SSL mode and port live here, and you can't connect to Supabase without changing them

A dedicated "I'm using Lightdash YAML" or "I don't use dbt" path would fix most of this.

### "Save & Test" doesn't actually test the connection

Hitting Save & Test reports success but doesn't verify the connection actually works. You don't find out it's broken until you try SQL Runner or run a query. There's no way to:

- Run a test query against the warehouse
- See connection error logs
- Distinguish "saved config" from "can actually reach the database"

A real connection test (even just `SELECT 1`) with raw error output would save a lot of debugging.

### SSL mode defaults break Supabase

The default SSL mode causes "self-signed certificate in certificate chain" with Supabase. You have to set it to `no-verify` — even `require` doesn't work. Suggestions:

- Default to `no-verify` (works with most hosted Postgres)
- Show a better error message suggesting the fix
- If the host looks like `*.pooler.supabase.com`, auto-suggest the right SSL config

### Direct vs pooler host confusion

Supabase shows several connection options (direct, shared pooler, session pooler, transaction pooler). The direct host doesn't resolve from Lightdash Cloud. There's no hint about this in the Lightdash connection form. A note like "If using Supabase, use the shared pooler host" would save people a confusing `ENOTFOUND` error.

### `lightdash lint` misses errors that `deploy` catches

A `count` metric without an explicit `sql` field passes `lightdash lint` but fails during `lightdash deploy`:

```
ERROR> my_model : Metric "my_count" in table "my_model" is missing a sql definition
```

Lint should catch everything deploy would reject.

### `lightdash deploy --create` needs a `--project-name` flag

The first deploy prompts interactively for a project name. Can't be piped or passed as a flag, which is annoying for scripts and CI.

### CLI version warning fires when you're ahead

```
Warning: CLI (0.2467.0) is running a different version than Lightdash (0.2459.3)
```

`brew install` pulls the latest release, which is often ahead of cloud. The warning says "consider upgrading" when you're already newer. Should only warn when the CLI is behind.

---

## Open questions (for the Lightdash eng team)

- **Does the setup wizard's "Save & Test" actually create the project?** If so, do you need `lightdash deploy --create` at all, or just `lightdash deploy`? If both create projects, do you end up with duplicates? The interaction between the wizard flow and the CLI `--create` flag is unclear.
- **What does "Save & Test" actually test?** It seems to save the config and maybe check the connection at the TCP level, but it doesn't surface errors like wrong SSL mode, bad credentials, or unreachable hosts in any useful way. What's it doing under the hood?
- **Should `require` SSL mode work with Supabase?** In standard Postgres semantics, `require` means "encrypt but don't verify certs" — so it should work with Supabase's pooler. The fact that only `no-verify` works suggests Lightdash's `require` mode might be doing cert verification (which would be `verify-ca` behavior). Is this intentional?

---

## Resources

- [Lightdash YAML guide](https://docs.lightdash.com/guides/lightdash-yaml)
- [Lightdash CLI docs](https://docs.lightdash.com/guides/cli/how-to-install-the-lightdash-cli)
- [Lightdash metrics reference](https://docs.lightdash.com/references/metrics)
- [Lightdash dimensions reference](https://docs.lightdash.com/references/dimensions)
- [Supabase docs](https://supabase.com/docs)
