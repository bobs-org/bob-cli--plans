---
tier: tale
title: Fix extra blank line when atomic-start capture creates a Pomodoro
goal:
  Pomodoro-linked `=<X>` start captures that create a new ledger entry insert it before
  the first existing Pomodoro, after any blank lines under the heading, exactly like
  non-start `#name` creation, so no stray blank line splits the ledger.
size: small
proposed_by: bbugyi200.apollo.2b
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.2b](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.2b.md)
- **COMMITS:**
  - [b068045](https://github.com/bobs-org/bob-cli/commit/b068045cf57e2e36e59179460de80f278e47d053)
    — fix(capture): place new started pomodoro before first open entry

# Fix extra blank line when `=<X>` start capture creates a new Pomodoro

## Problem

Capturing a Pomodoro-linked task with the atomic-start suffix (`@route:id#pomodoro=<X>`,
or unnamed `@route:id=<X>` when no untimed placeholder exists) creates a new started
ledger entry. When today's daily note has no completed Pomodoros, the new entry lands
_between_ the `## Pomodoros` heading and the blank line the daily template puts after
it, so the blank line ends up between the new entry and the first existing Pomodoro:

```markdown
## Pomodoros (…)

- [ ] (**0715-0805** [t:: 50m]) — RELAUNCH
  - [[sase#^relaunch-260927]]

- [ ] () — GTD
  - [[#^gtd]]
```

Expected (the placement the non-start `#name` creation path already produces):

```markdown
## Pomodoros (…)

- [ ] (**0715-0805** [t:: 50m]) — RELAUNCH
  - [[sase#^relaunch-260927]]
- [ ] () — GTD
  - [[#^gtd]]
```

This was confirmed against the real vault history for `2026/20260927.md` (the template
writes `## Pomodoros (…)\n\n- [ ] () — GTD…`) and reproduced with the current binary for
three variants: named-new start, unnamed start with no placeholder, and a CRLF daily
note. The non-start `@sase:id#relaunch` capture on the same fixture places the entry
correctly.

## Root cause

`src/native/capture.rs` has two near-duplicate "create a new Pomodoro entry" routines
that pick the insertion point differently:

- `insert_named_pomodoro_child_block` (non-start `#name` creation) inserts after the
  current/last-completed anchor's block, **otherwise at the start of the first open
  top-level ledger entry**, otherwise at `section.start`. This is the documented order
  in `docs/capture.md` ("otherwise before the first Pomodoro in the section").
- `create_started_pomodoro_entry` (used by `plan_pomodoro_start` for both the named-new
  and unnamed-created `=<X>` cases) inserts after the last completed entry's block,
  otherwise **directly at `line_start(lines, section.start)`**. It dropped the "before
  the first open Pomodoro" fallback.

`pomodoro::pomodoros_section_range` returns `section.start` = the line right after the
heading. In the real template that line is blank, so the fallback inserts the new entry
above the blank line. When completed Pomodoros exist the bug is masked, because the
insertion point is the end of the last completed entry's block.

## Implementation

1. In `src/native/capture.rs`, add one small shared helper that computes the byte offset
   for a newly created Pomodoro entry, e.g.
   `new_pomodoro_insertion_index(lines, section, anchor: Option<usize>, first_open: Option<usize>) -> usize`:
   `task_block_end(lines, anchor)` when there is an anchor, otherwise
   `line_start(lines, first_open)` when there is a first open entry, otherwise
   `line_start(lines, section.start)`. Give it a short doc comment stating that order
   (after the anchor's complete block, otherwise before the first open Pomodoro so blank
   lines after the heading stay put, otherwise the top of the section).
2. Use the helper in `insert_named_pomodoro_child_block` with
   `anchor = timed.first().or(completed.last())` and `first_open = open.first()`. This
   must be behavior-preserving for the non-start path.
3. Use the helper in `create_started_pomodoro_entry` with `anchor = completed.last()`.
   Extend its existing section loop (which already skips fenced and indented lines) to
   also record the first top-level `pomodoro::open_ledger_task` line index as
   `first_open`, the same set of lines `insert_pomodoro_child_block` puts in `open`. An
   open _timed_ entry can't exist here because `plan_pomodoro_start` rejects it before
   calling this function, so no timed anchor is needed.
4. Leave indentation selection, the entry/child text, line-ending handling
   (`insertion_text_preserving_line_endings`), and the returned `created_line`
   unchanged. `created_line` (and so JSON `pomodoro_start.pomodoro_line` and the human
   output) is already derived from the insertion offset, so it will point at the
   corrected line automatically.
5. In `docs/capture.md` "Starting the session atomically", add one sentence saying that
   a newly created started entry uses the same placement as named creation: after the
   last completed Pomodoro's complete block, otherwise before the first Pomodoro in the
   section. No README change is needed unless it describes placement. No bob-mac-capture
   change is needed, because Mac Capture delegates vault mutation and preview to `bob`.

## Tests

Add a regression test to `tests/cli.rs` next to the existing `capture_pomodoro_start_*`
tests (same `TempDir`/`write_file`/`bob_command` helpers, `BOB_DAY_FILE`, and
`BOB_NOW="2026-09-27 07:12:00"`). Use a template-shaped daily note with a blank line
after the heading, tab-indented children, and a following section, and assert **exact**
whole-file contents:

- **Named-new start**: day
  `"# Day\n\n## Pomodoros\n\n- [ ] () — GTD\n\t- [[#^gtd]]\n\n## Tasks\n"`, marker
  `@sase:relaunch#relaunch=10` → day becomes
  `"# Day\n\n## Pomodoros\n\n- [ ] (**0715-0805** [t:: 50m]) — RELAUNCH\n\t- [[sase#^relaunch]]\n- [ ] () — GTD\n\t- [[#^gtd]]\n\n## Tasks\n"`,
  with `pomodoro_start.created_pomodoro == true` and
  `pomodoro_start.pomodoro_line == 5`.
- **Unnamed start with no placeholder**: day
  `"# Day\n\n## Pomodoros\n\n- [ ] Review inbox\n\t- [[#^gtd]]\n\n## Tasks\n"`, marker
  `@sase:u2=` → the `- [ ] (**0715-0740** [t:: 25m])` entry and its `\t- [[sase#^u2]]`
  child sit after the blank line and directly before `- [ ] Review inbox`, with
  `pomodoro_line == 5`.
- **CRLF**: the named-new case with `\r\n` endings keeps `\r\n` on every line and gives
  the same placement.
- **Parity guard**: the non-start `@sase:n1#relaunch` capture on the named-new fixture
  still yields the same placement (a `- [ ] () — RELAUNCH` entry after the blank line,
  before GTD). This guards the refactor of `insert_named_pomodoro_child_block`.

Existing tests that must keep passing unchanged include
`capture_pomodoro_start_creates_unnamed_when_no_placeholder` (completed anchor, no blank
line) and its empty-section case (`"## Pomodoros\n"`), the named existing/new start
tests, dry-run/batch rollback, and all non-start named-creation tests.

## Verification

- `cargo test --test cli capture_pomodoro_start` (new and existing start tests).
- `cargo test` (full suite) plus `cargo clippy --all-targets --all-features`. Run
  `cargo fmt` only on touched code, because repo-wide rustfmt drift is a known,
  separately tracked issue. Don't reformat unrelated files.
- Manual repro with the built binary: run
  `bob capture -b <tmp vault> -f json "Relaunch" "@sase:relaunch#relaunch=10"` with
  `BOB_DAY_FILE` pointing at the template-shaped fixture and
  `BOB_NOW="2026-09-27 07:12:00"`. Confirm the entry follows the blank line and no blank
  line separates it from `() — GTD`.
