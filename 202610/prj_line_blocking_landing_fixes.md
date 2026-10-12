---
tier: epic
title: Finish landing ^prj project blocking — fix the defects the land audit found
goal:
  "Every defect that bob-cli-6u (project blocking from the ^prj line, #prj_is_blocked,
  and [unblocked::]) introduced is fixed in bob-cli and bob-plugins. bob-cli `just
  check` fails only the bob-cli-6m percent-listen tests. bob-plugins `npm test` fails
  only the known pre-epic failures (bob-cli-6v, bob-cli-6w, plus the bob-cli-3w timing
  flake). The fixed bob and plugins are installed on the MacBook and on this host."
phases:
  - id: cli-reconcile-close
    title: Reconcile and close-path fixes plus the two epic-caused CLI test failures
    depends_on: []
    size: medium
    description:
      "cli-reconcile-close: fix the two failing CLI fixtures, strip #prj_is_blocked from
      every resident when ^prj recovers (PDC3), guard internal ^prj targets on the
      adoption/legacy paths, stop marking status-only flips structural, recurring and
      CRLF fixes, the direct-link ^prj rule, dead code, and missing reconcile tests."
  - id: cli-projects-sync
    title: Projects sync hoist, Sub-projects placement, and inline ^prj schedule issues
    depends_on:
      - cli-reconcile-close
    size: small
    description:
      "cli-projects-sync: gate the hoist on an open ^prj, insert Sub-projects after the
      whole Cancel Log / Depends-On subtree, match only direct Depends-On children,
      report invalid or duplicate inline ^prj dates, skip fenced lines, tag-helper
      tests."
  - id: cli-capture
    title: Capture link-guard, & on ^prj, and output-compatibility fixes
    depends_on: []
    size: medium
    description:
      "cli-capture: restore pull-forward for residents of unblocked projects, stop
      claiming links in existing capture JSON fields, resident cycle edges and id-less
      residents on & to ^prj, never raise Blocked seeds, struck links, `+` residents,
      keep-link consumption, and no whole-vault read for non-Blocked link targets."
  - id: nav-fixes
    title: Navigation-hotkeys project-blocking fixes
    depends_on: []
    size: medium
    description:
      "nav-fixes: resident recovery only from a blocked ^prj, resident cycle edges when
      ^prj gains a prerequisite, tag drift in schedule and move paths, [unblocked::] and
      emptied-entry cleanup on counted/batch paths, conversion and move gaps, lenient
      frontmatter scheduled, out-of-scope skip flags, docs."
  - id: cycler-blockid
    title: Task-status-cycler and block-id-prompt project-blocking fixes
    depends_on: []
    size: small
    description:
      "cycler-blockid: strip the tag from every resident when the project recovers,
      recover without [unblocked::] when ledger freshness v12 is missing, pass the
      out-of-scope skip flags, collapse redundant inline-date checks, block-id-prompt
      refusal scope and order, README rows."
  - id: rollout
    title: Install the fixed bob and plugins on the MacBook, then this host
    depends_on:
      - cli-reconcile-close
      - cli-projects-sync
      - cli-capture
      - nav-fixes
      - cycler-blockid
    size: small
    description:
      "rollout: bob on the MacBook first, then this host, then plugins on both; dry-run
      verification only; record versions and what Bryan still has to do."
decisions:
  direct_prj_link:
    ask:
      Should a ^prj with its own live Task Link today rise to Next like any linked task?
    choices:
      raise:
        ADJ-11 limits only dependency promotion; a directly linked ^prj rises to Next
        everywhere
      never:
        No engine raises ^prj; the close path changes to recover a linked ^prj to Ready
    default: raise
    why:
      6u Design §2 scopes ADJ-11 to ^prj targets; the close path already lands a linked
      ^prj in Next
    answer: raise
proposed_by: bbugyi200.apollo.bob-cli-6u.land
parent_bead: bob-cli-6u
decided_by: auto
create_time: 2026-10-12 18:07:05
status: wip
---

