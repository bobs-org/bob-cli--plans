---
tier: tale
title: Fix the bob-mac-capture just install build break
goal:
  Make `just install` build again by returning the Highlights tab buttons from their
  closure, and verify on macOS.
size: small
decisions:
  fix_test_compile:
    ask:
      Also fix the RefsInspectorTests async-assert compile error so `just
      typecheck-tests` passes?
    choices:
      include:
        Hoist the three awaited asserts into locals; the mac type-check gate and CI test
        build pass
      skip:
        Leave the test file; typecheck-tests still fails on lines 483-485 and the
        follow-up bead owns it
    default: include
    why: Without it the required mac type-check gate cannot pass
    answer: include
proposed_by: bbugyi200.athena.10n
decided_by: reviewer
decided_via: tui
create_time: 2026-10-16 09:04:08
status: wip
---

# Plan: Fix the `just install` build break in bob-mac-capture

All work happens in the linked `bob-mac-capture` repository. Open it with the
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), use only the path it
prints, and read its `AGENTS.md` before editing. Its changes need a `commit` decision in
the final declaration.

## Root cause (already diagnosed)

`just install` runs `Scripts/install.sh` → `Scripts/bundle.sh`, which does a release
`swift build --product BobMacCapture`. That build fails with exactly two errors, both in
`Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`, which was added in `1e2a9bd`
("feat(refs): wire Highlights Tabs entry point and settings"):

- `198:45 cannot convert value of type 'Any' to expected argument type 'AXUIElement'`
- `223:33 cannot convert value of type '[Any]' to expected argument type '[AXUIElement]'`

The real defect is a few lines up, in `readSnapshot(_:deadline:)` (lines 174-178):

```swift
let tabButtons = try tabGroup.flatMap {
    setMessagingTimeout(on: $0)
    try optionalAttribute($0, "AXTabs" as CFString, deadline: deadline)
        as? [AXUIElement]
} ?? []
```

This is a multi-statement closure with no `return`. Implicit return only applies to
single-expression closures, so the `optionalAttribute(...) as? [AXUIElement]` value is
discarded. The build log shows this as the warning
`177:21 expression of type '[AXUIElement]?' is unused`. Nothing then pins the closure's
result type, so `?? []` gives the empty literal its default type `[Any]`. That makes
`tabButtons` an `[Any]`, and both of its uses fail: `setMessagingTimeout(on: button)` at
line 198 and `WindowRecord(tabButtons:)` at line 223. The "missing return" error you
would expect never appears, because Swift reports it in a SIL pass that only runs after
type-checking succeeds.

Evidence:

- GitHub Actions run `38899044286` (Xcode 26.6, macOS 26.5 SDK) fails its **Build** step
  with the same two errors, so this is not specific to Bryan's Command Line Tools setup.
- A standalone `swiftc -typecheck` repro with the same closure shape and a stand-in
  class gives the same unused-expression warning and `[Any]` errors. Adding `return`
  plus an explicit `[Element]` annotation makes it type-check with no diagnostics.
- The other warnings in Bryan's log (actor isolation, `#selector`, deprecated `Text +`,
  and so on) are only warnings. They are not part of this failure and are out of scope.

## Changes

### 1. Return the tab buttons from the closure

In `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`, replace lines 174-178
with:

```swift
let tabButtons: [AXUIElement] = try tabGroup.flatMap { group in
    setMessagingTimeout(on: group)
    return try optionalAttribute(group, "AXTabs" as CFString, deadline: deadline)
        as? [AXUIElement]
} ?? []
```

- The explicit `return` is the fix.
- The `[AXUIElement]` annotation makes any future regression fail at this line, with a
  clear missing-return or type error, instead of two downstream `Any` errors.
- The named `group` parameter matches the `button in` closure style a few lines below.
- Do not change any other behaviour. The function already handles an empty or missing
  `AXTabs` value (the `tabButtons.isEmpty` branch).

### 2. Make the test target compile again

> [!decision] fix_test_compile = include

