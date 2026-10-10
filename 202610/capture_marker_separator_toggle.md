---
tier: tale
title: Toggle capture block-ID markers with the opposite separator
goal:
  Let typing an adjacent caret or colon switch plain capture markers in Bob Mac Capture
  while preserving the ID, caret, and native editing behavior.
size: medium
proposed_by: bbugyi200.apollo.6g
create_time: 2026-10-10 12:40:00
status: wip
---

# Toggle capture block-ID markers by typing the other separator

## Outcome and scope

In Bob Mac Capture, typing `^` immediately after `@file:id` changes the marker to
`@file^id`; typing `:` immediately after `@file^id` changes it back to `@file:id`. The
newly typed character is consumed, the existing separator changes, and the caret remains
immediately after the unchanged ID. The user can switch repeatedly without moving back
into the marker.

This is a **medium tale**: one implementation agent can deliver the bounded Rust
typing-assist rule, Mac integration, documentation, and regression tests. The two
repositories share one contract and do not need independent phases. No implementation
changes were made while preparing this plan.

## Established architecture and source map

Follow the accepted `decisions:mac-capture-is-a-thin-client` memory: bob-cli owns
capture syntax and returns edits; Swift owns input events, subprocess requests,
presentation, and native editing. Read that memory through `sase memory read` before
implementation. There is no need for another CLI command, flag, daemon, or grammar
implementation in Swift.

Relevant bob-cli sources, inspected at `8e6d946`:

- `src/native/capture_language/rewrite.rs`: `rewrite_draft`, `DraftRewrite`,
  `RewriteRule`, `TextEdit`, and edit application/cursor helpers. Currently implements
  bare-`@@` absorption only.
- `src/native/capture_language/{tokens,markers,editor_parse,editor_classify,model}.rs`:
  shared marker validation, token boundaries, source offsets, and contextual marker
  selection. Plain caret and colon markers already have different meanings, and their
  suffix families are not interchangeable.
- `src/native/capture_rewrite.rs`: the lexical, vault-independent
  `capture-rewrite --cursor BYTE --format json` API, schema version 1.
- `src/native/capture_language/tests/rewrite.rs`, `tests/cli/capture/rewrite.rs`, and
  `docs/capture.md`.

Relevant bob-mac-capture sources, inspected at `4c22d09`:

- `Sources/BobMacCapture/CapturePanelModel.swift`: `editorTextDidChange`,
  `editorSelectionDidChange`, `isBareAtAtTrigger`, `startCaptureRewrite`,
  `applyCaptureRewrite`, `invalidateRewrite`, and `scheduleAnalysis`. Rewrite requests
  already use an immediate independent process lane, but only `@@` triggers them.
  Application checks the draft text, not selection identity. The current generic rewrite
  path replaces `attributedDraft`.
- `Sources/BobMacCapture/CapturePanelController.swift` and
  `CaptureKeyCommandRouter.swift`: main-editor key events and native `NSTextView`
  insertion patterns, including composition guards.
