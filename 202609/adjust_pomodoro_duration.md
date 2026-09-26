---
tier: epic
title: Adjust the current Pomodoro from capture with +N and -N
goal: Whole-item +N and -N captures adjust the current Pomodoro reliably in Bob CLI
  and Bob Mac Capture, with accurate previews and atomic bulk behavior.
phases:
- id: adjustment_core
  title: Parse and atomically apply Pomodoro duration adjustments
  size: medium
  depends_on: []
  description: 'adjustment_core: add exact-item signed-count grammar and a staged
    daily-ledger edit, with integration coverage for bulk capture, timing, failure,
    and rollback.'
- id: editor_contract
  title: Expose and document the adjustment contract
  size: medium
  depends_on:
  - adjustment_core
  description: 'editor_contract: expose additive parse and capture JSON semantics,
    clear human output, help, docs, and protocol tests for adjustment items.'
- id: mac_presentation
  title: Show Pomodoro adjustments in Bob Mac Capture
  size: medium
  depends_on:
  - editor_contract
  description: 'mac_presentation: decode Bob''s additive adjustment result and present
    accurate dry-run and committed before/after timing in Mac Capture, with fixtures
    and tests.'
proposed_by: bbugyi200.apollo.21
create_time: 2026-09-26 19:06:50
status: wip
bead_id: bob-cli-27
---

- **PROMPT:** [prompts/202609/adjust_pomodoro_duration.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/adjust_pomodoro_duration.md)
- **BEAD:** [bob-cli-27](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-27/README.md)

# Problem and outcome

Typing `+5` as a complete capture item should extend today's current Pomodoro by 25
minutes in one operation; `-5` should shorten it by up to 25 minutes. A
blank-line-separated draft may mix these items with ordinary captures, and all effects
remain atomic. This gives the capture panel and CLI the useful part of the Obsidian
bob-ledger-tools `\p`/`\P` keymaps without requiring a cursor in an open Obsidian
editor. No task, task link, or new Pomodoro is created by an adjustment item.

The earlier bob-cli-26 epic established five-minute units, the current timed-ledger
format, batch staging, and additive editor/JSON fields for session starts. The Obsidian
keymaps call `changePomodoroUnits` with a positive or negative Vim repeat count. That
function obtains the duration from a valid `[t:: ...]` field, then a legacy stopwatch
duration, then the time span; it adds five minutes per unit, clamps at zero, recomputes
the end from the unchanged start, and writes `[t:: Nm]`. Capture has no cursor, so it
must resolve the current session from the daily ledger rather than emulate the keymap's
cursor-line override.

# Behavior to implement

- Recognize only a whole capture item's trimmed text matching `^[+-][0-9]+$` (ASCII
  decimal magnitude). `+1` and `-1` are one five-minute unit; `+5` and `-5` are five
  units. Require a positive magnitude: `+0`/`-0` produce an actionable usage error, and
  oversized magnitudes fail checked parsing/arithmetic before any write.
  Leading/trailing whitespace is fine. A standalone sign is incomplete and fails strict
  capture with a helpful error. Text such as `Plan +5` remains ordinary prose. If a
  valid signed-count token starts an item but has extra text, a marker, or a child line,
  reject it as an invalid adjustment rather than silently create a task; the item must
  contain only the signed count.
- Treat an adjustment as its own item-local mode. A separate `@@route` declaration can
  still route ordinary items in the same draft, but must never turn `+5` into a task or
  change its destination. Reject forced destination/task/section and forced clipboard
  options on adjustment items with a specific error; `--bob-dir`, `BOB_DAY_FILE`,
  `--dry-run`, `--no-clip`, and output formatting remain usable. Do not let route,
  schedule, priority, or clipboard markers silently attach to an adjustment.
- Select exactly one open timed, column-zero entry in today's `## Pomodoros` section
  using the existing ledger scanner. A planned `()` entry, completed entry, cancelled
  entry, fenced lookalike, or child line is not a target. Missing/unreadable day file or
  section, no open timed entry, multiple open timed entries, or an unparseable target
  range fail without writing. The current entry remains eligible even after its
  displayed end time has passed, consistent with bob-cli-26's open-session rule.
