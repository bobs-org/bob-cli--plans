---
tier: epic
title: Capture links a dependency-blocked task by dropping its open prerequisites
goal: 'Typing an existing-task link gesture (`@route:id`, `^route:id`, `@route+id`,
  `@route+id!`, `@route:id=x…`) in `bob capture` or Bob Mac Capture on a task that
  is Blocked only by its own open prerequisites removes those prerequisites from its
  Depends-On line and then performs the gesture, while every case where dropping is
  unsafe keeps today''s consumption and says why.

  '
decisions:
  mac_guard_detail:
    ask: Should Bob Mac Capture also show bob's not-linked sentence when a link gesture
      still cannot link?
    default: true
    why: a refused link renders as a generic preview today, which reads as "nothing
      happened"
    answer: true
  memory_supersede:
    ask: Should publish add the capture dependency-drop decision record and mark the
      two records it partly supersedes?
    memory:
    - decisions:projects-block-from-prj-line
    - decisions:task-deps-are-depends-on-links
    - decisions:capture-link-drops-blocking-deps
    default: true
    corrects: a gesture that would link one raises its open prerequisites instead
      (or reports a project date)
    answer: true
phases:
- id: cli-core
  title: Contract, dependency-drop writer, and the solo link gesture
  depends_on: []
  size: medium
  description: 'cli-core: write the §13.5 dependency-drop contract and PDG18–PDG27
    vectors first, then add the drop verdict, the staged drop writer, the shared guard
    helper, JSON/human output, and wire `@route:id` / `^route:id` (with `#name` and
    `=<X>`).'
- id: cli-gestures
  title: Toggle, Ensure Next, and link-then-close adopt the drop
  depends_on:
  - cli-core
  size: medium
  description: 'cli-gestures: route `@route+id!`, `@route+id[#name]`, and `@route:id=x…`
    through the shared guard helper, print the drop line in their human output, and
    convert their PDG1 tests into drop tests plus refused-drop consumption tests.'
- id: mac
  title: Bob Mac Capture presents dropped and refused prerequisites
  depends_on:
  - cli-core
  size: medium
  description: 'mac: decode `dropped_dependencies` and `link_guard`, add the dropped-dependency
    chip to the link, toggle, Ensure Next, and close presentations, and (per mac_guard_detail)
    render bob''s not-linked sentence; verify through macOS CI.'
- id: publish
  title: Decision record, supersession marks, and docs sweep
  depends_on:
  - cli-gestures
  - mac
  size: small
  description: 'publish: sweep the docs for stale consumption wording and, under memory_supersede,
    add the new decisions record, mark the two records it partly supersedes, and run
    `sase memory init`.'
proposed_by: bbugyi200.apollo.7c.w0
decided_by: auto
create_time: 2026-10-13 08:12:06
status: done
bead_id: bob-cli-7b
---

- **PROMPT:** [prompts/202610/capture_link_drops_blocking_deps.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_link_drops_blocking_deps.md)
- **BEAD:** [bob-cli-7b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7b/README.md)

# Plan: Capture links a dependency-blocked task by dropping its open prerequisites

## Context

Bryan asked: when an `@file:id` link gesture in `bob capture` (or in Bob Mac Capture,
which only spawns `bob`) targets a Blocked task with one or more open dependencies,
"remove the dependency and complete the operation when it is safe to do so. I think we
always fail to do anything currently."

What happens today (bob-cli `576a3e0`):

- Every existing-task link gesture calls `assess_link_target` in
  `src/native/capture/link_guard.rs`. It derives the target's reasons from the staged
  dependency graph (`local_dependency`, `local_schedule`, `project_dependency`,
  `project_schedule`) and returns `Link`, `PullForward` (only its own single future
  `scheduled`), `Consume`, or the PDG6 refusal for a date-only `^prj`.
- On `Consume` (`docs/task-dependencies.md` §13.5 gesture rule, vector PDG1) the gesture
  does not link. The task stays `[?]`, its open prerequisites are promoted to Next, and
  the result carries `link_guard` (`reasons`, `promoted`, `detail`). The standalone
  `@route:id` form even reports the neutral kind `"link_guard"`. `=<X>` never starts its
  session.
