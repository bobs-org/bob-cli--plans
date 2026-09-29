---
tier: epic
title: Named and linked project tasks with ` :id` in `bob capture` and Bob Mac Capture
goal: 'A project-note capture (`@route^id+` or `@route^id+#pomodoro`) can name any
  of its task bullets with a trailing ` ^id`, or name them and link them into the
  current/next (or named) Pomodoro with a trailing ` :id`. The `^prj` task is never
  linked, the retired `@route:id+` forms fail with a message that teaches the new
  spelling, and Bob Mac Capture highlights, completes, previews, and reports the new
  syntax.

  '
phases:
- id: marker
  title: 'Project-note marker grammar: `@route^id+#pomodoro`, retire `@route:id+`'
  depends_on: []
  size: medium
  description: 'marker: move the Pomodoro name onto the `^` project-note marker (`@route^id+#pomodoro`),
    retire the `:` project-note forms with teaching errors, stop linking or starring
    `^prj`, and keep execution, capture-parse, capture-complete, and help text in
    agreement.

    '
- id: task-id-grammar
  title: Project task IDs in the capture grammar and capture-parse
  depends_on:
  - marker
  size: medium
  description: 'task-id-grammar: lex trailing ` :id` / ` ^id` tokens on project-note
    lines, enforce the placement, validity, duplicate, checkbox, and unused-`#pomodoro`
    rules in both the execution and editor parsers, and expose spans, needs, modes,
    and `sub_bullet_task_ids` through capture-parse.

    '
- id: task-id-execution
  title: Render named project tasks and write their Task Links
  depends_on:
  - task-id-grammar
  size: medium
  description: 'task-id-execution: render `^id` onto named tasks (`[*]` for `:` tasks),
    write one Task Link per `:` task into the selected Pomodoro atomically with the
    new note, and report `project_note.task_links` in JSON and human output.

    '
- id: task-id-completion
  title: Block-ID completion for project task IDs
  depends_on:
  - task-id-grammar
  size: small
  description: 'task-id-completion: add the `project_task_block_id` capture-complete
    context with a `block_id` object (new intent, sibling and `prj` used IDs, body-derived
    suggestions) so editors can offer a New ID picker after ` :` or ` ^`.

    '
- id: docs
  title: Capture docs for named and linked project tasks
  depends_on:
  - task-id-execution
  - task-id-completion
  size: small
  description: 'docs: rewrite the capture guide''s grammar tables, Project notes section,
    JSON contract, and capture-parse/capture-complete references for the new syntax,
    with the worked example.

    '
- id: mac
  title: Bob Mac Capture support for project task links
  depends_on:
  - task-id-execution
  - task-id-completion
  size: medium
  description: 'mac: in bob-mac-capture, color the new spans, route `project_task_block_id`
    into a Project task New ID picker, fix stale `+` teaching and `:` locator text,
    preview and notify linked tasks, update README, fixtures, and tests, and land
    a green macOS CI run.'
proposed_by: bbugyi200.apollo.35
create_time: 2026-09-29 15:35:24
status: wip
bead_id: bob-cli-2n
---

- **PROMPT:** [prompts/202609/project_task_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/project_task_links.md)
- **BEAD:** [bob-cli-2n](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2n/README.md)

# Plan: Named and linked project tasks with ` :id`

## Background

`bob capture` can create a brand-new sub-project note today:

- `@route^id+` creates `<route>_<id>.md` (with `-` in the ID replaced by `_`) and does
  not touch the daily note.
- `@route:id+` and `@route:id+#pomodoro` do the same and also link the new note's `^prj`
  lifecycle task into today's implicit current/next Pomodoro, or into a named one. The
  `^prj` line then starts as `[*]`.

Authored child bullets under a project-note item become `## Tasks` entries
(`- [ ] #task <body> [created::YYYY-MM-DD]`), or `## Title Case` sections when a
first-level ALL-CAPS bullet has nested bullets. None of them can be named or linked, and
a typed ` ^foo` ends up in the middle of the rendered line (`... ^foo [created::...]`),
where it is not a valid Obsidian block ID.

Bryan never wants the `^prj` task linked. He wants to name the real work items and put
the right ones in the Pomodoro.

Relevant code (bob-cli):

- **Marker grammar**, in `src/native/capture_language/`:
  - `tokens.rs`: `parse_task_block_id_route_token` (the `^` family),
    `parse_colon_link_tail` and `parse_pomodoro_route_token` (the `:` family),
    `is_block_id`, `is_pomodoro_selector_component`.
  - `markers.rs`: error constants.
  - `model.rs`: `CaptureKind::ProjectNote` and `ProjectNotePomodoro`.
  - `line.rs`: `resolve_line`.
  - `item.rs`: `parse_capture_item`, which resolves the parent line and child lines and
    then the item kind.
  - `draft.rs`: `classify_authored_line`.
- **Editor grammar**, same directory:
  - `editor_classify.rs`: `classify_task_block_id_token` and `classify_pomodoro_token`,
    which carry the `+` project-note handling, plus `marker_parse` / `MarkerShape` with
    its optional third component.
  - `editor_parse.rs`: `parse_editor_line` and the child-line loop.
  - `editor_model.rs`: `EditorMode`, `SpanKind`, `Need`, `Diagnostic`.
  - `completion.rs`: `completion_field_at` and `completion_field_from_parts`.
  - `rewrite.rs`: `@@` absorption notices for project notes.
