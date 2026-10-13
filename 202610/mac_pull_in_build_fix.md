---
tier: tale
title: Fix the Mac capture build broken by the pull-in card redesign
goal: just install builds again on the Mac, and the test target compiles in CI.
size: medium
decisions:
  test_typecheck_gate:
    ask:
      Should we add a `just typecheck-tests` gate so agents can type-check test targets
      on the no-Xcode Mac?
    choices:
      add: Test compile errors are caught on the Mac before landing, not only in CI
      skip:
        Only the five errors get fixed; future test compile errors still reach master
    default: add
    why:
      No local recipe compiles the tests, so 395c521, 76faeb6 and b377146 shipped test
      errors
    answer: add
proposed_by: bbugyi200.apollo.7g.f0.f0
decided_by: auto
create_time: 2026-10-13 19:34:00
status: wip
---

# Plan: Fix the Mac capture build broken by the pull-in card redesign

## Context

Bryan ran `just install ~/Applications -` on the Mac at bob-mac-capture `b377146`
(`feat(pull-in): redesign successor pull-in choice card as picker sibling`, plan
`202610/mac_pull_in_choice_redesign.md`). The install failed with two compile errors in
`Sources/BobMacCapture/CapturePullInChoiceView.swift`. Everything else in that output is
pre-existing warnings. CI run 38477024330 for `b377146` fails at **Build** on the same
two errors.

Work in the linked repo **bob-mac-capture**. Open it with `/sase_repo`
(`bob-mac-capture`) and read its `AGENTS.md` before editing. Every path below is
relative to that repo.

## Diagnosis (already established; do not re-derive)

### Product errors: the only ones blocking `just install`

1. `CapturePullInChoiceView.swift:326`, in `projectedLinksAccessibilityLabel`:

   ```swift
   guard let capsule = choice.projectedLinksCapsuleText,
       let budget = choice.planBudget,
       let projected = choice.projectedLinkCount
   else {
       return capsule ?? ""   // error: cannot find 'capsule' in scope
   }
   ```

   A name bound by a `guard` condition is not in scope inside that guard's own `else`.
   The intent was to fall back to the capsule text, or to empty. Fix:
   `return choice.projectedLinksCapsuleText ?? ""`.

2. `CapturePullInChoiceView.swift:511`, the section header's "N marked" label:
   `.foregroundStyle(.accentColor)` fails with
   `type 'ShapeStyle' has no member 'accentColor'`. `foregroundStyle` takes a generic
   `ShapeStyle`, so leading-dot lookup only finds `ShapeStyle` statics such as `.tint`
   and `.secondary`. `accentColor` is a static on `Color`. Fix:
   `.foregroundStyle(Color.accentColor)`. That matches the repo convention
   (`CapturePickerView.swift:523`, `BlockDiffCard.swift:190`) and the same file's lines
   554, 703 and 784.

These two fixes were verified on a scratch copy on the Mac with only those edits
applied. `./Scripts/xcode-swift.sh build` printed `Build complete!` and
`./Scripts/bundle.sh --identity -` exited 0. `install.sh` builds through `bundle.sh`, so
`just install` works once these land.

### Test-target errors: masked by the failed product build

The test target has not compiled since `76faeb6`. The CI run for `76faeb6` (38473639929)
failed at **Test** on the first error below. The other two were added by `b377146` and
found by checking the source against the real declarations.

3. `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift:160-161`:
   `CaptureKeyRoutingContext(pullInChoiceVisible: true, pickerVisible: true, completionVisible: true)`
   fails with
   `argument 'completionVisible' must precede argument 'pullInChoiceVisible'`.
   Memberwise-init labels must follow the declaration order in
   `Sources/BobMacCapture/CaptureKeyCommandRouter.swift:61`. Fix:
   `CaptureKeyRoutingContext(completionVisible: true, pullInChoiceVisible: true, pickerVisible: true)`.
4. `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift:395-398`, the `causedOffer`
   helper in `testContextTextCoversOneThreeAndNoCauses`, passes `unblockedBy:` after
   `notLinked:`. `TaskCompleteUnblocked.init`
   (`Sources/CaptureCore/CaptureModels.swift:3711`) declares
   `… statusName, inbox, unblockedBy, link, notLinked, offer, trigger, hideRemoved`.
   Fix: move the `unblockedBy: [...]` argument directly above
   `notLinked: "not_selected"`.
5. `Tests/BobMacCaptureTests/CapturePullInChoiceDesignTests.swift:238-241`, the
   `cardOffer` helper, has the same mistake. Move `unblockedBy: cause.map { ... } ?? []`
   directly above `notLinked: "not_selected"`. Keep `offer:` and `hideRemoved:` after
   it.

A label-order scan of every `CaptureKeyRoutingContext(` and `TaskCompleteUnblocked(`
call in `Sources/` and `Tests/` found no other misordered calls. A source audit of every
symbol the `76faeb6` and `b377146` test code uses found no other errors. Tracing the new
assertions through the implementation predicts they all pass. Both checks were done by
reading, not by compiling. CI is the authority for test results.

