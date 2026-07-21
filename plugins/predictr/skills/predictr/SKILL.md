---
name: predictr
description: Drive the predictr.io data-analytics platform from the shell — connections, datasets, models, workflows, and the MBA / RFM / forecast analysis slates — using the predictr-cli (or raw REST). Trigger whenever the user mentions predictr.io, predictr-cli, or asks to list/create/fit/predict against connections, datasets, models, workflows, market-basket analysis, RFM clustering, or sales forecasting on a predictr.io tenant. Also trigger when the user wants to upload a CSV to a fileupload connection, schedule a workflow, or call a predictr.io endpoint by hand.
---

# predictr.io

predictr.io is a multi-tenant data-analytics SaaS. This skill is for **consuming** a predictr.io tenant from a shell — not for developing the platform itself. The primary tool is `predictr-cli` (a thin wrapper over the REST API at `https://api.predictr.io`); fall back to raw `curl` only when an endpoint isn't exposed by the CLI.

## When to use this skill

Trigger on phrases like:

- "list my connections / datasets / models / workflows"
- "create an MBA / RFM / forecast analysis"
- "fit / train / predict on …"
- "upload this CSV to predictr"
- "run / schedule a workflow"
- "what's my predictr plan / capabilities"
- "I'm new to predictr — help me get started"
- raw mentions of `predictr-cli`, `predictr.io`, `PREDICTR_API_KEY`

Do **not** trigger when the user is editing the predictr.io source code (mr-slate, dino, wilma, bamm-bamm, bedrock) — that's platform development, not platform usage.

## Preflight: check the environment once per session

Before the first real call, verify three things in one shot:

```bash
command -v predictr-cli >/dev/null && predictr-cli --version
echo "org=${PREDICTR_ORG:-<unset>} url=${PREDICTR_API_URL:-https://api.predictr.io}"
[ -n "$PREDICTR_API_KEY" ] || [ -n "$PREDICTR_BEARER_TOKEN" ] && echo "auth: ok" || echo "auth: MISSING"
```

If `predictr-cli` is missing: `uv tool install git+https://github.com/predictr-io/predictr-cli`.

If auth is missing, ask the user. **Never** print, log, or write a token to any file. If you must pass it for one command, use the env var inline (`PREDICTR_API_KEY=… predictr-cli …`) and don't echo it back.

If `PREDICTR_ORG` is unset, ask which org to use (or pass `--org-name` per call). Most commands require it.

Cheapest connectivity smoke test (no org needed): `predictr-cli meta info`.

## Detect: is this a first-time user?

After the preflight succeeds, probe the tenant. **A brand-new org is not empty of connections** — mr-slate's `org.created` webhook auto-provisions a default fileupload connection called "File Uploads" with no tables. So "no connections" is the wrong heuristic; "no tables anywhere" is the right one.

```bash
# Snapshot every connection's table count in one go.
predictr-cli connections list --output json \
  | jq -r '.[] | "\(.conn_id)\t\(.conn_name)\t\(.conn_type)"' \
  > /tmp/predictr-conns.tsv

while IFS=$'\t' read -r cid cname ctype; do
  n=$(predictr-cli connections tables "$cid" --output json 2>/dev/null | jq 'length // 0')
  echo "$cid  $cname  $ctype  tables=$n"
done < /tmp/predictr-conns.tsv
```

Decision rules:

- **No connections, or only the default "File Uploads" fileupload connection with 0 tables** → treat as a first-time user. Walk them through the **First-time flow** (next section).
- The default "File Uploads" connection looks like: `conn_name == "File Uploads"`, `conn_type == "fileupload"`, `tables == []`. Recognise it and **use it** in Step 1 — don't create a second fileupload connection.
- **Tables exist somewhere** but they're asking "how do I get started" / "what should I do next" — start the first-time flow from whichever step matches their existing state (skip connection setup, jump straight to dataset matching).
- **Tables exist** and they have a specific operational request → skip the first-time flow, use the **Reference** sections lower down.

