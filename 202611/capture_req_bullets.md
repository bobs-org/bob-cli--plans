---
tier: epic
title: Requirement bullets in capture (grammar 29)
goal: 'Capture admits requirement bullets at every eligible position and depth, refuses
  the ineligible ones with taught codes, shows them as req_marker spans, expands a
  typed `- [` + space into `- [ ] `, and writes them into the reqs zone with the NOTES
  rule and the first-requirement scoop, completing phase bob-cli-on.6 (capture-reqs).

  '
phases:
- id: depth-model
  title: Numeric authored depth with today's behavior kept
  depends_on: []
  size: medium
  description: 'depth-model: turn AuthoredDepth into a numeric level, teach the authored-line
    classifier any even-space level with a capped form, move every consumer and the
    grouping helpers onto it, and make the sub-bullet renderers depth-preserving,
    all without changing any capture, parse, or rewrite output.'
- id: req-positions
  title: Shared requirement position resolver, refusals, req_marker span, grammar
    29
  depends_on:
  - depth-model
  size: medium
  description: 'req-positions: add the pure req-position resolver shared by bob capture
    and capture-parse, admit nested req chains past level 2, refuse req_in_drawer,
    req_parent, req_state, and req_depth, emit req_marker spans with depth, and bump
    the grammar to revision 29.'
- id: new-block-writes
  title: Requirements in the blocks capture creates
  depends_on:
  - req-positions
  size: medium
  description: 'new-block-writes: render new tasks, list tasks, Pomodoro-created tasks,
    and project-note rows so reqs sit in the reqs zone and notes follow the NOTES
    rule through task_block_layout, and report an additive requirements summary in
    JSON and human output.'
- id: existing-target-writes
  title: Requirements added to existing tasks, the scoop, and req_link
  depends_on:
  - new-block-writes
  size: medium
  description: 'existing-target-writes: write note-less @route+id requirement bullets
    and the same-line `[ ] text @route+id` form through insert_reqs with the first-requirement
    scoop, refuse forced-drawer reqs and Task Links to req anchors (req_link), and
    preview the scoop.'
- id: req-rewrite
  title: The expand_req_marker typing rewrite
  depends_on:
  - req-positions
  size: medium
  description: 'req-rewrite: add expand_req_marker to capture-rewrite with resolver-driven
    eligibility and the notebook policy preflight, and document client-side overtype
    absorption and atomic Backspace in capture-rewrite help and docs/capture.md.'
proposed_by: bbugyi200.athena.bob-cli-on.6
proposed_with: claude/opus
parent_bead: bob-cli-on.6
create_time: 2026-11-21 13:50:27
status: wip
bead_id: bob-cli-on.6.1
---

- **PROMPT:** [prompts/202611/capture_req_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_req_bullets.md)
- **PARENT:** [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)
- **BEAD:** [bob-cli-on.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.6.1.md)

# Requirement bullets in capture (grammar 29)

## Scope and authority

This child epic implements phase **`bob-cli-on.6`** (`capture-reqs`) of epic
`bob-cli-on` (Task requirements). Its land agent verifies the complete phase and closes
`bob-cli-on.6`. Child phase workers close only their own assigned beads. Nobody here
closes `bob-cli-on` or any other ancestor.

Read before implementing any phase:

- `sase bead read bob-cli-on.6 -r "<why>"` and its epic plan
  `plan:202611/task_requirements.md` (read it with `sase artifact read`). Its **§1, §3,
  §5** and the **capture-reqs** phase section are authoritative. The epic's DECISIONS
  are final: implement only `notes_drawer_birth = with_first_note` (`NOTES` exists only
  when a note lands in it).
- `docs/task-requirements.md`, the shipped contract. §3 (layout, NOTES rule, writers)
  and §5 (authoring) govern this work. Each phase updates the §5 text it ships.
- The research refs `research:202611/task_reqs_nested_and_notes_impact.md` (wins on
  conflicts) and `research:202611/task_reqs_design/task_reqs_design.md` (Authoring and
  Capture alias sections), read with `sase artifact read`.
