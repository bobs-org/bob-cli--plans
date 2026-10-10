---
tier: tale
title: Align daily NEXT and PENDING badges with the dashboard
goal:
  Daily NEXT and PENDING badges share the dashboard's live section counts, warning
  colors, and unavailable states while preserving clearly labeled whole-lane warnings.
size: medium
proposed_by: bbugyi200.apollo.68.f0
create_time: 2026-10-10 09:41:32
status: wip
---

# Align daily NEXT and PENDING badges with the dashboard

## Outcome and scope

Every daily note's `bob-plan` NEXT and PENDING badges should show the same current
section counts and warning states as `dash.md`. Both surfaces exclude TODAY from these
two numerators, show the configured cap, and turn red only for a strict section excess.
Exactly at the cap stays normal-colored. Whole-lane pressure remains explicitly labeled
in the badge details and the daily warning text.

This is one medium tale: a bounded change to bob-ledger-tools' daily presentation,
shared rendering integration, refresh coverage, and documentation in bob-cli and
bob-plugins. One coding agent can implement and verify it together. No reviewer choice
is needed: Bryan has requested that the daily badges follow the already-approved
dashboard policy. There are no new commands, options, settings, task states, or memory
changes.

## Findings and evidence

- The supplied screenshot, `~/tmp/screenshots/20261010_093245.png`, shows daily
  `PENDING 8/10` and red `NEXT 17/15`, plus
  `NEXT has 17/15 tasks; release some with Alt+N next_cap_exceeded` underneath.
- The earlier approved plan, `plan:202610/dashboard_badge_warnings.md`, selected
  `lane_warning_scope = section`. Its inspected snapshot had seven of seventeen Next
  tasks in TODAY, leaving dashboard `NEXT 10/15`, and four of eight Pending tasks in
  TODAY, leaving `PENDING 4/10`. These values explain the mismatch; they are fixtures,
  not permanent assertions about Bryan's changing vault.
- The opened bob-plugins checkout contains that implementation at `bec01f6`, plugin
  version 1.41.0. In `plugins/bob-ledger-tools/src/070-ready-and-review.js`,
  `dashboardLaneBudgetFromTasks()` computes the dashboard section with the existing
  visibility and status rules. `dashboardLaneBadgeModel()` shows `section/cap` and
  derives color from `section > cap`, with whole-lane and TODAY details.
- `180-plugin-plan-and-ready.js` provides `dashboardLaneBudget()` with Tasks-cache and
  current-day Today-cache guards, and `paintDashboardLaneElement()` provides the shared
  text, tooltip, accessible label, and link behavior. Its public renderer adds
  component-owned dashboard widget registration.
- The daily path is separate: `340-dashboard-views-and-plan-block.js` computes
  `planBlockModel().next/pending`, text, and cap lints through `laneBudgetFromTasks()`.
  `290-plugin-today-and-location.js::paintPlanBlock()` hand-builds NEXT/PENDING anchors
  from those whole-lane results with generic tooltips. The outer status label also
  repeats those numbers and can say “over plan” for a lane-only excess.
- The vault's `2026/20261010.md` and `_templates/daily.md` contain empty `bob-plan`
  blocks. No note or template migration is needed. `dash.md` already consumes the shared
  dashboard renderer and has the selected section-count fallback.
- Daily blocks already have view registration, async paint-generation guards, and a
  debounced rerender path shared with Tasks/Today changes. However,
  `rebuildTodayCache()` schedules rerenders only when keys change. A transition from
  uninitialized to ready with an empty key set needs attention once daily lanes adopt
  the guarded budget.
- The four focused baseline suites (dashboard parity, plan budget, READY badge, Today)
  passed during planning. The plan-budget test currently explicitly expects a linked
  Next task to remain in the daily numerator. This is a deliberate presentation-contract
  update, not a cap-configuration repair.

The sticky-lane, ledger-Today, and freshness-gated READY decisions were read through
SASE memory. They continue to govern task state and membership. This plan supersedes
only the earlier plan's exception that daily NEXT/PENDING **badges** retain whole-lane
presentation.

## Behavior contract

1. Daily NEXT/PENDING are live views of the same sections as the dashboard, including in
   an older daily note. Use the current local day and today's live ledger cache, not the
   date or links of the note containing the block. Preserve the existing containing-note
   behavior of the TODAY theme/link budget.
2. Reuse the exact dashboard membership rules, including PENDING's `IN_PROGRESS` status
   type, NEXT's `*` symbol, hidden/future/blocked/done/template/conflict exclusions, and
   exclusion of `dash.md`. Do not approximate this as whole-lane count minus every TODAY
   task: these populations and status predicates are not universally identical.