## First-time user flow

This is the opinionated walk-through to take a brand-new predictr.io tenant from "empty" to "first prediction". Follow the order. Pause for the user to confirm at each step — don't barrel through.

### Step 1 — Pick the right connection

**Almost always: use the pre-provisioned "File Uploads" connection.**

Every new org has a fileupload connection auto-created by mr-slate's `org.created` webhook. It's empty (no SQLite file in S3 yet, no tables) but the connection row exists. **Do not create a second fileupload connection.** Use the existing one — find its `conn_id` from `predictr-cli connections list` (look for `conn_name == "File Uploads"` and `conn_type == "fileupload"`).

If the user has a CSV / TSV on disk, upload it to that connection:

```bash
# Pull the existing fileupload connection's id
DEFAULT_CONN=$(predictr-cli connections list --output json \
  | jq -r '.[] | select(.conn_type == "fileupload" and .conn_name == "File Uploads") | .conn_id' \
  | head -1)

# --table-name is REQUIRED — it's the destination table name on the connection.
predictr-cli connections upload "$DEFAULT_CONN" --file ./their-data.csv --table-name their_data
```

The upload endpoint imports the CSV into the connection's SQLite blob in S3 **and** auto-crawls the new tables synchronously. There is no separate crawl step for fileupload — see Step 2. (Large files can be sent in chunks via `--part-number` / `--total-parts`; the short flag for `--file` is `-F`.)

**Only create a different connection** if the user explicitly tells you their data lives in a cloud warehouse (Snowflake / BigQuery / Redshift / Athena). In that case:

| Type | When to suggest it |
|---|---|
| **snowflake** | Production data warehouse on Snowflake. |
| **bigquery** | GCP, BigQuery datasets. |
| **redshift** | AWS, Redshift cluster. |
| **athena** | AWS, Athena over S3 / Glue catalog. |

Discover the required fields:

```bash
predictr-cli connections create --help
```

For cloud sources the user will need credentials, an endpoint / account / region, and (for warehouse types) a schema / dataset. Help them assemble these but **never** persist credentials to a file you created. Pass via inline JSON, or by pointing them at a file *they* wrote.

Before saving, dry-test the config:

```bash
predictr-cli connections test-config -d '<config-json>'
```

If that succeeds, create the connection:

```bash
predictr-cli connections create -f conn.json
# or
predictr-cli connections create -d '<config-json>'
```

When creating, **let the server assign `conn_id`** — never include a `conn_id` field in the create payload. If the create payload includes an existing `conn_id`, the API will reject with `Connection already exists` (line 148–149 of `blueprints/app_connections.py`).

### Step 2 — Scan metadata (crawl)

How metadata gets discovered depends on the connection type:

- **fileupload / sqlite3** — crawled **automatically and synchronously** the moment a CSV is uploaded (the upload endpoint runs the crawler inline). **Never call `predictr-cli connections crawl <conn-id>` on a fileupload connection** — the API rejects it with `ValueError("Crawling is performed automatically for built-in databases")`.
- **snowflake / bigquery / redshift / athena** — crawl is async and needs to be kicked off explicitly:

  ```bash
  predictr-cli connections crawl <conn-id>
  ```

  For large schemas it can take seconds to minutes.

Either way, once metadata exists, verify what was found:

```bash
predictr-cli connections tables  <conn-id> --output table                          # tables / views
predictr-cli connections columns <conn-id> --table <table-name> --schema <schema>  # column types (--table/-t and --schema/-s both required)
```

Surface the table list to the user. Often they have **dozens** of tables and only one or two are relevant — let them point at the candidate(s) rather than guessing.

### Step 3 — Peek at the data

For any candidate table, fetch a sample to see actual values:

```bash
predictr-cli connections table-sample <conn-id> <table-name> --schema <schema>
```

`--schema`/`-s` is required (e.g. `--schema main` for sqlite3/fileupload). The command returns a fixed-size key-value sample; there is no row-count flag.

