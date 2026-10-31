---
tier: epic
title: 'Project-note seals: @route^id@, @route:id@, and the project successor prompt'
goal: 'A project note is sealed with `@`. `@route^id@` creates it, and `@route:id@`
  creates it and then lets Bryan pick which of its tasks get Pomodoro Task Links in
  the existing successor choice card. `+` typed after `@route^id` now switches to
  `@route+id`, like every other separator. Bob Mac Capture draws both sealed markers
  as one "manila chip" so a project note is easy to spot in a draft.

  '
decisions:
  initial_marks:
    ask: When the project successor prompt opens, should every offered task start
      marked?
    choices:
      none: Nothing marked, as on every successor card; Return links the highlighted
        task, ⌘A marks all
      all: Every offer starts marked; Return links them all and Space unmarks the
        ones to leave out
    default: none
    why: A new project usually needs one or two next actions in today's plan, not
      its whole backlog
    answer: none
  memory_glossary:
    ask: Update the Successor Offer and Task Successor glossary strands for @route:id@
      project offers?
    memory:
    - glossary:successor-offer
    - glossary:task-successor
    corrects: pick links the task, raises it to Next, and stamps it fresh.
    default: true
    answer: true
phases:
- id: seal_grammar
  title: Seal the caret project-note form with @ in bob
  depends_on: []
  size: medium
  description: 'seal_grammar: bob-cli switches `@route^id+` to `@route^id@` in the
    execution grammar, editor spans, completion, type markers, and `@@` handling.
    The old `+` forms are retired with teaching diagnostics. `capture-parse` emits
    a separator span and the `project_note: "create"` span modifier. `+` after `@route^id`
    now toggles to `@route+id`. CLI help, docs, and tests are migrated.'
- id: offer_form
  title: '@route:id@ creates the project and offers its tasks'
  depends_on:
  - seal_grammar
  size: large
  description: 'offer_form: bob-cli adds the `@route:id@` grammar with `project_note:
    "offer"` spans, plus sealed `:`/`^` toggles. The project-note planner offers the
    new note''s open tasks (never `^prj`) as successor offers in `project_note.successors`,
    using the existing `-p/--pull-in` and `--require-pull-in-choice` plumbing. A pick
    renders exactly like a ` :` bullet. Docs, glossary memory, and tests are updated.'
- id: mac_seal
  title: Mac manila-chip highlighting and @ seal migration
  depends_on:
  - seal_grammar
  size: medium
  description: 'mac_seal: Bob Mac Capture decodes the `project_note` span modifier
    and the new span kinds. It paints the manila chip: a brown wash, brown name and
    seal, and a green separator and seal for the offer form. It moves picker teaching
    copy, fake-bob fixtures, and separator-toggle tests from `+` to `@`, and updates
    the Mac docs.'
- id: mac_offer_card
  title: Mac project successor prompt
  depends_on:
  - offer_form
  - mac_seal
  size: medium
  description: 'mac_offer_card: Bob Mac Capture harvests `project_note.successors`
    into the existing Pull-In card. The card shows a "New project" context, a folder.badge.plus
    glyph, and a preview row. It applies the initial-marks decision, keeps notifications
    free of double counts, regenerates offer-form fixtures from real bob, and updates
    the Mac docs.'
proposed_by: bbugyi200.apollo.bp.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 12:01:09
status: wip
bead_id: bob-cli-ex
---

- **PROMPT:** [prompts/202610/project_note_seals.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/project_note_seals.md)
- **BEAD:** [bob-cli-ex](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ex/README.md)

# Project-note seals: `@route^id@`, `@route:id@`, and the project successor prompt

## Why this shape

Today a capture item creates a project note with `@route^block-id+`. That `+` blocks two
things Bryan wants:

1. `+` cannot work as a separator toggle after `@route^id`. The previous change made `+`
   after `@route:id` switch to `@route+id`, but it had to exempt `^` because
   `@route^id+` was taken.
2. There is no way to create a project note and then choose which of its tasks go into
   today's Pomodoros. The `@route:id+` spelling was retired because "a project note
   never links its own `^prj`", and ` :<id>` bullets are the only way to link.

The fix is a **seal**. A trailing `@` right after the block ID means "this ID names a
new project note". The separator before the ID keeps its normal meaning:

| Draft                                              | Meaning                                                                                              |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `@cash^goog-exit`                                  | New task with ID `goog-exit` in `cash.md` (unchanged)                                                |
| `@cash:goog-exit`                                  | Link / new Next task in `cash.md` (unchanged)                                                        |
| `@cash+goog-exit`                                  | Sub-bullet under task `goog-exit` (unchanged)                                                        |
| `@cash^goog-exit@`                                 | **New project note** `cash_goog_exit.md`; only ` :<id>` bullets are linked                           |
| `@cash:goog-exit@`                                 | **New project note, then the project successor prompt**: pick which of its open tasks get Task Links |
| `@cash^goog-exit@#admin`, `@cash:goog-exit@#admin` | Same, with links and picks going under the `ADMIN` Pomodoro                                          |

