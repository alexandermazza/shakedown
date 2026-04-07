---
name: shakedown
description: Map every user interaction in an app, identify test gaps, and write tests for uncovered paths. Use after completing a feature, before merging, or when you want confidence that nothing is broken.
---

# Shakedown

Map the full interaction surface of an application, identify what's tested and what isn't, then write targeted tests to close the gaps.

## When to Use

- After completing a feature branch (before merge/PR)
- When joining a project and want to understand coverage
- After a large refactor to verify nothing broke
- When the user asks to "test everything" or "find bugs"

## Process

```
0. Baseline     → Run existing tests, learn the test infra
1. Scope        → What changed? What's the blast radius?
2. Map+Catalog  → (parallel) Enumerate interactions AND audit existing tests
3. Prioritize   → Rank uncovered paths by risk
4. Test         → Write tests in rounds — pure logic first, stateful second
5. Report       → Summary of coverage, gaps, and findings
```

### Step 0: Baseline

Before mapping anything, understand the existing test infrastructure. This prevents wasted effort writing tests that don't match the project's patterns.

**Run existing tests first:**

```bash
# Discover the test command from package.json, Makefile, pyproject.toml, etc.
npm test          # or pytest, go test, etc.
```

**Discover test infrastructure:**
- Test framework (Jest, Vitest, pytest, Go testing, etc.)
- Test file location and naming convention (`__tests__/`, `*.test.ts`, `*_test.go`, etc.)
- Manual mocks directory (`__mocks__/`, `testutil/`, etc.)
- Global setup needs (e.g., `(global as any).__DEV__ = true` for React Native)
- State management testing patterns (Zustand: `store.getState().action()`, Redux: dispatch + selector)
- Mock patterns for DB, external APIs, platform modules

**Output:** A short infra summary — framework, test command, mock patterns, globals needed. This gets passed to every test-writing agent.

### Step 1: Scope

Before mapping the whole app, narrow the focus:

- **Is this a full-app audit or scoped to recent changes?**
  - If a feature branch: `git diff main...HEAD --name-only` to find changed files
  - If full audit: skip scoping, map everything
- **What are the entry points?** (API routes, pages, stores, background workers, webhooks)

Output a short scope statement: "Auditing [X files / Y features / the full app], focusing on [area]."

### Step 2: Map + Catalog (Parallel)

These two steps are independent — dispatch them as **parallel agents** to cut wall-clock time in half.

#### Agent A: Map Interactions

Dispatch an **Explore agent** to enumerate every user-facing interaction.

**Agent prompt template:**

```
Map every user interaction in this application. Work from: [directory]

Focus areas: [list entry points — API routes, pages, stores, workers, webhooks]

For each interaction, return ONE LINE in this format:
[CATEGORY] [ACTION] → [ENDPOINT/FUNCTION] | Edge: [edge cases]

Examples:
[ONBOARDING] Click "Connect Slack" → GET /api/connections/slack | Edge: OAuth failure, missing state
[QUIZ] Answer question → quizStore.submitAnswer() | Edge: timeout, already answered
[WORKER] Process reminders → processReminders() | Edge: missing connection, stale data

Group by category. Be exhaustive — check every route file, every button handler,
every store action, every background job.
Do NOT write paragraphs. One line per interaction. Keep it tight.
```

#### Agent B: Catalog Existing Coverage

Dispatch an **Explore agent** to audit existing test files independently.

**Agent prompt template:**

```
Analyze ALL test files in [test directory] and report:

For EACH test file:
1. How many test cases (describe blocks and it/test blocks)
2. What specific functions/behaviors are tested
3. What mocking patterns are used
4. What is NOT covered within each tested module

Also identify which source modules have ZERO test files.

List test infrastructure details:
- Framework and version
- Mock directory contents
- Global setup requirements
- Common mock patterns (module mocks, manual mocks, etc.)

Output a structured summary:
- Total test files and test cases
- Modules with tests vs modules with zero tests
- Key mock patterns to reuse
```

**Why parallel:** The catalog agent reads test files while the map agent reads source files. Neither depends on the other's output. You merge their results in Step 3.

### Step 3: Prioritize

Merge the interaction map with the coverage catalog. For each uncovered module, assign a risk level:

| Priority | Criteria | Examples |
|----------|----------|----------|
| **Critical** | User-facing + complex logic + zero tests | Quiz generation, SRS algorithms, payment logic |
| **High** | User-facing + new code, or revenue-critical | Free tier limits, streaming, rate limiting |
| **Medium** | Stateful code with business logic | Store actions, data migrations, background tasks |
| **Lower** | Utility functions, thin wrappers | Date formatting, analytics, platform API wrappers |

**Present the priority table to the user** before proceeding. This is a checkpoint — the user may want to adjust priorities or skip certain categories.

For a full-app audit, target **30-50 tests** in the first round, **30-50 more** in a second round.

### Step 4: Write Tests (In Rounds)

Tests should be written in **rounds**, not all at once. Each round targets a different complexity tier.

#### Round 1: Pure Functions and Utilities

Target exported functions with no side effects — the easiest to test and highest confidence per test.

