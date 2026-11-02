---
tier: tale
title: "`=` starts the MEET in progress instead of the first future Pomodoro"
goal:
  An unnamed `=` during (or right at the start of) a calendar event starts the open MEET
  with its calendar span, and explicit non-MEET starts mid-meeting warn in bob and in
  Bob Mac Capture.
size: medium
decisions:
  decision_record:
    ask:
      Should the implementer add a decisions record for "an unnamed `=` joins the
      meeting in progress"?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.171
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 09:01:14
status: wip
---

# Plan: `=` starts the MEET in progress instead of the first future Pomodoro

## Problem

In Bob Mac Capture, typing `=` at 08:04 on 2026-11-02 previewed
`Start BUILD 0805-0830 (25m)`. Swings! (`**0800-0815**`, `#offclock`) was running, and
its link sat under the open `- [ ] () — MEET` placeholder. `=` should have started that
MEET with its calendar span, `(**0800-0815** [off:: 15m])`, exactly as `=meet` already
does.

Root cause: in `src/native/capture/pomodoro_start.rs`, `plan_pomodoro_start_item`
handles an unnamed start with no running session (no open timed entry) by always
choosing `capture_pomodoros::next_future_pomodoro(&scan)`, the first open untimed
placeholder in ledger order. The calendar MEET rule (`apply_meet_rule` with
`SpanRule::Calendar`) only applies when that first placeholder already happens to be
MEET. Also, an explicit non-MEET start in the middle of a meeting (`=build` at 08:04)
raises no warning. `event_task::span_conflict` deliberately skips a meeting that is
already in progress at the session start (`meeting_in_progress_at`: `start < S < end`),
so `pomodoro_runs_past_event` never fires for it.

Bob Mac Capture is a thin client (`decisions:mac-capture-is-a-thin-client`). Its preview
card and the `Would start …` footer already render whatever `pomodoro_start` bob
returns. The new target therefore shows up in the app with no Swift change. Only the new
warning needs a small Swift change.

## Behavior to build

Definitions, all in minutes on today's timeline:

- **S** is the start minute the typed start writes. It is computed by
  `compute_pomodoro_start_range`: now minus `5·offset`, rounded up to 5 minutes. Use the
  unwrapped timeline value, not the `HHMM` modulo 1440.
- The **start block** is `[S, S + 5)`.
- A MEET placeholder's **event run** is
  `meet_event_run::compute_event_run(..., now_min)` on the staged contents. It uses
  buffered spans, as `=meet` does (`decisions:event-buffers-widen-the-span`).

### 1. Unnamed starts pick the meeting in progress

This applies to every unnamed start that reaches the "no running session" branch: `=`,
`=3`, `=-2`, `=~2`, and the `=` after `x` in an `x =` chain. If at least one open,
untimed MEET placeholder has an event run that overlaps the start block
(`run.start_min < S + 5 && run.end_min > S`), start that MEET instead of
`next_future_pomodoro`.

- If several MEET placeholders qualify, take the earliest `run.start_min`. Break ties by
  ledger line.