- Memory: `decisions:capture-bullets-mirror-the-task-block` ("one resolver serves the
  menu, `capture-parse`, the dry-run preview, and the commit") and
  `decisions:mac-capture-is-a-thin-client`. Reference memory `docs.md` and
  `cli_rules.md` before touching docs, help, or CLI output.

**No memory edits here.** This work makes
`decisions:capture-bullets-mirror-the-task-block` partly stale (third authored level,
checkbox bullets are reqs). The epic's later `memory` phase owns that supersession;
phase workers do not touch memory files.

### What already exists (do not rebuild)

- `src/native/task_reqs.rs`: `req_forest`, `is_req_line`, `owner_of_line`,
  `checkbox_mark_without_filter`, `ReqMark`, `MAX_REQ_DEPTH = 16`.
- `src/native/task_block_layout.rs` and `task_block_layout/{writers,tidy}.rs`:
  `insertion_offset`, `note_home` (`Loose` / `Notes` / `NewNotes`),
  `ensure_notes_drawer`, `plan_tidy`, `insert_reqs` (end of the reqs zone, first-req
  scoop, `req_state` / `req_depth` / `req_empty` / `not_a_task` refusals), and the RL/RW
  vectors.
- `capture/sub_bullet.rs::plan_sub_bullet_capture` and the dry-run placement in
  `capture/plan.rs` already route a note through `note_home`, so a note to a req-owning
  task files into `NOTES` today.

## Shared design: the requirement position resolver

One pure module, `src/native/capture_language/req_bullets.rs` (added in
`req-positions`), decides every requirement question for both resolvers:
`capture_language` item parsing (`bob capture`) and `editor_parse.rs` (`capture-parse`).
Both parsers build their authored bullet list, derive a host, and call the same
function. They never re-implement the rules. The typing rewrite and the writers read its
answers.

### Hosts

The host is derived from the item's resolved shape, never from the vault.

| Host           | Item shape                                                                                                                                                                                           | Editor mode                                         |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `NewTask`      | The item creates a task: `Task`, `TaskWithBlockId`, a task-creating `Pomodoro { close: None }`, and each `@route` **task list** bullet. An item with no route is an inbox new task.                  | `Task`, `PomodoroTask`                              |
| `ExistingTask` | A note-less `@route+id` with first-level bullets (the multi-note form).                                                                                                                              | `SubBullet` with an empty body and no glued section |
| `SameLineReq`  | `@route+id` whose item text starts with a checkbox (`[ ] Handles CRLF @bob+export`), including forced `--task` / `--task-ref` targets.                                                               | `SubBullet` whose body starts with a checkbox       |
| `SingleNote`   | `@route+id <note text>`: bullets nest under that note.                                                                                                                                               | `SubBullet` with body text                          |
| `Drawer`       | `@route+id#drawer` (glued) or a forced `--drawer` destination: every bullet is inside a drawer.                                                                                                      | `SubBullet` with a glued section                    |
| `ProjectNote`  | A sealed project-note item (`@route^id@`, `@route:id@`): first-level bullets are `## Tasks` rows, sections, or moves.                                                                                | `ProjectNote`, `PomodoroProjectNote`                |
| `Ineligible`   | Everything else: notebook routes and lists, `@route` notes lists, `@route#section` and forced `--section` bullets, records, close logs, Pomodoro start/close/adjust, transfers, merge-in, templates. | Every other mode                                    |

### Levels, roles, and depth

A bullet's **level** is its authored level after any list shift. A `@route` task-list
bullet is the item line, so its nested lines are one level higher, as today.

- **Levels 1 and 2** are admitted exactly as today. A level-2 bullet needs a preceding
  level-1 bullet (`orphaned_nested_bullet`).
- **Level L ≥ 3** is admitted only beneath a req chain:
  - its parent (the nearest preceding bullet at level L−1) is a req; or
  - its parent is a note whose own parent is a req. A note of a req keeps today's one
    nested note level.
- Anything else at level ≥ 3, or a jump of more than one level, stays
  `invalid_child_line`. In task hosts the copy gains a hint that deeper bullets nest
  only beneath a requirement (`- [ ] …`). `Ineligible` hosts keep today's exact copy and
  behavior byte for byte. A task-list bullet counts levels after the shift, so its notes
  gain no extra nesting.

**Roles.** A checkbox bullet is a bullet whose body starts with `[c]` followed by
whitespace or end of line (`split_leading_checkbox`).

| Host and position                                | Checkbox bullet                                                                                         | Plain bullet                                                        |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `NewTask` / `ExistingTask`, level 1              | **req, depth 1**                                                                                        | note, or drawer header (ALL-CAPS with children, `- #slug`)          |
| Under a drawer header                            | `req_in_drawer`                                                                                         | drawer entry (today)                                                |
| Under a req at depth d                           | **req, depth d+1**                                                                                      | note of that req                                                    |
| Under a note                                     | `req_parent`                                                                                            | nested note (today)                                                 |
| `SameLineReq`: the item line                     | **req, depth 1** (`req_state` if pre-checked)                                                           | —                                                                   |
| `SameLineReq`: level-1 bullets                   | req, depth 2                                                                                            | note of the item's req (a `- #slug` stays `drawer_bullet_position`) |
| `SingleNote`: level-1 bullets                    | `req_parent`                                                                                            | nested note (today)                                                 |
| `Drawer`: any bullet, and a same-line `[ ] text` | `req_in_drawer`                                                                                         | drawer entry (today)                                                |
| `ProjectNote`, level 1                           | `## Tasks` row (today, status kept)                                                                     | row, section, or move (today)                                       |
| `ProjectNote`, level 2 under a task row          | **req, depth 1** of that row                                                                            | note of that row                                                    |
| `ProjectNote`, level 2 under a section           | `req_in_drawer`                                                                                         | section entry (today)                                               |
| `Ineligible`                                     | today's semantics (notebooks still refuse checkboxes; `@route` lists still strip `[ ]` on list bullets) | today                                                               |

**Checks, in order, per bullet:**

1. position: `req_in_drawer`, then `req_parent`;
2. `req_state`: any mark other than space;
3. `req_depth`: depth above `MAX_REQ_DEPTH`.

A refused bullet's subtree reports no further req diagnostics. `bob capture` refuses the
first problem in draft order as a coded usage error. `capture-parse` reports every
problem as an error diagnostic on the bullet's full line range.

**Empty req.** `- [ ] ` with no text at an admitted req position:

- with no children, it is a harmless placeholder, like a marker-only `- ` row: no
  diagnostic and no write, but `capture-parse` still emits its `req_marker` span;
- with children, it reports `empty_child_after_markers` on its line.

**Copy.** These strings are exact. Hoist them into one shared place (for example a
`task_reqs::copy` module) and make `task_block_layout`'s writers use the same constants:

| Code            | Copy                                                                            |
| --------------- | ------------------------------------------------------------------------------- |
| `req_in_drawer` | `Drawers hold notes — put the requirement directly under the task`              |
| `req_parent`    | `A requirement needs a task or requirement directly above it`                   |
| `req_state`     | `Requirements start open — they're met when the task closes`                    |
| `req_link`      | `Requirements aren't planned — link “{Owner}”, or Ctrl+Shift+T makes it a task` |
| `req_depth`     | `Requirements nest at most 16 deep — carry this branch into its own task`       |

`req_link` needs the vault, so it is an execution refusal (`bob capture`, dry run
included) in `existing-target-writes`. `capture-parse` stays lexical, and the Mac sees
`req_link` through its dry-run preview error.

### The `req_marker` span

- **Span kind.** Add a new kind, `req_marker` (`SpanKind::ReqMarker`). It covers the
  `[`, the mark, the `]`, and one following space or tab when present. It never covers
  the bullet marker or the text, so `merge_spans` keeps any route or other span on the
  same bullet.
- **Where it is emitted.** Only on admitted reqs: bullets, the `SameLineReq` item line,
  and empty placeholders. Refused checkboxes get no span.
- **Depth.** Add `depth: Option<u8>` only on the wire `CaptureParseSpan`, holding the
  req depth (1 = direct child of the owner). The internal `Span` stays unchanged. Carry
  the depth in the editor item's req list and stamp it when the wire spans are built.
- **`sub_bullet_depths`.** It stays the authored level after any list shift, and may now
  exceed 2.

## Phases

### depth-model — numeric depth, no behavior change

1. In `capture_language/model.rs`, make `AuthoredDepth` a `Copy` newtype over a numeric
   level. Use `FIRST` / `NESTED` constants, `level()`, `indent_units()`, `is_first()`,
   and `is_nested()` (true for level ≥ 2). Rename the ~158 `AuthoredDepth::First` /
   `::Nested` uses mechanically. Exhaustive `match`es gain explicit arms.
2. In `draft.rs`, `classify_authored_line` recognizes a bullet with exactly `2·(L−1)`
   leading ASCII spaces as level L, with a sanity bound (for example L ≤ 64). Odd
   counts, tabs, and mixed prefixes stay `Invalid`. Add
   `classify_authored_line_capped(line, max_level)`, which returns `Invalid` for deeper
   levels. With `max_level = 2` it is exactly today's classifier.
3. **Audit every caller.** There are about 45; find them by searching for
   `classify_authored_line`. Each gets one of two treatments:
   - **Capped at 2: today's behavior byte for byte.** This covers every caller serving
     an `Ineligible` host or a first-level-only rule: `notebook.rs`, `record.rs`,
     `close_log.rs`, `section_bullet.rs`, `editor_pomodoro.rs`, the drawer menu and
     section paths in `completion.rs`, and the notebook preflight in `rewrite.rs`.
   - **Uncapped, with level ≥ 3 handled like level 2 (`is_nested()`).** This covers the
     task-host lexical helpers that today treat nested lines as the item's children:
     `tokens.rs` (parent-task selectors), `dependencies.rs` child lines, `type_marker`,
     `target_runs` / `item.rs` child runs, the completion child-line contexts,
     `drawer_bullet.rs`, and `draft.rs::group_list_blocks`.

   In the task-host parsers (`item.rs` main and list paths, and their `editor_parse.rs`
   twins), a level ≥ 3 line keeps refusing with today's `invalid_child_line` code, copy,
   and range. `req-positions` lifts that refusal beneath req chains.

4. Generalize `drawer_bullet.rs::first_level_groups` and its users: a group is a
   first-level bullet plus every following bullet with level ≥ 2. Replace `== Nested`
   filters in `capture/plan.rs` (`plan_multi_note_sub_bullet_item`),
   `capture_project_note.rs`, `capture/project_note.rs`, and `capture/move_choice.rs`
   with level-aware code that still produces today's output for levels 1–2.
5. **Depth-preserving renders.** Every write path must keep an authored subtree intact
   once `req-positions` admits deeper levels, so a level-3+ bullet is never dropped or
   flattened:
   - new-task rendering (`render_authored_sub_bullets`) indents by level;
   - `plan_multi_note_sub_bullet_item` renders a group's nested lines at
     `indent × (level − 1)` relative to the group head, instead of skipping non-`Nested`
     children;
   - `capture_project_note.rs` renders nested lines under a row at `\t × (level − 1)`.

   Levels 1–2 render exactly as today. Unit-test the pure renderers with level-3 bullets
   (the `bullet(body, depth)` helpers in `capture_project_note.rs` show the pattern).

6. **Gate.**
   - `just check` passes with existing tests unchanged, except for mechanical renames
     inside test helpers.
   - Add unit tests for the classifier: levels 1–4, odd indentation, tabs, the capped
     form, and the sanity bound.
   - No docs or grammar revision change.
   - Commit as `refactor(capture): …`.

### req-positions — resolver, refusals, span, grammar 29

1. **Resolver.** Add `capture_language/req_bullets.rs` per the shared design.
   - The input is a host plus a slice of bullet shapes: level, checkbox mark, drawer
     header or section flag, empty body, and byte range. The output is one role per
     bullet (`Note`, `Req { depth }`, `DrawerHeader`, `DrawerEntry`, `ProjectRow`,
     `Refused`) plus the refusals.
   - Add `ReqHost::from_kind(...)` for execution and `ReqHost::from_mode(...)` for the
     editor, next to each other, so the two mappings are reviewed together.
   - Add table-driven unit tests for every row of the roles table and every check.
2. **Execution parser** (`item.rs` main path and task-list path; `drawer_bullet.rs`
   `finish_execution_drawer_bullets`):
   - admit level ≥ 3 beneath req chains;
   - record each bullet's role (add a `req: Option<ReqBulletInfo>` with the depth to
     `AuthoredSubBullet`, or an equivalent);
   - refuse with the shared codes through `CaptureItemParseError` / `coded(...)`, and
     map them in the `code_for_message` tables so `bob capture --format json` reports
     `code`.

   The same-line form is recognized here: `SameLineReq` items keep their checkbox in the
   body and carry the req role. In task hosts, a marker-only row at any even level
   (`   -`, which deeper typing produces) is a harmless placeholder, like `- ` today.

