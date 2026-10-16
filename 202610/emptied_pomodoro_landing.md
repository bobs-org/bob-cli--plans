---
tier: tale
title: Repair emptied-Pomodoro edge cases and land bob-cli-88
goal:
  The cleanup contract is correct across full entry blocks, newline and cursor
  semantics, the intended plugins are deployed, and bob-cli-88 is closed.
size: medium
proposed_by: bbugyi200.athena.bob-cli-88.land
bead: bob-cli-88
status: done
---

- **PARENT:**
  [202610/emptied_pomodoro_cleanup.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/emptied_pomodoro_cleanup.md)
- **BEAD:**
  [bob-cli-88](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-88/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-88.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-88.land.md)
- **COMMITS:**
  - [e90c260](https://github.com/bobs-org/bob-plugins/commit/e90c2603382af8bea8563ef380c6b3db08a24670)
    — fix(block-id-prompt,task-status-cycler): delete whole emptied Pomodoro blocks with
    EOF and cursor mapping

# Finish emptied-Pomodoro cleanup and land bob-cli-88

## Scope and settled decisions

Finish the remaining correctness and deployment work for epic `bob-cli-88`, then close
that epic in this same coding turn. This is a bounded repair tale, with no land agent
after it. Do not wait for this tale's own commit, SHA, push, or CI result: the host
commits after the turn ends. Closeout precedes that host commit.

Read `sase bead read bob-cli-88 -r "Need landing evidence and follow-up outcomes"` and
the four children `bob-cli-88.1` through `.4` with audited reads. Read
`plan:202610/emptied_pomodoro_cleanup.md` through `sase artifact read`. The original
PLAN file is `202610/emptied_pomodoro_cleanup.md` in the plans sidecar; resolve its
current path through `sase bead read` and open that repo through `/sase_repo` before
editing. This tale plans only the remaining work, not the completed gesture wiring.

Open `bob-plugins` through `/sase_repo`; all `plugins/` and `scripts/test-*.cjs` paths
below are relative to that returned checkout. Read its `AGENTS.md`. Edit source
fragments, regenerate `main.js` with `npm run build`, and keep every hand-edited
fragment within 1000 lines. Extend existing test files.

The epic's decisions are final: **childless**, **delete eligible closed entries**, and
**silent notices**. An unfenced list-item child anywhere in the entry's block keeps it,
including a prose child after a blank or continuation line. Fenced example bullets do
not keep it. Delete the whole emptied block, including its continuation and fenced text.
Preserve already-childless placeholders, eligible status gates, link counts, newline
convention and final-newline state, cursor mapping, and one undo group per gesture.
Today's open cleanups stay open-only; selected-link deletion may delete a closed owner,
and deferred-only completion may delete its just-closed owner. The optional standing
decision is separately tracked by `bob-cli-8d`.

## What landing already verified

- Every phase and every note was read. The epic initially had no notes. The reopened nav
  phase's uncertainty (`.1` note 3) is resolved by its final implementation commit and
  passing relevant tests, not by assuming the first worker completed it.
- Plugin commits: `f49f9c8` (cycler 1.42.0), `5205c3a` (Block ID Prompt 1.33.0),
  `86cebc9` (navigation 2.41.0). Main-repo docs commit: `298ebdf`. Source and commit
  review confirms unconditional nav cleanup, retired `removeEmptiedEntries`, dedicated
  decorated-link recognition, BIP grouped selected-link and insertion cleanup, cycler
  take-back and deferred-close wiring, silent notices, and the documentation. No memory
  change was made.
- `npm run validate` passes on clean `86cebc9`. Full `npm test`: 2846/2852 pass; the six
  failures are the existing `bob-cli-6v` (four glyph tests), `bob-cli-6w` (Keep notice),
  and `bob-cli-7i` (reverse-conversion BLOCKS link). Audited log:
  `file:explicit:d2f93586f3a04ed5c17275e1`. All relevant existing cleanup tests pass,
  but the concrete cases below are missing from them.
- Drift since the epic began was reviewed in both repos, including changes before its
  first implementation commit. `f5517b8` adds counted task-move dispatch; it already
  routes into existing move planners and the now-unconditional cleanup, with its source
  and tests retained. Main-repo `ca88db3`, `426b07c`, `1a57e47`, `29f5f4e` add the
  read-only snapshot engine/profile; `c1b1cef` adds template persistence after capture.
  They add no plugin link-removal writer requiring this cleanup. Main HEAD and fetched
  `origin/master` were both `298ebdf` during review. Refresh this drift review before
  landing if more commits have arrived.
- `sase bead epic-symbols bob-cli-88` reports none. Neither reviewed repo currently
  defines a `symvision` recipe. Recheck both at closeout.
- `bob-cli-88` has `parent_id: null` and no ancestors as verified by audited JSON read.
  Recheck after closing; no parent needs closing in the current graph.

## 1. Fix cycler block ownership and complete-block deletion

In `plugins/task-status-cycler/src/136-plugin-successor-receipt.js`,
`successorTakeBackEmptiedEntryLine` uses `getSubBulletBlockRange` from
`060-pomodoro.js`. That old helper scans only consecutive indented list lines and stops
at a blank, continuation, or fence. The new cleanup consequently mistakes surviving
children for an empty block, and its writer removes only the headline. Reproduction
using the existing task-status-cycler harness:

```text
## Pomodoros
- [ ] () — KEEP
  - [[Tasks#^a]]

  - retain this prose
- [ ] () — NEXT
  - next note
```

Call `takeBackSuccessorLinks` with the matching `line_text`, `entry_headline`,
`entry_created: false`, and the note's `daily_path`. The current vault-process path
deletes KEEP and leaves the prose orphaned. After the link is removed, KEEP must survive
because it still owns the prose bullet. With only `  continuation only` or an indented
fenced example remaining instead of the prose bullet, the entire KEEP block must go;
currently only its headline goes.

Use a fence-aware full entry-block scan bounded by the Pomodoros section and the next
real peer/top-level content. Judge remaining unfenced list items over that whole block
after all matching receipt removals. Reuse suitable existing structural helpers or add a
narrow cleanup helper; do not globally change the old contiguous sub-bullet helper
without auditing its unrelated callers. Preserve exact receipt matching and
authored-child safety. Apply the same planned result to both editor and `vault.process`
paths, as one editor change/undo group or one vault write. Keep the returned count a
link count, and map cursors through deleted blocks.

`060-pomodoro.js::buildPomodoroCompletionPlan` has the same range defect. With a timed
headline, `  - [[Tasks#^a|A]]#`, then a blank and `  - retain this prose`, it currently
sets `deferredOnlyClose: true`, deletes the completed headline, and reattaches that
prose to the carried placeholder. The closed original must remain. When the only
remainder is continuation/fenced text, delete it with the original block and carry the
stripped Task Link into the fresh placeholder. Keep placeholder placement, cursor
position, work-log behavior, and successor relocation/placement in `135`, `137`, and
`170` consistent with the corrected postimage. Use the actual move-only spelling
`[[Tasks#^a|A]]#` in regressions (immediately adjacent `#`).

Extend `test-task-status-cycler-successor-polish.cjs` and
`test-task-status-cycler-pomodoro-completion.cjs`: both take-back write paths,
blank/continuation before surviving prose, fence-only leftovers, whole-block deletion,
adjacent entries, LF/CRLF with and without final newline, and editor transaction/cursor
behavior. Retain the existing copyable/prose/closed/placeholder and exact-receipt
safeguards. Test deferred-only successor relocation with the resulting placeholder
rather than trusting the old headline lookup.

## 2. Finish BIP newline and cursor semantics

`plugins/block-id-prompt/src/060-link-cleanup-and-deletion.js` uses
`listItemSubtreeEdit` for entry deletions, which starts at the headline offset and does
not normalize EOF. Reproduce through `planAllOpenPomodoroLinkCleanup` with
`targetPath: "Tasks.md"`, `targetBlockId: "a"`, `sourcePath: "Daily.md"`, and
`resolveTarget: () => "Tasks.md"`:

```js
const content = ["## Pomodoros", "", "- [ ] () — LAST", "  - [[Tasks#^a]]"].join("\n");
```

It currently returns `"## Pomodoros\n\n"` for both final-newline variants. Expected:
`"## Pomodoros"` without an original final newline and `"## Pomodoros\n"` with one.
Cover CRLF too. Normalize whole-entry deletion seams before composing/merging edits;
preserve inserted text at a deletion boundary. Retest `composeInsertionWithLinkCleanup`
for new/existing destinations, adjacent deletions, EOF insertion, and the preserved
child in the chosen destination.

`130-plugin-task-link-open-and-notices.js::restoreCursorClampedToContent` only clamps an
old numeric line. For selected-link deletion in:

```text
## Pomodoros
- [ ] () — REMOVE
  - [[Tasks#^ship]]
- [ ] () — KEEP
  - note
## Notes
more
```

with `Tasks.md` containing the harness's `NEXT_TASK`, the existing runtime harness
leaves `{line: 2, ch: 6}` after removal. The cursor was inside REMOVE and must land at
the entry's former start, line 1 (KEEP), then clamp its column. Map against the actual
merged source-note edits, not just the final line count. Do the equivalent for same-note
unlink in `120-plugin-pomodoro-links.js`, including a task below removed entries and
Work-summary edits that insert lines. Keep the same one-write path. Extend
`test-block-id-prompt-task-link-deletion.cjs`,
`test-block-id-prompt-pomodoro-unlink-runtime.cjs`, and insertion/context tests as
appropriate. Keep counts as link counts and notices silent.

## 3. Verify, refresh integration, and deploy the intended checkout

Increment patch versions for repaired BIP/cycler in manifests and README; regenerate all
built entrypoints. Adjust docs only if needed to express the settled contract. Run
`npm run validate`, focused affected tests, and full `npm test`. Compare any failure
against the baseline above; new failures caused by this tale must be fixed. The six
independently corroborated failures stay with their existing tasks and must not cause
new duplicate tasks or expansion of this epic.

Run `just check` in bob-cli for file-change verification. **Do not run
`just check-full`.** Use `/sase_monitor` if a long check requires a handoff, with an
explicit continuation that finishes this tale and the closeout below; do not end a
normal turn with a verification process still running or rely on a later lander.

Review new commits on both repos' base branches for conflicting cleanup changes. Retain
counted task-move dispatch and snapshot/template changes already integrated.

The docs phase's deployment was not correct: `.4` note 3 shows a bare sync from the
default checkout downgrading BIP 1.33.0 to 1.32.0, nav 2.41.0 to 2.40.1, and cycler
1.42.0 to 1.41.0. Its final claim of "up to date" compared against that wrong checkout.
Landing's explicit dry run at `86cebc9` still reports six files needing copy (three
manifests and three main.js files). After repairs, deploy with:

