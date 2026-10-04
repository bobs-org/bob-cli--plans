---
tier: tale
title: Split navigation-hotkeys tests into a harness and per-area files
size: medium
goal:
  Replace scripts/test-navigation-hotkeys.cjs with one shared harness and per-area test
  files of at most 1000 lines, listed explicitly in package.json. Test names and test
  bodies stay unchanged, and the same 500 names pass.
proposed_by: bbugyi200.apollo.bob-cli-47.4
bead: bob-cli-47.4
create_time: 2026-10-04 09:37:17
status: wip
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:**
  [bob-cli-47.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-47/bob-cli-47.4.md)

# Split navigation-hotkeys tests into a harness and per-area files

Implement phase `bob-cli-47.4` (`split-navigation-hotkeys-tests`) in the linked
`bob-plugins` repo. This is one medium tale: a single throwaway extractor can cut the
file, and the file map below is already measured. An epic would only split one scripted
edit across agents.

## Source of record

Open the repo with `sase repo open bob-plugins` and edit only the checkout path it
prints. Read that checkout's `AGENTS.md` before editing.

Measured on 2026-10-04 at bob-plugins `c8ec83fae6a094a5c3762418200e2e5b91f4e0f3`.
`scripts/test-navigation-hotkeys.cjs` is blob
`7290cb61569302b94bf2f98066f08e9bf036bff0`, `wc -l` 19813, 500 top-level `test()` calls,
zero duplicate names. The file matches the epic's line numbers (unchanged since
`f4b3562` for this path). `git hash-object` must still be that blob before applying the
ranges. If it differs, stop and rebuild the ranges with the membership rules in this
plan. Apply the same rules to the new text.

`package.json` lists this file once, inside the explicit `npm test` file list, between
`scripts/test-dependency-line-migration.cjs` and
`scripts/test-navigation-jump-history.cjs`. Keep every other path in that list as it is.
`scripts/test-navigation-dependencies.cjs`,
`scripts/test-navigation-dependencies-stage.cjs`, and
`scripts/test-navigation-dependencies-writer.cjs` are different suites. Leave them
alone.

## Rules

- Move test bodies and helper bodies byte-for-byte. The extractor writes each
  destination by concatenating the inclusive line ranges below, in the order listed,
  joined by a single blank line when a file has more than one range.
- New code is limited to the per-file require header and the harness `module.exports`.
  Keep headers to the modules and exported names that file actually references.
- Drop the unused `spawnSync` import (line 2) and the unused `os` import (line 5). Those
  identifiers occur only on those two lines.
- Move `const fs = require("node:fs");` (line 3) into
  `scripts/test-navigation-hotkeys-priority-notice.cjs`. That file contains the
  stylesheet test, the only `fs` use.
- Repeat `const test = require("node:test");` as header glue in every test file. The
  harness has no tests, so it does not require `node:test`.
- Repeat `const assert = require("node:assert/strict");` and
  `const path = require("node:path");` in the harness and in every test file whose
  slices reference them. The original lines live in the harness.
- Keep every new file in `scripts/`. The stylesheet test reads
  `__dirname + "/../plugins/bob-navigation-hotkeys/styles.css"`.
- Every hand-edited file this phase creates is at most 1000 lines (`wc -l`). The
  fallback cuts below are the only allowed way to shorten a file that the header pushes
  over that limit.
- Each `node --test` file runs in its own process. Tests already reset `notices` and
  restore the globals they swap. Preserve that. Leave the real clock alone. The
  runtime-prune and cancel tests read it.
- The harness is the only file that installs the `Module._load` stub and the only file
  that requires `plugins/bob-navigation-hotkeys/main.js`. Test files receive
  `NavigationHotkeysPlugin`, `helpers`, and `notices` from the harness so the stub's
  `Notice` and the tests share one array.
- Delete `scripts/test-navigation-hotkeys.cjs` after the new files contain its slices.
  No plugin file changes: no `main.js`, no `src/`, no `manifest.json`, no version bump.

## Membership rules

These rules are how the ranges were chosen. Use them again if the blob changed.