- **CLI JSON surfaces**, in `src/native/`:
  - `capture_parse.rs`: JSON (`CaptureParseResult`, `CaptureParseItem`, `sub_bullets`,
    `sub_bullet_depths`) and help text.
  - `capture_complete.rs` and `capture_block_ids.rs`: `BlockIdField`, `BlockIdIntent`,
    `build_block_id_field`, `marker_token_range`, `suggest_ids`.
- **Execution**:
  - `src/native/capture/project_note.rs`: `plan_project_note_item` and
    `plan_project_note_pomodoro_link`.
  - `src/native/capture_project_note.rs`: the pure renderer (`render_project_note`,
    `render_authored_task`, `split_leading_checkbox`, `ProjectNoteRenderInput`,
    `RenderedProjectNote`).
  - `src/native/capture/pomodoro_insert.rs`: `insert_pomodoro_block_link`, with
    `PomodoroSelection::{CurrentOrFuture, NamedOrCreate}`.
  - `src/native/capture/plan.rs`: `capture_task_status`.
  - `src/native/capture/output.rs`: `ProjectNoteSummary`, `CaptureItemResult`, and the
    human printer.
  - `src/native/capture/cli.rs`: the long help and examples.
- **Tests**:
  - Unit tests:
    `src/native/capture_language/tests/{grammar,editor_modes,editor_spans,completion,rewrite}.rs`
    and the `capture_project_note.rs` test module.
  - CLI tests:
    `tests/cli/capture/{project_note,parse,complete_block_id,pomodoro_start,pomodoro_close}.rs`.

The original design is `plan:202609/capture_project_notes.md`. It fixed a rule this plan
keeps: the project-note `+` sits **immediately after the block ID**. A `+` that follows
a `#name` is always part of the Pomodoro name; for example, `@sase:deep-fix#bugs+` names
the Pomodoro `BUGS+`.

## Design

### Vocabulary

In markers, `^` means "a task with this block ID" and `:` means "a Next task with this
block ID, linked into the Pomodoro". Project task IDs use the two sigils with the same
meanings, so there is nothing new to learn:

| You write                           | Where              | Meaning                                                                                                                                    |
| ----------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `@route^id+`                        | item marker        | Create the project note `<route>_<id>.md`. Its `^prj` task is never linked                                                                 |
| `@route^id+#pomodoro`               | item marker        | Same. ` :<id>` Task Links go under the named open Pomodoro, which is created as a named future Pomodoro when missing                       |
| `- <task> ^<task-id>`               | first-level bullet | Name the task: `- [ ] #task <task> [created::DATE] ^<task-id>`                                                                             |
| `- <task> :<task-id>`               | first-level bullet | Name it, make it Next (`[*]`), and add the Task Link `[[<route>_<id>#^<task-id>]]` to the current/next Pomodoro, or to the `#pomodoro` one |
| `@route:id+`, `@route:id+#pomodoro` | retired            | Error that teaches `@route^id+` / `@route^id+#pomodoro` plus ` :<id>`                                                                      |
| `@route^id#pomodoro+`               | misordered         | Error that teaches `@route^id+#pomodoro`                                                                                                   |

A task ID token is recognized **only inside a project-note item**. Everywhere else,
` :x` and ` ^x` stay literal text, exactly as today.

### Worked example

```text
Finish the Google exit packet! @cash^goog-exit+#admin
- Draft the resignation memo :draft-memo
  - keep it short
- Call Morgan Stanley about the 401k :call-ms
- Collect the equity paperwork ^equity-docs
- FUTURE WORK
  - Revisit the severance terms
```

This creates `cash_goog_exit.md` (with `BOB_NOW=2026-09-29T14:31:07`):

```markdown
---
parent: "[[cash]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-09-29T14:31:07-0400
---

- [ ] #task #prj Finish the Google exit packet! #hide ^prj

## Tasks

- [*] #task Draft the resignation memo [created::2026-09-29] ^draft-memo
  - keep it short
- [*] #task Call Morgan Stanley about the 401k [created::2026-09-29] ^call-ms
- [ ] #task Collect the equity paperwork [created::2026-09-29] ^equity-docs

## Future Work

- Revisit the severance terms
```

In the same atomic write, it appends two links to the daily note's `ADMIN` Pomodoro, in
source order. If no open `ADMIN` Pomodoro exists, it first creates the future
placeholder `- [ ] () — ADMIN`:

```markdown
- [ ] () — ADMIN
  - [[cash_goog_exit#^draft-memo]]
  - [[cash_goog_exit#^call-ms]]
```

Without `#admin`, the links go to today's implicit current/next Pomodoro, which is
`PomodoroSelection::CurrentOrFuture`, the same rule that `@route:id` uses.

### Rules

1. **Recognition.**
   - A task ID token is one whitespace-free token made of a sigil (`:` or `^`) followed
     by at least one character, where the first character after the sigil is an ASCII
     letter or digit. So `:)`, `:-)`, and `:(` stay prose.
   - A **lone** `:` or `^` is an unfinished ID; see rule 5.
   - The token counts only when it is the **last word of a line's text after that line's
     item-wide markers are set aside**. Those markers are trailing `s:<N>`, `p:<N>`,
     `%…`, and a trailing `@route…` marker.
   - So `- Draft memo :draft-memo s:2` and `- Draft memo :draft-memo @cash^goog-exit+`
     both work. `- Draft memo s:2 :draft-memo` leaves `s:2` as literal text, just as a
     mid-body marker stays prose today.
   - Recognition runs only after the item's kind is known to be `ProjectNote`. The
     project-note marker may sit on any line, so this is a post-pass over the parsed
     lines.