```text
bob plugins sync --repo <path returned by sase repo open bob-plugins>
bob plugins list --repo <same returned path>
```

Confirm every plugin is synced and an explicit dry run reports zero copies/skips. Never
use a bare sync from bob-cli for this work. Do not force over local vault edits.
Exercise the repaired gesture/undo cases through the runtime harness; if real Obsidian
smoke is available, also run the original plan's five gestures on disposable entries and
undo them. State accurately which validation ran.

## 4. Close bob-cli-88 in this turn

Follow-up triage is already complete and recorded on the epic. The close note must
retain every outcome: repeated test proposals from `.1` notes 1/5, `.2` note 2, and `.4`
note 2 were corroborated on `bob-cli-6v`, `bob-cli-6w`, and `bob-cli-7i` (also reported
in `.3` note 2). Standing-decision proposals from `.1` note 4, `.2` note 1, `.3` note 1,
and `.4` note 1 became small memory task `bob-cli-8d`. Duplicate task creation was
declined because the three reports already tracked the exact failures; no substantive
proposal was discarded. `bob-cli-7q` is related consumption policy work, not a duplicate
of the standing-decision task.

1. Reread the epic and all four children/notes; verify all repaired cases, relevant
   decisions, integration, builds, tests, and deployment. Record any new note's outcome.
   There are currently no nested descendants or parent ancestors.
