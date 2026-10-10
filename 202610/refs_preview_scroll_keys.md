---
tier: tale
title: Scroll the Bob Refs preview with Ctrl+D and Ctrl+U
goal: Make Ctrl+D and Ctrl+U scroll the reference preview down and up by half its
  visible height while preserving search focus, query, and list selection.
size: medium
proposed_by: bbugyi200.apollo.66
status: done
---

# Scroll the Bob Refs preview with Ctrl+D and Ctrl+U

## Outcome and size

In Bob Mac Capture's Bob Refs panel, opened with Cmd+Ctrl+Shift+O, Ctrl+D scrolls the
right-hand reference inspector down by half its visible height and Ctrl+U scrolls it up
by the same amount. The commands work while the search field holds focus, preserve the
query and selected reference, and consume the keystroke before native text editing can
handle it.

This is a **medium tale**: one coding agent can implement the bounded key-routing,
SwiftUI scrolling, documentation, and macOS regression work in one repository. It
requires no independently delivered phases or bob-cli contract changes. Half-height
movement follows the conventional Ctrl+D/U interaction; the user specified direction but
did not specify distance.

## Repository access and evidence

All implementation paths below are relative to **bob-mac-capture**. Use `/sase_repo` and
open `bob-mac-capture` with an audit reason before reading or changing it. During
planning the configured linked opener failed because its primary checkout was absent;
the supported fallback
`sase repo open gh:bobs-org/bob-mac-capture -r "Implement Bob Refs preview scroll keys"`
successfully opened the repository. Use the path printed by the opener, read any
applicable AGENTS.md, and inspect status before editing. Do not hard-code another
agent's checkout path.

Planning examined commit `131e377e0d89af5d9fe8823b9fac0a2f94563250`:

- `Sources/BobMacCapture/Refs/RefsKeyRouter.swift` has no D/U key cases. Ctrl+J/K and
  Ctrl+N/P move list selection; Page Up/Down also move the list.
- `Sources/BobMacCapture/Refs/RefsPanel.swift` installs a local key monitor gated on the
  Refs panel being key. It repairs orphaned search focus, routes keys through
  `RefsKeyRouter`, and consumes events when `RefsPanelModel.perform` returns true.
- `Sources/BobMacCapture/Refs/RefsPanelModel.swift` defines `RefsCommand` and performs
  navigation and other actions; it has no inspector scroll command or delivery path.
- `Sources/BobMacCapture/Refs/RefsInspectorView.swift` wraps its non-lazy column in a
  vertical SwiftUI `ScrollView` with no programmatic position. Its `previewMode`
  deliberately renders a static column for design fixtures.
- `Sources/BobMacCapture/Refs/RefsPanelView.swift` hosts list and inspector beside each
  other. At panel widths below 760 pt it omits the inspector. Inspector content hydrates
  asynchronously for the selected reference.
- `Package.swift` requires macOS 26 and excludes the AppKit app and its tests on Linux.
  `justfile`, `Scripts/xcode-swift.sh`, and `.github/workflows/ci.yml` provide the macOS
  build and validation path.

The source establishes missing wiring, rather than a reproduced runtime failure.
Planning ran on Linux without Swift; no live Mac behavior or app tests were executed.
Recheck the relevant source against the implementation checkout before applying the
plan.

The audited `decisions:mac-capture-is-a-thin-client` memory assigns presentation and
keyboard handling to the Mac app. This change stays there; capture grammar, subprocess
contracts, vault data, global hotkey registration, and capture-editor Ctrl+U behavior
remain outside the change.

## Interaction contract

1. Exact Control+D and Control+U invoke down/up inspector scrolling in both browse and
   filtered results. Plain letters and combinations adding Command, Option, or Shift
   retain existing routing. Follow the router's existing modifier-mask convention. While
   the search field has marked IME text, pass these keys through to the input method.
2. Each accepted key press, including native key repeat, requests half of the **current
   inspector viewport height**, not half the panel or screen. Clamp to the content's
   valid vertical range. Use immediate movement so repeats and direction reversals do
   not queue overlapping animations.
3. Keep the current first responder, search selection/caret, query, scope, selected
   reference, list offset, and panel visibility intact. The ordinary search-focused
   workflow must need neither a click in the preview nor a mouse hover over it. Preserve
   existing list navigation and open shortcuts.
4. Once recognized outside IME composition, consume the command even at the top/bottom
   or when content is short, empty, unavailable, not yet laid out, or the inspector is
   absent. These cases are harmless no-ops and must never fall back to deleting search
   text or scrolling the left list.
5. Base a command after trackpad/mouse scrolling on the actual current offset. Rapid
   identical presses and alternating directions must not be lost through coalesced view
   updates or stale geometry.
6. Scroll requests are transient. A different selected reference or a fresh presentation
   starts at the top and discards pending offset state. Re-showing after an open error
   through `represent()` retains the current presentation. Hydration of the same
   reference must not reset its position; changed content/viewport sizes update bounds
   and clamp when necessary. Keys pressed while the inspector is absent must never
   replay on its return.

## Implementation

### 1. Route distinct inspector commands

Add a small, typed inspector direction and a `RefsCommand` case such as
`scrollInspector(direction)`. Add the D/U hardware key codes (D = 2, U = 32) and
exact-Control/IME conditions to `RefsKeyRouter`.

In `RefsPanelModel.perform`, send each scroll command through a dedicated non-replaying
Combine `PassthroughSubject` and return true, including with no selected row. Expose
only the publisher to the view. Keep this stream separate from list `RefsMove` and from
persistent/published library state. Do not encode commands as an optional direction
observed with `onChange`: successive presses of the same key must all be delivered.

