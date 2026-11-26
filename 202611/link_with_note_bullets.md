---
tier: tale
title: Link an existing task and file its note bullets in one @route:id capture
goal:
  A note-less @route:id or ^route:id link followed by bullets links the task and files
  the bullets as its notes in one capture.
size: medium
decisions:
  guarded_notes:
    ask:
      When the link guard leaves a Project-blocked target unlinked, should the item's
      note bullets still land?
    choices:
      land: Notes land; output says "not linked" and then lists the notes
      refuse: The item fails before any write; notes land only with the link
    default: land
    why:
      Notes don't depend on the link, and a guarded link is already a reported success
    answer: land
  close_notes:
    ask:
      Should link-close forms (`@route:id=x…`, `^route:id=x…`) also take child bullets
      as task notes?
    choices:
      refuse:
        Close forms keep refusing child bullets; only non-close link forms take notes
      notes:
        Close forms file bullets as notes before closing, unlike whole-item `x` Work Log
        bullets
    default: refuse
    why:
      Under whole-item `x`, child bullets are Work Log entries, so link-close notes
      would read ambiguously
    answer: refuse
  memory_link_notes_record:
    ask:
      Should the implementer add a link-notes decision strand and mark
      capture-bullets-mirror-the-task-block superseded?
    memory:
      - decisions:capture-bullets-mirror-the-task-block
      - decisions:existing-task-links-carry-notes
    default: true
    corrects:
      That position is a new task, a new project note, or a note-less `@route+id`
    answer: true
proposed_by: bbugyi200.apollo.g4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 12:12:54
status: wip
---

# Link an existing task and file its note bullets in one `@route:id` capture

