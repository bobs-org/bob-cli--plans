---
tier: tale
title:
  Fix bob-mac-capture CI test-target compile errors and stale relation-badge assertion
goal:
  bob-mac-capture's macOS CI workflow is green again on master, verified by a full macOS
  CI run before landing.
size: medium
decisions:
  relation_badge:
    ask:
      How should the stale final badge assertion in the refused-cancel relation test be
      fixed?
    choices:
      expect_prefix:
        Assert "one-off · 1 link", pinning that one-off mode survives row selection
      reset_one_off:
        Turn one-off off before the last selection and keep asserting "1 link"
    default: expect_prefix
    why:
      One-line change that pins the documented sticky one-off behavior instead of hiding
      it
    answer: expect_prefix
proposed_by: bbugyi200.athena.16d
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 11:34:24
status: wip
---

# Fix bob-mac-capture CI: uncompilable test target and a stale relation-badge assertion

## Diagnosis

`bobs-org/bob-mac-capture` CI has been red on `master` for nine consecutive pushes (CI
runs #492–#500, commits `c8c4366` through current HEAD `2f9a70a`). The last green run
was `2c3407d`. Every failure is in the `macOS 26 SwiftPM` job, step **Test**. Check
scripts, lint, and Build pass, while Bundle, Verify, Launch smoke test, and Install are
skipped after the red Test step. There are two independent root causes, and the second
hides the first.

### Cause 1: stale assertion (runs #492–#495: `c8c4366`, `5726b4f`, `072055d`, `d41bb61`)

One assertion fails out of about 3,300 tests:

```text
CaptureRelationPanelTests.swift:1566: error: -[BobMacCaptureTests.CaptureRelationPanelTests
testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible] : XCTAssertEqual failed:
("Optional("one-off · 1 link")") is not equal to ("Optional("1 link")")
```

`c8c4366` added this test. Partway through, it sets
`model.relationLabelPrompt?.oneOff = true` to show that refused rows never offer ↩, even
in one-off mode. Later it calls `selectRelationLabelSuggestion(key: "duplicate of")` and
expects the badge to read `"1 link"`.

The app code is correct, and the last assertion is stale:

- `selectRelationLabelSuggestion(key:)` changes only `selectedKey`.
- One-off is sticky prompt state. Per `docs/keyboard.md`, a typed `|` turns it on, and
  only Backspace on an empty field or Ctrl-D with no selection turns it off.
- Since `556bb67`, `relationLabelPromptBadgeText` (in
  `Sources/BobMacCapture/CapturePanelModel.swift`) prefixes `one-off · ` whenever
  `prompt.oneOff` is set.

### Cause 2: `BobMacCaptureTests` does not compile (runs #496–#500: `031f100` → `2f9a70a`)

The Test step fails while compiling the test target, so no tests run at all. That also
hides cause 1. There are 13 errors of the form `argument 'X' must precede argument 'Y'`:

- `Tests/BobMacCaptureTests/CaptureWikilinkTaskPanelTests.swift` (added by `031f100`):
  the four fixture builders `identifiedCandidate()`, `idlessCandidate()`,
  `historyCandidate()`, and `disabledCandidate()`.
- `Tests/BobMacCaptureTests/CapturePickerDesignTests.swift` (added by `8f303e2`): all
  nine `CaptureCompletionCandidate(...)` calls inside the static
  `wikilinkTaskRenderCandidates` fixture.

These calls pass `notePath:` and `locator:` before `linkTarget:` and `cursorAfter:`.
`CaptureCompletionCandidate`'s `public init` in
`Sources/CaptureCore/CaptureModels.swift` declares its parameters in this order:

`… text, section, depth, childCount, cursorAfter, linkTarget, path, … blockIDSuggestions, group, scheduled, pullsForward, notePath, locator, … disabledReason, hidden, …`

Swift requires labeled arguments in declaration order. The init did not change; the
tests were written in the wrong order and never type-checked.

### Evidence gathered while planning

- On the `mac` tailnet host, `Scripts/typecheck-tests.sh` ran against a scratch copy of
  `master`. It reproduced exactly these 13 errors (`FAIL BobMacCaptureTests`), and
  `BobMacCaptureInstallHelperTests`, `CaptureCoreTests`, and `RefsCoreTests` passed.
- The same scratch copy was then patched with the argument reorder below and with the
  `expect_prefix` assertion change. All four test targets passed type-check, and
  `swift-format lint` on the three files exited 0, with no warnings on changed lines.
  The scratch copy was deleted.
- `just portable-check` on athena passes on current `master` (1781 portable tests). The
  failing files are macOS-only, so that gate cannot see them.

## Scope

This is a test-only change in the linked `bob-mac-capture` repository: three test files,
plus anything step 5 finds.

- Do not change the parameter order of `CaptureCompletionCandidate.init`. Many call
  sites and the Codable decoder depend on it.
- No `Sources/`, `docs/`, or workflow changes are expected.
- `bob-cli`'s own CI failures (lint and macOS lint) are out of scope.

## Steps

### 0. Open the repository

Use `/sase_repo` (`sase repo open bob-mac-capture -r "<reason>"`) and work only in the
path it prints. Read that repository's `AGENTS.md` first: it covers the compile rules,
house formatting (4-space indent, 100 columns, format only the lines you change), and
`mac` scratch-copy etiquette.

### 1. Reorder the 13 `CaptureCompletionCandidate(...)` calls

The rule for each failing call:

- Move `cursorAfter:` and then `linkTarget:` (in that order) to directly after `text:`,
  or after `section:` when the call has one. They then come before
  `blockIDSuggestions:`, `group:`, `notePath:`, and `locator:`.
- Leave `disabledReason:` and `hidden:` after `locator:`.
- Move trailing commas so that only the last argument has none.
- Change only the calls in the four fixture builders and in
  `wikilinkTaskRenderCandidates`. Other `CaptureCompletionCandidate(` calls in these
  files already compile; leave them alone.

Example (`identifiedCandidate()`):

```swift
        CaptureCompletionCandidate(
            replacement: "[[sase_cleanup#^no-idle]]",
            taskRef: "1:fa53e656",
            blockID: "no-idle",
            requiresBlockID: false,
            statusSymbol: "/",
            statusName: "In Progress",
            statusType: "IN_PROGRESS",
            text: "Stop idling when no agents run",
            cursorAfter: 35,
            linkTarget: "sase_cleanup",
            group: "in_progress",
            notePath: "sase_cleanup.md",
            locator: "sase_cleanup"
        )
```

Two variants follow the same rule:

- **ID-less rows** (no `cursorAfter`):
  `…, text:, linkTarget:, blockIDSuggestions:, group:, notePath:, locator:`.
- **The `hidden-chore` design row**:
  `…, text:, cursorAfter: 24, linkTarget: "sase", group: "open", notePath: "sase.md", locator: "sase", hidden: true`.

### 2. Fix the stale assertion

The target is the last line of
`testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible` in
`Tests/BobMacCaptureTests/CaptureRelationPanelTests.swift`.

> [!decision] relation_badge = expect_prefix Change
> `XCTAssertEqual(model.relationLabelPromptBadgeText, "1 link")` to
> `XCTAssertEqual(model.relationLabelPromptBadgeText, "one-off · 1 link")`.

> [!decision] relation_badge = reset_one_off Insert
> `model.relationLabelPrompt?.oneOff = false` directly before
> `model.selectRelationLabelSuggestion(key: "duplicate of")`, and keep the `"1 link"`
> assertion.

Do not change `relationLabelPromptBadgeText` or `selectRelationLabelSuggestion`.

### 3. Verify locally (never land Swift you have not compiled)

1. On athena, run `just portable-check` (format lint plus the portable targets). It must
   pass.
2. On the `mac` (best effort, per `AGENTS.md`):
   - rsync the checkout to a unique scratch directory under `/tmp`, excluding `.build`
     and `.git`.
   - With `/opt/homebrew/bin` on `PATH`, run `./Scripts/bundle.sh`, then
     `./Scripts/typecheck-tests.sh`. Call the scripts directly, because `just` may not
     be on the non-interactive SSH `PATH`.
   - Expect `PASS` for all four targets: `BobMacCaptureTests`,
     `BobMacCaptureInstallHelperTests`, `CaptureCoreTests`, and `RefsCoreTests`. The
     first build takes about 5 minutes.
   - Delete the scratch copy. Never use Bryan's checkout or `just install` there.

   If the `mac` is offline, say in the final report that the macOS targets are
   unverified locally, and rely on step 4.

### 4. Run real macOS CI before landing

The Mac has only Command Line Tools and no XCTest, so CI is the only place the tests
run. From the repository checkout, run `Scripts/macos-ci-check`:

- It snapshots the working tree into a throwaway commit object and pushes it to a unique
  `ci/precommit/*` branch.
- It waits for the CI workflow on that SHA and prints every macOS step's verdict.
- It deletes the remote branch on every exit path, and never touches `HEAD`, the index,
  or `master`.

A run takes roughly 15–25 minutes, and the script's total deadline is 5400 s. Either run
it in the foreground with an explicit Bash timeout of at least 5,700,000 ms, or hand it
to `/sase_monitor`.

| Exit | Meaning                                                                                                   | Next step                                         |
| ---- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| 0    | The whole workflow is green, including Bundle, Verify plist and signature, Launch smoke test, and Install | Go to step 6                                      |
| 1    | A real CI failure                                                                                         | Go to step 5                                      |
| 2    | Verification incomplete (auth, discovery, deadline, or cleanup)                                           | Read its message, fix the precondition, and rerun |

### 5. Fix failures that were hidden until now

Tests added by `031f100`, `8f303e2`, `7ac2bd9`, `2609650`, and `2f9a70a` have never run
in CI. The Bundle, launch, and install gates last ran at `2c3407d`. Expect this step to
be needed.

1. Pull the failing log with
   `gh run view <run-id> --repo bobs-org/bob-mac-capture --log-failed`, then grep it for
   `error: -\[`, `' failed (`, `: error: `, and `::error::`.
2. Decide whether the test or the app is wrong, using `docs/`, the code history
   (`git log -L` / `git blame` on the asserted behavior), and the `decisions` memory
   web.
3. Fix the root cause in this same change, then repeat steps 3–4 until
   `Scripts/macos-ci-check` exits 0.

A Test step that dies at the 15-minute watchdog is a hang. The sampled xctest stack in
the log, or the `test-hang-samples` artifact, names the stuck test, and async tests must
never await an untimed continuation.

If a failure needs a product decision you cannot defend, do not weaken or delete the
test. Leave it failing, finish the rest, and report the diagnosis.

### 6. Land

Finish through the normal SASE final flow. The opened repository is a commit obligation.
Use a commit message along the lines of
`fix(tests): restore BobMacCaptureTests compilation and fix stale one-off badge assertion`
and list in the body any extra fixes made in step 5.

## Acceptance criteria

- `Scripts/typecheck-tests.sh` on the `mac` reports `PASS` for all four test targets, or
  the report states that the `mac` was offline.
- `just portable-check` passes on athena.
- `Scripts/macos-ci-check` exits 0 for the final working tree.
- The diff touches only `CaptureWikilinkTaskPanelTests.swift`,
  `CapturePickerDesignTests.swift`, and `CaptureRelationPanelTests.swift`, plus any step
  5 fixes, each explained in the final report.