### Why it reached master

- The `b377146` coder could not reach the Mac. `just portable-check` cannot type-check
  `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, and `swiftc -parse` only parses.
  `AGENTS.md` allows landing with a "macOS targets unverified" note, so the change
  landed.
- The recipe `AGENTS.md` gives for the test targets,
  `xcode-swift.sh build --build-tests`, can never succeed on the Mac. It has Command
  Line Tools only: `xcode-select -p` is `/Library/Developer/CommandLineTools`, with
  Apple Swift 6.3.2 and the macOS 26.5 SDK. Command Line Tools ship `Testing.framework`
  but no XCTest, so every test target fails with `no such module 'XCTest'`. Test code is
  therefore never compiled before CI. The `76faeb6` report said its tests "compile", but
  they had not been compiled.

### Separate, pre-existing, and out of scope

Since `2837025` (refs-inspector notes paging), CI's **Test** step has also failed
deterministically in `RefsInspectorScrollTests`, on `testHostedNotesAtBottomRevealPages`
and `testHostedShortContentRevealsWithNoOffsetChange`. The values were identical at
`2837025` (run 38438769006) and `6c4209b` (run 38464978866):

- `:392`: 6 ≠ 8
- `:396`: 1 ≠ 2
- `:422`: 3 ≠ 6
- `:632`/`:652`: "inspector offset did not change/settle", with `docHeight=239.0` inside
  a `405×462` clip

These failures are independent of the pull-in card. After this fix, CI should go green
through **Build** and stay red at **Test** until they are fixed. **Bundle** and the
install gates stay skipped behind **Test**. Step 3 files them as a bead.

## Changes

### 1. Fix the five compile errors (always)

Make exactly the five edits listed in the Diagnosis and nothing else in those files.
Keep the surrounding formatting: test files use 2-space indents, source files use
4-space indents.

### 2. Test type-check gate

> [!decision] test_typecheck_gate = add
>
> Add a type-check-only path for the macOS test targets on the Mac, which has Command
> Line Tools but no Xcode:
>
> - **`Scripts/typecheck/XCTest.swift`** is a stand-in for XCTest. Its header comment
>   says it is never linked or run. It declares exactly the XCTest API the four test
>   targets use (grep-verified), with real XCTest signatures:
>   - expressions take the form `@autoclosure () throws -> T`
>   - messages take the form `_ message: @autoclosure () -> String = ""`
>   - every function ends with `file: StaticString = #filePath, line: UInt = #line`
>
>   The API to declare:
>   - `open class XCTestCase: NSObject`, nonisolated like the real one, with `open`
>     `setUp()`, `tearDown()`, `setUpWithError() throws` and
>     `tearDownWithError() throws`
>   - `public struct XCTSkip: Error` with `init(_ message: String? = nil, file:, line:)`
>   - `XCTFail`, `XCTAssert`, `XCTAssertTrue`, `XCTAssertFalse`, `XCTAssertNil` and
>     `XCTAssertNotNil`
>   - `XCTAssertEqual`, both the `Equatable` form and `accuracy:` overloads for
>     `FloatingPoint` and `Numeric`
>   - `XCTAssertNotEqual`
>   - `XCTAssertGreaterThan`, `XCTAssertGreaterThanOrEqual`, `XCTAssertLessThan` and
>     `XCTAssertLessThanOrEqual`, all on `Comparable`
>   - `XCTAssertThrowsError`, with a defaulted trailing `(_ error: Error) -> Void`
>     handler
>   - `XCTAssertNoThrow`
>   - `XCTUnwrap`, which `throws -> T`
>
>   Bodies are empty, or `fatalError()` where a value must be returned. If a target
>   type-checks only after you widen the stub, widen it to match the real XCTest
>   signature. Never loosen the stub just to silence an error.
>
> - **`Scripts/typecheck-tests.sh`**, written in the same style as `install.sh`:
>   - Use `set -euo pipefail` and a `--help` usage block, and resolve the package root
>     from `BASH_SOURCE`.
>   - On a non-macOS host, fail with exit 69 and a message.
>   - Run `Scripts/xcode-swift.sh build`. That is a debug build, and SwiftPM debug
>     builds enable testable imports. Take the module directory from
>     `$(Scripts/xcode-swift.sh build --show-bin-path)/Modules`.
>   - If real XCTest is importable, run `Scripts/xcode-swift.sh build --build-tests` and
>     exit with its status. Probe by type-checking a one-line `import XCTest` file with
>     the same toolchain. This is the full-Xcode and CI path.
>   - Otherwise, emit the stub with
>     `swiftc -emit-module -parse-as-library -module-name XCTest -swift-version 5 -target "$(uname -m)-apple-macosx26.0"`
>     into `.build/xctest-typecheck-stub/`.
>   - Then type-check each test target from `Package.swift` (`BobMacCaptureTests`,
>     `BobMacCaptureInstallHelperTests`, `CaptureCoreTests`, `RefsCoreTests`) with
>     `swiftc -typecheck -swift-version 5 -module-name <Target> -target … -I <stub dir> -I <modules dir>`
>     over `find Tests/<Target> -name '*.swift'`.
>   - Prefer one whole-module frontend per target (`-wmo`). If the driver rejects `-wmo`
>     with `-typecheck`, use `-j "$(sysctl -n hw.ncpu)"` instead.
>   - Resolve `swiftc` through the same
>     `/usr/bin/xcrun --sdk macosx --toolchain default` that `xcode-swift.sh` uses,
>     after its build has validated the tools.
>   - Run the targets in sequence and print one pass/fail line per target. End with a
>     note that this was a type-check against a stub and CI still runs the tests. Exit 1
>     if any target failed.
> - **`justfile`**: add `typecheck-tests:` running `./Scripts/typecheck-tests.sh`, next
>   to `portable-check`.
> - **`AGENTS.md`**: in the scratch-copy recipe, replace
>   `xcode-swift.sh build --build-tests` with `just typecheck-tests`. Add one sentence:
>   the Mac has only Command Line Tools, which ship no XCTest, so `--build-tests` cannot
>   work there; `typecheck-tests` type-checks every test target against a stub, and CI
>   still runs the tests. Keep every other line.
> - **`README.md`**, under `## Development`: after the `portable-check` paragraph, add
>   one sentence that `just typecheck-tests` type-checks all test targets on a Mac with
>   only Command Line Tools.
>
> **Fallback:** take the `skip` branch instead, and file a `feature` task bead with
> `/sase_new_task` recording what failed, if any of these happens:
>
> - the stub produces false errors that matching real XCTest signatures cannot fix
> - one target's type-check runs longer than about 10 minutes
> - the Mac is unreachable for the whole session; the gate is Swift, and `AGENTS.md`
>   forbids landing Swift you have not compiled

