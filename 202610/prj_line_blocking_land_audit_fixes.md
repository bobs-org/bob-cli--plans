---
tier: epic
title: Finish ^prj project blocking — fix what the bob-cli-6u.18 land audit found
goal: Every defect the bob-cli-6u.18 land audit found in the ^prj project-blocking
  fixes is fixed in bob-cli and bob-plugins, with regression tests that fail on the
  old code. bob-cli `just check` fails only the bob-cli-6m percent-listen tests. bob-plugins
  `npm test` fails only the known bob-cli-6v, bob-cli-6w and bob-cli-3w cases. The
  fixed bob and plugins are installed on this host and, when it is reachable, on the
  MacBook.
parent_bead: bob-cli-6u.18
phases:
- id: cli-reconcile-projects
  title: Reconcile internal-id, warning, and projects-sync hoist fixes
  depends_on: []
  size: medium
  description: 'cli-reconcile-projects: filter same-note ids from inherited resident
    edges, warn only for field-named internal targets, scope the internal-id filter
    to recognised projects and to [id::] values, stop the frontmatter scheduled bypass
    of the hoist gate, revert ^prj matching to exact, CRLF and test-strength fixes,
    and the stale direct-^prj-link docs.'
- id: cli-capture
  title: Capture struck-link insert, closed-status, and consumed-link output fixes
  depends_on: []
  size: small
  description: 'cli-capture: the toggle insert ignores struck links, + rendering uses
    configured closed statuses, consumed @r:id gets honest human and JSON output,
    a fenced-index regression test that fails on the old code, and complete capture
    docs.'
- id: nav-fixes
  title: Navigation-hotkeys reversal order, schedule tag, and move promotion fixes
  depends_on: []
  size: medium
  description: 'nav-fixes: write nothing before the reversal parent write, make the
    schedule tag follow ^prj status, never raise ^prj on move promotion, add the missing
    plugin-level tests, nav README row and bob-cli docs.'
- id: cycler-blockid
  title: Task-status-cycler and block-id-prompt residual project-blocking fixes
  depends_on: []
  size: small
  description: 'cycler-blockid: never strip the tag from closed lines on recovery,
    recover without [unblocked::] on the successor path when ledger v12 is missing,
    keep residents in step when Alt+] clears a ^prj date, restore lost block-id tests,
    restore the cycler README row.'
- id: rollout
  title: Install the fixed bob and plugins on this host and the MacBook
  depends_on:
  - cli-reconcile-projects
  - cli-capture
  - nav-fixes
  - cycler-blockid
  size: small
  description: 'rollout: bob then plugins on this host, then best-effort on the MacBook
    with a detached install; dry-run verification only; record versions and what Bryan
    still has to do.'
proposed_by: bbugyi200.apollo.bob-cli-6u.18.land
create_time: 2026-10-12 20:32:24
status: done
bead_id: bob-cli-6u.18.7
---

- **PROMPT:** [prompts/202610/prj_line_blocking_land_audit_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prj_line_blocking_land_audit_fixes.md)
- **PARENT:** [202610/prj_line_blocking_landing_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/prj_line_blocking_landing_fixes.md)
- **BEAD:** [bob-cli-6u.18.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6u/bob-cli-6u.18.7.md)

# Plan: fix what the bob-cli-6u.18 land audit found

## Context

Epic **bob-cli-6u** moved project dates and prerequisites onto the `^prj` task line and
added the machine-owned `#prj_is_blocked` tag and the `[unblocked::]` freshness input.
Its land agent found defects and planned **bob-cli-6u.18**
(`plan:202610/prj_line_blocking_landing_fixes.md`, "the 6u.18 plan"), whose six phases
are closed. The 6u.18 land agent then audited every 6u.18 commit against the source and
found the defects below. This plan fixes them and finishes the rollout, which could not
reach the MacBook.

The governing contract is unchanged:

- `tag_scope = every_resident`: every open resident of a blocked project carries exactly
  one `#prj_is_blocked`. No other open line carries it, `^prj` never does, and closed
  lines are never edited.
- `unblock_keeps = yes`. Every automatic Blocked→Ready recovery records
  `[unblocked:: today]` unless a WRITE skip applies.
- `direct_prj_link = raise` (6u.18 decision): only **dependency** promotion never raises
  a `^prj` target (ADJ-11). A directly linked `^prj` rises to Next like any task, and
  the walk continues through it.

