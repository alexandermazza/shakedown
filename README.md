# Shakedown

**Two shakedowns in one skill — automated test generation, and runtime UX audit via browser.**

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that maps every interaction in your app and either:

- 🧪 **`code` mode** — audits test coverage, finds gaps, and writes tests to close them
- 🔍 **`ui` mode** — tours the running app in a browser, interacts with every control, and reports bugs, friction, and dead features from a user's perspective

Works with any stack — Next.js, React Native, Django, Rails, Go, and more.

Named after the nautical term — a **shakedown** is a thorough test of a new ship to find problems before it sets sail.

> **224 → 474 tests** in a single session on a real Expo/React Native app. 12 critical untested modules → 0. *(code mode)*
>
> **43 screenshots, 15 bugs / friction points, 3 dead pages identified** in a single session on a React+Express app. *(ui mode)*

## What It Does

### 🧪 `code` mode

```
Baseline → Scope → Map + Catalog (parallel) → Prioritize → Test (rounds) → Report
```

1. **Baselines** your test infrastructure — framework, mocks, conventions
2. **Maps** every user interaction — routes, UI actions, stores, workers, webhooks
3. **Catalogs** existing coverage in parallel with mapping
4. **Prioritizes** uncovered paths by risk — user-facing + complex logic = highest
5. **Writes** tests in rounds — pure logic first, stateful code second
6. **Reports** coverage before/after with remaining gaps and recommendations

### 🔍 `ui` mode

```
Baseline → Scope → Inventory → Tour (parallel) → Evaluate → Prioritize → Report
```

1. **Baselines** the running app — finds or starts your dev server, verifies Playwright MCP
2. **Scopes** the tour — full-app, scoped, or regression check on recent changes
3. **Inventories** every route + interactive surface by reading your router
4. **Tours** each surface in a real browser — navigates, clicks, types, captures screenshots, watches for console errors
5. **Evaluates** findings into bugs / friction / dead features / worth-keeping
6. **Reports** a tiered markdown writeup with embedded screenshots and concrete cut-or-keep recommendations

## Install

Add the marketplace and install the plugin:

```bash
claude plugins marketplace add https://github.com/alexandermazza/shakedown
claude plugins install shakedown@alexandermazza
```

Or install from a local clone:

```bash
git clone https://github.com/alexandermazza/shakedown.git
claude plugins marketplace add /path/to/shakedown
claude plugins install shakedown@alexandermazza
```

