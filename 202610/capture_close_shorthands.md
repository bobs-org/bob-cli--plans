---
tier: tale
title: Capture close shortcuts with clear task defaults
goal:
  Support =* and =! close aliases with task 1 defaults consistently across Bob capture,
  editor interfaces, and the Mac app's live preview and submission.
size: medium
proposed_by: bbugyi200.athena.0vf
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0vf](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0vf.md)
- **COMMITS:**
  - [f589d07](https://github.com/bobs-org/bob-cli/commit/f589d0783cb1d63a2194d5a8e6b8da7e8282d512)
    — feat(capture): add =\* and =\! close shorthands defaulting to task 1

# Capture close shortcuts with clear task defaults

## Outcome and scope

Add `=*` and `=!` as first-class spellings of the existing Pomodoro close operations in
bob-cli and Bob Mac Capture. `=*<N>` means `=x*<N>`; `=!<N>` means `=x!<N>`. An omitted
park or complete list selects task 1, in both the short and long spellings. Keep the
draft exactly as typed and show its resolved effects immediately in the existing close
preview.

This is a medium tale for one implementation agent. The work is substantial but bounded:
extend the shared Rust grammar and its contract tests, then update the Swift
presentation, real CLI fixtures, and integration tests. These are sequential steps
within one tale, not independently scheduled phases. No new close planner, task status,
CLI option, transport, or JSON schema version is needed.

Implement in the primary bob-cli checkout and the linked `bob-mac-capture` repo. Before
accessing the latter, use `/sase_repo` and
`sase repo open bob-mac-capture -r "Implement capture close shorthand presentation and contract tests"`;
use the path it returns and follow any local instructions. Do not rely on this planner's
checkout paths. Keep implementation and verification in fixture vaults; this plan does
not call for editing the live vault or installing the app.

## Evidence and architectural constraints

- `src/native/capture_language/close_selection.rs` shares selection lexing between
  execution and the editor, but its recognition helpers currently require `x`, and
  trailing `*` / `!` are incomplete states. Internal empty `*` is rejected; an internal
  empty `!` currently behaves as an omitted list.
- `item.rs::session_equals_token` recognizes only `x` / `X` as closes; `draft.rs` uses
  this classification to split session chains and attach Work Log entries. Recognition
  must change there as well as in the selection lexer.
- `tokens.rs`, `editor_classify.rs`, `editor_parse.rs`, and `editor_pomodoro.rs` contain
  assumptions that a close prefix occupies two bytes. Aliases have a one-byte `=` prefix
  followed by the outcome sigil.
- `capture/pomodoro_close.rs::selection_from_spec` already maps normalized selections
  into the existing atomic planner. Reuse this path unchanged unless a test exposes an
  assumption about raw spellings.
- In Bob Mac Capture, `CapturePomodoroClosePresentation` builds its numbered rows and
  outcome summary from Bob's response. `CapturePanelModel.closePendingTrim` only trims
  placeholders when Bob reports a pending need. The app already has teal park, green
  complete, and pink session categories, plus accessible parked rows and a single
  teaching/summary/pending caption under the lineup.
- Follow the accepted `decisions:mac-capture-is-a-thin-client` and
  `decisions:task-lanes-are-sticky` records (read through `/sase_memory_read`). Grammar,
  selection defaults, numbering, preview, and mutations belong to Bob; Swift renders its
  results. Parking remains ordinary In Progress work without carry, not a new task
  status. The `[/]` status name remains In Progress.

## Product contract

### Spellings and task selection

`N` is a one-based Task Link index, not a count of Pomodoros or tasks. The existing
comma-separated list syntax remains available. For a running session whose first two
numbered links are ordinary plain links:

| Typed input               | Equivalent explicit input | Meaning                                                          |
| ------------------------- | ------------------------- | ---------------------------------------------------------------- |
| `=*`, `=*1`, `=x*`, `=X*` | `=x*1`                    | Close, park task 1; other plain links defer                      |
| `=!`, `=!1`, `=x!`, `=X!` | `=x!1`                    | Close, complete task 1; other links retain their ledger outcomes |
| `=*2,3`                   | `=x*2,3`                  | Close and park tasks 2 and 3                                     |
| `=!2,3`                   | `=x!2,3`                  | Close and complete tasks 2 and 3                                 |
| `=*!2`                    | `=x*1!2`                  | Park 1 and complete 2                                            |
| `=!2*`                    | `=x!2*1`                  | Complete 2 and park 1                                            |
| `=!~2`, `=x!~2`           | `=x!1~2`                  | Complete 1 and drop 2                                            |
| `=x2*`                    | `=x2*1`                   | Continue 2 and park 1                                            |
| `=x0!`                    | `=x0!1`                   | Complete 1, defer other eligible links                           |

Apply one consistent rule: a present `*` or `!` group with no digits before the next
outcome sigil or token end means `[1]`. An absent group remains absent. This
deliberately changes `=x!~2` from an empty complete group to completing 1, and `=x*!2`
from an error to parking 1 and completing 2. Document and test both. Default before
validating overlaps. Never choose the next unused number or the linked/new task
automatically: `=*!`, `=x1*`, `=x1!`, `=*~1`, and `=!~1` are conflicting selections and
must fail.

The shortcut simply omits `x` before an initial `*` or `!`. All further selection groups
retain their existing order independence and validation. Plain `=x`, `=X`, `=x0`,
`=x<N>`, and existing explicit valid lists retain their meaning. Neither the unmarked
in-progress list nor `~` gains a default. In particular, `=~2` still starts a session
while dropping link 2; it is not a close alias, and trailing `=x~` remains incomplete.

Parking retains normal work history and In Progress / PENDING lane effects, suppresses
carry for the selected links, and activates the existing defer-rest selection behavior
(including the existing exception for embedded links). Complete-only selections retain
ledger outcomes for unlisted links. Preserve this distinction throughout help and
preview copy.

### Every existing close context

Support these spellings through the shared grammar wherever an explicit close selection
already works:

- Whole-item captures, argv and stdin batches, and same-line chains such as `+2 =*`,
  `=! =`, `=* =#bugs`, and `=! =~2`.
- Inline Work Log entries (`=* wrote the tests`, `=! shipped it =`) and child entries
  (`=*` followed by `- 1 wrote the tests`), including chains. Retain current
  loggable-task validation, inline index defaults, and attachment rules.
- Existing-task `@route:id=*` / `^route:id=!` suffixes, and new-task
  `Draft docs @route:id=*2`. Numbers, including default 1, refer to the **post-link
  lineup**; a newly linked task may be last, not first.
- Retain existing restrictions on close suffixes for project notes, named destinations,
  scheduling/priority markers, clipboard markers, and forced destinations. The aliases
  must not bypass those checks or become starts.

Whitespace retains its existing meaning: `=*2` selects task 2, whereas a number after a
space belongs to the inline Work Log entry. `=* 1` needs entry text; it does not mean
another selection spelling. Completion continues to operate in Work Log wikilinks as
before.

### Invalid and incomplete input

Claim whole-item tokens beginning `=*` or `=!` as close syntax even when their list is
malformed. `=*abc`, `=!0`, `=**2`, and `=!1,,2` must produce close diagnostics, never
ordinary task creation. Preserve numeric parsing limits and existing duplicate,
repeated-sigil, overflow, overlap, and out-of-range checks.

Only an entirely omitted `*`/`!` group defaults. Empty comma elements never do: `=*,2`
and `=!,` are errors; `=*1,` and `=!2,` remain incomplete with `pomodoro_close_task` and
a placeholder over the final comma. `=*~` remains incomplete for `~` while retaining the
resolved park selection `[1]`. Valid `=*`, `=!`, `=x*`, and `=x!` have no pending need
or placeholder.

No numbered links means that even an implicit 1 is out of range. Report the usual useful
error, quote the actual input, and suggest plain `=x` to just close the empty session.
No running session uses the existing no-running diagnostic. Both errors must leave every
file untouched, including in a mixed batch.

Do not claim mid-body prose (`Plan =*2`, `Plan =!`, `a=*2`) or unrelated ordinary equals
shapes (`=xx`, `==`). Preserve existing start forms, toggle suffixes, and prose
recognition. No new named-close shortcut is introduced; malformed named aliases get a
close error, never a named start.

## JSON, spans, and implementation design

1. Add a small shared close-token classifier in `close_selection.rs` that exposes the
   selection slice and prefix length for long and short spellings, including a plain
   long close. Replace or rename the `*_after_x` helpers as appropriate and update all
   callers. Have `session_equals_token` and the whole-item/link recognition paths
   consume the same classification. Preserve legacy near-miss recognition where it
   intentionally differs from valid input. Do not preprocess the draft by inserting `x`
   or `1`: that corrupts source positions, raw values, completion replacement ranges,
   and undo behavior.
2. Update `lex_close_selection` / `parse_selection_body` so empty `*` and `!` groups
   produce a parsed value of 1 before shared validation, with the source range of their
   actual sigil. Keep trailing comma and `~` handling distinct; retain validation
   precedence for malformed numeric lists. Defaulting belongs in this lexer, not in the
   vault planner or Swift.
3. Audit execution and editor call sites in `item.rs`, `tokens.rs`,
   `editor_classify.rs`, `editor_parse.rs`, `editor_pomodoro.rs`, `draft.rs`,
   `close_log.rs`, `completion.rs`, and `rewrite.rs`. Replace only close-specific
   two-byte assumptions with classifier-derived lengths. Ensure caret links ending in
   `=!` are recognized as closes before any trailing-toggle check. Keep log and chain
   item ranges relative to the original draft.
4. Keep schema version 1 and the existing modes, fields, need names, and span kinds.
   `pomodoro_close.raw` retains the exact `=`-prefixed token/suffix, including case and
   omitted digits. Normalized arrays carry defaults: `=*` has `in_progress: null`,
   `park: [1]`, `complete: []`; `=!` has `in_progress: null`, `complete: [1]`, with the
   existing empty-field omission rules. `task_links[].source` is `listed` for a
   defaulted selection. No new wire field for an implicit number is necessary.
5. For `=*`, emit `pomodoro_close` over `[0,1)` (`=`) and `pomodoro_close_park` over
   `[1,2)` (`*`). For `=!2`, emit close over `[0,1)` and complete over `[1,3)`. `=x*`
   retains close `[0,2)` and park `[2,3)`. Use the corresponding absolute offsets in
   links, batches, indented lines, and Unicode-containing drafts. Spans must be
   nonoverlapping, in bounds, and valid UTF-8 boundaries. Never manufacture a span for
   an untyped `1`; point an implicit-number conflict at the real outcome sigil.
6. `capture-complete` returns no candidates within these operators/suffixes; route/block
   completion before the suffix preserves its exact bytes. Existing capture rewrite
   operations preserve aliases/default omission. Human parse output combines typed input
   with normalized meaning, for example `=* (parked 1 · defer the rest)` and
   `=! (complete 1)`.

## Mac experience

Retain the current close card and its hierarchy: session title and timing, numbered task
rows with actual transitions, one outcome caption, then next session information. Enter
still submits through the existing Close action.

- An omitted index must be visible in the resolved outcome: `Parked 1` or `Complete 1`,
  plus the other outcome groups Bob reports. The chosen row uses its filled number
  badge. Parked text stays readable and unstruck, with the existing teal pause accent
  and `Parked · not carried` caption. Complete uses the existing green treatment.
  Continue to announce task number, text, actual status, and outcome to VoiceOver;
  meaning must not depend on color alone.
- Give the shorter syntax prominence in the existing teaching caption shown before a
  selection. Lead with `=* parks 1 · =! completes 1`; for lineups of two or more, follow
  with `· add numbers for other tasks (e.g. =*2)`. Preserve the existing
  continue/drop/defer/log examples after that short lead, using only indices present in
  the lineup. Suppress the hint with no numbered links. This also avoids the current
  two-link hint's out-of-range `~3` example. Use structured hint tokens: `=` pink,
  `*`/its list teal, `!`/its list green; prose secondary. Keep it one wrapping caption
  region, with no new panel or extra repeated explanation once the actual selection
  summary is shown.
- Let parse metadata drive readiness. Bare aliases and long-form defaults immediately
  get an actual dry-run and enable Close only after success. `=*1,` retains the existing
  pending preview behavior and disables submission. Transitions between valid, pending,
  invalid, and no-running drafts must clear stale summaries and must not submit a
  trimmed preview draft.
- Preserve raw draft text in subprocess argv, the editor, caret, undo history, and final
  submission. Do not implement a shortcut expander or list parser in Swift.
  `closePendingTrim` may retain support for `*`/`!` placeholder metadata returned by
  older Bob; updated Bob simply no longer sends that metadata for the newly valid
  defaults. Keep a deliberate legacy-response test rather than treating stale fixture
  output as current behavior.
- Keep existing tolerant optional-field decoding and unknown-schema rejection. Ship/use
  the CLI change before the app update. Document that these spellings and defaults
  require an updated Bob binary; an old Bob may treat aliases as task text or long-form
  defaults as incomplete. Do not claim the updated app makes an older binary understand
  them, or introduce a new capability protocol for this bounded feature.

Relevant app files: `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`,
`Sources/CaptureCore/CaptureModels.swift` (comments/decoding only if needed),
`Sources/BobMacCapture/CapturePanelModel.swift`,
`Sources/BobMacCapture/CapturePanelView.swift`, and
`Sources/BobMacCapture/CaptureEditorPalette.swift`. Prefer proving that the existing
model/view/palette already work over unnecessary UI refactoring.

## Implementation and verification sequence

### 1. Land the grammar contract and meaningful Rust coverage

Implement the classifier/defaulting and call-site changes above. Extend the existing
unit suites in `capture_language/close_selection.rs` and
`capture_language/tests/{grammar,editor_modes,editor_spans,chain,completion,rewrite}.rs`
as relevant. Use table-driven coverage rather than copying entire tests.

Require these groups of assertions:

- All equivalences in the product table, explicit multi-index lists, marker
  permutations, uppercase long forms, and mixed default/explicit groups.
- Errors and incomplete states above, including implicit/explicit overlaps, numeric
  overflow, duplicate indices, repeated sigils, zero, and unchanged
  start/drop/ordinary-prose behavior.
- Execution/editor agreement; exact raw values, normalized arrays, span slices, and
  diagnostics. Include a multibyte body before a task suffix and a later item after a
  multibyte draft item to detect byte/character offset mistakes.
- Chain segmentation and log ownership, actual typed replacement ranges, completion
  silence in aliases, suffix preservation on completion/rewrite, and all existing
  context restrictions.

Extend CLI integration coverage in
`tests/cli/capture/{pomodoro_close_selection,parse_pomodoro_close,pomodoro_chain,pomodoro_close_log,complete_editor,rewrite}.rs`
and adjacent suites as needed. On independent identical temporary vault copies with
fixed `BOB_NOW`, compare alias/default/explicit runs: the resulting file bytes and
semantic preview fields must match; compare raw spellings and their input-dependent
ranges separately, rather than erasing arbitrary fields. Cover whole-item, existing/new
linked-task post-link numbering, logs and chains, ordinary and embedded unlisted links,
empty lineup, no running session, an out-of-range index, and a batch whose earlier
adjustment/link/create is rolled back when a later shorthand close fails. Assert dry-run
changes no vault files.

### 2. Generate the contract fixtures and complete Mac support

Generate parse, completion, and dry-run JSON fixtures with the changed Bob binary
against deterministic temporary vaults. Store a focused set under the Mac repo's
`Tests/Fixtures/`, including default park/complete, explicit alias, linked suffix, mixed
selection, inline/chain log, incomplete comma, conflict, and no-running/out-of-range
responses. Use those real responses in `Tests/Fixtures/fake-bob`; do not fake a parser
by rewriting alias text.

Update `Tests/CaptureCoreTests/CapturePomodoroClosePresentationTests.swift` for decoded
defaults, raw values, exact spans, selected rows, outcome summaries, hint text/colors
for one/two/many links, and accessibility. Replace obsolete current-behavior fixtures
such as the `=x1*` incomplete fixture with real new responses (that input now
conflicts); retain clearly labeled historical fixtures only where an older-response
compatibility test needs them.

Extend `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` to exercise preview and
submission with exact original argv for bare and explicit aliases and long defaults.
Assert Close readiness, no completion picker, no accidental pending trim, stale-card
clearing, and blocked submission for malformed or genuinely incomplete drafts. Cover a
chain and a linked suffix. Reuse the existing span/palette validation infrastructure to
prove correct coloring and valid Unicode byte ranges rather than duplicating grammar
assertions in Swift.

### 3. Document and verify the feature as a whole

Update `docs/capture.md` grammar table, quick reference, selection rules, Work
Log/chaining examples, editor/JSON/completion contract, and error examples. Update the
relevant help text in `src/native/capture/cli.rs`, `src/native/capture_parse.rs`, and
`src/native/capture_complete.rs`, plus nearby source comments that still call every
trailing `*`/`!` incomplete. Include quoted shell examples (`bob capture '=*'`,
`bob capture '=!2'`) to avoid shell globbing/history expansion. Update the Mac README's
requirements, close examples, preview behavior, editing states, and CLI-first
compatibility note. Explain that omitted indices mean task 1, also on linked-task forms;
document the deliberate mixed-empty-group behavior change.

Run the focused Rust tests during development, then the repo's required `just all`
(`cargo fmt --check`, `cargo clippy --all-targets --all-features`, `cargo test`).
Validate fixture JSON and `bash -n Tests/Fixtures/fake-bob`. For Mac verification use
the selected Apple toolchain via `Scripts/xcode-swift.sh`: `just format-lint`,
`just build`, `just test`, and `just bundle` on macOS 26 / the existing macOS CI
workflow. Do not report Swift checks as passed from a Linux-only host; arrange macOS
execution through the project's authorized workflow or clearly report the outstanding
platform gate.

Manually inspect one- and several-task previews in light/dark appearance at the narrow
supported panel width, including VoiceOver, these transitions: `=` → `=*` → `=*2` →
`=*2,` → `=*2,3`, and `=!` → `=!0` → `=!1`. Check that valid defaults never flash a
persistent pending instruction, numbering is visible, the caption wraps without
obscuring the Close action, parked and completed states remain distinct, and error
recovery clears stale results. Exercise submit only against the disposable fixture
vault.

Use `/sase_monitor` for long-running checks per session instructions. Record which
checks ran, their results, and any macOS-only verification still pending.

## Acceptance criteria

The table's shorthand and explicit spellings produce identical vault effects, timing,
work logs, carry decisions, and task transitions. Both CLI parsers, completion, rewrite,
dry-run, and execution agree on the same normalized meaning while preserving the input.
The Mac app previews default task 1 with the existing visual language and submits the
untouched draft through one Bob capture call. Invalid and incomplete drafts cannot
mutate the vault. Existing explicit valid captures and old JSON decoding remain
compatible, with the documented empty-`*`/`!` interpretation changes covered by tests.
Required checks and platform-specific verification are reported accurately.
