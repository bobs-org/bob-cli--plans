---
tier: tale
title: Refs list rebuilds rows that change section so ⌘0 Newest highlights its first row
goal:
  After ⌘0 (and any re-rank that moves rows between sections), every visible Refs list
  row shows the current selection highlight and caption, so the first Newest row is
  visibly selected.
size: medium
proposed_by: bbugyi200.athena.12b
proposed_with: claude/opus
create_time: 2026-10-20 06:52:05
status: wip
---

# Plan: Fix ⌘0 Newest leaving the first row unhighlighted (stale Refs list rows after a cross-section re-rank)

All work happens in the **bob-mac-capture** linked repo. Open it with the `/sase_repo`
skill (`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md`, and use only
the path that command prints. Nothing changes in bob-cli itself.

## Symptom

In the Bob Refs panel, pressing ⌘0 switches the list to the Newest timeline. The model
selects the first Newest row, and the inspector shows that row. But no list row has the
selection highlight (`Color.accentColor.opacity(0.16)` from `RefsRowView.background`).
Bryan's screenshot shows the first row, under `TODAY 1`, without the tint, while the
inspector on the right shows that same reference.

## Root cause (diagnosed)

**The model is correct.** `RefsPanelModel.setNewest` → `rerankSelectingFirst()` sets
`listing = libraryListing()` and then `selectedID = RefsSelectionPolicy.initial(in:)`,
which is `orderedIDs.first`. `testNewestToggleSelectsFirstAndKeepsMembership` already
covers this, and the inspector, which renders `model.selectedID`, confirms it.

**The list view shows stale rows.** The same screenshot proves that the rows on screen
were never re-rendered after the re-rank:

- `docs/refs.md` says a row in a Newest day group must show `added 9:41 AM` (see
  `RefsCaption.newestPhrase`).
- Rows that were not on screen in the ranked listing do show Newest captions
  (`added 16:35`, `added 12:30`, …).
- Rows that were on screen in the ranked listing still show ranked captions:
  - The first row reads `added today`, the Ready-section `addedPhrase`.
  - Others read `added yesterday`.
  - Two read `opened 17h ago`, the Reading-section phrase.
- Those stale rows also kept their old `isSelected == false`. That is the missing
  highlight.

**Mechanism.** `RefsListView` (live path, non-preview) builds this structure:

```swift
LazyVStack(pinnedViews: [.sectionHeaders]) {
  ForEach(model.listing.sections, id: \.sectionID) { section in
    Section {
      ForEach(section.ids, id: \.self) { id in
        row(for: id)
          .id(id)          // leftover from the old ScrollViewReader scrolling
      }
    } header: { ... }
  }
}
```

A row's identity is its bare ref id, through both the inner `ForEach` id and the
explicit `.id(id)`. The ⌘0 toggle replaces every section. Ranked kinds such as
`section-ready` and `section-reading` become Newest groups such as
`section-day-2026-10-20`, so every materialized row _moves_ to a different section
`ForEach`. SwiftUI's lazy stack keeps the moved row's subtree under its bare id. That
subtree is still fed by the removed section's `ForEach` closure, so `row(for:)`'s values
never reach it: `isSelected`, `content.caption`, and `content.section` stay stale. This
is a known class of lazy-stack glitch: rows moved between `ForEach` groups do not
refresh their visual state. Rows that stay inside the same section `ForEach` update
normally, which is why arrow-key selection and content refreshes work.

The same bug hits every re-rank that moves visible rows across sections:

- ⌘0 back to Library. The old first Newest row can keep a ghost highlight.
- Typing a query from browse, which moves rows into the single `search` section.
- Arrivals joining through `freshListingKeepingSelection`.

The fix below covers all of these.

The explicit `.id(id)` is dead weight. Commit 61bdd22 replaced
`ScrollViewReader.scrollTo(id)` with `ScrollPosition.scrollTo(y:)` and
`scrollTo(edge:)`, and no code looks rows up by id anymore. The only `.id(id)` in
`Sources/` is `RefsListView.swift`'s.

## Fix

### 1. Section-qualified row identity in the live list

In `Sources/BobMacCapture/Refs/RefsListView.swift`, live (non-`previewMode`) branch:

- Add a fileprivate `Hashable` key, for example
  `struct RefsListRowKey: Hashable { let section: String; let id: String }`. Add a
  fileprivate helper on `RefsSection` next to the existing `sectionID` extension that
  maps `ids` to keys built from `sectionID`.
- Iterate the inner `ForEach` over those keys (`id: \.self`) and call
  `row(for: key.id)`. The ids themselves still feed `row(for:)`, the model, and
  selection exactly as before.
- Delete the `.id(id)` modifier.

With this identity, a row that changes section is a new subview, built fresh from the
current model. A row that stays in its section keeps its identity, which preserves:

- the tab-close fold-out: the Tabs listing has one section, and close removals stay
  within it;
- the NEW-dot `.animation(value: content.isNew)` after ⌥N, which is a content-only
  refresh with frozen sections;
- normal arrow-key updates.

Leave the `previewMode` static `VStack` path alone. It is not lazy, and design fixtures
render it once.

Update the `RefsListView` doc comment to say why rows are section-qualified. One short
sentence is enough, for example: a re-rank that moves a row to another section rebuilds
it, so its highlight and caption always match the new listing.

### 2. Test-only render probe

The bug lives in what the lazy stack _applies_, not in what the model computes, so the
regression test needs to see each row's applied inputs. Add a minimal, internal (not
`public`) hook:

- In `RefsPanelModel.swift`, add a small `final class RefsRowRenderProbe`.
  - It holds `private(set) var rows: [String: RefsRowRenderState]` and a
    `record(_ id: String, _ state: RefsRowRenderState)` method.
  - `RefsRowRenderState` is an `Equatable` struct with three fields: `isSelected: Bool`,
    `section: RefsSectionKind?`, and `caption: String`.
  - Add an internal stored property `var rowRenderProbeForTests: RefsRowRenderProbe?`.
    It is nil in production and not `@Published`. Follow the precedent of
    `rowContentEvaluationsForTests`.
- In `RefsListView.row(for:)`, inside `if let content`, attach to the `RefsRowView` an
  `.onChange(of: RefsRowRenderState(isSelected:section:caption:), initial: true) { _, state in model.rowRenderProbeForTests?.record(id, state) }`.
  - Because the modifier lives in the same subtree as `RefsRowView`, it fires only when
    new inputs are actually applied to that row. A stale moved row never fires, so the
    probe keeps its old state.
  - Attach it unconditionally. A conditional modifier would change the row's view type.
    The per-row cost is one small struct comparison.
  - Do not put the record call directly in `row(for:)`'s body evaluation. That would
    only prove the closure ran, not that the row applied it.

### 3. Hosted regression test

Add `Tests/BobMacCaptureTests/RefsListRowRefreshHostedTests.swift`, a `@MainActor`
`XCTestCase`. It mirrors `RefsListScrollHostedTests`'s hosted harness, which you may
copy or factor:

- a `RefsLibrary` with temp stores and `FakeSpotlight`;
- a `RefsPanelModel` with `FakeOpener`, `FakeLocator(appURL: nil)`, and
  `FakePasteboard`;
- `installForPreviews`;
- `RefsPanelController(model:).makePanelIfNeeded()`;
- a frame of `1344×798`, then `orderFrontRegardless()`;
- a runloop `settle`.

Set `model.rowRenderProbeForTests = RefsRowRenderProbe()` **before** the panel lays out.

Use a deterministic fixture. Set `RefsSignals(now:calendar:)` to a fixed instant and a
UTC Gregorian calendar, as `RefsPanelDesignTests.fixedNow()` does. Give every item
`annotationCount: 1` so nothing lands in NEW.

