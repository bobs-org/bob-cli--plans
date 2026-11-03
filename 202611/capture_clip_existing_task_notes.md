---
tier: tale
title: "Clipboard notes under an existing task: @route+id %c"
goal:
  A note-less @route+id with %c (or --clip) files the clipboard as notes directly under
  that existing task, in bob capture, its dry-run preview, and capture-parse alike.
size: medium
decisions:
  relationships:
    ask:
      Should a note-less `@route+id %c` also accept `&`/`*`/`~` relationships, like the
      multi-note bullet form?
    choices:
      keep_refusal:
        "`@foo+bar %c &dep` keeps its `relationship_needs_text` refusal; that gate stays
        untouched"
      allow:
        Clip notes and relationships commit together; relaxes the shared gate in parse,
        editor, planner
    default: keep_refusal
    why:
      Bryan asked only for clip notes; the current refusal stays truthful and teaches
      the fix
    answer: keep_refusal
  decision_record:
    ask:
      Should the implementer add a `decisions:` memory strand recording the
      clipboard-notes capture shape?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.17i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 12:48:39
status: wip
---

# Clipboard notes under an existing task: `@route+id %c`

## Problem

In Bob Mac Capture, Bryan typed `@sase+restart-failed-athena %c` to paste the clipboard
as notes under the existing `^restart-failed-athena` task in `sase.md`. Both the preview
and Capture failed with:

```text
capture item 1 starting on line 1: task text is required; pass TEXT or pipe it on stdin
```

Reproduced against a scratch vault (`cash.md` with task `^goog-exit`):

| Draft                                  | Today                                                                            |
| -------------------------------------- | -------------------------------------------------------------------------------- |
| `@cash+goog-exit %c`                   | `task text is required`                                                          |
| `%c @cash+goog-exit`                   | `task text is required`                                                          |
| `@cash+goog-exit#requirements %c`      | `drawer_text`: "add the note text (`Postgres 17 minimum @foo+bar#requirements`)" |
| `@cash+goog-exit #fix %c`              | `type_marker_needs_task`                                                         |
| `bob capture --clip '@cash+goog-exit'` | `task toggle capture cannot be combined with --clip`                             |
| `@cash+goog-exit hello %c`             | works: `- hello` with the clip lines nested under it                             |

`bob capture-parse` already reports `@cash+goog-exit %c` as mode `sub_bullet` with
`needs: []` and no diagnostics. Only execution refuses it, so the editor and the commit
disagree.

**Root cause.** `resolve_sub_bullet_kind` in `src/native/capture_language/item.rs`
handles an empty parent body this way: with no other markers it becomes the Ensure Next
`TaskToggle`; with first-level bullets it becomes the multi-note form. Any other
empty-body case, including `%c` alone, falls through to `missing_text_error()`. The
glued `#drawer` branch refuses empty bodies with the `drawer_text` error.

## Goal

A note-less `@route+id` whose only extra is a clip directive (`%c` or `%c<N>`) files the
clipboard as notes directly under that existing task, exactly as if Bryan had typed each
clipboard line as a first-level note:

```text
@cash+goog-exit %c        (clipboard: "first line\nsecond line")
```

```markdown
- [ ] #task Finish Google Exit Packet [created::2026-07-31] ^goog-exit
  - existing note
  - first line
  - second line
```

Name this shape **clipboard notes** in code comments and docs.

## Behavior contract

1. **Shape.** An item is clipboard notes when all of these hold: the kind is `SubBullet`
   with a block-ID target (`@route+id`, leading or trailing); the parent body is empty;
   there are no authored child bullets; and a clip directive is present (`%c`, `%c<N>`,
   or `--clip`).
2. **Placement.** Clipboard lines are rendered as they are today (inline text, 2–10 flat
   lines, 1–10 flat list items, attachments, or the `file/clip-*.md` snippet link). They
   are written with no wrapper bullet, as direct children of the task. Plain clipboard
   notes land at the plain-child anchor: the end of the task block, before the earliest
   direct Schedule, Work, or Rotten Log. They use the task's child indentation and keep
   clipboard order. Counted `%c<N>` flattens every entry in order, as it does today.
