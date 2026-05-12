---
name: predictr
description: Drive the predictr.io data-analytics platform from the shell — connections, datasets, models, workflows, and the MBA / RFM / sales-forecast analysis slates — using the predictr-cli (or raw REST). Trigger whenever the user mentions predictr.io, predictr-cli, or asks to list/create/fit/predict against connections, datasets, models, workflows, market-basket analysis, RFM clustering, or sales forecasting on a predictr.io tenant. Also trigger when the user wants to upload a CSV to a fileupload connection, schedule a workflow, or call a predictr.io endpoint by hand.
---

# predictr.io

predictr.io is a multi-tenant data-analytics SaaS. This skill is for **consuming** a predictr.io tenant from a shell — not for developing the platform itself. The primary tool is `predictr-cli` (a thin wrapper over the REST API at `https://api.predictr.io`); fall back to raw `curl` only when an endpoint isn't exposed by the CLI.

## When to use this skill

Trigger on phrases like:

- "list my connections / datasets / models / workflows"
- "create an MBA / RFM / sales-forecast analysis"
- "fit / train / predict on …"
- "upload this CSV to predictr"
- "run / schedule a workflow"
- "what's my predictr plan / capabilities"
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

## The mental model

```
connection ──▶ dataset ──▶ model | analysis (mba / rfm / salesforecast)
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
  - **salesforecast** — time-series sales forecasting
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
predictr-cli salesforecast list|get|create|update|delete|fit|set-active-model|predict|guess-schema|holidays
```

`<noun> --help` and `<noun> <verb> --help` are authoritative. Run them when in doubt.

## Standard workflow

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
- `predictr-cli <slate> guess-schema <conn-id>` (mba / rfm / salesforecast) — the server proposes a schema from the connection's data. Use this as a starting point and edit, don't hand-write.

When asking the server to guess, then create:

```bash
predictr-cli mba guess-schema <conn-id> > mba.json
$EDITOR mba.json   # or programmatically tweak with jq
predictr-cli mba create -f mba.json
```

### Upload a CSV (fileupload connection)

```bash
predictr-cli connections create -d '{"conn_name":"my-upload","conn_type":"fileupload"}'
# → grab the conn_id from the response
predictr-cli connections upload <conn-id> --file ./data.csv
predictr-cli connections crawl <conn-id>           # refresh discovered schema
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

- **Destructive commands** (`delete`, `unschedule`, overwriting via PUT): list & show the user what you're about to remove, then confirm. Never bulk-delete in a loop without explicit per-item confirmation or a clear blanket go-ahead.
- **Tokens**: never echo, log, write to disk, or include in commit messages or PR bodies. If a token leaks into a tool output you produced, tell the user immediately and ask them to rotate it.
- **Cross-org**: if `--org-name` is passed explicitly, use it; otherwise rely on `PREDICTR_ORG`. Do not silently switch orgs.
- **Retries**: the CLI already retries 5xx/429 with backoff. Don't wrap it in your own retry loop. For known-bad input (4xx), fix the input — don't re-fire.
- **Capacity**: before creating connections / datasets / models in bulk, check `predictr-cli capabilities` so you don't trip the plan limit mid-run.

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