3. **Editor parser** (`editor_parse.rs` main path and task-list path): call the same
   resolver; push refusals as diagnostics on each bullet's line range; collect req
   markers with depth.
4. **Wire.**
   - In `capture_parse.rs`, add the `req_marker` span and the `depth` field on
     `CaptureParseSpan`.
   - `sub_bullet_depths` may exceed 2.
   - Keep `merge_spans` behavior for co-located spans.
   - The human `capture-parse` output lists `req_marker` like other spans.
5. **Interim writes (no data loss).** Until the next two phases place reqs by zone,
   admitted reqs write through `depth-model`'s depth-preserving renders, in authored
   order at today's note slots. Add a CLI regression test per path (new task, multi-note
   `@route+id`, single-note, project-note row) proving a level-3+ bullet is written at
   its depth, never dropped or flattened.
6. **Grammar revision 29.**
   - Bump `GRAMMAR_REVISION` from 28 to 29. Prepend to its doc comment: "Revision 29
     admits requirement bullets: checkbox bullets under a task or requirement are
     requirements, nest beyond two levels beneath requirement chains, report
     `req_marker` spans with a depth, and refuse `req_in_drawer`, `req_parent`,
     `req_state`, and `req_depth`."
   - Update the hard-coded help strings, test assertions (`tests/cli/capture/*`,
     `capture_complete` tests, `capture_language/tests/notebook.rs`), and the docs that
     cite 28: `docs/reference/compatibility.md`, `docs/reference/json-outputs.md`,
     `docs/capture.md`, `docs/reference/capture-syntax.md`, and
     `docs/reference/cli/capture-{parse,complete}.md` via `just docs-gen`.
