---
tier: tale
title: Scroll Bob Mac Capture Settings and page it with Ctrl+D / Ctrl+U
goal:
  The Settings window fits on screen, scrolls by trackpad, and pages by half a viewport
  with Ctrl+D / Ctrl+U from any field, with unclipped labels and fully wrapped captions.
size: medium
decisions:
  motion:
    ask:
      Should Ctrl+D/U glide to each half page instead of jumping like the Bob Refs
      preview?
    choices:
      glide:
        A ~0.2 s ease-out glide that shows where the content went; instant under Reduce
        Motion
      jump:
        An instant jump, identical to the Refs preview; no animation or pending-target
        state
    default: glide
    why:
      Motion keeps your place in a 13-section form, and a held key still pages smoothly
    answer: glide
  hint:
    ask:
      Should the Settings window show a slim pinned footer that reads "⌃D Down  ⌃U Up"?
    choices:
      footer:
        Pinned footer under a soft scroll-edge blur; the keys are discoverable in place
      none: No footer; only the Keyboard docs teach the keys
    default: footer
    why:
      Nobody guesses vim keys in a native Settings window; Bob panels teach keys in
      footers
    answer: footer
proposed_by: bbugyi200.athena.1h0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-27 08:13:48
status: wip
---

# Bob Mac Capture Settings: scroll it, and page it with Ctrl+D / Ctrl+U

## Problem

Bryan's screencast (`~/tmp/screencasts/20261127_075808.mp4`, 15 s) opens **Bob Mac
Capture Settings** from the menu-bar item. The window is taller than the screen and runs
off its bottom edge. Bryan presses ⌃D three times and ↓ four times. Nothing moves, so
Snoozed Tasks, Similar Tasks, Typing Tips, Go To Notes, Stash, Templates, Launch,
Notifications, and Diagnostics can't be reached.

Root cause, in `Sources/BobMacCapture/SettingsView.swift` (bob-mac-capture): the body is
a bare columns-style `Form` with `.padding(20).frame(width: 560)` and **no scroll
view**. The SwiftUI `Settings` scene (built by `BobSettingsScene` in
`AppDelegate.swift`) sizes the window to the form's full ideal height and makes it
non-resizable. So the trackpad, the scroll wheel, and the keys all fail. The same frames
show two more defects that scrolling will put on screen:

- Long labels such as "In Highlights, open Bob Refs with" are clipped at the window's
  left edge. Some captions run off the right edge. The columns form needs about 540 pt
  but gets 520 pt.
- Several captions are cut off mid-sentence with "…". Examples: "Command-\ is global:
  while this is on, other apps never receiv…" and "Runs bob ref scan -a -w once
  Highlights finishes saving.…". These are warnings Bryan needs to read in full.

## Design

All of this work is in the linked **bob-mac-capture** repo. It is app-owned
presentation, so no `bob` contract changes (decisions:mac-capture-is-a-thin-client).
Open the repo with `/sase_repo` (`sase repo open bob-mac-capture`) and read its
`AGENTS.md` before editing.

### The window

- **It scrolls.** A vertical `ScrollView` wraps the form. Trackpad, scroll wheel, and
  scroller dragging start working, which is half of Bryan's complaint on its own. The 20
  pt padding moves _inside_ the scroll content, so the overlay scroller hugs the
  window's trailing edge instead of floating 20 pt in.
- **It fits the screen.** The window opens at about 820 pt tall. It is never taller than
  the current screen's `visibleFrame` minus a 64 pt margin, and never shorter than 420
  pt. Compute this ideal height once, when the view is created (for example in an
  `@State` initial value). If `body` recomputes it, a later re-evaluation can resize the
  open window.
- **It resizes vertically** between 420 pt and unbounded, and its width stays fixed. Use
  `.frame(minHeight:idealHeight:maxHeight: .infinity)`. The `Settings` scene's automatic
  resizability is `.contentSize`. If the window still won't resize, add
  `.windowResizability(.contentSize)` to `BobSettingsScene`.
- **It is 620 pt wide, up from 560.** That leaves room for the longest Highlights labels
  next to the widest toggle ("Show today's Pomodoros when the draft is empty"), with
  margin. Nothing may clip at either edge.
- **Captions wrap and are never truncated.** Add one private `View.settingsCaption()`
  helper: `.font(.caption)`, `.foregroundStyle(.secondary)`, and
  `.fixedSize(horizontal: false, vertical: true)`. Replace the repeated font/style pairs
  on every caption `Text` with it. Pin `.formStyle(.columns)` so moving the form into a
  scroll view can't change its look.

### The keys

| Key    | Action                                 |
| ------ | -------------------------------------- |
| Ctrl-D | Scroll down by half the visible height |
| Ctrl-U | Scroll up by half the visible height   |

These are the same half-viewport semantics as the Bob Refs preview (`docs/refs.md`) and
vim. The rules:

- **Focus doesn't matter.** Settings opens with the Executable override field focused
  (blue ring in the screencast), so the keys must work from inside text fields. They
  take precedence over AppKit's Emacs Ctrl+D (delete forward) in Settings' fields, as
  Ctrl+D already does in the Refs search field. Focus never moves.
- **IME composition passes through.** While the first responder is an `NSTextView` with
  marked text, both keys go to the text system, as in Refs.
- **Only the exact chord.** Ctrl alone plus D (key code 2) or U (key code 32), matched
  by key code like `RefsKeyRouter`. Ignore Caps Lock, numeric-pad, and function flags.
  Shift, Option, or Command added to the chord passes it through.
- **Only the Settings window.** The key must belong to the Settings window while it is
  key with no attached sheet. Confirmation dialogs (sheets), the Highlights app chooser
  (`NSOpenPanel`), the Capture and Refs panels, and every other window never see these
  keys.
- **Held keys repeat.** Holding Ctrl+D keeps paging until the bottom. Successive presses
  count from the outstanding target, not from a mid-animation offset, so no distance is
  lost.
- **Edges are quiet.** At the bottom, Ctrl+D is consumed and moves nothing; Ctrl+U does
  the same at the top. No beep, because a held key would beep repeatedly. Every press,
  edges included, calls `flashScrollers()`. The native overlay scroller appears briefly
  and shows where you are.

> [!decision] motion = glide Animate with `NSAnimationContext.runAnimationGroup`
> (duration 0.2, ease-out timing, implicit animation allowed) on
> `clipView.animator().setBoundsOrigin(_:)`, then `reflectScrolledClipView`. Keep one
> pending target per anchor so presses accumulate. Clear the pending target (a) in the
> group's completion handler, but only if no newer command has started (use a generation
> counter, because a superseded group's completion can fire late), and (b) on
> `NSScrollView.willStartLiveScrollNotification`, so a trackpad gesture takes over. When
> `NSWorkspace.shared.accessibilityDisplayShouldReduceMotion` is on, jump instantly
> instead (read it on every press, as `RefsPanelModel` does).

> [!decision] motion = jump No animation, pending target, generation counter, or
> live-scroll observer. Each press measures live geometry, jumps with `clip.scroll(to:)`
> and `reflectScrolledClipView` exactly like `RefsPanel.scrollInspector`, then flashes
> the scrollers. Drop the accumulation and takeover tests.

### The footer (decision `hint`)

