---
tier: tale
title: Open Bob Refs link-picker links with Shift
goal:
  In the Bob Refs Copy card's link picker, Shift-1–9, Shift-Return, and Shift-click open
  the link in the default browser; plain keys still copy.
size: medium
decisions:
  link_schemes:
    ask:
      "Should Shift-open also open mailto: and obsidian: links, not just http and https?"
    choices:
      web: Open only http and https; other schemes show a red reason and still copy
      web_mail_obsidian:
        Also open mailto and obsidian links; still refuse file and every other scheme
    default: web
    why:
      Saved pages can carry app-action URIs (Advanced URI, Shortcuts); the catalog is
      mostly https
    answer: web
proposed_by: bbugyi200.apollo.am
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 10:46:28
status: wip
---

# Open links from the Bob Refs link picker with Shift

## Goal

In bob-mac-capture's Bob Refs Copy card, the link picker (opened with `L`, or with
Command-L from the panel or from Highlights) can only copy links today. Add **open in
the default browser**, using Shift as the user suggested:

- **Shift-1 … Shift-9** opens that visible link (the `!`…`(` keys on a US keyboard).
- **Shift-Return** (and Shift-keypad-Enter) opens the selected link. So **Command-L
  Shift-Return** opens the current PDF page's link in one chord, next to the existing
  Command-L Return, which copies it.
- **Shift-click** on a picker row opens that row. VoiceOver gets an "Open link" action
  on each row.

Opening a link hides Bob Refs (the browser comes forward) and does not touch the
pasteboard. Everything else stays the same: plain digits and Return still copy, Option
still selects the Markdown form, `A` still copies all, and Escape still returns to the
card.

### Why Shift (design rationale, already decided)

- Option already means "the alternate payload" (Markdown form), so opening needs a
  different modifier.
- Command doesn't fit. The card's Command chords are entry chords (⌘Y, ⌘B, ⌘L), and
  Command-1 … Command-5 switch the panel scope. Using Command-digit here would collide
  with that muscle memory.
- Shift-digit is free today. It produces `!`, `@`, … and falls through to `.unknown`
  ("Nothing on ‘!’"). Shift-Return is free too: `RefsCopyKeyMap` consumes it. So nothing
  that works now changes meaning.
- Shift-held previews work the same way Option-held previews already do.

### Layout-robust detection (important)

`RefsPanel.processKeyEvent` passes `event.charactersIgnoringModifiers`, which **keeps
Shift**. So Shift-1 arrives as `"!"` on a US layout, as `"!"` on German too, and Shift-2
arrives as `"\""` on German. Don't map shifted symbols. Map the **physical top-row digit
key codes**, which are positional on ANSI, ISO, and JIS boards (kVK_ANSI_1…9):

| keyCode | 18  | 19  | 20  | 21  | 23  | 22  | 26  | 28  | 25  |
| ------- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| digit   | 1   | 2   | 3   | 4   | 5   | 6   | 7   | 8   | 9   |

Rule: Shift is held, the key code is in this table, and the produced character is
**not** an ASCII digit → `.openWindowRow(n)`. The "not a digit" clause keeps today's
contract wherever Shift already yields a digit:

- On AZERTY, you type digits with Shift, so they must keep copying.
- The existing test
  `RefsCopyAlternatesTests.testOptionAndShiftKeyMappingKeepsOtherModifiersConsumed`
  asserts that `("1", keyCode 0, [.option, .shift, .capsLock])` stays
  `.key(.note(1), alternate: true)`. It must keep passing unchanged.

Keypad digits and `0` are unaffected. RefsCore must stay Linux-portable, so use literal
key codes with comments (the way the file already does `case 45, 38: // N, J`), not
Carbon constants.

### What opens (safety)

The URL comes verbatim from bob's `ref links` catalog: `RefsLink.url`, the same bytes
that Return copies. That keeps the `decisions:code-copy-reads-render-time-catalog` and
`decisions:mac-capture-is-a-thin-client` contracts: the app presents bob's data and
never re-parses the source.

Before calling `NSWorkspace`, RefsCore gates the scheme:

> [!decision] link_schemes = web Only `http`/`https` URLs with a non-empty host open.
> Any other scheme (`mailto:`, `obsidian:`, `file:`, `ftp:`, custom app schemes,
> relative links) leaves the card open, selects that row, and shows the red message
> `Only web links open — ↵ copies it`.