3. **Drawers.** `@route+id#drawer %c` files every clipboard line inside that drawer. An
   existing drawer appends at its end; a missing drawer is created at the plain-child
   anchor (one header, then the lines). On a `^prj` target the selector names a `##`
   section, as it does for body-bearing notes. Reserved, ambiguous, near-duplicate, and
   empty-selector refusals are unchanged.
4. **Type marker.** A separate `#type` (`@route+id #fix %c`) retypes the target in the
   same transaction, exactly as a body-bearing `@route+id note #fix` does. There is no
   Ensure Next, no lane change, and no ledger edit.
5. **Parent untouched otherwise.** Like every sub-bullet capture, clipboard notes never
   change the task's status, Task Links, schedule, or freshness stamp. They never touch
   the daily note.
6. **`--clip`.** `bob capture --clip '@route+id'` (and with `#type`) behaves exactly
   like `@route+id %c`. Convert the parsed Ensure Next `TaskToggle` to the
   clipboard-notes `SubBullet` where the forced clip is injected in `plan_capture_item`
   (`src/native/capture/plan.rs`, the `parsed.clip.is_none() && request.forced_clip`
   block). The explicit `@route+id!` toggle still refuses `--clip` and `%c` with today's
   errors.
7. **Single-note markers.** `s:<N>` and `$<N>` decorate one note line, which this shape
   lacks. `@route+id %c s:2` and `@route+id %c $2` refuse with the existing
   `multi_note_marker` code and message (`multi_note_schedule_error` /
   `multi_note_priority_error`), not with `task text is required`. `@route+id s:2`
   without `%c` keeps today's missing text error.
8. **Unchanged refusals.** These keep today's errors: the bullets form plus `%c`
   (`@route+id %c` + `- a`, `multi_note_marker`); `@route+id! %c`
   (`unsupported_explicit_toggle`); `%n`; Pomodoro-link forms; `--task-ref` selections;
   and forced `-r/-t` with only `%c`, which still requires text. That last one is a
   non-goal; leave it alone.
9. **Relationships.**

   > [!decision] relationships = keep_refusal `@route+id %c &dep` (and `*`, `~`) keeps
   > the current `relationship_needs_text` refusal in both parse and execution, with no
   > message change.

   > [!decision] relationships = allow Exempt the clipboard-notes shape from the
   > empty-owner relationship gate, the same way `is_multi_note_candidate` is exempted.
   > That means both `refuse_relationship_needs_text` call sites in `item.rs`, the
   > editor diagnostic in `editor_parse.rs` (~line 4087), and
   > `relationship_only_existing_owner` in `plan.rs`, which must return `None` when a
   > clip is present so the clip is not silently dropped. The target stays the owner.
   > Add CLI tests for `&`, `*`, and `~` beside `%c`. Update RX9 and the
   > `relationship_needs_text` prose in docs/capture.md, plus the `capture-parse` help
   > sentence in `src/native/capture_parse.rs` that lists `%c` as an extra.

10. **Dry run.** `--dry-run` with the clipboard read previews exactly the lines that
    would be written. This is what Bob Mac Capture runs, because `capture-parse` already
    reports the `%c` directive span. With `--dry-run --no-clip`, report
    `clip: {"status": "not_read", "count": N}`, still validate the parent lookup
    (missing ID, duplicate ID, non-task errors) and the drawer selector, stage no bytes,
    and exit 0. Never insert an empty line, and never create a drawer header in this
    case. A real capture with `--no-clip` refuses with `clip_not_read`, as today.
11. **Atomicity.** The plan's existing batch rules apply. A later failing item rolls
    back the note write, and newly created clip files are removed.

## Output contract

