---
tier: epic
title: Bob Refs Tabs filter — a Highlights tab switcher
goal: 'In Bob Mac Capture''s Bob Refs panel, ⇧⌘O pressed while Highlights is frontmost
  (or ⌘T inside the panel) shows a "Tabs" filter. It lists every document open in
  Highlights, matched to its reference note where one exists, with the current tab first
  and the previous tab pre-selected. ↵ switches to that existing tab without touching
  the vault or the open log. Bob Mac Capture installs with a stable signing identity, so
  the Accessibility grant the feature needs survives every `just install`.

  '
decisions:
  identity_setup:
    ask:
      How should Bryan create the stable "Bob Mac Capture Local" code-signing identity?
    choices:
      manual:
        README walks through Keychain Access › Certificate Assistant; no keychain script
      script:
        Also ship a one-time Scripts/create-signing-identity.sh for Bryan to run in
        Terminal
    default: manual
    why:
      The research verified the Certificate Assistant path; agents cannot test a
      keychain script
    answer: manual
  decision_record:
    ask:
      Add a decisions record saying Bob Mac Capture uses Accessibility only to read and
      select Highlights tabs?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: signing
    title: Stable signing identity and Accessibility status
    depends_on: []
    size: small
    description:
      'signing: `just install` signs with a stable self-signed "Bob Mac Capture Local"
      identity when it exists, and otherwise falls back to ad-hoc with a loud warning.
      `bundle.sh` keeps its ad-hoc default for agent scratch builds. The ad-hoc warning
      now mentions Accessibility. Settings › References shows "Highlights tabs: Allowed
      / Not allowed" through a small AccessibilityTrust helper. The README documents
      creating the identity.'
  - id: tabs-core
    title: RefsCore tab snapshot, matcher, ordering, and session
    depends_on: []
    size: medium
    description:
      'tabs-core: pure Foundation types in RefsCore. They cover the snapshot, the reader
      protocol and its results, the full-path → unique-file-name → collision → none
      matcher, the A5 ordering and pre-selection, the stale-target locator, tab recency,
      the scoped and searchable tabs listing, the "N tabs" count text, the
      `.inHighlights` section, and a main-actor session that owns generations and
      recency. Golden tests run with `just portable-check`.'
  - id: ax-reader
    title: Highlights-only Accessibility reader
    depends_on:
      - signing
      - tabs-core
    size: medium
    description:
      "ax-reader: `AXHighlightsTabsReader` in BobMacCapture implements the RefsCore
      reader protocol, using only public AX API against Highlights' process. It never
      launches Highlights, never prompts, and works off the main actor with messaging
      timeouts and a time budget. It reads window AXDocument and the native tab bar, and
      presses one verified tab button for select. Tab names stay in memory only. An
      optional read-only probe runs on the Mac."
  - id: tabs-model
    title: Panel model Tabs filter, switching, and states
    depends_on:
      - tabs-core
    size: medium
    description:
      "tabs-model: RefsPanelModel gains the presentation source, the `tabsFilter` flag,
      the tabs state, and the tab-row map. It handles ⌘T and ⌘R re-snapshots, the Esc/⌫
      ladder, and a ↵ held until the first snapshot lands. Matched rows switch through
      `openInHighlights` without `recordOpen`; collision and unmatched rows switch
      through the session's select, with a stale-tab banner. Synthetic rows survive
      library publications. Preview hooks and model tests use a fake reader."
  - id: tabs-views
    title: Tabs rendering, keys, and design tests
    depends_on:
      - tabs-model
    size: medium
    description:
      'tabs-views: the Tabs token and "9 tabs" count in the search bar, the In
      Highlights header, the "Current · p. N/M" pill, dim file-name rows for collision
      and unmatched tabs, minimal inspector cards, and cards for every Tabs state. Also
      the "↵ Switch" footer, VoiceOver labels, and router bindings for ⌘T and the ⌫
      context, plus design fixtures and router tests.'
  - id: tabs-entry
    title: ⇧⌘O entry point, live wiring, settings, and docs
    depends_on:
      - signing
      - ax-reader
      - tabs-views
    size: medium
    description:
      "tabs-entry: adds `HotKeyAction.refsHighlightsTabs = 4` with a configurable
      ⇧⌘O/⌘E/Off key that registers only while Highlights is frontmost, and splits the
      hotkey dispatch so ⌘O and ⌃⇧⌘O are unchanged. Adds coordinator toggle-to-Tabs
      semantics, wires the live AX reader, session, and ad-hoc flag, and samples recency
      when Highlights deactivates. Adds the Settings key picker and the README Tabs,
      keyboard, privacy, and troubleshooting docs, plus the optional decision record."
proposed_by: bbugyi200.apollo.7w
decided_by: auto
create_time: 2026-10-15 17:50:47
status: wip
---

- **PROMPT:**
  [prompts/202610/highlights_tabs_switcher.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/highlights_tabs_switcher.md)

# Plan: Bob Refs Tabs filter — a Highlights tab switcher

## Context

Highlights labels its tabs with file stems; one tab today is literally
`442a6bf555914893e9891c11519de94f.pdf`. Its own switching is ⌃Tab cycling or a thumbnail
grid. Bob Refs already knows the title, kind, reading state, and annotations of every
library PDF. This epic adds a **Tabs** filter to Bob Refs that lists what Highlights has
open, so the panel works as a tab switcher.