> [!decision] link_schemes = web_mail_obsidian The allowlist is `http`, `https`
> (non-empty host required), `mailto`, and `obsidian`. Everything else, including
> `file:`, gets the red message `Can't open this kind of link — ↵ copies it`. Docs name
> the four schemes.

Keep the allowlist in one constant so it is easy to extend later.

URL construction must be deterministic on macOS and on Linux, where the RefsCore tests
run. Try `URL(string: link.url)` first. If that returns nil (for example
`https://example.com/a b` in the fixture), retry after percent-encoding only the
characters outside the RFC 3986 legal set, which is unreserved + reserved + `%`. Reject
the URL if the scheme (lowercased) is not allowed or, for http/https, if the host is
empty.

### After opening

The model closes the card (`closeCopyCard()`), hides Refs through `hidePanel()`, and
calls `opener.openWithDefaultApp(url)`. That is the existing `RefsOpening` method, which
runs `NSWorkspace.open` with `activates = true`. This is the same hide-then-open order
as the reference opens (`openWithDefaultApp(_ item:)`).

- Use the plain `hidePanel()`, **not** the copy dismisser. A retained Capture draft must
  not pop back over the browser; it stays retained for the next Capture show, as with
  the other Refs opens.
- Show no glass HUD, because the browser coming forward is the confirmation. Don't call
  `copyCompletion` either.
- On an open error, set `lastDiagnostic = RefsLibrary.boundedMessage(for:)`, call
  `panelRepresenter()`, and show an error banner
  `Couldn’t open the <site> link.\n<diagnostic>` with actions `[.copyDiagnostic]`. Don't
  add `.tryAgain`: it is wired to reference-scoped `pendingOpen`. On success, leave
  `banner` untouched.

## Implementation (bob-mac-capture)

Open the linked repo with your `/sase_repo` skill
(`sase repo open bob-mac-capture -r "..."`) and read its `AGENTS.md` first. All paths
below are relative to that checkout.

### 1. `Sources/RefsCore/RefsCopy.swift`: commands, key map, card

- `RefsCopyCommand`: add `case openWindowRow(Int)` and `case openSelected`.
- `RefsCopyEffect`: add `case openLink(RefsLinkOpenRequest)`. `RefsLinkOpenRequest` can
  live in `RefsLinks.swift`. It is an `Equatable, Sendable` struct with `link: RefsLink`
  and `url: URL`, plus a computed `site` for the failure banner.
- `RefsCopyKeyMap.command`:
  - In the Return / keypad-Enter branch (`keyCode == 36 || keyCode == 76`),
    `flags == .shift || flags == [.shift, .option]` → `.openSelected`. Empty flags and
    `.option` still give `.copySelected`, and any other combination is still `.consume`.
  - After the existing `guard flags.subtracting([.shift, .option]).isEmpty …`, add the
    top-row Shift-digit rule from "Layout-robust detection". Put it **before**
    `RefsCopyKey.parse`. Option held alongside Shift doesn't change the result: the URL
    opens, not the Markdown form.
- `RefsCopyCard`:
  - `handleLinkPicker`: `.openWindowRow(n)` calls `picker.selectWindowPosition(n)`. If
    that returns nil, show `No link n` (same as copy) and return `.none`. Otherwise
    return the open effect for that entry. `.openSelected` returns the open effect for
    `selectedEntry`. Write one private helper that returns `.openLink(...)`, or sets the
    red refusal message and returns `.none` when `link.webURL` (or the chosen allowlist
    accessor) is nil.
  - `perform` (main card), `handleCodePicker`, and `handlePicker` (highlights):
    `.openWindowRow(n)` clears `pending` and shows the transient `Nothing on ‘⇧n’`.
    `.openSelected` is a no-op returning `.none`, which keeps today's consumed
    Shift-Return.
  - `isTerminal`: `.openLink` is terminal. Queued `L` then `⇧↵` while links load must
    replay into a single open and clear the queue, just as queued `L` then `↵` copies.
  - Every `switch` over these enums must stay exhaustive. Update any other exhaustive
    switch the compiler finds.
- `Sources/RefsCore/RefsLinks.swift`:
  - Add `RefsLink.webURL: URL?`. Name it for what it checks; if you pick
    `web_mail_obsidian`, `openableURL` is the better name. It implements the scheme gate
    and the deterministic URL construction above.
  - Update `RefsLinkPicker.openingAnnouncement` to
    `Links. Press a number or Return to copy. Hold Shift to open.`

### 2. `Sources/BobMacCapture/Refs/RefsPanelModel.swift`: effects, Shift state, taps

