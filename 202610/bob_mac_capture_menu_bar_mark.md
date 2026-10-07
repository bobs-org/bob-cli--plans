---
tier: tale
title: Bob Mac Capture bullet-b menu-bar mark
goal:
  Bob Mac Capture's menu-bar item shows a code-drawn bullet-b template glyph instead of
  the `Bob` text, with a same-footprint "!" variant and an Open Settings menu row when
  bob is unresolved, plus a Reduce-Motion-aware pulse when a capture lands.
size: medium
proposed_by: bbugyi200.apollo.5m
create_time: 2026-10-07 15:27:58
status: wip
---

<!-- sase:links:start -->

## Links

| Relation | Artifact                               | Why                                                               |
| -------- | -------------------------------------- | ----------------------------------------------------------------- |
| related  | file:explicit:8fdf10639d359af917318ba6 | Concept sheet rendered from the exact glyph geometry in this plan |

<!-- sase:links:end -->

# Bob Mac Capture: replace the `Bob` menu-bar title with a drawn "bullet-b" mark

## Goal

Bob Mac Capture's only always-visible surface is its `NSStatusItem`. Today it is the
text `Bob` (and `Bob ⚠️` when `bob` is not resolved), set in
`Sources/BobMacCapture/AppDelegate.swift` (`configureStatusItem()` and
`updateStatusItemAppearance()`). Replace that text with a purpose-designed monochrome
**template glyph** that is distinctive, sits naturally beside the system's SF Symbol
icons, and still says "something is wrong" when `bob` cannot be resolved. Add one small
moment of delight: the bullet briefly swells when a capture lands.

All app code lives in the linked **bob-mac-capture** repo. Open it with
`sase repo open bob-mac-capture -r "<reason>"`. If that fails because the linked repo's
primary checkout is missing on this host, use
`sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` and work only in the path it
prints. One string in **bob-cli** (`scripts/install_all`) also changes.

## The design (decided; implement exactly)

A concept sheet rendered from this exact geometry is stored as SASE artifact
`file:explicit:8fdf10639d359af917318ba6`. It shows 14× construction views of both states
on the point grid, actual-size @2x strips on light, dark, and highlighted (blue) menu
bars beside mock Wi-Fi and battery icons, an @1x strip, and the landed-pulse filmstrip.

### The mark

A geometric lowercase **b**: a vertical stem plus a circular bowl, both one stroke
weight, with a **solid bullet** centered in the bowl.

- It reads as "b" for Bob at a glance, and the bowl-plus-bullet doubles as a
  target/bobber.
- The bullet is the captured thought: every capture becomes a bullet in the vault.
- It is monochrome, so it does not compete with the colorful Hammerspoon Pomodoro
  indicator (`… · 🍅 12:34`) that shares the menu bar. It deliberately avoids any tomato
  motif so the two items never read as one.

Rejected alternatives (for the record, do not implement): a fishing bobber (read as a
pendant lamp or Pokéball at 18pt), an inbox tray with a falling dot (generic, collides
with system download/inbox icons), a "b" in a rounded keycap (inner letter too small at
menu-bar size), a plain "b" with an empty bowl (reads as typography, not an icon), and a
stock SF Symbol (not ownable).

### Geometry (design space: 18×18pt canvas, y grows **downward**)

Draw with `NSImage(size: NSSize(width: 18, height: 18), flipped: true) { … }` so the
numbers below are used verbatim. Fill and stroke with opaque black and set
`isTemplate = true`; AppKit then tints for light and dark menu bars, Liquid Glass
wallpapers, and the highlighted (menu-open) state.

| Element                    | Spec                                                                                                                                                            |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stroke weight              | 1.5pt. This matches the menu bar's SF Symbol weight and lands every stem and bowl edge on whole pixels at 2×.                                                   |
| Stem                       | Line from (4.25, 3.25) to (4.25, 10.0), `lineWidth` 1.5, `.round` cap. Ink top is y = 2.5, and the stem meets the bowl tangentially.                            |
| Bowl                       | Stroked circle, center (9.0, 10.0), centerline radius 4.75, `lineWidth` 1.5. Outer radius 5.5 and inner (counter) radius 4.0.                                   |
| Bullet (ready)             | Filled circle, center (9.0, 10.0), radius 1.6.                                                                                                                  |
| Alert "!" (bob unresolved) | Replaces the bullet. Bar: line from (9.0, 7.7) to (9.0, 9.8), `lineWidth` 1.4, `.round` cap (ink 7.0–10.5). Dot: filled circle, center (9.0, 12.2), radius 0.8. |
| Ink bounds                 | x 3.5–14.5, y 2.5–15.5. The bounding box is centered in the canvas.                                                                                             |