7. **Docs.**
   - `docs/capture.md`: the bullets and drawer-bullet sections gain the requirement
     positions, levels, refusals, and the `req_marker` span.
   - `docs/reference/capture-syntax.md`.
   - `docs/reference/json-outputs.md`: the span kind and `depth`.
   - The `sub_bullet_depths` wording in `docs/reference/mac-capture/protocol.md` and
     `capture-previews.md`: depths can exceed 2 beneath requirements.
   - `docs/task-requirements.md` §5: positions and refusals have shipped. Writes land in
     the next phases.
   - The `bob capture` / `capture-parse` long help where it describes bullets.
8. **Tests.**
   - Language tests: a new `capture_language/tests/reqs.rs`.
   - CLI tests: a new `tests/cli/capture/parse_reqs.rs` and a `bob capture` refusal test
     file, registered in `tests/cli/capture/mod.rs`.
   - Cover every host, depth 3 and deeper, notes between req branches, a note's nested
     note under a req, the level-jump refusal, the depth bound at 16/17, an empty
     placeholder, task lists after the shift, and sealed project notes.
   - Add a **parity test**: over one corpus of drafts, the `bob capture --dry-run` code
     equals the first `capture-parse` diagnostic code, and req depths agree.
9. **Gate.** Run `just check`, `just docs-gen`, and `just docs-check`. Commit with `!`
   (`feat(capture)!: …`).