| Rows        | `state`    | `added`                    | `addedAt` | Title           |
| ----------- | ---------- | -------------------------- | --------- | --------------- |
| R0–R2       | `.reading` | ~40 days ago (month group) | none      | `Reading row N` |
| X           | `.ready`   | today                      | 09:41     | `Alpha newest`  |
| Y           | `.ready`   | today                      | 08:00     | `Bravo newest`  |
| ~40 fillers | `.ready`   | 2–30 days ago              | none      | `Filler row N`  |

The fixture puts X and Y on screen in the ranked listing while neither is the ranked
first row.

Every wait spins the runloop against a deadline of about 2 s, then calls `XCTFail` with
diagnostics: the model's `selectedID`, `listing.order`, and the probe's entries for the
ids involved. Never await an untimed continuation (see `AGENTS.md`).

1. **Precondition (ranked).** After settling:
   - the selection is `R0`;
   - the probe shows X rendered with `isSelected == false` and section `.ready`.

   This proves X was materialized before the toggle, which is the screenshot's stale
   path.

2. **⌘0 to Newest.** `XCTAssertTrue(model.perform(.toggleNewest))`, then wait until the
   probe shows X with:
   - `isSelected == true`;
   - section `.added` whose group `id` is `day-<today>`;
   - a caption containing `added ` and the shortTime of 09:41. Prefer comparing with
     `RefsCaption.datePhrase` or `newestPhrase` over a hard-coded locale string.

   Also assert `model.selectedID == X`.

3. **⌘0 back to Library.** Toggle again, then wait until:
   - the probe shows X with `isSelected == false` and section `.ready`, so no ghost
     highlight remains;
   - `R0` is rendered selected.
4. **Same-section guard.** Turn Newest on again. Then `model.setQuery("bravo")` (or the
   model's query entry point that re-ranks). Y stays in the same `day-<today>` group, so
   its identity is unchanged. Wait until the probe shows Y with `isSelected == true`.

The test should fail on the current code at step 2 and pass with the fix. Only CI can
run XCTest (see below), so make step 1's precondition strict. The test must provably
drive the stale path, not pass vacuously.

## Verification

Per bob-mac-capture's `AGENTS.md`:

1. Run `just portable-check` on a Linux host with Swift (athena). It parse-lints every
   file, including the new test, and runs the portable suites.
2. Compile the macOS targets on the Mac (best effort). Read the `tailnet` reference
   memory first (`sase memory read tailnet.md -r "..."`) for SSH access.
   - rsync a scratch copy without `.build`/`.git`;
   - run `Scripts/bundle.sh`;
   - run `just typecheck-tests` (the Mac has no XCTest);
   - delete the scratch copy.

   Never touch Bryan's checkout or run `just install` there. If the Mac is offline, say
   in the final report that the macOS targets are unverified.

3. Run `swift-format lint` cleanly. A lint parse error skips every later CI step.
4. After the change lands, check the bob-mac-capture CI run (`gh run list` /
   `gh run view`), whose macOS 26 job runs the hosted tests. Report whether
   `RefsListRowRefreshHostedTests` passed. If the run hasn't finished within the turn,
   say so rather than claiming success.
5. In the final report, give Bryan these manual checks to run after his next
   `just install`:
   - open the Refs panel and press ⌘0: the first `TODAY` row is highlighted, and every
     visible caption reads `added HH:MM`;
   - press ⌘0 again: the first library row is highlighted, and no other row keeps a
     highlight;
   - type a query from browse: the first match is highlighted.

## Out of scope

- No model, RefsCore, ranking, or selection-policy changes. They are already correct and
  tested.
- No docs change. `docs/refs.md` and the README already describe the intended behavior,
  which this fix restores: Newest captions, and a re-rank selecting the first row.
- Leave `RefsListScrollPolicy` and the `ScrollPosition` reveal logic alone. They key off
  model geometry, not view ids.