Preamble lines 1–235, minus the test at 66–75, go to the harness. That is the
`global.window` default, `notices`, `parseTestYaml`, the `obsidian` and
`@codemirror/view` stubs, the plugin require, `helpers`, `TestEditor`,
`TransactionEditor`, `vimTransactionEditor`, `RecordingFallbackEditor`,
`compatibleTasksSettings`, and `assertLineBoundedTransaction`.

The contiguous picker block at 3957–4245 goes to the harness as one slice. That includes
the fragment-node builders. The epic's "about L4438–4575" note is the priority-notice
renderer test, which stays a test.

The pomodoro fixture functions at 10335–10412 go to the harness. Callers in the picker
lifecycle, open-task jump, and later pomodoro files use them, and some of those callers
sit above the definitions today. `function` declarations are hoisted in the monolith.
After the split they must exist before any test file runs, which a harness export does.

These cluster helpers also go to the harness because more than one output file calls
them:

- `buildScheduleReasonConfig` (9731–9746), called from the schedule-review tests and the
  cancel-task tests.
- `parseFrontmatterFromContent` (15488–15504), called from project-reversal and
  forward-promotion.
- `createLinkPickerEnergyConfig` and `createLinkPickerHarness` (16359–16458). The
  harness function calls the energy-config function. Link-picker, lane-toggle, and
  cancel-task tests call `createLinkPickerHarness`.

Every other top-level helper stays in the test file of its only caller, at its original
position among those tests:

- `createPomodoroEntryMoveSession`
- `makeSiblingTabsFixture`, `siblingTabIds`
- `openBareScheduleStage` (it calls harness-exported `findPropertyItem`)
- `buildDeferredPomodoroNoteIndex`
- `getTodayDailyFile`
- `workedExampleProjectNote`, `workedExampleParentNote`,
  `expectedRestoredGymHabitBlock`, `createProjectReversalHarness`
- `openLinkPickerValueStage`
- `cancelRowIndex`, `confirmCancelReasonStage`, `openCancelReasonStage`
- `promotionDailyPath`, `createProjectForwardHarness`
- `confirmSchedulingWorkLogStage`, `schedulingWorkLogConfig`, `openSchedulingReview`,
  `schedulingReviewHarness`
- `createTaskMoveVimHistoryHarness`

The test at 66–75, "Pomodoro-marked links are not managed dependency bullets", sits
between the stub and `TestEditor`. It uses only `assert` and `helpers`. Place it at the
top of the schedule-writes file, then that file's main range. Order across files is not
significant. `node --test` isolates processes, and the epic allows gathering tests that
do not depend on order.

## Harness

Create `scripts/navigation-hotkeys-harness.cjs` (no `test-` prefix, same idea as
`scripts/modal-harness.cjs`). It is not an `npm test` entry.

Slices, in this order:

```text
1-1
4-4
6-6
8-8
10-64
77-235
3957-4245
9731-9746
10335-10412
15488-15504
16359-16458
```

That body is 718 lines. End with a `module.exports` object, glue, listing every name
tests call. In source order:

`notices`, `parseTestYaml`, `NavigationHotkeysPlugin`, `helpers`, `TestEditor`,
`TransactionEditor`, `vimTransactionEditor`, `RecordingFallbackEditor`,
`compatibleTasksSettings`, `assertLineBoundedTransaction`,
`createTaskMovePickerHarness`, `createPomodoroMovePickerHarness`,
`createTaskMoveDestinationFocusHarness`, `createBulletPropertyPickerHarness`,
`createPriorityPickerConfig`, `choosePriorityLevel`, `createFragmentNode`,
`createFragmentChild`, `applyFragmentSpec`, `findFragmentNode`, `collectFragmentNodes`,
`nodeHasClass`, `openPropertyStage`, `openBulletPropertyValueStage`, `runCardAction`,
`findPropertyItem`, `confirmScheduleReasonStage`, `buildScheduleReasonConfig`,
`pomodoroFixtureLines`, `countedPomodoroReorderLines`, `currentPomodoroSwapLines`,
`currentPomodoroSwappedLines`, `countedCurrentPomodoroSwapLines`,
`parseFrontmatterFromContent`, `createLinkPickerEnergyConfig`,
`createLinkPickerHarness`.

