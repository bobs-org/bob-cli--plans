---
tier: tale
title: Finish and land the Ctrl+Shift+P Task Card epic
goal: Task Card deletions, focus, and review counts behave per the epic contract,
  the date grammar is documented, navigation-hotkeys 2.0.1 is deployed, and epic bob-cli-42
  is closed with its plan marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-42.land
bead: bob-cli-42
status: done
---

- **PARENT:**
  [202610/ctrl_shift_p_task_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ctrl_shift_p_task_card.md)
- **BEAD:**
  [bob-cli-42](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-42/README.md)

# Finish and land the Ctrl+Shift+P Task Card epic (bob-cli-42)

## Context

Epic `bob-cli-42` (plan `plan:202610/ctrl_shift_p_task_card.md`) shipped the Task Card
first screen of `BulletPropertyPickerModal` in **bob-plugins** (`bob-navigation-hotkeys`
2.0.0) plus bob-cli docs and date-aware `bob ready` hints. All eight phase beads are
closed. The land agent verified the following:

- bob-plugins `npm test` passes 1715/1715 and `npm run validate` passes 6/6.
- bob-cli `cargo test` and `cargo fmt --check` are green.
- The vault is file-identical to the source.
- `sase bead epic-symbols bob-cli-42` lists nothing.
- Concurrent work was integrated: Pending refresh Work Log, the persistent review
  footer, and `N]s` jumps.
- `bob-cli-3z` is closed.
- Follow-up triage outcomes are recorded as a note on `bob-cli-42`. No further triage is
  needed.

The land agent's code review confirmed a few small gaps that the epic itself caused.
This tale fixes them, then closes the epic. That close is the final step.

Do not fix the pre-existing `just lint` failure (clippy `overly_complex_bool_expr` at
`tests/cli/capture/pomodoro_name.rs:808`). It is owned by active epic `bob-cli-28`. The
bob-cli justfile has no `check` recipe. For Rust edits, verification is
`cargo fmt --check` plus `cargo test`.

## Repository access

From the bob-cli checkout, run
`sase repo open bob-plugins -r "Finish bob-cli-42 Task Card landing fixes"`. Use only
the printed path, and read its `AGENTS.md`. Never edit installed vault copies under
`~/bob`. Declare source changes in both repositories through `/sase_final`, not with
manual git commits.

All code edits below are in `plugins/bob-navigation-hotkeys/main.js` of bob-plugins
unless stated otherwise. Locate code by symbol name; line numbers drift.

## 1. Keep failed Task Card deletions on the card

`BulletPropertyPickerModal.deleteTaskCardProperty(propertyName)` handles two inputs:

- `0`, through `clearTaskCardPriority`.
- `Ctrl+D`, through the `delete-property` intent in `dispatchTaskCardIntent`.

The problem:

- It calls `ensureTaskCardStageChrome()` and `showPropertyStage(...)` before it knows
  whether the property is set.
- `deleteSelectedProperty()` then shows "`<name>` is not set on this bullet" and returns
  `false`.
- The modal is left in classic search mode rather than on the card.

Concrete repros:

- `0` on an unprioritized task.
- `Ctrl+D` with the default Schedule selection on an unscheduled task.

Fix:

- When the deletion does not succeed, keep the existing Notice and return the user to
  the card in the same session, with nothing written. This covers a missing item, an
  undefined property, or a stale/refused writer result.
- When a refusal refreshed `this.lineText`, rebuild the card model from the new line.
- For single sessions, prefer a cheap pre-check through
  `getTaskCardPropertyItem(propertyName)` (its `defined` flag) so the card is never torn
  down.
- Counted, linked, and `^prj` sessions must still go through the existing
  `deleteSelectedProperty` writers. Do not add a new mutation path.

Add harness tests in `scripts/test-navigation-task-card-view.cjs`:

- `0` on an unprioritized single task: no write, the modal stays open, and the stage is
  `task-card`.
- `Ctrl+D` with default selection on an unscheduled task: same expectations.
- `0` on a task with both priority and `[scheduled:: …]`: deletes only priority, and the
  written line keeps the scheduled field byte-for-byte. Today's model test only asserts
  the `keepScheduled` flag.

