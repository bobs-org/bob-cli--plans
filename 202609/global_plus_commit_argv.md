---
tier: epic
title: Fix global plus-commit capture-complete argv
goal: Bob Mac Capture's swift test suite passes on macOS 26, including the global
  route plus-commit that must request task completion at cursor 12.
parent_bead: bob-cli-26.4
phases:
- id: fix-global-plus-commit
  title: Fix global plus-commit capture-complete argv
  depends_on: []
  size: small
  description: 'fix-global-plus-commit: make the multiline global route plus-commit
    record capture-complete at cursor 12 and leave swift test green on macOS 26.'
proposed_by: bbugyi200.apollo.bob-cli-26.4.land
create_time: 2026-09-26 18:20:13
status: wip
bead_id: bob-cli-26.4.3
---

- **PROMPT:** [prompts/202609/global_plus_commit_argv.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/global_plus_commit_argv.md)
- **PARENT:** [202609/pomodoro_mac_verification.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/pomodoro_mac_verification.md)
- **BEAD:** [bob-cli-26.4.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-26/bob-cli-26.4.3.md)

# Fix global plus-commit capture-complete argv

## Context

Epic `bob-cli-26.4` repaired Pomodoro diagnostic range decoding and source-checked the
Mac session preview. Its land audit found one remaining failure before that epic can
close.

Open `bob-mac-capture` with `sase repo open gh:bobs-org/bob-mac-capture`. The checkout
reviewed for this plan was `7282a7a`
(`fix(mac-capture): repair Pomodoro diagnostic range decoding`, bead `bob-cli-26.4.1`).
`bob-cli` has no commits after `bob-cli-26.4` started. `bob-mac-capture` has none after
`7282a7a`. There is no separate integration edit in either tree.

`CaptureDiagnostic.init(from:)` flattens `(try? decodeIfPresent(...)) ?? nil` and
accepts an object range, a two-element `[start, end]` pair, an absent range, null, and
malformed ranges. `testCaptureDiagnosticDecodesAllRangeShapesTolerantly` covers those
shapes. Rust still serializes diagnostic ranges as a nullable pair. Panel session text
and accessibility labels already read `CapturePomodoroStartPresentation` in
`Sources/BobMacCapture/CapturePanelView.swift`.

GitHub Actions run
[36274365429](https://github.com/bobs-org/bob-mac-capture/actions/runs/36274365429) on
commit `7282a7a`:

- Lint and `./Scripts/xcode-swift.sh build` succeeded.
- `./Scripts/xcode-swift.sh test` executed 503 tests with 1 failure.
- The only failure is `testPlusCommitsGlobalRouteDeclarationAndOpensTaskPicker` at
  `Tests/BobMacCaptureTests/CapturePanelModelTests.swift:1911` (`XCTAssertTrue failed`).
- That run did not report `Condition not met before timeout`, so the wait for
  `completionResponse?.context == "task"` returned.
- The asserts immediately above passed: the draft is `@@mac_inbox+` plus a newline plus
  `First task`, and `collapsedSelectionUTF8Offset()` is 12.
- Commit `54861b3` had a green CI run. Commit `7fae3fd` failed at compile time and never
  ran tests. `7282a7a` is the first test run of this behavior.

Host `mac` (`xcode-select` prints `/Library/Developer/CommandLineTools`, Swift 6.3.2)
compiles `CaptureCore` and then fails tests with `no such module 'XCTest'`. Reproduce
with `./Scripts/xcode-swift.sh test` on the GitHub `macos-26` runner or any selected
Xcode 26+ toolchain that provides XCTest.

`Tests/Fixtures/fake-bob` returns `"context": "task"` for draft
`@@mac_inbox+\nFirst task` at every cursor. A `capture-complete` call with that draft
and a cursor other than 12 still satisfies the context wait and fails only this argv
assertion:

```text
argv=capture-complete --all-tasks --cursor 12 --format json -- @@mac_inbox+
First task
```

The newline in that expected substring is a real line feed. Other tests on the same CI
run already assert multiline `argv=` records, including `Restored café @Cash\n- child`,
and those passed. The failing call is not using cursor 12 with that exact draft, even
though the selection offset is 12 when the test reads it.

`commitRouteCompletionOnPlus` rewrites `@@ma\nFirst task` to `@@mac_inbox+\nFirst task`
and calls `scheduleAnalysis(cursorUTF8Offset: caret, requestCompletion: true)` with
`caret == 12`. Single-line plus-commit tests passed on the same run. Find which later or
different `capture-complete` invocation wins for this multiline global draft. A strong
lead is a second `scheduleAnalysis` that cancels the cursor-12 task before `fake-bob`
appends the record; the fixture ignores cursor, so the replacement call can still report
context `task`.

The phase-2 proposal to install full Xcode on host `mac` is not part of this epic. It is
host administration, the matching task type is not agent-creatable, and CI's `macos-26`
runner already executes `swift test`.

## Phase: fix-global-plus-commit

slug: fix-global-plus-commit size: small depends_on: []

1. Reproduce `testPlusCommitsGlobalRouteDeclarationAndOpensTaskPicker` with
   `./Scripts/xcode-swift.sh test --filter testPlusCommitsGlobalRouteDeclarationAndOpensTaskPicker`
   on a macOS 26 XCTest toolchain. On failure, capture the `FAKE_BOB_RECORD_PATH`
   contents so the actual `capture-complete` cursor and draft are visible.
2. Fix the plus-commit path so accepting the cached global route records exactly
   `capture-complete --all-tasks --cursor 12 --format json --` followed by
   `@@mac_inbox+\nFirst task`. Keep the caret immediately after `+`. Do not change
   single-line plus-commit behavior.
3. Keep the diagnostic decoder tolerant of object ranges, two-element pairs, absent
   ranges, null, and malformed ranges. Do not add a Swift-side clock or vault writer.
4. Run the full `./Scripts/xcode-swift.sh test` suite and leave it green, including
   `testCaptureDiagnosticDecodesAllRangeShapesTolerantly` and the existing plus-commit
   tests.

## Done when

- `testPlusCommitsGlobalRouteDeclarationAndOpensTaskPicker` passes.
- `./Scripts/xcode-swift.sh test` passes on macOS 26 for the updated `bob-mac-capture`
  checkout.
- The version-one capture JSON contract and the tolerant diagnostic-range decoder stay
  in place aside from whatever the argv fix requires in the Mac app or its fake-bob
  fixture.
