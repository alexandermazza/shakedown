# Shakedown

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that maps every user interaction in your app, identifies test gaps, and writes tests to close them.

Named after the nautical term — a **shakedown** is a thorough test of a new ship to find problems before it sets sail.

## What It Does

```
Baseline → Scope → Map + Catalog (parallel) → Prioritize → Test (rounds) → Report
```

1. **Baselines** your test infrastructure — framework, mocks, conventions
2. **Maps** every user interaction — routes, UI actions, stores, workers, webhooks
3. **Catalogs** existing coverage in parallel with mapping
4. **Prioritizes** uncovered paths by risk — user-facing + complex logic = highest
5. **Writes** tests in rounds — pure logic first, stateful code second
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

Or clone locally and reference the path instead:

```json
{
  "skills": [
    "/path/to/shakedown"
  ]
}
```

> **Tip:** If the remote URL isn't detected as a skill, clone the repo and use the local path. Remote skill resolution depends on your Claude Code version.

## Usage

```
/shakedown
```

Or just describe what you want:

```
Run a shakedown on this feature branch
Map all interactions and find test gaps
Test everything before we merge
```

## How It Works

### Parallel Architecture

Shakedown is designed around parallel agent dispatch at every opportunity:

```
                  ┌─ Map Agent (reads source) ──┐
  Baseline ──────►│                              ├──► Prioritize
                  └─ Catalog Agent (reads tests) ┘       │
                                                         ▼
                  ┌─ Test Agent 1 (pure fns) ────┐
                  ├─ Test Agent 2 (DB layer) ────┤
                  ├─ Test Agent 3 (stores) ──────┤──► Run tests ──► Report
                  └─ Test Agent 4 (utils) ───────┘
```

- **Map and Catalog run simultaneously** — one reads source files, the other reads test files
- **3-5 test-writing agents run in parallel** per round, grouped by shared mock setup
- **Tests are written in rounds** — Round 1: pure functions (easy wins). Round 2: stateful code with mocking

### Each Agent Gets a Complete Brief

No agent relies on another agent's context. Each one receives:
- Source code for the modules it's testing
- Mock setup patterns from the baseline step
- Specific test cases to write
- Project conventions to follow

### Multi-Round Testing

| Round | Targets | Mocking Complexity |
|-------|---------|-------------------|
| **1** | Pure functions, utilities, validators, algorithms | None or minimal |
| **2** | Store actions, DB layers, API clients, background tasks | Heavy — DB mocks, API mocks, state management |
| **3** *(if needed)* | Integration flows, edge cases on covered paths | Cross-module mocking |

After each round: run the full suite, fix failures, then proceed.

## Example Output

From a real run against a 90+ component Expo/React Native app:

```
Shakedown Report

Scope: Full-app audit — RomaQuotidiana (Expo/React Native)
Interactions mapped: 147

Coverage
┌──────────────────────┬────────┬───────┐
│ Metric               │ Before │ After │
├──────────────────────┼────────┼───────┤
│ Test files           │ 18     │ 35    │
│ Test cases           │ 224    │ 474   │
│ Modules with tests   │ 13/30  │ 22/30 │
│ Zero-test critical   │ 12     │ 0     │
└──────────────────────┴────────┴───────┘

Tests added: 250 across 17 new test files
Bugs found: 0

Remaining Gaps
- Components (90+) — need React Native Testing Library
- Edge functions (3) — need deployment testing

Recommendations
- Add integration tests for the full quiz flow
- Consider dependency injection for route handlers
```

## Works With Any Stack

| Stack | Entry Points | State |
|---|---|---|
| **Next.js / Express** | API routes, page components, middleware | Zustand, Redux, React context |
| **React Native / Expo** | Screens, stores, background tasks, notifications | Zustand, MobX |
| **Django / Flask** | URL routes, views, management commands, Celery tasks | Django ORM |
| **Rails** | Controllers, jobs, mailers | ActiveRecord |
| **Go** | HTTP handlers, gRPC services, CLI commands | — |
| **CLI tools** | Command handlers, subcommands, config parsing | — |
| **Libraries** | Public API surface, edge cases per function | — |

## Token Efficiency

Shakedown minimizes token usage through its architecture:

| Technique | Savings |
|-----------|---------|
| Scope to changed files on feature branches | Skip unchanged code entirely |
| Run Map + Catalog in parallel | 2 agents, half the wall-clock time |
| One-line interaction format | Map stays under 3K tokens for most apps |
| Read source once, brief agents precisely | Pass function signatures, not raw files |
| Batch test-writing by shared mock setup | Agents reuse setup, don't re-discover |
| Pass only gaps to test agents | Not the full 147-interaction map |

## Contributing

Issues and PRs welcome. If you run the skill on a stack that isn't covered well, open an issue with:
- What stack/framework you used
- What worked and what didn't
- Any agent prompt changes that improved results

## License

[MIT](LICENSE)