- Add `@Published public private(set) var copyLinkOpenActive = false`.
  - Set it in `updateCopyModifiers` as
    `(copyCard != nil || isPendingHighlightsCopyArmed) && modifiers.contains(.shift)`,
    next to `copyAlternatesActive`.
  - Reset it in `closeCopyCard()`.
- `applyCopyCardEffect`: handle `.openLink(request)` as described in "After opening".
  Add a private `finishLinkOpen(_:error:)` modeled on `finishClosedTabDefaultAppOpen`,
  using the same `Task { await self?.… }` hop.
- `copyCardPickerRowTapped(_:)`: when a link picker is open and `copyLinkOpenActive` is
  true, call `handleCopyCardCommand(.openWindowRow(position))`. Otherwise keep the
  current copy path. Shift-click then reuses the tracked Shift state, just as
  Option-click reuses `copyAlternatesActive`.
- Add `public func copyCardPickerRowOpenRequested(_ position: Int)`. It is guarded on
  `copyCard?.linkPicker != nil` and dispatches `.openWindowRow`; the VoiceOver action
  uses it.
- No change is needed in the Highlights pending buffer. It already stores
  `characters`/`keyCode`/`modifiers` and replays them through `handleCopyCardKey`. Cover
  it with a test anyway (see Tests).

### 3. Views

- `Sources/BobMacCapture/Refs/RefsCopyCardView.swift`:
  - Add `var linkOpenActive = false` and
    `var onPickerRowOpen: (Int) -> Void = { _ in }`.
  - In `linkPickerBlock`, build the well's `RefsLinkPreview` with
    `markdown: alternatesActive && !linkOpenActive`, so the well shows the URL that will
    open, and pass the open flag to the well.
  - In `linkPickerRow`, add
    `.accessibilityAction(named: "Open link") { onPickerRowOpen(position) }`.
  - Hint line (`hintText`, link picker branch):
    - Default: `↵ copies · ⌥↵ Markdown · ⇧↵ opens · A all · 1–9 pick · esc back`.
    - While `linkOpenActive`: `⇧↵ opens in browser · ⇧1–9 open that link · esc back`.
- `Sources/BobMacCapture/Refs/RefsLinkPreviewWell.swift`: add
  `var opensInBrowser = false`. When it is true, `headerTrailing` appends `Open` in
  place of `Markdown`, for example `p. 7 · Open`.
- `Sources/BobMacCapture/Refs/RefsPanelView.swift`: pass
  `linkOpenActive: model.copyLinkOpenActive` and
  `onPickerRowOpen: { model.copyCardPickerRowOpenRequested($0) }`.

## Tests

### RefsCore

These run on Linux via `just portable-check`.

**`Tests/RefsCoreTests/RefsCopyTests.swift`**, or a new `RefsLinkOpenTests.swift`, for
the key map:

- `("!", 18, [.shift])` → `.openWindowRow(1)`.
- `("(", 25, [.shift])` → `.openWindowRow(9)`.
- German Shift-2 `("\"", 19, [.shift])` → `.openWindowRow(2)`.
- `("!", 18, [.shift, .option])` → `.openWindowRow(1)`.
- AZERTY-style `("1", 18, [.shift])` → `.key(.note(1))`.
- `("!", 18, [.shift, .command])` → `.consume`.
- `(")", 29, [.shift])` → `.unknown(")")`.
- Return and keypad Enter with `[.shift]` or `[.shift, .option]` → `.openSelected`.
- Return with `[.shift, .control]` → `.consume`.
- The existing Option/Shift test passes unmodified.

**`Tests/RefsCoreTests/RefsLinksTests.swift`**, using the `refs-links.json` golden. The
fixture already contains https, `mailto:`, `obsidian://`, a URL with a space, and one
with a paren.

- In the picker with `highlightedPage: 7`, `.openWindowRow(2)` → `.openLink` with
  `url.absoluteString == "https://ai.google.dev/gemini-api/docs/imagen"`. Use the same
  position→link mapping as `testLinkPickerKeysCopyDigitsReturnAllAndEsc`. The selection
  moves to that entry.
- `.openSelected` opens the page-preselected link.
- `.openWindowRow` past the window → `No link n` and `.none`.
- `webURL` table:
  - `https`, upper-case `HTTPS://Example.COM/x`, the paren URL, and
    `https://example.com/a b` (absoluteString `https://example.com/a%20b`) → non-nil.
  - `mailto:`, `obsidian://open?vault=bob`, `file:///etc/hosts`, `javascript:alert(1)`,
    `ftp://…`, relative `docs/x.md`, and `https:///nohost` → nil.
  - Under `web_mail_obsidian`, adjust `mailto:` and `obsidian:` to non-nil.