3. Display `NEXT section/cap` and `PENDING section/cap`, with red only when that
   displayed section exceeds the cap. Tooltips and accessible labels carry the same
   section, whole-lane, TODAY, and separately labeled excess details as dashboard
   badges.
4. Missing Tasks data, a non-Warm cache, an unready or wrong-day Today cache, or failed
   section evaluation yields a non-red `–`. A successfully evaluated empty section is
   `0/cap`. Never substitute a whole-lane number or a guessed zero for unavailable
   section data. Readiness recovery must update already-open daily blocks.
5. Keep whole-lane APIs and warnings intact: `nextBudget()`, `pendingBudget()`,
   `laneBudgetFromTasks()`, and the raw `dashboardLaneBudget().over` field retain their
   meanings, as do native CLI and navigation notices. The daily `next_cap_exceeded` and
   `pending_cap_exceeded` lints keep their codes, counts, strict thresholds, order, and
   release hint, but explicitly name the **whole lane, including TODAY**. For example:
   `NEXT whole lane has 17/15 tasks (including TODAY); release some with Alt+N`. Thus a
   normal `NEXT 10/15` can coexist with a clearly explained whole-lane warning. Keep the
   ledger budget's own status independent of lane pressure; an accessible aggregate
   summary must not call a lane-only excess “over plan.”
6. Preserve TODAY and READY behavior, badge order, daily visual density, dashboard
   destinations, and existing dashboard grouping/fallback behavior. Lane badge parity
   does not mark the GTD review complete or mutate tasks or ledger entries.

## Implementation

1. **Reopen and recheck the sources.** Use `sase repo open bob-plugins` with a
   task-specific reason, read its `AGENTS.md`, and check repository status. The source
   may have advanced since planning. Use the printed checkout for all plugin work. Open
   `gh:bobs-org/bob` through `sase repo open` only if additional vault inspection is
   needed; preserve unrelated changes. Implementation should require plugin changes and
   bob-cli docs, with no vault Markdown edits.

2. **Give the daily model shared section presentation.** In
   `340-dashboard-views-and-plan-block.js`, extend `planBlockModel()` to consume
   prepared per-lane dashboard budgets and derive its daily badge text/presentation
   through `dashboardLaneBadgeModel()`. In the runtime paint path, obtain these through
   the guarded `this.dashboardLaneBudget(lane, paintDay)` for both lanes. Sample current
   Tasks/caps/Today state for the successful paint after the asynchronous note read,
   retaining generation checks so stale reads cannot restore old badges.

   Keep whole-lane results distinct for the existing cap lints; do not repurpose their
   `count` or `over` fields as section values. Update `nextText`/`pendingText` and the
   outer status summary to describe the rendered section badges. If standalone pure
   model callers compute section budgets, delegate to `dashboardLaneBudgetFromTasks()`
   using explicit task/Today inputs. A supplied unavailable budget must stay
   unavailable; it must not trigger an ungated fallback. Update the daily lint wording
   as specified, and scope accessible warnings to ledger, section, or whole-lane
   pressure correctly.

3. **Reuse the lane painter and existing daily lifecycle.** Replace the hand-built
   lane-anchor loop in `290-plugin-today-and-location.js` with
   `paintDashboardLaneElement()` using the same prepared budgets that feed the model,
   the containing `sourcePath`, and the config-validity flag. This shares the dashboard
   painter without adding a second set of registered daily widgets. The daily block's
   existing rerender/unload lifecycle owns those anchors. Keep click, Ctrl/Cmd-click,
   Enter/Space, focusability, and hover links working through the shared painter.

   Reuse the existing debounced invalidation for Tasks changes, Today link changes,
   configured cap changes, and day rollover. Ensure the Today cache's transition from
   unready/wrong-day to ready schedules the existing daily/dashboard refresh even when
   its key set remains empty. Preserve the existing meaning of methods that report
   whether membership keys changed. Do not add per-badge timers. Repeated rendering and
   note unload must not accumulate widgets, listeners, or outstanding paints.

   The shared painter uses separate label/value spans. Extend the daily-only rules in
   `plugins/bob-ledger-tools/styles.css` to cover PENDING/NEXT alongside READY's
   single-tone text and word spacing. Keep dashboard styling and the shared overflow
   class unchanged.

4. **Cover the integration and update the contract.** Extend the existing dashboard
   parity, plan-budget, READY-badge daily-paint, and Today-cache tests as appropriate.
   Prefer actual daily rendering and dashboard rendering against the same fixture over
   comparing two direct calls to one helper. Preserve the independent dashboard-query
   membership oracle and whole-lane API assertions. Keep dates/configuration isolated so
   new tests do not depend on wall-clock date or Bryan's live settings.

   Update bob-cli `docs/dashboard.md` and `docs/plan.md` (lane discussion and output
   surface table), plus bob-plugins `README.md`, to remove the old daily-badge exception
   and document live section parity, old-note behavior, unavailable states, and the
   separately labeled whole-lane lints. Follow the repository's plugin version/README
   convention for the release. Edit source fragments, run `npm run build` to regenerate
   committed `main.js`, and keep hand-edited fragments at or below 1000 lines.

