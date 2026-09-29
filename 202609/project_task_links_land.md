---
tier: tale
size: medium
title: Finish and land epic bob-cli-2n (named and linked project tasks)
goal:
  The epic-caused defects found while landing bob-cli-2n are fixed and tested, the stale
  README/docs/comments/help text is refreshed, and epic bob-cli-2n is closed with its
  plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-2n.land
bead: bob-cli-2n
create_time: 2026-09-29 17:59:54
status: wip
---

- **PARENT:**
  [202609/project_task_links.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/project_task_links.md)
- **BEAD:**
  [bob-cli-2n](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2n/README.md)

# Plan: finish and land epic bob-cli-2n

## Context

Epic `bob-cli-2n` (plan `plan:202609/project_task_links.md`) added named and linked
project tasks to `bob capture`: `@route^id+[#pomodoro]` project notes whose first-level
bullets end in ` :id` (named, Next, and linked into a Pomodoro) or ` ^id` (named only).
It also retired the `@route:id+` forms. All six phase beads (`bob-cli-2n.1` to `.6`) are
closed. The epic's bob-cli commits are `65f5b43`, `e9c4dae`, `5f6c761`, `301e809`, and
`42cd336`. The bob-mac-capture commit is `ff41276`, and its macOS CI run `36634195459`
passed.

The land agent confirmed the following:

- `cargo test` passes: 1192 lib tests, 544 CLI tests, and every other target.
- `cargo fmt --check` is clean.
- `sase bead epic-symbols bob-cli-2n` lists nothing.
- The only non-epic commit since the epic started is `afb2e5c` (the bob-cli-2k close
  selection landing). It landed before the epic's first commit, so it is already
  incorporated.

The land audit found the defects below. The epic caused all of them, so this tale fixes
them and then closes the epic.

Follow-up triage is already done and recorded on `bob-cli-2n` (note "LAND TRIAGE"):

- The clippy deny at `tests/cli/capture/pomodoro_name.rs:808` belongs to active epic
  `bob-cli-28`.
- The clippy warnings are tracked by `bob-cli-v`.

Do **not** fix either one here, and do not create task beads for them.

Bob Mac Capture needs no change: it renders Bob's `pomodoro_name`, diagnostics, and
modes as-is. All work below is in this bob-cli repository.

## 1. Report the resolved Pomodoro name for project-note Task Links

**Defect.** In `src/native/capture/project_note.rs` (the `linked` branch, around the
`canonical_pomodoro_name` / `creates_pomodoro` computation), `pomodoro_name` is set to
`canonicalize_pomodoro_name(selector)`, which is just the typed text upper-cased.
Selectors match an open Pomodoro by slug or by slug prefix
(`capture_pomodoros::select_named`). So with an existing open `- [ ] () — ADMIN`:

- `Finish it @cash^y+#adm` + `- Draft :dr` correctly links under `ADMIN`
  (`creates_pomodoro: false`).
- But it reports `"pomodoro_name": "ADM"`, and the human output prints `under ADM`.

The `@route:id#name` flow (`src/native/capture/pomodoro_link.rs`) reports the matched
`entry.name`. The docs (`docs/capture.md`, the JSON contract) promise the resolved
canonical destination name.

**Fix.** Call `select_named(&scan, selector)` once on the original day contents:

- `Found(entry)`: report `entry.name` (fall back to the canonicalized selector if it is
  `None`) and set `creates_pomodoro = false`.
- `CompletedOnly` / `Missing`: report the canonicalized selector and set
  `creates_pomodoro = true`.

**Test.** In `tests/cli/capture/project_note.rs`, use a daily note that already has an
open `ADMIN` placeholder and the input `@cash^y+#adm` with a ` :dr` bullet. Assert:

- JSON has `pomodoro_name == "ADMIN"` and `creates_pomodoro == false`;
- the link lands under the existing `ADMIN` entry and no second `ADMIN` is created;
- human output shows `under ADMIN`.

## 2. Keep project-note `=` wording off `^` tokens that have no project-note `+`

**Defect.** After `65f5b43`, any `@route^…` token containing `=` gets the project-note
start/close message (`POMODORO_START_PROJECT_NOTE_ERROR` /
`POMODORO_CLOSE_PROJECT_NOTE_ERROR`, editor code `invalid_project_note_marker`). For
example, `Do thing @dev^foo=3` now says "…not project-note `@<route>^<block-id>+`
forms", although there is no `+`.

The epic plan limited that message to "a `=` anywhere after the `+`". Before the epic,
this token got `TASK_BLOCK_ID_ERROR` in execution and the `invalid_task_block_id`
diagnostic in the editor. See `git show afb2e5c:src/native/capture_language/tokens.rs`
and `…/editor_classify.rs`.

**Fix.** Make the change in both places:

- execution: `parse_task_block_id_route_token` in
  `src/native/capture_language/tokens.rs`, the `block_id.contains('=')` check after the
  `#` branch;
- editor: `classify_task_block_id_token` in
  `src/native/capture_language/editor_classify.rs`, the `block_part.contains('=')`
  check.

Use the project-note `=` messages only when the text before the first `=` ends with the
project-note `+` (e.g. `@dev^foo+=3`, `@dev^foo+=x`). Otherwise fall through to the
pre-epic `TASK_BLOCK_ID_ERROR` / `invalid_task_block_id`.

**Tests.** In `capture_language/tests/grammar.rs` and `tests/editor_modes.rs`:

- `Do thing @dev^foo=3` and `Do thing @dev^foo=x` get the task-block-ID error and code.
- `@cash^goog-exit+=3` and `@cash^goog-exit+#bugs=x` keep their project-note messages.

## 3. Reject a `^` task whose only text is a checkbox

**Defect.** In `ProjectTaskPass::check_child`
(`src/native/capture_language/project_tasks.rs`), the empty-body check
(`stripped.split_whitespace().next().is_none()`) runs on text that still holds the
authored leading checkbox. So `- [x] ^foo` and `- [ ] ^foo` are accepted, and the
renderer writes a task with no text. Rule 4 says an empty remaining body gets
`invalid_project_task_id` ("line N names a task but has no task text").

**Fix.** Strip the leading checkbox with the shared `split_leading_checkbox` before the
emptiness test. Keep the evaluation order (empty body, then checkbox, then duplicate),
so `- [x] :foo` now reports the empty-body error. The pass is shared, so both execution
and the editor change.

**Tests.** A grammar test for `Finish it @cash^x+\n- [x] ^foo` checks the exact message
and line number. An `editor_modes.rs` row checks the diagnostic code and the ID-token
range.

## 4. No `unused_project_note_pomodoro` diagnostic while a ` :` ID is unfinished

**Defect.** In `src/native/capture_language/editor_parse.rs` (the project-task
post-pass), `unused_project_note_pomodoro` fires whenever
`section.is_some() && !pass.has_link`. So `Finish it @cash^x+#admin\n- Draft :` reports
`mode: "incomplete"` and `needs: ["block_id"]` plus that error diagnostic. Rule 5 says
an unfinished ID is an incomplete state with no diagnostic.

**Fix.** In the editor, count a first-level
`ChildTaskOutcome::Unfinished { sigil: ':' }` as a pending link for the unused-name
rule. A lone `^` still does not count, per Rule 7. Execution already fails the lone
sigil first, so it does not change.

**Tests.** Add `editor_modes.rs` rows:

- `…#admin\n- Draft :` is incomplete, needs `block_id`, and has no diagnostics;
- `…#admin\n- Draft ^` still reports `unused_project_note_pomodoro`.

## 5. Make `bob capture` reject the route-less retired `@:<id>+` like the editor does

**Defect.** For `Finish it @:goog-exit+`, the editor (`classify_pomodoro_token` in
`editor_classify.rs`) reports `retired_project_note_marker` with the `<route>` message
from the plan. `bob capture`, however, keeps the token as literal text and writes an
inbox task. The reason is that `is_pomodoro_marker_candidate` (`tokens.rs`) requires a
letter in the route.

**Fix.** Make execution reject `@:<block-id>+` and `@:<block-id>+#<name>` in marker
position, reusing the same retired-message builder in `markers.rs` that the editor uses.
Do this narrowly, for example in the terminal-marker validation in `tokens.rs`. `@:` and
`@:id` must stay literal in execution, so
`interactive_markers_are_the_only_divergence_from_execution` stays unchanged.

**Tests.**

- Grammar tests: `Finish it @:goog-exit+` and `Finish it @:goog-exit+#bugs` fail with
  exactly the editor's message.
- Add both inputs to an editor/execution agreement test.

## 6. Refresh stale text

- **`README.md`:**
  - The vault-layout row for `<route>_<id>.md` (drop "or `@route:id+`").
  - The capture grammar table rows for `@route:id+` / `@route:id+#pomodoro`. Replace
    them with a `@route^id+#pomodoro` row, add rows for a first-level bullet ending in
    ` :<task-id>` / ` ^<task-id>`, and add a retired `@route:id+` row.
  - The `+` paragraph after the `#` paragraph: `^prj` is never linked, and ` :id`
    bullets are linked into the current/next or `#pomodoro` Pomodoro.
  - Mirror the wording in `docs/capture.md`.
- **`docs/capture.md`:**
  - Around line 1962, remove `@:id+` from the "Incomplete interactive markers are valid
    input" list. It is now a `retired_project_note_marker` diagnostic, as the paragraph
    around line 2021 already says.
  - Around line 2281, drop "`/ @route:block-id+`" from the `@@` rewrite non-absorbable
    list.
  - Around line 2431, change the example to `@sase^x+`. `@sase^x+` at cursor 7 returns
    `task_block_id` with replacement `{6, 7}`; `@sase:x+` returns no context.
