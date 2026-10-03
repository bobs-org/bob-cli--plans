---
tier: epic
title: 'Finish task dependency links: fix the hooks reconcile, chips, nav writer,
  mirror, and stage defects found at landing'
goal: 'Every defect the bob-cli-3n landing found in the shipped Depends-On contract
  is fixed and pinned by a test: the hooks reconcile never corrupts notes, chips act
  on the right task, the nav writer works across notes, the hand-edit mirror works
  in the live editor, the stage matches its design, the legacy writers are gone, the
  docs and decision record are accurate, and the fixed bob and plugins are installed
  across the fleet.

  '
parent_bead: bob-cli-3n
phases:
- id: hooks-correctness
  title: Fix R1-R10 reconcile correctness bugs in bob task-status-hooks
  depends_on: []
  size: medium
  description: 'hooks-correctness: fix the reconcile edit ordering that duplicates
    task lines, the task-line rewrite merge, dropped current-daily edits, previous-daily
    writes, field placement before trailing tags, archived legacy children, and the
    silent unencodable target, and add the regression and missing CLI tests.'
- id: hooks-docs-cleanup
  title: Hooks dependency docs, Summary line, helper dedupe, and reconcile split
  depends_on:
  - hooks-correctness
  size: medium
  description: 'hooks-docs-cleanup: add the Dependency lines, guarded-write, and Output
    docs, README and long_about, append the counts to Summary, dedupe the copied link
    helpers, split reconcile.rs, remove per-run whole-vault copies, and fill the DW/DR
    unit-test gaps.'
- id: chips-compat-fix
  title: Fix dependency chips and align the Depends-On recognisers
  depends_on: []
  size: medium
  description: 'chips-compat-fix: pin api v1 ref.line as 0-based and fix the chip
    off-by-one and stale widget meta, restrict chips to real Depends-On lines, fix
    Reading view, hover, and the lookup cache, align the ledger-tools, cycler, and
    block-id-prompt recognisers with new DP vectors, and add the missing tests.'
- id: nav-writer-fix
  title: Fix the navigation-hotkeys dependency writer across notes
  depends_on: []
  size: medium
  description: 'nav-writer-fix: await cross-note preparation, load or tolerate every
    linked note, check link uniqueness vault-wide, fix the same-note +id batch, ADJ-8
    recovery, counted line shifts, undo grouping, and stale checks, and add async
    plugin-level tests.'
- id: nav-mirror-gestures
  title: Rebuild the hand-edit mirror and finish gesture cleanup and legacy removal
  depends_on:
  - nav-writer-fix
  size: medium
  description: 'nav-mirror-gestures: move the hand-edit mirror to a CM6 update listener
    with correct ownership, guards, and status effects; fix field-only and counted
    Ctrl+D and counted N!; delete the leftover legacy writers and identity migration
    script; fix README claims.'
- id: nav-stage-polish
  title: Bring the Depends on stage to its design
  depends_on:
  - nav-mirror-gestures
  size: medium
  description: 'nav-stage-polish: fix ranker tie order, empty-query order, and the
    row cap hint; finish badges, tooltips, muting, classes, and colours; reopen stale
    stages; fix notices and Task Link batches; and add the missing harness tests.'
- id: docs-memory-fix
  title: Correct the decision record and sweep stale dependency docs
  depends_on:
  - hooks-docs-cleanup
  - chips-compat-fix
  - nav-stage-polish
  size: small
  description: 'docs-memory-fix: add the missing rejected alternative, decided date,
    and plugin evidence to the decision record; fix stale docs/projects.md, freshness.md,
    capture.md, and today.rs wording; settle the DC7/DC8 and archive-link contract
    gaps.'
- id: rollout
  title: Reinstall bob and resync plugins across the fleet with the fixes
  depends_on:
  - hooks-docs-cleanup
  - chips-compat-fix
  - nav-stage-polish
  size: small
  description: 'rollout: reinstall bob from master and sync the plugins on athena
    and apollo, update the MacBook on a best-effort basis now that the hooks are safe,
    dry-run the fixed hooks against the real vault, and record versions and what is
    left for Bryan.'
proposed_by: bbugyi200.athena.bob-cli-3n.land
create_time: 2026-10-02 23:24:07
status: done
bead_id: bob-cli-3n.12
---

- **PROMPT:** [prompts/202610/task_dep_links_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_dep_links_fixes.md)
- **PARENT:** [202610/task_dep_links.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_dep_links.md)
- **BEAD:** [bob-cli-3n.12](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3n/bob-cli-3n.12.md)

# Plan: Finish task dependency links (bob-cli-3n remaining work)

## Context

