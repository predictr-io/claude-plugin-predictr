# claude-plugin-predictr

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin for [predictr.io](https://predictr.io) — drive the platform from your shell via [`predictr-cli`](https://github.com/predictr-io/predictr-cli).

## Install

In Claude Code:

```
/plugin marketplace add predictr-io/claude-plugin-predictr
/plugin install predictr@predictr
```

## What it does

Installs a skill (`predictr`) that triggers whenever you mention predictr.io concepts — connections, datasets, models, workflows, and the MBA / RFM / forecast analysis slates. Claude then drives `predictr-cli` (or raw REST) on your behalf.

You'll need `predictr-cli` installed and the following env vars exported in the shell session Claude Code is using:

```bash
uv tool install git+https://github.com/predictr-io/predictr-cli

export PREDICTR_API_URL="https://api.predictr.io"
export PREDICTR_ORG="<your-org-name>"
export PREDICTR_API_KEY="<your-api-key>"
# or, for short-lived sessions:
export PREDICTR_BEARER_TOKEN="<your-bearer-token>"
```

Both env vars and API key can be copied from the welcome page of your predictr.io tenant.

## License

MIT — see [LICENSE](LICENSE).