## Validation and acceptance

Use synthetic tasks and a controlled current-day cache. Verify visible text, warning
classes, title, each anchor's aria label, and the daily container's accessible summary:

| Fixture                                                                   | Expected daily and dashboard badge                                      |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| NEXT whole lane 17, TODAY 7, section 10, cap 15                           | `NEXT 10/15`, non-red; details and daily lint identify whole lane 17/15 |
| PENDING whole lane 8, TODAY 4, section 4, cap 10                          | `PENDING 4/10`, non-red                                                 |
| PENDING whole lane 11, TODAY 7, section 4, cap 10                         | `PENDING 4/10`, non-red; whole-lane warning remains explicit            |
| NEXT section 15, whole lane 16, cap 15                                    | `NEXT 15/15`, non-red                                                   |
| NEXT section 16, cap 15                                                   | `NEXT 16/15`, red                                                       |
| PENDING section 10 then 11, cap 10                                        | At cap non-red, above cap red                                           |
| Valid empty section; nondefault caps                                      | `0/cap`; all comparisons use effective configuration                    |
| Tasks unavailable/non-Warm, Today uninitialized/stale, evaluation failure | `–`, non-red; no fabricated section value                               |

- Test matching membership for both lanes with hidden, future-scheduled, blocked, done,
  template/conflict, `dash.md`, and custom `IN_PROGRESS` tasks. Keep the existing
  dashboard's independent filter oracle as the authority for these expectations.
- Exercise the real daily paint path and shared dashboard renderer with the screenshot
  fixtures. Confirm both counts and colors, not only the pure model. Verify the daily
  accessible summary contains the new counts; a whole-lane-only excess must not mark the
  badge red or describe the ledger budget as exceeded.
- While views remain open, link/unlink a task in today's ledger, change a task lane,
  cross a cap in both directions, and change a configured cap. Drive existing refresh
  entry points/events and await the repaint; verify the two surfaces converge and the
  warning class clears below/at cap. Use fixture edits, not mutations to real tasks.
- Cover unready → ready with an empty TODAY set, unavailable → available, date rollover,
  and an older daily note whose ledger differs from today's. Its NEXT/PENDING must
  follow today's dashboard while its own TODAY theme/link budget retains its contract.
- Verify click/keyboard destinations and modifiers, source-path handling, repeat
  render/unload cleanup, and stale async paint rejection. Check the daily span styling
  remains compact and readable. Existing TODAY/READY model and render tests must pass.
- In bob-plugins, run `npm run build:check`, `npm run validate`, and:

  ```sh
  node --test scripts/test-ledger-tools-dashboard-parity.cjs scripts/test-ledger-tools-plan-budget.cjs scripts/test-ledger-tools-ready-badge.cjs scripts/test-ledger-tools-today.cjs
  ```

  Run the repository's full `npm test` once on the final tree. The earlier
  implementation reported date-sensitive task-status-cycler successor failures; assess
  any current failure against the current unchanged baseline rather than assuming the
  old report excuses it. Report unrelated failures with evidence. Use `/sase_monitor`
  for a command requiring a background handoff. No Rust behavior change calls for new
  Rust tests.

## Deployment and completion

Deploy only the built bob-ledger-tools plugin from the opened source checkout, first
previewing and then applying:

```sh
bob plugins sync --no-pull --repo <opened-plugin-checkout> --plugin bob-ledger-tools --dry-run --format json
bob plugins sync --no-pull --repo <opened-plugin-checkout> --plugin bob-ledger-tools --format json
bob plugins list --no-pull --repo <opened-plugin-checkout> --format json
```

Inspect managed-file copy/skip results and source-specific sync status; exit zero is not
sufficient proof of deployment. Do not edit installed plugin copies. Existing and future
daily notes pick up the renderer through their `bob-plan` blocks, without rewriting
notes or syncing a changed vault dashboard.

If an Obsidian session is available, reload the updated plugin as needed and compare a
current daily, an older daily, and `dash.md` in Live Preview and Reading view. Verify
the matching lane counts, tooltip explanations, navigation, and compact daily styling in
light/dark appearance. If runtime reload or visual checks cannot be observed, report
that precise limitation; file deployment alone does not prove an open Obsidian process
has loaded the code.

Include plugin and bob-cli documentation changes in their respective SASE finalization
obligations. The implementation report should state the section-count policy, preserved
whole-lane warning policy, tests, and deployment/visual status. This planning turn makes
no implementation or deployment changes.