- **PROMPT:**
  [prompts/202610/prj_line_blocking_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prj_line_blocking_landing_fixes.md)
- **PARENT:**
  [202610/prj_line_project_blocking.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/prj_line_project_blocking.md)

# Plan: finish landing ^prj project blocking (bob-cli-6u remaining work)

## Context

Epic **bob-cli-6u** (`plan:202610/prj_line_project_blocking.md`, "the 6u plan") moved
project dates and prerequisites onto the `^prj` task line. It added the machine-owned
`#prj_is_blocked` tag and the `[unblocked::]` freshness input. All 17 phases are closed
and deployed. The land agent's audit of every phase commit found defects that the epic
itself caused. This plan fixes them. The 6u plan's Design, its contract docs, and its
DECISIONS still govern:

- `tag_scope = every_resident`: every open resident of a blocked project carries exactly
  one `#prj_is_blocked`. No other open line carries it, `^prj` never does, and closed
  lines are never edited.
- `unblock_keeps = yes`.

**Read first, in this order:**

1. The 6u plan (`sase artifact read plan:202610/prj_line_project_blocking.md "<why>"`).
   Read Design §§1–6 and the phase section your phase fixes.
2. `docs/task-dependencies.md` §13 (PD/PDT/PDC/PDL/PDG vectors).
3. `docs/freshness.md` (UB vectors, WRITE/CLEAR/READ rules).
4. Memory: `decisions:projects-block-from-prj-line` and
   `decisions:unblocked-tasks-resurface-as-ticklers`, through `/sase_memory_read`.

**Every finding below is an audit finding with approximate line numbers.** Reproduce or
confirm it before you fix it. When one turns out wrong or already fixed, skip it and say
so in a bead note on your phase. Add a regression test for every fix, citing the vector
or rule it enforces.

### Commits being fixed

| Repo        | Commit    | 6u phase                     |
| ----------- | --------- | ---------------------------- |
| bob-cli     | `f6fbd7d` | cli-model (6u.2)             |
| bob-cli     | `f51ba28` | cli-freshness (6u.3)         |
| bob-cli     | `23f2fd9` | cli-projects (6u.4)          |
| bob-cli     | `34d4ab6` | cli-reconcile-block (6u.5)   |
| bob-cli     | `2340689` | cli-reconcile-consume (6u.6) |
| bob-cli     | `13d183f` | cli-close (6u.7)             |
| bob-cli     | `66c4ddb` | cli-capture-deps (6u.8)      |
| bob-cli     | `36f997b` | cli-capture-links (6u.9)     |
| bob-plugins | `f47be84` | ledger-freshness (6u.10)     |
| bob-plugins | `c9938d8` | nav-model (6u.11)            |
| bob-plugins | `a42f45d` | nav-schedule (6u.12)         |
| bob-plugins | `046028c` | nav-deps-stage (6u.13)       |
| bob-plugins | `1abde2f` | nav-convert-move (6u.14)     |
| bob-plugins | `744247c` | plugins-compat (6u.15)       |

## Cross-cutting rules for every phase

- **bob-cli:** run `just check`. The only allowed failures are the two `capture::r#ref`
  percent-listen tests tracked by **bob-cli-6m**. After `cli-reconcile-close`, the
  blocked.rs and schedule.rs fixtures are not baseline failures. Keep touched Rust files
  under about 1500 lines where practical; put new code in new modules. Before touching
  any `--help` text, read `sase/memory/cli_rules.md` through `/sase_memory_read`. Update
  `README.md` and `docs/*.md` in the phase that changes behavior.
