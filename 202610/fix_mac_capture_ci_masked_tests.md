---
tier: tale
title: Fix bob-mac-capture CI test compile error and the masked test failures
goal:
  bobs-org/bob-mac-capture CI (macOS 26 SwiftPM) is observed green on master, with the
  BobMacCaptureTests target compiling and every previously masked test passing.
size: medium
proposed_by: bbugyi200.athena.0w2
create_time: 2026-10-04 05:47:25
status: wip
---

# Fix bob-mac-capture CI: unbreak the test-target compile and the tests it has been masking

## Goal and scope

Get `bobs-org/bob-mac-capture` GitHub Actions (`CI` workflow, job `macOS 26 SwiftPM`)
green again on `master`, with every step passing: Lint, Build, Test, Bundle, plist and
signature check, launch smoke test, and install/reinstall. All changes belong in the
linked **bob-mac-capture** repository. The work is mostly tests and fake-bob fixtures,
plus one line of production code (§3) that fixes a real bug in the shipped vault-wide
`+` picker. No bob-cli, workflow, or Scripts changes are needed.

Before reading or editing, use `/sase_repo`:

```sh
sase repo open bob-mac-capture -r 'Fix failing macOS CI: test compile error and the masked test failures'
```

Use the printed path for every read and write. Paths below are relative to that
checkout.

This plan resolves the existing task beads **bob-cli-3m** (the `.pending` compile error)
and **bob-cli-3x** (close tests that fail once the suite compiles). Read both with
`sase bead read <id> -r '<why>'` before starting.

## Diagnosis (already established; do not re-derive)

`actstat` and `gh run list -R bobs-org/bob-mac-capture` show `master` red for its last 8
CI runs, starting with `15f930e` (Oct 2). The last green run was `b8b054f` (run
37025611761). Lint and Build pass in every red run. The job dies at step 6 (`Test`).

1. **Persistent root cause: a test-target compile error since `aa1e73a`** (runs
   37041777255, 37057373509, 37139244480, 37157187639, 37158683771, 37159557402 on HEAD
   `fdd73ad`). In `testCloseAliasInvalidBlocksSubmissionAndClearsStaleCard`,
   `Tests/BobMacCaptureTests/CapturePanelModelTests.swift:2024` does
   `if case .pending = model.previewState`, but `CapturePreviewState` in
   `Sources/BobMacCapture/CapturePanelModel.swift` has only `idle`, `loading`,
   `ready(_)`, and `failed(_)`. This is the only compile error in the log. A pending
   close card is really `.ready(success)` on the **trimmed** draft, with
   `closePendingText` / `isClosePending` set and the pending action disabled (see
   `startLivePreview`). **Do not add a `.pending` enum case** (bob-cli-3m says the
   same).
2. **Since `aa1e73a`, no test in `BobMacCaptureTests` has run in CI.** Commits
   `aa1e73a`, `68ed00d`, `680e17f`, `ebee52d`, `e9b5f81`, `098e67e`, and `fdd73ad` added
   about 50 app-level tests that have never executed. A static audit of those tests
   checked the panel model and `Tests/Fixtures/fake-bob` against real `bob` output
   (read-only `bob capture-parse`, `bob capture-complete`, and `bob capture --dry-run`).
   It predicts four more deterministic runtime failures once the compile is fixed. §§1–4
   below fix them.
   - Two are fake-bob gaps of one kind. fake-bob's `capture)` handler falls through to a
     generic **successful** task capture (`"text": "captured"`) for any draft it has no
     branch for. So a test that waits for `.failed`, on a draft real Bob rejects, times
     out.
   - Another is a real app bug from `e9b5f81`: a lone `+` never asks Bob for completion,
     so the vault-wide picker never opens.
3. **CaptureCoreTests are healthy.** `swift test` on Linux (Swift 6.0.3) at `fdd73ad`
   runs 655 tests with 0 failures. bob-cli-3x item 1
   (`testShortAliasPreviewShowsDefaultTaskOne`, "Parked" vs "parked") is already gone:
   `fdd73ad` replaced that test with `testShortAliasPreviewShowsWildcardOutcomes`, which
   passes. No CaptureCore change is needed.
4. **Already fixed, no action:** run 37134501063 (`680e17f`) failed compiling production
   code (`argument 'taskRef' must precede argument 'notePath'`); `ebee52d` fixed it.
