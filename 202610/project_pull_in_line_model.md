---
tier: epic
title: Pull an emptied project's ^prj in as a successor link, on the ^prj line model
goal: When a close gesture completes the last remaining task of a project note, its
  ^prj task is linked right after the closed task's link, becomes Next, and loses
  its hide tag in the same write. This works in bob capture (! and =x) and in Obsidian
  (Ctrl+Enter, Alt+]). "Empty" is exactly bob projects sync's surface predicate, so
  sync never re-hides a pulled-in ^prj. docs/task-dependencies.md §12.10/§12.11 become
  true (resolving bob-cli-6x). bob-cli just check fails only the bob-cli-6m tests,
  and bob-plugins npm test fails only the known bob-cli-6v, bob-cli-6w and bob-cli-3w
  cases. The new bob and plugins are installed on this host and, when reachable, on
  the MacBook.
phases:
- id: contract
  title: Reconcile the §12.10 pull-in contract with sync and the shipped line model
  depends_on: []
  size: small
  description: 'contract: make §12.10 EMPTY exactly bob projects sync''s surface predicate
    (dated recount, plannable recognition, marker-line sub-project states, sync''s
    schedule), specify the recover-rank [unblocked::] ordering, the PROJECT RECOVERY
    cascade, surfaced rows, SL20 and the still-blocked reason shape, fix the freshness
    §5 cross-refs, and add PP25–PP28.'
- id: cli-model
  title: Shared sync surface predicate and the Rust pull-in planner
  depends_on:
  - contract
  size: medium
  description: 'cli-model: factor sync''s surface predicate into one crate-visible
    projects helper, widen normalize_task_hide_tag, add the pure task_complete/project_pull_in.rs
    planner, wire it into plan_successors (gate, merge, breaker, #hide edit, [unblocked::],
    cascade, row fields), and pin the planner-level PP vectors.'
- id: cli-capture
  title: Project pull-in in bob capture ! and =x closes, JSON and human output
  depends_on:
  - cli-model
  size: medium
  description: 'cli-capture: keep id-less C in the ! and =x close paths, feed staged
    project-note text to the planner, build the dependents snapshot only per the Gate,
    extend SL20, add trigger/hide_removed JSON and surfaced/project-empty human rows,
    add CLI PP tests including PP23, and measure performance.'
- id: cycler
  title: Project pull-in in task-status-cycler, with whole-line undo
  depends_on:
  - contract
  size: medium
  description: 'cycler: add 089-successor-project.js (surface predicate, #hide strip,
    pull-in planner) and the 137 mixin, relax the gate, integrate in 085, store whole-line
    receipts in 136, add notice copy in 087, emit reason project for still-blocked
    residents, test the PP vectors, bump, and deploy on this host.'
- id: nav
  title: Render pull-in and project rows in nav's Unblocked card
  depends_on:
  - contract
  size: small
  description: 'nav: in 285-unblocked-notice.js add the no-open-tasks chip, the Project
    empty title, the still-blocked pull-in suffix, the project member label and blocked-by-project
    copy (also for legacy project_dependency/project_schedule reasons), mirror the
    fallback text, test, bump, and deploy on this host.'
- id: rollout
  title: Install, verify on a vault copy, reach the MacBook, and record
  depends_on:
  - cli-capture
  - cycler
  - nav
  size: small
  description: 'rollout: install bob and sync plugins on this host, verify a pull-in
    on a temporary vault copy (including sync agreement), install bob and then plugins
    on the MacBook best-effort, close bob-cli-6x, and record versions and Bryan''s
    pilot steps.'
decisions:
  decision_record:
    ask: Add a decisions-web record project-pull-in-on-empty for the project pull-in
      rule?
    default: false
    memory:
    - decisions
    answer: false
proposed_by: bbugyi200.apollo.73
decided_by: auto
create_time: 2026-10-12 23:40:10
status: done
bead_id: bob-cli-74
---

- **PROMPT:** [prompts/202610/project_pull_in_line_model.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/project_pull_in_line_model.md)
- **BEAD:** [bob-cli-74](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-74/README.md)

# Plan: pull an emptied project's `^prj` in as a successor link (re-plan of bob-cli-6t)

## Context

Bryan's request (2026-10-12), unchanged:

> We already support pulling in successor tasks when closing a task link that unblocks
> one or more tasks. … if the final task in a project note is closed via its task link,
> then we should pull in a successor task link that points to the project task. The
> project task should be given a next (i.e. `[*]`) status when this happens. The `#hide`
> tag should also be auto-removed from the project task when this happens. The goal of
> this change is to make me immediately aware when the project corresponding with the
> task that I just finished no longer has any child tasks that can be worked. When this
> happens, I will then either close out the project task (if the project is really done)
> or add a new task … to that project note.

### What happened to bob-cli-6t