The design follows four rules:

- **`^` creates. `:` links.** That is already true for single tasks, and now it is true
  for projects.
- **`@` seals.** It cannot collide with anything. Routes allow `[A-Za-z0-9_-]` and block
  IDs allow `[A-Za-z0-9-]`, so an `@` right after an ID is an error today, never prose
  and never another construct. Every other `@` meaning (start times after `=NAME`, close
  lists `x@2`, `@@`/`@@@`, the standalone `URL @`) lives in a different slot.
- **Picks never auto-link.** This follows `decisions:unblocked-tasks-are-offered`. The
  CLI links nothing without `-p`, and the Mac writes nothing until the card is answered.
- **The last separator you type wins.** `+` after `@route^id` switches to `@route+id`.
  After a sealed marker, `:` and `^` switch between the two project forms.

### Invariants every phase must keep (and test)

1. **A pick is a ` :` bullet.** `@r:id@` with `-p all` writes the same files, byte for
   byte, as `@r^id@` with every offered bullet authored as ` :<minted-id>`. The minted
   IDs are the same ones the planner would give. Picks are therefore raised to Next
   (`[*]`, or `[?]` under a resolved schedule exactly as ` :` tasks are). They are
   retyped by `#name` and placed by type. Like every created task they are **not**
   stamped `[fresh::]`, because creation never stamps.
2. The `^prj` task is never offered and never linked. The
   `(REPLACE WITH TASK DESCRIPTION)` placeholder is never offered.
3. Offer keys are stable between preview and submit, using `note_tasks::stable_task_key`
   on the pre-pick render.
4. Apart from the marker spelling and the new span data (separator span plus modifier),
   an `@route^id@` capture produces the same JSON and human output as `@route^id+` did.
5. An old spelling never gets a silent new meaning. `@route^id+` and `@route:id+` fail
   with a teaching diagnostic.

### Non-goals

- No changes in bob-plugins or Obsidian. The nav "Create project note from task" flow
  and its promotion successors are unchanged.
- No changes to ` :<id>` / ` ^<id>` bullet semantics, `^prj` rendering, the successor
  engine for Done gestures, or freshness rules.
- No automatic rewrite of saved templates, stashes, or Keep notes that still say `^…+`.
  They get the teaching diagnostic, and Bryan re-saves them.
- bob-cli ships on the daily release train. Agents never edit the `Cargo.toml` version
  or `CHANGELOG.md`. Mark CLI/JSON contract breaks with `!` in the commit subject.

## Contracts (shared by every phase)

### Grammar and diagnostics (final state after `offer_form`)

| Input                                               | Result                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@r^id@`, `@r^id@#name`, `@r^id@#"Lit Name"`        | `CaptureKind::ProjectNote { form: Create, … }`                                                                                                                                                                                                                                                                                      |
| `@r:id@`, `@r:id@#name`, `@r:id@#"Lit Name"`        | `CaptureKind::ProjectNote { form: Offer, … }`                                                                                                                                                                                                                                                                                       |
| `@r^id+`, `@r^id+#name`                             | `retired_project_note_marker`: `` `@cash^goog-exit+` is retired: a project note is sealed with `@`. Write `@cash^goog-exit@`, or `@cash:goog-exit@` to also choose which of its tasks to link `` (with `#bugs`, both suggestions carry `@#bugs`). With no `#name`, append ``(`@cash+goog-exit` adds a sub-bullet under that task)`` |
| `@r:id+`, `@r:id+#name`                             | `retired_project_note_marker`: `` `@cash:goog-exit+` is retired: a project note is sealed with `@`. Write `@cash:goog-exit@` to choose which of its tasks to link, or `@cash^goog-exit@` to link none ``, with the same `#name` and sub-bullet handling                                                                             |
| `@r^id#name@`, `@r:id#name@`                        | `invalid_project_note_marker`: ``put the project-note `@` right after the block ID: `@cash^goog-exit@#bugs` (an `@` after `#bugs` would be part of the Pomodoro name)``                                                                                                                                                             |
| `@r^id@x`, `@r^id@@`, `@r^id@+` (and the `:` forms) | `invalid_project_note_marker`: `` the project-note `@` ends the marker: write `@cash^goog-exit@` or `@cash^goog-exit@#<pomodoro>` ``                                                                                                                                                                                                |
| `@r^id@=…`, `@r:id@=…`                              | the existing project-note start/close errors, with the `@` spelling                                                                                                                                                                                                                                                                 |
| `@r^@`, `@r:@`                                      | the family's existing "requires a block ID" wording                                                                                                                                                                                                                                                                                 |
| `@^id@`, `@:id@` (no route)                         | editor: project-note mode, `needs: ["route"]`; execution: the existing route-required wording                                                                                                                                                                                                                                       |
| `@@r^id@`, `@@r:id@`                                | `invalid_global_destination`: `@@ cannot take a project note: leave @cash^goog-exit@ on this item, or delete it`                                                                                                                                                                                                                    |
| `@r+id@`                                            | `invalid_sub_bullet_block_id`, plus the hint `` `+` adds a sub-bullet under an existing task; a new project note is `@cash^goog-exit@` (or `@cash:goog-exit@`) ``                                                                                                                                                                   |
| `^r:id@` (caret link item)                          | `invalid_pomodoro_link` (existing message with the `@` spelling)                                                                                                                                                                                                                                                                    |