2. Run `sase bead epic-symbols bob-cli-88`. Resolve every exemption keyed to the epic or
   its phases by wiring up, privatizing, adding a justified non-test pragma, or deleting
   the symbol per the Symvision policy. Re-key only if a still-open later bead genuinely
   needs it. Repeat until none remain.
3. Run
   `sase bead close bob-cli-88 --note "<concrete verification of the four phases, repaired block/newline/cursor cases, integration review, validation and deployment, known failure references, and every follow-up outcome above>"`.
   Include the correction to `.4`'s stale-checkout deployment. If symbols reject the
   close, finish them and retry. If a named phase is incomplete, finish or reopen it.
   Never force merely to succeed or to advance a successful nested landing; force is
   only for a deliberate canceled/superseded outcome with a reason.
4. Run `just symvision` if available in the relevant repositories. The reviewed trees
   lack that recipe; if still absent, explicitly record its absence and the clean
   `epic-symbols` result. Do not invent a passing Symvision result.
5. Set `status: done` in the frontmatter of the original epic's PLAN file
   `202610/emptied_pomodoro_cleanup.md` in the opened plans sidecar. Use its audited
   read for content and only the `/sase_repo` path for edits. Include this changed
   sidecar, and any changed linked/main repository, in the final declaration.
6. After closing, run `sase bead read bob-cli-88 -r "Need the parent link"`. The
   verified parent is null; finish normally if unchanged. If a parent has been attached
   meanwhile, follow the land prompt's phase-vs-plan rules: verify and close only a
   direct phase parent; for a plan parent, reread its previous landing note, all
   descendants/notes and linked plan, review post-child drift, retire symbols, close
   normally only when fully ready, run available symvision, mark its plan done, and
   repeat through complete direct plan ancestors. At the first incomplete or ambiguous
   parent, record its blocker and report it instead of forcing closure.

Finish with `/sase_final`, declaring all changed repositories. There is no need for this
turn's own commit to exist before the successful epic close.