- 6t (`plan:202610/project_task_pull_in.md`) was planned against bob-cli-6s's YAML
  project model. Bryan canceled 6s and replaced it with **bob-cli-6u** (the `^prj` line
  model), then canceled 6t (`research:202610/bob_cli_6t_6u_epic_conflict.md`).
- Only 6t.1's contract landed (`ec7251e`). 6u.1 and 6u.17 ported it to the line model,
  so `docs/task-dependencies.md` §12.10 / §12.11 (PP1–PP24) and the pointers in
  `README.md`, `docs/projects.md`, `docs/plan.md` and `docs/capture.md` describe the
  pull-in in the present tense. **No code implements it.** bob-cli-6x tracks that gap.
- 6t.2 and 6t.3 never committed. Their uncommitted drafts were written against the
  deleted YAML helpers, and their fragment slot (088) now holds 6u's
  `088-project-blocking.js`. Nothing in them is worth recovering; this plan salvages
  6t's design and the ported contract.
- **bob-cli-6u is closed `done`** (6u.land note #4: all phases and child epics closed;
  `just check` passes except bob-cli-6m). Every model this plan builds on is on master:
  bob-cli `2fcf41a`, bob-plugins `5d8975b`. This epic has **no external bead gates**.
  (Lesson from 6t: a "wait until bead X closes" gate must check `resolution: done`.)

### What exists now (verified at bob-cli `2fcf41a`, bob-plugins `5d8975b`)

**Rust successor engine** (`src/native/task_complete/`)

- `successors.rs` (1608 lines). `plan_successors(SuccessorInput) -> SuccessorPlan`
  (≈522):
  - It returns `SuccessorPlan::default()` when no C task carries `[id::]` (≈525).
  - Candidates are only tasks whose `dependsOn` names a C id. Verdict order: WaitsOn,
    Scheduled, StillBlockedProject (residents only), then `Recover("project_task")` for
    a `^prj` (≈604), `Recover("hidden")` for `#hide`, and so on.
  - The BREAKER is `MAX_SUCCESSORS_PER_GESTURE = 5`.
  - `apply_verdicts` (≈877) uses an `edit_line` closure (≈1013). It sets status, mints,
    runs `apply_recovery_writes` (strips `#prj_is_blocked`, then `mark_unblocked`), and
    can strip the tag. **It never removes `#hide`.**
  - Recover rows are emitted only when the status was `[?]` and changed. A status-less
    "surfaced" row has no path.
  - The `[unblocked::]` scope is computed from the **pre-edit** line (≈1098), so a line
    still carrying `#hide` is `hidden` and skipped.
- `UnblockedRow` / `StillBlockedRow` have **no `trigger` or `hide_removed`**.
- `project_close.rs::recover_project_members` (≈154) already runs the PDC2 cascade for
  any `^prj` recover verdict ("§12.10 pull-in verdicts included", ≈153).
- `recovery.rs` has no production caller.
- **Callers.**
  - `capture/task_complete.rs` (`!`) clears C when no task has an id (≈183).
  - `capture/pomodoro_close.rs::plan_close_successors` (≈621; three call sites) returns
    `empty("checked")` there (≈695).
  - The dependents snapshot (`capture/dependencies.rs`) prefilters to notes containing
    `dependsOn` or `id::`. **A project note without those markers is never in it, even
    when staged.** Pull-in must read N's text from the planner's staged text.
  - The SL20 net pass (`task_complete.rs` ≈931–1009) only checks rows with a `link`.
- `capture/output.rs` (2561 lines):
  - Still-blocked project rows print `· project <path>`.
  - The strings `surfaced` and `· project has no open tasks` do not exist yet.

**`bob projects sync`** (`src/native/projects/`)

- `scan.rs::parse_project` computes `open_unhidden_count`. It counts open `#task` lines
  (every mark except `x`/`X`/`-`, so `[?]` and custom marks count), skips fences, and
  includes nested, blockquoted and `#ref` lines. It skips whole-token `#hide` lines and
  lines ending in `^prj`.
- `sync.rs::plan_project_sync_at` (≈544–620) surfaces with
  `effective_unhidden == 0 && !has_open_subprojects`, but only when there is no project
  schedule or its date ≤ today. When a project schedule applies, `effective_unhidden` is
  **recounted**: a hidden ordinary open task with fewer than two scheduled fields counts
  as unhidden, because sync is about to strip its `#hide`
  (`TaskSchedulePolicy::ordinary_hide_needs_removal`).
- Other sync behaviour:
  - The schedule comes from `project_schedule_for_sync`: a leftover frontmatter
    `scheduled` on an open `^prj`, else the inline `^prj` date.
  - Open sub-projects come from **child notes' `parent:`**, not from the marker line.
  - Any scan issue (two `^prj`-anchored lines, two inline dates on `^prj`, …) makes the
    file `can_plan = false`.
- `edits::normalize_task_hide_tag(line, future, include_prj)` is `pub(super)`.
- `sync.rs::display_for_wikilink` reads marker states: `~~[[x]]~~ ✅` is Done and
  `~~[[x]]~~ ❌` is Canceled. A link struck without ✅/❌ is **Open**.

