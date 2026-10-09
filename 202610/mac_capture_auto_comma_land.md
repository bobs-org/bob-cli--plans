---
tier: epic
title: Land the Bob Mac Capture close-task auto-comma on master
goal: 'Bob Mac Capture''s master carries a compiling, CI-green close-task auto-comma
  (typing `=x12` in a sub-10-link Pomodoro shows `=x1,2`), salvaged from the failed
  PR #4, and PR #4 is closed as superseded.

  '
phases:
- id: land-assist
  title: Salvage PR
  size: medium
  depends_on: []
  description: 'land-assist: cherry-pick PR #4 (3842ee9) without committing onto fresh
    master, rename the colliding CapturePomodoroEntry model, trigger the assist parse
    from the digit key event, prefetch the count in CapturePanelController.show(),
    guard refresh races, extend fake-bob and tests, and land it through the /sase_final
    commit decision rather than a branch or PR.'
- id: ci-green
  title: Drive the macOS 26 SwiftPM CI run green for the landed assist and close PR
  size: small
  depends_on:
  - land-assist
  description: 'ci-green: watch the CI run for the landed commit via /sase_monitor,
    fix feature-caused failures until green, close PR #4 with --delete-branch, and
    give Bryan the bob and app reinstall steps plus the manual check.'
proposed_by: bbugyi200.apollo.61.w1.f0
create_time: 2026-10-09 14:19:58
status: wip
bead_id: bob-cli-60
---

- **PROMPT:** [prompts/202610/mac_capture_auto_comma_land.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/mac_capture_auto_comma_land.md)
- **BEAD:** [bob-cli-60](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-60/README.md)

# Land the Bob Mac Capture close-task auto-comma on master

## Why it did not work

The approved plan `202610/mac_capture_close_list_auto_comma.md` had two parts.

- **Part 1 (bob-cli) landed.** Commit `5601235` makes
  `bob capture-pomodoros --format json` report `task_link_count`: an integer on the
  `is_current` entry and `null` on every other entry. Nothing in bob-cli needs to
  change.
- **Part 2 (Bob Mac Capture) never reached `master`.** The coder did not let the SASE
  finalizer commit the opened repo. It hand-made the branch `close-list-auto-comma` (one
  commit, `3842ee9`) and opened PR #4, "feat(close): auto-insert commas between close
  task numbers". That PR's `macOS 26 SwiftPM` CI run (`37967023700`) **fails to
  compile**: the new minimal model `CapturePomodoroEntry` reuses the name of the
  existing `capture-pomodoro-name` entry model in
  `Sources/CaptureCore/CaptureModels.swift`. The errors are "invalid redeclaration of
  'CapturePomodoroEntry'" and a cascade of "ambiguous for type lookup" / "does not
  conform to Decodable" errors. Bryan's app never received the feature, and its final
  report claimed success anyway.
- The linked repo name does not open today: `sase repo open bob-mac-capture` fails
  because its primary checkout directory is missing. Use
  `sase repo open gh:bobs-org/bob-mac-capture`. It works, and it is how every other Mac
  Capture change has landed.

The fix is to salvage PR #4's code onto current `master`, fix the compile error, close
three design gaps the review found, and then drive CI green. The behavior contract is
unchanged from the approved plan: read it with
`sase artifact read plan:202610/mac_capture_close_list_auto_comma.md "<why>"`. In short:

- Typing `1`–`9` right after a task number inside a close-selection list inserts `,`
  first: `=x1` + `2` gives `=x1,2`, and `=x2!14*3` typed gives `=x2!1,4*3`. The lists
  are the `=x` keep list and the `*`/`!`/`~` groups, including the `=*`/`=!` aliases and
  link or new-task closes.
- It fires only when the running Pomodoro has fewer than 10 numbered Task Links (bob's
  `task_link_count`).
- It never fires on the first number of a group, on `0`, on Work Log numbers, on
  start-drop lists, or outside close lists.
- The app stays a thin client (decision `mac-capture-is-a-thin-client`). It only filters
  bob's `capture-parse` close-list spans (`pomodoro_close_in_progress`,
  `pomodoro_close_park`, `pomodoro_close_complete`, `pomodoro_close_drop`) and never
  recognizes close syntax itself.

## Landing rules (both phases)

- Open the repo only with `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` and
  use the printed path. The repo's `AGENTS.md` (if named on stderr) applies.
- Work directly on `master` in that checkout. **Do not create branches, do not
  `git commit` or `git push` by hand, and do not open pull requests.** Leave the changes
  uncommitted. In `/sase_final`, give the opened `gh:bobs-org/bob-mac-capture`
  repository a `commit` decision. The host's stitch finalizer commits and pushes it to
  `master`, exactly as it did for the earlier Mac Capture tales. Before finalizing, run
  `git status` in the checkout and confirm the expected files are dirty, so the
  obligation is real.
