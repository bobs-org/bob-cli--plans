---
tier: epic
title: 'Land task dependency link fixes: nav writer and mirror bugs, Reading-view
  chips, DP29, hooks test gaps, rollout'
goal: 'Every gap the bob-cli-3n.12 landing audit confirmed is fixed and pinned by
  a test: nav writes the right ids and one undo step per gesture, the hand-edit mirror
  and stage match their design, Reading-view chips act on the right task, all five
  recognisers agree on every DP vector including DP29, the hooks DW/DP/Summary tests
  are real, and the fixed bob and plugins are installed across the fleet.'
parent_bead: bob-cli-3n.12
phases:
- id: chips-reading-align
  title: Reading-view chips, recogniser alignment, and the DP29/DP30 contract
  depends_on: []
  size: medium
  description: 'chips-reading-align: settle DP29 (blockquote is not-a-line everywhere)
    and add DP30 in the contract and Rust DP test, give Reading view the owning-line
    rules and a section-derived line, make cycler/bip agree with every DP vector,
    stop full-document copies in Live Preview, cache the lookup index through freshnessEnsureMemo,
    and make the vacuous chip tests real.'
- id: nav-writer-bugs
  title: Fix the nav dependency writer bugs the landing audit confirmed
  depends_on: []
  size: medium
  description: 'nav-writer-bugs: reject blockquoted Depends-On lines in nav (DP29),
    fix the kept-link field id lookup and the dropped same-note target id write, make
    counted add/remove one transaction, stop whole-vault reads on every writer call,
    and give the hand-edit clear path vault-snapshot recovery.'
- id: nav-mirror-stage
  title: Finish the hand-edit mirror baseline and the Depends on stage
  depends_on:
  - nav-writer-bugs
  size: medium
  description: 'nav-mirror-stage: seed the mirror from the CM6 start state and map
    the owner through later changes, delete the dead scheduleDependencyHandEditMirror
    path, count only open prerequisites in the waits-on badge, show readable cycle
    tooltips, reopen on every stale refusal, and add the missing stage harness tests.'
- id: hooks-gaps
  title: Close the hooks DW, DP, Summary, docs, and per-run copy gaps
  depends_on: []
  size: medium
  description: 'hooks-gaps: replace the hollow DW1/DW2/DW3/DW5/DW6 tests, pin the
    Summary dependency counts, fix the task-status-hooks JSON example, narrow a needless
    pub(crate), and stop ReconcileWorker::new copying every vault note per run.'
- id: rollout
  title: Reinstall bob and resync plugins across the fleet with the landing fixes
  depends_on:
  - chips-reading-align
  - nav-writer-bugs
  - nav-mirror-stage
  - hooks-gaps
  size: small
  description: 'rollout: reinstall bob from master and sync the plugins on athena
    and apollo, update the MacBook best effort, dry-run the hooks against the real
    vault and explain every dependency count, and record versions and what is left
    for Bryan.'
proposed_by: bbugyi200.athena.bob-cli-3n.12.land
create_time: 2026-10-03 01:27:47
status: wip
bead_id: bob-cli-3n.12.9
---

- **PROMPT:** [prompts/202610/task_dep_links_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_dep_links_landing_fixes.md)
- **PARENT:** [202610/task_dep_links_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_dep_links_fixes.md)
- **BEAD:** [bob-cli-3n.12.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3n/bob-cli-3n.12.9.md)

# Plan: Land task dependency link fixes (bob-cli-3n.12 remaining work)

## Context

Epic bob-cli-3n.12 (`plan:202610/task_dep_links_fixes.md`; read it with
`sase bead read bob-cli-3n.12 -r "<why>"`) fixed the defects the bob-cli-3n landing
found in the Depends-On contract. All eight of its phases closed. Its land agent then
audited every phase against that plan, and confirmed by reading the code (and, for the
nav bugs, in-memory node probes) that the gaps below remain. This epic fixes only those
gaps. When it lands, its land agent resumes the bob-cli-3n.12 landing through
`parent_bead`, so no phase here closes bob-cli-3n.12 or edits its plan file.

