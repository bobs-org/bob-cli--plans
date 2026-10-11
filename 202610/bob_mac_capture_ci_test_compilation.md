---
tier: tale
title: Repair bob-mac-capture CI test compilation
goal:
  Restore passing macOS tests and the complete CI job while preserving
  inspector-scrolling coverage.
size: small
proposed_by: bbugyi200.athena.0zk
create_time: 2026-10-11 07:03:33
status: wip
---

# Repair bob-mac-capture CI test compilation

## Outcome and scope

Restore the `macOS 26 SwiftPM` GitHub Actions job in `bobs-org/bob-mac-capture` by
repairing two compiler errors in the inspector-scrolling tests, preserving their
behavioral assertions, and validating the entire macOS suite. This is focused work for
one implementation agent, so use a `tale` with `size: small`.

All implementation paths below are relative to the linked **bob-mac-capture**
repository, not the host bob-cli repository. Open it with `/sase_repo`:

```sh
sase repo open bob-mac-capture -r "Implement the approved CI test-compilation repair"
```

Read any instructions reported by that command, and inspect the current status before
editing. The planning checkout was clean at `a82a4acb41111cc5d7c342fc8b13416d6633587c`.
Preserve any subsequent user changes and refresh the diagnosis if newer commits have
already repaired either defect.

## Confirmed diagnosis

On October 11, 2026, `actstat --repo bobs-org/bob-mac-capture --limit 25 --format json`
showed eight consecutive failures beginning with `4c22d09`. Relevant runs:

| Evidence                                                          | Commit    | Run                                                                             |
| ----------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------- |
| Last green, including tests, bundle, launch, and installer checks | `131e377` | [CI #255](https://github.com/bobs-org/bob-mac-capture/actions/runs/38022607121) |
| First failure                                                     | `4c22d09` | [CI #256](https://github.com/bobs-org/bob-mac-capture/actions/runs/38048990629) |
| Latest failure inspected                                          | `a82a4ac` | [CI #263](https://github.com/bobs-org/bob-mac-capture/actions/runs/38131757374) |

The first and latest failed logs contain the same 11 compiler diagnostics. These are
test-target compilation failures, before test execution. The latest job passed toolchain
checks, formatting lint, and application build. Bundle, signature, launch, and install
checks were skipped after the test failure; they are not separate confirmed failures.
The runner used Xcode 26.6 and Apple Swift 6.3.3.

Both defects were introduced by `4c22d09`
(`feat(refs): scroll the inspector preview with Ctrl+D and Ctrl+U`), as confirmed by
local git blame:

1. `Tests/BobMacCaptureTests/RefsInspectorScrollTests.swift`, around lines 170 and 182:
   `let listOffset = listOffset(in: harness.panel)` shadows the helper method. The later
   `listOffset(in: harness.panel)` assertion tries to call that `CGFloat` value,
   producing `cannot call value of non-function type 'CGFloat'`.
2. `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, around lines 807–829: the
   nested `PanelSnapshot` struct lacks main-actor isolation. Its initializer and
   `assertUnchanged` method read `RefsPanelModel`'s main-actor-isolated `query`,
   `scope`, `selectedID`, `listing`, and `banner` properties. The outer test class's
   annotation does not make these nested-type methods actor-isolated. There are five
   errors in the initializer and five in XCTest autoclosures.

`Package.swift`, `.github/workflows/ci.yml`, and `Scripts/xcode-swift.sh` are unchanged
between the last green commit and the inspected head. `Package.swift` includes
`BobMacCaptureTests` only on macOS, explaining why a Linux core-test pass cannot detect
these defects.

To refresh the evidence, run `actstat` and use
`gh run view RUN_ID --repo bobs-org/bob-mac-capture --log-failed`. Compiler batch output
repeats diagnostics; deduplicate error lines when comparing runs.

## Implementation

1. In `testHostedControlDUScrollsInspectorWhileSearchKeepsFocus`, rename the saved local
   offset to `listOffsetBefore` and update the expected value in the assertion. Keep the
   actual `listOffset(in:)` measurement and `accuracy: 1`, so the test still proves that
   inspector scrolling leaves the reference list stationary.
2. Annotate the nested `PanelSnapshot` struct with `@MainActor`, following the nearby
   actor-isolated `Harness` pattern. Preserve all snapshot fields and assertions. The
   two call sites already execute in the main-actor-isolated test class.
3. Review the diff for a focused repair. Keep the scrolling feature, all existing tests,
   Swift language settings, and CI gates intact. Do not suppress concurrency checking or
   skip tests. Existing failing tests provide the regression coverage; additional tests
   that merely restate these syntax changes are unnecessary.

## Validation and completion

The planning host is Linux with Swift 6.0.3; do not use its test results as evidence
that the macOS target compiles. A read-only probe confirmed a reachable compatible Mac
at `bbugyi@kellys-macbook-pro.tail297af1.ts.net`, with Command Line Tools, Apple Swift
6.3.2, and macOS SDK 26.5. The short `mac` SSH alias did not resolve on the planning
host. Read `tailnet.md` through `/sase_memory_read` before remote work and recheck
connectivity. Use a disposable validation copy of the authorized working tree on that
Mac, transferred from the opened checkout, with a separate build directory; do not
replace an existing development checkout or install/restart the user's app. Exclude
`.git` and `.build` from the transfer and preserve executable fixture modes. Verify the
two changed test files match the local working tree used for the repair.

Run commands from that validation tree using the repository's Apple-toolchain wrapper:

```sh
./Scripts/xcode-swift.sh --version
./Scripts/xcode-swift.sh test --filter 'RefsInspectorScrollTests|RefsPanelModelTests|RefsKeyRouterTests'
just format-lint
./Scripts/xcode-swift.sh build
```

Then run the full `./Scripts/xcode-swift.sh test` with `BOB_MAC_CAPTURE_RENDER_DIR`
pointing to a disposable render-output directory, as CI does. The filtered invocation
compiles the complete test target and exercises the hosted scroll assertions, snapshot
invariants, and related key routing. A full run is required because the compile failure
previously prevented all runtime tests from executing. Address any reproducible failures
exposed by this repair using their actual diagnostics; preserve assertions rather than
weakening tests to obtain green. Revalidate after any additional edits. Finish with
`git diff --check` and review the final patch. If Mac access becomes unavailable, obtain
equivalent macOS CI evidence instead of substituting Linux results.

Use `/sase_monitor` for commands or CI waits that outlast a provider turn; wait for the
monitor handoff command itself to exit. Submit repository changes through the normal
SASE finalization workflow. Once a workflow run for the repaired commit exists, use
`gh run view` plus `actstat` to verify the exact SHA and the complete job outcome. Do
not mistake rerunning an old failing SHA for validating this repair. The existing
workflow must also complete bundle assembly, plist/signature checks, launch smoke
testing, and install/reinstall testing on its disposable runner. Resolve any newly
revealed failure relevant to restoring this job, and verify the resulting commit.

Acceptance requires both compilation defects removed, preserved scrolling/snapshot
assertions passing on macOS, the full macOS suite passing, formatting and build checks
passing, and a successful full CI run for the repaired commit. Record the run URL,
commit SHA, test results, and any remaining validation limitation accurately; never
report a green pipeline based only on successful compilation or a Linux test run.
