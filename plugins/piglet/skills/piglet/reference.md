# Piglet CLI Reference

Quick-access sheet for `piglet` commands. Pair with `SKILL.md` for workflow guidance.

## Install & setup

```bash
uv tool install cased-piglet
piglet --version
piglet --help
```

## Configuration

```bash
# Environment variables
export POSTHOG_API_KEY="phx_..."
export POSTHOG_PROJECT_ID="12345"
export POSTHOG_HOST="us"           # us, eu, or full URL

# Config file: ~/.piglet/config.toml
# api_key = "phx_..."
# project_id = 12345
# host = "us"
```

## Global options

| Flag | Description |
| --- | --- |
| `--api-key TEXT` | PostHog personal API key |
| `--host TEXT` | Host: `us`, `eu`, or full URL |
| `--project-id INT` | PostHog project ID |
| `--json` | Output as JSON |
| `--plain` | Output as tab-separated text |
| `--version` | Show version |
| `--help` | Show help |

## Feature flags

| Command | Description |
| --- | --- |
| `piglet flags list` | List all flags |
| `piglet flags list --active` | List only active flags |
| `piglet flags list --inactive` | List only inactive flags |
| `piglet flags list --limit N` | Limit results |
| `piglet flags get ID` | Get flag by ID |
| `piglet flags get KEY` | Get flag by key |
| `piglet flags create --key KEY` | Create flag |
| `piglet flags create --key KEY --name NAME` | Create with display name |
| `piglet flags create --key KEY --rollout-percentage N` | Create with rollout % |
| `piglet flags create --key KEY --inactive` | Create as inactive |
| `piglet flags update ID --name NAME` | Update name |
| `piglet flags update ID --rollout-percentage N` | Update rollout |
| `piglet flags update ID --active` | Enable flag |
| `piglet flags update ID --inactive` | Disable flag |
| `piglet flags delete ID` | Delete (with prompt) |
| `piglet flags delete ID --yes` | Delete (skip prompt) |

## Cohorts

| Command | Description |
| --- | --- |
| `piglet cohorts list` | List all cohorts |
| `piglet cohorts list --static` | List static cohorts only |
| `piglet cohorts list --dynamic` | List dynamic cohorts only |
| `piglet cohorts get ID` | Get cohort details |
| `piglet cohorts create --name NAME` | Create dynamic cohort |
| `piglet cohorts create --name NAME --static` | Create static cohort |
| `piglet cohorts create --name NAME --description DESC` | Create with description |
| `piglet cohorts update ID --name NAME` | Update name |
| `piglet cohorts update ID --description DESC` | Update description |
| `piglet cohorts delete ID` | Delete cohort |

## Dashboards

| Command | Description |
| --- | --- |
| `piglet dashboards list` | List all dashboards |
| `piglet dashboards list --pinned` | List pinned only |
| `piglet dashboards get ID` | Get dashboard details |
| `piglet dashboards create --name NAME` | Create dashboard |
| `piglet dashboards create --name NAME --pinned` | Create as pinned |
| `piglet dashboards update ID --name NAME` | Update name |
| `piglet dashboards update ID --pinned` | Pin dashboard |
| `piglet dashboards update ID --not-pinned` | Unpin dashboard |
| `piglet dashboards delete ID` | Delete dashboard |

## Insights

| Command | Description |
| --- | --- |
| `piglet insights list` | List all insights |
| `piglet insights list --saved` | List saved only |
| `piglet insights list --favorited` | List favorited only |
| `piglet insights get ID` | Get by numeric ID |
| `piglet insights get SHORT_ID` | Get by short ID |
| `piglet insights create --name NAME` | Create insight |
| `piglet insights update ID --name NAME` | Update name |
| `piglet insights update ID --saved` | Mark as saved |
| `piglet insights update ID --unsaved` | Mark as unsaved |
| `piglet insights delete ID` | Delete insight |

## Projects

| Command | Description |
| --- | --- |
| `piglet projects list` | List all projects |
| `piglet projects get ID` | Get project details |

## Output examples

```bash
# Default rich table
piglet flags list

# JSON for parsing
piglet flags list --json | jq '.[].key'

# Plain for scripting
piglet flags list --plain | awk -F'\t' '{print $2}'

# Export all flags
piglet flags list --json > flags.json
```

## Host shortcuts

| Shortcut | URL |
| --- | --- |
| `us` | https://us.posthog.com |
| `eu` | https://eu.posthog.com |
| Custom | Any full URL |

```bash
piglet --host eu flags list
piglet --host https://posthog.internal.co flags list
```
