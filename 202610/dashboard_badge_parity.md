---
tier: tale
title: Align dashboard badges with their task sections
goal: Make PENDING, NEXT, and READY dashboard counts match their Tasks sections while
  preserving whole-lane budgets and freshness semantics.
size: medium
proposed_by: bbugyi200.apollo.40
status: done
---

# Make dashboard badge counts agree with their task sections

## Outcome and scope

The number displayed by each dashboard lane badge must equal the number of tasks in its
corresponding Tasks section after the caches settle. Fix PENDING and the same defect in
NEXT, close a confirmed READY filter-parity gap, and add regression coverage that
compares badges with the actual dashboard query contract. Preserve sticky task lanes,
ledger-derived TODAY, freshness-gated READY, and the existing whole-lane budgets used
outside the dashboard.

This is a medium tale: one coding agent can implement and validate the bounded change
across bob-plugins, the Bob vault, and bob-cli documentation. An epic and parallel
phases would add coordination without independent implementation work.

## Investigation and evidence

No implementation, vault, or configuration files were changed while planning. The
following was checked on 2026-10-01, including the running Mac Obsidian vault at
approximately 15:30–15:35 America/New_York:

1. The vault checkout at `e572d2d057971cead18d698cd9f6ee5f15280ed4` matched the
   successful local `bob vault-sync status --json` revision. Its `dash.md` has
   `TQ_extra_instructions` for template exclusion, dependency blocking, dashboard
   self-exclusion, scheduling, exact `#hide` exclusion, grouping, and sorting. The Tasks
   global query additionally excludes `_conflicts`.
2. The dashboard PENDING badge prefers `api.pendingBudget()`. In bob-plugins,
   `plugins/bob-ledger-tools/main.js`, `laneBudgetFromTasks()` explicitly counts the
   whole lane including TODAY. The PENDING section explicitly excludes
   `api.isToday(task)`. The NEXT badge/section have the same scope mismatch.
3. Live Obsidian, with Tasks 8.4.0 and bob-ledger-tools 1.13.2, returned PENDING **50**
   from `pendingBudget()` and **49** from the actual Tasks query engine with `dash.md`
   query-file defaults. The extra task was `sase.md#memory-file-versions`, linked under
   the open MEMORY entry in today's ledger. This is a confirmed current root cause, not
   an indexing guess.
4. READY has a different confirmed code defect: `readyTaskVisible()` excludes only the
   exact case-sensitive tag `#hide`, and tests deliberately preserve eligibility for
   `#hide/x` and `#Hide`. The READY Tasks block also says `tag does not include #hide`,
   whose matching is case-insensitive substring matching. A read-only synthetic example
   with `#hide/example` produces badge count **1** and section count **0**. Path
   comparisons also need an explicit parity audit: Tasks built-in path/folder filters
   are case-insensitive, while some READY helper checks are not.
5. That READY defect does **not** explain an observed current live discrepancy: there
   were no eligible hide-subtag/case variants in the sampled data. The running READY
   badge, `readyBudget()`, and actual Tasks READY query all returned **210**, with the
   shared model reporting lane total 211 = 1 NEW + 0 ROTTEN + 210 READY. Do not claim
   the user's earlier READY mismatch was reproduced or that its historical cause is
   proven. It may have been transient or changed before inspection; that remains
   unconfirmed.
6. Actual Tasks results were obtained without changing a note or task: create a detached
   element, call the running Tasks plugin's
   `queryRenderer.addQueryRenderChild(source, element, context)` with
   `context.sourcePath = "dash.md"` and an `addChild` callback that captures the child,
   then call `child.queryResultsRenderer.query.applyQueryToTasks(plugin.getTasks())`.
   Read the result's task groups/counts and always `child.unload()` in `finally`. This
   uses Tasks' real parser, global query, and query-file defaults. It is a diagnostic
   technique for the installed version, not a new production API. The ordinary CLI eval
   context could not `require("obsidian")`; do not depend on that approach for this
   diagnostic.
7. The existing Today, plan-budget, READY-badge, and freshness Node suites all passed:
   **112 tests, 0 failures**. They establish a clean baseline but do not establish
   dashboard-query parity. Some READY fixtures currently assert the mismatching old hide
   semantics and must be corrected.

## Governing contracts and repositories

Before implementation, read the applicable instructions and use `sase memory read` for
`obsidian.md`, `decisions:task-lanes-are-sticky`,
`decisions:today-is-read-from-the-ledger`, `decisions:ready-is-freshness-gated`, and
`glossary:freshness` with a specific reason. Consult `tailnet.md` through the same
command if using live Mac runtime diagnostics. No durable memory change is part of this
plan.

