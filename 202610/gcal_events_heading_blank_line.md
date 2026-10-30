---
tier: tale
title: Keep a blank line under the Events heading in bob gcal sync
goal:
  Every bob gcal sync run leaves a blank line between the `## Events` heading and the
  section's first row, repairing sections that lack it.
size: small
decisions:
  heading_gap_above:
    ask:
      When sync creates a new `## Events` section, should it also keep a blank line
      above the new heading?
    choices:
      above_too:
        Also keep a blank line above a new heading, so it stops sticking to the bob-plan
        fence
      below_only:
        Fix only the gap below; a new heading still sits right under the previous line
    default: above_too
    why:
      Same function; its doc already promises tidy blank lines, and both gaps broke
      together
    answer: below_only
proposed_by: bbugyi200.apollo.bg
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 14:30:09
status: wip
---

# Keep a blank line under the `## Events` heading in `bob gcal sync`

## Goal

After every `bob gcal sync` run, a daily note's `## Events` heading is followed by
exactly the layout Bryan writes by hand and the one `docs/reference/vault-format.md`
already shows:

```markdown
## Events

- **all day** School holiday
- [ ] #task **0800-0830** Product interview … ^product-interview

## Pomodoros
```

That means a blank line between the heading and the section's first line, and no blank
line splitting the rows. This holds when sync creates the section, when it inserts a row
at the top, when it re-slots a row to the top, and when it finds an existing section
without the gap. In that last case sync adds the gap even if nothing else changed.

## Root cause

All of this is in `src/native/gcal/plan.rs`, `impl WorkingNote`:

1. `ensure_events_section` adds `## Events` followed by one blank line. That blank line
   is meant to separate the section from `## Pomodoros`. Before adding the heading, the
   function steps back over the blank lines above `## Pomodoros` and removes them, so
   the new heading also lands directly under the previous non-blank line.
2. `insert_row` / `insert_block` with `slot == None` ("top of the section") insert at
   `events_range().start`, which is the line directly under the heading. So the first
   row of a new section, any row that sorts before every existing row, and a block
   re-slotted to the top all land with no gap under the heading. In a section that
   already has the gap, the new row goes _above_ the blank line, which also splits the
   list (`## Events` / new row / blank / old rows).
3. No later step checks the gap again.

Evidence: the first sync of `2026/20261030.md` produced this (the bob-plan fence, then
the heading and rows with no blank lines):

````text
```
## Events
- [ ] #task **0800-0830** Product Engineer - … ^product-engineer-bryan
- **0915-1000** Little Gym · *maybe* [📅](…)

## Pomodoros
````

Bryan then added the missing blank line by hand.

## Design

All code changes are in `src/native/gcal/plan.rs`. Rendering, the journal, span hashes,
the JSON schema, and the Pomodoros-section logic are unchanged.

### 1. Rows inserted at the top go below the gap

Add a `WorkingNote` helper (for example `events_top_index`) that returns the line to
insert at when `insert_slot` returns `None`:

- If the section has a non-blank line, return the index of the first one. The new row
  then joins the existing list below the gap, and any extra blank lines stay where they
  are.
- If the section has only blank lines and there are two or more of them, return
  `range.start + 1`. This keeps one blank line above and one below; it is the case where
  re-slotting the only row has just removed it.
- Otherwise, including a section with no lines at all, return `range.start`. The repair
  step in §2 then adds the gap.

Use the helper for the `None` arm of both `insert_row` and `insert_block`, so
`reslot_block` is covered too. The `Some(after)` arms stay as they are.

### 2. One repair step that guarantees the gap

Add `WorkingNote::ensure_events_heading_gap(&mut self)`. If an Events section exists,
`range` is non-empty, and `lines[range.start]` is not blank (`trim().is_empty()` is
false), insert one empty line at `range.start`. It never removes or merges blank lines,
never adds a line to an empty section (`## Events` directly followed by the next
heading), and does nothing when there is no Events section.

Call it once in `plan_sync`, **after `apply_lifecycle` and before `plan_meet_links`**:

- After lifecycle: lifecycle cancels and removals work on line indices taken earlier, so
  they must run before any line is inserted.
- Before the MEET links: `plan_meet_links` reports `plan.pomodoro_entry_line` (1-based)
  from the contents it sees. The new line sits above `## Pomodoros`, so adding it
  afterwards would leave the reported `pomodoro.entry_line` off by one.

`plan.changed` already compares contents, so a run whose only change is the added gap
counts as changed and is written. It produces no row, link, or journal records. Under
the vault lock, `sync.rs` plans again and writes through the preimage-checked staged
writer as usual. The next run produces identical bytes and reports `unchanged`. No
change to `sync.rs` is needed:

- The human summary for such a run is the existing fallback,
  `✓ Events +0 tasks · Pomodoros +0 entry, +0 links`.
- The JSON output is `status: "written"` with `rows: []`.
- `-d` shows a one-line `+` diff.

### 3. New sections keep a blank line above the heading

