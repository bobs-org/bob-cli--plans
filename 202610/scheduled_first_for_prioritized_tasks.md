---
tier: tale
title: Put scheduled first for prioritized tasks
goal: Opening the task property picker on a prioritized task selects scheduled first
  so the existing recommended action is one Ctrl+Enter away.
size: small
proposed_by: bbugyi200.apollo.4a
status: done
---

# Put scheduled first in the prioritized task property picker

## Goal

When Bryan opens Obsidian's `Ctrl+Shift+P` / **Set bullet property** picker on a task
with a priority property, the `scheduled` row must be the first visible option and the
initial selection. He can then immediately press `Ctrl+Enter` (`Cmd+Enter` on macOS) to
apply the existing previewed recommendation, without navigating past the lane, refresh,
or priority rows.

This is a `tale` with `size: small`: the root cause is known, the menu change is
localized to one plugin, and one coding agent can implement, verify, document, and
deploy it. There are no independent implementation phases.

## Findings and ownership

The implementation belongs to the linked **bob-plugins** repository, not the installed
plugin files in the Bob vault. From the implementing agent's own workspace, run:

```sh
sase repo open bob-plugins -r "Implement scheduled-first ordering for prioritized tasks"
```

Read its `AGENTS.md` and use the printed checkout path for all plugin work. The primary
**bob-cli** repository owns the user-facing contract in `docs/projects.md`; update that
document in the agent's own primary checkout. No Rust CLI behavior or Bob Mac Capture
interface needs to change.

Relevant code in bob-plugins:

- `plugins/bob-navigation-hotkeys/main.js`: `createBulletPropertyItems`,
  `createCountedBulletPropertyItems`, and `createLinkPickerPropertyItems` sort existing
  properties before absent ones, with configuration order as the tie-breaker.
- `BulletPropertyPickerModal.showPropertyStage` then prepends the lane action and
  refresh action, appends Cancel, and resets `selectedIndex` to zero. Changing only a
  property-builder sort would still leave action rows above `scheduled`.
- `FilteredPickerModal.getFilteredItems` preserves list order while filtering.
  `applyOptions` supplies the assembled list to both `items` and `visibleItems`.
- The existing `BulletPropertyPickerModal.handleKeydown` already applies the cached
  recommendation when `Ctrl+Enter` is pressed on the selected schedule property.
  Recommendation creation happens before `showPropertyStage`.
- Existing runtime coverage in `scripts/test-navigation-roll-decay.cjs` manually calls
  `selectRollPropertyRow(picker, "scheduled")` before testing the shortcut, so it does
  not verify the requested immediate-keypress flow.

Baseline verification passed before plan submission:

```sh
node --test --test-reporter=dot scripts/test-navigation-hotkeys.cjs scripts/test-navigation-roll-decay.cjs scripts/test-navigation-stamps.cjs
```

## Behavior contract

1. On a real task with a defined configured priority property, promote its associated
   date-property row to the front of the fully assembled stage-one menu. The default
   association is `priority.schedules: scheduled`. Honor the configured `schedules` name
   rather than hard-coding a different date field. Resolve multiple configured priority
   properties deterministically in configuration order.
2. Presence, rather than an existing scheduled date or a valid recommendation, controls
   the promotion. The date row comes first even when the schedule is absent, another
   field is already defined, or priority is an unconfigured value. This display policy
   does not make otherwise ineligible tasks rollable: closed tasks, unsupported priority
   values, and unavailable recommendations keep their current shortcut fallback and
   refusal behavior.
3. Apply the same policy in counted and Task Link sessions when at least one actual
   target defines priority. Use the aggregate rows' `defined` / source states; a mixed
   aggregate's empty `currentValue` does not mean priority is absent. Inspect linked
   targets, not metadata on the link bullet itself.
4. Preserve the relative order and identity of every other row. For a prioritized task
   the result is schedule, then the existing lane and refresh actions when available,
   then the remaining property rows, with Cancel last. For tasks without priority and
   non-task bullets, retain the existing menu order. If the configured schedule row is
   missing, retain the existing list.
