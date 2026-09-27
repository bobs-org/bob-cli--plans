---
tier: epic
title: Fix Mac adjustment notification test calls
goal: bob-mac-capture's macOS CI test step compiles the Pomodoro adjustment notification
  tests and those tests pass.
parent_bead: bob-cli-27
phases:
- id: fix_calls
  title: Reorder the adjustment notification test calls
  size: small
  depends_on: []
  description: 'fix_calls: put relativeTarget last on the two new NotificationServiceTests
    capture() calls so Swift accepts them, and confirm the macOS CI test step for
    that commit is green.'
proposed_by: bbugyi200.apollo.bob-cli-27.land
create_time: 2026-09-26 20:02:58
status: done
bead_id: bob-cli-27.4
---

- **PROMPT:** [prompts/202609/fix_mac_adjust_notification_tests.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/fix_mac_adjust_notification_tests.md)
- **PARENT:** [202609/adjust_pomodoro_duration.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/adjust_pomodoro_duration.md)
- **BEAD:** [bob-cli-27.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-27/bob-cli-27.4.md)

# Plan: Fix Mac adjustment notification test calls

bob-cli-27.3 landed the Mac Capture presentation for whole-item `+N`/`-N` Pomodoro
adjustments in `bobs-org/bob-mac-capture` commit `3c9fe83` (on `master`). The macOS CI
run [36280594092](https://github.com/bobs-org/bob-mac-capture/actions/runs/36280594092)
built and passed Swift formatting, then failed in the Test step before any test ran.
`Tests/BobMacCaptureTests/NotificationServiceTests.swift` does not compile. Both new
tests call the file's private `capture(...)` helper with labeled arguments out of
declaration order:

- `testSinglePomodoroAdjustSuccessContentAppendsBeforeAfterDetail` (line 66)
- `testSingleClampedAdjustSuccessContentKeepsRequestedNote` (line 99)

The compiler reports the have-labels
`kind:routeLabel:relativeTarget:target:text:pomodoroAdjust:` against the expected order
`kind:routeLabel:target:text:scheduled:parentText:blockID:pomodoroStart:pomodoroAdjust:relativeTarget:`.
Swift requires labeled arguments to follow parameter order. `relativeTarget` is the
helper's last parameter; these calls pass it before `target`.

The helper signature is the one every other test in that file already uses. Leave it
alone. Move `relativeTarget: "day.md"` to after the `pomodoroAdjust:` argument in those
two calls only. The subtitle assertion expects `"day.md"`, which `displayLabel` takes
from `relativeTarget` when `routeLabel` is empty, so the argument must stay.

Production notification copy already matches the assertions in those tests: title
`Adjustment captured`, body containing the draft text and
`CapturePomodoroAdjustPresentation.statusText` (`Adjusted …` plus the clamped
`(requested … clamped)` note). Do not change `NotificationService` or the presentation
model unless a later assertion failure shows production disagrees with that contract.

This Linux host has no Swift toolchain. Open the repo with
`sase repo open gh:bobs-org/bob-mac-capture` (the linked name `bob-mac-capture` points
at `~/projects/github/bobs-org/bob-mac-capture`, which is not on this machine). After
the fix, the acceptance check is a green Test step on the macOS 26 SwiftPM workflow for
the new commit. The earlier steps of run 36280594092 (format and build) already passed;
the only reported errors were these two calls.

## Reorder the helper calls

Keep the existing `PomodoroAdjustSummary` values. The corrected call shape is
`capture(kind:routeLabel:target:text:pomodoroAdjust:relativeTarget:)`, with `target`
still the absolute day-file path used today and `relativeTarget` still `"day.md"`.