- **`src/native/capture_language/model.rs`:** the `CaptureKind::ProjectNote` doc comment
  wrongly says `pomodoro_name: None` "writes no Task Links". ` :` tasks link into the
  current/next Pomodoro when it is `None`, and `Some` names the Pomodoro.
- **`src/native/capture_language/completion.rs`:** around line 919, change the comment
  example `@route:id+#name` to `@route^id+#name`.
- **`src/native/capture/cli.rs`:** in the project-note help paragraph, drop the stale
  "once task bullets can name IDs".
- **`src/native/capture_project_note.rs`:** the module doc still lists "the
  Pomodoro-link flag" as an input. Replace it with the real inputs (for example, the
  authored sub-bullets with their task IDs).
- **`src/native/capture_block_ids.rs`:** the `build_block_id_field` doc names only
  `pomodoro_block_id` / `task_block_id`. Mention the `project_task_block_id` builder
  too.
- **`src/native/capture_complete.rs`:** in the help text, the Pomodoro-name completion
  sentence lists only the `:` forms. Add `@route^id+#prefix` and a bare `@route^id+#`,
  which already complete as `pomodoro_name`.

## 7. Tighten the plan-specified tests that are weaker than the plan

- **Renderer unit test** (`src/native/capture_project_note.rs`,
  `named_tasks_render_ids_last_with_next_status_for_links` or a new test): assert the
  plan's worked example byte for byte, including frontmatter, the `^prj` line,
  `## Tasks`, and `## Future Work`.
- **CLI test** `capture_project_note_task_links_write_the_worked_example`: assert the
  full new-note bytes and the full daily-note bytes, meaning exactly one
  `- [ ] () — ADMIN` with both links in source order.
- **Dry-run test:** assert that the dry-run JSON equals a real run's JSON except for
  `dry_run`, and that nothing is written.

## 8. Verify

- `cargo fmt --check` is clean.
- `cargo test` passes in full. If a pre-existing clippy deny ever blocks compilation,
  use `RUSTFLAGS=--cap-lints=warn`.
- `cargo clippy --all-targets --all-features`:
  - The only error allowed is the pre-existing `overly_complex_bool_expr` at
    `tests/cli/capture/pomodoro_name.rs:808` (owned by `bob-cli-28`).
  - There must be no warnings beyond the baseline of 17 lib warnings plus the cli-test
    `single_element_loop` at `src/native/capture_parse.rs:1117`.
- Rerun each defect's repro with `cargo run -q -- …` in a temp vault (`-b <vault>`,
  `BOB_DAY_FILE`, `BOB_NOW`, `TZ=UTC0`), and confirm the new behavior for items 1–5.
- `grep -rn ':goog-exit+\|:block-id>+\|@route:id+' README.md docs src` shows only
  retired-form explanations and tests.

## 9. Close out epic bob-cli-2n (final step)

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-2n`. For every listed
   `--epic-symbol` entry, resolve it: wire it up, privatize it, add a non-test pragma,
   or delete it. Re-key a Justfile line to a still-open bead only if that bead genuinely
   still needs the exemption. At planning time the list was empty.
2. **Close the epic** with `sase bead close bob-cli-2n --note "<verification>"`. The
   note should state:
   - All six phases were verified against the plan and commits `65f5b43`, `e9c4dae`,
     `5f6c761`, `301e809`, and `42cd336`.
   - Mac commit `ff41276` passed macOS CI run `36634195459`.
   - The only post-start non-epic commit, `afb2e5c`, predates the epic's commits and is
     incorporated.
   - The fixes from items 1–7, with this tale's commit, and the fmt/test/clippy results.
   - Triage outcomes: the pomodoro_name.rs:808 deny went to a DISCOVERED ISSUE note on
     `bob-cli-28` (proposed by `.1`–`.4`); `single_element_loop` got a +1 on `bob-cli-v`
     (proposed by `.5`); `.6` proposed nothing; the declined audit nits are listed in
     the epic's "LAND TRIAGE" note.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic symbols, finish that cleanup and close again.

3. **Symvision.** Run `just symvision` if the recipe exists. This justfile currently has
   none, so check with `just --list` and record that it is absent.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file. Use
   the PLAN path printed by
   `sase bead read bob-cli-2n -r "Need the plan path for closeout"`, which is
   `202609/project_task_links.md` in the plans repo. Change `status: wip` to
   `status: done`.
5. **Parent bead.** `bob-cli-2n` has no `parent_bead`. Confirm this in the
   `sase bead read` output; there is nothing further to close.
