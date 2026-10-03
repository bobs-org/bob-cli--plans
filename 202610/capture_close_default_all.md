---
tier: tale
title: Default omitted Pomodoro park and complete lists to all Task Links
goal:
  Bare park and complete capture syntaxes target all current Pomodoro Task Links, with
  matching Bob Mac Capture previews and preserved explicit selections.
size: medium
proposed_by: bbugyi200.athena.0vv
create_time: 2026-10-03 15:19:26
status: wip
---

# Default omitted Pomodoro park and complete lists to all Task Links

## Objective and scope

Change `bob capture` so `=*` parks all numbered Task Links in today's running Pomodoro
and `=!` completes all of them. Explicit lists keep selecting only the specified
numbers: `=*1` and `=!1` retain the previous single-task behavior. Make Bob Mac Capture
decode and present the same behavior, including previews, teaching hints, accessibility
text, and checked-in JSON fixtures.

This is a `tale` with implementation size `medium`: it is one bounded grammar and
selection change with a coordinated Swift presentation update, suitable for one coding
agent. No separate agents or epic phases are needed. Implement bob-cli first, then its
linked Mac client in the same work session.

The authorized work changes capture behavior and its documentation and tests. It does
not change Pomodoro timing, task-lane policy, the definition of a numbered Task Link,
the vault's contents, plugins, or installation/deployment. Do not add a configuration
switch, CLI option, or a Swift implementation of capture grammar. Do not edit SASE
memory or decision records.

## Findings that determine the design

- `src/native/capture_language/close_selection.rs` is the shared execution/editor lexer.
  `parse_selection_body` currently fabricates task number 1 for each present-but-empty
  `*` or `!` group before overlap validation. Aliases and long `=x` forms already use
  this same grammar; link suffixes do too.
- `PomodoroCloseSpec` in `capture_language/model.rs` and the lexer's valid and
  incomplete structures currently contain only concrete lists. `has_selection` and
  `capture/pomodoro_close.rs::selection_from_spec` would accidentally treat an empty
  wildcard list as an ordinary close unless they retain its intent.
- `capture_pomodoro_close/selection.rs::number_task_links` owns the current numbering
  rules. `apply_close_selection` validates bounds, assigns outcomes, detects conflicting
  duplicate targets, rewrites markers, and resolves logs. This is the appropriate place
  to resolve a wildcard against the actual staged session. The linked-task planner
  already handles parked work without carrying and completed work with its existing
  status guards and warnings.
- `capture_language/close_log.rs` computes inline defaults and positional bullet
  assignments from lexical lists. A wildcard needs the session lineup, so retaining the
  old fabricated 1 or treating empty lists as no worked tasks would misassign or reject
  valid logs.
- `capture/pomodoro_close.rs::build_close_summary_json` currently echoes the parsed
  lists. Capture previews additionally expose resolved `task_links`, task effects,
  carried links, and before/after Pomodoro blocks.
- The Mac app's `Sources/CaptureCore/CaptureModels.swift` decodes parse specs and close
  summaries additively. `CapturePomodoroClosePresentation.swift` derives selection
  presence from nonempty arrays and explicitly teaches “parks 1” and “completes 1.” Its
  tests and `Tests/Fixtures/pomodoro-close-*-alias-*.json` encode the current task-1
  behavior.

Read the project instructions in each repository. Open `bob-mac-capture` using
`sase repo open bob-mac-capture -r "Implement the approved capture default-all client contract"`
and use its returned checkout. Repository paths below are relative to their own
checkout.

## Behavioral contract

Use one consistent rule for aliases, long forms, whole-item closes, chains, and
existing/new-task link-close suffixes:

1. An absent `*` or `!` group selects nothing for that outcome. A present group with
   explicit numbers selects those numbers. A present group without numbers selects every
   numbered Task Link not explicitly assigned to another outcome. With no other groups
   this means all numbered Task Links in the current Pomodoro. Preserve equivalence of
   `=*` with `=x*`, and `=!` with `=x!`.
