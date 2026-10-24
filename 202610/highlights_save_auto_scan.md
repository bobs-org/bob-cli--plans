---
tier: tale
title: Scan the reference library when ⌘S saves a PDF in Highlights
goal:
  ⌘S in Highlights runs an attended `bob ref scan -w` once the save lands, safely
  alongside coordinated marker writes and in-place reloads, and the Bob Refs inspector
  refreshes itself whenever a scan rewrites reference notes.
size: medium
decisions:
  save_confirmation:
    ask:
      Should a scan started by ⌘S in Highlights notify when it only updated existing
      notes?
    choices:
      silent:
        Quiet on success; the PDF reload, Refs footer, and existing notices report it
      notify: One passive, silent notification per such scan naming the updated notes
    default: silent
    why:
      A notice on every save is noise when the reload and footer already confirm the
      sync
    answer: silent
proposed_by: bbugyi200.apollo.a6
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 10:52:29
status: wip
---

# Scan the reference library when ⌘S saves a PDF in Highlights

## Goal

1. Pressing ⌘S in the Highlights app (the PDF reader) makes Bob Mac Capture run
   `bob ref scan -a -w -f json` on its existing Bob Refs scan lane, starting once
   Highlights' save has landed on disk.
2. That scan works with the coordinated marker-write integration
   (`highlights.coordinate_pdf_writes`, already `true` in the managed Mac bob config):
   it is attended, it never races Highlights' own save, it never loops on bob's own
   writes or Highlights' reloads, and a save made while a scan is running is not lost.
