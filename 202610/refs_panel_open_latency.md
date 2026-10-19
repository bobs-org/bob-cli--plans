---
tier: tale
title: Make the Bob Refs panel open fast
goal:
  Opening Bob Refs (⌃⇧⌘O, Highlights ⌘O, or the menu) no longer blocks the main thread
  for 0.2–1.2 s.
size: medium
decisions:
  capture_targets_watcher:
    ask:
      Should Capture's targets watcher also skip batches that touch only dot paths
      (.git/, .obsidian/, .trash/)?
    choices:
      keep:
        Only Refs changes; Capture still runs capture-targets twice every ~16 s during
        vault sync
      filter:
        Capture's targets refresh also skips dot-path-only batches, ending that churn
        too
    default: keep
    why:
      Refresh-on-every-batch is a deliberate Capture contract; change it in its own
      review
    answer: filter
proposed_by: bbugyi200.athena.11q
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 09:43:47
status: wip
---

# Make the Bob Refs panel open fast (⌃⇧⌘O and every other entry point)

All code changes in this plan are in the **`bob-mac-capture`** linked repo. Open it with
your `/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), work only in the
path it prints, read that repo's `AGENTS.md` first, and commit that repo when you are
done. No bob-cli files change.

## Diagnosis (measured on the Mac, 2026-10-19)

Evidence came from the app's own `OSSignposter` intervals
(`log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"'`). A
`sample` of the running app was also taken while ⌃⇧⌘O was synthesized twice.

- **The hotkey is not the slow part.** `refs-hotkey-received` → `refs-panel-show` begin
  is 0–3 ms whenever the main thread is idle. The Carbon `.refs` hotkey, the Highlights
  ⌘O takeover (`.refsHighlightsOpen`), and the menu-bar "Bob Refs…" item all run the
  same `BobPanelCoordinator.toggleRefs()` → `RefsPanelController.show()`. So all three
  entry points are equally slow. A few opens queued 125–394 ms behind other main-thread
  work (see cause 3).
- **`show()` itself blocks the main thread for 237 ms – 1.19 s** (12 real opens: 237,
  269, 279, 337, 389, 390, 417, 1021, 1040, 1191 ms, …; synthetic opens 744 and 406 ms).
  Capture's `panel-order` takes ~100 ms by comparison. After `show()` returns, SwiftUI
  runs about two more full passes over the rows (display-link layout and run-loop
  observer flush) before the fade-in settles. So the felt delay is longer than the
  signpost.
- Library size on the Mac: 1,080 references (`refs-snapshot.json` is 650 KB).

### Root cause 1 (primary): per-row O(n) lookups that copy `RefItem`s

Nearly all of `show()` sits under `NSWindow.makeKeyAndOrderFront` →
`NSHostingView.layout()` → `RefsListView.body` ForEach row closure
(`RefsListView.swift:117`) → `RefsListView.row(for:)`. Inside it, the two hot frames
are:

- `RefsPanelModel.rowContent(for:)` at `library.items.first(where: { $0.id == id })`
  (`RefsPanelModel.swift:1394`)
- `RefsPanelModel.pomodoroName(for:)` at the same lookup (`RefsPanelModel.swift:616`)

Their self time is `outlined init with copy of RefItem` / `destroy for RefItem`. Each
linear scan copies and destroys a ~20-field `RefItem` per element. `rowContent(for:)`
also scans linearly on `listing.orderedIDs.contains(id)` (:1366) and
`listing.sections.first { $0.ids.contains(id) }` (:1371). `RefsExplanation.whyHere`
repeats the section scan (`RefsCaption.swift:366`).

### Root cause 2: the "lazy" list evaluates every row

The measured cost per pass (~345 ms) only fits about 1,000+ row evaluations, not the ~20
visible rows. With a ~20-row window the same lookups would cost ~6 ms. `row(for:)`
returns `AnyView(EmptyView())` or `AnyView(RefsRowView…)`. That gives each `ForEach`
element a non-constant view count (0 or 1). Apple's `ForEach` docs say lazy containers
like `LazyVStack` query elements lazily only when each element produces a _constant_
number of views. So cause 1 runs over the whole library roughly three times per open
(O(n²) per pass). That is why it gets worse as the library grows. The pattern dates from
the panel's first commit (2026-10-09), not from a recent regression.