**task-status-cycler** (bob-plugins, manifest 1.34.0)

- `085-successor-plan.js` (991 lines, 9 lines of headroom). Its gate returns early with
  no closed `taskId` (≈634). Rule 3 recovers any `^prj` as `project_task` (≈801), and
  `#hide` recovers as `hidden` (≈824).
- Still-blocked residents report `reason: "project_schedule" | "project_dependency"`
  (≈780). Rust and §12.5 say `project`.
- `135-plugin-successors.js` is at **1000 lines** (the build enforces ≤ 1000):
  - `quietGate` is at ≈661.
  - The Warm path's `noteTexts` (≈195–251) omits the closed task's own project note.
- `136` receipt:
  - It remembers only linked rows.
  - `restoreSuccessorStatuses` swaps the status symbol only.
- `088-project-blocking.js` has `rewriteRecoveredTaskLine` (`[unblocked::]` via
  `api.freshness.markUnblockedLine` v12) and `applyProjectResidentRecovery`.
- Fragments 089 and 137 are free.

**bob-navigation-hotkeys** (manifest 2.27.0)

- `285-unblocked-notice.js` (680 lines) has no `project_member` label.
- Its still-blocked suffix knows only `scheduled`, so project rows read
  `· waits on 0 more`.
- The title is fixed at `Unblocked`, and there is no `no open tasks` chip.

### Gaps between the ported contract and reality (why `contract` exists)

1. **§12.10 EMPTY is not sync's predicate in four places.** In each case sync would
   re-hide a pulled-in `^prj`:
   - **(a)** a project date ≤ today makes hidden ordinary tasks count;
   - **(b)** sync's recognition also needs a file it can plan;
   - **(c)** the marker reading: struck without ✅/❌ is open;
   - **(d)** a leftover frontmatter `scheduled` is sync's schedule.
