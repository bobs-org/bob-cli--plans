---
tier: tale
title: Fix the Mac Capture freeze when typing an unconfigured Pomodoro theme
goal:
  Typing a theme name outside the configured themes vocabulary in Bob Mac Capture never
  freezes the app, and the dashed unconfigured underline still appears once the caret
  leaves the name.
size: medium
decisions:
  red_first_repro:
    ask:
      Prove the root cause on CI by running the new tests against the unfixed app before
      landing the fix?
    choices:
      run:
        One extra throwaway ci/ branch run (10-20 min) whose hang sample shows the loop
      skip:
        Land the fix with the model test and a manual check; the cause stays inferred
    default: run
    why:
      Bryan asked to confirm the root cause; a CI hang sample is the only direct stack
      proof
    answer: run
proposed_by: bbugyi200.athena.158
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-28 11:30:54
status: wip
---

# Plan: Fix the Mac Capture freeze when typing an unconfigured Pomodoro theme

## Context

With a `themes:` block in `~/.config/bob/config.yml`, typing a theme name that is not
configured (for example `=#fo`) freezes Bob Mac Capture. The menu stops updating (it
still shows the `=#` empty-query rows FIX, CHORE, PLAN, BUILD), the preview spinner
never resolves, and keystrokes stop landing.

**Verdict on the suspicion: confirmed and narrowed.** The bug comes from epic
`bob-cli-dk`, but only from phase `bob-cli-dk.4`, which is bob-mac-capture commit
`2497c41` ("render unconfigured Pomodoro themes with dashed treatment"). The bob-cli
phases (`dk.1`/`dk.2`) are correct. bob only supplies the trigger, `configured: false`
on parse spans, and it does so correctly.

### Evidence gathered during planning

- **bob is fast and correct.** `bob capture-parse`, `bob capture --dry-run`, and
  `bob capture-complete` for `=#f` and `=#fo` each return in under 0.5 s. This holds on
  the Mac with its installed bob and real vault, and on Linux with bob-cli master and a
  copy of the vault. No bob processes were left running on the Mac.
- **When the trigger arms.** From `=#f` onward, the `pomodoro_name` span's theme ref
  carries `"configured": false, "hue_source": "unconfigured"` (key `f`, then `fo`).
  `=#fix` carries `configured: true`, and with no `themes:` block the field is absent.
  So the trigger arms on the first letter of any unconfigured name, but never for
  configured names or vaults without a vocabulary.
- **The Mac unified log shows a hard main-thread hang.** The frozen instance is pid
  97537, launched 11:08:54 right after `just install`.
  - The panel opened and took `=` and `#` at about 11:09:02.
  - Keystrokes followed at 11:09:04.58, .68, and .79.
  - The app's main thread logged its last line at 11:09:04.799.
  - WindowServer kept delivering key-downs to it at 11:09:09.0 and 11:09:09.5, and the
    app logged nothing in response.
  - The process died at 11:09:48 with no Quit AppleEvent (a force quit) and no crash
    report.

  So this is a synchronous main-thread wedge. It is not slow bob output and not a crash.

### Root cause

`2497c41` added this block to
`CapturePanelModel.editorSelectionDidChange(cursorUTF8Offset:)`
(`Sources/BobMacCapture/CapturePanelModel.swift`):

```swift
if let parse = latestParse,
   parse.spans.contains(where: { span in
       (span.kind == "pomodoro_name" || span.kind == "task_toggle_pomodoro_name")
           && span.themes.contains(where: { $0.configured == false })
   })
{
    applyHighlighting(parse: parse, draft: plainDraft)
}
```

Here is how that block causes the freeze:

1. **What drives the handler.** `editorSelectionDidChange` is driven by
   `.onReceive(model.$editorSelection.dropFirst())` in `AutosizingCaptureEditor`
   (`CapturePanelView.swift`). In other words, the SwiftUI
   `TextEditor(text: $model.attributedDraft, selection: $selection)` calls it whenever
   it writes its own selection binding: on every keystroke and every caret move.
2. **What the repaint writes.** `applyHighlighting` rewrites both of that editor's
   bindings through `attributedDraft.transform(updating: &editorSelection)`.
