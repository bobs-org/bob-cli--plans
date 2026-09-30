---
tier: tale
size: medium
title:
  Land epic bob-cli-2v - green the Task Link Picker's macOS CI, close the Mac gaps,
  close the epic
goal:
  Bob Mac Capture's Task Link Picker passes a fully green macOS 26 SwiftPM CI run. The
  mac_panel gaps against plan:202609/task_link_picker.md are fixed and covered by tests.
  Two small bob-cli leftovers are cleaned up. Epic bob-cli-2v is closed and its plan
  file is marked done.
proposed_by: bbugyi200.apollo.bob-cli-2v.land
bead: bob-cli-2v
status: done
---

- **PARENT:**
  [202609/task_link_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/task_link_picker.md)
- **BEAD:**
  [bob-cli-2v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2v/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2v.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2v.land.md)
- **COMMITS:**
  - [6191dec](https://github.com/bobs-org/bob-mac-capture/commit/6191dec5428e2e037492175b930415bdaa0dd0ac)
    — fix(capture): land task-link picker CI fix, gaps, tests, and docs

# Land epic bob-cli-2v: green the Task Link Picker's macOS CI and close the epic

## Context (verified by the bob-cli-2v land agent)

- Epic `bob-cli-2v` (plan `plan:202609/task_link_picker.md`) shipped the `:` Task Link
  Picker. All five phase beads are closed.
  - bob-cli commits: `5ef8eb7` (grammar), `d67bbb0` (discovery), and `b672114`
    (complete).
  - bob-mac-capture commit: `e9865e6`, which carries both `mac_core` and `mac_panel`.
- **bob-cli is complete and green.**
  - `cargo fmt --check` is clean.
  - `cargo test` passes: 1334 lib and 650 cli tests.
  - `cargo clippy --all-targets --all-features` has one error:
    `tests/cli/capture/pomodoro_name.rs:808` (`|| true`). It predates this epic, and
    active epic `bob-cli-28` owns it. **Do not fix it here.**
  - The remaining clippy warnings also predate this epic (`bob-cli-v`).
- **bob-mac-capture CI is red.** Run `36758703183` (on `e9865e6`, job
  `macOS 26 SwiftPM`) fails at step `Test` with a test-target compile error:
  - `Tests/CaptureCoreTests/TaskLinkPickerPresentationTests.swift:488` and `:494` call
    `presentation.pullForwardLine(for:)` on a `CapturePickerPresentation`.
  - That method only exists on `TaskLinkPickerIndex`.
  - As a result, no new Swift test has ever run.
  - The test bodies in `Tests/BobMacCaptureTests/CaptureTaskLinkPanelTests.swift` and
    `CapturePickerMarkerHighlightTests.swift` have never been type-checked.
- **GitHub Actions is the only Swift test gate.**
  - Linux has no Swift.
  - The `mac` tailnet host has only Command Line Tools. `just format-lint` and
    `just build` work there, but `just test` always fails with `no such module XCTest`,
    even on a clean tree.
- **Follow-ups are already triaged.** The outcomes are recorded in notes on
  `bob-cli-2v`:
  - the clippy `|| true` → a note on `bob-cli-28`;
  - the XCTest-less host → a `+1` on `bob-cli-1x`.

  Do not refile either.

- **No post-epic integration is needed.**
  - bob-cli `f7d9c58` (cancel-gesture docs, `bob-cli-2w.4`) does not interact: canceled
    tasks are already excluded, and a Cancel Log bullet is not a `#task`.
  - No other bob-mac-capture commit landed after the epic started.
- **Nothing extra to clear or close.** `sase bead epic-symbols bob-cli-2v` has no
  entries, and the epic has no parent bead.

## Steps

### 1. Open bob-mac-capture

1. Run `sase repo open bob-mac-capture -r "<why>"`.
2. If the host has no linked checkout (the command errors), run
   `sase repo open gh:bobs-org/bob-mac-capture -r "<why>"` instead.
3. Use only the path it prints, and read `AGENTS.md` there if one exists.
4. Make sure the checkout is at `origin/master` (`git pull --ff-only`).

### 2. Fix the CI compile error

In `Tests/CaptureCoreTests/TaskLinkPickerPresentationTests.swift`,
`testScheduledTextAndPullForward`:

- Build `let index = TaskLinkPickerIndex(candidates: try workedExampleCandidates())`.
- Call `index.pullForwardLine(for:)` for both assertions, the same way
  `testIdentifiedActionLine` calls `index.actionLine(for:)`.

Never weaken an assertion.

### 3. Close the Mac gaps against the epic plan

Paths are under `Sources/BobMacCapture/` unless noted.

1. **`CapturePanelModel.removePickerTrigger`, task-link branch.**
   - Delete the second fallback branch, commented "Empty the item when the `:` token is
     already gone". It deletes `[r.start, r.end)` without checking the byte.
   - `[r.start, r.end)` may only be deleted when the draft still matches the snapshot
     and the byte at `r.start` is `:` (58). Otherwise, call `cancelPicker()`.
2. **`cancelTaskIDPrompt(clearCompletion:)`, `.taskLink` branch.**
   - Today it always restores `returnPicker`. `editorTextDidChange` calls it with
     `clearCompletion: true` when the draft changed under the prompt, so a picker for
     the stale draft reappears and the editor is re-locked.
   - When `clearCompletion` is true: clear the prompt, dismiss the picker and
     completion, and leave the editor unlocked, matching what the `.parentTask` path
     does.
   - Only Escape (`clearCompletion == false`) restores the picker, without a refetch.
3. **`completeTaskIDAssignment`, link-mode success.** Build the link from Bob's response
   (`"@\(success.route):\(success.blockID)"`, plus `=` for `.start`), not from the
   request's `linkRoute`. The epic plan specifies this.
4. **`acceptPickerRowAndStart`.** On a stale draft (the draft no longer matches
   `picker.draftSnapshot`, or the range no longer resolves), call
   `closePickerAfterStaleDraft()`, exactly as `acceptPickerRow` does, instead of
   returning silently.
5. **`handleTaskLinkCompletion`.**
   - Bob's `task_link` replacement includes the `:`, so today `cursor != r.start` is
     always true. Every open therefore runs a second `capture-complete`, even for a bare
     `:` whose list is already complete.
   - Refetch at `r.start` only when the typed query is non-empty
     (`cursor > r.start + 1`). Keep the `snapshotIsPartial` fallback.
6. **Empty-state wording.**
   - `CapturePickerEmptyState.noMatches(query:)`, in
     `Sources/CaptureCore/CapturePickerPresentation.swift`, says "No active tasks match
     …".
   - Make the task-link index show "No open tasks match “<query>” — Esc clears the
     filter.", for example with a noun parameter that defaults to the current wording.
   - Keep the `^` picker's text byte-identical, and update any test that pins it.
7. **Colors and highlights.**
   - In `CapturePanelView.swift`, the link-mode `TaskIDPromptCard` line
     `Inserts @route:<typed>[=]` is plain secondary text. Render the route in
     `CaptureEditorPalette.color(for: .route)` and the block ID in the block-ID color
     that the picker locators already use.
   - In `CapturePickerView.swift`, the ID-less trailing locator passes `matches: []` for
     the route. Pass the row's `routeMatchRanges` so filtered ID-less rows highlight
     their route.
   - Do not change any layout constant pinned by `CapturePickerDesignTests`.
8. **`README.md`.**
   - Replace the loose task-link bullets (around the "Typing `:` as the whole capture
     item" bullet) with a real `### Task Link Picker` subsection after "Active Task
     Picker". It covers:
     - the trigger;
     - the included notes and statuses;
     - groups and order;
     - fuzzy filtering;
     - the keys;
     - the ID-less link-mode Add block ID flow;
     - bulk drafts.
   - Update both keyboard tables:
     - For the `:` picker, Shift-Return is Link & Start; it stays consumed for the other
       pickers.
     - In the link-mode Add block ID prompt, Tab and Shift-Tab cycle suggestions; they
       stay consumed in the `@route+` prompt.
   - Confirm that Requirements names the `task_link` Bob feature.

### 4. Add the missing Mac panel tests

Put the tests in `Tests/BobMacCaptureTests/CaptureTaskLinkPanelTests.swift`. Follow the
existing `^` active-task picker tests in `CapturePanelModelTests.swift` for how they
drive `Tests/Fixtures/fake-bob` and count subprocess calls.

- **Opening:**
  - typing `:` auto-opens the `taskLink` card;
  - `Buy milk\n\n:` opens it on the second item;
  - a selection-only move shows the chip rather than opening the card;
  - typing in the filter does not run a subprocess per keystroke;
  - a bare `:` does not run a second `capture-complete` (step 3.5).
- **Keys:**
  - ⌘↩ inserts `@sase:deep-fix` and submits;
  - ⌫ on an empty filter leaves an empty item;
  - the first Esc clears the filter, and the second shows the reopen chip.
- **ID-less flow:**
  - success splices `@sase:fix-flaky-gkeep`, with the `=` variant and the submit
    variant;
  - a duplicate-ID failure keeps the prompt open with Bob's error;
  - the `@route+` Add block ID flow is unchanged.
- **Status and hints:**
  - the calm status text "Pick any open task — press Tab to browse";
  - the task-link key hints are pinned in `CapturePickerDesignTests`.
- **Regressions:**
  - a non-`:` byte at `r.start` cancels and deletes nothing (step 3.1);
  - a draft change under a link-mode prompt dismisses it instead of restoring the picker
    (step 3.2).
- **fake-bob:**
  - Add cases for:
    - `capture-parse` of `Buy milk\n\n:`;
    - `capture-complete` on that batch's second item;
    - `capture-task-id` success;
    - the `capture-task-id` duplicate-ID error.
  - Make the `:dee` completion case honor `--cursor`, so a refetch at cursor 0 returns
    `task-link-complete-full.json`.
  - Either use or delete the three fixtures that nothing references:
    `task-link-capture-link.json`, `task-link-capture-start.json`, and
    `task-link-parse-link-start.json`.
- **New fixture JSON:**
  - Generate it with a real `bob`: run `cargo build` in the bob-cli checkout, then run
    it on the epic plan's worked-example vault. `task_link_fixture` in
    `src/native/capture_complete.rs` and `tests/cli/capture/complete_task_link.rs` build
    that vault.
  - Pin `BOB_NOW=2026-09-30 09:02:00` and `BOB_DAY_FILE`.
  - Never hand-write fixture JSON.

### 5. bob-cli cleanups (the bob-cli workspace checkout)

1. **`src/native/capture_language/completion.rs`.**
   - A two-paragraph doc comment starting "Identify the completable marker component at
     `cursor`" sits directly above `task_link_completion_field_at`'s own doc comment. It
     was already misplaced before this epic, and the epic made it worse.
   - Move it to directly above `pub(crate) fn completion_field_at`, which has no doc
     comment today.
2. **`src/native/capture_task_toggle.rs`.**
   - Phase discovery widened `find_single_future_scheduled_field` to `pub(crate)`.
   - Phase complete then replaced its only outside caller with
     `capture_link_tasks::scheduled_facts`.
   - Make it private again (`fn`).
3. **Checks:**
   - `cargo fmt --check` must be clean.
   - `cargo clippy --all-targets --all-features` may show only the known
     `pomodoro_name.rs:808` error and the warnings that were already there. Nothing new
     is allowed.
   - `cargo test` must pass.

### 6. Verify on macOS and iterate CI to green

1. **Pre-check on the mac host.**
   - Run `ssh -o ConnectTimeout=8 mac true`.
   - If it works, rsync the bob-mac-capture checkout (excluding `.build`) to
     `mac:/tmp/bob-mac-capture-task-link/`, then run `just format-lint build` there.
   - Do not run `just test` there as a signal; it cannot pass on that host.
2. **Commit each iteration.** This plan explicitly instructs you to use
   `/sase_git_commit` in the bob-mac-capture checkout for each CI iteration.
   - Use conventional subjects, for example `fix(capture): …` or `test(capture): …`.
   - Confirm that the commit reached `origin/master`.
3. **Find and watch the run.**
   - Find it with `gh run list -R bobs-org/bob-mac-capture -L 3`.
   - Wait in the foreground with
     `gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`, using a tool timeout
     of about 30 minutes.
   - Read failures with
     `gh run view <id> -R bobs-org/bob-mac-capture --log-failed | grep -E " error: |error: -\[|failed \("`.
4. **Definition of done.** One `macOS 26 SwiftPM` run has every step green:
   - Lint Swift formatting
   - Build
   - Test
   - Bundle
   - Verify plist and signature
   - Launch smoke test
   - Install and reinstall

   Record the run ID. Never weaken or skip an assertion to go green.

5. **Failures this epic did not cause** (a flake, or a defect in pre-existing code):
   take them through `/sase_new_task` with details that identify `bob-cli-2v`. Do not
   paper over them.

### 7. Close out epic bob-cli-2v (final step)

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-2v`. There were no entries at
   landing time.
   - If any appear, resolve each per the Symvision epic-whitelist policy: wire it up,
     privatize it, add a non-test pragma, or delete it.
   - Re-key an entry only when a still-open later bead needs it.
2. **Close the epic.** Run `sase bead close bob-cli-2v --note "<verification>"`. The
   note must cover:
   - the five phases verified, with commits `5ef8eb7`, `d67bbb0`, `b672114`, `e9865e6`,
     and this tale's bob-mac-capture commits;
   - the bob-cli fmt, test, and clippy results;
   - the green CI run ID;
   - the Mac gaps fixed in step 3 and the tests added in step 4;
   - the integration review: `f7d9c58` has no interaction, and no post-epic Mac commits
     landed;
   - the follow-up triage: the `bob-cli-28` note, the `bob-cli-1x` `+1`, and the CI
     proposal declined because it was epic work.

   Never use `--force`.

3. **Symvision.** Run `just symvision` if the recipe exists. bob-cli's justfile
   currently has none; if so, say that in your final response.
4. **Plan file.** Set `status: done`, replacing `status: wip`, in the frontmatter of the
   epic's plan file: the PLAN path shown by
   `sase bead read bob-cli-2v -r "Need the plan path"`
   (`plan:202609/task_link_picker.md`).
5. **Parent.** `bob-cli-2v` has no `parent_bead`, so nothing further is needed.