> [!decision] test_typecheck_gate = skip
>
> Add no script. In `AGENTS.md`, replace `xcode-swift.sh build --build-tests` with
> `xcode-swift.sh build`. Add one sentence: the Mac's Command Line Tools ship no XCTest,
> so test targets compile only in CI, and a final report must say test code was not
> compiled.

### 3. File the RefsInspector CI failures as a bead (always)

Use `/sase_new_task` to file a `ci` task bead, size `large`, for the two
`RefsInspectorScrollTests` failures. Include the commit they started at, the run IDs,
and the assertion values from the Diagnosis. Check for duplicates first, as the skill
requires. Do not try to fix them in this plan.

### 4. Clean up the diagnosis scratch files on the Mac (always, best effort)

Diagnosis left these on the Mac after it went to sleep mid-command:

- the scratch copy `~/tmp/bmc-diag-1791932070`
- `~/tmp/bmc-diag-1791932070-stub.swift`
- `~/tmp/bmc-diag-1791932070-stubmod/`
- `/tmp/bmc_bundle.log`, `/tmp/bmc_tests.log` and `/tmp/bmc_typecheck.log`
- possibly an interrupted `swift-frontend` type-check process

Kill any process whose arguments contain `bmc-diag-1791932070`, then delete those paths.
Touch nothing else, and never touch Bryan's checkout.

## Verification

- Use
  `ssh -o ConnectTimeout=10 -o ServerAliveInterval=15 -o ServerAliveCountMax=4 mac …` so
  a sleeping Mac fails fast instead of hanging.
- Run `just portable-check` per `AGENTS.md`, on athena if this host lacks `swift`.
  Format-lint must exit 0 and the portable tests must stay green.
- On a fresh scratch copy on the Mac, made per `AGENTS.md` (rsync without
  `.build`/`.git`):
  - `./Scripts/xcode-swift.sh build` and `./Scripts/bundle.sh --identity -` succeed.
  - **`add` branch:** `just typecheck-tests` exits 0 on the fixed tree. Record its wall
    time. Then, in the scratch copy only, put one `unblockedBy:` back after
    `notLinked:`. The gate must exit non-zero and name that file and line. Restore the
    edit afterwards.
  - Delete the scratch copy when done. Never run `just install` and never use Bryan's
    checkout.
- If the Mac is offline, the product fixes are still the verified pair above, and the
  test fixes are reading-verified. Land step 1 and say in the final report that the
  macOS targets are unverified.
- In the final report, tell Bryan:
  - Rerun `just install ~/Applications -`.
  - Expected CI result: **Build** green, and **Test** red only on the two
    `RefsInspectorScrollTests` cases, with the bead ID from step 3.

## Out of scope

- The pre-existing warnings in the install log: actor isolation, `Sendable`,
  `Selector(...)`, and the deprecated `Text +`.
- Installing full Xcode on the Mac.
- Changing the `AGENTS.md` rule that lets an agent land with "macOS targets unverified".
- Any change to the pull-in card's design, keys or `bob capture --pull-in` contract.