- Everything after target selection stays the same code path: the drop plan (`~K`
  numbers are the MEET lineup's), `start_existing_pomodoro_entry`, and
  `apply_meet_rule(SpanRule::Calendar)`.
- An empty suffix therefore writes the run's span with the late-join clamp. A counted
  suffix keeps its explicit span and takes the run's clock key. Both match the existing
  `=meet` / `=3#meet` behavior.
- When no MEET qualifies, behavior is unchanged: start the first future placeholder.
- Out of scope:
  - Named starts (`=build`) always start what they name.
  - A bare `=` while a session runs still restarts that session.
  - Link and task starts (`@route:id=`) keep their own placement.
  - Obsidian `se<X>` is untouched, because the user chooses the line there.

Worked examples:

- Screenshot case: Swings 0800-0815 `#offclock` is in MEET's lineup, it is 08:04, and
  the closed BUILD ends at 0800. The result is `(**0800-0815** [off:: 15m]) — MEET`. The
  entry moves to the current slot, and `event_run.source` is `calendar`.
- At 07:57 (S = 0800) with an event at 0800, the start block is the event's first block,
  so MEET starts at 0800.
- At 07:54 (S = 0755), MEET does not qualify. BUILD starts 0755-0820, and the existing
  `pomodoro_runs_past_event` warning fires, unchanged.
- At 08:12 (S = 0815) with a 0800-0815 event, the run ends at S. MEET does not qualify,
  and BUILD starts with no warning.

### 2. Non-MEET starts in the middle of a meeting warn

Add a structured warning `pomodoro_starts_during_event`, plus an additive
`pomodoro_start.event_in_progress` object. They fire when a start-family write leaves a
non-MEET session whose written start S is strictly inside an open Event Task's meeting
span. Use the existing `meeting_in_progress_at` test: the meeting span, not the buffered
one, overnight-aware.

- Exclude events that are already in that session's own lineup (`lineup_exclude_ids`).
- Only open Event Tasks count. Done (`x`) and cancelled (`-`) rows never fire.
- MEET sessions never fire.
- This partitions cleanly with `pomodoro_runs_past_event`. Starting at the meeting
  start, or inside a lead-in or lead-out, is already a conflict, while strictly
  mid-meeting is exactly what `span_conflict` skips. Both warnings can fire for one
  start only when they name different events.
- Hook it into `attach_start_event_conflict` (`src/native/capture/event_conflict.rs`).
  Every start path calls that function: plain start, restart, named start, swap, and the
  link starts in `pomodoro_link.rs`. Do not apply an "only new" rule.
  - A kept-ledger swap that keeps a span starting at the meeting start stays quiet. That
    falls out naturally, because the start is not strictly inside the meeting.
- Do not change `event_task::event_span_conflicts`, `span_conflict`, or
  `tests/fixtures/event_buffer/vectors.json`. They are a shared contract with
  bob-plugins. Add a small separate predicate in `src/native/event_task.rs`, for example
  `events_in_progress_at(start, events, exclude_ids)`, that reuses
  `meeting_in_progress_at`.
- JSON, omitted when nothing fires:

  ```json
  "event_in_progress": {
    "event": {"title": "Swings!", "time_range": "0800-0815", "line": 27,
              "off_clock": true},
    "more": 1
  }
  ```

  - `event` is the earliest qualifying event by meeting start, then line.
  - `time_range` is the meeting span.
  - `buffer` (`EventBufferJson`) is present only when the event has one.
  - `more` is omitted when 0.
  - Do not add the deprecated `untracked` key to this new object.

- Message, built like `event_conflict_message`:
  `⚠ BUILD 0805-0830 starts during Swings! (0800-0815)`.
  - A buffered event uses `event_with_buffer_label`: `Lunch (1200-1300 −30m +15m)`.
  - Append ` and N more` when `more > 0`.
  - The name falls back to `session`.
- Harvest it per item next to `warning_from_item` in `src/native/capture/plan.rs:181`.
  Add a sibling harvest so that an item can report both warnings.

### 3. Completion "Next up" agrees with `=`

`bob capture-complete`'s start-name rows mark `next_up` for "a planned placeholder that
a bare `=` would start" (`capture_complete/cli.rs` help text). Currently
`today_start_rows` uses `next_future_pomodoro`. When the daily note has no open timed
entry, compute the target with the same helper as §1:

- use `env::current_datetime()` and a bare `=` spec;
- get the day-relative path the same way capture does (`close_day_relative` or
  equivalent);
- use the contents `read_daily` already returns.

Thread the resulting line into `today_start_rows` and `theme_menu::compose` (and its
callers and tests). Keep the old behavior when a session runs. Leave the other
`next_future_pomodoro` callers alone: split, merge, close, adjust, and the idle agenda's
`AgendaRole::Next`.

## Implementation steps

### bob-cli

1. **Shared target helper**
   - In `src/native/capture/meet_event_run.rs`, add a helper that returns the open
     untimed MEET placeholder whose run overlaps the start block, with its run. Use
     `is_meet_name`, `entry.placeholder`, and `time_range.is_none()`.
   - Add a wrapper `unnamed_start_target(...)` that falls back to
     `next_future_pomodoro`.
   - Expose it `pub(crate)` through a re-export in `src/native/capture/mod.rs` so
     `capture_complete` can call it. Prefer returning the entry or its line over
     widening `EventRun`.
   - Factor S out of `compute_pomodoro_start_range`, for example
     `start_block_minute(now, spec) -> i64`, so the rounding has one owner.
2. **Use the helper**
   - Call it in `plan_pomodoro_start_item` in place of `next_future_pomodoro`.
   - Keep the existing "no future Pomodoro" error when both lookups find nothing.
3. **Add the in-progress predicate and JSON**
   - Add the predicate to `src/native/event_task.rs`.
   - Add the JSON structs and message to `event_conflict.rs`.
   - Add `event_in_progress: Option<…>` (`skip_serializing_if` None) to
     `PomodoroStartSummary` (`src/native/capture/batch.rs`). Initialize it to `None` at
     all 16 construction sites in `batch.rs`, `pomodoro_start.rs`, and
     `pomodoro_link.rs`.
   - Fill it in `attach_start_event_conflict`.
4. **Harvest the warning** in `plan.rs`, and update the `event_conflict.rs` module doc.
   It currently says "A session starting mid-meeting stays quiet".
5. **Completion `next_up`**, per §3.
6. **CLI help**: in `src/native/capture/cli.rs` (~line 262), change "start today's next
   future Pomodoro now" so it also says that while an open MEET's event run occupies the
   start block, the unnamed start takes that MEET. Then run `just docs-gen` to
   regenerate `docs/reference/cli/capture.md`.
7. **Docs** (behavior contract; read `sase memory read docs.md` first):
   - `docs/capture.md`:
     - the lifecycle table `=` row and the mnemonic line under it (~3130);
     - "Starting the next Pomodoro" (~3115): one paragraph on the meeting default;
     - "MEET starts follow the calendar" (~3213): a short
       `#### An unnamed start joins the meeting in progress` subsection with the
       start-block rule and the worked examples above; update the table row
       `` `=` or `=meet` (empty suffix) on a MEET ``;
     - "Running into an event" (~3865): replace "Starting mid-meeting stays quiet,
       because…" with the new `pomodoro_starts_during_event` rule and the JSON block.
   - `docs/reference/json-outputs.md`, `bob capture` row: add
     `pomodoro_start.event_in_progress` and warning `pomodoro_starts_during_event`.
   - Do not edit `CHANGELOG.md` or the `Cargo.toml` version
     (`decisions:bob-cli-releases-ride-a-daily-train`). Use a `feat(capture):` commit.
8. **Tests**
   - Unit tests in `meet_event_run.rs`:
     - block-overlap edges: 07:54, 07:57, 08:04, 08:12;
     - lead-in and lead-out of a buffered event;
     - an ended or done event;
     - two MEET placeholders;
     - no MEET placeholder.
   - Unit tests in `event_task.rs` for the in-progress predicate, including an overnight
     meeting and the exclude set.
   - A new `tests/cli/capture/meet_event_default.rs`, registered in
     `tests/cli/capture/mod.rs`, run through `bob capture -f json` with `BOB_DAY_FILE`
     and `BOB_NOW`. Assert exact day bytes:
     - the screenshot regression: Swings `[*]` 0800-0815 `#offclock`; Lincoln Park
       1900-2000 `#offclock` `[buffer:: -30m]`; closed BUILD 0755-0800; then
       `() — BUILD` (with a task link), `() — MEET` (links lincoln and swings), and
       `() — PLAN`; now 08:04; `=` gives MEET `0800-0815 [off:: 15m]` with
       `event_run.events` = Swings only and no new warning;
     - `=3` gives MEET `0805-0820` with source `explicit`;
     - `=build` starts BUILD with `event_in_progress` and the exact warning message;
     - `=meet` is unchanged;
     - with no MEET placeholder, `=` starts BUILD and warns;
     - an event in BUILD's own lineup gives no warning;
     - a restart of a running BUILD mid-meeting warns and is not redirected;
     - `--dry-run` gives the same JSON with no write.
   - Update
     `tests/cli/capture/event_start_warnings.rs::start_in_a_lead_out_warns_and_a_mid_meeting_start_stays_quiet`
     (~458): rename it, keep the `event_conflict` absence assertion, and assert the new
     warning.
   - Add a `capture-complete` test, next to the existing start-name completion tests,
     showing MEET `next_up` at 08:04 and BUILD at 07:30.
   - Leave `tests/fixtures/meet_event_run/vectors.json` and `event_buffer/vectors.json`
     unchanged. bob-plugins mirrors them.
9. **Gates**: run `just fmt`, `just lint`, `just test`, and `just docs-check`.

### bob-mac-capture (linked repo; open it with `/sase_repo` and read its AGENTS.md)

10. In `Sources/CaptureCore/CaptureModels.swift`, add `isPomodoroStartsDuringEvent` next
    to `isPomodoroClockMixed`.
11. In `Sources/CaptureCore/CapturePomodoroStartPresentation.swift`, add
    `startsDuringEventWarning: String?`:
    - fill it from `capture.warnings` the same way `pomodoroClockMixedWarning` is
      filled;
    - strip a leading `⚠ `, because the banner draws its own triangle;
    - append it to the VoiceOver text.
12. In `Sources/BobMacCapture/CapturePanelView.swift` `startPreviewItem`, render it with
    the existing
    `eventWarningBanner(headline:detail:nil fit:nil accessibilityText: interpretingInlineMarkdown: true)`
    right after the `eventConflict` banner.
13. Add a case to `Tests/CaptureCoreTests/CapturePomodoroStartPresentationTests.swift`.
    Add one sentence to `docs/capture-previews.md` § Calendar-aware start previews.
14. Run `just portable-check`. The view file needs the Mac scratch-copy compile that
    AGENTS.md describes. If the Mac is offline, say in the final report that the macOS
    targets are unverified.

> [!decision] decision_record Use `/sase_memory_write`. Create the strand
> `sase/memory/decisions/unnamed-start-joins-meeting-in-progress.md` in the existing
> record format: applies to bob-cli and Bob Mac Capture; the start-block claim; why
> (Bryan's 2026-11-02 request and this plan); rejected alternatives (now-only detection,
> scanning all Event Tasks instead of MEET lineups, swapping a running session); cost;
> reopens-when. Then run `sase memory init`.

## Acceptance

- At 08:04 against the screenshot ledger, `bob capture --dry-run -f json -- =` reports
  `pomodoro_name: "MEET"`, `time_range: "(**0800-0815** [off:: 15m])"`, and the Mac card
  reads `Start MEET … 0800-0815`.
- `=build` at 08:04 still starts BUILD 0805-0830 and shows the orange
  `starts during Swings! (0800-0815)` banner.
- All existing capture, gcal, and completion tests pass, except the one intentionally
  updated test.
