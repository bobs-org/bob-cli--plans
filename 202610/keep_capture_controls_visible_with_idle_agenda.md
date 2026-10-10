---
tier: tale
title: Keep capture controls visible when the idle agenda returns
goal:
  Restore the idle agenda without clipping the editor or footer, with stable panel
  sizing and native regression coverage.
size: medium
proposed_by: bbugyi200.apollo.6g.w1.w0.f0
create_time: 2026-10-10 14:31:55
status: wip
---

# Keep the capture editor and controls visible when the idle agenda returns

## Outcome and scope

Opening Bob Mac Capture with a populated idle agenda, or clearing a settled draft to
restore it, must leave the editor, agenda, and footer visible in that order. The agenda
fits below the existing editor eye line; it cannot occupy the titlebar or push the
editor or footer outside the window. Repeated clear/type/show cycles must settle without
clipping, jumping, or a height feedback loop.

Implement this as one medium tale in `bob-mac-capture`. The work is bounded to the
agenda's SwiftUI/AppKit sizing integration and native regression coverage. Preserve the
draft-state cleanup and stale-preview protection from `63840fe`; reverting that fix
would restore the original missing-agenda bug. No bob-cli, capture grammar, JSON
contract, vault, task semantics, or SASE memory changes are needed.

## Repository and context

Open the app with `/sase_repo` from the implementation agent's host checkout:

```sh
sase repo open bob-mac-capture -r "Repair capture panel geometry when the idle agenda returns"
```

Read any instructions reported by the command. On the planning host the configured
primary checkout was absent. If that condition persists, the supported fallback is:

```sh
sase repo open gh:bobs-org/bob-mac-capture -r "Inspect agenda panel geometry; configured primary checkout is absent"
```

Use only the path returned by SASE. Planning inspected clean app revision
`63840fe879fcea6d565de7739cb2f2b2596d917a`; recheck relevant code before implementing
against a newer revision.

Read these governing decisions with `/sase_memory_read`:

- `decisions:idle-capture-shows-ledger-agenda`: cached content returns immediately;
  fitting preserves the editor eye line and folds farthest-first; bob supplies facts.
- `decisions:mac-capture-is-a-thin-client`: Swift owns presentation and process
  orchestration, with no vault parsing.

Read `plan:202610/restore_idle_agenda_after_clear.md` through `sase artifact read` for
the preceding fix. The new evidence is `~/tmp/screenshots/20261010_142424.png`. Inspect
that screenshot if it is available on the implementation host; its relevant observations
are recorded below so implementation does not depend on that path.

Relevant files, relative to the app repository:

- `Sources/BobMacCapture/CapturePanelView.swift`
- `Sources/BobMacCapture/CapturePanelController.swift`
- `Sources/BobMacCapture/CapturePanelWindowSizer.swift`
- `Sources/BobMacCapture/CaptureAgendaView.swift`
- `Sources/BobMacCapture/CaptureAgendaRowMeasurer.swift`
- `Sources/BobMacCapture/CapturePanelModel.swift`
- `Sources/CaptureCore/CaptureAgendaLayout.swift`
- `Sources/CaptureCore/CaptureAgendaFitPlanner.swift`
- `Tests/BobMacCaptureTests/CapturePanelEyeLineTests.swift`
- `Tests/BobMacCaptureTests/CaptureAgendaHeightConsistencyTests.swift`
- `Tests/BobMacCaptureTests/CaptureAgendaDesignTests.swift`
- `Tests/BobMacCaptureTests/CaptureAgendaModelTests.swift`
- `Tests/BobMacCaptureTests/CapturePreviewFullHeightTests.swift`
- `Tests/Fixtures/fake-bob`, `agenda-nothing-running.json`, `agenda-current.json`, and
  `agenda-heavy.json`
- `README.md`, `justfile`, `.github/workflows/ci.yml`

## Diagnosis and evidence limits