The design follows the research report
`research:202610/highlights_tab_switcher/highlights_tab_switcher.md`. Phase workers
should read it with `sase artifact read` (if the research sidecar is not cloned, the
explicit ref is `file:explicit:9ba3692a69678bff4d21e72b`). Bryan accepted every
requirement adjustment in it, **A1–A11**:

- **A1:** show every open Highlights document. Unmatched files appear as dim rows titled
  by file name, marked "Not in your library".
- **A2:** Tabs is a filter flag that combines with the kind scopes. It shows as a token
  in the search bar, and ⌘1–⌘5 still narrow inside it.
- **A3:** ⇧⌘O while Highlights is frontmost opens Tabs (configurable: ⇧⌘O, ⌘E, or Off).
  ⌘T toggles Tabs inside the panel. ⌘O and ⌃⇧⌘O are unchanged. Nothing persists between
  shows.
- **A4:** ↵ switches to the existing tab. It never jumps pages, and never changes
  reading state, lanes, or the vault.
- **A5:** the current tab comes first ("Current · p. 5/25"), then the most recently
  active tabs, then tab-bar order, with unmatched files last. If Highlights is
  frontmost, row 2 is pre-selected; otherwise row 1 is.
- **A6:** the feature is named "Tabs". The count reads "9 tabs" or "2 of 9 tabs", never
  "open". The section header is "In Highlights".
- **A7:** stable signing comes first.
- **A8:** the permission is requested lazily, through an in-panel card.
- **A9:** a snapshot is taken on show, on ⌘T, and on ⌘R, with no polling. A stale target
  is detected at ↵ time.
- **A10:** switches never append to `refs-open-log.json`.
- **A11:** non-goals: no closing, reordering, or moving tabs; no Screen Recording, Apple
  Events, `lsof`/`libproc`, or private API; no bob-cli change; no persisted Highlights
  state; no page numbers for background tabs.

The research's "later, only if wanted" items are out of scope: the open-tab glyph in
every scope (A12), an `AXObserver` for exact recency, kept window handles, ⌘W to close,
and `refs-rank --tabs`.

**How the research's Phase 1 spikes are handled.** The research lists about an hour of
Mac spikes (AXPress switching, multiple windows, Spaces, kept handles, notifications).
They need a test Highlights session, and agents must not change Bryan's live one. So
this plan does not create a spike phase. The design does not depend on their outcomes:

- Matched rows switch through `open -a`, which researchers observed switching tabs and
  keeping the page (E10).
- `AXPress` serves only collision and unmatched rows (today 1 and 0 rows). Its result is
  verified by a re-read, and a failure becomes a banner.
- The reader iterates every `AXWindows` element, so extra windows simply appear.
- `ax-reader` runs an optional read-only probe.
- Behaviours that change state (AXPress, Spaces, full screen) move to Bryan's smoke test
  at the end.

## Where the work lives