- Bob Mac Capture decodes none of `link_guard`. `CapturePomodoroLinkPresentation` only
  matches kind `pomodoro_link`, and `CaptureTogglePresentation` returns `nil` without
  `toggle_direction`. A guarded gesture therefore renders as a generic preview, and
  submitting it only raises the prerequisites' checkboxes: the target is never linked or
  started. That is the "we fail to do anything" Bryan sees.

The four call sites are:

| Gesture                                                     | Planner                                                                 |
| ----------------------------------------------------------- | ----------------------------------------------------------------------- |
| `@route:id` / `^route:id`, optionally `#name` and/or `=<X>` | `plan_pomodoro_link_capture`, `src/native/capture/pomodoro_link.rs`     |
| `@route+id!` (link direction)                               | `plan_task_toggle_capture`, `src/native/capture/task_toggle.rs`         |
| `@route+id`, `@route+id#name` (Ensure Next)                 | `plan_ensure_next_capture`, `src/native/capture/ensure_next.rs`         |
| `@route:id=x…` / `^route:id=x…` (link-then-close)           | `plan_pomodoro_close_link_item`, `src/native/capture/pomodoro_close.rs` |

Governing records (read them with `sase memory read decisions:<slug> -r "<why>"` before
changing behavior): `projects-block-from-prj-line`,
`subprojects-inherit-project-blocking`, `task-deps-are-depends-on-links`,
`task-status-is-derived`, `task-lanes-are-sticky`,
`unblocked-tasks-resurface-as-ticklers`, and `mac-capture-is-a-thin-client`. This epic
changes the capture half of the §13.5 gesture rule for one case only. "Blocked tasks
keep no Task Links under today's open Pomodoros" stays true: the task is unblocked
before it is linked.

## Design

### 1. The rule: dependency drop

When a capture link gesture targets an existing task that is Blocked **only by its own
open prerequisites** (optionally together with its own single future `scheduled`, which
PDG2 already pulls forward), capture:

1. removes those open prerequisites from the task's Depends-On line and `[dependsOn::]`
   field, then
2. performs the gesture exactly as it would for an unblocked target. The task rises
   `[?]` → `[*]`; it is linked, moved, started, or linked-and-closed; and it is
   reported.

The drop is a capture-only behavior. bob-plugins gestures (Ctrl+Shift+Enter, `^^`) keep
PDG1 unchanged.

### 2. The safety gate

The drop runs only when **every** condition below holds. Otherwise today's behavior runs
unchanged: PDG1/PDG3/PDG15 consumption, or the PDG6 refusal. When `local_dependency` was
among the reasons, the refusal also reports why the prerequisites were kept (§5).

| #   | Condition                                                                                                                                                                                                                                                                                                                                                                   | Refusal code (`drop_refused`) |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| G1  | The reasons include `local_dependency`, so there is something to drop.                                                                                                                                                                                                                                                                                                      | — (no drop, no code)          |
| G2  | There is no project reason: no `project_dependency` / `project_schedule` from its own project or an ancestor, and no tag-proxy inheritance (`inherit_blocks`). Dropping its own links would not unblock it, and project prerequisites belong to the Task Card's subtree pass.                                                                                               | `project_blocked`             |
| G3  | The target is not a recognised project's `^prj`. Its prerequisites block every resident and cascade to sub-projects: that is a project-level edit for the Task Card Depends on stage, which runs the subtree pass (§13.10).                                                                                                                                                 | `project_task`                |
| G4  | Any own future `scheduled` has the single-field shape `plan_task_link` retires (the PDG2 gate `task_update_has_single_future_scheduled`), so the same gesture pulls it forward. Several `scheduled` fields would still block it.                                                                                                                                            | `schedule`                    |
| G5  | The dependent has exactly one managed Depends-On child (`managed_children` in `src/native/capture/dependencies.rs`), and it parses `Accepted`. Every prerequisite being dropped resolves from a link on that line. Several managed lines or a malformed line are refused, as the `&` writer refuses them.                                                                   | `dependency_line`             |
| G6  | The task line is unchanged between assessment and write. Re-look it up by block ID against the staged text and compare digests, as `plan_existing_task_dependencies` does. A mismatch is an error, not a refusal.                                                                                                                                                           | — (error)                     |
| G7  | The drop is staged in the same batch planner as the rest of the gesture. Any later failure rolls the whole batch back, so nothing is ever dropped unless the gesture lands. Examples: `=<X>` while a timed Pomodoro runs, no running session for link-then-close, a missing daily note, a duplicate-link invariant, or another item failing. Dry runs plan the same writes. | — (batch rollback)            |

