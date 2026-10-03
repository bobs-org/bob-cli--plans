---
tier: epic
title: 'Finish the task dependency landing fixes: nav regressions, mirror owner, stage
  badge, DP30 chips, Reading-view line, R9 hooks, rollout'
goal: 'Every gap and regression the bob-cli-3n.12.9 landing audit confirmed is fixed
  and pinned by a test that fails on the pre-fix source: nav recovers, refuses, and
  commits correctly on every writer path; the mirror edits the right task; the stage
  never shows "waits on 0"; chips render on DP30 and act on the right Reading-view
  row; the hooks apply R9 to label-only lines; and the fixed bob and plugins are installed
  across the fleet.'
parent_bead: bob-cli-3n.12.9
phases:
- id: nav-writer-regressions
  title: Fix the nav writer regressions and finish its missing tests
  depends_on: []
  size: medium
  description: 'nav-writer-regressions: build the recovery snapshot for field-only
    clears and gate the hand-edit clear read, load source-linked notes in the counted
    vault path, make counted remove one transaction with the blockquote notice on
    every Ctrl+D path, align nav with DP30, and add the plugin-level writer tests
    the previous phase left out.'
- id: nav-mirror-stage-fixes
  title: Fix the mirror owner lookup, the waits-on badge, and the remaining stale
    refusals
  depends_on:
  - nav-writer-regressions
  size: medium
  description: 'nav-mirror-stage-fixes: resolve the hand-edit mirror owner in baseline
    coordinates, stop the stage showing "waits on 0", route the cross-note batch and
    counted vault stale paths through refuseDependencyStale, and make the stage tests
    drive the real write and marking paths.'
- id: chips-dp30-reading
  title: Render chips on DP30, pick the right Reading-view row, and finish the DP
    tables
  depends_on: []
  size: medium
  description: 'chips-dp30-reading: drop the ledger-tools "Work Log anywhere above"
    rule so DP30 renders and the ancestor scan stays bounded, map each Reading-view
    row to its own line, add DP31 (prose-only line, malformed) to the contract and
    every recogniser, and add the DP19/DP20 rows to the cycler and block-id-prompt
    tables.'
- id: hooks-r9-split
  title: Apply R9 to label-only lines in the hooks and bring the touched files under
    size
  depends_on: []
  size: small
  description: 'hooks-r9-split: stop the hooks re-adopting a label-only Depends-On
    line (R9/DW5/DR16), pin it with an adoptable-id test, correct the DW3/DW6 comments,
    and bring reconcile.rs and dependency_lines.rs back under about 1500 lines.'
- id: rollout
  title: Reinstall bob and resync the plugins with the remaining fixes
  depends_on:
  - nav-writer-regressions
  - nav-mirror-stage-fixes
  - chips-dp30-reading
  - hooks-r9-split
  size: small
  description: 'rollout: reinstall bob and sync the plugins on athena and apollo,
    update the MacBook best effort, dry-run the hooks against the real vault and explain
    every dependency count, and record versions and what is left for Bryan.'
proposed_by: bbugyi200.athena.bob-cli-3n.12.9.land
create_time: 2026-10-03 02:54:23
status: done
bead_id: bob-cli-3n.12.9.6
---

- **PROMPT:** [prompts/202610/task_dep_links_landing_remaining.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_dep_links_landing_remaining.md)
- **PARENT:** [202610/task_dep_links_landing_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_dep_links_landing_fixes.md)
- **BEAD:** [bob-cli-3n.12.9.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3n/bob-cli-3n.12.9.6.md)

# Plan: Finish the task dependency landing fixes (bob-cli-3n.12.9 remaining work)

## Context

Epic bob-cli-3n.12.9 (`plan:202610/task_dep_links_landing_fixes.md`; read it with
`sase bead read bob-cli-3n.12.9 -r "<why>"`) fixed the gaps the bob-cli-3n.12 landing
audit found. All five of its phases closed (bob-cli `eb9d846`, `72964be`; bob-plugins
`e09b424`, `b168458`, `2bd875d`). Its land agent then audited every phase against that
plan by reading the code and running probes. The suites are green, but the audit
confirmed the regressions and gaps below. This epic fixes only those. When it lands, its
land agent resumes the bob-cli-3n.12.9 landing through `parent_bead`, so no phase here
closes bob-cli-3n.12.9, bob-cli-3n.12, or bob-cli-3n, and no phase edits their plan
files.

