---
tier: tale
title: Ctrl+Shift+P dependency adds unlink the task from today's Pomodoros
goal:
  Adding an open prerequisite through the Ctrl+Shift+P Task Card immediately removes the
  newly Blocked task's live Task Links from today's open Pomodoros, deleting any
  Pomodoro entry the removal empties.
size: medium
decisions:
  hand_edits:
    ask:
      Should hand-typed Depends-On links also unlink the task from today's open
      Pomodoros right away?
    default: false
    why:
      You named the Ctrl+Shift+P keymap; the existing ^prj consumption also skips hand
      edits
    answer: false
proposed_by: bbugyi200.apollo.7j
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.7j](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.7j.md)
- **COMMITS:**
  - [ae9e6ef](https://github.com/bobs-org/bob-cli/commit/ae9e6efdced09d880bda63ead7b73aa82206dd86)
    — docs(task-dependencies): consume today's Pomodoro links on Depends-On add

# Plan: Ctrl+Shift+P Depends-On adds unlink the newly Blocked task from today's Pomodoros

## Goal

Adding a dependency with the `Ctrl+Shift+P` Task Card (Depends on stage) can make a task
Blocked `[?]`. When that happens, delete every live Task Link to that task in
**today's** daily note right away. If a removed link was the last child bullet of its
Pomodoro entry, delete the entry too. Today this happens right away only for a `^prj`
dependent. For an ordinary task, the stale link stays until the next
`bob task reconcile`.

## Background (current state, verified)

- **Contract.** `docs/task-dependencies.md` §5 (bob-cli) says a Blocked task keeps no
  live Task Link under today's open Pomodoros. Reconcile enforces this ("Blocked task
  links are consumed", `docs/task-status-hooks.md`). It removes link-only bullets with
  their subtrees, strips tokens from prose bullets, and then applies the empty-Pomodoro
  rule.
- **Capture already does this.** In capture `&`, `src/native/capture/dependencies.rs`
  (bob-cli `66c4ddb`) has the comment "Any `&` that leaves its dependent Blocked
  consumes the dependent's own live links the same way." It runs `consume_live_links` on
  the dependent's block id whenever the dependent ends up `[?]` with an open target.
  This plan gives the plugin the same behavior.
- **The nav plugin only does this for `^prj` dependents.** All of the following are in
  the bob-plugins linked repo, under `plugins/bob-navigation-hotkeys/src/`:
  - `planDependencyEdit` (`080-dependency-identity.js`) emits
    `summary.project.pomodoroTargets` (`{path, blockId}`) only through
    `summarizeProjectDependencyEdit` (`216-project-dependency-stage.js`).
  - `prepareProjectPomodoroCleanup`, `finishProjectPomodoroCleanup` and
    `projectPomodoroNotice` (`216`) read only those project targets.
  - `prepareProjectPomodoroCleanup` reads today's daily note with
    `readDeferredPomodoroSnapshot` (`500-plugin-transclusion-and-link-picker.js`). It
    plans the removal with
    `planDeferredPomodoroLinkCleanup(..., {removeEmptiedEntries: true})`
    (`160-pomodoro-reorder.js`). When the edited note is today's daily note, it folds
    the removal into the same write. Otherwise `writeDeferredPomodoroCleanup` writes it
    afterwards with a preimage check.
  - Only `applyDependencyEdit` (`580-plugin-properties-and-dependency-edit.js`) and the
    same-note marked batch `executeDependencyBatchWithoutInboxRoute`
    (`430-picker-vault-batch.js`) call that pair.
  - The same-note single-select writer `setLocalTaskDependencyLink`
    (`600-plugin-dependency-mirror.js`) never consumes Pomodoro links. Neither do the
    counted writers `applyCountedLocalTaskDependency`
    (`570-plugin-link-roll-and-schedule.js` + `planCountedLocalTaskDependency` in
    `270-counted-batches.js`) and `applyVaultCountedDependencyRefWithoutInboxRoute`
    (`420-picker-counted-dependency.js`). The only exception is the cleanup of
    sub-project subtree targets.

## Behavior (the rule every Ctrl+Shift+P Depends-On writer follows)

1. **Trigger.** The gesture adds at least one **open** prerequisite, and the dependent
   is `[?]` after planning. This includes a dependent that was already `[?]`, as in
   capture `&`. Removals, Ctrl+D on the Depends on row, and adds of only closed targets
   never consume anything.
2. **Identity.** The consumed target is the dependent's own `{path, blockId}` from its
   trailing `^block-id`. A dependent without a block id cannot have a Task Link, so
   nothing is read or written for it. On `^prj`, keep the existing project targets and
   take the deduplicated union.
3. **Scope.** Only today's daily note (`scheduledRecoveryDailyPaths(...).current`, as
   `readDeferredPomodoroSnapshot` already resolves it). Only live (unstruck) links under
   **open** Pomodoro entries count: the current timed entry and planned placeholders.
   Closed or cancelled entries, `🍅` history, other days, Depends-On and BLOCKS lines,
   and fenced code are never touched. Closed entries are history, and reconcile and
   capture never edit them either. Matching reuses `planDeferredPomodoroLinkCleanup`
   unchanged, so every link form it resolves counts: plain, embedded, 🍅-marked, `#`
   move-only, same-note `[[#^id]]`, and full routes.
