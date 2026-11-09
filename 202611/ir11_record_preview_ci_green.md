---
tier: tale
title: Get ir.11 record-preview Mac tests green on CI
goal:
  Fix the two deterministic CaptureRecordPreviewPanelTests CI failures, gate on macOS
  CI, and close bob-cli-ir.11.
size: small
decisions:
  ci_watchdog:
    ask:
      Raise the macOS CI Test watchdog from 15 to 20 minutes so a slow runner cannot
      fail a passing suite?
    choices:
      raise:
        Watchdog fires at 1200 s, inside the 25-minute step timeout; real hangs surface
        5 min later
      keep:
        Watchdog stays at 900 s; a slow runner can fail a passing run, which gets one
        rerun and a note
    default: keep
    why:
      The watchdog is repo-wide hang policy outside this phase; typical runs take 11-12
      minutes
    answer: keep
proposed_by: bbugyi200.athena.bob-cli-ir.11
proposed_with: claude/opus
bead: bob-cli-ir.11
decided_by: auto
create_time: 2026-11-09 12:07:59
status: wip
---

- **BEAD:**
  [bob-cli-ir.11](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.11.md)

# Finish bob-cli-ir.11: get the record-preview Mac tests green on CI and close the phase

## Where the phase stands

Every ir.11 deliverable has already landed, and none of it needs redoing:

- **bob-cli** `84ff0bb8`: per-item `record_note` summary in capture JSON, with tests and
  docs.
- **bob-mac-capture** `fe9ce15`: `CaptureRecordPreviewPresentation`,
  `RecordNotePreviewView`, the refusal card, `CapturePanelModel` and
  `NotificationService` wiring, fixtures, and tests. `90fa59d` (bob-cli-ix, the
  sized-close test fix) sits on top of it and is master HEAD.

The bead stayed open for one reason: no macOS CI Test verdict. Bryan canceled both
earlier `Scripts/macos-ci-check` snapshot runs mid-Test. The master push runs for
`fe9ce15` (run 42656980197) and `90fa59d` (run 42657112529) have now finished, and both
are **red** in ir.11's own tests. The Mac has only Command Line Tools and no XCTest, so
CI is the only place these tests have ever run.

In run 42657112529 (`90fa59d`), all 4 failures of the 4493-test run are in
`Tests/BobMacCaptureTests/CaptureRecordPreviewPanelTests.swift`. Every other ir.11 suite
passed: `CaptureRecordPreviewDesignTests`, CaptureCore `CaptureRecordPreviewTests`, and
the `NotificationServiceTests` record cases. The two failures reproduce with identical
timings (17.6 s and 10.0 s) in both runs, so they are deterministic and not flakes. The
other failures in the `fe9ce15` run (`CapturePomodoroClosePresentationTests`,
`CaptureSeparatorToggleTests`) come from its parent `b81302b`, and `90fa59d` already
fixed them.

### Failure 1: `testExplicitPreviewOfCompleteDraftIsReadyWithSummary` (test bug)

The test calls `model.preview()` and then waits for `previewState == .ready`. Only the
live path (`startLivePreview` in `Sources/BobMacCapture/CapturePanelModel.swift`) ever
assigns `previewState = .ready(...)`. The explicit path (`preview()` →
`completePreview`) reports success through `assignPreviewOutputs`, which fills
`previewResults` and `previewResult` but never touches `previewState`. The wait
therefore can never succeed. Existing explicit-preview tests in
`CaptureMergeInPanelTests` and `CaptureInboxRouteChoiceModelTests` wait on
`!model.isPreviewing`, after a live preview has already made the state ready. This is
long-standing explicit-preview design, so do not change production code.

### Failure 2: `testDelayedStaleSuccessReplyNeverClobbersCurrentRefusal` (test race)

The model uses `debounceNanoseconds: 0`. The test performs two edits back to back, with
no await between them. The second `editorTextDidChange` bumps the analysis generation
and cancels the first analysis before that analysis spawns its dry run. fake-bob
therefore sees only one `capture --dry-run`. Its sequence counter
(`${FAKE_BOB_RECORD_PATH}.dry-run-count`) is 0 for that call, so it serves
`FAKE_BOB_CAPTURE_DRY_RUN_FIRST_FIXTURE`, the restaurant **success**, for the invalid
draft. The refusal never appears, and the wait times out after 15 s plus the 2.5 s
sleep, which matches the observed 17.6 s. Exit status is not a factor:
`BobProcessClient.decodeCaptureResult` decodes non-empty stdout using the `ok`
discriminator.

## Changes

All edits are in bob-mac-capture. Open it with `/sase_repo`
(`sase repo open bob-mac-capture`) and read its `AGENTS.md` first. The only file
expected to change is `Tests/BobMacCaptureTests/CaptureRecordPreviewPanelTests.swift`.
Format only the lines you change: `.swift-format` sets 4-space indentation and
100-column lines.

1. **Explicit preview test.** Keep it a pure explicit-path test. After
   `model.preview()`, wait for
   `!model.isPreviewing && model.previewResults.first?.recordNote != nil`, using the
   default 10 s bound. Keep the existing assertions: one result, the summary path
   `eat/cafe_møø_ramen.md`, and no `recordRefusalPresentation`. Add one assertion that
   `previewState` is not `.failed`. Do not assert `.ready`, because explicit preview
   never sets it.