2. **Validity.**
   - The ID after the sigil must satisfy `is_block_id` (`[A-Za-z0-9-]+`). Otherwise the
     error is `invalid_project_task_id`, for example "task ID `draft_memo` may use only
     A-Z, a-z, 0-9 or '-'".
   - `prj` in any letter case is reserved and gets `invalid_project_task_id`.
   - Two IDs in the same item that compare equal (exact, case-sensitive, matching
     `reject_duplicate_block_id`) get `duplicate_project_task_id`. The message names
     both draft line numbers.
3. **Placement.** Only first-level child bullets are tasks that can be named. A
   well-formed or invalid task ID token anywhere else in a project-note item gets
   `misplaced_project_task_id`:
   - **The parent line.** Message: the project's own task is always `^prj` and is never
     linked; put ` :<id>` on a task bullet.
   - **A nested bullet**, whether it sits under a task or under an ALL-CAPS section.
   - A lone sigil on those lines stays literal.
4. **Task-ness.**
   - A first-level bullet with a task ID is always rendered as a task, even when it is
     ALL-CAPS and has nested bullets. Naming it states the intent.
   - The ID token is removed from the rendered body.
   - An empty remaining body gets `invalid_project_task_id` ("line N names a task but
     has no task text").
5. **Unfinished IDs.** A lone `:` or `^` ending a first-level bullet of a project-note
   item is an unfinished ID:
   - `bob capture` fails with `invalid_project_task_id` ("` :` needs a task ID: end the
     bullet with ` :<block-id>`").
   - `bob capture-parse` reports the item as `mode: "incomplete"` with `needs`
     containing `block_id` and an `interactive_placeholder` span over the sigil, and no
     diagnostic. This mirrors `@cash^`.
6. **Status.**
   - A `:` task renders as `[*]`, or as `[?]` when the project resolved a scheduled date
     through `s:<N>` or a rolled `p:<N>`. This is the same
     `capture_task_status("*", scheduled)` rule as `@route:id`, and it still gets its
     Task Link.
   - A `:` task with an authored checkbox (`- [x] Foo :foo`) gets
     `invalid_project_task_id`: "` :foo` makes the task Next and links it, so it takes
     no `[x]` checkbox; remove the checkbox or write ` ^foo` to keep it". Use the same
     leading-checkbox shape that `split_leading_checkbox` recognizes.
   - A `^` task keeps an authored checkbox status, or `[ ]` without one.
   - The `^prj` line is `[ ]`, or `[?]` when scheduled, and is never `[*]` again.
7. **The Pomodoro name.**
   - `#pomodoro` on the marker selects where the ` :` links go.
   - A project-note item with a `#pomodoro` name and **no** ` :` task gets
     `unused_project_note_pomodoro` (range: the `#name` component): "`#admin` picks the
     Pomodoro for ` :<id>` task links, but no task bullet ends with ` :<id>`; add one or
     remove `#admin`".
   - ` ^` tasks alone do not satisfy this rule.
8. **Daily note.**
   - A project note with no ` :` task never reads the daily note, as today.
   - With at least one, it needs the daily note and a `## Pomodoros` section.
     Missing-file, no-eligible-Pomodoro, multiple-open-timed, and "ledger already
     contains `[[…]]`" errors reuse the existing messages.
   - The note and the daily note are staged together, so any failure writes nothing, and
     later batch items see the staged daily note.
9. **Unchanged restrictions.** Project notes still reject `%…`/`--clip`, forced
   destination flags, `!`, `@@` declarations, and `=<X>` / `=x` suffixes. Reword the
   suffix messages to name `@<route>^<block-id>+`.

Evaluation order in `bob capture`, so the first error is deterministic:

1. The marker token.
2. The parent line.
3. The children, in source order. For each child: shape and charset, then reserved, then
   placement, then empty body, then checkbox, then duplicate.
4. The unused `#pomodoro`.

`bob capture-parse` reports every one of these as a diagnostic, never as a failure.

### The `#pomodoro` placement decision (flag for review)

Bryan's request spelled the new form `@file^id#pomodoro+`. This plan makes
**`@file^id+#pomodoro`** canonical and turns `@file^id#pomodoro+` into a focused error
that prints the corrected token. The reasons:

- It keeps the one rule the grammar documents for `+` after a name. In every family, a
  `+` after `#name` is part of the Pomodoro name. If the `^` form read `#bugs+` as "name
  `bugs`, then the sigil", then `@sase:deep-fix#bugs+` (a task under `BUGS+`) and
  `@sase^deep-fix#bugs+` (a project with its links under `BUGS`) would differ by one
  character and mean opposite things.
- It is prefix-closed, which suits a live editor. Every prefix of
  `@cash^goog-exit+#bugs` is either complete or naturally incomplete: `…^goog-exit`,
  then `…+`, then `…+#` (needs `pomodoro_name`), then `…+#bugs`. By contrast,
  `@cash^goog-exit#bugs` would be a dead state while typing `#bugs+`.
- It is the spelling the existing `+#pomodoro` completion, picker type-through (`+`
  commits the ID), and the "Then type +" teaching line already produce.

If Bryan prefers the trailing-`+` spelling, only the `marker` phase changes. Everything
else in this plan is independent of that choice.

### Shared contracts

These names are fixed across phases. Later phases and bob-mac-capture depend on them.

**Rust model** (`capture_language/model.rs`):

- `CaptureKind::ProjectNote { block_id: String, pomodoro_name: Option<String> }`.
  `ProjectNotePomodoro` is deleted.
- `ProjectTaskId { block_id: String, link: bool }`, where `link` is `true` for `:`.
- `AuthoredSubBullet` gains `task_id: Option<ProjectTaskId>`. It is always `None`
  outside project-note items.
- Put the lexer and the item-level rule checks in a new focused module
  `capture_language/project_tasks.rs`, shared by the execution and editor parsers. The
  existing grammar files are already large.

**Editor model** (`editor_model.rs`):

- New `SpanKind`s:
  - `ProjectTaskLinkMarker` → `"project_task_link_marker"`, over the `:` sigil only.
  - `ProjectTaskBlockId` → `"project_task_block_id"`, over the ID for both sigils.
  - The `^` sigil gets no span, just as separators never do.
- The existing `project_note_marker` still covers the marker's `+`. The `^` marker's
  name uses the existing `pomodoro_name` span.
- Modes:
  - `project_note` is a project note that writes no Task Links.
  - The existing `pomodoro_project_note` string is **reused** at item level: a project
    note with at least one ` :` task, meaning the daily note is involved.
  - Token-level classification of a `^…+` marker always yields `ProjectNote`. The
    item-level pass upgrades it.
- New diagnostic codes: `retired_project_note_marker`, `invalid_project_task_id`,
  `duplicate_project_task_id`, `misplaced_project_task_id`, and
  `unused_project_note_pomodoro`.
- Existing codes that are kept: `invalid_project_note_marker` (misordered or `+`-less
  `#name`, `=` suffixes), `invalid_pomodoro_name`, `invalid_task_block_id`, and
  `invalid_task_block_id_route`.
- No new `Need`. An unfinished task ID needs `block_id`.

**capture-parse JSON:**

- A new optional `sub_bullet_task_ids` array appears at the top level and per item. It
  is parallel to `sub_bullets`, and each entry is `null` or
  `{"block_id": "...", "link": bool}`.
- It is emitted only when at least one entry is non-null.
- `sub_bullets` bodies exclude the ID token.
- `section` carries the marker's Pomodoro name, as the `:` form did.
- `schema_version` stays `1`, because every change is additive. The only removal is the
  retired `:` marker spelling itself.

**capture-complete JSON:**

- New context `"project_task_block_id"`, with empty `candidates` and a `block_id`
  object:
  - `route`: the project-note stem, for example `cash_goog_exit`.
  - `relative_target`: for example `cash_goog_exit.md`.
  - `note_exists`
  - `marker`: `":"` or `"^"`.
  - `marker_range`: the sigil and the ID.
  - `intent`: `"new"`.
  - `body`: the bullet body without the ID or a leading checkbox.
  - `allowed_character` / `allowed_description`: the shared block-ID rule.
  - `suggestions`: `suggest_ids` over the body, excluding used IDs.
  - `used`: `prj`, then every other task ID already typed in the item.
- A `used` entry has these fields:
  - `line`: the 1-based physical draft line.
  - `task`: `true`.
  - `status_symbol` / `status_name`: `null`.
  - `text`: that bullet's body, or the project body for `prj`.
- Older clients route an unknown context to their inline list, which has no candidates,
  so they degrade silently.

**`bob capture --format json`** (project-note results):

- The `project_note` object gains `task_links`, which is **always present** and may be
  empty: `[{"block_id", "block_link", "text", "task_line"}]` in source order.
  `block_link` is `[[<stem>#^<id>]]`, and `task_line` is the rendered `[*]`/`[?]` line.
- When at least one link was written, the result also reports these top-level fields
  with their `pomodoro_task` meanings:
  - `day_file`
  - `pomodoro_link_placement` (from the first insertion)
  - `pomodoro_name` (canonical, when named)
  - `creates_pomodoro`
- Project-note results **omit** the top-level `block_link`, which used to be the `^prj`
  link.
- `block_id` stays `"prj"`, and `task_line` stays the `^prj` line.

### Error message texts

Use these messages. Each one substitutes the actually typed route, IDs, and name. Keep
them as `markers.rs` constants or `format!`s, and use the same text in the diagnostics.

- **Retired colon form.** `@cash:goog-exit+` → "`@cash:goog-exit+` is retired: a project
  note never links its own `^prj` task. Write `@cash^goog-exit+` and end each task
  bullet you want in the Pomodoro with ` :<id>`".
  - With a name, suggest `@cash^goog-exit+#bugs`.
  - With no route, as in `@:goog-exit+`, use `<route>`.
  - An empty block ID keeps the family's existing "requires a block ID" wording.
- **Misordered.** `@cash^goog-exit#bugs+` → "put the project-note `+` right after the
  block ID: `@cash^goog-exit+#bugs` (a `+` after `#bugs` would be part of the Pomodoro
  name)".
- **Name without `+`.** `@cash^goog-exit#bugs` → "`#bugs` after `@cash^goog-exit` needs
  the project-note `+` (`@cash^goog-exit+#bugs`); to link a task under a Pomodoro, use
  `@cash:goog-exit#bugs`".
- **Active-task spelling.** `^cash:goog-exit+` → "`^route:block-id+` is not a capture
  form; create a project note with `@route^block-id+`". This rewords
  `POMODORO_LINK_PROJECT_ERROR`.
- **Project-task and unused-name messages:** as quoted under Rules.

## Phase `marker`: project-note marker grammar

Scope: the marker token only. Child task IDs are out of scope for this phase.

1. **Execution grammar** (`tokens.rs`, `markers.rs`, `model.rs`):
   - Reshape `CaptureKind::ProjectNote` as specified in Shared contracts.
   - In `parse_task_block_id_route_token`, split the part after `^` at the first `#`
     into `block_part` and an optional `name`. `block_part` must end with exactly one
     `+` when a name is present.
   - The name is validated with `is_pomodoro_selector_component`. An empty name gets a
     required-name error naming `@<route>^<block-id>+#<pomodoro>`.
   - A `=` anywhere after the `+` is rejected with the reworded start or close message.
   - Replace `PROJECT_NOTE_POMODORO_NAME_ERROR`, which rejected `+#`, with the
     misordered and name-without-`+` errors.
   - In `parse_colon_link_tail`, a block part ending in `+` now returns the retired
     error for both `@` and `^` sigils; the `^` sigil gets the reworded
     `^route:block-id+` message. Delete the `project_note` field from `ColonLinkParts`.
   - Confirm that `is_task_block_id_marker_candidate`, `is_pomodoro_marker_candidate`,
     and `is_sub_bullet_marker_candidate` still route `@r^id+#name`, `@r^id#name+`,
     `@r:id+`, and `@r:id+#name` to the intended family. Pin each with a unit test.
2. **Item level** (`item.rs`):
   - A `ProjectNote` with `pomodoro_name: Some` is rejected with
     `unused_project_note_pomodoro`. This is the final rule with zero ` :` tasks, and
     `task-id-grammar` makes it count them.
   - Keep the `%`/`--clip`, forced-flag, `!`, and `@@` behavior.
3. **Editor** (`editor_classify.rs`):
   - Give `classify_task_block_id_token` a third component for project notes:
     - `+` gets a `project_note_marker` span;
     - `#name` gets a `pomodoro_name` span;
     - `@cash^goog-exit+#` is `incomplete` with `needs: ["pomodoro_name"]` and an
       `interactive_placeholder` span over `#`;
     - `section` is the name.
   - Emit the misordered, `+`-less, and `=` diagnostics as `invalid_project_note_marker`
     over the whole token.
   - In `classify_pomodoro_token`, any `+` directly after the block part becomes a
     `retired_project_note_marker` diagnostic over the whole token. Remove the
     `PomodoroProjectNote` token-level path.
   - The item-level `unused_project_note_pomodoro` diagnostic is emitted over the
     `#name` component's range.
4. **Completion:**
   - In `completion.rs`, completing inside `#…` after `@route^id+` reports the
     `pomodoro_name` context, served by the existing `pomodoro_name_candidates`. The
     replacement is just the name.
   - The retired `:`+`+` token offers nothing.
   - In `capture_block_ids.rs`, `project_note` intent applies only to the `^` marker (a
     `+` after the replacement). `marker_token_range` keeps covering the whole token.
5. **Execution:**
   - Delete `ProjectNoteRenderInput.pomodoro_link`, the `^prj` Task Link path in
     `plan_project_note_item`, and `plan_project_note_pomodoro_link`. The
     `task-id-execution` phase builds its multi-link replacement, and leaving the old
     planner in place would trip clippy's dead-code lint.
   - The `^prj` line is `[ ]`, or `[?]` when scheduled.
   - The project-note result stops reporting `day_file`, `block_link`,
     `pomodoro_link_placement`, `pomodoro_name`, and `creates_pomodoro`.
6. **Rewrite:** in `rewrite.rs`, project-note markers stay non-absorbable with the same
   notice. Only update tests that used `@cash:goog-exit+`.
7. **Help text:**
   - In `capture/cli.rs`, update the project-note paragraph: the `^` form, the optional
     `#<pomodoro>`, and "`@<route>:<block-id>+` is retired". Replace the example
     `bob capture '@cash:goog-exit+#bugs' …` with a `@cash^goog-exit+` example; the
     ` :id` example comes in `task-id-execution`.
   - Update the `capture_parse.rs` and `capture_complete.rs` help strings that describe
     `@:id+` / `pomodoro_project_note` and "either block-ID part". The `:` form no
     longer carries the project-note intent.
8. **Tests:**
   - Rewrite every `@…:…+` project-note case into either retirement assertions or `^`
     equivalents:
     - `tests/grammar.rs` `project_note_markers_stay_in_their_families`,
       `execution_parses_project_note_markers`, and
       `execution_rejects_project_note_shape_errors`;
     - the `editor_modes.rs` and `editor_spans.rs` rows;
     - `tests/completion.rs` `block_id_project_note_sigil_is_excluded_from_replacement`;
     - `tests/cli/capture/project_note.rs` `…pomodoro_variants_link_the_prj_task`, which
       should now assert the retired error and no writes;
     - `parse.rs` `capture_parse_json_reports_project_note_markers`;
     - `complete_block_id.rs`, `pomodoro_start.rs` (`@sase:b1+=3`), and
       `pomodoro_close.rs` (`Do @bob:new+=x`);
     - `capture_parse.rs` `json_reports_invalid_start_suffixes_as_diagnostics`.
   - Add these cases:
     - `@cash^goog-exit+#bugs` parses to `ProjectNote { pomodoro_name: Some("bugs") }`,
       and `bob capture` rejects it with `unused_project_note_pomodoro`;
     - `@cash^goog-exit#bugs+` and `@cash^goog-exit#bugs` get their errors with the
       corrected token;
     - `@cash^goog-exit+#` is incomplete in the editor;
     - `@cash^goog-exit+#c++` names `C++`;
     - `@cash^goog-exit+=3` and `@cash^goog-exit+#bugs=x` are rejected;
     - completion at `@cash^goog-exit+#bu` gives the `pomodoro_name` context;
     - `@@cash^goog-exit+#bugs` stays `invalid_global_destination`;
     - a `^` project note never reads a missing daily note.
   - Keep `editor_agrees_with_execution_for_resolved_captures` and
     `interactive_markers_are_the_only_divergence_from_execution` green with the new
     inputs.
9. `just all` (cargo fmt --check, clippy with all targets and features, and the tests)
   passes.

## Phase `task-id-grammar`: project task IDs in the grammar and capture-parse

1. **Lexer** (`project_tasks.rs`):
   - `lex_project_task_id(token) -> Option<TaskIdLex>` distinguishes these cases, per
     Rules 1, 2, and 5:
     - `Lone { sigil }`;
     - `Valid { sigil, id }`;
     - `Invalid { sigil, raw }`: starts with an alphanumeric but fails `is_block_id`;
     - `None` for prose.
   - Unit-test the boundary set: `:D`, `:)`, `:-)`, `:draft-memo`, `^Equity-2`, `:a_b`,
     `:`, `^`, `::x`, `:x:`, and `10:30`.
2. **Execution:**
   - In `line.rs`, `LineOutcome` carries the last remaining body token and its span as a
     candidate. It stays in `body` until the item kind is known.
   - In `item.rs`, after `resolve_sub_bullet_kind`, and only for `ProjectNote`, apply
     Rules 1–7 in the evaluation order above:
     - strip accepted IDs from bodies;
     - set `AuthoredSubBullet.task_id`;
     - turn the phase-1 unconditional `#pomodoro` rejection into "at least one ` :`
       task".
   - Checkbox detection must match the renderer's `split_leading_checkbox`. Share one
     helper rather than duplicating it: move it to a place both can import, or re-export
     it.
3. **Editor:**
   - Make `editor_parse.rs` do the same post-pass with offset-tagged tokens:
     - emit `project_task_link_marker` / `project_task_block_id` spans for accepted IDs;
     - emit diagnostics with the ID token's range for violations;
     - for unfinished IDs, set `mode: "incomplete"`, add `needs: ["block_id"]`, and put
       an `interactive_placeholder` span over the sigil;
     - upgrade the mode to `pomodoro_project_note` when at least one ` :` task exists
       and nothing else made the item incomplete.
   - Editor `AuthoredSubBullet`s carry `task_id` so the agreement tests compare it.
   - Non-project items stay byte-identical: no spans, and the ID token stays in the
     body.
4. **capture-parse** (`capture_parse.rs`):
   - Emit `sub_bullet_task_ids` at the top level and per item.
   - Add `project_task_link_marker` and `project_task_block_id` to the documented span
     list, and describe the reused `pomodoro_project_note` mode, the new diagnostic
     codes, and the unfinished-ID need in the help text.
   - Human output: after each sub-bullet, show its ID as ` ^id` or ` :id`.
5. **Tests:**
   - `editor_modes.rs` / `editor_spans.rs` rows cover the worked example, each Rule
     violation, and the lone-sigil incomplete state.
   - Grammar tests for execution cover each error message verbatim, including the
     duplicate-line wording.
   - Agreement tests include the worked example and a child-line marker case
     (`- Draft :d @cash^x+`).
   - Add a regression test that a non-project item (`Fix @sase` / `- ratio 3 :1`) keeps
     `:1` literal on both paths.
   - Add a `tests/cli/capture/parse.rs` JSON case for `sub_bullet_task_ids`, spans, and
     `pomodoro_project_note`.
6. Execution may ignore `task_id` until the next phase (links are not written yet). Keep
   `just all` green.

## Phase `task-id-execution`: render named tasks and write their Task Links

1. **Renderer** (`capture_project_note.rs`):
   - A first-level bullet with `task_id` always takes the task branch.
   - Extend `render_authored_task` to append ` ^<id>` as the **last** token, after
     `[created::…]`.
   - Status per Rule 6: pass the resolved `scheduled` into the renderer input.
   - `RenderedProjectNote` gains
     `named_tasks: Vec<RenderedNamedTask { block_id, link, text, task_line }>` in source
     order.
   - `task_count` semantics are unchanged.
   - Unit tests cover:
     - the worked example byte for byte;
     - an ALL-CAPS named bullet with nested children, which renders as a task;
     - the scheduled `[?]` status;
     - an authored `[/]` status on a `^` task;
     - an existing `[created::…]` in the body.
2. **Planner** (`capture/project_note.rs`):
   - When at least one named task has `link`:
     1. Validate the daily note: it must exist, and it must not be the new note.
     2. Read the staged day contents once.
     3. For each link in source order, reject a ledger that already contains
        `[[<stem>#^<id>]]`, then apply
        `insert_pomodoro_block_link(&day, &link, pomodoro_name)` to the evolving
        contents.
   - Compute `creates_pomodoro` and the canonical `pomodoro_name` from the **original**
     day contents. Exactly one named Pomodoro may be created, and every link lands under
     it in source order.
   - Stage the note and the day file together.
   - With no links, the daily note is never read.
3. **Output** (`capture/output.rs`, `capture/project_note.rs`):
   - JSON per Shared contracts: `ProjectTaskLinkJson`, `project_note.task_links` always
     serialized, top-level link fields only when links exist, and no top-level
     `block_link`.
   - Human output, after the tasks/sections line:
     - `✓ linked   <day_file>` (dry-run: `would link`), followed by `  under <NAME>`
       with ` (created)` when a named Pomodoro was created;
     - then one dim `  - [[stem#^id]]` line per link;
     - then the existing hint.
4. **Help:** in `capture/cli.rs`, add the ` :<id>` / ` ^<id>` paragraph and a `printf`
   example of the worked example to `bob capture --help`.
5. **CLI tests** (`tests/cli/capture/project_note.rs`):
   - the worked example end to end: note bytes, daily-note bytes, and JSON;
   - the implicit current/next Pomodoro;
   - an existing named Pomodoro versus creating one;
   - dry run writes nothing but reports the same JSON;
   - a batch where the second item's links append after the first item's;
   - a rollback when the ledger already contains a link;
   - an error with no daily note only when ` :` is used;
   - `s:2` gives `[?]` tasks that are still linked;
   - `^`-only tasks leave the daily note untouched.
6. `just all` passes.

## Phase `task-id-completion`: block-ID completion for project task IDs

1. **Context detection:**
   - In `completion.rs` and `capture_complete.rs`, on a first-level child line of an
     item whose editor parse is a project note with a resolved route and block ID, a
     cursor inside a trailing task ID token (from just after the sigil to the token end)
     yields `CompletionContext::ProjectTaskBlockId`.
   - The replacement is the ID span, empty at the cursor for a lone sigil.
   - Reuse the `project_tasks.rs` lexer and the same "last body word after markers"
     rule. Items with an unresolved route or project ID, non-project items, and nested
     or parent lines yield no context.
2. **`block_id` object** (`capture_block_ids.rs`): add a builder variant that fills the
   object per Shared contracts. It reads no routed note except to set `note_exists` for
   the project note, and it gets `used` from the item's own parse.
3. **Help:** document the context in the `capture_complete.rs` help text.
4. **Tests:**
   - unit tests in `tests/completion.rs`: a lone `:`, a partial `:dr`, `^`, a child-line
     route marker after the ID, a nested line (none), a non-project item (none);
   - a CLI test in `tests/cli/capture/complete_block_id.rs` asserting the full JSON for
     `Finish it @cash^goog-exit+\n- Draft the memo :` (suggestion `draft-the-memo` or
     whatever `suggest_ids` yields; `used` contains `prj`) and for a second bullet whose
     used list includes the first bullet's ID.
5. `just all` passes.

## Phase `docs`: capture docs

Update `docs/capture.md` so it describes only the new grammar:

1. **Grammar at a glance:**
   - Replace the `@route:block-id+` and `@route:block-id+#pomodoro` rows with a
     `@route^block-id+#pomodoro` row.
   - Add rows for `- <task> :<task-id>` and `- <task> ^<task-id>` on project-note
     bullets.
   - In the `#` table, update the `@cash:goog-exit+` rows to the `^` equivalents and add
     a retired-form row. Keep the `@sase:deep-fix#bugs+` row, since it explains why the
     `+` comes first.
2. **Project notes:**
   - Lead with the worked example: input, note, and ledger.
   - Then the rules: recognition, placement, status, `#pomodoro`, daily-note behavior,
     and atomicity.
   - Then the errors, with one example each.
   - Then the retired forms.
   - Remove every statement that `^prj` is linked or `[*]`.
3. **JSON contract:** `project_note.task_links`, the top-level link fields, and the
   removed `block_link`.
4. **Authored sub-bullets:** drop `@route:block-id+` from the item-wide marker list.
5. **capture-parse:** the span kinds, the reused mode, `sub_bullet_task_ids`, the
   diagnostic codes, and the unfinished-ID incomplete state.
6. **capture-complete:** the `project_task_block_id` context and its `block_id` object.
7. **Interactive editor markers:** the ` :` / ` ^` New ID flow.
8. `docs/projects.md`: one sentence in the capture pointer paragraph noting that capture
   can name and link project tasks.
9. **Verification:**
   - Re-read the edited sections against the shipped behavior by running the documented
     examples in a temp vault (`BOB_DIR`, `BOB_DAY_FILE`, and `BOB_NOW`).
   - Grep `docs/` and `src/` for `:block-id>+`, `:goog-exit+`, and `^prj` linking
     claims.

## Phase `mac`: Bob Mac Capture support

**Repository.** Open bob-mac-capture with `/sase_repo`:
`sase repo open bob-mac-capture`. On Linux hosts the linked checkout is missing, so fall
back to `sase repo open gh:bobs-org/bob-mac-capture`. Use only the printed path, and
read its `AGENTS.md` if one exists. The app never parses grammar; it follows Bob's
spans, needs, contexts, and JSON.

1. **Spans:**
   - In `captureSemanticCategory` (`CompletionRowContent.swift`), map
     `project_task_link_marker` → `.explicitToggle` (the operator color shared with
     `project_note_marker`) and `project_task_block_id` → `.blockID`.
   - Add both kinds to `completionSpanKinds` (`CapturePanelModel.swift`).
2. **Completion routing:**
   - Add `CaptureCompletionContext.projectTaskBlockID` (`"project_task_block_id"`).
   - Route it to the Block ID Picker alongside `pomodoro_block_id` / `task_block_id` in
     the picker gate and the "usable response" check (`CapturePanelModel.swift` around
     1013–1072).
   - Carry a picker scope such as `CaptureBlockIDScope.projectTask` on the block-ID
     context.
   - Intent is `new`: availability comes from `used`, suggestions from Bob, and
     type-through uses `BlockIDRules`.
3. **Project task scope presentation:**
   - Scope token: ` :` or ` ^`, with no `@route`.
   - Scope caption: "Linked task" for `:` and "Task ID" for `^`.
   - Outcome for `:`: "Next task in ▣ cash_goog_exit.md, linked into today's Pomodoro".
   - Outcome for `^`: "Task in ▣ cash_goog_exit.md".
   - Locator and "↩ inserts :draft-memo", announcement "Inserted :draft-memo", and no
     teaching line.
   - Fix the existing bug where the locator hard-codes `:` for every row
     (`CapturePickerView.swift` around 667 and 817). Use the context marker.
4. **Teaching lines** (`CapturePickerView.swift` around 787–803):
   - The `:` line drops the project-note `+`: "Then type #name or = to start".
   - The `^` line teaches the new marker: "Then type + for a project note (+#name picks
     its Pomodoro)".
   - Update `BlockIDPickerIndex` outcome wording only where it assumed the `:` project
     note.
5. **Preview:**
   - Decode `project_note` (including `task_links`) on `CaptureCommandSuccess`,
     tolerating absent fields.
   - Add a `CaptureProjectNotePresentation` in CaptureCore that exposes the linked-task
     rows and the destination text ("today's Pomodoro", or `ADMIN` / "new ADMIN" from
     `pomodoro_name` / `creates_pomodoro`).
   - In `standardPreviewItem`, render a link section: a link symbol, "Links 2 tasks into
     ADMIN (new)", then one row per task showing its text and `^id`.
   - Treat the day file as changed when `task_links` is non-empty (`dayFileChanged`,
     `captureWroteDayFile`) so Open Note(s) offers the daily note.
6. **Notifications:** a project note with links adds a body line "Linked 2 tasks into
   ADMIN". Batch lines stay as they are.
7. **README:**
   - Update the picker paragraphs (around 249–251, 321–343, 357–365) and the marker list
     (around 607–621) for `@route^id+#pomodoro`, ` :id` / ` ^id`, and the retired `:`
     form.
   - Bump the list of required Bob features.
8. **Fixtures and tests:**
   - Build bob from this bob-cli checkout (`cargo build`). Regenerate the real-bob
     block-ID fixtures with the recipe in the `BlockIDDecodingTests.swift` header, and
     add:
     - a `project_task_block_id` completion fixture;
     - a project-note `capture --dry-run --format json` fixture with two `task_links`.
   - Add `fake-bob` cases as needed.
   - Tests:
     - decoding the new context, scope, and `project_note.task_links`;
     - presentation strings for both sigils;
     - span categories;
     - teaching lines;
     - locator marker;
     - preview rows;
     - day-file gating;
     - a picker design test for the project-task scope.
   - Update tests that asserted the old teaching text or the `:` locator.
9. **Verify on macOS.** Swift is not available on Linux:
   - Try `ssh -o ConnectTimeout=8 mac true` first. If it works, rsync the checkout
     (excluding `.build`) to `mac:/tmp/bob-mac-capture-task-links/` and run
     `just format-lint build test` there.
   - Otherwise iterate on GitHub Actions. This plan explicitly instructs you to use
     `/sase_git_commit` for each CI iteration in bob-mac-capture, with conventional
     `feat(capture): …` or `fix(capture): …` subjects.
   - Find the run with `gh run list -L 3`. Wait in the foreground with
     `gh run watch <id> --exit-status` and a tool timeout of about 30 minutes, or hand
     the wait to `/sase_monitor`.
   - Read failures with
     `gh run view <id> --log-failed | grep -E " error: |error: -\[|failed \("`.
   - Done means one `macOS 26 SwiftPM` run with every step green. Record the run ID in
     the final response.
   - Never weaken or skip an assertion to go green.

## Out of scope and follow-ups

- **Previewing the whole planned note in Bob Mac Capture.** Every task line and section
  would need a new `project_note` field. Leave it for a follow-up if Bryan wants it.
- **Starting the linked Pomodoro in the same capture.** An example would be
  `@cash^goog-exit+#admin=`. The `=` suffix stays rejected on project notes.
- **Task IDs on non-project items.** This covers sub-bullets under ordinary tasks, which
  are notes rather than tasks. Their ` :x` stays literal.