- Opening a refused link (fixture n 5 or 6) keeps the card open, selects that row, shows
  the red refusal, and returns `.none`. A following `.copySelected` copies it.
- Queued: card with `links: .loading`, then `.key(.markdownLink)`, then `.openSelected`.
  `linksLoaded` returns a single `.openLink` for the preselected link, and nothing
  remains queued.
- Outside the link picker:
  - On the main card, `.openWindowRow(1)` → transient `Nothing on ‘⇧1’`.
  - `.openSelected` → `.none` and no message.
  - Same for the code and highlight pickers.
- Update the `openingVoiceOverLabel` assertion to the new announcement.

### App

Add these to `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, reusing
`FakeOpener.defaultAppOpens`/`defaultAppError`, `harness.dismissed`,
`harness.represented`, and `Self.linksResponse`:

- Shift-digit open: open the Copy card on a saved-Markdown reference with links loaded,
  press `L`, then
  `handleCopyCardKey(characters: "@", keyCode: 19, modifiers: [.shift])`.
  - `defaultAppOpens` contains link 2's URL.
  - The pasteboard is empty, `copyCompletion` was never called, `copyCard` is nil, and
    `dismissed == 1`.
- Buffered Highlights chord: mirror
  `testHighlightsLinkEntryAndBufferedReturnCopiesPageLink`, but buffer `"\r"`, keyCode
  36, `[.shift]`. It opens the page link and copies nothing.
- Failure: with `defaultAppError` set, the open represents the panel
  (`represented == 1`), and the banner message contains the diagnostic with actions
  `[.copyDiagnostic]`.
- Shift-click: after `updateCopyModifiers([.shift])`, `copyCardPickerRowTapped(2)`
  opens. After `updateCopyModifiers([])`, it copies as before.
- `copyLinkOpenActive` follows `updateCopyModifiers` and resets when the card closes.
- `copyCardPickerRowOpenRequested` opens, and is a no-op without a link picker.

### Design fixture

In `Tests/BobMacCaptureTests/RefsCopyCardDesignTests.swift`, add
`refs-copy-link-picker-open`: the link picker rendered with `linkOpenActive: true`.
Extend the private `write(card:name:alternates:)` helper with a defaulted parameter.

## Docs (bob-mac-capture `docs/`, mounted into Bob's site)

- **`docs/refs.md` › Copy card**, link-picker paragraph:
  - Shift-1–9 opens that visible link and Shift-Return opens the selected one in the
    default browser.
  - Shift-click does the same.
  - Holding Shift adds `· Open` to the well header and swaps the hint.
  - Only the decided schemes open; others explain why and still copy.
  - Opening hides Refs, shows no HUD, and leaves the pasteboard alone. An open failure
    brings Refs back with an error banner.
  - Next to "Command-B Return and Command-L Return copy…", add that Command-L
    Shift-Return opens the current page's link.
- **`docs/keyboard.md` › Copy card keys**:
  - Add a table row: `Shift-1–9 / Shift-Return` — "In a link picker, open that visible /
    the selected link in the default browser (web links only)".
  - Extend the Command-L paragraph with the Shift-Return chord.
- Run `just docs-check`, as `AGENTS.md` requires.

## Verification

`AGENTS.md` forbids landing uncompiled Swift.

1. Run `just portable-check` on a host with Swift (athena if the current host has none).
   It covers swift-format lint plus RefsCore build and tests.
2. Run the macOS gate: `Scripts/macos-ci-check` through your `/sase_monitor` skill, with
   profile `verify` and `--timeout 95m` (see `docs/develop/ci.md`). It compiles and
   tests `BobMacCapture*` on `macos-26`. Optionally, first do the best-effort Mac
   scratch `bundle.sh` + `just typecheck-tests` that `AGENTS.md` describes. If the Mac
   or CI can't be reached, say in the final report that the macOS targets are
   unverified.
3. Run `just docs-check`.
4. Commit in bob-mac-capture with a `feat(refs): …` message. bob-cli itself needs no
   change: `bob ref links` already supplies `url`.

## Out of scope

- Keeping Refs open to open several links in a row. The panel hides on resign-key, and
  background opens (`activates = false`) depend on the browser, so CI can't verify them.
- Opening from the main Copy card's `U` source-URL row, or from the code and highlight
  pickers.
- Opening every link with Shift-A.
- Any bob-cli, JSON contract, or memory change.