Keep the existing panel-local monitor as the event entry point. If needed for
integration tests, extract its event-processing body into a narrow internal method that
the production monitor also calls. Preserve the key window guard and existing focus
repair; add no global monitor or hotkey.

### 2. Attach native scrolling to the inspector alone

Pass the publisher from `RefsPanelView` to `RefsInspectorView`, subscribing only in its
live scroll-view branch. Give existing standalone design fixtures a harmless default
empty publisher if necessary. Keep `previewMode` static and free of live scrolling
machinery.

Use SwiftUI `ScrollPosition`, `.scrollPosition`, and `.onScrollGeometryChange`
**directly on the inspector's ScrollView**. These APIs are available below the app's
macOS 26 deployment floor. Apple documents offset scrolling in
[ScrollPosition](https://developer.apple.com/documentation/swiftui/scrollposition) and
[scrollTo(y:)](<https://developer.apple.com/documentation/swiftui/scrollposition/scrollto(y:)>).
The
[geometry modifier](<https://developer.apple.com/documentation/swiftui/view/onscrollgeometrychange(for:of:action:)>)
selects the first scroll view if applied to a shared ancestor, so applying it to the
panel or two-column container would target the wrong pane.

Keep position and an equatable geometry projection local to the inspector; track offset,
content extent, viewport extent, and insets needed for bounds. Use one coordinate
convention with explicit inset normalization. With zero insets, the target is current
offset plus/minus viewport height / 2, clamped to 0 through max(content height minus
viewport height, 0). Respect the native inset-aware bounds for nonzero insets and ignore
invalid or zero-height geometry. Let a small local scroll-state helper centralize the
target/bounds logic so its edge cases can be tested.

Reconcile actual geometry with an outstanding keyboard target: successive commands
before a layout update accumulate from the outstanding target; user scrolling takes over
from the measured offset. Do not blindly replace a newer target with a delayed geometry
callback. Use native scroll-phase information if needed to distinguish user movement
from keyboard settling. Keep geometry updates local so trackpad movement does not
republish the entire panel model. Avoid a new general-purpose scrolling abstraction.

Tie the inspector scroll state to the selected reference and `presentationCount`,
clearing it when that identity changes or the view disappears. Maintain the existing
inspector loading callbacks and preserve same-reference content hydration. Do not save
offsets to disk.

### 3. Document the binding

Update the Bob Refs keyboard table in `README.md` to describe Ctrl+D/U as half-preview
down/up movement while retaining search focus. Clarify that Page Up/Down still page the
results list. Document the preview-hidden no-op and the IME exception briefly beside the
table. No footer redesign or new settings are needed for this focused change.

## Regression coverage and validation

Add focused tests that prove behavior, not just the new enum cases:

- Extend `Tests/BobMacCaptureTests/RefsKeyRouterTests.swift` for both directions,
  browse/search contexts, exact modifiers, plain-letter fallthrough, and marked-text
  fallthrough. Retain list paging coverage.
- Extend `RefsPanelModelTests.swift` to subscribe to the real command publisher, verify
  ordered delivery of repeated and alternating requests, and verify true/consumed with
  no subscriber/no selection. Snapshot query, scope, selection, listing, and banner to
  prove scroll commands do not mutate them or trigger open/refresh operations.
- Add `RefsInspectorScrollTests.swift` for concrete expected offsets: a 400 pt viewport
  over 1400 pt content moves 0 -> 200 -> 400 -> 200, clamps at 0 and 1000, and remains
  at 0 for fitting content. Cover a manually changed offset, viewport resizing,
  inset-aware bounds, invalid geometry, rapid commands before geometry catches up,
  same-reference hydration, and identity/disappearance reset without stale replay.
- Include a macOS hosted-view test with **both live scroll views** and overflowing
  inspector content. Drive the actual routed command path while the search field is
  first responder; observe inspector geometry changing in the expected direction while
  the left list, selected ID, text/caret, and first responder stay unchanged. Verify an
  event is consumed at a boundary and with the inspector hidden, and passes through when
  Refs is not the key panel. Use bounded layout/run-loop settling and existing
  fixtures/fake services. Static `ImageRenderer` screenshots cannot prove this because
  `previewMode` removes the scroll views.

On macOS 26 with the supported Apple toolchain, run focused tests first:

```sh
./Scripts/xcode-swift.sh test --filter RefsKeyRouterTests
./Scripts/xcode-swift.sh test --filter RefsPanelModelTests
./Scripts/xcode-swift.sh test --filter RefsInspectorScrollTests
just format-lint
just build
just test
```

Fix failures caused by this change. Use the existing macOS CI workflow for the complete
bundle/launch validation when delivering through the normal repository workflow. Linux
core tests cannot validate this AppKit/SwiftUI change; report macOS checks as pending if
that execution environment is unavailable, rather than claiming runtime success. Follow
`/sase_monitor` for long-running checks or CI waits; do not leave an unmonitored command
running at turn end.

When an interactive Mac session is available, smoke-test the built app: open Bob Refs
with Cmd+Ctrl+Shift+O, search for a reference with a long preview, press and hold each
key, reverse direction, and reach both ends. Scroll manually then use the keys again.
Change selection while details hydrate, reopen the panel, and exercise a short/empty
preview. Verify the query/caret and selected row remain unchanged by scrolling, mouse
scrolling still works, and the separate Capture editor retains its Ctrl+U behavior.
Exercise the below-760-pt layout in the hosted test if the actual screen cannot
naturally produce it. Record any unavailable manual checks honestly.

The work is ready when the bindings reach the live right-hand preview, repeat and
boundaries behave as specified, existing navigation/editing behavior is preserved,
documentation matches, and the macOS validation results are recorded. This plan adds no
CLI options, memory changes, or vault mutations.