Epic bob-cli-3n (`plan:202610/task_dep_links.md`; read it with
`sase bead read bob-cli-3n`) shipped the Depends-On contract across bob-cli and
bob-plugins, and all 11 of its phases closed. Its land agent then audited every phase
against the plan and found real defects that the green suites (bob-cli `just all`;
bob-plugins `npm test` 1291/1291) do not catch. Two hooks bugs were reproduced in a
scratch vault, and two plugin bugs were confirmed in the source. This epic fixes only
that remaining work. Once it lands, its land agent resumes the bob-cli-3n landing
through `parent_bead`.

**Contract:** `docs/task-dependencies.md` in bob-cli is the single source of truth
(grammar, identity, R1–R10, semantics, stage, chips, api v1, DP/DW/DR/DK/DC vectors).
The parent plan's Design sections explain the intent.

**Live-risk ordering.** The MacBook cron is the only live hooks runner, and it has not
been updated yet. Do not tell anyone to install bob at the current master on the Mac
until `hooks-correctness` has landed. `rollout` does that. The plugins already deployed
to the vault (nav 1.55.0, ledger-tools 1.18.0) carry the plugin bugs below until each
plugin phase deploys its fix.

Line numbers below are from bob-cli `f21e856` and bob-plugins `46ddd1e`. They will
drift, so locate code by symbol name.

## Cross-cutting rules for every phase

- **Read first:** `docs/task-dependencies.md`. Before changing behaviour that a decision
  record governs, read `decisions:task-deps-are-depends-on-links` and
  `decisions:task-lanes-are-sticky` (`sase memory read … -r "<why>"`).
- **Vectors are the contract.** If an implementation disagrees with a vector, fix the
  doc first in the same phase and say so in the commit. A new behaviour gets a new
  numbered vector before it gets code.
