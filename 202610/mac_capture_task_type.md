---
tier: tale
title: Present task-type markers in Bob Mac Capture
goal:
  Complete bob-cli-ee.4 with task-type painting, completion, previews, and verified
  compatibility.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ee.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-ee.4
status: done
---

- **PARENT:**
  [202610/capture_type_marker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_type_marker.md)
- **BEAD:**
  [bob-cli-ee.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ee/bob-cli-ee.4.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ee.4](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ee.4.md)
- **COMMITS:**
  - [68aa63a](https://github.com/bobs-org/bob-mac-capture/commit/68aa63a1536b677f4ed57d8492864665a3bb73f8)
    — feat(capture): present t:\<type\> menu, pills, and relocation preview

# Present capture task-type markers in Bob Mac Capture

Implement and finish **bob-cli-ee.4** against the landed bob-cli contract. Typing
`Fix the parser t:` opens a compact task-type menu; choosing FIX inserts Bob's
replacement, paints the marker name as a type pill, and previews the type change and any
ledger relocation. Existing capture submission remains one Bob transaction.

## Scope, decisions, and current state

The normative parent is `plan:202610/capture_type_marker.md`, especially sections 2 and
3 and phase `mac`. Read it through `sase artifact read`, and read the assigned phase
through `sase bead read bob-cli-ee.4 -r "Need current scope and prior work"`. The parent
decisions are final:

- `menu_order=config`: retain Bob's returned configured order and initially select row
  zero. Without configured types, Bob supplies history order.
- `unknown_types=warn`: typed unknown names stay selectable and use Bob's warning and
  similarity data. Do not implement the rejected vocabulary-refusal policy.
- `memory_record=no`: edit no memory. A `PROPOSED FOLLOW-UP:` for the skipped decision
  record has already been appended to this phase; do not duplicate it.

At planning time the phase is in progress, its grammar/existing/completion dependencies
are closed, and it has no earlier implementation or remaining-work notes. bob-cli
includes `0e178061` (existing-task typing and relocation) and `03a3497f` (task-type
completion). The linked Mac repository is clean at `2f9a70a`; searching its Sources and
Tests finds no `task_type` or `type_relocation` support. Recheck current history before
coding to preserve anything that lands meanwhile.

This is a medium tale because the grammar, ranking, mutation, and relocation algorithms
already exist in Bob. One coding agent can extend the existing Swift decoders, menu,
painter, preview, and test fixtures. No new subsystem or child epic is needed. Only the
plan is authored in this planning turn.

Use `/sase_repo` to open `bob-mac-capture` and read its `AGENTS.md` before editing. All
app changes below are relative to that returned checkout. Bob's existing Rust
implementation and its documentation are contract evidence, not a request to redesign
them. Use `/sase_memory_read` for the thin-client decision, hide decision, task-type
glossary, docs rules, and tailnet host details when needed.

## 1. Decode the landed contract and generate truthful fixtures

Extend `Sources/CaptureCore/CaptureModels.swift` additively, preserving schema 1,
existing initializer call sites, and older Bob payloads:

- `CaptureCompletionResponse.subject` is optional (`new` or `existing`).
- `CaptureCompletionCandidate` gains task-type `current`, `inherited`, `effect`, and
  optional `refused`. Reuse its existing `key`, `hidden`, `themes`, `origin`,
  `suggestionLabel`, `similarTo`, usage, and `cursorAfter` fields. `current` is distinct
  from the existing Pomodoro `is_current` property.
- Capture items gain optional `typeRelocation` with `outcome`, `from`, and `to`.
- `CaptureTaskThemeChange.action` already accepts a string; support `kept` in its
  documentation and presentation. Context, span kind, and needs also already use
  strings; recognize `task_type` without introducing an unnecessary closed enum.
- Use `decodeIfPresent` with the established nil/false defaults for new data and
  tolerate unknown additive values without breaking the whole response.

Use the actual Rust shapes in `src/native/capture_complete/model.rs`,
`src/native/capture_complete/task_types.rs`, and capture output, plus
`tests/cli/capture/complete_task_types.rs` and `type_marker_existing.rs`, to verify the
wire details. In particular `task_theme` still contains `theme`, not `type`; completion
effects are `set/clear/keep`, whereas capture actions are `set/cleared/kept`.

Add `Scripts/generate-task-type-fixtures.sh`, following the existing relation-label
generator. Require an explicit executable `BOB_BIN` built from the landed CLI; isolate
vault, config, cache, date, and day file so no personal vault or journal is used. Seed
configured types in intentional nonalphabetical order, a destination area with
`task_type: BUILD`, existing tasks with and without explicit fields, and ledger entries
sufficient for relocation examples. Store real JSON under `Tests/Fixtures/` with
task-type names, keeping existing shared theme vectors intact.

Capture representative parse, completion, dry-run, and commit results: empty
leading/trailing/child markers; prefix and literal names; configured, hidden-exact,
history-only, and typed unknown candidates; new and existing subjects; set, clear, and
kept outcomes; and all five relocation outcomes. Commit operations in this generator may
affect only the scratch vault. Document inputs and fixture provenance. Extend
`Tests/Fixtures/fake-bob` with deterministic task-type responses for the hosted tests,
without changing the behavior of unrelated scenarios.

## 2. Add portable presentation models

Extend `Sources/CaptureCore/CompletionRowContent.swift` with a `taskType` semantic
category and a task-type row branch. Add
`Sources/CaptureCore/CaptureTaskTypeMenuPresentation.swift` for menu header, badges,
usage caption, consequence footer, and accessibility copy. Keep copy and state decisions
testable without AppKit; Swift renders server fields rather than calculating type
inheritance or parsing marker syntax.

- Header: `TASK TYPE` and the supplied subject context, such as `dev suggests BUILD`,
  `this task is FIX`, or `no default here`. Preserve the supplied note hue for its area
  dot and the type reference for its swatch/pill.
- Row: name, query emphasis, type swatch, inherited suggestion label when applicable,
  `Current` for an existing subject's current row, then `Pinned`, `Exempt`, and `Hidden`
  as indicated. Usage caption is `7 days`, `never used`, or `today` from the existing
  usage convention. Do not change Bob's candidate ordering.
- Typed row: divider, dashed unconfigured pill, `Not in your types`, and
  `did you mean FIX?` from `similar_to`. It remains selectable under the final warn
  policy. Decode a future `refused` value and safely suppress acceptance when actually
  present, without inventing that value locally.
- Footer from the selected candidate's effect:
  `this task becomes FIX · dev suggests BUILD`,
  `task type cleared · dev suggests BUILD`, or `stays BUILD · from dev`. Correctly
  distinguish an existing explicit type from its inherited suggestion, including an
  explicit null inherited suggestion.
- Accessibility includes the type name, hue, consequence, and inheritance. The list says
  `Task type suggestions, N results`; meaning must survive without color.

Extend `CaptureTaskThemePresentation` with kept-action copy using Bob's effective source
label. Add a shared ledger presentation mapping:

| Outcome              | Copy                                         |
| -------------------- | -------------------------------------------- |
| `rename_entry`       | `BUILD → FIX · running entry renamed`        |
| `append_component`   | `BUILD → BUILD + FIX · running entry shared` |
| `move_to`            | `moved under FIX`                            |
| `rename_placeholder` | `BUILD → FIX · planned entry renamed`        |
| `create_below`       | `moved under new FIX`                        |

Use supplied names; do not reconstruct placement or elapsed time. Absent optional fields
preserve existing UI, and an unknown outcome must not crash rendering.

## 3. Connect editor painting and the inline menu

In `Sources/BobMacCapture/CapturePanelModel.swift`:

- Generalize `paintHighlighting` to apply existing ThemeRef painting to every span
  carrying themes, rather than just the two Pomodoro name kinds. Respect Bob's UTF-8
  subranges and existing malformed-range guards. Type names get normal foreground text
  over the tint; `t:` and surrounding quotes stay tertiary via
  `CaptureEditorPalette.swift`. Unknown names remain neutral while the caret is in the
  name, then receive the existing dashed warning treatment.
- Include `task_type` in `visibleThemeNotConfiguredWarnings`, completion needs, and
  completion span kinds. Use the existing 50 ms debounce and server context; add no
  Swift marker-position parser. Leading, trailing, and child-line markers open the menu,
  including a bare `t:`; mid-line prose remains prose as Bob reports.
- Preserve separator-toggle ordering and prove that typing the colon in `t:` survives
  `noteSeparatorToggleIntent` and `capture-rewrite` unchanged. Keep the completion
  request from acquiring an additional debounce/rewrite cycle.
- For task-type match emphasis use the server's decoded `query`. The current
  `completionQueryText()` slices the whole replacement range, which now includes `t:`
  and quotes; do not use that raw slice as a type-name query.
- Retain row-zero selection for this context. Accept Return/Tab using Bob's whole token
  `replacement` and `cursor_after`, including leading spacing and literals. No name
  normalization or cursor arithmetic beyond the existing fallback belongs in the app. A
  refused row consumes acceptance without submitting the draft.
- Extend `selectedHistoryThemeCandidate` and existing Ctrl+D handling to configured and
  history task-type rows. Reuse `setPomodoroThemeHidden` for writes. The effective
  visibility must combine the server's `hidden` with a draft-scoped local override: a
  row returned already hidden must offer Show and actually unhide on the first press.
  Preserve selection and the ghost row through hide/show until the draft changes. Typed
  ordinary rows do not write visibility preferences. Acceptance of a hidden row never
  automatically unhides it. Preserve Pomodoro-menu behavior.

In `CapturePanelView.swift`, give `CompletionList` a task-type rendering branch using
the presentation model, existing `PomodoroThemeSwatchView` and pill styling. Use
approximately 30 pt single-line rows, an 8 pt swatch, and up to eight visible rows with
scroll-to-selection. Size to actual row count; seven types fit without scrolling. Adapt
both the list and its caller's current fixed 5×48 viewport so the header and
consequence/footer are visible. Other completion contexts keep their existing geometry.
Footer keycaps are `⏎ choose` and `⌃D hide/show` when applicable. Retain existing click,
Ctrl+N/P, Escape, focus, and stale-response protections.

## 4. Render previews and success notifications

Add a shared Type row with the ThemeRef pill and presentation copy to every preview that
can carry `task_theme`: standard new tasks, Pomodoro-linked new tasks, existing
links/toggles, and relationship-only existing owners. Use the actual capture item, not
syntax inference, to decide whether the row exists. Extend the existing
`CapturePomodoroLinkPresentation` and `CaptureTogglePresentation` paths without showing
duplicate type rows.

Render the optional ledger relocation beside that item's type information. Extend
preview accessibility/announcements with the same text. Add `type FIX` to successful
set/kept notification bodies through the existing notification assembly, preserving
bulk-item behavior and avoiding duplicate text on link/toggle notifications. Cleared
actions retain their accurate cleared copy. Old Bob responses render as before.

## 5. Tests and user documentation

Add focused CaptureCore tests for real fixture decoding and older payload defaults; all
effect/source/header/footer combinations; configured order preservation; row badges and
usage; quoted-name query emphasis; unknown warnings/similarity; kept copy; all
relocation outcomes; and nil/unknown optional data.

Add hosted tests alongside `CaptureEditorHighlightTests`,
`CaptureUnconfiguredThemeHostedTests`, and `CapturePanelModelTests` for:

- Marker name tint versus dim sigil/quotes, Unicode offsets, unknown caret-entry and
  caret-exit treatment, stale/malformed spans, and existing Pomodoro painting.
- Bare/leading/trailing/child completion, an adjacent route token, and mid-line prose;
  the colon rewrite path must not alter or delay the type completion flow.
- Return/Tab replacement and exact cursor placement for slug and literal rows; keyboard
  navigation, Escape, initial index zero, and refused-row no-op handling.
- Ctrl+D hide/show from both an initially visible and initially hidden row,
  configured/history eligibility, nonwriting typed rows, error handling, and draft
  changes clearing ghost state. Keep existing Pomodoro visibility tests green.
- Compact menu layout, header and selected consequence footer, Type/ledger preview
  coverage, and success notification text. Use bounded waits for async tests.

Add a render test using `RenderFixtureWriter` for the menu and typed/relocation preview
in light and dark appearance so CI produces reviewable images. Inspect actual renders
when available; do not present synthetic renders as manual smoke.

Update the Mac repository's `docs/protocol.md`, `docs/capture-panel.md`,
`docs/keyboard.md`, and `docs/capture-previews.md`: additive fields and compatibility,
marker highlighting and completion, consequences and unknown warnings, Type/ledger rows,
and Ctrl+D in both type and existing Pomodoro-name menus. Keep paths stable. Do not
change CLI versions, changelogs, shared theme vectors, or memory notes.

## Verification and completion

1. Generate fixtures with the landed Bob binary, verify source provenance, and run the
   focused portable tests while implementing. Format only changed Swift lines.
2. Run the Mac repo's `just portable-check` (lint all Swift; compile/test portable
   targets). If Swift is missing locally, use a disposable copy on `athena` after
   reading tailnet memory. New Swift must be compiled before landing.
