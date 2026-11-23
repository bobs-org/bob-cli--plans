---
tier: tale
title: Highlights Control-Return opens the reference's reading task
goal:
  While Highlights is frontmost, Control-Return opens the focused PDF's reading task in
  Obsidian, as Command-T then Control-Return in Bob Refs would.
size: medium
decisions:
  presentation:
    ask:
      Open the reading task without showing Bob Refs when the focused PDF resolves
      cleanly?
    choices:
      headless:
        Obsidian jumps straight to the task; Bob Refs appears only when the task can't
        open
      panel:
        Bob Refs flashes up in Tabs, then the task opens exactly as ⌘T then ⌃↩ would
    default: headless
    why:
      Showing Refs only to vanish when Obsidian takes focus adds nothing; failures still
      land on Tabs
    answer: headless
proposed_by: bbugyi200.athena.1fm
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 11:37:43
status: wip
---

# Plan: Highlights Control-Return opens the reference's reading task

## Goal

Add a Highlights-frontmost takeover key, **Control-Return** by default, to Bob Mac
Capture. It opens the focused PDF's reading task (the reference's single `#task #ref`
line, called the Reference Task in the glossary) in Obsidian. The result must match what
the user gets today by pressing **Command-T** in Highlights (Bob Refs opens in Tabs from
fresh defaults with the focused PDF selected) and then **Control-Return** in Bob Refs
(open the selected row's reading task).

All code and docs changes live in the linked **`bob-mac-capture`** repo. Open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`) and read its `AGENTS.md`
before editing. bob-cli needs no change: `docs/reference/mac-capture/*` in bob-cli is
generated from bob-mac-capture's `docs/` and is gitignored, so do not edit it.

## How the pieces work today

All paths are relative to the bob-mac-capture repo root.

- **Takeover keys.** `Sources/BobMacCapture/HighlightsTakeover.swift` registers Carbon
  hotkeys through `HotKeyRegistry` only while Highlights (or the Settings override app)
  is frontmost. Each key has a per-key settings enum (`RefsHighlightsOpenKey`,
  `RefsHighlightsTabsKey`, `RefsHighlightsCopyKey`, `RefsHighlightsCodeKey`,
  `RefsHighlightsLinkKey` in `Sources/BobMacCapture/RefsHighlights*Key.swift`), a
  `HotKeyConfiguration` constant, and a `HotKeyAction` case (ids 1–9 in
  `HotKeyRegistry.swift`). The Copy, Code, and Link keys step aside while Bob Refs is
  visible (`isRefsVisible()`). `RefsPanelController.onVisibilityChange` re-runs
  `takeover.sync()`.
- **Dispatch.** `AppDelegate`'s `HotKeyRegistry { action in … }` switch routes each
  action to `BobPanelCoordinator`: `.refsHighlightsTabs` goes to `toggleRefsTabs()` and
  the copy keys go to `openRefsCopyFromHighlights(entry:)`. `configureRefsStack` builds
  the takeover. `observeRefsSettings()` re-syncs it live through a `combineLatest` of
  the `@Published` key settings.
- **Settings.** `AppSettings` persists each key under a `UserDefaults` key with a
  default. `SettingsView` shows a picker per key in Settings › References.
- **Command-T from Highlights.** `RefsPresentationSource.highlightsTabs` makes
  `RefsPanelModel.prepareForPresentation` stash the remembered view and call
  `resetToDefaults(tabs: true, …)`. The tab read (`beginTabsRefresh` →
  `applyTabsReadResult`) matches tabs with `HighlightsTabMatcher.match` and ranks them
  with `RefsTabsRanker.listing`, which preselects `currentID` (the focused PDF).
- **Control-Return in Bob Refs.** `RefsKeyRouter` maps ⌃↩ (and ⌃keypad-Enter) to
  `.open(.task)`. `RefsPanelModel.openSelected(.task)` resolves
  `library.taskURL(for: item)` and calls
  `opener.openTask(url, blockID: item.task?.blockID)`, which is
  `WorkspaceRefsOpener.taskDeepLink` → `obsidian://bob-goto-task`. With no task it beeps
  and posts "No reading task for this reference". Task opens never count as opens. The
  model never dismisses the panel: Obsidian activating makes the non-activating panel
  resign key, and `onResignKey` hides it.
- **Prior art for "resolve the focused PDF".** The Command-Y/B/L copy session
  (`armPendingHighlightsCopy` / `resolvePendingHighlightsCopy` in `RefsPanelModel`)
  already owns the failure wording, a 2 s deadline
  (`highlightsCopyDeadlineNanoseconds`), and the tests in `RefsPanelModelTests`
  (`testHighlightsCopy*`, `FakeTabsReader`, `tabsSnapshot(…)`).

### Capture panel conflict (must handle)

Both Bob panels are `.nonactivatingPanel`s, so Highlights stays frontmost while Capture
is open over it. Capture uses Control-Return heavily (capture and open, picker "Insert,
Capture & Open", Go To). A ⌃↩ takeover that ignores Capture would steal those presses.
The new key must therefore yield while **Bob Refs or Capture** is visible. Today
`CapturePanelController` has no visibility callback and the takeover never looks at
Capture, so this plan adds both. The existing ⌘O/⌘T/⌘Y/⌘B/⌘L keys keep their current
rules. Capture binds none of those chords, so this plan leaves them unchanged.

## Design

### 1. The setting and its registration

- Add `Sources/BobMacCapture/RefsHighlightsTaskKey.swift`:
  `enum RefsHighlightsTaskKey: String, CaseIterable, Sendable { case ctrlReturn = "ctrlReturn"; case off = "off" }`.
  Display names are "Control-Return" and "Off". `configuration` returns
  `.highlightsControlReturn` or nil. Follow the shape of `RefsHighlightsLinkKey`.
- `HotKeyRegistry.swift`:
  - Add
    `static let highlightsControlReturn = HotKeyConfiguration(keyCode: UInt32(kVK_Return), modifiers: UInt32(controlKey), displayName: "Control-Return")`.
  - Add `HotKeyAction.refsHighlightsTask = 10` with purpose `"open the reading task"`.
    Append it; never renumber the existing ids.
  - Register Return only. Keypad Enter would need a second action, and Mac keyboards
    rarely send it.
- `HighlightsTakeover`:
  - Add `taskKey: () -> RefsHighlightsTaskKey` (default `{ .off }`) and
    `isCaptureVisible: () -> Bool` (default `{ false }`).
  - Add `taskKey` to `SettingsSnapshot` and clear its reported conflicts when it
    changes.
  - Register `.refsHighlightsTask` only when the expected app is frontmost and
    **neither** Refs nor Capture is visible:
    `isRefsVisible() || isCaptureVisible() ? nil : taskKey.configuration`.
  - Unregister it on every path that unregisters the others: the not-frontmost branch
    and `stop()`.
  - Extend the public `sync(openKey:…appPath:)` overload with
    `taskKey: RefsHighlightsTaskKey = .off`.
- `HotKeyConflict.toastValue`: `.refsHighlightsTask` → "Opening the reading task from
  Highlights won't work". Keep the switch exhaustive.
- `AppSettings`:
  - Add `@Published var refsHighlightsTaskKey` persisted under
    `"refsHighlightsTaskKey"`.
  - Default it to `.ctrlReturn` when the key is unset or unparsable, the same way as the
    other keys.
- `SettingsView`: add a picker "Reading task key in Highlights" with Control-Return and
  Off, right after "Copy link key in Highlights".

### 2. Capture visibility drives a re-sync

- `CapturePanelController`:
  - Add `var onVisibilityChange: () -> Void = {}`.
  - Call it after the panel orders in, at the end of `show()`, and after it orders out
    in `hidePanel()`.
  - Check every other path that orders the capture panel in or out: grep `orderOut`,
    `makeKeyAndOrderFront`, `close()`, and resign-key handling. Each of those paths must
    also fire the callback, so the takeover never stays registered over a visible
    Capture panel.
- `AppDelegate`:
  - Set
    `panelController?.onVisibilityChange = { [weak self] in self?.highlightsTakeover?.sync() }`.
  - Pass `isCaptureVisible: { [weak self] in self?.panelController?.isVisible == true }`
    and `taskKey: { settings.refsHighlightsTaskKey }` into `HighlightsTakeover`.
  - Add `settings.$refsHighlightsTaskKey` to the `observeRefsSettings()` combine chain
    (for example
    `$refsHighlightsCodeKey.combineLatest($refsHighlightsLinkKey, $refsHighlightsTaskKey)`).
    Each sink applies the values it receives, never re-reading settings. That is the
    existing rule, because `@Published` emits before the property changes.

### 3. Dispatch

In `AppDelegate`'s hotkey switch, add:

```swift
case .refsHighlightsTask:
    CaptureSignpost.event("refs-task-hotkey-received")
    Task { @MainActor in self?.panelCoordinator?.openRefsTaskFromHighlights() }
```

Add `BobPanelCoordinator.openRefsTaskFromHighlights()`. What it does depends on the
`presentation` decision.

> [!decision] presentation = headless The key resolves the focused PDF **without showing
> Bob Refs** and opens the task directly. Bob Refs appears only when the task cannot
> open.
>
> - **Model API** (`RefsPanelModel`):
>   `public func openFocusedHighlightsTask() async -> RefsHighlightsTaskOutcome`, with
>   `enum RefsHighlightsTaskOutcome: Equatable, Sendable { case opened, dropped, fallback(message: String?) }`.
>   Steps, in order:
>   1. If a previous resolve is still in flight, return `.dropped`. A repeated press
>      never opens the task twice.
>   2. If `!tabsSession.reader.isTrusted()`, return `.fallback(nil)`. The Tabs
>      permission card explains why; never prompt.
>   3. If the library has no successful snapshot yet (`library.lastSuccessAt == nil`),
>      return `.fallback(nil)`. Tabs shows its loading state; do not claim "isn't in
>      your library".
>   4. Read the tabs, racing a deadline. Add `highlightsTaskDeadlineNanoseconds`,
>      default 2 s, as a test seam like the copy one.
>      - Timeout: `.fallback("Highlights didn't answer — try ⌃↩ again")`. Discard the
>        late result.
>      - A concurrent ⌘S-scan read must not cancel this read.
>        `HighlightsTabsSession .refresh()` returns nil when a newer refresh starts.
>        Either retry once inside the deadline, or add a session read that touches
>        recency the way `refresh()` does but does not bump the shared generation.
>   5. Map the read result:
>      - `.notRunning`: `.dropped`. Highlights quit; stay silent.
>      - `.notTrusted`: `.fallback(nil)`.
>      - `.failed`: `.fallback("Couldn't read Highlights — ⌘R retries")`.
>      - `.snapshot`: match with
>        `HighlightsTabMatcher.match(snapshot, items: library.items, vaultRoot: library.vaultRootURL)`.
>        Rank with
>        `RefsTabsRanker.listing(…, query: "", scope: .all, signals: library.signals, recency: tabsSession.recency, highlightsIsFrontmost: snapshot.highlightsIsFrontmost)`,
>        exactly as a fresh ⌘T show would. Take its **`currentID`**, the focused PDF. Do
>        not use the `ids.first` fallback.
>   6. Resolve the current tab with the copy session's wording:
>      - No current tab: `.fallback("Highlights has no open document")`.
>      - `.collision`:
>        `.fallback("Two references match “<fileName>” — pick one, then ⌃↩")`.
>      - `.none` or no `itemID`: `.fallback("“<fileName>” isn't in your Bob library")`.
>      - Item with no `library.taskURL(for:)`:
>        `.fallback("“<title>” has no reading task")`.
>   7. Drop the result (`.dropped`) when `presentationCount` changed during the read.
>      That means Bob Refs was shown in the meantime, for example by ⌘T or ⌘O.
>   8. Otherwise open through the **same helper** that `openSelected(.task)` uses, then
>      return `.opened`. Extract a private `openReadingTask(for item: RefItem) -> Bool`
>      from the `.task` branch so both paths build the identical deep link. Record
>      nothing in the open log.
> - **Fallback presentation**:
>   - Add `RefsPresentationSource.highlightsTaskFallback(String?)`, with
>     `isHighlightsKeySession == true`. It behaves exactly like `.highlightsTabs`: stash
>     the view, `resetToDefaults(tabs: true, announcement: nil)`, and the next ordinary
>     show restores the stash.
>   - When the message is non-nil, it also sets
>     `banner = RefsBanner(kind: .warning, message:, actions: [])` after the reset.
>   - Verify with a test that the Tabs load that follows does not clear that banner.
> - **Coordinator**:
>   - `openRefsTaskFromHighlights()` returns at once when Refs is visible. Otherwise it
>     starts a `Task { @MainActor in … }` that awaits an injected
>     `resolveHighlightsTask: () async -> RefsHighlightsTaskOutcome`. Wire that to
>     `model.openFocusedHighlightsTask()` in `AppDelegate`; the default is
>     `{ .dropped }`, so existing coordinator tests compile unchanged.
>   - On `.fallback(message)`, while Refs is still hidden, call
>     `showRefs(source: .highlightsTaskFallback(message))`. A visible Capture draft is
>     retained exactly as `showRefs` already does.
>   - `.opened` and `.dropped` present nothing.

> [!decision] presentation = panel The key literally replays ⌘T then ⌃↩.
>
> - **Presentation source**: add `RefsPresentationSource.highlightsTask`, with
>   `isHighlightsKeySession == true`. `prepareForPresentation` arms a pending task open
>   (`pendingHighlightsTask` token plus `highlightsTaskDeadlineNanoseconds`, 2 s), then
>   calls `resetToDefaults(tabs: true, announcement: nil)`. This mirrors
>   `.highlightsCopy`.
> - **Resolution**:
>   - Add `resolvePendingHighlightsTask()` next to every
>     `resolvePendingHighlightsCopy()` call site: the untrusted branch of
>     `beginTabsRefresh` and the end of `applyTabsReadResult`.
>   - On `.loaded`, when the selected row is the current tab and matched to one item
>     with a reading task, clear the pending state and open through the shared
>     `openReadingTask(for:)` helper (extracted from `openSelected(.task)`). Obsidian's
>     activation then hides the panel through the existing resign-key path.
>   - On failure, set a warning banner with the wording listed under the headless
>     branch.
>   - `.needsPermission` and `.notRunning` clear the pending state silently; the Tabs
>     card already explains them.
>   - A timeout sets the "didn't answer" banner.
> - **Cancellation**: clear the pending state when another presentation source runs, in
>   `panelDidHide()`, and on any routed panel command (Escape, selection moves) before
>   it resolves. The user's own keys win.
> - **Coordinator**: `openRefsTaskFromHighlights()` calls
>   `showRefs(source: .highlightsTask)`.

### 4. Failure surfaces stay on the ⌘T view

Any outcome that does not open the task lands on the view ⌘T would have shown: Bob Refs
in Tabs from fresh defaults, with the focused PDF preselected. The reason sits in a
warning banner. Without Accessibility, the existing Tabs permission card is the
explanation, so there is no banner. Nothing is written to the vault, and reading state,
the open log, and NEW marks are untouched.

## Tests

Swift tests run only on macOS CI, so mirror the existing patterns exactly.

- **`Tests/BobMacCaptureTests/RefsEntryPointsTests.swift`** (takeover and coordinator)
  - Extend `makeTakeover` / `TakeoverFixture` with `taskKey` and a mutable
    `captureVisible`.
  - New cases:
    - Registers `.highlightsControlReturn` only for Highlights or the override app.
    - `.off` never registers.
    - Yields while Refs is visible, and separately while Capture is visible; it
      re-registers when both hide.
    - The other takeover keys stay registered while Capture is visible (no regression).
    - Changing the key swaps the registration live.
    - `stop()` and Highlights termination unregister it.
    - A conflict names "Control-Return (open the reading task)" and toasts once with the
      new toast value.
  - Coordinator, headless: `.fallback("x")` presents `.highlightsTaskFallback("x")`;
    `.opened` and `.dropped` present nothing; a visible Refs panel skips the resolve.
  - Coordinator, panel: presents `.highlightsTask`.
  - `AppSettings`: defaults to `.ctrlReturn`, round-trips `.off`, and ignores an unknown
    raw value.
- **`Tests/BobMacCaptureTests/RefsPanelModelTests.swift`** (reuse `FakeTabsReader`,
  `tabsSnapshot(…)`, and the task-record harness behind
  `testTaskOpenOpensReadingTaskFile`)
  - Success:
    - Opens the focused PDF's task URL with its block ID.
    - Opens no notes and records no open-log entry.
    - For headless, shows no presentation (`presentationCount` unchanged).
  - Each failure produces its documented message:
    - unmatched PDF
    - collision
    - no documents
    - failed read
    - timeout (shortened deadline)
    - no reading task
    - untrusted (no read attempted, nil message)
    - library not yet loaded
  - Headless only:
    - A second call while one is in flight returns `.dropped`.
    - A presentation during the read drops the result.
    - The fallback show keeps Tabs on with the banner, and a following `.library` show
      restores the stashed view.
  - Panel only: Escape before the tabs load cancels without opening.
- `HotKeyRegistryTests`: `handle(hotKeyID:)` routes id 10 to `.refsHighlightsTask`.
- Capture visibility: if existing `CapturePanelController` tests can drive `show()` /
  dismissal, assert the callback fires on both. Otherwise the takeover fixture covers
  the rule.

## Docs

Edit these files in bob-mac-capture's `docs/`; they mount into Bob's site:

- `docs/keyboard.md` › Bob Refs shortcuts:
  - Add a row: "Control-Return (in Highlights)". While Highlights or its override app is
    frontmost and no Bob panel is visible, it opens the focused PDF's reading task in
    Obsidian at the task line, the way Command-T then Control-Return would. It falls
    back to Tabs with a reason when it can't, and can be set to Off.
  - Add a short "When the Highlights task key can't open" list next to "When a
    Highlights copy key fails", with the messages above.
- `docs/refs.md`:
  - Add a bullet under "Opening Bob Refs".
  - Add one sentence in "Tabs" saying the key lands there on failure.
  - State that it yields to visible Bob Refs **and** Capture, so Capture's own
    Control-Return keeps working over Highlights.
- `docs/settings.md`:
  - Add a row "Reading task key in Highlights | Control-Return | `Control-Return` or
    `Off`…".
  - Add "task key (id 10)" to the `HotKeyRegistry` id list.
- `docs/permissions.md`: line 12's sentence about Command-Y/B/L reusing the tab read
  also covers Control-Return.
- Describe what the key does, not how it is built. The headless branch never mentions a
  panel flash; the panel branch says Bob Refs opens in Tabs first.

Run `just docs-check` in bob-mac-capture.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

1. Run `just portable-check` on a host with Swift (athena works). It lints formatting
   and builds and tests the portable targets. It cannot type-check
   `Sources/BobMacCapture*` / `Tests/BobMacCapture*`.
2. Compile those on the mac, best effort:
   - rsync a scratch copy without `.build`/`.git`.
   - Run `Scripts/bundle.sh`, then `just typecheck-tests`.
   - Delete the copy.
   - Never use Bryan's checkout or `just install`.
   - If the mac is offline, say so in the final report: the macOS targets are
     unverified.
3. Format only the lines you change, with `.swift-format`: 4-space indent, 100 columns.
4. Run `Scripts/macos-ci-check` before landing.
5. After the push, watch the exact-SHA macOS CI run to green with `/sase_monitor`. If
   **Lint Swift formatting** is red, check the later steps too.
6. Manual check, best effort, on the mac with the installed app:
   - With a matched PDF focused in Highlights, ⌃↩ lands Obsidian on its `^ref-…` task
     line.
   - An unmatched PDF lands on Tabs with the banner.
   - With Capture open over Highlights, ⌃↩ still captures and opens.
   - Check whether Highlights itself binds ⌃↩ anywhere (menus, note editor). If it does,
     name that in the keyboard docs row as what Off restores.

## Out of scope

- The existing ⌘O/⌘T/⌘Y/⌘B/⌘L takeovers keep their Capture-visibility behavior.
- No bob-cli, bob-plugins (`obsidian://bob-goto-task` already exists), or vault change.
- No keypad-Enter variant of the takeover.
