---
tier: tale
title: Green the Mac Pomodoro start preview CI and close bob-cli-2c
goal: The bob-mac-capture macOS 26 SwiftPM job is green for the whole-item Pomodoro
  start preview, and epic bob-cli-2c is closed with its plan marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-2c.land
bead: bob-cli-2c
status: done
---

- **PARENT:**
  [202609/pomodoro_start_next_operator.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/pomodoro_start_next_operator.md)
- **BEAD:**
  [bob-cli-2c](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2c/README.md)

# Green the Mac start preview and close bob-cli-2c

Epic bob-cli-2c is otherwise complete. This tale fixes the three failing Mac tests from
CI run 36461295564 on commit `147159e`, waits until the new macOS run is green, then
closes the epic. Do not change bob-cli capture behavior. Do not fix the pre-existing
`|| true` clippy deny or the `BOB_DAY_FILE` test lock; both were triaged during landing
and are recorded on the epic.

## Where to edit

Open the Mac repo with `/sase_repo`. The linked name fails on this machine because
`~/projects/github/bobs-org/bob-mac-capture` does not exist. Open the GitHub checkout
instead:

```sh
sase repo open gh:bobs-org/bob-mac-capture -r "Fix Pomodoro start live-preview status and the queued-task JSON test"
```

Use only the path that command prints. At the time of planning, that path was
`sase/repos/external/gh/bobs-org/bob-mac-capture` under this workspace, on `master` at
`147159e`. Read `AGENTS.md` there if the open command names one.

## Fix 1: live preview never publishes start status or the failure error

`Tests/BobMacCaptureTests/CapturePanelModelTests.swift` failed in CI:

- `testStartLivePreviewAndSubmitUseStartArgvFooterAndStatus` line 782: `statusText`
  stayed `"Ready"` instead of `"Would start CAPTURE 0945-1010 (25m) at line 4"`. The
  title and `Start` footer assertions above it passed, so the presentation exists.
- `testRunningStartDryRunSurfacesStillRunningError` lines 845–846: `statusText` stayed
  `"Ready"` instead of `"Preview failed"`, and `errorMessage` did not contain
  `"still running"`. fake-bob exits 1 for draft `=3` with
  `Tests/Fixtures/pomodoro-start-running.json`, whose `error` includes `still running`.

Both tests drive `editorTextDidChange`, which calls `startLivePreview` in
`Sources/BobMacCapture/CapturePanelModel.swift` (the success branch around the
`soleTogglePresentation` / `soleLinkPresentation` / `soleClosePresentation` chain, and
the `.failure` branch under it). `completePreview` already sets session-start status
and, on failure, `errorMessage` plus `statusText = "Preview failed"`. `startLivePreview`
does not.

In the success branch of `startLivePreview`, clear `errorMessage` and, after the close
arm, set status from `soleSessionStartPresentation` the same way `completePreview` does.
In the `.failure` branch, set `errorMessage = failure.error` and
`statusText = "Preview failed"`, matching `completePreview`. Leave the transport-error
`catch` alone. Do not change wording in `CapturePomodoroStartPresentation`: its dry-run
status text is already `"Would start CAPTURE 0945-1010 (25m) at line 4"` for the `=`
fixture, and the committed fixture (`dry_run` rewritten to false) yields
`"Started CAPTURE 0945-1010 (25m) at line 4"`.

## Fix 2: queued-task test builds invalid JSON

`Tests/CaptureCoreTests/CapturePomodoroStartPresentationTests.swift`
`testQueuedTaskRowsMapStatusGlyphsAndLocators` calls
`sessionStartJSON(..., tasks: queuedTasksJSON)`. `queuedTasksJSON` is a comma-separated
list of objects, not an array. The decoder fails at the second `{` (CI: unexpected `{`
around line 28). The singular-task test already wraps its payload in `[...]`.

Pass `"[\(queuedTasksJSON)]"` instead. The glyph expectations
(`[.inProgress, .ready, .other, .unresolved]` for `/`, space, `?`, and an unresolved
row) already match `taskRow(for:)`. Do not change production glyph mapping unless this
test, once given valid JSON, fails for a real mapping bug.

## Commit, then wait for macOS CI