`seal_grammar` ships the `^` rows. Until `offer_form` lands, its copy names only
`@cash^goog-exit@` and `@r:id@` stays invalid exactly as it is today. `offer_form` then
adds the `:` rows and the `:` suggestions to the shared copy. Fix the stale claim in
`docs/capture.md` (~4483-4486): `@cash^goog-exit#bugs` parses as a typed task. It does
not produce a "needs the project-note `+`" hint, and the `grammar.rs` test asserts this.

### `capture-parse` spans

The span kinds for the route and the ID stay the same in each family, so route and
block-ID completion keep working, even on an older Mac. The changes:

- New kind **`project_note_separator`** covers the `^` or `:` byte, but only inside a
  project-note marker. Other markers keep their separators unspanned.
- **`project_note_marker`** now covers the `@` seal of the `^` form (it used to cover
  `+`).
- New kind **`project_note_offer_marker`** covers the `@` seal of the `:` form.
- New optional span field **`project_note`** has the value `"create"` or `"offer"`.
  Every span of the marker token carries it: route, separator, block ID, seal, and any
  `interactive_placeholder` that stands in for a missing route or ID. It is omitted
  everywhere else, so other drafts stay byte-identical. It never appears on the
  `pomodoro_name` span; `#name` keeps its own theme-tinted pill.
  - Add it to `editor_model::Span`, and serialize it on
    `capture_parse::CaptureParseSpan` with `skip_serializing_if`, like `cancels_source`.
- Modes do not change. The `^` form reports `project_note`, or `pomodoro_project_note`
  when it has a ` :` bullet. The `:` form always reports `pomodoro_project_note`,
  because it reads the ledger.

`Finish it @cash:goog-exit@#admin` (offsets in bytes):

| Bytes | Text        | Kind                                                        | `project_note` |
| ----- | ----------- | ----------------------------------------------------------- | -------------- |
| 10–15 | `@cash`     | `pomodoro_route` (`task_block_id_route` for `^`)            | `offer`        |
| 15–16 | `:`         | `project_note_separator`                                    | `offer`        |
| 16–25 | `goog-exit` | `pomodoro_block_id` (`task_block_id` for `^`)               | `offer`        |
| 25–26 | `@`         | `project_note_offer_marker` (`project_note_marker` for `^`) | `offer`        |
| 27–32 | `admin`     | `pomodoro_name` + `themes`                                  | —              |

### Completion and rewrite

- **Block-ID intent `project_note`.** It comes from either separator followed by the
  seal (`capture_block_ids.rs` ~122-135). `marker_token_range` includes the seal.
  `completion_field_from_parts` strips a trailing seal instead of `+`. There is no new
  intent value, because older Macs map unknown intents to `link`.
  - With no seal yet, the existing intents stay: `@cash:goo` is link/new-Next, and
    `@cash^goo` is new.
- **`capture-rewrite` `switch_block_id_separator`:**
  - `+` after `@r^id` gives `@r+id`. Remove the exemption at `rewrite.rs` ~316-322; the
    candidate must still select `Task` mode.
  - _(offer_form)_ `:` typed right after `@r^id@` gives `@r:id@`, and `^` after `@r:id@`
    gives `@r^id@`. The candidate must select the matching project-note mode with no
    needs, section, start, or close.
  - The same separator, or `+`, after a seal stays literal.
  - None of these fire with `#name` present, which matches the plain toggles.
