---
tier: tale
title: Fix the bob-mac-capture build that breaks just install
goal:
  bob-mac-capture's BobMacCapture target and all test targets compile on macOS again, so
  just install and the CI Build step pass.
size: small
proposed_by: bbugyi200.athena.16v
proposed_with: claude/opus
create_time: 2026-11-02 06:02:55
status: wip
---

# Plan: Fix the bob-mac-capture build that breaks `just install`

## Problem

`just install` in `bob-mac-capture` fails while compiling the `BobMacCapture` target. CI
on `master` has failed its **Build** step on every push since `6f13e48`, so nothing
after `01c753b` has compiled. Bryan's install pulled `01c753b → 0f2f46f` and hit the
same errors. Of the many diagnostics, only three are errors; the rest are pre-existing
warnings that do not fail the build. CI run `41422994551` on `0f2f46f` shows exactly
these three:

1. **`Sources/BobMacCapture/CapturePanelView.swift` ~3639**: "closure containing control
   flow statement cannot be used with result builder 'ViewBuilder'". Introduced by
   `90bd9ba` (feat(capture): add block presentation and pull-in choices). That commit
   added a `for relationComplete in success.relationCompletes` loop with
   `guard … else { continue }` inside the `@ViewBuilder func previewItem(…)`. SwiftUI's
   `ViewBuilder` has no `buildArray`, so it allows neither `for` loops nor `continue`.
2. **`Sources/BobMacCapture/CapturePanelView.swift` ~4890**: "function declares an
   opaque return type, but has no return statements in its body". Introduced by
   `6f13e48` (fix(mac-capture): resolve Calendar band landing issues). That commit put a
   `let displayHeadline: Text` and an `if/else` that assigns it in front of the `VStack`
   in `private func eventWarningBanner(…) -> some View`. The function is not a
   `@ViewBuilder`, so once its body has more than one statement it needs an explicit
   `return`.
3. **`Sources/BobMacCapture/NotificationService.swift` ~854**: "the compiler is unable
   to type-check this expression in reasonable time". Introduced by `90bd9ba`, which
   added a third
   `+ capture.relationCompletes.compactMap(\.taskComplete).flatMap(\.keepMoving)`
   operand to the closure inside `captures.flatMap { capture in … }.filter { … }` in
   `pulledInSummary(captures:)`. The solver has to infer the closure's return type
   across the `flatMap` overloads, three `+` overloads, `??`, and key-path
   `compactMap`/`flatMap`, and that blows the time budget.

None of these is a toolchain regression. The commits landed Swift that was never
compiled on macOS. CI is red, and the Mac Command Line Tools toolchain (Swift 6.3.2,
macOS 26.5) reports the same errors.

Separate and out of scope: before `6f13e48`, CI was already red in its **Test** step,
from a test-target compile error and then a 15-minute xctest watchdog hang. `6f13e48`
already changed the test-target error (`CaptureAgendaDesignTests` now uses
`\._colorSchemeContrast`), but no CI run has compiled the tests since, because Build
fails first. `just install` does not run tests. Do not chase the hang here, but do
confirm that the test targets type-check (see Verification).

## Changes

All paths are relative to the `bob-mac-capture` linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<reason>"`) and read its `AGENTS.md` first. Format
only the lines you change (4-space indent, 100 columns); do not re-indent whole files.

### 1. CaptureCore: one accessor per offer-row kind

`taskComplete.unblocked/keepMoving`, `pomodoroClose.unblocked/keepMoving`, and each
`relationCompletes[].taskComplete` are concatenated by hand at three call sites with the
same `+`/`??` chain. One of those chains is the expression the solver gave up on.
Replace the chains with explicitly typed, imperative accessors in the portable
`CaptureCore` target. That way they compile and get tested on Linux, and the next offer
source becomes a one-line change.

Add a small new file `Sources/CaptureCore/CaptureSuccessorOfferRows.swift`:

```swift
extension CaptureCommandSuccess {
    /// Unblocked successor rows from every close this capture carries, in
    /// preview order: `task_complete`, then `pomodoro_close`, then each
    /// `relation_completes` close.
    public var allUnblockedRows: [TaskCompleteUnblocked] {
        var rows: [TaskCompleteUnblocked] = taskComplete?.unblocked ?? []
        rows.append(contentsOf: pomodoroClose?.unblocked ?? [])
        for relationComplete in relationCompletes {
            rows.append(contentsOf: relationComplete.taskComplete?.unblocked ?? [])
        }
        return rows
    }

    /// Keep-moving offer rows from the same closes, in the same order.
    public var allKeepMovingRows: [TaskCompleteUnblocked] {
        // same shape, reading `keepMoving`
    }
}
```

