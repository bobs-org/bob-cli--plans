---
tier: epic
title: Finish ^prj project blocking — fix what the bob-cli-6u.18.7 land audit found
goal: Every defect the bob-cli-6u.18.7 land audit found in the ^prj project-blocking
  fixes is fixed in bob-cli and bob-plugins, each with a regression test that fails
  on the old code. bob-cli `just check` fails only the bob-cli-6m percent-listen tests.
  bob-plugins `npm test` fails only the known bob-cli-6v, bob-cli-6w and bob-cli-3w
  cases. The fixed bob and plugins are installed on this host and, when it is reachable,
  on the MacBook.
parent_bead: bob-cli-6u.18.7
phases:
- id: cli-fixes
  title: Reconcile PDR7 warning, exact close-path ^prj, capture output and docs fixes
  depends_on: []
  size: medium
  description: 'cli-fixes: restore the PDR7 warning for internal Depends-On line links,
    match ^prj exactly in the close paths, keep removal fields out of the nested link_guard
    object, test consumed @r:id human output, use the configured-open check in + rendering,
    restore lost PDB4/PDB6 and CRLF coverage, and correct docs/capture.md.'
- id: nav-fixes
  title: Navigation-hotkeys move walk-through, reversal rewrite failures, and wiring
    tests
  depends_on: []
  size: medium
  description: 'nav-fixes: walk through a [?] ^prj on move promotion, report post-write
    reversal link-rewrite failures and keep the project note, add schedule-tag and
    recovery-writer wiring tests that fail on the old code, drop dead exports.'