The screenshot shows the installed update `26aac7d -> 63840fe`, a successful build, and
an agenda headed "Today ... Nothing running". JOB, BOB, and SASE groups are visible,
including wrapped titles and a Work log. The traffic-light buttons overlap the agenda
header. The editor and footer controls are absent from the visible window, and the last
visible task is cut off at the bottom. Compiler warnings in the terminal are not
evidence of the cause; the app did build and launch.

`63840fe` modifies only model cleanup, its tests, and README wording. It enables a
previously blocked transition into an existing layout path. The source-level defect is
an inconsistent height contract between that agenda path and the window, with an
additional unsafe sizing side effect:

1. `CapturePanelView`'s `agendaVisible` change handler unconditionally resets
   `measuredAuxiliaryContentHeight` to zero and immediately reports content metrics. The
   incoming agenda has an already measured `CaptureAgendaPlan.totalHeight`, but panel
   metrics ignore it and depend on another geometry callback. Callback order and
   unchanged geometry can therefore matter even though the plan is already ready.
2. `currentAuxiliaryHeight` caps the **reported** agenda height to the budget plus
   padding. `CaptureAgendaPaneView` does not enforce that same viewport for an ordinary
   `!plan.overflows` agenda; only its overflow branch has an explicit height. A planned
   height, the rendered height under the current proposal, and the height passed to
   AppKit can disagree. A low layout priority is not a height bound.
