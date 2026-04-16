---
name: shakedown
description: Map every user interaction in an app, then either (a) audit test coverage and write missing tests (code mode), or (b) tour the running app in a browser and report UX bugs, friction, and dead features (ui mode). Use after a feature, before a redesign, or to get a fresh look at what the app actually does.
---

# Shakedown

A shakedown is a thorough test of a new ship before it sets sail. This skill does that in two modes:

- **`code`** — audit test coverage, identify gaps, and write missing tests.
- **`ui`** — tour the running app in a browser, interact with every control, and report bugs, friction, dead features, and what's worth keeping.

Both modes share the same philosophy: enumerate the surface, figure out what's missing, produce a structured report. One targets code coverage; the other targets user experience.

## Picking a Mode

**Default to `code`.** Switch to `ui` when:

- The user says "UI shakedown", "UX shakedown", "shakedown the UI", or similar.
- The user asks you to "look at the app", "see what it looks like", or "audit the experience".
- The user wants to know what's confusing or unused from a user's perspective — not what's untested.

When the request is ambiguous and both modes could apply, ask which one.

## When to Use

### `code` mode

- After completing a feature branch (before merge/PR)
- When joining a project and want to understand coverage
- After a large refactor to verify nothing broke
- When the user asks to "test everything" or "find test gaps"

### `ui` mode

- Before a redesign or planning sprint — know what to keep and what to cut
- As a pre-flight pass before releasing a new feature
- When you suspect part of the app is dead or broken but haven't looked
- After a dependency or infra change that might have broken pages silently

---

# Mode: `code`

Map the full interaction surface of an application, identify what's tested and what isn't, then write targeted tests to close the gaps.

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

## Parallel Agent Strategy (code mode)

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

## Customization (code mode)

The skill works for any language or framework. Adapt the agent prompts:

| Project Type | Entry Points to Check | State Management |
|---|---|---|
| Next.js / Express | `src/app/api/**/route.ts`, page components, middleware | React context, Zustand, Redux |
| React Native / Expo | screens, stores, background tasks, push notifications | Zustand, MobX |
| Django / Flask | `urls.py` / route decorators, views, management commands, Celery tasks | Django ORM |
| Rails | `routes.rb`, controllers, jobs, mailers | ActiveRecord |
| CLI tool | Command handlers, subcommands, config parsing | — |
| Library | Public API surface, edge cases per function | — |

---

# Mode: `ui`

Tour the running application in a browser, interact with every surface, and produce a prioritized report of what's broken, confusing, unused, or worth keeping.

**The output is a report, not fixes.** Treat the audit as honest observation. The natural next step is a brainstorm about what to act on — typically leading to a spec and implementation plan. Don't try to fix things inline; that muddies the findings.

## Requirements

- A **running dev server**, or the ability to start one via `npm run dev` / `yarn dev` / `pnpm dev` / etc.
- **Playwright MCP** available — the `mcp__*__browser_navigate`, `_snapshot`, `_click`, `_type`, `_take_screenshot`, `_console_messages` tools. If unavailable, tell the user and stop.
- A clean-ish commit state. The tour creates screenshots; decide upfront whether they go in the repo or stay local.

## Process

```
0. Baseline     → Find or start the dev server; verify Playwright MCP; create screenshot dir
1. Scope        → Full-app tour, scoped to specific routes, or regression check on recent changes
2. Inventory    → Read the router to enumerate routes + their interactive surfaces
3. Tour         → Navigate each surface, interact with every control, capture screenshots + a11y snapshots, watch console/network for errors
4. Evaluate     → Per surface: bugs, friction, dead features, what's worth keeping
5. Prioritize   → Tier A (real bugs) / B (confusing or wasteful) / C (polish)
6. Report       → Markdown with embedded screenshots and concrete recommendations
```

### Step 0: Baseline

- Probe for a running dev server on common ports (5173 for Vite, 3000 for Next/CRA, 4173 for Vite preview, 8080 for webpack dev server). Example: `lsof -i :5173 -i :3000 -i :4173 -i :8080`.
- If none is running, look in `package.json` or the project's equivalent for a `dev` script and start it in the background. Wait until it responds before tours begin.
- Confirm Playwright MCP browser tools are available.
- Create a screenshot directory — `.ux-review/` is a reasonable default. Add it to `.gitignore` if the user doesn't want the artifacts committed.
- Resize the browser to a sensible desktop viewport (1440×900 is a good default; adjust if the app is clearly designed for a different size).

**Output:** a short infra summary — URL, framework, routing library, viewport.

### Step 1: Scope

- **Full-app tour** (default): every route in the router.
- **Scoped tour**: just the routes or flows the user called out.
- **Regression check**: routes touched by `git diff --name-only main...HEAD` on frontend files.

Output a short scope statement.

### Step 2: Inventory

Read the router file (typically `router.tsx`, `routes.tsx`, `app/routes.ts`, or equivalent) and any sidebar/nav component. For each route, note:

- Path
- Page component
- Global filters that affect it (owner dropdowns, date pickers)
- Whether it's in the sidebar (user-visible) or hidden (legacy / internal)

If the sidebar advertises a route that no longer exists — or a route exists that isn't in the sidebar — flag it as a finding immediately.

### Step 3: Tour

For each surface in the inventory:

1. `browser_navigate` to the route.
2. `browser_wait_for` 1–3 seconds of settling time.
3. `browser_take_screenshot` (full-page) to `.ux-review/NN-route.png`.
4. `browser_snapshot` for the a11y tree — find interactive controls (buttons, links, inputs, dropdowns, tabs).
5. For each interactive control:
   - Interact (`_click`, `_type`, `_select`).
   - Capture the follow-up screenshot.
   - Note what happened (navigation, modal, dropdown options, data change).
6. `browser_console_messages` at the end of the tour — capture errors.
7. Scan network requests (`browser_network_requests` if available, or check `performance.getEntriesByType('resource')` via `browser_evaluate`) for 4xx/5xx, unusually slow requests, oversized payloads.

### Common patterns to catch

- **Long loading spinners without skeletons** — often means oversized unvirtualized renders. Confirm with a `document.querySelectorAll('tbody tr').length` check.
- **Tables where whole columns are `—`** — dead data pipeline or broken endpoint.
- **Links that route to pages with lost query params / state** — click through and see if the target uses the passed data.
- **Section headers that render with no content** under filtered states — confirm by toggling filters.
- **Console errors**, especially 500s or 404s on endpoints the page relies on.
- **Clipped content** that overflows the viewport — check by measuring `scrollWidth` vs `clientWidth`.
- **Inner-container scroll** that traps content inside an overflow div instead of using page scroll.
- **Slow responses with no progress signal** — chat that streams but gives no "thinking" indicator.

### Step 4: Evaluate

For each surface, produce notes in four buckets:

- **Bugs** — broken behavior: errors, dead links, wrong data, 5xx, click-through that loses state.
- **Friction** — confusing or wasteful: redundant columns, long loads without progress, silent resets, tiny click targets, empty section headers.
- **Dead features** — sections that clearly add no user value (all-dashes tables, unused routes, duplicate interactions).
- **Worth keeping** — patterns that work well and should be preserved.

Trust the user's hypotheses. If they said "I think X is unused," check X specifically and confirm or refute with evidence.

### Step 5: Prioritize

| Tier | Criteria | Examples |
|------|----------|----------|
| **A** | Real bugs — would ship broken today | 500 on a visible endpoint, link that drops its param, broken filter state |
| **B** | Confusing or wasteful | Redundant columns, 25-second load with no skeleton, empty section headers on filtered data |
| **C** | Minor polish | Missing favicon, dropdown overlaps content briefly, date off-by-one in an edge case |

### Step 6: Report

Write the report as a markdown file (default: `.ux-review/UX-REVIEW.md`). Structure:

1. **Bottom line** — 2–3 sentence verdict. What's the real product? What's dead weight?
2. **What's good (keep, polish)** — surface by surface, specifics + screenshot references.
3. **What's broken or confusing** — tiered A/B/C list, each finding paired with a screenshot file name.
4. **Recommendation on what to cut** — if applicable, name the specific routes/pages to remove, with reasoning.
5. **Possible directions** — 2–4 sketches of what could replace cut functionality, explicitly framed as options.
6. **Suggested next steps** — short prioritized list. For anything non-trivial, point to a brainstorm / spec session.

Screenshots live alongside the report; reference them with relative paths.

## Parallel Tours (ui mode)

For large apps, split the tour across agents. Each route is independent of the others, so a tour agent can own a cluster:

```
[Tour Agent 1: /, /dashboard] ────┐
[Tour Agent 2: /accounts, /pipeline] ┤ (parallel)
[Tour Agent 3: /scan] ────────────┘   (owns deep flows with long interactions)
                                   │
                                   ▼
[You] merge notes → report
```

Each agent gets a **complete, self-contained brief**:
- Dev server URL
- Routes it owns
- Output directory for screenshots
- Any user hypotheses to specifically check ("confirm or refute that /accounts is unused")

Agents return a slice of findings; the controller merges them and writes the unified report.

## What UI Shakedown Is NOT

- **Not an auto-fix.** The skill produces a report. Acting on findings is a separate step — typically a brainstorm → spec → plan, not inline editing during the tour.
- **Not a replacement for automated E2E tests.** It finds what's broken right now; it doesn't build reproducible test harnesses. For that, `code` mode is the right tool.
- **Not a security audit.** It maps interactions but doesn't probe for vulnerabilities.
- **Not a pixel-perfect design review.** It flags functional UX friction. Pure aesthetics are out of scope unless they affect usability.

---

# Token Efficiency (both modes)

- **Scope first.** A full-app shakedown is expensive. For feature branches, scope to changed files.
- **Parallel Steps 2+3 in code mode, parallel tours in ui mode.** These are the biggest wall-clock wins.
- **One-line interaction format (code mode).** Keep the map under 3K tokens.
- **Read source files yourself and brief agents precisely (code mode).** Pass function signatures and mock setup rather than raw files.
- **Capture screenshots to disk, not into the transcript (ui mode).** Reference them by filename in the report; don't re-inline them into the chat context.
- **Skip known failures / known dead features.** Tell agents up front so they don't re-flag them.

# What Shakedown Is NOT (either mode)

- **Not a one-shot process.** Expect 2–3 rounds for a `code` shakedown, and for `ui` shakedown expect a follow-up brainstorm → spec cycle after the report.
- **Not exhaustive by definition.** The map (code) or tour (ui) is as good as the agent's exploration. Complex apps may need multiple passes or manual additions.