`Tests/BobMacCaptureTests/RefsInspectorTests.swift`, in
`testIntrinsicsDiscardDropsInFlightAndAllowsFutureLoads` (lines 483-485), passes `await`
expressions straight into `XCTAssertEqual`. XCTest's autoclosures are synchronous, so
both real XCTest (CI runs `38825142280` through `38895720736`) and the repo's
`Scripts/typecheck/XCTest.swift` stub reject those lines with "'async' call in an
autoclosure that does not support concurrency" and "capturing 'async let' variables is
not supported". Hoist each awaited value into a local before asserting, and keep the
same order of awaits so the test's in-flight and discard semantics and its
`counter.count == 2` expectation are unchanged:

```swift
let firstPageCount = await first.pageCount
XCTAssertEqual(firstPageCount, 7)
let secondPageCount = await loader.intrinsics(for: pdf, id: "roundtrip").pageCount
XCTAssertEqual(secondPageCount, 7)
let thirdPageCount = await loader.intrinsics(for: pdf, id: "roundtrip").pageCount
XCTAssertEqual(thirdPageCount, 7)
```

Test files use two-space indentation; match it.
`grep -rnE 'XCT[A-Za-z]+\((try )?await ' Tests` currently matches only these three
lines. Rerun it after the edit and expect no matches.

> [!decision] fix_test_compile = skip

Leave `RefsInspectorTests.swift` untouched. In step 4, expect `just typecheck-tests` to
fail only on `RefsInspectorTests.swift:483-485`, and say so in the final report. Include
that compile error in the step 3 bead.

### 3. File the pre-existing CI test failures as a task bead

CI on `master` has been red on every push since at least 2026-10-13, which is part of
why this build break went unnoticed. Use `/sase_new_task` (it de-duplicates) to record
one task bead in the `bob-mac-capture` context, typed `ci`, default size `large`, that
cites these observed failures:

- Run `38746202123`: xctest crashed with signal 5,
  `Swift/ContiguousArrayBuffer.swift:692: Fatal error: Index out of range` during
  `RefsPanelModelTests.testQueuedDigitCopiesWhenTheShowLoaderReturns`.
- Runs `38750465645`, `38807458658`, `38813536247`: assertion failures in
  `RefsInspectorScrollTests.testHostedShortContentRevealsWithNoOffsetChange`,
  `RefsPanelModelTests.testQueuedDigitCopiesWhenTheShowLoaderReturns`, and
  `CaptureTemplatePresentationTests.testTemplateRowNeedIsQuietAndUsesBobsDiagnostic`
  (timeout).
- Run `38819681612`: `BobMacCaptureTests.testTemplatePlainNewlineInsertsOnlyNewline` and
  `CapturePanelModelTests.testTemplateHeaderCandidateReachableWithPrimedCache`.

Do not try to fix these in this tale. Their root causes are unknown, and some may be CI
timing flakes.

## Verification

Follow the repo's `AGENTS.md`. Never land Swift you have not compiled.

1. **Linux gate (athena):** run `just portable-check` in the opened repo. It parse-lints
   every Swift file, including the two edited ones, and builds and tests
   `CaptureCore`/`RefsCore`. It cannot type-check `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`.
2. **macOS compile (best effort, `ssh mac`):** the mac is reachable only when powered on
   with its lid open, and it was offline while this plan was written. Read the `tailnet`
   reference memory with `/sase_memory_read` for access details. If it is reachable:
   - rsync the opened repo to a fresh scratch directory on the mac, excluding `.build`
     and `.git`;
   - in that copy run `./Scripts/bundle.sh --identity "-"`, which is the same release
     `BobMacCapture` build `just install` runs;
   - run
     `./Scripts/xcode-swift.sh build --configuration release --product BobMacCaptureInstallHelper`,
     the other build `install.sh` performs;
   - run `just typecheck-tests`;
   - delete the scratch copy.

   Expect the bundle build to finish with no `error:` lines. Expect
   `just typecheck-tests` to print PASS for all four test targets under the `include`
   branch. The `AXHighlightsTabsReader.swift:177` unused-expression warning must be
   gone. Never use Bryan's own checkout, and never run `just install` on the mac.

3. If the mac is offline, say plainly in the final report that the macOS targets are
   unverified. Then tell Bryan that rerunning `just install` on the mac, and CI's
   **Build** step on the pushed commit, are the confirmation.

## Out of scope

- The compiler warnings in the install log.
- Fixing the CI test-runtime failures (step 3 only files them).
- Any change to `install.sh`, `bundle.sh`, or the CI workflow.