- **bob-plugins:** open it with `sase repo open bob-plugins -r "<why>"` and read its
  `AGENTS.md`.
  - Edit `src/` fragments only; hand-edited fragments stay at or under 1000 lines, so
    new code goes in new fragments registered in `src/fragments.json`.
  - Run `npm run build`, then `npm test` and `npm run validate`. Add every new test file
    to `package.json`'s `test` script.
  - The allowed `npm test` failures are 5 pre-epic failures: the 4
    `test-ledger-tools-unblocked-glyph.cjs` cases (**bob-cli-6v**) and the decision-card
    "Keep counts once" case (**bob-cli-6w**). The **bob-cli-3w** stage-ranker timing
    flake may also appear.
  - Bump the touched plugin's `manifest.json` minor version and its README row.
  - Deploy on this host only, with
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`. Never sync
    plugins on the MacBook; that is `rollout`'s job.
  - `nav-fixes` and `cycler-blockid` touch different plugins and may run in parallel.
    When rebasing `package.json` or the README, merge both sides.
- **Plugin phases own the bob-cli docs for their keymaps** (`docs/projects.md`,
  `docs/task-dependencies.md` §§6/8/13). Commit both repos.
- **Hooks safety:** never run a live `bob task reconcile` or `bob projects sync` against
  `~/bob`. `BOB_DIR=~/bob bob … --dry-run -f json` is always fine. Only `rollout`
  touches the MacBook.
- **Capture JSON is additive only** (`decisions:mac-capture-is-a-thin-client`). No
  existing field may change meaning.
- **Memory:** no memory note is edited by this plan.
- **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on your own phase bead; never
  create beads.

## Phase: cli-reconcile-close

**Repo:** bob-cli: `src/native/task_status_hooks/`, `src/native/task_complete/`, and
`src/native/freshness/`.

1. **The two epic-caused CLI failures.** Both were mislabeled "wall-clock rot" in the 6u
   bead notes. They are the 6u.4 × 6u.5 integration: 6u.4 made the `^prj` inline date
   block `^prj`, and 6u.5 made a blocked project block, and tag, its residents.
   - `tests/cli/projects/schedule.rs`
     `projects_sync_then_task_status_hooks_blocks_and_recovers_propagated_tasks` (~131):
     after reconcile, the resident is
     `- [?] #task Implement milestone [scheduled:: 2026-07-17] #prj_is_blocked ^milestone`.
     Assert that, and assert that the tag is stripped once the date matures (~151).
     Check the sibling
     `project_schedule_tasks_flip_between_dash_and_blocked_queries_when_due` for the
     same shape.
   - `tests/cli/task_status_hooks/blocked.rs`
     `task_status_hooks_reconciles_future_schedules_and_combined_blocking_reasons`
     (~133-260). `tasks.md` is a `type: [[project]]` note whose `^prj` carries a future
     inline date, so every resident is now Blocked (11 marked, not 7). Keep this test
     about a task's own reasons:
     - drop the project frontmatter from `tasks.md`, along with its `^prj` line;
     - move the project case into its own note in the same vault (`type: [[project]]`,
       `status: wip`, leftover frontmatter `scheduled: 2099-01-01` that reconcile
       ignores, a `^prj` with a future inline date, and one resident);
     - assert `^prj` gets `local_schedule` only, the resident gets `project_schedule`
       plus the tag, and the summary counts.
2. **PDC3 tag strip on the close path.** When a `^prj` recovers and its project has no
   remaining reason, every open resident loses `#prj_is_blocked`. That includes
   residents that stay `[?]` for their own reasons and residents that are not `[?]`.
   Today `task_complete/project_close.rs` `recover_project_members` (~118-181) only
   touches recovering residents, and `recovery.rs` (~136-145) has the same gap. Emit
   tag-strip line edits for the rest (closed lines are never edited). Extend the PDC3
   tests in `successor_tests.rs` (~938) and `recovery_tests.rs` (~213) with a tagged
   resident that keeps its own prerequisite.
