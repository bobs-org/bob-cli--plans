---
tier: tale
title: Fix the Bob Mac Capture build broken by the parked-caption start-card hunk
goal:
  Bob Mac Capture compiles and installs again. The start card's invalid parked-caption
  branch from commit 1056569 is removed, and macOS CI is green on bob-mac-capture
  master.
size: small
proposed_by: bbugyi200.apollo.3u
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.3u](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.3u.md)
- **COMMITS:**
  - [f8d530d](https://github.com/bobs-org/bob-mac-capture/commit/f8d530d47c2b5f6263702574599fa6c76dccf048)
    — fix(capture): drop the parked caption branch from the start card

# Fix the Bob Mac Capture build broken by the parked-caption start-card hunk

## Problem

`just install` in bob-mac-capture fails on Bryan's MacBook. The Swift compiler stops
`BobMacCapture` with two errors; every other diagnostic in the output is a pre-existing
warning:

```
Sources/BobMacCapture/CapturePanelView.swift:2286:40: error: value of type
'CapturePomodoroStartPresentation.TaskRow' has no member 'outcome'
Sources/BobMacCapture/CapturePanelView.swift:2286:52: error: cannot infer contextual
base in reference to member 'parked'
```

## Root cause (already diagnosed)

bob-mac-capture commit `1056569` ("feat(capture): decode and present parked pomodoro
links", plan `plan:202610/park_worked_pomodoro_links.md`) added a parked-row caption
accent to `PreviewPane` in `Sources/BobMacCapture/CapturePanelView.swift`. The commit
applied the same replacement to two caption blocks that had identical text before the
change:

1. **Close card** (around line 1947, inside the `close.visibleTaskRows` loop). `row` is
   `CapturePomodoroClosePresentation.TaskRow`, which has `outcome: TaskOutcome?`, and
   `TaskOutcome` now includes `.parked`. This hunk is correct and must stay.
2. **Start card** (around line 2285, inside the
   `ForEach(Array(start.visibleTaskRows.enumerated()), id: \.offset)` loop). Here `row`
   is `CapturePomodoroStartPresentation.TaskRow`, which has no `outcome` property, and
   starts have no parked concept. This hunk does not compile.

Parking is a close-only outcome. The parking plan says "Do not add star syntax to
session starts", and the start row's `caption` is documented as the dropped-row
`"stays <status>"` / nested-line text only. The fix is to restore the start card's
original caption rendering, **not** to add `outcome` to the start presentation.

Why it reached master: the commit was authored on a Linux host. Linux cannot compile the
AppKit/SwiftUI `BobMacCapture` target (`Package.swift` only declares it under
`#if os(macOS)`), and the change was pushed without anyone looking at macOS CI. CI run
`36820459667` on `1056569` passed lint but failed at **Build** with exactly these
errors, so **Test, Bundle, launch smoke, and install steps were skipped**. The new
parked tests in `Tests/CaptureCoreTests/CapturePomodoroClosePresentationTests.swift`
(`testParkedCloseSummaryGroupsAndRows`, `testStarOnlyCloseDefersUnlistedLikeOrdinary`,
`testParseParkSpecDecodesListsAndSpans`, and the updated teaching-hint assertion) have
**never run**. A static check of their fixtures against the presentation code looks
consistent, but only CI can confirm them. The same pattern (a feature commit red on
macOS CI, fixed by a later commit) also happened at `fe5d1d5` and `1c85058`. The
previous commit, `0ff0de9`, is green.

The bob-cli side of parking (`3dd833f`, `park` in the capture JSON) is already on
bob-cli master. No bob-cli code change is needed.

## Implementation

### 1. Open the bob-mac-capture checkout

Use the `sase_repo` skill. On hosts without the configured linked primary checkout,
`sase repo open bob-mac-capture` fails with "Primary workspace directory does not
exist". In that case open it as `gh:bobs-org/bob-mac-capture` with an audit reason, and
use only the printed path. Run `git pull --ff-only` there. Confirm that `HEAD` includes
`1056569` and that no later commit already fixed `CapturePanelView.swift` (if one did,
skip to step 3 to verify CI on the current head). Read the repo's `AGENTS.md` if the
open command names one.

### 2. Remove the invalid start-card branch

In `Sources/BobMacCapture/CapturePanelView.swift`, inside the start card's task-row
`VStack`, restore the pre-`1056569` caption rendering exactly:

```swift
if let caption = row.caption {
    Text(caption)
        .font(.caption)
        .foregroundStyle(.secondary)
        .textSelection(.enabled)
}
```

This replaces the
`if row.outcome == .parked { HStack { Image("pause.circle") … } } else { … }` block that
follows the start row's `if let warning = row.warning { … }`.

Constraints:

- Leave the close-card parked caption branch, `closeOutcomeColor`'s `.parked` case, and
  every other part of `1056569` untouched.
- Do not add `outcome`, `.parked`, or any parking state to
  `CapturePomodoroStartPresentation`, and do not change any other file for this fix.
- Do not "fix" the pre-existing Swift 6 `Sendable`, `#selector`, trailing-closure, or
  `no 'async' operations` warnings. They are warnings in Swift 5 language mode, did not
  block the install, and are out of scope.
- Confirm with `git diff` that the only change is this one hunk. As a static cross-check
  (no Swift toolchain is needed), verify that every remaining `row.outcome` reference in
  `CapturePanelView.swift` is inside a function or loop whose `row` is
  `CapturePomodoroClosePresentation.TaskRow`.

### 3. Commit, push, and verify on macOS CI

Swift cannot build the app target on Linux. If the host has no Swift toolchain, report
Linux inspection as inspection only, never as passing validation. macOS CI
(`.github/workflows/ci.yml`: lint, build, test, bundle, signature, launch smoke,
install/reinstall) is the authoritative gate. CI runs only on pushes to `master`.

Bryan's approval of this plan **explicitly authorizes** committing the bob-mac-capture
fix mid-turn with the `/sase_git_commit` skill, so that CI can run. Run it from the
bob-mac-capture checkout, use a conventional message such as
`fix(capture): drop the parked caption branch from the start card`, and follow that
skill's instructions, including verifying that the branch is clean and not ahead of
`origin/master`.

Then:

1. Record the pushed SHA. Poll
   `gh run list -R bobs-org/bob-mac-capture --commit <sha> --json databaseId,status`
   inline with short sleeps until the CI run appears.
2. Use the `/sase_monitor` skill to wait on
   `gh run watch <run-id> -R bobs-org/bob-mac-capture --exit-status` (a timeout of about
   30m is enough; a green run takes about 4–5 minutes). Do not wait on it inline. The
   `--next` instructions must tell the follow-up to:
   - If CI is green: finish with the report described in step 4.
   - If CI failed: inspect
     `gh run view <run-id> -R bobs-org/bob-mac-capture --log-failed`. Fix the failures
     in the bob-mac-capture checkout, commit again with `/sase_git_commit`, and monitor
     the new run the same way. Repeat until CI is green. Failures in the never-run
     parked tests, or in later steps (bundle, launch smoke, install) introduced by
     `1056569`, are in scope. Fix them in line with
     `plan:202610/park_worked_pomodoro_links.md` (parking contract) rather than by
     weakening assertions. Do not change bob-cli Rust output to satisfy Swift tests
     unless the failure proves the two sides disagree with that plan.

### 4. Report and follow-up

When CI is green on the fixed head, report to Bryan:

- The root cause above, the fixing commit(s), and the green CI run URL.
- The steps to apply it on the MacBook, in Bryan's existing checkout: `git pull` then
  `just install` (the install output path stays `~/Applications`). Note that typing the
  new `=x*<N>` parking syntax also needs a `bob` binary built from bob-cli at or after
  `3dd833f`, per the README's CLI-first rollout. The app itself installs and runs
  without it.
- That this agent did not run anything on the MacBook. The local `just install` there is
  Bryan's final confirmation.

Then use `/sase_new_task` to check for an existing task covering the process gap. If
none exists, file one bead: Linux-hosted agents land bob-mac-capture commits whose
`BobMacCapture` target was never compiled, and nobody observes macOS CI (red master at
`fe5d1d5`, `1c85058`, `1056569`). Record the evidence and let that bead decide the
remedy. Do not implement a process or CI change in this tale.

Also tell Bryan about a SASE problem hit while this plan was being proposed. Do not fix
it here. The first `sase plan propose` crashed in `publish_plan_artifact_link_inlet`
with
`RuntimeError: artifact-link event store is invalid: validation: operation_id de29d2e25c1cfb4381f223c44d576f8c was reused for different artifact link events`.
The plan was re-proposed without its `links:` frontmatter inlet, so the typed `related`
link to `plan:202610/park_worked_pomodoro_links.md` was not recorded. The SASE
artifact-link event store needs separate investigation in the sase project.

## Out of scope

- Any change to bob-cli source, docs, or the parking contract.
- Swift 6 concurrency or deprecation warning cleanup.
- Running commands on, deploying to, or installing on Bryan's MacBook.
- Changing SASE linked-repo configuration. The MacBook's checkout lives under
  `~/projects/github/bbugyi200/bob-mac-capture`, while bob-cli's `sase/sase.yml` points
  at `~/projects/github/bobs-org/bob-mac-capture`. This is unrelated to the build
  failure; mention it only if relevant.

## Completion criteria

- `CapturePanelView.swift` compiles: the start card renders `row.caption` as plain text
  again, and the close card keeps its parked accent.
- macOS CI is green on bob-mac-capture `master` at the fixing commit (or a later fix in
  this tale), with its run URL reported. Lint, build, test, bundle, signature, launch
  smoke, and install/reinstall must all pass.
- No bob-cli changes. Any bob-mac-capture changes beyond the one hunk are fixes for CI
  failures caused by `1056569`, each explained in the report.