5. **Suspected flake, no code change:** run 37029421396 (`15f930e`, before the compile
   break) failed only `testStartPendingListPreviewsTrimmedDraftWithStartDisabled`. It
   hit the 5 s `waitUntil` timeout after `=~2`. That commit touched only task-row
   strikethrough token splitting. The test passed in the four runs before it and in
   bob-cli-3x's out-of-CI run. Handle it only if it recurs (see Validation).
6. Since `b8b054f` (the last green run), nothing has changed `Scripts/`, `.github/`,
   `Resources/`, `Package.swift`, the install helper, or the justfile. The steps after
   Test are not expected to fail.

## Implementation

### 1. Rework `testCloseAliasInvalidBlocksSubmissionAndClearsStaleCard` (bob-cli-3m)

File: `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` (around lines 1998–2029).

- **fake-bob `=*abc` failure branch.** In the `capture)` section of
  `Tests/Fixtures/fake-bob`, add a branch for `=*abc`. Put it next to the other alias
  failures (for example near the `=*3` / `=x1*` branches). It should `cat` a new
  fixture, `Tests/Fixtures/pomodoro-close-select-alias-bad.json`, then `exit 2`. Use
  real Bob's exact single-line output (verified:
  `bob capture --dry-run --no-clip --format json -- '=*abc'` exits 2 with):

  ```json
  {
    "error": "capture item 1 starting on line 1: `=*abc` is not a task list: write `=x`, then comma-separated task numbers, then optionally `*` and the numbers to park, `!` and the numbers to complete, and `~` and the numbers to drop (for example `=x1*2!3~4`)",
    "ok": false
  }
  ```

  `BobProcessClient.decodeCaptureResult` decodes `ok:false` JSON whatever the exit
  status is. The live preview then becomes `.failure`: `previewState` is set to
  `.failed` and the stale card is cleared. The test's existing `.failed` wait and its
  `XCTAssertNil(model.closePresentation)` then hold. Keep them.

- **Pending step.** Give the model a `FAKE_BOB_RECORD_PATH` record file, using the same
  pattern as `testClosePendingListPreviewsTrimmedDraftWithCloseDisabled`. Replace the
  `.pending` wait after `=*1,` with `await waitUntil { model.closePendingText != nil }`.
  Then assert all of the following:
  - `closePendingText == "Type a task number after ,"`
  - `isClosePending`
  - `closePendingAction == "Close"`
  - `statusText == "Type a task number after , — Close is disabled"`
  - `primaryActionTitle == "Close"`
  - After `model.submit(openAfterCapture: false)`: `XCTAssertFalse(model.isSubmitting)`,
    the record contains `"argv=capture --dry-run --no-clip --format json -- =*1\n"` (the
    trimmed draft was previewed), and the record does **not** contain
    `"argv=capture --format json -- =*1,"`.