2. §12.10 and §12.2 cite "freshness.md §4's WRITE rule". The WRITE rule is §5 ("Who
   stamps").
3. The recover-rank `[unblocked::]` must be evaluated **after** `#hide` is removed, or
   the hidden WRITE skip suppresses it.
4. The rows for a surfaced R (status unchanged, `#hide` removed) and the PROJECT
   RECOVERY cascade for a `[?]` R are not spelled out as vectors.
5. Still-blocked reason drift across the engines and docs, as listed above.
6. SL20 must also drop **unlinked** pull-in rows whose R closes later in the batch
   (PP21).

## Design

The Terms, Trigger, Rule, Gate, Writes, Reporting, Undo and Interplay of §12.10 stand,
with the corrections that `contract` makes below. Decisions this plan makes itself:

- **One predicate, shared in Rust.**
  - Sync's surface computation is factored into one crate-visible helper: recognition,
    plannability, effective status, the sync schedule, and the recounted
    effective-unhidden count. Both `plan_project_sync_at` and the pull-in call it, so
    they cannot drift.
  - The cycler mirrors the helper under the PP vectors.
- **Sub-projects come from the marker line.** Sync reads child notes' `parent:`, which
  is a vault-wide walk. The pull-in instead reads R's sync-maintained
  `🧩 **Sub-projects:**` line through sync's own `display_for_wikilink` states.
  - A child is open unless it is Done or Canceled.
  - The marker is at most 15 minutes stale. Both stale outcomes are benign: a missed
    pull-in (sync surfaces R later), or an extra one (sync re-hides R, and R stays Next
    until Alt+N or a close).
- **Blocked residents are remaining work**, as in sync. A Blocked task surfaces on its
  own when its blocker clears. Bryan's two answers (close the project, add a task) don't
  fit it.
- **Status follows the successor rule.**
  - ` ` and `?` become `*`.
  - `*` and `/` are unchanged. A Pending `^prj` is never demoted
    (`decisions:task-lanes-are-sticky`).
  - A link always raises R to Next, consistent with `direct_prj_link = raise` (§12.10:
    dependency promotion never raises `^prj`, but a direct link does).
- **No surprise writes.**
  - Steps 1–3 leave R byte-identical, `#hide` included.
  - Beyond status and `#hide`, the only write is the recover-rank `[unblocked::]` for a
    `[?]` R landing Ready (freshness §5).
  - A `[?]` R that leaves Blocked runs the §12.2 PROJECT RECOVERY cascade, as any `^prj`
    recovery does.
- **Still-blocked rows say `reason: "project"` in both engines** (§12.5).
  - nav also renders the legacy `project_dependency` / `project_schedule` kinds, which
    older cycler bundles still send.
  - Rust's human `· project <path>` suffix stays; the docs are corrected to match.
- **Plugins deploy on this host per phase** (bob-plugins `AGENTS.md`). The MacBook is
  touched only in `rollout`, bob before plugins (ADJ-13).

## Cross-cutting rules for every phase

- **Read first:**
  - this plan's Design;
  - `docs/task-dependencies.md` §12 (incl. §12.10 / §12.11 as `contract` leaves them)
    and §13.3, §13.5, §13.7;
  - `docs/freshness.md` §5 and §10b;
  - `sase memory read decisions:task-lanes-are-sticky decisions:projects-block-from-prj-line decisions:unblocked-tasks-resurface-as-ticklers glossary:project-task -r "<why>"`.
- **Vectors are the contract.**
  - Copy the PP rows you implement into tests, citing the vector id in a comment.
  - If an implementation disagrees with a vector, fix §12.10 / §12.11 first, in the same
    change, and say so in the commit.
- **Known failures.**
  - bob-cli `just check` may fail only the bob-cli-6m percent-listen tests.
  - bob-plugins `npm test` may fail only the bob-cli-6v and bob-cli-6w cases and the
    bob-cli-3w timing flake.
  - Any other failure must reproduce on clean origin/master before you call it
    pre-existing.
- **bob-cli.**
  - No new CLI options or subcommands. Put new logic in new modules: `successors.rs` is
    already 1608 lines, so keep its net growth small.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md`):
  - Edit only `src/` fragments, each ≤ 1000 lines. Register new fragments in
    `src/fragments.json` and mixins in `200-install-methods.js`, and export pure helpers
    in `210-exports.js`.
  - Run `npm run build`, `npm test` and `npm run validate`. Append new test files to
    `package.json`'s `test` script.
  - Minor-bump the touched plugin's manifest from the latest master version and update
    its root `README.md` row.
  - Deploy on this host only:
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`.
- **Safety.**
  - Never edit the vault.
  - Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
    `bob capture` against `~/bob`. Dry runs are fine; experiments use
    `BOB_DIR=<temporary copy of ~/bob>`.
  - Never touch the MacBook outside `rollout`.
- **Memory:** only `rollout`, and only within `decision_record`.
- **Follow-ups:** append `PROPOSED FOLLOW-UP:` notes to your own phase bead; never
  create beads.

## Phase: contract — reconcile §12.10 with sync and the line model

**Repo:** bob-cli, docs only. Edit `docs/task-dependencies.md` §12.2, §12.5, §12.6,
§12.10 and §12.11; touch `docs/projects.md` and `docs/capture.md` only where stated.

1. **Terms (§12.10).**
   - **N recognition.** Keep the current recognition, and require that sync can plan N:
     its scan reports no issue for the file (e.g. exactly one `^prj`-anchored line, at
     most one valid inline `[scheduled::]` on `^prj`). The effective status is
     non-terminal as sync computes it.
   - **REMAINING(N)** is sync's effective-unhidden count on the staged text:
     - Open `#task` lines other than R, outside fences, count; `[?]`, custom marks,
       nested, blockquoted and `#ref` lines are included. Lines with a whole-token
       `#hide` are skipped…
     - …**except** when a project schedule applies (see SCHED). Then a hidden ordinary
       open task with fewer than two scheduled fields also counts, because sync strips
       its `#hide`.
   - **SCHED(N)** is sync's project schedule: a leftover frontmatter `scheduled` on an
     open `^prj`, else R's inline `[scheduled::]`. Rule step 2 compares SCHED(N), not
     only R's inline field, with today.
   - **SUBOPEN(N)**: R's `🧩 **Sub-projects:**` marker line lists a child that sync's
     marker reading treats as open, i.e. anything but `~~[[child]]~~ ✅` or
     `~~[[child]]~~ ❌`. State the marker-line proxy and its benign staleness (Design).
2. **Rule (§12.10).**
   - Step 3: state that under the line model `^prj` reports only its own reasons
     (§13.3), so steps 1–2 cover it and engines implement no separate check. Keep the
     numbering.
   - Recover rank:
     - The `[unblocked::]` WRITE rule is evaluated on R's line **after** step 4 removed
       `#hide`, so the hidden skip never applies to a pulled-in R.
     - When R was `[?]` and leaves Blocked through steps 5–9, §12.2 PROJECT RECOVERY
       runs for N exactly as for a `project_task` recover (PDC2/PDC3).
   - ROWS: a **surfaced** row has the status unchanged (`previous_status_symbol` ==
     `status_symbol`), `hide_removed: true`, `link: null`, and its `not_linked` reason.
   - Interplay → Freshness: "untouched except the recover-rank `[unblocked::]`".
   - Multi-item drafts: SL20 drops any pull-in row, linked or not, whose R is no longer
     open in the final staged text.
3. **Cross-references.** The `[unblocked::]` WRITE rule lives in freshness §5 ("Who
   stamps"). Repoint every WRITE-rule citation of §4 to §5, including the
   `freshness.md#4-evaluation` anchor, in `docs/task-dependencies.md` (§8 table, §12.2,
   §12.10, §13.10, PDC2) and `docs/task-status-hooks.md`. Leave the citations of §4 that
   mean evaluation or lane review, such as `docs/plan.md`. Check with
   `rg -n 'freshness.md §4|freshness.md#4' docs/`.
4. **Reason shape (§12.5 / §12.6).**
   - `still_blocked[].reason` is `waits_on | scheduled | project` in both engines. A
     `project` row carries `project` (path) and `scheduled` when the project date
     blocks.
   - nav also renders legacy `project_dependency` / `project_schedule` rows.
   - The human (Rust) still-blocked suffix for `project` is `· project <path>` (current
     code); keep the nav card's `· blocked by project` / `· project until <date>`.
5. **Vectors (§12.11).**
   - Amend PP6: an unstruck child → nothing; `~~[[child]]~~` without ✅/❌ → nothing;
     only ✅/❌ children → pull-in.
   - Amend PP20: add "N has a sync scan issue (two inline dates on `^prj`) → nothing".
   - Add:
     - **PP25.** R is `[?] … #hide` and depends only on P in another note. N's only
       other open task is `[?] #prj_is_blocked #hide`. Close P (planned today) → R is
       pulled in after P's bullet: `[?]`→`[*]`, R's `#hide` removed. The resident loses
       `#prj_is_blocked` and recovers to `[ ]` as `project_member`. It gets no
       `[unblocked::]` (it is still hidden, UB6).
     - **PP26.** R carries `[scheduled:: <today or past>]`; N's only other open task B
       carries `#hide`. Close A → nothing (B is remaining work under the dated recount).
       With no project date → pull-in.
     - **PP27.** R `[?]` with `[fresh::]`, not planned today, last blocker closed →
       `' '`, `#hide` removed, `[unblocked:: today]` written, `not_planned_today`.
       Linked instead → `[*]` and no `[unblocked::]` (UB4).
     - **PP28.** A leftover frontmatter future `scheduled` on N with an open `^prj` →
       `still_blocked{reason: scheduled}`, R byte-identical.
6. **Pointers.**
   - `docs/projects.md` Sync Rules: say that close gestures use the same surface
     predicate (one shared helper), with sub-projects read from the marker line.
   - `docs/capture.md`: check that the field list and the `surfaced` verb match; change
     nothing else.
7. **Finish.**
   - Run `just check`.
   - Append a note to bob-cli-6x saying this epic implements §12.10 and that `rollout`
     closes 6x.

## Phase: cli-model — shared predicate and the Rust pull-in planner

**Repo:** bob-cli. Code: `src/native/projects/`, `src/native/task_complete/`.

1. **Shared surface helper.**
   - Add `src/native/projects/surface.rs` (registered in `projects/mod.rs`):
     `pub(crate) fn project_surface(relative, contents, today) -> Option<ProjectSurface>`.
     It returns `None` unless N is recognised and plannable per the Terms (including
     `done/` and canonical-daily-note exclusions).
   - `ProjectSurface` carries:
     - R's 0-based line index and current line;
     - whether R carries whole-token `#hide`;
     - SCHED(N) as `Option<NaiveDate>`;
     - REMAINING;
     - SUBOPEN from the marker line (via `display_for_wikilink`).
   - Factor the effective-unhidden recount and the `should_surface` expression out of
     `sync.rs::plan_project_sync_at` into functions that both sync and `project_surface`
     call. Sync keeps passing its child-note sub-project states; `project_surface`
     passes the marker states.
   - **Sync's behaviour must not change:** every `projects` unit and CLI test passes
     untouched. Add tests showing that sync's surface decision and `project_surface`
     agree on the PP2–PP6, PP20, PP26 and PP28 fixtures when no child notes exist.
2. Make `projects::edits::normalize_task_hide_tag` `pub(crate)`. Behaviour and bytes are
   unchanged (PP22).
3. **Pure planner `src/native/task_complete/project_pull_in.rs`**, in `successors.rs`'s
   style:
   - **Candidate detection** from C plus a staged-text lookup for C's notes:
     - Trigger (a): C has a task in N other than R, R ∉ C, and EMPTY(N).
     - Trigger (b): R is a dependent candidate of C, and EMPTY(N) on R's note text.
   - **Verdicts** per the Rule (steps 1, 2, 4–9; step 2 uses SCHED(N)).
   - **`#hide` removal** through `normalize_task_hide_tag(line, false, true)`.
   - **Display text:** `#prj` / `#hide` dropped **token-bounded** (never touches
     `#prj_is_blocked`).
   - **`needs_dependents`:** some C task has `[id::]`, or a candidate R carries
     `[dependsOn::]`.
   - Export a detection entry point that callers can run before deciding whether to
     build the dependents snapshot.
4. **Wire into `plan_successors`** with small edits to `successors.rs`:
   - `SuccessorInput` gains the staged texts of C's notes (empty keeps today's
     behaviour).
   - Return early only when no C task has an id **and** there are no candidates.
   - **Rows.** One row per R. A pull-in replaces `Recover("project_task")` /
     `Recover("hidden")`. Pull-ins are exempt from the BREAKER and sort in ORDER.
   - **`edit_line`** removes `#hide` before any recover-rank write, and computes the
     `UnblockScope` from the post-removal line.
   - **Cascade.** `recover_project_members` fires when a pull-in verdict recovers a
     `[?]` R.
   - **Row fields.** `UnblockedRow` gains `trigger: &'static str` and
     `hide_removed: bool`; `StillBlockedRow` gains `trigger`. Existing rows say
     `"dependency"`.
   - Surfaced rows are emitted.
   - Move row builders into the new module if `successors.rs` would grow by more than
     about 60 lines.
5. **Tests.**
   - Add `src/native/task_complete/tests/project_pull_in_tests.rs` (registered in
     `tests/mod.rs`). Cover every PP vector the planner can express: PP1–PP15, PP17,
     PP20, PP22, PP24–PP28.
   - PP22 bytes go through `normalize_task_hide_tag`.
   - Existing SL/PDC/UB tests pass unchanged apart from the additive fields.
6. **Finish.**
   - The callers compile with empty staged texts; `cli-capture` wires them.
   - Run `just check`.

## Phase: cli-capture — pull-in in `!` and `=x`, JSON and human output

**Repo:** bob-cli. Code: `src/native/capture/{task_complete,pomodoro_close,output}.rs`
(plus `plan.rs` for SL20). Read `cli-model`'s commit first.

1. **`!` path** (`capture/task_complete.rs`):
   - Stop clearing C when the `[id::]` gate fails.
   - Collect the staged text of C's notes from the planner (`peek_text` /
     `current_contents`), never from the dependents snapshot.
   - Run candidate detection. Build the dependents snapshot and run `snapshot_complete`
     only when `needs_dependents`; otherwise pass an empty snapshot.
   - Build the `NoteIndex` only when there is a candidate or the gate passed.
   - `unblocked_check: "unavailable"` still means nothing is linked or pulled in.
