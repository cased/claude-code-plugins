# Piglet Plugin for Claude Code

A Claude Code skill for managing PostHog resources via the `piglet` CLI.

## What it does

This plugin enables Claude to help you manage PostHog feature flags, cohorts, dashboards, and insights through natural language requests like:

- "What feature flags do we have?"
- "Create a flag called dark-mode with 50% rollout"
- "Disable the beta-feature flag"
- "List all our cohorts"
- "Export all flags as JSON"

## Prerequisites

1. Install the piglet CLI:
   ```bash
   uv tool install cased-piglet
   ```

2. Configure PostHog credentials:
   ```bash
   export POSTHOG_API_KEY="phx_your_key"
   export POSTHOG_PROJECT_ID="12345"
   ```

## Installation

Install via Claude Code plugin system:
```
/plugin add cased/claude-code-plugins --path plugins/piglet
```

## Usage

Once installed, Claude will automatically use piglet when you ask about PostHog resources. Examples:

- "List my feature flags"
- "Is the new-checkout flag enabled?"
- "Roll out dark-mode to 100%"
- "Create a cohort for beta users"
- "What dashboards do we have?"

## Links

- [Piglet CLI Repository](https://github.com/cased/piglet)
- [PostHog](https://posthog.com)