- `@` itself is never a rewrite trigger. bob reports the seal as a non-completion span
  with no `needs`, so typing it opens no picker.

### Capture JSON for the offer form (`offer_form`)

`project_note` (`ProjectNoteSummary`, `capture/output.rs` ~452) gains two fields, both
emitted only for the offer form:

- `form: "offer"`. When it is absent, the form is create.
- `successors: [ … ]`. These rows use the `TaskCompleteUnblockedJson` shape, so clients
  reuse their decoders. The array may be empty.

```json
{
  "note_path": "cash_goog_exit.md",
  "block_id": "",
  "line": 13,
  "text": "Collect the equity paperwork",
  "previous_status_symbol": " ",
  "previous_status_name": "…",
  "status_symbol": " ",
  "status_name": "…",
  "inbox": false,
  "unblocked_by": [
    {
      "note_path": "cash_goog_exit.md",
      "block_id": "prj",
      "text": "Finish the Google exit packet!"
    }
  ],
  "trigger": "project_created",
  "hide_removed": false,
  "offer": {
    "key": "cash_goog_exit.md@1a2b3c4d",
    "n": 2,
    "group": "project",
    "selected": false,
    "block_id_needed": true,
    "project": "cash_goog_exit.md",
    "destination": {
      "day_file": "…",
      "entry_name": "ADMIN",
      "entry_role": "typed",
      "fallback": false
    }
  }
}
```

- **`unblocked_by`** names the new `^prj`. The glossary frames a promoted note's tasks
  as successors of the task whose place they take, and here the new project task is that
  predecessor. Both clients therefore always have a context line, including the refusal
  path.
- **`group` is `"project"`.** Older Macs already render it as "Project ‹stem›" with a
  folder glyph. **`trigger` is `"project_created"`**; it sits next to `PULL_IN_TRIGGER`
  and `DEPENDENCY_TRIGGER` in `task_complete/project_pull_in.rs`.
- **A picked row** reports `status_symbol` `*`, its final `block_id` and `line`,
  `offer.selected: true`, and a `link` (`TaskCompleteSuccessorLinkJson`). The pick also
  appears in `project_note.task_links`, because it is a ` :` link. Clients must not
  count it twice.
- **`pull_in.groups`** gains a
  `{group: "project", project: "<basename>", folded: {hidden, already_planned, sub_projects: 0}}`
  fold. `hidden` counts `#hide` tasks, and `already_planned` counts ` :` bullets.
- **`entry_role`** uses the existing vocabulary:
  - `typed` when the task's type, or `#name`, chooses the Pomodoro;
  - `created` when that Pomodoro will be created;
  - `implicit` (with `fallback: true`) when the task falls back to the current or next
    open Pomodoro.
- **Ordering.** The offer order key is `(1, (usize::MAX, usize::MAX), basename, line)`.
  These offers therefore sort after recovered-project offers and keep note line order in
  mixed drafts.

## Phase seal_grammar — seal the caret form with `@` (bob-cli)

Migrate the `^` project-note family from `+` to the `@` seal. This must leave the tree
shippable on its own. The touch points, from exploration at HEAD `061824f8`:

- **Model:** `capture_language/model.rs:89-101`. Add `form: ProjectNoteForm` (`Create` |
  `Offer`) to `CaptureKind::ProjectNote`. Only `Create` is produced in this phase.
- **Execution grammar** (`capture_language/tokens.rs`):
  - In `parse_task_block_id_route_token` (~491-623), the `strip_suffix('+')` checks
    become seal checks: name branch ~511-581, `=` branch ~582-597, plain ~598-622.
  - Add the retired-`+` branch, the trailing-junk branch, and the misordered `#name@`
    branch from the grammar table.
  - `parse_colon_link_tail` (~848-888) keeps refusing `@r:id+` with the updated retired
    copy.
  - Also update `retired_route_less_project_note_error` (~2292-2317),
    `classify_caret_close_suffix` (~1638-1646), the sub-bullet `@r+id@` hint, and the
    error constructors in `markers.rs` (~361-432, 666).
  - Body-less leading markers (`line.rs` ~140-170) keep the missing-text error.
  - `item.rs` ~1146-1188 keeps the unused `#name` check for `Create`.
- **Editor grammar** (`editor_classify.rs` `classify_task_block_id_token` ~317-560,
  `marker_parse` ~1406-1528):
  - Do the same seal parsing.
  - Emit `project_note_separator` and the `@` `project_note_marker`, with the
    `project_note: "create"` modifier on every marker span, including placeholders.
  - Keep `needs` behavior: `@^id@` needs `route`, `@r^@` needs `block_id`, and `@r^id@#`
    needs `pomodoro_name`.