- **JSON** (`bob capture -f json`): `kind: "sub_bullet"`, `text: ""`, and
  `task_line: ""` (the multi-note form already uses empty strings). Also report the
  usual `block_id`, `parent_line`, `parent_text`, `parent_status_*`, `drawer`
  (`{title, key, created}` or `null`), and the `clip` object. With no note line,
  `clip.lines` sits at the task's child level with no leading indent unit
  (`"- first line"`), while a body-bearing note keeps today's `"\t- first line"`.
  `task_blocks` shows the added lines under the parent, as it does for other sub-bullet
  captures. No `notes[]`.
- **Human**: keep the `under [ ] <parent>  ^id` header, with the `· DRAWER` /
  `new drawer` suffix when relevant. Skip the dim `task_line` row when it is empty; the
  existing clip-lines loop in `src/native/capture/output.rs` then prints the notes. File
  confirmations for attachments and snippets are unchanged.
- **No grammar bump.** `capture-parse` output for `@route+id %c` is unchanged, and the
  editor changes below only drop false needs and diagnostics on drafts that now commit.
  That relaxation is additive, so `GRAMMAR_REVISION` stays put.

## Implementation

### 1. Grammar (`src/native/capture_language/item.rs`)

- In `resolve_sub_bullet_kind`, add the clipboard-notes case. If the parent body is
  empty, there are no sub-bullets, and `clip_present` is true:
  - with `s:`/`$` also present, return `multi_note_marker_error()` (rule 7);
  - otherwise return `SubBullet { target, section }`. This covers both the plain branch
    (a `BlockId` target stays `SubBullet`, not `TaskToggle`) and the glued `#drawer`
    branch, which today returns the `drawer_text` error.
- Make sure the item-level code paths propagate the `multi_note_marker` code for the
  rule-7 refusal. The multi-note form already maps these strings to that code somewhere
  in `parse_capture_item`'s error mapping or in `plan.rs`; reuse that mapping, don't
  duplicate it.
- Confirm that `type_marker_only_error` / `directive_only_error` no longer fire for this
  shape (they only run on `missing_text_error`). Confirm that
  `is_existing_type_marker_subject` already accepts `SubBullet` (it does), so `#type`
  flows through to the existing sub-bullet retype in `plan.rs`.
- Keep `%c`-only on new tasks (`%c` alone, `%n %c`) refusing with missing text; that is
  out of scope.

### 2. Planner (`src/native/capture/plan.rs`, `src/native/capture/sub_bullet.rs`)

- Add a small predicate, e.g.
  `is_clip_note_capture(parsed: &ParsedCaptureText) -> bool`: a `SubBullet` kind, empty
  body, no sub-bullets, and `parsed.clip.is_some()`. Evaluate it after forced-clip
  injection.
- For that shape, plan the clipboard with an empty indent (pass `""` instead of
  `child_indent` to `plan_with_reservations` / `plan_history_with_reservations`). Build
  `capture_block` as the clip lines joined by `\n`, with no capture line and no schedule
  log; do not call `assemble_capture_block` with an empty first line. Hand it to the
  existing `plan_sub_bullet_capture` call, which prefixes the task's child indentation.
  Snippet and attachment `files` keep flowing through `clip_plan`, so persistence and
  rollback are unchanged.
- For `not_read` (dry run with `--no-clip`), skip staging. Factor the parent lookup at
  the top of `plan_sub_bullet_capture` (the `BlockIdLookup` / `RefLookup` match and its
  error messages) into a helper that both paths share. Build `SubBulletCaptureDetails`
  from it. For a drawer selector, resolve read-only with
  `capture_drawers::resolve_drawer_for_write` without staging the header, and report
  `drawer.created` from the outcome.
- At forced-clip injection, convert `TaskToggle { intent: EnsureNext, .. }` to
  `SubBullet { target: SubBulletTarget::BlockId(block_id), section: None }` (rule 6).
  Leave `TaskToggleIntent::Toggle` alone so `reject_task_toggle_conflicts` still refuses
  it.
