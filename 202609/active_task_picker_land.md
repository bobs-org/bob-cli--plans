---
tier: tale
size: medium
title: Repair and land the Active Task Picker epic
goal:
  Make the existing picker implementation pass macOS verification, review its rendered
  card, and close bob-cli-2g.
proposed_by: bbugyi200.apollo.bob-cli-2g.land
bead: bob-cli-2g
create_time: 2026-09-28 19:17:04
status: wip
---

- **PARENT:**
  [202609/mac_active_task_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/mac_active_task_picker.md)
- **BEAD:**
  [bob-cli-2g](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2g/README.md)

# Repair and land the Active Task Picker epic

## Context

Epic `bob-cli-2g` has three closed phases. Its linked plan is
`plan:202609/mac_active_task_picker.md`. The implementation is in the linked
`bob-mac-capture` repository; when `sase repo open bob-mac-capture` fails because its
primary workspace is missing, use
`sase repo open gh:bobs-org/bob-mac-capture -r "Repair and verify bob-cli-2g"` and use
the printed checkout path. Open `plans` through `sase repo open plans` before editing
the linked plan. Do not use a numbered workspace path from this planning turn.

The three epic commits are `309431b`, `039a322`, and `0a6cd0e`, consecutive at the
linked repository's current head; there are no later unrelated commits there as of this
review. The first two macOS 26 SwiftPM CI runs (`36494257046`, `36495588592`) fail in
tests; the third (`36496614673`) fails while compiling tests. CI formatting lint and
build passed on the third run. Tailnet `mac` timed out over SSH, so use the macOS CI
runner for verification. The existing epic note records triage of all three
`PROPOSED FOLLOW-UP` phase notes: they are one epic-caused verification obligation, with
no separate task bead.

## Remaining work

1. Fix the eight `CaptureCompletionCandidate` calls in
   `Tests/BobMacCaptureTests/ActiveTaskPickerDesignTests.swift`: `blockID:` must precede
   `statusSymbol:`. Confirm against the initializer in
   `Sources/CaptureCore/CaptureModels.swift`.
2. Fix `ActiveTaskMatchHighlights.coalesced` in
   `Sources/CaptureCore/ActiveTaskPickerPresentation.swift`. Its loop uses the _position
   value_ `end` as an index into `sorted`, causing `Index out of range` when filtering
   (CI crashes while running `testBobSurfacesBobRoute`). Track a separate array index
   and value, coalesce sorted adjacent positions, and test nonzero and sparse positions
   that previously crashed. Inspect any further failures after the crash is removed.
3. Correct the mismatched assertion in
   `ActiveTaskDisplayTextTests.testUnmatchedBacktickStaysLiteral`. The epic plan says a
   backtick is closed by the _next_ backtick, so `fix `oops and `ok`` displays `fix oops
   and ok`` with plain/code/plain segments. Preserve the implementation's next-backtick
   behavior; add a separate truly unmatched single-backtick case if needed.
4. Run the linked repository's `just format-lint build test` on macOS or use its
   `macOS 26 SwiftPM` CI. Iterate until the test job and downstream bundle/smoke steps
   pass on the resulting commit. The Linux host has no Swift toolchain; do not report
   static inspection as a Swift test pass. Do not run `just check-full` in the primary
   project. For primary-project file changes, use `just check`.
5. Complete the rendered-image review required by `picker-design`: run the existing
   `BOB_MAC_CAPTURE_RENDER_DIR` gated test on a macOS runner or reachable Mac, obtain
   its PNGs, inspect light and dark grouped, filtered, no-match, and no-task states, and
   adjust any visible defects. If no interactive GUI is available, record the exact
   smoke-test limitation in the epic close note; CI's bundle/launch smoke still needs to
   pass. Confirm source, README, and tests still cover the planned picker behavior: full
   snapshot, local fuzzy filter, grouped rows, quiet incomplete `^`, insert and submit,
   Escape/chip/Backspace, focus, sizing, and accessibility.
6. Recheck drift since `309431b`: inspect any later non-epic commits in the linked
   repository and relevant primary project commits, integrating code that should now use
   the picker. Recheck all three child notes. There is no current parent bead on
   `bob-cli-2g`; re-read it before close in case that changes.

## Final closeout (must be done by this tale's coder)

Run `sase bead epic-symbols bob-cli-2g` and resolve every entry or re-key it to a
still-open later bead that needs the exemption. Then close normally with
`sase bead close bob-cli-2g --note "<verification, integration, CI run ID, rendered-image review, and every follow-up triage outcome>"`.
Never force merely to make the close succeed; resolve unfinished phases deliberately if
the command rejects them. Run `just symvision` if available in the primary repository.
Set `status: done` in the frontmatter of `plan:202609/mac_active_task_picker.md` after
the close. Re-read `bob-cli-2g` for `parent_bead`; if a parent appeared, handle it
according to the landing prompt (close a completed parent phase normally, or recheck a
parent plan's descendants, notes, linked plan, drift, and symbols before closing,
stopping with a blocker note if incomplete).

## Acceptance

The macOS CI test and downstream jobs pass on the repaired picker commit; rendered PNGs
have been inspected or an exact external limitation recorded; the epic close note
accounts for the three phase proposals and integration review; `bob-cli-2g` is closed
without force; `just symvision` is clean when available; and the linked plan frontmatter
says `status: done`.