**Read first, in this order:**

1. The 6u.18 plan
   (`sase artifact read plan:202610/prj_line_blocking_landing_fixes.md "<why>"`),
   especially the phase section your phase continues, and the 6u plan
   (`plan:202610/prj_line_project_blocking.md`) Design §§1–6.
2. `docs/task-dependencies.md` §13 (PD/PDT/PDC/PDL/PDG vectors) and `docs/freshness.md`
   (UB vectors, WRITE/CLEAR/READ rules).
3. Memory, through `/sase_memory_read`: `decisions:projects-block-from-prj-line`,
   `decisions:unblocked-tasks-resurface-as-ticklers`, and (cli-capture only)
   `decisions:mac-capture-is-a-thin-client`.

### Commits this plan fixes

| Repo        | Commit    | 6u.18 phase                   |
| ----------- | --------- | ----------------------------- |
| bob-cli     | `54ac68b` | cli-reconcile-close (6u.18.1) |
| bob-cli     | `b09612e` | cli-projects-sync (6u.18.2)   |
| bob-cli     | `53af9cb` | cli-capture (6u.18.3)         |
| bob-plugins | `a7fafed` | nav-fixes (6u.18.4)           |
| bob-cli     | `524073e` | nav-fixes docs (6u.18.4)      |
| bob-plugins | `f1d5a4e` | cycler-blockid (6u.18.5)      |
| bob-cli     | `414b802` | cycler-blockid docs (6u.18.5) |

**Every finding below is an audit finding with approximate line numbers at bob-cli
`b09612e` / bob-plugins `a7fafed`.** Reproduce or confirm each one before fixing it.
When one is wrong or already fixed, skip it and say so in a note on your phase bead.
Every fix gets a regression test that **fails on the old code** (check by reverting the
fix locally before you commit), citing the vector or rule it enforces.

## Cross-cutting rules for every phase

- **bob-cli:** run `just check`. The only allowed failures are the two `capture::r#ref`
  percent-listen tests tracked by **bob-cli-6m**. Keep touched Rust files under about
  1500 lines where practical; new code goes in new modules (`capture/dependencies.rs` is
  already over 3000 lines: add nothing to it beyond call sites). Before touching any
  `--help` text, read `sase/memory/cli_rules.md` through `/sase_memory_read`. Update
  `README.md` and `docs/*.md` in the phase that changes behavior.