**Contract:** `docs/task-dependencies.md` in bob-cli is the single source of truth
(grammar, identity, R1–R10, stage, chips, api v1, DP/DW/DR/DK/DC vectors).

Line numbers below are from bob-cli `72964be` and bob-plugins `2bd875d`. They will
drift, so locate code by symbol name.

**Live usage.** 16 vault notes carry `⛓️ **DEPENDS ON:**` lines. No vault note has a
label-only Depends-On line or a blockquoted `#task`. Deployed today on athena and
apollo: nav 1.61.0, ledger-tools 1.21.0, cycler 1.22.0, block-id-prompt 1.20.0, bob at
`72964be`. They carry the bugs below until each phase deploys its fix.

## Cross-cutting rules for every phase

- **Read first:** `docs/task-dependencies.md`. Before changing behaviour that a decision
  record governs, read `decisions:task-deps-are-depends-on-links` and
  `decisions:task-lanes-are-sticky` (`sase memory read … -r "<why>"`).
- **Vectors are the contract.** A new behaviour gets a numbered vector before it gets
  code. If an implementation disagrees with a vector, fix the doc first in the same
  phase and say so in the commit.
- **Every fix gets a test that fails on the pre-fix source.** Prove it: run the new test
  against the phase's starting commit (for example a temp copy of `main.js` from
  `git show <start>:<path>`) and record pass/fail counts in the phase note. A test that
  passes on the pre-fix source does not count.
- **bob-cli:** run `just all` (fmt, clippy, test). This repo has no `check` recipe yet
  (bead bob-cli-3c tracks adding one). If `just check` exists when you run, use it
  instead. Do not run `just check-full`. Keep touched Rust files under about 1500 lines.
  Update `README.md` and `docs/*.md` in the phase that changes behaviour.
  `capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes` and
  `note_ready::tests::scan_excludes_r3_and_r7_paths` are known parallel env-race flakes
  (bead bob-cli-2e). The `completion::vault` and `completion::bash` readline flakes
  belong to epic bob-cli-3j. Rerun any of these alone before treating it as a
  regression, and do not fix them here.
- **bob-plugins** (open it with `/sase_repo`: `sase repo open bob-plugins -r "<why>"`,
  and use only the printed path):
  - `npm test` and `npm run validate` must pass. Add any new test file to the `test`
    script in `package.json`.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`, run
    from the opened path.
  - `plugins/bob-navigation-hotkeys/main.js` is about 43k lines and
    `plugins/bob-ledger-tools/main.js` is also large; use `grep -an` and read targeted
    ranges.
  - New tests must exercise the real async plugin paths (async vault and editor stubs,
    `createBulletPropertyPickerHarness` / `createLinkPickerHarness`), not only pure
    helpers and not injected shortcuts such as a prebuilt `vaultContents`.
  - `nav-writer-regressions` and `nav-mirror-stage-fixes` both edit nav and run in
    sequence. Parallel phases may both edit `package.json`, the plugins README, or
    `docs/task-dependencies.md`; when rebasing, keep both sides of single-line
    conflicts.
- **Vault:** no phase edits `~/bob` content. Plugin deploys write only
  `.obsidian/plugins/<id>/`. Never run a live `bob task-status-hooks` pass against
  `~/bob`; dry runs (`--dry-run -f json`) are fine. Reproduce in scratch vaults under
  `/tmp` with `BOB_DAY_FILE` set and backdated files (`touch -d '-1 hour' …`). The repo
  build lives in `$CARGO_TARGET_DIR/debug/bob`.
- **Follow-ups:** epic phase workers never create beads. Record `PROPOSED FOLLOW-UP:`
  notes on your own phase bead.

## Phase: nav-writer-regressions — writer regressions and missing writer tests

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`. Tests live in
`scripts/test-navigation-dependencies-writer.cjs`,
`scripts/test-navigation-hotkeys.cjs`, and `scripts/test-navigation-dependencies.cjs`.

