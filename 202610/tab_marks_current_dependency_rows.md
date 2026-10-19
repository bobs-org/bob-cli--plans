---
tier: tale
title: Tab marks CURRENT rows in the Depends on and Blocks stages
goal:
  Tab marks or unmarks CURRENT prerequisite/dependent rows for removal, and applying the
  marks actually removes same-note links.
size: small
proposed_by: bbugyi200.athena.11x
proposed_with: claude/opus
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.11x](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.11x.md)
- **COMMITS:**
  - [fcde586](https://github.com/bobs-org/bob-plugins/commit/fcde586d76a534eddbf86ff61afb6821c1429637)
    — fix(navigation): Tab marks CURRENT Depends-On and Blocks rows

# Fix: Tab cannot mark CURRENT rows in the Depends on / Blocks stages

## Problem

In the nav (`bob-navigation-hotkeys`) Depends on stage (Ctrl+Shift+B) and Blocks stage
(Ctrl+Shift+Alt+B), Tab marks RESULTS/BLOCKED rows but does nothing on CURRENT rows. It
neither marks the row nor moves the highlight. `docs/task-dependencies.md` §6.4 and
§15.5 (bob-cli) say ⇥ marks or unmarks **any** row: `＋ add`, `− remove`, or `＋ id`. So
marking a CURRENT row for removal is the intended behavior.

## Root cause (verified by reproduction against the existing test harness)

There are two independent defects. Both live in the `bob-plugins` linked repo (open it
with `/sase_repo`; read its `AGENTS.md`).

1. **Tab guard rejects CURRENT rows.** `toggleHighlightedLocalTaskMark()` in
   `plugins/bob-navigation-hotkeys/src/400-picker-local-task.js` returns early unless
   `item.kind === "local-task"`. That guard predates the vault-wide stage. When the
   vault stage landed (08d1560), CURRENT rows got `kind: "current"`:
   - Depends on: `resolveDependencyStageCurrent` (`290-local-task-items.js`) builds them
     with `kind: "current"`. `planDependencyStageView` (`300-dependency-stage.js`)
     spreads them into `markedCurrent` without resetting `kind`.
   - Blocks: `planBlocksStageView` (`302-blocks-stage-view.js`) writes
     `{ kind: "local-task", ...row, ... }`. The spread of the `decorateBlocksVaultStage`
     row (which has `kind: "current"`) overwrites the intended `"local-task"`.

   So Tab is a silent no-op on every CURRENT row in both stages. Existing tests never
   caught this because they put CURRENT mark keys straight into `modal.markedLines`
   instead of pressing Tab.

2. **Same-note marked removals of CURRENT rows are silently dropped (Depends on only).**
   This is hidden behind defect 1, but it surfaces as soon as defect 1 is fixed.
   `commitMarkedDependencies()` (`420-picker-counted-dependency.js`) builds each removal
   as
   `{ path, blockId, depValue: item.value, legacyDepValue, linkBlockId: item.existingBlockId || item.existingIdField || item.value }`.
   CURRENT rows carry `blockId` but none of `value`, `existingBlockId`, or
   `existingIdField`. Both same-note resolvers then search only `linkBlockId`,
   `depValue`, and `legacyDepValue`, find nothing, and push no `removeRef`:
   - `executeDependencyBatchWithoutInboxRoute` (`430-picker-vault-batch.js`, the
     "Removal marks carry the field value" block), used when every mark is in the
     dependent's note;
   - the same-note branch of `executeVaultDependencyBatchWithoutInboxRoute`
     (`440-picker-vault-commit.js`, the `for (const removal of batch.removals)` loop),
     used for mixed batches.

   The planner still runs, so the user sees "⛓ Dependencies updated · …" while the link
   stays. This is exactly Bryan's case: every prerequisite of `Write introduction!` in
   `sase_blog.md` is a same-note `[[#^…]]` link. Cross-note removals already work,
   because they use `removal.blockId`. The Blocks stage apply path (`commitMarkedBlocks`
   → `commitBlocksEdit`) already handles CURRENT rows correctly.

Reproduction results before the fix, using the harness in
`scripts/navigation-dependencies-stage-harness.cjs`:

- Tab on a CURRENT row: `getMarkedCount() === 0`, and `selectedIndex` is unchanged (both
  stages).
- A forced mark on a same-note CURRENT row followed by `commitMarkedDependencies()`
  resolves `true` with "Dependencies updated", but the Depends-On line and
  `[dependsOn::]` are unchanged.
- With only the Tab guard relaxed (prototyped on an instance), the subtitle reads "1 to
  remove · ↵ to apply", the footer reads "apply 1", and the row renders
  `is-marked-remove`. No render or footer work is needed beyond the guard.

## Changes (all in `bob-plugins`, plugin `bob-navigation-hotkeys`)

Edit `src/` fragments only, then run `npm run build`. Never hand-edit `main.js`. Every
touched fragment stays well under the 1000-line cap.

1. **`src/400-picker-local-task.js`, `toggleHighlightedLocalTaskMark()`**: accept
   CURRENT rows. Change the guard to reject only rows that are neither task rows nor
   CURRENT rows:
   `if (!item || (item.kind !== "local-task" && item.stageSection !== "current")) return;`
   Add a one-line comment saying CURRENT rows keep their builder's `kind` and mark as
   `− remove` (§6.4 / §15.5). Leave everything else unchanged: the disabled-row notice,
   the mark-key toggle, the footer refresh, and `moveSelection(1)`. `stage-more` rows
   stay rejected because they are never in the `current` section. Counted sessions are
   unaffected because they never reach this method.

   Fix it here rather than normalizing `kind` in the two row builders: one guard covers
   both stages, and nothing else branches on `kind` for these rows.

2. **Resolve same-note removals by the row's own `blockId` first.** In both same-note
   resolvers named in root cause 2:
   - In `430-picker-vault-batch.js`, at the top of the `removals.forEach` body: if
     `normalizeBulletPropertyValue(removal.blockId || "")` is non-empty, push
     `{ path: filePath, blockId: <that value> }` and return. Otherwise fall through to
     the existing `linkBlockId` / `depValue` / `legacyDepValue` / `[id::]` scan, which
     stays for legacy rows without a block id.
   - In `440-picker-vault-commit.js`, in the same-note branch of the `batch.removals`
     loop: apply the same direct `removal.blockId` short-circuit, pushing
     `{ path: ownerPath, blockId }`, before the existing candidate scan.

   This matches the single ↵ path (`removeSingleDependency` → `applyDependencyEdit`
   `remove: [{ path, blockId }]`) and the cross-note batch branch. Because it does not
   need the target task line to exist in the note, a missing/archived same-note target
   (`^x not found` CURRENT row) can also be removed. The planner already handles
   removing an unresolvable link. Update the two "Removal marks carry the field value"
   comments to mention the direct block id first.

3. **`manifest.json`**: bump `bob-navigation-hotkeys` from `2.46.0` to `2.46.1` (patch,
   per the repo's fix convention). The description text needs no change.

## Tests (regression coverage that drives the real Tab keydown)

Build every fixture on the existing harness
(`scripts/navigation-dependencies-stage-harness.cjs`). Drive Tab through
`modal.handleKeydown({ key: "Tab", preventDefault() {}, stopPropagation() {} })`, never
by writing `markedLines` directly. The stub modal has no `resultsEl`, so stub
`modal.renderResults = () => {}` before pressing Tab, because `moveSelection`
re-renders.

Add these to `scripts/test-navigation-dependencies-stage-writes.cjs`, using a warm empty
Tasks cache stub as the existing cycle test does:

- **Tab marks a same-note CURRENT row and ↵ removes it.** Use a dependent shaped like
  the vault: `[?]` with `[id::]`, `[dependsOn:: B__outline, B__read]`, `^write`, a
  tab-indented `⛓️ **DEPENDS ON:** [[#^outline]] • [[#^read]]` child, and open targets.
  Highlight `^read`, then press Tab and assert:
  - `getMarkedCount() === 1`;
  - `selectedIndex` advanced by 1;
  - `getVaultStageSubtitle(...)` contains `1 to remove`;
  - the footer ↵ label is `apply 1`;
  - `renderTaskValueItem` on a `stubRowEl()` adds `is-marked-remove`.

  Then `await modal.commitMarkedDependencies()`. The result is `true`, the Depends-On
  line no longer contains `[[#^read]]` but keeps `[[#^outline]]`, `[dependsOn::]` drops
  the `^read` id, and `editor.undoGroups === 1`.

- **Tab twice unmarks.** Tab on the CURRENT row, move the selection back, Tab again:
  `getMarkedCount() === 0`.
- **Mixed batch.** Tab a same-note CURRENT row and a RESULTS row, then apply. One commit
  removes the first and adds the second.
- **Missing target.** A CURRENT row for `[[#^gone]]` (no such block in the note) can be
  Tab-marked and removed by the batch.
- **Cross-note plus same-note CURRENT removal in one batch** (exercises the
  `440-picker-vault-commit.js` branch). Put the other note in an open buffer through
  `workspace.getLeavesOfType`, as `pluginWithNotes` does in
  `test-navigation-blocks-stage-view.cjs`. Tab both CURRENT rows and apply; both links
  are removed.

Add this to `scripts/test-navigation-blocks-stage-view.cjs`:

- **Tab marks a Blocks CURRENT row.** Open with `openBlocks` on a note N that has one
  same-note dependent and one cross-note dependent (`[?]` tasks whose Depends-On links
  `^n`). Set `modal.visibleItems = modal.items`, stub `renderResults`, highlight the
  first CURRENT row, and press Tab. Assert `getMarkedCount() === 1`, that the selection
  moved, and that `getBlocksStageFooterHints()` ↵ reads `apply 1`. Tab the second
  CURRENT row too, then `await modal.commitMarkedDependencies()`. The result is `true`,
  and neither dependent's line keeps a Depends-On link to `^n`.

Before the fix, every Tab assertion fails, and the same-note apply assertions fail with
the link still present. Run the new tests against the unfixed build first to confirm
they fail for the right reason.

## Verification

1. In `bob-plugins`, run `npm run build`, then `npm test` (this includes `build:check`),
   then `npm run validate`. Everything passes.
2. Run `bob plugins sync` to deploy to the vault, as `bob-plugins/AGENTS.md` requires.
3. Manual check (optional, report if skipped): in `sase_blog.md` on
   `Write introduction!`, press Ctrl+Shift+B, then Tab on a CURRENT row. The row shows
   the ✓ mark and `− remove` styling, the highlight moves down, and the subtitle reads
   `1 to remove · ↵ to apply`. Esc leaves the note untouched.

## Out of scope

- Counted sessions (`N<Ctrl+Shift+B>`): Tab marks stay disabled there by design.
- The same-note batch executor passes only the dependent note to `planDependencyEdit`,
  so a kept cross-note link with no matching `[dependsOn::]` field fails
  `target-not-found`. This is a pre-existing edge case, unrelated to Tab, and vault
  lines carry the derived field. Do not change it here.
- No memory or bob-cli doc changes. §6.4 / §15.5 already describe the fixed behavior.
