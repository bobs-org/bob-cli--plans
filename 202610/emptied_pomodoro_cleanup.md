---
tier: epic
title: Delete Pomodoros emptied by link-removing keymaps
goal: 'Every Obsidian keymap that removes a Task Link from a Pomodoro entry also deletes
  that entry, in the same write, when the removal leaves it empty.

  '
decisions:
  emptiness:
    ask: When does removing a Pomodoro's last Task Link delete the entry?
    choices:
      childless: Only when no child bullet of any kind remains; prose note bullets
        keep the entry
      linkless: When no Task Link remains; leftover prose note bullets are deleted
        with the entry
    default: childless
    why: Same result for link-only entries; matches bob task reconcile and never discards
      notes
    answer: childless
  closed_entries:
    ask: Should a gesture that empties a closed (past) Pomodoro delete that entry
      too?
    choices:
      delete: Delete it, timespan and all, as reconcile already does for childless
        closed entries
      keep: Delete only open entries; an emptied closed entry stays until reconcile
        removes it
    default: delete
    why: Reconcile already deletes childless closed entries in today's note; keeping
      them only delays that
    answer: delete
  notices:
    ask: Should gesture notices report each deleted Pomodoro?
    choices:
      silent: Notices unchanged; the deletion shows in the note and undoes with the
        gesture
      report: Add "· deleted N empty Pomodoro(s)" wherever a notice already reports
        removed links
    default: silent
    why: Keeps the change to the write paths; about a dozen notice builders and chip
      regexes stay as is
    answer: silent
  decision_record:
    ask: Add a decisions-web record making emptied-Pomodoro deletion a standing rule
      for new keymaps?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: nav
  title: bob-navigation-hotkeys deletes emptied entries on every cleanup
  depends_on: []
  size: medium
  description: 'nav: make planDeferredPomodoroLinkCleanup always delete emptied open
    entries, retire the removeEmptiedEntries flag, align its dedicated-bullet check,
    and cover Alt+N release, Task Card cancel, single and Task Link schedule/priority,
    and link-roll paths with tests.'
- id: bip
  title: block-id-prompt deletes emptied entries on unlink and Task Link removal
  depends_on: []
  size: medium
  description: 'bip: add an emptied-entry pass to the block-id-prompt cleanup planner
    and the Ctrl+Shift+Enter selected-link deletion, judged over all of a note''s
    edits, and align the dedicated-bullet check.'
- id: cycler
  title: task-status-cycler take-back and deferred-only close
  depends_on: []
  size: small
  description: 'cycler: delete every entry the same-day successor take-back empties,
    and (per closed_entries) the closed entry a deferred-only Pomodoro close empties.'
- id: docs
  title: bob-cli docs, optional decision record, and full deploy
  depends_on:
  - nav
  - bip
  - cycler
  size: small
  description: 'docs: document the plugin-side emptied-entry rule in bob-cli docs,
    write the decision record if accepted, then run the full plugin suite and deploy
    all plugins.'
proposed_by: bbugyi200.athena.10m
decided_by: auto
create_time: 2026-10-16 07:28:57
status: wip
bead_id: bob-cli-88
---

- **PROMPT:** [prompts/202610/emptied_pomodoro_cleanup.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/emptied_pomodoro_cleanup.md)
- **BEAD:** [bob-cli-88](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-88/README.md)

# Plan: Delete Pomodoros emptied by link-removing keymaps

## Context

A Pomodoro entry is a column-0 checkbox line under `## Pomodoros` in a daily note. Its
child bullets hold Task Links (`[[note#^id]]`, optionally `!`-embedded, `~~`-struck,
`🍅 `-marked, checkbox-prefixed, or suffixed with the move-only ` #`). Many Obsidian
keymaps remove Task Links from these entries. Only some of them then delete the entry
they leave empty, so planned Pomodoros with no children are left behind until the next
`bob task reconcile` run removes them.

What already works (do not change it):

- bob-cli reconcile: `plan_empty_pomodoro_removals`
  (`src/native/task_status_hooks/pomodoro.rs`) deletes every childless entry, open or
  closed, in today's daily note. Its rule is in `docs/task-status-hooks.md` § Empty
  Pomodoros. Capture deletes only the entries it empties (PDL9 in `docs/capture.md`).
- bob-navigation-hotkeys `planDeferredPomodoroLinkCleanup`
  (`src/160-pomodoro-reorder.js`) deletes emptied open entries, but only when a caller
  passes `removeEmptiedEntries: true`. Fifteen call sites pass it: Depends-On adds,
  counted property/priority/roll writes, `^prj` schedule, project conversion, task move
  into a blocked project, and sub-project cascades.