- `Sources/BobMacCapture/CapturePanelView.swift`: text and selection callbacks.
- `Sources/CaptureCore/{BobProcessClient,CaptureModels,CaptureTextRanges}.swift`:
  existing rewrite transport, string-valued rule decoding, and validated UTF-8 ranges.
  No incompatible schema change is needed.
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`,
  `Tests/CaptureCoreTests/{CaptureModelTests,BobProcessClientTests}.swift`,
  `Tests/Fixtures/fake-bob`, and `README.md`.

Use `/sase_repo` to open bob-mac-capture before reading or editing it. During planning,
the configured `sase repo open bob-mac-capture` failed because its source checkout was
absent; the audited fallback
`sase repo open gh:bobs-org/bob-mac-capture -r "Implement the capture marker separator toggle"`
succeeded. Use the path printed for the implementing agent, never a path copied from
another agent's checkout. Check both trees for existing changes and recheck these source
locations against their current revisions.

## Behavior contract

In this table `|` denotes the caret, not literal text.

| Before key        | Key      | Result |
| ----------------- | -------- | ------ | ----------------- | -------- |
| `Do work @file:id | `        | `^`    | `Do work @file^id | `        |
| `Do work @file^id | `        | `:`    | `Do work @file:id | `        |
| `@file:id         | `        | `^`    | `@file^id         | `        |
| `@file^id         | `        | `:`    | `@file:id         | `        |
| `@file:id         | Do work` | `^`    | `@file^id         | Do work` |
| `Do work @file:id | s:2`     | `^`    | `Do work @file^id | s:2`     |

The solo-marker cases intentionally work during drafting even if the result still needs
body text before submission. Do not require a successful vault lookup, an existing ID, a
valid complete capture, or a settled live preview. Both manually authored IDs and IDs
accepted from the picker work; the picker currently leaves the caret immediately after
its bare ID.

Use the existing route and block-ID validation and marker-position rules. Preserve
route/ID spelling and case, whitespace, line endings, body text,
schedule/priority/dependency markers, and other items byte-for-byte. A trailing marker
on an authored child line is eligible where the shared grammar already recognizes it as
the item's destination. Only the marker ending at this caret changes.

The assist is deliberately limited to the two complete, plain `@` forms:

- The new key must be adjacent to the ID, with no intervening whitespace. Typing after a
  space or newline remains ordinary input.
- The appended key must end the token: end of input or a following whitespace boundary.
  Inserting it inside an ID does not toggle a prefix.
- Incomplete or malformed routes/IDs, same-separator repetitions, `@@`, `@!`, standalone
  `^route:id`, `&note:id`, `!note:id`, and project-task `:id`/`^id` markers are outside
  this rule.
- Markers with `#name`, `+`, or `=...` suffixes are outside this rule, including
  inserting the key immediately before a suffix. Their meanings do not have a lossless
  plain colon/caret conversion. Do not strip suffixes.
- Prose, URLs, literal/code text, and marker-looking tokens that the existing contextual
  parser does not select remain unchanged. Do not fix unrelated parsing behavior as part
  of this work.

This remains an editor assist, not capture execution syntax. Calling `capture` or
`capture-parse` on an unrewritten `@file:id^` must keep its existing behavior. The
rewrite command is the explicit way to obtain the corrected draft.

## Implementation

### 1. Add the Rust rewrite rule and document its contract

Add `RewriteRule::SwitchBlockIdSeparator`, serialized as `switch_block_id_separator`, to
the existing rewrite dispatch. Keep the JSON schema at version 1 and reuse the existing
response fields.

For this rule require an explicit cursor. Examine only the token ending at that cursor
and only when its last character is `^` or `:`. Remove that final character in a
temporary candidate draft, then use the shared marker parsers/context selection to
establish that the preceding text is a plain marker with the opposite separator. Check
the exact marker shape as well as its context; accepting every colon/caret parser result
would also accept the suffix families excluded above. Do not require the whole item or
draft to have no diagnostics, since unfinished unrelated items and solo markers are
normal editor states.

Return a replacement for the claimed token containing the same route and ID with the new
separator and without the appended trigger. A single token replacement is sufficient and
makes native application straightforward. Its range indexes the original input; `cursor`
is the end of the replacement in the resulting draft. Return a concise summary such as
`Changed @file:id to @file^id`. Use existing UTF-8 edit helpers; never normalize the
entire draft or rebuild it from parsed bodies.

Successful output must be idempotent when passed back to `capture-rewrite`. When this
rule does not claim, preserve the current `@@` dispatch, notices, and cursor behavior.
With no cursor, retain the existing absorption-only behavior rather than searching the
draft for a toggle. Document the rule, examples, exclusions, and cursor requirement in
the command's help and `docs/capture.md`, including updating the existing two-value rule
listing.

### 2. Connect typed separators to the existing Mac rewrite lane

Keep grammar recognition in bob. The Mac side should only recognize an eligible **input
event** and forward the resulting draft/cursor; it must not use a regex or Swift
route/ID parser to decide whether a token toggles.

Use the existing key-monitor/controller path to record a one-shot edit intent for actual
`:`/`^` input in the focused main draft editor, with a collapsed selection and no
marked-text composition. Observe the produced character, not a keyboard scan code, so
Shift and keyboard layouts work. Respect modal picker/prompt routing and native command
shortcuts. Let the character insert normally, then consume the intent from the
text-change callback only if the resulting draft and caret equal the expected single
insertion. This also supplies the exact post-key cursor if the SwiftUI selection
callback lags behind the text callback. Bulk paste, selected-text replacement,
programmatic assignment, caret-only movement, and Undo/Redo must not manufacture a new
typing intent. Clear unused intents on mismatch and lifecycle changes.