### new-block-writes — reqs in blocks capture creates

1. **New task blocks** (`capture/plan.rs`, around `render_authored_sub_bullets` and
   `assemble_capture_block`). When the authored bullets hold at least one req:
   - Render the block in authored order, including clip notes and schedule-log lines.
   - Canonicalize it with `task_block_layout::plan_tidy` on the standalone block: reqs
     in the reqs zone with their subtrees, loose notes and clip notes filed into `NOTES`
     (created only when a note lands, per `with_first_note`), an authored `NOTES` drawer
     merged and kept first among the drawers, other drawers after, and logs last.
   - A block whose task is not open keeps authored order.
   - A block with no req stays byte-identical to today; existing tests pin this.
   - Assert in tests that a captured block has no `layout_violations`, and that
     `bob task tidy` would report nothing for it.
2. **Every new-task host.** `Task`, `TaskWithBlockId`, task-creating Pomodoro items, and
   `@route` task-list bullets all reach the same rendering. `&` Depends-On lines written
   later keep their existing first-child slots.
3. **Project-note rows** (`capture_project_note.rs`). Each `## Tasks` row with nested
   reqs is canonicalized the same way, with its notes in the row's `NOTES` drawer. Rows
   without reqs, sections, moves, and the `^prj` line stay byte-identical. Update the
   module comment: first-level bullets are task rows, and nested checkbox bullets are
   requirements of their row.