All code lives in the **`bob-mac-capture` linked repo**. Phase workers must open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`), read its `AGENTS.md`, and
commit there. bob-cli changes only if `decision_record` is accepted. The thin-client
rule (`decisions:mac-capture-is-a-thin-client`) holds: which PDFs Highlights has open is
macOS UI state that `bob` cannot see, and the app only filters and ranks the
`bob ref list` snapshot against it.

## Rules for every phase

- **Compile everything you change** (bob-mac-capture `AGENTS.md`). Run
  `just portable-check` on a host with Swift; this host has none, so use `athena`. Any
  phase that touches `Sources/BobMacCapture*` or `Tests/BobMacCapture*` must also
  compile a scratch copy on the Mac (`ssh mac`, best effort):
  1. rsync the repo without `.build` or `.git` into a temp directory;
  2. run `Scripts/bundle.sh` (ad-hoc default) and `just typecheck-tests`;
  3. delete the copy.

  Never use Bryan's checkout on the Mac, and never run `just install` there. If the Mac
  is offline, say in the final report that the macOS targets are unverified. CI runs the
  XCTest suites.

- **Follow the lint order.** Run `swift-format lint` first. A parse error in it skips
  the later CI steps, so check them after a red lint step.
- **Keep AX use contained.** Every `AXUIElement*` call lives in the single reader file
  from `ax-reader`. `AXIsProcessTrusted*` calls live only in the `AccessibilityTrust`
  helper from `signing`. No other file calls Accessibility APIs. Read only window and
  tab attributes, perform only `AXPress` on a tab button and `AXRaise` on its window,
  and target only Highlights' process.
- **Keep tab names private.** Tab names and paths stay in memory only. They never go
  into logs, signposts, diagnostics, `UserDefaults`, or files, and never into commits,
  bead notes, or test fixtures copied from Bryan's live session. Fixtures use invented
  or existing fixture-library names.
- **Match the house style.** Match the surrounding Swift: doc comments on every type and
  public member, injected protocols with fakes in tests (like `RefsOpening`), and pure
  logic in `RefsCore`.

## Shared contracts (owned by `tabs-core`)

`ax-reader` and `tabs-model` run in parallel against these RefsCore types. The names are
binding. `tabs-core` may add members or helpers, but must not rename these or change
their meaning.

- **`HighlightsTabsSnapshot`** (`Equatable`, `Sendable`): `windows: [Window]` in
  `AXWindows` order, plus `highlightsIsFrontmost: Bool`.
  - `Window` holds `isMain: Bool`, `pageCaption: String?` (already formatted as "p.
    5/25"), and `tabs: [Tab]` in tab-bar order.
  - `Tab` holds `fileName: String` (the AX tab title, as shown), `documentPath: String?`
    (an absolute, percent-decoded filesystem path, known only for each window's visible
    tab), and `isSelected: Bool`.
  - A window without a tab bar is one `Tab` built from its `AXDocument`.
- **`HighlightsWindowTitle.pageCaption(fromTitle:) -> String?`** parses a window title
  such as `name.pdf – Page 5 of 25` (en dash or hyphen) into "p. 5/25", or returns nil.
- **`HighlightsTabsReadResult`**: `.notRunning`, `.notTrusted`, `.failed(String)`, or
  `.snapshot(HighlightsTabsSnapshot)`. The `String` is a bounded, name-free reason.
- **`HighlightsTabTarget`** names a tab to select: `fileName` plus `windowOrdinal`.
- **`HighlightsTabSelectResult`**: `.selected`, `.notFound`, `.notRunning`,
  `.notTrusted`, or `.failed(String)`.
- **`HighlightsTabsReading`** (`Sendable` protocol):
  - `isTrusted() -> Bool` never prompts.
  - `requestTrust()` runs the prompting trust check and is called only from the
    permission card.
  - `read() async -> HighlightsTabsReadResult`.
  - `select(_ target: HighlightsTabTarget) async -> HighlightsTabSelectResult`.

  `UnavailableHighlightsTabsReader` is the placeholder until `tabs-entry` wires the live
  reader. Its `read()` and `select` return
  `.failed("Highlights tabs aren't available in this build")`, and `isTrusted()` returns
  false.

- **`HighlightsTabLocator.locate(_ target:in:) -> (window: Int, tab: Int)?`** is the
  stale-target rule:
  1. Return the single tab titled `fileName` in window `windowOrdinal`.
  2. Otherwise, return the single such tab across all windows.
  3. Otherwise (none, or ambiguous), return nil.

  It never returns "whatever tab now sits at the old index".

- **`RefsOpenTabMatch`**: `.path`, `.fileName`, `.collision`, or `.none`.
- **`RefsOpenTab`** (`Equatable`, `Sendable`) carries `rowID`, `itemID: String?`,
  `candidateIDs: [String]` (collision only), `fileName`, `windowOrdinal`, `tabOrdinal`,
  `isCurrent`, `pageCaption: String?`, `documentPath: String?`, and `match`, plus a
  `target: HighlightsTabTarget` accessor.
  - `rowID` is the note path (`item.id`) for `.path` and `.fileName` rows, so every
    existing row, inspector, copy, and ⌘K path works unchanged.
  - Collision and unmatched rows use the reserved prefix `highlights-tab:<fileName>`,
    with `#2`, `#3` and so on appended for repeats. These ids never collide with note
    paths.
  - A tab's index is never its identity.
- **`HighlightsTabMatcher.match(_ snapshot:items:vaultRoot: URL) -> [RefsOpenTab]`**
  applies these rules in order:
  1. **Full path.** Standardize `documentPath`, resolve symlinks (on both it and the
     vault root), strip the vault root, and compare the result with `item.pdfPath`.
  2. **Unique file name.** Compare the tab title, accepting `stem.pdf` and a bare `stem`
     (hidden extension) and stripping any trailing page suffix, against the last path
     component of every `pdfPath`. Exactly one hit is a match.
  3. **Collision.** Two or more hits make a collision row. Never guess from the open
     log.
  4. **None.** No hit makes an unmatched row. A path outside the vault falls through to
     rules 2–4.

  Dedupe to one row per library item, and one per unmatched file name. When the same
  document is open in several windows, keep the current occurrence, else the first in
  window order. `isCurrent` marks the selected tab of the `isMain` window (or of the
  first window when none reports main).

- **`RefsTabRecency`** (value type) maps file name → last-active `Date`, with
  `touch(fileName:at:)` and `lastActive(_:)`. It lives in memory only.
- **`RefsSectionKind.inHighlights`** has the title "In Highlights".
- **`RefsTabsRanker.listing(tabs:items:query:scope:signals:recency:highlightsIsFrontmost:) -> RefsTabsListing`**
  returns `listing: RefsListing`, `tabs: [String: RefsOpenTab]` (keyed by row id), and
  `preselectedID: String?`.
  - **Scope.** Matched rows filter by `item.kind`. Collision rows show when any
    candidate passes. Unmatched rows show only under All.
  - **Browse.** There is one `.inHighlights` section in A5 order:
    1. the current row;
    2. matched and collision rows, by recency (rows with a recency first, newest first),
       then `(windowOrdinal, tabOrdinal)`;
    3. unmatched rows in `(windowOrdinal, tabOrdinal)`.

    `preselectedID` is row 2 when `highlightsIsFrontmost` and at least 2 rows exist,
    else row 1.

  - **Search.** A typed query runs the existing tiered ranker (`RefsRanker.searchRows`
    tiers and tie-breaks, unchanged) over the matched items. Collision and unmatched
    rows compete as title-only candidates (title = file stem, no lane, frecency, or
    recency signal). Synthetic `RefItem`s with kind `.other` and state `.unknown` are
    fine internally. The result is one nil-kind section, and `preselectedID` is the
    first ranked row. The ranker's single-entry prepared cache will key on the tab
    subset; that is fine for about 10 rows, but never rank both sets per keystroke.
  - `totalCount` is the number of scoped tab rows.

