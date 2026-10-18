---
tier: tale
size: medium
title:
  Finish landing the Blocks… stage (bob-cli-8x) by fixing its writer and entry defects
goal: "Fix the defects that the bob-cli-8x land audit found in the reverse Blocks
  writer, the stage, and its entry points. The changes make the parity invariant hold
  for multi-task and mixed-note gestures, every notice name the right task, the stale
  check fire, Ctrl+Shift+P on a BLOCKS line open the Blocks stage, and the forward
  behavior return to byte-identical. Then close epic bob-cli-8x and task bob-cli-3p.

  "
proposed_by: bbugyi200.athena.bob-cli-8x.land
proposed_with: claude/opus
bead: bob-cli-8x
status: done
---

- **PARENT:**
  [202610/ctrl_shift_alt_b_blocks_stage.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ctrl_shift_alt_b_blocks_stage.md)
- **BEAD:**
  [bob-cli-8x](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8x/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-8x.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-8x.land.md)
- **COMMITS:**
  - [73e76c5](https://github.com/bobs-org/bob-cli--plans/commit/73e76c5a70319b0e130ea28aa8a296dfc09c9ff4)
    — docs(plans): mark the Ctrl+Shift+Alt+B Blocks stage plan done

# Finish landing the Blocks… stage (bob-cli-8x)

## Context

Epic **bob-cli-8x** (plan `plan:202610/ctrl_shift_alt_b_blocks_stage.md`) shipped
Ctrl+Shift+Alt+B, the Task Card ⇧B row, the reverse **Blocks…** stage, `api.blocks`, and
BLOCKS chip ＋ / ×. Its six phases are closed. The contract is
`docs/task-dependencies.md` §15 in bob-cli. The code is in the linked **bob-plugins**
repo:

- bob-plugins commits: `8ca6a65` writer, `e3f3329` stage, `10d12f4` entry, `254b32b`
  chips, `96a42c5` publish.
- bob-cli commits: `30bdfb8` contract, `3ec4d0d` glossary.
- Plugin versions: nav 2.45.0, ledger-tools 1.51.0.

The land audit read that code and reproduced the defects below in memory with the
`scripts/navigation-dependencies-stage-harness.cjs` stubs. All of them were caused by
the epic, so they are epic work. The simple cases already match forward byte for byte,
and `scripts/test-navigation-blocks-edit.cjs` (RBP1–RBP10, RBW1–RBW4, RBW6) passes only
because its fixtures avoid the failing layouts.

Follow-up triage is already done and recorded on bob-cli-8x. Do **not** file or +1
anything for these:

- The six full-suite `npm test` failures are pre-existing and tracked by bob-cli-6v
  (×4), bob-cli-6w, and bob-cli-7i. Expect exactly those six to fail.
- `!` on a BLOCKS line in nav is bob-cli-8z. Leave it alone.

Paths below are relative to bob-plugins unless they start with `docs/` or `sase/`. Nav
is `plugins/bob-navigation-hotkeys/src/`. Line numbers are approximate as of `96a42c5`.

## Rules

- Open bob-plugins with your `/sase_repo` skill
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`.
  - Edit `src/*.js` fragments and run `npm run build`; never hand-edit `main.js`.
  - Keep every fragment at or below 1000 lines. `590-plugin-dependency-stage.js` is
    already at 1000 and `080` is at 995, so put new helpers in a new or existing smaller
    fragment registered in `src/fragments.json`.
  - Export test helpers through `700-exports.js`.
- **Forward stays byte-identical.** The only allowed forward change is fix A2's shared
  stamp fix, which gets its own regression test.
- Tests cite vector ids in comments. If the code and a §15 vector disagree, fix the one
  that is wrong in the same change.
- Never reuse forward copy strings for reverse (§14.9's rule).
- Read the governing decisions before changing the writer:
  `sase memory read decisions:task-deps-are-depends-on-links decisions:task-lanes-are-sticky -r "<why>"`.
  No memory edits are needed or authorized.

## A. Writer correctness (`084-blocks-stage-plan.js`, `592-plugin-blocks-edit.js`)

1. **Re-locate rows after every per-T plan.**
   - **Problem.** `planBlocksEdit` (`084` ~519–560) plans each note's T's bottom-up but
     reuses each row's original `row.line`. The first `planDependencyEdit` can insert
     N's BLOCKS line when N sits above the remaining T's in the same note
     (`083-blocks-dependency-plan.js` ~178), and an emptied BLOCKS line can be deleted
     on removal. Either shifts the remaining rows.
   - **Observed.**
     - Note N, T1, T2 (N has no BLOCKS line yet): the gesture fails with `not-task`.
     - Note N, X, T1, T2: bystander X becomes `[?]` with a Depends-On line, T1 is
       stamped but not blocked, T2 is blocked but not stamped, and the gesture reports
       "Now blocks 2 tasks".
   - **Fix.** After each `planDependencyEdit`, map every not-yet-planned row and every
     planned row's final T line through the change. Use the trailing block id when there
     is one (`findTaskLineByTrailingBlockId`, `080`). Otherwise map through the plan's
     `structural` result plus the same-note BLOCKS insert or delete (see
     `mapLineThroughStructural` in `216`), or use a small line-map helper.
   - **Stamping.** `stampBlocksEditParentLine` must stamp T's post-plan line, never
     `row.line`.

2. **Shared stamp fix (forward).**
   - **Problem.** `applyDependencyEdit` (`580` ~596) stamps `parentLine`, the pre-insert
     index. When a same-note prerequisite above T gains a new BLOCKS line, the bystander
     line or the BLOCKS line gets `[fresh::]` and T does not. §1 requires T to be
     stamped.
   - **Fix.** Stamp the dependent's post-plan line in forward too. The reverse parity
     tests then compare correct bytes.
   - **Test.** Add a forward regression test: prerequisite N above T in one note, N
     without a BLOCKS line, and X between them.

3. **Mirror and committed refs use identities, not stale lines.**
   - **Problem.** `592` ~376–388 and ~406–418 read each committed T's block id from
     `write.nextContent` at the stale `entry.parentLine`. `592` ~449–452 finds N's own
     line by `source.line` before falling back to its block id.
   - **Observed.**
     - N above T1, X, T2 in a closed note: N's BLOCKS becomes `t1 • x` (forward gives
       `t1 • t2`).
     - With N in today's daily note (RBW4), the BLOCKS line lands under an unrelated
       task after the Pomodoro cleanup deletes lines above N.
   - **Fix.** Carry the final line or block id out of the planner (fix A1). Locate N by
     block id first in the post-write content.

4. **Notices name T, not N.**
   - **Problem.** `084` ~562–565 uses `plan.added[0] || plan.removed[0]`, which are the
     prerequisite descriptions, so in reverse they name N.
   - **Fix.** Use T's cleaned text (`cleanTaskDisplayText(row.text)`, cut as §7 says)
     for every notice: `Now blocks “T”`, `No longer blocks “T”`,
     `Now blocks project “T”`, and the counted form `2 tasks now block “T”`. The chip ×
     path goes through the same writer, so it is fixed too.
   - **Test.** Assert the real notice text. Do not pass a hand-built description.

5. **Mixed same-note and cross-note picks keep N's lane rise.**
   - **Problem.** When N's note also holds a T, `084` ~616–625 uses only that note's
     planned content and drops the `mergedPreparations` aimed at N's note from T's in
     other notes.
   - **Observed.** T1 Ready next to N and T2 `[/]` in another note: N stays `[ ]`
     (forward gives `[/]`), but the notice says "takes In Progress".
   - **Fix.** Apply the strongest of the cross-note preparations onto the same-note
     result at N's re-located line. Use the strongest-lane rule and never raise a `^prj`
     N.
   - **Also.** Any merged preparation for a path that is neither a source nor a T note
     must fail the plan loudly. Never drop it silently.

6. **The changed-since-open check must compare against live content.**
   - **Problem.** `425-picker-blocks-stage.js` ~364–385 passes the stage snapshot as
     both `expected` and `files`. `592` ~104–107 skips re-reading any path already in
     `files`, so the comparison at ~165–186 always matches. A changed T then lands
     partially (RBG5 and §6 step 1 are broken).
   - **Fix.**
     - `applyBlocksEdit` reads live content for every touched note (each source note,
       each T note, and today's daily note): the open buffer wins, otherwise the vault.
       It compares each against `expected` before planning.
     - The stage passes only `expected`, built from the open-time snapshot of the
       touched paths. Stop passing the whole `vaultStage.files`; that also stops the
       preflight from scanning every line of the vault (`592` ~138–164) when the pool
       was built cold.
   - **Notice.** Show the stale notice exactly once. The writer returns `notice` and
     lets its caller show it; today a stale stage commit shows "changed — reopen" twice
     and chip × shows it three times.

7. **Consume Pomodoro links only for T's that committed.**
   - **Problem.** `592` ~337–348 and ~476–481 plan the cleanup before the writes and run
     it whatever happens.
   - **Observed.** T2's note changes mid-write and T2 stays unblocked, but its link is
     removed anyway.
   - **Fix.**
     - Fold into a source write only the targets of T's committed in that same write.
     - After the dependent writes, run one cleanup for the T's that committed.
     - Report the counts from what was actually removed.
   - **Note.** A cross-note gesture whose daily note is N's note may get one extra undo
     entry; that is acceptable.

8. **Re-check guards on live content in the preflight (§6 step 1).**
   - **Problem.** With a warm Tasks cache, the stage loads only open buffers, and cached
     candidates carry no note content or `isProjectPrj` (`590` ~282–288, `300`
     ~487–518). So for T's in closed notes, the reverse guard (`302` ~425–465) silently
     skips:
     - the malformed or second Depends-On line check;
     - "inside a sub-project of this project";
     - the cycle through a `^prj` T's residents;
     - the `^prj` strip text.
   - **Fix.**
     - In `applyBlocksEdit`, after the live reads, run the swapped guards for every
       added T on the trial graph after the whole batch, reusing the `302` guard. Refuse
       the whole gesture with the guard reason as a 🧱 notice and write nothing.
     - The stage keeps disabling rows whose content it has.
     - Document in §15.5 that closed-note rows are checked when the gesture is applied.

9. **Use T's path for the daily-note flag in `^prj` removal recovery.**
   - **Problem.** `592` ~249–251 passes N's path where forward uses T's (`580` ~437). So
     when N lives in a daily note, a `^prj` T's residents that recover to Ready get no
     `[unblocked::]`.
   - **Fix.** Use T's path.

**Tests.** Add these to `scripts/test-navigation-blocks-edit.cjs`, as forward-parity
comparisons wherever forward has an equivalent:

- N above two T's in one note, with and without X between them, in an open note and in a
  closed note.
- N below two T's.
- Mixed same-note plus cross-note lanes (RBP7 variant).
- RBP1 with N truly lacking `^id`.
- RBW4 with a task after N, asserting the BLOCKS line's position.
- RBW2 asserting that the failed T's Pomodoro link is kept.
- A stage-originated stale T (RBG5 for T).
- A closed-note T with a malformed Depends-On line, and a closed-note `^prj` T whose
  sub-project contains N.
- `^prj` removal with N in a daily note.

## B. Forward regressions to revert

1. **Ctrl+Shift+P redirect** (`550-plugin-cancel-and-counted-property.js` ~312–345).
   - **Problem.** `10d12f4` made the redirect fire for any line whose owner differs
     (`entry.parentLine !== cursor.line`). `resolveDependencyStageEntry` (`300` ~668)
     returns the owner for every line inside a task block. So Ctrl+Shift+P,
     Ctrl+Shift+B, and counted P on a plain child bullet or a nested dedicated Task Link
     now act on the owner, and a nested Task Link no longer reaches the link picker.
   - **Fix.** Redirect only when `entry.lineKind` is `"depends-on"` or `"blocks"`:
     - `"blocks"`: Ctrl+Shift+P (no `initialTaskCardRow`) and Ctrl+Shift+Alt+B both open
       the owner's Blocks stage directly (`reopenBlocksStage: true`). This is the
       missing §15.2 / §14.8 / RBU7 behavior: today Ctrl+Shift+P on a BLOCKS line
       re-dispatches with `{}` and opens the Task Card.
     - `"depends-on"`: unchanged forward (`initialProperty: "dependsOn"`), except that
       Ctrl+Shift+Alt+B opens the Blocks stage.
     - Any other line: no redirect, exactly as at `3e359a6`.
   - **Tests** (in `scripts/test-navigation-dependencies-stage-entry-and-view.cjs`):
     - The current "Ctrl+Shift+P on a BLOCKS line…" test calls
       `openDependencyStageAtCursor`, the palette-only "Edit task dependencies" command.
       Change it, or add a test, to go through `openBulletPropertyPicker(editor)` with
       no options, which is the `set-bullet-property` hotkey path.
     - Add regression tests showing that Ctrl+Shift+P on a plain child bullet and on a
       nested Task Link behave as at `3e359a6`.

2. **Forward counted badge order** (`410-picker-filter-and-render.js` ~392).
   - **Problem.** The `linkState === "mixed"` branch moved ahead of the BLOCKED `🔒` and
     CURRENT `−` branches, so a forward counted mixed row in BLOCKED shows `k/n depend`
     instead of `🔒 waits on N`.
   - **Fix.** Restore the forward order. If reverse needs the earlier position, gate it
     on `this.blocksStage`.
   - **Test.** Add a forward test that pins the order.

## C. Entry and copy fixes

1. **Blockquoted N.** `taskCardBlocksAvailabilityReason` (`311-task-card-blocks.js`)
   must refuse a blockquoted task (`isBlockquotedMarkdownLine`) with
   `🧱 Blocks can't be edited inside a blockquote`. This is the reverse twin of
   forward's `⛓ Dependencies can't be edited inside a blockquote`. Add it to the RBG6
   tests.
2. **Card copy.** The Task Card unavailable reason for a closed N is shown without the
   🧱 (§3 / §15.4). The entry Notice keeps
   `🧱 A finished task can't block another task · reopen it first`.
3. **block-id-prompt twin notice.**
   - Update `TASK_BLOCKS_LINE_NOTICE` (`plugins/block-id-prompt/src/010-core.js` ~70) to
     nav's
     `🧱 BLOCKS is generated from DEPENDS ON lines · edit it with Ctrl+Shift+Alt+B`.
     §14.8 says Ctrl+Shift+Enter is "refused with the same notice".
   - Update `scripts/test-block-id-prompt-blocks-guards.cjs`.
   - Bump block-id-prompt's patch version.
4. **`removeDependent`.** When the owner has no `^id`, return `not-a-dependent`, not
   `source-id-missing` (`593-plugin-blocks-api.js` ~70). No dependent can list an owner
   with no id.
5. **Strip and subtitle copy** (`303-blocks-stage-copy.js`, `425`):
   - Pluralize correctly: `1 task waits`, not `1 tasks wait`.
   - With only removals marked, read `↵ removes N`, not `↵ blocks 0 · removes N`.
   - With marks, the strip never says `this task takes …` when N is a `^prj`; the
     single-row branch already handles this.
   - The counted subtitle drops `⇥ to mark several`, since ⇥ is off in counted sessions.
   - The BLOCKED-row count includes N: `Also waits on this (3 open)` counts after the
     add, matching §7's example.
6. **Today's links survive a pool refresh.** Keep the 🍅 today-links read when a vault
   pool refresh replaces the stage items (`425` ~237–239), rather than discarding it.
7. **Dead code.** Remove `blocksStripText` (written, never read, `425` ~198),
   `void self;` (`plugins/bob-ledger-tools/src/152-blocks-widget.js` ~111), and the
   `pendingMirrors` list if it is used only to compute `changed` (`084` ~645).
8. **Versions.** Bump nav to `2.45.1`, and block-id-prompt as above. Update the README
   plugin-table versions.

## D. Docs (bob-cli `docs/` and the bob-plugins README and manifest)

- **bob-plugins `README.md`.**
  - In the nav row, state Ctrl+Shift+Alt+B once, mention the ⇧B row and counted
    `N<Ctrl+Shift+Alt+B>`, and say that Ctrl+Shift+P on a BLOCKS line opens the owner's
    stage directly. Drop the false "through nav `api.blocks` v1" attribution: that
    keyboard path does not use the api, only the ledger chips do.
  - In the ledger row, move the BLOCKS ＋ / × clause so the Depends-On parenthetical no
    longer reads as if it applies to it.
  - Add the BLOCKS-line entries and the chip ＋ to the direct-entry list (~127–130).
- **Nav `manifest.json` description.** State Ctrl+Shift+Alt+B once, and say only the
  BLOCKS chips use `api.blocks`.
- **`docs/task-dependencies.md`.**
  - §2.4: add the Blocks stage to the Depends-On line's writers.
  - §13.7: name the Blocks stage on a `^prj` T in the cascade and resident-tag writer
    lists.
  - §14.1: "authored" becomes "stored". It now reads "the only place a dependency is
    authored" (~2753), and the glossary says stored on the dependent and edited from
    either end.
  - §9 and §15: list the full `api.blocks` refusal set that ships (`no-owning-task`,
    `picker-not-opened`, `open-failed`, `target-not-found`, `remove-failed`, the
    writer's reasons, `unexpected-result`, `api-failed`), or trim the code to the
    documented set. Choose one and make the doc and code agree.
  - §15.5: closed-note guards are checked when the gesture is applied (fix A8), and the
    blockquote refusal (fix C1).
  - §15.7: the notice names T. Fix any example still showing otherwise.
  - §15.8: document the existing fallback tooltip
    (`Generated from DEPENDS ON lines · edit on the dependent`) for frozen rows and for
    a nav without `api.blocks`. It stays as is: for a frozen row the owner is closed, so
    the dependent is the only end that can edit it.
- **`docs/freshness.md`** nav Blocks row: the BLOCKS chip ×, the palette command, and
  the line entries use the same writer and stamp T.
- **`docs/projects.md`** (~361–363): add the BLOCKS-line entries and chip ＋ to the
  direct-entry list.

## Declined audit items (recorded on bob-cli-8x; do not do these)

- **Whole strip turns warning-colored when it holds a ⚠.** Cosmetic, and still uses
  existing tokens.
- **Ledger × failure shows nav's notice plus the raw reason.** Identical to today's
  forward Depends-On ×. If it matters, it is a cross-cutting change for both chip kinds.
- **BLOCKS widget mousedown handler without `api.blocks`.** Copies the Depends-On widget
  and is harmless.
- **Second counted source note fails after the first was written.** Once fix A6 compares
  live content against `expected` before any write, only a sub-second race remains.
- **§6.1 table order, and the "0 tasks" card detail.** Cosmetic.

## Verification

1. In bob-plugins, run `npm run build`, `npm test`, and `npm run validate`.
   - `npm test` must show exactly the six known failures (bob-cli-6v ×4, bob-cli-6w,
     bob-cli-7i).
   - Every new test passes, and each test that targets a fixed defect fails on
     `96a42c5`.
2. Run `bob plugins sync`.
3. In bob-cli, run `just check` (docs only; do not run `just check-full`).
4. Run
   `rg -n "edit on the dependent|change it on the dependent|later epic \(bob-cli-3p\)"`
   across both repos. The only allowed hit is the documented §15.8 fallback tooltip.

## Final step: close the bob-cli-8x landing

Do this in the same turn after verification passes. Do not wait for this work's own
commit; the finalizer commits both repos after the turn.

1. Run `sase bead epic-symbols bob-cli-8x`. For every entry listed, resolve the symbol
   (wire it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy). The land audit found none.
2. Close the epic with:

   ```bash
   sase bead close bob-cli-8x --note "<what you verified>"
   ```

   The note must cover:
   - all six phases verified against the plan;
   - the fixes you made: writer re-location, the shared stamp fix, identity mirror,
     T-named notices, mixed-note lanes, the live stale check, committed-only
     consumption, preflight guards, the `^prj` daily flag, the Ctrl+Shift+P redirect and
     the BLOCKS-line stage, the forward badge order, blockquote, and copy;
   - the test counts, with the six known failures;
   - follow-ups: 6v / 6w / 7i corroborated and bob-cli-8z filed;
   - integration: no overlap from the seven post-start commits;
   - the declined items above.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic symbols, resolve them and close again.

3. Run `just symvision` in bob-cli if the recipe exists, and confirm it is clean.
4. In bob-cli, set `status: done` in the frontmatter of
   `sase/repos/plans/202610/ctrl_shift_alt_b_blocks_stage.md`. This is the PLAN path
   shown by `sase bead read bob-cli-8x -r "<why>"`; re-check it there.
5. Close task **bob-cli-3p** ("Reverse Blocks… stage for editing a task's dependents"):
   the epic plan states that bob-cli-8x implements it and that the landing closes it.

   ```bash
   sase bead close bob-cli-3p --note "Implemented by epic bob-cli-8x (Ctrl+Shift+Alt+B Blocks… stage, Task Card ⇧B, api.blocks, BLOCKS chip ＋/×)"
   ```

6. bob-cli-8x has no `parent_bead`, so there is no parent to close. Confirm with
   `sase bead read bob-cli-8x -r "Need the parent link"`.