Keep the export list compact, one name per line, so the harness stays under 1000 lines.
Measured size with that export is about 760 lines. Tighten the export formatting if
`wc -l` ever reaches 1000. A second harness module is out of scope: the picker helpers
close over the preamble scope, and a second file would be a require cycle.

Each test file destructures only the names its slices reference:

```js
const { helpers, notices } = require("./navigation-hotkeys-harness.cjs");
```

## Test files

Body line counts below exclude the require header. The largest body is 959 lines. A
header of the modules and names that file uses should stay under 40 lines. If `wc -l` of
a finished file is over 1000, apply that file's fallback cut and give the second file
the same kind of header. Split only on a `test(` boundary. Both pieces must still be at
most 1000 lines.

| File                                                        | Ranges                        | Body lines | Fallback if over 1000                                                                              |
| ----------------------------------------------------------- | ----------------------------- | ---------- | -------------------------------------------------------------------------------------------------- |
| `scripts/test-navigation-hotkeys-section-nav.cjs`           | 237–459                       | 223        |                                                                                                    |
| `scripts/test-navigation-hotkeys-schedule-recovery.cjs`     | 461–905                       | 445        |                                                                                                    |
| `scripts/test-navigation-hotkeys-schedule-properties.cjs`   | 907–1403                      | 497        |                                                                                                    |
| `scripts/test-navigation-hotkeys-schedule-writes.cjs`       | 66–75, then 1405–1919         | 525        |                                                                                                    |
| `scripts/test-navigation-hotkeys-runtime-schedule.cjs`      | 1921–2471                     | 551        |                                                                                                    |
| `scripts/test-navigation-hotkeys-dependencies.cjs`          | 2473–3339                     | 867        |                                                                                                    |
| `scripts/test-navigation-hotkeys-dependency-mirror.cjs`     | 3341–3753                     | 413        |                                                                                                    |
| `scripts/test-navigation-hotkeys-task-move-plan.cjs`        | 3755–3956                     | 202        |                                                                                                    |
| `scripts/test-navigation-hotkeys-priority-notice.cjs`       | 4247–5086                     | 840        |                                                                                                    |
| `scripts/test-navigation-hotkeys-priority-picker.cjs`       | 5088–5382                     | 295        |                                                                                                    |
| `scripts/test-navigation-hotkeys-picker-lifecycle.cjs`      | 5384–6131                     | 748        |                                                                                                    |
| `scripts/test-navigation-hotkeys-pomodoro-move-pickers.cjs` | 6133–6562                     | 430        |                                                                                                    |
| `scripts/test-navigation-hotkeys-task-move.cjs`             | 6564–7496                     | 933        | 6564–7299 stays; 7300–7496 becomes `scripts/test-navigation-hotkeys-task-move-runtime.cjs`         |
| `scripts/test-navigation-hotkeys-open-task-jump.cjs`        | 7497–7911                     | 415        |                                                                                                    |
| `scripts/test-navigation-hotkeys-vim-counts.cjs`            | 7913–8495                     | 583        |                                                                                                    |
| `scripts/test-navigation-hotkeys-tab-pin.cjs`               | 8497–8835                     | 339        |                                                                                                    |
| `scripts/test-navigation-hotkeys-schedule-log.cjs`          | 8836–9729                     | 894        | 8836–9649 stays; 9651–9729 becomes `scripts/test-navigation-hotkeys-schedule-log-writes.cjs`       |
| `scripts/test-navigation-hotkeys-schedule-review.cjs`       | 9747–9761, then 9763–9954     | 207        |                                                                                                    |
| `scripts/test-navigation-hotkeys-deferred-pomodoro.cjs`     | 9956–10334                    | 379        |                                                                                                    |
| `scripts/test-navigation-hotkeys-pomodoro-parse.cjs`        | 10413–11171                   | 759        |                                                                                                    |
| `scripts/test-navigation-hotkeys-pomodoro-bullet-move.cjs`  | 11172–12113                   | 942        | 11172–11855 stays; 11857–12113 becomes `scripts/test-navigation-hotkeys-pomodoro-bullet-entry.cjs` |
| `scripts/test-navigation-hotkeys-pomodoro-split-rename.cjs` | 12114–13072                   | 959        | 12114–12851 stays; 12853–13072 becomes `scripts/test-navigation-hotkeys-pomodoro-entry-merge.cjs`  |
| `scripts/test-navigation-hotkeys-pomodoro-reorder.cjs`      | 13073–13874                   | 802        |                                                                                                    |
| `scripts/test-navigation-hotkeys-runtime-prune.cjs`         | 13875–14434                   | 560        |                                                                                                    |
| `scripts/test-navigation-hotkeys-project-from-task.cjs`     | 14435–15388                   | 954        | 14435–15178 stays; 15180–15388 becomes `scripts/test-navigation-hotkeys-project-content.cjs`       |
| `scripts/test-navigation-hotkeys-project-reversal.cjs`      | 15389–15487, then 15505–16358 | 953        | Through 16079 stays; 16081–16358 becomes `scripts/test-navigation-hotkeys-project-convert.cjs`     |
| `scripts/test-navigation-hotkeys-link-picker.cjs`           | 16460–16819                   | 360        |                                                                                                    |
| `scripts/test-navigation-hotkeys-lane-toggle.cjs`           | 16820–17340                   | 521        |                                                                                                    |
| `scripts/test-navigation-hotkeys-cancel-log.cjs`            | 17341–17856                   | 516        |                                                                                                    |
| `scripts/test-navigation-hotkeys-cancel-task.cjs`           | 17858–18543                   | 686        |                                                                                                    |
| `scripts/test-navigation-hotkeys-forward-promotion.cjs`     | 18545–19191                   | 647        |                                                                                                    |
| `scripts/test-navigation-hotkeys-schedule-work-log.cjs`     | 19192–19503                   | 312        |                                                                                                    |
| `scripts/test-navigation-hotkeys-task-move-history.cjs`     | 19504–19813                   | 310        |                                                                                                    |

