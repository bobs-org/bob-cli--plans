---
tier: tale
title: Close Highlights tabs through the window close button
goal:
  Bob Refs ⌃X and the row close button close any Highlights tab, background or current,
  in tabbed windows.
size: small
proposed_by: bbugyi200.apollo.88
create_time: 2026-10-16 16:28:44
status: wip
---

# Fix Bob Refs tab close: stop pressing Highlights' per-tab close button, which does nothing

Bob Refs' Tabs close (⌃X, the row's close button, and ⌘K Close Tab) fails with "Couldn't
close “…” in Highlights." for every tab in a Highlights window that shows a tab bar.
Only a window with a single tab, where the tab bar is hidden, closes. This plan fixes
the Accessibility close ladder in bob-mac-capture so that every close goes through the
window's close button, which works. Context: closed epic `bob-cli-8l` (Bob Refs: close a
Highlights tab from Tabs), whose ladder came from phase `bob-cli-8l.2`. Lander commit
`175153c` moved the key to ⌃X but left the ladder alone, so ⌃X fails the same way.

## Root cause (diagnosed live on the mac, 2026-10-16)

Bryan's mac: macOS 26.5 (25F71), Highlights 2026.1.2 (`net.highlightsapp.universal`).
`AppleWindowTabbingMode = always`.

1. **Unified log.** Both failed presses, at 16:10:42 and 16:10:51, emitted the signpost
   `refs-tabs-close-tab-button`, then `refs-tabs-close-unverified` about 2 s later. The
   one close that worked, at 16:08:40 (a one-tab window with no tab bar, after which
   Highlights quit as designed), emitted `refs-tabs-close-window-button`.
2. **Read-only Accessibility dump** of the Highlights window: `AXWindow` → `AXTabGroup`
   (title "tab bar") → one `AXRadioButton` with subrole `AXTabButton` per tab. Each tab
   has a child `AXButton` with title "Close tab", description "Close", identifier
   `_closeButton`, no subrole, and the action `AXPress`. `AXWindows` lists only the
   visible tab's window. That window exposes `kAXCloseButtonAttribute` (an `AXButton`
   with subrole `AXCloseButton`).
3. **Controlled experiment** using a scratch PDF opened as an extra tab while Highlights
   was in the background, then removed:
   - `AXPress` on the scratch tab's `_closeButton` while the tab was **not selected**
     returned success. The tab stayed open for the whole 2 s poll.
   - `AXPress` on the same `_closeButton` while the tab **was selected** also returned
     success. The tab also stayed open.
   - Selecting the tab (`AXPress` on its `AXRadioButton`) worked in the background.
     `AXPress` on the window's `kAXCloseButtonAttribute` button then closed exactly that
     tab, and the other tabs stayed.

So `closeLocatedTab` in `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift` uses
the tab's own close button as its first rung. `findCloseButton` returns any child
`AXButton` whose subrole is `nil`, so the rung matches `_closeButton` on every tab in a
tabbed window. Phase `bob-cli-8l.2` loosened the epic plan's `AXCloseButton`-subrole
match to `nil` after a probe that only read attributes and never pressed anything. The
code discards the press result. The ladder treats the accepted press as final, without
falling back. Verification then times out, so every tabbed close returns
`.failed("Highlights didn't close that tab")`.

The press reports success either way, so code cannot tell that it did nothing. Remove
the rung rather than keeping it as a first attempt with a fallback. A first attempt
would add a verification wait to every close and leave a late-arriving press in flight.

## Changes

All changes are in the linked repo `bob-mac-capture`. Open it with your `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<why>"`) and read its `AGENTS.md` before
editing. Stay within the epic's Accessibility limits:

- Only `AXPress` on tab buttons and window close buttons, plus `AXRaise` or activate for
  the save-sheet hand-off.
- No keystrokes, context menus, Apple Events, or private API.
- No new privileges.
- Signposts carry event names only, never tab names.

### 1. `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`

- Add a pure, testable `enum CloseLadder` next to `CloseVerification`:

  ```swift
  enum CloseLadder {
      enum Rung: Equatable {
          /// Press the window's close button; the target is its visible tab.
          case windowCloseButton
          /// Select the background target, then press its window's close button.
          case selectThenWindowClose
      }

      static func rung(hasTabButton: Bool, isSelected: Bool) -> Rung

      /// Restore the previously visible tab only after a verified close that
      /// left Highlights running with documents open.
      static func shouldRestore(after result: HighlightsTabCloseResult?) -> Bool
  }
  ```

  - `rung` returns `.windowCloseButton` when there is no tab button (a window with no
    tab bar) or the target is selected. Otherwise it returns `.selectThenWindowClose`.
  - `shouldRestore` returns `true` only for `.closed(snapshot)` with
    `snapshot.documentCount > 0`. It returns `false` for `nil`, `.needsAttention`,
    `.failed`, `.notFound`, `.notRunning`, `.notTrusted`, and an empty `.closed`
    snapshot.
  - Give both a short doc comment saying why: Highlights' per-tab `_closeButton` accepts
    `AXPress` but does not close the tab.

