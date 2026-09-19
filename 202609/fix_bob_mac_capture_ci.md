---
tier: tale
title: Fix the bob-mac-capture Ensure Next notification test
goal:
  Restore green bob-mac-capture CI by correcting the inconsistent singular notification
  metadata assertion without changing the established production contract.
size: small
proposed_by: bbugyi200.apollo.0t
bead: sase-zr.7.3
create_time: 2026-09-19 13:18:41
status: wip
---

- **BEAD:** sase-zr.7.3

# Fix the bob-mac-capture Ensure Next notification test

## Context

GitHub Actions runs 86 through 88 fail in the `Test` step of the `macOS 26 SwiftPM` job.
The persistent failure is
`NotificationServiceTests.testEnsureNextNoOpOpensOnlyTheRouteNote`: the test expects
`targetPaths` to be absent, while `NotificationService.successContent` returns the
one-element array `["/tmp/bob/cash.md"]`.

That returned value is the established notification contract, not a product defect.
`successContent` intentionally stores both the legacy first `targetPath` and the ordered
`targetPaths` array whenever a notification has at least one destination. The README
documents the same compatibility behavior, and the pre-existing
`testTaskToggleSuccessContentOmitsDayFileFromTargetsWhenNothingChangedThere` test
already asserts a one-element `targetPaths` array for the equivalent single-route-note
case. The Ensure Next no-op correctly excludes the unchanged daily note; its new
assertion mistakes "one target" for "no targetPaths metadata."

Run 87 also failed
`CapturePanelModelTests.testPlusCommitsGlobalRouteDeclarationAndOpensTaskPicker`, but
the subsequent commit fixed its fake-Bob fixture and run 88 no longer reports that
failure. Do not broaden this repair to production notification routing or the resolved
transient failure.

## Implementation

1. In `Tests/BobMacCaptureTests/NotificationServiceTests.swift`, update
   `testEnsureNextNoOpOpensOnlyTheRouteNote` to assert that `targetPaths` equals the
   single route-note path, while retaining its assertions that the category is singular
   and `targetPath` names the route note. Add a concise comment if useful to distinguish
   exclusion of the unchanged day file from omission of the ordered-path metadata.
2. Keep `Sources/BobMacCapture/NotificationService.swift` unchanged: changing it to omit
   `targetPaths` for one target would violate the documented compatibility contract and
   make the Ensure Next path inconsistent with other singular notifications.

## Validation

1. On macOS with the repository-required Xcode 26+/macOS 26+ SDK, run the focused test:
   `./Scripts/xcode-swift.sh test --filter NotificationServiceTests/testEnsureNextNoOpOpensOnlyTheRouteNote`.
2. Run the full test suite exactly as CI does: `./Scripts/xcode-swift.sh test`.
3. Run the CI formatting gate:
   `swift-format lint --recursive Package.swift Sources Tests` when `swift-format` is on
   PATH, otherwise
   `./Scripts/xcode-swift.sh format lint --recursive Package.swift Sources Tests`.
4. Recheck the final diff to confirm it is limited to the test expectation (and optional
   explanatory comment), with no production behavior changes.

## Acceptance Criteria

- The Ensure Next no-op test verifies `targetPaths == ["/tmp/bob/cash.md"]` and still
  proves that the unchanged daily note is excluded.
- The focused notification test, full Swift test suite, and formatting lint all pass.
- No production source or unrelated test fixture is changed.