When several conditions fail, report the first failing one in table order.

### 3. What the drop writes

The drop is one staged edit to the dependent's note, made before `plan_task_link` runs:

- **Line.** Remove every link that resolves to one of the open own prerequisites.
  Resolve with the staged note index:
  `index.resolve(Some(dep_rel), link.target.trim())`, compared as
  `(display_relative(resolved), block_id)`. Duplicates of a dropped identity go too.
  - Kept links keep their verbatim text (`verbatim_link`) and order: Done/Cancelled and
    archived prerequisites (history chips, §5 History), unresolved links (R4), non-task
    links (R5), and self links (R6).
  - Rebuild the line with `task_status_hooks::rebuild_child_line`. When no link remains,
    delete the line (§2.1 empty list). Preserve line endings and the final-newline
    state.
- **Field.** Remove from `[dependsOn::]` every id that names a dropped prerequisite: its
  `[id::]` when it has one, else the canonical
  `task_dependencies::dependency_id(path, block)`. Keep every other id (closed history,
  R4 heal breadcrumbs, legacy-covered ids).
  - Write with `task_status_hooks::set_task_fields(line, existing_id, &kept)`, passing
    only the dependent's existing `[id::]`. Never mint one.
  - An empty kept list removes the field.
- **Prerequisites are never edited.** Their `[id::]`, status, and lane stay (sticky
  lanes).
- **Freshness.** When the drop rewrites the task line, stamp it with `stamp_fresh`, as
  the `&` writer does for an edited dependent (`docs/freshness.md` §5). The following
  `plan_task_link` stamp is then a same-day no-op. Write no `[unblocked::]`: the task
  lands in Next, so UB4 applies.
- **Then proceed.** Re-read the staged note and re-look up the task line. Then run the
  unblocked path unchanged: `plan_task_link` (status, single-schedule retirement plus
  Schedule Log), then the ledger insert, move, start, or close.

### 4. One shared guard helper

`cli-core` adds the helper; `cli-gestures` adopts it at the other three call sites.

- Replace each call site's
  `if let LinkGuardVerdict::Consume(plan) = assess_link_target(…)` with
  `resolve_link_guard(…)`, in `link_guard.rs` or a new `dependency_drop.rs`. It returns
  either:
  - `Proceed { dropped: Vec<DroppedDependency>, task_block_ref: Option<TaskBlockRef> }`,
    for `Link`, `PullForward`, or a successful drop; or
  - `Consume(Box<ConsumePlan>)`, carrying an additive
    `drop_refused: Option<&'static str>`.
- Add a `LinkGuardVerdict::DropDependencies(Box<DropPlan>)` variant, classified inside
  `assess_link_target` from values it already computes: `own_open`, `inherited_open`,
  `prj_scheduled`, `inherit_blocks`, `is_prj`, and the single-schedule check. `DropPlan`
  carries the open identities, the staged graph, and the `ConsumePlan` to fall back to.
  The writer's G5 refusal downgrades to that fallback with
  `drop_refused: "dependency_line"`.
- The drop reuses the graph and index the assessment already built. No second vault-wide
  scan, and the graph-free fast path for unblocked targets stays (the existing
  `graph_build_count` tests must keep passing).

### 5. Outputs

**Successful drop:**