1. **Recovery snapshot gate misses field-only clears (regression from `e09b424`).**
   `needsDependencyRecoverySnapshot` returns false when `remove` is empty and the call
   is not a mirror touch. `planDependencyEdit` also recovers when the field is cleared
   with no link removed (`fieldCleared`, the field-only Ctrl+D case). So a `[?]` task
   whose `[dependsOn::]` is cleared by field-only Ctrl+D, and which a Task Link under
   today's open Pomodoro points at, now recovers to `[ ]` instead of `[*]`. Rule: adds
   never recover; every other edit on a `[?]` parent builds the snapshot, including
   today's daily note. Test field-only Ctrl+D on a Pomodoro-linked `[?]` task through
   the plugin, with the real vault read (it must recover to `[*]`). Also test that an
   add, and a mirror touch on a non-`[?]` parent, read zero extra notes (use a vault
   stub that counts `cachedRead` calls).
2. **Hand-edit clear reads the whole vault every time.** `applyDependencyHandEditClear`
   now builds the snapshot on every clear, even for a parent that is not `[?]`. Gate it
   with the same rule. Rewrite the hand-edit clear recovery test so the plugin does its
   own vault read: no injected `vaultContents`. Assert `[*]` for a Pomodoro-linked
   dependent, and zero extra reads for a non-Blocked parent.
3. **Counted vault add refuses on unloaded cross-note links (regression from
   `e09b424`).** `applyVaultCountedDependencyRef` now loads only the target note. The
   old per-source `applyDependencyEdit` path also loaded the notes behind each source's
   existing links. When a source has an unloaded cross-note link and its field is
   missing or out of sync, the whole counted add now refuses with `target-not-found`.
   Before `e09b424` it succeeded. Load the notes behind every source's existing links
   (the same `wanted` loop `applyDependencyEdit` uses) before planning. Test that exact
   case through the plugin.
4. **Counted remove still writes per source.** `removeCountedDependency` writes one
   source at a time. A mid-run refusal (for example a blockquoted source) leaves the
   earlier sources written ("could not update the task on line 1 (in-blockquote)"). Give
   it the same shape as counted add: plan every source on one working copy (bottom-up),
   refuse before any write if any source fails to plan, commit once, and show one
   summary notice. In both counted notices, count only sources that actually changed
   ("Linked N tasks" / "Removed from N tasks"). Fix the comments near
   `applyVaultCountedDependencyRef` and `removeCountedDependency` that still describe
   per-source writes.
5. **The blockquote notice is missing on two Ctrl+D paths.** Counted Ctrl+D
   (`deleteCountedDependencyLinesAndFields`) says "Could not delete task dependencies",
   and single Ctrl+D on a quoted task with no field says "dependsOn is not set on this
   bullet". Both must refuse with `⛓ Dependencies can't be edited inside a blockquote`
   and write nothing. Test both.
6. **DP30 in nav.** The nav DP table in `scripts/test-navigation-dependencies.cjs` has
   no DP30 row. Add one: a Depends-On line that is the first direct child of a `#task`
   nested under a Work Log entry is `accept(1)` and owned by that inner task. If nav's
   reader or any gesture disagrees, fix nav to match the contract and Rust.
7. **Missing writer tests (all plugin-level, all failing on `e09b424~1` unless they pin
   a regression fixed here):**
   - Kept-link field id through `executeDependencyBatch`: line `[[#^a]] • [[Other#^x]]`
     with `Other.md` unloaded. Removing `a` must keep `Other`'s own id beside
     `[[Other#^x]]`. Replace the plugin-level half of the current test 13, which removes
     the unloaded link instead of keeping it and so passes on pre-fix code.
   - `pendingTargetLine` through `confirmSingleBlockId` and through
     `chooseTaskDependency`, each with and without an existing Depends-On line on the
     dependent. Target B gets `^b` and `[id::]`, the dependent gets the link, and
     everything lands in one undo group.
   - Counted remove: one undo group, plus refuse-before-write where the failing source
     is not the first in bottom-up order.
   - Convert the pure-helper Pomodoro-recovery and subfolder-target tests from
     `nav-writer-fix` into plugin-level async tests, then delete the pure duplicates.
8. **Finish.** Bump the manifest, update the README row, `npm test`, `npm run validate`,
   deploy nav.

