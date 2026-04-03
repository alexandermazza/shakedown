# Shakedown

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that maps every user interaction in your app, identifies test gaps, and writes tests to close them.

Named after the nautical term — a **shakedown** is a thorough test of a new ship to find problems before it sets sail.

## What It Does

1. **Maps** every user interaction (API routes, UI actions, background jobs, webhooks)
2. **Catalogs** existing test coverage against the interaction map
3. **Prioritizes** uncovered paths by risk (user-facing + new code = highest)
4. **Writes** targeted tests for the gaps
5. **Reports** coverage before/after with any bugs found

## Install

Add to your project's `.claude/settings.json`:

```json
{
  "skills": [
    "https://github.com/alexandermazza/shakedown"
  ]
}
```

Or clone locally and reference the path:

```json
{
  "skills": [
    "/path/to/shakedown"
  ]
}
```

## Usage

In Claude Code:

```
/shakedown
```

Or describe what you want:

```
> Run a shakedown on this feature branch
> Map all interactions and find test gaps
> Test everything before we merge
```

## How It Works

Shakedown dispatches specialized agents in sequence:

```
Scope → Map → Catalog → Prioritize → Test → Report
```

Each agent gets only what it needs — the interaction map uses a one-line-per-interaction format to stay under 3K tokens, even for large apps.

### Example Output

```
## Shakedown Report

Scope: feature/slack-only-task-management (29 files changed)
Interactions mapped: 47
Coverage before: 28/47 (60%)
Coverage after: 43/47 (91%)
Tests added: 41
Bugs found: 0

### Remaining Gaps
- Route handlers (4) — require integration test infra (module-level DB imports)

### Recommendations
- Consider dependency injection for route handlers to improve testability
```

## Works With Any Stack

| Stack | Entry Points |
|---|---|
| Next.js / Express | API routes, page components, middleware |
| Django / Flask | URL routes, views, management commands, Celery tasks |
| Rails | Controllers, jobs, mailers |
| CLI tools | Command handlers, subcommands |
| Libraries | Public API surface |

## Token Efficiency

Shakedown is designed to minimize token usage:

- **Scopes to changed files** on feature branches (skips unchanged code)
- **One-line format** for the interaction map (not verbose descriptions)
- **Passes only gaps** to the test-writing agent (not the full map)
- **Batches by file** so related tests are written in one pass

## License

MIT
