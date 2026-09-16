---
tier: tale
title: Fix bob-mac-capture CI Ctrl-J selection-replacement test expectations
goal:
  The bob-mac-capture macOS 26 SwiftPM CI job passes on master again after correcting
  the four wrong expected values in the failing Ctrl-J selection-replacement test, with
  no production behavior change.
size: small
proposed_by: bbugyi200.athena.0lu
create_time: 2026-09-16 09:03:33
status: wip
---

# Fix bob-mac-capture CI: correct the Ctrl-J selection-replacement test expectations

## Goal and scope

Get `bobs-org/bob-mac-capture` GitHub Actions (`CI` workflow, job `macOS 26 SwiftPM`)
back to green. The job fails because one test has wrong expected values. The production
code is correct. The fix is a test-only change of four expected values in one test
method. No source, README, workflow, or bob-cli changes are needed.

All changes belong in the linked **bob-mac-capture** repository. Before reading or
editing it, use `/sase_repo` and run:

```sh
sase repo open bob-mac-capture -r 'Fix failing macOS CI test expectations for Ctrl-J selection replacement'
```

Use the path it prints for every read and write. File paths below are relative to that
checkout.

## Diagnosis (already established; do not re-derive)

`actstat --repo bobs-org/bob-mac-capture -n 8` shows `master` has been red for its last
two commits:

- `c6273cf feat: remove dash bullet prefixes with ctrl-j` (CI run #82, run id
  `34523454068`)
