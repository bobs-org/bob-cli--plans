---
tier: tale
title: A started Pomodoro moves ahead of every planned Pomodoro
goal:
  Every `=<X>` capture start (new-task `@route:id[#name]=<X>` and solo
  `@`/`^route:id[#name]=<X>`) leaves the running Pomodoro, with its child block,
  directly after the last completed Pomodoro and above every open planned Pomodoro, with
  correct post-image line reporting.
size: medium
proposed_by: bbugyi200.apollo.2e
create_time: 2026-09-27 13:31:17
status: wip
---

# Plan: A started Pomodoro moves ahead of every planned Pomodoro

## Problem

`bob capture '^sase_goals:research=5'` started the queued `GOALS` placeholder but left
it where it was. In today's ledger, it stayed below the planned `() — SUBTABS` and
`() — GTD` placeholders. The ledger looked like this afterward:

```markdown
## Pomodoros

- [x] (**0715-0805** [t:: 50m]) — RELAUNCH
  - [[sase#^relaunch-260927]]
- [x] (**1010-1030** [t:: 20m]) — SUBTABS
  - 🍅 [[sase#^agents-sub-tabs]]
    - Started 0t4!
- [ ] () — SUBTABS
  - [[sase#^agents-sub-tabs]]
- [ ] () — GTD
  - [[#^gtd]]
- [ ] (**1315-1340** [t:: 25m]) — GOALS <- started, but not moved
  - [[sase_goals#^research]]
- [ ] () — SASE
  - [[sase#^recovery-panel]]
```

Expected: the running `GOALS` entry, with its child block, sits right after the
completed `SUBTABS` block and above every open, planned Pomodoro. All other entries keep
their relative order.

## Root cause

Every `=<X>` start path that picks an **existing** entry calls
`replace_placeholder_range` on that entry's line and never moves it. Those paths are in
`src/native/capture.rs`:

- `plan_pomodoro_start` (new-task `<text> @route:id[#name]=<X>`, and solo links with no
  queued Task Link):
  - the `NamedSelection::Found` branch;
  - the "first open untimed placeholder" branch.
- `plan_pomodoro_link_with_start` (solo `@route:id…=<X>` / `^route:id…=<X>`):
  - the named-`Found` branch where _Q_ (the entry that already holds the task's Task
    Link) is the destination;
  - the named-`Found` branch where _Q_ is not the destination (the link subtree moves);
  - the no-name branch that starts _Q_. This is the path in the screenshot. The task was
    already queued under `GOALS`, so _Q_ = `GOALS`, and it was started where it sat.

Only the entry-**creation** paths place the started entry correctly. These are
`create_started_pomodoro_entry` (fixed in b068045) and the queued `#new=<X>` branch that
uses `capture_task_toggle::insert_named_placeholder`. Both insert at the "current slot":
after the last completed Pomodoro's complete block, otherwise before the first open
Pomodoro, otherwise at the top of the section.

The in-place behavior was deliberate. It is choice 4 of the approved epic plan
`plan:202609/active_task_link.md` (bead `bob-cli-28`): "Starting happens in place and
never reorders the ledger. This matches `#name=` and the Obsidian `se<X>` snippet." That
premise is wrong for capture. The bob-ledger-tools `se<X>` snippet is only a text
expansion at the cursor, so in Obsidian the user has already put the line where they
want it. Capture has no cursor. There, "start" must also mean "make this the current
session". **This plan supersedes that choice for every `=<X>` start path.**

## Required behavior

After any successful `=<X>` start, the started entry sits in the **current slot**:

- The line plus its complete child block (the `task_block_end` extent) is placed right
  after the complete block of the last completed (`[x]`/`[X]`) top-level, non-fenced
  Pomodoro in the section.
- If there is no completed Pomodoro, it goes right before the first other open Pomodoro.
- This is exactly the placement newly created started entries already use.

Details:

- **Already in the slot:** the entry needs no move when both of these hold:
  - no completed entry comes after it;
  - no other open entry sits between the last completed entry (or the section start) and
    it.

  In that case the only change is the time range. Blank lines or prose between the
  anchor and the entry must not be shuffled.

- **What moves:** only the started entry's block. Its name, checkbox, children, interior
  blank lines (blank lines followed by indented lines), and line endings are preserved.
  Blank lines outside the block stay put. Every other entry keeps its relative order.
- **Not anchors:** cancelled `[-]` entries, nested lines, and fenced lookalikes. This is
  the same predicate `create_started_pomodoro_entry` uses today:
  `pomodoro::completed_ledger_task` / `pomodoro::open_ledger_task` on top-level,
  non-fenced lines.