## Phase: nav-mirror-stage-fixes — mirror owner, stage badge, stale refusals

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`. Runs after
`nav-writer-regressions`. Stage tests live in
`scripts/test-navigation-dependencies-stage.cjs`.

1. **Mirror owner is looked up in the wrong text (bug from `2bd875d`).** The burst maps
   the owner offset forward through later `update.changes`, so the result is a
   current-document position. `dependencyHandEditOwningLine` then looks that line up in
   the burst's baseline (pre-change) text. When a line is inserted above the owner in
   the same burst, the wrong task is picked. Repro: delete P's last-child Depends-On
   line, then insert a line under the heading within the debounce. The mirror plans
   `{kind: 'touch', owning: <next sibling S>}` instead of P's `clear-field`, and P keeps
   a stale field. Identify the owning task in baseline coordinates from the first
   update's `startState` and the first change. Map forward only to locate that same task
   in the current document, and never index the baseline with a current-document line.
   Tests through `scheduleDependencyHandEditMirrorFromUpdate` that actually run the
   mirror:
   - the repro above;
   - a first-edit deletion of P's line that is not a vim `dd` (for example selecting the
     line text and deleting it);
   - the existing vim `dd` case, kept.
2. **`🔒 waits on 0` (regression from `2bd875d`).** `planDependencyStageView` now
   accepts `openCount` 0 and looks the count up only for candidates with a `^blockId`. A
   Blocked task with no block id that waits on one open task shows `waits on 0`, and so
   does a task blocked only by a future `scheduled` date. Compute the open prerequisite
   count for every Blocked candidate from its own Depends-On line, whether or not it has
   a block id. Show `🔒 waits on N` only when N ≥ 1. A Blocked candidate with no open
   prerequisite shows `🔒 scheduled YYYY-MM-DD` when a future `scheduled` date blocks
   it, and otherwise `🔒 blocked`. State this rule in the stage section of
   `docs/task-dependencies.md` (§6), and add a stage test for each of the three labels.
3. **Remaining stale paths.** These still skip or report stale rows without the
   `changed — reopen` refusal: the cross-note `executeVaultDependencyBatch` (it skips
   stale rows, commits the rest, and reports "(N skipped)");
   `applyVaultCountedDependencyRef` ("Selected dependency changed"); and
   `commitVaultRefs` when its write fails as stale. Route all three through
   `refuseDependencyStale`: write nothing, refuse with `changed — reopen`, and reopen
   the stage fresh, exactly like the same-note `executeDependencyBatch`. Add tests for
   those three and for the stale guards `2bd875d` added to `confirmVaultSingleBlockId`,
   `confirmVaultCountedBlockId`, and `confirmSingleBlockId`, which have none.
4. **Stage tests that exercise the real paths.** Rework these existing tests to use
   `createBulletPropertyPickerHarness` / `createLinkPickerHarness` with a stubbed Tasks
   plugin:
   - Counted add: run the real write and assert the bytes. Do not stub
     `applyCountedLocalTaskDependency`.
   - Two-row cycle: mark two rows that only form a cycle together, through the real
     marking flow. Both stay guarded and nothing is written.
   - No default hotkey: capture the `addCommand` registration for
     `edit-task-dependencies` and assert it has no `hotkeys`. Do not grep the source.
5. **Finish.** Bump the manifest, update the README row, `npm test`, `npm run validate`,
   deploy nav.

## Phase: chips-dp30-reading — DP30 chips, Reading-view row, DP tables

**Repos:** bob-plugins (`plugins/bob-ledger-tools`, `plugins/task-status-cycler`,
`plugins/block-id-prompt`) and bob-cli (`docs/task-dependencies.md`,
`src/native/task_dependencies/tests.rs`).

1. **DP30 renders no chips, against the contract.** `dependencyChipLineOwnedByTask`,
   `dependencyChipLineOwnedByTaskAtDoc`, `dependencyReadingWorkLogAncestor`, and
   `dependencyReadingLineFor` reject any Depends-On line with a Work Log entry anywhere
   above its task. Rust's `dependency_child_of` has no such rule: ownership is "first
   direct child of a `#task` list item". DP20 is already rejected because its parent is
   the Work Log entry, not a `#task`. Drop the extra Work Log rule from all four. Flip
   the `worklogOwner` chip test to expect chips (DP30), and keep a DP20 no-chip test in
   both Live Preview and Reading view.