- **Type markers** (`type_marker.rs` ~353-410, 465-485): the `ends_with('+')` tests and
  the misordered suffix move to the seal.
- **Completion:**
  - `completion.rs` `marker_field_at_cursor` (~1579-1749) and
    `completion_field_from_parts` (~1898-2034);
  - `capture_block_ids.rs` (~122-135, ~275-281);
  - `capture_complete` shell (~768-776, 839-845).
- **Rewrite** (`rewrite.rs`):
  - Remove the `+` exemption and update the doc comments at ~287-291, ~419-422, ~582.
  - Update the `@@` notices (`classify_local_marker` ~601-616,
    `non_absorbable_marker_notice` ~646-648) to the new spelling.
- **User copy:** the `capture/task_theme.rs` ~819-822 project-note hint, and the
  help/examples in `capture/cli.rs` (~388-410, 483-484), `capture_parse.rs` (~327-338),
  `capture_complete/cli.rs` (~122, 234-236), and `capture_rewrite.rs` (~110-123).
  Regenerate `docs/reference/cli/*.md` with `just docs-gen`; never hand-edit them.
- **Docs:**
  - Every `^…+` mention in `docs/capture.md` (32 occurrences),
    `docs/reference/capture-syntax.md` (8; rename the anchor `{#project-plus}` to
    `{#project-note-seal}` and update its links), `docs/reference/vault-format.md` (3),
    the `+` sigil prose in `docs/task-pickers.md` ~185 and `docs/task-types.md` ~645,
    and `docs/_meta/search_smoke.yml` (query and location).
  - The toggle docs (`capture.md` ~6263-6295) must say `+` now switches `@route^id` too.
  - Rewrite the "`+` sigil sits immediately after the block ID…" rationale for the seal:
    an `@` can never be part of a route, ID, or Pomodoro name prefix.
- **Tests:** migrate all of them, including:
  - `capture_language/tests/{grammar,editor_modes,editor_spans,completion,rewrite}.rs`;
  - `tests/cli/capture/{project_note,parse,complete_block_id,rewrite}.rs`;
  - the incidental files listed by exploration (`capture_type_marker.rs`,
    `typed_placement.rs`, `task_theme_writes.rs`, `related.rs`, `plan_budget.rs`,
    `open_notes.rs`, `parse_blocked_task.rs`, `pomodoro_name.rs`, `pomodoro_start.rs`).

  Add tests for each `^` row of the grammar table, for the span table (separator plus
  modifier), for `rewrite_draft_caret_plus_switches_to_plus` (replacing
  `…_stays_a_project_note_marker`), and for invariant 4.

Verify with `just fmt`, `just lint`, `cargo test --test cli capture`, `just test`,
`just docs-gen`, and `just docs-check`. Commit as
`feat(capture)!: seal project notes with @ instead of +`.

## Phase offer_form — `@route:id@` creates the project and offers its tasks (bob-cli)

### Grammar

- Add the `:` rows of the grammar table:
  - execution: `parse_colon_link_tail` and the `:`-family dispatch;
  - editor: `classify_pomodoro_token` ~572-663 emits the offer spans with
    `project_note: "offer"` and mode `pomodoro_project_note`;
  - completion: route-less `@:id@` needs `route`; `@r:id@#` gives Pomodoro name
    completion; the block-ID intent is `project_note`;
  - `type_marker.rs`;
  - the sealed `:`/`^` toggles in `switch_block_id_separator`. Extend
    `plain_block_id_parts` and `candidate_selects_plain_marker` to accept one trailing
    seal, and keep the seal in the replacement.
- A solo `@r:id@` item with no body gets the missing-text error. It must never reach the
  solo `@route:id` to `PomodoroLink` conversion (`item.rs` ~938-967).
- `#name` on the offer form is never flagged at parse time. The planner refuses it only
  when the note has nothing to link or offer:
  `` `#admin` picks the Pomodoro for this project's task links, but it has no open task to offer or ` :` task to link; add a task bullet or remove `#admin` ``.
- Update the retired and sub-bullet copy to the final text in the grammar table.

### Planner (`capture/project_note.rs`, `capture_project_note.rs`)

- **Rendered tasks.** `render_project_note` also returns every rendered `## Tasks` row
  as `RenderedTask { line, status, block_id: Option, text, link, hidden, placeholder }`.
  This extends `named_tasks`.
- **Offerable rows.** These are open (`[ ]`, `[*]`, `[/]`) and not `^prj`, not the
  placeholder, not ` :`-linked (counted as `already_planned`), and not `#hide` (counted
  as `hidden`). `[?]`, terminal, and section lines are excluded.
  - When the project is blocked by a future schedule, the existing `project_schedule`
    `link_guard` applies and nothing is offered.