- **Remove the final `XCTAssertNil(model.closePresentation)`.** It holds only because
  fake-bob has no `=*1` capture branch and serves its generic non-close capture. In
  production a pending card keeps the dimmed close card for the trimmed draft (non-nil),
  so the assertion contradicts the model's design. Keep the comment's intent ("A
  genuinely incomplete draft disables submission with a pending card"). Its parse
  fixture `pomodoro-close-parse-alias-incomplete.json` already matches real Bob byte for
  byte.

### 2. `testMixedDraftSurfacesBobDiagnostic` (bob-cli-3x item 2): fake-bob only

- The parse for `=x\n- foo\n- 2 bar` (`close_log_mixed_draft` in fake-bob) is `ok:true`
  with an error diagnostic and no `needs`. So the model correctly runs the live dry run,
  which is the only path to `.failed`. fake-bob just lacks the failing `capture` branch.
- In the `capture)` section, next to the existing `close_log_invalid_draft` branch, add
  a `close_log_mixed_draft` branch. It should `cat` a new
  `Tests/Fixtures/pomodoro-close-log-mixed.json` (named after the existing
  `pomodoro-close-log-invalid.json`), then `exit 2`. Real Bob output (verified, exit 2):

  ```json
  {
    "error": "capture item 1 starting on line 1: Work Log bullets are numbered all or none, and the first one has no task number; to log text that starts with `2`, number every bullet: `- 1 2 bar`",
    "ok": false
  }
  ```

- No test or model change.

### 3. Lone `+` never requests completion (production fix; e9b5f81 bug)

- Real Bob's `capture-parse '+'` returns a valid `mode: pomodoro_adjust`, `needs: []`,
  and one `pomodoro_adjust` span (fixture `parent-task-parse-plus.json` matches). Yet
  `bob capture-complete --cursor 1 -- '+'` returns `context: task_parent` with the vault
  picker descriptor and `action_continuation_keys`.
- `shouldRequestCompletion` in `Sources/BobMacCapture/CapturePanelModel.swift` (the
  `completionSpanKinds` set, around line 4677) does not include `pomodoro_adjust`. So
  the app never asks, and the shipped lone-`+` vault picker can never open.
  `testPlusOpensVaultPickerFromFakeBob` and `testSelectionShowsChipNotPicker` both time
  out because of this.
- **Fix:** add `"pomodoro_adjust"` to `completionSpanKinds`. Next to the existing
  span-kind comments, add a short comment saying why:
  - a lone whole-item `+` is dual-use (`plan:202610/plus_task_picker.md`, rule 5);
  - Swift asks and Bob decides;
  - every other adjustment gets `context: null` and no candidates. Verified with real
    Bob for `+0`, `+2`, `+5`, `-`, and `-2`.

  This keeps to the thin-client rule: no Swift-side syntax classification.

- **fake-bob follow-up.** Adjust drafts now request completion, so extend the existing
  null-context `capture-complete` branch to cover `"+0"` and `"-"` as well. That is the
  branch whose condition lists `"+5" || "+2" || "++" || "-2" || ...` (around line 2159);
  real Bob returns null context for both drafts. Without this, existing `+0` tests
  (around lines 3339, 3374, 6031, and 6059 of `CapturePanelModelTests.swift`) and `-`
  tests would get fake-bob's generic `route`/`today` completion fallback.
- Then grep fake-bob's `capture-parse)` section for every draft whose response has a
  `pomodoro_adjust` span. Today those are `+`, `+2`, `+5`, `-2`, `+0`, `-`,
  `+5\n\njot idea`, and the `close_batch_draft` `-2\n\n=x`. For each, confirm that every
  test using it either puts the cursor outside the adjust span or hits a null-context
  completion branch.

### 4. `testPlusOpensVaultPickerFromFakeBob`: drop the wrong `.idle` assertion

- File: `Tests/BobMacCaptureTests/CaptureParentTaskPanelTests.swift` (around line 372).
- `XCTAssertEqual(model.previewState, .idle)` contradicts the approved plus-picker
  design. `plan:202610/plus_task_picker.md` says "Do not mark it incomplete, suppress
  its valid dry-run" and "a solo `+` continues to preview the adjustment and shows the
  Escape hint". Opening the picker never resets `previewState`.
- Replace the assertion with a check that the adjustment dry run was not suppressed.
  Build the model with `parentTaskModel(recordURL:)` and wait until the record contains
  `"argv=capture --dry-run --no-clip --format json -- +\n"`.
- Keep every other assertion; after §3 the audit traced them as passing.

### 5. `testExactIdentifiedScopedTaskSuppressesAutoOpen`: missing fake-bob parse branch

- fake-bob's `capture-parse)` section has no branch for `@cash+goog-exit`. Its generic
  reply has no needs or spans, so completion is never requested and the record wait at
  around line 448 times out.
- Add a branch serving a new `Tests/Fixtures/parent-task-parse-scoped-exact.json`, which
  pairs with the existing `parent-task-complete-scoped-exact.json` that the
  `capture-complete` branch for `@cash+goog-exit` already serves. Use real Bob's output
  (verified):

  ```json
  {
    "ok": true,
    "schema_version": 1,
    "input": "@cash+goog-exit",
    "body": "",
    "mode": "task_toggle",
    "route": "cash",
    "section": null,
    "block_id": "goog-exit",
    "needs": [],
    "spans": [
      { "start": 0, "end": 5, "kind": "task_toggle_route" },
      { "start": 6, "end": 15, "kind": "task_toggle_block_id" }
    ],
    "diagnostics": []
  }
  ```

- Cursor 15 sits in `task_toggle_block_id`, so completion is requested. The exact-match
  suppression then keeps the picker closed, as the test intends.

### Style

Match the surrounding Swift (4-space indent, existing `waitUntil` and record-file
patterns, comment density) and the fake-bob branch style (`cat` a fixture, then `exit`).
Fixtures are compact single-line JSON with a trailing newline, like their neighbors.

## Validation

No machine we have can run `BobMacCaptureTests`. Linux never compiles the AppKit targets
(`#if os(macOS)`). The `mac` tailnet host has only Command Line Tools, which ship no
`XCTest` ("no such module 'XCTest'"). **GitHub Actions is the only real test gate**, so
the work is not done until CI is observed green.

1. **Local checks before pushing:**
   - `bash -n Tests/Fixtures/fake-bob`.
   - `python3 -m json.tool` on each new fixture.
   - Run fake-bob directly for each new or extended branch and check stdout and exit
     code. Examples:
     `Tests/Fixtures/fake-bob capture --dry-run --no-clip --format json -- '=*abc'; echo $?`,
     plus `capture-parse --format json -- '@cash+goog-exit'` and
     `capture-complete --cursor 2 --format json -- '+0'`. Read the top of fake-bob for
     how it dispatches arguments.
   - On Linux, run `swift test` in a scratch copy (for example `git archive` or a copy
     of the working tree into `mktemp -d`) to confirm CaptureCore still passes 655/655.
   - Run `swift format lint --recursive Package.swift Sources Tests`; Linux has
     swift-format 600.
   - Best effort, if `ssh mac` answers: copy the working tree to a temp dir on the mac
     and run `./Scripts/xcode-swift.sh build` and `just format-lint` there. This catches
     production compile errors. Clean up the temp dir afterward.
   - Optional: bob-cli-3x describes a throwaway XCTest stand-in that let the suite run
     on the CLT-only mac. Don't commit anything like that.
2. **Commit and watch CI (explicitly authorized by this plan).** Once local checks pass,
   use `/sase_git_commit` from the bob-mac-capture checkout to land the change on
   `master`. Use a `test(capture):`/`fix(capture):` conventional header; §3 is a `fix`.
   Then find the run with
   `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
   Watch it with `gh run watch <id> -R bobs-org/bob-mac-capture --exit-status` through
   `/sase_monitor`, with a `--next` that tells the follow-up to read the result and
   continue this loop.
3. **Iterate until green.** On failure, read
   `gh run view <id> -R bobs-org/bob-mac-capture --log-failed` and grep for ` error:`
   and `failed`.
   - Fix genuine failures in tests the audit did not predict the same way: prefer
     correcting fake-bob or the test to match real Bob output (check it with read-only
     `bob capture-parse` / `capture-complete` / `capture --dry-run`), and change the
     model only when it contradicts its approved plan.
   - Commit again and re-watch. The whole job must pass: Lint, Build, Test, Bundle,
     plist/signature, launch smoke test, and install/reinstall.
4. **If `testStartPendingListPreviewsTrimmedDraftWithStartDisabled` fails again:** run
   `gh run rerun <id> --failed -R bobs-org/bob-mac-capture` on the unchanged tree.
   - If it then passes, it is a flake: use `/sase_new_task` to file it, or +1 an
     existing flake bead.
   - If it fails deterministically, diagnose and fix it as part of this work.
5. **After a green run:**
   - Close bob-cli-3m and bob-cli-3x with
     `sase bead close <id> --note '<what you verified, incl. green run URL/SHA>'`.
   - In bob-cli-3x's note, mention that item 1 was superseded by `fdd73ad` and item 2
     was fixed by the fake-bob branch.
   - In the final report, state plainly whether CI was observed green, and give the run
     URL.

## Out of scope

- The other repos `actstat` lists as failing (`sase-org/sase`, `sase-org/sase-core`).
- The Node.js 20 deprecation notices and the Swift 6 `Sendable` warnings in the CI log;
  both are warnings only.
- bob-cli-1x (documenting the watch-CI-after-push convention in memory) is a separate
  bead. Follow its practice here, but don't edit memory files.
- Fixture drift the audit noticed, which causes no test failure today:
  - `pomodoro-close-select-alias-long-park.json` and `-long-complete.json` (`=x*` /
    `=x!`) still show only task 1 under `fdd73ad`'s wildcard semantics.
  - fake-bob still maps the `=x1*` parse/capture to the repurposed
    `pomodoro-close-parse-alias-conflict.json` (now the `=*!` fixture).

  If worth tracking, file one follow-up with `/sase_new_task` rather than fixing it
  here.
