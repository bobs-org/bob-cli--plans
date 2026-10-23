---
tier: tale
title: Move the Bob Refs view reset from Ctrl-Z to Ctrl-C
goal:
  In Bob Mac Capture's Bob Refs panel, Ctrl-C resets the view to fresh-launch defaults
  and Ctrl-Z is no longer bound, with router, hosted tests, code comments, and mounted
  docs all agreeing.
size: small
proposed_by: bbugyi200.athena.130
proposed_with: claude/opus
create_time: 2026-10-23 05:42:36
status: wip
---

# Plan: Move the Bob Refs view reset from Ctrl-Z to Ctrl-C

## Goal

In Bob Mac Capture's Bob Refs panel, Ctrl-Z resets the view: query, scope, filters,
selection, revealed pages, and list and preview positions go back to fresh-launch
defaults. Move that keymap to **Ctrl-C**. This is a move, not an alias: after the change
Ctrl-C resets and Ctrl-Z is unbound in Bob Refs. Unbound means it passes through to the
search field, like Ctrl-B does today. Nothing else about the reset changes.

All work is in the linked **bob-mac-capture** repo. Open it with
`sase repo open bob-mac-capture -r "<reason>"`, read its `AGENTS.md`, and make every
edit in the path that command prints. bob-cli itself, bob-plugins, and SASE memory need
no changes. A sweep found no bob-cli docs or memory note that mentions the Refs Ctrl-Z
reset; the `Ctrl+Z` mentions in bob-cli's docs are about Obsidian undo and are
unrelated.

## Current wiring (for orientation)

- `Sources/BobMacCapture/Refs/RefsKeyRouter.swift` is a pure key-code router. Its
  `KeyCode` enum declares `z: UInt16 = 6`. The `case KeyCode.z:` branch returns
  `.resetView` when `flags == .control` (exactly Control, so Caps Lock or any extra
  modifier gives `nil`) and returns `nil` while `context.markedTextPresent` is true (IME
  passthrough). The doc comment above `command(keyCode:modifiers:context:)` says
  "Control-Z resets the view."
- `RefsPanelController.processKeyEvent` (`Refs/RefsPanel.swift`) runs the router. On
  `.resetView` it calls `cancelPendingViewRestores()`, then `model.perform`. It is keyed
  on the command, not the key, so it needs **no change**.
- The Link to Today, Rename, and Copy cards catch keys before the router
  (`RefsCore/RefsTodayLinkCard.swift`, `RefsRename.swift`, `RefsCopy.swift`). None of
  them binds Ctrl-C; the Copy card just consumes it. Card behavior is unchanged: Ctrl-C
  does nothing while a card is open, exactly as Ctrl-Z does today.
- Ctrl-C is not used anywhere in the Bob Refs router today, so there is no conflict. In
  the **Capture** panel, Ctrl-C means "stash draft and close"
  (`CaptureKeyCommandRouter`). That panel has its own local monitor, which only acts
  while the Capture panel is the key window (`CapturePanelController.swift`), so it
  cannot catch Ctrl-C while Bob Refs is key. Leave Capture's Ctrl-C alone. The Edit
  menu's Copy item is Command-C, not Control-C, so it is unaffected.

## Changes

### 1. Router (`Sources/BobMacCapture/Refs/RefsKeyRouter.swift`)

- In `private enum KeyCode`, replace `static let z: UInt16 = 6` with
  `static let c: UInt16 = 8` (ANSI virtual key code for C; Capture's router uses the
  same value). Drop `z`, since nothing else uses it.
- Change the `case KeyCode.z:` branch to `case KeyCode.c:` and keep its body exactly as
  it is: `flags == .control`, IME passthrough, `.resetView`. Place the branch where it
  reads naturally, e.g. next to the other Control shortcuts. There is no
  `case KeyCode.z` any more, so Ctrl-Z falls to `default: return nil`.
- Update the doc comment: "Control-C resets the view."

### 2. Comments that name the key

- `Sources/BobMacCapture/Refs/RefsPanelModel.swift`:
  - the `RefsPresentationSource.highlightsTabs` doc comment ("as after Ctrl-Z") becomes
    Ctrl-C;
  - the `case .highlightsTabs:` comment in the presentation switch ("as after Ctrl-Z")
    becomes Ctrl-C.

### 3. Tests

- `Tests/BobMacCaptureTests/RefsKeyRouterTests.swift`:
  - Add `static let c: UInt16 = 8` to the test `KeyCode` enum. Keep `z` for the
    assertions that Ctrl-Z now passes through.
  - Rename `testControlDAndUScrollAndControlZResets` to
    `testControlDAndUScrollAndControlCResets`. Switch its three
    `route(KeyCode.z, .control, …) == .resetView` assertions (browse, search, and
    Tabs-row contexts) to `KeyCode.c`. In each context, add
    `XCTAssertNil(route(KeyCode.z, .control, context: …))` so a regression back to
    Ctrl-Z fails.
  - In `testControlScrollAndResetRequireExactModifiers`, move the exact-modifier block
    from `KeyCode.z` to `KeyCode.c`. These cases must route to `nil`: plain C,
    Command-C, Shift-Command-C, Control-Shift-C, Control-Option-C, Control-Command-C,
    and Control-Caps-Lock-C. Assert `route(KeyCode.c, .control) == .resetView`. Keep
    plain Z and Command-Z as `nil`, and change
    `XCTAssertEqual(route(KeyCode.z, .control), .resetView)` to
    `XCTAssertNil(route(KeyCode.z, .control))`.
  - Rename `testControlDUZPassThroughWhileComposing` to
    `testControlDUCPassThroughWhileComposing` and change its `KeyCode.z` line to
    `KeyCode.c`.
