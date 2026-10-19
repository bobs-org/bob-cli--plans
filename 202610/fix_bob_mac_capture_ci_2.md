---
tier: tale
title: Fix the red bob-mac-capture macOS CI
goal:
  The bobs-org/bob-mac-capture macOS 26 SwiftPM job is green again, Test step included.
size: medium
decisions:
  ci_proof:
    ask:
      Should the fix be proven green on a throwaway draft PR before it lands on master?
    choices:
      draft_pr:
        Iterate on a temp PR until the whole macOS job is green, then land; 10–30 min CI
        wait per round
      land_only:
        Land after local gates; the next master run may expose more and need follow-up
        work
    default: draft_pr
    why:
      CI is the only place these tests run (the Mac lacks XCTest), and every failure
      landed unrun
    answer: draft_pr
proposed_by: bbugyi200.athena.128
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-19 17:01:20
status: wip
---

# Fix the red bob-mac-capture CI (macOS 26 SwiftPM → Test)

All work happens in the `bob-mac-capture` linked repo. Open it with your `/sase_repo`
skill (`sase repo open bob-mac-capture -r "..."`), use only the path it prints, and read
its `AGENTS.md` before editing. Line numbers below are approximate, taken at HEAD
`f7e635a`. Re-locate each one by test or function name.

## Diagnosis

`actstat` shows every recent `bobs-org/bob-mac-capture` master run failing in the
`macOS 26 SwiftPM` job at step 6, **Test**. The latest is CI #355, run 39316605254 on
f7e635a: 2654 tests, 26 assertion failures across 10 tests.

How it got here:

- **Last green run:** 4da688e on 2026-10-13. Since then every run was cancelled (hangs,
  before the watchdog existed), failed lint, or failed Test. So **Bundle, Verify plist
  and signature, Launch smoke test, and Install and reinstall have not run since
  2026-10-13.**
- **5a0bff5 → 6a6c0c0 (Oct 18–19): the test build did not compile.** A `zone` property
  in `RefsNewestTests.swift` clashed with `NSObject.zone()`, so no tests ran. The
  capture commits that landed in that window were never exercised by CI: 2c007c7 (bare
  `x` close), 4c391eb (override banner), 12219ef (`--refuse-override` guard), and
  others.
- **Since 5fd928b fixed the compile:** every run fails the same 7
  `CapturePanelModelTests`, plus a rotating set of Refs tests.

| Test                                                                              | Runs failing            | Verdict                         |
| --------------------------------------------------------------------------------- | ----------------------- | ------------------------------- |
| `CapturePanelModelTests` × 7 (listed in §1)                                       | every run since 5fd928b | stale tests (deterministic)     |
| `RefsInspectorLoaderTests.testIntrinsicsDiscardDropsInFlightAndAllowsFutureLoads` | ~8 of 14 runs           | production overload trap → race |
| `RefsPanelModelTests.testClosingRowIsNotResurrectedByInFlightRefresh`             | f7e635a only            | regression from f7e635a         |
| `RefsPanelModelTests.testTabsFilterBlocksArrivalRerank`                           | f7e635a only (new test) | test never scripted a Tabs read |
| `RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection`              | 5fd928b, b2f24d0        | timing race, currently dormant  |

## 1. `CapturePanelModelTests`: seven stale tests

All seven are in `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`. The production
code behaves as intended; the tests were never run after the commits that changed their
inputs.

### 1a. Cursor offsets past the end of the draft (4 tests)

2c007c7 rewrote `=x…` drafts as bare `x…`, which made each draft one byte shorter, but
left their cursor offsets unchanged. Analysis then fails with "Cursor is not on a UTF-8
boundary", so the preview never reaches `.ready`. Set each offset to the draft's UTF-8
length:

- `testRepresentationRefreshesStaleCloseTiming` (~3859): `"x"`, `cursorUTF8Offset: 2` →
  `1`
- `testCloseSpanStaysOutOfCompletionGating` (~3893): `"x"`, `2` → `1`
- `testCloseLivePreviewDecodesFakeBobFixtureAndShowsCloseAction` (~3908): `"x"`, `2` →
  `1`
- `testClosePendingListPreviewsTrimmedDraftWithCloseDisabled`, the second half (~2997):
  `model.plainDraft = "x1"`, `3` → `2`

A scan of every `plainDraft = "<literal>"` followed by
`editorTextDidChange(cursorUTF8Offset:)` under `Tests/BobMacCaptureTests/` found exactly
these four.

### 1b. `=3` now dry-runs successfully (2 tests)

4c391eb regenerated `Tests/Fixtures/fake-bob` to match real bob's override semantics.
`capture --dry-run -- =3` is now a successful counted restart ("Restart CAPTURE")
instead of a "still running" failure. Two tests still wait for `.failed`:

- **`testFailedCloseDryRunClearsStaleCard` (~2309).** Keep the live `x` close card as
  the setup. Change the draft that should fail from `=3` to `x3` (cursor 2).
  - In fake-bob, `capture-parse -- x3` falls through to the default `ok` task-mode
    parse, and `capture --dry-run ... -- x3` exits 1 with "`x3` names task 3, but
    CAPTURE has 2 numbered Task Links (1–2)".
  - Update the comment to describe that failure. Keep every assertion: close, start, and
    preview state all cleared, and `primaryActionTitle == "Capture"`.
  - If `primaryActionTitle` turns out to differ on this path, first read how
    `primaryActionTitle` is derived, then change only that one expectation.
- **`testStartSuffixCaretRequestsNoCompletion` (~3528).** Keep `=3` and wait for
  `.ready` instead of `.failed`. Update the comment: the dry run still runs and previews
  the counted restart. The completion assertions (no `capture-complete` call, no
  completion visible) do not change.

### 1c. `testOlderBobRetriesSubmitWithoutRefuseOverride` counts the wrong records (~3272)

The test counts `record.components(separatedBy: "argv=capture")`. That separator also
matches hyphenated subcommands that run asynchronously around the submit, for example
the idle-agenda `capture-pomodoros --tasks` fetch after the draft clears. CI saw 3 or 4
records instead of 2, varying between runs.

The sibling `testUnpreviewedEqualsSubmitSendsRefuseOverride` asserts that no dry run
happens in this flow, so only the two real submits should be counted:

```swift
let invocations = record.split(separator: "\n").filter { $0.hasPrefix("argv=capture --") }
XCTAssertEqual(invocations.count, 2, record)
XCTAssertTrue(invocations[0].contains("--refuse-override"))
XCTAssertFalse(invocations[1].contains("--refuse-override"))
```

Guard the indexing so a count mismatch fails cleanly instead of trapping.

## 2. `testIntrinsicsDiscardDropsInFlightAndAllowsFutureLoads`: overload trap (production fix)

The race is in `Sources/BobMacCapture/Refs/RefsInspectorLoader.swift`:

- `RefsPDFIntrinsicsProviding` requires `func discard(id: String) async`, and a protocol
  extension supplies an `async` no-op default (~56–62).
- The actor `RefsPDFIntrinsicsLoader` implements `public func discard(id: String)`
  **synchronously** (~191).
- In an async context, Swift prefers the async overload. So the test's
  `await loader.discard(id:)`, called on the concrete actor, hits the extension's no-op.
  A scratch program on Swift 6.0.3 reproduced this.

As a result, `discardGenerations` never advances, both in-flight reads cache their
result, and whichever read finishes last wins: 7 or 8. bc6eb30 only turned what had been
a deterministic failure into this race.

Production calls `discard` through the existential (`RefsInspectorLoader.discard`,
~604), so it reaches the real method today. Any call on the concrete type silently does
nothing.

**Fix:** make the actor's method an exact match:
`public func discard(id: String) async {`. The body has no awaits, and every caller
already uses `await`. Do not change the test.

## 3. `RefsPanelModel` ⌘R regression from f7e635a, plus the two new or racy tests

### 3a. Start the Tabs read immediately on ⌘R (fixes `testClosingRowIsNotResurrectedByInFlightRefresh`)

f7e635a moved `beginTabsRefresh(preservingSelection: true)` into the Task that first
awaits `library.pullAndWait()`. This is in `perform(.refresh)` in
`Sources/BobMacCapture/Refs/RefsPanelModel.swift` (~834–847).

A ⌃X close only supersedes Tabs reads that have already started, because
`HighlightsTabsSession.close` bumps the session generation. A close that lands during
the pull therefore no longer cancels the ⌘R read. That late read then re-adds the closed
row.

Restore the pre-f7e635a order for Tabs: start the read synchronously and let only the
library refresh wait for the pull. Deferring the read never helped matching, for two
reasons:

- Tabs are matched only at read time, against the current `library.items`.
- `library.refresh` is async, so the deferred read never saw freshly pulled items
  anyway.

### 3b. Set the re-rank flag only once the pull is done

Move `pendingRefreshRerank = true` into the Task, immediately before
`library.refresh(reason: .manual)`. Otherwise any publish that arrives during the pull,
from the `-g` git-dates lane or the Today pass, can consume ⌘R's re-rank using pre-pull
data. f7e635a widened that window by putting a network pull in front of the refresh.

```swift
case .refresh:
  if tabsFilter {
    beginTabsRefresh(preservingSelection: true)
  }
  Task { [weak self] in
    await self?.library.pullAndWait()
    self?.pendingRefreshRerank = true
    self?.library.refresh(reason: .manual)
  }
  return true
```

Update the comment above it. Check the README's Bob Refs ⌘R / `refs-pull` paragraph and
fix any sentence this makes false. ⌘R still pulls before refreshing the library.
`testRefreshAwaitsPullBeforeManualRefresh` must still pass; it asserts that `vault-sync`
runs before `ref list`.

### 3c. `testTabsFilterBlocksArrivalRerank` (~3883, added in f7e635a, has never passed)

`FakeTabsReader(isTrusted: true)` defaults to `.failed("No fake result")`, so the wait
for `.loaded || .noDocuments` can never succeed. Script a snapshot before presenting:

```swift
reader.setNextResult(.snapshot(Self.tabsSnapshot(
  harness.library.items, vault: harness.vault, frontmost: false)))
harness.model.prepareForPresentation(source: .highlightsTabs)
```

The test's real assertion is unchanged: beta never joins the Tabs listing.

### 3d. `testRefreshReordersFromNewDataKeepingSelection`: settle the `-g` lane before ⌘R

This test failed on 5fd928b and b2f24d0 and has been dormant since. Before
`serveAddedVariant` and ⌘R, wait for the git-dates lane the same way
`testOpenRefreshesTodayWithoutRefreshingSnapshot` already does:

```swift
await harness.waitForModel { _ in
  harness.library.items.contains { $0.id == "ref/blogs/small_opened.md" && $0.added != nil }
}
```

After that, no stale publish can consume the re-rank, and the fix in 3b closes the
production side.

## Out of scope (verified already fixed or not recurring)

- **Already fixed by bd52675:**
  `RefsInspectorLoaderTests.testRevealMoreWithOnlyHighlightsEmitsRange` and
  `RefsPanelDesignTests.testRefsWidePanelToPNG` failed deterministically on c424756 and
  1d7242a, and pass on bd52675 and f7e635a.
- **Not recurring:** `CapturePanelModelTests.testBlockIDLinkDeletingWholeIDShowsChip`
  and `RefsPanelModelTests.testRepeatedTodayLinksEachKeepTheirOutcomeAndPill` failed
  once, on db5df01, and in none of the 11 test-running runs since. If either recurs on
  the proof run, file a `flake` task bead through `/sase_new_task` rather than widening
  this change.

Rules for every change: no sleeps added as fixes, and no untimed continuations or gates
in async tests (per `AGENTS.md`).

## Verification

1. **Portable check on athena:** `just portable-check` in the repo. It lints every Swift
   file and builds and tests `CaptureCore`/`RefsCore`; it must pass.
2. **fake-bob sanity check** (run under `bash`, with `HOME=/tmp PATH=/usr/bin:/bin`):
   - `Tests/Fixtures/fake-bob capture --dry-run --no-clip --format json -- x3` exits 1.
   - `... -- "=3"` exits 0.
3. **Type-check on the Mac** (best effort, per `AGENTS.md`):
   - The Mac was reachable over SSH as `mac` when this plan was written.
   - rsync a scratch copy (excluding `.build` and `.git`), run `./Scripts/bundle.sh` and
     `just typecheck-tests` there, then delete the copy.
   - Never use Bryan's checkout or `just install`.
   - If the Mac is offline, say so in the final report: the macOS targets are
     unverified.
4. **CI proof:** follow the decision below.

> [!decision] ci_proof = draft_pr Prove the change on a throwaway draft PR before
> landing. CI runs on `pull_request`.
>
> **1. Push a commit object without touching the workspace branch or index.**
>
> ```bash
> tmp_index="$(mktemp)"
> GIT_INDEX_FILE="$tmp_index" git read-tree HEAD
> GIT_INDEX_FILE="$tmp_index" git add -A
> tree="$(GIT_INDEX_FILE="$tmp_index" git write-tree)"
> commit="$(git commit-tree "$tree" -p HEAD -m "ci: verify CI fix (throwaway)")"
> git push --force origin "$commit:refs/heads/ci/verify-ci-fix"
> gh pr create --draft --base master --head ci/verify-ci-fix \
>   --title "ci: verify CI fix (throwaway)" --body "Throwaway verification PR; will be closed."
> ```
>
> **2. Watch the run.** Find the run with
> `gh run list -R bobs-org/bob-mac-capture --branch ci/verify-ci-fix`. Watch it with
> `gh run watch <id> -R bobs-org/bob-mac-capture --exit-status` in the foreground, with
> a timeout of at least 45 minutes.
>
> **3. Iterate until green.** On red, read `gh run view <id> --log-failed`, fix, then
> rebuild and force-push a new commit object. The whole `macOS 26 SwiftPM` job must be
> green, including Bundle, Verify plist and signature, Launch smoke test, and Install
> and reinstall. Those steps have not run since 2026-10-13.
>
> If a post-Test step fails for a reason unrelated to this change, and the fix is not
> clear-cut:
>
> - Stop iterating.
> - File a `ci` task bead through `/sase_new_task`.
> - Report it.
>
> **4. Clean up and land.** Run `gh pr close <n> --delete-branch`, then land the change
> through the normal SASE commit flow.

> [!decision] ci_proof = land_only Land after verification steps 1–3. In the final
> report, say that the macOS test run is unverified until the next master CI run, and
> give the command to check it: `actstat --repo bobs-org/bob-mac-capture`.

## Acceptance criteria

- The ten tests in §1–§3 are fixed as described, with the production changes limited to
  `RefsInspectorLoader.swift` (one signature) and `RefsPanelModel.swift`
  (`perform(.refresh)`), plus any README sentence the latter makes false.
- `just portable-check` passes on athena.
- The Mac type-check passes, or the report says the macOS targets are unverified.
- With `draft_pr`: the `macOS 26 SwiftPM` job is fully green on the proof PR, and the PR
  and branch are removed before landing.