4. **Reporting** (additive).
   - **JSON.** Add `requirements` to each capture item result, omitted when empty: one
     entry per owner, `{owner, added, notes_filed, notes_created, summary}`.
   - **Human output.** Print one line per entry, with singular forms for one:
     - dry runs and previews: `⌗⌗ adds {k} requirements to “{S}”`, plus
       ` · files {n} notes under NOTES` when `n > 0`;
     - committed: `⌗⌗ added {k} requirements to “{S}”`, plus
       ` · filed {n} notes under NOTES`.
   - Document both in `docs/reference/json-outputs.md` and `docs/capture.md`.
5. **Tests.**
   - CLI write tests in a new `tests/cli/capture/reqs_new.rs` (or similar) for:
     - every new-task host;
     - depth 3 and deeper;
     - interleaved notes, reqs, and drawers;
     - clip notes with reqs;
     - an authored `NOTES` drawer;
     - project-note rows with nested reqs and notes;
     - task lists;
     - zero-req byte identity;
     - dry-run and commit byte parity.
6. **Gate.** Run `just check`, `just docs-gen` if help changed, and `just docs-check`.
   `docs/task-requirements.md` §5 records that new blocks shipped.

### existing-target-writes — insert_reqs, the scoop, req_link

1. **Multi-note `@route+id`** (`plan_multi_note_sub_bullet_item`).
   - Collect every level-1 req group in draft order. Render each as an `insert_reqs`
     block with relative indentation in the target note's indent unit.
   - Run **one** `insert_reqs` call on the staged target before planning the item's
     notes. Its scoop then counts only pre-existing loose notes.
   - Plan the notes and drawer bullets as today. They file into `NOTES` through
     `note_home`.
   - A reqs-only item is valid: drop the "no notes planned" invariant for it, and take
     the parent details from the insert.
   - Map `WriteRefusal` codes to `CaptureError::usage_code`.
   - The whole draft stays all-or-nothing.
