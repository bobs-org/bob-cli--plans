---
tier: epic
title: Shift the running Pomodoro from capture with ++N and --N
goal: 'A whole capture item `++[N]` / `--[N]` moves today''s running Pomodoro N five-minute
  units later / earlier exactly like Obsidian''s `N\o` / `N\O`, every Pomodoro session
  operator''s count is optional and defaults to 1 (so `+`, `-`, `++`, and `--` all
  work), and Bob CLI and Bob Mac Capture preview and apply these operators with identical,
  atomic results.

  '
phases:
- id: shift_core
  title: Parse and atomically apply Pomodoro session shifts
  depends_on: []
  size: medium
  description: 'shift_core: add the unified whole-item session-operator lexer (one
    sign resizes, two signs shift, count defaults to 1), the PomodoroShift capture
    kind, a staged planner that translates the running session''s start and end like
    `N\o`/`N\O`, additive `pomodoro_shift` capture JSON plus human output, and CLI
    integration tests.'
- id: editor_contract
  title: Expose and document the session-operator contract
  depends_on:
  - shift_core
  size: medium
  description: 'editor_contract: surface `pomodoro_shift` mode, span, spec, and diagnostics
    in capture-parse, make bare `+`/`-`/`++`/`--` complete editor states, keep completion/rewrite/`@@`
    away from operator items, and update every help text, docs/capture.md, and README
    with protocol tests.'
- id: mac_shift
  title: Preview and submit session shifts in Bob Mac Capture
  depends_on:
  - editor_contract
  size: medium
  description: 'mac_shift: decode the shift spec/summary tolerantly, add a pure shift
    presentation with a double-chevron row, name the footer action, carry shifts into
    notifications and the Pomodoro palette, guarantee literal ASCII hyphens in the
    editor, and cover it with real-bob fixtures, tests, README, and green macOS CI.'
proposed_by: bbugyi200.apollo.2k
create_time: 2026-09-28 10:35:56
status: done
bead_id: bob-cli-2a
---

- **PROMPT:** [prompts/202609/pomodoro_shift_operators.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_shift_operators.md)
- **BEAD:** [bob-cli-2a](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2a/README.md)

# Problem and outcome

`bob capture` already treats a whole item `+N` / `-N` as a Pomodoro _resize_ that
emulates the Obsidian bob-ledger-tools `N\p` / `N\P` keymaps (bob-cli-27). The sibling
keymaps `N\o` / `N\O` (`bobLedgerMovePomodoroLater` / `bobLedgerMovePomodoroEarlier` →
`offsetPomodoroRange`) move the selected Pomodoro's whole time range N × 5 minutes later
or earlier, keeping its duration. Typing `++3` in a capture should do exactly what `3\o`
does with the running Pomodoro selected in Obsidian, and `--3` what `3\O` does.

Vim's repeat count is optional (`getVimRepeat` defaults to 1), so every session operator
must accept an omitted count: `+`, `-`, `++`, and `--` mean one unit. Today a bare `+`
or `-` is an "incomplete" error; that changes.

Both the CLI and the Bob Mac Capture panel must support the new syntax. Bob stays the
only implementation of grammar, clock/ledger math, and vault writes. The Mac app only
decodes and presents Bob's JSON.

# Design: one sign resizes, two signs shift

The grammar becomes a small, symmetric family of whole-item **Pomodoro session
operators**. All of them act on today's running (open, timed) Pomodoro:

| Item         | Operator | Effect on the running Pomodoro                               | Obsidian     |
| ------------ | -------- | ------------------------------------------------------------ | ------------ |
| `+` / `+N`   | resize   | Extend: end moves N × 5 min later, start fixed               | `\p` / `N\p` |
| `-` / `-N`   | resize   | Shorten: end moves N × 5 min earlier (duration clamps at 0)  | `\P` / `N\P` |
| `++` / `++N` | shift    | Move start **and** end N × 5 min later, duration unchanged   | `\o` / `N\o` |
| `--` / `--N` | shift    | Move start **and** end N × 5 min earlier, duration unchanged | `\O` / `N\O` |
| `=x`         | close    | Close it (unchanged)                                         | Ctrl+Enter   |

