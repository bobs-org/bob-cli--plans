---
tier: epic
title: Several @file+id targets on one capture line
goal: "One capture line may name several existing tasks with adjacent `@route+block-id`
  markers. Its note text, the bullets below it, and every operation on the line
  (clipboard notes, `s:`/`$`, `#type`, `&`/`*`/`~`, Ensure Next, `!` toggle, drawers)
  then apply to each named task in one atomic capture. `bob capture`, `capture-parse`,
  `capture-complete`, and `capture-rewrite` agree on the grammar, and Bob Mac Capture
  keeps working unchanged.

  "
decisions:
  split_edges:
    ask:
      Refuse a line whose @file+id markers sit at both edges, instead of keeping the
      trailing one as note text?
    choices:
      refuse:
        "`@a+x note @b+y` fails with a teaching error to put every target in one run"
      literal: Today's leading-wins rule stays; `@b+y` is written into the note under ^x
    default: refuse
    why:
      Someone who names targets at both edges means all of them; silently misfiling is
      worse
    answer: refuse
  memory_bulk_capture:
    ask:
      Teach the Bulk Capture glossary strand that one line's @file+id run fans out into
      one item per task?
    memory:
      - glossary:bulk-capture
    default: false
    answer: false
phases:
  - id: grammar
    title: Target-run grammar in both resolvers, fan-out, and grammar bump
    depends_on: []
    size: large
    description: "grammar: recognize a leading or trailing run of `@route+id` markers in
      the execution resolver and the capture-parse resolver, fan each run out into one
      parsed item per target, report `targets[]` and the new refusals in capture-parse,
      bump the grammar revision, and document the grammar contract.

      "
  - id: execute
    title: Planner, output, and capture contract for fan-out groups
    depends_on:
      - grammar
    size: medium
    description: "execute: share one clipboard read and one `$N` roll per fan-out group,
      refuse targets that resolve to the same task, name the failing target in errors,
      dedupe group warnings, add the `fanout` JSON context and human header, cover every
      operation with CLI tests, and finish the capture docs.

      "
  - id: assist
    title: Completion, rewrite, and @@ absorption inside target runs
    depends_on:
      - grammar
    size: medium
    description:
      "assist: make capture-complete and capture-rewrite treat every member of a target
      run (and the `@` token extending one) as a marker position, give a run the
      non-absorbable `@@` notice, and update the editor-command docs and help."
proposed_by: bbugyi200.athena.17k
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 13:55:31
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_multi_target.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_multi_target.md)

# Plan: Several `@file+id` targets on one capture line

## Why

Today `@work+prep Called Morgan` files a note under one existing task. To file the same
note (or the same bullets, clipboard, retype, prerequisite, or Ensure Next) under two
tasks, Bryan has to write the item twice. Typing a second marker does not help either:
only the first leading token (or the last trailing token) routes, and
`bob capture '@work+prep @dev+parser Called Morgan'` writes the note
`@dev+parser Called Morgan` under `^prep`. `bob capture-parse` reports exactly that
body.

After this epic, **adjacent `@route+block-id` markers name several tasks, and the line
applies to each of them**:

```text
@work+prep @dev+parser Called Morgan Stanley today.
```

writes `- Called Morgan Stanley today.` under both `work.md ^prep` and `dev.md ^parser`.

```text
@work+prep @dev+parser
- Ping Alice about the cutover
- REQUIREMENTS
  - Postgres 17 minimum
```

files both notes under each task, merging into or creating each task's `REQUIREMENTS`
drawer. `@work+prep @work+outline #fix` retypes both tasks to FIX and ensures both are
Next.

## Shared design contract

Every phase implements against this section. Where this plan and the code disagree,
follow this plan and record the discrepancy as a `PROPOSED FOLLOW-UP:` note on your
phase bead.

### Vocabulary

- **Target marker**: one complete or partial `@route+block-id` token. This is any token
  `is_sub_bullet_marker_candidate` accepts (`src/native/capture_language/tokens.rs`),
  with an optional glued `#drawer` or a terminal `!`. `@route:id`, `@route^id`,
  `@route`, `@route#Section`, and `^route:id` are never target markers.
- **Target run**: two or more target markers that are adjacent on one physical line,
  after the line's `&`/`*`/`~` modifiers, separate `#type` marker, `%` directives, and
  terminal `s:<N>`/`$<N>` markers have been stripped. Those are stripped before route
  selection today, so they may sit before, between, or after the run's members.