- Pomodoro bullet move, whole-entry move, and merge (`Ctrl+Shift+M`, nav `150`/`160`)
  already delete the emptied source entry.

Audit of the gaps. Paths are relative to the bob-plugins repo; open it with
`/sase_repo`.

| Plugin | Gesture                                                                                              | Where                                                                       | Gap                                             |
| ------ | ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------- |
| nav    | Alt+N release on task lines (bare/counted)                                                           | `510` `toggleTaskLaneOnTasksWrite` (2 calls)                                | no flag                                         |
| nav    | Alt+N release on Task Links                                                                          | `520` `toggleTaskLaneOnLinks`                                               | no flag                                         |
| nav    | Task Card lane row (Alt+N) release, task and link branches                                           | `540` `applyLaneToggleFromPicker` (3 calls)                                 | no flag                                         |
| nav    | Task Card Cancel `x` (bare/counted), decay-card Drop, single Ctrl+Enter recommended cancel           | `540` `applyTaskCancelFromEditor` (2 calls)                                 | no flag                                         |
| nav    | Task Card Cancel on Task Link(s)                                                                     | `550` `applyTaskCancelFromLinkPicker`                                       | no flag                                         |
| nav    | Single-task Ctrl+Shift+P scheduled / priority roll, decay "Less often", single Ctrl+Enter roll/decay | `219` `setInlineBulletPropertyValues` (2 calls)                             | no flag                                         |
| nav    | Task Link Ctrl+Shift+P scheduled/priority/delete                                                     | `510` `commitLinkPickerPlans`                                               | no flag                                         |
| nav    | Ctrl+Enter recommended roll/decay/cancel on a Task Link session                                      | `570` `applyLinkRecommendedRoll`                                            | no flag                                         |
| bip    | Ctrl+Shift+Enter unlink (incl. the In Progress Work-summary path)                                    | `120` `applyPomodoroTaskUnlink` → `060` `planAllOpenPomodoroLinkCleanup`    | no emptied-entry logic at all                   |
| bip    | Ctrl+Shift+Enter on a selected Task Link                                                             | `130` `applyTaskLinkOpen` → `060` `planTaskLinkDeletion` + all-open cleanup | none                                            |
| bip    | Ctrl+Shift+Enter link: duplicate removal from other open entries                                     | `075` `planExplicitPomodoroLinkInsertion`                                   | none                                            |
| bip    | `^^` completion: future duplicate cleanup                                                            | `140` `completeTaskSourceLink` → `planFuturePomodoroLinkCleanup`            | none                                            |
| cycler | Ctrl+Enter same-day reopen: successor take-back                                                      | `136` `takeBackSuccessorLinks`                                              | deletes only entries the successor pass created |
| cycler | Ctrl+Enter Pomodoro close whose children are all `#` move-only links                                 | `060` `buildPomodoroCompletionPlan`                                         | leaves the closed entry childless               |

There is a second defect. Both cleanup planners' dedicated-bullet checks (nav
`isDedicatedPomodoroLinkLine` in `140`, bip `isDedicatedLinkBullet` in `060`) reject a
checkbox prefix (`- [ ] [[…]]`) and the trailing move-only ` #`. bip's check also
rejects the `🍅 ` run. Those bullets lose only the token and leave stubs such as
`- [ ]`, `- #`, or `- 🍅`. A stub still counts as a child, so the entry never looks
empty. bip's `isDedicatedTaskLinkBullet` (`DEDICATED_TASK_LINK_PREFIX_RE` /
`..._SUFFIX_RE` in `010-core.js`) already defines the accepted decorations, and nav's
promotion expansion (`120`) handles the same set.

No removal paths exist in bob-ledger-tools, bob-project-tasks, or bob-vim-surround. The
audit confirmed this, so they are out of scope.

## The emptied-entry rule (shared by every phase)

An entry is **emptied** by a gesture when both of these hold:

1. The gesture's edits to that note remove at least one line or Task Link token inside
   the entry's block.
2. After all of the gesture's edits to that note are applied, the block holds no child.

Every emptied entry is deleted in the same write as the link removal: the same editor
transaction or undo group, or the same `vault.process` / `vault.modify` content. Its
entry line and its whole block are deleted, including indented continuation text and
fenced lines, as reconcile deletes them. Entries that were already childless, such as
placeholders the user just planned, are never touched.