3. **The cycle.** Once the latest parse holds an unconfigured theme, every
   editor-originated selection callback pushes new text and a new selection back into
   the same editor, from inside that editor's update. Two things keep the repaint from
   ever becoming a no-op:
   - The paint depends on the caret. The dashed underline shows only when the caret is
     outside the name (`CaptureUnconfiguredThemePresentation.caretIsInName`).
   - The parse can be one keystroke stale: right after typing, it repaints the new draft
     using the previous draft's spans.

   The editor re-applies the changed value and reports a selection again, which repaints
   again. The cycle never settles.

4. **Why the existing guard doesn't stop it.** `isApplyingProgrammaticDraft` cannot
   break the cycle: it is reset before the editor's echo arrives.
5. **Why the menu and preview stall.** Each pass also calls
   `scheduleAnalysis(trigger: .selection)`. That cancels the pending analysis and sets
   `previewState = .loading`, which explains the stuck spinner and the stale `=#` menu.

**Why this is new.** Before `2497c41`, nothing on the selection path wrote the editor
bindings. Painting happened only in `applyParse`, after a debounced analysis, outside
editor callbacks. That is why every other draft, every configured name, and every vault
without `themes:` stays stable.

**The repaint was also redundant.** A caret move already runs
`scheduleAnalysis(trigger: .selection)`. After the 50 ms debounce, that re-parses and
calls `applyParse` → `applyHighlighting`, and `paintHighlighting` reads the current
caret. So the "dashed underline once the caret leaves the name" behavior (PT6 in bob-cli
`docs/pomodoro-themes.md`) still holds without it, one debounced parse later.

**What remains unproven.** The exact SwiftUI scheduling of the echo is not visible
without a stack sample. The red-first step below captures one.

## Scope and repositories

All code changes are in the linked repo **bob-mac-capture**:

- Open it with `/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`).
- Read its `AGENTS.md` before editing.
- Follow its rules: compile every Swift change, use 4-space indentation and 100-column
  lines, and format only the lines you change.

Out of scope:

- **No changes to bob-cli, bob-plugins, the JSON protocol, or any docs.** The Mac docs
  never promised an immediate underline, and PT6's "once committed or the cursor leaves
  it" stays true.
- **No memory notes change.**
- **Not making `applyHighlighting` skip no-op writes globally.** That would touch every
  parse application and is not needed for this fix.

## Fix

1. **Delete the repaint.** In
   `CapturePanelModel.editorSelectionDidChange(cursorUTF8Offset:)`, delete the
   `latestParse` / `configured == false` / `applyHighlighting` block quoted above.
   - In its place, add one short comment stating the invariant: selection callbacks
     never write `attributedDraft` or `editorSelection`, because those writes echo back
     through the `TextEditor`; caret-dependent paint (the unconfigured underline) is
     refreshed by the selection-triggered analysis below.
   - Leave unchanged: `paintHighlighting`'s caret rule, the `cursorUTF8` parameter,
     `applyParse`, and `visibleThemeNotConfiguredWarnings` (a pure read).
2. **Check nothing else re-adds the pattern.** Grep the selection path
   (`editorSelectionDidChange` and the helpers it calls synchronously) for any other
   write to `attributedDraft` or `editorSelection` added by `2497c41`. There should be
   none. If one exists, remove it under the same invariant.

## Tests

1. **Parse fixture.** Add `Tests/Fixtures/capture-parse-unconfigured-theme-start.json`.
   - Content: real output of `bob capture-parse --format json -- '=#fo'`.
   - Generate it from bob-cli master with a temporary config holding a `themes:` block,
     for example `BOB_CONFIG_FILE=<tmp>/config.yml` containing `themes: {FIX: red}`.
   - The `fo` theme ref must carry `"configured": false` with `start: 2, end: 4`.
   - Wire it through the existing `FAKE_BOB_PARSE_FIXTURE` hook in
     `Tests/Fixtures/fake-bob`. Check how that hook resolves the file name.