## 2. Card accelerators from the banner and priority strip

The card view builder attaches `options.onKeydown` (`handleTaskCardKeydown`) only to
`listEl`. Two other elements can take focus:

- The recommendation banner (`role="button"`, `tabindex=0`).
- The priority-strip radios (`role="radio"`).

When either one has focus, none of these card keys reach the resolver: digits, `0`,
`b`/`f`/`x`, `Ctrl/Cmd+Enter`, `Ctrl+R`, `Alt+N`, `/`, and printable search seeding.

Fix:

- Route keydown from the banner and radios to the same handler after their own
  Enter/Space/arrow handling. Alternatively, attach a single handler at the card
  container and remove the `listEl`-only attachment.
- Each physical key must dispatch exactly once, including when focus is on `listEl`.
- Make the banner and radio Enter/Space activation ignore `event.repeat === true` and
  IME composition (`event.isComposing === true` or `keyCode === 229`). The plan contract
  requires that held keys and composition never approve a write.

Add tests:

- Focus a radio and press `2`: exactly one P2 commit through the existing path.
- A held Enter repeat on a radio or the banner writes nothing.
- A key pressed on `listEl` dispatches once (use a spy).

## 3. Combined review counts unique tasks

`planScheduleReview` computes
`totalCount = Math.max(totalIdentities.size, targets.length)`.

- `freezeSchedulingWorkLogTargets` guarantees every target has an identity
  (`path#^blockId` or `path::line`).
- So `totalCount` never de-duplicates.
- Two Task Links to the same task therefore read "Schedule 2 tasks" or "1 of 2 …
  qualify".

Fix: use the identity count, falling back to `targets.length` only when there are no
identities. This matches the existing serial adapter, which already uses
`collectSchedulingWorkLogTargetIdentities(targets).size`.

Add a duplicate-target case to `scripts/test-navigation-task-card-review.cjs`.

## 4. Remove dead Task Card code

Remove these, after grepping `main.js` and `scripts/` for every reference:

- `async setTaskCardPilotEnabled(enabled)` on the plugin class. It has no callers;
  `setTaskCardPreference` is the live API.
- `taskCardRowAction(rowId)` on the modal. It has no callers.
- The `move-selection`, `open-search`, and `back` branches inside
  `dispatchTaskCardIntent`. `handleTaskCardKeydown` handles these intents before
  dispatch. Remove a branch only if no click or tap path produces that intent; keep it
  otherwise.
- The unreachable search-mode tail at the end of `resolveTaskCardKey`, after its early
  search-mode return.

Keep the `module.exports.helpers` convention, and adjust tests only if they referenced
removed members.

## 5. Docs, manifest, and version

Bump `bob-navigation-hotkeys` to **2.0.1** in `manifest.json`. In the bob-plugins
`README.md`, update the plugin table version and the "ahead of the others at `2.0.0`"
sentence.

Make these documentation edits concise.

bob-plugins `README.md`, Task Card section:

- Add a short **Scheduling input** paragraph. It applies in Task Card flows and in
  search reached from an enabled card. It covers:
  - bare `N` days (`0` today, `1` tomorrow);
  - unsigned `Nd`/`Nw`/`Nm`;
  - weekday names `mon`…`sun`, meaning the next occurrence strictly after today;
  - the existing ISO, M/D, M-D, `+Nd/w/m`, and preset forms;
  - the preview row: weekday, ISO date, relative distance, and the year at rollover;
  - an inline reason after a complete date token plus whitespace (`3 waiting on API`);
  - `Shift+Enter`, which skips the reason via the blank-reason rule without skipping an
    applicable Work summary;
  - otherwise, one combined Reason/Work summary review;
  - invalid, negative, overflow, or ambiguous input never writes;
  - Classic list keeps the old parser and serial prompts.
- In the key table's `1`–`4` row, note that extra configured levels get unique digits up
  to `9`.

bob-plugins `README.md`, navigation-hotkeys table row:

- Label these phrases as classic-search behavior: "pinned lane row", "pinned Cancel
  row", "`scheduled` row opens first and selected", and "prompts for an optional reason
  after a `scheduled` date is chosen".