`N` defaults to 1. The mnemonic "one sign moves the end, two signs move the whole
session" is how docs and help should teach it. The doubled glyph also carries into the
Mac preview's double-chevron icon.

Naming: the new operation is a **shift** (`pomodoro_shift`), not a "move". Bob already
uses "move" for relocating ledger entries and Task Links ("Moved Task Link BUGS →
FOCUS", "started entry moves to the current slot"). "Shift" names translating a time
range unambiguously. Human verbs are `shifted` / `would shift`, and directions are
`later` / `earlier`.

## Recognition rules (exact)

Let `t` be the item's first physical line trimmed of leading/trailing whitespace. An
**operator token** is a sign run of exactly `+`, `-`, `++`, or `--` followed by zero or
more ASCII digits. One sign means resize; two identical signs mean shift. Recognition
happens where the adjustment check runs today: after the whole-item `=x` close check and
before Pomodoro-link and ordinary parsing.

1. **Exact operator:** `t` is exactly one operator token and the item has exactly one
   physical line. An omitted count means 1. Leading zeros are accepted (`++03` is 3
   units). A zero magnitude (`+0`, `-0`, `++0`, `--0`, `++00`) is an error, and an
   oversized magnitude fails checked parsing/arithmetic. Both fail before any write.
2. **Claimed near miss (error, never a task):** `t` begins with a _counted_ token (sign
   run plus at least one digit) followed by anything else (`+5 more`, `++3@work`,
   `--2 s:1`, `++3++`). The other near miss is `t` exactly equal to a token (bare or
   counted) in an item that also has child lines. Errors belong to the operator's own
   family: invalid adjustment for one sign, invalid shift for two.
3. **Not an operator (unchanged ordinary parsing):** everything else. That covers a bare
   sign run followed by more text (`- foo`, `+ idea`, `-- aside`, `++ plan`), a sign run
   longer than two or mixed (`+++`, `---`, `+-`, `-+3`), and any mid-body token
   (`Plan ++3`, `C++`). This follows the existing precedent that only exact operator
   shapes claim an item, e.g. `==` and `=3` stay prose.

Error copy must be actionable and must teach the default. Suggested wording (adjust
minor phrasing to match the codebase):

- Shift shape:
  ``Pomodoro shift items must contain only the operator (for example `++3` or `--`); remove extra text, markers, or child lines``
- Shift zero:
  ``Pomodoro shift magnitude must be positive; `++0` and `--0` move nothing (for example `++3` moves today's Pomodoro 15 minutes later)``
- Shift overflow: `Pomodoro shift is too large; use a smaller unit count`
- Shift conflicts:
  ``Pomodoro shift `++N`/`--N` cannot be combined with <flag or marker>; capture the shift alone``
- Adjust shape (updated example):
  ``… must contain only the signed count (for example `+5` or `-`) …``
- The existing "Pomodoro {name} is already running; finish the current Pomodoro first
  (close it with `=x`) or use `+N`/`-N` to adjust it" hint should also mention
  `++N`/`--N` to shift it.

The `POMODORO_ADJUST_INCOMPLETE_ERROR` path disappears.

## Shift execution semantics

- **Target:** the same selection as the adjustment. That is exactly one open, timed,
  column-zero entry in today's `## Pomodoros` section, via `capture_pomodoros::scan` on
  the staged daily file. A missing or unreadable day file or section, no open timed
  entry ("… no open timed Pomodoro to shift"), multiple open timed entries ("… finish
  all but one before shifting"), or an unparseable range fail without writing. An entry
  past its nominal end is still eligible.
- **Math:** `delta = ±N × 5` minutes with checked arithmetic. Translate **both**
  endpoints the way `offsetPomodoroLineRange` does:
  `new_start = (start + delta) mod 1440` and `new_end = (end + delta) mod 1440`, using
  Euclidean remainder. The duration is the entry's resolved duration (the existing
  `adjustment_duration_minutes` chain: `[t::]`, then legacy stopwatch, then range span)
  and is unchanged. Shifts never clamp. They wrap across midnight like Obsidian and like
  the adjustment's end.
- **Rewrite:** replace only that line's range with
  `format_adjusted_range(new_start, new_end, duration, metadata)`. This is Bob's
  canonical `(**HHMM-HHMM** [t:: Nm] …)` form, and it keeps other parenthetical
  metadata. Checkbox, name, children, surrounding bytes, and CRLF must be preserved. For
  an already-canonical line the result must be byte-identical to Obsidian's `N\o` /
  `N\O`, e.g. `- [ ] (**0900-0925** [t:: 25m]) — FOCUS` with `++3` becomes
  `- [ ] (**0915-0940** [t:: 25m]) — FOCUS`.
- **Deliberately mirrors `\o`:** there is no check against neighbouring entries, no
  "start must be before now" guard, and no cap on N (a `++288` wraps a full day). The
  preview makes the effect visible before Return.
- **Transaction:** read and stage the file through `CaptureBatchPlanner`, as the
  adjustment does. Later items see earlier staged edits (a session started earlier in
  the draft can be shifted; `--2` then `+` in one draft composes). `--dry-run` computes
  without committing, and any failure rolls back the whole batch.
- **Isolation:** a shift is item-local. A `@@` declaration never applies to it. Forced
  `--route`/`--section`/`--task`/`--task-ref`/`--task-section`/`--clip`, `%`, `s:<N>`,
  and `p:<N>` are rejected with shift-specific errors. `--dry-run`, `--no-clip`,
  `--bob-dir`, `BOB_DAY_FILE`, and output formats keep working.

## Contracts

`bob capture --format json` for a shift keeps schema version 1 and every existing key.
It reports kind `"pomodoro_shift"`, `text` = the raw token, `task_line` = the new ledger
line, the same `placement` value the adjustment uses, `route: null`, `relative_target` =
the day file, and `pomodoro_name`. It adds this object (omitted for every other kind):

```json
"pomodoro_shift": {
  "direction": "later",
  "requested_units": 3,
  "delta_minutes": 15,
  "before_start": "0900", "before_end": "0925",
  "after_start": "0915", "after_end": "0940",
  "duration_minutes": 25,
  "pomodoro_line": 5,
  "pomodoro_name": "FOCUS",
  "time_range": "(**0915-0940** [t:: 25m])"
}
```

`delta_minutes` is signed (negative for `earlier`). `pomodoro_name` is omitted when the
entry is unnamed. Batches carry it per item in `captures`, and the first item's value is
also top-level, as for `pomodoro_adjust`. The existing `pomodoro_adjust` object is
unchanged. A bare `+` simply reports `requested_units: 1`.

Human output mirrors the adjustment's layout:

```text
✓ shifted  2026/20260928.md
  FOCUS 0900-0925 to 0915-0940 (25m), 15m later at line 5
  - [ ] (**0915-0940** [t:: 25m]) — FOCUS
```

Dry-run uses `[dry-run] ok would shift  …`. An unnamed entry reads "current session",
and an earlier shift reads `…, 5m earlier at line 5`.

`bob capture-parse` stays purely lexical:

- Mode `pomodoro_shift`.
- A `pomodoro_shift` span covering only the operator token.
- An additive `pomodoro_shift` spec `{"raw": "++3", "later": true, "units": 3}`,
  top-level for the first item and per item, and omitted otherwise.
- An `invalid_pomodoro_shift` diagnostic whose range policy matches
  `invalid_pomodoro_adjustment`: the token range for zero/overflow, the item range for
  shape errors.

Bare `+` / `-` now report mode `pomodoro_adjust` with spec
`{"raw": "+", "plus": true, "units": 1}` instead of `incomplete`. Bare `++` / `--`
report `pomodoro_shift` with `units: 1`. The human parse output gains
`shift     ++3 (15m later, 3 units)`. Unit pluralization should be correct for both
lines (`- (5m, 1 unit)`).

## Command-line spelling

Capture text reaches the parser verbatim for `bob capture ++3`, `bob capture --3`,
`bob capture +`, `bob capture -`, and `bob capture -- -2`, because both the top-level
dispatcher and the native TEXT argument allow hyphen values. A bare `--` is the
universal end-of-options marker, so `bob capture --` and `bob capture -- --` carry no
text. Document `bob capture --1` (or `printf -- '--\n' | bob capture`) as the CLI
spelling of a one-unit earlier shift. Do not change the `bob` dispatcher's `--` handling
in this epic. The Mac app is unaffected because it always passes the draft after its own
`--`.

# shift_core

Work in bob-cli.

- `src/native/capture_language.rs`:
  - Add one small lexer for operator tokens that returns the operator (resize or shift),
    the direction, and the optional digits. Build both the execution grammar and (in the
    next phase) the editor grammar on it, instead of growing `signed_count_prefix_len`
    special cases.
  - Replace `parse_pomodoro_adjust_item` with a session-operator item parser that
    implements the recognition rules above. It emits `CaptureKind::PomodoroAdjust` (now
    with `units: 1` for a bare sign) or a new
    `CaptureKind::PomodoroShift { spec: PomodoroShiftSpec { raw, later, units } }`.
  - Add the new shift error constants.
  - Extend the editor/exec parity test's `expected_mode` match so it still compiles. The
    editor side may temporarily map shift to a placeholder only if the next phase
    completes it; prefer adding `EditorMode::PomodoroShift` now.
- `src/native/capture.rs`:
  - Extract the adjustment's current-session selection and line/range resolution into a
    shared helper. It covers day-file existence, scan, the exactly-one open timed
    target, line span/segment start, and parsed range, and it takes a verb for error
    messages. `plan_pomodoro_adjust_item` and the new `plan_pomodoro_shift_item` both
    use it.
  - Add `reject_pomodoro_shift_conflicts`.
  - Add `PomodoroShiftSummary` and `CaptureItemResult.pomodoro_shift` (skip when `None`;
    add `pomodoro_shift: None` at every existing constructor).
  - Add kind label `pomodoro_shift`, the dispatch arms beside every `PomodoroAdjust` arm
    (grep `PomodoroAdjust` / `pomodoro_adjust` to find them all), and
    `print_human_pomodoro_shift_success`.
  - Update the "already running" hint.
  - Keep the adjustment JSON and behavior byte-stable apart from the new default count.
- `tests/cli.rs`: follow the structure of the `capture_pomodoro_adjust_*` tests. Cover:
  - `++3` / `--2` JSON and human output.
  - Bare `++` / `--` / `+` / `-` defaulting to one unit.
  - Midnight wrap both ways (`--2` on `0005-0030` → `2355-0020`; `++3` on `2350-0015` →
    `0005-0030`).
  - Metadata, name, children, and CRLF preservation, plus the stopwatch/range duration
    fallbacks.
  - A byte-for-byte comparison of representative lines with the bob-plugins
    `offsetPomodoroLineRange` result. Open bob-plugins read-only with `/sase_repo` and
    read `plugins/bob-ledger-tools/main.js`; do not modify the plugin.
  - Mixed batches in order: start-then-shift, `--2` then `+`, and a shift plus an
    ordinary `@@`-routed task.
  - Dry-run with no mutation, and late-item failure rolling back.
  - Every error: no/multiple timed entries, zero, overflow, extra text, child line, each
    forced flag/marker, and `++3 more`.
  - Prose that must stay prose: `- foo`, `-- aside`, `+++`, `---`, `+-`, `Plan ++3`.
  - `bob capture --3`, `bob capture -`, and `bob capture --1` argv spellings.
- Run the focused tests, `cargo clippy --all-targets --all-features`, and
  `cargo fmt --check`.

# editor_contract

Work in bob-cli, building on the `shift_core` lexer.

- `src/native/capture_language.rs`:
  - Replace `parse_editor_adjust_item` with an editor session-operator parser that
    mirrors the execution grammar but never fails. Exact operators report
    `pomodoro_adjust` / `pomodoro_shift` with their spec and a span over only the token.
    Near misses report the family mode plus `invalid_pomodoro_adjustment` /
    `invalid_pomodoro_shift`. Bare signs are complete operators and no longer
    `incomplete`.
  - Add a small builder for operator `EditorItemParse` values rather than copying the
    long struct literal again.
  - Add `SpanKind::PomodoroShift`, `EditorMode::PomodoroShift`, and
    `pomodoro_shift: Option<PomodoroShiftSpec>` on item and top-level parses (plus
    `None` at each existing literal).
  - Skip `@@` inheritance for shift items, as for adjust and close.
  - Add the `@@ cannot take a Pomodoro shift …` notice and a `NonAbsorbable` arm.
  - Update completion suppression (`capture-complete` returns an empty success on any
    operator item).
  - Extend the parity inputs with `+`, `-`, `++`, `--`, `++3`, `--2`, `  ++3  `, and
    near misses.
- `src/native/capture_parse.rs`:
  - JSON fields.
  - The `shift` human line.
  - Singular/plural units.
  - Help text: examples `'++3'`, `'--'`, `'+'`, and a mixed draft. Add `pomodoro_shift`
    to the Modes list.
- Help text: update `bob capture --help` (`src/native/capture.rs`) and
  `bob capture-complete --help` (`src/native/capture_complete.rs`). Drop "a standalone
  `+`/`-` is incomplete", teach the operator table and the default count, and include
  the `--1` CLI note.
- `docs/capture.md`:
  - Grammar-at-a-glance rows: change `+N` / `-N` to `+[N]` / `-[N]` and add `++[N]` /
    `--[N]`.
  - "You typed" rows for `++3`, `--`, `-`, `- foo`, `+++`, and `++3 more`.
  - A new "Shifting the current Pomodoro" section after "Adjusting the current
    Pomodoro", with the operator table, semantics, examples (including
    `printf -- '--2\n\n+\n' | bob capture`), and the CLI `--` note. Update the
    adjustment section for the optional count.
  - Parse-contract updates: modes, span kinds, diagnostic codes, human lines, and the
    completion-suppression sentence. Add the contents entry.
- `README.md`: update the capture grammar summary to match.
- `tests/cli.rs`: follow `capture_parse_pomodoro_adjust_protocol`,
  `capture_parse_pomodoro_adjust_human_and_help`, and
  `capture_complete_and_rewrite_ignore_adjustments`. Cover:
  - Exact and bare operators, whitespace, mixed drafts, and `@@` drafts (including the
    shadow notice).
  - Near-miss diagnostics and ranges.
  - Prose non-matches.
  - JSON compatibility: spec objects omitted for other modes.
  - Completion and rewrite ignoring operator items.
  - Help text mentions.
- Finish with `just fmt`, `just lint`, and `just test`.

# mac_shift

Work in bob-mac-capture. Open it with `/sase_repo`; if the linked name does not resolve,
open `gh:bobs-org/bob-mac-capture`. Follow the pattern of the adjustment commit
`3c9fe83` and the close preview commits.

- `Sources/CaptureCore/CaptureModels.swift`:
  - Tolerant `decodeIfPresent` of `PomodoroShiftSpec` (`raw`, `later`, `units`) on
    `CaptureParseResponse` and `CaptureParseItem`.
  - `PomodoroShiftSummary` (the capture JSON fields above) on `CaptureCommandSuccess`,
    including nested `captures`.
  - Older Bob output decodes as nil. An older Bob that parses `++3` as a task simply
    previews a task.
- New `Sources/CaptureCore/CapturePomodoroShiftPresentation.swift`: pure wording only,
  with no Swift clock or ledger math.
  - `statusText` mirrors the CLI: "Would shift FOCUS 0900-0925 to 0915-0940 (25m), 15m
    later at line 5" or "Shifted …".
  - `sessionText`: `0900-0925 → 0915-0940 (25m), 15m later`.
  - `destinationText`: `FOCUS · line 5`.
  - `symbolName`: `chevron.forward.2` for later, `chevron.backward.2` for earlier. The
    doubled chevron echoes the doubled sign.
  - A spoken `accessibilitySummary`.
  - `notificationDetail`.
- `Sources/BobMacCapture/CapturePanelView.swift`:
  - Add a shift row in the standard preview item, styled as the sibling of the
    adjustment row: glyph, monospaced semibold session text, secondary destination.
  - Include its accessibility summary in the preview label.
- `Sources/BobMacCapture/CapturePanelModel.swift`: `primaryActionTitle` names the action
  for a single operator preview. Use "Shift" for a shift, and also "Adjust" for an
  adjustment so Return's meaning is explicit, as it already is for Close, toggle, and
  link.
- `Sources/CaptureCore/CompletionRowContent.swift`: map `pomodoro_shift` to the Pomodoro
  session category (`.pomodoroStart`), like adjust and close.
- `Sources/BobMacCapture/NotificationService.swift`:
  - Friendly kind "Shift".
  - Single-capture body detail.
  - Batch-line suffix.
  - Subtitle falls back to the day file.
- **Literal hyphens:**
  - Make sure macOS smart-dash substitution can never rewrite `--3` / `-2` in the draft
    editor. On the draft's backing `NSTextView`, disable only automatic dash
    substitution; leave quotes and the app's own Tab `--`→`—` snippet unchanged.
  - Put the configuration in a small testable helper.
  - Document in README that Tab after `--` still produces an em dash, so type `--` and
    press Return.
- Fixtures and tests:
  - Generate real-bob JSON fixtures from the `editor_contract` build against a sandbox
    vault (`BOB_DIR` + `BOB_NOW`): `Tests/Fixtures/pomodoro-shift-*.json` for later,
    earlier, bare `--`, a no-running-session failure, a mixed draft, and parse responses
    for `++3` and `-`.
  - Extend `Tests/Fixtures/fake-bob`.
  - Add or extend `CaptureModelTests`, `CapturePomodoroShiftPresentationTests`,
    `CompletionRowContentTests`, `NotificationServiceTests`, `CapturePanelModelTests`
    (preview, footer title, mixed draft, error surface), and `BobProcessClientTests`.
    Watch argument order in test helpers; bob-cli-27.4 fixed a `relativeTarget`/`target`
    ordering mistake in this suite.
- README: update the grammar requirement and behavior paragraphs (`+[N]` / `-[N]`,
  `++[N]` / `--[N]`, default 1, older-Bob compatibility).
- Verification: this Linux host has no Apple Swift toolchain. After committing, confirm
  the macOS CI workflow run for that exact commit passes (`gh run list` / `gh run view`
  in bob-mac-capture, waiting through `/sase_monitor`). Fix forward until it is green.

# Non-goals

- No changes to bob-plugins or to Obsidian keymaps.
- No shifting of a named or non-running Pomodoro (e.g. `#name++3`), and no combined
  tokens such as `++3=x`.
- No overlap validation against neighbouring ledger entries.
- No change to the top-level `bob` dispatcher's argument forwarding.

# Acceptance

- `++3`, `--3`, `++`, `--`, `+`, and `-` produce the same ledger line in Bob as the
  corresponding `N\o`, `N\O`, `\p`, and `\P` keymaps in Obsidian (canonical lines
  compare byte-for-byte).
- Near misses never become tasks, and prose non-matches stay prose.
- Mixed drafts apply in order, dry-run never writes, and any failure leaves every file
  unchanged.
- Resize and shift previews in Bob Mac Capture show exactly Bob's dry-run result, and
  submission produces the same result.
- bob-cli `just fmt`, `just lint`, and `just test` pass. bob-mac-capture macOS CI is
  green for the landed revision.