5. The first row is initially selected when the picker opens. When rebuilding stage one,
   retain the existing explicit `selectPropertyName` behavior. Filtering continues to
   exclude nonmatching rows; whenever the promoted schedule row matches, it precedes the
   other matches.
6. Ordinary Enter still opens the date picker. `Ctrl+Enter` applies the existing
   previewed roll, decay, or cancel; it must not compute a new date merely because the
   list was reordered. `Ctrl+R`, delete-property, stale-write refusal, recurring-task
   refusal, guarded batch writes, project-frontmatter scheduling, Schedule Logs,
   freshness stamps, and live-link cleanup retain their existing behavior.

## Implementation

1. Add the promotion at the end of `showPropertyStage`'s row assembly, after lane,
   refresh, and Cancel insertion and before `applyOptions`. Keep it a stable move of the
   existing schedule row. A small pure helper is appropriate to express this policy and
   make edge cases easy to verify; use the existing property parser and task-context
   checks rather than a second regex parser. Leave the three builders' general
   defined-first ordering intact.
2. Add focused coverage to `scripts/test-navigation-hotkeys.cjs` and
   `scripts/test-navigation-roll-decay.cjs`, using their existing modal and transaction
   harnesses. The essential regression must open a prioritized task picker, assert that
   `visibleItems[0]` is the schedule property and `selectedIndex` is zero, then press
   `Ctrl+Enter` without calling a selection helper. Assert the previewed date, expected
   Schedule Log, one undo group, and modal closure. Retain tests that explicitly select
   other rows.
3. Cover ordering with schedule present and absent, priority before schedule in
   configuration, another defined property, and both lane and refresh rows available.
   Also cover a task without priority, a plain bullet carrying priority-like metadata,
   an unconfigured priority value, and a closed task. Verify that the rest of the row
   sequence is unchanged and filtering can hide the schedule row normally.
4. Exercise the same initial-selection shortcut for a prioritized `^prj` task, a mixed
   counted batch, and a Task Link session. Verify that a mixed batch with some
   unprioritized targets still promotes schedule, and a batch with no priorities retains
   its old order. Reuse existing roll/decay/cancel and refusal tests rather than
   duplicating their full behavioral matrices.
5. Update the **Recommended roll and priority decay** section of bob-cli's
   `docs/projects.md` to describe the immediate `Ctrl+Shift+P`, `Ctrl+Enter` flow and
   priority-dependent first-row behavior. Update bob-plugins' README description and the
   navigation plugin manifest version consistently. Keep the documentation explicit that
   the shortcut takes the recommended roll, decay, or cancel. No memory edits are
   needed.

## Verification and deployment

Run from the opened bob-plugins checkout:

```sh
npm test
npm run validate
```

Resolve any failures caused by this change, then review the diff for a stable row move
and confirm that existing mutation paths were not changed.

The linked repository's `AGENTS.md` requires `bob plugins sync` after edits. Use the
actual opened checkout as the source, skip pulling to deploy exactly the verified code,
and scope deployment to this plugin:

```sh
bob plugins sync --no-pull --repo <opened-bob-plugins-checkout> --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo <opened-bob-plugins-checkout> --plugin bob-navigation-hotkeys
```

The angle-bracket value is a placeholder for the path printed by `sase repo open`, not a
literal shell argument. Do not use the default repo source or edit vault plugin files
directly. Preserve sync's normal backup and drift checks. Report sync's actual result
and whether an Obsidian plugin reload is needed. A GUI smoke check, when Obsidian is
available, should confirm that opening the picker on a prioritized task immediately
selects `scheduled`, ordinary Enter opens dates, and the recommended shortcut uses the
displayed action. Report any GUI validation that could not be performed accurately.

Completion requires the prioritized first-row behavior in all existing task picker
modes, passing plugin tests and manifest validation, aligned docs and version metadata,
and deployment from the verified linked checkout.
