---
tier: tale
title: Fix the latent test failures behind bob-mac-capture's red CI
goal: The macOS 26 SwiftPM CI job passes again, with all five latent test failures
  fixed at their root causes.
size: medium
decisions:
  inspector_scroll:
    ask: Should Ctrl+D/U scroll the inspector's NSScrollView from the panel controller
      instead of SwiftUI?
    choices:
      appkit: Yes; synchronous AppKit scroll, passes on the Mac, reverses the original
        SwiftUI-only design
      swiftui: No; keep SwiftUI, fix only lookup and diagnostics; CI likely stays
        red another round
    default: appkit
    why: Removes the CI-only SwiftUI dependency instead of waiting on another red
      CI round
    answer: appkit
proposed_by: bbugyi200.athena.0zk.f0
decided_by: auto
status: done
---

# Fix the latent test failures behind bob-mac-capture's red CI

## Outcome and scope

Make the `macOS 26 SwiftPM` job in `bobs-org/bob-mac-capture` green again by fixing the
five test failures (11 assertions) that CI run #264 exposed on `37250d9`, then validate
the whole suite on the Mac before handing the commit back. This is bounded work for one
implementation agent: a `tale` sized `medium`.

All implementation paths are relative to the linked **bob-mac-capture** repository, not
the host bob-cli repository. Open it with `/sase_repo` before reading or editing:

```sh
sase repo open bob-mac-capture -r "Fix the latent CI test failures exposed by run #264"
```

Read the repo's `AGENTS.md` if `sase repo open` names one. Do not touch the bob-cli host
checkout.

## Why CI is still red