3. **Internal-target guard on every path.** Today only the accepted-line path
   (`reconcile.rs` ~832-856) keeps same-note `^prj` links out of `[dependsOn::]`. R2
   adoption (`plan_adoption`, ~1338+) and legacy children (`plan_legacy`, ~593) let a
   field-only same-note id through. That blocks the project on a resident (deadlock),
   tags the residents, and consumes their links until the next run. Apply the same rule
   on both paths, with the `project_internal_dependency` warning. Also make the
   project-reason inputs in `sync.rs` (~496-499) and the close path's
   `project_states_for_files` filter same-note ids instead of assuming the field is
   clean.
4. **Status-only flips are not structural.** `sync.rs` (~689-692) marks a file
   structural whenever `new_line` is `Some`. `plan_task_line_rewrite` returns `Some` for
   every status change, so every plain flip now waits out the quiet period
   (`preflight.rs` ~17-70). Mark a file structural only when the rewrite differs from
   the plain status-byte swap: a tag edit, an `[unblocked::]` write, or a prune.
5. **Recurring rows.** `line_is_recurring` (`sync.rs` ~993-995) accepts only `🔁` and
   `[repeat::`. Reuse the shared recurring check, which also accepts `(repeat:: …)`, so
   UB5's skip holds.
6. **CRLF in `recovery.rs`** (~158-181). Writing `[unblocked::]` through
   `mark_unblocked` drops the `\r`. Preserve it the way `successors.rs` `edit_line`
   does.
7. > [!decision] direct_prj_link = raise
   >
   > `references.rs` (~633-643) sends every `^prj` seed in `direct` to `propagated`. So
   > a `^prj` with its own live link today is never raised to Next, and a `[?]` `^prj`
   > whose last reason clears while its link is kept recovers to `[ ]`. The close path
   > lands `*`. Limit ADJ-11 to `^prj` targets reached through dependency edges: a
   > directly linked `^prj` is desired at Next like any linked task, and the walk still
   > continues through it. Reword `docs/task-dependencies.md` §13.4: "Dependency
   > promotion never raises a `^prj` target…" and drop "the one path that makes `^prj`
   > Next", stating that a direct link raises it like any task. Add a reconcile test and
   > a parity assertion against the close path.

   > [!decision] direct_prj_link = never
   >
   > Keep reconcile as is. Change the close path (`successors.rs` ~1077) so a linked
   > `^prj` recovery lands `[ ]`, not `*`. Document in §13.4 that nothing in reconcile
   > or the close path raises `^prj`, and add a parity test.

8. **Reporting nits.**
   - `normalize_prj_is_blocked_tag` moves a misplaced existing tag and counts it under
     `prj_is_blocked_tags_added`. Count only real additions.
   - `removed_empty_pomodoros` mixes first-pass (original) and consumption-pass
     (composed) line numbers. Report both passes in original-text line numbers, or say
     in `docs/task-status-hooks.md` which text each entry refers to.
9. **Dead code.**
   - `freshness/placement.rs` (~284): `clear_unblocked` keeps
     `#[cfg_attr(not(test), allow(dead_code))]` with no production caller. Delete it, or
     use it where a writer clears the field.
   - The `UnblockScope` doc comment is duplicated (~94-98).
   - `task_status_hooks/structure.rs` (~176-345): `plan_structural_changes_with_blocked`
     is only called with an empty Blocked set. Remove the dead blocked branches, or the
     parameter.
10. **Missing reconcile tests** from 6u's cli-reconcile-block step 9:
    - the Databricks `#ref` fixture (both `#ref` residents go `[*]` → `[?]` plus the tag
      and regroup under `### Blocked`);
    - PDB5 (sub-project not cascaded);
    - PDB6;
    - PDB8;
    - the PDR8 cycle warning;
    - PDT10 (CRLF tag rewrite).
11. **Finish.** `just check`. Also run
    `BOB_DIR=~/bob <new build> task reconcile --dry-run -f json`, and list every planned
    change in your bead note, explaining any non-trivial one.

## Phase: cli-projects-sync

**Repo:** bob-cli, `src/native/projects/`. This phase runs after `cli-reconcile-close`
because both touch `tests/cli/projects/`.