### Root cause 3 (secondary): Refs refreshes on `.git/` churn every ~16 s

The vault-sync agent's git fetches rewrite `~/bob/.git/` (`.git`,
`.git/refs/remotes/origin`) about every 16 s. `RefsLibrary.startWatcher()` watches the
vault root recursively and calls `handleWatcherChange()` for **every** batch. So every
16 s the app runs a full Refs pass: `bob ref list`, a Spotlight sweep of 1,080 PDFs, a
~40 ms main-thread `RefsSnapshotStore.save` of 650 KB, and an items/signals republish
into the model. That main-actor work is what occasionally delays a hotkey press.
Capture's `CaptureAgendaRefreshFilter` already ignores dot-path churn; the Refs watcher
does not.

Ranking (`prepareForPresentation` → `RefsRanker.listing`) costs ~30 ms per open. That is
not the problem and is out of scope.

## Changes

### 1. O(1) item lookup in `RefsLibrary`

- In `Sources/BobMacCapture/Refs/RefsLibrary.swift`, keep a private `[String: Int]`
  index from id to position in `items`. Rebuild it whenever `items` is assigned, for
  example in a `didSet` on the `@Published` property, so `start()`, `runSnapshotPass`,
  `runGitDatePass`, and `installSnapshotForPreviews` all stay in sync. If ids repeat,
  the first occurrence wins, matching today's `first(where:)`.
- Add `public func item(id: String) -> RefItem?`.
- Replace every `library.items.first(where: { $0.id == … })` /
  `library.items.first { $0.id == … }` in `RefsPanelModel.swift` (today at lines 616,
  627, 1095, 1375, 1394, 1444, 2598, 2628, 2652; re-grep, since line numbers drift) with
  `library.item(id:)`.

### 2. O(1) membership and section lookup on `RefsListing`

- In `Sources/RefsCore/RefsRanking.swift`, have `RefsListing.init` build derived lookups
  once per listing: a set of `orderedIDs`, and an id → section index map. The first
  section that contains the id wins, matching `sections.first { $0.ids.contains(id) }`.
- Expose `contains(id:)`, `sectionIndex(for:)`, and `sectionKind(for:)`.
- Keep `RefsListing`'s equality defined by its existing fields. The derived lookups must
  not change what `==` means: write `==` by hand, or otherwise make sure the derived
  storage cannot diverge.
- Use the new lookups on every per-row or per-render path:
  - `RefsPanelModel.rowContent(for:)` (the `orderedIDs.contains` guard and the section
    lookup)
  - `RefsExplanation.whyHere` (`RefsCaption.swift`)
  - `RefsSelectionPolicy`'s section-index lookup (`RefsSelection.swift`)
  - Any other `listing.orderedIDs.contains` / `sections.first { $0.ids.contains }` in
    the model that runs per render. One-shot user actions may also switch, for
    consistency.

### 3. Make each list row a constant single view

- In `Sources/BobMacCapture/Refs/RefsListView.swift`, remove `AnyView` from `row(for:)`.
  Each `ForEach` element must resolve to exactly one view of one concrete type. For
  example, put the optional-content branch inside a single unary container
  (`ZStack`/`VStack(spacing: 0)`), or extract a concrete row view.
- Keep `RefsRowContent`, `isSelected`, `pomodoroName`, and the callbacks computed in the
  parent's `ForEach` closure and passed in as values. Do not make each row hold `model`
  and derive its content in its own body. A row struct that only stores the model
  reference may skip re-rendering when the model changes.
- Preserve today's behavior exactly:
  - `.id(id)` for `ScrollViewReader`
  - the `dimRows` opacity
  - the selected-row `.onGeometryChange` rect reporting used by the ⌘K anchor
  - the close button handler
  - the tab-close removal transition. Put `.transition(removalTransition)` on the
    outermost per-element view so fold-out still animates.
- Leave `previewMode` (design fixtures) rendering unchanged.

### 4. Ignore dot-path-only batches in the Refs watcher

