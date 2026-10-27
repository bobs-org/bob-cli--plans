---
tier: tale
title: Highlights code and link copy keys for Bob Refs
goal:
  While Highlights is frontmost, one chord opens the Bob Refs Copy card with the code
  picker already chosen and another with the link picker already chosen, and every
  failure ends in a clear, specific message.
size: medium
decisions:
  code_key:
    ask: Which chord should open the Copy card's code picker from Highlights?
    choices:
      cmd_b:
        Command-B ("block"); one modifier like ⌘Y/⌘L; unbound in Highlights and the Refs
        panel
      ctrl_cmd_c:
        Control-Command-C keeps the C mnemonic; two modifiers, among Highlights' ⌃⌘
        tools
    default: cmd_b
    why:
      Single modifier like ⌘Y and ⌘L; collides with nothing in Highlights 2026.1.2 or
      the panel
    answer: cmd_b
  link_without_markdown:
    ask:
      On references without saved Markdown, should ⌘L copy their Markdown link at once,
      exactly like ⌘Y L?
    choices:
      same_as_l: ⌘L always equals ⌘Y L; such references copy [Title](source URL) at once
      picker_only:
        ⌘L always means the link picker; such references show the red no-saved-Markdown
        line
    default: same_as_l
    why: The chord then saves exactly one keypress and never has a second meaning
    answer: same_as_l
  offer_cmd_c:
    ask:
      Also offer Command-C as an opt-in code key in Settings, though it replaces
      Highlights' Edit › Copy?
    default: false
    why:
      Highlights' ⌘C copies selected PDF text, and a Carbon hotkey cannot pass the key
      through
    answer: false
proposed_by: bbugyi200.athena.14k.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 22:09:17
status: wip
---

# Highlights code and link copy keys for Bob Refs

## Goal

Add two Highlights-frontmost chords to Bob Mac Capture. They work alongside the existing
Command-Y, which opens the Bob Refs Copy card on the focused Highlights PDF.

- **Code chord**: opens the same Copy card with the `C` (code picker) option already
  pressed. Command-B by default; see DECISION `code_key`.
- **Link chord (Command-L)**: opens the same Copy card with the `L` (link picker /
  Markdown link) option already pressed.

Each chord saves exactly one keypress: chord + Return equals Command-Y C Return or
Command-Y L Return, so it copies the code block or link on the current PDF page.

Every way these chords can fail must end in a clear, specific, user-visible message.
Silently doing nothing is never acceptable.

## Why not Command-C

The user asked for Command-C and Command-L, and allowed better keys if those could not
or should not be used. Highlights 2026.1.2 is installed on the Mac
(`net.highlightsapp.universal`, no `NSUserKeyEquivalents` overrides). Its `MainMenu.nib`
binds:

- Command-C = **Edit › Copy**
- Option-Command-C = **Smart Copy**
- Shift-Command-C = **Custom Copy**

A Carbon hotkey consumes the chord outright and cannot pass it through. Taking over
Command-C would therefore break copying selected PDF text, the most common thing anyone
does in Highlights. A context-sensitive takeover (inspect the selection over AX, or use
a CGEventTap) is fragile and needs more permissions; it is rejected.

Command-L, Command-B, and Control-Command-C are unbound in Highlights' menus. Command-B
and Command-L are also unbound in the Bob Refs panel router (`RefsKeyRouter`), so the
same chords can work inside the panel. Command-K is not an option: it is the panel's
actions menu, and the Copy card maps it to `.showActions`.

The rest of this plan writes the code chord as **Command-B** (`⌘B`, enum case `.cmdB`).

> [!decision] code_key = ctrl_cmd_c Substitute Control-Command-C everywhere: `⌃⌘C`,
> keyCode 8 with `[.control, .command]`, enum case `.ctrlCmdC`, configuration
> `highlightsControlCommandC`. That covers constants, router, messages, tests, and docs.
> Plain Control-C must keep resetting the view.

## Where the work happens