- **Per item, `dropped_dependencies`.** An array in line order. Each row is
  `{note_path, block_id, link, text, status_symbol, status_name}`:
  - `note_path`: the prerequisite's vault-relative path with its extension;
  - `link`: the link exactly as it stood on the line;
  - `text`: clean display text.

  The key is omitted when nothing was dropped. It appears on kinds `pomodoro_link`,
  `task_toggle` (link direction and Ensure Next), and `pomodoro_close`
  (link-then-close).

- **The gesture reports as linked.** `link_guard` is absent, and the kind is the
  gesture's own, never `"link_guard"`. `previous_status_symbol` `?` → `status_symbol`
  `*`, `status_changed: true`, and `previous_task_line` is the pre-drop line.
- **Batch `task_blocks`.** It gains the dependent with the new role
  `"dependency_dropped"` (`TaskBlockRole::DependencyDropped` in
  `src/native/capture/task_blocks.rs`). Its cumulative diff shows the removed or changed
  Depends-On line and the changed task line.
- **Human output.** One dim line after the status line:
  - `⛓️ dropped dependency on [[job#^prep]]`, or
  - `⛓️ dropped 2 dependencies: [[job#^prep]], [[cash#^budget]]`.

  Dry run: `would drop`.

**Refused drop.** The existing `link_guard` gains the additive `drop_refused` code from
§2, only when `local_dependency` was a reason. Its `detail` sentence gains a suffix
` · dependencies kept: <phrase>`:

| Code              | Phrase                                                   |
| ----------------- | -------------------------------------------------------- |
| `project_blocked` | `its project is blocked`                                 |
| `project_task`    | `edit a project's prerequisites from its ^prj Task Card` |
| `schedule`        | `several [scheduled::] fields still block it`            |
| `dependency_line` | `repair its Depends-On line first`                       |

Dry-run JSON equals real-run JSON except for `dry_run`. Everything is additive: older
clients ignore the new keys, and they now see an ordinary successful link instead of a
`link_guard` kind.

### 6. New gesture vectors (add to `docs/task-dependencies.md` §13.15)

| #     | Gesture / situation                                                                                         | Outcome                                                                                                                                                                   |
| ----- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PDG18 | capture `@route:id` (also `^route:id`, `#name`, `=<X>`) on a target whose only reason is `local_dependency` | open prerequisite links dropped from the line and field; `[?]` → `[*]`; linked (and started for `=<X>`); `dropped_dependencies` listed; prerequisite notes byte-identical |
| PDG19 | the same target through `@route+id!`, `@route+id` / `@route+id#name` Ensure Next, and `@route:id=x…`        | dropped, then linked, relocated, or linked-and-closed exactly like an unblocked target (`=x1` leaves it `[/]`, `=x!1` completes it)                                       |
| PDG20 | the dropped links were the line's only links                                                                | line deleted; `[dependsOn::]` removed                                                                                                                                     |
| PDG21 | own open prerequisites plus a single own future `scheduled`                                                 | both retired in one write (PDG2 pull-forward plus the drop); linked                                                                                                       |
| PDG22 | the line also holds a Done prerequisite and an unresolved link                                              | the Done and unresolved links are kept verbatim in order; their field ids are kept                                                                                        |
| PDG23 | own open prerequisites, and the task is a resident of a blocked project (own or inherited reason)           | nothing dropped; PDG1/PDG15 consumption; `drop_refused: project_blocked`                                                                                                  |
| PDG24 | a `^prj` target with an open prerequisite                                                                   | nothing dropped; PDG1 consumption; `drop_refused: project_task`                                                                                                           |
| PDG25 | two managed Depends-On lines, or several `scheduled` fields                                                 | nothing dropped; consumption; `drop_refused: dependency_line` or `schedule`                                                                                               |
| PDG26 | `^route:id=` while a timed Pomodoro runs, or a later item in the batch fails                                | the batch fails; nothing is dropped; the dry run reports the same failure                                                                                                 |
| PDG27 | plugin Ctrl+Shift+Enter or `^^` on a dependency-blocked target                                              | unchanged PDG1; plugins never drop                                                                                                                                        |