Or, once a dataset exists:

```bash
predictr-cli datasets sample <dataset-id>            # add --feature <col> to sample a single column
```

Read the sample. You want to answer:

- What grain is each row at? (one customer, one order line, one timestamped event, one daily aggregate?)
- What columns are identifiers (high-cardinality strings/ints)?
- What columns are timestamps?
- What columns are numeric measures (price, revenue, quantity)?
- What columns are categorical (status, category, region)?

This is the input to the next step.

### Step 4 — Match the data to an analysis type

Three analysis slates are available. Each needs a specific data shape. Use the sample + column types to choose.

#### Market Basket Analysis (`mba`) — "what sells together"

**Needs**: a **container / detail** structure — long-format rows where each row is one item within a transaction.

| Required column | What it looks like |
|---|---|
| Basket / container id | `order_id`, `invoice_id`, `basket_id`, `transaction_id` — repeats across multiple rows |
| Item identifier | `sku`, `product_code`, `item_id`, `product_name` — what was in the basket |

**Optional**: quantity, unit price, line total, timestamp.

**Pattern-match heuristics** when scanning a column list:
- Two columns whose names include `order` / `invoice` / `basket` and `item` / `sku` / `product` — strong MBA candidate
- Tables named `order_lines`, `invoice_lines`, `basket_items`, `transaction_items`, `pos_lines`, `line_items` — almost certainly MBA-shaped

**Anti-pattern**: a one-row-per-order table where items are in JSON / comma-separated columns. Not suitable without flattening first.

#### Customer Clustering (`rfm`) — "who are my customers, really"

**Needs**: per-customer transaction history. Each row is one transaction (or one customer-day aggregate) with three signals: who, when, and how much.

| Required column | What it looks like |
|---|---|
| Customer id | `customer_id`, `user_id`, `account_id`, `loyalty_card`, `email_hash` |
| Transaction timestamp | `transaction_date`, `order_date`, `created_at` |
| Monetary amount | `amount`, `total`, `revenue`, `order_value`, `line_total` |

**Pattern-match heuristics**:
- Any table with `customer_*` + a date column + a numeric value column
- The MBA-shaped tables above will *also* work for RFM if they carry the customer id

**Anti-pattern**: anonymous/guest-checkout-heavy tables with no usable customer id will cluster mostly into "anonymous".

#### Forecasting (`forecast`) — "what will sales look like next"

**Needs**: a time series of numeric values. Each row is one observation at one point in time, optionally with grouping dimensions.

| Required column | What it looks like |
|---|---|
| Timestamp | `date`, `day`, `week`, `month`, `transaction_date` |
| Numeric value to forecast | `revenue`, `units`, `sales`, `volume`, `quantity` |

**Optional**: dimensions to forecast per-group (per-store, per-region, per-product).

**Pattern-match heuristics**:
- Aggregated tables with one numeric column + one date column — directly suitable
- Granular transaction tables (like the MBA / RFM ones above) — *also* work, will be auto-aggregated by date

**Anti-pattern**: very sparse / very short series (< ~12 timepoints) — forecast quality will be poor regardless of model. Tell the user.

#### When multiple slates fit

A typical transactional table (`order_lines` with customer + order + product + amount + date) fits **all three**. Suggest the one most aligned with their stated business question:

| User says they want to know… | Slate |
|---|---|
| "what sells with what" / cross-sell / bundle recommendations | mba |
| "who my customers are" / segments / retention / lifetime value | rfm |
| "how much I'll sell" / demand / capacity / cash forecast | forecast |

If they don't have a stated question, **ask**. Don't pick for them.

### Step 5 — Let the server guess a schema

Once a table is identified, ask predictr to propose a config. **Both `--schema-name` and `--table-name` are required** — the API 422s ("Both schema_name and table_name query parameters are required") if either is missing:

```bash
predictr-cli mba      guess-schema <conn-id> --schema-name main --table-name <table> > mba.json
predictr-cli rfm      guess-schema <conn-id> --schema-name main --table-name <table> > rfm.json
predictr-cli forecast guess-schema <conn-id> --schema-name main --table-name <table> > sf.json
```

(For fileupload / sqlite3 the schema is `main`.)

The server inspects columns + samples and proposes a complete analysis config. **This is the recommended starting point** — the heuristics it uses are richer than column-name matching.

**guess-schema needs a reachable LLM backend (AWS Bedrock `Converse`).** Against a local dev API without Bedrock it 500s (`ClientError … 405 … Converse operation`). When it's unavailable, hand-author the config instead — the required fields per slate are listed in Step 6.

Show the proposed JSON to the user and let them tweak field mappings before creating.

### Step 6 — Create, fit, predict

Once the config is right:

```bash
predictr-cli <slate> create -f <config>.json    # returns an analysis id
predictr-cli <slate> fit <analysis-id>          # async — submits a Batch job
```

Minimum required create fields per slate (beyond `name` / `connection_id` / `schema_name` / `table_name`) — useful when hand-authoring because guess-schema is unavailable:

| Slate | Required field(s) |
|---|---|
| mba | `basket_identifier_column`, `item_identifier_column` |
| forecast | `date_column`, `value_column` |
| rfm | customer / date / monetary columns (see `guess-schema` output or probe with a minimal payload) |

If you're unsure of the field names, send a minimal payload once and read the 422 — the Pydantic validation error names every missing required field. That's a cheap, non-destructive way to discover the shape; don't loop firing variations.

Poll status until done — analyses expose a top-level `status` and `build_history`, **not** a `runs` array:

```bash
predictr-cli <slate> get <analysis-id> | jq '{status, model_error, build_history}'
```

When `status` is `fitted` (and `model_error` is empty), promote the model:

```bash
predictr-cli <slate> set-active-model <analysis-id> --model-id <fit-model-id>
```

Then predict:

```bash
predictr-cli <slate> predict <analysis-id> ...
# — exact args differ per slate; check `predict --help`. Known flags:
#   rfm predict      -R/--recency  -F/--frequency  -M/--monetary  [--ml-model-id]
#   forecast predict -p/--periods (14)  --include-history/--no-include-history  --max-history-days (365)  [--ml-model-id]
#   mba predict      --ml-model-id   (query-param driven; defaults to the active model)
```

For MBA specifically, after fit there are also `rules` and `items` commands worth showing the user — they're often what people actually want to see. **Neither takes `--output`** — they emit JSON; pipe to `jq`. `rules` supports `--sort-field` / `--sort-order` / `--filter` / `--page` / `--page-size` / `--ml-model-id`; `items` only takes `--ml-model-id`.

```bash
predictr-cli mba rules <analysis-id> --sort-field lift --sort-order desc \
  | jq -r '.[] | "\((.given_items|join("+"))) => \((.suggested_items|join("+")))\tconf=\(.confidence)\tlift=\(.lift)"' | head
predictr-cli mba items <analysis-id> | jq . | head
```

### What "done" looks like for the first-time flow

A clean first-time onboarding ends with the user having:

1. ✅ A working connection
2. ✅ A crawled schema with at least one candidate table
3. ✅ A created analysis (mba / rfm / forecast) with a sensible config
4. ✅ A successful fit
5. ✅ Their first prediction or rules output

Don't declare "done" until they've seen useful output from step 5. The whole point is the answer, not the pipeline.

---

## The mental model (reference)

```
connection ──▶ dataset ──▶ model | analysis (mba / rfm / forecast)
                                         │
                                         └─▶ predict
workflow = scheduled / orchestrated runs of the above
```