- **bob-plugins:** open it with `sase repo open bob-plugins -r "<why>"` and read its
  `AGENTS.md`.
  - Edit `src/` fragments only; each hand-edited fragment stays at or under 1000 lines
    (several are at 990+), so new code goes in new fragments registered in
    `src/fragments.json`.
  - Run `npm run build`, then `npm test` and `npm run validate`. Add every new test file
    to `package.json`'s `test` script.
  - Allowed `npm test` failures: the 4 `test-ledger-tools-unblocked-glyph.cjs` cases
    (**bob-cli-6v**), the decision-card "Keep counts once" case (**bob-cli-6w**), and
    the **bob-cli-3w** stage-ranker timing flake.
  - Bump each touched plugin's `manifest.json` minor version and its README row.
  - Deploy on this host only, with
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <plugin-id>`. Never sync
    plugins on the MacBook; that is `rollout`'s job.
  - `nav-fixes` and `cycler-blockid` may run in parallel. When rebasing `package.json`,
    `README.md`, or `docs/task-dependencies.md`, merge both sides.
- **Plugin phases own the bob-cli docs for their keymaps.** Commit both repos.
- **Hooks safety:** never run a live `bob task reconcile` or `bob projects sync` against
  `~/bob`. `BOB_DIR=~/bob bob … --dry-run -f json` is always fine. Only `rollout`
  touches the MacBook.
- **Capture JSON is additive only.** No existing field may change meaning.
- **Memory:** no memory note is edited by this plan.
- **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on your own phase bead; never
  create beads.

## Phase: cli-reconcile-projects

**Repo:** bob-cli: `src/native/task_status_hooks/`, `src/native/project_dependencies/`,
`src/native/task_complete/`, `src/native/projects/`.

1. **Inherited resident edges skip same-note ids.** `task_status_hooks/references.rs`
   `add_resident_implicit_edges` (~464-505) builds targets from the raw
   `prj.depends_on`; its doc comment assumes the field never holds a same-note id, but
   the adoption guard now leaves one there. Filter the ids through the same internal-id
   rule the blocking inputs use. Test: `proj.md` with `^prj [dependsOn:: local]`, a task
   `^local [id:: local]`, and a resident `^r1`; today's Pomodoro links `[[proj#^r1]]`.
   Reconcile must not raise `^local` or report it promoted.
2. **Warn only for field-named internal targets.** In `plan_legacy` (`reconcile.rs`
   ~683) the `project_internal_dependency` guard runs before `legacy_cover`, so every
   link-only child bullet under `^prj` that points into the same note warns, even when
   the field does not name it, and a field-named legacy child warns twice (`plan_legacy`
   and `plan_adoption`). Warn only when the `[dependsOn::]` field names the internal id,
   and at most once per (file, line, id) per run. Tests: `^prj [dependsOn:: job__x]`
   with a child bullet `- ![[#^r1]]` warns nothing; a field-named legacy internal child
   warns exactly once. This also adds the legacy-path test 6u.18.1 left out.
3. **The internal-id filter follows project recognition.**
   `project_dependencies/ blocking.rs` `blocking_open_dependency_ids` (~133-142) filters
   same-note ids for any `^prj` line, while reconcile's guard applies only to a
   recognised open project. Apply the filter only when `ProjectContext` recognises the
   note as a project. Test: a `status: done` note (PDB4) and a two-`^prj` note (PDB6)
   whose `^prj` has a same-note Depends-On link to an open task `[id:: n__a]`: the
   `^prj` stays Blocked by it, as an ordinary task would.
4. **The internal-id set resolves by `[id::]` only.** `blocking.rs` (~166-180) also
   counts raw same-note block ids and their canonical `note__block` ids. Dependencies
   resolve only through `[id::]`, so build the set from the `[id::]` values of the
   note's own tasks. Test: a task moved out of `proj.md` keeps `[id:: proj__setup]`, a
   new `^setup` without that id is minted in `proj.md`, and
   `^prj [dependsOn:: proj__setup]` still blocks the project.
5. **Frontmatter `scheduled` must not bypass the hoist gate.** `projects/sync.rs`
   `project_schedule_for_sync` (~718-733) returns the frontmatter `scheduled` even when
   there is no open `^prj`. A `status: wip` note with `scheduled: <future>`, no `^prj`,
   and one open task then pushes `ReconcileTaskSchedules` with `include_prj = true`;
   `prj_line_edit` fails ("failed to locate ^prj task"), the note is skipped, and live
   sync exits 1 while dry-run exits 0. With a past date it writes `[scheduled::]` onto
   ordinary tasks from a key the warning says was "left in place". Use the frontmatter
   date only when this run hoists it onto an open `^prj`; otherwise propagate only the
   `^prj` inline date. Fix the tests to pin the clock: the unit test
   (`projects/tests/ sync.rs` ~579) must cover a `today` both before and after the date,
   and the CLI test (`tests/cli/projects/sync.rs` ~975) must set `BOB_NOW` instead of
   relying on the real date. Assert live sync exits 0 and leaves the note unchanged.
6. **`^prj` matching stays exact.** 6u.18.2 made `blocking.rs` (~22-24) match `prj`
   case-insensitively, while `projects/tags.rs` `has_trailing_prj_anchor`,
   `task_status_hooks/reconcile.rs` (~325, ~577), `capture/link_guard.rs` (~129) and
   `capture/project_deps.rs` (~23, ~101) match exactly, and `capture/project_deps.rs`
   (~18-21) documents why. So `^PRJ` is a project to hooks but not to
   `bob projects list`, sync, capture, or the reconcile guard. Revert to exact matching
   and put the reason in a comment there:
   `capture_language::is_reserved_project_task_id` is case-insensitive only to stop
   capture from minting look-alike ids. Test: a note whose only project-like line ends
   `^PRJ` is not treated as a project by reconcile.
7. **Tests that prove their fixes.**
   - The CRLF tests `recovery_tests.rs` (~247) and the PDT10 test in
     `tests/cli/task_status_hooks/blocked.rs` (~1169) assert only `contains("\r\n")`,
     which the frontmatter satisfies. Assert the rewritten line itself ends in `\r\n`
     (for example `"[unblocked:: 2026-10-05] ^pitch\r\n"`).
   - Status-only flips (`task_status_hooks/sync.rs` ~704-711) have no test. Add one
     showing a plain status flip is not marked structural and is not held back by the
     quiet period, while a tag edit or `[unblocked::]` write is.
   - The PDF1 Databricks `#ref` test (`blocked.rs` ~876) must assert both rows sit under
     `### Blocked`, not just that the heading exists. Give PDB5 (~971) and PDB8 (~1076)
     a positive control showing the parent project really is Blocked.
8. **Sub-projects insert at end of file.** `projects/edits.rs` (~497-503): when the
   subtree-end anchor is the last line with no trailing newline, `line_ending()` yields
   `"\n"` and the Sub-projects line gets LF in a CRLF file. Use the note's line ending.
   Add tests for a Depends-On anchor with nested children and a CRLF subtree insert. (No
   vault note currently has a marker misplaced by the old Cancel Log bug, so no repair
   is needed.)
9. **Docs.**
   - `docs/task-dependencies.md` §12.10 (~1300-1302) still says the close-gesture rule
     "is the one path that makes `^prj` Next". Replace it with the direct-link rule from
     §5 and §13.4. Make the ADJ-11 bullet (~1619) read "Dependency promotion never
     raises a `^prj`…".
   - `docs/task-status-hooks.md` (~406) "promotion never raises a `^prj` target": add
     the direct-link exception.
   - `docs/projects.md` Sync Rules and the `bob projects sync` help text in
     `projects/mod.rs`: only a **missing** `^prj` gets the "cannot hoist" warning; a
     malformed or duplicate `^prj` is a project issue (exit 1), and a terminal project
     is skipped silently. Say exactly that.
10. **Finish.** `just check`, then read-only
    `BOB_DIR=~/bob <new build> task reconcile --dry-run -f json` and
    `BOB_DIR=~/bob <new build> projects sync --dry-run`. List every planned change in
    your bead note and explain any non-trivial one.

## Phase: cli-capture

**Repo:** bob-cli: `src/native/capture/`, `src/native/capture_task_toggle/`,
`src/native/capture_project_note.rs`.

1. **The toggle's insert ignores struck links.** `capture_task_toggle/links.rs`
   `insert_link_into_entry` (~284-285) decides "already linked" by substring, so a
   struck `~~[[cash#^id]]~~` counts, while `plan_link_removal` (6u.18.3) now skips
   struck links. Scenario: `^id` was completed, its link struck under the still-open
   CURRENT entry, then reopened to `[ ]`. `@cash+id!` raises it to `[*]`, skips the
   insert, and reports `toggle_direction: "link"` with `pomodoro_already_linked: true`:
   Next with no live Task Link. Use the unstruck-range matcher for the insert check too.
   Test that scenario: a live link is inserted and the struck history link stays.
2. **Configured closed statuses in `+` rendering.** `capture_project_note.rs` (~359)
   treats only `x`, `X` and `-` as closed, so a `[>]` configured as Cancelled becomes
   `[?] … #prj_is_blocked`, reopening a closed line. Use the configured status types,
   the same check reconcile and `apply_new_task_project_block` use. Test with a
   configured Cancelled status.
3. **Honest output for a consumed `@r:id`.** The new `kind: "link_guard"` (needed so the
   current Mac app does not fall back to "Linked under X") prints the generic "✓
   captured" line in human output (`capture/output.rs` ~1015-1020). Give it a dedicated
   human printer that says nothing was linked. When the consume removed existing links,
   also report `removed_pomodoro_links` and `day_file` (additive), matching ensure-next
   (`capture/plan.rs` ~772, ~787). Tests for both.
4. **A fenced-index test that fails on the old code.** 6u.18.3's fix
   (`capture/project_deps.rs` ~131-145, rebuilding `ProjectContext` from the staged
   lines) is right, but neither of its tests would fail without it. Add one where a
   resident sits directly after `^prj` and directly before a fenced block, so the
   inserted Depends-On line moves the stale fenced range onto that resident.
5. **Docs.** `docs/capture.md`: for every consumed shape (`@r+id!`, `@r:id`,
   ensure-next, link-then-close) list each field that is omitted, including
   `creates_pomodoro` on toggles and `pomodoro_link_action` on ensure-next, plus the new
   `removed_pomodoro_links`/`day_file` on `@r:id` and the `kind: "link_guard"` value.
   Add a note on **bob-cli-73** if the fields its native rendering should read changed.
6. **Finish.** `just check`. Confirm in your bead note that every capture JSON change is
   additive and list the fields.

## Phase: nav-fixes

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src`. Docs go in bob-cli
(`docs/task-dependencies.md` §13.8 and §13.10, `docs/projects.md`).

1. **HIGH — no write before the reversal's parent write.** a7fafed moved
   `applyBlockIdLinkRewrites` ahead of the parent write
   (`665-plugin-project-conversion.js` ~759-781). That helper writes each file as it
   goes and never rolls back. When the parent itself links to the project's `^prj` (for
   example `Health.md`'s `⛓️ **DEPENDS ON:** [[Health_gym_habit#^prj]]`), the rewrite
   edits the parent first, the parent write then sees changed content and stops with
   "Parent note changed; nothing was converted", and the link now points at a block that
   does not exist. Any later failure also leaves rewritten links dangling. Restore the
   a7fafed^ order: parent write first, link rewrites after. Still consume the live
   `#^prj` link in today's Pomodoro when the parent is blocked: plan the cleanup against
   both the project note's `^prj` link and the restored block id, in memory, before any
   write. Tests: the `Health.md` case converts correctly (it fails at a7fafed); a
   positive test that the live link is removed when the parent is blocked; a failure
   test that leaves every file unchanged.
2. **The schedule tag follows `^prj` status.** `265-project-schedule-inline.js`
   `projectPrjKeepsDependsOnReason` (~60-72) returns true for any Depends-On link or
   non-empty `[dependsOn::]`, but closed links stay on that line as history (§5). A
   `^prj` Blocked only by its future date, with one closed link, flips to `[ ]` when the
   date is set to today or deleted (Ctrl+D), while its residents keep `#prj_is_blocked`.
   Make the tag follow `^prj`'s post-gesture status: strip it from every open resident
   when `^prj` leaves Blocked, and keep it while `^prj` stays `[?]`. Tests: that
   scenario through both `planProjectScheduledUpdate` and `planProjectScheduledDelete`,
   and Ctrl+D with an open link remaining (tags stay). Correct the §13.8 bullet in
   `docs/task-dependencies.md` ("keep the tag while `^prj` holds Depends-On links") to
   match.
3. **Move promotion never raises `^prj`.** `250-task-move.js` `promoteMovePrerequisite`
   (~259-349) promotes any open target, so Ctrl+Shift+M into project P whose `^prj`
   depends on `[[Q#^prj]]` turns Q's `^prj` into `[*]` and skips Q's own prerequisites.
   Follow the planner (`080` ~395-409, ADJ-11/PDB9): walk through a `^prj` target to its
   open prerequisites without raising it. Also promote only from Depends-On lines of
   open tasks, and give each task's prerequisites that task's own lane, not the
   strongest lane in the moved block. Tests for all three.
4. **Missing tests** from 6u.18.4 item 8, each at plugin level where a harness exists:
   - the vault commit guard's implicit resident edges (`440` ~269-315);
   - a counted `^prj` schedule session and a counted Depends-on session writing
     `[unblocked::]`;
   - the recommended roll (`070`) and Task Link batch (`500`) recovery writer;
   - a forward conversion with a carried Depends-On line, and a round trip;
   - an inbox route into a blocked project.
5. **Docs.** The nav row in the plugins `README.md` must describe the current behavior
   (the cycler row belongs to `cycler-blockid`). `docs/projects.md` conversion paragraph
   (~289-293): a `[?]` source alone seeds a Ready project; only a carried open
   prerequisite or a future inline date blocks it.
6. **Finish:** manifest bump (2.25.0 → 2.26.0), README row, build/test/validate, deploy
   on this host.

## Phase: cycler-blockid

**Repo:** bob-plugins, `plugins/task-status-cycler/src` and
`plugins/block-id-prompt/src`. Docs in bob-cli as needed.

1. **Never edit closed lines on recovery.** `080-dependencies.js` (~530-535) sets
   `stripTag` without checking the task is open, and both apply paths skip the status
   check when `recover === false` (`080` ~584, `130` ~339). So a closed
   `- [x] #task Done #prj_is_blocked ✅ 2026-10-01` loses its tag when the `^prj`'s
   prerequisite is cancelled from the nav Task Card (`api.recoverBlockedDependents`).
   Require an open status (as the successor path at `080` ~287 does). Test that case.
2. **Recover without ledger v12 on the successor path.** `088-project-blocking.js`
   `applySuccessorNoteEditsWithUnblockedGuard` (~65-100), called from
   `135-plugin-successors.js` (~799) for every Ctrl+Enter, Alt+] and Pomodoro close,
   still drops every Blocked→Ready edit when the freshness writer is missing and keeps
   the tag-strip edits, leaving a mixed project. Apply the recoveries without
   `[unblocked::]` and `console.warn` once, as `130-plugin-references.js` (~311-316) now
   does. Fix the stale comment at `190-plugin-targets.js` (~455). Add a plugin-level
   test through the successor apply path with no writer (the current pure-helper test
   passes on the old code).
3. **Alt+] that clears a `^prj` date keeps residents in step.** Alt+] on a `[?]` `^prj`
   retires its own future date and sets it Ready (`150-plugin-commands.js` ~43-46) but
   leaves every resident tagged, so Alt+] on a resident is refused with "resolve the
   ^prj prerequisite" until reconcile runs. When the gesture leaves `^prj` with no
   remaining reason, strip the tag from every open resident and recover residents whose
   only reason was the project, writing `[unblocked::]` per the WRITE rules, through the
   same recovery helper as the successor path. Test it.