3. The root hosting view uses `sizingOptions = []`, while the controller pins the
   window's content min/max height. The root stack specifies horizontal sizing but no
   outer vertical filling/top anchoring. Oversized SwiftUI content can therefore be
   centered within a smaller host, losing both ends. Apple's
   [NSHostingView sizingOptions documentation](https://developer.apple.com/documentation/swiftui/nshostingview/sizingoptions)
   explicitly describes this centering behavior; it is consistent with the screenshot.
4. `applyContentMetrics` calls `updateAgendaBudget` before setting
   `isApplyingContentHeight`. On a cache miss, `eyeLineTop` resizes and centers the real
   panel to measure the compact eye line, restores its min/max limits, but does not
   restore its frame. A later unchanged-target early return compares only
   `appliedContentHeight`, so it can leave the actual frame different from that cached
   value. Synchronous geometry/model publication can also reenter before the guard
   becomes active. The method's "hidden panel" comment is not an enforced precondition:
   screen and inset changes can reach it while visible.

Existing tests miss the composition: visibility tests assert model booleans, eye-line
tests mostly inject synthetic `CapturePanelContentMetrics`, height tests measure
`CaptureAgendaRowsView` alone, and design renders show the agenda pane alone. None
proves that the real editor and footer fit inside the controller-sized window after
clearing a draft.

These source findings are confirmed; the precise AppKit event ordering responsible for
the supplied screenshot remains a hypothesis until the native reproduction below. The
planning host is Linux and has no Swift executable. Do not present plan schema
validation or source inspection as a macOS reproduction. The first implementation step
must establish a failing native regression and use its geometry to distinguish
measurement reset, underestimated rows, and eye-line probe effects.

## Implementation

### 1. Reproduce the failure in a real hosted capture panel

Add focused `CapturePanelAgendaLayoutTests` (or extend an existing native suite if that
keeps the harness smaller). Instantiate the production `CapturePanelController` and its
actual `NSPanel`/`NSHostingView`, with fixture-backed model/store state and the existing
fake-bob client. Pin the fixture day consistently to `2026-08-28` and await the real
snapshot and settled preview rather than a loading plan.

Cover both a cached blank first show and the actual settled-preview-to-blank edit
transition. Use `agenda-nothing-running` plus a larger nothing-running fixture or an
in-memory fixture variant with several groups, multiline titles, and logs, matching the
screenshot's shape. Also include a current agenda and the heavy fixture. Use 760 pt and
620 pt window widths and a constrained height budget.

Observe actual editor, agenda, and footer rectangles in a common window coordinate
space, together with the host bounds and `contentLayoutRect`. Use a small internal
geometry observer/test seam if needed; do not replace the production root view or
manually feed expected content metrics to make this test pass. Obtain the native
editor's visible rectangle and first responder as well as its nominal frame, so an
offscreen editor cannot pass merely because it exists.

Record, in failure diagnostics, the plan total/budget/overflow state, measured and
reported auxiliary heights, actual panel and host frames, titlebar inset, and
editor/footer rectangles. Await a bounded layout-settled condition instead of assuming
one `layoutSubtreeIfNeeded` call flushes SwiftUI. Assert containment and ordering,
retained editor focus, and stability over subsequent layout turns. Run the test before
the fix on macOS and retain the failing evidence. If its first fixture does not
reproduce the screenshot, refine the fixture/transition using these measurements; do not
declare the specific callback race proven by inference.

### 2. Give the agenda pane and window one authoritative height

Define a small shared agenda viewport calculation from the existing measured plan, its
finite available budget, and the pane's vertical padding. For an ordinary plan that
fits, its content height is `plan.totalHeight`; for content that exceeds the available
budget it is bounded by that budget. Add outer padding exactly once. Treat an actual
zero budget as zero available agenda space, not an unbounded sentinel; the model already
supplies a positive startup fallback before live geometry is known. Keep invalid-value
handling explicit.

Use this same result in `CaptureAgendaPaneView`'s real top-aligned layout and in
`CapturePanelView.currentAuxiliaryHeight`. A visible cached plan must produce usable
metrics synchronously even if no new geometry event fires. Transition handlers may still
discard stale measurements from other auxiliary owners, but must not publish an agenda
height of zero while a positive-height plan owns the region. Do not use a stale preview
measurement or its `max` as the agenda's steady-state height.

Keep rendered-height observation for useful consistency checks, not as a circular input
that decides how much room the already measured agenda receives. Reflow width must still
reach the row measurer and trigger replanning. If native diagnostics show that row
totals underestimate wrapped content, repair that measurement/width contract using the
same row views and extend its consistency coverage; merely clipping an underestimated
plan is not a fix.

Constrain and top-align the hosted root within its actual available bounds while keeping
editor and footer at their measured natural heights. Preserve the existing editor
budget, typed-preview scrolling, stash/picker minimum heights, and ordinary
farthest-first agenda folding. Do not turn the entire capture panel into a scroll view,
add an arbitrary large fixed height, weaken `agendaVisible`, or force repeated store
reloads. This repair does not redesign fold/expansion semantics; any existing overflow
presentation stays inside the shared bounded agenda viewport.

Audit the budget arithmetic against the measured chrome: titlebar safe-area inset, drag
inset, editor, footer, inter-section gaps, root padding, and agenda padding. Correct
only demonstrated omissions/double counting, with focused assertions; avoid guessing new
constants to match one screenshot.

### 3. Make eye-line measurement and window sizing a coherent transaction

Guard the complete geometry transaction before operations that can publish model state
or change AppKit geometry, including inset/budget refresh. Preserve the latest metrics
arriving during the transaction and apply the newest report after it finishes without
unbounded recursion or lost updates.

Make eye-line derivation non-destructive to the presented panel. Preserve AppKit's
actual compact `center()` semantics and screen-aware caching; do not replace it with a
guessed screen midpoint. Prefer a separate hidden measurement panel with matching window
style and explicit target screen. If reusing the existing panel probe is necessary, save
and restore its entire frame and sizing constraints under the guard on every exit, and
prevent observers from treating that temporary geometry as final. The visible panel must
not flash at the probe size. Keep the approach local to the controller rather than
introducing a generic window-management layer.

Make the no-op height check account for actual applied content geometry as well as the
cached target; a stale cache must not prevent repairing a mismatched frame. Keep fixed
eye-line placement for agenda-only changes and the existing screen clamp for tall typed
previews. Recompute on actual screen/inset/compact-chrome changes without making the
agenda budget depend on the currently applied agenda height. Add a regression for a
cache miss or inset/screen invalidation followed by equal target metrics, checking the
actual panel size and editor position after settling.

### 4. Prove the composed behavior and retain the previous fix

The native regression matrix must cover these distinct boundaries:

| Case                                                                  | Required result                                                            |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Cached populated agenda on first show                                 | Editor, full planned agenda viewport, and footer contained below titlebar  |
| Settled preview cleared to empty, then whitespace                     | Agenda immediately restores; editor remains visible, editable, and focused |
| Clear while prior preview is finishing                                | Existing request retirement still prevents stale UI; geometry stays valid  |
| Repeated type/clear and hide/reopen with identical cached bytes       | No zero-height dependency on a fresh snapshot; no cumulative drift         |
| Width change and heavy/nothing-running agenda                         | Rows remeasure and fold within the shared budget; controls remain visible  |
| Small/zero budget and existing expanded-overflow state                | Bounded agenda region cannot displace editor/footer or become unbounded    |
| Titlebar/screen change with unchanged target metrics                  | No destructive probe, stale applied-height skip, or reentrant sizing loop  |
| Agenda disabled; completion, picker, stash, error, and normal preview | Existing auxiliary layouts and controls continue to fit                    |

Reuse existing model tests for asynchronous cancellation, submit identity, and
blank-draft cleanup rather than rewriting them. Extend policy tests for viewport
padding, finite/zero budgets, and plan/render agreement as needed, but do not let pure
arithmetic tests substitute for native geometry coverage.

When `BOB_MAC_CAPTURE_RENDER_DIR` is set, export representative full hosted-panel images
from the regression harness after first show and after clear at both widths. Capture the
native hosting view/window so the AppKit text editor is actually included; an isolated
`ImageRenderer` agenda render cannot expose this bug. Reuse the CI artifact directory
and inspect the resulting images for titlebar overlap, missing controls, bottom
clipping, and wrapped-title layout. Keep geometry assertions active without the optional
render environment variable.

Update README's layout/idle-agenda explanation only as needed to describe the final
shared height contract. Remove temporary diagnostics. Review the diff for scope and for
preservation of `63840fe`'s reset and preview-request protections.

## Validation and acceptance

On macOS 26 using the repository's Xcode wrapper, first run the new native regression on
the unfixed code, then run it with the fix. Run the relevant suites:

```sh
./Scripts/xcode-swift.sh test --filter CapturePanelAgendaLayoutTests
./Scripts/xcode-swift.sh test --filter CapturePanelEyeLineTests
./Scripts/xcode-swift.sh test --filter CaptureAgendaHeightConsistencyTests
./Scripts/xcode-swift.sh test --filter CaptureAgendaModelTests
./Scripts/xcode-swift.sh test --filter CapturePreviewFullHeightTests
just format-lint
just build
just test
```

Use the corresponding suite name if extending an existing harness instead. The full
suite covers the other auxiliary owners, capture success/reopen, and preview lifecycle.
The existing macOS CI workflow enables rendered fixtures and uploads them; use it for
native validation when the coding host lacks macOS. Use `/sase_monitor` for long test or
CI waits, and report any unavailable native verification explicitly. A Linux-only run
does not validate this SwiftUI/AppKit fix.

Manual Mac acceptance: open with a populated nothing-running agenda like the screenshot,
type a valid start draft and wait for its preview, select all/delete, then type again.
Repeat by deleting the last character, with a running Pomodoro, after resizing width,
and after hide/reopen. The editor must stay at its established eye line during agenda
transitions, retain focus, and accept the next keystroke; footer controls must remain
visible and usable. Long agenda content must fold within the available region without
the root extending beyond the host. Also verify normal tall previews still clamp
correctly. Use fixtures or a disposable vault for any write-path smoke test; no real
capture is needed to reproduce clearing a preview.

Completion requires a demonstrated native regression and passing corrected geometry
tests, successful format/build/full tests, and inspected full-panel images or the manual
Mac reproduction. If native access is unavailable, distinguish an implemented candidate
from a verified repair and leave that acceptance status explicit.