Put these numbers in one `Metrics` namespace so code, tests, and docs share a single
source of truth.

### States

1. **Ready** (`processClient != nil`): the bullet-b. Tooltip and accessibility label are
   `Bob Mac Capture`.
2. **bob unresolved** (`processClient == nil`): the bullet becomes the "!"; the stem and
   bowl are unchanged.
   - The glyph keeps the same 18pt footprint, so other menu-bar items never shift (the
     old `Bob ⚠️` title changed width).
   - The tooltip keeps today's text:
     `Bob Mac Capture — bob is not resolved. Check Settings.`
   - The accessibility label is `Bob Mac Capture, bob not resolved`.
   - The status menu gains an explanatory, actionable first row: an enabled item
     `bob Not Resolved — Open Settings…` with SF Symbol `exclamationmark.triangle`,
     action `openSettings`, followed by a separator. Both rows are removed again as soon
     as `bob` resolves (for example after **Recheck Bob**).
   - The healthy menu stays exactly
     `Capture, Settings, Recheck Bob, —, Restart Bob Mac Capture, Quit Bob Mac Capture`.
3. **Capture landed** (transient, ready state only): after every successful submit, the
   bullet swells and settles once.
   - Radius is `r(t) = 1.6 + 1.3·sin(π·t)`, t from 0 to 1, over 420 ms in 24 equal
     frames (peak 2.9pt, leaving a 1.1pt gap to the counter).
   - The animation always ends on the resting image of the _current_ state.
   - It is skipped entirely when
     `NSWorkspace.shared.accessibilityDisplayShouldReduceMotion` is true or the state is
     not Ready.
   - A new pulse cancels an in-flight one. A state change cancels it, and the new
     state's resting image wins.

## Implementation (bob-mac-capture)

### 1. `Sources/BobMacCapture/StatusItemGlyph.swift` (new)

- Header doc comment: what the mark is and why (bullet-b, the bullet as the captured
  thought, a template image so it adapts to every menu-bar appearance).
- `enum StatusItemGlyph`:
  - `enum State: Equatable { case ready, bobUnresolved }`
  - `enum Metrics` holds every number from the geometry table, plus derived
    `counterRadius` (4.0) and `inkBounds`
    (`NSRect(x: 3.5, y: 2.5, width: 11, height: 13)`, y-down).
  - `static func image(for state: State, bulletRadius: CGFloat = Metrics.bulletRadius) -> NSImage`.
    It draws with `NSBezierPath` in a flipped drawing-handler image, returns
    `isTemplate == true`, and sets `accessibilityDescription` from the presentation
    strings. The drawing handler is resolution-independent, so it stays crisp at 1×, 2×,
    and 3× with no bundled assets.
  - Do not add image files or SwiftPM resources. `Scripts/bundle.sh` copies only the
    executable and `Info.plist`, and `Bundle.module` resources would break the
    hand-assembled, codesigned `.app`. Drawing in code is the reliability choice.
- `enum StatusItemPulse`: `duration` (420 ms), `frameCount` (24), `peakBulletRadius`
  (2.9), `static func bulletRadius(atProgress:) -> CGFloat` (clamped sin hump), and
  `static var bulletRadii: [CGFloat]` (`frameCount + 1` samples; the first and last
  equal `Metrics.bulletRadius`).
- `struct StatusItemPresentation: Equatable`, built with `init(isBobResolved:)`. Fields:
  `glyphState`, `toolTip`, `accessibilityLabel`, and `issueMenuTitle: String?` (`nil`
  when healthy). These are the exact strings from "States" above.
- `struct StatusItemFrame: Equatable`: `glyphState`, `bulletRadius`, `toolTip`,
  `accessibilityLabel`. This is the value handed to the renderer, which keeps everything
  above it testable without a real `NSStatusItem`.