- **Connection** — credentials + endpoint to a data source (snowflake, redshift, fileupload CSV, etc.). Crawled to discover tables/columns.
- **Dataset** — a SELECT-style view over a connection (table refs + transformations + schema).
- **Model** — generic ML model fit on a dataset.
- **Analysis slates** — opinionated pipelines for one algorithm family:
  - **mba** — market-basket / association rules
  - **rfm** — recency-frequency-monetary clustering
  - **forecast** — time-series forecasting

  Each has `create → fit → set-active-model → predict`. Each fit produces a model artefact; `set-active-model` is what `predict` resolves against.
- **Workflow** — orchestrated run definition; can be scheduled (`workflows schedule …`) or fired ad-hoc (`workflows run …`).

## Command index

```
predictr-cli meta info | transformations | schema-model | schema-dataset
predictr-cli capabilities                # current org's plan/limits
predictr-cli analyses                    # all analyses across slates

predictr-cli connections   list|get|create|update|delete|test|test-config|crawl|upload|tables|columns|table|table-sample
predictr-cli datasets      list|get|create|update|delete|sample|analyze
predictr-cli models        list|get|create|update|delete|predict
predictr-cli workflows     list|get|create|update|delete|run|history|schedule|unschedule|zoneinfo

predictr-cli mba           list|get|create|update|delete|fit|set-active-model|rules|items|predict|guess-schema
predictr-cli rfm           list|get|create|update|delete|fit|set-active-model|predict|guess-schema
predictr-cli forecast list|get|create|update|delete|fit|set-active-model|predict|guess-schema|holidays
```

`<noun> --help` and `<noun> <verb> --help` are authoritative. Run them when in doubt.

## Standard workflows (for experienced users)

Most user requests reduce to one of these flows. Pick the closest, then improvise.

### Inspect what's there

```bash
predictr-cli connections list --output table
predictr-cli datasets    list --output table
predictr-cli models      list --output table
predictr-cli analyses                       # cross-slate index
```

`--output table` is friendlier for humans; default JSON pipes into `jq`.

### Build a JSON payload

For `create` / `update` calls, payloads are JSON. Three equivalent inputs:

```bash
predictr-cli connections create -f conn.json
predictr-cli connections create -d '{"conn_name":"prod","conn_type":"snowflake", ...}'
cat conn.json | predictr-cli connections create -f -
```

Discover the shape:

- `predictr-cli meta schema-model` / `meta schema-dataset` — JSON Schema for the resource.
- `predictr-cli meta transformations` — supported dataset transforms.
- `predictr-cli <slate> guess-schema <conn-id>` (mba / rfm / forecast) — the server proposes a schema from the connection's data. Use this as a starting point and edit, don't hand-write.

### Upload a CSV (fileupload connection)

```bash
predictr-cli connections create -d '{"conn_name":"my-upload","conn_type":"fileupload"}'
# → grab the conn_id from the response
predictr-cli connections upload <conn-id> --file ./data.csv --table-name my_data  # --table-name is required
# No crawl step for fileupload — upload auto-crawls synchronously (crawl would be rejected).
predictr-cli connections tables <conn-id>
```

### Fit an analysis (async)

`fit` submits a Batch job and returns immediately with a run/job id. Poll for status:

```bash
predictr-cli mba fit <id>                          # kicks off, returns job info
predictr-cli mba get <id> | jq '.runs[-1]'         # most recent run state
# or, if the run is exposed as a workflow run:
predictr-cli workflows history <wf-id>
```

Container size for fit jobs: `--model-build-infra s|m|l|xl|xxl` (cheaper → bigger). Default is fine for small data; bump if a fit times out or OOMs.

After a successful fit, promote it before predicting:

```bash
predictr-cli mba set-active-model <id> --model-id <fit-model-id>
predictr-cli mba predict <id> ...
```

### Predict

Each slate's `predict` takes its own input shape — check `--help`. Generic models:

```bash
predictr-cli models predict <model-id> -f features.json
```

### Schedule a workflow

```bash
predictr-cli workflows zoneinfo | head -20         # pick a tz
predictr-cli workflows schedule <wf-id> -d '{
  "cron": "0 6 * * *",
  "timezone": "Europe/London"
}'
predictr-cli workflows unschedule <wf-id>          # to remove
```