> [!decision] emptiness = childless "No child" means no list-item line remains in the
> block. Fenced example bullets do not count. Prose bullets keep the entry, exactly like
> reconcile's empty-Pomodoro rule. nav's existing `remaining` count of
> `PROJECT_LIST_ITEM_RE` lines already implements this.

> [!decision] emptiness = linkless "No child" means no `[[…#^…]]` Task Link, live or
> struck, remains on a list-item line of the block. Depends-On and BLOCKS lines never
> count. The leftover prose bullets are deleted with the entry. Implement this as one
> predicate per plugin, so the childless/linkless switch lives in one function.

Which entries are eligible:

- Cleanups of "today's open Pomodoros" only ever touch open entries.
- Bip's selected-link deletion can hit an entry of any status, in any note that has a
  `## Pomodoros` section.
- Cycler's deferred-only close empties a just-closed entry.

> [!decision] closed_entries = delete Delete an emptied entry whatever its status. This
> covers bip's selected-link deletion under a closed entry, on today or another day, and
> cycler's deferred-only close.

> [!decision] closed_entries = keep Only open entries (`[ ]`, `[/]`, and other
> non-`x`/`X`/`-` statuses) are deleted. Bip skips closed owners. Cycler's close keeps
> today's behaviour.

Write mechanics that every phase keeps:

- **Removal counts stay link counts.** `removedLinkCount` / `removedCount` /
  `removedBulletCount` keep counting links or bullets, never entries.
  `pomodoroTaskLinkPresence` depends on `removedCount`. Report entries separately as
  `removedEntryCount`. Today nav counts entry candidates in `removedBulletCount`; fix
  that.
- **Seams and line endings.** Deleting a block never leaves a doubled blank seam or a
  dangling trailing newline at EOF. Follow nav's EOF handling in
  `planDeferredPomodoroLinkCleanup` and `removePomodoroBulletRanges` in `150`. CRLF and
  the final-newline state are preserved.
- **Cursors.** Any reported `removedLineRanges` include the entry deletions, so existing
  cursor math still works. A cursor that was inside a deleted entry lands on the line
  now at the entry's former start, clamped to the last line.
- **Undo.** A gesture that was one undo step stays one undo step.

> [!decision] notices = report Wherever a notice or chip already reports removed
> Pomodoro links, append `· deleted N empty Pomodoro(s)` when `removedEntryCount > 0`.
> In nav, the chip builders in `270` (`getPriorityNoticeOutcomeParts`, anchored
> tone/text regexes) need their own part plus matching rules; never append to the
> anchored `removed N Pomodoro link(s)` text. bip puts the suffix before the plan-budget
> meter.

Out of scope: hand edits such as Vim `dd` (reconcile repairs those), striking or
`🍅`-marking links (not deletions), the `=NAME` split that deliberately preserves its
source entry, nav `218`'s existing same-file cascade limitation, and bob-cli / Bob Mac
Capture, which already follow PDL9.

## Phase nav: bob-navigation-hotkeys

Work in `plugins/bob-navigation-hotkeys/src/`.

1. **Make emptied-entry deletion unconditional.** In `planDeferredPomodoroLinkCleanup`
   (`160`), always delete emptied entries. Remove the `removeEmptiedEntries` option and
   every `removeEmptiedEntries: true` argument (15 call sites). No caller opts out. The
   13 unflagged call sites in the audit table then get the rule with no per-site change.
   Re-read each one anyway: confirm its cursor math uses `removedLineRanges` and its
   folded same-file plans (`520`'s `folded` path, `510`'s `finalContent` path) still
   apply as one transaction.
2. **Align `isDedicatedPomodoroLinkLine` (`140`).** It must accept the decorations bip's
   `DEDICATED_TASK_LINK_PREFIX_RE` / `SUFFIX_RE` accept: an optional checkbox, a `🍅 `
   run, `~~…~~`, `!`, and a trailing ` #`. A single-link dedicated bullet in those forms
   must then be removed whole. Struck links stay non-candidates.
3. **Apply the emptiness predicate** from the shared rule, and stop counting entry
   candidates in `removedBulletCount`.
