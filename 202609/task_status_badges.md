---
tier: tale
title: Status-count badges under grouped Tasks headings
goal:
  Every Tasks container that bob task-status-hooks decorates with status groups also
  carries a machine-owned row of linked status-count badges directly beneath its
  heading, refreshed idempotently on every run.
size: medium
proposed_by: bbugyi200.athena.0is
create_time: 2026-09-10 13:33:01
status: wip
---

# Status-count badges under grouped Tasks headings

## Outcome and tier

`bob task-status-hooks` already rearranges eligible `[[area]]` and `[[project]]` `Tasks`
sections into the three generated status groups. Extend that same transform so every
container it decorates also carries a machine-owned badge row directly beneath its
heading: one compact, clickable chip per bucket showing how many task blocks that bucket
currently holds, with each chip linking to the heading it counts.

This is a tale of size `medium`. All of the real work lands inside one existing pure
module (`src/native/task_status_groups.rs`) whose transform contract, ownership-marker
convention, fail-closed discipline, and unit-test harness already exist; the ripples
into the hooks report, `bob capture`'s insertion point, docs, and acceptance tests are
small and mechanical. It is substantial but bounded, needs no second planning pass, and
no part of it is independently deliverable in a way that would justify an epic.

Development and acceptance use disposable fixture vaults only. Do not run a mutating
`bob` command against the personal `~/bob` vault as an implementation test. No linked
plugin, macOS frontend, configuration, memory, or deployment change is required, and no
new CLI subcommand or option is added.

## Concern to record before implementing

Badges are written bytes, not a live view. They are recomputed only when
`bob task-status-hooks` runs, so a `bob capture`, a checkbox toggled by hand in
Obsidian, or a `bob move-done-tasks` archive pass leaves the numbers stale until the
next run. That is inherent to putting counts in the note, which is what was asked for,
and it is acceptable: the counts are a snapshot and the next run reconciles them.
Implement it as specified, document the snapshot semantics in
`docs/task-status-hooks.md`, and do not try to compensate by hooking other commands. If
live counts are wanted later, the natural home is the existing `bob-project-tasks`
Obsidian plugin, which already recomputes `task_count` / `open_task_count` frontmatter
on edit; that is out of scope here.

## Relevant existing behavior

- `src/native/task_status_groups.rs` is the pure transform: it does not read the
  filesystem, mutate statuses, resolve links, or write files.
  `transform(contents, &classification) -> TransformOutput` returns rewritten bytes plus
  `grouped_sections` and `warnings`.
- A _container_ is a `Tasks` heading section or one of its authored descendant heading
  sections. `visit_tasks_roots` finds `Tasks` roots; `rewrite_container` recurses into
  authored children. Each container owns its own three generated groups.
- `HeadingNode` carries `level`, `title`, `heading_line`, `heading_span`, `body_span`,
  `section_span`, `ancestry` (heading titles from the note root down to and including
  this heading), and `children`. `exclusive_spans(node)` yields the container body
  regions not covered by child heading sections; the first such span is the unheaded
  intake.
- `classify_children` decides which children are authored topics, nested `Tasks`
  containers, or managed groups, and sets `fail: Option<GroupingSkipCode>` for the
  fail-closed cases. `stray_marker_in_exclusive` fails a container closed when a
  `bob:task-status-group:v1:` marker appears outside a group heading body.
- `parse_container_regions` splits the container into `intake: Vec<DirectPiece>` and
  per-group pieces. `DirectPiece` is either a `Task(TaskBlock)` or an
  `Other { raw, byte_range }` run of verbatim bytes; `flush_other` merges every non-task
  byte range between tasks into one `Other` piece.
- `grouping_plan` produces `intake_staying`, `recovered` (tasks pulled back out of a
  managed group into the intake), `groups`, `group_leading` / `group_trailing` prose,
  and `counts: [usize; 3]` in `GroupKind::ALL` order (`Active`, `Blocked`, `Closed`).