- `20815a4 fix(capture): commit route completion on plus, fix stale selection caret` (CI
  run #83, run id `34790194135`)

Both fail at step 6 (`Test`), and both have the same 4 XCTest failures, all in one test.
The format-lint and build steps pass. Every other test (459 total in run #83) passes.
The older red runs (#76–#78, InstallRelauncher compile and path failures) were already
fixed by `cf73955` and are unrelated. The last green run was `b979528`.

Failing test: `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`,
`testBulletNewlineResolverKeepsSelectionReplacementBehaviorOnPopulatedDashPrefixSelections`.
It was added in `c6273cf` and fails at lines ~1879, 1880, 1886, and 1887. CI log excerpt
(actual value first, expected value second):

```
XCTAssertEqual failed: ("Parent\n \n  -  child") is not equal to ("Parent\n \n-  child")
XCTAssertEqual failed: ("{13, 0}") is not equal to ("{11, 0}")
XCTAssertEqual failed: ("Parent\n \n  - Next") is not equal to ("Parent\n \n- Next")
XCTAssertEqual failed: ("{13, 0}") is not equal to ("{11, 0}")
```

Root cause: both cases use a **non-collapsed** selection, so
`CaptureBulletNewlineEditResolver.resolve(in:selectedRange:)` in
`Sources/BobMacCapture/CapturePanelController.swift` skips the placeholder branch and
the new populated-dash-prefix branch, since both require `selectedRange.length == 0`. It
falls through to the normal insertion fallback, which has existed since `4c22525`. That
fallback replaces the selection with
`terminator + supportedAuthoredIndent(lineContent) + "- "`. The caret line is
`"  - child"`, so the copied indent is two spaces and the replacement is `"\n  - "`. The
test author wrote the expected values without that copied indent; this agent could not
run AppKit tests on Linux.

The production behavior is the intended one. The approved plan behind `c6273cf`
(`plan:202609/ctrl_j_bullet_prefix.md`) says: "Noncollapsed selections also keep the
current selected-text replacement behavior". Its test list says: "Keep noncollapsed
selection replacement, including a selection beginning before the marker and a multiline
selection, unchanged." The test name says the same thing
("KeepsSelectionReplacementBehavior"). The existing test
`testBulletNewlineResolverCopiesSupportedAuthoredIndentation` covers the same two-space
indent copying. So the fix is to correct the test's expected values. Do **not** change
the resolver: changing it to satisfy the wrong expectations would break the plan's
requirement that selection replacement stays unchanged.

The corrected values were checked on Linux by compiling the unchanged resolver on its
own with `swiftc` and running it on both inputs. It produced exactly
`"Parent\n \n  -  child"` / `{13, 0}` and `"Parent\n \n  - Next"` / `{13, 0}`, matching
the CI "actual" values above.

## Implementation

In `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, inside
`testBulletNewlineResolverKeepsSelectionReplacementBehaviorOnPopulatedDashPrefixSelections`,
change only the four expected values. Keep the inputs, selected ranges, method name, and
structure unchanged.

1. `prefixSelection` (input `"Parent\n  - child"`, selection starting at
   `"Parent\n ".utf16.count` with length `" -".utf16.count`):
   - text: `"Parent\n \n-  child"` → `"Parent\n \n  -  child"`
   - selection: `NSRange(location: "Parent\n \n- ".utf16.count, length: 0)` →
     `NSRange(location: "Parent\n \n  - ".utf16.count, length: 0)`
2. `multilineSelection` (input `"Parent\n  - child\nNext"`, selection starting at
   `"Parent\n ".utf16.count` with length `" - child\n".utf16.count`):
   - text: `"Parent\n \n- Next"` → `"Parent\n \n  - Next"`
   - selection: `NSRange(location: "Parent\n \n- ".utf16.count, length: 0)` →
     `NSRange(location: "Parent\n \n  - ".utf16.count, length: 0)`

Keep the existing formatting style of the file (4-space indentation, trailing-comma-free
argument lists). The edited lines stay well under the line-length limit.

## Out of scope

- The CI log's `Node.js 20 is deprecated … actions/checkout@v4` notice is a warning only
  and does not fail the job. Leave `.github/workflows/ci.yml` unchanged. If you want to
  track bumping `actions/checkout` / `actions/upload-artifact`, file it as a follow-up
  with `/sase_new_task` rather than folding it into this fix.
- The other repositories `actstat` reports as failing (`bbugyi200/dotfiles`,
  `sase-org/sase`, `sase-org/sase-core`) are not part of this task.

## Validation

- The `BobMacCaptureTests` target needs AppKit and cannot build on Linux. Plain
  `swift test` on Linux fails for that reason, so do not report a Linux run as covering
  this test. On a macOS 26 host with Xcode, run
  `./Scripts/xcode-swift.sh test --filter 'BobMacCaptureTests.BobMacCaptureTests/test.*BulletNewline'`
  and then `just format-lint`, `just build`, and `just test`.
- If only Linux is available (the expected case on athena), optionally re-confirm the
  four values without touching the repo. Copy `CaptureBulletNewlineEdit`,
  `preferredCaptureLineTerminator(in:)`, and `CaptureBulletNewlineEditResolver` from
  `CapturePanelController.swift` into a scratch `main.swift` under a `mktemp -d`
  directory, prefixed with `import Foundation`. Apply each edit with
  `NSString.replacingCharacters(in:with:)` and check the new expected values. Compile
  with `swiftc` after `export PATH="$HOME/.local/share/swiftly/bin:$PATH"`. Delete the
  scratch directory afterward.
- Review `git diff` and confirm that only the four expected-value lines in that one test
  changed.
- Commit the linked-repo change through the normal `/sase_final` commit decision. After
  the commit reaches `master`, the `CI` workflow reruns. The fix is confirmed when
  `actstat --repo bobs-org/bob-mac-capture` (or
  `gh run list -R bobs-org/bob-mac-capture --workflow CI --limit 1`) shows the new
  commit green. The remaining post-test steps (Bundle, plist/signature verification,
  launch smoke test, install/reinstall) have not run since `b979528` and must also pass.
  `c6273cf` and `20815a4` did not touch `Scripts/` or the workflow, so no new failure is
  expected there. If one does appear, diagnose it from
  `gh run view <run-id> -R bobs-org/bob-mac-capture --log-failed` rather than assuming
  it is related to this fix. In the final report, state plainly whether CI was observed
  green or is still pending after landing.