Reword PDG1 to name the plugin gestures plus the capture cases refused by §2.

### 7. Deliberately not doing

- **bob-plugins gestures.** Ctrl+Shift+Enter, `^^`, and the Task Card keep PDG1. Parity
  is a separate request.
- **Dropping a `^prj`'s prerequisites, or anything for a project-blocked target.** See
  G2 and G3.
- **New-task captures.** `<text> @route:id`, `&`, `s:N`, project-note `:id` bullets, and
  new tasks in blocked projects keep their behavior: prerequisites typed with `&` in the
  same item are explicit intent.
- **`!note:id` completion.** It already accepts Blocked roots.
- **Hints in the `:`, `+`, or `^` pickers.** No hint that linking will drop
  dependencies, for example a `drops_dependencies` candidate flag. The live preview
  shows it before submit. Record a follow-up if it seems wanted.
- **History for dropped prerequisites.** Nothing records them on the task: the writer
  form forbids struck links, so the output reports the drop and vault git keeps history.
- **Reconcile changes.** None are needed. Reconcile reads the line and the field, which
  are exactly what the drop edits, and it keeps the final word.

## Cross-cutting rules for every phase

- **Contract first.** `docs/task-dependencies.md` §13.5 and §13.15 own the rule and its
  vectors. If an implementation disagrees with a vector, fix the doc first in the same
  phase and say so in the commit.
- **No live runs on Bryan's vault.** Never run live `bob capture`, `bob task reconcile`,
  or `bob projects sync` against `~/bob`. Use temporary vaults in tests; dry runs are
  fine.
- **Verification.** bob-cli phases finish green on `just check` (fmt, clippy, every test
  binary). If a failure looks unrelated, rerun it on a clean base, cite it, and finish.
- **Thin client.** The Mac phase never decides grammar or computes the drop
  (`decisions:mac-capture-is-a-thin-client`). It renders bob's fields, and its chip
  wording mirrors bob's human line.
- **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on your own phase bead instead of
  filing beads.
- **Memory.** Only the `publish` phase edits memory, and only under accepted memory
  decisions.

## Phase: cli-core — contract, dependency-drop writer, and the solo link gesture

**Repo:** bob-cli.

1. **Contract** (land in the same commit as the code):
   - `docs/task-dependencies.md`:
     - §5 Consumption bullet: add the capture drop exception, pointing at §13.5.
     - §13.5 gesture rule: a new first bullet for the capture dependency drop.
     - New §13.5.1 "Capture dependency drop": §1–§5 of this plan (gate G1–G7, writes,
       outputs, refusal codes and phrases).
     - §13.15: PDG1 reworded and PDG18–PDG27 added (§6).
   - `docs/capture.md`:
     - the grammar table rows for the no-text `@route:block-id[#pomodoro][=<X>]`,
       `^route:…`, `@route+block-id`, `@route+block-id!`, and `@route:block-id=x…`
       forms;
     - "Linking and starting existing tasks": the guard paragraph and the JSON
       paragraph;
     - the Task Link toggle table row (the "§13.5 link guard finds the target Blocked"
       row) and the Ensure Next paragraph;
     - the link-then-close guard paragraph in "Closing the running Pomodoro";
     - the JSON output section's link-guard paragraphs: `dropped_dependencies`,
       `drop_refused`, and the detail suffix;
     - the Task blocks role list: `"dependency_dropped"`.

     Write every gesture's behavior here now, so that `cli-gestures` implements against
     a finished contract.

   - `docs/freshness.md` §5: add the drop to the `bob capture` stamps row.

2. **Verdict.** Add `LinkGuardVerdict::DropDependencies` and its classification in
   `assess_link_target` (§4). Add `drop_refused` to `ConsumePlan`. Extend the unit tests
   in `link_guard.rs` to cover each G1–G5 branch, and keep the existing
   `graph_build_count` tests green.
