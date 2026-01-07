# Sentry Plugin

Fetch Sentry error context and diagnose bugs with systematic root cause analysis.

## Prerequisites

- [cased/sentry-cli](https://github.com/cased/sentry-cli) installed via uv:
  ```bash
  uv tool install git+https://github.com/cased/sentry-cli
  ```
- Environment variables set:
  - `SENTRY_AUTH_TOKEN` - Your Sentry authentication token
  - `SENTRY_ORG` - Your Sentry organization slug

## Installation

```bash
claude plugin add cased/sentry
```

## Usage

```
/sentry <issue-id>
```

Examples:
- `/sentry PROJ-123`
- `/sentry 12345678`
- Paste a Sentry URL and the skill will extract the issue ID

## What It Does

1. **Fetches error context** from Sentry using sentry-cli
2. **Analyzes the error** - identifies key application frames, breadcrumbs, and secondary errors
3. **Traces upstream** to find the root cause (not just where it crashed)
4. **Proposes a fix** that addresses the underlying issue, not just the symptom

## Error Class Examples

The skill includes detailed examples for common error classes:

| Class | Description |
|-------|-------------|
| [Crashes & Unhandled Exceptions](skills/sentry/examples/crashes-and-unhandled-exceptions.md) | Null dereferences, type errors, uncaught promises |
| [State & Concurrency Bugs](skills/sentry/examples/state-and-concurrency-bugs.md) | Race conditions, stale closures, double submissions |
| [Data Integrity & Serialization](skills/sentry/examples/data-integrity-and-serialization-bugs.md) | Enum drift, schema mismatches, date parsing |
| [Configuration & Environment](skills/sentry/examples/configuration-and-environment-bugs.md) | Missing env vars, feature flag issues |
| [Dependency & Third-Party Failures](skills/sentry/examples/dependency-and-third-party-failures.md) | Timeouts, SDK bugs, API contract changes |

## Philosophy

This skill emphasizes **fixing root causes, not symptoms**:

- Don't just add null checks where it crashes
- Trace bad data back to its source
- Fix secondary errors found in breadcrumbs
- Match existing codebase patterns

See the [SKILL.md](skills/sentry/SKILL.md) for the full methodology.