- **Fan-out group**: the N parsed capture items that one target run produces, one per
  member, in typed order.

### Where a run is recognized

A run fills the same single **route/mode slot** that one marker fills today. It sits
wherever one marker may sit:

| Position             | Today (one marker)                           | With a run                                                        |
| -------------------- | -------------------------------------------- | ----------------------------------------------------------------- |
| Leading, parent line | First token routes, rest is the note         | Leading run routes, rest is the note                              |
| Leading, whole line  | Marker-only item (Ensure Next or `!` toggle) | Marker-only item for each target                                  |
| Trailing, any line   | Last token routes the item                   | Trailing run routes the item; body text must precede it, as today |
| Mid-line             | Literal note text                            | Literal note text (unchanged)                                     |

- **A run is maximal and ends at the first non-target token.** `@work+a @dev:b note` is
  a one-target line with the note `@dev:b note`, as today. Likewise
  `@work+a @bob said hi` keeps `@bob said hi` as the note.
- **Every member is parsed the way a single marker is parsed.** A malformed member is
  that marker's existing error or `needs`; for example, `@work+a @dev+ note` needs
  `block_id`. It is never literal text.
- **One slot per item.** A run on the parent line plus a marker on a child line is the
  existing `duplicate_capture_marker` ("route/mode marker") error, and so are two runs
  on two lines.
- **Forced destinations disable detection, as today.** With `-r/--route`, `--task`, or
  `--drawer`, every `@…` token stays literal.
- **List bullets** under a lone `@route` header keep refusing local route markers. A run
  there gets the same `list_bullet_marker` refusal.

> [!decision] split_edges = refuse A target marker at the **leading** edge plus a target
> marker at the **trailing** edge of the same line, with note text between them, is
> refused with code `split_capture_targets`. The message is: "`@a+x` and `@b+y` are at
> opposite ends of the line; put every `@route+id` target in one run at the start or
> end". The editor reports this diagnostic on the trailing marker's byte range. This
> also changes today's single-marker `@a+x note @b+y`, which currently writes
> `note @b+y` under `^x`. A trailing marker that is not a target marker
> (`@a+x note @work`) stays literal as today.

> [!decision] split_edges = literal Keep today's precedence: when the parent line starts
> with a target marker (or run), later route-shaped tokens are never inspected, so
> `@a+x note @b+y` writes `note @b+y` under `^x`. Document this in the grammar contract.

### Equivalence principle

A line whose target run has N members captures **exactly like N items, one per member**.
Each of those items is the same line with only that member's marker. Every member is
judged as if it stood alone, so per-marker suffixes keep their meanings:

- a glued `#drawer` applies only to its own marker;
- a terminal `!` makes only that member an explicit toggle;
- a mixed `@a+x! @b+y` toggles `^x` and ensures `^y` is Next.

The fan-out keeps the existing whole-batch semantics:

- planning is sequential against staged snapshots;
- later members see earlier members' staged edits;
- any member's failure rolls back the whole draft;
- `--dry-run` uses the same planner without writing.

The fan-out differs from N hand-written items in exactly these ways:

1. **Duplicates are refused.** Two members naming the same task fail with
   `duplicate_capture_target`: "`@work+a` names ^a twice on this line; name each task
   once". The parse layer compares the lower-cased route and the block ID. The planner
   also compares the resolved note path and block ID, which catches aliases (phase
   `execute`).
2. **One clipboard read per group.** `%c`, `%c<N>`, or `--clip` reads the clipboard (or
   its history) once. Every member uses the same values. A text snippet is saved to one
   `file/clip-…md` that every member links. Attachments are already reused.
3. **One `$<N>` roll per group.** Every member's note gets the same rolled
   `[scheduled::]` date: one line, one date. This already holds because members share
   the item index that seeds the roll. Keep it, and test it.
4. **Grouped output.** Results carry a `fanout` context and print under one header (see
   Output).
5. **Errors name the target.** A member's planning error reads
   `capture item <i> starting on line <l> (target @dev+b): …`.
6. **Group warnings print once.** Identical warnings from members of one group (for
   example `drawer_near_catalog` or a relation label hide) are emitted once.

### What each operation does with a run