This plan explicitly instructs you to commit the bob-mac-capture fix with
`/sase_git_commit` before waiting on CI. macOS CI runs only on the pushed commit, and
the epic must not close on a red run. A turn-ending finalizer commit is too late to
observe that run.

From the opened bob-mac-capture checkout, commit with message subject
`fix(capture): publish Pomodoro start live-preview status`. Body: one short paragraph
naming the `startLivePreview` status/error gap and the `queuedTasksJSON` array. If
`$SASE_BEAD_ID` is set, pass `-B keep`. Do not close bob-cli-2c from the commit flag.
Confirm `git status` is clean and the commit is on `origin/master`.

Then wait with `/sase_monitor` (not an inline sleep, and not a built-in background
tool). Watch the `CI` workflow run for that exact SHA, for example `gh run watch` on the
run id from `gh run list --commit <sha> --limit 1` in the Mac checkout. Timeout at least
20 minutes. The follow-up prompt must say: if the run succeeded, perform the closeout
below; if it failed, read `gh run view <id> --log-failed`, fix forward in
bob-mac-capture, commit again with `/sase_git_commit`, and watch the new run. Do not
close the epic while the latest macOS run for the fix is red, cancelled, or still
queued.

Previous successful Mac runs on this repo finished in about 3–4 minutes. Run 36461295564
is the red baseline; do not treat it as the new result.

## Closeout (only after that run is green)

Do this in the follow-up that sees the green run. There is no parent bead
(`sase bead read bob-cli-2c -r "Need the parent link"` shows `parent_id` null). Finish
normally after the epic close.

1. `sase bead epic-symbols bob-cli-2c`. At planning time this printed no entries. If it
   is still empty, continue. If any `--epic-symbol` line is present, resolve it (wire it
   up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy) or, only when a still-open later bead still needs the
   exemption, re-key that Justfile line to that open bead. `sase bead close` refuses
   while any entry remains. Do not use `--force` to get past that refusal.

2. Close the epic:

```sh
sase bead close bob-cli-2c --note "<the note below, with the green SHA and run id filled in>"
```

Note text:

```text
Verified phases bob-cli-2c.1/.2/.3 in commits 47a4b59, 193e9f9, and d223926: shared session_equals_token lexer, CaptureKind::PomodoroStart, next_future_pomodoro shared by the unnamed link start, the close next-up hint, and the whole-item start planner, POMODORO_CLOSE_INCOMPLETE_ERROR removed, pomodoro_start.tasks lineup, editor mode/spans/diagnostics, and zsh-quoted docs. Phase bob-cli-2c.4 is bob-mac-capture 147159e. The only bob-cli commit after the epic started, 55fdb18 (bob-cli-2d.1 gkeep), publishes format_task_line and does not parse or duplicate whole-item starts; no other mac commit landed during the epic. Mac CI 36461295564 failed because startLivePreview omitted session-start status and the failure error, and testQueuedTaskRowsMapStatusGlyphsAndLocators interpolated queuedTasksJSON without array brackets. Fixed both; macOS 26 SwiftPM run <RUN_ID> for <SHA> passed. Follow-ups: the tests/cli.rs:31821 || true clippy deny stays on bob-cli-28 (no new task); the unlocked BOB_DAY_FILE mutation is bug bob-cli-2e, not epic work. No --epic-symbol entries. No parent bead. just symvision is not a recipe in this justfile.
```

The bug id is `bob-cli-2e`. Never pass `--force` merely to make close succeed, and never
use `--force` to advance a successful nested landing. If close is rejected for
unfinished phases, finish or reopen them, or record a deliberate non-done resolution; do
not force a successful close.

3. `just symvision` is not in this repo's justfile (`fmt`, `lint`, `test`,
   `check-scripts` are). Run `just symvision` only if `just --summary` lists it;
   otherwise record the absence in the close note, which the text above already does. Do
   not run `just check-full`. Do not run `just lint`: it fails on the pre-existing
   `|| true` deny owned by bob-cli-28.

4. Set `status: done` in the frontmatter of
   `sase/repos/plans/202609/pomodoro_start_next_operator.md` (the PLAN path from
   `sase bead read bob-cli-2c`). That file lives in the plans sidecar. Leave every other
   frontmatter field unchanged.