2. **`=x` path** (`capture/pomodoro_close.rs::plan_close_successors`): apply the same
   change in place of the early `empty("checked")`, for all three callers.
3. **SL20** (`apply_successor_net_post_pass` / `successor_row_survives`): also drop
   pull-in rows without a link whose R is no longer open in the final staged text.
   Linked-row behaviour is unchanged.
4. **Output** (`capture/output.rs`):
   - JSON: unblocked rows always serialize `trigger` and `hide_removed`; still-blocked
     rows always serialize `trigger`.
   - Human rows:
     - `surfaced` / `would surface` for a surfaced row;
     - `· project has no open tasks` before any reason suffix on every `project_empty`
       row, still-blocked included.
   - Dry-run JSON equals live JSON except `dry_run`.
   - Bob Mac Capture needs no change (additive JSON).
5. **CLI tests.** Add `tests/cli/capture/project_pull_in.rs`, registered in
   `tests/cli/capture/mod.rs`, following `pomodoro_close_successors.rs` and
   `project_close.rs`:
   - `!` and `=x` paths, JSON and human output (PP1, PP9–PP12, PP14, PP16, PP21, PP24,
     PP26, PP27);
   - `--dry-run` writes nothing;
   - **PP23:** after a live pull-in on a temp vault, `bob projects sync --dry-run`
     prints no event line naming N.