| Line shape                       | Each named task gets                                                                                                                      |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `@a+x @b+y <note>` (or trailing) | `- <note>` at the plain-child anchor; authored bullets nest under it                                                                      |
| `@a+x @b+y` + bullets            | Each first-level bullet as its own note; drawer bullets merge into or create that task's drawer (`capture-bullets-mirror-the-task-block`) |
| `@a+x#req @b+y <note>`           | The note inside ^x's `REQUIREMENTS` drawer, and as a plain child of ^y                                                                    |
| `@a+x @b+y <note> %c`            | The note with the shared clipboard children under it                                                                                      |
| `@a+x @b+y %c`                   | The shared clipboard lines as notes directly under the task (Clipboard notes)                                                             |
| `@a+x @b+y <note> s:3` / `$1`    | The note carrying the same `[scheduled::]` (and the same priority roll)                                                                   |
| `@a+x @b+y`                      | Ensure Next and Task Link relocation (refused per task when it has no link, as today)                                                     |
| `@a+x! @b+y!`                    | The `!` link-presence toggle                                                                                                              |
| `@a+x @b+y #fix` (± note)        | A retype to FIX in the same transaction; marker-only also ensures Next                                                                    |
| `… &p:q`                         | A dependency on `p:q`; `p:q`'s BLOCKS line lists both owners                                                                              |
| `… *t:u`                         | The task `t:u` depends on both owners                                                                                                     |
| `… ~r:s` / `~r:s\|label`         | A relation to `r:s`; a complete or cancel label closes each owner                                                                         |
| `… ~~r:s\|label`                 | The relation from `r:s`; a closing label closes `r:s` once, and later members report it already closed                                    |

Existing refusals still apply to each member. They include `multi_note_marker`,
`!`-with-body, closed or missing targets, and an `&`/`*`/`~` modifier naming the
member's own task (which fails with that member's "itself" error, now naming the
target).

### Output

- **`bob capture -f json`**
  - Each member is its own entry in `captures[]`, with an additive per-item
    `fanout: {line, position, count, marker}`. `line` is the run's 1-based physical
    line, `position` is 1-based, and `marker` is that member's exact text. This mirrors
    the existing `list` context.
  - As for every multi-item result, the legacy top-level fields describe the first
    member.
  - `fanout` is absent for single-marker items, so their JSON is byte-identical to
    today's.
- **Human output**
  - Members print as ordinary numbered batch items (`1/N`, …).
  - One dim header line precedes each group's first member:
    `targets  @work+prep · @dev+parser` (mirroring `print_list_summary`).
- **`bob capture-parse -f json`**
  - The draft item stays one item; capture-parse does not fan out.
  - The legacy `mode`/`route`/`section`/`block_id` fields describe the first member.
  - An additive `targets: [{start, end, marker, mode, route, block_id, section}]` lists
    every member and is present only when there are two or more.
  - Every member gets its existing spans: `sub_bullet_*`, `task_toggle_*`, and `drawer`.
  - `needs` is the union over members, in member order without duplicates.
  - Diagnostics carry the member's exact byte range.
  - Single-marker output is byte-identical to today's.

### Compatibility and the grammar revision

This changes the meaning of existing text: `@a+x @b+y note` used to be a note on ^x. It
is therefore a grammar break.

- Bump `GRAMMAR_REVISION` (`src/native/capture_language/mod.rs`) exactly once, in phase
  `grammar`, to one more than the highest revision that either the code or the docs
  claim when the phase starts. The in-flight note-pull epic `bob-cli-gj` already
  documents `grammar: 7`, so this is 8 unless that changed.
- The revision must never go down. If `bob-cli-gj` is still open, append a note to its
  bead naming the revision you took.
- Update every hard-coded revision in tests and docs. Known copies (search for more):
  - `capture_parse.rs` `json_shape_is_stable`
  - `capture_complete/tests/output.rs`
  - `tests/cli/capture/complete_drawers.rs`
  - `tests/cli/capture/complete_relationship_owner.rs`
  - the `docs/capture.md` / `compatibility.md` / `json-outputs.md` / generated CLI
    reference mentions
  - the stale `grammar: 4` sentence in `capture_complete/cli.rs`
- Commit phase `grammar` as `feat(capture)!:` with a `BREAKING CHANGE:` footer. Never
  edit the `Cargo.toml` version or `CHANGELOG.md` (decision
  `bob-cli-releases-ride-a-daily-train`).