3. **Writer.** Add `plan_dependency_drop` (new `src/native/capture/dependency_drop.rs`
   or alongside the guard) implementing §3.
   - Reuse `managed_children`, `verbatim_link`, `current_note_text`,
     `lookup_staged_task`, and `staged_note_index`; widen visibility to `pub(super)`
     where needed.
   - Unit-test:
     - partial removal keeping order and verbatim kept links;
     - last-link line deletion plus field removal;
     - removing field ids for targets with and without `[id::]`;
     - R4 breadcrumbs kept;
     - CRLF and final-newline preservation;
     - the G5 refusals.
4. **Helper and output.**
   - Add `resolve_link_guard` (§4).
   - Add `DroppedDependencyJson` and `CaptureItemResult.dropped_dependencies`
     (`skip_serializing_if` empty).
   - Add `LinkGuardJson.drop_refused`, plus the detail suffix in
     `LinkGuardJson::not_linked` or a sibling constructor.
   - Add `TaskBlockRole::DependencyDropped` (serialized `"dependency_dropped"`). Thread
     the dependent's `TaskBlockRef` into the item's `task_block_refs` (currently
     `Vec::new()` for these kinds in `src/native/capture/plan.rs`).
   - Add a shared human printer for the `⛓️ dropped …` / `would drop` line.
5. **Wire the solo link** in `plan_pomodoro_link_capture`:
   - After a drop, re-read the staged contents and re-look up the task before
     `plan_task_link`. The `=<X>` start and `#name` paths then run unchanged.
   - In `plan.rs`, keep kind `pomodoro_link` for a drop. Kind `"link_guard"` is only for
     `Consume`.
   - Print the drop line from the `pomodoro_link` human printer in
     `src/native/capture/output.rs`.
6. **Tests** (`tests/cli/capture/link_guard.rs`, plus
   `tests/cli/capture/pomodoro_link.rs`):
   - Integration tests for PDG18, PDG20, PDG21, PDG22, PDG23, PDG24, PDG25, and PDG26
     through `@route:id` / `^route:id` (one of them with `=` start, one with `#name`).
   - In each, assert:
     - the dependent note's exact bytes;
     - byte-identical prerequisite notes;
     - the ledger;
     - `dropped_dependencies`, the `task_blocks` role, and `drop_refused` / `detail`;
     - dry-run JSON equal to real-run JSON apart from `dry_run`.
   - Rewrite `capture_pomodoro_link_on_blocked_target_consumes_and_promotes`
     (`tests/cli/capture/pomodoro_link.rs`): it becomes PDG18. Move its consumption
     assertions onto a project-blocked resident so PDG1 consumption keeps coverage.
   - The other three call sites still consume until `cli-gestures`. Their existing PDG1
     tests must stay green in this phase.

## Phase: cli-gestures — toggle, Ensure Next, and link-then-close adopt the drop

**Repo:** bob-cli. Implements against the contract `cli-core` wrote; fix the doc first
if it is wrong.

1. **`@route+id!`, link direction** (`task_toggle.rs`). Replace the `Consume` branch
   with `resolve_link_guard`. On `Proceed`, re-read the staged note and re-look up the
   task line before `plan_task_link`. Print the drop line from
   `print_human_link_toggle_summary`.
2. **Ensure Next, `@route+id[#name]`** (`ensure_next.rs`). Same, ahead of
   `plan_task_link` and the relocation. Keep `toggle_behavior: "ensure_next"`.
3. **Link-then-close, `@route:id=x…`** (`plan_pomodoro_close_link_item`,
   `pomodoro_close.rs`).
   - On `Proceed` after a drop, take the normal (non-history) path: `plan_task_link`
     flips the task to Next, and the close's keep/park/complete/drop selection applies
     to it like any linked task.
   - `Consume` keeps today's history-link behavior.
   - Report `dropped_dependencies` on the `pomodoro_close` item. Thread the task block
     ref.
4. **Results and printers.** The `task_toggle` and `pomodoro_close` result builders in
   `plan.rs` and `pomodoro_close.rs` emit `dropped_dependencies` and the `task_blocks`
   ref. Every human printer for these gestures prints the drop line.