4. **Removal.** A dedicated link bullet goes with its subtree. A prose bullet loses only
   the link token. With `removeEmptiedEntries: true`, an open entry left with no child
   bullets is deleted, even a running timed entry. This matches reconcile and capture.
   An entry that still has any other child bullet stays.
5. **Write order and undo.** If the dependent's note _is_ today's daily note, the
   removal folds into the dependent's single editor transaction (one Ctrl+Z). Otherwise
   it is a follow-up write after the dependent's commit, with a preimage check: an
   editor transaction if the daily note is open, else `vault.process`. A failed
   follow-up is never rolled back. It adds `; Pomodoro links not removed` to the notice,
   and reconcile repairs it later.
6. **Promotion is unchanged.** Commitment transfer (`getDependencyPromotionStatus`)
   already raises each added open prerequisite to the dependent's pre-gesture lane. A
   task linked today is Next or In Progress under sticky lanes, so its prerequisite
   already reaches at least Next. Capture `&` behaves the same way. Do not add a
   separate promotion seed.
7. **Notice.** When N > 0 links are removed, append ` · removed N Pomodoro link(s)` (via
   `formatCountLabel(N, "Pomodoro link")`) to the gesture's existing notice. For
   example:
   `⛓ Now waits on "File for unemployment" · Blocked · removed 1 Pomodoro link`.
   Counted notices get the same suffix. Keep the `^prj` notices exactly as they are
   today.
8. **Reads.** At most one extra note is read: today's daily note, with an open buffer
   preferred, and only when there is a consumption target. Never take the vault-wide
   recovery snapshot for an add. Without a daily note, `readDeferredPomodoroSnapshot`
   returns null before any `cachedRead`, so the existing tests that count zero reads
   ("adds without recovery read zero extra vault notes", "adds on Blocked parents and
   non-Blocked mirror touches read nothing" in
   `scripts/test-navigation-dependencies-writer.cjs`) must stay green unchanged.

Entry points covered, since they all reach the writers above:

- Task Card `b` / Depends on, for same-note and vault rows and for marked batches.
- The `＋ id` prompt flows (`430` `confirm*BlockId`).
- Counted `N<Ctrl+Shift+P>`, both same-note and vault.
- Ctrl+Shift+P on a Task Link bullet. Task Link mode opens the linked task's own stage
  through `openLinkedDependencyStage`, so the link under the cursor is itself consumed.
- The ledger-tools chip `api.openDependencyStage`.
- Inbox-routed commits. These wrap the same writers through `runInboxRoutedCommit`, so
  they need no inbox-specific code. Verify that the consumed link resolves for the path
  the write actually targets.

> [!decision] hand_edits Also pass the consumption target through when
> `applyDependencyEdit` runs with `mirrorTouch: true` (the hand-edit mirror, `600`
> `applyDependencyHandEditClear` / `fireDependencyHandEditMirror` →
> `applyDependencyEdit`). Planner trigger for the mirror is "the dependent is `[?]`
> after the touch and the line has an open prerequisite" (the mirror's existing blocking
> rule). The `^prj` mirror stays as today. Add one mirror test.

> [!decision] hand_edits = no `planDependencyEdit` emits no dependent target when
> `mirrorTouch` is true, and `applyDependencyEdit` keeps skipping cleanup for mirror
> touches. Hand-typed links stay until `bob task reconcile` consumes them.

Out of scope:

- Capture (it already consumes).
- `bob task reconcile` (unchanged; it stays the backstop).
- Ctrl+Shift+Enter and `^^` (PDG1/PDG27 unchanged).
- Project conversion, reversal, and Ctrl+Shift+M, which keep their own cleanup.
- Schedule writes (they already prune).

