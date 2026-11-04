---
tier: tale
title: Render past Pomodoro stacks in Bob Mac Capture
goal:
  Complete bob-cli-h2.6 with chronological past-work cards, live clock refreshes,
  fixtures, docs, and verification.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-h2.6
proposed_with: codex/gpt-6-astra
bead: bob-cli-h2.6
create_time: 2026-11-04 16:53:28
status: wip
---

- **PARENT:**
  [202611/past_pomodoro_recording.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/past_pomodoro_recording.md)
- **BEAD:**
  [bob-cli-h2.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h2/bob-cli-h2.6.md)

# Render past Pomodoro stacks in Bob Mac Capture

Implement the Mac phase of `bob-cli-h2.6`, governed by
`plan:202611/past_pomodoro_recording.md`. A successful `x=` preview must show one
chronological **Record past work** card per stack, the effects Bob resolved for each
member, and any trims to closed history. Refresh the preview as the wall clock crosses a
five-minute boundary. Complete and close only `bob-cli-h2.6`.

## Scope and evidence

The phase was `in_progress`, assigned to `bob-cli-h2.6`, with no earlier work notes. Its
prerequisite `bob-cli-h2.5` was closed. The primary checkout at `78834b66` already
contains the grammar, recording engine, overlap guard, and documentation. The linked Mac
repository at `0245a75` contains no `pomodoro_past` or Record past work implementation
in Sources or Tests. The initial `sase bead epic-symbols bob-cli-h2.6` reported no
entries; check again before close. Recheck these facts on implementation start and
retain any work that has since landed. Do not change bead status manually or reopen an
already active phase.

This is one bounded implementation task: additive wire models, one presentation and
card, one preview-refresh lifecycle, and focused fixtures/tests/docs. The existing close
rows and live-preview request guards provide the integration seams. There is no
unfinished earlier attempt requiring a child epic.

Open `bob-mac-capture` with `/sase_repo`, use only its returned checkout, and read its
`AGENTS.md`. Paths below are relative to that repository unless explicitly identified as
bob-cli paths. Read the applicable reference memories through `/sase_memory_read`:
`decisions:mac-capture-is-a-thin-client`, `docs.md`, and `tailnet.md` for remote
validation. Read the governing epic with `sase artifact read` and the assigned bead with
`sase bead read`.

The epic decision `memory_decision_record = no` is final. No memory files are in scope.
Bob remains the sole authority for grammar, names, windows, overlap repairs, task
effects, and vault writes. The app renders Bob's data and submits the entire draft
through the existing aggregate capture call. Do not add a grammar gate for this feature,
edit Cargo versions or CHANGELOG, or implement the Rust phases again.

## 1. Add tolerant wire models and preserve close effects

Extend `Sources/CaptureCore/CaptureModels.swift` so both `CaptureParseResponse` and
`CaptureParseItem` decode optional `pomodoro_past`. Mirror the real parse contract from
bob-cli's `src/native/capture_parse.rs`: `raw`, optional `name`, `blocks`, `minutes`,
`repair`, stack position including `offset_blocks`, and nested `close` using
`PomodoroCloseSpec`. Add initializer defaults and coding keys so existing fixtures and
call sites continue working.

Add optional `pomodoroPast: PomodoroPastSummary` to `CaptureCommandSuccess`. Model the
authoritative success contract in bob-cli's `src/native/capture/output.rs`:

- `raw`, optional `name`, `blocks`, `repair`, `anchor`, `closed_at`, and final `line`;
- `target` with `source` and original `line`;
- `stack` with one-based `index` and `size`;
- `window` with `start`, `end`, and `minutes`;
- `trims`, each with `line`, `name`, `before`, `after`, `minutes`, `clock`, and
  `calendar`;
- `close`, decoded using the existing `PomodoroCloseSummary`.

Use `decodeIfPresent`, defaults for additive arrays, and a tolerant boundary for
malformed optional past metadata (`try?` where needed) so older successes still decode.
Keep schema validation and errors for the established required envelope. Do not
synthesize a top-level `pomodoroClose`: Bob intentionally omits it to avoid presenting
this operation as a running-session close.

