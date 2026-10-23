---
tier: tale
title: Fix the Bob Mac Capture build broken by RefsScanReceiptObserver
goal: Bob Mac Capture compiles, bundles, and type-checks its tests again, so `just
  install` and macOS CI succeed. Fix the notify module import, the Self default argument,
  and the Void-returning test assertion.
size: small
proposed_by: bbugyi200.athena.13h
proposed_with: claude/opus
status: done
---

# Plan: Fix the Bob Mac Capture build broken by `RefsScanReceiptObserver`

## Goal

Make `just install ~/Applications` (and macOS CI) build Bob Mac Capture again. All edits
are in the linked `bob-mac-capture` repo; open it with your `/sase_repo` skill
(`sase repo open bob-mac-capture -r "..."`) and read its `AGENTS.md` first. No bob-cli
changes.

## Diagnosis (verified while planning)

The build broke at `bob-mac-capture` commit `6dc2c12` ("feat(refs): observe scan
receipts and notify for open PDF updates", epic phase `bob-cli-at.6`). That commit's
macOS CI run (`40015370455`) fails in its **Build** step on exactly the three errors
Bryan saw locally. Every other diagnostic in the output is a pre-existing warning (the
app targets build in Swift 5 language mode).

All three errors are in `Sources/BobMacCapture/Refs/RefsScanReceiptObserver.swift`:

1. **`cannot find 'notify_cancel' / 'notify_register_dispatch' in scope`** (lines 30 and
   101). On the macOS 26 SDK (Command Line Tools, Swift 6.3.2), `notify.h` is its own
   top-level Clang module (`usr/include/notify.modulemap` declares
   `module notify [system]`, pulled in by `extern module notify` in
   `usr/include/module.modulemap`). `import Darwin` does not re-export it, so the file
   needs `import notify`.
2. **`covariant 'Self' type cannot be referenced from a default argument expression`**
   (line 29, `= Self.registerLive`). Swift rejects `Self` in a class's default argument,
   even on a `final class`. Name the type instead:
   `RefsScanReceiptObserver.registerLive`.
3. **Related warning on the same line** ("converting function value of type
   `@MainActor (...) -> Int32?` ... loses global actor 'MainActor'", an error in Swift 6
   mode). The class is `@MainActor`, so `registerLive` inherits main-actor isolation,
   but it is used as a non-isolated `register` closure. It only calls the thread-safe
   `notify` API and hands the handler to a global dispatch queue, so it should be
   `nonisolated`.

There is also a fourth, **hidden test-only error**. CI stopped at Build, so the test
targets were never compiled. `Scripts/typecheck-tests.sh` then reports
`Tests/BobMacCaptureTests/RefsPanelModelTests.swift:2549:33: cannot convert value of type '()' to expected argument type 'Bool'`.
The same commit wrote `XCTAssertTrue(harness.model.performBannerAction(.scanAgain))`,
but `RefsPanelModel.performBannerAction(_:)` returns `Void`. The next line,
`XCTAssertTrue(harness.model.isScanning)`, already asserts that the scan started.

Why it landed red: the phase agent could not reach the Mac (`ssh mac` timed out), and
the Linux `just portable-check` cannot type-check `Sources/BobMacCapture*`. That left
macOS CI as the only check, and it went red on push with no follow-up.

During planning, a throwaway rsync copy on the Mac with exactly the changes below passed
all of these, and the copy was then deleted:

- `./Scripts/bundle.sh --identity "-"` (the release build plus the bundle)
- `./Scripts/typecheck-tests.sh` (all four test targets: PASS)
- `swift format lint` on the two files (exit 0)

The CLI contract also matches: bob-cli posts `org.bobs.bob.ref.scan-finished`
(`src/native/highlights_ref/scan_receipt.rs`), the same name as
`RefsScanReceiptReader.darwinNotificationName`.

## Changes

### 1. `Sources/BobMacCapture/Refs/RefsScanReceiptObserver.swift`

Keep this file's existing 4-space indentation, and keep lines within 100 columns.

- Add `import notify` after `import Foundation`. Keep `import Darwin`,
  `import Dispatch`, and `import RefsCore`.
- Replace the `register` default argument's `Self.registerLive` with the type name,
  wrapped:

  ```swift
          register: @escaping (@escaping @Sendable () -> Void) -> Int32? =
              RefsScanReceiptObserver.registerLive,
  ```

- Mark the live registrar `nonisolated`, wrapping its signature:

  ```swift
      nonisolated private static func registerLive(
          _ handler: @escaping @Sendable () -> Void
      ) -> Int32? {
  ```

  Leave its body and the `cancel` default (`{ token in _ = notify_cancel(token) }`)
  unchanged; `import notify` makes `notify_cancel` resolve.

None of this changes behaviour. `AppDelegate.startScanReceiptObserver` still uses the
defaults, and `RefsScanReceiptObserverTests` injects `register`/`cancel`.

### 2. `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`

In the busy-PDF "Scan Again" banner test (around line 2549), replace

```swift
    XCTAssertTrue(harness.model.performBannerAction(.scanAgain))
```

with a bare call, matching the other `.scanAgain` call later in the file:

```swift
    harness.model.performBannerAction(.scanAgain)
```

Keep the following `XCTAssertTrue(harness.model.isScanning)` and the
`pdfUpdateFooterClauses.isEmpty` assertions as they are.

### Out of scope

- The many pre-existing Swift 6 / Sendable / deprecation warnings elsewhere in the build
  output. They are warnings under `.swiftLanguageMode(.v5)` and are not part of this
  break.
- `CaptureCore` and `RefsCore`, which are untouched.
- Docs. Behaviour and contracts are unchanged, so `just docs-check` is not needed.

## Verification

Never land Swift you have not compiled (bob-mac-capture `AGENTS.md`). These changes
touch only `Sources/BobMacCapture*` and `Tests/BobMacCapture*`, which only the Mac can
type-check:

1. Run `ssh mac` (best effort; the MacBook may be offline). Make a fresh scratch dir
   with `mktemp -d` on the Mac. `rsync` the opened bob-mac-capture checkout into it,
   excluding `.build` and `.git`. In the scratch dir, run:
   - `./Scripts/bundle.sh --identity "-"`: must finish with the `.app` bundle path and
     no `error:` lines.
   - `./Scripts/typecheck-tests.sh` (or `just typecheck-tests`): must exit 0 with `PASS`
     for `BobMacCaptureTests`, `BobMacCaptureInstallHelperTests`, `CaptureCoreTests`,
     and `RefsCoreTests`.
   - `swift-format lint` (or `swift format lint`) on the two edited files: must exit 0.
     Warnings are fine; this file's 4-space indentation already produces them, and CI
     fails only on a nonzero exit.

   Then delete the scratch dir. Never build in Bryan's own checkout, and never run
   `just install` on the Mac.

2. If the Mac is unreachable, say in the final report that the macOS targets are
   unverified. macOS CI on push is then the only check.
3. Linux `just portable-check` is not required, because no portable target changes.

Done when the scratch release build and test type-check both pass. Then Bryan's
`just --justfile ~/projects/github/bobs-org/bob-mac-capture/justfile install ~/Applications`
builds and installs again.
