---
tier: tale
title: Correct capability-aware CapturePanelModel submit assertions
goal:
  Make the three reported submit-argv tests pass with strict capability-aware
  expectations and real macOS XCTest evidence.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.4
create_time: 2026-10-24 10:02:07
status: wip
---

- **PARENT:**
  [202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.4.md)

# Correct capability-aware CapturePanelModel submit assertions

Complete only phase `bob-cli-an.6.6.6.5.6.4` of
`plan:202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md`.
The tracked CI task is `bob-cli-be`; leave its closure to the parent epic's lander. The
phase is in progress, with no earlier-work notes or existing child plan at planning
time. This is one bounded test correction and verification task, so a tale is
sufficient.

## Evidence and root cause

Open `bob-mac-capture` through `/sase_repo`, use only the path it prints, and read its
`AGENTS.md`. Investigation used clean bob-cli `453b07e7` and linked client
`f9cd88add5f2b4bdb8ca5a0dffde9b42617b7253`. Recheck the current tree before editing.

The original macOS CI run is
https://github.com/bobs-org/bob-mac-capture/actions/runs/40095658827, on client
`6d41b2e`. Its Test log confirms exactly three failures among 2,938 tests:

- `CapturePanelModelTests.testBareXSelectionPreviewSubmitsUntouchedDraft`, line 2472,
  only the `x*!2` case;
- `CapturePanelModelTests.testOverrideRestartLivePreviewAndSubmitUseRestartArgvFooterAndStatus`,
  line 3260;
- `CapturePanelModelTests.testUnpreviewedOverrideRefusalKeepsDraftAndSecondReturnCommits`,
  line 3359.

The task's suspicion that fake-bob never advertises pull-in support is false for the two
relevant fixture branches. Running the unchanged
`Tests/Fixtures/fake-bob capture --dry-run --no-clip --format json -- <draft>` gives
this matrix:

| Draft and environment              | Root `pull_in.supported` | Override |
| ---------------------------------- | ------------------------ | -------- |
| `x*`, `x!`, `x*2`                  | absent                   | absent   |
| `x*!2`                             | true, zero offers        | absent   |
| `=`, idle                          | absent                   | absent   |
| `=`, `FAKE_BOB_POMODORO_RUNNING=1` | true, zero offers        | restart  |

The supporting JSON files are
`Tests/Fixtures/pomodoro-close-select-park-all-complete-two.json` and
`Tests/Fixtures/pomodoro-start-override-restart.json`. Both explicitly advertise
`{"supported":true,"offer_count":0,"choice":null,"picked_count":0}` under `pull_in`. The
script logs argv at entry, before choosing either response branch.

`Sources/BobMacCapture/CapturePanelModel.swift` stores the response's `pullIn` in
`makePullInPreviewSnapshot`, bound to the draft, executable identity, and analysis
generation. `submit` passes the positive capability to `dispatchCapture`.
`Sources/CaptureCore/BobProcessClient.swift::capture` emits `--require-pull-in-choice`
when both `requirePullInChoice` and `pullInCapabilityAdvertised` are true. Zero current
offers does not disable the guard. The same current preview determines whether
`--refuse-override` is needed.

Commit `54131b1` removed `--require-pull-in-choice` from the expectations of all three
tests, including these capability-positive cases. Thus the old expected substring no
longer matches the correctly guarded submit. Preserve the capability gate and the
fixtures; correcting strict expected arguments fixes the defect in the tests.

## Implementation

1. Read the assigned phase and task through `sase bead read`, and the parent design
   through `sase artifact read`. Honor its final decisions. Do not change memory,
   application behavior, capture grammar, package versions, release files, or workflows.
   Preserve `decisions:mac-capture-is-a-thin-client`.

2. In `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`, make the bare-x cases an
   explicit expectation table: retain all four drafts, with only `x*!2` expecting a
   capability guard. Assert the ready preview's advertised capability matches this
   independent table before submission. Require these exact write argv lines:

   ```text
   argv=capture --format json --refuse-override -- x*
   argv=capture --format json --refuse-override -- x!
   argv=capture --format json --refuse-override -- x*2
   argv=capture --format json --require-pull-in-choice --refuse-override -- x*!2
   ```

   For each case, filter the record's single-line write invocations and compare the
   complete ordered array to the one expected line, with draft and record diagnostics.
   Keep the live-preview argv, Close action, close-card, and no-picker assertions.
   Require successful submission with the original draft intact in argv. Do not derive
   expected flags from the actual write record or make either flag optional.