Include the nested close's unblocked/keep-moving/relation offers, held reasons, and
relation-check state in `CaptureSuccessorOfferRows.swift` and
`CapturePanelModel.makePullInPreviewSnapshot`. Preserve Bob's order and the existing
deduplication by offer key. Audit the other close-effect consumers used by the card and
submission so a past close does not lose its continuation, task outcomes, or explicit
successor choices. Reuse existing close-row builders via a small initializer/helper
accepting a close summary if necessary; avoid copying the close presentation
implementation or changing normal `x` behavior.

## 2. Build one presentation and card per stack

Add a pure `CapturePomodoroPastPresentation` in CaptureCore. Group adjacent
`kind == "pomodoro_past"` captures using Bob's stack index/size, starting a new group at
index 1 and ending at a non-past item. Never merge separate stacks just because their
sizes or anchors match. Preserve typed/member order, which Bob has already resolved
chronologically; sorting clock strings would mishandle midnight. Malformed or incomplete
group metadata must degrade safely without losing items.

Each group exposes:

- **Record past work** heading and total minutes from the supplied member minutes;
- a noninteractive timeline such as `09:00 ── BUILD 10m ── 09:10 ── FIX 15m ── 09:25`;
- member name, exact range, duration, and the nested close task rows, typed Work Log
  details, transitions, and continuation;
- amber trim rows identifying the affected name, before/after spans, minutes it
  **gives**, and `(calendar)` when reported. Each trim appears once even though Bob also
  supplies a structured `pomodoro_past_trim` warning;
- accessible text covering time, outcomes, and trims without depending on color.

Integrate grouping at the item loop in `CapturePanelView.swift` (`PreviewPane`), where
`CaptureListPreview` currently coordinates per-item rendering. Render one past card for
each group and retain ordinary item order, list behavior, batch-level task/Pomodoro
blocks, plan-budget output, and explicit offer UI. Reuse existing close task row
components inside each member rather than nesting full running-session Close cards. Keep
large timelines and member text usable at minimum panel width. Render midnight end
`0000` faithfully from Bob's window.

Connect the presentation to the panel footer action **Record**, VoiceOver summary, and
`NotificationService.swift` title **Recorded past Pomodoro**. For homogeneous past
stacks, use the new presentation; mixed batches retain the established batch submission
semantics. Backend refusals continue through the existing error path, which clears
obsolete success cards. Missing optional metadata must not crash or mislabel a past
capture as a normal Close card.

## 3. Style the new spans and use Bob's name completion

In `CompletionRowContent.swift`, map `pomodoro_past` to the existing session operator
category and add a distinct warning category for `pomodoro_past_repair`. Supply its
color in `CaptureEditorPalette.swift`, with light/dark coverage and no changes to the
existing close-selection `!` style.

Check `CapturePanelModel.completionSpanKinds`, quiet incomplete-picker handling, and the
Pomodoro-name completion context switches in model/view code against real
`capture-parse` and `capture-complete` fixtures. Recognize Bob's `pomodoro_past_name`
where the existing name-menu presentation applies. Request and apply Bob's candidates
and replacement ranges; do not parse `x=` in Swift or reuse start-only rewrite logic.
Name completion must retain `@D`, repair `!`, and the second `=` close selection.
Incomplete names and selections should preserve the existing calm incomplete-state
behavior.

## 4. Refresh a visible past preview at five-minute boundaries

Add a cancellable clock-boundary task owned by `CapturePanelModel`, using an injectable
clock/scheduling seam for deterministic tests. While the panel shows a valid current
past preview, schedule one dry-run request at the next five-minute wall-clock boundary
and re-arm for subsequent boundaries. Local clock arithmetic is only for deciding when
to request Bob again; all displayed session windows and effects come from the new
response.

Reuse `startLivePreview` and its request/analysis generation, draft identity,
process-client identity, and clipboard-authorization checks. A clock refresh must not
alter the draft, selection, completion, chosen offers, or submit anything. Preserve
valid remembered choices through the existing snapshot rules; never let an old response
resurrect retired offers or a stale card. Avoid triggering a new request on every view
render or creating overlapping tickers.

Cancel/re-arm appropriately on draft/parse changes, panel hide/show, client replacement,
submission, disappearance of past previews, deinit, and clock/day changes or wake. A
hidden panel does no periodic work; showing it refreshes from current Bob data. Failed
requests must leave the normal refusal visible and not restore the old success. Test the
precise boundary behavior against Bob's seconds-discarding clock contract, including an
exact boundary and a jump forward.

## 5. Generate fixtures, add focused tests, and document behavior