- Bob Mac Capture gates on `grammar >= 5`, renders spans by kind, and stacks
  `captures[]` results, so it needs no change (decision `mac-capture-is-a-thin-client`).
  The additive `fanout` and `targets` fields let a later Mac change group a fan-out
  under one header. That change is a follow-up, not part of this epic.

### Non-goals

- Runs of `@route:id`, `^route:id`, `@route^id`, `!note:id`, or plain `@route` markers.
- A single drawer or `!` applying to the whole run.
- Fan-out inside `capture-parse` items.
- Bob Mac Capture changes.
- Changing what a single marker does, apart from the `split_edges` answer.

## Phase `grammar`: Target-run grammar in both resolvers, fan-out, and grammar bump

The execution resolver and the editor resolver must change in lockstep. They choose
markers separately today.

### Execution path

- **`resolve_line_inner`** (`src/native/capture_language/line.rs`): return the whole
  leading or trailing run, not one `LineRoute`. One option is to keep `route` as the
  first member and add the remaining members. Keep the existing leading, marker-only,
  and trailing branches' rules for each member, and apply the `split_edges` answer.
- **`AggregateMarkers::absorb`**: treat a run as one route slot.
- **`extract_terminal_markers`** (`markers.rs`): treat a trailing run as one unit. Then
  `note s:3 @a+x @b+y`, `note @a+x @b+y s:3`, and `note @a+x s:3 @b+y` all extract the
  schedule. Today only one trailing route token is recognized.
- **`validate_special_terminal_markers_line`** (`tokens.rs`): extend it the same way.
- **Item parsing** (`item.rs`, `parse_capture_item_masked`): resolve each member's kind
  and ownership as if it stood alone. That covers the toggle and Pomodoro candidate
  checks, `resolve_sub_bullet_kind`, `execution_item_dependencies`, and the multi-note
  refusals. The most faithful way is to run the existing per-item resolution once per
  member with that member selected and the other members removed from the body. Do not
  hand-copy the rules.
- **Fan-out in the draft** (`draft.rs` `parse_ordinary_draft`, after
  `finish_item_dependency_target`; also the template-row path): push one
  `ParsedCaptureItem` per member.
  - Members share `index`, `start`, `end`, `line_start`, `line_end`, `list`, and
    `template_row`.
  - Members differ in `parsed.route`, `parsed.kind`, and `parsed.dependency_target`.
  - Add an additive `fanout: Option<FanoutContext>` with `line`, `position`, `count`,
    and `marker`. It is `None` everywhere else.
  - Check the duplicate refusal before pushing.
- **Error codes**: add `duplicate_capture_target` (and `split_capture_targets` when
  `split_edges = refuse`) as coded `CaptureItemParseError`s, so the codes reach the
  capture JSON exactly like `list_bullet_marker`.

### Editor path (`capture-parse`)

- **Marker selection**: `select_marker_token` (`editor_classify.rs`) and the
  single-marker removal in `parse_editor_line` (`editor_parse.rs`, `LineEditorParse`)
  must select the same run as execution and remove every member from the body.
- **Duplicates across lines**: `SeenMarkers::absorb_route` treats a run as one slot.
- **Item fields**: compute per member through the existing toggle, drawer, and
  dependency-owner logic. Report the first member in the legacy fields, add `targets[]`
  (`capture_parse.rs` `CaptureParseResult` and the item builder), union `needs`, and
  emit `duplicate_capture_target` / `split_capture_targets` diagnostics on exact byte
  ranges.
- **Single-target subjects**: the type-marker subject, `drawer_bullet_target`, and the
  dependency owner keep naming the first member. Phase `assist` owns the drawer-bullet
  menu.

### Tests

- **`src/native/capture_language/tests/grammar.rs`** (execution). Cover:
  - leading, trailing, marker-only, and child-line runs;
  - per-member `!` and glued `#drawer`, including mixed `!`;
  - a run combined with `#type`, with `%c`, and with `&`/`*`/`~`;
  - terminal `s:`/`$` before, after, and between run members;
  - a run of three;
  - duplicates, and the `split_edges` behavior;
  - `@a+x @b:y note` and `@a+x @bob hi` staying one-target;
  - a malformed member;
  - a list bullet run refused;
  - a forced `-r` keeping markers literal;
  - a template row with a run fanning out.

  Then assert:
  - fanned items share `index` and span;
  - each item's `fanout`, `kind`, and `dependency_target` are correct;
  - every pre-existing single-marker test passes unchanged.