1. **Gate the hoist on an open `^prj`.** `sync.rs` (~524-530) pushes `HoistScheduled`
   for any non-terminal project, even one with no `^prj`. `prj_line_edit` (`edits.rs`
   ~55-57, ~398) then fails ("failed to locate ^prj task"), the whole note is skipped,
   and sync exits 1. Dry-run still prints "would hoist". Hoist only when the `^prj`
   state is Open. Otherwise leave the key in place and report a per-file warning. Add a
   CLI test.
2. **Sub-projects placement.** When there is no Depends-On line,
   `subprojects_anchor_child` (`sync.rs` ~844-856) anchors on the `❌ **CANCEL LOG**`
   line itself. The marker is then inserted at `anchor.next_start` (`edits.rs` ~494),
   between the log line and its nested entries. Insert after the anchor's whole subtree,
   for both the Cancel Log and the Depends-On line. Also make `is_depends_on_child`
   (~828) match only a **direct** child line of `^prj` that starts with the Depends-On
   marker; today a nested task's own Depends-On line, or prose, can match. Extend
   `projects/tests/edits.rs` (~149) with a Cancel Log that has entries.
3. **Per-line inline `^prj` schedule issues** (6u plan, cli-projects step 3). `scan.rs`
   (~139-147) silently turns an invalid or duplicated inline `[scheduled::]` on `^prj`
   into `None`. Report it as a project issue, matching the old frontmatter
   "multiple/invalid scheduled" issues. Make `issues_allow_subproject` (~387) match the
   message actually emitted; today it keys on a `"^prj scheduled "` message that nothing
   emits. Confirm `bob projects list` then validates it, as its help text (`mod.rs`
   ~69-71) claims.
4. **Fenced lines.** `prj_line_edit` (`edits.rs` ~371-378) must skip fenced lines, so a
   fenced `^prj` example above the real line never receives the hoisted date.
5. **cli-model leftovers.**
   - `project_dependencies/blocking.rs` (~20-21, 97, 104) matches exactly `"prj"`, while
     `capture_language::is_reserved_project_task_id` is case-insensitive. Align the two,
     or put the reason exact is right in a comment there.
   - Make the tag-helper tests (`projects/tests/edits.rs` ~79-101) cite the PDT vectors
     and add a CRLF case.
6. **Docs:** update `docs/projects.md` Sync Rules for the hoist gate and the issue.
7. **Finish.** `just check`, and a read-only
   `BOB_DIR=~/bob <new build> projects sync --dry-run`: expect no hoists and no errors.

## Phase: cli-capture

**Repo:** bob-cli, `src/native/capture/` (`dependencies.rs` is over 2700 lines, so new
code goes in new modules).

1. **HIGH — pull-forward regression.** `link_guard.rs` (~209-210) computes
   `only_own_date = own_scheduled.is_some() && own_open.is_empty() && !is_resident`.
   `is_resident` is true for every open task in any recognised project, blocked or not.
   So a task in an **unblocked** project whose only reason is its own future date is
   consumed ("not linked") instead of pulled forward. That breaks PDG2 for `@r+id!`,
   `@r:id`, ensure-next, and link-then-close, where the task now stays `[?]`. Gate on
   `own_open.is_empty() && inherited_open.is_empty() && prj_scheduled.is_none()`. Add
   tests for a resident of an unblocked project at each of the four callers.