4. **Tests.** Extend existing files: add cases to files already listed in the root
   `package.json` test script, and add no new test files.
   - Update the cases that assert an emptied entry survives:
     `test-navigation-hotkeys-cancel-task.cjs` (same-file cancel ~552, link cancel ~613)
     and `test-navigation-hotkeys-runtime-prune.cjs` (~132, ~354, ~562).
   - Add one case per gesture family in the table: Alt+N release on a task (same-file
     and cross-file), Alt+N release on a Task Link, the Task Card lane row, Task Card
     Cancel (editor and link), single-task scheduled, Task Link Ctrl+Shift+P, and the
     Task Link recommended roll.
   - Add cases proving that a sibling prose bullet keeps the entry (or, under
     `linkless`, goes with it), that already-childless placeholders and closed entries
     are untouched, that EOF/CRLF are handled, that a `- [ ] [[…]] #` bullet leaves no
     stub, and that the gesture stays one undo group.
5. **If `notices = report`:** thread `removedEntryCount` into the notice builders that
   report removed links. These are `216` `dependencyPomodoroNoticeSuffix`, `270`, `230`
   `buildLaneToggleNotice`, `280` `buildCancelNoticeModel`, and the inline suffixes in
   `219`/`550`/`510`. `216`'s helpers currently drop the entry count, so return it.
6. **Keep each hand-edited fragment at or below 1000 lines.** `570` is at 998, `280` at
   974, and `218` at 989. Removing flag lines helps. Split a fragment through
   `src/fragments.json` if needed.
7. **Ship it.**
   - Run `npm run build` and `npm test`.
   - Bump the minor version in `manifest.json` and in the README version column, and add
     a short clause to the README row: every Pomodoro link cleanup deletes the entries
     it empties.
   - Deploy only this plugin with
     `bob plugins sync --repo <opened checkout> -p bob-navigation-hotkeys`.

## Phase bip: block-id-prompt

Work in `plugins/block-id-prompt/src/`. Put new logic in `060` (609 lines), because
`075` (978) and `120` (961) have no room.

1. **Add an emptied-entry pass to `060`.** Write a pure helper, for example
   `planEmptiedPomodoroEntryEdits(content, edits, { statuses })`.
   - Given a note snapshot and the link-removal edits a gesture will apply to it, it
     returns block-deletion edits for every emptied entry, per the shared rule.
   - It reuses `findPomodorosSectionRange`, `isPomodoroEntryLine`,
     `pomodoroEntryEndLine`, and `computeFencedLineFlags`.
   - Merge its output with `mergeCoveringEdits`, so an entry deletion absorbs the link
     edits inside it.
2. **Use it inside `planPomodoroLinkCleanupForRanges`, for open ranges only.** This
   reaches unlink (`120`), link duplicate removal (`075`, both the existing-target and
   new-target branches), `^^` future cleanup (`140`), and
   `planAllOpenPomodoroLinkCleanup`. Keep `removedCount` as the link count, and add
   `removedEntryCount` and its edits.
   - A test asserts the empty return equals `{edits: [], removedCount: 0}`
     (`test-block-id-prompt-pomodoro-context.cjs` ~135, ~223). Either keep that empty
     shape or update those tests deliberately.
   - In `075`, an insertion into the chosen entry can sit at the same offset where an
     emptied neighbouring entry's deletion starts. Make sure the inserted link lands in
     the chosen entry, and add a test for it.
3. **Ctrl+Shift+Enter on a selected Task Link (`130`).** After the per-file groups are
   built, run the emptied-entry pass over each group's merged `linkEdits`, before
   `validateNonOverlappingEdits`. When the selected link sits in today's daily note,
   emptiness is then judged over both the selected deletion and the all-open cleanup.
   The selected link's owner is eligible according to `closed_entries`. Today's cleanup
   stays open-only.
4. **Align `isDedicatedLinkBullet`** with `isDedicatedTaskLinkBullet`'s accepted
   decorations: checkbox, `🍅 ` run, `~~`, `!`, and a trailing ` #`. A single-link
   dedicated bullet must then be removed whole.
5. **Tests.** Extend existing files listed in `package.json`, and add no new files.
   - Update the cases that currently keep emptied entries:
     `test-block-id-prompt-task-link-deletion.cjs` (~216, ~264, ~305, ~329),
     `test-block-id-prompt-pomodoro-unlink-runtime.cjs` (~111, ~160, ~212, ~256),
     `test-block-id-prompt-pomodoro-insertion.cjs` (~169, ~387, ~542), and
     `test-block-id-prompt-pomodoro-targets.cjs` (~418).
   - Add cases for: same-note unlink (one write), cross-note unlink (daily-note write),
     and the In Progress Work-summary unlink; the selected link in an open entry of
     today's note, merged with cleanup; the selected link in a closed entry and in
     another day's note, per `closed_entries`; a sibling prose bullet, per `emptiness`;
     untouched pre-existing placeholders; EOF/CRLF; and cursor clamping.
   - The harness loads the generated `main.js`, so run `npm run build` before testing.