- **Interleaved ledgers** (an open placeholder above a completed one) follow chronology.
  The started entry goes after the last completed block, the same as creation does.
- **Line endings:** CRLF and a missing final newline are preserved.
  - If the moved block used to end the file with no final newline, the line that now
    ends the file loses its line ending, and the moved block's last line gets the
    document's ending.
  - If the slot is the end of a file with no final newline, add the ending before the
    moved block and leave its last line unterminated.
- **Every start path does this:** new-task `=`, new-task `#name=`, and solo `@`/`^` with
  or without `#name` and with or without _Q_. Created entries already land in the slot;
  their output must stay byte-identical to today's.
- **Link-only captures are unchanged.** Captures without `=<X>`, Ensure Next (`+`), `!`,
  `+N`/`-N` adjustment, and task-status-hooks never reorder the ledger.
- **Reporting:**
  - `pomodoro_start.pomodoro_line`, `pomodoro_link_destination.line`, and the human
    `… at line N` use the **post-image** line of the moved entry.
  - `pomodoro_link_source` stays the pre-image endpoint.
  - `pomodoro_link_action` keeps its meaning. It is `already_current` when _Q_ is the
    destination, even though _Q_ moved.
  - Any `appended`/`inserted` Task Link placement describes the final post-image.
  - No new JSON keys or human phrasing, so Bob Mac Capture (which only renders these
    fields) needs no change.
- **Atomicity:** dry run, batches, and rollback behave as today. A later `+N` in the
  same batch adjusts the moved entry.

## Implementation (`src/native/capture.rs`)

1. **Share one placement rule.** Factor the section scan out of
   `create_started_pomodoro_entry` into a small helper that
   `create_started_pomodoro_entry` and the new mover both use. The scan covers
   top-level, non-fenced lines, collecting completed indices and open indices. Keep
   `new_pomodoro_insertion_index` as the single rule for insertion offsets. Creation
   output must not change.
2. **Add a pure mover.** Add
   `move_started_pomodoro_to_current_slot(contents: &str, entry_index: usize) -> Result<(String, usize), CaptureError>`.
   It returns the updated text and the entry's new 0-based line index, and implements
   the rules above:
   - Compute the entry's block with `task_block_end`.
   - Detect the already-in-slot case and return the input unchanged.
   - Otherwise cut the block and splice it in at the slot. Compute the slot on the
     original text; it can never fall inside the entry's block.
   - Handle CRLF and a missing final newline. `capture_task_toggle`'s
     `insert_lines_after` shows the EOF-without-newline pattern.
3. **Add one start primitive.** Add
   `start_existing_pomodoro_entry(contents, index, time_range) -> Result<(String, usize), CaptureError>`.
   It calls `replace_placeholder_range` and then the mover. Use it instead of the direct
   `replace_placeholder_range` call in each existing-entry path listed under Root cause,
   then run the follow-up steps against the **returned** index:
   - `plan_pomodoro_start`: append the new Task Link under the moved entry with
     `append_pomodoro_child_link`. Set `PomodoroStartSummary.pomodoro_line` to the new
     index + 1, and read the resolved name at the new index.
   - `plan_pomodoro_link_with_start`, _Q_ ≠ named destination: start and move the
     destination first. Then rescan the started text for the single movable link and
     call `move_subtree_to_entry` into the moved destination index.
   - `plan_pomodoro_link_with_start`, _Q_ = destination (named or no-name): stage the
     started-and-moved text. Build `pomodoro_link_destination` and
     `summary.pomodoro_line` from the post-image entry at the moved index, not from
     `q_entry_index` or `dest_index`.
   - The queued `#new=<X>` branch and `create_started_pomodoro_entry` may route through
     the primitive for uniformity, where moving is a no-op. Their bytes must stay
     identical.