- Pure logic (validation, calculation, formatting)
- Utility modules (date utils, parsers, converters)
- Constants validation
- Algorithm correctness (SRS intervals, scoring, ranking)

#### Round 2: Stateful Code with Mocking

Target modules that need database, API, or store mocking.

- Database layers (verify correct SQL/queries via mocked DB)
- Store actions (Zustand/Redux — test state transitions)
- API clients (mock fetch/XHR, test parsing and error handling)
- Background tasks (mock dependencies, test orchestration)

#### Dispatching Test Writers

**Dispatch 3-5 parallel agents**, each responsible for a batch of related test files. Group by dependency similarity so each agent can share mock setup.

**Agent prompt template:**

```
Write tests for these modules. Work from: [directory]

## Test Infrastructure
- Framework: [jest/vitest/pytest/etc]
- Test location: [__tests__/lib/, __tests__/stores/, etc.]
- Global setup: [e.g., (global as any).__DEV__ = true]
- Mock patterns: [describe existing patterns from Step 0]
- Manual mocks: [list __mocks__/ contents]

## Modules to Test

[For each module, include:]
- File path and exported functions
- Which functions are pure vs need mocking
- Specific test cases to write
- Mock setup needed

## Rules
- Follow existing test file naming and structure conventions
- Mock external dependencies — don't hit real services
- Test behavior, not implementation
- One test per distinct behavior or edge case
- Run tests after writing: [test command]
```

**After each round:**
1. Run the full test suite to verify no regressions
2. Fix any failures before starting the next round
3. Count: new tests added, modules now covered

### Step 5: Report

Summarize findings after all rounds complete:

```
## Shakedown Report

**Scope:** [what was audited]
**Interactions mapped:** [count]

### Coverage

| Metric | Before | After |
|--------|--------|-------|
| Test files | X | Y |
| Test cases | X | Y |
| Modules with tests | X/Z | Y/Z |
| Zero-test critical modules | X | Y |

**Tests added:** [count] across [N] new test files
**Bugs found:** [count — list if any]

### New Test Files
[Table: file | test count | what it covers]

### Remaining Gaps
[List UNCOVERED items that couldn't be tested and why]
- Components (N) — require component testing library setup
- Edge functions (N) — require deployment/integration testing
- E2E flows — not in scope for unit testing

### Recommendations
[Structural issues that limit testability, suggested next steps]
```

## Parallel Agent Strategy

Shakedown makes heavy use of parallel agents. Here's the dispatch pattern:

```
Step 2:  [Map Agent] ──────────────┐
         [Catalog Agent] ──────────┤ (parallel)
                                   ▼
Step 3:  [You] merge + prioritize
                                   │
Step 4:  [Test Agent 1: pure fns] ─┐
         [Test Agent 2: DB layer] ─┤
         [Test Agent 3: stores]  ──┤ (parallel, per round)
         [Test Agent 4: utils]   ──┘
                                   │
         Run tests, fix failures   │
                                   │
         [Test Agent 5: round 2] ──┐
         [Test Agent 6: round 2] ──┤ (parallel)
         [Test Agent 7: round 2] ──┘
                                   │
Step 5:  [You] report
```

Each agent gets a **complete, self-contained brief** — source file contents, mock setup, test cases to write, and project conventions. Never assume an agent has context from a previous step.

## Token Efficiency

- **Scope first.** A full-app audit of a large codebase can blow through tokens. For feature branches, scope to changed files and their callers.
- **One-line format.** The interaction map is the token bottleneck. Keep it structured and terse.
- **Parallel Steps 2+3.** The map and catalog agents don't depend on each other — run them simultaneously.
- **Read source files yourself, brief agents precisely.** Read the high-priority source files between Steps 3 and 4. Pass specific function signatures and mock setup to test agents rather than making them re-explore.
- **Batch by dependency.** Group test files by shared mock setup (e.g., all DB-dependent tests in one agent, all store tests in another).
- **Skip known failures.** If the project has pre-existing test failures, tell agents to ignore them.

## Customization

The skill works for any language or framework. Adapt the agent prompts:

| Project Type | Entry Points to Check | State Management |
|---|---|---|
| Next.js / Express | `src/app/api/**/route.ts`, page components, middleware | React context, Zustand, Redux |
| React Native / Expo | screens, stores, background tasks, push notifications | Zustand, MobX |
| Django / Flask | `urls.py` / route decorators, views, management commands, Celery tasks | Django ORM |
| Rails | `routes.rb`, controllers, jobs, mailers | ActiveRecord |
| CLI tool | Command handlers, subcommands, config parsing | — |
| Library | Public API surface, edge cases per function | — |

## What This Skill Is NOT

- **Not a replacement for integration/E2E tests.** This finds gaps in unit and contract test coverage. It doesn't spin up browsers or real databases.
- **Not a security audit.** It maps interactions but doesn't probe for vulnerabilities.
- **Not exhaustive by definition.** The map is as good as the agent's exploration. Complex apps may need multiple passes or manual additions to the map.
- **Not a one-shot process.** Expect 2-3 rounds for a full-app audit. The first round covers the easy wins; subsequent rounds tackle stateful code that needs heavier mocking.