2. **Unbounded ancestor scan.** With that rule gone, the ownership check walks only from
   the candidate up to its parent item. Add a test that counts line reads: one visible
   candidate near the end of a 5,000-line note reads only O(depth) lines, not the whole
   note.
3. **Reading view acts on the wrong row.** `dependencyReadingLineFor` takes the section
   range from `ctx.getSectionInfo(el)` and requires exactly one line with matching block
   ids. Obsidian renders a whole list as one section, so two tasks that both depend on
   `[[#^x]]` lose `×`/`＋` on both rows. Map each rendered `li` to its source line by
   order instead: the k-th list item in `el`, in document order, is the k-th list-item
   line in the section range (skipping fenced lines). Then check that the mapped line is
   the expected owned Depends-On line, and hide the actions only when the counts
   disagree. Replace the test that asserts the actions are hidden for identical lines,
   and the different-target test, with one where two tasks depend on the same `[[#^x]]`
   and the second row's `×` sends the second line.
4. **DP31 (new): prose-only line.** `⛓️ **DEPENDS ON:** needs review` (label followed by
   prose, no link) is malformed per §2.3, but no vector pins it. Today block-id-prompt
   refuses it and the cycler does not guard it. Add DP31 (`malformed`) to the DP table,
   add the matching row to `dp_vectors` in `src/native/task_dependencies/tests.rs`
   (confirm Rust's parser agrees; if it does not, fix whichever side is wrong and say so
   in the commit), and make the cycler's `isTaskDependencyLine` guard it. Add the row to
   the nav (`scripts/test-navigation-dependencies.cjs`), cycler, block-id-prompt, and
   chip suites. The nav row is a test-only edit; if nav's reader disagrees, record a
   `PROPOSED FOLLOW-UP:` rather than editing nav `main.js` in this phase.
5. **Complete the DP tables.** The cycler and block-id-prompt tables mention DP19 and
   DP20 only in a comment. Add rows for both. They are context vectors and those guards
   are line-level, so pin each plugin's actual behaviour and say in the row comment why
   a context-free guard treats them that way. The tables must then cover DP1–DP31.
6. **Finish.** Bump the three manifests, update their README rows (block-id-prompt's row
   changed only its version last time; describe the behaviour), `npm test`,
   `npm run validate`, deploy the three plugins; `just all` in bob-cli.

## Phase: hooks-r9-split — R9 for label-only lines, comments, file sizes

**Repo:** bob-cli.

1. **R9 is not applied when the field has adoptable ids.** In
   `src/native/task_status_hooks/reconcile.rs`, `plan_empty` sends a label-only line
   through `plan_adoption`, so a label-only line with `[dependsOn:: tasks__a]` and an
   existing `^a` task gets `[[#^a]]` re-adopted onto it. This contradicts R9, DR16, and
   DW5 in the contract and `docs/task-status-hooks.md` (the label-only paragraph). The
   line is the source of truth: a label-only line with no legacy children deletes the
   line and removes the field (`line_removed`), whatever the field holds. Legacy
   children still follow R8/R1. This behaviour predates bob-cli-3n.12.9 (`043d9c5`), but
   the previous phase's DW5 test sidestepped it with an unadoptable `tasks__ghost` id.
   Fix the engine, and change `reconcile_dw5_label_only_line_deletes_line_and_field` (or
   add a sibling test) to use an adoptable id whose `^a` task exists. Keep the
   unadoptable case too. Rerun the full `dependency_lines` CLI suite. The vault has no
   label-only Depends-On lines today, so no live data changes.
2. **DW3/DW6 comments.** These are nav writer vectors, pinned by
   `DW3-DW5 append, remove the middle, and delete on the last link` and
   `DW6-DW7 re-add moves to the end and the field mirrors the line` in bob-plugins
   `scripts/test-navigation-dependencies.cjs`. The comments in
   `src/native/task_dependencies/tests.rs` claim an engine test pins DW3. Make them say
   the Rust side only renders a given order, and point at the nav tests.
