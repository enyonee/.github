# .github

Reusable GitHub Actions workflows for enyonee projects.

## Workflows

### python-ci.yml

Lint (ruff) + test (pytest) + Telegram notification.

```yaml
jobs:
  ci:
    uses: enyonee/.github/.github/workflows/python-ci.yml@v1
    with:
      project: MyProject
      test-command: "uv run pytest tests/ -x -q --tb=short"
    secrets: inherit
```

**Inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `project` (required) | - | Project name for notifications |
| `python-version` | `3.12` | Python version |
| `uv-sync-args` | `--extra dev` | Extra args for `uv sync` |
| `uv-sync-env` | `""` | Env vars for sync/test (`KEY=VAL` per line) |
| `test-command` | `uv run pytest tests/ -x -q --tb=short` | Pytest command |

**Required secrets:** `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`