The epic's "about 22–26 files" counted the 26 themes before the six themes over 1000
lines were cut, and before the harness slices were lifted out of those themes.
Thirty-three area files is the cut that keeps each theme together and every body
under 1000. Task-move tests stay in source order (plan, picker lifecycle, insertion,
runtime, Vim history) so the extractor remains a list of contiguous ranges. Gathering
them is unnecessary.

These ranges cover all 500 tests and every retained helper exactly once. Lines 2, 3, 5,
and 7 are the require lines handled above. Every other uncovered line is blank.

Area meanings, for the names above:

- section-nav: section-header and open-task scan tests.
- schedule-recovery: project schedule validation, scheduled recovery, task
  classification.
- schedule-properties: priority config and counted property planning.
- schedule-writes: the early dependency-bullet test, counted schedule writes, project
  schedule propagation, and dependency-line identity.
- runtime-schedule: runtime transclusion toggles and guarded schedule writes.
- dependencies: counted dependencies, transclusion guards, and hand-edit mirror behavior
  through the owning-task clear.
- dependency-mirror: mirror scheduler and the counted Ctrl+D / `N!` tests that follow
  it.
- task-move-plan: pure task-move planning.
- priority-notice: priority notice model, stylesheet, and counted priority picker writes
  through the batch that skips an unchanged rolled date.
- priority-picker: scheduled priority-roll picker behavior.
- picker-lifecycle: bullet, task-move, and Pomodoro bullet-move picker sessions through
  the stale-content guards. `createPomodoroEntryMoveSession` is the first line of the
  next file, not this one.
- pomodoro-move-pickers: that helper plus Pomodoro entry-move picker tests, including
  the task-move picker close test at the end of the original picker cluster.
- task-move: insertion tests and runtime task move.
- open-task-jump: dash restore and `jumpToOpenObsidianTask` / dispatch guards.
- vim-counts: physical chords and pending Vim counts.
- tab-pin: sibling-tab helpers and pin/close tests.
- schedule-log: priority-roll reasons and Schedule Log planning. The shared
  `buildScheduleReasonConfig` slice is in the harness, so this file ends at 9729.
- schedule-review: `openBareScheduleStage` and the combined review stage.
- deferred-pomodoro: deferred link cleanup, including its local index helper.
- pomodoro-parse: parse, rows, and notices. Fixtures are in the harness.
- pomodoro-bullet-move, pomodoro-split-rename, pomodoro-reorder: the three planner
  clusters.