**Contract:** `docs/task-dependencies.md` in bob-cli is the single source of truth
(grammar, identity, R1–R10, stage, chips, api v1, DP/DW/DR/DK/DC vectors).

Line numbers below are from bob-cli `3b04a06` and bob-plugins `0a7ee3d`. They will
drift, so locate code by symbol name.

**Live usage now exists.** 16 vault notes already carry `⛓️ **DEPENDS ON:**` lines, and
no vault note has a blockquoted `#task`. The deployed plugins (athena vault: nav 1.59.0,
ledger-tools 1.20.0, cycler 1.21.0, block-id-prompt 1.19.0) carry the bugs below until
each phase deploys its fix.

## Cross-cutting rules for every phase

- **Read first:** `docs/task-dependencies.md`. Before changing behaviour that a decision
  record governs, read `decisions:task-deps-are-depends-on-links` and
  `decisions:task-lanes-are-sticky` (`sase memory read … -r "<why>"`).
- **Vectors are the contract.** A new behaviour gets a numbered vector before it gets
  code. If an implementation disagrees with a vector, fix the doc first in the same
  phase and say so in the commit.
- **bob-cli:** `just check` must pass (do not run `just check-full`). Keep touched Rust
  files under about 1500 lines. Update `README.md` and `docs/*.md` in the phase that
  changes behaviour.
  `capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes` is
  a known parallel flake (bead bob-cli-2e): rerun it in isolation before treating it as
  a regression, and do not fix it here.