- Output (`src/native/capture/output.rs`): skip printing an empty `task_line` row for
  sub-bullet results.

### 3. Editor parity (`src/native/capture_language/editor_parse.rs`)

- Drawer needs (~line 3133): when the glued-drawer `SubBullet` has an empty body and a
  clip directive (`seen.clip`), drop `Need::DrawerText`, the same way first-level
  bullets already do.
- Type marker (~line 1745): extend the `relationship_sub_bullet` exemption. When the
  mode is `SubBullet`, the body is empty, and the notes come from first-level bullets
  **or** a clip directive, the marker retypes the target instead of reporting
  `type_marker_needs_task`. The bullets half fixes an existing disagreement found while
  planning: `capture-parse` flags `@cash+goog-exit #fix` + `- a` while `bob capture`
  accepts it and retypes. Keep it, since it is the same predicate.
- Do not add a `multi_note_marker` editor diagnostic here. That pre-existing gap is
  tracked separately as bead `bob-cli-gk`.

### 4. Help and docs

- `src/native/capture/cli.rs` (the `%c` paragraph near "`%c` composes with s:<N>,
  $<N>, #<type>, and every route kind"): add one sentence, e.g. "On a note-less
  `@route+id` (or `@route+id#drawer`), `%c` files the clipboard lines as notes directly
  under that task (or drawer)." Then run `just docs-gen` to regenerate
  `docs/reference/cli/capture.md` and the help goldens.
- `docs/capture.md`:
  - _Clipboard_ section: add the note-less `@route+id` / `@route+id#drawer` composition
    with a short before/after example. This is where rule 2's "no wrapper bullet" lives.
  - _Sub-bullets under existing tasks_: add a short `#### Clipboard notes` subsection
    after _Several notes at once_. Cover rules 1–4, 6, 7, and 10 and the JSON shape, and
    cross-link _Clipboard_. In _Several notes at once_, keep "Item-wide `s:<N>`, `$<N>`,
    `%c`, and `--clip` are refused on this form" (still true for the bullets form) and
    point to the new subsection.
  - _Task Link toggle_: the rejection list names `%c` with
    `task toggle capture cannot be combined with %c` and `--clip` with its message.
    Reword it so the marker-only form with `%c`/`--clip` files clipboard notes, while
    `@route+block-id!` still refuses them.
- `docs/reference/capture-syntax.md`: in the `@route+block-id` section, add a one-line
  clipboard-notes example. In the `#` table, change the glued-drawer row "requires note
  text or child bullets" to also name `%c`. Do not add new token-table rows. A new row
  needs a `search_smoke.yml` literal query (see bead `bob-cli-fo`), and none is needed
  here.
- `docs/reference/json-outputs.md` (`bob capture` row): add a clause saying a
  clipboard-notes item reports empty `text`/`task_line` and its notes in `clip.lines` at
  the task's child level.
- Do not edit `CHANGELOG.md` or the `Cargo.toml` version; the release train owns them.
  Use a `feat(capture): …` commit subject; this is additive, so no `!`.

> [!decision] decision_record Use `/sase_memory_write` to add one accepted strand to the
> `decisions` memory web, e.g. `decisions:clip-notes-under-existing-task`. Claim: a
> note-less `@route+id` (or `@route+id#drawer`) whose only extra is `%c` / `--clip`
> files the clipboard lines as notes directly under that task (or drawer); `s:`/`$`
> refuse and the parent never changes lane. Why: Bryan's request (cite this plan and the
> feature commit). Rejected alternatives: a wrapper note bullet, and Ensure Next plus
> clip. Link it to `[[decisions:capture-bullets-mirror-the-task-block]]` and
> `[[decisions:capture-sigils-say-where-what-how]]`, then run `sase memory init`. When
> this decision is off, make no memory change.

## Tests