5. **Tests:**
   - Convert the three PDG1 tests into PDG19 drop tests, each with a sibling
     refused-drop test (a project-blocked resident) that asserts today's consumption and
     `drop_refused: project_blocked`:
     - `capture_task_toggle_on_blocked_target_consumes_and_promotes`
       (`tests/cli/capture/task_toggle.rs`);
     - `capture_ensure_next_on_blocked_target_consumes_and_promotes`
       (`tests/cli/capture/ensure_next.rs`);
     - `capture_pomodoro_close_link_on_blocked_target_keeps_history`
       (`tests/cli/capture/pomodoro_close.rs`).
   - Add a link-then-close drop with `=x1` (`[/]`) and with `=x!1` (completed).
   - `promotion_walks_through_a_blocked_seed_without_raising_it` (a project reason)
     stays as it is.
   - Keep the existing pull-forward tests in `tests/cli/capture/link_guard.rs` green.

## Phase: mac — Bob Mac Capture presents dropped and refused prerequisites

**Repo:** bob-mac-capture (open it with `sase repo open bob-mac-capture`; read its
`README.md` first). Depends only on the JSON contract from `cli-core`.

1. **Decode** in `Sources/CaptureCore/CaptureModels.swift` (`CaptureCommandSuccess`),
   tolerating older bob:
   - `dropped_dependencies` → `[CaptureDroppedDependency]` (`decodeIfPresent`, default
     `[]`);
   - `link_guard` → `CaptureLinkGuard?` (`reasons`, `promoted`, `detail`,
     `dropRefused`).
2. **Chips.** Add one chip to `CapturePomodoroLinkPresentation`,
   `CaptureTogglePresentation` (link and Ensure Next), and the link row of
   `CapturePomodoroClosePresentation`:
   - `dropped dependency on <text>` for one prerequisite;
   - `dropped N dependencies` for more.

   Include it in each presentation's notification body and accessibility summary. The
   status transition already comes from bob's `[?]` → `[*]` fields.

3. **README.** Update the `@route+block-id!` / Ensure Next / link paragraphs to say that
   a task Blocked only by its own prerequisites drops them and links, and how refusals
   read.
4. **Tests** (`Tests/CaptureCoreTests`):
   - `CaptureModelTests` decoding with and without the new keys;
   - the link, toggle, and close presentation tests for the chip.

   Build the JSON fixtures from real `bob capture --dry-run -f json` output against a
   temporary vault, using a bob built from `cli-core`.

5. **Not-linked presentation.** Its scope is set by `mac_guard_detail`:

   > [!decision] mac_guard_detail Add a small `CaptureLinkGuardPresentation` for results
   > that carry `link_guard`: kind `"link_guard"`, or a `task_toggle` without
   > `toggle_direction`.
   >
   > - Headline `Not linked` (dry run `Would not link`), with a `lock.fill` glyph.
   > - Bob's `detail` sentence verbatim, then the promoted prerequisites.
   > - Notification title `Not linked`.
   > - Footer enablement does not change: submitting still runs bob's consumption.
   > - Wire it into `CapturePanelView.swift` ahead of the generic preview, and cover it
   >   in `Tests/CaptureCoreTests`.

   > [!decision] mac_guard_detail = no Skip it: refused links keep today's generic
   > preview.

6. **Verification.** This Linux host has no Swift toolchain.
   - On macOS, run `just format-lint`, `just build`, and `just test`.
   - Otherwise, after the commit lands, check the macOS CI with
     `gh run list -R bobs-org/bob-mac-capture`. Wait with `/sase_monitor` instead of
     blocking, and fix any failures.
   - Record a manual check for Bryan: `^job:blocked` on a task Blocked by one open
     prerequisite previews the chip and links on Return.

## Phase: publish — decision record, supersession marks, and docs sweep

**Repo:** bob-cli.

1. **Docs sweep.** Grep `docs/`, `README.md`, and the bob-mac-capture `README.md` for
   wording that still says a capture link to a Blocked task "never links", "stays
   unlinked", or is "not linked" without the §2 qualifier. Fix each hit.
2. **What is left for Bryan.** Note the next steps for him in the phase notes: install
   the new bob and Bob Mac Capture on the MacBook through his usual install path. No
   vault migration is needed.