- **Keys.** Use
  `note_tasks::stable_task_key(basename, block_id, text, global_filter, ordinal)` on the
  pre-pick render.
- **Selection.** Read `request.selection` (`SuccessorSelection` None/All/Keys), the
  field `pull_in::plan_with_pull_in` already sets for every item.
- **Picks become ` :` bullets.**
  - Before rendering again, set `link = true` on the picked sub-bullets.
  - Give an ID-less pick `capture_block_ids::mint_block_id(text, used)`, where `used`
    holds every authored ID, `prj`, and earlier mints.
  - Then run the existing ` :` path unchanged: day-note staging, `#name` /
    `placement_for_task` destinations, `insert_pomodoro_block_link`, and
    `apply_new_task` retyping. This is invariant 1, and it reuses the same "ledger
    already contains" and missing-day-note errors.
- **Daily note.** It is read only when at least one ` :` bullet exists or something is
  offered.
- **Destinations for unpicked offers.** Run the same placement on a scratch copy of the
  day text and discard it. Refactor the link loop (~227-275) into one helper that
  returns each task's insertion destination, so both uses share one code path.
- **Rows.** Build the `successors` rows and the fold from the contract.

### Offer plumbing (`capture/output.rs`, `capture/pull_in.rs`, `capture/task_complete.rs`)

Every reader of the offer universe sees `project_note.successors`:

- `all_unblocked`, `unblocked_vecs_mut`, and `all_project_folds` (~1788-1864);
- `completion_summary_count`, so a single project-note item keeps its engine numbering
  `1..k`;
- `number_draft_offers`, `reject_unknown_keys`, `unknown_pick_error`, and
  `choice_required_error`, which provide `--require-pull-in-choice` refusals with flat
  `offers`;
- the batch net pass (`apply_successor_net_post_pass` ~1000 / `keep_unblocked_row`), so
  a later item in the same batch that links or closes a new task drops its offer.

Human output (`print_human_successor_rows` ~3898) prints a `New project <basename>`
header for `project_created` rows. `print_pull_in_hint` works unchanged.

### Docs, memory, tests

- **`docs/capture.md`.** Add an "Offer a new project's tasks" subsection under Project
  notes. It covers the example draft, the dry-run JSON excerpt, the `-p 1,3` / `-p all`
  / `-p none` CLI flow, ` :` / `^` / `#hide` interplay, blocked projects, and
  invariant 1. Also update the span list (~6062-6095), the modes text (~5850, ~5915),
  the toggles section, and the `pull_in` and `-p` text (~2708, ~4885).
- **Other docs:**
  - `docs/reference/capture-syntax.md` (offer row and section);
  - `docs/task-dependencies.md` §12.1 trigger table (`project_created`), §12.5 (the
    `project_note.successors` location, trigger, and fold), and §12.14;
  - `docs/projects.md` "Promotion successors" (capture's `@route:id@` is promotion at
    birth).
- **Memory.** Edit memory only as the decisions below allow, then run `sase memory init`
  and `just docs-gen` (which regenerates `docs/reference/glossary.md`).

> [!decision] memory_glossary
>
> - `glossary:successor-offer`: rewrite the last two sentences so they state:
>   - a capture `@route:id@` offers the new project note's open tasks (never `^prj`);
>   - a pick links the task and raises it to Next;
>   - a pick of an existing task also stamps it fresh, while a task the same capture
>     creates is never stamped.
>
>   Keep `[[decisions:unblocked-tasks-are-offered]]`.
>
> - `glossary:task-successor`: add that creating a project note with capture's
>   `@route:id@` makes the new note's open tasks successors of its `^prj` task, the same
>   way promotion does. Also add that only an existing task's pick is stamped fresh, and
>   update its `summary:` to match.

- **Tests** (`tests/cli/capture/`: extend `project_note.rs`, plus unit tests):
  - preview offers, keys, and `n`;
  - invariant 1, comparing `-p all` against authored ` :<id>`;
  - `-p <key>`, `-p 2`, and `-p none`;
  - `--require-pull-in-choice` refusal carrying `offers`;
  - `^prj` and placeholder never offered;
  - the `#hide` / ` :` folds;
  - blocked project;
  - `#name` destination, plus a created placeholder only when something lands there;
  - typed placement;
  - minted IDs avoiding authored IDs;
  - multi-item numbering next to a `!` completion;
  - the sealed toggles and the full grammar table.

Verify with `just fmt`, `just lint`, `cargo test --test cli capture`, `just test`, and
`just docs-check`. Commit as
`feat(capture): offer a new project's tasks with @route:id@`.