## Implementation (bob-plugins linked repo, `plugins/bob-navigation-hotkeys/src/`)

Open the repo with `sase repo open bob-plugins -r "<reason>"`. Read its `AGENTS.md`,
edit only `src/` fragments (each stays ≤ 1000 lines), and regenerate `main.js` with
`npm run build`.

1. **Planner target (`080-dependency-identity.js`).** In `planDependencyEdit`, add
   `summary.pomodoroTargets`. For a non-`^prj` dependent it is
   `[{path: parentPath, blockId}]` when `addedOpen.length > 0`, the final `parentText`
   is `[?]`, the original parent line has a trailing block id, and (per `hand_edits`)
   not `mirrorTouch`. Otherwise it is an empty frozen array. `^prj` keeps using
   `summary.project.pomodoroTargets`.
2. **Shared seam (`216-project-dependency-stage.js`).**
   - Add `dependencyPlanPomodoroTargets(plan)`: the deduplicated union (`path\0blockId`)
     of `summary.pomodoroTargets` and `summary.project.pomodoroTargets`.
   - Make `prepareProjectPomodoroCleanup` read its targets through it. Callers that
     append subtree targets to `summary.project.pomodoroTargets` keep working.
   - Add a target-list variant (for example
     `prepareDependencyPomodoroCleanup(readSnapshot, targets, parentPath, beforeContent, nextContent, today)`)
     for the counted writers, which plan several sources on one working copy. Have the
     plan-based function delegate to it.
   - Add `dependencyPomodoroNoticeSuffix(removed, failed)`.
   - Make the non-project branch of `projectPomodoroNotice` (or a new
     `dependencyPomodoroNotice`) return `plan.notice` + suffix when removed > 0 or the
     cleanup failed, instead of `null`.
   - Export new helpers in `700-exports.js` for tests.
3. **`applyDependencyEdit` (`580`).** The cleanup gate becomes "targets from the seam"
   (still `!mirrorTouch` unless `hand_edits` is accepted). Make sure the
   ordinary-dependent result carries the suffixed notice. This one change covers vault
   single rows and vault batches (`440` `commitVaultRefs` shows `outcome.notice`), Task
   Link mode, and the api.
4. **Same-note marked batch (`430` `executeDependencyBatchWithoutInboxRoute`).** It
   already calls the prepare/finish pair. Run its notice through the generalized helper
   so ordinary dependents get the suffix.
5. **Same-note single select (`600` `setLocalTaskDependencyLink`).** Keep its
   synchronous boolean contract: `430` `confirmSingleBlockId` and sync tests depend on
   it.
   - Same-file fold: if the dependent's note is today's daily note, resolve the daily
     path synchronously (`scheduledRecoveryDailyPaths` +
     `createScheduledRecoveryNoteIndex` over `vault.getMarkdownFiles()`). Apply
     `planDeferredPomodoroLinkCleanup` to `plan.nextContent` before the one transaction.
   - Cross-file: after the commit, run the follow-up in a guarded async continuation, in
     the same style as the existing subtree continuation in that method. Generalize
     `runSubtreePomodoroCleanup` (`218`) into a shared runner, or add a sibling runner.
   - Show one combined notice. When a cross-file cleanup is pending, defer the main
     notice to the continuation. With `showNotice: false`, show only the cleanup result,
     and only if removed > 0 or it failed.
6. **Counted writers.**
   - `planCountedLocalTaskDependency` (`270`) returns `pomodoroTargets`, the union of
     `dependencyPlanPomodoroTargets(parentPlan)` over its sources.
   - `applyCountedLocalTaskDependency` (`570`) prepares before its transaction (folding
     the same-file case) and finishes after the commit. Leave the existing subtree
     cleanup in place, then add the suffix.
   - `applyVaultCountedDependencyRefWithoutInboxRoute` (`420`) collects the same union
     per source plan, using identities, not line numbers, because sources shift on the
     working copy. Prepare before commit, finish after, and add the suffix.
   - Counted `^prj` sources therefore also consume their project targets. This closes
     the existing gap with §6.6 step 3.
7. **Release bookkeeping.**
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.37.0 to 2.38.0, and add
     one sentence to its description: "Adding an open prerequisite from Ctrl+Shift+P
     also removes the newly Blocked task's live links from today's open Pomodoros,
     deleting an emptied entry."
   - Update the README table row: the version, and the phrase "adding an open
     prerequisite marking the dependent Blocked and raising the target to its lane". It
     should now also say the gesture removes the dependent's live links from today's
     open Pomodoros, together with any emptied entry.
   - Run `npm run build`, `npm test`, `npm run validate`, then `bob plugins sync`.

