---
tier: tale
size: small
title:
  Land bob-cli-fk — fix the agenda buffer_part compile break, retire test-only conflict
  wrappers, correct two gcal docs lines, and close the epic
goal:
  bob-cli master compiles and passes just check with event buffers intact, the gcal
  buffer docs are accurate, and epic bob-cli-fk is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-fk.land
proposed_with: claude/opus
bead: bob-cli-fk
create_time: 2026-11-02 00:50:43
status: wip
---

- **PARENT:**
  [202611/gcal_event_buffers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gcal_event_buffers.md)
- **BEAD:**
  [bob-cli-fk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fk/README.md)

# Land bob-cli-fk: finish the event-buffer epic and close it

## Why this exists

Epic `bob-cli-fk` ("Pre/post-event buffers for calendar events",
`plan:202611/gcal_event_buffers.md`) has all six phases closed. The landing review found
that **bob-cli master does not compile**. Two phases landed in parallel and their
changes don't fit together:

- `a4044f40` (phase `bob-cli-fk.3`, capture-buffers) deleted `BufferPart::as_str` from
  `src/native/event_task.rs`, made `BufferPart` derive `serde::Serialize` with
  `#[serde(rename_all = "lowercase")]`, and moved production run-into-event detection
  onto `event_span_conflicts`.
- `1b796cd7` (phase `bob-cli-fk.4`, agenda-buffers) still calls `part.as_str()` at
  `src/native/capture_pomodoros_agenda.rs` (in the agenda conflict builder,
  `buffer_part: conflict.buffer_part.map(|part| part.as_str().to_string())`). It also
  kept the now test-only `first_event_start_crossed` alive with an
  `#[allow(dead_code)]`.

The landing reviewer applied the one-line type fix in Step 1 below as a scratch change.
`just check` then passed in full: fmt, the gated clippy, lib 4406 passed, cli 2767
passed, and every other test binary green. The JSON golden
`tests/fixtures/capture_pomodoros/agenda-events-buffered.json` was unchanged.

The review also found two inaccurate lines in `docs/gcal.md`. One comes from integrating
with docs commits `42dd9453` and `8c0dce81`. They were written before gcal sync used
buffers, so they dropped buffers from the directive-line example. The other is a phase-2
misstatement.

Everything else is verified, so this tale does not redo it:

- the cross-repo vectors (`event_buffer`, `meet_event_run`, `event_task_close`) are
  byte-identical in bob-plugins and pinned to bob-cli commits `2e49a561` and `a4044f40`,
  both on master;
- bob-mac-capture's `Tests/Fixtures/agenda-events-buffered.json` is byte-identical to
  bob-cli's;
- the bob-plugins buffer tests pass (58/58);
- `sase bead epic-symbols bob-cli-fk` reports no entries;
- follow-up triage is already recorded on the epic (new tasks `bob-cli-fo`,
  `bob-cli-fp`, and `bob-cli-fq`).

## Steps

### 1. Fix the compile break (bob-cli)

In `src/native/capture_pomodoros_agenda.rs`:

- In `struct AgendaEventConflict`, change `pub(crate) buffer_part: Option<String>,` to
  `pub(crate) buffer_part: Option<event_task::BufferPart>,`. Keep its
  `#[serde(skip_serializing_if = "Option::is_none")]`. The module already imports
  `event_task`.
- Where the conflict is built, replace
  `buffer_part: conflict.buffer_part.map(|part| part.as_str().to_string()),` with
  `buffer_part: conflict.buffer_part,`.

`BufferPart` serializes as `"before"` / `"after"`, so the agenda JSON stays
byte-identical. Do **not** reintroduce `BufferPart::as_str`. Phase 3 deliberately
switched the shared vectors test to serde.

### 2. Retire the test-only conflict wrappers (bob-cli)

In `src/native/event_task.rs`:

- Delete `pub(crate) fn event_starts_crossed` and
  `pub(crate) fn first_event_start_crossed`, together with their doc comments.
  Production code no longer calls either. Capture and the agenda call
  `event_span_conflicts` directly, and the only remaining caller is one unit test.
  Confirm with `grep -rn "event_starts_crossed\|first_event_start_crossed" src tests`.
- Rewrite unit test `first_event_start_crossed_wraps_overnight_and_skips_exclusions`:
  - rename it `event_span_conflicts_wraps_overnight_and_skips_exclusions`;
  - make each lookup
    `event_span_conflicts(start, end, None, &events, &exclude).into_iter().next().map(|(event, _)| event)`,
    optionally through a small local closure;
  - keep all three assertions: overnight 2300→0100 finds `late`; excluding `late` finds
    `after`; an empty 1200→1200 session finds nothing.