- **bob-cli:**
  - `just all` (fmt, lint, test) must pass.
  - Keep touched Rust files under about 1500 lines where practical.
  - Update `README.md` and `docs/*.md` in the phase that changes behaviour.
  - If a phase touches `--help` / `long_about` text, read `sase/memory/cli_rules.md`
    with `/sase_memory_read` first.
  - `capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
    is a known parallel flake (bead bob-cli-2e). Do not fix it here. Rerun it in
    isolation before treating it as a regression.
- **bob-plugins** (open it with `/sase_repo`: `sase repo open bob-plugins -r "<why>"`,
  and use only the printed path):
  - `npm test` and `npm run validate` must pass.
  - Add every new test file to the `test` script in `package.json`.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy with `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`, run
    from the opened path.
  - Phases that touch the same plugin run in sequence. Parallel phases may both edit
    `package.json` or README.md. When rebasing, keep both sides of single-line
    conflicts.
  - `plugins/bob-navigation-hotkeys/main.js` is about 42k lines; use `grep -an` and read
    targeted ranges.
  - New tests must exercise the real async plugin paths (async stubs that return
    Promises, the picker harnesses), not only the pure helpers. The bugs below hid
    behind synchronous stubs.
- **Vault:** no phase edits `~/bob` content. Plugin deploys write only
  `.obsidian/plugins/<id>/`.
- **Hooks safety:** never run a live `bob task-status-hooks` pass against `~/bob`. Dry
  runs (`--dry-run -f json`) are always fine. Reproduce in scratch vaults under `/tmp`
  with `BOB_DAY_FILE` set. Backdate the files (`touch -d '-1 hour' …`) so the 2 s quiet
  interval doesn't defer projection writes. The repo's build lives in
  `$CARGO_TARGET_DIR/debug/bob`, not `target/debug/bob`.
- **Memory:** only `docs-memory-fix` edits memory, through `/sase_memory_write`. This
  plan's approval is the authorization to correct the bob-cli-3n publish step's
  omissions.
- **Follow-ups:** epic workers record `PROPOSED FOLLOW-UP:` notes on their own phase
  bead instead of creating beads.

## Phase: hooks-correctness — reconcile correctness bugs

**Repo:** bob-cli. **Files:** `src/native/task_status_hooks/reconcile.rs`, `compose.rs`,
`sync.rs`, CLI tests in `tests/cli/task_status_hooks/dependency_lines.rs`.

Fix each defect and pin it with a CLI regression test that uses the exact repro. Each
test must also assert that a second run is clean.

1. **Same-index edit ordering corrupts notes (reproduced).** `ReconcileWorker::apply`
   sorts each file's edits by `(line_index, kind as u8)` and then reverses them. Because
   `EditKind` is `Replace, Remove, Insert`, an `Insert` and a `Replace` at the same
   index apply Insert first, and the Replace then overwrites the inserted line.
   - Repro, `tasks.md`:
     ```
     - [ ] #task A [dependsOn:: tasks__c] ^a
     - [ ] #task B ^b
     - [ ] #task C [id:: tasks__c] ^c
       - ⛓️ **DEPENDS ON:** [[#^b]]
     ```
   - One live run produces `B [id:: tasks__b] ^b` followed by the original `B ^b` (task
     `^b` duplicated), and A's adopted line is lost. The next run inserts it.
   - Fix: at the same original index, apply Replace/Remove before Insert, so an insert
     lands before the rewritten original line. Audit Remove+Insert at the same index
     too.
2. **Task-line rewrites overwrite each other.** `project_id` reuses the line's original
   `depends_on`. `project_field` and `plan_adoption` reuse the original `task_id`. The
   merged values saved in `task_rewrites` are never read back.
   - Repro: a chain A→B→C (lines) where B has no `[id::]`. One of B's two edits is lost,
     A's dependency resolves as unresolved, and A isn't Blocked until a second run.
   - Fix: every task-line rewrite starts from the pending rewrite for that line, if any.
     One run must settle the chain.
3. **Current daily-note reconcile edits are dropped.** In `compose.rs` (the
   `normalized_daily_contents` fallback near the top of `compose_outputs`), when the
   daily note has no status change, compose writes the pre-reconcile normalized
   contents. Adoptions and stamps there are reported but never written. Fix it and add a
   test for an adoption plus a target stamp inside today's daily note.
4. **The previous daily can be written.** `project_id` stamps `[id::]` on targets
   without checking for the previous-daily input, which breaks contract §4 ("The
   previous daily note snapshot is never written").
   - Fix: never write it. Leave such a link unprojected with a named warning kind. Add
     the kind to contract §4 (and its output keys) first, then add a test.
5. **Field writer stops at trailing tags (reproduced).** `set_task_fields` peels only
   trailing Dataview fields. Given `- [ ] #task D [dependsOn:: other__x] #hide ^d` with
   the line `[[#^e]]`, it appends a second `[dependsOn:: other__e]` after `#hide`. The
   parser keeps reading `other__x`, so D is never Blocked and the change repeats. The
   vault has lines like `[id:: …] #hide ^prj`.
   - Fix: find and replace existing `[id::]` / `[dependsOn::]` fields anywhere in the
     task line's Tasks suffix. Place new ones per contract §3 (right of `[fresh::]`,
     before `^id`). Build on `freshness::placement::tasks_suffix_start`,
     `task_fields::inline_fields`, and the `projects/edits.rs` upsert helpers
     (`upsert_task_scheduled`, `remove_all_inline_fields`,
     `task_metadata_insertion_offset`; promote them to `pub(crate)`), as the parent plan
     required, rather than a bespoke peeler.
   - Tests: trailing tags, tags between fields, and a `[id:: …] #hide ^prj` target
     stamp.
6. **Archived legacy children are never handled.** Explicit `done/` legacy children are
   skipped early (the `done/` check before the Archive arm in the R8 legacy-child scan),
   so the Archive arm can never run. Their field ids are either dropped or warned as
   `unadoptable_dependency_id`, which contradicts contract §4.3. Fix it so archive
   targets follow §4.3, and test both a line-owning and a field-only dependent.
7. **Unencodable target is silently unprojected.** In `project_id`, a target whose path
   can't encode and that has no `[id::]` is skipped with no warning. Emit the contract's
   warning kind (add one to §4 if none fits) and test it.
8. **`dependency_field_ids_dropped`.** R1/DR3 drops appear only in detail text. If
   contract §4's output keys list them, emit and count them; otherwise add the key to
   the contract first. Test it.
9. **Missing CLI tests from the parent plan's hooks-reconcile step 6:**
   - removing the last open prerequisite unblocks a `[?]` dependent in the same run;
   - breadcrumb retention followed by a later heal (DR9);
   - quiet-interval deferral of a projection-only write;
   - a cross-note target `[id::]` write;
   - the legacy window (a line plus legacy children together).
10. **Verify.**
    - `just all`.
    - Rerun both repros above in a scratch vault with the new build.
    - Run
      `BOB_DIR=~/bob "$CARGO_TARGET_DIR/debug/bob" task-status-hooks --dry-run -f json`
      and paste every dependency count and warning kind into the phase notes. Explain
      any non-zero projection, adoption, heal, or canonicalisation (for example, `#hide`
      lines the old writer mishandled).

## Phase: hooks-docs-cleanup — docs, Summary, dedupe, split

**Repo:** bob-cli. Runs after `hooks-correctness`, because it touches the same files.

1. **Docs** (`docs/task-status-hooks.md`):
   - Add a dedicated "Dependency lines" section that summarises R1–R10 and points to the
     contract. Move the R1–R10 summary out of "Derived Blocked" and leave a one-line
     pointer there.
   - "Guard Rails" / guarded writes: the quiet interval now also covers projection-only
     writes.
   - "Output": add every new key (`dependency_projection_updates`,
     `adopted_dependency_lines`, healed, canonicalized, `legacy_dependency_children`,
     `dependency_warnings`, plus any key `hooks-correctness` added) to the JSON example
     and the field list.
2. **README hooks section and `long_about`** (`src/native/task_status_hooks/mod.rs`):
   mention Depends-On reconciliation and the new output. Read `cli_rules.md` first.
3. **Summary line.** Append the dependency counts to the **end** of the `Summary:` line,
   as the parent plan specified, instead of (or in addition to) the separate
   `Dependencies:` line. Existing prefix assertions must keep passing.
4. **Dedupe the copied helpers.** Share one implementation of each, and remove the
   needless `pub(crate)` widening:
   - `strikethrough_spans` (`task_dependencies/mod.rs` and
     `task_status_hooks/pomodoro.rs`);
   - `inline_code_spans` (`task_dependencies/mod.rs` and `collect_done/link_repair.rs`);
   - `vault_relative_link_target` (`task_dependencies/mod.rs` and
     `collect_done/transform.rs`);
   - `block_link_spans` decoding vs `block_link_occurrences`.
5. **Split `reconcile.rs`** (about 1786 lines) into a `reconcile/` module with each file
   under about 1500 lines. No behaviour change.
6. **Efficiency.**
   - `sync.rs` clones every file in the vault for reconcile on each run; borrow or clone
     only touched files.
   - `heal_links` rebuilds `by_block` from the whole vault for every dependent; build it
     once per run.
7. **Unit-test gaps.**
   - In `task_dependencies/tests.rs`, add the missing DW vectors DW1, DW2, DW3, DW6,
     DW15, and DW16.
   - Replace DW5's empty assertion
     (`let empty = Vec::new(); assert!(empty.is_empty())`).
   - Add table-driven DR unit tests for the reconcile engine where feasible.
   - Add an end-to-end move-done-tasks test for `archive_stayed_block_ids` (pathless
     links rewritten in an archived block whose target stays).
8. **Finish.** `just all`.

## Phase: chips-compat-fix — chips and recognisers

**Repo:** bob-plugins (`plugins/bob-ledger-tools`, `plugins/task-status-cycler`,
`plugins/block-id-prompt`) and bob-cli `docs/task-dependencies.md`.

1. **api v1 `ref.line` convention.**
   - Contract §9 doesn't say whether `line` is 0- or 1-based. Nav
     (`openDependencyStageForRef`, `removeDependency`) and its tests treat it as a
     0-based line index.
   - Pin "0-based line index" in §9 first.
   - Chips compute `lineNumber = line.number - 1` and then send `lineNumber + 1` (in
     `dependencyOpenStage` and the remove handler). So with task A, task B, B's
     Depends-On line, then task C, clicking `×`/`＋` on B's chips acts on C. Send the
     0-based index. Add a test that pins it against a stubbed nav api.
2. **Stale widget meta.** `DependencyChipWidget.eq()` compares only the serialised
   model, so a reused DOM node keeps a stale `lineNumber` (after lines are inserted
   above) and a stale `interactive` flag (nav's api loading later never reveals the
   buttons).
   - Resolve the line at click time (`view.posAtDOM` → line) and re-check the api at
     click/hover time, or include the meta in `eq()`.
3. **Chips only on real Depends-On lines.**
   - No render path checks for a list marker, a direct child of a `#task` line, or a
     Work Log entry, so chips render on paragraphs, grandchildren, and Work Log lines.
     DP19/DP20 are enforced only through test-only `opts` flags.
   - Apply the contract's owning-line rules in Live Preview (use the document: the
     parent line must be a `#task` list item and this line a direct child) and in
     Reading view.