- **`RefsTabsCount.text(shown:total:isSearch:)`** returns "1 tab", "9 tabs", or "2 of 9
  tabs".
- **`HighlightsTabsSession`** (`@MainActor` final class in RefsCore, constructed as
  `init(reader:now:)`):
  - `refresh() async -> HighlightsTabsReadResult?` bumps a generation counter and
    returns nil when a newer refresh started meanwhile, so an older result never
    replaces a newer one.
  - It keeps `lastSnapshot` and a `RefsTabRecency`.
  - Every applied snapshot touches its current tab.
  - `sampleCurrentTab() async` reads, and touches the current tab, only when
    `reader.isTrusted()`. It never prompts.
  - `recordSwitch(fileName:)` touches recency now.
  - `select(_ tab:) async` forwards `tab.target`, and on `.selected` records the switch.
  - It exposes `reader` for `isTrusted()` and `requestTrust()`.

## Phase `signing`: Stable signing identity and Accessibility status

**Why.** Bob Mac Capture is ad-hoc signed (`identity="-"`). macOS ties an ad-hoc app's
Accessibility grant to its exact `cdhash`, so every `just install` silently revokes the
grant while System Settings still shows it on (research E13, E14). A stable self-signed
identity yields a `certificate leaf` designated requirement that survives rebuilds.

Work:

1. **Install-time identity resolution.** In `Scripts/install.sh`:
   1. An explicit `--identity` or `CODESIGN_IDENTITY` wins, including `-`.
   2. Otherwise, if `/usr/bin/security find-identity -p codesigning` lists
      `"Bob Mac Capture Local"`, sign with it. Search without `-v`, so an untrusted
      self-signed identity still counts.
   3. Otherwise, fall back to `-` and print a loud stderr warning. The warning says the
      Accessibility and notification grants will reset on this install, and points to
      README › Signing.

   Then change the `justfile` `install` recipe's default identity to empty, meaning
   "resolve". **`Scripts/bundle.sh` and `just bundle` keep their ad-hoc `-` default.**
   Agent scratch builds run `bundle.sh` over SSH, where the login keychain is locked, so
   they must never pick up Bryan's identity. CI passes `--identity "-"` explicitly and
   must stay green. Keep the resolution in a small shell function that can be exercised
   on its own, and run `shellcheck` when it is available.

2. **Inspector text.** Update the `BundleSigningInspector` ad-hoc diagnostic so it says
   that notification **and Accessibility** grants reset on each reinstall, and names
   README › Signing. Update `BundleSigningInspectorTests`. `.signed(identity:)` already
   reports "Signed: Bob Mac Capture Local".
3. **Trust helper.** Add `AccessibilityTrust` in BobMacCapture: `isTrusted()` wraps
   `AXIsProcessTrusted()` and never prompts, and `requestPrompt()` wraps
   `AXIsProcessTrustedWithOptions` with `kAXTrustedCheckOptionPrompt`. It is the only
   home for trust calls. Nothing calls it at launch.
4. **Settings row.** Settings › References gains a "Highlights tabs" row next to the
   resolved Highlights app row, reading "Allowed" or "Not allowed" from `isTrusted()`.
   It refreshes when Settings appears and when the app becomes active. On ad-hoc builds
   it adds the caption "Grants reset when an ad-hoc build is reinstalled." The row lets
   Bryan verify the signing fix before Tabs exists.
5. **README.** Add a "Signing" subsection under Development, and fix the Development
   paragraph that says `just install` signs ad-hoc. It covers:
   - creating the identity (see the decision below);
   - that `just install` now picks the identity up automatically;
   - the first-use "codesign wants to access key" prompt (choose Always Allow);
   - running `just install` from a GUI Terminal session, because the keychain is locked
     over SSH;
   - verifying with `codesign -d -r- "~/Applications/Bob Mac Capture.app"`, which should
     show `certificate leaf = H"…"` rather than a `cdhash`;
   - `tccutil reset Accessibility org.bobs.bob-mac-capture` to clear a stale ad-hoc
     grant.

   Update the "Updating, Reinstalling, and Rollback" text on identity changes to mention
   Accessibility.

> [!decision] identity_setup = manual The README gives step-by-step Keychain Access ›
> Certificate Assistant instructions for "Bob Mac Capture Local": identity type
> Self-Signed Root, certificate type Code Signing, a long validity period, stored in the
> login keychain. No script touches the keychain.

> [!decision] identity_setup = script Also add `Scripts/create-signing-identity.sh`,
> which Bryan runs once in a GUI Terminal. It refuses when the identity already exists.
> It generates a key and a certificate with `openssl` (code-signing EKU, `CA:false`,
> ten-year validity, CN `Bob Mac Capture Local`), exports a PKCS#12 with `-legacy` when
> OpenSSL 3 needs it, imports the result into the login keychain with
> `-T /usr/bin/codesign`, and deletes its temp files. It must never run in an agent
> session. The README documents it first, with the GUI steps as a fallback.