- `emit_grouped_body` assembles the container body as intake, then authored child
  sections, then the three groups. `section_newline` picks the container's newline
  style; `apply_final_newline` preserves the file's final-newline convention.
- `heading_mask` masks frontmatter, fenced code, and HTML-comment lines, so a standalone
  HTML comment is never a heading and is never parsed as a list item.
  `markdown::standalone_html_comment` extracts the inner text of a whole-line comment.
- Reporting: `GroupedSection` → `grouped_section_report` → `GroupedTaskSectionReport`
  (JSON `grouped_task_sections`), and `print_grouped_task_sections` renders the human
  line `next/in progress N · blocked N · done/canceled N · moved N`.
- `capture.rs::tasks_section` returns `heading_end` (byte offset at the end of the
  `Tasks` heading line) and the intake line range, stopping at the first subsequent
  heading. `insert_task_line` appends after the last top-level task block in that range,
  or falls back to `heading_end` when the intake holds no task.
- `collect_done.rs` link repair only rewrites block-id links (`#^id`); heading anchors
  are outside its scope. The `bob-project-tasks` plugin counts `#task` lines under an H2
  `Tasks` heading and is unaffected by a non-task prose line.
- Checks: `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
  `cargo test`). Contracts live in `docs/task-status-hooks.md`, `docs/capture.md`, and
  the README's task-status-hooks paragraph. Acceptance coverage lives in `tests/cli.rs`
  (`task_status_hooks_syncs_fixture_and_is_idempotent`, the grouping test that asserts
  `grouped_task_sections`, and
  `task_status_hooks_reports_grouping_warnings_without_noop_text`).

## User-facing layout

A decorated container renders like this. The ownership comment is hidden in Obsidian's
reading and live-preview modes; only the chip row is visible.

```markdown
## Tasks

<!-- bob:task-status-badges:v1 -->