> [!decision] hint = footer A slim footer, about 28 pt, pinned under the scroll view. It
> reads `⌃D Down  ⌃U Up` in `.caption` `.secondary`, single line, leading-aligned at the
> form's 20 pt inset. This is the Refs footer's text-hint format (`RefsFooter`: "⌃X
> Close Tab"). Attach it with macOS 26's `safeAreaBar(edge: .bottom)` so the system
> scroll-edge effect softens the rows as they pass under it. If `safeAreaBar` is
> unavailable on the CI SDK, use `.safeAreaInset(edge: .bottom)` over a `.bar` material
> background with a hairline top divider. The footer counts toward the window's height.
> It is always shown, because the form always overflows.

> [!decision] hint = none No footer and no `safeAreaBar`. The scroll view runs to the
> window's bottom edge. The Keyboard doc row is the only place the keys are taught.

## Implementation

1. **`Sources/BobMacCapture/SettingsScrollContainer.swift`** (new). Everything below is
   `@available(macOS 26.0, *)` where the surrounding code needs it.
   - `SettingsScrollKeys.direction(keyCode:modifiers:markedTextPresent:) -> RefsInspectorScrollDirection?`
     is the pure routing rule from "The keys".
   - `SettingsScrollContainer<Content: View>` holds the width and height rules, the
     `ScrollView(.vertical)`, and the footer per `hint`. It puts the anchor in the
     content's `.background(...)`, the same placement as `RefsInspectorView`'s
     `RefsInspectorScrollAnchor`, so `enclosingScrollView` resolves.
   - `SettingsScrollKeyAnchor` (an `NSViewRepresentable`) and its `NSView` subclass. In
     `viewDidMoveToWindow`, install a local `.keyDown` monitor while the view has a
     window and remove it when the window goes away and in `deinit`. Keep the gated
     entry `handleKeyDown(_:)` (Settings window, key, no sheet) separate from the
     ungated `processKeyEvent(_:)`, mirroring `RefsPanel.handleKeyDown` /
     `processKeyEvent`, so tests can drive the ungated path.
   - Measure and set the offset with `RefsInspectorScrollGeometry`
     (`offsetFromContentTop`, `clipOriginY(forContentTopOffset:…)`, `clamped`,
     inset-aware bounds). Accumulate with `RefsInspectorScrollPlanner.command(_:)`. Both
     are view-agnostic half-viewport math in the same target. Reuse them as they are;
     don't rename them or edit any Refs file.
   - Put the scroll performer behind an injectable closure, so tests can force instant
     motion and observe targets.
2. **`SettingsView.swift`**:
   - Wrap the existing `Form` in `SettingsScrollContainer`, pin `.formStyle(.columns)`,
     move the padding inside, and switch the width to 620 pt.
   - Apply `settingsCaption()` to every caption.
   - Keep the `.confirmationDialog`s, `.task`, and `.onReceive` on the outer view.
   - Change no section order, labels, bindings, or behavior.
3. **`BobSettingsScene`** (`AppDelegate.swift`): add `.windowResizability(.contentSize)`
   only if needed (see "The window").
4. Format only the lines you change (`.swift-format`: 4-space indentation, 100 columns).

## Tests (Tests/BobMacCaptureTests)

`NotificationService` needs a real `UNUserNotificationCenter`, so tests can't build the
full `SettingsView`. Test the container with synthetic content instead.

- **`SettingsScrollKeysTests`** (pure routing):
  - Ctrl+D routes down and Ctrl+U routes up, including with Caps Lock on.
  - Plain d/u, Ctrl+Shift, Ctrl+Option, and Command chords return nil.
  - Marked text returns nil.
  - Repeats still route.
- **`SettingsScrollContainerTests`**: host `SettingsScrollContainer` around a tall
  synthetic columns `Form` (about 60 rows) in an `NSWindow` with a fixed content height.
  Lay it out, find the anchor by walking subviews, and synthesize events with
  `NSEvent.keyEvent(…)` using instant motion. Assert:
  - Ctrl+D moves the clip offset by exactly half the viewport, and a second press by a
    full viewport.
  - Ctrl+U moves it back.
  - Repeated Ctrl+D clamps at the inset-aware maximum. At that maximum, Ctrl+D returns
    nil (consumed) and doesn't move. Ctrl+U at the top does the same.
  - Plain `d` returns the event unchanged.
  - `handleKeyDown` passes through events whose window is another window, and passes
    through while a sheet is attached.
- **`glide` only**: two presses before the animation settles target 2 × half a viewport,
  through the injected performer. Posting `willStartLiveScrollNotification` between
  presses makes the next press count from the measured offset.
- **`SettingsScrollDesignTests`**: follow `MeetingNotesDesignTests` (`NSHostingView`
  plus `cacheDisplay`, because `ImageRenderer` can't draw AppKit-backed controls). Write
  `settings-scroll-{top,middle,bottom}-{light,dark}.png` to
  `BOB_MAC_CAPTURE_RENDER_DIR`, rendering the container at 620 × 640 pt after 0, 2, and
  many Ctrl+D presses. Use a synthetic form that copies Settings' worst rows: the "In
  Highlights, show open tabs with" picker, the Pomodoro toggle, and the long Snoozed
  Tasks captions. Skip when the env var is unset, as the existing design tests do.

## Docs (bob-mac-capture `docs/`)

- **`settings.md`**: the intro sentence becomes "a 620 pt form that scrolls". Say that
  it opens no taller than the screen, resizes vertically, and that Ctrl-D / Ctrl-U page
  it by half its height. Mention the footer under `hint = footer`, and link to Keyboard.
- **`keyboard.md`**: add a `## Settings window` section with the two rows from "The
  keys" and short notes:
  - The keys work from any field.
  - IME composition passes through.
  - Edges consume the key silently.
  - Reduce Motion jumps instead of gliding (under `glide`).
  - The trackpad, wheel, and scroller also scroll.
- Run `just docs-check` with `BOB_CLI_DIR` pointing at the bob-cli checkout that opened
  this linked repo.

## Verification and landing

- Follow bob-mac-capture `AGENTS.md` exactly. Never land Swift you haven't compiled. Run
  `just portable-check` (on athena if `swift` is missing). Then compile a scratch copy
  on the mac: rsync without `.build`/`.git`, run `bundle.sh` and `just typecheck-tests`,
  then delete the copy. Never use Bryan's checkout or `just install`. If the mac is
  offline, say in the final report that the macOS targets are unverified.
- Gate with `Scripts/macos-ci-check` before landing. Run it through `/sase_monitor`,
  because it can take up to 90 minutes. Then download the run's `render-fixtures`
  artifact (`gh run download RUN_ID --name render-fixtures`) and look at every
  `settings-scroll-*` PNG. Check:
  - Nothing is clipped at either edge.
  - Captions wrap fully.
  - The footer reads cleanly in light and dark (under `footer`).
  - The bottom render ends exactly at the last row.

  Fix anything ugly before landing. After the host pushes, watch the exact-SHA macOS run
  to green with `/sase_monitor`.

- The final report must ask Bryan to check these on the installed build, because no
  agent can drive the real window:
  - The window opens fully on screen.
  - Ctrl+D/U page from the focused Executable override field.
  - The trackpad scrolls.
  - The labels are unclipped.

## Out of scope

- Esc, q, and Ctrl+[: the Settings window never closed on Esc, so
  decisions:q-and-ctrl-bracket-mirror-esc doesn't apply. Add no close keys.
- No other scroll keys: arrows move the caret and controls, and Ctrl+F/B/E move the
  caret in fields.
- No grouped-style redesign and no section reordering or relabeling.
- No bob-cli changes, versions, or changelogs.