6. **Performance.** On a temporary copy of `~/bob` (release build), time `!` and `=x`
   dry runs before and after, for:
   - a non-project close;
   - a project close with tasks left;
   - a project's last-task close.

   Non-project closes must stay within noise of the old numbers and within §12.7's
   targets. Record the numbers in a phase note.

7. **Finish.**
   - Fix `docs/capture.md` if the shipped output differs.
   - Run `just check`.

## Phase: cycler — project pull-in in task-status-cycler

**Repo:** bob-plugins, `plugins/task-status-cycler/src/`.

1. **New pure fragment `089-successor-project.js`:**
   - `projectSurfaceState(text, path, today)` mirrors the contract exactly:
     - recognition and plannability;
     - REMAINING with the dated recount;
     - SCHED;
     - marker-line SUBOPEN with ✅/❌ states.

     Reuse 080's `projectDependencyInfo` where it agrees.

   - `removeProjectHideTags(line)`, byte-identical to PP22.
   - `planProjectPullIns(...)`, returning marks, recovers, rows and still-blocked rows
     in `planSuccessors`' shapes, with `computeSuccessorAnchors` for anchors.
   - Export the pure helpers in `210-exports.js`.

2. **New mixin `137-plugin-successor-project.js`:**
   - Before `planAndApplySuccessorsNow` returns `quietGate`, detect candidates from the
     gate cache's post-close note texts. Run a pull-in pass even when gated: it reads
     today's daily note and builds the dependents pool only per the Gate.
   - Put each candidate note's post-close text into `noteTexts` on the Warm path.
   - Register the fragment and the mixin. `135` is at 1000 lines, so offset every line
     added there by moving code into `137`.