## Phase mac_seal — manila-chip highlighting and the `@` migration (bob-mac-capture)

Open the linked repo with `/sase_repo` and follow its `AGENTS.md`.

### The look

A project-note marker should read as one object, a folder tab sitting in the sentence:

```
Finish the Google exit packet!  ▕@cash^goog-exit@▏#admin      create: blue route, brown name + seal
Finish the Google exit packet!  ▕@cash:goog-exit@▏#admin      offer:  blue route, brown name, green : and @
                                 └─ soft manila wash ─┘ └ theme pill (unchanged)
```

- **The wash.** Every span with a `project_note` value gets a soft `NSColor.systemBrown`
  background: about 0.16 alpha, rising to about 0.26 under Increase Contrast. Tune it by
  eye in light and dark appearances. Spans are contiguous across the marker, so the wash
  forms one chip. `#` stays unspanned, so the `#admin` theme pill reads as a separate
  object. This uses the existing background path in `paintHighlighting`
  (CapturePanelModel.swift ~13194-13333), and `clearEditorHighlightAttributes` already
  resets backgrounds.
- **Brown means "new project note"** (a manila folder). Brown is the only system hue the
  editor palette has not already given a role. Green on the offer form's `:` and `@` is
  the Pull-In card's "will be linked" color, so the marker tells you a choice card will
  follow.
- **Categories.** Add these to `CaptureSemanticCategory` (CaptureCore
  `CompletionRowContent.swift`) and `CaptureEditorPalette.color(for:)`:
  - `.projectNoteName` (brown);
  - `.projectNoteSeal` (brown);
  - `.projectNoteOfferSeal` (green).

  In `captureSemanticCategory(forSpan:)`:
  - a block-ID span with a `project_note` value maps to `.projectNoteName`;
  - `project_note_separator` follows the modifier (`create` gives seal, `offer` gives
    offer seal);
  - `project_note_marker` maps to `.projectNoteSeal`;
  - `project_note_offer_marker` maps to `.projectNoteOfferSeal`;
  - routes stay `.route`.

  Unknown modifier values still get the wash but no recoloring. Unknown kinds stay
  `.neutral`.

- **Spans must stay valid.** `validatedSpanRanges` rejects overlaps, so the wash is
  painted per span and never as an outer range.

### The work

- **`CaptureSpan`** (CaptureModels.swift ~784-846): decode the optional `project_note`
  string tolerantly.