- Add a pure
  `RefsWatcherRelevance.isRelevant(_ batch: VaultChangeBatch, vaultRoot: String) -> Bool`
  in RefsCore. RefsCore already depends on CaptureCore, which defines
  `VaultChangeBatch`. Its rules:
  - true when the flags include `mustScanSubDirs`, `rootChanged`, `kernelDropped`, or
    `userDropped`;
  - true when any path lies outside the vault-root prefix (conservative), or when the
    batch has no paths;
  - true when any path under the root has no path component starting with `.`, whatever
    its extension (PDFs in `lib/` matter here, unlike the agenda's `.md`-only rule);
  - false otherwise, so `.git/`, `.obsidian/`, and `.trash/` churn is ignored.
  - Match the prefix against both the configured root path and its
    `resolvingSymlinksInPath()` form.
- In `RefsLibrary.startWatcher()`, pass `onBatch:` (with a no-op `onChange`). Call
  `handleWatcherChange()` only when the batch is relevant. The scan/rename deferral
  inside `handleWatcherChange()` stays as it is.
- Update the README "Updates" paragraph (the `RefsLibrary` refresh-service description,
  "watches the vault root (root `*.md` notes) plus …") to say dot-path-only changes
  (`.git/`, `.obsidian/`, `.trash/`) no longer trigger a Refs refresh.

> [!decision] capture_targets_watcher = filter Also gate
> `AppDelegate.configureVaultWatcher()`'s `onChange` targets refresh on a dot-path
> relevance check: reuse `RefsWatcherRelevance` or put a shared helper in CaptureCore.
> Keep `onBatch` → `agendaStore.vaultDidChange` unchanged. Update the README
> Capture-targets bullet to match. With `keep`, do not touch the Capture watcher.

### 5. Finer show signposts

- Inside `RefsPanelController.show()`, add intervals `refs-panel-prepare` (around
  `prepareForPresentation` + `refreshForOpen`) and `refs-panel-order` (around
  `orderFront()`). Use the existing `CaptureSignpost` helpers, names only, no payloads.
- Note them in the README's Signposts list next to the existing intervals.

## Tests

- **RefsCoreTests (portable; run with `just portable-check`):**
  - `RefsListing` lookups agree with the old linear scans for browse, search, Newest,
    and Tabs-shaped listings (unknown ids, ids in later sections, first-section-wins).
  - `refreshingContent` keeps the lookups correct.
  - `RefsWatcherRelevance` table cases: `.git/index`; `.git/refs/remotes/origin` plus
    the dir flag; `.obsidian/workspace.json` → false. `ref/x.md`, `lib/y.pdf`, root
    `inbox.md`, `done/z.md`, a `.git` + `ref/` mix, any drop/rescan flag, a path outside
    the root, an empty batch → true.
- **BobMacCaptureTests (macOS; CI runs them):**
  - `RefsLibrary.item(id:)` tracks `items` after `start()` from a cached snapshot, after
    a refresh pass, after the git-date merge, and after `installSnapshotForPreviews`.
  - Laziness regression: host the live (non-preview) `RefsListView` or `RefsPanelView`
    in an `NSHostingView` at panel size with ~1,000 preview items. Lay it out and assert
    that far fewer rows were built than items (for example under 150). Count builds with
    a `rowContentEvaluationsForTests` counter on `RefsPanelModel`, following the
    existing `closeWorkerLaunchCountForTests` pattern. If CI shows this failing, find
    the remaining eager path (SwiftUI's `LogForEachSlowPath` user default) before
    loosening the bound. Never delete the test.
  - Existing Refs design, tabs-close, and model tests must stay green.

## Verification

- Follow the repo's AGENTS.md exactly. Run `just portable-check`.
- Then compile a scratch copy on the Mac (`ssh mac`, best effort): rsync without
  `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the
  copy. Never use Bryan's checkout or `just install` there.
- If the Mac is offline, say in the final report that the macOS targets are unverified.
- In the final report, tell Bryan how to confirm after he installs. Open the panel with
  ⌃⇧⌘O a few times, then run
  `log show --last 10m --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"' --style compact | grep -E 'refs-(hotkey|panel)'`.
  `refs-panel-show` should drop from the 237 ms – 1.19 s baseline to well under 100 ms.
  With no edits under `ref/`, `lib/`, `done/`, or root notes, `refs-refresh` should no
  longer appear every ~16 s.

## Out of scope

- Moving `RefsSnapshotStore.save` or ranking off the main actor.
- A `startCloseWorkerIfNeeded` CPU spin. Oct 16 `cpu_resource` reports flagged one, but
  it predates the close-worker repair in `175153c`.
- The vault-sync agent's ~16 s fetch cadence itself.