The order matters because `pulledInSummary` names the first-seen project. Keep it
exactly as the current chains have it.

### 2. `NotificationService.pulledInSummary(captures:)`

- Count closure (~lines 836–841): replace the six-operand `let rows = … + …` chain with
  `let rows = capture.allUnblockedRows + capture.allKeepMovingRows`. The chain still
  compiles today, but it is the same slow pattern with more operands.
- Failing expression (~line 854): replace it with
  `let keepMovingRows: [TaskCompleteUnblocked] = captures.flatMap(\.allKeepMovingRows).filter { $0.offer?.selected == true }`,
  wrapped to 100 columns.

Leave the rest of the function unchanged.

### 3. `CapturePanelView.swift`

- **`previewContent(_:)` (~line 3449)**: replace the identical three-operand
  `keepMovingRows` chain with `captures.flatMap(\.allKeepMovingRows)`. This is
  behavior-preserving deduplication.
- **`previewItem(…)` (~line 3639)**: replace the `for … { guard … continue }` loop with
  a `ForEach`. This follows the house pattern already used in
  `Refs/RefsCopyCardView.swift` and `Refs/RefsLinkPreviewWell.swift`:

  ```swift
  ForEach(
      Array(success.relationCompletes.compactMap(\.taskComplete).enumerated()),
      id: \.offset
  ) { _, complete in
      pullInPreviewSummary(
          for: complete.unblocked + complete.keepMoving,
          pullIn: success.pullIn
      )
  }
  ```

  It renders one summary per relation complete that carries a `task_complete`, exactly
  what the loop was meant to do.

- **`eventWarningBanner(…)` (~line 4890)**: add `return` before the `VStack(…)`
  expression that follows the `displayHeadline` assignment. Do not add `@ViewBuilder`:
  the `displayHeadline = …` assignments are not valid result-builder statements.

Leave `CapturePanelModel.swift`'s `for relationComplete in capture.relationCompletes`
loop (~line 12018) alone. It is ordinary imperative code that also collects
`stillBlocked`, and it compiles.

### 4. Test

Add a CaptureCore unit test, for example in
`Tests/CaptureCoreTests/CaptureRelationCompleteTests.swift`, or next to the pull-in
decoding tests in `CapturePullInDecodingTests.swift` (copy their fixture or initializer
style). Build one `CaptureCommandSuccess` with:

- a `taskComplete` that has one unblocked and one keep-moving row,
- a `pomodoroClose` that has one of each,
- two `relationCompletes`, one with a `taskComplete` that has one of each and one whose
  `taskComplete` is `nil`.

Assert that `allUnblockedRows` and `allKeepMovingRows` return the rows in task-complete
→ pomodoro-close → relation order and skip the `nil` relation.

## Verification

Never land Swift you have not compiled, and do not skip any of these steps.

1. **Linux (portable targets, including the new accessor and test):** from the repo
   root, run `just portable-check`. This host has the `swift` toolchain and
   `swift format` 600, which the recipe falls back to.
2. **Mac (BobMacCapture and all test targets):** the Mac (`ssh mac`, user `bbugyi`) was
   reachable while this plan was written: macOS 26.5, Swift 6.3.2, `just` and `rsync`
   installed, Command Line Tools only. Following the repo's `AGENTS.md`:
   - rsync the repo to a fresh scratch directory on the Mac (for example under `/tmp/`),
     excluding `.build` and `.git`;
   - in the scratch copy, run `./Scripts/bundle.sh`. It runs the same release
     `--product BobMacCapture` build that `just install` performs, and it must succeed
     with zero errors;
   - run `just typecheck-tests` in the scratch copy. It must pass, which also confirms
     that `6f13e48`'s test-target fix compiles;
   - delete the scratch copy.

   Never touch Bryan's own checkout at `~/projects/github/bobs-org/bob-mac-capture`, and
   never run `just install` there. If the Mac is offline, say in the final report that
   the macOS targets are unverified.

3. Grep the Mac build output for `error:` to confirm that none remain. The existing
   deprecation and concurrency warnings are out of scope. Do not fix them in this
   change.

## Out of Scope

- The pre-existing `+`-on-`Text` deprecation warnings and Swift 6 concurrency warnings.
- The earlier CI **Test** watchdog hang. If Build and Test still go red for a reason
  other than these three errors once this lands, report it, and use `/sase_new_task`
  before filing a bead.
- Docs: nothing user-visible changes.