3. **`085` integration**, within its 9 lines of headroom (move code into `089` if
   needed):
   - relaxed gate;
   - one row per R; the pull-in wins over `project_task` / `hidden`;
   - breaker exemption;
   - `#hide` stripped in `after`;
   - `trigger` / `hide_removed` on rows (existing rows `"dependency"`);
   - token-bounded display stripping.
   - Still-blocked residents now report `reason: "project"` with `project` (path) and
     `scheduled` (§12.5). Update `test-task-status-cycler-successors.cjs` (≈594).
4. **Recover rank and cascade.**
   - A `[?]` R landing Ready goes through 088's `rewriteRecoveredTaskLine`
     (`markUnblockedLine`, v12) **after** `#hide` removal. Without v12, warn and apply,
     as 088 does.
   - A `[?]` R that leaves Blocked runs `applyProjectResidentRecovery`.
5. **Receipt (`136`).**
   - Remember pull-in rows, linked or surfaced, with `before_line` / `after_line`.
   - `restoreSuccessorStatuses` restores R's whole line when the current line equals
     `after_line`; otherwise it keeps the symbol-swap fallback.
   - Existing rows are unchanged.
6. **Notice text (`087` `successorNoticeText`):** the §12.6 pull-in and surfaced copy.
   Mixed notices count a pull-in as linked.