- **bob-plugins** (open it with `/sase_repo`: `sase repo open bob-plugins -r "<why>"`,
  and use only the printed path):
  - `npm test` and `npm run validate` must pass. Add every new test file to the `test`
    script in `package.json`.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`, run
    from the opened path.
  - `plugins/bob-navigation-hotkeys/main.js` is about 42k lines; use `grep -an` and read
    targeted ranges. `plugins/bob-ledger-tools/main.js` is also large.
  - New tests must exercise the real async plugin paths (async vault/editor stubs, the
    picker harnesses), not only pure helpers, and every bug fix gets a test that fails
    on the pre-fix source.
  - `nav-writer-bugs` and `nav-mirror-stage` both edit nav and run in sequence. Parallel
    phases may both edit `package.json`, the plugins README, or
    `docs/task-dependencies.md`; when rebasing, keep both sides of single-line
    conflicts.
- **Vault:** no phase edits `~/bob` content. Plugin deploys write only
  `.obsidian/plugins/<id>/`. Never run a live `bob task-status-hooks` pass against
  `~/bob`; dry runs (`--dry-run -f json`) are fine. Reproduce in scratch vaults under
  `/tmp` with `BOB_DAY_FILE` set and backdated files (`touch -d '-1 hour' …`). The repo
  build lives in `$CARGO_TARGET_DIR/debug/bob`.
- **Follow-ups:** epic phase workers never create beads. Record `PROPOSED FOLLOW-UP:`
  notes on your own phase bead.

## Phase: chips-reading-align — Reading view, recognisers, DP29/DP30

**Repos:** bob-plugins (`plugins/bob-ledger-tools`, `plugins/task-status-cycler`,
`plugins/block-id-prompt`) and bob-cli (`docs/task-dependencies.md`,
`src/native/task_dependencies/tests.rs`).

1. **Contract first.**
   - **DP29** (`> - ⛓️ **DEPENDS ON:** [[#^a]]`) is `not-a-line` in every recogniser,
     with no carve-out. Rewrite its context cell to drop "nav's writer still round-trips
     quoted lines it manages", and state in the grammar section that a blockquoted task
     cannot own a Depends-On line (nav refuses dependency gestures there;
     `nav-writer-bugs` implements that). The vault has no blockquoted `#task`, so
     nothing migrates.
   - **DP30 (new):** a Depends-On line that is the first direct child of a `#task` which
     is itself nested under a Work Log entry. First run the Rust parser's discovery on
     this shape and pin the verdict Rust gives (the audit read it as `accept(1)`); if
     Rust's answer contradicts DP20's intent, fix whichever side is wrong and say so in
     the commit. Reword DP20's context so the two vectors cannot be confused.
   - In `src/native/task_dependencies/tests.rs` `dp_vectors` /
     `dp_discovery_context_vectors`, add rows for **DP24–DP30**. Today the Rust test has
     no rows past DP23 (bob-cli `a069239` touched only the doc).
2. **Reading view** (`renderDependencyChipsIn` and `dependencyReadingLineFor` in
   ledger-tools):
   - Apply the same owning-line rules as Live Preview (`dependencyChipLineOwnedByTask`):
     the parent `li` must be a `#task` item, this `li` its direct child, not inside a
     blockquote, not inside a Work Log entry per DP20/DP30, and the line must parse as
     `accept`/`empty` (never chips on DP16 malformed, DP19 grandchild, DP20, or DP29).
   - Derive the 0-based line from `ctx.getSectionInfo(el)` plus the item's offset in the
     section. `dependencyReadingLineFor` returns the **first** line whose block ids
     match, so with two tasks that both depend on `[[#^x]]`, `×`/`＋` on the second
     task's row edits the first task. When the line cannot be derived uniquely, hide the
     actions.
   - Hide the `⛓️` emoji text as Live Preview does.
   - Tests: two tasks with identical Depends-On lines (the second row's `×` sends the
     second line), and DP16/DP19/DP20/DP29 render no chips.
3. **Live Preview cost.** The decoration rebuild copies every document line whenever a
   candidate line is visible (the block near `main.js:13506-13524`). Walk ancestors with
   `state.doc.line(n)` lookups and touch only visible ranges (contract §7.4).
4. **Lookup cache.** `dependencyTasksIndex` stores its index on `freshnessMemo` but
   never calls `freshnessEnsureMemo`, so when the memo is stale the index is rebuilt on
   every decoration pass. Go through `freshnessEnsureMemo` so there is one cache with
   one invalidation; add a test that two passes over an unchanged vault build the index
   once.
5. **Cycler and block-id-prompt alignment.** Both `isTaskDependencyLine` copies return
   true for DP23/DP25/DP26 but false for DP15/DP16, though the contract calls all five
   `malformed`. Make every `malformed` vector (DP15, DP16, DP23, DP25, DP26) behave the
   same way (guarded: cycler never strikes or retires it, block-id-prompt refuses
   Ctrl+Shift+Enter) and every `not-a-line` vector (DP24, DP27, DP28, DP29) unguarded.
   Block-id-prompt's `isMalformedTaskDependencyLine` must return false for DP24, and its
   comment must say which vectors it covers. Copy the full DP1–DP30 table into both
   suites. Add the missing cycler handler test for Ctrl+Enter **reopening** a dependency
   target.
6. **Chip test gaps** (`scripts/test-ledger-tools-dependency-chips.cjs`):
   - The "visible ranges only" test is vacuous: removing the `offscreen.visibleRanges`
     override still passes, because its duplicate line sits under a non-task parent. Put
     an **owned** Depends-On line outside the visible ranges and confirm the test fails
     without the override.
   - The nested-`li` post-processor fixture's parent must be a `#task` row.
7. **Finish.** Bump the three manifests, update README rows, `npm test`,
   `npm run validate`, deploy the three plugins; `just check` in bob-cli.

## Phase: nav-writer-bugs — writer correctness, cost, DP29

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **DP29.** `DEPENDENCY_NAVIGATION_BULLET_RE` (near `main.js:270`) accepts quote
   prefixes (`^(?<indent>\s*(?:>\s*)*)`). Drop them so nav's reader returns `not-a-line`
   for DP29, matching the contract, Rust, and the other three recognisers. Every
   dependency gesture (stage, counted, Ctrl+D, mirror) on a task inside a blockquote
   refuses with `⛓ Dependencies can't be edited inside a blockquote`. Change the DP29
   row in `scripts/test-navigation-dependencies.cjs` (near line 153) to `not-a-line`,
   and replace the counted-writer test in `scripts/test-navigation-hotkeys.cjs` (near
   lines 2491-2523, which writes `> \t- ⛓️ **DEPENDS ON:**`) with a refusal test.
2. **Kept-link field id (wrong id written).** In `planDependencyEdit`, a kept link whose
   note is not loaded takes its field value by its index in `keptExisting`, but
   `parentFieldValues` is in the original line order (near `main.js:4276`). With line
   `[[#^a]] • [[Other#^x]]` and `Other.md` unloaded, removing `a` writes the id of `a`
   next to the remaining link `[[Other#^x]]`. Look the value up by the link's position
   in `existing` (or, better, by matching the field id's encoded block id), and refuse
   rather than guess when the field and line disagree in length. This path is hit by
   `executeDependencyBatch` (it loads only the current note) and by any deleted target.
   Test both callers.
3. **Same-note target id dropped.** `setLocalTaskDependencyLink` writes
   `options.pendingTargetLine` into its local `content`, then diffs that modified
   `content` against `plan.nextContent`, and skips the commit when `plan.changed` is
   false. So when the line count does not change (the dependent already has a Depends-On
   line), `confirmSingleBlockId` writes `[[#^b]]` while target B never gets `^b` /
   `[id::]`; `chooseTaskDependency` loses the `[id::]` the same way. Diff from the
   editor's original value, always commit the target edit with the link, and keep one
   undo group. Add plugin-level tests for `confirmSingleBlockId`,
   `chooseTaskDependency`, and `pendingTargetLine` with and without an existing
   Depends-On line.
4. **Counted add/remove is N writes.** `applyVaultCountedDependencyRef` calls
   `plugin.applyDependencyEdit` once per counted source task in the same editor note, so
   one counted gesture takes N undo steps and a mid-run refusal leaves the earlier
   sources written (`Updated k tasks; could not update …`). Plan every source on one
   working copy (bottom-up), refuse before any write if any source fails to plan, and
   commit once with one summary notice. Test one undo group and the refuse-before-write
   case.
5. **Whole-vault read on every writer call.** `readDependencyRecoveryVaultContents`
   (near `main.js:35797`) awaits `cachedRead` for every markdown file in the vault
   (about 4,600) and is called from `applyDependencyEdit` on every call, including each
   mirror pass, plus the batch and counted paths. Build the recovery snapshot only when
   a recovery can actually fire (a prerequisite was removed or closed and the dependent
   is Blocked), and keep today's daily note in it when it does. Test with a vault stub
   that counts reads: a mirror touch or an add with no recovery reads zero extra notes.
6. **Hand-edit clear path recovery.** `applyDependencyHandEditClear` still builds
   recovery from the edited note only (near `main.js:36847`), so a Pomodoro-linked
   dependent recovers to Ready instead of Next/In Progress. Use the same vault-snapshot
   recovery (with today's daily note) as the other writer paths. Add a plugin-level
   test, and convert the pure-helper-only Pomodoro-recovery and subfolder-target tests
   from `nav-writer-fix` into plugin-level async tests.
7. **Finish.** Bump the manifest, update the README row (mention the blockquote
   refusal), `npm test`, `npm run validate`, deploy.

## Phase: nav-mirror-stage — mirror baseline and stage polish

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`. Runs after `nav-writer-bugs`.

1. **Mirror baseline and owner.** `enqueueDependencyHandEditMirror` takes its baseline
   from the cached last-seen content (near `main.js:36993`), never seeds it when a note
   opens, and overwrites the owner with the latest edit instead of mapping it. Deleting
   P's last-child Depends-On line as the first edit after opening a note therefore
   mirrors `touch` onto the following sibling and leaves P's field stale. Take the
   pre-change content from the first `ViewUpdate.startState` of a pending burst, map the
   owner position through later `update.changes` (`ChangeSet.mapPos`), and run once the
   cursor leaves the line. Tests through the real CM6 listener path: that first-edit
   deletion with a following sibling, and a vim `dd` of the line.
2. **Dead mirror path.** `scheduleDependencyHandEditMirror` (near `main.js:36950`) has
   no production caller and still reads `change.from.line`; the baseline test near
   `scripts/test-navigation-hotkeys.cjs:3336` exercises it instead of the CM6 listener.
   Delete it and point those tests at the listener.
3. **Stage.**
   - The `🔒 waits on N` badge uses the cycle-edge count (`planDependencyStageView`,
     near `main.js:21940`), so one closed plus one open prerequisite shows `waits on 2`.
     Count open prerequisites only, matching DC7/DC8 and the chip fix in bob-plugins
     `0a7ee3d`.
   - The disabled row's cycle tooltip joins raw row keys (`path\x00blockId`) in
     `renderTaskValueItem` (near `main.js:25906`). Show task descriptions joined by `→`.
   - Stale refusals in `confirmVaultSingleBlockId`, `confirmVaultCountedBlockId`,
     `confirmSingleBlockId`, and the batch stale skips still only say "Task changed;
     dependency not added". Route them through `refuseDependencyStale` so they refuse
     with `changed — reopen` and reopen the stage fresh.
4. **Missing stage tests** (`createBulletPropertyPickerHarness` /
   `createLinkPickerHarness` with a stubbed Tasks plugin): stale guard plus reopen;
   `＋ id`; Esc writes no bytes; one undo group; counted; a cycle formed by two marked
   rows in one batch (guarded, nothing written); `edit-task-dependencies` registered
   with no default hotkey; the waits-on count with a closed prerequisite; the cycle
   tooltip text.
5. **Finish.** Bump the manifest, update the README row, `npm test`, `npm run validate`,
   deploy.

## Phase: hooks-gaps — hooks tests, docs, and per-run copies

**Repo:** bob-cli.

1. **DW tests that test something** (`src/native/task_dependencies/tests.rs` and, where
   the writer is the reconcile engine,
   `tests/cli/task_status_hooks/dependency_lines.rs`):
   - DW5 still has the empty assertion
     (`let links: Vec<String> = Vec::new(); assert!(links.is_empty());`, near line 157).
     Assert DW5's outcome: line deleted, field removed.
   - DW1 (line inserted before a `🗓️ **SCHEDULE LOG**` child) and DW2 (inserted after a
     `❌ **CANCEL LOG**` child, before prose) are not tested anywhere, although comments
     near lines 166-178 claim a CLI test pins them. Test the real slot logic
     (`child_slot`, `is_cancel_log_line`) through the engine, and fix the comments.
   - DW3 and DW6 only format an already-ordered list; drive them through the writer.
2. **Summary line.** No test checks the dependency counts appended to `Summary:` or the
   `Dependencies:` line. Add CLI assertions for both.
3. **Docs.** The JSON example in `docs/task-status-hooks.md` (near lines 985-1003) shows
   `detail` strings that do not match the real formats (`dependsOn := …`,
   `link … kept verbatim; it never blocks`). Generate it from a real scratch-vault dry
   run.
4. **Visibility.** `collect_done/transform.rs` `vault_relative_link_target` is
   `pub(crate)` but used only inside `collect_done`; narrow it.
5. **Per-run copies.** `ReconcileWorker::new` (`reconcile.rs`, near lines 192-215) still
   builds an owned per-line copy of every vault note and runs `logical_lines` /
   `fenced_lines` twice per note, and the doc comment in `reconcile/apply.rs` (near
   lines 184-188) claiming "the whole vault is never copied per run" is false. Borrow,
   or build lines lazily for notes that hold a Depends-On line, a dependency field, or a
   targeted block; compute each note's line views once. Report the real-vault dry-run
   wall time before and after in the phase notes. No behaviour change: the full suite
   and the 25 `dependency_lines` CLI tests must pass unchanged.
6. **Finish.** `just check`.

## Phase: rollout — fleet reinstall and real-vault dry run

1. **athena.** From an up-to-date bob-cli master,
   `cargo install --path . --locked --force`. Pull the opened bob-plugins checkout to
   origin master, then `bob plugins sync -n -r "<opened bob-plugins path>"`. Check
   `bob plugins list`: 0 drift and the new versions of bob-navigation-hotkeys,
   bob-ledger-tools, task-status-cycler, and block-id-prompt.
2. **Real-vault dry run.** `bob task-status-hooks --dry-run -f json` against `~/bob` (if
   today's daily note does not exist yet, record the refusal and retry later in the turn
   or note it). The vault now has 16 notes with Depends-On lines, so expect non-zero
   counts: paste every dependency count and warning kind into the phase notes and
   explain each non-zero projection, adoption, heal, or canonicalisation. Never run a
   live pass.
3. **apollo.** `git pull --ff-only` in its bob-cli and bob-plugins checkouts
   (bob-cli-3n.12's rollout left it at bob-plugins `40e2e3a`, nav 1.58.0 / ledger-tools
   1.19.0), then `cargo install --path . --locked --force` and sync the plugins. It runs
   no hooks. Read `tailnet.md` with `/sase_memory_read` for access.
4. **MacBook** (best effort; `ssh -o ConnectTimeout=15 mac`, retrying for about 10
   minutes). Find how `bob` was installed (`~/.cargo/.crates2.json`): a clean
   `path+file://` checkout on `master` → `git pull --ff-only` then
   `cargo install --path . --locked --force`; a `git+` install → reinstall from the same
   source with `--locked --force`; anything else → stop and record the steps. Pull and
   sync its bob-plugins checkout. Verify with
   `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json' | jq 'has("dependency_projection_updates")'`.
5. **Record** in the phase notes, per machine: bob commit, hooks capability, plugin
   versions. List exactly what is left for Bryan: the MacBook steps if unreachable,
   reloading the four plugins in each running Obsidian, and the pilot checklist (add
   prerequisites from two projects, remove a completed one, follow a chip in Live
   Preview and Reading view, edit while another note has unsaved changes; optionally
   bind a chord to **Edit task dependencies**).

## Deliberately not doing

- Closing bob-cli-3n.12 or bob-cli-3n, or editing their plan files: their land agents do
  that after this epic lands.
- The deferred features already filed: bob-cli-3o (retire R8 legacy readers), bob-cli-3p
  (reverse Blocks… stage), bob-cli-3q (task-line mini-badge), bob-cli-3r (v2 path
  codec), bob-cli-3k (cycler Alt+] `finalizeClosedTasks` for ordinary tasks).
- The capture_pomodoros parallel flake (bob-cli-2e).
- Supporting Depends-On lines inside blockquotes or callouts.
- Any vault content edit or live hooks pass.

## Risks

| Risk                                                                   | Mitigation                                                                                                                                 |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Live Depends-On lines (16 notes) hit a fixed-but-different writer path | Every writer fix gets a plugin-level async test that fails on the pre-fix source; rollout dry-runs the real vault and explains every count |
| Lazy recovery snapshot misses a recovery that should fire              | Recovery tests cover Pomodoro-linked dependents on every writer path, including the hand-edit clear path                                   |
| The DP29 refusal blocks a real workflow                                | The vault has no blockquoted `#task`; the refusal notice says why                                                                          |
| Parallel phases conflict in `package.json`, README, or the contract    | Keep both sides of single-line conflicts; nav phases run in sequence                                                                       |