- The `==` Pomodoro override epic (`bob-cli-5z`, phases `.5` and `.6`) edits Bob Mac
  Capture concurrently, mostly `CapturePanelModel.swift`, `CaptureModels.swift`,
  `Tests/Fixtures/fake-bob`, and `README.md`. Start from a freshly pulled
  `origin/master`. Right before `/sase_final`, fetch again. If `master` moved, run
  `git stash && git pull --ff-only && git stash pop` and resolve conflicts by keeping
  both sides.
- Linux agent hosts have no Swift toolchain, so GitHub Actions is the compiler. Read
  every Swift edit against the surrounding code. The package uses tools 6.0 with
  `.swiftLanguageMode(.v5)`, `CapturePanelModel` is `@MainActor`, and `swift-format`
  line width applies.
- The final report must name the landed (or to-be-landed) commit, and must not claim the
  feature works on the Mac before CI is green.

## Phase 1: Salvage PR #4 onto master and fix it

Starting point:

```bash
git fetch origin master close-list-auto-comma
git checkout master && git pull --ff-only
git cherry-pick --no-commit origin/close-list-auto-comma   # 3842ee9, 15 files
```

Resolve any conflicts, then make these changes on top.

### 1a. Fix the compile error

In `Sources/CaptureCore/CaptureModels.swift`, rename the PR's new minimal model
`CapturePomodoroEntry` to `CapturePomodorosListEntry`. Leave the existing
`CapturePomodoroEntry` (around line 3406, used by `CapturePomodoroNameSuccess`)
untouched. Make `CapturePomodorosResponse` and `CapturePomodorosListEntry`
`Decodable, Equatable` only, since nothing encodes them. Keep their tolerant
`decodeIfPresent` decoding, so a bob without `task_link_count` still decodes with a
`nil` count. Update every reference in Sources and Tests. Then grep both Sources and
Tests for any other new top-level name that collides with an existing one. Also check
that `CaptureSpan` gaining `Sendable` (the PR adds it) does not conflict with anything.

### 1b. Trigger the immediate assist parse from the key event, not the SwiftUI caret

The PR starts the un-debounced `close-task-comma` parse in `editorTextDidChange` only
when `isCloseTaskCommaAssistTrigger(in:cursorUTF8Offset:)` sees a `1`–`9` byte before
the caret. That caret comes from `collapsedSelectionUTF8Offset()`, and the model's own
comment on `commitRouteCompletionOnPlus` warns that SwiftUI "can report one edit
behind". A fast `=x` + `1` + `2` can then miss its snapshot. Replace the trigger with a
key-driven request:

- Add `private var closeListAssistParsePending = false` and
  `func requestCloseListAssistParse()`, which sets it, to `CapturePanelModel`.
- `CapturePanelController.insertCloseTaskNumberInEditableTextView` calls
  `model.requestCloseListAssistParse()` first, before its guard. Every armed `1`–`9` key
  in the main editor then requests a parse of the draft it produces, whether the helper
  inserts `,<digit>` or declines and AppKit types the digit.
- In `editorTextDidChange`, at the PR's hook point just before `scheduleAnalysis`: if
  `closeListAssistParsePending` is set, clear it, and if `closeTaskCommaArmed` is true,
  call `startCloseListAssistParse(draft:)`. Delete `isCloseTaskCommaAssistTrigger`.
- Clear the pending flag where the PR clears `closeListParseSnapshot`
  (`resetAnalysisState`, `setPlainDraft`, and `setProcessClient(nil)`).
- Keep the PR's other snapshot rules: `applyParse` stores a snapshot only when
  `plainDraft == draft`, and the assist parse stores one only if the draft is still
  current.

### 1c. Prefetch the count on every panel show

The PR refreshes the count only in `AppDelegate.showCapturePanel()`, which is the hotkey
path. The status-item and notification-click paths reach `CapturePanelController.show()`
through `BobPanelCoordinator` and skip it. Move the call into
`CapturePanelController.show()`, right after `model.prepareForPresentation()`, and
remove it from `showCapturePanel()`. No existing test calls `show()`. Keep the PR's
vault-watcher refresh, which runs only while `panelController?.isVisible == true`.

### 1d. Make overlapping count refreshes race-free

Two refreshes share the `pomodoros` lane, so the newer one terminates the older process.
In the PR, the older task's `catch` can then set the count back to `nil` after the newer
one stored it. Add a `pomodoroCountGeneration` counter. Each refresh bumps it, and only
the latest generation writes, on both success and failure. Follow the existing `Task`
and `[weak self]` patterns in `CapturePanelModel` (for example, the analysis task that
calls `applyParse`), not a new style.

### 1e. Tests

Keep the PR's tests (`Tests/CaptureCoreTests/CapturePomodorosTests.swift`,
`CaptureCloseTaskCommaAssistTests.swift`,
`Tests/BobMacCaptureTests/CaptureCloseTaskCommaTests.swift`, and the four
`Tests/Fixtures/pomodoros-*.json` fixtures, which are real `bob` output). Update them
for the rename, then add:

- `Tests/Fixtures/fake-bob`: a `capture-pomodoros)` case that prints
  `${FAKE_BOB_POMODOROS_FIXTURE:-pomodoros-current-3.json}` from the fixtures directory
  and exits non-zero when `FAKE_BOB_POMODOROS_FAIL=1`. Match the style of the
  neighbouring cases.
- Model tests built on fake-bob. Use the existing `fakeBobPath()` and record helpers in
  `CapturePanelModelTests.swift`, and poll as the existing async tests do.
  - `refreshCurrentPomodoroTaskLinkCount()` with the default fixture arms the assist
    (`closeTaskCommaArmed == true`).
  - `pomodoros-current-12.json`, `pomodoros-legacy-no-count.json`,
    `pomodoros-none-running.json`, and `FAKE_BOB_POMODOROS_FAIL=1` each leave it
    disarmed.
  - A failing refresh after a successful one clears the count.
- A model test for 1b. Build the model with a very long `debounceNanoseconds` so only
  the assist parse can run. Arm the count, call `requestCloseListAssistParse()`, set the
  draft `=x1` through the normal edit path, and wait until
  `closeTaskCommaEdit(typed: "2", text: "=x1", selectedRange: NSRange(location: 3, length: 0))`
  returns `,2`. Serve the parse from an existing fake-bob `=x1` close-parse fixture, or
  add one copied from real `bob capture-parse --format json -- '=x1'` output:
  `pomodoro_close` [0,2) and `pomodoro_close_in_progress` [2,3). Also test the
  negatives: without the request, or while disarmed, no snapshot appears before the
  debounce.
- A controller test showing that `insertCloseTaskNumberInEditableTextView` sets the
  pending request on both the accept and decline paths. Use a test-visible accessor in
  the style of the PR's `...ForTests` setters.

### 1f. Docs

Keep the PR's README text (the keyboard table `1–9` row and the close-section
paragraph), and update it where 1c changed behavior: the count is fetched whenever the
panel is shown, by any entry point. If the README has a runtime command list that names
the bob subcommands the app runs (near line 45), add `capture-pomodoros` to it.

### Phase 1 verification

- On Linux: `git diff --stat` shows the salvaged files plus the fixes, `git status` is
  dirty in the opened checkout, and `grep -rn "struct CapturePomodoroEntry" Sources`
  finds exactly one declaration.
- Reread every changed Swift file in full around each edit for Swift 5-mode, actor
  isolation, and `swift-format` issues. If `swift-format` happens to be installed, run
  `swift-format lint --strict` on the changed files (the repo's CI uses the same
  linter).
- Finalize with a `commit` decision for `gh:bobs-org/bob-mac-capture`, using a
  Conventional Commit subject such as
  `feat(close): auto-insert commas between close task numbers`. Say plainly in the final
  report that CI has not run yet and that phase `ci-green` will drive it.

## Phase 2: Drive macOS CI green and retire PR #4

1. Open the repo as in Landing rules, then `git pull --ff-only`. Find phase 1's commit
   on `origin/master` (subject `feat(close): auto-insert commas…`). If it is absent,
   stop and report that phase 1 did not land; do not redo phase 1's work here.
2. Find that commit's `CI` workflow run (`macOS 26 SwiftPM` job) with
   `gh run list --repo bobs-org/bob-mac-capture --workflow CI --commit <sha>`. Wait for
   it through `/sase_monitor` (for example `gh run watch <id> --exit-status`), never by
   ending the turn. A later push from the concurrent `bob-cli-5z` phases can supersede
   the run. Judge the newest run that contains the commit, and attribute each failure to
   its source before fixing it.
3. On failure, read `gh run view <id> --log-failed`, fix every error caused by this
   feature (compile, `swift-format` lint, tests, bundle, or launch smoke), and finalize
   with a `commit` decision (`fix(close): …`). If more CI rounds are needed in the same
   turn, use `/sase_monitor` again. Do not touch failures that belong to other work;
   report them instead.
4. Once a run containing the feature is green, close PR #4 as superseded:
   `gh pr close 4 --repo bobs-org/bob-mac-capture --delete-branch --comment "Superseded by <sha> on master (salvaged and fixed by epic phase land-assist)."`
5. The final report tells Bryan how to try it on the Mac:
   - update `bob` from bob-cli `master` (`just install` in bob-cli), because
     `task_link_count` ships in commit `5601235`; an older bob silently disables the
     assist;
   - pull and reinstall the app (`just install` in bob-mac-capture);
   - run the manual check: with a running Pomodoro of 3 links, typing `=x12!3` shows
     `=x1,2!3`, `=x2!14*3` shows `=x2!1,4*3`, and `=x 12 fixed` stays as typed; with 10
     or more links, `=x12` stays `=x12`.

## Out of scope

Everything the approved plan listed as out of scope stays out: start-drop lists, Work
Log numbers, the `0` key, paste, grammar changes, and `capture-rewrite`/`capture-parse`
changes. There are no bob-cli code changes and no SASE memory or decision-record
changes.