2. **Existing JSON fields must not claim a link that did not happen.** Today:
   - `pomodoro_link.rs` (~327) emits a new value, `pomodoro_link_action: "not_linked"`;
   - a consumed `@r+id!` keeps `toggle_direction: "link"` and `block_link`
     (`task_toggle.rs` ~176);
   - a consumed ensure-next drops `pomodoro_link_action`, which it always reported
     before (`ensure_next.rs` ~82).

   Open bob-mac-capture read-only (`sase repo open bob-mac-capture -r "<why>"`) and read
   how it renders these fields: `CapturePomodoroLinkPresentation.swift` (~94-106,
   149-156) and `CaptureTogglePresentation.swift` (link branch). Then choose the
   bob-side output so the **current** Mac app never shows a false "Linked" or
   `+ [[link]]` confirmation, without changing any existing field's meaning. For
   example, omit the link-only fields when nothing was linked, and carry the outcome
   only in the additive `link_guard` object. Document the shapes in `docs/capture.md`.
   **bob-cli-73** tracks native Mac rendering of `link_guard`; note on that bead which
   fields to read.

3. **`&` on `^prj` misses resident cycles** (`dependencies.rs` ~2118-2127,
   `reject_new_cycles` ~1307). The extra edges add only `prj → new target`. Also add
   every open resident → new target (the implicit edges), so `@proj+prj &ext:tee` is
   refused when `tee` depends on a resident. Add the test for that direction.
4. **Id-less residents.** `project_deps.rs` (~135-137) `continue`s on a missing block id
   before the edit, so those residents are never flipped to `[?]` or tagged. Flip and
   tag every open resident; the block id is needed only for link removal.
5. **Promotion never raises a Blocked seed.** `dependencies.rs` (~1441-1450) rewrites a
   `[?]` or future-scheduled seed to `[*]`/`[/]`. 6u made this reachable through
   consumption seeds (`link_guard.rs` ~234-245) and the `^prj` commitment transfer.
   Mirror reconcile: an effectively Blocked task is never raised, and the walk continues
   through it to its own open prerequisites. Test: `^prj` depends on X, X is `[?]`
   waiting on Y; linking a resident raises Y, not X.
6. **Struck links survive.** `&` consumption uses `capture_task_toggle/links.rs`
   `plan_link_removal` (~336), which matches by substring and so also removes a struck
   `~~[[…]]~~` link (PDL5). Remove only live, unstruck links.
7. **Consume also removes existing links.** The gesture rule covers a link the gesture
   would create **or keep**. The consume branches in `ensure_next.rs` (~35-90) and
   `pomodoro_link.rs` (~289-330) leave the Blocked target's existing live links in
   today's open Pomodoros. Remove them in the same write (then the empty-Pomodoro rule).
8. **`+` with a future date blocks every resident.** `capture_project_note.rs` (~366)
   blocks only `:` tasks; `^` and plain tasks in the new blocked project land Ready and
   untagged. Per `every_resident`, every open task the gesture writes into the note
   lands `[?]` plus the tag, and only `:` tasks are additionally unlinked. Update the
   test at ~839.
9. **Fenced-line indices.** `is_resident` checks pre-write fenced line numbers against
   post-write tasks (`dependencies.rs` ~2026-2032, ~2276-2283; `project_deps.rs`
   ~124-152). An inserted Depends-On line shifts them by one. Recompute the fenced set
   from the text being edited.
10. **No whole-vault read for a non-Blocked link target.** `assess_link_target` builds
    the full `DependencyGraph` (a prefiltered read of every vault note) on every link
    gesture and every capture into a project note. That undoes much of **bob-cli-5v**
    for Bob Mac Capture live previews. First assess the target from its own note:
    - if the target line is not `[?]` and has no `[dependsOn::]`, no future
      `[scheduled::]`, and no unchecked Depends-On links;
    - and, when it is a resident, `^prj` has none of those either;

    then return `Link` without building the graph. Add a test that counts reads, or
    asserts that the graph is not built, mirroring bob-cli-5v's tests.

11. **Cosmetic:** the doc comment of `print_human_link_toggle_summary` sits on
    `print_human_link_guard` (`output.rs` ~1578-1583).
12. **Finish.** `just check`. Confirm in your bead note that every capture JSON change
    is additive and list the fields.

