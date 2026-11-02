---
tier: tale
title: Keep Event Task links in time order on every gcal sync
goal:
  Every bob gcal sync run leaves each Pomodoro entry's Event Task links sorted by
  meeting time, repairing misordered notes.
size: medium
decisions:
  memory_link_order_record:
    ask:
      Record the new rule as a decisions strand and mark synced-meetings-link-by-phase
      partly superseded by it?
    memory:
      - decisions:synced-meetings-link-by-phase
      - decisions:sync-sorts-event-task-links
    default: true
    corrects:
      Sync only creates MEET entries and appends links; it never starts, closes,
      retimes, renames, or moves an existing entry.
    answer: true
proposed_by: bbugyi200.athena.16y
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 07:56:15
status: wip
---

# Keep Event Task Links In Time Order On Every `bob gcal sync`

## Problem

`bob gcal sync` files each new Event Task under a MEET Pomodoro by **appending** its
`[[#^id]]` link to the end of the entry's children (`append_links` →
`insert_link_into_entry` in `src/native/gcal/plan.rs`). Order therefore follows _when
sync first saw the event_, not when the event happens. In the vault's
`2026/20261102.md`, the 19:00 `^lincoln-park-w` row was synced at 06:44 and the 08:00
`^swings` row at 07:40, so the MEET placeholder reads:

```markdown
- [ ] () — MEET
  - [[#^lincoln-park-w]]
  - [[#^swings]]
```

Bryan wants sync to keep Event Task links in scheduled-time order, so `[[#^swings]]`
sits above `[[#^lincoln-park-w]]`, and wants the existing misordered note repaired by
the next sync. Link order is not cosmetic: `capture_active_tasks` and the capture agenda
order work by `(entry, link)` position, so a time-ordered lineup also orders the agenda.

## Behavior Contract

On every `bob gcal sync` run (any day, `-D` included), after rows, lifecycle, and MEET
filing are planned and before the stamp step:

1. **Which lines move.** Within each Pomodoro entry in `## Pomodoros` (closed, running,
   or planned; any name), only _Event Task link lines_ move. An Event Task link line is
   a direct-child dedicated Task Link as listed by
   `capture_pomodoro_start::session_links` (the start/close lineup rule, struck
   `~~[[…]]~~` links included; 🍅 markers allowed) whose target is this note — an empty
   path part (`[[#^id]]`), or a path part whose last component without `.md` is the
   note's `YYYYMMDD` stem — and whose block ID names exactly one Event Task row in this
   note's `## Events` section. Every other child (notes, non-event Task Links,
   mixed-text bullets, links to other notes, links whose ID is ambiguous or not an Event
   Task) keeps its line position.
2. **Sort key.** The meeting span, exactly the order `event_task::events_in_note`
   already defines for Event Tasks: start minute, then end on the timeline
   (cross-midnight aware), then the row's line in `## Events`. Not the buffered span:
   link order must match the bold `**HHMM-HHMM**` labels and the Events section Bryan
   reads, which sync already keeps in meeting-start order (`insert_slot` /
   `row_sort_key`). Two links to the same block ID keep their relative order (stable
   sort).
3. **Slot-stable permutation.** The positions occupied by an entry's Event Task link
   lines are refilled with those same lines in sorted order. Each link line moves
   together with its own child subtree (`child_block_end_line`). Gaps between link
   blocks (including blank lines and non-event siblings) stay exactly where they were.
   Entry header lines and the note's total line count never change, so
   `MeetPomodoroEntry.line` values already computed stay valid.
4. **Writes.** An entry whose Event Task links are already in order is left
   byte-identical. A run whose only change is a reorder still writes the note (and the
   stamp refreshes because `plan_stamp` sees `other_changed`). A rerun after a sort is a
   no-op apart from the normal per-minute stamp rule. The journal is untouched (link
   records are keyed by block ID, not position).
5. **Reporting.** The plan counts `links_sorted`: the number of Event Task link lines
   whose position changed (`[b, a]` → 2; `[c, a, b]` → 3).

For `2026/20261102.md`, the next sync rewrites the MEET placeholder to:

```markdown
- [ ] () — MEET
  - [[#^swings]]
  - [[#^lincoln-park-w]]
```

## Implementation

### 1. New pure module `src/native/gcal/link_order.rs`

Register it in `src/native/gcal/mod.rs` (`mod link_order;`). Expose one function to the
planner, for example:

```rust
/// Reorder each Pomodoro entry's same-note Event Task links by meeting
/// span, leaving every other child in its slot. Returns the new contents
/// and how many link lines moved.
pub(super) fn sort_event_task_links(contents: &str, day: NaiveDate) -> (String, usize)
```

Build it from existing helpers rather than new parsers:

- Rank map: `event_task::events_in_note(contents, day)`, keep `EventKind::Task` rows
  with a non-empty `block_id`, rank = index in that already-sorted list; drop any block
  ID that appears on more than one row. Return the input unchanged when the map is
  empty.
- Entries: `capture_pomodoros::scan(contents)`; return unchanged when the note has no
  Pomodoros section.
- Per entry (`entry.line - 1` is the 0-based index):
  `capture_pomodoro_start::session_links(contents, entry_index)` gives
  `StartLink { ledger_line (1-based), path_part, block_id, .. }`. Keep links whose
  `path_part` names this note (rule 1) and whose `block_id` is in the rank map. Skip
  entries with fewer than two.
- Block range per kept link: `ledger_line - 1` through
  `child_block_end_line(&line_spans(contents), ledger_line - 1)`.
- Compute the stable sorted permutation by rank. If it is the identity, skip the entry.
  Otherwise rebuild the region from the first block's start to the last block's end: gap
  segments copied verbatim, slot _i_ filled with the _i_-th sorted block. Count slots
  whose block changed.
- Because every region keeps its length, all entries can be computed from one scan;
  apply the rebuilt regions to a `Vec<String>` of lines and re-join preserving the
  input's trailing newline (mirror `WorkingNote::finish`).

`session_links` is the struck-inclusive variant of the lineup rule (`list_queued_links`
excludes struck links); using it means `task_status_hooks::settle`'s `🍅 ~~[[#^id]]~~`
lines sort too. It is already `pub(crate)` in `crate::native::capture_pomodoro_start`.

### 2. Wire it into `plan_sync` (`src/native/gcal/plan.rs`)