[`⚪ 1 open`](#Alpha#Tasks) · [`🔵 2 next/wip`](#Alpha#Tasks#Next%20&%20In%20Progress) ·
[`🔴 1 blocked`](#Alpha#Tasks#Blocked) ·
[`🟢 2 done/canceled`](#Alpha#Tasks#Done%20&%20Canceled)

Short context for this project.

- [ ] #task An idea to pick up later ^idea

### Next & In Progress

<!-- bob:task-status-group:v1:active -->

- [/] #task Finish the design ^design
- [*] #task Review the implementation ^review

### Blocked

<!-- bob:task-status-group:v1:blocked -->

- [?] #task Ship when the dependency is ready ^ship

### Done & Canceled

<!-- bob:task-status-group:v1:closed -->

- [x] #task Agree on the scope ^scope
- [-] #task Superseded experiment ^experiment
```

### Why this rendering

Each chip is a Markdown link whose visible label is a code span. Obsidian styles inline
code as a rounded, tinted pill, so the row reads as a set of badges with no CSS snippet,
no vault theme dependency, no plugin, and no network fetch; the coloured circle supplies
the only per-bucket colour, which shields.io-style remote images would otherwise have to
fetch on every render. Obsidian resolves `[label](#Heading)` and the nested
`[label](#Outer#Inner)` heading-path form natively in both reading and live preview, and
in source mode the row degrades to a legible plain-text line. Do not substitute inline
HTML `<span style=...>`, a remote badge image, a CSS snippet, or a callout.

### Decisions the row encodes

- **Four chips, not three.** The request named open, wip/next, and done/canceled; the
  transform also owns a `Blocked` group, and omitting it would make the row an
  incomplete census that does not sum to the section's task count. `Blocked` is
  therefore included.
- **Zero counts are still rendered.** The three group headings are already retained when
  empty, so the row keeps a fixed shape and a fixed set of link targets instead of
  changing width every time a bucket empties.
- **The open chip links too.** It targets the container heading itself, because Ready
  tasks live in the unheaded intake directly under that heading. A uniformly linked row
  is worth the self-referential jump.

## Badge block contract

### Shape and bytes

The badge block is exactly two lines placed as the first content of the container body,
immediately after the heading line and followed by one blank line:

1. the ownership marker line, `<!-- bob:task-status-badges:v1 -->`;
2. the chip row.

The chip row is the four chips in this fixed order joined by `" \u{b7} "` (space, U+00B7
MIDDLE DOT, space):

| Order | Emoji | Label           | Count                        | Link target           |
| ----- | ----- | --------------- | ---------------------------- | --------------------- |
| 1     | `⚪`  | `open`          | intake task blocks           | the container heading |
| 2     | `🔵`  | `next/wip`      | `Next & In Progress` members | that group heading    |
| 3     | `🔴`  | `blocked`       | `Blocked` members            | that group heading    |
| 4     | `🟢`  | `done/canceled` | `Done & Canceled` members    | that group heading    |

Each chip is ``[`<emoji> <count> <label>`](<anchor>)`` — a Markdown link whose label is
a single code span containing the emoji, a space, the decimal count, a space, and the
label. No thousands separators, no pluralisation, no padding.

Newlines in the block use `section_newline` for that container, so a CRLF note gets a
CRLF badge block. The trailing blank line separates the row from following prose or
tasks and keeps the row from ever being read as part of a lazy continuation.

### Counts

Counts describe membership of the container's own regions after the transform has placed
everything, so they always match what the reader sees directly below them:

- `open` is the number of task blocks emitted into the container's unheaded intake — the
  tasks in `intake_staying` plus every task in `recovered`. A task the transform leaves
  in the intake because it is structurally ineligible to move (ordered-list root, nested
  under an ordinary list item) counts as open, because that is where it sits.
- The other three counts are `grouping_plan`'s existing `counts` array, unchanged.

Counts are per-container and non-recursive. A `## Tasks` container's row counts only its
own direct task blocks and its own three groups; an authored `### Backend` child with
its own generated groups gets its own row with its own numbers, and neither row rolls
the other one up.

### Anchors

Build each anchor from the target heading's full ancestry so it is unambiguous even when
a note has several decorated containers with identically titled groups:

- the open chip targets `node.ancestry`;
- a group chip targets `node.ancestry` with the group title appended.

Render the anchor as `#` followed by the percent-encoded segments joined by literal `#`,
for example `#Alpha#Tasks#Next%20&%20In%20Progress`. This is Obsidian's documented
nested heading-path subpath form, and it also disambiguates the case a leaf-only anchor
gets wrong, where Obsidian would jump to the first same-titled heading in the note.

Percent-encoding is per UTF-8 byte, uppercase hex, applied to exactly this set: ASCII
space, `(`, `)`, `<`, `>`, `"`, `%`, backslash, and any ASCII control byte. Encode `%`
first so an authored `%` never collides with an escape this step introduces. Every other
byte, including `&` and non-ASCII text, is emitted literally, which keeps the source
line readable and matches the links Obsidian itself writes.

If any segment of a chip's ancestry contains `#`, that path cannot be expressed as a
subpath. In that case emit the whole row unlinked — the four bare code spans joined by
the same separator — rather than emitting a link that cannot resolve. This is a
per-container decision, not a per-chip one, so the row never mixes linked and unlinked
chips.

### When a row exists

A badge row exists exactly when the container has the three generated group headings —
the same gate that already governs the groups themselves. Undecorated and Ready-only
`Tasks` sections stay bare. Because badges must also be refreshed and cleaned up when
nothing else about the container changes, extend the rewrite gate in `rewrite_container`
from `parsed.has_groupable() || classified.has_groups()` to additionally fire when the
container holds a badge block.

An orphaned badge block — a badge marker in a container that has no managed groups and
nothing groupable — is removed. It is this tool's own output with nothing left to
describe, and deleting it is self-healing rather than destructive.

When a container fails closed for any reason, its existing badge row is left exactly as
it is, like the rest of that container.

### Ownership and manual edits

The marker owns its own line and, when the immediately following line is non-blank, that
one line as well. Both are regenerated on every rewrite. Nothing else in the container
is owned: authored prose above, below, or between tasks is preserved verbatim.

The marker is recognised anywhere in the container's intake region, not only at the
slot. When it is found lower down — which is what happens after `bob capture` inserts a
task above it into a previously empty intake — the block is lifted back to the slot and
the surrounding authored content keeps its relative order. This self-heals instead of
freezing the container.

Fail closed for the container, leaving it byte-identical and emitting a warning, when:

- more than one badge marker appears in the container;
- a badge marker appears inside a managed group body, or in a group heading's body, or
  anywhere in the container other than its intake region;
- a whole-line comment starts with `bob:task-status-badges:` but does not match
  `bob:task-status-badges:v1` exactly.

Add one skip code for these, `GroupingSkipCode::MalformedBadgeMarker`, with `as_str()`
value `"malformed_badge_marker"` and a message that names the container title and the
heading line, matching the existing codes' phrasing. Do not silently repair ownership by
deleting user text.

Re-running on unchanged semantic input must be byte-identical and must produce no write.

## Implementation

### `src/native/task_status_groups.rs`

1. Add `pub(crate) const BADGE_MARKER: &str = "<!-- bob:task-status-badges:v1 -->";` and
   a `BADGE_MARKER_PREFIX: &str = "bob:task-status-badges:"` next to the existing group
   marker constants, plus a `parse_badge_marker(line) -> Option<Result<(), ()>>` helper
   shaped like `parse_group_marker`.
2. Add `GroupingSkipCode::MalformedBadgeMarker` with its `as_str()` and `message()`
   arms.
3. Add a `DirectPiece::Badges { byte_range }` variant. In `parse_direct_pieces`, before
   the existing masked-line skip, detect a badge-marker line: flush the pending `Other`
   run, consume the marker line plus the immediately following non-blank line, push the
   `Badges` piece, and advance the cursor past it. Only do this for the intake parse; a
   badge marker seen while parsing a group body is a `MalformedBadgeMarker` error.
4. Extend the container-level marker audit alongside `stray_marker_in_exclusive` so a
   badge marker outside the intake region, a second badge marker, or an unrecognised
   `bob:task-status-badges:` comment sets `fail = Some(MalformedBadgeMarker)`.
5. Carry a `has_badges: bool` on `ParsedContainer` and include it in the `should_group`
   gate.
6. In `grouping_plan`, drop `DirectPiece::Badges` from `intake_staying` (the row is
   re-emitted, never preserved) and compute `open_count` as described above. Store it on
   `GroupingPlan`.
7. Add a `render_badges(node, plan, newline) -> String` function plus a small
   `encode_anchor_segment` helper, and unit-test the encoder directly.
8. In `emit_grouped_body`, emit the badge block first, then the intake, then authored
   children, then the groups. Make sure the existing leading-newline normalisation at
   the end of that function still yields exactly one newline between the heading line
   and the marker, and exactly one blank line between the row and whatever follows,
   including when the intake is empty.
9. Extend `GroupedSection` with `pub open: usize` and populate it from the plan.

### `src/native/task_status_hooks.rs`

10. Add `open` to `GroupedTaskSectionReport` and `grouped_section_report`. Place it
    first in the struct so the JSON object reads `open`, `next_and_in_progress`,
    `blocked`, `done_and_canceled`, `moved_block_count`, `moved_blocks`.
11. Extend `print_grouped_task_sections`'s line to
    `open N · next/in progress N · blocked N · done/canceled N · moved N`.

### `src/native/capture.rs`

12. In `tasks_section`, advance `heading_end` past a badge block that begins on the line
    immediately after the `Tasks` heading (marker line, plus the next line when
    non-blank), so an ordinary capture into an otherwise empty `Tasks` section lands
    below the row instead of between the heading and the row. Leave `start_line` /
    `end_line` alone: the row is not a task line and never becomes an insertion anchor.
    Detect the marker with a local whole-line comment check; do not add a dependency
    from `capture.rs` on the grouping module beyond the shared constant if that is the
    cleanest way to keep the two spellings from drifting.

### Documentation

13. `docs/task-status-hooks.md`: in **Status Grouping**, add the badge row to the
    rendered example and a short subsection covering the four chips and their order, the
    per-container non-recursive count semantics, the anchor form and the unlinked `#`
    fallback, marker ownership and the self-healing relocation, the fail-closed cases
    and the new `malformed_badge_marker` code, and the snapshot/staleness note from the
    concern above. In **Output**, document the new `open` field and the changed human
    line.
14. `README.md`: extend the task-status-hooks paragraph's grouping clause to mention the
    linked status-count badge row.
15. `docs/capture.md`: update the sentence about inserting "after one blank line below
    the `Tasks` heading when the section has no tasks yet" to say the insertion point is
    below the generated badge row when one is present.

## Tests

### Unit tests in `src/native/task_status_groups.rs`

Add to the existing `mod tests`:

1. First decoration of a plain `Tasks` section emits the marker, the row, and one blank
   line before the intake, with correct counts and anchors.
2. Re-running on that output is byte-identical and reports `changed == false`.
3. Changing a task's status changes only the affected counts in the row.
4. A bucket with no members still renders its chip with `0`.
5. An authored `### Backend` child container gets its own row, and both rows carry
   ancestry-qualified anchors that differ.
6. A badge marker found below intake prose and tasks is relocated to the slot with all
   authored content preserved in order.
7. An orphaned badge block in a container with no groups and nothing groupable is
   removed.
8. Two badge markers in one container fail closed: bytes unchanged, one
   `malformed_badge_marker` warning.
9. A badge marker inside a managed group body fails closed the same way.
10. `<!-- bob:task-status-badges:v2 -->` fails closed the same way.
11. A container whose ancestry contains a `#` renders the row unlinked.
12. Heading titles containing spaces, `(`, `)`, and `%` encode correctly; `&` and
    non-ASCII stay literal. Cover `encode_anchor_segment` directly as well as end to
    end.
13. A CRLF note produces a CRLF badge block and preserves its final-newline convention.
14. A Ready-only `Tasks` section stays bare — no marker, no row.
15. The row is never classified as a task and never triggers an ambiguous-boundary skip,
    including when a task list follows it immediately after the blank line.

### Acceptance tests in `tests/cli.rs`

16. Extend the existing grouping acceptance test to assert that each written project and
    area note contains its badge marker and a row whose counts match the reported
    `grouped_task_sections` entry, that the JSON entry carries the new `open` field,
    that the human output shows the `open N` segment, and that an immediately repeated
    live run is a no-op leaving the file bytes unchanged.
17. Add a case covering `malformed_badge_marker`: a fixture area note with duplicate
    badge markers is reported as a grouping warning and left byte-identical, alongside
    the existing `h6_container` case.

## Acceptance criteria

- `just all` passes: `cargo fmt --check`, `cargo clippy --all-targets --all-features`
  with no new warnings, and `cargo test`.
- On a disposable fixture vault, a dry run reports the badge counts without writing, a
  live run writes them, and an immediately repeated live run is a byte-stable no-op.
- On that same fixture vault, `bob capture` into a decorated note inserts the new Ready
  task below the badge row, and the following `bob task-status-hooks` run increments the
  open count without duplicating or churning the row.
- On that same fixture vault, `bob move-done-tasks` still archives whole task subtrees
  out of a decorated note, leaves the managed group headings and the badge row intact,
  and the next `bob task-status-hooks` run converges the done/canceled count back to a
  byte-stable note.
- Notes that fail closed keep their previous badge row and their previous bytes.
- No mutating `bob` command is run against the personal `~/bob` vault at any point.

## Out of scope

- Any new CLI flag, option, or subcommand, and any way to turn badges off.
- CSS snippets, theme changes, plugin changes, remote badge images, and inline HTML.
- Live badge refresh from Obsidian, and any change to the `bob-project-tasks` plugin's
  `task_count` / `open_task_count` frontmatter.
- Changing group titles, group order, ownership markers, intake placement, or any other
  part of the existing grouping contract.
- Recursive or note-wide rollup counts.