Use `/sase_repo` to open `bob-plugins` and `gh:bobs-org/bob`; use only the returned
checkout paths for repository reads/edits and inspect their AGENTS.md and status. The
vault is not registered under the bare name `bob` in this project's repo inventory, so
that spelling failed during investigation. Treat `obsidian.md`'s current git-sync
instructions as authoritative over the vault AGENTS.md's stale statement that Obsidian
Sync is active. Keep all user/sync changes intact.

Relevant implementation locations:

- bob-plugins: `plugins/bob-ledger-tools/main.js`, its manifest, README, and
  `scripts/test-ledger-tools-{today,plan-budget,ready-badge,freshness}.cjs`.
- Bob vault: `dash.md`; inspect `rotten.md` only as needed for shared freshness
  visibility, and `.obsidian/plugins/obsidian-tasks-plugin/data.json` for the effective
  settings. Do not modify the third-party Tasks plugin.
- bob-cli: `docs/plan.md` and, only where the visible-pool clarification needs it,
  `docs/freshness.md`. No Rust behavior or new CLI option is required.

## Implementation

### 1. Pin the failure and the intended scopes

Recheck the live/versioned baseline because task counts are mutable. Capture task
identities as `(vault path, zero-based line number)` with block IDs as additional
context; compare sets as well as counts. Separate differences due to TODAY, visibility,
freshness, and a rendered value that is behind its model. Headless
`bob query --tasks-note dash.md` has no live ledger-tools API and thus uses the
documented ungated/no-TODAY fallback: it is useful for parsing and base filters, but is
not the live READY/TODAY oracle.

Create focused failing regressions for the confirmed PENDING/TODAY discrepancy and READY
hide variants before changing their implementations. Cover NEXT's equivalent discrepancy
in the same tests. If READY differs live again, capture the model count, actual Tasks
results, rendered badge value, cache readiness, plugin versions, and differing
identities before reloading anything. Only add another behavioral repair when that
evidence or a regression identifies it.

### 2. Separate dashboard section counts from whole-lane budgets

Keep no-argument `pendingBudget()` and `nextBudget()` and their whole-lane cap semantics
unchanged for `bob-plan`, navigation notices, native CLI parity, and other callers. Do
not globally subtract TODAY from `laneBudgetFromTasks()`.

Add an additive, guarded dashboard-specific API, such as
`dashboardLaneBudget("pending" | "next")`, backed by a shared pure membership helper.
Return the section count separately from the whole-lane count and cap. Use
unavailable/null counts when Tasks or the required current-day Today cache is not ready;
an unavailable count must not become a successful zero.

The section model excludes TODAY and dash.md itself, dependency-blocked tasks, future
schedules, templates, conflict paths, and hidden tasks. Align hidden-tag and path
matching with the effective built-in Tasks filters. Use IN_PROGRESS for PENDING and
symbol `*` for NEXT, and make the corresponding dashboard query selectors explicit. The
standard configured statuses are unchanged.

On the dashboard, render the section count as the primary number, e.g. **PENDING 49**,
rather than presenting the whole-lane 50 as the section count. Preserve the existing
whole-lane cap warning and explain it in the tooltip and accessible label: **49 in this
section; whole lane 50/10; 1 in TODAY**. The cap fraction belongs in that explanation so
it cannot be mistaken for a different section count. Apply the same presentation to
NEXT. READY retains `n/cap`, because its cap already applies to the gated dashboard
backlog.

Wire dashboard lane widgets into lifecycle-owned updates for Tasks-cache, TODAY-link,
day-rollover, and cap changes; do not rely exclusively on Dataview noticing a file
change. Reuse the existing refresh/debounce machinery and component disposal pattern.
Avoid per-row disk reads and accumulating timers or event listeners when the dashboard
rerenders.

### 3. Make READY and section visibility agree

Factor the dashboard visibility checks so READY and the new section counts use one
tested base predicate, with status/TODAY/freshness applied at their proper layers. Bring
READY's hide and path semantics into agreement with its actual query and the documented
freshness visible pool. Replace tests that deliberately expected READY to include
`#hide/x` or `#Hide`.

Keep the existing snapshot-backed bucket evaluator: READY excludes `new` and `rotten`,
while recurring and canonical-daily-note tasks remain eligible when otherwise visible.
Do not replace this with `state === "fresh"`, invent a second freshness evaluator,
change task statuses/stamps, or broaden review scope merely to force counts to match.
Preserve the existing per-row identity and duplicate-block-ID handling introduced in
ledger-tools 1.13.2.