3. The Bob Refs inspector (the selected reference's notes and highlights) refreshes by
   itself whenever a `bob ref scan` rewrites reference notes. That covers scans started
   by ⌘S in Highlights, ⌘S in Bob Refs, the 15-minute cron, or a terminal.

## Repositories and scope

- **bob-mac-capture** (linked repo; open it with `/sase_repo` and read its `AGENTS.md`
  first): all code, tests, and app docs.
- **bob-cli** (this repo): docs only. There are no CLI, JSON, or contract changes. Do
  not touch `Cargo.toml`'s version or `CHANGELOG.md`; the release train owns them.
- No vault, chezmoi, bob-plugins, or SASE memory changes. No memory note becomes false.
  `decisions:open-pdf-writes-are-coordinated` still holds: the app reacts to a person's
  own ⌘S and never presses keys or drives Highlights.
  `decisions:mac-capture-is-a-thin-client` also still holds: the app only orchestrates
  `bob` processes and never reads or writes the vault itself.

## Current behavior (verified in the code)

- Bob Refs ⌘S → `RefsPanelModel.perform(.scan)`
  (`Sources/BobMacCapture/Refs/RefsPanelModel.swift`) → `RefsLibrary.scan()`
  (`Sources/BobMacCapture/Refs/RefsLibrary.swift`) → `BobRefsFetcher.scan()`
  (`Sources/RefsCore/RefsFetching.swift`) runs `bob ref scan -a -w -f json` on the
  `refs-scan` lane with a 300 s timeout. If an older `bob` rejects `-a`, it is retried
  without it. The lane is single-flight: `scan()` returns false while a scan or rename
  runs. Afterwards a post-scan snapshot refresh runs, and then `scanState` publishes
  `.finished`. A hidden panel notifies only for failures, partial scans, or created
  references.
- `RefsScanReceiptObserver` (`Sources/BobMacCapture/Refs/RefsScanReceiptObserver.swift`)
  handles bob's Darwin `org.bobs.bob.ref.scan-finished` ping for every writing scan. It
  presents open-PDF refreshes and holds: a footer clause, "Updated in Highlights", or
  "Highlights needs you".
- The Highlights-frontmost keys ⌘O and ⌘T are Carbon hotkeys that `HighlightsTakeover`
  (`Sources/BobMacCapture/HighlightsTakeover.swift`) registers only while Highlights, or
  the Settings override app, is frontmost. A Carbon hotkey **consumes** the key, so ⌘S
  cannot use this mechanism: Highlights must still receive ⌘S and save.
- `RefsLibrary.startWatcher()` watches the vault root plus `ref/`, `lib/`, and `done/`
  through `VaultTargetWatcher` (0.5 s latency). Each relevant batch refreshes the
  snapshot and bumps `snapshotRefreshGeneration`. Batches that arrive during a scan or
  rename are deferred into one post-scan refresh.
- `RefsInspectorLoader` (`Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`) caches
  `bob ref show` per `(note id, snapshotFetchedAt)`. Hydration is requested only when
  the selection changes (`RefsPanelView`'s `.onChange(of: selected)`) and at
  presentation (`inspectorRequested()`). **Nothing re-requests hydration when a snapshot
  refresh lands while the panel is visible**, so the selected row shows stale highlights
  until the selection moves or the panel is shown again. A typical case: ⌘S in
  Highlights, then ⌘T or ⌘O opens Bob Refs while the scan is still running. The
  inspector hydrates from pre-scan data and never catches up.
- bob-cli already provides `-a/--attended`, coordinated `.forMerging` marker writes with
  holds, the scan receipt, and the Darwin ping. See bob-cli
  `docs/highlights-ref-sync.md` § "PDFs open in Highlights" and § "Scan receipt".

## Design

### 1. Observe ⌘S in Highlights without taking it

- Add `HighlightsSaveKeyMonitor` (AppKit glue in `Sources/BobMacCapture/`). It uses the
  passive `NSEvent.addGlobalMonitorForEvents(matching: .keyDown)`. Never use a Carbon
  hotkey or an event tap that can swallow or modify events, and never re-post keys:
  Highlights receives ⌘S untouched and saves as usual.
- Install the monitor only while all three hold, and remove it otherwise:
  - the new setting is on;
  - the frontmost app's bundle id equals Highlights or the Settings override. Use the
    same resolution as `HighlightsTakeover.expectedBundleID`;
  - `AccessibilityTrust.isTrusted()` is true. Never prompt.

  Re-evaluate on every activate, deactivate, and terminate notification and on setting
  changes. A grant given later then takes effect at the next Highlights activation. Wire
  it beside `HighlightsTakeover`: either extend `sync()` through an injected seam, or
  add a sibling observer of the same workspace notifications. Keep it testable with
  injected `frontmostBundleID`, `isTrusted`, and a small `GlobalKeyMonitorInstalling`
  protocol so tests never need real events.

- Matching is a pure RefsCore helper, `HighlightsSaveKey.matches(...)`, that takes the
  event's `charactersIgnoringModifiers`, the command/shift/option/control booleans, and
  `isARepeat`. The rule: the character is `s` (case-insensitive), Command is the only
  modifier among command/shift/option/control (Caps Lock, Fn, and keypad flags are
  ignored), and the event is not a repeat. ⇧⌘S, ⌥⌘S, and ⌃⌘S do not match.
- Privacy: the handler inspects only that one event's characters and modifier flags.
  Nothing is logged or stored. Signposts carry event names only (`refs-save-scan-armed`,
  `refs-save-scan-skipped`, `refs-save-scan-fired`).
- Global monitors never see events delivered to Bob Mac Capture itself. ⌘S inside Bob
  Refs or the Capture panel therefore keeps its current meaning, and nothing fires
  twice. Bob Refs is non-activating and hides when it resigns key, so ⌘S typed into
  Highlights always means Highlights' Save.
- Contingency, checked during Mac verification: if no key events arrive while
  Accessibility is granted (because that macOS version wants Input Monitoring for this
  API), gate on `CGPreflightListenEventAccess()` as well. Show its status in Settings ›
  References and request it only from an explicit Settings button via
  `CGRequestListenEventAccess()`. Never prompt at launch.

### 2. Wait for the save to land, then scan

The scan reads the page-1 marker from the PDF bytes and the highlights from the sidecar
that Highlights autosaves. A scan that starts before the save lands plans from stale
bytes and races the coordinated write. So ⌘S _arms_ a trigger, and the save's own file
events (plus a short quiet period) _fire_ it.

- Add a pure RefsCore state machine, `HighlightsSaveScanTrigger`. It is a value type,
  all times are passed in as `Date`, and it owns no timers. API: `keyPressed(at:)`,
  `libraryChanged(_ batch: VaultChangeBatch, at:)`, `disarm()`, `nextDeadline: Date?`,
  and `poll(at:) -> Bool` (true exactly once per armed cycle, when the scan should
  start). Constants:
  - **quiet**: 1.0 s after the last relevant library change;
  - **fallback**: 2.5 s after ⌘S with no relevant change (nothing was unsaved, or an
    event was coalesced away);
  - **cap**: 10 s after the first ⌘S of the cycle, regardless of ongoing changes.

  A repeated ⌘S while armed coalesces into the same single scan. It may push the
  fallback out but never the cap. Changes are ignored unless the trigger is armed.

- A change is _relevant_ when a path has no dot-prefixed component and sits under
  `<vault>/lib/` or `<vault>/xlib/`, including PDFs, Markdown sidecars, and
  `.textbundle` contents. Incomplete-batch flags (`mustScanSubDirs`, `rootChanged`,
  `kernelDropped`, `userDropped`) also count as relevant. Standardize and resolve
  symlinks the way `RefsWatcherRelevance` does.
- Library scope: right after arming, read the front document through the existing
  `HighlightsTabsSession` / `AXHighlightsTabsReader`, using the main window's selected
  tab `documentPath`. A pure helper,
  `HighlightsSaveScanScope.isLibraryDocument(path:vaultRoot:) -> Bool?`, decides the
  result:
  - a path outside `<vault>/lib/` and `<vault>/xlib/` disarms the trigger (signpost
    `refs-save-scan-skipped`);
  - an unknown path (read failed, no document path) keeps the trigger armed, because an
    extra scan is harmless.
- Feeding changes: give `RefsLibrary` an observer hook, for example
  `var onWatcherBatch: (VaultChangeBatch) -> Void`. The watcher's `onBatch` calls it for
  every batch **before** the relevance and deferral logic, so the trigger sees the save
  even while a running scan defers refreshes.
- Timer: the glue owns one cancellable timer (a `Task.sleep` or `DispatchSourceTimer`)
  aimed at `nextDeadline`. When `poll` returns true, it calls
  `RefsPanelModel.requestScan(origin: .highlightsSave)` (section 3).

### 3. Scan lane: one queued follow-up

- `RefsLibrary.scan()` keeps its contract for Bob Refs ⌘S: busy means it returns false,
  and "a second ⌘S while one runs does nothing" stays true for the panel key.
- Add `RefsLibrary.requestScan(queueIfBusy: Bool) -> RefsScanRequest`, returning
  `.started`, `.queued`, or `.ignored`. With `queueIfBusy`, a request during a scan or a
  rename sets one `followUpScanPending` Bool, so any number of requests collapse to one.
  A pending follow-up starts exactly one new scan after `runScanPass` has finished its
  post-scan refresh and published `.finished`, or after a rename ends. The reason: the
  running scan may have planned from bytes older than the newest save. Its coordinated
  write would then hold with `marker_changed`, and without a follow-up the next retry
  would be the 15-minute cron.
- Add `RefsPanelModel.requestScan(origin: RefsScanOrigin)`:
  - `.panelKey` is today's `.scan` action body, unchanged. `perform(.scan)` calls it.
  - `.highlightsSave` runs the same preparation when a scan actually starts (clear a
    scan banner, `scanNotice`, `pdfUpdateFooterClauses`, set `selectionAtScanStart`) and
    passes `queueIfBusy: true`. Extract the preparation into one helper so both origins
    share it, including when a queued follow-up starts.

  Remember the origin of the scan in flight (needed by the `save_confirmation`
  decision).

- Same argv, `bob ref scan -a -w -f json`, through `BobRefsFetcher.scan()`; no new argv.
  `-a` is essential: the user has just been typing in Highlights, so an unattended scan
  would hold every marker write as `editor_active`.

### 4. Coordinated marker writes and in-place reloads

Nothing new is needed in bob. The design keeps the integration safe:

- The scan is attended (`-a`), so marker writes are not held while the user is reading.
- The scan starts after the save has landed (section 2). The `.forMerging` coordinated
  write therefore meets a document Highlights has just saved, writes on top of the fresh
  bytes, and Highlights reloads the same tab in place.
- bob's own coordinated write, Highlights' save-first, its reload, and its sidecar
  re-export all produce library file events but no keypress. They can never arm the
  trigger, so no scan loop is possible. A unit test pins this.
- Holds and refreshes keep presenting through the existing receipt observer, unchanged.
- Coordination must be on. The managed config sets
  `highlights.coordinate_pdf_writes: true`, and Mac verification checks `bob ref doctor`
  for `pdf_write_coordination: on`. App docs note that with coordination off, a marker
  write to an open PDF can raise Highlights' "changed by another application" dialog.
  Pick **Save Anyway**, as the bob-cli docs say.

### 5. The inspector follows library refreshes

- In `RefsPanelModel.init`, subscribe to `library.$snapshotRefreshGeneration`, dropping
  the initial value, and call a new `refreshInspectorAfterLibraryRefresh()`. It acts
  only when all of these hold:
  - the panel is visible;
  - a selected item still exists in the library;
  - the id is not preview-pinned;
  - no rename is in flight.

  It then re-requests hydration for the selected item. Add a loader `refresh(_:)` entry
  that skips the 120 ms selection settle but otherwise shares `request`'s code path. The
  new `snapshotFetchedAt` changes the `ref show` cache key, so `bob ref show` runs
  fresh. The intrinsics cache keys on file size and mtime, so a rewritten PDF reloads
  its thumbnail and outline too.

- Keep the user's place:
  - Do not bump `inspectorGeneration`. It is part of the inspector's view identity, so
    the scroll offset survives.
  - Keep the old content on screen until the new content publishes; never fall back to
    the skeleton. The loader already publishes only on change.
  - Keep revealed highlight pages, clamped if the feed shrank. Check `RefsNotesPage` and
    add the clamp if it is missing.
- A hidden panel needs nothing new: presentation already calls `inspectorRequested()`,
  and the generation subscription covers a refresh that lands after the show.
- Open cards stay as they are. The Copy card keeps the notes it loaded when it opened,
  frozen as today, and the Rename and Link to Today cards are untouched.
- This covers every scan source. In-app scans end with the post-scan refresh; cron and
  terminal scans write `ref/`, which the vault watcher turns into a refresh.

### 6. Settings and permissions

- Add `AppSettings.refsScanOnHighlightsSave`, stored in `UserDefaults` under
  `refsScanOnHighlightsSave` and on by default (a missing key reads as on, like
  `refsNotifyOpenPDFUpdates`).
- In Settings › References, add the toggle **Scan after ⌘S in Highlights** with the
  caption "Runs `bob ref scan -a -w` once Highlights finishes saving. Uses the
  Accessibility grant shown under Highlights tabs." Toggling it re-syncs the monitor
  immediately through a sink in `AppDelegate.observeRefsSettings()`, which applies the
  emitted value.
- When the toggle is on but Accessibility is not trusted, show "Needs Accessibility"
  beside it. Never prompt from Settings (existing rule); the grant path stays Bob Refs ›
  Tabs › Open Accessibility Settings.

### 7. Hidden-panel confirmation

> [!decision] save_confirmation = silent Add no new notification code. Today's rules
> decide what appears when Bob Refs is hidden: failures, partial scans, and created
> references notify through `scanNotifier`; open-PDF refreshes and holds notify through
> the receipt observer.

> [!decision] save_confirmation = notify When a `.highlightsSave` scan finishes
> successfully while Bob Refs is hidden, with updated notes but no created references or
> failures (those already notify), post one passive, silent notification. Title it
> "Synced from Highlights" and put the updated titles, joined by `·`, in the body
> (bounded like the existing multi-title copy). Thread it on
> `org.bobs.bob-mac-capture.refs-pdf-updates` in the `refs-scan` category. Clicking it
> opens Bob Refs and selects the first updated reference. Turning off "Tell me when Bob
> updates an open PDF" suppresses it. Build the copy in `RefsCore` and cover it with a
> presentation test.

## Docs

- **bob-mac-capture** (mounted into Bob's site; run `just docs-check`):
  - `docs/refs.md`: in the intro and **Updates**, cover ⌘S in Highlights, the queued
    follow-up, and the inspector refresh. Add a subsection, "Scan after saving in
    Highlights", covering the trigger, settle timing, library scope, the coordinated
    reload, the coordination-off caveat, and scan time (about 8 s for roughly 350 PDFs).
  - `docs/refs-inspector.md`: the inspector refreshes in place when the library
    refreshes while the panel is visible.
  - `docs/keyboard.md`: add a "Command-S (in Highlights)" row.
  - `docs/settings.md`: add the References row.
  - `docs/permissions.md`: Accessibility now also observes ⌘S while Highlights is
    frontmost; add the privacy sentence.
  - `docs/protocol.md` and `README.md`, wherever they list scan triggers or
    Accessibility uses.
- **bob-cli**: read `sase memory read docs.md -r "<why>"` first, then run
  `just docs-check`.
  - `docs/highlights-ref-sync.md` § "`-a, --attended`": "Bob Refs ⌘S passes it" becomes
    "Bob Refs ⌘S and Bob Mac Capture's scan after ⌘S in Highlights pass it".
  - The same file's § "Writer lock" example and § "Mac notification", where they name
    ⌘S.
  - `docs/ref.md`, around the sentence that says the Refs panel runs the scan "when
    Bryan presses ⌘S".
  - Commit these as `docs(ref): …`.

## Tests

Async tests must never await an untimed continuation or gate (bob-mac-capture
`AGENTS.md`).

- **RefsCore** (portable, run by `just portable-check`):
  - `HighlightsSaveKeyTests`: ⌘S matches. Uppercase or Caps Lock still matches. ⇧⌘S,
    ⌥⌘S, ⌃⌘S, plain `s`, and repeats do not.
  - `HighlightsSaveScanTriggerTests`:
    - no change fires at the fallback;
    - a change fires after the quiet period;
    - continuous changes fire at the cap;
    - repeated ⌘S coalesces into one fire;
    - changes before arming, hidden paths, and paths outside `lib/` and `xlib/` are
      ignored;
    - incomplete-batch flags count;
    - library changes with no keypress never fire (the no-loop guarantee);
    - `disarm()` cancels;
    - `poll` fires exactly once per cycle.
  - `HighlightsSaveScanScopeTests`: `lib/` and `xlib/` paths are inside; a path
    elsewhere is outside; nil is unknown; a symlinked vault root resolves; `..` escapes
    are outside.
- **BobMacCaptureTests** (macOS CI):
  - `RefsLibraryTests`:
    - `requestScan(queueIfBusy: true)` during a scan runs exactly one follow-up after
      `.finished`, and many requests still give one;
    - a request during a rename runs after the rename;
    - `scan()` still returns false when busy;
    - the fake fetcher records the argv `ref scan -a -w -f json`.
  - `RefsPanelModelTests`:
    - `.highlightsSave` shares the `.panelKey` preparation;
    - a snapshot refresh while the panel is visible re-hydrates the selected inspector
      with the fake `ref show`'s new highlights;
    - `inspectorGeneration` is unchanged and the old content stays until the new content
      publishes;
    - a hidden panel spawns no `ref show`;
    - the Copy card stays frozen;
    - revealed pages clamp when the feed shrinks.
  - Monitor wiring tests, in `RefsEntryPointsTests` or a new
    `HighlightsSaveKeyMonitorTests`:
    - the monitor is installed only while Highlights or the override app is frontmost,
      the setting is on, and the app is trusted;
    - it is removed on deactivate, terminate, setting off, or lost trust;
    - nothing ever prompts;
    - a fake key event plus a fake clock leads to exactly one
      `requestScan(.highlightsSave)`.
  - With `save_confirmation = notify` only: the presentation test from section 7.

## Verification

1. **bob-mac-capture.** Run `just format-lint` where available and
   `just portable-check`. If `swift` is missing on this host, run the check on athena.
   Then compile the `BobMacCapture` targets on the Mac (best effort), following
   `AGENTS.md`:
   - rsync a scratch copy without `.build` or `.git`;
   - run `Scripts/bundle.sh`, then `just typecheck-tests`;
   - delete the copy.

   Never use Bryan's checkout or `just install`. Run
   `sase memory read tailnet.md -r "<why>"` for SSH access. If the Mac is offline, say
   in the final report that the macOS targets are unverified. Push, and confirm the
   "macOS 26 SwiftPM" CI run passes: lint, build, test, bundle, and the install gates.

2. **bob-cli.** Run `just docs-check`.
3. **Manual acceptance for Bryan.** List these steps in the final report. On the Mac,
   after `just install`, run `bob ref doctor` and expect `pdf_write_coordination: on`.
   Then:
   - Open a `lib/` PDF in Highlights, add a highlight, and press ⌘S. Within about 10 s
     the reference note gains the highlight (`git -C ~/bob diff`), with no Highlights
     dialog.
   - Change the page-1 marker's `status`, press ⌘S, and make an unsaved annotation right
     away. The note's status follows. If bob rewrites the marker, the tab reloads in
     place and the unsaved annotation survives.
   - Press ⌘S, then immediately ⌘T or ⌘O to open Bob Refs on that reference. When the
     scan finishes, the inspector shows the new highlight without moving the selection
     or the scroll position.
   - Press ⌘S on a PDF outside the vault. No scan runs: the footer never shows "Scanning
     library…".
   - Turn the toggle off; ⌘S in Highlights no longer scans.

## Rejected alternatives

- **Carbon hotkey takeover of ⌘S** (as for ⌘O and ⌘T): it consumes the key, so
  Highlights would never save. Re-posting ⌘S, or pressing File › Save through
  Accessibility, would be the app scripting Highlights, which the coordinated-writes
  decision rejected for bob's write path.
- **Scanning on any file change in `lib/`** (FSEvents or `NSFilePresenter` alone): this
  also fires on bob's own coordinated writes, Highlights' reloads, and Git pulls of
  library PDFs, which invites loops and wasted scans. Arming on the keypress and
  settling on the file events gives the ⌘S intent plus correct timing.
- **A fixed delay after ⌘S**: large PDFs can save slower than any fixed delay, and the
  scan would then plan from stale sidecar bytes.
- **A single-PDF scoped scan**: the request is `bob ref scan -w`, and a full scan takes
  about 8 s, which is acceptable. A scoped scan would need a new bob-cli contract.