- Remove the stale `#[allow(dead_code)]` (and its comment) on `EventBuffer::field`,
  which `gcal/render.rs` now calls.
- Remove the `#[allow(dead_code)]` on `EventNoteRow::is_open_task`, which
  `event_span_conflicts` and the agenda now call.
- Leave the other `#[allow(dead_code)]` attributes that predate this epic alone:
  `EventKind`, the `EventNoteRow` struct, `as_event_task`, and `cluster_by_trackedness`.

### 3. Correct `docs/gcal.md` (bob-cli)

- In "## Untracked events", the directive-line paragraph currently reads: "With a
  resolvable `love` root, `@love<br>#untracked` sets `[root:: love]` and marks the Event
  Task untracked; the task stays in the daily note." Change it to show buffers again:
  "With a resolvable `love` root, `@love<br>#untracked -30` sets `[root:: love]`, marks
  the Event Task untracked, and reserves a 30-minute [buffer](#buffers) before it; the
  task stays in the daily note."
- In "## Buffers", the sentence "Hand-written Event Tasks may carry the field too; sync
  reads it from the vault, never from Google." is wrong, because sync reads Google's
  description. Change it to: "Hand-written Event Tasks may carry the field too: capture
  and the `capture-pomodoros --tasks` agenda read the buffer from the vault field, while
  sync reads it from the Google description."
- Keep the line wrapping consistent with the surrounding prose.

### 4. Verify (bob-cli)

- `cargo fmt --all` on the touched files, then `just check`. It must pass, and
  `agenda_events_buffered_golden` plus the buffered event tests must stay green.
- `just docs-check`. The only failure allowed is the pre-existing
  `tools/docs/tests/test_capture_token_coverage.py::test_every_capture_token_has_a_literal_query_at_its_heading`,
  which comes from docs commit `42dd9453` and is tracked by task bead `bob-cli-fo`. Any
  other failure is new and must be fixed.
- Do not run `just check-full`.

### 5. Close out epic bob-cli-fk (the final step of this tale)

1. Run `sase bead epic-symbols bob-cli-fk`. It is expected to report none. If any entry
   appears, resolve it per the Symvision epic-whitelist policy: wire it up, privatize
   it, add a non-test pragma, or delete it. Do not re-key an entry, because no open bead
   under this epic needs one.
2. Close the epic. Never use `--force`; if the close is rejected, fix the cause and
   close again.

   ```bash
   sase bead close bob-cli-fk --note "Landing verified all six phases against source and commits 2e49a561, a4044f40, b86fb2d0, 1b796cd7 (bob-cli), 13e63d4 (bob-plugins), 4f35460 (bob-mac-capture). Epic-caused fix: fk.3 removed BufferPart::as_str while fk.4 called it, breaking the bob-cli build; AgendaEventConflict.buffer_part now holds Option<BufferPart> (serde lowercase, JSON byte-identical). Retired the test-only event_starts_crossed/first_event_start_crossed wrappers and two stale dead_code allows. Integration with docs commits 42dd9453/8c0dce81: restored the buffer in docs/gcal.md's directive-line example and corrected the Buffers section (sync reads Google, capture/agenda read the vault field). Verified: just check green; just docs-check green except bob-cli-fo; event_buffer/meet_event_run/event_task_close vectors byte-identical in bob-plugins and pinned to master commits; bob-mac-capture agenda-events-buffered fixture byte-identical; plugin buffer tests 58/58; memory record decisions:event-buffers-widen-the-span present with synced-meetings-link-by-phase superseded in part; glossary:event-task untouched per decision. macOS: Mac offline; macOS CI Build on 4f35460 fails only on non-epic errors (bob-cli-fj, bob-cli-fp) with no CaptureAgendaView.swift errors. Follow-up triage is recorded in the epic notes (bob-cli-fo, bob-cli-fp, bob-cli-fq filed; the rest declined as resolved or already tracked). No epic symbols."
   ```

   Adjust the note only if verification results differ, for example if docs-check shows
   another failure that you fixed.

3. Run `just symvision` if the recipe exists. As of master `37e68c09` it does not, so
   state that it was unavailable.
4. Set `status: done` (it is currently `status: wip`) in the frontmatter of the epic's
   plan file. That is the PLAN path printed by
   `sase bead read bob-cli-fk -r "Need the plan path to mark it done"`
   (`plan:202611/gcal_event_buffers.md`, in the plans sidecar repo). Commit that sidecar
   change through the normal final-declaration flow along with the bob-cli changes.
5. Epic `bob-cli-fk` has no `parent_bead`, so nothing further cascades. Finish normally.