- runtime-prune: runtime pruning, including local `getTodayDailyFile`.
- project-from-task: project creation from a task.
- project-reversal: worked-example helpers and reversal, with
  `parseFrontmatterFromContent` removed into the harness.
- link-picker: `openLinkPickerValueStage` and link-picker tests. The two helpers above
  it are in the harness.
- lane-toggle: lane toggle.
- cancel-log: Cancel Log grammar and `planTaskCancelBatch`.
- cancel-task: cancel row helpers and runtime cancel.
- forward-promotion: promotion helpers and forward conversion.
- schedule-work-log: scheduling Work Log helpers and review.
- task-move-history: `createTaskMoveVimHistoryHarness` and its tests.

## package.json and README

In `package.json`, replace the single `scripts/test-navigation-hotkeys.cjs` token with
the test files in the table order. Keep the explicit space-separated list. Do not switch
the script to a glob. Do not list the harness.

In `README.md`, add `navigation-hotkeys-harness.cjs` to the `scripts/` layout block, and
add one sentence to the `npm test` paragraph: the navigation hotkeys tests are the
per-area `scripts/test-navigation-hotkeys-*.cjs` files and they share
`scripts/navigation-hotkeys-harness.cjs`. Leave the build-contract documentation as it
is. This phase does not change plugin source.

`docs/plugins.md` in bob-cli does not name this test file. Leave bob-cli untouched.

## Procedure

1. Confirm the blob hash. Capture the current test names before editing, from the
   monolith alone:

   ```bash
   node --test --test-reporter=tap scripts/test-navigation-hotkeys.cjs
   ```

   Save the `ok` test names. There are 500, and this isolated run is the baseline. A
   full `npm test` is not the baseline, because
   `scripts/test-navigation-dependencies-stage.cjs` has a known timing flake that is
   outside this phase.

2. Run a throwaway extractor from `/tmp`. It writes the harness and the area files, then
   checks that each source range appears, unchanged and in order, in exactly one
   destination. It also checks that every top-level helper name referenced by a test
   file is either defined in that file or destructured from the harness. Fix the header
   until that check is clean. Keep the extractor out of the repo.

3. Delete the monolith only after that check passes.

4. `wc -l` every new file. Apply a fallback cut only when a file is over 1000.
   `node --check` every new file.

5. Run the new files and diff TAP names against the baseline. The same 500 names pass,
   with no additions and no renames:

   ```bash
   node --test --test-reporter=tap scripts/test-navigation-hotkeys-*.cjs
   ```

6. Run `npm test` and `npm run validate` in bob-plugins. `npm test` already runs
   `build:check` first. Plugin bytes did not change, so the staleness check stays green.

7. If the only `npm test` failure is the 1,000-task timing assertion in
   `scripts/test-navigation-dependencies-stage.cjs` (the 16 ms threshold already noted
   on `bob-cli-47.3`), rerun that file alone. When the isolated rerun passes, record a
   `PROPOSED FOLLOW-UP:` note on `bob-cli-47.4` that cites those existing notes, and
   continue. Any failure in a new `test-navigation-hotkeys-*.cjs` file has to be fixed
   first. A failure that also happens on the clean base tree gets a
   `PROPOSED FOLLOW-UP:` note and does not by itself keep the phase open.

8. Run `bob plugins sync -p bob-navigation-hotkeys`. This phase does not change plugin
   bytes. If sync refuses because vault plugin files are dirty, do not pass `--force`.
   Record the refusal as a `PROPOSED FOLLOW-UP:` note and continue closing once the test
   split itself is verified.

## Close this phase only

Before closing, run `sase bead epic-symbols bob-cli-47.4`. On 2026-10-04 that command
reported no `--epic-symbol` entries. If a later run prints any, re-key each Justfile
line to a still-open bead (the parent epic or a later phase) before close.
`sase bead close` refuses while leftovers remain.

Close only `bob-cli-47.4`:

```bash
sase bead close bob-cli-47.4 --note "<what you verified>"
```

Do not close parent epic `bob-cli-47` or any ancestor. Do not create beads. Discovered
follow-ups go on this phase with
`sase bead note bob-cli-47.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.

The host finalizer commits the bob-plugins checkout. Do not hand-commit.