## Tests (bob-plugins `scripts/`)

Add `scripts/test-navigation-dependency-link-consumption.cjs` and append it to the
`test` list in `package.json`. Model it on:

- `test-navigation-dependencies-writer.cjs` (`stubPlugin`, `TransactionEditor`,
  `cachedRead` counting);
- `test-navigation-project-dependency-stage.cjs` ("writer: Pomodoro links of `^prj` …
  consumed with emptied entries");
- `test-navigation-project-subtree-gestures.cjs` (dated daily path helper).

Cover:

1. **Planner.**
   - An ordinary dependent with a block id plus an open add yields one target.
   - Adding only a closed target yields none.
   - No block id yields none.
   - A dependent already `[?]` with an open add yields a target.
   - A removal yields none.
   - `mirrorTouch` follows `hand_edits`.
   - For `^prj`, project targets are unchanged and the union is deduplicated.
2. **`applyDependencyEdit` cross-file**, with today's daily note in the stub vault and
   also as an open buffer:
   - a link-only bullet and its subtree are removed;
   - when it was the entry's last child, the open entry is deleted;
   - an entry with another child stays;
   - closed-entry and struck links are byte-identical;
   - a prose bullet keeps its text;
   - the notice ends with `· removed 1 Pomodoro link`;
   - a daily preimage change gives `; Pomodoro links not removed` and the dependent's
     write stands.
3. **Same-file.** A dependent living in today's daily note gets the dependency and the
   link/entry removal in one undo group.
4. **`setLocalTaskDependencyLink`.**
   - It still returns a boolean synchronously.
   - The same-file fold is a single transaction.
   - The cross-file continuation removes the link and shows one combined notice.
5. **Other writers.**
   - The marked same-note batch, counted same-note, and counted vault writers each
     consume every newly Blocked source.
   - A counted `^prj` source consumes its project targets.
6. **Reads.**
   - With a daily note present, a dependent without a block id does zero `cachedRead`s.
   - The existing zero-read tests pass unchanged.

## Docs (bob-cli, this repo)

- `docs/task-dependencies.md`:
  - §5, "Adding a dependency": an add that leaves the dependent `[?]` also consumes the
    dependent's own live links in today's open Pomodoros, together with any emptied
    entry.
  - §6.6 step 3: generalize "Consume Pomodoro links after a `^prj` block" to every newly
    Blocked dependent, plus `^prj` residents. Name the same-file fold and the follow-up
    write, and say counted writers are included.
  - §6.7: add the example notice from Behavior 7.
  - §13.15: add PDG28. A Ctrl+Shift+P Depends on add (single, marked, counted, or Task
    Link mode) of an open target to an ordinary task that has a live link in today's
    open Pomodoro leaves the task `[?]`. Its live links are removed and an emptied open
    entry is deleted. Closed entries are untouched. Commitment is transferred, and the
    notice gets `· removed N Pomodoro link(s)`.
  - §13.15: add PDG29. When the dependent lives in today's daily note, the same effects
    land in one undo group. A stale daily note leaves the dependent write in place, adds
    `; Pomodoro links not removed` to the notice, and reconcile repairs it.
  - If `hand_edits` is accepted, extend §6.6's closing paragraph on hand edits to say
    the mirror also consumes.
- `docs/projects.md` Task Card table, row `b`: say the dependent is unlinked from
  today's open Pomodoros once it becomes Blocked. On `^prj` the residents are unlinked
  too, as the row already says.

No SASE memory note becomes false. The `projects-block-from-prj-line` decision already
states that Blocked tasks keep no live Task Links, and this change enforces that sooner.

## Verification

- bob-plugins: `npm run build:check`, `npm test`, `npm run validate` all pass. Then run
  `bob plugins sync` to deploy to the vault.
- bob-cli: `just check` still passes. The change here is docs only.
- Manual check in Obsidian:
  - Link a Next task into a planned Pomodoro that has no other children, then use
    Ctrl+Shift+P → `b` and pick an open task. The task becomes `[?]`, the link and the
    emptied Pomodoro disappear, and the notice reads
    `… · Blocked · removed 1 Pomodoro link`.
  - Repeat with a second child bullet under the Pomodoro: the entry stays.
  - Repeat from the Task Link bullet itself (Task Link mode).