- Add `pub links_sorted: usize` to `SyncPlan` (document it as "Event Task link lines
  this run moved into meeting-time order").
- In `plan_sync`, call the sorter **after** `plan_meet_pomodoros` and **before**
  `plan_stamp`, so links appended this run are sorted in the same write and the stamp
  sees the change:

  ```rust
  plan_meet_pomodoros(input, &mut working, &mut plan);
  sort_event_task_links_in(input, &mut working, &mut plan); // replace working contents when changed, set plan.links_sorted
  plan_stamp(input, &mut working, &mut plan);
  ```

  Use `note_day(input)` for the day.

- In the `!plan.changed` reset block, also zero `links_sorted`.
- Leave `append_links` / `insert_link_into_entry` as they are; the pass after them
  handles placement. Do not touch Events-section row ordering.
- Update the module doc comment at the top of `plan.rs`: rows are still inserted in time
  order without reordering existing rows, and every run also keeps each Pomodoro entry's
  Event Task links in meeting-time order.

### 3. Report it (`src/native/gcal/sync.rs`)

- Add `links_sorted: usize` to `SyncReport`; fill it from the plan for `dry_run` and
  `written`, `0` for `unchanged` and `waiting` (and in `run`'s error-path report literal
  around line 66).
- JSON (additive, `schema_version` stays 1): `pomodoro.links_sorted` next to
  `links_added`.
- Human summary:
  - `stamp_only` also requires `report.links_sorted == 0`, so a sort-only run no longer
    claims "stamp refreshed".
  - Sort-only run (written, no rows, no links added, no entries created,
    `links_sorted > 0`): `✓ Events unchanged · Pomodoros ↕2 links sorted`.
  - Otherwise append `, ↕N sorted` to the Pomodoros part when N > 0:
    `✓ Events +1 tasks · Pomodoros +0 entries, +1 links, ↕2 sorted`.
- `record_status` / `status.json` need no new field.

### 4. Tests

Unit tests in `link_order.rs` (pure, string in/out):

- Two out-of-order links under a placeholder swap; already-sorted input is
  byte-identical with count 0.
- Non-event children stay in their slots: an entry `[[#^late]]`, `[[sase#^task]]`, a
  plain note bullet, `[[#^early]]` becomes `[[#^early]]`, `[[sase#^task]]`, note,
  `[[#^late]]`.
- A link's nested child bullets move with it; blank lines between children stay put.
- 🍅 and struck (`🍅 ~~[[#^id]]~~`) links in a closed entry sort; mixed-text bullets and
  deeper descendants do not move.
- Tie on start sorts by end, then Events row order; duplicate links to the same ID keep
  relative order.
- Links to other notes, to unknown IDs, and to a block ID duplicated across two Events
  rows stay put; `[[2026/20261030#^id]]` on the 2026-10-30 note sorts like `[[#^id]]`.
- Each entry sorts independently; no cross-entry moves.
- A cross-midnight row (`**2330-0030**`) sorts by its start.

Planner tests in `src/native/gcal/plan/tests.rs`:

- The reported scenario: a note whose MEET placeholder already holds the journaled,
  linked 19:00 row's link; a new 08:00 future occurrence is created this run → contents
  list `[[#^swings]]` above `[[#^lincoln-park-w]]`, `links_added == 1`,
  `links_sorted == 2`.
- Sort-only: misordered links, unchanged calendar → `changed`, stamp refreshed,
  `links_sorted == 2`; replanning the output is unchanged apart from the stamp rule
  (extend the existing idempotence pattern in
  `replanning_output_with_journal_is_idempotent`).
- Adjust any existing test whose expected output encoded append order rather than time
  order; do not weaken assertions otherwise.

Integration test in `tests/gcal_sync.rs` (fake gog, as the existing tests do): seed
today's note with a journaled later event already linked under the MEET placeholder, add
an earlier event to the fixture, run `bob gcal sync -f json` → the earlier link is
first, `pomodoro.links_sorted` is reported, and a second run at the same minute is
`unchanged`. Also assert the human sort-only summary line once.

### 5. Docs (`docs/gcal.md`)

Behavior change → contract update in the reference page:

- § Sync, after the sentence about filing created Event Tasks: every run also keeps
  Event Task links in meeting-time order within each Pomodoro entry (rule 1–3 in plain
  words: only dedicated same-note Event Task links move, each with its child bullets;
  other children keep their places; a hand reorder is undone on the next sync).
- Replace "Sync never starts, closes, retimes, renames, or moves an existing Pomodoro:
  it only creates entries and appends links." with "…: it only creates entries, adds
  links, and reorders Event Task links within an entry."
- Ownership table:
  `MEET Task Link | Bob, once | Never re-added or moved; removed only by a cancel` →
  `Never re-added; kept in Event Task time order within its entry, so a hand reorder is undone; removed only by a cancel`.
- JSON: list `links_sorted` in the `pomodoro` object in both § Sync and § JSON output,
  with the human `↕N sorted` example.

### 6. Memory

> [!decision] memory_link_order_record Use `/sase_memory_write`. Add
> `sase/memory/decisions/sync-sorts-event-task-links.md` in the existing record format
> (frontmatter `keyword`, `aliases`, `summary`, `metadata.status: accepted`,
> `metadata.decided: 2026-11-02`, `title`). Title "Sync Sorts Event Task Links By
> Meeting Time"; summary: "Every bob gcal sync run reorders each Pomodoro entry's
> same-note Event Task links by meeting span; other children keep their slots." Body:
> **Applies to** bob-cli and the vault; **Claim** = rules 1–4 above; **Why** = Bryan's
> 2026-11-02 request (`^swings` at 08:00 below `^lincoln-park-w` at 19:00 in
> `2026/20261102.md`) and agenda/lineup order follows link position; **Rejected**:
> sorted insertion only (never repairs an already-misordered entry or hand/capture-added
> links), buffered-span order (disagrees with the bold labels and the Events section),
> sorting every child (scrambles notes and non-event lineup order), grouping event links
> contiguously (moves non-event children); **Cost**: a hand reorder of Event Task links
> is undone on the next sync, and a sort-only run writes the note; **Reopens when**
> Bryan wants a manual meeting order to stick or link order should follow buffered
> spans; **Evidence**: `plan:202611/gcal_sort_event_task_links.md`, `docs/gcal.md` §
> Sync, `tests/gcal_sync.rs`; link `[[decisions:synced-meetings-link-by-phase]]`,
> `[[decisions:event-buffers-widen-the-span]]`, `[[glossary:event-task]]`,
> `[[glossary:task-link]]`.

> [!decision] memory_link_order_record In
> `sase/memory/decisions/synced-meetings-link-by-phase.md`, append
> `decisions:sync-sorts-event-task-links` to `metadata.superseded_by` (status stays
> `superseded-in-part`) and add one paragraph after the existing "Superseded in part"
> paragraphs: "Superseded in part: sync no longer only appends links; every run reorders
> each entry's Event Task links by meeting time. See
> [[decisions:sync-sorts-event-task-links]]." Do not otherwise edit the accepted body.

After memory edits: `sase memory init`, then `just docs-gen` (regenerates
`docs/develop/decisions.md` from the decisions web).

## Verification

- `just fmt`, `just lint`, `just test` (or the narrower `cargo test gcal` /
  `cargo test --test gcal_sync` while iterating, then the full gate).
- `just docs-check`.
- Optional manual check, read-only: `-d` skips the `gcal.writer` guard, so on a host
  with working gcal credentials `cargo run -- gcal sync -d` previews today's note; the
  diff should show only the Event Task link swap plus the stamp line. Never run a
  writing sync from a host other than `gcal.writer`; when credentials are unavailable,
  rely on the tests.

## Out Of Scope

- Other writers that add Event Task links (capture, plugins, task-status hooks) keep
  their placement; the next sync run sorts them.
- Events-section row order (already time-ordered at insert and on retime).
- Commits, version bumps, and `CHANGELOG.md` (release train owns those); the change is
  additive (`feat(gcal): …`, no `!`).