> [!decision] heading_gap_above = above_too In `ensure_events_section`, when
> `insert_at > 0` (the step-back loop has already made `lines[insert_at - 1]`
> non-blank), add an empty line before `"## Events"`. When `insert_at == 0`, add
> nothing. The fixture note `# Friday 2026-10-30\n\n## Pomodoros\n` then becomes
> `# Friday 2026-10-30`, blank, `## Events`, blank, rows, blank, `## Pomodoros`.

> [!decision] heading_gap_above = below_only Leave `ensure_events_section` as it is; §1
> and §2 still guarantee the gap below the heading. Tests then expect
> `# Friday 2026-10-30\n## Events\n\n…`.

Under either choice, also correct the `ensure_events_section` doc comment so it
describes the layout the function actually produces. Add one clause to the module doc
(`//! The plan inserts new rows in time order …`) saying the plan keeps a blank line
under the `## Events` heading.

### Out of scope

- Merging extra blank lines under the heading. "At least one" is the guarantee, and the
  vault owns the section's layout beyond that.
- The blank line before `## Pomodoros` or between rows. Sync already keeps rows
  together; this plan only stops it from splitting them.
- A blank line above an `## Events` heading that already exists.
- Other writers. Only `gcal/plan.rs` creates `## Events` in bob-cli. bob-plugins is not
  touched.

## Tests

### Unit tests: new `src/native/gcal/plan/tests.rs`

Wire the file in with `#[cfg(test)] mod tests;` at the **end** of `plan.rs`; clippy
denies `items_after_test_module`. Follow the `capture_task_toggle.rs` /
`capture_task_toggle/tests.rs` layout. Add a small builder for `PlannedOccurrence` with
these fields:

- `html_link: Some("https://calendar.google.com/event?eid=<id>")`
- `link_eligible: false`
- `needs_type: false`
- `root: None`
- empty `spans`

Add a `run(contents, occurrences, journal)` helper with `is_today: false`,
`day_string: "2026-10-30"`, and empty `lifecycle` / `series_roots`. Cases:

1. **New section**: `# Day\n\n## Pomodoros\n` plus an all-day context row and a 0900
   task gives exactly the layout from the decision callout: heading, blank line, both
   rows next to each other, blank line, `## Pomodoros`.
2. **Earlier row joins a well-formed list**: `## Events`, blank, a 0900 task row, blank,
   `## Pomodoros`, plus a new 0800 occurrence. Expect heading, blank, the 0800 row, the
   0900 row with no blank line between them, blank, `## Pomodoros`.
3. **Gap-only repair**: `## Events` directly followed by a row, and no occurrences.
   Expect `changed == true`, contents equal to the input with exactly one `\n` added
   after the heading, and all record vectors empty.
4. **Left alone** (`changed == false`, identical bytes) for each of:
   - a well-formed section;
   - `## Events\n## Pomodoros\n` (empty section);
   - `## Events\n\n## Pomodoros\n` (only a blank line);
   - `## Events\n\n\n- row…` (extra blank lines are kept).
5. **Re-slotting the only row keeps the gap**: a journaled task row (a `RowRecord` whose
   `spans` are `split_row_spans(line).hashes()`) whose occurrence now has a different
   time. The result is still heading, blank, row, blank, `## Pomodoros`, with the new
   time.
6. **Idempotent bytes**: planning case 1's output again with the same occurrences gives
   byte-identical contents. Rows are adopted, but the contents don't change.

### Integration tests: `tests/gcal_sync.rs`

- In `first_sync_creates_events_meet_and_links`, or a new focused test, assert the note
  contains `"# Friday 2026-10-30\n\n## Events\n\n- **all day** Conference"`
  (`below_only`: `"# Friday 2026-10-30\n## Events\n\n- **all day** Conference"`) and
  that the line before `## Pomodoros` is blank.
- New `sync_restores_missing_blank_under_events_heading`:
  1. Sync once.
  2. Remove the blank line directly under `## Events` from the note.
  3. Sync again. Expect exit 0, the gap back, an unchanged journal line count, and a
     status file that says `written`.
  4. A third sync prints `unchanged` and leaves the bytes identical.
- Optional: in `tests/gcal_lifecycle.rs` `span_updates_rewrite_clean_spans_and_reslot`,
  assert the line after `## Events` is still blank after the re-slot.

## Docs

The route is contract plus reference: read `sase memory read docs.md` first.

- `docs/gcal.md` § Sync: after the paragraph that starts "The run fetches every
  configured calendar…", add one sentence. It should say that sync keeps a blank line
  between the `## Events` heading and the section's first line, and adds it to a section
  that lacks it, even on a run with no calendar changes. Under `above_too`, also say
  that a section sync creates gets a blank line above its heading.
- `docs/reference/vault-format.md` § Events section and Event Tasks: extend
  "`bob gcal sync` writes rows in this shape" to mention the blank line under the
  heading. The example there already shows it.
- No CLI flags change, so `just docs-gen` is not needed.

## Verification

```bash
cargo test --lib gcal::plan
cargo test --test gcal_sync --test gcal_lifecycle
just check        # fmt, clippy gate, every test binary
just docs-check
```

## Delivery

One commit, `fix(gcal): keep a blank line under the Events heading`. Do not edit
`Cargo.toml`'s version or `CHANGELOG.md`; the daily release train owns both. No memory
notes change: no glossary or decision record describes the blank lines in the Events
section.