4. **Update help and docs**, keeping the wording short:
   - `bob capture --help` long text in `src/native/capture.rs`:
     - The new-task `=<X>` paragraph ("The start replaces the selected untimed '()'
       placeholder …") gains a sentence saying the started entry moves ahead of every
       open Pomodoro, right after the last completed one.
     - The solo-link paragraph's "'=<X>' starts that entry in place" becomes "starts
       that entry and moves it to the front of the queue".
   - `docs/capture.md`:
     - "Starting the session atomically": extend the "A newly created started entry uses
       the same placement …" sentence so that starting an existing placeholder moves it,
       with its child block, to that same slot.
     - "Linking and starting existing tasks": replace "no-name with _Q_ starts _Q_ in
       place" and "The started entry keeps its position, and the ledger is never
       reordered." with the current-slot rule.
     - The worked-example rows stay valid, because `BUGS` already sits right after
       `PLAN`.
   - `README.md`: add one sentence to the `=<X>` paragraph.

## Tests

Add new `#[test]` functions. Do **not** edit the existing
`capture_pomodoro_link_solo_grammar_and_atomic_execution` body, which the `bob-cli-28`
land agent may still be editing. All existing `capture_pomodoro_start_*`, solo-link, and
adjust tests must pass unchanged.

**Unit tests** for the mover, in the `#[cfg(test)]` module of `src/native/capture.rs`:

- already in the slot returns the input unchanged, including with a blank line between
  the anchor and the entry;
- moving above open entries when there is no completed entry;
- moving after a completed anchor that has nested grandchildren;
- an interleaved `[x] A`, `() X`, `[x] B`, `() Y` ledger, where X moves after B's block;
- `[-]`, fenced, and nested lookalikes are not anchors;
- an interior blank line moves with the block, and trailing blank lines stay;
- CRLF;
- a block that ended the file with no final newline, moved up;
- a block moved down to the end of a file with no final newline.

**Integration tests** in `tests/cli.rs`, asserting exact day-file bytes plus JSON lines.
Use `write_toggle_task_settings`, `BOB_DAY_FILE`, and `BOB_NOW`:

1. **Screenshot repro.**
   - Setup: the ledger above, before the start. `GOALS` is a `() — GOALS` placeholder at
     line 11, with its link at line 12 under a `## Pomodoros` first line and tab
     indentation. `sase_goals.md` contains `- [*] #task Research goals ^research`. Use
     `BOB_NOW=2026-09-27 13:12:00`.
   - Run `^sase_goals:research=5`. It yields `- [ ] (**1315-1340** [t:: 25m]) — GOALS`
     plus its link as lines 7–8, right after `\t\t- Started 0t4!`. `SUBTABS`, `GTD`, and
     `SASE` follow in their original order.
   - JSON: `pomodoro_link_action` is `already_current`, `pomodoro_link_source.line` is
     11, and both `pomodoro_link_destination.line` and `pomodoro_start.pomodoro_line`
     are 7.
   - Repeat with `@sase_goals:research=5` and `^sase_goals:research#goals=5`. The bytes
     must be identical.
   - Run `-d` (dry run): it reports the same lines and leaves both files unchanged.
2. **Solo named, _Q_ ≠ destination.** The link, with a `- notes` child, is queued under
   `GTD`. `^sase_goals:research#goals=` starts `GOALS`, moves it to the slot, and moves
   the link subtree from `GTD` into it. `pomodoro_link_action` is `moved`, the source is
   pre-image, and the destination is post-image.
3. **New-task named existing.** Ledger `- [ ] () — BUGS\n- [ ] () — FOCUS\n` with no
   completed entry. `Work @sase:w1#focus=` puts `FOCUS` first, with the new link under
   it. `pomodoro_start.pomodoro_line` is 2.
4. **New-task unnamed.** A completed `DONE`, then an open non-placeholder
   `- [ ] Review inbox`, then `- [ ] () — GTD`. `Work @sase:w2=` starts `GTD` and places
   it between `DONE` and `Review inbox`. Include a variant with no completed entry,
   where `GTD` moves before `Review inbox`.
5. **Already in the slot.** A completed entry, a blank line, then the queued
   placeholder. Only the time range changes, and the blank line is untouched.
6. **CRLF plus no final newline.** The queued placeholder block ends the file with no
   final newline. After the start and move, every line ending is CRLF and the file still
   has no final newline.
7. **Batch.** `printf '^sase_goals:research=5\n\n+2\n'` over the repro ledger ends with
   the moved `GOALS` at `(**1315-1350** [t:: 35m])`, still right after the completed
   `SUBTABS` block.

## Validation

Run `just all`, which runs `cargo fmt --check`,
`cargo clippy --all-targets --all-features`, and `cargo test`. Format the files you
touch with `cargo fmt`.

## Coordination and non-goals

- Before committing, add a note on epic bead `bob-cli-28`, reading the `sase_beads`
  reference memory first. The note records that "start in place" (deliberate choice 4)
  is superseded: every `=<X>` start now moves the started entry to the current slot. Do
  not change the bead's status.
- Non-goals:
  - changing the Obsidian `se<X>` snippet (bob-plugins);
  - changing Bob Mac Capture;
  - changing link-only placement, Ensure Next, `!`, `+N`/`-N`, or task-status-hooks;
  - new CLI subcommands or options.
