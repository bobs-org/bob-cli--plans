---
tier: tale
title: Show only the lightbulb for Keep source links in the idle agenda
goal:
  Render Keep source links as lightbulbs throughout the current and future Pomodoro
  agenda.
size: small
proposed_by: bbugyi200.apollo.6g.w1.w0.f0.w0
create_time: 2026-10-10 14:37:08
status: wip
---

# Show only the lightbulb for Keep source links in the idle agenda

## Goal

When the capture draft is empty, Bob Mac Capture shows the running Pomodoro and every
future Pomodoro under Now, Next, and Later. A task imported from Google Keep currently
exposes its Markdown source link in that display. Render the source link as just its
existing `💡` label.

Example:

```text
Before: Call dentist [💡](https://keep.google.com/u/0/#NOTE/example "Open in Google Keep")
After:  Call dentist 💡
```

Keep the task's title, status, operator number, children, and fold controls. The
lightbulb remains a display glyph with the existing link tint; it does not acquire an
action. This is a focused presentation fix that one coding agent can complete, so use a
small tale with no phases or reviewer decisions.

## Findings and constraints

- In bob-cli, `src/native/gkeep/render.rs::render_note` emits the
  `[💡](URL "Open in Google Keep")` suffix. Its `encode_source_url` percent-encodes
  Markdown delimiters and whitespace. `docs/gkeep.md` documents this format.
- `src/native/capture_pomodoros_agenda.rs::resolve_item` passes the task description
  through to the agenda's `text` field. The source link belongs in that data and in the
  vault; hiding it is the app's presentation responsibility.
- The app was inspected at `63840fe`
  (`fix(capture): restore the idle agenda after clearing the draft`).
  `Sources/CaptureCore/CaptureAgendaInlineText.swift` parses code, wikilinks, emphasis,
  fields, and tags, but no inline Markdown lightbulb links. `parseField` consequently
  leaves this link literal.
- `Sources/BobMacCapture/CaptureAgendaView.swift::CaptureAgendaRowInlineText` renders
  those segments for task headlines, duplicate/struck rows, and agenda detail rows.
  `CaptureAgendaRowMeasurer` measures the same row views, so the existing measurement
  path can account for the shorter display automatically.
- `CaptureAgendaPresentation` builds full, no-log, and one-line task variants from the
  same raw title. Its accessibility labels also copy raw titles and notes, which need
  the same narrow lightbulb substitution.
- `CaptureAgendaPresentation.makeOneRowRow` joins titles and truncates the raw summary
  to `maxOneRowLength` before inline rendering. A long source URL can therefore become
  an incomplete link that the parser cannot collapse. Project lightbulb links before
  this truncation as well as in the inline renderer.
- Follow `decisions:idle-capture-shows-ledger-agenda` and
  `decisions:mac-capture-is-a-thin-client`: bob supplies facts; the app owns pixels.
  Preserve caching, agenda visibility, numbering, and the farthest-first fold ladder. No
  CLI/schema, capture grammar, vault, import, or memory changes are needed. Keep other
  Markdown links and the shared `TaskDisplayText` behavior.

## Implementation

1. Open the app through `/sase_repo`, using
   `sase repo open bob-mac-capture -r "Implement lightbulb-only source links in the idle agenda"`.
   If that configured checkout cannot open, try
   `sase repo open gh:bobs-org/bob-mac-capture` with a reason identifying the fallback,
   as succeeded during planning. Use only the returned checkout and read any applicable
   instructions. Recheck the named renderer against its current revision before editing.
2. Extend the agenda-only inline parser to consume a complete
   `[💡](destination "title")` or `[💡](destination)` and append just `💡` as an
   existing `.link` segment. Support the actual generated, nonempty percent-encoded
   destination and optional double-quoted title; consume the closing parenthesis and
   tooltip together. Do not decode or classify URLs, query Keep, introduce a Markdown
   library, or generalize the change to other link labels. A malformed or unsupported
   form stays literal without swallowing subsequent prose. Existing code spans and
   wikilink aliases remain opaque to this substitution.
3. Reuse that recognition for a narrow accessibility-text projection in
   `CaptureAgendaPresentation.swift`, retaining existing task numbers, statuses,
   duplicate/completed wording, and all unrelated label text. Apply it to labels built
   from affected titles and detail text. Use the same narrow projection on each title in
   `makeOneRowRow` before joining/truncating the group summary, so folding cannot expose
   a cut-off destination. Keep the snapshot and ordinary row source text intact.
   Preserve stable task/group IDs, and let the derived summary text flow through the
   existing row key and shared measurer; avoid a second ad hoc regex in the SwiftUI
   view. Keep emphasis, tags, fields, trailing block-ID handling, and Character-based
   segment ranges consistent with their current behavior.
4. Add a short sentence to the app README's Idle agenda section explaining that Keep
   source links display as a lightbulb. All implementation changes belong in
   bob-mac-capture.

## Verification

- Extend `Tests/CaptureCoreTests/CaptureAgendaInlineTextTests.swift` with the exact
  generated example and the no-tooltip form. Cover surrounding Unicode, multiple bulbs,
  a percent-encoded URL, and neighboring code/wikilink/tag/field segments; assert both
  the final text and that segment ranges tile it exactly. Check that malformed links and
  ordinary non-bulb Markdown links remain literal, and that an example inside a code
  span is preserved.
- Extend agenda presentation coverage for Now, Next, and Later, including a repeated
  task and a completed task. Check full, no-log, and one-line variants through the
  display parser and check affected accessibility labels: only the source link is
  shortened, and operator numbers and statuses survive. Include a task child or ledger
  note containing a bulb link so the shared detail path is covered. Add a group-one-row
  regression whose raw URL exceeds `maxOneRowLength` but whose projected summary fits,
  proving that truncation happens after the lightbulb substitution.
- Add one app-local `Tests/Fixtures/agenda-lightbulb.json` regression fixture based on
  the existing agenda fixtures, with realistic bulb-bearing titles across current and
  future entries. Include it in `CaptureAgendaDesignTests` and
  `CaptureAgendaHeightConsistencyTests`. Inspect the generated PNGs at 760/620 panel
  widths in light and dark appearances, and check full/folded/expanded rendering where
  task titles remain visible. The bulb must not leak its URL, tooltip, or Markdown
  brackets, and measured/planned height must still agree within the existing 2 pt
  tolerance.
- Run available CaptureCore tests on a Swift-equipped host. On macOS 26, use the
  repository's Apple-toolchain wrapper: `just format-lint`,
  `./Scripts/xcode-swift.sh build`, and `./Scripts/xcode-swift.sh test`, with
  `BOB_MAC_CAPTURE_RENDER_DIR` set to a writable scratch directory for image review. The
  existing macOS CI already runs these and uploads render fixtures. This planning host
  has no Swift executable; report that limitation accurately and use macOS validation
  for the actual app instead of claiming Linux checks prove its rendering. Hand long
  validation commands to `/sase_monitor`.

## Acceptance

Every supported Keep source link in visible idle-agenda task/detail rows renders as
exactly one lightbulb per link, including current and future sessions and their folded
task rows. Accessibility labels no longer expose that link's Markdown or destination.
The real URL remains in bob's payload and the vault. Other link forms, task semantics,
fold ordering, editor position, and capture previews retain their existing behavior.
Formatting, build, relevant tests, height checks, and macOS render review pass before
the coding agent reports completion.