Send this post-key draft to `captureRewrite` immediately, sharing the existing
transport/lane and preserving the `@@` trigger. Do not wait for parse/preview or consume
ordinary unmatched punctuation. An older bob returns `changed: false`; transport errors
keep the literal text and normal parse/completion/preview behavior.

Bind the request to a generation, process client, draft, and caret. Before applying,
require those identities to remain current, a collapsed unchanged selection, and
`response.input` matching the request. Cancel/invalidate on subsequent edits, user
selection movement, dismissal, submission, client replacement, or a newer request.
Selection notifications caused by the original key itself must not invalidate the
matching post-key caret. Drop stale results rather than rebasing them onto a different
draft. This keeps rapid continued typing intact even when a delayed assist cannot be
applied.

Validate the returned byte ranges, replacement text, and cursor before conversion to
native UTF-16 ranges. Apply the new rule through the focused `NSTextView`'s native
undoable replacement path, with a narrow controller bridge from the model if needed,
rather than assigning `attributedDraft` and losing editing history. Treat the correction
as one native edit; Undo may restore the literal post-key draft, and Redo restores the
correction. Neither should immediately retrigger the assist. Do not hold an Undo group
open across a subprocess await. Preserve the existing `@@` behavior and avoid
refactoring every programmatic editor replacement.

After an accepted correction, suppress recursive rewrite/completion callbacks, restore
the returned caret, announce Bob's summary, and request fresh parse/preview without
immediately reopening the block-ID picker. Discard analysis of the pre-correction draft
through the existing generation mechanism. Continue normal completion on the next
genuine user edit.

### 3. Add focused regression coverage and user documentation

Rust grammar and CLI tests must cover both directions, repeated alternation,
solo/leading/trailing/child-line markers, selection of one item in a batch, uppercase
and hyphenated IDs, surrounding emoji/non-ASCII text, and preserved CRLF/whitespace.
Assert original-input edit ranges, output text, caret, summary/rule, version 1,
idempotence, no-cursor behavior, and read-only operation without vault configuration.
Verify the corrected drafts have the expected parse modes with body text, while raw
capture parsing still does not silently apply the assist. Exercise the exclusions above
and retain all existing absorption tests, including a draft containing both an inactive
toggle-shaped token and a separate `@@` trigger.

Extend `fake-bob` with representative rewrite responses and delay/failure cases. Use
fixtures from the real Rust response shape. Mac model/controller tests should exercise
real input intent and native editing, not only direct model assignment. Cover both keys,
a picker-accepted ID followed by a toggle, Unicode cursor conversion, subsequent typing,
unchanged caret after a successful correction, no automatic picker reopen,
no-op/old-bob/failure responses, and malformed response ranges. Verify delayed responses
are ignored after text edits, caret moves, selection, dismissal, submission, and
changing away from and back to the same draft (generation protection). Include native
Undo/Redo and IME/selection/modal-field negative coverage. Keep existing bare-`@@`
rewrite and stale-response tests passing.

Update the Mac README's typing-assist section with both key sequences, adjacency and
plain-marker limits, and the requirement for a bob build supporting this new rule. Keep
product explanations focused on the gesture.

## Verification and acceptance

- Run focused Rust rewrite/CLI regressions during implementation, then `just check` in
  bob-cli for formatting, clippy, and all test binaries.
- Run `just format-lint`, `just build`, and `just test` in bob-mac-capture using the
  documented macOS 26 Apple toolchain. Its existing CI workflow also bundles and
  smoke-tests the app. Linux core tests do not validate AppKit input, Undo, or the
  SwiftUI integration. The planning host has no Swift executable; report any unavailable
  macOS validation explicitly. Use `/sase_monitor` if waiting for long commands or CI is
  necessary.
- On macOS, type `Do work @file:id`, append `^`, append `:`, and repeat. Verify the ID
  stays unchanged and the caret stays at its end; repeat after selecting an ID from the
  picker and with emoji earlier in the draft. Check Undo/Redo and normal continuation
  with a space and body text. Check a suffix-bearing marker and moving the caret during
  a delayed reply.
- Both core transformations must work without an extra key, picker choice, or moving the
  caret backward. Only bob decides whether the marker is eligible. The assist itself
  never submits a capture or mutates the vault.

Deliver the coordinated changes in both repositories. Existing capture semantics, ID
generation, task status, and other marker families stay under their existing contracts;
no memory updates or unrelated follow-up work are part of this tale.