4. **Restore lost block-id-prompt coverage.**
   - `scripts/test-block-id-prompt-dependencies.cjs` (~54-79) had its target moved from
     the tagged resident (line 5) to `^prj` (line 4). Keep the `^prj` case and add the
     resident case back as its own test.
   - Add a test that `120-plugin-pomodoro-links.js` (~302-318) reports dependency
     reasons before the schedule reason, using
     `test-block-id-prompt-pomodoro-link-runtime.cjs`'s harness.
5. **README.** Restore the Task Status Cycler row (`README.md` ~19), which a42f45d
   overwrote with nav schedule text: recover the cycler sentence from
   `git show a42f45d -- README.md` and bring it up to date with items 1–3.
6. **Finish:** manifest bumps for each touched plugin, README rows, build/test/validate,
   and deploy on this host.

## Phase: rollout

1. **This host.** From up-to-date bob-cli master, run `just install` (never pass
   `--force`). Verify `BOB_DIR=~/bob bob task reconcile --dry-run -f json` and
   `BOB_DIR=~/bob bob projects sync --dry-run` exit 0 with no errors and only changes
   you can explain. Pull the opened bob-plugins checkout to origin master and run
   `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **MacBook bob, best effort.** The MacBook's SSH link drops often (the 6u.18 rollout
   timed out five times; the 6u.18 land agent got two short windows). Retry
   `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
   - Run the install detached so a dropped link cannot kill it:
     `cd ~/projects/github/bobs-org/bob-cli && nohup zsh -lc 'git pull --ff-only && just install; echo INSTALL_EXIT=$?' > /tmp/bob-rollout-install.log 2>&1 < /dev/null &`,
     then poll the log. (The 6u.18 land agent started the same command at `b09612e` with
     log `/tmp/bob-6u18-install.log`; check it, but install the new master.)
   - Verify `bob task reconcile --dry-run -f json` and `bob projects sync --dry-run`
     there: both exit 0 with no errors, and the reconcile plan contains only changes you
     can explain.
3. **MacBook plugins, only after step 2 verifies:** `ssh mac 'bob plugins sync'`, then
   `ssh mac 'bob plugins list'`. If bob could not be updated on the Mac, do not sync its
   plugins; record exactly what is left.
4. **No vault edits** and no live reconcile or sync from this phase; the Mac cron runs
   them.
5. **Record** in a bead note: the bob commit on each machine, the plugin versions, and
   what is left for Bryan. He reloads the plugins (or restarts Obsidian) on the Mac,
   then pilots:
   - Ctrl+Shift+P → `b` on a `^prj`, add and remove;
   - Schedule on `^prj`, and Ctrl+D on a `^prj` date;
   - a link gesture on a resident of an unblocked project with a future date, which
     should pull it forward;
   - Alt+] on a `[?]` `^prj` whose only reason is its date.

## Deliberately not doing

- Native Mac rendering of `link_guard` (bob-cli-73).
- The pre-epic test failures and flakes: bob-cli-6m, bob-cli-6v, bob-cli-6w, bob-cli-3w,
  bob-cli-5l.
- The 6u feature follow-ups bob-cli-6x, -6y, -6z, -70, -71, -72.
- Splitting `capture/dependencies.rs`; this plan only keeps new code out of it.