2. **The same-line form** (`SameLineReq` in `plan_capture_item`'s sub-bullet branch).
   The item line plus its level-1+ bullets form one `insert_reqs` block under the
   target. It applies to forced `--task` / `--task-ref` targets too. A forced `--drawer`
   with a req refuses `req_in_drawer` at execution.
3. **`^prj` targets.** Reqs insert under the `^prj` line like any task's, and notes keep
   `note_home`'s shipped behavior. If that interplay with `## Notes` sections looks
   wrong, record a `PROPOSED FOLLOW-UP:` instead of changing the layout module here.
4. **`req_link`.**
   - Where a Task Link resolves an existing task by block ID
     (`capture/pomodoro_link.rs`, and the close/link paths that share its "no task with
     block ID" resolver), a miss whose anchor names a req line (`task_reqs` anchor)
     refuses `req_link` with the owner's clean text.
   - Other block-ID resolvers keep today's errors.
   - Add CLI tests.
5. **Reporting.**
   - Reuse the `requirements` entries from `new-block-writes`; `notes_filed` is the
     scoop count.
   - Notes keep their `SubBulletNoteJson.drawer`.
   - The dry run prints the scoop line, so the Mac preview shows it.
6. **Tests.**
   - CLI tests in a new `tests/cli/capture/reqs_existing.rs` for:
     - a first req on a zero-req target, with the scoop of loose notes and `NOTES`
       creation only when a note moved;
     - a target that already owns reqs, appending at the end of the reqs zone;
     - an existing `NOTES` drawer;
     - interleaved notes and reqs;
     - a reqs-only item;
     - depth 3 and deeper;
     - the same-line form, with and without children;
     - forced targets;
     - the forced-drawer refusal;
     - a `^prj` target;
     - `req_link`;
     - dry-run and commit byte parity;
     - all-or-nothing rollback.
7. **Gate.** Run `just check` and `just docs-check`. Update `docs/capture.md` and
   `docs/task-requirements.md` §3/§5 for the shipped writes.

### req-rewrite — `expand_req_marker`

1. **Rule.** Add `RewriteRule::ExpandReqMarker` (code `expand_req_marker`) to
   `capture_language/rewrite.rs`. It runs only with `--cursor`, before the sigil
   aliases.
   - **Trigger.** The byte before the cursor is a just-typed space, and the line up to
     the cursor is `{2k spaces}{-|*|+}{spaces/tabs}[ `.
   - **Rest of the line.** It must be empty, or a lone auto-paired `]`, which the edit
     absorbs.
   - **The edit.** One `TextEdit` replaces `[ ` (and the absorbed `]`) with `[ ] `, with
     the caret after it. Undo in the client gives back `- [ `.
   - **No-fire cases.** `- [x`, `- [[`, `- [link`, and item lines never match.
2. **Eligibility.** Build the candidate draft with the expanded marker plus placeholder
   text, and run the shared resolver on that item.
   - It fires only when the bullet resolves to an admitted req.
   - An item without a typed route is a new task.
   - It never fires on a sealed project-note first-level bullet, in a drawer, under a
     note, or in an `Ineligible` host.
   - Extend the `capture-rewrite` adapter's notebook preflight
     (`bullet_sigil_may_need_notebook_policy` or a sibling) so a `- [` + space under a
     notebook route loads the policy and stays literal.
   - Templates never rewrite.
3. **Client contract (docs only; bob returns just the expansion).** Document these in
   the `bob capture-rewrite` long help and in `docs/capture.md`:
   - **Overtype absorption.** Straight after an expansion, a typed `]` is absorbed, and
     so is a space typed right after it. The first other keystroke or caret move ends
     absorption, so typing `- [ ` or `- [ ] ` gives the same bytes.
   - **Atomic Backspace.** With the caret right after a bullet-start `req_marker` span,
     Backspace removes the whole `[ ] `, leaving `- `.
   - **Never rewritten.** Paste, templates, and `bob capture` never rewrite.
   - Also add the rule to `docs/reference/json-outputs.md` and to
     `docs/task-requirements.md` §5, and add one sentence to the revision-29 doc comment
     on `GRAMMAR_REVISION`. A release that ships 29 before this phase lands is harmless,
     because clients apply whatever rule bob returns.
4. **Tests.**
   - Rewrite tests in `capture_language/tests/rewrite.rs` (or a new `rewrite_reqs.rs`):
     - every eligible host and depth;
     - auto-paired `]`;
     - every no-fire case;
     - notebook routes, with the policy;
     - sealed project-note first-level bullets;
     - the exact edit and undo bytes.
   - `bob capture-rewrite` CLI tests asserting `rule: "expand_req_marker"` and the
     cursor.
5. **Gate.** Run `just check`, `just docs-gen`, and `just docs-check`.

## Landing and completion

The child epic's land agent owns final integration and closes **`bob-cli-on.6`** after
every child phase lands. It verifies the whole phase, not merely that the child beads
closed:

- **Walkthrough.** In a temporary vault, capture each of these, dry run then commit:
  - a new task with three req levels, interleaved notes, and a drawer;
  - a first req on a zero-req `@route+id` target with loose notes, then a second capture
    to it;
  - the same-line form;
  - a project-note row with nested reqs and notes.

  Confirm that `capture-parse` spans and depths match the written bytes, and that
  `bob task tidy` then reports nothing.

- **Gates.** `just check`, `just docs-gen` (no diff), and `just docs-check` pass on the
  integrated tree. Hand long commands to `/sase_monitor`, or run them in the foreground
  with explicit long timeouts. A failure that reproduces identically on the clean base
  is recorded with evidence and does not keep the phase open.
- **Contract.** `docs/task-requirements.md` §5 reads as shipped for capture, with the
  Mac and Obsidian parts still attributed to their phases.
- **Follow-ups.** Carry the child beads' unresolved `PROPOSED FOLLOW-UP:` notes onto
  `bob-cli-on.6` for the outer land agent. Do not create beads.
- **Epic symbols.** Run `sase bead epic-symbols bob-cli-on.6`, and the same for each
  child bead before closing it, and resolve or re-key every leftover.
- **Closing.** Close only `bob-cli-on.6`, with a note citing the landed commits and the
  verification. Leave `bob-cli-on` to its own land agent. Finish every SASE turn through
  `/sase_final`.
