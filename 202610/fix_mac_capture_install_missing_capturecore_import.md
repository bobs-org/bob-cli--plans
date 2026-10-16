---
tier: tale
title: Fix the Bob Mac Capture install build by importing CaptureCore in SettingsView
goal:
  "`just install` in bob-mac-capture builds and installs again on the Mac, because
  SettingsView.swift imports CaptureCore for BobClientError and the CI Build step on
  master is green again."
size: small
proposed_by: bbugyi200.athena.10p
create_time: 2026-10-16 12:00:37
status: wip
---

# Fix the `just install` compile break: `SettingsView.swift` never imports `CaptureCore`

## Problem

On the Mac, `just install` in `bob-mac-capture` runs `Scripts/install.sh`, which builds
the release product with `Scripts/xcode-swift.sh`. That build fails with exactly one
error (every other diagnostic in Bryan's output is a pre-existing, non-fatal warning):

```
Sources/BobMacCapture/SettingsView.swift:342:34: error: cannot find type 'BobClientError' in scope
342 |             } catch let error as BobClientError {
```

## Root cause

- `BobClientError` is a `public enum` in the `CaptureCore` module
  (`Sources/CaptureCore/BobErrors.swift`). `BobMacCapture` depends on `CaptureCore`
  (`Package.swift`), but Swift only resolves a top-level type name in files that import
  its module.
- Commit `6a7046e` ("feat(saved-templates): implement Mac Capture template checkout",
  phase bead `bob-cli-8b.5`) added `clearSavedTemplates()` to
  `Sources/BobMacCapture/SettingsView.swift` with `catch let error as BobClientError`.
  That file imports only `AppKit`, `Combine`, `ServiceManagement`, `SwiftUI`, and
  `UniformTypeIdentifiers`. The `import CaptureCore` lines in sibling files such as
  `SavedTemplateShelf.swift` and `CaptureAgendaStore.swift` do not make the name visible
  in this file. Member access through inferred types, such as the result of
  `savedTemplateShelf.clearSavedTemplates()`, compiles without the import, so only the
  explicit type name fails.
- **Why it reached `master`:** the `BobMacCapture` target is macOS-only, so the Linux
  `just portable-check` cannot type-check it. The phase agent's closing note on
  `bob-cli-8b.5` says "macOS app/test targets remain unverified because the Mac host was
  offline." CI did catch the break: run `38923816910` for `6a7046e` failed its **Build**
  step with this exact error. But `master` CI had already been red at the **Test** step
  since its last green run on 2026-10-13 (`38424226217`), so the new Build failure went
  unnoticed. That process gap is already tracked by task bead `bob-cli-1x` ("Document
  that agents must watch bob-mac-capture CI after every push").

## Diagnosis already verified on the Mac

The planner tested a throwaway scratch copy of `origin/master` (`6a7046e`) on the `mac`
tailnet host (Apple Swift 6.3.2, Command Line Tools). The copy has since been deleted.

- **Unmodified:** `./Scripts/xcode-swift.sh build` exits 1. Its only error is the one
  above.
- **With `import CaptureCore` added after `import AppKit`:**
  - the debug build exits 0;
  - `./Scripts/bundle.sh` (the release `BobMacCapture` product plus the bundle and
    signing) exits 0;
  - the release `BobMacCaptureInstallHelper` build, which `install.sh` also runs, exits
    0;
  - `./Scripts/typecheck-tests.sh` passes for `BobMacCaptureTests`,
    `BobMacCaptureInstallHelperTests`, `CaptureCoreTests`, and `RefsCoreTests`;
  - `swift format lint` on the file exits 0, with no import-order finding.

The one-line import is therefore the complete fix for the install failure. No other
compile error is hiding behind it.

## Change

Work in the linked `bob-mac-capture` repo (open it with `/sase_repo`; read its
`AGENTS.md` first).

In `Sources/BobMacCapture/SettingsView.swift`, add `import CaptureCore` between
`import AppKit` and `import Combine`, keeping the imports alphabetical:

```swift
import AppKit
import CaptureCore
import Combine
import ServiceManagement
import SwiftUI
import UniformTypeIdentifiers
```

Make no other source change. In particular, leave alone the Swift 6 concurrency,
`#selector`, and deprecation warnings from Bryan's output. They predate this break and
do not fail the build.

Suggested commit subject: `fix(settings): import CaptureCore for BobClientError`.

## Verification

Follow `bob-mac-capture/AGENTS.md`. Never land Swift you have not compiled.

1. Run `just portable-check` in the `bob-mac-capture` checkout. If `swift` is missing on
   the current host, run it on `athena`.
2. Compile on the Mac (best effort): rsync the checkout, excluding `.build` and `.git`,
   into a fresh temp directory on the `mac` host. Run `./Scripts/bundle.sh` and
   `just typecheck-tests` there and confirm both exit 0, then delete the copy. Never use
   Bryan's own checkout, and never run `just install` there. If the Mac is offline, say
   in the final report that the macOS targets are unverified.
3. After the commit reaches `master`, watch the GitHub Actions CI run for that SHA
   (`gh run list` / `gh run watch` in the `bob-mac-capture` checkout).
   - **Required:** the **Lint Swift formatting** and **Build** steps pass.
   - **Expected:** the **Test** step still fails, so the **Bundle** and **Install**
     gates are skipped. These are pre-existing XCTest failures already tracked by
     `bob-cli-7h`, `bob-cli-7p`, `bob-cli-8e`, and `bob-cli-8j`. Do not fix them in this
     tale.
   - Compare the failing test names with run `38921797357`, the last run that got past
     Build. Report any failure missing from that run, because it would be new.
   - Because CI skips the Bundle and Install gates, step 2's `bundle.sh` result is the
     install-path evidence. Say so in the report.
4. Use `/sase_new_task` to record this incident as corroboration on `bob-cli-1x`. Do not
   create a new bead for it. The evidence: `6a7046e` landed with the macOS targets
   unverified, and its red Build (run `38923816910`) went unnoticed because Test was
   already red on `master`.

## After landing

Tell Bryan to `git pull` in his Mac `bob-mac-capture` checkout and rerun `just install`.

## Out of scope

- The red CI **Test** step (tracked by `bob-cli-7h`, `bob-cli-7p`, `bob-cli-8e`, and
  `bob-cli-8j`).
- Silencing the existing compiler warnings.
- Changing the agent CI-watch convention or `AGENTS.md` (`bob-cli-1x`).