### 2. `StatusItemController` (same file or `StatusItemController.swift`), `@MainActor final class`

- Init takes a render sink `(StatusItemFrame) -> Void` and the status `NSMenu`.
- Injectable seams with production defaults: `reduceMotion: () -> Bool`
  (`NSWorkspace.shared.accessibilityDisplayShouldReduceMotion`) and
  `sleep: (Duration) async -> Void` (`try? await Task.sleep(for:)`).
- `update(isBobResolved:)`:
  1. Cancel any pulse.
  2. Store the presentation.
  3. Render the resting frame.
  4. Apply the issue rows to the menu.
- `playCaptureLandedPulse()`:
  1. Return early unless the state is Ready and Reduce Motion is off.
  2. Cancel any running pulse.
  3. Start a main-actor `Task` that renders each `StatusItemPulse.bulletRadii` frame
     after the first, sleeping `duration / frameCount` between frames.
  4. Check `Task.isCancelled` before every render, and finish with the resting frame
     unless cancelled.
- Expose the in-flight pulse as an internal `pulseTask` so tests can `await` it.
- Issue rows: a static, idempotent `applyIssue(title: String?, to menu: NSMenu)`.
  - Remove every item carrying a dedicated tag constant, then, when the title is
    non-nil, insert the issue item at index 0 and a tagged separator at index 1.
  - Leave the target nil and use action `#selector(AppDelegate.openSettings)`, the same
    responder-chain routing the existing items use.

### 3. `Sources/BobMacCapture/AppDelegate.swift`

- `configureStatusItem()`:
  1. Create the item with `NSStatusItem.squareLength`.
  2. Set `button.imagePosition = .imageOnly` and an empty title.
  3. Keep `item.menu = Self.makeStatusMenu()`. Leave `makeStatusMenu()` unchanged.
  4. Build the `StatusItemController` whose sink sets
     `button.image = StatusItemGlyph.image(for:bulletRadius:)`, `button.toolTip`, and
     `button.setAccessibilityLabel(_:)`, capturing the item weakly.
  5. Render Ready immediately. `configureProcessClient()` runs right after and corrects
     the state.
- `updateStatusItemAppearance()` becomes
  `statusItemController?.update(isBobResolved: processClient != nil)`. Keep the existing
  comment about the glyph being the only always-visible surface.
- In `applicationDidFinishLaunching`, after `panelModel` is created, subscribe to
  `model.$successAnnouncementTick.dropFirst()` and call `playCaptureLandedPulse()`.
  - Store the subscription in a **new** `statusItemCancellables` set. Do not use
    `settingsCancellables`: `observeCanceledDraftStashCapacity()` guards on that set
    being empty.
  - Mirror that method's `[weak self]` sink pattern so the closure inherits main-actor
    isolation.
- `successAnnouncementTick` already increments exactly once per successful submit (in
  `CapturePanelModel`, just before `panelDismisser()`). Do not add a new model hook.

### 4. Tests (`Tests/BobMacCaptureTests/`)

New `StatusItemGlyphTests.swift`:

- Both state images are `isTemplate`, 18×18pt, and carry a non-empty, state-specific
  `accessibilityDescription`.
- Rasterize each image at 2× into a 36×36 `NSBitmapImageRep` (via
  `NSGraphicsContext(bitmapImageRep:)`; row 0 is the top row) and check:
  - every pixel with alpha > 0 lies inside `Metrics.inkBounds` scaled ×2, with 1px
    tolerance;
  - the Ready and bob-unresolved bitmaps differ, and every differing pixel lies inside
    the counter circle (radius `counterRadius` around the bowl center, +1px);
  - the canvas center column has ink in both states.
- `StatusItemPulse.bulletRadii`:
  - the count is `frameCount + 1`;
  - the first and last values equal the resting radius;
  - the maximum equals `peakBulletRadius` within 0.01;
  - every value stays at or below `counterRadius - 1.0`;
  - the values rise monotonically to the peak, then fall monotonically.
- `StatusItemPresentation` produces the exact strings for both states.
- `applyIssue`:
  - unresolved inserts the issue item (exact title, `openSettings` action, an image)
    plus a separator ahead of the unchanged six;
  - applying it twice does not duplicate;
  - applying `nil` restores exactly the six healthy items.
