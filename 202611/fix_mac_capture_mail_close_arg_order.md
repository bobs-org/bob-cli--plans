---
tier: tale
title: Fix bob-mac-capture install break from out-of-order mailCloseVisible argument
goal:
  "`just install` for bob-mac-capture builds again. The CaptureKeyRoutingContext call
  passes mailCloseVisible in memberwise-init declaration order, and the fix is compiled
  on macOS before landing."
size: small
proposed_by: bbugyi200.athena.1ft
proposed_with: claude/opus
create_time: 2026-11-24 16:07:38
status: wip
---

# Fix bob-mac-capture `just install`: `mailCloseVisible` passed out of declaration order

## Context

Restore `just install` (and `just install-all`) for the `bob-mac-capture` linked repo.
The build broke at `c2a44267` ("feat(mac-capture)!: show the Mail Close Card for
email-task closes"). Make the smallest possible source change, prove it compiles on
macOS, and land it.

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture -r "..."`). Read its
`AGENTS.md` before editing. All paths below are relative to that checkout.

## Root Cause (diagnosed)

The production build fails with exactly one error:

```
Sources/BobMacCapture/CapturePanelController.swift:3355:21: error: argument
'moveChoiceVisible' must precede argument 'mailCloseVisible'
```

- `CaptureKeyRoutingContext` (`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`,
  around line 169) is a struct whose call sites use its synthesized memberwise
  initializer. Swift requires memberwise arguments in stored-property declaration order.
- `c2a44267` declared `var mailCloseVisible = false` after `moveChoiceFilterIsEmpty`,
  just before `pullInChoiceVisible`. The same commit passed `mailCloseVisible:` at the
  `CaptureKeyRoutingContext(...)` call in `CapturePanelController.swift`, around
  line 3354. There it sits before `moveChoiceVisible:`, which is out of order.
- The commit landed without a macOS compile. Its own message says "macOS targets
  unverified at handoff". Linux `swift test` cannot type-check `Sources/BobMacCapture*`.
  The exact-SHA CI run (45473754866) never got a macOS runner ("job was not acquired by
  Runner"), and the precommit snapshot run (45471925794) was cancelled.

Evidence that this is the only break:

- The install is a production build, which compiles each module as a whole. Its full log
  on the mac has this one error and nothing else in the `BobMacCapture` module.
- No other `CaptureKeyRoutingContext(...)` caller passes `mailCloseVisible:`. That
  includes the test files under `Tests/BobMacCaptureTests/`.
- During planning, a scratch copy on the mac had only the change below applied. With it,
  `./Scripts/bundle.sh --identity -` built, bundled and signed successfully. After that,
  `./Scripts/typecheck-tests.sh` passed all four test targets: BobMacCaptureTests,
  BobMacCaptureInstallHelperTests, CaptureCoreTests and RefsCoreTests.

## Change

Edit exactly one file: `Sources/BobMacCapture/CapturePanelController.swift`.

In the `CaptureKeyRoutingContext(...)` argument list inside the key-routing path (the
`keyRouter.command(for: event, context: ...)` call, around line 3354):

1. Delete the line `mailCloseVisible: model.mailCloseVisible,`, which currently sits
   right after `pickerSourceIsRelation: model.pickerSourceIsRelation,`.
2. Re-insert the same line right after
   `moveChoiceFilterIsEmpty: model.moveChoice?.filterText.isEmpty ?? true,` and right
   before `pullInChoiceVisible: model.pullInChoiceVisible,`. Use the same indentation as
   those neighbouring argument lines (20 spaces).

The resulting diff is one line removed and one line added:

```diff
                     pickerSourceIsRelation: model.pickerSourceIsRelation,
-                    mailCloseVisible: model.mailCloseVisible,
                     moveChoiceVisible: model.moveChoiceVisible,
 ...
                     moveChoiceFilterIsEmpty: model.moveChoice?.filterText.isEmpty ?? true,
+                    mailCloseVisible: model.mailCloseVisible,
                     pullInChoiceVisible: model.pullInChoiceVisible,
```

Fix the call site rather than reordering the struct's stored properties. The router
already checks `context.mailCloseVisible` before `context.moveChoiceVisible`, and that
check does not depend on declaration order. Moving the argument is a one-line change
that leaves every other memberwise-init caller alone, so behavior does not change at
all.

Do not re-indent the surrounding block. That argument run is already indented 4 extra
spaces, and lint accepts it today. `.swift-format` house style says to format only the
lines you change.

## Non-Goals

- Do not touch the warnings in the install log. All of them predate `c2a44267`:
  - the always-failing `NSTextViewDelegate` cast at `CapturePanelController.swift:4029`
    (from `1d5735a6`)
  - the implicit-`self` warnings at `CapturePanelModel.swift:4541` (from `16c79f69`)
  - the unused `armEscDiscard()` result at `CapturePanelModel.swift:5707` (from
    `6f4e2799`)
- Do not fix the pre-existing red Test step on master (see Verification step 4). Only
  record it.
- No docs change. The fix restores what `c2a44267` already documented.

## Verification

Follow the repo's `AGENTS.md`. Never land Swift you have not compiled.

1. **Portable gate (athena/Linux):** run `just portable-check`. It parses and lints
   every Swift file, then builds and tests CaptureCore/RefsCore.
2. **macOS compile on a scratch copy (best effort, required if the mac is online):**
   - rsync the checkout to a fresh `mktemp -d` directory on the `mac` tailnet host,
     excluding `.build` and `.git`.
   - There, run `./Scripts/bundle.sh --identity -` and `./Scripts/typecheck-tests.sh`.
     Both must exit 0.
   - Delete the scratch directory afterwards.
   - Never build in Bryan's checkout on the mac, and never run `just install` there.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
3. **Precommit CI gate:** run `Scripts/macos-ci-check` from the repo. It can take up to
   90 minutes, so give the command a large explicit timeout.
   - If the job is never acquired by a macOS runner (a capacity failure), rerun it once.
     If it still gets no runner, rely on step 2 and say so in the report.
4. **Known pre-existing Test failure:** master's CI Test step was already red at
   `0b441d3b` (run 45262574122) on
   `RefsPanelModelTests.testHighlightsTaskFallbackShowKeepsTabsAndBannerUntilLibraryRestores`
   (`XCTAssertEqual failed: ("small_reading") is not equal to ("small")`).
   - If that test is the only failure in step 3, land the fix anyway. In that case,
     Build must be green and step 2's `bundle.sh` must have passed.
   - Any other new failure blocks landing.
   - Record the pre-existing failure through `/sase_new_task`, which checks for an
     existing bead first. Do not fix it here.
5. **Land:** commit with the message
   `fix(capture): pass mailCloseVisible in memberwise-init order`. Then watch the
   exact-SHA macOS CI run after the push with `/sase_monitor`. Apply the same rule as
   step 4: only the known `RefsPanelModelTests` failure is acceptable.
6. **Report:**
   - State the root cause in one sentence.
   - List which gates ran and their results.
   - Tell Bryan to rerun `just install-all` (or `just install` in his `bob-mac-capture`
     checkout on the mac) to pick up the fix.
