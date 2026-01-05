---
name: piglet
description: Manage PostHog feature flags, cohorts, dashboards, and insights via the `piglet` CLI. Use when the user needs to list, create, update, or delete PostHog resources, check flag status, manage rollouts, or export configuration.
---

# Piglet - PostHog Management CLI

Use the `piglet` command-line interface to manage PostHog resources. This skill handles feature flags, cohorts, dashboards, insights, and projects across US Cloud, EU Cloud, or self-hosted PostHog instances.

## When to activate

- User asks about feature flags: "what flags do we have", "is X flag enabled", "create a flag for Y"
- User wants to manage rollouts: "roll out feature X to 50%", "disable the beta flag"
- User needs cohort information: "list our cohorts", "create a cohort for power users"
- User asks about dashboards or insights: "what dashboards exist", "show me insight X"
- User wants to export PostHog configuration: "export all flags as JSON"

## Prerequisites

1. Install piglet:
   ```bash
   uv tool install cased-piglet
   piglet --version
   ```

2. Configure authentication (one of):
   ```bash
   # Environment variables (recommended)
   export POSTHOG_API_KEY="phx_your_key"
   export POSTHOG_PROJECT_ID="12345"

   # Or config file: ~/.piglet/config.toml
   # api_key = "phx_your_key"
   # project_id = 12345
   # host = "us"
   ```

3. Verify setup:
   ```bash
   piglet projects list
   ```

## Core workflow

1. **Confirm context**
   - Check which PostHog project the user wants to work with
   - Verify credentials are configured: `piglet projects list`
   - Note the host if not US Cloud (eu, or custom URL)

2. **Feature flags**
   ```bash
   # List all flags
   piglet flags list
   piglet flags list --active      # only active
   piglet flags list --json        # for parsing

   # Get specific flag (by ID or key)
   piglet flags get my-feature-flag

   # Create a flag
   piglet flags create --key new-flag --name "New Feature" --rollout-percentage 50

   # Update rollout or status
   piglet flags update 123 --rollout-percentage 100
   piglet flags update 123 --active
   piglet flags update 123 --inactive

   # Delete (with confirmation)
   piglet flags delete 123
   piglet flags delete 123 --yes   # skip confirmation
   ```

3. **Cohorts**
   ```bash
   piglet cohorts list
   piglet cohorts list --static    # only static cohorts
   piglet cohorts get 456
   piglet cohorts create --name "Power Users"
   piglet cohorts create --name "Beta Testers" --static
   piglet cohorts update 456 --name "New Name"
   piglet cohorts delete 456
   ```

4. **Dashboards**
   ```bash
   piglet dashboards list
   piglet dashboards list --pinned
   piglet dashboards get 789
   piglet dashboards create --name "Engineering Metrics"
   piglet dashboards update 789 --pinned
   piglet dashboards delete 789
   ```

5. **Insights**
   ```bash
   piglet insights list
   piglet insights list --saved
   piglet insights get abc123      # by short_id
   piglet insights create --name "Daily Active Users"
   piglet insights update 101 --name "Weekly Active Users"
   piglet insights delete 101
   ```

6. **Projects**
   ```bash
   piglet projects list            # list all accessible projects
   piglet projects get 12345       # get project details
   ```

## Output formats

Piglet supports three output formats:

```bash
# Rich table (default) - human readable
piglet flags list

# JSON - for programmatic use
piglet flags list --json

# Plain tab-separated - for scripting
piglet flags list --plain
piglet flags list --plain | cut -f2    # extract just flag keys
```

## Working with different hosts

```bash
# US Cloud (default)
piglet flags list

# EU Cloud
piglet --host eu flags list

# Self-hosted
piglet --host https://posthog.mycompany.com flags list
```

## Examples

### Check if a flag exists and its status
```bash
piglet flags get my-feature-flag
```

### Create a gradual rollout
```bash
# Start at 10%
piglet flags create --key gradual-release --name "Gradual Release" --rollout-percentage 10

# Later, increase to 50%
piglet flags update <flag_id> --rollout-percentage 50

# Full rollout
piglet flags update <flag_id> --rollout-percentage 100
```

### Export configuration for backup
```bash
piglet flags list --json > flags-backup.json
piglet cohorts list --json > cohorts-backup.json
```

### Quick flag toggle
```bash
# Disable a flag immediately
piglet flags update 123 --inactive

# Re-enable
piglet flags update 123 --active
```

### Find flags by grepping plain output
```bash
# Find all flags with "beta" in the key
piglet flags list --plain | grep beta
```

## Common pitfalls & recoveries

- **"API key required"** - Set `POSTHOG_API_KEY` env var or use `--api-key`
- **"Project ID required"** - Set `POSTHOG_PROJECT_ID` env var or use `--project-id`
- **403 Permission denied** - API key lacks required scopes; generate a new key with appropriate permissions
- **Wrong project** - Verify project ID with `piglet projects list`, then use `--project-id`
- **EU data** - Remember to use `--host eu` for EU Cloud data

## Best practices

- Use environment variables for credentials rather than CLI flags
- Use `--json` output when parsing results programmatically
- Use `--plain` output when piping to other CLI tools
- Always verify the target project before making changes
- Use `--yes` flag in scripts to skip confirmation prompts

## References

- Command reference: [reference.md](reference.md)
- Piglet repository: https://github.com/cased/piglet