4. **Reading view** (`renderDependencyChipsIn`):
   - It matches any `li` whose text contains `DEPENDS ON` anywhere, so the parent task's
     `li` is decorated too (made `bob-dep-row`, given a label and summary, and two `×`
     per anchor). Match only an `li` whose own leading text is the label.
   - The separator-hiding loop does nothing, and the bold label isn't hidden. Fix both.
   - Add the status-symbol box, Done strike-through, and the `✓×N` collapse, to match
     Live Preview and DC.
   - `×`/`＋` send `line: null` (always `invalid-ref`). Derive the 0-based line from
     `ctx.getSectionInfo(el)` plus the item's offset, or hide the actions when the line
     can't be derived.
5. **Hover preview.** The `hover-link` trigger passes `hoverParent: null` and the whole
   row as `targetEl`. Pass a real hover parent and the chip element, as the existing
   bob-plan chips do.
6. **Lookup cache.** `dependencyTasksIndex` keeps its own `dependencyChipIndexCache`.
   Fold it into the `freshnessEnsureMemo` lifecycle, as the parent plan required (one
   cache, same invalidation).
7. **Align the recognisers.** The four recognisers (ledger-tools `parseDependencyLine`,
   the cycler and block-id-prompt copies of `isTaskDependencyLine`, and nav
   `parseDependencyLine`) agree on every current DP vector, but diverge on inputs the
   contract doesn't cover:

   | Input                                      | ledger-tools | cycler / block-id-prompt | nav        |
   | ------------------------------------------ | ------------ | ------------------------ | ---------- |
   | `🔗` + VS16                                | accept       | reject                   | not-a-line |
   | bare `[[note]]` or `[[note#Heading]]` link | malformed    | treated as a line        | malformed  |
   | label followed only by separators          | empty        | treated as a line        | malformed  |
   | lowercase label                            | accept       | reject                   | reject     |
   | no list marker                             | accept       | reject                   | reject     |
   | `> -` blockquote                           | malformed    | cycler yes, bip no       | accept     |
   - Add numbered DP vectors for these to the contract, choosing verdicts that match nav
     and the Rust parser. Run the Rust DP test to confirm, and if Rust disagrees, fix
     whichever side is wrong in this phase.
   - Then align the three JS copies with those vectors.
   - Block-id-prompt must also refuse Ctrl+Shift+Enter on a **malformed** Depends-On
     line (for example DP16, trailing prose). Today it deletes the link token there.