Build `bob` from the primary checkout containing the landed upstream phases, and add
`Scripts/generate-pomodoro-past-fixtures.sh`, following the existing fixture scripts.
Require an explicit built `BOB_BIN`, use isolated synthetic vaults and
config/state/cache, pin `BOB_NOW` and the day file, and normalize only unstable
temporary paths. Never read or modify the personal vault. Capture real parse,
completion, success, and refusal JSON, retaining the expected nonzero status for
refusals. Register script syntax checks alongside comparable generators.

Cover these meaningful cases:

- Single `x=build`, two-member `x=build@2 x=fix@3`, multiple stacks separated by another
  item, and a chain mixing an ordinary close with past recording.
- The approved BUILD+FIX repair example against PLAN 0855–0920, including one
  deduplicated trim; a calendar trim with textual `(calendar)` indication.
- Nested close task effects, Work Log details, continuation, and explicit successor
  offers. An older payload without `pomodoro_past` still decodes.
- Malformed optional metadata degrades safely; midnight ranges remain in member order; a
  refused overlap clears a previous successful preview.
- Parse spans distinguish repair `!` from close-complete `!`; Bob's completion replaces
  only the name and preserves the rest of the token.

Add portable decoding/grouping/presentation/semantic tests under
`Tests/CaptureCoreTests`. Add hosted lifecycle tests for one refresh per boundary,
cancellation when hidden or edited, client replacement, stale responses, clock changes,
and preservation of ordinary preview behavior. Use bounded async waits; do not introduce
continuations that can hang the test suite.

Add a `Tests/BobMacCaptureTests` design test following `PomodoroCloseDesignTests.swift`
for a plain stack and a repair stack in light/dark appearances at normal and minimum
panel widths. Render and inspect images when the Mac test environment supports it; the
test must also compile through the required Mac type-check gate.

Update `docs/capture-previews.md` with the card, timeline, trim wording, and refresh
behavior. Update `docs/protocol.md` with grammar 12 and additive past parse/summary
fields, tolerant compatibility behavior, and the unchanged existing grammar gate. No
memory changes or generated image assets unrelated to this feature are needed.

## Verification and completion

Format only touched Swift lines under the repository's `.swift-format` rules. Run the
fixture generator and focused tests, then the required gates:

1. `just portable-check` in the Mac repository. This host has no `swift` on PATH; if
   still missing, use an isolated scratch copy on `athena` and compile/test there. Read
   `tailnet.md` first and clean up only the copy created for this work.
2. Best-effort scratch compilation on `mac`: rsync the authorized linked checkout
   without `.git` or `.build`, run `Scripts/bundle.sh` and `just typecheck-tests`, then
   remove that scratch copy. Never use Bryan's working checkout or run `just install`.
   The Mac's Command Line Tools have no XCTest; full hosted test execution belongs to an
   XCTest-capable macOS environment/CI. Fix compilation errors; if the host is offline,
   explicitly report the macOS targets as unverified rather than claiming compilation
   succeeded.
3. `just docs-check`, using `BOB_CLI_DIR` to point the linked wrapper at the current
   authorized primary checkout so it does not fetch a separate CLI clone. Use
   `/sase_repo` for any additional mounted repository the check needs.
4. `just check` in bob-cli, as required by the governing epic.

Use `/sase_monitor` for long commands and follow its staging/finalization rules; do not
end a turn with an ordinary tool subprocess still running. Do not repeat passed full
gates unless changes or evidence justify it. If a check fails, fix regressions
introduced here. A failure reproduced identically on the clean base does not keep this
phase open: append a `PROPOSED FOLLOW-UP:` note to `bob-cli-h2.6` with the reproduction
and any existing tracking bead. Do not create beads. Include any platform verification
limitation in the bead evidence and final report.

Before completion, rerun `sase bead epic-symbols bob-cli-h2.6`. Resolve each remaining
symbol or re-key its justfile exemption to an appropriate still-open bead. Then run
`sase bead close bob-cli-h2.6 --note "<implemented behavior and actual verification>"`.
Close no parent or ancestor, even if an ancestor completion instruction appears
elsewhere. Finish through `/sase_final`, declaring the linked Mac repository and any
other repository actually changed. The phase is complete when its new presentation,
refresh behavior, fixtures, docs, and required verification evidence are present, with
no new unexplained check failure.
