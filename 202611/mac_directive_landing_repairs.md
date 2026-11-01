---
tier: tale
title: Repair Mac directive regressions and finish bob-cli-ey.3.1
goal:
  Restore capture submission and clipboard behavior, verify the remaining migration, and
  close bob-cli-ey.3.1 and parent phase bob-cli-ey.3.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ey.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ey.3.1
create_time: 2026-11-01 01:55:56
status: wip
---

- **PARENT:**
  [202610/mac_directives.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/mac_directives.md)
- **BEAD:**
  [bob-cli-ey.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.3.1.md)

# Repair Mac directive regressions and finish bob-cli-ey.3.1

Finish the remaining work in `bob-cli-ey.3.1`, then close that epic and its delegated
parent phase `bob-cli-ey.3` in this same coding turn. This is bounded work for one
agent: repair the new grammar gate's submission continuation, complete the fixture
migration, and repair clipboard lifecycle/presentation and its hosted tests. Do not
reimplement the four completed phases or add another epic.

## Scope and evidence already established

Read `bob-cli-ey.3.1` and its landing note, its four closed children, and
`plan:202610/mac_directives.md` through the audited bead/artifact commands. Also read
`bob-cli-ey.3` and `plan:202610/capture_drawers_and_directives.md` for inherited scope.
Open `bob-mac-capture` through `/sase_repo` and read its `AGENTS.md`. Source paths below
are relative to that linked repo unless explicitly qualified as bob-cli.

The land audit reviewed all child notes and commits `04904ee`, `a580c0d`, `cea18c1`, and
`a8e2ab6`, along with the actual Rust wire serializers and Swift consumers. Wire
decoding, directive rows/tint, explicit no-selection state, and retired-path deletion
are present. All four children have one completion note and no `PROPOSED FOLLOW-UP:`
entries. Their portable tests passed (1877 at the final phase), but portable checks
exclude the hosted Mac tests. The final phase's docs and scripts checks passed, and the
lander reran `just check-scripts` successfully (8 docs-script tests and 42 CI-tool
tests). Its printed mock CI failures are test data, not real CI.

Both origins were fetched during the audit. bob-cli stayed at `a1dcb64e`; Mac stayed at
`a8e2ab6`. There were no non-epic commits after the Mac pre-epic base `91a5d2b`, and no
bob-cli commits after this child epic began. Recheck drift before closeout.

Actual macOS CI evidence, available without a new commit:

- [Base run 41260240529](https://github.com/bobs-org/bob-mac-capture/actions/runs/41260240529),
  job `124711823950`, SHA `91a5d2b`: lint/build pass; 3463 tests, 19 assertion failures
  in seven already-known nodes.
- [Menu run 41280243768](https://github.com/bobs-org/bob-mac-capture/actions/runs/41280243768),
  job `124770529844`, SHA `cea18c1`: lint/build pass; 3504 tests, 61 assertion failures.
  New failures include the submission continuations and fixture cases below.
- [Final-phase run 41282066807](https://github.com/bobs-org/bob-mac-capture/actions/runs/41282066807),
  job `124775827805`, SHA `a8e2ab6`: lint/build pass; new clip tests time out and
  `testBatchItemsEachCarryTheirOwnClip` crashes with `Index out of range` after getting
  zero preview results. Later hosted tests never execute. Bundle and later gates skip.

The lander completed `/sase_new_task` triage and recorded its outcome on the epic.
Existing `bob-cli-f3` received a +1 with independent base/menu CI evidence. Its seven
baseline nodes concern line-case undo selection, two plus-toggle undo cases, the
whole-item-plus footer, task-type retype strings, typed-menu opening, and visibility. Do
not create duplicate tasks or make this repair fix unrelated baseline defects. The new
typed-hash filtering test overlaps the old menu-fixture defect; fix that overlap here
because filtering is this epic's acceptance requirement, and add a note to `bob-cli-f3`
explaining exactly which overlapping cases the repair resolved.

The inherited decisions remain final: Bob owns grammar, spans, candidates, replacement
ranges, preview content and writes. Keep schema 1 and minimum grammar 2; accept 3+. Do
not add Swift syntax parsing, compatibility aliases, drawer behavior, memory edits,
version changes, or changelog edits. Drawer implementation belongs to `bob-cli-ey.5`. Do
not install/deploy the app or touch Bryan's working checkouts or real clipboard.

## 1. Keep explicit submit gestures through the current-draft grammar gate

In `CapturePanelModel.swift`, `captureGrammarAllowsSubmit` requires the exact current
draft and client identity. Completion/picker acceptance changes the draft, schedules
analysis, then immediately calls `submit`, which returns while that parse is pending.
There is no continuation, so the user must repeat the gesture. This is a product
regression, not a test that should simply install fake compatibility evidence.

Preserve a one-shot explicit submit intent across the required analysis of the final
inserted draft. Bind it to that draft, the process client and analysis/request identity,
including the open-after flag. Execute it once only after compatible parse evidence is
accepted. Clear it on user editing, replacement client/Recheck Bob, hide/dismissal,
incompatible or failed parse, cancellation, or a newer gesture. Preserve normal
follow-up prompts and their explicit confirmation boundaries. Do not authorize from the
pre-insertion draft, a default revision, or stale parse evidence; a plain Return while
analysis is pending must not become an unrelated deferred submission.

Audit all calls that submit after programmatic insertion: generic completion,
`acceptPickerRow`, and ID-assignment continuations near
`completeBlockedTaskIDAssignment` and the relationship/task-link variants. Preserve one
aggregate `bob capture` call, open-after behavior, disabled-row consumption, and
unselected Return submitting the unchanged draft. Audit resumed submission paths too;
incompatible/missing grammar must still show exactly
`Update bob to use drawers and % directives` and keep the draft.

Cover ordinary completion, block-ID and task-link pickers, an ID-less row's Add block ID
prompt, command/control submit-and-open, and cancellation/stale/failed/incompatible
parse races with bounded hosted tests. Existing failing tests include:
`testBlockIDLinkCommandAcceptSubmits`, `testCommandAcceptInsertsAndSubmits`,
`testControlCompletionAcceptsThenCapturesAndOpensWhenNoFollowUpAppears`,
`testControlPickerAcceptCapturesAndOpensThePlan`,
`testControlPickerAcceptKeepsOpenIntentThroughAddBlockIDPrompt`,
`testTaskLinkCommandAcceptInsertsAndSubmits`, and
`testTaskLinkIdlessCommandAcceptSubmits` in `CapturePanelModelTests.swift`.

Update the three `CaptureOpenHintTests` that currently construct models without a client
or parse: distinguish pending/incompatible state from a compatible model and keep
meaningful chip/destination assertions. Do not loosen the production gate to make those
fixtures pass.

## 2. Complete the current-wire fixture and scenario migration

Use generated Bob responses for the scenarios; change generators then regenerate, rather
than hand-editing semantic JSON or offsets. Use an explicit current `BOB_BIN` and
isolated vault/config/time, as the existing generators require. Keep saved-template
identity stable for the same canonical definition, and keep selection/order assertions
consistent with the actual scenario being tested.

- `FAKE_BOB_TASK_TYPE_CHOOSER=1` currently returns `text` instead of required `body` for
  typed hashes and the generic fallback. The fallback for `jot idea #f` also has no
  spans/needs, so the new `testTypingAfterAnUnselectedMenuRanksAFreshSelection` times
  out. Serve correct parse and completion fixtures for bare-to-typed, leading, trailing
  and child markers. Do not infer syntax in Swift or use unquoted shell heredocs that
  execute backticks in Bob-authored error copy. Verify real decoding, initial selection,
  filtering, and typed-row visibility semantics.
- Regenerated shelf list/save fixtures changed definitions, ordering and IDs while
  hosted tests still assume `6cd8000f52eb` and the old first row. Repair
  `CaptureSavedTemplateShelfTests` and `CaptureTemplateShelfModelTests` with coherent
  list/save/delete/replace fixture scenarios. The latter's `threeTemplates` still has
  old cursor values 57/38/30 after shortening the drafts. Use truthful UTF-8 offsets,
  preserve rows-first insertion and one-step native undo, origin exclusion/replacement,
  delete failure restoration, and Bob's actual non-template refusal copy.
- `CapturePanelModelTests.testPlusSubBulletAndCaretIdOnlySubmitThroughDirectArgv` now
  receives `kind: task`/`new-id` instead of `sub_bullet`/`parent-id` from fake-bob.
  Restore the valid plain sub-bullet scenario removed during global-parent retirement;
  do not restore retired global-parent syntax.
- The menu CI's `testFileUnderAcceptPreservesListenDirective` received a generic route
  completion inside the URL. Phase 4 adjusted this test: recheck that adjustment
  resolves the race and still verifies preservation of `%l` using valid Bob ranges.

Retain saved-template save/reload and aggregate capture tests. Audit the remaining
menu-run failures against the baseline list above; fix epic-caused failures rather than
deleting assertions or reclassifying them as unrelated infrastructure.

## 3. Repair clipboard test execution, lifecycle, and history rendering

- `CaptureClipPreviewHostedTests.fakeBobPath()` removes only two path components before
  appending `Tests/Fixtures/fake-bob`, yielding `Tests/Tests/Fixtures/fake-bob`. Fix the
  path consistently with neighboring hosted suites and fail early if the executable is
  unavailable. Make bounded waits throw/return a checked failure so timeout does not
  continue into `cards[0]`/`cards[2]` and crash the test process. Use a parse fixture
  for the exact three-item batch draft; the current test overrides it with
  `directive-parse-batch-later-item.json`, which describes two different items. Check
  other overridden fixtures against their actual drafts.
- `startClipboardPolling` retains `self` across every `Task.sleep` in `while let self`,
  and `deinit` does not cancel `clipboardPollTask`. Release ownership across suspension,
  cancel on teardown, and add a weak-reference deallocation/counter-stop test. Keep
  polling stopped after hide, focus loss and directive removal, and verify the actual
  observation frequency, not just that an interval constant equals one second.
- Clipboard refreshes for the same draft share `analysisGeneration`, while
  `startLivePreview` creates an untracked task. Make each preview request supersede
  older results/errors for that same draft as well as older drafts, and cancel or
  invalidate on lifecycle changes. Recheck current draft/client/compatible clip-span
  evidence before issuing a clipboard-reading subprocess. Retain priority seed and
  process-lane cancellation. Add delayed-result tests for two clipboard changes on an
  unchanged draft, directive removal, client replacement, focus/hide and teardown. Keep
  refresh debounced and counter observation at or below 1 Hz.
- `CaptureClipPresentation` supplies flattened `childLines` and entry lines for history,
  and `ClipCardView` renders both unconditionally. Render history once under its entry
  dividers, with non-history modes retaining exact lines, snippet destination,
  attachment rows and readable/unreadable image behavior. Preserve existing task-block
  deduplication. Add a presentation/render assertion that catches duplicated history.

## 4. Verify the repaired working tree

Format changed Swift lines only. Run `just portable-check` on a Swift host (use an
isolated scratch copy on athena when Swift is absent locally). Run `just check-scripts`
for fixture tooling and `just docs-check` if docs change. If bob-cli source changes
become necessary, run its `just check`. Do not run `just check-full`.

Follow the linked repo's compile rules: use a disposable scratch copy on the Mac, rsync
excluding `.git`/`.build`, run `Scripts/bundle.sh` and `just typecheck-tests`, and
remove the copy. Read `tailnet.md` through the memory skill first. On a Mac with Xcode,
run the affected hosted suites and the full hosted suite when feasible; Command Line
Tools alone has no XCTest, so the typecheck stub is not a test execution result. Use
`/sase_monitor` for long verification commands, with continuation including this plan's
closeout. The lander's bounded Mac SSH attempt timed out. If still unreachable, record
exactly **macOS targets unverified**, preserve actual portable evidence, and state the
hosted-test limit; do not pretend source inspection executes tests.

Known baseline failures belong to `bob-cli-f3`; record any overlapping typed-menu cases
this repair resolves there. New failures introduced by these repairs remain this work.
Recheck all new hosted cases structurally if execution is unavailable. Do not wait for
this turn's final commit, SHA, push, or its CI result to close: the finalizer commits
only after the provider turn ends. Existing CI logs are diagnostic evidence, not a
demand to publish a validation snapshot or install the app.

## 5. Finish the landing in this same turn

After the repairs and applicable checks, re-read the epic/children and the parent phase,
inspect any drift on both repos, and confirm all required work and note entries are
handled. Preserve the inherited decisions and rerun the retirement sweep in the Mac repo
(`git grep -n -e '@@template' -e TemplateStart -e start_template -e global_sub_bullet -e 'stay literal'`),
plus the removed span/header/global-parent checks. Do not change valid `@@route` or
semantic `ref_listen`/`pomodoro_note` fields.

Before closing ancestors, satisfy any open descendant/linked-plan readiness
requirements. If the runtime created a separate descendant bead for this tale, close
that separate bead normally once its code and validation are complete, and mark its
linked plan done if required. If the assigned bead remains `bob-cli-ey.3.1`, close it
only in the explicit step below. Nothing in readiness depends on this turn's future
commit. Do not force a successful nested landing.

Run `sase bead epic-symbols bob-cli-ey.3.1`. Resolve each listed exemption (wire it up,
privatize, add an appropriate non-test pragma, or remove it) or re-key its Justfile line
only to a still-open later bead that actually needs it. The land audit found no entries,
but recheck. Then run:

`sase bead close bob-cli-ey.3.1 --note "<source/commit and note audit; repaired submit, fixture and clipboard behavior; post-start integration review; actual validation and limits; no child follow-up proposals; bob-cli-f3 corroboration and any overlap resolved>"`

Run `just symvision` if available after closing (the audited bob-cli and Mac justfiles
had no recipe; report absence if unchanged). Open the plans repo through `/sase_repo`,
resolve the PLAN path from the bead, and set `status: done` in the frontmatter of
`plan:202610/mac_directives.md` (`202610/mac_directives.md` relative to that plans
repo). If close refuses leftover symbols, resolve them and retry. If unfinished
descendants are named, finish them or deliberately cancel/supersede genuinely abandoned
scope; never use `--force` to make a successful nested landing advance.

Re-read `bob-cli-ey.3.1` for its parent link and `bob-cli-ey.3` for its phase scope.
Confirm this repaired child fulfills that phase; retire any
`sase bead epic-symbols bob-cli-ey.3` entries, then close only that phase with
`sase bead close bob-cli-ey.3 --note "<integrated directive menus, grammar gate, retired-path removal and clip preview behavior, actual verification and limits>"`.
Leave the containing epic `bob-cli-ey` and its plan to its waiting land agent. If a new
blocker makes that phase incomplete or ambiguous, note it there and report it.

Use `/sase_final` as the final action, with commit decisions for every changed repo,
including plans. The final report must say which beads closed, which checks passed, and
any remaining macOS verification limit.