Unit tests, in `src/native/capture/tests/grammar.rs` beside
`bare_sub_bullet_markers_toggle_instead_of_erroring`:

- `@cash+id %c`, `%c @cash+id`, and `@cash+id %c3` parse as
  `SubBullet { BlockId("id"), None }` with an empty body and the right `ClipRequest`.
- `@cash+id#requirements %c` parses as `SubBullet` with the drawer section.
- `@cash+id %c s:2` and `@cash+id %c $2` fail with the `multi_note_marker` messages.
- Still true: `@cash+id` is `TaskToggle(EnsureNext)`, `@cash+id s:2` needs text,
  `@cash+id! %c` refuses as an explicit toggle, and `@cash+id %c` + `- a` refuses
  `multi_note_marker`.

Editor tests (the `capture_language/tests/` editor suites or the `capture_parse.rs`
tests):

- `@cash+id#requirements %c` has `needs: []`.
- `@cash+id #fix %c` and `@cash+id #fix` + `- a` report no `type_marker_needs_task`
  diagnostic; mode stays `sub_bullet`.

CLI tests, in a new `tests/cli/capture/clip_notes.rs` registered in
`tests/cli/capture/mod.rs`. Use `BOB_CLIPBOARD_CMD` / `BOB_CLIPBOARD_HISTORY_CMD` and
`BOB_NOW`, like `tests/cli/capture/clip.rs`, and pin exact note bytes:

- One-line clip, both edges: one direct child under the task.
- Two-line clip: two direct children, in order, inserted before an existing direct
  `🗓️ **SCHEDULE LOG**`.
- Two-space indented note: the lines use two spaces.
- `%c2` history: both entries flattened in order.
- Attachment path: the file is copied under `img/` or `file/`, and its link line is a
  direct child.
- Snippet fallback: `file/clip-*.md` is created and linked.
- `#requirements %c` into an existing drawer and into a missing drawer (created with the
  lines). A `^prj` target with `#section`.
- `#fix %c`: the task gains `[type::]` and the clip notes, its status checkbox is
  unchanged, and the daily note is untouched (create one and compare bytes).
- `--clip '@cash+id'` matches `@cash+id %c` byte for byte. `--clip '@cash+id!'` still
  refuses.
- JSON: `kind`, empty `text`/`task_line`, unindented `clip.lines`, `drawer`, and the
  added `task_blocks` lines.
- Human output: no blank row; the clip lines are printed under the header.
- `--dry-run --no-clip`: exit 0, `clip.status: not_read`, bytes unchanged, no drawer
  header created. Missing block ID: the same error as body-bearing capture. Real
  `--no-clip`: `clip_not_read`.
- Batch atomicity: `@cash+id %c` (snippet-sized clip), a blank line, then a failing
  item. The note bytes are unchanged and no `file/clip-*.md` file remains.
- The relationships decision: `@cash+id %c &cash:other` keeps `relationship_needs_text`
  (keep_refusal), or files both (allow).

## Verification

- `just check` (fmt, clippy, all tests) passes.
- `just docs-gen`, then `just docs-check` passes.
- Smoke-test against a scratch vault with an installed debug build. Set
  `BOB_CLIPBOARD_CMD` to a script that prints two lines, then run
  `bob capture -b <scratch> '@cash+goog-exit %c'` and `--dry-run -f json`. Confirm the
  bytes and JSON above.
- No bob-mac-capture change is expected. The app already reads the `%c` directive span
  from `capture-parse` (so its preview reads the clipboard), and it already tolerates
  empty `text`/`task_line` from the multi-note form. After the next release ships, Bryan
  can retry `@sase+restart-failed-athena %c` in the panel.

## Coordination

The in-progress epic `bob-cli-gj` (note-pull for a lone `@route`) also edits capture
grammar code under `src/native/capture_language/` and `docs/capture.md`. Rebase onto
current master before starting, and keep the edits to `resolve_sub_bullet_kind` and the
sub-bullet planner local.