3. **Memory.** Use `/sase_memory_write` before touching memory. Its scope is set by
   `memory_supersede`:

   > [!decision] memory_supersede The record and both marks land together, in one
   > change.
   >
   > - **New record.** Add the decisions strand
   >   `sase/memory/decisions/capture-link-drops-blocking-deps.md`. Use the frontmatter
   >   shape of its peers: keyword "Capture Links A Dependency-Blocked Task By Dropping
   >   Its Prerequisites", aliases, a one-sentence `summary` usable as a roster rule,
   >   `metadata.status: accepted`, and `decided: <date the epic lands>`.
   >   - **Applies to.** bob-cli, Bob Mac Capture.
   >   - **Claim.** The §1 rule and the G1–G7 gate in brief. Plugins keep PDG1.
   >     `docs/task-dependencies.md` §13.5.1 owns the contract.
   >   - **Why.** Bryan's 2026-10-13 request, quoted. Consumption read as a failure in
   >     both clients. Rejected alternatives: keep consumption; drop for every reason,
   >     including project and `^prj` reasons; an opt-in sigil; strike dropped links as
   >     history (the writer form forbids strikes); plugin parity in the same change.
   >   - **Cost.** Dropped edges leave no trace on the task beyond output and vault git.
   >     Capture and plugins diverge. Project-blocked and `^prj` targets still refuse.
   >   - **Reopens when.** Bryan wants plugin gestures to drop too; he wants a `^prj`
   >     drop through the subtree pass; or dropped edges need a record on the task.
   >   - **Evidence.** The epic's plan ref and the `cli-core`, `cli-gestures`, and `mac`
   >     commits. Link `[[decisions:projects-block-from-prj-line]]` and
   >     `[[decisions:task-deps-are-depends-on-links]]`.
   > - **Mark `decisions:projects-block-from-prj-line`.** Make `metadata.superseded_by`
   >   a list that keeps `decisions:subprojects-inherit-project-blocking` and adds
   >   `decisions:capture-link-drops-blocking-deps`. Append one back-link sentence:
   >   capture link gestures on a task blocked only by its own prerequisites now drop
   >   them and link (`[[decisions:capture-link-drops-blocking-deps]]`). Leave the body
   >   otherwise untouched.
   > - **Mark `decisions:task-deps-are-depends-on-links`** the same way: set
   >   `metadata.status: superseded-in-part` plus `superseded_by`, and append one
   >   back-link sentence: linking a dependency-blocked dependent from capture drops its
   >   open prerequisites instead of raising them.
   > - **Regenerate.** Run `sase memory init` and check that the regenerated roster
   >   phrases stay short.

   > [!decision] memory_supersede = no Edit no memory. If no human reviewed the plan,
   > record a `PROPOSED FOLLOW-UP:` note on this phase bead naming the record and the
   > two marks. If a reviewer turned the decision off, record nothing.

## Risks

| Risk                                                                                    | Mitigation                                                                                                                                    |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| A prerequisite edge disappears without the user noticing                                | The Mac live preview shows the chip and the `task_blocks` diff before submit. CLI output prints `⛓️ dropped …`. Vault git keeps history       |
| CLI habits relied on the old "link a Blocked task to promote its prerequisites" gesture | Documented in capture.md and the new record. Linking the prerequisites directly still promotes them. Project-blocked targets keep consumption |
| A partial write: dropped, but not linked                                                | G7: one staged batch, so any later failure rolls everything back. PDG26 tests it                                                              |
| An earlier item in the same draft adds `&x` and a later item links the same task        | Items run in order against the staged snapshot, so the later link drops it. Documented in §13.5.1 as ordinary batch semantics                 |
| Extra cost on every link preview                                                        | The drop reuses the assessment's graph and index. Unblocked targets keep the graph-free fast path (`graph_build_count` tests)                 |
| Older Mac app against the new bob                                                       | Additive keys. The kind is now the ordinary link kind, so the old app shows a normal link preview, without the chip                           |