- `StatusItemController`, with a recording sink and `sleep` that only yields:
  - a pulse renders 24 frames and ends on the resting Ready frame;
  - Reduce Motion true renders nothing beyond the resting frame;
  - the bob-unresolved state never pulses;
  - `update(isBobResolved: false)` called right after starting a pulse leaves the alert
    resting frame as the last render, with no Ready frames after it.

Keep `testStatusMenuOffersRestartBeforeQuit` passing unchanged.

New `StatusItemGlyphDesignTests.swift`, following `PomodoroBlockDesignTests`:

- `testRenderStatusItemGlyphsToPNG` is skipped unless `BOB_MAC_CAPTURE_RENDER_DIR` is
  set.
- It writes PNGs for each state at 1×, 2×, and 3× on light, dark, and accent
  (highlighted) backgrounds, an 8× enlargement of each state, and a pulse filmstrip.
- Bryan uses these to eyeball the real AppKit rendering on his Mac.

### 5. Docs (`README.md` in bob-mac-capture)

- Add a short **Menu-Bar Icon** section, near "Runtime Contract", covering:
  - the mark and its meaning;
  - the two states and the landed pulse, including the Reduce Motion behavior;
  - that the glyph is a code-drawn template image (no assets);
  - how to render the review PNGs with
    `BOB_MAC_CAPTURE_RENDER_DIR=… ./Scripts/xcode-swift.sh test --filter StatusItemGlyphDesignTests`.
- Update every reference to the text title so prose matches the icon:
  - the Runtime Contract bullet "The `Bob` status-item menu…" becomes "The menu-bar
    icon's menu…" and mentions the conditional issue row;
  - each `Bob → Restart Bob Mac Capture` becomes
    `menu-bar icon → Restart Bob Mac Capture`;
  - Troubleshooting's "**The `Bob` menu-bar item does not appear**" becomes "**The
    menu-bar icon does not appear**";
  - the "Bob is not resolved" entry notes the "!" glyph and the menu's **Open
    Settings…** row.
- Find all of these with `grep -n "Bob →\|\`Bob\` " README.md`.

## Implementation (bob-cli)

`scripts/install_all`: the exit-72 warning detail
`restart it via Bob → Restart Bob Mac Capture` becomes
`restart it via the Bob Mac Capture menu-bar icon → Restart Bob Mac Capture`. No other
bob-cli change. The capture JSON contract is untouched, so the thin-client decision
(`decisions:mac-capture-is-a-thin-client`) is unaffected.

## Verification

- Agent hosts have no Swift toolchain. The macOS 26 CI workflow
  (`.github/workflows/ci.yml`: swift-format lint, build, test, bundle, launch smoke
  test) is the only compiler. Recent history shows CI catching compile slips, so write
  conservatively:
  - Swift 5 language mode, `@MainActor` on every AppKit-touching type;
  - only long-stable AppKit APIs: `NSImage(size:flipped:drawingHandler:)`,
    `NSBezierPath`, `NSStatusItem.squareLength`,
    `NSImage(systemSymbolName:accessibilityDescription:)`,
    `NSWorkspace.accessibilityDisplayShouldReduceMotion`;
  - swift-format default style (4-space indent, lines ≤ 100 columns).
- For the bob-cli `scripts/install_all` edit, run `just check-scripts` (`bash -n`).
- After the change lands in bob-mac-capture, confirm its CI run is green
  (`gh run list -L 3` / `gh run watch` from that checkout) and fix forward until it is.
- On the Mac (Bryan), after `just install`:
  - the bullet-b appears in place of `Bob`;
  - setting a bogus executable override and choosing **Recheck Bob** shows the "!" glyph
    and the **Open Settings…** row;
  - restoring the override clears both;
  - a successful capture produces one subtle pulse, and none with System Settings →
    Accessibility → Display → Reduce motion enabled.

## Non-goals

- No app icon (`.icns`). The app has none today, and reusing the mark for Settings and
  notifications is a separate follow-up Bryan can request.
- No change to the hotkey, the capture panel, notifications, or the menu's healthy item
  set.
- No colored (non-template) rendering in any state.