**For `ui` mode:** install the [Playwright MCP server](https://github.com/microsoft/playwright-mcp) in Claude Code so the skill can drive a real browser.

## Usage

Invoke the skill and it'll pick the right mode:

```
/shakedown
```

Or just describe what you want — the skill picks the mode from your phrasing:

```
# Code mode
Run a shakedown on this feature branch
Map all interactions and find test gaps
Test everything before we merge

# UI mode
Give me a UI shakedown
Tour the whole app and tell me what looks broken
Audit the UX before I plan this redesign
```

## How It Works

Both modes share the same philosophy: **enumerate the surface → figure out what's missing → produce a structured report**. Under the hood they use parallel agent dispatch at every opportunity.

### Parallel Architecture (`code` mode)

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

### Parallel Architecture (`ui` mode)

```
                  ┌─ Tour Agent 1 (dashboard, briefing) ─┐
  Baseline ──────►│  Tour Agent 2 (accounts, pipeline)   ├──► Evaluate
                  └─ Tour Agent 3 (scan, deep flows)  ───┘       │
                                                                 ▼
                                                              Prioritize → Report
```

- **Tours run in parallel per route cluster** — each agent owns a group of routes, drives its own browser session, captures screenshots
- **The controller merges findings** into a tiered A/B/C report
- **Screenshots live on disk**, referenced by filename in the report — they don't pollute the chat context

### Each Agent Gets a Complete Brief

No agent relies on another agent's context. Each one receives:
- Source code for the modules it's testing (code mode) OR the routes it's touring (ui mode)
- Mock setup patterns / dev server URL
- Specific test cases to write OR user hypotheses to specifically check
- Project conventions to follow

### Multi-Round Testing (`code` mode)

| Round | Targets | Mocking Complexity |
|-------|---------|-------------------|
| **1** | Pure functions, utilities, validators, algorithms | None or minimal |
| **2** | Store actions, DB layers, API clients, background tasks | Heavy — DB mocks, API mocks, state management |
| **3** *(if needed)* | Integration flows, edge cases on covered paths | Cross-module mocking |

After each round: run the full suite, fix failures, then proceed.

## Example Output (`code` mode)

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

## Example Output (`ui` mode)

From a real run against a React+Express sales tool:

```
UX Shakedown Report

Scope: Full-app tour — Distill (React + Express)
Routes visited: 5
Screenshots: 43

Bottom line
One great product (Scan) wrapped in four pages the user doesn't need
(Accounts, Pipeline, High Value, Dashboard). Recommend cutting all four.

Findings
┌────────────────────────┬───────┐
│ Tier                   │ Count │
├────────────────────────┼───────┤
│ A — real bugs          │ 5     │
│ B — confusing/wasteful │ 10    │
│ C — minor polish       │ 3     │
└────────────────────────┴───────┘

Tier-A Bugs (would ship broken today)
- Briefing deal card → /scan?company=... — param silently dropped
- Accounts row → /scan — DOMAIN/OWNER fields render empty
- /api/dashboard/high-value/summaries — returns 500 on every request
- Accounts table renders all 16,621 rows unvirtualized (~25s load)
- Empty section headers still render under filtered Briefing

Recommendation on what to cut
- /accounts — paginated list HubSpot already provides. Delete.
- /pipeline — 158 deals already in briefing + HubSpot. Delete.
- /high-value — all useful columns blank, summary API 500s. Delete.

Suggested next steps
1. Fix the three dead-end links (5-minute wins)
2. Remove the three dead pages from the router
3. Brainstorm what replaces them (sketched 4 directions)
```

## Works With Any Stack

| Stack | `code` mode entry points | `ui` mode dev server |
|---|---|---|
| **Next.js / Express** | API routes, page components, middleware | `next dev` / `npm run dev` |
| **Vite / React** | App routes, stores | `vite` (port 5173) |
| **React Native / Expo** | Screens, stores, background tasks, notifications | (code mode only) |
| **Django / Flask** | URL routes, views, management commands, Celery tasks | `manage.py runserver` |
| **Rails** | Controllers, jobs, mailers | `rails s` |
| **Go** | HTTP handlers, gRPC services, CLI commands | Any HTTP server |
| **CLI tools** | Command handlers, subcommands, config parsing | (code mode only) |
| **Libraries** | Public API surface, edge cases per function | (code mode only) |

## Token Efficiency

Shakedown minimizes token usage through its architecture:

| Technique | Savings | Mode |
|-----------|---------|------|
| Scope to changed files on feature branches | Skip unchanged code entirely | both |
| Run Map + Catalog in parallel | 2 agents, half the wall-clock time | code |
| Parallel tour agents | N agents, ~1/N wall-clock time | ui |
| One-line interaction format | Map stays under 3K tokens for most apps | code |
| Read source once, brief agents precisely | Pass function signatures, not raw files | code |
| Screenshots to disk, not transcript | Tour stays context-light | ui |
| Batch test-writing by shared mock setup | Agents reuse setup, don't re-discover | code |
| Pass only gaps to test agents | Not the full 147-interaction map | code |

## FAQ

**How is this different from just asking Claude to write tests / tour my app?**
Shakedown is systematic. In `code` mode it maps your entire interaction surface, cross-references existing coverage, prioritizes by risk, and dispatches parallel agents to close gaps methodically — 100-250+ tests in a session, not 5-10. In `ui` mode it tours *every* surface in a real browser, not just the one you mentioned, and surfaces dead features you didn't know about.

**Does `code` mode work with my test framework?**
Yes. Jest, Vitest, pytest, Go testing, RSpec, JUnit — the skill discovers your framework in the baseline step and generates tests that match your existing conventions and mock patterns.

**Does `ui` mode require Playwright?**
It uses the [Playwright MCP server](https://github.com/microsoft/playwright-mcp). If that MCP isn't available in your Claude Code setup, the `ui` mode will tell you and stop.

**Can I run `ui` mode on production or staging?**
You can, but by default it assumes localhost with a dev server. Point it at any URL if you want. Be aware that it will interact with real UI controls — probably not what you want against prod.

**What does `ui` mode output?**
A markdown report in `.ux-review/UX-REVIEW.md` with tiered findings (A/B/C), embedded screenshot references, and concrete recommendations for what to cut and what to keep. **It does not fix anything** — the next step is typically a brainstorm with the findings as input.

**How many tokens does it use?**
Full-app `code` shakedown on a medium app typically runs 2-3 rounds of parallel agents. Full-app `ui` shakedown depends on the number of routes and how deep the interactions go; a 5-route app finishes in one session, a 20-route app benefits from parallel tour agents.

**Can I run it on just part of my app?**
Yes. On feature branches it automatically scopes to changed files (`code`) or changed frontend routes (`ui`). You can also specify directories, routes, or flows.

**What if I already have good test coverage / a well-polished UI?**
Shakedown will report that. Both modes catalog what exists and flag only the genuine gaps.

## Contributing

Issues and PRs welcome. If you run the skill on a stack that isn't covered well, open an issue with:
- What stack/framework you used
- Which mode (`code` or `ui`)
- What worked and what didn't
- Any agent prompt changes that improved results

## License

[MIT](LICENSE)