6. **If `notices = report`:** add the suffix to the unlink, selected-link, and link
   notices in `130`, before the plan-budget meter.
7. **Ship it.** Run `npm run build` and `npm test`. Bump the minor version in
   `manifest.json` and in the README version column, and add a README-row clause. Then
   run `bob plugins sync --repo <opened checkout> -p block-id-prompt`.

## Phase cycler: task-status-cycler

Work in `plugins/task-status-cycler/src/`.

1. **Successor take-back (`136` `takeBackSuccessorLinks`).**
   - Today only entries the pass created (`link.entry_created`) are deleted. Change it
     to delete every open entry the take-back empties: track the `entry_headline` of
     every removed link, then delete each such headline whose block has no remaining
     child. Do this in both the editor path and the `vault.process` path.
   - This also covers an existing entry whose lone `-` stub the placement replaced
     (`087`). After take-back it has no child, so it is deleted rather than left bare.
   - Keep the exact-match safety checks in `findSuccessorReceiptLine`.
2. **Deferred-only close (`060` `buildPomodoroCompletionPlan`).**

   > [!decision] closed_entries = delete When every child of the closing entry is a
   > move-only `#` link that the plan removes, so no list-item child would remain,
   > remove the entry line and its block instead of rewriting it to `[x]`. Still insert
   > the carried placeholder at the same point. Fix up `cursorTargetLine` and the line
   > offsets of the later edits, and keep the close as one editor edit sequence (`180`
   > `removeEditorLine` / the plan applier).

   > [!decision] closed_entries = keep Leave `buildPomodoroCompletionPlan` unchanged.

3. **Fragment headroom.** `060` (910 lines) and `180` (930) have little room. Put
   helpers in `136` or `070` if needed.
4. **Tests.** Extend `scripts/test-task-status-cycler-successor-polish.cjs` (take-back
   from a pre-existing entry, from a stub-replaced entry, and from an entry that keeps
   another child) and `scripts/test-task-status-cycler-pomodoro-completion.cjs`
   (deferred-only close, and a close where a copyable link remains). Cover both the
   editor and `vault.process` take-back paths.
5. **If `notices = report`:** append the suffix to the take-back and close notices.
6. **Ship it.** Run `npm run build` and `npm test`, bump the minor version (manifest and
   README), and add a README-row clause. Then run
   `bob plugins sync --repo <opened checkout> -p task-status-cycler`.

## Phase docs: bob-cli docs, decision record, full deploy

1. **bob-cli docs.**
   - In `docs/task-status-hooks.md` § Empty Pomodoros, add a short paragraph. Bob's
     Obsidian gestures apply the same rule immediately to the entries they empty, within
     the eligible statuses, while reconcile still removes every childless entry.
   - In `docs/task-dependencies.md` §6.6 step 3, reword the `removeEmptiedEntries`
     mention, since the option no longer exists. PDG28's "closed entries are untouched"
     stays true.
2. **Decision record, only if the decision is accepted.**
   > [!decision] decision_record Follow `/sase_memory_write`.
   >
   > - Add a new strand to the `decisions` memory web, slug
   >   `link-removal-deletes-emptied-pomodoros`, titled "Link-Removing Gestures Delete
   >   The Pomodoros They Empty", in the format of the existing records (Applies to /
   >   Claim / Why / Rejected alternatives / Evidence / Cost / Reopens when).
   > - The claim is the shared rule, with the accepted `emptiness` and `closed_entries`
   >   answers.
   > - The evidence is this plan, the three plugin versions, and the docs above.
   > - Run `sase memory init`.
3. **Verify the merged tree.** In bob-plugins, run `npm run validate` and `npm test`.
   Then run a full `bob plugins sync --repo <opened checkout>` and confirm
   `bob plugins list` shows every plugin synced.

## Verification

- `npm test` passes in bob-plugins.
- bob-cli gets docs-only changes and no Rust changes, so `just check` stays green.
- **Manual smoke** in the vault after sync. Plan a `- [ ] () — SMOKE` entry holding one
  Task Link, then confirm each of these deletes the entry, as one undo step:
  - Ctrl+Shift+Enter unlink on the task.
  - Ctrl+Shift+Enter on the link itself.
  - Alt+N release.
  - Task Card Cancel.
  - A future Ctrl+Shift+P date.
- An entry with a second, prose child survives each gesture (under `childless`).