- `Tests/BobMacCaptureTests/RefsInspectorScrollTests.swift`:
  - Rename `testHostedControlZResetsPreviewAndQuery` to
    `testHostedControlCResetsPreviewAndQuery`. Replace the private `zKey: UInt16 = 6`
    with `cKey: UInt16 = 8` (only this test uses `zKey`). Send
    `keyEvent(keyCode: Self.cKey, modifiers: .control, characters: "c")`.
  - In the same test, after the Ctrl-D presses and before the Ctrl-C press, add a
    live-controller regression check. Send Ctrl-Z (key code 6, characters "z") through
    `harness.controller.processKeyEvent` and assert:
    - the event is returned (`XCTAssertNotNil`, meaning it passed through);
    - `harness.model.query` is unchanged;
    - the inspector offset is still `> 100`.

    Use a local constant or literal for key code 6. Then the existing Ctrl-C assertions
    run as before.

- `Tests/BobMacCaptureTests/RefsListScrollHostedTests.swift`: in the class doc comment,
  "Ctrl-Z resets the list to its top" becomes Ctrl-C. The test itself calls
  `perform(.resetView)` directly and needs no change.

Tests that call `model.perform(.resetView)` directly (in `RefsPanelModelTests.swift` and
elsewhere) are key-agnostic. Leave them alone.

### 4. Docs (mounted into Bob's site)

Replace each Refs-reset mention of Ctrl-Z with Ctrl-C and keep the wording otherwise
unchanged:

- `docs/keyboard.md`, "Bob Refs shortcuts" table:
  - the `Ctrl-Z | Reset the query, filters, …` row becomes `Ctrl-C | …`, in the same
    position;
  - the `Command-T (in Highlights)` row's "(as after Ctrl-Z)" becomes Ctrl-C;
  - leave the Capture editor table's `Control-C` stash row as it is.
- `docs/refs.md`:
  - the view-memory paragraph: "as Ctrl-Z resets" and "Ctrl-Z resets it to the
    fresh-launch defaults";
  - the key table row `Ctrl-Z | Reset query, scope, …`;
  - the paragraph under the key table: "Ctrl-D, Ctrl-U, Ctrl-Z, and Ctrl-X also pass
    through while IME-composing. Ctrl-Z always resets the view …";
  - the Highlights-takeover bullet: "from fresh-launch defaults, as after Ctrl-Z";
  - the ordinary-shows bullet: "Ctrl-Z resets the view to fresh-launch defaults."
- `docs/refs-inspector.md`: "A new-reference reset clears them like Ctrl-Z." becomes
  Ctrl-C.

Do not add a migration note. These pages describe current behavior.

### 5. Final sweep

From the bob-mac-capture root, run
`grep -rniE "ctrl[-+ ]?z|control[-+ ]?z|⌃z" Sources Tests docs README.md`. The only
remaining hits should be the new Ctrl-Z passthrough assertions and comments, plus the
unrelated `^zzz` fixtures in the capture tests and `Tests/Fixtures/fake-bob`. Also
confirm nothing in `Sources/` (footer hints, actions menu, help strings, Settings) shows
a ⌃Z reset hint. The investigation found none, so there should be nothing to change.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

1. **Portable gate (athena has Swift):** `just portable-check` from the bob-mac-capture
   root. This parse-lints every Swift file with `swift format lint`, including the
   changed `Sources/BobMacCapture` and `Tests/BobMacCaptureTests` files, and builds and
   tests the portable targets. It must pass.
2. **macOS targets (best effort):** `portable-check` cannot type-check
   `Sources/BobMacCapture*` or `Tests/BobMacCapture*`. Copy the repo to a scratch
   directory on the `mac` tailnet host with rsync, excluding `.build` and `.git`. Run
   `Scripts/bundle.sh` and then `just typecheck-tests` there, then delete the scratch
   copy. Never use Bryan's checkout or `just install` on the mac. If the mac is
   unreachable (it is often offline), say in the final report that the macOS targets are
   unverified. CI still runs the XCTest suites.
3. **Docs gate:** `just docs-check` from the bob-mac-capture root. When run from the
   linked-repo checkout, set `BOB_CLI_DIR` to the root of the bob-cli checkout that
   hosts it so the gate uses that tree and not a cache clone. It must pass.
4. Re-run the grep sweep from step 5 of Changes.

## Commit

Commit only in bob-mac-capture, one commit, following the repo's conventional-commit
style, for example:

`feat(refs): move the Bob Refs view reset from Ctrl-Z to Ctrl-C`

The body should say that Ctrl-C now resets the query, scope, filters, selection,
revealed pages, and list and preview positions, and that Ctrl-Z is unbound in Bob Refs.