- Start from the entry's valid duration metadata, falling back to its displayed range
  when metadata is absent or invalid; honor the legacy stopwatch duration where
  supported by the keymap. Compute
  `new_duration = max(0, old_duration + signed_units * 5)` with checked arithmetic and
  `new_end = (start + new_duration) modulo 24 hours`. Keep the start fixed. Rewrite only
  that ledger line, preserving its checkbox, name, non-duration metadata, child bullets,
  newline style, and surrounding bytes; normalize the adjusted range and duration field
  to Bob's canonical `(**HHMM-HHMM** [t:: Nm])` representation while retaining other
  parenthetical metadata. Subtraction never makes a negative duration. Report both
  requested units and actual applied minutes when clamping changes the delta.
- Read and stage the daily file through `CaptureBatchPlanner`, so later items observe
  earlier edits to that same staged file. A `+5` after a session-start item may adjust
  the session created earlier in the draft. An adjustment before any timed session
  fails. Dry-run computes the same result without committing; a later invalid item rolls
  back all earlier staged changes. Nothing is written for a failed parse, invalid
  ledger, overflow, or commit failure.
- Give success a distinct `pomodoro_adjust` kind, a semantically accurate updated
  placement, daily-note destination, and an additive `pomodoro_adjust` object containing
  direction/requested units, actual delta minutes, before/after start/end/duration,
  resolved line/name, and rendered new range. Human CLI output should say what changed
  and where; dry-run should say what would change. Keep existing JSON keys and schema
  version 1 stable for other captures and for batch `captures` arrays.
- `bob capture-parse` should return `pomodoro_adjust` mode for an exact item, an
  adjustment span covering only the signed token, and an additive per-item/top-level
  adjustment spec with sign and unit count. It stays purely lexical and never guesses
  current times. Invalid standalone counts (`+0`, overflow) and malformed
  adjustment-first items produce a diagnostic with a useful range; `+`/`-` alone are
  incomplete editing states and strict capture errors. Other nonmatching words retain
  normal task semantics. Ensure the rewrite/completion paths do not reinterpret an
  adjustment as a route, task, or destination marker.
- Bob Mac Capture must decode the optional adjustment spec/result without breaking older
  Bob versions. Use the `bob capture --dry-run --no-clip --format json` preview result
  as the sole source for resolved timing, including in mixed drafts. Show a calm,
  legible before-to-after line, signed minute effect, target name/line, and an
  accessible description; committed feedback uses the same result fields. Do not compute
  ledger timing in Swift. Update fake-Bob fixtures, model tests, panel tests, and
  notification copy as appropriate.

# Implementation path and acceptance

`adjustment_core` should add the semantic variant in `src/native/capture_language.rs`,
route an exact item through a dedicated `src/native/capture.rs` planner branch, and use
shared scanner/range helpers in `capture_pomodoros.rs`/`pomodoro.rs` where needed. Keep
replacement narrowly scoped to the selected line and use the existing staged-file commit
path. Tests in `tests/cli.rs` should prove `+5` extends by 25 minutes, `-N` clamps at
zero, a range crosses midnight correctly, metadata and CRLF survive, mixed batches apply
in order, dry-run is nonmutating, and bad targets/overflow/late batch failure leave the
vault unchanged. Compare representative edits to `plugins/bob-ledger-tools/main.js` in
the `bob-plugins` repository, without changing the plugin.

`editor_contract` should expose the new mode/span/spec and structured diagnostics
through `capture_language.rs` and `capture_parse.rs`, update `bob capture` and
`bob capture-parse` help plus `docs/capture.md`, and add protocol tests for exact-unit
recognition, whitespace, mixed drafts, declarations, text/marker near misses, errors,
and JSON compatibility. The displayed examples should include `+5`, `-2`, and a mixed
draft using blank lines. The mode is an action, so do not request route or task
completion candidates for it.

`mac_presentation` should add tolerant decoding in
`Sources/CaptureCore/CaptureModels.swift`, a small pure presentation model like
`CapturePomodoroStartPresentation`, and a dedicated row in `CapturePanelView.swift` for
the adjustment result. Update the semantic palette only if the new span needs a distinct
category. Cover valid preview, submission, failure, mixed drafts, and accessibility via
`Tests/Fixtures/fake-bob` and existing Swift test suites; use the project's macOS CI
when host Swift cannot compile macOS-only targets.

Each phase runs focused tests for its changes, then the landing pass checks Rust
formatting, relevant Rust unit/integration suites and Clippy, plus the Mac suite/CI for
the exact landed app revision. The feature is complete when the CLI and Mac
preview/submit agree on the same before/after result for `+5` and `-N`, with no write on
any failed batch.