## Phase: nav-fixes

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src`. Docs go in bob-cli
(`docs/task-dependencies.md` §§6/8/13, `docs/projects.md`).

1. **Removal recovers residents only from a blocked project.**
   `216-project-dependency-stage.js` `shouldRecoverProjectResidents` (~305-322) runs
   whenever the last projected reason clears. But the vault is loaded only when `^prj`
   is `[?]` (`020` ~534), and without the vault the recovery check skips unknown ids
   (`140`: `if (!matches) continue`). So removing a closed link from an unblocked `^prj`
   flips residents that are Blocked by their own prerequisites to `[ ]`, and writes
   `[unblocked::]`. Recover residents only when `^prj` was `[?]` before the gesture, and
   only residents that carry the tag. Test it.
2. **Cycle guard with implicit resident edges** when `^prj` gains a prerequisite. Both
   guards (`300` ~203, `440` ~265-289) add only `^prj → X`. Also add resident → X for
   every open resident, so X depending on a resident is refused.
3. **Tag drift (every_resident):**
   - **schedule:** `265-project-schedule-inline.js` (~170, ~198, ~342) strips the tag
     from every resident when the date becomes due or is deleted, even while `^prj`
     stays blocked by an open Depends-On link. Strip only when the project has no
     remaining reason.
   - **move/convert:** `250` (~176-191) and `665` (~605-619) strip the tag from closed
     lines. Never edit closed lines.
4. **Counted and batch `^prj` schedule paths.** Only `550` (~640) passes
   `markUnblockedLine`. These recover residents without `[unblocked::]`, and reconcile
   never adds it later because the rows are already Ready:
   - `560` (~120, counted priority roll);
   - `570` (~390, counted delete);
   - `500` (~655, ~729, ~856, Task Link batch);
   - `070` (~150, recommended roll);
   - counted Depends-on sessions.

   Pass the writer on every path. The Pomodoro link cleanups at `560` (~172, 193,
   470, 491) also need `removeEmptiedEntries`.

5. **Conversion and move gaps:**
   - Forward conversion (`110` ~718-722) treats any `[?]` source as a blocked project.
     Block the new project only when the carried Depends-On line has an open
     prerequisite or the inline date is future. Otherwise `^prj` and the residents land
     by their own reasons.
   - Reversing a project into a blocked parent leaves `^prj`'s live link in today's
     Pomodoro. Cleanup is planned against the new block id (`665` ~717-738) before the
     `#^prj` link is rewritten (~794). Order them so the link is consumed.
   - A move into a blocked project, and forward conversion, delete links without
     promoting the open prerequisites (Design §5; `250` ~249-286). Promote them. Include
     blocked and recovered counts in the Ctrl+Shift+M, inbox-route, and reversal
     notices.
   - `070-recommended-roll-batch.js` `getDependencyInsertLine` (~528-546) inserts at
     `cancel.line + 1`, splitting a Cancel Log from its entries. Insert after its
     subtree.
6. **Lenient frontmatter `scheduled`.** `210` `parseProjectNoteFrontmatter` (~495-551)
   still refuses multiple or invalid frontmatter `scheduled` values. That blocks the
   `^prj` property context and reversal, and sync deliberately leaves invalid values in
   place. Nav no longer uses that key: ignore it.
7. **Out-of-scope skip flags.** Nav's `markUnblockedLine` callers (`260` ~73-76 and the
   wrappers at `580` ~54, ~831, `570` ~692) never pass `isDailyNote` / `isToday`, so a
   daily-note task can get an inert `[unblocked::]`. Pass them. Let the missing-API
   `console.warn` fire when ledger freshness v12 is absent.
8. **Tests** (extend the deps-stage, schedule, move, and conversion suites):
   - Ctrl+D on `^prj`;
   - counted sessions;
   - a plugin-level forward conversion with a Depends-On line;
   - a round trip;
   - an inbox route into a blocked project;
   - every item above.
9. **Docs and comments:**
   - `README.md`: the Task Status Cycler row, which a42f45d overwrote with nav schedule
     text (restore the cycler sentence), and the nav row's behavior text;
   - the stale "project frontmatter" comments at `260` (~164, ~408) and `580` (~207).