- **Teaching and detail copy:**
  - `CapturePickerView.teachingLine`: `^` IDs say "Then type @ for a project note
    (@#name picks its Pomodoro)". `:` IDs on an item with text say "Then type #name, =
    to start, or @ for a project note whose tasks you pick".
  - New-ID detail strip: for project-note intent with marker `:`, say "the new project
    note, then choose which tasks to link".
  - Tint the Block ID Picker's "Project note" caption brown so it matches the chip.
- **Typing `@`.** Make sure typing `@` right after `@route^id` or `@route:id` opens no
  route picker and no completion, and is not sent to `capture-rewrite`. If the local
  route-completion fallback (`leadingSingleAtRouteReplacementRange` / `isRouteBoundary`
  ~13579-13646) would treat the seal as a new route, fix it there.
- **Fixtures.** Migrate every `^…+` draft and reply in `Tests/Fixtures/fake-bob` (~1618,
  ~3792, …) and the JSON fixtures (`block-id-project-note.json`,
  `block-id-project-task.json`, `project-note-task-links.json`, …). Regenerate the
  generated ones with the documented `BOB_BIN` commands against a bob built from
  `seal_grammar`. Hand-author offer-form parse replies exactly per the span table;
  `mac_offer_card` re-checks them against real bob.
- **Tests:**
  - `testPlusAfterCaretProjectNoteMarkerStaysLiteral` becomes `@file^id` + `+` giving
    `@file+id` through a native single-edit undo step;
  - sealed `:`/`^` toggles through the fake-bob rewrite replies;
  - `@` opens nothing;
  - category mapping (`CompletionRowContentTests`);
  - wash and colors (`CaptureEditorHighlightTests`), with the chip contiguous and
    `#name` unwashed.
- **Docs** (this repo's `docs/`, mounted into Bob's site):
  - `capture-panel.md` (~169-170, ~209-222): the `+` toggle now covers `^`, plus the
    sealed toggles and the chip;
  - `task-pickers.md` (~178-298): `+` becomes `@`, plus the teaching lines;
  - `protocol.md` (~62-65): span kinds, the modifier, and compatibility rows. An older
    bob sends no modifier, so there is no chip; an older Mac shows yellow and neutral
    seals with no wash.

Verify following `AGENTS.md`:

- `just portable-check`. If `swift` is missing here, run it on athena; read `tailnet.md`
  with `/sase_memory_read` for access.
- On the Mac, a scratch rsync copy (without `.build`/`.git`), then `bundle.sh` and
  `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
  `just install`.
- Never land Swift that has not been compiled. If the Mac is offline, say in the final
  report that the macOS targets are unverified.
- Format only changed lines.

Commit as `feat(mac-capture): seal project notes with @ and paint the project chip`.

## Phase mac_offer_card — the project successor prompt (bob-mac-capture)

The existing Pull-In card is the prompt. A new project's tasks arrive as ordinary
offers, so this phase is about harvesting them and setting the card's context.

```
┌ 📁⁺ New project · Finish the Google exit packet! ───────────────────────────┐
│ [📁⁺ New project] filter…                                       3 offers     │
│ 📁⁺ Project cash_goog_exit                                                   │
│   1  Draft the resignation memo                     → ADMIN · typed          │
│   2  Collect the equity paperwork   adds ^id        → ADMIN · typed          │
│   3  Book the exit interview        adds ^id        → DEEP (new) · created   │
│   1 already planned                                                          │
│ ⌃N ⌃P Move  space Mark  ⌘A All  esc Back  ⌃↩ Pull In & Open                  │
│                                             [Skip ⇧↩]   [Pull In 2 ↩]        │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Decode** `form` and `successors` on `CaptureProjectNoteSummary` (CaptureModels.swift
  ~2759-2805), tolerant when they are absent.
- **Harvest.** `makePullInPreviewSnapshot` (CapturePanelModel.swift ~11624-11669)
  collects `projectNote.successors` alongside `taskComplete` / `pomodoroClose` /
  `relationCompletes`. Return then opens the card through the existing `submit` gate
  (~2352-2370). The refusal path (~11416-11446) already works.
- **Card context** (`CapturePullInChoice.swift` / `CapturePullInChoiceView.swift`):
  - When every offer row's trigger is `project_created`:
    - the context line reads "New project · ‹^prj text›" (taken from `unblocked_by`);
    - the filter capsule reads "New project" with `folder.badge.plus`;
    - the group header glyph is `folder.badge.plus`;
    - the status line reads "Choose tasks to pull in".
  - Never show a `trigger: project_created` meta detail.
  - Mixed sets keep "Unblocked by …" and the per-row "from ‹cause›" suffixes.
  - Keys, Skip (create the note and link none), Escape (back to the unchanged draft),
    Control-Return, and "adds ^id" all work unchanged.

> [!decision] initial_marks = none The card opens with nothing marked, exactly like
> every other successor card.

> [!decision] initial_marks = all When every offer row is `project_created`, the card
> opens with all offers marked. Return pulls in every marked offer and Space unmarks.
> Mixed sets keep the normal unmarked start. Document this in `capture-previews.md` and
> test it.

- **Preview row.** The project-note section of the preview card gets the existing
  `pullInPreviewSummary` row (CapturePanelView.swift ~3568-3573, ~3655-3712): "Offers N
  tasks · ↩ choose what to pull in".
  - When `form == "offer"`, `successors` is empty, and there is no `link_guard`, the row
    reads "No open tasks to offer".
  - The blocked-project copy is unchanged.
- **Notifications.** Picks already appear in `task_links` ("Linked N tasks into NAME").
  Leave project-note successors out of `pulledInSummary` (NotificationService.swift
  ~832-840) so nothing is counted twice.
- **Fixtures and tests:**
  - Add `Scripts/generate-project-note-offer-fixtures.sh`, modelled on the existing
    generators. It uses a bob built from `offer_form`, a scratch vault, a parse, a dry
    run, a `--require-pull-in-choice` refusal, and a `-p` commit.
  - Replace `mac_seal`'s hand-authored offer-form replies with generated output and fix
    any drift.
  - Tests cover the harvest, the card copy and glyph, the decision's initial marks, the
    preview row, Skip, Escape, the notification, and decoding without the new fields.
- **Docs:**
  - `capture-previews.md` "Successor offers": offers now also come from `@route:id@`.
    Document the card copy and the preview row.
  - `keyboard.md` ~112-114.
  - `protocol.md`: `project_note.form` / `successors`, and the compatibility table row.
    With an older Mac and a new bob, the refusal path still opens the card, titled
    "Project ‹stem›".

Verify exactly as in `mac_seal`. Commit as
`feat(mac-capture): choose a new project's tasks in the successor card`.
