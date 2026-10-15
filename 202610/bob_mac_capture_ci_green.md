---
tier: tale
title: Get bob-mac-capture CI green again
goal:
  The macOS 26 SwiftPM job passes on master again, with the three root-cause defects
  fixed and no test able to crash xctest.
size: medium
proposed_by: bbugyi200.apollo.7u
create_time: 2026-10-15 10:05:11
status: wip
---

# Plan: Get bob-mac-capture CI green again

All work happens in the linked `bob-mac-capture` repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`), use only the path it prints, and read its
`AGENTS.md` before editing. Every path below is relative to that checkout.

## Diagnosis (from the GitHub Actions logs)

`actstat` is not installed on apollo, so the runs were read with `gh run list` /
`gh run view --log-failed`. The single job, **macOS 26 SwiftPM**, has failed on every
push since `2837025 feat(refs-inspector): page revealed notes...` (2026-10-13). The last
green run was `4da688e` (run 38424226217). HEAD `7533626` (run 38746202123) still fails.
Lint and Build pass; the **Test** step fails, which skips Bundle, plist/signature
verification, the launch smoke test, and the install gates. Three independent defects
cause the failures:

### 1. Hosted `RefsInspectorScrollTests` fail on every run (canned inspector content is overwritten)

Failing: `testHostedNotesAtBottomRevealPages` and
`testHostedShortContentRevealsWithNoOffsetChange` on every run since `2837025`.
`testHostedTwoRapidRevealsStackTwoPages`,
`testHostedNotesAboveBottomScrollsHalfPageAndRevealsNothing`, and
`testHostedControlDUScrollsInspectorWhileSearchKeepsFocus` fail intermittently. The
diagnostics always report `docHeight=239.0` against a 462pt clip, and the reveal never
happens (`visibleCount == 3`, `generation == 0`).

Root cause: the hosted harness seeds the inspector with
`RefsPanelModel.installForPreviews(inspector:)`, which calls
`RefsInspectorLoader.installContentForPreviews`. That method's doc comment promises
"skipping every process". But the real `RefsPanelView` calls
`model.inspectorRequested()` from `.onAppear` and `.onChange(of: selected)`
(`Sources/BobMacCapture/Refs/RefsPanelView.swift` ~183). That runs
`RefsInspectorLoader.request(item)` (~line 484). After the 120 ms settle delay the real
load runs. The harness library has `fetcher: nil`, so `ref show` throws, and the PDF
does not exist, so intrinsics come back `failed`. `rememberContent` then replaces the
canned long content with an empty
`{previewMessage: "Preview unavailable", showFailed: true, notes: []}` value. The
inspector shrinks to 239pt, which cannot scroll, and `revealMoreInspectorNotes()` sees
`feed.total == 0`, so nothing reveals. `settle()` still passes, because it briefly sees
the tall canned content before the replacement lands. That is why the outcome depends on
timing.

### 2. `RefsPanelModelTests.testQueuedDigitCopiesWhenTheShowLoaderReturns` fails and crashes xctest

Fails on every run since `24e4bc5 fix(refs): repair copy card landing behavior`, the
commit that added the test; it has never passed.
`Tests/BobMacCaptureTests/RefsPanelModelTests.swift` ~108-139 calls
`perform(.select(id:))` right after `waitForSnapshot()` without
`harness.model.prepareForPresentation()`. The model's listing is frozen at init, when it
is empty. `libraryDidPublish()` only refreshes content in place and never adds rows, so
`.select` returns false (line 122). Every sibling test that selects calls
`prepareForPresentation()` first, which is how they pass. With no selection there is no
copy card, `waitForModel` times out, and line 136 `harness.pasteboard.strings[0]`
indexes an empty array: `Fatal error: Index out of range` kills xctest with signal 5.

The crash also means every suite that sorts after `RefsPanelModelTests` has not run
since `24e4bc5`: the rest of `RefsPanelModelTests`, `RefsRankingTests`, `RefsScanTests`,
`RefsStoresTests`, `RefsTodayTests`, `StatusItemGlyphDesignTests`,
`StatusItemGlyphTests`, `TaskBlockDesignTests`, `TaskCompleteDesignTests`,
`TaskCompletePickerPresentationTests`, `TaskDisplayTextTests`,
`TaskGotoPickerPresentationTests`, and `TaskLinkPickerPresentationTests`. Since then,
`TaskBlockView.swift`, `CaptureTaskBlockPresentation.swift`,
`CapturePickerPresentation.swift`, and `CompletionRowContent.swift` have all changed, so
new failures may be hiding there.

### 3. Midnight crash: `NSCalendarDayChanged` handled off the main thread (production bug)

Run 38656909109 (`e5b2ef3`) ran its tests at exactly 00:00:00 UTC and died with
`NSInternalInconsistencyException: Modifications to the layout engine must not be performed from a background thread`.
The stack runs `__postAndResetMidnight` (a libdispatch worker) → the
`CapturePanelModel.subscribeToAgendaStore` sink → `noteAgendaDayChanged()` →
`publishAgendaLoading` → `CaptureAgendaHeightResolver` →
`CaptureAgendaRowMeasurer.renderHeight` (`NSHostingView` layout). Foundation posts
`NSCalendarDayChanged` on a background queue. The `NotificationCenter` publisher at
`Sources/BobMacCapture/CapturePanelModel.swift` ~1140-1149 (merged with
`NSSystemClockDidChange`) has no main-queue hop, so this `@MainActor` model runs AppKit
layout off the main thread. The app itself can crash the same way at local midnight.
`CaptureAgendaStore` (~line 384) already registers its observers with `queue: .main`;
the panel model is the only subscriber without that hop (it is the only
`.publisher(for:)` in `Sources`).

### Already resolved / not actionable

- The `CaptureDependencyPanelTests` Blocks-picker failures (runs for `e41dee2` and
  `e5b2ef3`) no longer appear from `ac0172e` on. No action.
- One-off timeouts seen once each:
  `CapturePanelModelTests.testStartPendingListPreviewsTrimmedDraftWithStartDisabled`
  (run 38609278084) and
  `RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection` (run
  38486137058). Do not chase these unless they recur.
- swift-format indentation _warnings_ and the Node.js 20 deprecation annotations for
  `actions/checkout@v4` / `actions/upload-artifact@v4` do not fail the job. Out of
  scope.

## Changes

### A. Preview-installed inspector content is never re-hydrated (fixes defect 1)

In `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`:

- Add `private var previewContentIDs: Set<String> = []`, with a short doc comment in the
  file's style: ids whose content came from `installContentForPreviews`, which a later
  `request` keeps instead of hydrating over.
- `installContentForPreviews(_:thumbnails:)`: after the existing assignments, set
  `previewContentIDs = Set(content.keys)`.
- `reset()`: also clear `previewContentIDs`.
- `request(_:)`: cancel the previous load as today (`current?.cancel()`). If
  `previewContentIDs.contains(item.id)`, set `current = nil` and return without
  scheduling a load. Otherwise keep the existing debounce/load path unchanged.
- Update the `installContentForPreviews` doc comment to say the installed ids stay
  pinned: a later `request` for them is a no-op until `reset()` or the next install.

Production never calls `installContentForPreviews` (only
`RefsPanelModel.installForPreviews`, a preview/test API), so live hydration does not
change. Do not touch the `.onAppear` / `.onChange` hydration in `RefsPanelView`;
production needs it.

Regression test in `Tests/BobMacCaptureTests/RefsInspectorTests.swift`
(`RefsInspectorLoaderTests`): use the existing `makeFixtures()` (fake fetcher plus stub
intrinsics). Install preview content for `fixtures.paper.id` with a distinctive summary,
call `loader.request(fixtures.paper)`, and run past the settle delay (about 400 ms, e.g.
`try await Task.sleep`). Assert the content still equals the installed value and
`fixtures.fetcher.showPaths` is empty. Then call `loader.request(fixtures.chat)`, an id
that is not pinned, and use the existing `waitForContent` helper to assert it still
hydrates. Leave the hosted `RefsInspectorScrollTests` assertions as they are; they
should pass once the canned content survives.

### B. Fix `testQueuedDigitCopiesWhenTheShowLoaderReturns` and stop crashing the run (fixes defect 2)

In `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`:

- In `testQueuedDigitCopiesWhenTheShowLoaderReturns`, call
  `harness.model.prepareForPresentation()` after `await harness.waitForSnapshot()` and
  before `perform(.select(id: id))`, matching the sibling tests (for example
  `testOpenHidesDispatchesAndRecords`).
- Make its pasteboard check non-crashing: assert `strings.count == 1` first, then check
  the content through `strings.first` (for example
  `XCTAssertTrue(harness.pasteboard.strings.first?.contains("Comment: Loaded comment") == true)`
  or `try XCTUnwrap(harness.pasteboard.strings.first)`). A future failure then fails one
  test instead of killing xctest and hiding a dozen suites.
- Harden the other unguarded pasteboard subscripts in the same file the same way:
  `strings[0]` at ~507, ~697, ~1374 and `strings[1]` at ~531-532. Use `XCTUnwrap` /
  `.first` / a count guard plus a safe index. Keep each test's meaning unchanged.

### C. Hop day/clock-change notifications to the main queue (fixes defect 3)

In `Sources/BobMacCapture/CapturePanelModel.swift` `subscribeToAgendaStore()` (~1140),
add `.receive(on: DispatchQueue.main)` after the `.merge(...)` and before `.sink`. Add a
one-line comment saying that Foundation posts `NSCalendarDayChanged` on a background
queue, so the hop keeps agenda measurement (`NSHostingView` layout) on the main thread,
matching `CaptureAgendaStore`'s `queue: .main` observers. Use `DispatchQueue.main`, not
`RunLoop.main`, so the hop also runs during event-tracking run-loop modes. Do not add a
regression test that posts these notifications from a background queue. Before the fix
the failure is a process-killing `NSException` that only fires on a row-height cache
miss, so such a test would be nondeterministic. It would also poke the shared
`CaptureAgendaStore` observers.

## Verification

Follow `AGENTS.md` and never land Swift you have not compiled:

1. `just portable-check` on a host with Swift (athena over the tailnet; apollo has no
   `swift`). This runs the swift-format lint that the CI **Lint Swift formatting** step
   runs, plus the portable `CaptureCore` / `RefsCore` builds and tests. That also covers
   the previously hidden `RefsRankingTests`, `RefsScanTests`, `RefsStoresTests`,
   `RefsTodayTests`, `TaskCompletePickerPresentationTests`, `TaskDisplayTextTests`,
   `TaskGotoPickerPresentationTests`, and `TaskLinkPickerPresentationTests`. If any of
   them fail, fix them in this change when the cause is clear and local. Otherwise
   record them with `/sase_new_task` (type `ci`).
2. Changes A-C touch `Sources/BobMacCapture*` and `Tests/BobMacCapture*`, so compile a
   scratch copy on the mac as `AGENTS.md` describes (best effort): rsync the checkout
   without `.build` / `.git` into a temp dir, run `./Scripts/bundle.sh --identity "-"`
   and `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
   `just install` there. The mac has only Command Line Tools (no XCTest), so the
   hosted/design suites only _run_ in CI. If the mac is offline, say in the final report
   that the macOS targets are unverified.
3. Statically review the CI-only suites that the crash has hidden since `24e4bc5`
   (`StatusItemGlyphDesignTests`, `StatusItemGlyphTests`, `TaskBlockDesignTests`,
   `TaskCompleteDesignTests`, and the `RefsPanelModelTests` cases after
   `testQueuedDigit...`). Compare them against source changes since `2b7b330`, above all
   `TaskBlockView.swift` and `CaptureTaskBlockPresentation.swift`. Fix any assertion
   that clearly no longer matches intended behavior.
4. Acceptance: after the change lands on `master`, the next **macOS 26 SwiftPM** run
   passes Test and the later Bundle / plist-signature / launch-smoke / install gates.
   Check it with `gh run list -R bobs-org/bob-mac-capture -L 3` and
   `gh run view <id> --log-failed`. Any remaining failure in a suite this plan did not
   touch is follow-up work: file it with `/sase_new_task` (type `ci`) and include the
   run id.

## Out of scope

- Installing or fixing `actstat` on apollo.
- Swift-format warning cleanup and GitHub Actions Node 20 → 24 action bumps.