3. Correct the two restart tests using the same exact-line discipline:
   - The live-preview restart submits exactly
     `argv=capture --format json --require-pull-in-choice -- =`, once. Verify the
     preview advertised support and retain the restart footer/status, preview argv, and
     absence of `--refuse-override` assertions.
   - The unpreviewed refusal test submits first
     `argv=capture --format json --refuse-override -- =`, then after its automatically
     fetched preview submits `argv=capture --format json --require-pull-in-choice -- =`.
     Assert that exact ordered pair, retain the interim draft-preservation, refusal
     message, override presentation, preview argv, cleared final draft, and successful
     restart status.

   Keep the comparisons local to these single-line drafts; do not build a new general
   parser for the multiline fake-bob log. Preserve missing-capability, stale-preview,
   and unsupported-flag fallback coverage in adjacent tests and the existing
   `CapturePullInClientTests` / `CapturePullInChoiceModelTests`.

## Verification and macOS execution

The planning-time Mac is reachable, but `xcode-select -p` reports
`/Library/Developer/CommandLineTools`; `xcrun --find xctest` fails and
`xcodebuild -version` reports that full Xcode is required. The linked repo's `AGENTS.md`
confirms that `just typecheck-tests` uses a non-executing XCTest stub on this host. This
is a known infrastructure limit, not a reason to claim the tests passed or install
Xcode.

1. Recheck the fixture capability matrix and baseline CI evidence. If an existing
   full-Xcode Mac environment is now available, reproduce the three failures there
   before editing. Otherwise use the recorded real XCTest baseline above and identify
   that limitation explicitly; do not spend time retrying unavailable XCTest in CLT.

2. Run the linked repo's `just portable-check`. It lints all Swift files and executes
   the portable targets, including existing positive/negative capability-gate tests. It
   does not execute `BobMacCaptureTests`. Use `/sase_monitor` for long checks, with
   explicit continuation instructions that retain this plan and bead obligations.

3. On the Mac, use only a unique disposable scratch copy populated from the opened
   linked checkout with rsync excluding `.build` and `.git`. Run
   `./Scripts/bundle.sh --identity -` and `just typecheck-tests`, record their results,
   and remove the scratch copy when finished. Never touch Bryan's checkout, run
   `just install`, launch the app against the live vault, or change developer-tool
   selection globally. If full Xcode is available, run the three named tests with
   `./Scripts/xcode-swift.sh test --filter <test-filter>`, then the full
   `BobMacCaptureTests` suite. A stub typecheck remains compilation-only evidence.

4. Obtain actual macOS XCTest execution on the changed commit. With the current CLT
   host, the existing GitHub Actions `macOS 26 SwiftPM` job is the execution path: its
   Test step runs all targets, including the three named tests. To make that
   verification concrete before closure, this plan instructs the implementation agent to
   invoke `/sase_git_commit` for the reviewed, locally validated test correction, with
   `-B keep`, a `test(capture): ...` message, and the configured SASE commit method.
   Never use raw `git commit`. Resolve rebase conflicts under that skill and verify the
   resulting tree. Record the actual published SHA and its CI run URL. Use
   `/sase_monitor` to wait for that run; every handoff must name the phase, SHA,
   remaining checks, and required close action. A handoff does not complete the bead.

5. Inspect the exact-SHA CI log for passing executions of all three tests and the full
   `BobMacCaptureTests` suite, not merely a successful build or a skipped Test step.
   Check later CI stages even if lint fails. Fix failures caused by this change. For
   unrelated failures, reproduce the same failure on the clean base and record
   `PROPOSED FOLLOW-UP:` on this phase, citing an existing task owner when known; those
   proven baseline failures do not keep the phase open. Do not weaken assertions, remove
   `x*!2`, strip fixture capability data, change production guards, accept a missing
   write record, or increase timeouts to conceal the deterministic mismatch.

## Completion

Record root cause, exact expected argv, local verification, Mac compilation limits, and
actual macOS XCTest/CI evidence in a note on `bob-cli-an.6.6.6.5.6.4`. Give the parent
lander the linked-repo SHA and CI URL to verify and close `bob-cli-be`. Create no task
beads; use `PROPOSED FOLLOW-UP:` notes on this phase for discoveries.

Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.4` immediately before closing. Planning
found no entries; resolve any that appear or re-key them to a still-open owning bead.
Then close only `bob-cli-an.6.6.6.5.6.4` with
`sase bead close ... --note "<verified results>"`. Never close the task, parent epic, or
any ancestor plan bead. Do not set runtime-owned statuses by hand. Finish through
`/sase_final` when not performing a successful monitor handoff, accounting for every
changed repository.