- **`editor_spans.rs` and `editor_modes.rs`**: spans, body, `targets[]`, `needs`, and
  diagnostics for the same shapes. Keep `Discuss @dev+id later` literal.
- **`tests/cli/capture/parse.rs`**: end-to-end `capture-parse` JSON for one leading run,
  one trailing run, and one incomplete member. Also check single-marker JSON is
  unchanged.

### Docs

Read the `docs.md` reference memory first.

- **`docs/capture.md`**
  - Fix the "Routing and insertion" precedence sentence.
  - Fix the "Authored sub-bullets" sentence on marker slots (a run is one slot).
  - Add a "Several tasks on one line" subsection under "Sub-bullets under existing
    tasks" with the vocabulary, the recognition table, the equivalence principle, the
    operation table, and the refusals.
  - Add the `targets[]` field and the two diagnostic codes to the `bob capture-parse`
    section.
  - Update the grammar revision text.
- **`docs/reference/json-outputs.md`**: the capture-parse row.
- **`docs/reference/compatibility.md`**: the grammar history.
- **Help text**: if the `capture-parse` help changes, run `just docs-gen`.

### Done when

`just fmt`, `just lint`, `just test`, and `just docs-check` pass. These live checks must
also hold:

- `bob capture-parse -f json -- '@work+a @dev+b note'` reports body `note` and two
  `targets`.
- `bob capture --dry-run -f json -- '@work+a @dev+b note'` in a scratch vault plans two
  `sub_bullet` captures.

## Phase `execute`: Planner, output, and capture contract for fan-out groups

Phase `grammar` already makes runs execute through the existing batch planner. This
phase adds the group-level behavior and the user-facing contract.

### Planner (`src/native/capture/plan.rs`, `src/native/capture_clip/`)

- **Clipboard**
  - Read once per fan-out group per planning pass. Keep a memo beside
    `clip_reservations`, keyed by the group, holding the raw values. Each member still
    builds its own `ClipPlan` with its own child indentation.
  - A snippet whose bytes match one this batch already reserved reuses that reservation.
    Mirror the attachment reuse in `choose_attachment_destination` (`files.rs`). Each
    member then links the same file, and the file is saved and reported once.
  - `--dry-run --no-clip` still reports `clip: {status: "not_read", count}` per member.
- **Resolved duplicates**: before staging a group, resolve every member's note path and
  block ID through the planner's existing lookup. Refuse a repeat with
  `duplicate_capture_target`, for example an alias plus its canonical route.
- **Error context**: wrap member errors as
  `capture item <i> starting on line <l> (target <marker>): …` (the prefix built near
  `plan.rs` `capture item {index+1} starting on line {line_start}`). Keep codes intact.
- **`$<N>` roll**: confirm members share one roll (`item_roll_seed` uses the shared
  index), and assert it in tests.
- **Group warnings**: emit identical warnings from members of one group once.

### Output (`src/native/capture/output.rs`)

- Thread `fanout` from `ParsedCaptureItem` into `CaptureItemResult`, the same way `list`
  is threaded, and serialize it per item.
- Print the `targets  <m1> · <m2> …` header before position 1, next to
  `print_list_summary`.

### Tests

Add `tests/cli/capture/multi_target.rs` and register it in `tests/cli/capture/mod.rs`.
Follow the fixtures and clipboard stubs that `clip_notes.rs` and `sub_bullet.rs` use.
Every row of the operation table needs an end-to-end byte-for-byte assertion on the
written notes (and daily note where relevant), plus:

- two targets in one note, including a target nested under the other;
- an atomic rollback when the second member fails, with the error naming that target;
- an alias duplicate refused;
- one clipboard read (count the stub's invocations) and one snippet file;
- the same rolled date for `$1`;
- `~~` with a complete label closing the source once;
- `!` toggles and Ensure Next creating a `— T` placeholder only once;
- JSON `captures[].fanout` and legacy top-level fields;
- the human header;
- deduplicated warnings;
- `--dry-run` writing nothing.

### Docs

- **`docs/capture.md`**
  - Complete "Several tasks on one line" with the clipboard, roll, error, and output
    rules.
  - Add one sentence each to "Multi-item capture", "Clipboard", "Clipboard notes", and
    "Task Link toggle".
- **`docs/reference/capture-syntax.md`**
  - Add a Token index row (`@route+id @route+id …` → "Apply the line to each named
    task").
  - Add a heading with an example captured in an isolated `HOME`/`XDG_*` scratch vault,
    as the page's header describes.
  - Add the matching smoke query in `docs/_meta/search_smoke.yml`; the token coverage
    test (`tools/docs/tests/test_capture_token_coverage.py`) requires it.
- **`docs/reference/json-outputs.md`**: describe `fanout` in the `bob capture` row.
- **Help text**: add a sentence to `bob capture --help` (`src/native/capture/cli.rs`,
  the `@<route>+<block-id>` paragraph), then run `just docs-gen`.

> [!decision] memory_bulk_capture When accepted, add one sentence to the
> `glossary:bulk-capture` strand: a line whose adjacent `@route+id` markers name several
> tasks fans out into one item per task, sharing one clipboard read and one output
> header. Use the `/sase_memory_write` procedure, then `sase memory init` and
> `just docs-gen` (the docs glossary page is generated from that web). When declined,
> record the skipped change as that skill directs.

### Done when

`just fmt`, `just lint`, `just test`, and `just docs-check` pass. A live `bob capture`
of the "Why" examples in a scratch vault (isolated `HOME`/`XDG_*`,
`BOB_REF_JOBS_KICK=off`; never Bryan's vault) writes the expected blocks. Its
`--dry-run -f json` shows `fanout` on both captures.

## Phase `assist`: Completion, rewrite, and `@@` absorption inside target runs

### Completion (`src/native/capture_language/completion.rs`, `src/native/capture_complete/`)

`completion_marker_index` picks token 0 or the last token today. Generalize it to the
same run phase `grammar` selects. A cursor may complete any run member, or the `@`-token
that extends a run at its inner edge (the token right after a leading run, or right
before a trailing run). Per-token field parsing and drawer targets
(`marker_field_at_cursor`, `drawer_completion_target`) already use the token's own route
and block ID. Required results:

| Draft (cursor at `\|`)       | Completes                                                |
| ---------------------------- | -------------------------------------------------------- |
| `@work+a @de\|`              | routes                                                   |
| `@work+a @dev+\|`            | block IDs in `dev`                                       |
| `@work+a @dev+b#re\|`        | drawers of `dev ^b`                                      |
| `@work+a @dev:\|`            | block IDs (the existing `:` path, so `+` can rewrite it) |
| `Called Morgan @wo\| @dev+b` | routes                                                   |
| `note @work+a @dev+b#re\|`   | drawers of `dev ^b` (unchanged)                          |
| `Discuss @de\| later`        | nothing (unchanged)                                      |

The bullet drawer menu below a note-less run keeps offering the first member's drawers.
Commit matching stays exact per member, and a drawer missing on another member is
created there.

### Rewrite (`src/native/capture_language/rewrite.rs`)

- **Separator switch**: `switch_block_id_separator` must accept a token that, once
  switched, becomes a run member. `candidate_selects_plain_marker` currently requires
  the line's single selected marker. Required results:
  - `@work+a @dev:b+|` → `@work+a @dev+b`
  - `note @work:a+| @dev+b` → `note @work+a @dev+b`
  - `@work:a+| @dev+b` → `@work+a @dev+b`
- **`@@` absorption**: a target run gets the existing non-absorbable-marker notice. It
  no longer falls into Rule A6's silent no-op. Update the A6 comment in
  `editor_model.rs` (`local_destination_markers`).

### Tests and docs

- Add the completion and rewrite tables to
  `src/native/capture_language/tests/completion.rs` and `tests/rewrite.rs`, plus CLI
  cases in `tests/cli/capture/complete_drawers.rs` / `rewrite.rs`.
- Update `docs/capture.md` (`bob capture-rewrite` and `bob capture-complete` sections).
- Update the `capture-complete` help sentence that says a mid-line cursor returns empty
  (`src/native/capture_complete/cli.rs`), then run `just docs-gen`.

### Done when

`just fmt`, `just lint`, `just test`, and `just docs-check` pass. Live
`bob capture-complete -c N` and `bob capture-rewrite -c N` runs reproduce the tables.

## Coordination

- Phases `execute` and `assist` both depend only on `grammar` and may run in parallel.
  Both touch `docs/capture.md`, in different sections, so rebase before committing.
- Each phase commits through the normal SASE flow. Only `grammar` carries `!`, because
  the `fanout` and `targets` fields are additive.