Tests: inspector text tests, plus any Settings-row logic you factor into a testable
function. Verify `install.sh` on the Mac read-only, for example with a dry-run flag or
by running the resolver function. Today the keychain holds no identity, so this
exercises the fallback warning path. Do not install.

## Phase `tabs-core`: RefsCore tab snapshot, matcher, ordering, and session

Implement every type in [Shared contracts](#shared-contracts-owned-by-tabs-core) in
`Sources/RefsCore`, for example `RefsHighlightsTabs.swift` plus `RefsTabsRanking.swift`.
They use only Foundation and CaptureCore.

- **Section kind.** Adding `RefsSectionKind.inHighlights` requires cases in
  `RefsCaption` and `RefsExplanation`. Matched rows caption like Library rows. `whyHere`
  reads "Current tab in Highlights" or "Open in Highlights".
- **Mac check.** Grep `Sources/BobMacCapture` for exhaustive switches over the enum.
  None exist today; if any appear, the Mac typecheck rule applies.
- **Fixtures.** Snapshot fixtures may be inline Swift values, or JSON under
  `Tests/Fixtures/` if you make the snapshot `Codable` (decoding only for tests). Use
  fixture-library names, not Bryan's live tabs.

Golden tests in `Tests/RefsCoreTests/` (for example `RefsHighlightsTabsTests.swift`) run
with `just portable-check` on athena:

- a 9-tab, one-window snapshot where every tab maps to exactly one fixture item: the
  visible tab by path, the rest by unique file name;
- a `harness_engineering.pdf` collision between `lib/papers/` and `lib/blogs/` items,
  giving one row with both candidates;
- an unmatched `~/Downloads/cv.pdf`;
- a hidden-extension title (`stem`), and a title with a trailing page suffix;
- a single window without a tab bar;
- two windows: the main window's selected tab is current, and the other window's visible
  tab matches by path;
- an empty snapshot, and a document open in two windows (deduped);
- A5 order and pre-selection, both frontmost and not, plus recency ordering and the
  tab-bar tie-break;
- scope filtering: unmatched rows only under All, collision rows under a candidate's
  kind;
- search tiers over tab rows, including an unmatched title, and the count text;
- page-caption parsing (en dash, hyphen, no page);
- every locator rule: same window, moved to another window, closed, and now ambiguous
  (nil), proving a moved index is never pressed;
- session generations (an older result is dropped), recency touches, and
  `sampleCurrentTab` doing nothing when untrusted. Use a fake reader in RefsCoreTests.

## Phase `ax-reader`: Highlights-only Accessibility reader

Add `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`, the only file that calls
`AXUIElement*`. It implements `HighlightsTabsReading`.

1. **Process.** `init(bundleIdentifier: @escaping @Sendable () -> String)` resolves the
   Highlights process through
   `NSRunningApplication.runningApplications(withBundleIdentifier:)`. `tabs-entry`
   passes the Settings override's bundle id or `HighlightsLocator.bundleIdentifier`. The
   reader **never launches Highlights**: no process gives `.notRunning`.
2. **Trust.** Check trust through `AccessibilityTrust.isTrusted()`, which never prompts;
   untrusted gives `.notTrusted`. `requestTrust()` calls
   `AccessibilityTrust.requestPrompt()`.
3. **Queue and timeouts.** `AXUIElementCreateApplication(pid)` with
   `AXUIElementSetMessagingTimeout` of about 0.4 s, and an overall budget of about 1 s
   per call. All AX objects live on one private serial queue, bridged to `async` with
   continuations, so nothing runs on the main actor.
4. **Read.** For each `kAXWindowsAttribute` element:
   - read `kAXMainAttribute`, `kAXDocumentAttribute` (decode the file URL into a path),
     and `kAXTitleAttribute`, passing the title through
     `HighlightsWindowTitle.pageCaption`;
   - search children to a bounded depth (3) for `kAXTabGroupRole`, then read its
     `AXTabs`: each tab's title, and `AXValue == 1` for the selected tab;
   - give the window's `documentPath` to its selected tab;
   - treat a window with no tab group as one selected tab named from its document.

   Never walk into PDF content nodes. **Never cycle tabs to learn their paths.** Set
   `highlightsIsFrontmost` from the running application's `isActive`.

5. **Select.**
   1. Do a fresh read on the queue.
   2. Run `HighlightsTabLocator.locate` against it; nil gives `.notFound`.
   3. Re-check that title, then `AXPress` the tab button.
   4. Activate Highlights and `AXRaise` the window.
   5. Re-read that window's tabs and require the target to be selected. If it is not,
      return `.failed("Highlights didn't switch tabs")`. Log the mismatch by event name
      only, and never retry in a loop.
6. **Errors.** Map AX errors (API disabled, cannot complete or timeout, invalid element)
   to `.failed` reasons that contain no names.

Tests: Bob Mac Capture's tests can only type-check on the Mac. Add a small
`BobMacCaptureTests` case that CI runs: a reader built with an unknown bundle id returns
`.notRunning` for both `read()` and `select`, and never launches anything.

**Optional read-only probe (best effort).** In the Mac scratch copy, run a throwaway
script that exercises the same read path against Bryan's running Highlights:

- read only: never press, open, close, or activate anything;
- confirm the attribute shapes (window count, tab-group presence, selected flag, that a
  document path resolves);
- record only counts and shapes in your final report, never tab names or paths;
- delete the script with the copy.

If the probe cannot run (Mac offline, or SSH lacks AX trust), say so.

## Phase `tabs-model`: Panel model Tabs filter, switching, and states

Change `RefsPanelModel` and `RefsPanelController`. Keep every existing library-mode
behaviour unchanged.

- **Init.** Add `tabs: HighlightsTabsSession` (default: a session over
  `UnavailableHighlightsTabsReader`) and `isAdHocSigned: () -> Bool` (default false) to
  `init`. `tabs-entry` wires the live values, and `AppDelegate` keeps compiling
  unchanged here.
- **Presentation source.** Add `RefsPresentationSource { library, highlightsTabs }`. Use
  `prepareForPresentation(source: = .library)` and
  `RefsPanelController.show(source: = .library)`. Preparing resets `tabsFilter` to
  `source == .highlightsTabs`, so nothing persists between shows. `represent()` keeps
  the current state.
- **State.** Publish `tabsFilter: Bool`, a `tabsState` enum, and the current
  `RefsTabsListing` tab map. The states are:
  - `.off`;
  - `.loading`;
  - `.loaded`;
  - `.needsPermission(adHoc: Bool)`;
  - `.notRunning`;
  - `.noDocuments`;
  - `.failed(message:, hasStaleRows:)`. The last snapshot stays listed and dimmed when
    one exists.

  Entering Tabs, through presentation, ⌘T, or ⌘R while in Tabs, checks `isTrusted()`
  first and never prompts, then calls `session.refresh()`. A nil (superseded) result is
  ignored.

- **Commands.** Add these to `RefsCommand`: `.toggleTabs` (⌘T), `.setTabsFilter(Bool)`
  (for the coordinator), and `.requestTabsPermission` (the card button:
  `requestTrust()`, then a re-check).
  - **⌘T** keeps the query and scope and re-ranks.
  - **⌘R in Tabs** refreshes the library and takes a new tab snapshot.
  - **Esc ladder:** copy card → banner → query → **Tabs filter** → scope → hide.
  - **⌫ on an empty query** removes the Tabs filter before the scope, matching A2's
    "removes it first".
- **Matching.** Run `HighlightsTabMatcher.match` against `library.items` when each
  applied snapshot lands. Expose the vault root from `RefsLibrary` (for example a
  `vaultRootURL` accessor over its existing closure). Re-match only on a new snapshot,
  never on a library publication.
- **Listing.** In Tabs mode, build the listing with `RefsTabsRanker`, not
  `RefsRanker.listing`, in each of these places: `rerankSelectingFirst`,
  `freshListingKeepingSelection`, `buildFreshListingForScan`, and `installForPreviews`.
  - Pre-selection uses `preselectedID`.
  - A new tab snapshot may re-rank, because the row set changed; it keeps `selectedID`
    when that row still exists.
  - Late library publications stay content-only. **`libraryDidPublish` must count the
    current synthetic `highlights-tab:` row ids as available**, or every unmatched row
    would turn "unavailable".
  - A matched row whose item vanished becomes unavailable as today.
- **Row content.** Add `openTab: RefsOpenTab?` to `RefsRowContent`. Collision and
  unmatched rows get `item: nil`, the file name as title, the caption "Not in your
  library" or "Matches N notes" (with candidate titles available), and
  `isUnavailable: false`.
- **Held ↵.** If ↵ (or Tab) arrives while the first snapshot of a presentation is
  `.loading` and no rows exist, hold it. When the snapshot lands, act on the
  pre-selected row. A query edit, or any non-loaded outcome, drops the hold.
- **Switching** (A4, A10):
  - **Matched rows** (`.path`, `.fileName`):
    `openInHighlights(item, recordsOpen: false)`. Thread the flag through
    `finishHighlightsOpen`, so success skips `library.recordOpen`. On success, call
    `session.recordSwitch(fileName:)`. Never add `#page=` or any `highlights://` deep
    link. A closed tab simply reopens.
  - **Collision and unmatched rows:** hide the panel, then `await session.select(tab)`.
    Map the result:
    - `.notFound` → re-present through `panelRepresenter` with the warning banner "That
      tab was closed in Highlights.";
    - `.notRunning` → the same, with "Highlights isn't running.";
    - `.notTrusted` → re-present in `.needsPermission`;
    - `.failed` → an error banner "Couldn't switch to that tab in Highlights."
  - ↵, Tab, and double-click (`.activate`) all switch.
  - Default-app opens from ⌘K still record, as before, because they are opens, not
    switches.
- **Note actions on collision and unmatched rows.** ⌘↵, ⌥↵, ⌘Y, and ⌘K consume the key
  and show a warning banner instead of passing through:
  - a collision: "“name.pdf” matches N notes — press ⌘T and search to act on one.";
  - unmatched: "“name.pdf” isn't in your library."

  On matched rows they are unchanged.

- **Preview hook.** Add an `installTabsForPreviews(...)` hook (snapshot, state,
  frontmost) so `tabs-views` can render every state without a reader.

Tests in `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, or a new
`RefsPanelTabsTests.swift`, with a fake reader. These type-check on the Mac, and CI runs
them:

- each source sets the filter, and a library show resets it;
- ⌘T keeps query and scope;
- ⌘R re-snapshots and keeps the selection;
- the Esc ladder order, and ⌫ removing Tabs before the scope;
- a matched switch dispatches `openInHighlights` and leaves the open log and
  `signals.opens` unchanged;
- an unmatched switch calls `select` with the right target, and `.notFound` shows the
  banner;
- the held ↵ fires once the snapshot lands, and is dropped on failure;
- a superseded refresh is ignored;
- `.notTrusted` maps to `needsPermission(adHoc:)`, with ad-hoc taken from the injected
  flag;
- a library publication never marks synthetic rows unavailable;
- note actions on synthetic rows show their banners.

## Phase `tabs-views`: Tabs rendering, keys, and design tests

SwiftUI and router changes under `Sources/BobMacCapture/Refs/`. They render only what
`tabs-model` publishes.

- **Search bar.** Draw a "Tabs" token before the scope token, styled like it with an SF
  Symbol such as `square.on.square`. Clicking it removes the filter. The count uses
  `RefsTabsCount` ("9 tabs", "2 of 9 tabs") with matching accessibility labels and
  debounced announcements ("2 of 9 tabs"). Library mode keeps "N open · M".
- **List.** The `.inHighlights` header reads "IN HIGHLIGHTS" plus the count. The current
  row shows a "Current · p. 5/25" pill, or "Current" when there is no caption, in the
  trailing pill slot.
- **Collision and unmatched rows.** These rows are dimmed (not the unavailable 0.45
  style; pick a distinct, legible treatment) and titled by file name. Their caption
  reads "Not in your library" or "Matches 2 notes: A · B", with a neutral document glyph
  in place of the kind tile and state glyph.
- **VoiceOver.** Rows append ", open in Highlights". The current row appends ", current
  tab, page 5 of 25".
- **Inspector.** Collision and unmatched rows get a minimal card. It shows the file name
  and "Not in your library", or "Matches N notes" with the candidate titles and paths
  (plus the document path when known), and the hint "↵ switches to this tab". Matched
  rows keep the full inspector.
- **State cards** in the content well, styled like `RefsLoadFailedView` and
  `RefsEmptyStateView`:
  - **Needs permission:** "Bob Refs needs Accessibility access to see Highlights' tabs."
    Buttons: **Open Accessibility Settings** (`.requestTabsPermission`) and **Show
    library (⌘T)**. On ad-hoc builds it adds: "Grants reset when an ad-hoc build is
    reinstalled. Re-add the app, or run
    `tccutil reset Accessibility org.bobs.bob-mac-capture`."
  - **Not running:** "Highlights isn't running." with "⌘T shows the library".
  - **No documents:** "No documents are open in Highlights."
  - **Failed:** the stale rows dimmed, with a banner-style notice "Couldn't read
    Highlights' tabs · ⌘R to retry". With no stale rows, show a card with the same text.
  - **Loading:** show `RefsSkeletonList` only when the read runs longer than about 150
    ms, so a fast read never flashes a skeleton.
  - **Search with no match:** "No tabs match “q”".
  - **No library snapshot yet:** Tabs mode does not wait for one. The tab states and
    rows render, with every tab unmatched until a snapshot exists, instead of the
    library skeleton or load-failed card.
- **Footer.** In Tabs mode the first hint is "↵ Switch", and the variants mention "⌘T
  Library". Library mode's widest variant may add "⌘T Tabs". Keep the `ViewThatFits`
  fallbacks.
- **Router.** `RefsKeyRouter` maps ⌘T (key code 17) to `.toggleTabs`, unless
  IME-composing. Add `tabsFilterOn` to `RefsKeyContext`, so ⌫ on an empty query returns
  `.deleteBackwardOnEmpty` when Tabs is on, even with scope All. `RefsPanelController`
  fills the new context field. Update the README-keyboard comment in the router if it
  enumerates keys.

Tests:

- `RefsKeyRouterTests`: ⌘T, IME pass-through, and ⌫ with Tabs on and scope All.
- `RefsFooterTests`: the hint variants.
- Design tests in `RefsPanelDesignTests` style (via `installTabsForPreviews` and
  `RenderFixtureWriter`): a loaded 9-row list with the current pill and a pre-selected
  row 2, a collision row selected with its inspector card, an unmatched row, the
  permission card in both ad-hoc and signed variants, not running, no documents, failed
  with stale rows, and search "2 of 9 tabs".

## Phase `tabs-entry`: ⇧⌘O entry point, live wiring, settings, and docs

- **Hotkeys.**
  - Add `HotKeyAction.refsHighlightsTabs = 4`.
  - Add the presets `HotKeyConfiguration.highlightsShiftCommandO` (`kVK_ANSI_O`,
    `cmdKey | shiftKey`) and `.highlightsCommandE` (`kVK_ANSI_E`, `cmdKey`).
  - Add `RefsHighlightsTabsKey { shiftCmdO (default), cmdE, off }`, mirroring
    `RefsHighlightsOpenKey`, and persist it in `AppSettings` (`refsHighlightsTabsKey`).
- **Takeover.**
  - `HighlightsTakeover` registers both `.refsHighlightsOpen` and `.refsHighlightsTabs`,
    each only while Highlights (or the override app) is frontmost and its key is not
    Off. `stop()` unregisters both.
  - `sync(key:appPath:)` grows to carry the tabs key, and Settings sinks re-sync live
    with the values they receive.
  - Add `onHighlightsDeactivated`. It fires on `didDeactivateApplicationNotification`
    for the expected bundle id only, and calls `session.sampleCurrentTab()`. That method
    no-ops unless trusted and never prompts.
- **Dispatch.**
  - Split `case .refs, .refsHighlightsOpen:` in `AppDelegate`. Those two keep
    `toggleRefs()`, which shows in library mode, so ⌘O and ⌃⇧⌘O are unchanged.
  - `.refsHighlightsTabs` calls a new `BobPanelCoordinator.toggleRefsTabs()`:
    - hidden → `showRefs(source: .highlightsTabs)`, which still retains a visible
      Capture draft;
    - visible but not in Tabs → `perform(.setTabsFilter(true))`;
    - visible in Tabs → hide.

  The coordinator gains the closures it needs (an `isRefsInTabs` query and a
  source-aware present), with defaults so existing call sites and tests keep compiling.

- **Live wiring.** In `configureRefsStack`:
  - build `AXHighlightsTabsReader` with a bundle-id closure (the override app's id, else
    `HighlightsLocator.bundleIdentifier`) and a `HighlightsTabsSession`;
  - pass both, plus `isAdHocSigned` from `BundleSigningInspector`, into
    `RefsPanelModel`. Compute the signing state before the Refs stack, or lazily, rather
    than after it as today.

  No AX call or prompt may happen at launch.

- **Settings › References.** Add the picker "In Highlights, show open tabs with" (⇧⌘O /
  ⌘E / Off) beside the existing open-key picker. The "Highlights tabs" status row
  already exists from `signing`.
- **README.**
  - Add a "Tabs" subsection under Bob Refs covering:
    - what it shows (A1);
    - how to enter it (⇧⌘O, ⌘T);
    - order and pre-selection (A5);
    - that switching never records an open (A10);
    - the states;
    - the limits: background tabs match by file name only, and other Spaces and
      full-screen windows are untested.
  - Update "Opening Bob Refs" and the Keyboard table (⌘T, ⇧⌘O).
  - Update Privacy:
    - Accessibility is used only against Highlights, to read window and tab attributes
      and press tab buttons;
    - tab names stay in memory;
    - the new `refsHighlightsTabsKey` defaults key exists.
  - Update Troubleshooting: the permission card, stale ad-hoc grants, and
    `tccutil reset`.
  - Update Sorting with one sentence on the Tabs order.

> [!decision] decision_record Use `/sase_memory_write`, which this accepted decision
> authorizes. Add one strand to the `decisions` web in the bob-cli repo
> (`sase/memory/decisions/`), following the web descriptor's record format: Applies to,
> Claim, Why, Cost, Reopens when, and evidence.
>
> - **Claim.** Bob Mac Capture reads Highlights' UI through Accessibility only, inside
>   the one reader type. It reads only window and tab attributes, performs only
>   `AXPress` on a tab button and `AXRaise` on its window, and never edits documents,
>   cycles tabs, uses Screen Recording, Apple Events, or private API, or persists tab
>   names.
> - **Evidence.** The research report, E2/E3 (`lsof`/`libproc` see no PDFs), and the
>   landed commits.
>
> Then run `sase memory init` and commit the bob-cli memory change.

Tests:

- `HotKeyRegistryTests`: id 4 is distinct under the `'BOBC'` signature, and the presets
  do not collide with Capture or Refs.
- `RefsEntryPointsTests`:
  - the takeover registers both keys only while Highlights is frontmost, and an Off tabs
    key never registers;
  - changing the key swaps the registration live;
  - termination and `stop()` unregister both;
  - deactivation of Highlights calls the sampler, but deactivation of other apps does
    not;
  - the coordinator's three `toggleRefsTabs` cases, including Capture-draft retention;
  - the settings default, persistence, and round trip.

## Manual verification (Bryan)

These need the GUI Mac and cannot be done by agents.

**After `signing` lands:**

1. Create the "Bob Mac Capture Local" identity as the README describes.
2. Run `just install` from Terminal and choose Always Allow at the keychain prompt.
3. Check that `codesign -d -r- "$HOME/Applications/Bob Mac Capture.app"` shows
   `certificate leaf`.
4. Add the app under System Settings › Privacy & Security › Accessibility.
5. Check that Settings › References › Highlights tabs reads **Allowed**.
6. Reinstall twice more and confirm it still reads Allowed.

**After `tabs-entry` lands** (smoke test, ideally with a few fixture PDFs open in
Highlights):

- the permission card on a fresh grant;
- ⇧⌘O ↵ flipping between two tabs, from Highlights frontmost and from the menu-bar
  global key while Obsidian is frontmost;
- ⌘T and Esc;
- a tab closed while the panel is up;
- Highlights quit and relaunched;
- a collision row (`harness_engineering.pdf`) switching via AXPress;
- two Highlights windows;
- IME, Esc, and the retained Capture draft still behaving as before.

## Follow-ups (not in this epic)

- Rename one `harness_engineering.pdf`, or add an intake guard against duplicate PDF
  file names, so file-name matching becomes exact by construction.
- After real use, consider A12's open-tab glyph, an `AXObserver` for exact recency, and
  exact background paths through kept window handles.