The previous fix (`37250d9`) was correct but incomplete in effect: from `4c22d09` (run
#256) through `a82a4ac` (run #263), **the test targets did not compile**, so nine
commits landed without CI ever executing a single test. Once `37250d9` restored
compilation, run #264 executed all 1985 tests and five latent failures surfaced. The
last green run was #255 (`131e377`).

| #   | Failing test                                                                        | Introduced by                     | Root cause                                                                                                                                                   | Kind                         |
| --- | ----------------------------------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------- |
| 1   | `CaptureModelTests.testUtf16RangeRoundTripsEmojiMarker`                             | `26aac7d`                         | `utf8Offset(in:utf16Offset:)` trusts `Range(NSRange, in:)` to return nil inside a surrogate pair; macOS 26 Foundation rounds down instead                    | product bug                  |
| 2   | `CaptureSeparatorToggleTests.testNativeUndoAndRedoDoNotRetriggerAssist`             | `26aac7d`                         | test's windowless `NSTextView` has no undo manager; behind it, the native rewrite coalesces into the typed key's undo run and corrupts redo                  | test bug + product bug       |
| 3   | `CapturePanelAgendaLayoutTests.testSmallBudgetAndExpandedOverflowStaysBounded`      | `787569d`                         | test sets `model.agendaBudget = 0` on a live controller that republishes the screen budget on every metrics report (since `27c0c2d`), so it could never pass | test design                  |
| 4   | `RefsInspectorScrollTests.testHostedControlDUScrollsInspectorWhileSearchKeepsFocus` | `4c22d09`                         | CI-only: the SwiftUI scroll pipeline never moves the inspector on the macOS 26.6.2 runner; the test's named scroll-view lookup never matches                 | environment-sensitive design |
| 5   | `RefsLibraryTests.testTriggersDuringRefreshRunExactlyOneFollowUp`                   | pre-existing test, passed in #255 | lost-update race: the snapshot pass copies git dates before awaiting the file/Spotlight sweep and publishes over a `-g` merge that lands meanwhile           | product race, timing-exposed |

### Evidence gathered during planning

The Mac (`ssh mac`, macOS 26.5, Apple Swift 6.3.2) has Command Line Tools only, so
`swift test` cannot import XCTest there. Planning built a scratch harness (see the
appendix): a minimal `XCTest` shim module plus runners that compile the real
`BobMacCaptureTests` and `CaptureCoreTests` sources, run them in XCTest's alphabetical
order, and spin the main run loop like XCTest's async waiter. Findings:

- **Reproduced at HEAD on the Mac:** failure 1
  (`XCTAssertNil failed: "3" - interior surrogate is not a caret`), failure 2 (same
  three assertions as CI), and failure 3 (`liveBudget=470` on the Mac's larger screen
  instead of CI's 323). A standalone probe confirmed
  `Range(NSRange(location: 4, length: 0), in: "Hi 🧪 …")` is non-nil on macOS 26 while
  `String.Index(utf16Index, within:)` is nil. Linux Swift 6.0.3 Foundation still returns
  nil, which is why Linux core tests never caught it.
- **Failure 2's hidden product bug:** with a window-hosted text view, the current native
  apply undoes correctly but **redo produces `Do work @file^id^`** and the history is
  corrupted. Adding `textView.breakUndoCoalescing()` before the rewrite insert gives
  undo `Do work @file:id^`, redo `Do work @file^id`, and two further undos `@file:id^` →
  `@file:id`.
- **Failure 3:** `CapturePanelController.applyContentMetrics` calls
  `updateAgendaBudget()`, which reassigns `model.agendaBudget` from screen geometry on
  every SwiftUI metrics report. The test's `0` is overwritten as soon as the re-plan
  reports new metrics.
- **Failure 4:** passes on the Mac in isolation, in CI order (after
  `testHostedControlDConsumesAtBoundaryAndWhenInspectorHidden`), and after the whole
  881-test suite. CI differs: macOS 26.6.2 (25G83), Xcode 26.6, Swift 6.3.3, a VM
  display. The exact CI mechanism is unconfirmed. Planning also found that
  `RefsPanelController.findInspectorScrollView` **never** matches on the Mac. SwiftUI's
  `.accessibilityIdentifier` does not reach an `NSView`, so the hosted tests silently
  use a "rightmost scroll view" fallback, and the narrow-case `XCTAssertNil` is vacuous.
- **Failure 5:** a deterministic gated test (below) fails on unmodified code with two
  `ref list -g` calls instead of one, which is the same extra list call CI counted, and
  passes after the fix (3/3 repeated runs).
- **Prototype result:** with every change in this plan applied (the `appkit` branch for
  failure 4), the harness ran **882/882 BobMacCaptureTests** (881 existing + 1 new) and
  **940/940 CaptureCoreTests** green on the Mac, and Linux
  `swift test --filter CaptureCoreTests` passed 940/940.

## Implementation

### 1. Validate UTF-16 carets explicitly (`Sources/CaptureCore/CaptureTextRanges.swift`)

Replace the body of `utf8Offset(in:utf16Offset:)` so it no longer depends on
`Range(NSRange, in:)` rounding behavior. This is the only `Range(_:in:)` caller in
`Sources`.

```swift
/// UTF-8 byte offset of a UTF-16 caret, or `nil` when `utf16Offset` is not
/// a character boundary (for example inside a surrogate pair) or past the
/// end of `text`.
public func utf8Offset(in text: String, utf16Offset: Int) -> Int? {
    let utf16 = text.utf16
    guard utf16Offset >= 0, utf16Offset <= utf16.count else {
        return nil
    }
    // Validate explicitly: `Range(NSRange, in:)` rounds an interior
    // surrogate down to its scalar on current Foundation.
    let position = utf16.index(utf16.startIndex, offsetBy: utf16Offset)
    guard let index = String.Index(position, within: text) else {
        return nil
    }
    return text.utf8.distance(from: text.utf8.startIndex, to: index)
}
```

`String.Index(_:within:)` checks for a character boundary, matching `stringRange`'s
existing validation. The existing test needs no change.

### 2. Make the separator swap its own undo step

In `Sources/BobMacCapture/CapturePanelController.swift`,
`applySeparatorToggleInEditableTextView`, call `breakUndoCoalescing()` immediately
before the `insertText` that applies the edit:

```swift
        // End the typed key's coalesced typing run first, so the swap is
        // its own undo step: Undo restores the literal post-key draft and
        // Redo replays exactly this replacement.
        textView.breakUndoCoalescing()
        textView.insertText(edit.replacementText, replacementRange: edit.replacementRange)
```

In `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`,
`testNativeUndoAndRedoDoNotRetriggerAssist`, host the text view in a window so it has a
real undo manager. Keep the existing assertions and add a history check:

```swift
        // Native undo lives on the window: a windowless text view has no
        // undo manager, so undo and redo would silently do nothing.
        let window = NSWindow(
            contentRect: NSRect(x: 0, y: 0, width: 480, height: 80),
            styleMask: [.titled],
            backing: .buffered,
            defer: true
        )
        window.isReleasedWhenClosed = false
        defer { window.close() }
        let textView = makeTextView("Do work @file:id")
        window.contentView?.addSubview(textView)
        window.makeFirstResponder(textView)
        textView.setSelectedRange(NSRange(location: 16, length: 0))
        XCTAssertNotNil(textView.undoManager)
        wireNativeApply(model, textView: textView)
        // … existing type / undo / redo assertions unchanged …
        // History stays intact: the swap undoes again, then the typed key.
        textView.undoManager?.undo()
        XCTAssertEqual(textView.string, "Do work @file:id^")
        textView.undoManager?.undo()
        XCTAssertEqual(textView.string, "Do work @file:id")
```

Put the two extra undos after the redo checks and before the `capture-rewrite` record
count. The README sentence "Undo restores the literal post-key draft" stays accurate.

### 3. Give the hosted zero-budget check a real seam

In `CapturePanelController.swift`, let the controller own an optional upper bound on the
budget it publishes, and apply it in `updateAgendaBudget()`:

```swift
        let value = agendaBudgetLimit.map { min($0, budget.value) } ?? budget.value
        if model.agendaBudget != value {
            model.agendaBudget = value
        }
    }

    /// Test seam: an upper bound on the published below-eye-line budget,
    /// so hosted tests can model a screen with little room below the eye
    /// line. Production leaves it nil and the screen alone decides.
    var agendaBudgetLimit: Double? {
        didSet {
            updateAgendaBudget()
        }
    }
```

In `Tests/BobMacCaptureTests/CapturePanelAgendaLayoutTests.swift`,
`testSmallBudgetAndExpandedOverflowStaysBounded`, replace `model.agendaBudget = 0` with
`controller.agendaBudgetLimit = 0` and leave every assertion as is. The model-level
`CaptureAgendaModelTests` that set `model.agendaBudget` directly have no controller and
stay unchanged.

### 4. Refs inspector Ctrl+D/U

Both branches first replace the dead accessibility-identifier lookup with a real handle.
Add an `NSViewRepresentable` marker inside the live inspector scroll content. Its
`enclosingScrollView` is exactly the inspector's scroll view.

```swift
/// Marker inside the live inspector's scroll content: its
/// `enclosingScrollView` is the inspector's native scroll view.
final class RefsInspectorScrollAnchorView: NSView {
    override func hitTest(_ point: NSPoint) -> NSView? {
        nil
    }
}

struct RefsInspectorScrollAnchor: NSViewRepresentable {
    func makeNSView(context: Context) -> RefsInspectorScrollAnchorView {
        RefsInspectorScrollAnchorView()
    }

    func updateNSView(_ nsView: RefsInspectorScrollAnchorView, context: Context) {}
}
```

Attach it as `column.background(RefsInspectorScrollAnchor())` inside the live
`ScrollView`. Leave `previewMode` static, with no anchor. Rewrite
`RefsPanelController.findInspectorScrollView(in:)` to find a
`RefsInspectorScrollAnchorView` and return its `enclosingScrollView`. Delete
`firstScrollView(in:)` and `refsInspectorScrollAccessibilityIdentifier`, with the
`.accessibilityIdentifier` modifier, once nothing uses them.

In `Tests/BobMacCaptureTests/RefsInspectorScrollTests.swift`:

- Make `inspectorScroll(in:)` use only `findInspectorScrollView`, removing the rightmost
  fallback.
- Assert `XCTAssertNotNil(RefsPanelController.findInspectorScrollView(in:))` for every
  wide (880 pt) harness. The existing narrow `XCTAssertNil` then means something.
- Append a diagnostics string to the inspector-offset assertions and the
  `waitForInspectorOffset` failure: lookup result, clip `bounds`, document height,
  `isFlipped`, and `contentInsets`. A residual CI failure must then explain itself.

> [!decision] inspector_scroll = appkit The panel controller performs the scroll; the
> SwiftUI view no longer does.
>
> - `RefsPanel.swift`: `import Combine`, hold an `AnyCancellable` subscribed in `init`
>   to `model.inspectorScrolls` (the model's subject, publisher, and `perform` stay
>   unchanged, so `RefsPanelModelTests` stay valid), and add
>   `func scrollInspector(_ direction: RefsInspectorScrollDirection)`:
>
>   ```swift
>   func scrollInspector(_ direction: RefsInspectorScrollDirection) {
>       guard let scrollView = Self.findInspectorScrollView(in: panel?.contentView),
>             let documentView = scrollView.documentView
>       else {
>           return
>       }
>       let clip = scrollView.contentView
>       let content = documentView.frame.height
>       let viewport = clip.bounds.height
>       let flipped = documentView.isFlipped
>       let offset = flipped ? clip.bounds.origin.y : content - viewport - clip.bounds.origin.y
>       var planner = RefsInspectorScrollPlanner(
>           geometry: RefsInspectorScrollGeometry(
>               offsetY: offset,
>               contentHeight: content,
>               viewportHeight: viewport,
>               topInset: clip.contentInsets.top,
>               bottomInset: clip.contentInsets.bottom
>           )
>       )
>       guard let target = planner.command(direction) else {
>           return
>       }
>       let y = flipped ? target : content - viewport - target
>       clip.scroll(to: NSPoint(x: clip.bounds.origin.x, y: y))
>       scrollView.reflectScrolledClipView(clip)
>   }
>   ```
>
>   The prototype measured `isFlipped == true` and zero insets for SwiftUI's
>   `HostingScrollView` on macOS 26.5. Keep the non-flipped arm as a guard.
>
> - `RefsInspectorView.swift`: remove `scrollCommands`, the `planner` and
>   `scrollPosition` state, and `.scrollPosition`, `.onScrollGeometryChange`,
>   `.onScrollPhaseChange`, `.onReceive`, the `.onDisappear` reset, `applyGeometry`,
>   `scroll`, `projection`, and `jump`. Drop `import Combine` if it becomes unused.
>   `RefsPanelView.swift` stops passing `scrollCommands:`.
> - Keep `RefsInspectorScrollPlanner` and its pure unit tests unchanged. Each command
>   measures live geometry synchronously, so production uses only `command`. Update the
>   planner's doc comment to say so. Do not delete the reconciliation API in this fix.
> - Behavior is unchanged: half-viewport steps, clamped at both ends, consumed at a
>   boundary and with the preview hidden, list and search focus untouched, reset per
>   reference through the existing `.id`. The README keyboard table needs no edit.

> [!decision] inspector_scroll = swiftui Keep the SwiftUI pipeline in
> `RefsInspectorView` as it is. Add only the anchor lookup, the stricter test lookup,
> and the diagnostics above. Report that CI may still fail test 4 on the 26.6.2 runner.
> Name the diagnostics to read from the next run, and give the likely candidates to
> confirm there: the geometry callback never validating the planner, or
> `ScrollPosition.scrollTo(y:)` not applying.

### 5. Close the git-date lost update (`Sources/BobMacCapture/Refs/RefsLibrary.swift`)

In `runSnapshotPass`, build the Spotlight requests from the freshly listed records.
Carry `gitAddedDates` forward only **after** `checkFilesAndSweep` returns. Everything
after that await runs synchronously on the main actor, so a `-g` merge can no longer be
overwritten:

```swift
            let response = try await fetcher.list(gitDates: false)
            var next = RefsSnapshot(
                fetchedAt: now(),
                records: response.refs,
                gitAddedDates: [:]
            )
            let root = vaultRoot()
            let requests = RefsCatalog.items(from: next).map {
                RefsSpotlightRequest(id: $0.id, url: root.appendingPathComponent($0.pdfPath))
            }
            let (missing, facts) = await Self.checkFilesAndSweep(
                requests: requests,
                fileExists: fileExists,
                spotlight: spotlight
            )
            // Carry git dates forward only after the sweep: the `-g` lane
            // can merge while this pass awaits, and reading them earlier
            // would publish over that merge and re-run the `-g` pass.
            let paths = Set(response.refs.map(\.path))
            next.gitAddedDates = (snapshot?.gitAddedDates ?? [:])
                .filter { paths.contains($0.key) }
            let items = RefsCatalog.items(from: next)
            snapshot = next
            // … rest unchanged …
```

Request ids and PDF paths come from the records, not from git dates.

Add a deterministic regression test to `RefsLibraryTests.swift`,
`testSnapshotPassKeepsGitDatesMergedDuringItsSweep`, plus file-level fakes:

- `RefsCallGate` (`@unchecked Sendable`, `NSLock`): holds its Nth `pass()` on a
  `CheckedContinuation` until `release()`. It exposes `isHolding`. A `release()` before
  the hold resumes immediately.
- `GatedSpotlight: RefsSpotlightProviding`: `await gate.pass()`, then return `[:]`.
- `ScriptedRefsFetcher: RefsFetching`: `list(gitDates: false)` returns the decoded
  `refs-list.json`; `list(gitDates: true)` counts calls (`gitListCount`), awaits
  `gitGate.pass()`, and returns `refs-list-git.json`. `plan()` returns
  `RefsPlanResponse(schemaVersion: 2)`; `show` and `scan` mirror `TimedOutRefsFetcher`.

The test uses `makeContext(spotlight: GatedSpotlight(gate: RefsCallGate(holding: 2)))`,
`setFetcher(ScriptedRefsFetcher(…, gitGate: RefsCallGate(holding: 1)))`, and these
steps:

1. Call `refresh(.manual)` and wait for `gitListCount == 1`.
2. Call `refresh(.manual)` again and wait for `sweepGate.isHolding`.
3. Release the git gate and wait for `ref/blogs/small_opened.md` to have `added`.
4. Release the sweep gate, wait for `refreshState` `.idle`, then sleep 300 ms.
5. Assert that the row still has `added` and that `gitListCount == 1`.

This test fails on unfixed code with `2 != 1`. The existing
`testTriggersDuringRefreshRunExactlyOneFollowUp` stays unchanged and becomes
deterministic once the race is closed.

## Validation

Report every check as run, pending, or unavailable, with output. Never claim CI is green
without the run for the pushed SHA.

1. **Linux, optional:** if a Swift toolchain exists, run
   `swift test --filter CaptureCoreTests` from the linked checkout. Expect 940 to pass.
   This covers fix 1 for regressions only, because Linux never reproduced the bug.
2. **Mac harness, required when the Mac is reachable:** build the appendix harness in a
   fresh scratch directory from the edited working tree, never inside the linked
   checkout. Run `Diag` with no filter, expecting all of BobMacCaptureTests to pass,
   including the new race test. Run `DiagCore` with no filter, expecting all of
   CaptureCoreTests to pass. Also run the focused set three times:
   `.build/debug/Diag CaptureSeparatorToggleTests CapturePanelAgendaLayoutTests RefsInspectorScrollTests RefsLibraryTests RefsPanelModelTests`.
3. **Mac toolchain checks:** from a scratch copy, run `./Scripts/xcode-swift.sh build`
   and `just format-lint`, or the CI lint command
   `./Scripts/xcode-swift.sh format lint --recursive Package.swift Sources Tests`. Fix
   any lint findings in touched lines.
4. **Before finishing:** run `git diff --check` and review the diff for leftover
   diagnostics or prototype `print`s.
5. **CI:** the host commits after the turn, so CI for the new SHA is pending. Tell the
   user to confirm with `actstat -n 1` and
   `gh run view <run> -R bobs-org/bob-mac-capture --log-failed` that `macOS 26 SwiftPM`
   passes end to end: tests, bundle, signature, launch, and install checks.

Access notes: the Mac is best-effort, online only with the lid open, and has no
`timeout` binary. If `ssh mac` cannot resolve inside the agent sandbox, use
`ssh -F ~/.ssh/tailnet.conf mac`. If the Mac stays offline, say the macOS validation is
pending rather than skipping it silently.

## Follow-ups (file via `/sase_new_task` only if still warranted)

- Commit a supported way to run the XCTest suites on a CLT-only Mac. The appendix
  harness proved the need: this regression hid behind nine red runs that nobody could
  reproduce locally.
- If `inspector_scroll = appkit`: simplify `RefsInspectorScrollPlanner` now that its
  reconciliation API, meaning the pending target, `applyGeometry`, and
  `beginUserScroll`, is test-only.

## Appendix: CLT-only XCTest harness (scratch only, never committed)

Copy the working tree, without `.build` or `.git`, to a fresh Mac directory such as
`/tmp/bmc-ci-<id>`. Then overlay these files. The overlay replaces `Package.swift` in
the scratch copy only.

`Package.swift`:

```swift
// swift-tools-version: 6.0
import PackageDescription

let v5: [SwiftSetting] = [.swiftLanguageMode(.v5)]

let package = Package(
    name: "BobMacCaptureDiag",
    platforms: [.macOS("26.0")],
    targets: [
        .target(name: "CaptureCore", swiftSettings: v5),
        .target(name: "RefsCore", dependencies: ["CaptureCore"], swiftSettings: v5),
        .target(
            name: "BobMacCapture",
            dependencies: ["CaptureCore", "RefsCore"],
            exclude: ["BobMacCaptureMain.swift"],
            swiftSettings: v5
        ),
        .target(name: "XCTest", path: "Diag/XCTestShim", swiftSettings: v5),
        .executableTarget(
            name: "Diag",
            dependencies: ["BobMacCapture", "CaptureCore", "RefsCore", "XCTest"],
            path: "Tests",
            exclude: ["Fixtures", "CaptureCoreTests", "RefsCoreTests",
                      "BobMacCaptureInstallHelperTests", "DiagCoreMain"],
            sources: ["BobMacCaptureTests", "DiagMain"],
            swiftSettings: v5
        ),
        .executableTarget(
            name: "DiagCore",
            dependencies: ["CaptureCore", "XCTest"],
            path: "Tests",
            exclude: ["Fixtures", "BobMacCaptureTests", "RefsCoreTests",
                      "BobMacCaptureInstallHelperTests", "DiagMain"],
            sources: ["CaptureCoreTests", "DiagCoreMain"],
            swiftSettings: v5
        ),
    ]
)
```

`Diag/XCTestShim/XCTestShim.swift` must start with `@_exported import Foundation`,
because real XCTest re-exports Foundation. It defines:

- `nonisolated(unsafe) public var shimFailureCount = 0`.
- `public func _shimRecord(_ message: String, file: StaticString, line: UInt)`, which
  increments the counter and prints `FAIL file:line: message`.
- `open class XCTestCase: NSObject`, with `override init()`, `open func setUp()`,
  `tearDown()`, `setUpWithError() throws`, and `tearDownWithError() throws`, all empty.
- `struct XCTSkip: Error` with `init(_ message: String? = nil, file:line:)`.
- `XCTFail`, `XCTAssert`, `XCTAssertTrue/False`, `XCTAssertNil/NotNil` taking `Any?`,
  `XCTAssertEqual` for `Equatable` and for `FloatingPoint` with `accuracy:`,
  `XCTAssertNotEqual`,
  `XCTAssertGreaterThan/GreaterThanOrEqual/LessThan/LessThanOrEqual` for `Comparable`,
  `XCTUnwrap` (throws), and `XCTAssertThrowsError` with an error handler.

Every expression parameter in those functions is `@autoclosure () throws -> T`, every
message is `@autoclosure () -> String = ""`, and the trailing parameters are
`file: StaticString = #filePath, line: UInt = #line`. A thrown expression records a
failure instead of propagating.

`Tests/DiagMain/main.swift` and `Tests/DiagCoreMain/main.swift` each contain the same
runner, with only the generated case list differing:

```swift
import AppKit
import Foundation
import XCTest

@MainActor
struct DiagCase {
    let name: String
    let run: @MainActor () async throws -> Void
}

@MainActor
func diagCase<T: XCTestCase>(
    _ make: @escaping @MainActor () -> T,
    _ name: String,
    _ body: @escaping @MainActor (T) async throws -> Void
) -> DiagCase {
    DiagCase(name: "\(T.self).\(name)") {
        let test = make()
        try test.setUpWithError()
        test.setUp()
        defer {
            test.tearDown()
            try? test.tearDownWithError()
        }
        try await body(test)
    }
}

@MainActor
func allCases() -> [DiagCase] {
    [
        // generated: diagCase({ Class() }, "testName") { try await $0.testName() },
    ]
}

let filters = Array(CommandLine.arguments.dropFirst())
_ = NSApplication.shared
Task { @MainActor in
    var failed: [String] = []
    var ran = 0
    for testCase in allCases().sorted(by: { $0.name < $1.name })
    where filters.isEmpty || filters.contains(where: { testCase.name.contains($0) }) {
        ran += 1
        print("RUN \(testCase.name)")
        fflush(stdout)
        let before = shimFailureCount
        do {
            try await testCase.run()
        } catch let skip as XCTSkip {
            print("  SKIP \(skip)")
        } catch {
            _shimRecord("threw \(error)", file: #filePath, line: #line)
        }
        let ok = shimFailureCount == before
        print(ok ? "  PASS" : "  FAILED")
        fflush(stdout)
        if !ok {
            failed.append(testCase.name)
        }
    }
    print("SUMMARY ran=\(ran) failed=\(failed.count)")
    for name in failed {
        print("  FAILED \(name)")
    }
    exit(failed.isEmpty ? 0 : 1)
}
while true {
    _ = RunLoop.main.run(mode: .default, before: Date().addingTimeInterval(0.05))
}
```

Generate the case list per test directory with a small script. For each `*.swift` file,
track the current `class X: XCTestCase`, and for each `func testY()` under it emit
`diagCase({ X() }, "testY") { try await $0.testY() },`. Keep test files at their
original paths so `#filePath`-based fixture lookups such as `Tests/Fixtures/fake-bob`
still resolve.

Build and run with `xcrun swift build --product Diag`, then
`.build/debug/Diag [name substrings…]`. Do the same for `DiagCore`. The harness checks
assertions and run-loop behavior; it does not replace CI's real XCTest, signing, or
launch checks.