All code and docs changes are in the linked **bob-mac-capture** repo. Open it first with
your `/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), and use only the
path it prints. Read that repo's `AGENTS.md` before editing. bob-cli itself needs no
changes:

- bob already serves `bob ref code` and `bob ref links`.
- bob-cli's `docs/guides/resolve-mac-capture-hotkey-conflict.md` quotes
  `Refs hotkey conflict: …`. That prefix is kept, so the guide stays correct.

Governing records (read before changing behavior they cover):

- `decisions:mac-capture-is-a-thin-client`
- `decisions:code-copy-reads-render-time-catalog`

Nothing here parses Markdown or the PDF, and nothing writes the vault. The chords only
pre-press a key on the existing Copy card, which reads bob's catalogs.

## Current shape (what to build on)

- `Sources/BobMacCapture/HotKeyRegistry.swift`:
  - `HotKeyAction` currently has ids 1–5; `.refsHighlightsCopy = 5`.
  - `HotKeyConfiguration.highlightsCommandY` is defined here.
- `Sources/BobMacCapture/RefsHighlightsCopyKey.swift`: enum `.cmdY` / `.off` with
  `displayName` and `configuration`.
- `Sources/BobMacCapture/AppSettings.swift`:
  - `@Published refsHighlightsCopyKey`, persisted under `refsHighlightsCopyKey`.
  - A bogus stored value falls back to the default.
- `Sources/BobMacCapture/SettingsView.swift`: the "Copy key in Highlights" picker.
- `Sources/BobMacCapture/HighlightsTakeover.swift`:
  - Registers the takeover keys only while Highlights (or the override app) is
    frontmost.
  - Command-Y always yields while Refs is visible, so the panel's own ⌘Y handles it.
  - A registration failure calls `onError("Refs hotkey conflict: \(error)")`.
- `Sources/BobMacCapture/AppDelegate.swift`:
  - The hotkey `switch` routes `.refsHighlightsCopy` to
    `panelCoordinator?.openRefsCopyFromHighlights()`.
  - The `HighlightsTakeover(...)` construction sets `onError` →
    `settings.diagnosticStatus`, which only shows in Settings › Diagnostics.
  - `observeRefsSettings()` combines `$refsHighlightsOpenKey`, `$refsHighlightsTabsKey`,
    `$refsHighlightsCopyKey`, and `$refsHighlightsAppPath`; Combine's `combineLatest`
    caps out at four publishers. Sinks must use the values they receive, because
    `@Published` emits before the property changes.
- `Sources/BobMacCapture/BobPanelCoordinator.swift`: `openRefsCopyFromHighlights()`
  calls `showRefs(source: .highlightsCopy)`.
- `Sources/BobMacCapture/Refs/RefsPanelModel.swift`:
  - `RefsPresentationSource.highlightsCopy`.
  - `RefsHighlightsCopyPending` (`.finding(token:)` / `.failed(message:)`).
  - `armPendingHighlightsCopy()`, `bufferPendingHighlightsCopyKey`, the 2 s deadline,
    and `pendingHighlightsCopyTimedOut()` ("Highlights didn't answer — try ⌘Y again").
  - `resolvePendingHighlightsCopy()`: maps Tabs state and match to failure messages,
    including "Two references match “…” — pick one, then ⌘Y". It then calls
    `openCopyCard(optionHeld:)` and replays the buffered keys through
    `handleCopyCardKey`.
  - `perform(.showCopyCard)` guards linking-today and synthetic tabs.
- `Sources/BobMacCapture/Refs/RefsPanel.swift`:
  - `show(source:)` and `processKeyEvent` compare `source == .highlightsCopy`.
  - Keys go to the pending buffer while it is armed, otherwise to the open card.
- `Sources/BobMacCapture/Refs/RefsKeyRouter.swift`: ⌘Y → `.showCopyCard`, passing
  through while IME text is marked.
- `Sources/RefsCore/RefsCopy.swift`:
  - `RefsCopyKey.parse("c")` = `.codePicker` and `parse("l")` = `.markdownLink`.
  - `RefsCopyCard.handle` remaps `.markdownLink` → `.linkPicker` when
    `item.sourceMarkdown != nil`.
  - While a catalog is `.loading`, the card sets `pending` ("Loading code… esc cancels"
    / "Loading links… esc cancels") and queues later commands. A 5 s timeout ends in
    "Code unavailable — nothing copied" / "Links unavailable — nothing copied".
  - `openCodePicker()` / `openLinkPicker()` show `hiddenReason` in red (`isError: true`)
    when nothing can be copied, for example "No code blocks in this reference — M copies
    its Markdown", "No links in this reference", "Code copy needs a newer bob", or
    "Saved Markdown isn't on this Mac yet — sync the vault".
- `Sources/BobMacCapture/Refs/RefsCopyCardView.swift`: `RefsCopySkeletonView` (header
  "COPY", "Finding the open PDF in Highlights…", red failure line).
- `Sources/BobMacCapture/Refs/RefsCopyHUD.swift`: `RefsGlassToastStyle` and
  `RefsGlassToastController`. `RefsPanelController` already shows toasts while the panel
  is hidden (`copySucceeded` uses `centeredOn:`).

## Design

### 1. One entry concept, shared by the hotkey and the panel

Add a small pure enum in **RefsCore**, so `swift test` on Linux covers it, for example
`RefsCopyEntry` in `Sources/RefsCore/RefsCopy.swift` or a sibling file:

- `.card`: the Command-Y behavior, no seed.
- `.code`: seed command `.key(.codePicker)`; retry glyph `⌘B`.
- `.links`: retry glyph `⌘L`; the seed command depends on the decision below.

> [!decision] link_without_markdown = same_as_l Seed `.key(.markdownLink)`, exactly what
> typing `L` produces. The card's existing remap turns it into the link picker when
> saved Markdown exists. Without saved Markdown, it copies the reference's Markdown link
> at once, as ⌘Y L does today.

> [!decision] link_without_markdown = picker_only Seed `.key(.linkPicker)`. Without
> saved Markdown, `openLinkPicker()` shows the red "No saved Markdown — link copy needs
> a reference bob rendered from Markdown" line, and nothing is copied.

Expose `seedCommand: RefsCopyCommand?` and `retryGlyph: String` (`⌘Y` for `.card`). The
glyph names the chord the user just pressed, and the panel router binds the same fixed
chords.

> [!decision] offer_cmd_c The entry still names the default chord. Command-C is only a
> Highlights-takeover alias and is never routed inside the Refs panel, where Command-C
> stays text copy in the search field.

### 2. Settings and hotkey registration

- `HotKeyAction`: add `.refsHighlightsCode = 6` and `.refsHighlightsLinks = 7`.
- `HotKeyConfiguration`:
  - `highlightsCommandB` (`kVK_ANSI_B`, `cmdKey`, "Command-B").
  - `highlightsCommandL` (`kVK_ANSI_L`, `cmdKey`, "Command-L").
  - Plus `highlightsControlCommandC` and/or `highlightsCommandC` only if the decisions
    call for them.
- New enums beside `RefsHighlightsCopyKey`, in the same style (`displayName`,
  `configuration`, raw values persisted as strings):
  - `RefsHighlightsCodeKey`: `.cmdB` (default) / `.off`, plus `.cmdC` only when DECISION
    `offer_cmd_c` is on.
  - `RefsHighlightsLinkKey`: `.cmdL` (default) / `.off`.
- `AppSettings`:
  - `refsHighlightsCodeKey` and `refsHighlightsLinkKey`, persisted under keys
    `refsHighlightsCodeKey` / `refsHighlightsLinkKey`.
  - Missing or bogus values load the default.
- `SettingsView`, in the References section after "Copy key in Highlights":
  - "Copy code key in Highlights" picker.
  - "Copy link key in Highlights" picker.
  - When DECISION `offer_cmd_c` is on, the extra option reads "Command-C (replaces
    Highlights' Copy)". Settings, keyboard, and settings docs then all say Highlights
    can no longer copy text with Command-C while that option is chosen, Highlights is
    frontmost, and Bob Refs is hidden. Add a persistence test for `"cmdC"`.
- `HighlightsTakeover`:
  - Takes `codeKey` and `linkKey` providers with `.off` defaults, so existing call sites
    and tests keep compiling.
  - Registers `.refsHighlightsCode` / `.refsHighlightsLinks` exactly like Command-Y:
    - only while the expected bundle is frontmost;
    - unregistered while Refs is visible, so the panel's own chords act on the
      selection;
    - unregistered in `stop()`.
  - The explicit-values `sync(...)` overload grows to carry the new keys. Bundling the
    takeover keys into one small `Equatable` struct is fine.
- `AppDelegate`:
  - Wire the providers.
  - Extend `observeRefsSettings()` so a change to either new setting re-syncs live,
    passing received values. Use a nested `combineLatest`, or a second sink that feeds
    the same struct; Combine has no 6-way `combineLatest`.
  - Add the two `HotKeyAction` cases to the hotkey `switch`, each with its own
    `CaptureSignpost.event` (`refs-code-hotkey-received` /
    `refs-links-hotkey-received`). They call
    `panelCoordinator?.openRefsCopyFromHighlights(entry:)`.
- `BobPanelCoordinator.openRefsCopyFromHighlights(entry: RefsCopyEntry = .card)` →
  `showRefs(source: .highlightsCopy(entry))`.

### 3. Pending session with a pre-pressed key

- Change `RefsPresentationSource.highlightsCopy` to carry the entry:
  `.highlightsCopy(RefsCopyEntry)`.
  - Replace every `== .highlightsCopy` comparison (in `RefsPanel.show` and
    `RefsPanelModel.prepareForPresentation`) with pattern matching or a computed helper
    such as `isHighlightsKeySession`.
  - Update existing call sites and tests to `.highlightsCopy(.card)`.
- Store the entry on the pending session, for example `.finding(token:, entry:)` or a
  sibling property cleared with the session. Both the failure messages and the skeleton
  need it.
- Refactor `handleCopyCardKey(characters:keyCode:modifiers:)` into a key-mapping wrapper
  around a new internal `handleCopyCardCommand(_ command: RefsCopyCommand)`. The command
  method keeps the shared timer, effect, and announcement work.
- In `resolvePendingHighlightsCopy()`, after `openCopyCard(optionHeld:)` succeeds:
  1. Dispatch `entry.seedCommand` (if any) through `handleCopyCardCommand`.
  2. Then replay the buffered keys as today.

  Order matters: chord, then quickly typed `Return`, must copy the page's block or link
  once the catalog loads. The card already queues commands behind a loading pick. If the
  seed's effect writes and closes the card (same_as_l on a reference without Markdown),
  later buffered keys must fall on a closed card harmlessly. They already do, because
  the card is nil.

- Failure messages name the pressed chord via `entry.retryGlyph`:
  - "Highlights didn't answer — try ⌘L again"
  - "Two references match “…” — pick one, then ⌘L"

  The `.card` strings stay byte-identical, so existing ⌘Y tests keep passing.

- `RefsCopySkeletonView`:
  - Header reads `COPY · CODE` / `COPY · LINKS` for the new entries; `.card` keeps
    `COPY`.
  - The accessibility label says "Copy code menu" / "Copy link menu" accordingly.
  - Keep the tokens and layout unchanged so the skeleton still morphs into the card.

### 4. The same chords inside Bob Refs

- `RefsKeyRouter`:
  - ⌘B (keyCode 11) and ⌘L (keyCode 37), Caps Lock ignored, map to a new
    `RefsCommand.showCopyEntry(RefsCopyEntry)`.
  - Return nil while IME text is marked, exactly like ⌘Y.
  - Other modifier combinations stay unrouted.
- `RefsPanelModel.perform(.showCopyEntry(entry))`: same guards as `.showCopyCard`
  (linking today; synthetic tab → `showNoteActionWarning`). Then `openCopyCard()`, and
  when a card opened, dispatch `entry.seedCommand` through `handleCopyCardCommand`.
- While a Copy card is already open, ⌘B and ⌘L stay consumed by the card's key map, as
  today. The letters C and L already work there. This keeps a repeated chord press
  during a Highlights pending session harmless.
- Leave the Bob Refs footer hint strings unchanged; they are width-budgeted and covered
  by `RefsFooterTests`. Discoverability comes from Settings and docs.

### 5. Error messages: every failure is visible and specific

| Failure                                                                                            | What the user sees                                                                                                                                 |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Accessibility not granted                                                                          | Pending session clears; panel shows the existing Tabs permission card (as with ⌘Y)                                                                 |
| Highlights has no open document / focused tab not current                                          | Skeleton red line "Highlights has no open document"                                                                                                |
| Focused PDF isn't a library reference                                                              | "“file.pdf” isn't in your Bob library"                                                                                                             |
| Two references match the PDF                                                                       | "Two references match “…” — pick one, then ⌘B" (or ⌘L); the in-panel chord makes that advice true                                                  |
| Tabs read failed                                                                                   | "Couldn't read Highlights — ⌘R retries"                                                                                                            |
| Highlights read takes > 2 s                                                                        | "Highlights didn't answer — try ⌘B again" (or ⌘L)                                                                                                  |
| Catalog still loading                                                                              | Card line "Loading code… esc cancels" / "Loading links… esc cancels", then the picker opens                                                        |
| Catalog load exceeds 5 s                                                                           | Red "Code unavailable — nothing copied" / "Links unavailable — nothing copied"                                                                     |
| No blocks / links, no saved Markdown, source missing or invalid, catalog not recorded, bob too old | The card opens, the red line shows the existing `hiddenReason` text, and the rest of the card stays usable (for example `M` still copies Markdown) |
| Hotkey registration fails (another app owns the chord)                                             | See below; previously only a raw Carbon status in Diagnostics                                                                                      |

**Registration conflicts.** A failed Carbon registration means the press never reaches
Bob, so the error must surface at registration time.

In `HighlightsTakeover.sync(action:configuration:)`, report a structured conflict
(action, `HotKeyConfiguration`, underlying error) instead of a bare string. Do this for
**all** Highlights takeover keys: they share this one code path, and the open, Tabs, and
Copy keys benefit equally.

- **Diagnostics line**, every failed sync, as today. Keep the `Refs hotkey conflict:`
  prefix that the bob-cli guide quotes, then name the chord, its action, and the fix.
  For example:
  `Refs hotkey conflict: Command-L (copy a link in Highlights) is already in use by another app — change it in Settings › References (Carbon RegisterEventHotKey failed with status -9878)`.
- **Glass toast**, at most once per launch per (action, chord):
  - Shown through a new `RefsPanelController` method that works while the panel is
    hidden, centered where the panel would appear (use
    `RefsVisualTokens.panelFrame(for: Self.visibleFrame())`).
  - Uses a new `RefsGlassToastStyle.warning` (`exclamationmark.triangle.fill`, orange).
  - Copy, for example: title "⌘L is taken by another app", value "Copy link in
    Highlights won't work", detail "Change it in Bob Settings › References".
  - Post the same text as an accessibility announcement.
  - A later successful registration of that action clears its "already reported" mark,
    and so does changing that setting. A new conflict then toasts again.

Give each takeover `HotKeyAction` a short purpose string ("open Bob Refs", "show open
tabs", "open the Copy card", "copy code", "copy a link") for these messages.

## Tests (mirror the existing ⌘Y tests)

- `Tests/RefsCoreTests` (runs under `just portable-check` on Linux):
  - `RefsCopyEntry` seed commands and retry glyphs.
  - On a `RefsCopyCard`:
    - the code seed while code is loading → pending, then the picker opens on load;
    - the code seed with a `.noCode` response → red hidden reason;
    - the link seed with saved Markdown → link picker;
    - the link seed without saved Markdown → per DECISION `link_without_markdown`.
- `HotKeyRegistryTests`: ids 6 and 7 are routed and registered independently.
- `RefsEntryPointsTests`:
  - The new keys register only for Highlights or the override app, and step aside while
    Refs is visible.
  - Off never registers them.
  - Settings default, persist, reload, and bogus-value fallback.
  - Changing either setting re-syncs live through `AppDelegate`.
  - `openRefsCopyFromHighlights(entry:)` presents `.highlightsCopy(entry)`.
  - Conflict reporting: a fixture with `failIDs` gets a diagnostic naming the chord on
    every sync, but exactly one toast callback across repeated syncs; it toasts again
    after a success-then-failure.
- `RefsPanelModelTests`:
  - A `.highlightsCopy(.code)` / `.links` session opens the card and lands in the code /
    link picker.
  - A buffered Return after the seed copies the page's block or link.
  - The timeout and collision messages name ⌘B / ⌘L, while `.card` keeps ⌘Y.
  - `perform(.showCopyEntry(...))` honors the synthetic-tab and linking-today guards.
- `RefsKeyRouterTests`: ⌘B and ⌘L (with and without Caps Lock) route; marked text passes
  through; ⇧⌘B, ⌥⌘L, and plain B/L stay unrouted. Plain Control-C still resets the view.
  If `code_key = ctrl_cmd_c`, test that ⌃⌘C routes.
- Update any skeleton or Copy card design test that asserts the `COPY` header, but only
  for the new entries.

## Docs (bob-mac-capture `docs/`, mounted into Bob's site)

`docs/refs.md` is at about 5,995 words against the 6,000-word reference-page cap
(bob-cli `tools/docs/bob_docs/lint.py`). Its net word count must not grow:

- Put the detail in `docs/keyboard.md`, which has ample room.
- In refs.md, edit in place and trim equal words elsewhere if needed.
- Never raise a ceiling.

Changes by page:

- `docs/keyboard.md`:
  - Bob Refs shortcuts table: rows for Command-B and Command-L (open the Copy card for
    the selected row with the code or link picker pressed; same as Command-Y C /
    Command-Y L), and a "(in Highlights)" row covering frontmost-only registration,
    yielding while Bob Refs is visible, configurable to Off, and replayed keys.
  - Copy card keys intro: mention the two chords.
  - Replace the two "Command-Y C Return / Command-Y L Return" lines with the one-press
    forms (Command-B Return copies the page's code block; Command-L Return copies the
    page's link).
  - A compact "When a Highlights copy key fails" list of the messages in the table
    above, including the conflict toast and the Settings fix.
- `docs/refs.md`, net-neutral edits:
  - The shortcuts table rows.
  - The IME pass-through sentence (add Command-B / Command-L).
  - The Copy card paragraph's last sentence.
  - The "Opening Bob Refs" Command-Y bullet: fold the two chords into it, say Command-C
    stays Highlights' Copy, and link to keyboard.md for detail.
- `docs/settings.md`:
  - Two new rows in the References table (defaults, choices, frontmost-only, yields
    while Bob Refs is visible).
  - The Carbon paragraph lists ids 6 and 7.
  - State that conflicts now also toast once per launch and name the chord.
- `docs/permissions.md`: "Command-Y in Highlights uses that same tab read" becomes
  "Command-Y, Command-B, and Command-L in Highlights use that same tab read".
- `docs/diagnostics.md`: the conflict sentence notes that takeover conflicts name the
  chord and action, and toast once per launch.

Do not edit bob-cli's mounted copies under `docs/reference/mac-capture/`; they are
generated from these sources.

## Verification (do not skip compiling)

1. In bob-mac-capture on athena: `just portable-check` (swift-format lint, plus build
   and test of the portable RefsCore/CaptureCore targets). It must pass.
2. `portable-check` cannot type-check `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`, so compile a scratch copy on the Mac (best effort, per that
   repo's `AGENTS.md`):
   - `rsync` the repo to a fresh scratch directory on `mac`, excluding `.build` and
     `.git`.
   - Run `Scripts/bundle.sh` and `just typecheck-tests` there.
   - Delete the scratch copy.
   - Never use Bryan's checkout or `just install`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.
3. `just docs-check` from bob-mac-capture with `BOB_CLI_DIR` set to the bob-cli checkout
   root (the directory containing `sase/repos/linked/bob-mac-capture`). The linked
   layout has no `../bob-cli` sibling. It must pass, including the refs.md word budget.
4. Report what could not be exercised live: the real Carbon conflict toast and the real
   Highlights round trip need a GUI session on the Mac.

The bob-mac-capture changes are committed through this turn's final declaration, as that
repo's obligation. Use Conventional Commit style, for example
`feat(refs): add Highlights code and link copy keys`.

## Non-goals

- Taking over Command-C by default, or any AX- or event-tap-based "smart" Command-C.
- New ⌘K actions-menu items or footer hints for the chords.
- Any bob-cli, bob JSON contract, or vault change.