- Briefly name the Task Card equivalents: `Alt+N`, `x`, Enter → Schedule, and the
  combined review.

`manifest.json` `description`:

- Lead with one sentence about the Task Card first screen (exact P-level previews,
  classic search, automatic from 2026-10-19).
- Keep the remaining feature mentions.
- `npm run validate` must still pass.

bob-cli `docs/projects.md`, `### Task Card` section:

- Add the same concise scheduling-input grammar paragraph.
- Point to the existing combined-review text in "Schedule-log reason prompt" rather than
  duplicating it.

## 6. Verify and deploy

In bob-plugins:

- `node --check plugins/bob-navigation-hotkeys/main.js`
- `node --test scripts/test-navigation-task-card-model.cjs scripts/test-navigation-task-card-view.cjs scripts/test-navigation-task-card-schedule.cjs scripts/test-navigation-task-card-review.cjs scripts/test-navigation-decision-card.cjs scripts/test-navigation-refresh-modal.cjs`
- `npm run validate`
- `npm test` once after integration. The pre-change baseline is 1715 passing; expect
  more with the new tests and zero failures.

Deploy from the opened bob-plugins checkout:

1. `bob plugins sync --repo <opened bob-plugins path> --no-pull --dry-run`
2. The real sync.
3. A final dry run that reports every plugin up to date.

Obsidian GUI access is not expected. If it is absent, say so in the close note rather
than claiming a reload.

bob-cli has only docs edits here, so no Rust run is needed. If you touch Rust, run
`cargo fmt --check` and `cargo test`.

## 7. Close out epic bob-cli-42 (final step)

1. Run `sase bead epic-symbols bob-cli-42`. For each listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it,
   per the Symvision epic-whitelist policy. Re-key a Justfile line to an open bead only
   if a still-open later bead needs the exemption. The land agent saw no entries.
2. Close the epic with `sase bead close bob-cli-42 --note "<verification>"`. The note
   must state:
   - All 8 phases are closed, and their notes and PROPOSED FOLLOW-UPs were triaged (see
     the land-triage note on `bob-cli-42`).
   - Epic commits were verified: bob-plugins aa38c1d, 8813271, c0ff974, b3269d9,
     48f0466, a0b788c, e872aee, dae2dd2, plus this tale's commit; bob-cli 2b754c8, plus
     this tale's docs.
   - Test results: `npm test` and `npm run validate` counts; bob-cli `cargo test` and
     `cargo fmt --check` green.
   - `just lint` is red only from the pre-existing clippy deny owned by `bob-cli-28`.
   - Integration was checked against concurrent bob-plugins e4aa7d0, 84cdbc7, and
     10cfee3 and bob-cli 223974d, 0b7693b, and 787365f, with no conflicts. `bob-cli-3z`
     was closed as fixed by aa38c1d.
   - This tale's fixes (sections 1–5) are done, and the vault is file-identical to the
     2.0.1 source.
   - **Outstanding manual GUI acceptance for Bryan before 2026-10-19:**
     - light/dark/narrow screenshots of the card and the combined review;
     - reload Obsidian for the palette rename;
     - smoke-test the date input;
     - exact digit previews;
     - the reason vs Work summary distinction;
     - the Oct 18/19 activation boundary;
     - warm paint ≤100 ms and search ≤50 ms;
     - a 20–30-use tally.
   - Setting **Classic list** holds the default if acceptance fails.

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   `--epic-symbol` entries, clean them up and close again.

3. Run `just symvision` if the recipe exists (`just --list`). The bob-cli justfile
   currently has none. In that case, record that it is unavailable in your final
   response.
4. Set `status: done` in the frontmatter of the epic's plan file. Use the PLAN path
   printed by `sase bead read bob-cli-42 -r "Need the plan path"`
   (`plan:202610/ctrl_shift_p_task_card.md`), and change only the `status:` line.
5. `bob-cli-42` has no `parent_bead` (`parent_id` is null), so there is no parent phase
   or plan to close. Finish normally and declare both repositories through
   `/sase_final`.