- id: cycler-fixes
  title: Task-status-cycler cross-note project reasons, Alt+[ and transcluded ^prj
    recovery
  depends_on: []
  size: medium
  description: 'cycler-fixes: resolve cross-note Depends-On ids when deciding whether
    a ^prj still blocks, recover residents when ^prj leaves Blocked for any status
    and when it is cycled through a transclusion, split the block-id resident test,
    tighten the notice-order test, and restore the cycler README row.'
- id: rollout
  title: Install the fixed bob and plugins on this host and the MacBook
  depends_on:
  - cli-fixes
  - nav-fixes
  - cycler-fixes
  size: small
  description: 'rollout: bob then plugins on this host, then best-effort on the MacBook
    with a detached install; dry-run verification only; record versions and what Bryan
    still has to do.'
proposed_by: bbugyi200.apollo.bob-cli-6u.18.7.land
create_time: 2026-10-12 22:09:25
status: wip
bead_id: bob-cli-6u.18.7.6
---

- **PROMPT:** [prompts/202610/prj_line_blocking_final_audit_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prj_line_blocking_final_audit_fixes.md)
- **PARENT:** [202610/prj_line_blocking_land_audit_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/prj_line_blocking_land_audit_fixes.md)
- **BEAD:** [bob-cli-6u.18.7.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6u/bob-cli-6u.18.7.6.md)

# Plan: fix what the bob-cli-6u.18.7 land audit found

## Context

Epic **bob-cli-6u** moved project dates and prerequisites onto the `^prj` task line and
added the machine-owned `#prj_is_blocked` tag and the `[unblocked::]` freshness input.
Two rounds of land-audit fixes followed: **bob-cli-6u.18**
(`plan:202610/prj_line_blocking_landing_fixes.md`) and **bob-cli-6u.18.7**
(`plan:202610/prj_line_blocking_land_audit_fixes.md`, "the 6u.18.7 plan"). All 6u.18.7
phases are closed. Its land agent audited every 6u.18.7 commit against the source and
found the defects below. This plan fixes them and finishes the rollout, which again
could not reach the MacBook.

The governing contract is unchanged:

- `tag_scope = every_resident`: every open resident of a blocked project carries exactly
  one `#prj_is_blocked`. No other open line carries it, `^prj` never does, and closed
  lines are never edited.
- `unblock_keeps = yes`. Every automatic Blocked→Ready recovery records
  `[unblocked:: today]` unless a WRITE skip applies.
- `direct_prj_link = raise`: only **dependency** promotion never raises a `^prj` target
  (ADJ-11). A directly linked `^prj` rises to Next like any task.
- A note is `project(T)` only with project frontmatter, a non-terminal `status:`, and
  exactly one open, unfenced `^prj` task (`ProjectContext::for_note` in
  `src/native/project_dependencies/blocking.rs`). A missing dependency id never blocks
  (DC7).

**Read first, in this order:**

1. The 6u.18.7 plan
   (`sase artifact read plan:202610/prj_line_blocking_land_audit_fixes.md "<why>"`),
   especially the phase section your phase continues.
2. `docs/task-dependencies.md` §13 (PD/PDT/PDC/PDL/PDG/PDB/PDR vectors) and
   `docs/freshness.md` (UB vectors, WRITE/CLEAR/READ rules).
3. Memory, through `/sase_memory_read`: `decisions:projects-block-from-prj-line` and
   `decisions:unblocked-tasks-resurface-as-ticklers`; for `cli-fixes` also
   `decisions:mac-capture-is-a-thin-client`.

### Commits this plan fixes

| Repo        | Commit    | 6u.18.7 phase                      |
| ----------- | --------- | ---------------------------------- |
| bob-cli     | `d2836e0` | cli-reconcile-projects (6u.18.7.1) |
| bob-cli     | `e45c3ab` | cli-capture (6u.18.7.2)            |
| bob-plugins | `d34e078` | nav-fixes (6u.18.7.3)              |
| bob-cli     | `12abfd7` | nav-fixes docs (6u.18.7.3)         |
| bob-plugins | `39fd9b1` | cycler-blockid (6u.18.7.4)         |
| bob-cli     | `1851d77` | cycler-blockid docs (6u.18.7.4)    |

**Every finding below is an audit finding, with approximate line numbers at bob-cli
`12abfd7` / bob-plugins `d34e078`.** Reproduce or confirm each one before you fix it.
When one is wrong or already fixed, skip it and say so in a note on your phase bead.
Every fix gets a regression test that **fails on the old code**. Check this by reverting
the fix locally before you commit, and cite the vector or rule the test enforces. The
last audit found several 6u.18.7 tests that passed on the old code, so this check is
mandatory.

## Cross-cutting rules for every phase

- **bob-cli:** run `just check`. The only allowed failures are the two `capture::r#ref`
  percent-listen tests tracked by **bob-cli-6m**. Keep touched Rust files under about
  1500 lines where practical; new code goes in new modules (`capture/dependencies.rs` is
  already over 3000 lines: add nothing to it beyond call sites). Before touching any
  `--help` text, read the CLI-rules reference memory with `/sase_memory_read`. Update
  `README.md` and `docs/*.md` in the phase that changes behavior.
- **bob-plugins:** open it with `sase repo open bob-plugins -r "<why>"` and read its
  `AGENTS.md`.
  - Edit `src/` fragments only; each hand-edited fragment stays at or under 1000 lines
    (several are at 940+), so new code goes in new fragments registered in
    `src/fragments.json`.
  - Run `npm run build`, then `npm test` and `npm run validate`. Add every new test file
    to `package.json`'s `test` script.
  - Allowed `npm test` failures: the 4 `test-ledger-tools-unblocked-glyph.cjs` cases
    (**bob-cli-6v**), the decision-card "Keep counts once" case (**bob-cli-6w**), and
    the **bob-cli-3w** stage-ranker timing flake.
  - Bump each touched plugin's `manifest.json` minor version and its README row. A
    plugin whose only change is in `scripts/` tests needs no bump.
  - Deploy on this host only, with
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`. Never sync
    plugins on the MacBook; that is `rollout`'s job.
  - `nav-fixes` and `cycler-fixes` may run in parallel. When rebasing `package.json`,
    `README.md`, or `docs/task-dependencies.md`, merge both sides.
- **Plugin phases own the bob-cli docs for their keymaps.** Commit both repos.
- **Hooks safety:** never run a live `bob task reconcile` or `bob projects sync` against
  `~/bob`. `BOB_DIR=~/bob bob … --dry-run -f json` is always fine. Set `BOB_NOW` to the
  vault's local date if the dry-run fails only because today's UTC daily note does not
  exist yet. Only `rollout` touches the MacBook.
- **Capture JSON is additive only.** No existing field may change meaning.
- **Memory:** no memory note is edited by this plan.
- **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on your own phase bead; never
  create beads.

## Phase: cli-fixes

**Repo:** bob-cli: `src/native/task_status_hooks/`, `src/native/task_complete/`,
`src/native/capture/`, `src/native/capture_project_note.rs`, `docs/capture.md`.

1. **PDR7 warns again for internal Depends-On line links.** d2836e0 put the "warn only
   when the `[dependsOn::]` field names the id" gate in the shared helper
   `warn_project_internal_target` (`task_status_hooks/reconcile.rs` ~549-588). The
   Depends-On line path (~880-910) calls it too. Internal line links are never
   projected, so the field never names them, and a user's
   `⛓️ **DEPENDS ON:** [[#^local]]` under `^prj` is now silently ignored. PDR7
   (`docs/task-dependencies.md` ~1847, §13.2, §13.6) and `docs/task-status-hooks.md`
   (~399-400) say it warns.
   - Fix: a resolved internal link on the `^prj` Depends-On line always warns
     `project_internal_dependency`. The field-name gate applies only to legacy link-only
     children (`plan_legacy`) and field-only adoption (`plan_adoption`).
   - Keep the once-per-(file, line, id) dedup across all three paths.
   - Restore `reconcile_keeps_prj_internal_links_unprojected`
     (`tests/cli/task_status_hooks/blocked.rs`) to assert exactly one
     `project_internal_dependency` warning for `proj.md`.
   - Keep `reconcile_does_not_warn_for_an_unfielded_internal_legacy_child` and
     `reconcile_keeps_field_only_internal_prj_ids_unprojected` passing.
   - Correct the helper's doc comment and the stale "warned once per link" comment
     (~894).
2. **Exact `^prj` matching in the close paths.** d2836e0 reverted `blocking.rs` to exact
   `prj` matching, and its comment says `^PRJ` is an ordinary task everywhere. But
   `task_complete/successors.rs` (~605-610), `task_complete/recovery.rs` (~117-122) and
   `task_complete/project_close.rs` (~129-133, ~170-174) still use the case-insensitive
   `capture_language::is_reserved_project_task_id`.
   - The bug: a `[?] … ^PRJ` task recovering when its prerequisite closes gets
     `Recover("project_task")` and is not linked as a successor.
   - Fix: use the exact matcher from `project_dependencies/blocking.rs`, made
     `pub(crate)` if needed.
   - Test: closing the prerequisite of an ordinary `^PRJ` task recovers it and links it
     like any other successor.
3. **Removal fields stay out of the nested `link_guard` object.**
   `LinkGuardJson.removed_pomodoro_links` and `day_file` (`capture/output.rs` ~438-441)
   only skip serialization when unset. So a consumed `@r:id` that removed links also
   emits `link_guard.removed_pomodoro_links` and `link_guard.day_file`, which are
   undocumented duplicates of the top-level fields.
   - Fix: mark both `#[serde(skip)]` (they are carriers for `plan.rs` ~772-796). Keep
     the top-level fields exactly as they are.
   - Test: the nested object has neither key while the top level has both.
4. **Test the consumed `@r:id` human output.** `print_human_link_guard`
   (`capture/output.rs` ~944-960, ~1606) has no test. Add CLI tests for human (non-JSON)
   output of a consumed `@r:id`, one where existing Pomodoro links were removed and one
   where none were. Each asserts the "not linked" wording, the absence of the generic
   `✓ captured` line, and, for JSON, that `removed_pomodoro_links` and `day_file` are
   omitted when nothing was removed.
5. **`+` rendering uses the configured-open check.** `capture_project_note.rs`
   (~361-364) treats any status that is not configured-terminal as open. So an
   unconfigured `[!]` (or an Empty-type status) under a blocked project still renders as
   `[?] … #prj_is_blocked`, a line reconcile never treats as a resident
   (`ProjectContext::is_resident` needs a recognised, open status).
   - Fix: block only when the status is configured and open, the same check as
     `apply_new_task_project_block` (`capture/link_guard.rs` ~603).
   - Test: an unconfigured `[!]` authored under a future-scheduled project keeps `[!]`
     and gets no tag.
   - Also give the `[>]` assertion added to the `link_guard.rs` test a case that fails
     on the old code, or drop the claim that it is a regression test.
6. **Restore lost coverage.**
   - `recovery_preserves_crlf_when_writing_unblocked`
     (`task_complete/tests/recovery_tests.rs` ~247) still asserts only
     `contains("\r\n")`, which the frontmatter satisfies. Assert the rewritten task line
     itself, `[unblocked:: …]` and all, ends in `\r\n`. (6u.18.7.1 reported this done;
     it was not.)
   - d2836e0 rewrote the PDB4 test (`reconcile_ignores_terminal_projects`) and the PDB6
     test (`reconcile_ignores_notes_with_two_open_prj_lines`). That dropped PDB4's
     `status: done` project with a `[?] ^prj` blocked by an open prerequisite in another
     note (residents are not inherited-Blocked, and the tag is stripped). It also
     dropped PDB6's two-scheduled-`^prj` variant. Restore both as separate tests next to
     the new ones.
   - The fenced-index test
     `resident_after_prj_before_fence_is_not_hidden_by_stale_fence_range`
     (`capture/project_deps.rs` ~341) inserts the Depends-On line at index 1, inside the
     frontmatter. Insert it directly after `^prj`, and confirm it still fails without
     the staged-lines `ProjectContext` rebuild.
7. **`docs/capture.md`.**
   - ~3442-3444 says a consumed link-then-close omits `block_link` and
     `pomodoro_link_action`. The code always emits both (`pomodoro_close.rs`
     ~2063/~2081, test `tests/cli/capture/pomodoro_close.rs` ~1611), and ~2070-2073
     agrees with the code. Fix the omission list.
   - ~959-961 still says a consumed `@r:id` omits `day_file`. It emits `day_file` (and
     `removed_pomodoro_links`) when the consume removed links.
   - ~3441, Ensure Next: list every field the consumed shape omits (`plan.rs` ~645-667:
     `toggle_direction`, `block_link`, `creates_pomodoro`, `pomodoro_already_linked`,
     `pomodoro_selector_unused`, placement and name fields, `pomodoro_link_action`).
   - ~3105-3107 says the ledger is unchanged on an Ensure Next consume, but
     `consume_live_links` (`ensure_next.rs` ~43) removes live links. Correct it.
   - If the fields bob-cli-73's native renderer should read changed, add a note on
     **bob-cli-73** that supersedes its note #1 (which says `link_guard` omits
     `day_file`).
8. **Finish.** `just check`. Then run read-only
   `BOB_DIR=~/bob <new build> task reconcile --dry-run -f json` and
   `BOB_DIR=~/bob <new build> projects sync --dry-run`. In your bead note, list every
   planned change, explain any non-trivial one, and confirm every capture JSON change is
   additive.

## Phase: nav-fixes

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src`. Docs go in bob-cli
(`docs/task-dependencies.md` §13.8 and §13.10, `docs/projects.md`).

1. **Move promotion walks through a `[?]` `^prj`.** In `promoteMovePrerequisiteTarget`
   (`215-project-blocking.js` ~355-410), the `status === "?"` early return (~367-369)
   runs before the `^prj` walk-through (~399).
   - Scenario: P's `^prj` depends on `[[Q#^prj]]`, and Q's `^prj` is `[?]` because its
     prerequisite R is open. That is the consistent state. Ctrl+Shift+M of a `[*]` task
     into P promotes nothing, and R stays `[ ]`.
   - The 6u.18.7.3 test only passes because it sets Q's `^prj` to `[ ]` while it has an
     open dependency.
   - Fix: run the `^prj` walk-through before the `?` gate (the dependency-edit path at
     `080` ~395 already does).
   - Test: the consistent state above. R becomes `[*]`, and Q's `^prj` stays `[?]` and
     unchanged.
2. **Reversal reports post-write link-rewrite failures.** The parent-first reversal
   (`665-plugin-project-conversion.js` ~895-913) applies the planned link rewrites after
   the parent write. It swallows `vault.process` errors and ignores a
   `current !== change.before` mismatch, then trashes the project note anyway and counts
   the link as updated. That leaves a dangling `[[Project#^prj]]` with no warning. When
   today's daily note goes through this path, the "Pomodoro links not removed" suffix is
   also lost (~915-923).
   - Fix: count each rewrite that throws or finds changed content. When any failed, keep
     the project note and show
     `Restored the task in <parent>, but N links could not be updated; "<note>" was kept`,
     as a7fafed^ did (~800-806).
   - Keep the Pomodoro-cleanup failure suffix when the daily note's rewrite failed.
   - Tests: a linking note changed between planning and writing, and a daily-note
     rewrite failure.
3. **Schedule-tag tests that fail on the old code.** The three 6u.18.7.3
   `schedule-writes` tests pass on a7fafed, because their resident has a ready recovery.
   - Add tests through `planProjectScheduledUpdate` and `planProjectScheduledDelete`
     where the resident's own recovery is missing or blocked. For example, a resident
     `[?]` with its own open Depends-On, under a `^prj` whose only remaining Depends-On
     link is closed. Setting the date to today, or Ctrl+D, strips the tag while the
     resident stays `[?]`.
   - Remove `projectPrjKeepsDependsOnReason` and `projectResidentRecoveryClears`, now
     unused, and their exports (`700-exports.js` ~270-271), unless something still uses
     them.
4. **Recovery-writer wiring tests at plugin level.** The 6u.18.7.3 counted Depends-on,
   `^prj` schedule and roll tests call planners with a hand-made `markUnblockedLine`, so
   they pass on the old code.
   - Add a test through the real plugin method `applyCountedLocalTaskDependency`
     (`570-plugin-link-roll-and-schedule.js` ~492). It must fail if the
     `markUnblockedLine: navFreshnessMarkUnblockedLine(...)` wiring is removed.
   - Add the missing Task Link batch recovery-writer test (`500` ~666/~746/~883): a Task
     Link batch that recovers a Blocked task writes `[unblocked::]` through the
     freshness API.
   - Use the existing fake-app harnesses.
5. **Finish:** manifest bump (2.26.0 → 2.27.0), README nav row, and bob-cli §13.10 if
   the move or reversal wording changes. Then build/test/validate and deploy on this
   host.

## Phase: cycler-fixes

**Repo:** bob-plugins, `plugins/task-status-cycler/src` and
`plugins/block-id-prompt/src` plus `scripts/`. Docs in bob-cli
(`docs/task-dependencies.md` §13.10 ~1791, `docs/task-status-hooks.md`).

1. **Cross-note project reasons are resolved, not assumed open.**
   `projectHasRemainingBlockingReason` (`088-project-blocking.js` ~75-89) reads only the
   active note, so every cross-note Depends-On id counts as open. Nav refuses same-note
   `^prj` targets, and closed prerequisites stay on the Depends-On line as history. So
   Alt+] recovers residents only when the `^prj` has no dependencies at all.
   - Scenario:
     `- [?] #task #prj Launch [scheduled:: <future>] [dependsOn:: jobs__review] ^prj`
     with `jobs.md` holding `- [x] … [id:: jobs__review]`. Alt+] retires the date and
     sets `^prj` Ready, but `- [?] Resident #prj_is_blocked` stays Blocked and tagged.
   - The same root cause hits residents (~106-111). A resident whose own cross-note
     prerequisite closed while the project was blocked loses its tag but stays `[?]`
     with no reason left.
   - Fix: before planning, resolve every cross-note id against the vault. Reuse the
     successor path's note reader (`135-plugin-successors.js` ~76-120) or the existing
     id lookup. Only an id that resolves to an open task is a reason; a missing id never
     blocks (DC7).
   - Make `recoverProjectResidentsAfterPrjUnblock` (`180-plugin-editor-edits.js` ~904)
     resolve first, then plan.
   - Tests: a closed cross-note target (residents recover), an open one (tags stay), and
     a resident whose own cross-note prerequisite closed (it recovers).
2. **`^prj` leaving Blocked for any status recovers residents.** Alt+[ on a `[?]` `^prj`
   can move it to Cancelled. `projectDependencyInfo` then returns null (the note is no
   longer `project(T)`), so residents keep the tag. Reconcile would strip it and recover
   them, because without an open `^prj` the note has no project reason.
   - The 1851d77 docs and the comment at `180` ~901 already claim Alt+[ recovers
     residents.
   - Fix: when a cycle takes `^prj` out of `[?]` to any non-Blocked status, apply the
     same resident recovery, treating a note with no open `^prj` as having no project
     reason.
   - Test it. Make the comment and the bob-cli docs match the code exactly.
3. **A transcluded `^prj` recovers its residents.** `160` ~712-714 skips recovery when
   the cycled `^prj` lives in a different note than the active editor. Run the same
   resident recovery against that note through `vault.process` (no editor), and test it.
4. **block-id-prompt coverage.**
   - In `scripts/test-block-id-prompt-dependencies.cjs` (~54), the resident case was
     restored as the second half of the `^prj` test. Make it its own test, as the
     6u.18.7 plan asked.
   - In `scripts/test-block-id-prompt-pomodoro-link-runtime.cjs` (~97), the ordering
     test's positive regex `/Project blocked by dependency|Task blocked/` also matches
     the "Task blocked · project reason" fallback. Assert the exact dependency notice.
5. **README cycler row.** The Task Status Cycler row still opens with nav's text
   ("Project Schedule is the inline [scheduled::] field on ^prj; … prune their live
   links from today's open Pomodoros"), and the cycler does no such pruning. Fix it:
   - Restore the cycler sentence from `git show a42f45d -- README.md` ("…tagged
     #prj_is_blocked residents stay Blocked until recovery…").
   - Drop the nav-only text and fix the stray double space.
   - Describe items 1–3 accurately.
6. **Finish:** manifest bumps (cycler 1.33.0 → 1.34.0; block-id-prompt only if its
   `src/` changes), README rows, build/test/validate, and deploy on this host.

## Phase: rollout

1. **This host.** From up-to-date bob-cli master, run `just install` (never pass
   `--force`). Verify `BOB_DIR=~/bob bob task reconcile --dry-run -f json` and
   `BOB_DIR=~/bob bob projects sync --dry-run` exit 0 with no errors and only changes
   you can explain. Pull the opened bob-plugins checkout to origin master and run
   `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **MacBook bob, best effort.** The MacBook's SSH link drops often; the 6u.18 and
   6u.18.7 rollouts both lost it. Retry
   `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
   - First read `/tmp/bob-rollout-install.log` there. The 6u.18.7 rollout started a
     detached install of `12abfd7` and never saw `INSTALL_EXIT`.
   - Run the install detached so a dropped link cannot kill it:
     `cd ~/projects/github/bobs-org/bob-cli && nohup zsh -lc 'git pull --ff-only && just install; echo INSTALL_EXIT=$?' > /tmp/bob-rollout-install-2.log 2>&1 < /dev/null &`,
     then poll the log.
   - Verify `bob task reconcile --dry-run -f json` and `bob projects sync --dry-run`
     there. Both must exit 0 with no errors, and the reconcile plan may contain only
     changes you can explain.
3. **MacBook plugins, only after step 2 verifies:** `ssh mac 'bob plugins sync'`, then
   `ssh mac 'bob plugins list'`. If bob could not be updated on the Mac, do not sync its
   plugins; record exactly what is left.
4. **No vault edits** and no live reconcile or sync from this phase; the Mac cron runs
   them.
5. **Record** in a bead note the bob commit on each machine, the plugin versions, and
   what is left for Bryan. He reloads the plugins (or restarts Obsidian) on the Mac,
   then pilots:
   - Ctrl+Shift+P → `b` on a `^prj`, add and remove;
   - Schedule on `^prj`, and Ctrl+D on a `^prj` date;
   - Ctrl+Shift+M of a Next task into a project whose `^prj` depends on another
     project's `[?]` `^prj`;
   - Alt+] on a `[?]` `^prj` whose remaining Depends-On target is closed.

## Deliberately not doing

- Native Mac rendering of `link_guard` (bob-cli-73).
- The pre-epic test failures and flakes: bob-cli-6m, bob-cli-6v, bob-cli-6w, bob-cli-3w,
  bob-cli-5l.
- The 6u feature follow-ups bob-cli-6x, -6y, -6z, -70, -71, -72.
- Splitting `capture/dependencies.rs`; this plan only keeps new code out of it.
- Two audit observations the 6u.18.7 land agent declined, recorded on bob-cli-6u.18.7:
  the reversal's pre-write block-id mint on target notes, and the resident-edge filter
  keying by note path.