2. Explicit lists are exceptions to an omitted-number group. Resolve explicit
   in-progress, park, complete, and drop assignments first, then assign the remaining
   indices to the one wildcard. Explicit lists still reject duplicate numbers, explicit
   overlaps, invalid zeros, and out-of-range indices. The wildcard does not hide a bad
   explicit selection.
3. Two omitted-number outcome groups compete for the same remainder and are invalid
   regardless of group order or the current lineup. Reject `=*!`, `=!*`, and their long
   forms lexically, with one shared actionable diagnostic asking for numbers on at least
   one group. Do not choose an outcome silently.
4. “All” uses the existing numbered lineup, including plain, deferred `#`, and embedded
   links, at the depths already supported by numbering. Struck links, incidental links
   in prose, fenced content, and other unnumbered entries do not become wildcard
   targets. Keep existing terminal-task/status guards and unresolved-target warnings;
   selecting all does not bypass them.
5. A running session with zero numbered Task Links closes successfully for bare `=*` or
   `=!`, just as an empty session can close with `=x`. A positive explicit number still
   fails bounds validation. Work Log text with no eligible task still fails atomically.
   Missing or ambiguous running sessions keep the existing diagnostics.
6. Link-close forms insert or move their task into the running session first; expand the
   wildcard against that staged lineup, including the linked/new task. In a chain or
   batch, use the current staged running session at that item's position, not a snapshot
   taken before earlier items.
7. Parking records normal work, does not carry the selected links onward, and does not
   release sticky task lanes. Completing applies the existing complete outcome and Work
   Log behavior. Preserve duplicate-target conflict detection and merge behavior so
   repeated links do not duplicate task mutations/logs.

Examples for three numbered links, assuming existing status guards permit the requested
outcome:

| Input             | Expected outcome                                               |
| ----------------- | -------------------------------------------------------------- |
| `=*` or `=x*`     | Park 1, 2, 3; carry none of these links                        |
| `=!` or `=x!`     | Complete 1, 2, 3; carry none of these links                    |
| `=*1`             | Park 1; other links retain existing explicit-park behavior     |
| `=!1`             | Complete 1; other links retain their existing ledger behavior  |
| `=*!2` or `=x*!2` | Complete 2; park 1 and 3                                       |
| `=!2*`            | Complete 2; park 1 and 3                                       |
| `=!~2`            | Drop 2; complete 1 and 3                                       |
| `=x1*`            | Continue 1; park 2 and 3                                       |
| `=x0*`            | No ordinary continuing selections; park all numbered links     |
| `=x0!`            | No ordinary continuing selections; complete all numbered links |
| `=*1!1`           | Explicit overlap error                                         |
| `=*!`             | Competing wildcard error                                       |
| `=*~` or `=*1,`   | Incomplete editing state; execution rejects it                 |

These mixed-selection rules deliberately replace old defaulted-1 overlap errors such as
`=x1*` and `=*~1`. Document this migration and keep explicit `*1`/`!1` examples for
users who want the old scope.

## Implementation steps

### 1. Carry wildcard intent through the shared language and JSON contracts

Add explicit boolean intent fields `park_all` and `complete_all` to `CloseSelectionLex`,
`CloseSelectionIncomplete`, `PomodoroCloseSpec`, and the pure planner's
`CloseSelection`. A small internal selection abstraction is acceptable if it reduces
argument proliferation, but keep the existing JSON arrays and their types. Never use a
magic index, zero sentinel, fabricated 1, or raw-token reparsing to represent all.

In `close_selection.rs`, replace the fabricated-1 default with the appropriate flag and
an empty concrete list. Carry flags through valid/incomplete conversions and all
constructors and plain-close defaults. Validate explicit lists as before; add the
two-wildcard diagnostic in `capture_language/markers.rs`. Preserve exact typed ranges,
including the lone `*`/`!` span. `=*` and `=!` remain complete valid actions, and
commas/`~` remain genuine incomplete states. Update execution and editor call sites
together so they report matching diagnostics.

