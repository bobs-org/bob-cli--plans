---
tier: tale
title: "Bob Refs: Ctrl-U scrolls the preview up again, Ctrl-Z resets the view"
goal:
  In Bob Refs, Ctrl-U scrolls the preview up by half its height and the new Ctrl-Z
  resets the remembered view to fresh-launch defaults.
size: small
decisions:
  ctrl_shift_u:
    ask:
      Should Ctrl-Shift-U keep scrolling the preview up as a second key beside Ctrl-U?
    choices:
      drop:
        Unbound again and passed through to search, as before the remember-view change;
        one key per action
      keep: Stays a silent alias for Ctrl-U's half-page-up scroll, documented beside it
    default: drop
    why:
      Ctrl-Shift-U only existed because Ctrl-U was taken; dropping it restores the
      original keymap
    answer: drop
proposed_by: bbugyi200.athena.12e.f0
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.12e.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.12e.f0.md)
- **COMMITS:**
  - [239aaef](https://github.com/bobs-org/bob-mac-capture/commit/239aaefa37d90155cfb2ac9e096b7f3b675f48ec)
    — feat(refs): restore Ctrl-U preview scroll and bind Ctrl-Z to reset

# Bob Refs: Ctrl-U scrolls the preview up again, Ctrl-Z resets the view

## Goal

The remember-view change (bob-mac-capture commit `5935822`, "feat(refs): remember panel
view across shows") moved Bob Refs' preview half-page-up scroll from Ctrl-U to
Ctrl-Shift-U so that Ctrl-U could reset the remembered view. Reverse the key half of
that trade:

- **Ctrl-U** scrolls the preview up by half its height again (the pre-`5935822`
  behavior, mirroring Ctrl-D).
- **Ctrl-Z** (new) resets the query, kind scope, Newest and Tabs filters, selection,
  revealed highlight pages, and list and preview positions to the fresh-launch defaults
  — exactly what Ctrl-U does today.

Only the key bindings move. `RefsCommand.resetView`,
`RefsPanelModel.perform(.resetView)`, `resetToDefaults(tabs:)`, and the controller's
pre-reset cancellation of the reveal follow and pending preview restore stay as they
are. Esc still closes while keeping the view, and Shift-Command-O still restores it.

## Where the work happens

All changes are in the linked **`bob-mac-capture`** repository. Open it with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<reason>"`), use the printed
path for every read and write, and read its `AGENTS.md` before editing. Nothing changes
in bob-cli itself; bob-cli's own `docs/` has no Bob Refs key references.

## Changes

### 1. Router — `Sources/BobMacCapture/Refs/RefsKeyRouter.swift`

- Add `static let z: UInt16 = 6` (kVK_ANSI_Z) to the private `KeyCode` enum.
- `case KeyCode.u`: exact `.control` returns
  `context.markedTextPresent ? nil : .scrollInspector(.up)`.

> [!decision] ctrl_shift_u = drop Delete the `[.control, .shift]` branch, so
> Ctrl-Shift-U returns `nil` and passes through to the search field like any other
> unbound chord (the pre-`5935822` shape).

> [!decision] ctrl_shift_u = keep Keep the `[.control, .shift]` branch, but it now
> returns the same `.scrollInspector(.up)` as Ctrl-U (it already does); only its doc
> comment changes to call it an alias.

- New `case KeyCode.z`: exact `.control` returns
  `context.markedTextPresent ? nil : .resetView`; every other modifier set returns
  `nil`. Match Ctrl-D/Ctrl-U's exact-flags check (`flags == .control`, so Caps Lock,
  Shift, Option, and Command variants pass through). Command-Z must keep reaching the
  search field as Undo — it must never route to `.resetView`.
- Update the `command(keyCode:modifiers:context:)` doc comment: Control-D and Control-U
  always scroll the inspector; Control-Z resets the view (mention the Ctrl-Shift-U alias
  only under `keep`).

Ctrl-Z has no default Cocoa text-system binding and no other Bob Mac Capture binding
(the Capture editor does not use it), and the Refs local key monitor runs before the
field editor, so the new key needs no other routing changes. The card key handlers in
`RefsPanelController.processKeyEvent` still run first, so Ctrl-Z inside an open Copy,
Rename, or Link to Today card behaves exactly as Ctrl-U does today (the card owns the
key; `perform(.resetView)` also no-ops while renaming or linking).

### 2. Doc comments in sources

- `Sources/BobMacCapture/Refs/RefsInspectorScroll.swift` (top of file): "Control-D is
  down, Control-U is up." (add "Control-Shift-U is an alias" only under `keep`).
- `RefsPanel.swift`'s `if command == .resetView` block needs no change; it is keyed on
  the command, not the key.

### 3. Tests

Update the tests to the new keymap; do not weaken assertions.

- `Tests/BobMacCaptureTests/RefsKeyRouterTests.swift`
  - Add `static let z: UInt16 = 6` to the test `KeyCode` enum.
  - Rename `testControlDAndControlShiftUScrollAndControlUResets` to
    `testControlDAndUScrollAndControlZResets`. In browse and search contexts assert
    Ctrl-D → `.scrollInspector(.down)`, Ctrl-U → `.scrollInspector(.up)`, Ctrl-Z →
    `.resetView`; for Ctrl-Shift-U assert `nil` (`drop`) or `.scrollInspector(.up)`
    (`keep`). Keep the closable-Tabs-row Ctrl-D assertion and add the same for Ctrl-Z →
    `.resetView`.
  - `testControlScrollAndResetRequireExactModifiers`: Ctrl-U → `.scrollInspector(.up)`;
    Ctrl-Shift-U per decision; add `XCTAssertNil` for plain Z, Command-Z,
    Shift-Command-Z, Ctrl-Shift-Z, Ctrl-Option-Z, Ctrl-Command-Z, and Ctrl-Z with Caps
    Lock (`[.control, .capsLock]`), and
    `XCTAssertEqual(route(KeyCode.z, .control), .resetView)`.
  - `testControlDAndUPassThroughWhileComposing` (rename to
    `testControlDUZPassThroughWhileComposing`): add `XCTAssertNil` for Ctrl-Z while
    composing; keep the Ctrl-Shift-U composing assertion (it is `nil` either way).
- `Tests/BobMacCaptureTests/RefsInspectorScrollTests.swift`
  - Header comment: "drives the routed Control-D/Control-U path".
  - `testHostedControlDUScrollsInspectorWhileSearchKeepsFocus`: send the up key as
    `keyEvent(keyCode: Self.uKey, modifiers: .control, characters: "u")`. Under `keep`,
    additionally send Ctrl-Shift-U afterwards and assert it scrolls up further (or stays
    at the top when already there) without changing selection or focus.
  - Add `private static let zKey: UInt16 = 6` next to `dKey`/`uKey`.
  - Rename `testHostedControlUResetsPreviewAndQuery` to
    `testHostedControlZResetsPreviewAndQuery` and send
    `keyEvent(keyCode: Self.zKey, modifiers: .control, characters: "z")`; its assertions
    (preview back at the top, empty query, first row selected) stay. Together with the
    updated Ctrl-D/Ctrl-U hosted test above — which already asserts the up key leaves
    selection, query, list offset, and focus alone — this covers the swap end to end.
    `RefsPanelModelTests.testResetViewRestoresDefaultsAndAlwaysConsumes` keeps covering
    the full reset of query, scope, Newest, Tabs, banner, and remembered offset.
- `Tests/BobMacCaptureTests/RefsListScrollHostedTests.swift`: the header comment says
  "Ctrl-U resets the list to its top" → "Ctrl-Z". The test drives `perform(.resetView)`
  directly; no code change.
- `RefsPanelModelTests` calls `perform(.resetView)` / `perform(.scrollInspector(.up))`
  directly; leave them alone except any comment that names Ctrl-U as the reset key.
  `BobMacCaptureTests.testKeyRouterMatchesControlUAsLinePrefixDeletionOnlyWithControlModifier`
  is the Capture editor's Ctrl-U and must not change.

Grep `Sources Tests` afterwards for `Control-Shift-U`, `Ctrl-Shift-U`, `ControlU`,
`Control-U`, and `Ctrl-U` and confirm every remaining Refs mention describes scrolling,
and every reset mention names Ctrl-Z / Control-Z.

### 4. Docs (mounted into Bob's site)

- `docs/keyboard.md`, "Bob Refs shortcuts" table:
  - `Ctrl-U` → "Scroll the preview up by half its height".
  - New `Ctrl-Z` row directly after it: "Reset the query, filters, selection, revealed
    pages, and scroll positions".
  - `drop`: delete the `Ctrl-Shift-U` row. `keep`: change it to "Same as Ctrl-U".
  - Leave the "Editor and completion keys" Ctrl-U row (Capture editor) untouched.
- `docs/refs.md`:
  - "Updates" paragraph: "Ctrl-Z resets it to the fresh-launch defaults".
  - Keyboard table: `Ctrl-U` → "Scroll the preview up by half its height; search keeps
    focus"; new `Ctrl-Z` row with today's Ctrl-U reset text; `Ctrl-Shift-U` row deleted
    (`drop`) or "Same as Ctrl-U" (`keep`).
  - The paragraph under the table: Ctrl-D, Ctrl-U, Ctrl-Z (and Ctrl-Shift-U only under
    `keep`), and Ctrl-X pass through while IME-composing; "Ctrl-Z always resets the view
    (even when it already has defaults), and Ctrl-D and Ctrl-U consume the key even when
    the preview is hidden, short, empty, or already at an edge."
  - Highlights-integration bullet: "Ctrl-Z resets the view to fresh-launch defaults."
- `docs/refs-inspector.md` has no Ctrl-U mention; leave it unless the grep below finds
  one.

Finally grep `docs/` for `Ctrl-U`, `Ctrl-Shift-U`, and `Ctrl-Z` and confirm each Bob
Refs mention matches the new keymap.

## Verification

Never land Swift you have not compiled (bob-mac-capture `AGENTS.md`). Run all of these:

1. `just portable-check` on a host with Swift (athena, which has it) — formatter lint
   plus the portable targets' build and tests.
2. macOS targets on the Mac, best effort, exactly as `AGENTS.md` describes: rsync a
   scratch copy of the bob-mac-capture checkout to a fresh `/tmp` directory on `mac`
   (excluding `.build` and `.git`), run `./Scripts/bundle.sh` and `just typecheck-tests`
   there, then delete the scratch copy. Never use Bryan's own checkout on the Mac or
   `just install`. The Mac has no XCTest, so the Refs tests are type-checked only there;
   CI runs them. If the compile takes longer than a couple of minutes, hand it to your
   `/sase_monitor` skill rather than waiting in the foreground. If the Mac is offline,
   say in the final report that the macOS targets are unverified.
3. `just docs-check`. The previous run of this gate had two pre-existing Material-theme
   failures unrelated to Bob Refs; report any failure honestly and distinguish those
   from anything this change introduces.

## Acceptance

- In Bob Refs, Ctrl-U scrolls the preview up by half its height with search keeping
  focus, and never touches the query, filters, selection, or list position.
- Ctrl-Z resets the view to the fresh-launch defaults (the current Ctrl-U behavior,
  including cancelling the reveal follow and pending preview restore), passes through
  while IME-composing, and Command-Z still undoes in the search field.
- Ctrl-Shift-U follows the `ctrl_shift_u` decision.
- Esc, Shift-Command-O, the remembered-view behavior, and the Capture editor's Ctrl-U
  line deletion are unchanged.
- Router, hosted, and model tests, source doc comments, and `docs/keyboard.md` /
  `docs/refs.md` all describe the same keymap.
