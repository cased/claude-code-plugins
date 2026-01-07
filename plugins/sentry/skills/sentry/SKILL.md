---
name: sentry
description: Fetches Sentry error context using sentry-cli and helps diagnose and fix bugs in the current codebase. Use when the user wants to investgate or fix a Sentry error.
---

## Prerequisites

**Required:**
- [cased/sentry-cli](https://github.com/cased/sentry-cli) installed via uv:
  ```bash
  uv tool install git+https://github.com/cased/sentry-cli
  ```
- `SENTRY_AUTH_TOKEN` - Your Sentry authentication token
- `SENTRY_ORG` - Your Sentry organization slug

## Usage

```
/sentry <issue-id>
```

Examples:

- `/sentry PROJ-123`
- `/sentry 12345678`

## Extracting Issue IDs from URLs

If the user provides a Sentry URL instead of an issue ID, extract the numeric issue ID from the URL before using the CLI:

| URL Format                                                | Issue ID     |
| --------------------------------------------------------- | ------------ |
| `https://cased-sentry.sentry.io/issues/7170946994/`       | `7170946994` |
| `https://sentry.io/organizations/my-org/issues/12345678/` | `12345678`   |
| `https://sentry.io/issues/12345678/?project=123`          | `12345678`   |

The issue ID is the numeric portion in the `/issues/<id>/` path segment. Strip any trailing slashes or query parameters.

## Instructions

When this skill is invoked and there is a valid issue ID:

### Phase 1: Gather Context

1. **Fetch the Sentry error context** by running:

   ```bash
   sentry-cli context $ARGUMENTS
   ```

2. **Parse the Sentry output** and identify:
   - **Error message and type** - The core issue
   - **Key Application Frames** - Stacktrace frames marked with `>` are in-app code and the most likely location of the bug
   - **Breadcrumbs** - Events leading up to the error (read top to bottom chronologically)
   - **Secondary errors in breadcrumbs** - Look for other errors/warnings that may indicate related issues
   - **Tags and context** - Environment, user info, request data, etc.

### Phase 2: Ensure you understand the Codebase

3. **Look for any CLAUDE.md or AGENT.md files**:
   - These files contain documentation about the codebase and its architecture.
   - They will often provide important CLI commands you can use for testing
   - They will often provide information on code style preferences, conventions, and best practices.

4. **Use the kit skill to understand the relevant code**:
   - Run `/kit-cli symbols <file>` on files from the Key Application Frames to understand the structure
   - Run `/kit-cli search <function-name>` to find usages and callers of the failing function
   - Run `/kit-cli file-tree` if you need to understand the overall project structure
   - Run `/kit-cli dependencies <file>` to understand what the failing code depends on

5. **Read and trace the code path**:
   - Read the source files from the stacktrace
   - Follow the call chain from entry point to error location
   - Identify all inputs that flow into the failing code
   - Note any assumptions the code makes about data shape, state, or preconditions

6. **Trace upstream to the data source** (CRITICAL - do not skip):
   - When you find bad data (null, undefined, NaN, wrong type), ask: "Where did this value come from?"
   - Trace each problematic value back through the call chain to its origin
   - Continue tracing until you reach: an API response, user input, config, or hardcoded value
   - Document the full data flow path from source to crash site

### Phase 3: Root Cause Analysis

6. **Identify the root cause** (not just the symptom):
   - Ask: "Why did this specific input/state reach this code path?"
   - Ask: "What assumption was violated?"
   - Ask: "Where should this have been caught or handled?"
   - Consider:
     - Is this a data validation issue?
     - A race condition?
     - Missing null check?
     - Type coercion problem?
     - State management bug?
     - Something else?

7. **Explain the root cause clearly**:
   - Describe the chain of events that led to the error
   - Identify where the actual bug is (which may be upstream from where the error occurred)
   - Distinguish between the proximate cause (where it crashed) and the root cause (why the bad state existed)

### Phase 4: Propose a Proper Fix

8. **Design a fix that resolves the underlying issue**:
   - **Prefer fixes at the source** - Fix where bad data/state originates, not just where it crashes
   - **Avoid suppression** - Don't just wrap in try/catch or add null checks that hide the real problem
   - **Consider the contract** - If a function shouldn't receive null, fix the caller rather than accepting null
   - **Check for similar patterns** - Use kit to search for similar code that might have the same bug
   - **Consider defense in depth** - Sometimes fixing at multiple layers is appropriate

9. **Present the fix to the user**:
   - Summarize the root cause in 1-2 sentences
   - Explain why this fix addresses the root cause (not just the symptom)
   - Show the specific code changes needed
   - Note any related code that should be reviewed or updated
   - Offer to implement the fix

### Fix Quality Checklist

Before proposing a fix, verify:

- [ ] I traced the bad data upstream to its source (not just where it crashed)
- [ ] I can explain the full data flow from origin to crash site
- [ ] The fix addresses the root cause, not just the crash location
- [ ] The fix doesn't just suppress or swallow the error
- [ ] Similar patterns in the codebase have been checked
- [ ] The fix maintains the code's contract/API expectations
- [ ] Edge cases have been considered
- [ ] I fixed ALL secondary errors/warnings found in the breadcrumbs (not just documented them)

## Tips

- The `context` command output is markdown optimized for AI consumption
- Focus primarily on frames marked with `>` - these are in-app code
- Third-party library frames (without `>`) provide context but the fix is usually in your code
- Breadcrumbs are chronologically ordered - read from top to bottom to understand the sequence

## Anti-Patterns to Avoid

These "fixes" mask bugs when used **as the sole fix**:

| Anti-Pattern                             | Why It's Bad                                |
| ---------------------------------------- | ------------------------------------------- |
| `try { ... } catch { /* ignore */ }`     | Silently hides all errors                   |
| Null check only at crash site            | Allows bad data to flow through system      |
| Defensive defaults without fixing source | Masks data integrity issues                 |
| Optional chaining to suppress crashes    | Converts crashes into silent wrong behavior |

**Key distinction:** These same patterns become appropriate **after** fixing the root cause, as defense in depth. The anti-pattern is using them _instead of_ fixing the source, not using them _in addition to_ fixing the source.

## When Defense in Depth is Appropriate

After fixing the root cause, consider adding defensive fallbacks when:

- Data crosses a trust boundary (API responses, user input, external config)
- The function is called from multiple places with different data sources
- The codebase already uses this pattern elsewhere (match existing conventions)
- A failure here would be particularly severe

**Fix the source first, then add safety nets where appropriate for the codebase.**

## Match Existing Codebase Patterns

When implementing fixes:

- Study how similar problems are handled elsewhere in the repo
- Use the same error handling patterns the codebase already uses
- Don't introduce new abstractions or patterns unless necessary
- If the repo uses defensive fallbacks at similar boundaries, use them here too
- If the repo prefers strict validation at entry points, follow that pattern

The goal is a fix that looks like it belongs in the codebase, not one that stands out as different.

## Examples

See the `examples/` directory for detailed walkthroughs of diagnosing and fixing different classes of Sentry errors. Each example demonstrates bad approaches (fixing symptoms), good approaches (tracing to root cause), and comprehensive fixes.

### Error Classes

| Class | File | Description |
|-------|------|-------------|
| **Crashes & Unhandled Exceptions** | [crashes-and-unhandled-exceptions.md](examples/crashes-and-unhandled-exceptions.md) | Process dies or request fails hard: null dereferences, type errors, uncaught promise rejections, index out of bounds |
| **State & Concurrency Bugs** | [state-and-concurrency-bugs.md](examples/state-and-concurrency-bugs.md) | Timing and ordering issues: race conditions, stale closures, double submissions, lost updates |
| **Data Integrity & Serialization** | [data-integrity-and-serialization-bugs.md](examples/data-integrity-and-serialization-bugs.md) | Malformed or mismatched data: enum drift, JSON schema mismatches, date parsing, version skew |
| **Configuration & Environment** | [configuration-and-environment-bugs.md](examples/configuration-and-environment-bugs.md) | Misconfiguration issues: missing env vars, feature flag problems, region-specific config drift |
| **Dependency & Third-Party Failures** | [dependency-and-third-party-failures.md](examples/dependency-and-third-party-failures.md) | External system issues: payment provider timeouts, SDK breaking changes, API contract changes |