8. **Cycler gaps.**
   - Counted Alt+]/Alt+[ shows no notice when the cursor isn't on a link. Show
     `⛓ Put the cursor on a dependency link to cycle it`, as the single form does.
   - Add handler-level tests: Ctrl+Enter on a dependency link closes or reopens the
     target only (no strike or unstrike) and runs the `finalizeClosedTasks` recovery;
     single and counted Alt+] cycle the target under the cursor and show the notice
     otherwise.
9. **Chip test gaps** (`scripts/test-ledger-tools-dependency-chips.cjs`):
   - assert "visible ranges only" (the `offscreen` fixture is built but unused);
   - a nested-`li` post-processor test (the parent task stays undecorated);
   - label the DC vectors by id and assert the symbols DC2, DC5, and DC6 specify, plus
     DC9's summary.
10. **Finish.** Bump all three manifests, update the README rows, run `npm test` and
    `npm run validate`, and deploy the three plugins.

## Phase: nav-writer-fix — the dependency writer across notes

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`.

1. **Cross-note preparation is never awaited.**
   - `applyDependencyEditTransaction` is synchronous, but the plugin passes it the async
     `prepareDependencyTargetNote`. `result.ok` is undefined on the returned Promise, so
     every add that needs target preparation (a cross-note target with `^id` but no
     `[id::]`, or a commitment transfer to a cross-note target) fails with
     `target-preparation-failed`.
   - Meanwhile the preparation still writes in the background, so a failed gesture can
     leave the target promoted with no link.
   - Make the transaction async and await each preparation; abort before the commit on
     failure. Update every caller and the tests
     (`scripts/test-navigation-dependencies.cjs`, the one-undo-group and
     failed-preparation tests) to use async stubs.
2. **Links to unloaded notes block every edit.**
   - `applyDependencyEdit` loads only the parent note plus the add/remove target notes,
     and the same-note writers pass only the current note. `planDependencyEdit` returns
     `target-not-found` for any existing link whose note isn't in `files`.
   - So once a task has one cross-note prerequisite, every other add or remove refuses,
     including removing a link to a deleted note (the design promises missing targets
     can always be removed).
   - Fix: load the notes of every link already on the line (from open buffers, else the
     vault), or let the planner keep unresolved existing links verbatim and resolve ids
     from the field or pool. Removing a missing target must always work.
3. **Link form uses only loaded notes.**
   - `canonicalDependencyLink` checks basename uniqueness against the loaded notes only,
     so it emits `[[cash#^a]]` although the vault has many duplicate basenames.
   - `dependencyPathForLinkNote` then maps `[[cash#^a]]` to the root `cash.md`, so a
     target in a subfolder (for example `chat/`) breaks every later edit on that
     dependent.
   - Fix: check uniqueness against the vault's markdown files, and resolve link paths
     with `app.metadataCache.getFirstLinkpathDest(linkpath, sourcePath)`. Keep the pure
     helpers testable by injecting the file list and resolver.
4. **Same-note marked-batch `＋ id` refuses** (bob-cli-3n.7's follow-up, a regression
   from bob-plugins `5194bc8`).
   - The gesture: Ctrl+Shift+P → Depends on, Tab-mark rows in the dependent's own note
     with at least one `＋ id`, ↵, confirm the ids, apply.
   - Path: `promptNextBatchBlockId` → `confirmBatchBlockId` → `executeDependencyBatch`.
     `collectAddition` returns `{ blockId: confirmedId }` without writing the target's
     `^id` / `[id::]`, so the planner fails `target-not-found` and nothing is written.
   - Fix: stamp the confirmed ids into a working copy
     (`applyPromptedBlockIdPreservingLegacyId`, as `planCountedLocalTaskDependency`
     does), plan on that copy, and commit once with `applyEditorContentTransaction`.
5. **ADJ-8 recovery.**
   - `executeDependencyBatch` and the counted same-note path pass no `recovery`.
   - `applyDependencyEdit`'s recovery index covers only the loaded notes, so any task in
     them that links to an unloaded note defers recovery ("Still blocked").
   - Today's daily note is never loaded, so a dependent linked from today's Pomodoro
     recovers to Ready instead of Next/In Progress.
   - Fix: build recovery per the parent plan's nav-model step 5:
     `buildScheduledRecoveryIndex` over the vault snapshot with the edited buffer
     overriding, including today's daily note.
6. **Counted vault add/remove uses stale line numbers.**
   `applyVaultCountedDependencyRef` walks the targets top-down, so the first inserted
   Depends-On line shifts the rest. Walk bottom-up, as
   `deleteCountedDependencyLinesAndFields` does.
7. **One gesture, one undo step.** `chooseTaskDependency` and `confirmSingleBlockId`
   write the same-note target id with a separate `replaceEditorLine` before the
   planner's transaction. Fold it into the one transaction.
8. **Staleness and smaller gaps.**
   - Counted CURRENT rows lack a `line`, so removing a fully linked same-note
     prerequisite refuses or throws.
   - The `applyDependencyEdit` commit doesn't re-check that the editor still matches the
     content it read before its awaits. Refuse and report "changed — reopen" if it
     doesn't.
   - `removeDependency` must re-read the dependent and refuse when the target isn't on
     its line (contract §9), instead of returning `ok`/"unchanged" with a generic
     notice.
9. **Tests.**
   - Plugin-level tests with async vault/editor stubs covering: a cross-note add with
     preparation; a dependent with an existing link to an unloaded note (add, remove,
     and remove a deleted target); a subfolder target with a duplicate basename; the
     same-note marked-batch `＋ id` with one undo group; recovery for a Pomodoro-linked
     dependent; a counted vault add over two siblings; a stale editor between await and
     commit.
   - Cover `executeDependencyBatch`, `confirmBatchBlockId`, and
     `commitMarkedDependencies`, which no test touches today.
10. **Finish.** Bump the manifest, update the README row, and deploy.

## Phase: nav-mirror-gestures — mirror, gestures, legacy removal

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`. Builds on `nav-writer-fix`.

1. **Mirror trigger.**
   - The mirror hooks Obsidian's `editor-change` and reads `change.from.line`. That
     event passes `(editor, info)` with no change object, so the edited line is always 0
     and the mirror only ever acts on the task owning line 0.
   - Re-implement it as a CM6 `EditorView.updateListener` (via
     `registerEditorExtension`) that reads the real changed ranges.
   - Skip while `view.composing` (IME) or while a modal is open.
   - Use the contract's debounce (about 400 ms; today it is 350).
2. **Ownership.**
   - The scheduler passes the post-edit content as `oldContent`. Deleting a Depends-On
     line that is its task's last child therefore makes the next sibling look like the
     owner, and a repro cleared and unblocked the sibling's field.
   - Re-arming on the same line index after vim `dd` resets `removedText`, which drops
     the deletion.
   - Fix: capture the pre-change content and owning task at the first change, then map
     positions through later changes. Run once the cursor has left the line.
3. **Status effects.**
   - The touch path calls the writer with `add: []` and `remove: []`. Blocking fires
     only for `add` rows, and recovery needs `removedCount > 0`. So adding an open
     prerequisite by hand doesn't block, and removing the last open prerequisite (while
     a closed one stays) doesn't recover.
   - Diff the old field against the new line and apply the same status effects as the
     stage, except commitment transfer.
4. **Scope.**
   - Project the field even when the line has cross-note links (this needs
     `nav-writer-fix` item 2).
   - Never write cross-note targets' `[id::]` (the hooks handle them).
   - Skip malformed lines.
   - Never stamp freshness: drop the stamp in the touch path
     (`stampDependencyParentLine`) and the explicit stamp in the clear path.
5. **Ctrl+D.**
   - A field-only task (a field but no line) gets no ADJ-8 recovery, because `remove` is
     empty. The test near `scripts/test-navigation-hotkeys.cjs:3078` asserts this bug;
     fix the code and the test.
   - Counted Ctrl+D (`deleteCountedBulletPropertyValue`) runs one transaction and one
     notice per target. Make it one transaction with one summary notice.
6. **`!` / `N!`.**
   - Counted `N!` silently skips Depends-On lines. Show the refusal notice, as bare `!`
     does.
   - Use the parent plan's copy:
     `⛓ Dependencies use plain links — edit them with Ctrl+Shift+P`.
7. **Delete the legacy leftovers** (keep the R8 legacy **readers**):
   - `scripts/migrate-task-dependency-identities.mjs`,
     `scripts/test-task-dependency-identity-migration.cjs`, its `package.json` test
     entry, the README layout line, and the README "Dependency identity migration"
     section. The id-encoding explanation already lives in bob-cli `docs/dataview.md`
     and `docs/task-dependencies.md` §3.
   - Dead code: `normalizeDependencyNavigationBlockIds`,
     `formatDependencyNavigationBulletFromDetails`,
     `reconcileDependencyNavigationBullets`, `parseRecoveryTransclusion`, and the legacy
     branch in `setLocalTaskDependency`.
   - Then, once they have no callers: `planDependencyNavigationBulletSync`,
     `applyDependencyNavigationBulletSyncPlan`, `applyDependencyNavigationPlanToLines`,
     and `transformDependencyBulletsInContent`.
   - Delete or update the tests that exercise them.
   - Fix the "retired" comment in `scripts/migrate-dependency-lines.mjs`.
   - In the README, list `scripts/migrate-dependency-lines.mjs` in the layout and
     replace the deleted section with a short "Depends-On line migration" usage note
     (`--vault`, `--write`, dry-run by default).
8. **README row.** Make its claims true: Ctrl+D's immediate recovery, and the bare and
   counted `!` notice.
9. **Tests.**
   - A half-typed `[[` (a malformed line, no write).
   - Deleting the whole line, including as the task's last child, with a following
     sibling.
   - Adding an open prerequisite by hand (blocks it).
   - Removing the last open prerequisite (recovers it).
   - Scheduler tests: debounce, the cursor leaving the line, vim `dd`, IME composition,
     and modal open.
   - Field-only Ctrl+D, and counted Ctrl+D as one undo group.
10. **Finish.** Bump the manifest, update the README row, and deploy.

## Phase: nav-stage-polish — the Depends on stage

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys`. Builds on
`nav-mirror-gestures`.

1. **Ranking** (contract §6):
   - The tie order compares the lane before `#hide`, so a hidden In Progress task
     outranks a visible Ready one. `#hide` ranks last.
   - The empty query lists Ready tasks from other notes. Show only what the design
     lists: CURRENT, same-note tasks, then In Progress and Next.
   - Add the "type to search N more" row when the ~60-row cap truncates.
2. **Layout** (parent plan Design "The Depends on stage"):
   - Add the `🔒 waits on N` badge, the cycle path in the disabled row's tooltip, and
     muted `#hide` rows.
   - Use new `bob-cnp-dep-*` classes in `styles.css` with `--task-status-*` colours.
   - Badges: `−`, `＋`, and `＋ id`.
   - Short disabled reasons: `would create a cycle`, `this task`, `path can't be an id`,
     `changed — reopen`.
   - Footer: `↑↓ navigate · ⇥ mark · ↵ toggle · esc dismiss`, with ↵ reading `apply N`
     when rows are marked.
3. **Stale rows.** A stale target or dependent refuses and then reopens the stage fresh
   (it only refuses today).
4. **Notices.** Removal notices name the task's description, not its block id.
5. **Task Link mode batches.** Batches still apply the old `dependsOn` property filter
   and the "Dependencies cannot be set through a Task Link" refusal. Remove both, as the
   single path already does.
6. **Tests**, using `createBulletPropertyPickerHarness` / `createLinkPickerHarness` with
   a stubbed Tasks plugin:
   - the stale guard and reopen;
   - `＋ id`;
   - Esc changes no bytes;
   - one undo group;
   - counted;
   - a cycle created by a batch;
   - `edit-task-dependencies` registered with no hotkey;
   - Task Link batches;
   - `#hide` ordering and empty-query order;
   - a performance check: filtering 1,000 synthetic tasks takes under 16 ms per
     keystroke.
7. **Finish.** Bump the manifest, update the README row, and deploy.

## Phase: docs-memory-fix — decision record and stale docs

**Repos:** bob-cli (docs, memory) and bob-plugins (README). Use `/sase_memory_write` for
any memory file. Mirror the existing strands' shape, then run `sase memory init` and
confirm `sase memory init --check` is clean.

1. **Decision record `decisions:task-deps-are-depends-on-links`.** These are corrections
   to the bob-cli-3n publish step, which the approved parent plan specified. They are
   not a course change. If `/sase_memory_write` routes a correction elsewhere, follow
   its routing.
   - Add the missing rejected alternative "`!` as the gesture" (the parent plan lists 8;
     the record has 7).
   - Fix `decided: 2026-10-03` to the local decision date `2026-10-02`.
   - Add to Evidence the bob-plugins commits `1831db4` (chips), `e7baeb5` (compat),
     `5194bc8` (nav-model), `08d1560` (nav-stage), `82aec34` (nav-gestures), `46ddd1e`
     (migration script), the vault migration commit `66c6d47c`, and this epic's fix
     commits.
2. **Stale bob-cli docs:**
   - `docs/projects.md`: the Schedule Log example (around line 519) still shows a
     dependency as a legacy embed child `- ![[#^blocked-by-this]]`. Around line 279,
     `dependsOn` is described as a plain inline field; it is now derived from the
     Depends-On line.
   - `docs/freshness.md` (around line 282): "the `!` transclusion toggle when it
     rewrites the parent task line" is stale, because `!` is a pure toggle now.
   - `docs/capture.md` (around line 1303): say that the recursive `![[…#^id]]` close
     skips Depends-On lines, so closing never closes prerequisites.
   - `src/native/plan_budget/today.rs` (around line 105): replace the "Transcluded
     dependencies never inherit Today" comment with the Depends-On wording publish used
     in `docs/plan.md`.
   - Nits: call the picker row "Depends on" (contract §6.1) where docs say "dependsOn
     row".
3. **Contract clarifications** (`docs/task-dependencies.md`):
   - DC7/DC8 count broken and non-task links as "waiting" in the chip summary, while
     R4/R5 say those never block, so a Ready task can show "waiting on 1". Decide which
     is intended, make DC7/DC8 and §7 say it explicitly, and fix whichever
     implementation then disagrees (ledger-tools `dependencyChipModel`) in this phase.
   - State in §3/§4.3 that archive (`done/`) links keep the explicit `done/` path form.
     That is the only form the hooks re-resolve (`docs/task-status-hooks.md`), and it
     binds the JS canonicalisers too. Fix nav's `canonicalDependencyLink` if it violates
     this.
4. **bob-plugins README.** Line ~14's "sub-task dependency transclusions are refused" →
   wording that matches the new contract (legacy transclusion children are still
   refused).
5. **Finish.** Run `just all` in bob-cli. If any plugin changed, run `npm test`,
   `npm run validate`, and deploy it.

## Phase: rollout — fleet reinstall and real-vault dry run

1. **This host.**
   - From an up-to-date bob-cli master, run `cargo install --path . --locked --force`.
   - Pull the opened bob-plugins checkout to origin master, then run
     `bob plugins sync -n -r "<opened bob-plugins path>"`.
   - Check with `bob plugins list`. Expect 0 drift and the new versions of
     bob-navigation-hotkeys, bob-ledger-tools, task-status-cycler, and block-id-prompt.
2. **Real-vault dry run.** `bob task-status-hooks --dry-run -f json` against `~/bob`.
   Paste every dependency count and warning kind into the phase notes, and explain every
   non-zero projection, adoption, heal, or canonicalisation: these are the writes the
   Mac cron will make. Never run a live pass.
3. **apollo** (or athena, whichever this isn't): `git pull --ff-only`, then
   `cargo install --path . --locked --force`, and sync the plugins from its bob-plugins
   checkout. It runs no hooks.
4. **MacBook** (best effort; `ssh -o ConnectTimeout=15 mac`, retrying for about 10
   minutes). The hooks are now safe to deploy there.
   - Find how `bob` was installed (`~/.cargo/.crates.toml` / `.crates2.json`):
     - **a clean `path+file://` checkout on `master`:** `git pull --ff-only`, then
       `cargo install --path . --locked --force`;
     - **a `git+` install:** reinstall from the same source with `--locked --force`;
     - **anything else:** stop and record the exact steps.
   - Pull and sync its bob-plugins checkout.
   - Verify with
     `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json' | jq 'has("dependency_projection_updates")'`.
5. **Record** in the phase notes, per machine: bob commit, hooks capability, and plugin
   versions. List exactly what is left for Bryan: the MacBook steps if it was
   unreachable, reloading the four plugins in each running Obsidian, and the parent
   plan's pilot checklist (add prerequisites from two projects, remove a completed one,
   follow a chip, edit while another note has unsaved changes; optionally bind a chord
   to **Edit task dependencies**).

## Deliberately not doing

- The deferred features already filed as tasks:
  - retiring the R8 legacy readers (bob-cli-3o, time-gated);
  - the reverse Blocks… stage (bob-cli-3p);
  - the task-line mini-badge (bob-cli-3q);
  - the v2 path codec (bob-cli-3r).
- The capture_pomodoros parallel flake (bob-cli-2e).
- The cycler's Alt+] `finalizeClosedTasks` gap for ordinary tasks (bob-cli-3k).
- Any vault content edit or live hooks pass.

## Risks

| Risk                                                                    | Mitigation                                                                                                          |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| The Mac gets the buggy hooks before `hooks-correctness` lands           | bob-cli-3n carries a WARNING note; only `rollout` updates the Mac, and it depends on the fixed hooks                |
| Async writer changes reintroduce partial writes                         | Preparations awaited before one commit; stale preimage re-check; async plugin-level tests                           |
| Recogniser alignment changes what renders or refuses                    | New numbered DP vectors land in the contract first and are copied into every suite                                  |
| The fixed field writer rewrites many vault lines on the first cron pass | `hooks-correctness` and `rollout` dry-run the real vault and explain every non-zero count before the Mac is updated |
| Parallel plugin phases conflict in `package.json` / README              | Keep both sides of single-line conflicts; same-plugin phases are sequential                                         |