10. **Finish:** manifest bump, README row, build/test/validate, and deploy on this host.

## Phase: cycler-blockid

**Repo:** bob-plugins, `plugins/task-status-cycler/src` and
`plugins/block-id-prompt/src`.

1. **Cycler tag strip (PDC3 mirror).** When a project recovers, residents that stay
   `[?]` for their own reasons keep `#prj_is_blocked` (`080-dependencies.js` ~509-520,
   `085`). Every JS path trusts the tag, so Alt+] on such a resident is refused with
   "resolve the ^prj prerequisite" even though `^prj` is Ready. Strip the tag from every
   open resident when the project has no remaining reason.
2. **Missing ledger v12.** `130-plugin-references.js` (~311-321) and
   `088-project-blocking.js` (~50-84) refuse recovery with "update bob-ledger-tools".
   The 6u plan says to recover without the field and `console.warn`. Do that.
3. **Skip flags.** The cycler's `markUnblockedLine` calls (`088` ~28-30) never pass
   `isDailyNote` / `isToday`. Pass them.
4. **Cleanup.** Collapse the redundant inline-date checks (`080` ~247, ~467) per 6u's
   plugins-compat step 1.
5. **block-id-prompt:**
   - `120` (~302-320) shows the date refusal before the dependency report. Report
     dependency reasons first.
   - The `^prj` refusal (`030` ~581) must apply only when `^prj`'s **only** reason is
     its own date.
   - Check whether `070` (~396, 406) raises a Ready `^prj` target to Next. If it does,
     follow `direct_prj_link`'s default: a direct link raises it like any task, and
     dependency promotion never does.
6. **Tests** in the existing `test-task-status-cycler-*` and `test-block-id-prompt-*`
   suites, for every item above.
7. **Finish:** manifest bumps, README rows, build/test/validate, and deploy both plugins
   on this host.

## Phase: rollout

1. **MacBook bob first.** This is best effort: `ssh -o ConnectTimeout=15 mac`, retrying
   for about 10 minutes.
   - In `~/projects/github/bobs-org/bob-cli`, run `git pull --ff-only` then
     `just install`.
   - Verify `bob task reconcile --dry-run -f json` and `bob projects sync --dry-run`
     there. Both should exit 0 with no errors, and the reconcile dry run should plan
     only changes you can explain.
2. **This host.** From up-to-date bob-cli master, run `just install` (never pass
   `--force`). Pull the opened bob-plugins checkout to origin master and run
   `bob plugins sync -n -r "<opened bob-plugins path>"`. Check `bob plugins list`.
3. **MacBook plugins, only after step 1 verifies:** `ssh mac 'bob plugins sync'`, then
   `ssh mac 'bob plugins list'`. If bob could not be updated on the Mac, do not sync its
   plugins; record exactly what is left.
4. **No vault edits** and no live reconcile or sync from this phase; the Mac cron runs
   them.
5. **Record** in a bead note: the bob commit on each machine, the plugin versions, and
   what is left for Bryan. He reloads the plugins (or restarts Obsidian) on the Mac,
   then pilots:
   - Ctrl+Shift+P → `b` on a `^prj`, add and remove;
   - Schedule on `^prj`;
   - a link gesture on a resident of an unblocked project with a future date, which
     should pull it forward.

## Deliberately not doing

- Native Mac rendering of `link_guard` (bob-cli-73).
- The pre-epic plugin test failures (bob-cli-6v, bob-cli-6w) and flakes (bob-cli-3w,
  bob-cli-5l).
- Fixing the canceled bob-cli-6t pull-in docs (bob-cli-6x).
- The 6u feature follow-ups bob-cli-6y, -6z, -70, -71, -72.
- Shared Rust/JS `vectors.json` rows for the `[unblocked::]` writer. Each engine already
  runs the UB vectors.
