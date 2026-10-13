---
tier: tale
title: Finish mac_choice on macOS CI and close bob-cli-7c.7
goal: bob-mac-capture's pull-in tests compile and pass on macOS CI, and bob-cli-7c.7
  is closed with evidence.
size: medium
proposed_by: bbugyi200.apollo.7f
status: done
---

# Plan: Get mac_choice (bob-cli-7c.7) green on macOS CI, then close it

## Context

bob-cli-7c.7 is the `mac_choice` phase of epic bob-cli-7c (Successor Offers,
`plan:202610/successor_offers.md`). Everything in the phase is already published in
linked repo **bob-mac-capture** commit `395c521` ("feat(capture): add in-panel successor
pull-in choice"): decoding, numbered preview rows, the in-panel choice step, `--pull-in`
on submit, the `--require-pull-in-choice` race guard with one write-free retry, the copy
fixes, and fake-bob fixtures/tests. Note #1 on the bead says the only remaining close
gate is GitHub Actions CI on the published commit. Note #2 is an existing
`PROPOSED FOLLOW-UP:` about the Mac host having no XCTest; leave it alone.

Verified facts:

- **CI on `395c521` is red at test compile.** Run 38457123354: Lint Swift formatting and
  Build passed; Test failed compiling
  `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift:269:19` and `:270:36` with
  "value of optional type 'String?' must be unwrapped to a value of type 'String'". The
  file's private `keyEvent(_:modifiers:repeat:)` helper declares
  `let characters: String? = keyCode == 49 ? " " : nil` and passes it to
  `NSEvent.keyEvent(...characters:charactersIgnoringModifiers:...)`, and both of those
  parameters take a non-optional `String`. That is the only compile error in the log.
  CaptureCoreTests compiled. Because the BobMacCaptureTests target never compiled,
  **none of the new pull-in tests has ever run**. Bundle, signature, launch smoke, and
  install were skipped.
- The same file has one warning at line 78: `var state` in
  `testMutedResidentsAndFoldedProjectCountsAreNeverSelectable` is never mutated.
- `CaptureKeyCommandRouter.pullInChoiceCommand` routes by key code and modifiers, not
  characters, so passing `""` instead of `nil` for non-Space keys changes nothing in
  behavior. The sibling helpers (for example
  `Tests/BobMacCaptureTests/CaptureDependencyPanelTests.swift`) already use
  `characters: String = ""`.
- **Master was already red before mac_choice.** Commit `2837025` (refs-inspector notes
  paging, from tale `plan:202610/refs_inspector_notes_paging.md`, which has no bead)
  failed run 38438769006 in `RefsInspectorScrollTests`:
  `testHostedNotesAtBottomRevealPages` (lines 392, 396, 652: "inspector offset did not
  settle around 2265.0 … docHeight=239.0") and
  `testHostedShortContentRevealsWithNoOffsetChange` (line 422: 3 ≠ 6). Nothing tracks
  this failure. The last fully green master is `4da688e` (run 38424226217).
- **No agent host can compile the Mac test targets.** apollo has no Swift. athena's
  `just portable-check` runs swift-format lint over `Sources`/`Tests` but cannot
  type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`. The mac's Command Line
  Tools have no XCTest. macOS CI is the compile and test authority for this file.

## Steps

1. **Open the repo.** Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`.
   Work only in the printed path.

2. **Fix the compile error.** In
   `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift`, change the helper to
   `let characters = keyCode == 49 ? " " : ""` (a non-optional `String`). Change the
   line-78 `var state` to `let` after confirming the test body never mutates it. Keep
   the file's existing 2-space indentation and swift-format style. Make no other edits
   unless step 5 calls for them.

3. **Local checks** (per `AGENTS.md`; tailnet access via
   `sase memory read tailnet.md -r "<why>"`):
   - On athena, run `just portable-check` from a fresh scratch copy (rsync without
     `.build`/`.git`), then delete the scratch copy. It must pass, since a red lint step
     on CI skips every later step.
   - Best effort on the mac: in a scratch copy (never Bryan's checkout, never
     `just install`), run `./Scripts/bundle.sh --identity -` and
     `./Scripts/xcode-swift.sh build --build-tests`, then delete the copy. The app
     product should compile. The test build is expected to stop at the known
     `no such module 'XCTest'` error. Record that and continue. Any other error must be
     fixed first. If the mac is offline, say so in the close note and final report.

4. **Publish.** Bryan asked for this bead to be finished and closed, and its close gate
   is CI on a published commit. So invoke `/sase_git_commit` in the bob-mac-capture
   checkout now; do not wait for the end-of-turn finalizer, which would publish too late
   to watch CI. Use a `test(capture): …` header and `-B keep` if `$SASE_BEAD_ID` is set.
   Then confirm `git status --short --branch` shows the branch clean and not ahead of
   upstream.

5. **Watch CI and triage.** Find the CI run for the pushed SHA
   (`gh run list --commit <sha> --workflow CI`; it can take a few seconds to appear).
   Wait on it with `/sase_monitor`, not inline. For example, monitor
   `gh run watch <run-id> --exit-status --interval 30` with a 60m timeout, distinct
   start/stop labels, and a `--next` that carries steps 5–7 of this plan. Then read
   `gh run view <run-id> --log-failed` and classify every failure:
   - **Lint or Build red:** fix it, rerun step 3, republish (step 4), and watch again.
   - **Failures in what `395c521` added or changed:** fix these in this tale. That
     covers `CapturePullInChoiceTests`, `CapturePullInChoiceModelTests`,
     `CapturePullInChoiceDesignTests`, `CapturePullInClientTests`,
     `CapturePullInDecodingTests`, `CaptureSuccessorPresentationTests`, and
     `NotificationServiceTests`, plus any other suite broken by its source or
     `Tests/Fixtures/fake-bob` changes. Fix whichever side is wrong against the
     mac_choice section of `plan:202610/successor_offers.md` (production code or test),
     and never weaken an assertion just to pass. Republish and watch again. Repeat until
     none remain.
   - **`RefsInspectorScrollTests` failing as on `2837025`:** this failure predates
     mac_choice and belongs to the refs-inspector paging tale, so do not edit
     refs-inspector code or tests here. Append one note to bob-cli-7c.7:
     `PROPOSED FOLLOW-UP: RefsInspectorScrollTests fail deterministically on macOS CI since 2837025 — <test names, assertion lines, and messages>; reproduced on runs 38438769006 and <new run id>; introduced by plan:202610/refs_inspector_notes_paging.md, which has no bead; masks the Bundle, signature, launch, and install gates for every later commit.`
     The epic's land agent triages it. It does not block closing bob-cli-7c.7; the close
     note must say those four gates stay unverified.
   - **Any other failure in code `395c521` did not touch** (known flake suspects:
     bob-cli-61's RefsPanelModelTests refresh-reorder timeout and bob-cli-4k's
     StartPending preview timeout): run `gh run rerun <run-id> --failed` once. If it
     passes on rerun, it is a flake; cite the existing bead in the close note or, if no
     bead covers it, append a `PROPOSED FOLLOW-UP:` note to bob-cli-7c.7. If it fails
     again and also fails on `2837025`'s run, treat it like the refs failure. Otherwise
     it is ours: fix it.

6. **Close the bead.** Run `sase bead close bob-cli-7c.7 --note "<evidence>"`. The note
   names: the fix commit SHA(s); the final CI run URL with each step's result; the
   pull-in suites that compiled and passed; every remaining failure with its
   classification (pre-existing, flake, or follow-up) and where it is recorded; which
   gates are still unverified; and the athena/mac local-check results. Do not close epic
   bob-cli-7c and do not touch bob-cli-7c.8 or the land bead; closing 7c.7 unblocks
   them.

7. **Report.** Tell Bryan what failed, what was fixed, the CI evidence, and any
   follow-ups recorded. If the mac was offline, say the macOS targets were not verified
   locally.

## Acceptance

- The latest master CI run on bob-mac-capture has Lint and Build green, and both test
  targets compile.
- Zero failures in the pull-in suites or in anything `395c521` changed.
- The only remaining failures, if any, are the pre-existing `RefsInspectorScrollTests`
  cases plus proven flakes, each recorded on bob-cli-7c.7 or cited by its existing bead.
- bob-cli-7c.7 is closed with the evidence note. bob-mac-capture is clean and pushed.

## Out of scope

- The Mac host's missing XCTest (already proposed in bead note #2).
- Fixing the `RefsInspectorScrollTests` failures from `2837025` (recorded as a proposed
  follow-up instead).
- bob-cli-7f, the lint failure in `CaptureTogglePresentationTests` (lint has passed on
  every run since `4da688e`). Leave that bead for its own triage.
- Installing the app on Bryan's mac (`just install`).