- In `closeLocatedTab`, switch on `CloseLadder.rung(...)`:
  - Delete the per-tab close-button rung.
  - Delete `tabCloseButton(of:deadline:)`, `findCloseButton(in:depth:deadline:)`, and
    the `refs-tabs-close-tab-button` signpost.
  - Keep the title re-check (`CloseNotFound`) whenever a tab button exists.
  - Keep the signposts `refs-tabs-close-window-button` and
    `refs-tabs-close-select-fallback`.
  - Have `closeLocatedTab` return the title to restore. That is the window's previously
    selected tab title, `window.value.tabs.first(where: \.isSelected)?.fileName`, for
    `.selectThenWindowClose`, and `nil` otherwise.
- In `closeBySelecting`:
  - Keep these steps: select the target, poll within the deadline until a fresh
    `AXWindows` read shows it selected, then press that window's close button.
  - Remove the `restoreTab` call from this function.
  - Restore must wait for verification. Otherwise, a save sheet on the target's window
    would be hidden behind the restored tab, and sheet detection only sees visible
    windows.
- In `closeOnQueue`, after `verifyClosed` returns:
  - If `CloseLadder.shouldRestore(after: verified)` is true and a restore title exists:
    - Call `restoreTab` with its own fresh deadline
      (`DispatchTime.now().uptimeNanoseconds + timeBudgetNanoseconds`), so a close that
      used most of the 2 s budget still switches back.
    - Then make a best-effort `readSnapshot` with that same restore deadline.
    - If the read succeeds and `titleCount(target.fileName, …)` is still below
      `beforeCount`, return `.closed(freshSnapshot)`. The fresh snapshot carries the
      right selection, so the panel's Current chip and the session's recency stay
      correct.
    - Otherwise, return the verified result unchanged.
  - Never restore after `.needsAttention`. The sheet stays visible on the target tab.
  - Never restore on the `nil` / `needsAttentionOrFailed` path.
  - `restoreTab` keeps its existing guards (Highlights exited, remembered tab gone).
- Keep the doc comments accurate. For example, the `close(_:)` comment should describe
  the window-close ladder, not a tab close button.

### 2. `Tests/BobMacCaptureTests/AXHighlightsTabsReaderTests.swift`

Add pure tests next to the `CloseVerification` tests:

- `rung(hasTabButton: true, isSelected: true)` returns `.windowCloseButton`. This is the
  reported case: the Current tab in a tabbed window.
- `rung(hasTabButton: false, isSelected: true)` and
  `rung(hasTabButton: false, isSelected: false)` return `.windowCloseButton`.
- `rung(hasTabButton: true, isSelected: false)` returns `.selectThenWindowClose`.
- `shouldRestore` is `true` for `.closed` with at least one remaining tab. It is `false`
  for:
  - an empty `.closed` snapshot;
  - `.needsAttention`, `.failed("x")`, `.notFound`, `.notRunning`, and `.notTrusted`;
  - `nil`.

### 3. `README.md`

- Two privacy passages say Accessibility "presses a tab's or window's close button": the
  Tabs section near "Accessibility reads Highlights' window and tab attributes" and the
  data/privacy section near "it reads window and tab attributes, selects a tab button".
  Change both to say it presses a window's close button.
- In the Tabs section's close paragraph, add one sentence: a background tab closes by
  briefly switching Highlights to it, closing it with the window's close button, and
  switching back.

No SASE memory note or decision record describes this ladder, so no memory change is
needed.

## Verification

1. **Stray references.**
   `grep -rn "_closeButton\|findCloseButton\|tabCloseButton\|close-tab-button" Sources Tests README.md`
   in the bob-mac-capture checkout prints nothing.
2. **Portable checks.** Swift is not installed on apollo. Rsync a scratch copy (exclude
   `.build` and `.git`) to `athena`. Run `just portable-check`,
   `swift build --build-tests`, and `swift test`, then delete the copy. Report the exit
   codes and test counts.
3. **macOS targets.** `portable-check` cannot type-check `Sources/BobMacCapture*`.
   - The mac (`ssh mac`, user `bbugyi`) was online during this diagnosis. Follow
     `AGENTS.md`:
     1. Rsync a scratch copy without `.build`/`.git` to a temp dir on the mac.
     2. Run `./Scripts/bundle.sh` (or `just bundle`) and `just typecheck-tests`.
     3. Delete the copy.
   - Never use Bryan's checkout or `just install` there.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
4. **CI.** After pushing, check that the macOS CI run is green, including Lint Swift
   formatting and the later Build/Test/Bundle steps.
5. **Manual acceptance (Bryan, after installing the build).** Open two or more PDFs as
   tabs in one Highlights window, open Bob Refs Tabs (⇧⌘O or ⌘T):
   - ⌃X on the **Current** row closes that tab, shows the "Closed tab" toast, and the
     panel stays up.
   - ⌃X on a **background** row closes it. Highlights still shows the same tab it showed
     before.
   - ⌃X ⌃X in quick succession closes two tabs.
   - Clicking a row's close button behaves like ⌃X.
   - Closing the last tab quits Highlights and hides the panel.

Commit message subject:
`fix(refs): close Highlights tabs through the window close button`.