3. **File sizes.** `reconcile.rs` grew to 1531 lines and
   `tests/cli/task_status_hooks/dependency_lines.rs` to 1580. Extract the duplicated
   `self_named`/`unadoptable` warning loops in `reconcile.rs` into one helper. Move a
   coherent group of `dependency_lines.rs` tests (for example the DW and Summary tests)
   into a new sibling module registered next to it. Both files must end under about 1500
   lines, with no behaviour change.
4. **Finish.** Update `docs/task-status-hooks.md` if any wording changes, then
   `just all`.

## Phase: rollout — fleet reinstall and real-vault dry run

1. **athena.** From up-to-date bob-cli master,
   `cargo install --path . --locked --force`. Pull the opened bob-plugins checkout to
   origin master, then `bob plugins sync -n -r "<opened bob-plugins path>"`. Check
   `bob plugins list`: 0 drift, and the new versions of bob-navigation-hotkeys,
   bob-ledger-tools, task-status-cycler, and block-id-prompt.
2. **Real-vault dry run.** Run `bob task-status-hooks --dry-run -f json` against
   `~/bob`. The last rollout could not: `2026/20261003.md` did not exist yet. If today's
   daily note is still missing, record the refusal, retry later in the turn, then note
   it. Paste every dependency count and warning kind into the phase notes and explain
   each non-zero projection, adoption, heal, `line_removed`, or canonicalisation. Never
   run a live pass.
3. **apollo.** `git pull --ff-only` in its bob-cli and bob-plugins checkouts, then
   `cargo install --path . --locked --force` and sync the plugins. It runs no hooks.
   Read `tailnet.md` with `/sase_memory_read` for access.
4. **MacBook** (best effort; `ssh -o ConnectTimeout=15 mac`, retrying for about 10
   minutes; it was unreachable at the last rollout). Find how `bob` was installed
   (`~/.cargo/.crates2.json`):
   - a clean `path+file://` checkout on `master`: `git pull --ff-only`, then
     `cargo install --path . --locked --force`;
   - a `git+` install: reinstall from the same source with `--locked --force`;
   - anything else: stop and record the steps.

   Pull and sync its bob-plugins checkout. Verify with
   `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json' | jq 'has("dependency_projection_updates")'`.

5. **Record** in the phase notes, per machine: bob commit, hooks capability, plugin
   versions. List exactly what is left for Bryan:
   - the MacBook steps, if it was unreachable;
   - reloading the four plugins in each running Obsidian;
   - the pilot checklist: add prerequisites from two projects, remove a completed one,
     follow a chip in Live Preview and Reading view (including two tasks that share a
     prerequisite), edit while another note has unsaved changes, and optionally bind a
     chord to **Edit task dependencies**.

## Deliberately not doing

- Closing bob-cli-3n.12.9, bob-cli-3n.12, or bob-cli-3n, or editing their plan files:
  their land agents do that after this epic lands.
- The deferred features already filed: bob-cli-3o (retire R8 legacy readers), bob-cli-3p
  (reverse Blocks… stage), bob-cli-3q (task-line mini-badge), bob-cli-3r (v2 path
  codec), bob-cli-3k (cycler Alt+] `finalizeClosedTasks` for ordinary tasks).
- The parallel test flakes: bob-cli-2e (env races) and the completion flakes recorded on
  epic bob-cli-3j.
- Adding a `just check` recipe (bob-cli-3c).
- Supporting Depends-On lines inside blockquotes or callouts.
- Any vault content edit or live hooks pass.

## Risks

| Risk                                                                    | Mitigation                                                                                                                               |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| The R9 fix deletes a field someone meant to keep                        | The vault has no label-only lines today; rollout dry-runs the real vault and explains every `line_removed` before Bryan's next live pass |
| Wider snapshot gate brings back whole-vault reads on hot paths          | Read-counting tests pin zero extra reads for adds and for non-`[?]` parents                                                              |
| Mapping Reading-view rows by order misfires on unusual lists            | Hide the actions whenever the item and line counts disagree; tests cover identical lines and nested items                                |
| A stage badge rule change surprises Bryan                               | The rule is written into contract §6 before the code changes                                                                             |
| Parallel phases conflict in `package.json`, the README, or the contract | Keep both sides of single-line conflicts; the two nav phases run in sequence                                                             |