3. Run `just docs-check` in the Mac repo with `BOB_CLI_DIR` explicitly set to the active
   bob-cli checkout, so the wrapper uses the authorized checkout rather than locating or
   cloning another repo. Include script syntax validation for the new fixture generator.
4. Follow Mac `AGENTS.md`: attempt bounded SSH access to `mac`; rsync a disposable
   scratch copy excluding `.build`/`.git`; run `Scripts/bundle.sh --identity -` and
   `just typecheck-tests`; then remove the scratch copy. Never use Bryan's checkout or
   `just install`. CLT has no XCTest; hosted tests run in macOS CI. Record exact
   compilation results. If offline, explicitly report **macOS targets unverified**,
   retaining the successful portable compilation evidence.
5. Verify the resulting Mac CI Build, Test, and Bundle gates, not just lint. Capture the
   revision and any failing test names. Use `/sase_monitor` for long commands or CI
   waits and finish the handoff command itself before ending a turn.
6. When the Mac and graphical session permit, smoke the changed scratch app with the
   installed compatible Bob: choose FIX for `Fix the parser t:`, leading `t:`, `- t:fix`
   on a child line, existing linked `^route:id t:fix`, an unconfigured name, and Ctrl+D
   hide/show. Use an isolated fixture vault for captures and preference changes. Record
   menu and preview screenshots through SASE artifacts. Report any unavailable
   smoke/screenshots precisely; never claim them as passed.
7. Record verified work and test/CI evidence on **bob-cli-ee.4**. Record discovered
   unrelated work only as `PROPOSED FOLLOW-UP:` notes on this phase; create no beads. A
   failure reproduced identically on the clean base is such a follow-up with evidence
   (and an existing tracking bead if known), not a reason to leave completed
   implementation open.
8. Run `sase bead epic-symbols bob-cli-ee.4` immediately before closing. There were no
   entries during planning; resolve any new entries or re-key their justfile lines to an
   appropriate still-open parent/later bead. Once the phase is complete, run
   `sase bead close bob-cli-ee.4 --note "<implementation and verified checks>"`. Close
   only this assigned phase; never the parent epic or an ancestor plan bead. Do not
   hand-edit status. Finalize all modified repositories through `/sase_final`.