Update `dash.md`'s normal lane-badge path and query filters together. Preserve section
order and mutual exclusion. Its supported older/unloaded-plugin fallback must apply the
same base visibility and available TODAY predicates as its queries, retaining the
documented ungated READY fallback when freshness is absent. Distinguish missing
capability from a present API returning an unavailable result. Keep the shared READY
renderer and daily READY parity.

Do not rewrite the dashboard as a custom monolithic renderer or change the Tasks
plugin/global status configuration. Keep BLOCKED and unrelated features out of this
repair.

### 4. Test parity and transitions

Add a focused dashboard parity harness/fixture that exercises the dashboard query
definitions and badge models, including query-file defaults and global conflict
exclusion. It must compare identities/counts at both boundaries, not simply call the
same helper twice. Keep personal vault contents out of committed test fixtures. Use live
Tasks 8.4.0 as an integration check where available; if any headless harness is used,
explicitly provide the plugin API context or test its documented fallback separately.

Required cases:

- One ordinary PENDING task and one PENDING task linked TODAY: whole lane 2,
  section/badge 1; analogous NEXT coverage. Unlinking changes section membership without
  changing sticky status or the whole-lane count.
- READY fresh, NEW, resurfaced, rotten, recurring, daily-note-exempt, and TODAY-linked
  cases; review partitions and daily/dashboard READY still agree.
- Exact `#hide`, `#hide/x`, `#Hide`, template/conflict path case variants, dashboard
  self-exclusion, future/today/absent schedules, and dependencies on tasks outside the
  visible subset.
- Duplicate block IDs, tasks without block IDs, and query objects with a different
  object identity; retain the existing no-borrowed-classification regressions.
- No Tasks plugin, non-Warm cache, initial/midnight Today readiness, old plugin API, and
  errors: no false zero or partially gated count.
- Task edits, freshness stamps, TODAY-only link/unlink, frontmatter refresh interval
  changes, midnight, and repeated rerenders: after the existing debounce/tick settles,
  rendered values equal their models and query results; component cleanup leaves no
  accumulated widgets/listeners.
- PENDING/NEXT tooltip and accessible label distinguish section count from lane
  pressure, and cap warnings still use the full lane. READY's cap and unavailable
  behavior remain unchanged.

Run the four focused Node suites first, then `npm test` and `npm run validate` in the
opened bob-plugins checkout. Add the parity suite to the package test command if it is a
new file. Run additional bob-cli checks only if executable code there changes;
documentation-only edits do not justify an unrelated Rust test run. Correct any failure
caused by this work before deployment.

### 5. Document, deploy, and verify the visible result

Update `docs/plan.md` to remove the false implication that a whole-lane badge and a
TODAY-excluding section always agree. Document the dashboard primary section number,
whole-lane tooltip/cap warning, unchanged non-dashboard lane budgets, exact visibility
rules, and fallback/unavailable behavior. Update the plugin README/API description and
bump its manifest version following local conventions. Do not modify accepted decision
records as part of this fix.

Deploy from the opened source-of-truth plugin checkout with
`bob plugins sync --repo <opened-plugin-checkout> --no-pull --plugin bob-ledger-tools`
as required by that repository's instructions. Use the established SASE and vault
git-sync workflow to deliver the approved dash.md change to the live vault, preserving
unrelated work. Never patch deployed plugin main.js by hand or re-enable Obsidian Sync.
Follow the normal SASE finalizer obligations for every modified repository; do not make
an ad hoc manual git commit.

Verify the active runtime has the new plugin version and intended dashboard query
revision before claiming the visible fix. Compare the actual Tasks PENDING/NEXT/READY
results with rendered badge values on the same settled snapshot, including a dashboard
reopen and an available representative normal edit/link transition without modifying
real tasks solely for a test. Use synthetic fixtures for transitions that cannot be
observed safely. If the live runtime cannot be reached, report the missing live check
explicitly rather than calling headless fallback output proof of UI parity.

## Acceptance criteria

- On the captured pre-fix example, the dashboard displays PENDING 49 while clearly
  reporting whole-lane pressure 50/10; the PENDING section contains 49. Equivalent NEXT
  membership agrees, and all non-dashboard whole-lane budgets retain their previous
  semantics.
- READY's badge and query agree for all regression fixtures, including hide variants,
  and for the settled live snapshot. The report accurately states that READY was already
  210/210 during planning; it does not attribute the user's historical READY discrepancy
  to an unobserved cause.
- Caches becoming available or changing do not leave stale dashboard counts; missing
  data remains distinguishable from zero. READY still includes freshness-exempt tasks
  and excludes NEW/ROTTEN/TODAY.
- Focused and full plugin tests plus manifest validation pass. Changes are deployed
  through supported mechanisms, with live verification completed or its precise
  limitation recorded, and only task-scoped changes are submitted.
