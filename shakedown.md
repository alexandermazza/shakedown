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
1. Scope       → What changed? What's the blast radius?
2. Map         → Enumerate every user interaction path
3. Catalog     → Check existing test coverage against the map
4. Prioritize  → Rank uncovered paths by risk
5. Test        → Write tests for the highest-risk gaps
6. Report      → Summary of coverage, gaps, and findings
```

### Step 1: Scope

Before mapping the whole app, narrow the focus. Ask yourself:

- **Is this a full-app audit or scoped to recent changes?**
  - If a feature branch: `git diff main...HEAD --name-only` to find changed files
  - If full audit: skip this step, map everything
- **What are the entry points?** (API routes, pages, background workers, webhooks)

Output a short scope statement: "Auditing [X files / Y features / the full app], focusing on [area]."

### Step 2: Map Interactions

Dispatch an **Explore agent** to enumerate every user-facing interaction. The agent should return a structured list, not prose.

**Agent prompt template:**

```
Map every user interaction in this application. Work from: [directory]

Focus areas: [list entry points — API routes, pages, workers, webhooks]

For each interaction, return ONE LINE in this format:
[CATEGORY] [ACTION] → [ENDPOINT/FUNCTION] | Edge: [edge cases]

Examples:
[ONBOARDING] Click "Connect Slack" → GET /api/connections/slack | Edge: OAuth failure, missing state
[SLACK] Complete task → POST /api/slack/interactions (complete_task_{id}) | Edge: already resolved, unauthorized
[WORKER] Process reminders → processReminders() | Edge: missing Slack connection, stale remindAt

Group by category. Be exhaustive — check every route file, every button handler, every background job.
Do NOT write paragraphs. One line per interaction. Keep it tight.
```

**Why one-line format:** The full interaction map from Step 2 is passed to Step 3's agent. Verbose descriptions waste tokens. One line per interaction keeps the map under 3K tokens for most apps.

### Step 3: Catalog Existing Coverage

Dispatch an **Explore agent** to cross-reference the interaction map against existing tests.

**Agent prompt template:**

```
Here is every user interaction in this app:

[paste interaction map from Step 2]

Check existing test files at [test directories] and classify each interaction:

COVERED   — has a direct test
PARTIAL   — tested indirectly (e.g., helper is tested but not the route)
UNCOVERED — no test coverage

Return the same one-line format with a coverage tag prepended:
[COVERED] [SLACK] Complete task → POST /api/slack/interactions
[UNCOVERED] [WORKER] Process reminders → processReminders()

Also note: what test framework is used, what mocking patterns exist, and any
infrastructure limitations (e.g., "route handlers use module-level DB imports,
not injectable — unit tests need to mock at the module level").
```

### Step 4: Prioritize

From the catalog, extract all UNCOVERED and PARTIAL items. Rank by:

1. **User-facing + new code** — highest risk (recently written, untested)
2. **User-facing + existing code** — medium risk
3. **Background/worker + new code** — medium risk
4. **Background/worker + existing code** — lower risk
5. **Edge cases on covered paths** — lowest (but still worth testing)

Pick the top items based on available budget. For a typical feature branch, 20-40 new tests is a good target.

### Step 5: Write Tests

Dispatch a **general-purpose agent** to write the tests. Give it:
- The prioritized gap list
- The test framework and patterns (from Step 3)
- Specific instructions to follow existing conventions

**Agent prompt template:**

```
Write tests for these uncovered interaction paths. Work from: [directory]

## Gaps to Cover (priority order)

[paste prioritized list]

## Test Conventions

- Framework: [vitest/jest/pytest/etc]
- Existing test patterns: [describe key patterns from Step 3]
- Mock patterns: [how DB, external APIs are mocked]
- Test locations: [where test files live]

## Rules

- Follow existing test file naming and structure conventions
- Mock external dependencies (DB, APIs, auth) — don't hit real services
- Test the behavior, not the implementation
- One test per distinct behavior or edge case
- Run tests after writing to verify they pass: [test command]
- Commit passing tests

If a path is not unit-testable (e.g., requires integration test infra that
doesn't exist), write a contract test that verifies the inputs/outputs of
the underlying functions instead. Note what would need integration tests.
```

### Step 6: Report

Summarize findings to the user:

```
## Shakedown Report

**Scope:** [what was audited]
**Interactions mapped:** [count]
**Coverage before:** [X covered / Y total]
**Coverage after:** [X covered / Y total]
**Tests added:** [count]
**Bugs found:** [count — list if any]

### Remaining Gaps
[List any UNCOVERED items that couldn't be tested and why]

### Recommendations
[Any structural issues that limit testability, suggested fixes]
```

## Token Efficiency Tips

- **Scope first.** A full-app audit of a large codebase can blow through tokens. For feature branches, scope to changed files and their callers.
- **One-line format.** The interaction map is the token bottleneck — it gets passed between agents. Keep it structured and terse.
- **Don't re-read in Step 5.** The test-writing agent should get the gap list and test conventions upfront. It reads source files as needed, but doesn't re-explore the full codebase.
- **Batch by file.** Group related gaps so the agent can write multiple tests per file in one pass.
- **Skip known failures.** If the project has pre-existing test failures, tell agents to ignore them so they don't waste tokens investigating.

## Customization

The skill works for any language or framework. Adapt the agent prompts:

| Project Type | Entry Points to Check |
|---|---|
| Next.js / Express | `src/app/api/**/route.ts`, page components, middleware |
| Django / Flask | `urls.py` / route decorators, views, management commands, celery tasks |
| Rails | `routes.rb`, controllers, jobs, mailers |
| CLI tool | Command handlers, subcommands, config parsing |
| Library | Public API surface, edge cases per function |

## What This Skill Is NOT

- **Not a replacement for integration/E2E tests.** This finds gaps in unit and contract test coverage. It doesn't spin up browsers or real databases.
- **Not a security audit.** It maps interactions but doesn't probe for vulnerabilities.
- **Not exhaustive by definition.** The map is as good as the agent's exploration. Complex apps may need multiple passes or manual additions to the map.