Bob task: `^sub-bullet-and-today` in `~/bob/bob.md` ("Add support for `@foo:bar` syntax
with sub-bullet!").

## Problem

Bryan typed this into Bob Mac Capture:

```text
@bob:h2s-to-area-prj-card
- foo
```

He wanted one gesture that does two things: link the existing task into today's Pomodoro
ledger, and add `foo` as a note under that task. Instead, capture refused the whole
item:

```text
capture item 1 starting on line 1: Pomodoro link capture cannot be combined with authored child bullets
```

His workaround was two items, which shows exactly what he expected:

```text
@bob:h2s-to-area-prj-card

@bob+h2s-to-area-prj-card
- foo
```

That preview made `^h2s-to-area-prj-card` Next, linked it under a new `BUILD` Pomodoro,
and inserted `- foo` as the task's child. He should not have to name the task twice.

A same-line note can't do this job. `text @route:id` already means "create a new task
with this ID", so child bullets are the only way to attach notes to an existing-task
link.

## Target behavior

A note-less existing-task link item followed by authored child bullets links the task
exactly as the solo marker does today. It then files the bullets under that task exactly
as a note-less `@route+id` with the same bullets does today (the multi-note form). It is
one item, one transaction, and one `pomodoro_link` result.

This applies to every existing-task link form without a close:

- `@route:id` and the `^route:id` active-task spelling;
- named links: `@route:id#name`, `@route:id#"NAME"`;
- link starts: `@route:id=`, `@route:id=<X>`, `@route:id=name`, `@route:id#name=<X>`,
  and their `^` twins;
- a free `#type` marker on the parent line (`@route:id #fix`), which retypes, links, and
  files the notes in one transaction.

The bullets get the full multi-note grammar from
`decisions:capture-bullets-mirror-the-task-block`, `decisions:drawers-keep-their-names`,
and `decisions:task-reqs-are-met-with-their-task`:

- every plain first-level bullet, with its nested bullets, is one note, filed in draft
  order through the task block layout (the loose note slot before managed logs, or the
  `NOTES` drawer when the task owns reqs);
- `- **Title**` with nested bullets, and the `- #drawer` / `- #"TITLE"` typing shortcut,
  file into (or create) that drawer;
- `- [ ] text` is a requirement (req) under the task;
- a `^prj` target routes notes through its project-note sections, as `@route+id` does
  today.

Order inside the item: the link runs first. That covers the optional retype, Ensure
Next, the §13.5.1 own-prerequisite drop, the pull-forward Schedule Log, the Pomodoro
link, and the optional session start. The notes are then planned against the staged
post-link text, and the snooze `finish_link_now` carry runs last, as it does today. Any
refusal from either half rolls back the whole batch.

Concrete example:

```text
@bob:h2s-to-area-prj-card
- foo
- **Future Work**
  - try the card
```

Before (`bob.md`, the task is Ready and nothing is linked):

```markdown
- [ ] #task Notebooks/albums/... cards (not H2 sections)! [created::2026-11-23]
      ^h2s-to-area-prj-card
  - 🧱 **BLOCKS:** [[#^equals-under-key]]
```

After (`bob.md`; today's daily note gains `- [[bob#^h2s-to-area-prj-card]]` under the
Pomodoro that the task's type places, exactly as the solo link does today):

```markdown
- [*] #task Notebooks/albums/... cards (not H2 sections)! [created::2026-11-23]
  ^h2s-to-area-prj-card
  - 🧱 **BLOCKS:** [[#^equals-under-key]]
  - foo
  - **Future Work**
    - try the card
```

### What stays refused (unchanged messages)

- A forced destination (`--route`, `--section`, `--task`, `--task-ref`, `--drawer`,
  `--clip`) with a link still refuses.
- `%c` / `--clip`, `s:<N>`, and `$<N>` on a link item keep their existing
  `Pomodoro link capture cannot be combined with …` messages. They do not switch to the
  multi-note `multi_note_marker` wording, so the first failure stays the link's own.
- A snooze `@route:id z:<when>` with children keeps `snooze_existing_children_error`.
- Relationship modifiers (`&`, `*`, `~`) with children keep the
  `relationship_needs_text` refusal, as RX10 does for `@route+id`.
- Close forms: see the decision below.
- A same-line note on a link (`@route:id text`) keeps its new-task meaning.
- The authored-child rules that already apply to `@route+id` (empty drawer bullets,
  `drawer_bullet_text`, req placement and depth refusals) apply here with the same codes
  and messages.

> [!decision] close_notes = refuse `@route:id=x…` and `^route:id=x…` with child bullets
> keep today's refusal, and `docs/capture.md`'s "link-form closes keep their current
> behavior" sentence stays true. Only the non-close link forms change.

> [!decision] close_notes = notes `plan_pomodoro_close_link_item` files the bullets as
> task notes through the same helper, **before** the link-close, so the closed task
> carries its notes. Whole-item `x` keeps child bullets as Work Log entries. Update the
> docs sentence that says link closes reject children, and add CLI tests for `@r:id=x`
> and `^r:id=x!1` with notes.

> [!decision] guarded_notes = land When the §13.5 link guard leaves the target unlinked
> (project-blocked or `^prj` reasons), the result keeps today's `link_guard` kind and
> "not linked" output, and the notes still land and are reported under it.

> [!decision] guarded_notes = refuse When the link guard would leave the target unlinked
> and the item carries bullets, the item fails before staging with a usage error. Name
> the guard reason and teach the `@route+id` spelling for notes alone.

## Governing decisions

- `decisions:capture-bullets-mirror-the-task-block`: the draft reads like the block it
  writes, and one resolver serves the menu, `capture-parse`, the dry-run preview, and
  the commit. This plan adds the note-less existing-task link as one more position where
  first-level bullets are the task's children.
- `decisions:capture-sigils-say-where-what-how`: `:` (link) and `+` (target) keep their
  meanings; the plan adds no new sigil.
- `decisions:capture-link-drops-blocking-deps`: the link half keeps the dependency drop
  and the link guard unchanged.
- `decisions:task-reqs-are-met-with-their-task` and
  `decisions:drawers-keep-their-names`: the bullet grammar is reused, not forked.
- `decisions:today-is-read-from-the-ledger`: "today" still means only the Task Link
  under today's open Pomodoro; the notes write no lane or field.
- `decisions:mac-capture-is-a-thin-client`: this is bob-cli only. Bob Mac Capture
  renders bob's preview as-is. Its per-file post-image card already shows the inserted
  bullets (it did in the two-item workaround). The new JSON is additive, and the app
  gates `CaptureGrammarGate` on a _minimum_ grammar revision, so a bump is safe. No Mac
  change is needed.
- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` version or
  `CHANGELOG.md` edits. This turns a refusal into acceptance, which is an additive
  change, so commits use `feat(capture):` without `!`.

## Implementation

### 1. Execution parse (`src/native/capture_language/`)

- **`item.rs`, the existing-task link aggregate** (around the
  `parent_is_pomodoro_candidate && parent_body_is_empty` block, ~L2920–3070):
  - Stop refusing non-empty `sub_bullets` in the `!has_dependencies` branch, for
    non-close links only (and for close links too under `close_notes = notes`).
  - Keep the clip, schedule, and priority refusals.
  - Before building the `PomodoroLink` outcome, run the same finishing passes the
    note-less `SubBullet` path runs on its bullets: `finish_execution_drawer_bullets`
    and `apply_execution_reqs`.
  - Pass the finished `sub_bullets` through in `ParsedCaptureText` instead of
    `Vec::new()`.
  - The snooze and relationship gates above it stay first and unchanged.
- **`tokens.rs`, the `^route:id` path** (`parse_pomodoro_link_item`, ~L2879–3200, plus
  the shape check near L2868 that returns `POMODORO_LINK_CHILD_CONFLICT_ERROR`): the
  caret spelling must produce the same `ParsedCaptureText` as `@route:id` for the same
  bullets. Prefer deferring a non-solo caret item with only child lines to the shared
  `item.rs` aggregate (if that path already claims `^route:id`), or else factor out the
  child-bullet parse so both call it. Do not duplicate the child grammar.
- **The kind-keyed mappings that must agree with their editor twins**: extend each so a
  note-less, non-close `CaptureKind::PomodoroLink` with first-level bullets counts as an
  existing-task, under-task host.
  - `drawer_bullet.rs`: `is_multi_note_sub_bullet`, or a sibling predicate used next to
    it, so that `finish_execution_drawer_bullets` treats drawer bullets as under-task.
    Also `drawer_bullet_position_error` wording, if it names the kind.
  - `req_bullets.rs`: `ReqHost::from_kind` maps it to `ReqHost::ExistingTask`.
- **`GRAMMAR_REVISION` 33 → 34** in `mod.rs`, with a doc-comment line: "Revision 34
  admits authored child bullets under a note-less existing-task link (`@route:id`,
  `^route:id`, named and start forms) as notes, drawers, and requirements of that task;
  previously `invalid_pomodoro_link`." Update every test or fixture that pins 33.

### 2. Editor parse and completion (`capture-parse`, `capture-complete`)

- **`editor_parse.rs` ~L5810**: drop the
  `!sub_bullets.is_empty() → POMODORO_LINK_CHILD_CONFLICT_ERROR` arm for non-close links
  (and for close links under `close_notes = notes`). Keep the schedule, priority, and
  clip arms and their order. Mode stays `pomodoro_link`.
- **Editor twins of the mappings above**: each must agree with its execution twin.
  - `is_under_task_mode` and the `under_task` computation at `editor_parse.rs` ~L3722:
    add `EditorMode::PomodoroLink` with an empty parent body and first-level bullets.
  - `ReqHost::from_mode` → `ExistingTask`.
  - The `DrawerBulletTarget` mapping in `drawer_bullet.rs` ~L205: an `ExistingTask`
    target built from the link's route and block ID, so the bullet drawer menu (`- #`)
    offers the target's drawers and catalog entries.
- Child lines must emit the same spans they emit under a note-less `@route+id`:
  `drawer_marker`, `drawer_bullet`, and `req_marker` spans, plus the drawer and req
  diagnostics. `capture-parse` must show no diagnostic exactly when `bob capture` would
  succeed, and the same first failure when it would not.
- After the change, delete `POMODORO_LINK_CHILD_CONFLICT_ERROR` and the duplicated
  string literals if nothing still uses them. If close forms keep refusing, keep one
  shared constant for that case instead of the three inline copies.

### 3. Executor (`src/native/capture/`)

- **`pomodoro_link.rs::reject_pomodoro_link_conflicts`**: drop the `sub_bullets` refusal
  for non-close links. The `--clip`, forced-flag, `%c`, `s:`, and `$` refusals stay.
- **Shared note-filing helper**: extract the note-filing body of
  `plan.rs::plan_multi_note_sub_bullet_item` (~L6125–6460) into one helper that both
  callers use.
  - The extracted body covers the req-group insert through `plan_insert_reqs_capture`,
    then the glued-drawer, drawer-bullet, and plain-note loops through
    `plan_sub_bullet_capture`.
  - The helper takes the planner, bob dir, target path, route, a `SubBulletTarget`, an
    optional glued drawer, the authored bullets, and the child indent.
  - It returns the notes (`SubBulletNoteJson`), first-touch `drawers`, the requirement
    write reports, and first-parent details.
  - `plan_multi_note_sub_bullet_item` keeps its own marker refusals, retype, and
    dependency and relation handling around the call.
  - The refactor must leave every existing multi-note output byte-identical.
- **`plan.rs` `CaptureKind::PomodoroLink` branch** (~L3386): after
  `plan_existing_task_pomodoro_link_item` and the existing post-link steps (marker
  retype and the named-destination theme merge), call the helper with
  `SubBulletTarget::BlockId(block_id)` when `parsed.sub_bullets` is non-empty. Then fold
  its output into the planned item:
  - set `result.notes` and `result.drawers`;
  - append requirement writes wherever the multi-note item records them, so
    `print_req_card` shows them;
  - merge the helper's task and Pomodoro block refs into `task_block_refs` /
    `pomodoro_refs`, so later items in the same batch track the shifted lines.
  - This must hold on every return path of the branch, including the early return after
    `apply_existing_marker_after_gesture`. Restructure that branch so the notes step
    runs once, at the end.
- **Link guard**: under `guarded_notes = land`, run the same notes step when the planner
  returns the `link_guard` outcome. Under `refuse`, detect the guard before staging and
  return the usage error.
- **Snooze carry**: `plan_capture_item` already runs `snooze::finish_link_now` after
  `plan_capture_item_inner`. Confirm that a snoozed target with bullets carries its
  snooze-row notes and also files the new bullets, with no duplicates.
- `batch.rs`, `relations_merge.rs`, `type_marker.rs`, and `draft.rs` match on
  `PomodoroLink`. Check that none assumes an empty `sub_bullets` for that kind.

### 4. Output

- **JSON**: the result keeps `kind: "pomodoro_link"` (or `link_guard`). It adds the
  existing optional `notes` and `drawers` arrays, populated as the multi-note
  `sub_bullet` item populates them. Both are empty and omitted when the item has no
  bullets, so solo-link JSON stays byte-identical.
- **Human**: in `output.rs`, after `print_human_pomodoro_link_success` (and after the
  `link_guard` "not linked" block), call `print_human_multi_note_notes` when
  `result.notes` is non-empty. The req card already follows. Solo-link human output must
  not change.

### 5. Docs (read `sase/memory/docs.md` with `/sase_memory_read` first)

`docs/capture.md`:

- **Grammar-at-a-glance table**: the
  `@route:block-id[#pomodoro][=<X>] with no other text` and `^route:block-id…` rows gain
  "or with only note bullets below, which land under that task as notes (the multi-note
  form)". Add an example row.
- **"Linking and starting existing tasks"**: replace "no authored child bullets" with
  the new rule. Spell out the order (link first, then notes), which refusals stay, and
  the `guarded_notes` and `close_notes` outcomes. Add a worked example like the one
  above.
- **"Sub-bullets under existing tasks"** (~L7383): add one cross-reference sentence
  saying `@route:id` followed by bullets also links the task.
- **`capture-parse` section**: grammar 34 (`grammar: 34`, plus a "Grammar 34 admits …"
  paragraph). Update the other places that state the current revision, near ~L7650 and
  ~L10740.
- If `docs/task-drawers.md` or `docs/task-requirements.md` list the capture positions
  that accept drawer or req bullets, add the link form there too.
- Run `just docs-check` (and `just docs-gen` if generated pages change).

### 6. Memory (only as the memory decisions allow; use `/sase_memory_write`)

- `memory_link_notes_record = true`: add
  `sase/memory/decisions/existing-task-links-carry-notes.md` (keyword "Existing-Task
  Links Carry Note Bullets"). Follow the web's record shape:
  - **Applies to:** bob-cli and Bob Mac Capture.
  - **Claim:** a note-less existing-task link (`@route:id` / `^route:id`, named and
    start forms; not close forms unless `close_notes = notes`) followed by bullets links
    first, then files each first-level bullet exactly as note-less `@route+id`.
  - **Why:** Bryan's `^sub-bullet-and-today` task. He hit the refusal and had to write
    the task twice. The link-only and note-only spellings stay intact.
  - **Rejected:** a same-line note (it already means a new task) and a new sigil.
  - **Cost:** grammar 34, and capture `notes` on `pomodoro_link` results.
  - **Reopens when:** close-form or snooze-form notes are wanted.
  - **Evidence:** this plan, the bob.md task, and the implementing commit.
- Under the same decision, on
  `sase/memory/decisions/capture-bullets-mirror-the-task-block.md`, append the new
  strand to `metadata.superseded_by` (status stays `superseded-in-part`). Add one
  back-link sentence: "Superseded in part: a note-less existing-task link `@route:id`
  followed by bullets is also an under-task position — see
  [[decisions:existing-task-links-carry-notes]]." Do not touch the accepted body
  otherwise.
- Run `sase memory init` afterwards.

## Tests

Add CLI tests, for example in a new `tests/cli/capture/task_link_notes.rs`, registered
in `tests/cli/capture/mod.rs` or wherever siblings are listed. Use the fixtures and
helpers `tests/cli/capture/task_link.rs` and `note_bullets.rs` already use. Assert exact
file bytes after commit, the JSON shape, human output, and dry-run byte-for-byte
no-writes.

1. `@r:id` + `- foo`: a Ready task becomes Next, its Task Link lands where the solo link
   would put it, `- foo` is the task's child before managed logs, JSON has
   `kind: pomodoro_link` and `notes[0].text == "foo"`, and human output lists the note.
2. Several first-level bullets with nested children land in draft order. A task that
   owns reqs puts its notes in `NOTES`.
3. Drawer bullets: `- **Future Work**` with a nested note merges into an existing
   drawer, and `- #future-work` resolves to the same drawer. A new drawer is reported
   with `created: true`.
4. `- [ ] req` under the link writes a requirement and prints the req card.
5. `^r:id` + bullets: identical file bytes and JSON (apart from spelling) to `@r:id`.
6. `@r:id#build` + bullet, and `@r:id=` + bullet: named destination, and a started
   session, both with the note.
7. `@r:id #fix` + bullet: retype, link, and note in one capture.
8. A target Blocked only by its own prerequisites: prerequisites dropped, task linked,
   note filed.
9. A project-blocked target, per `guarded_notes`.
10. A task already linked under an open Pomodoro: nothing moves and the note lands.
11. A snoozed target: snooze notes carried, new note filed, no duplicates.
12. A ledger or target refusal (missing ID, Done task, duplicate ID) writes nothing.
13. A batch with the link-and-notes item followed by another item editing the same note:
    both land at the right lines.
14. Unchanged refusals, each with its existing message: `%c`, `s:2`, `$1`, a forced
    `--route`, `z:19`, `&p:q`, and the close form (per `close_notes`). All are asserted
    with bullets present.

Add parse and completion tests:

- Flip `src/native/capture_language/tests/editor_spans.rs` (~L609, ~L816) to expect no
  diagnostic, with mode `pomodoro_link` and the child spans.
- Add `capture-parse` cases in `tests/cli/capture/parse_pomodoro.rs` (or a sibling) for
  plain, drawer, and req bullets under both spellings.
- Add parity cases showing that `capture-parse` reports no diagnostic exactly where
  `bob capture --dry-run` succeeds, and the same first error where it fails (the refusal
  list above).
- Add a `capture-complete` case in which `@r:id` followed by `- #` offers the target's
  drawers.
- Add a unit test that the `from_kind` / `from_mode` and under-task predicates agree for
  the link kind.

Run, and make green before finishing:

```bash
just fmt
just lint
just test          # or: cargo test --test cli capture:: then the full suite
just docs-check
```

## Out of scope

- Same-line notes on a link, clipboard (`%c`) notes on a link, and snooze-plus-notes.
- Link-close notes, unless `close_notes = notes`.
- Any Bob Mac Capture change. If Bryan later wants the Mac link header to say "· N
  notes", that is a separate Mac task.
- `bob-plugins` gestures.