2. **Stale reply test.** Make the first dry run really start before the second edit:
   - Add a `settledDryRuns()` helper that reads `recordURL.path + ".dry-run-count"` and
     returns 0 while the `.lock` symlink exists. Copy the exact pattern from
     `Tests/BobMacCaptureTests/CaptureClipPreviewHostedTests.swift`, roughly lines
     318–326.
   - Sequence: set `completeDraft` and call `editorTextDidChange`, then
     `await waitUntil(timeout: 10) { settledDryRuns() >= 1 }`. fake-bob increments the
     counter before it sleeps, so the first reply is still in flight. Only then set
     `invalidDraft` and call `editorTextDidChange`.
   - Keep the existing
     `waitUntil(timeout: 15) { model.recordRefusalPresentation != nil }`, the 2.5 s
     sleep, and the existing assertions. Add
     `XCTAssertGreaterThanOrEqual(settledDryRuns(), 2)` so the test proves the stale
     request reached Bob.
   - In the existing `defer`, also remove the `.dry-run-count` file and its `.lock`.
3. Change no production code unless CI shows failure 2 persisting after this fix. That
   would mean a real stale-reply bug in the live-preview guard
   (`isCurrentLivePreviewRequest` / `showRecordRefusal` in `CapturePanelModel`). Fix it
   there, and say so in the bead note.

> [!decision] ci_watchdog = raise In `.github/workflows/ci.yml`, in the Test step's
> watchdog loop, change `>= 900` to `>= 1200` and the `::error::` text from "15 minutes"
> to "20 minutes". Nothing else references the value. Leave `timeout-minutes: 25`
> unchanged.

> [!decision] ci_watchdog = keep Leave `.github/workflows/ci.yml` untouched.

## Verification (do not skip any compile step)

1. **Compile on the Mac.** `portable-check` cannot type-check `Tests/BobMacCapture*`, so
   this step is required, following the bob-mac-capture `AGENTS.md` procedure. Read the
   `tailnet.md` reference memory with `/sase_memory_read` for Mac access.
   - rsync the bob-mac-capture checkout to a fresh scratch directory on the Mac,
     excluding `.build` and `.git`. Never use Bryan's checkout or `just install`.
   - Run `Scripts/bundle.sh` and `just typecheck-tests`. All 4 test suites must
     type-check.
   - Run swift-format lint on the changed file(s): `swift-format lint`, or
     `./Scripts/xcode-swift.sh format lint <files>`.
   - Delete the scratch copy afterwards.
   - If the Mac is offline, say so in the bead note and the final report, and do not
     land uncompiled Swift.
2. **Gate on CI before landing.** Run `Scripts/macos-ci-check` from the bob-mac-capture
   checkout. It snapshots the working tree to a throwaway `ci/precommit/*` branch and
   waits up to 5400 s. Run it in the foreground with an explicit Bash timeout of at
   least 95 minutes, or hand it to `/sase_monitor`. Never end the turn while it runs.
   Required verdict: Check scripts, Lint, Build, **Test**, Bundle, Verify, Launch smoke,
   and Install all PASS. In the log, confirm both fixed tests passed and the
   `CaptureRecordPreviewPanelTests` suite reports 0 failures.
   - **Watchdog-only failure:** the Test step dies at the watchdog but the log shows 0
     failures and the xctest sample shows no stuck test. Treat that as a slow runner,
     not a hang, and rerun `Scripts/macos-ci-check` once. If it trips again, record a
     `PROPOSED FOLLOW-UP:` with both run ids and the Test-step durations. Typical green
     runs take 11–12 minutes. Run 42657112529 hit 900 s with xctest executing for 729 s.
   - **Person cancels a run:** if Bryan or anyone else cancels the snapshot run, do not
     start another. Commit the test-only fix anyway, because master is already red on
     exactly these two tests. Leave bob-cli-ir.11 open, and record the cancel in a bead
     note.
   - **Real test failure:** fix it and repeat steps 1–2.
3. **Close the phase** only after a full-green Test verdict:
   - Run `sase bead epic-symbols bob-cli-ir.11`. It should report none.
   - Run `sase bead close bob-cli-ir.11 --note "<verdict>"`. The note gives the run id,
     the snapshot SHA, the step verdicts, and the two fixed test names. It also states
     that bob-cli `84ff0bb8` and bob-mac-capture `fe9ce15` carry the phase's feature
     work.
   - Never close the epic `bob-cli-ir` or any other bead.
4. **Finish** through `/sase_final`, with a `commit` decision for bob-mac-capture. Use a
   message like `test(capture): settle record preview panel tests on CI`, or `fix(...)`
   if production code changed. Add a commit decision for the primary bob-cli checkout
   only if it changed; it should not.

## Out of scope

- Changing explicit-preview semantics so that `preview()` sets `previewState`. This is
  long-standing behavior for every capture kind.
- The pre-existing docs-check word-budget overage and the root help snapshot drift. Both
  are already recorded as `PROPOSED FOLLOW-UP:` notes on bob-cli-ir.11.
- Any memory note edits. Under the epic DECISIONS, only `glossary:capture` may be
  edited, and this work leaves no memory note stale.