Expose flags additively on `capture-parse.pomodoro_close` and `capture.pomodoro_close`
without bumping `schema_version`. Default them to false when absent. The lexical parse
shows typed concrete arrays plus wildcard intent, never guessed indices. The resolved
capture summary expands the wildcard into its concrete `park` or `complete` array in
ascending order, while retaining the intent flags. Keep explicit selections' JSON
unchanged apart from optional additive flags. `has_selection` must recognize wildcard
intent even when its resolved set is empty.

Update `capture_parse.rs::format_pomodoro_close` to describe lexical wildcard intent as
“parked all” / “complete all,” or “the remaining links” when explicit exceptions are
present. Do not print an ordinary `=x`, an empty selection, or “defer the rest” for a
wildcard that assigns the whole remainder. Resolved human capture output must agree with
the actual per-link outcomes.

### 2. Resolve against the staged session and preserve existing close effects

Pass the new intent through `selection_from_spec`. In
`capture_pomodoro_close/selection.rs`, number links and bounds-check every explicit list
first. Expand the one wildcard over the complement of all explicit assignments, then
feed those selections through the existing outcome, duplicate, ledger, status, carry,
and Work Log machinery. Do not introduce a second numbering scan with different rules or
expand against the live filesystem after the batch planner has staged changes.

Keep existing wire values for `task_links[].source`: wildcard-selected rows use
`listed`, meaning selected by the close, while untouched rows keep current
`ledger`/`unlisted` behavior. Update documentation/comments describing that meaning
rather than adding a source string older apps cannot render. Build resolved wildcard
arrays from selected lineup outcomes in `build_close_summary_json`, including unresolved
selected links; leave their warnings visible. Preserve before/after blocks and
plan-budget accounting.

### 3. Make Work Log parsing aware of unresolved selections

Update the shared `close_log.rs` helpers and their execution/editor callers, especially
`is_loggable`, `default_log_index`, lexical worked-task computation, inline lexing, and
positional bullets. A wildcard makes worked-task membership session-dependent even when
another explicit group is present. Defer positional assignment whenever a wildcard is
present instead of resolving only the concrete list or issuing a premature “no worked
tasks”/“too many bullets” error.

For wildcard closes, resolve an unnumbered inline entry to the first eligible top-level
worked Task Link; retain explicit inline numbers as written. One inline entry still
creates one log entry, not a copy on every selected task. Unnumbered child bullets keep
the existing rule over the final eligible lineup: one worked task takes every bullet;
otherwise bullet i goes to worked task i, and excess bullets fail. Numbered entries can
target wildcard-selected links but never dropped, unnumbered, or otherwise ineligible
targets. Resolve these checks before any vault write.

Reuse optional log indices for deferred resolution, preserving `CloseLogOrigin` so the
planner distinguishes a default inline entry from positional bullets and keeps
actionable diagnostics and literal/nested text. Do not alter log assignment for closes
without wildcard intent. Editor spans cover only typed indices; `capture-parse` may omit
an unresolved index, while execution JSON always reports the concrete resolved index.
Test that parser, completion, rewrite, and execution agree on valid/incomplete shapes
and preserve the raw spelling.

### 4. Update the Mac client and user-facing documentation

In the linked Mac checkout, add tolerant `parkAll`/`completeAll` decoding, constructor
defaults, and coding keys to both `PomodoroCloseSpec` and `PomodoroCloseSummary` in
`CaptureModels.swift`. Decode missing flags as false so older Bob responses still render
their actual returned behavior. Keep all execution, selection expansion, and task
numbering in bob-cli.

In `CapturePomodoroClosePresentation.swift`, include flags in `hasSelection`, including
empty-lineup closes. Build per-task rows, summary text, completed counts, carry text,
notification wording, and accessibility text from Bob's resolved outcomes. Change the
teaching lead to “parks all” / “completes all” and teach explicit `=*1`/`=!1` or
numbered lists as narrowing the scope. Preserve semantic colors and the existing
linked/new-task and batch presentation paths.

