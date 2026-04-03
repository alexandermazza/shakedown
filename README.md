# Shakedown

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that maps every user interaction in your app, identifies test gaps, and writes tests to close them.

Named after the nautical term — a **shakedown** is a thorough test of a new ship to find problems before it sets sail.

## What It Does

1. **Baselines** existing test infrastructure (framework, mocks, conventions)
2. **Maps** every user interaction (API routes, UI actions, stores, background jobs)
3. **Catalogs** existing test coverage in parallel with mapping
4. **Prioritizes** uncovered paths by risk (user-facing + complex logic = highest)
5. **Writes** targeted tests in rounds — pure logic first, stateful code second
6. **Reports** coverage before/after with remaining gaps and recommendations

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

> **Note:** Remote skill URLs require Claude Code to fetch the skill at runtime. If the skill isn't detected, clone the repo locally and use a file path instead.

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

Shakedown dispatches specialized agents, parallelizing where possible:

```
Baseline → Scope → Map + Catalog (parallel) → Prioritize → Test (rounds) → Report
```

### Key Design Decisions

- **Map and Catalog run in parallel** — the interaction mapper reads source files while the coverage cataloger reads test files. Neither depends on the other.
- **Tests are written in rounds** — Round 1 covers pure functions (easy wins, high confidence). Round 2 tackles stateful code with mocking (stores, DB layers, API clients).
- **Multiple test-writing agents run in parallel** — grouped by shared mock setup (e.g., all DB tests in one agent, all store tests in another). 3-5 agents per round.
- **Each agent gets a complete brief** — source code, mock patterns, specific test cases, and project conventions. No agent relies on another agent's context.

### Example Output

```
## Shakedown Report

Scope: Full-app audit — RomaQuotidiana (Expo/React Native)
Interactions mapped: 147

### Coverage

| Metric               | Before | After |
|----------------------|--------|-------|
| Test files           | 18     | 35    |
| Test cases           | 224    | 474   |
| Modules with tests   | 13/30  | 22/30 |
| Zero-test critical   | 12     | 0     |

Tests added: 250 across 17 new test files
Bugs found: 0

### Remaining Gaps
- Components (90+) — require React Native Testing Library
- Edge functions (3) — require deployment testing

### Recommendations
- Add integration tests for quiz flow (generate → store → answer → SRS update)
- Consider dependency injection for route handlers
```

## Works With Any Stack

| Stack | Entry Points | State Management |
|---|---|---|
| Next.js / Express | API routes, page components, middleware | React context, Zustand, Redux |
| React Native / Expo | Screens, stores, background tasks, notifications | Zustand, MobX |
| Django / Flask | URL routes, views, management commands, Celery tasks | Django ORM |
| Rails | Controllers, jobs, mailers | ActiveRecord |
| CLI tools | Command handlers, subcommands | — |
| Libraries | Public API surface | — |

## Token Efficiency

Shakedown is designed to minimize token usage:

- **Scopes to changed files** on feature branches (skips unchanged code)
- **Runs Map + Catalog in parallel** (2 agents, not sequential)
- **One-line format** for the interaction map (not verbose descriptions)
- **Reads source files once, briefs agents precisely** (passes function signatures and mock setup, not raw files)
- **Batches test writing by shared mocking** so related tests reuse setup
- **Passes only gaps** to test-writing agents (not the full map)

## License

MIT