2. **Model regression test:** `CapturePanelModelTests`, named something like
   `testCaretMoveWithUnconfiguredThemeDoesNotRewriteEditor`. This test must fail on
   current master.
   - **Setup:** a `CapturePanelModel` with `plainDraft = "=#fo"` and the caret at the
     end. Set `latestParse` to a parse whose `pomodoro_name` span (2..4) has a
     `configured: false` ref; reuse the shape of
     `testVisibleThemeNotConfiguredWarningsHoldBackConfiguredPrefixes`.
   - **Act:** subscribe to `model.$attributedDraft.dropFirst()` and
     `model.$editorSelection.dropFirst()`, then call
     `editorSelectionDidChange(cursorUTF8Offset: 0)` and `(cursorUTF8Offset: 4)`.
   - **Assert:** zero emissions on both publishers. Master emits once per call.
3. **Behavior-preservation test:**
   `testCaretLeavingUnconfiguredThemeUnderlinesAfterAnalysis`.
   - **Setup:** `CapturePanelModel(debounceNanoseconds: 0)` with a fake-bob
     `processClient` that sets `FAKE_BOB_PARSE_FIXTURE` to the new fixture. Draft
     `=#fo`.
   - **Caret leaves the name:** move the caret to 0 and call `editorSelectionDidChange`.
     Await (bounded `waitUntil`) the dashed orange underline on `fo` in
     `model.attributedDraft`. Use the same `NSUnderlineStyle([.single, .patternDash])`
     check as `CaptureEditorHighlightTests`.
   - **Caret returns:** move the caret back to 4 and await the underline clearing.
4. **Hosted regression test:** new file
   `Tests/BobMacCaptureTests/CaptureUnconfiguredThemeHostedTests.swift`, modeled on
   `CapturePanelSwitchHostedTests`.
   - **Setup:** a real `CapturePanelController` panel, with the model's `processClient`
     on fake-bob and the fixture.
   - **Type:** find the backing `NSTextView` and insert `=#fo` with
     `insertText(_:replacementRange:)`. Wait for a `latestParse` with
     `configured == false`.
   - **Move the caret:** `setSelectedRange` to 0 and then back to the end. Await the
     underline appearing and clearing as in test 3, and assert
     `model.plainDraft == "=#fo"`.
   - **If it regresses:** the main thread wedges, and the CI Test-step watchdog samples
     `xctest`.

## Red-first proof

> [!decision] red_first_repro = run
>
> 1. Before committing the fix, push a throwaway branch
>    `ci/mac-unconfigured-theme-freeze-repro`. Base it on bob-mac-capture master and
>    include only the new tests and fixture; CI runs on `ci/**`.
> 2. On that branch only, shorten the run:
>    - Narrow the CI Test step to the new hosted test class and the model test (pass a
>      `swift test --filter`-style argument through `Scripts/xcode-swift.sh test`).
>    - Lower the 900 s watchdog to about 180 s.
> 3. Expected results:
>    - The model test fails with one emission.
>    - The hosted test hangs. The `test-hang-samples` artifact's main-thread stack then
>      shows `editorSelectionDidChange` → `applyHighlighting` re-entering through the
>      SwiftUI `TextEditor` update.
> 4. Record the run URL and the key stack frames in the bead notes, then delete the
>    branch.
> 5. If the hosted test does not hang on CI, record that too. The red model test still
>    proves the bad write. Do not spend more than this one extra run.

> [!decision] red_first_repro = skip
>
> Push no throwaway branch. Land the fix with the tests, and rely on the model test and
> Bryan's manual check.

## Verification

- **Linux:** run `just portable-check` on a Linux host with Swift. It lints and covers
  only the portable targets.
- **Mac scratch compile (best effort), per bob-mac-capture `AGENTS.md`:**
  - Rsync a scratch copy (without `.build`/`.git`) to the Mac.
  - Run `./Scripts/bundle.sh` and `just typecheck-tests` there, then delete the copy.
  - Never use Bryan's checkout or `just install` there.
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- **CI:** land the fix as a normal bob-mac-capture commit and confirm its master CI run
  is green: Lint, Build, Test (including the three new tests), Bundle, and the install
  gates.
- **Manual acceptance for Bryan (list it in the final report).** After `just install` on
  the Mac, with the six-theme `themes:` block:
  - Type `=#fo`. The menu updates (create row with the "Not in your themes" badge), the
    preview settles with the orange "FO isn't one of your configured themes" caption,
    and the app stays responsive.
  - Press Home or move the caret to the start: a dashed orange underline appears under
    `fo`. Move back to the end: it disappears.
  - Typing `=#fix` still tints FIX red.