## Output handling

Default is JSON, pretty-printed only when stdout is a TTY — so pipes Just Work:

```bash
predictr-cli connections list | jq -r '.[] | "\(.conn_id)\t\(.conn_name)"'
```

Use `--output table` for terminal viewing, `--output yaml` if the user prefers it. Never reformat JSON output by hand when `jq` will do.

## Calling the REST API directly

When something isn't surfaced by the CLI (or you need to debug), hit the API with `curl`:

```bash
curl -sS -H "x-api-key: $PREDICTR_API_KEY" \
     "${PREDICTR_API_URL:-https://api.predictr.io}/${PREDICTR_ORG}/connections" | jq .
```

Auth header is **either** `x-api-key: <key>` **or** `Authorization: Bearer <jwt>`, never both. The path is `/<org-name>/<resource>...`. `meta` endpoints are unscoped: `/meta`, `/meta/transformations`.

If a request 401s, the token's expired or invalid — ask the user for a fresh one, don't try to refresh.

## Safety rules

- **NEVER DELETE TO RECOVER FROM AN ERROR.** This is the single most important rule. When any command fails — duplicate ID, validation error, 500, timeout, anything — **stop**. Report the failure verbatim. Ask the user what to do. Do not call `delete`, `update`, or any "let me try again from scratch" action that destroys the state mid-debug. The state at the point of failure is the only thing that lets the user (or them and the platform team) work out what went wrong. Wiping it makes the bug uninvestigatable. If the user asks you to clean up after diagnosing, that's fine — but only on explicit ask.
- **Destructive commands** (`delete`, `unschedule`, overwriting via PUT): list & show the user *exactly* what you're about to remove, name and id and resource type, then wait for confirmation. Never bulk-delete in a loop without explicit per-item confirmation or a clear blanket go-ahead. Never delete the default "File Uploads" connection that mr-slate auto-provisions — even if the user asks, push back because it's not easily recreatable without a fresh org.
- **Credentials**: connection configs may include database passwords / service-account JSON / API tokens. **Never** write these to a file you created; if the user wants to template a config, have them put the secret in their own file (or pipe via stdin) and reference it. Never echo secrets back in chat. If a secret leaks into a tool output you produced, tell the user immediately and ask them to rotate it.
- **Tokens**: never echo, log, write to disk, or include in commit messages or PR bodies.
- **Cross-org**: if `--org-name` is passed explicitly, use it; otherwise rely on `PREDICTR_ORG`. Do not silently switch orgs.
- **Retries**: the CLI already retries 5xx/429 with backoff. Don't wrap it in your own retry loop. For known-bad input (4xx), fix the input — don't re-fire. For *unknown* errors, surface them and stop; don't loop trying variations.
- **Capacity**: before creating connections / datasets / models in bulk, check `predictr-cli capabilities` so you don't trip the plan limit mid-run.
- **Pace yourself in the first-time flow**: don't auto-run the next step until the user has acknowledged the previous one. A user new to the platform needs to see *what just happened* before the next thing happens.
- **conn_id is server-assigned**: never include a `conn_id` in a connection create payload. If you find yourself wanting to, you're trying to recreate something that already exists — stop and use the existing one.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | success |
| 1 | user/config error (missing org, bad input) |
| 2 | API error (4xx) |
| 3 | network error / retries exhausted |

Use these in scripts to branch on outcome.

## Diagnostics checklist

When something fails, check in this order:

1. `predictr-cli meta info` — server reachable?
2. `predictr-cli capabilities` — auth working, plan healthy?
3. Re-run with `-v` (verbose: prints request URL on stderr).
4. For a connection issue: `predictr-cli connections test <conn-id>` (or `test-config` for an unsaved config).
5. For a fit/training issue: `<slate> get <id>` and inspect the most recent run's status + error message.
6. Only then escalate to raw `curl` to isolate CLI vs API.