7. **Gestures.**
   - Ctrl+Enter and Alt+] into Done pull in.
   - Alt+[ into Cancelled only surfaces (`cancelled`).
   - Raw checkbox clicks, reopen and walk cancel do nothing new.
8. **Tests.** Add `scripts/test-task-status-cycler-project-pull-in.cjs` to
   `package.json`:
   - pure PP vectors (planFixture style), including PP6, PP20, PP22 and PP25–PP28;
   - harness wiring:
     - Ctrl+Enter on the last task's link under an open Pomodoro inserts the link and
       rewrites R in one pass;
     - PP18 cancel;
     - PP19 take-back restores R byte-for-byte;
     - the existing zero-read test still passes, and a close in a project note with
       tasks left reads nothing beyond the gate.
9. **Finish.**
   - Bump the manifest (1.34.0 → 1.35.0 at plan time; re-check master) and its README
     row.
   - Build, test and validate.
   - Deploy `-p task-status-cycler` on this host and commit.

## Phase: nav — pull-in and project rows in the Unblocked card

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src/285-unblocked-notice.js`.

1. **Labels and still-blocked copy.**
   - `project_member` → `project member`.
   - `reason` `project`, `project_dependency` or `project_schedule` →
     `· blocked by project`, or `· project until <Mon D>` when `scheduled` is set (the
     same date format as `· waits until Oct 13`).
2. **Project rows** (`trigger: "project_empty"`):
   - a muted `no open tasks` chip after the lane pill on linked and not-linked rows;
   - the title `Project empty` when every row is a pull-in row;
   - `· no open tasks` on still-blocked pull-in rows;
   - `unblockedFallbackText` mirrors the cycler's pull-in and surfaced copy.

   Rows without `trigger` render as today.

3. **Tests:** extend `scripts/test-navigation-hotkeys-unblocked-notice.cjs`.
4. **Finish.**
   - Bump the manifest (2.27.0 → 2.28.0 at plan time; re-check master) and its README
     row.
   - Build, test and validate.
   - Deploy `-p bob-navigation-hotkeys` on this host and commit.

## Phase: rollout — install, verify, reach the MacBook, record

1. **This host.**
   - From up-to-date bob-cli master, run `just install` (never `--force`).
   - Check that `BOB_DIR=~/bob bob task reconcile --dry-run -f json` and
     `BOB_DIR=~/bob bob projects sync --dry-run` exit 0 with only explainable changes.
   - Pull the opened bob-plugins checkout to origin master, run
     `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **Verify on a copy.**
   - Copy `~/bob` to a temporary directory and use `BOB_DIR=<copy>` (set `BOB_NOW` if
     needed).
   - Pick a `wip` project whose only open task has a block ID. Link that task under an
     open Pomodoro in the copy's daily note, creating the note in the copy if needed.
   - Run `bob capture --dry-run '!<note>:<id>' -f json` and confirm the pull-in row:
     `trigger: project_empty`, a `link`, and `hide_removed` matching R's `#hide`.
   - Run it live on the copy, then `bob projects sync --dry-run` there: there must be no
     event line for that project.
   - Delete the copy. Never touch `~/bob`.
3. **MacBook bob, best effort.**
   - Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
     First read the last `/tmp/bob-rollout-install*.log` there; the 6u rollout could not
     reach the Mac, so this install also carries 6u's final fixes.
   - Install detached:
     `cd ~/projects/github/bobs-org/bob-cli && nohup zsh -lc 'git pull --ff-only && just install; echo INSTALL_EXIT=$?' > /tmp/bob-rollout-install-6t.log 2>&1 < /dev/null &`.
     Poll the log for `INSTALL_EXIT=0`.
   - Verify `bob task reconcile --dry-run -f json` and `bob projects sync --dry-run`
     there: exit 0, with only explainable changes.
4. **MacBook plugins, only after step 3 verifies:** `ssh mac 'bob plugins sync'`, then
   `ssh mac 'bob plugins list'`. If bob could not be updated there, do not sync its
   plugins; record exactly what is left.
5. > [!decision] decision_record **yes:**
   >
   > - Use `/sase_memory_write` (Edit And Republish) to add the decisions strand
   >   `project-pull-in-on-empty`, mirroring existing strands' frontmatter and sections.
   > - **Applies to:** bob-cli, bob-plugins, vault.
   > - **Claim:** a close that empties a project, by `bob projects sync`'s own
   >   predicate, links its `^prj` at the closed task's anchor as Next and removes
   >   `#hide`; without an anchor it only surfaces.
   > - **Why:** quote Bryan's request. Sync already surfaced emptied projects, but up to
   >   15 minutes late and with no ledger slot.
   > - **Rejected:** ignoring Blocked residents; leaving `#hide`; linking on cancel;
   >   auto-releasing R when a task is added; reading child notes for sub-projects; the
   >   sub-project cascade.
   > - **Cost:** R stays Next after a new task until Alt+N; two engines; a ≤ 15-minute
   >   marker staleness.
   > - **Reopens when:** pull-ins are routinely released unanswered.
   > - **Evidence:** this plan's ref and the phase commits.
   > - **Links:** `[[decisions/task-lanes-are-sticky]]`,
   >   `[[decisions/projects-block-from-prj-line]]`, and
   >   `[[decisions/closed-task-hands-slot-to-successors]]` only if bob-cli-63 has
   >   written it.
   > - Then run `sase memory init`.
   >
   > **no:** append a `PROPOSED FOLLOW-UP:` note for that record instead.
6. **Close bob-cli-6x:**
   `sase bead close bob-cli-6x --note "<commits that make §12.10 true>"`.
7. **Record** in a phase note:
   - the bob commit and plugin versions on each machine;
   - the verification output;
   - what is left for Bryan. He reloads the plugins (or restarts Obsidian) on the Mac,
     then pilots:
     - Ctrl+Enter on a project's last task link → `^prj` linked, `[*]`, no `#hide`;
     - a same-day reopen restores it;
     - `bob capture '!note:id'` on a last task.
   - Follow-ups: distinct Bob Mac Capture copy for `trigger: project_empty`; the
     ledger-tools 🔓 glyph on pulled-in links.

## Deliberately not doing

- **Auto-releasing a pulled-in `^prj` when its project gains a task.** It would need
  sync or a hook to lower Next, which `decisions:task-lanes-are-sticky` forbids. Alt+N
  or closing R answers it.
- **Treating Blocked or future-scheduled residents as "not workable"**, or changing
  `bob projects sync`'s surface rule. Sync's behaviour stays byte-identical.
- **Reading child notes' `parent:` for SUBOPEN**, or the sub-project cascade.
- **Pulling in on raw clicks, `bob task reconcile`, `bob projects sync`, reopen or `=*`
  parks**, and linking on cancel.
- **Out of scope:**
  - Bob Mac Capture code;
  - bob-ledger-tools;
  - glossary strands;
  - Rust reopen undo;
  - `recovery.rs`.
- **Splitting `pomodoro_close.rs` / `output.rs`.** New code stays out of them where it
  can.
- **Pre-existing failures and 6u follow-ups:** bob-cli-6m, -6v, -6w, -3w, -5l, -6y, -6z,
  -70, -71, -72, -73.

## Risks

| Risk                                              | Mitigation                                                                                                                                              |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pull-in and sync disagree, so sync re-hides R     | One shared Rust helper for recognition, schedule and recount; the cycler mirrors it under PP vectors; PP23 runs real sync after a pull-in               |
| Stale Sub-projects marker                         | Both outcomes are benign and documented; the marker is sync-maintained every 15 minutes                                                                 |
| Extra reads slow every close                      | Detection uses already-staged text; the snapshot and index are built only per the Gate; `cli-capture` measures; the cycler's zero-read test stays green |
| `successors.rs` / `135` / `085` size limits       | New modules (`project_pull_in.rs`, `089`, `137`); `135` growth offset by moving code                                                                    |
| `[unblocked::]` silently skipped on a pulled-in R | Scope computed after `#hide` removal; PP27 pins it in both engines                                                                                      |
| Mixed plugin versions during rollout              | nav renders rows with and without `trigger` and the legacy reason kinds; Mac gets bob before plugins                                                    |
| MacBook unreachable again                         | Best-effort detached install; record exactly what is left for Bryan                                                                                     |