Generate updated/new fixture JSON with the changed bob binary against temporary,
deterministic vault fixtures using fixed `BOB_DAY_FILE` and `BOB_NOW`; do not manually
invent resolved preview outcomes. Retain dedicated legacy responses without the new
fields for compatibility tests. Update Mac parse-decoding, presentation, hint,
accessible summary, and preview/submit tests, especially
`testShortAliasDefaultsDecodeWithRawAndSpans` and
`testShortAliasPreviewShowsDefaultTaskOne`.

Update bob-cli's `docs/capture.md` syntax/recipe tables, close-selection section, Work
Log explanation, and lexical/resolved JSON contract, plus help strings in
`capture/cli.rs`, `capture_parse.rs`, and any relevant `capture_complete/cli.rs`
wording. Explain empty sessions, explicit exceptions, two-wildcard errors, and explicit
1 for the old scope. Update the Mac README's task-1 claims and say that default-all
behavior requires the corresponding updated Bob executable. A newer app with older Bob
must display what that Bob actually previews and submits; it must not rewrite the draft
to simulate this feature. Audit relevant source/docs/tests for stale default-to-1
statements.

## Verification and acceptance

Extend existing meaningful lexer, planner, CLI, and Swift suites rather than creating
tests that only mirror new field setters. Use a three-link fixture to distinguish “all”
from the old task-1 behavior and two-link mixed cases that distinguish wildcard
remainder from fabricated 1.

- Exercise bare alias/long-form parity with zero, one, and several numbered links;
  explicit single numbers and comma lists; all group orders; explicit exceptions;
  competing wildcards; zero, duplicate, explicit-overlap, bad-byte, overflow,
  out-of-range, and dangling-separator diagnostics and ranges.
- Cover mixed ledger markers, nested numbered links, struck/incidental/fenced
  exclusions, unresolved and guarded terminal targets, repeated links to one task, and
  conflicting duplicate outcomes. Verify actual note status changes, completion
  metadata, work logs, ledger markers, carry lists, and task rows, not just returned
  selection arrays.
- Cover whole-item closes, `@`/`^` existing-link suffixes, body-bearing new-task closes,
  same-line chains, and earlier staged link/unlink changes in a batch. Assert the
  linked/new task and all other currently numbered links are affected.
- Cover inline default and explicit logs, numbered and positional child logs, nested
  details, a first numbered link that is nested/ineligible, dropped exceptions, excess
  bullets, and empty-session log failures. Confirm logs remain literal and a single
  inline entry is not replicated across all tasks.
- Assert dry-run previews match real execution effects while leaving every vault file
  unchanged. A failing later item must roll back earlier staged changes in the daily
  note and every involved task note.
- Assert Mac parse flags and exact spans, “all” hints, narrowed numbered hints, all
  selected row outcomes/badges, completion and carry summaries, accessible text, empty
  sessions, mixed exceptions, and absent-field legacy decoding. Check that previews and
  submission pass the same unchanged draft to Bob.

Relevant Rust suites are in `capture_language/close_selection.rs`,
`capture_language/close_log.rs`, `capture_pomodoro_close/selection_tests.rs`,
`tests/cli/capture/parse_pomodoro_close.rs`, `pomodoro_close_selection.rs`,
`pomodoro_close_log.rs`, and `pomodoro_chain.rs`, plus completion/rewrite tests where
shared parsing changes their behavior. Run targeted tests during development and finish
with the repository's `just all` (`cargo fmt --check`,
`cargo clippy --all-targets --all-features`, `cargo test`).

For Bob Mac Capture, run `just format-lint`, `just build`, and `just test` with the
required macOS 26 Apple toolchain, as used by its existing CI. If the coding host lacks
that toolchain, validate the real JSON fixtures locally, record that Swift checks remain
pending, and require the existing macOS CI before merge; do not claim Linux fixture
checks are a Swift build/test pass. Use SASE's monitor workflow for any genuinely long
command that must outlive a turn.

The change is accepted when bare park/complete affects every eligible numbered link in
the current staged Pomodoro, explicit selections retain their scope, the defined
mixed/empty/log cases pass, and both clients accurately show the same resolved action
with compatible JSON and corrected documentation. The intentional behavioral migration
must be clearly called out in the implementation summary. Do not install binaries,
deploy the app, or mutate the user's vault as part of verification.
