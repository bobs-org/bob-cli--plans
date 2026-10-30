---
tier: tale
title: Finish the Task Wikilink Picker and land bob-cli-e7
goal:
  Repair the remaining wikilink picker defects, verify macOS behavior and renders, and
  close bob-cli-e7.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-e7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-e7
create_time: 2026-10-29 20:19:29
status: wip
---

- **PARENT:**
  [202610/wikilink_task_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/wikilink_task_picker.md)
- **BEAD:**
  [bob-cli-e7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e7/README.md)

# Finish the Task Wikilink Picker and land bob-cli-e7

## Scope and fixed decisions

Finish only the defects and missing acceptance work found while landing `bob-cli-e7`,
whose approved plan is `plan:202610/wikilink_task_picker.md`. This is one bounded
implementation task across bob-cli and its linked `bob-mac-capture` repository. The
tale's coder must finish this epic's landing; there is no later tale land agent. Do not
wait for the coder's own final commit, push, SHA, or post-commit CI before closing the
epic.

The epic's decisions remain final: **history = all** and **link_form = shortest**. Keep
Done and Cancelled tasks, shortest unambiguous cross-note targets, explicit `done/`
paths, and additive JSON at the existing schema version. Bob owns the grammar,
candidates, query, ranges, replacements, and vault writes. Swift owns presentation and
local filtering. Do not change release versions, CHANGELOG, memory, or unrelated
task-type/mention features.

Before work, use `/sase_repo` to open `bob-mac-capture` and `plans`, read the linked
repo's AGENTS.md, and use `sase artifact read` for the approved plan. Read `bob-cli-e7`
and its landing/triage notes with `sase bead read`. Use `/sase_memory_read` for
`decisions:mac-capture-is-a-thin-client`, `docs.md`, and `tailnet.md` as needed. Never
use Bryan's live vault for fixtures or his Mac checkout for scratch compilation.

## Evidence already established

All four phase beads are closed, but the feature is incomplete. The lander read all
child notes and reviewed CLI commits `30e093ac`, `53bf3dfa` and Mac commits `031f100`,
`8f303e2` against source and the approved plan.

Mac master run
[40975062017](https://github.com/bobs-org/bob-mac-capture/actions/runs/40975062017) at
`8f303e2` passed formatting and Build, then failed to compile the tests.
`CaptureWikilinkTaskPanelTests.swift:29,46,63,80` supplies `cursorAfter` and
`linkTarget` out of initializer order. The new
`CapturePickerDesignTests.wikilinkTaskRenderCandidates` has the same pattern. No actual
review of the twenty required PNGs has occurred. The Mac's SSH connection timed out
during this audit; GitHub macOS CI is available.

Further source findings:

- `capture_task_wikilinks::field_from_opener` scans to the physical line end and rejects
  forbidden punctuation even beyond the caret. A harness compiled from that exact
  function returns `None` for `Follow up ((foo and [existing](url)` at byte cursor 15,
  although the query is only `foo` and the replacement must preserve the suffix.
- For `Follow up ((foo bar`, the same function returns range `(10,19)` at byte cursor 18
  but `(10,15)` at byte cursor 12. The latter is the full-pool refetch position. Swift
  requires equal end ranges, so multiword chip reopen fails, and auto-open falls back to
  a partial filtered pool.
- `handleWikilinkTaskCompletion`, `openPickerFromChip`, `presentPicker`,
  `removePickerTrigger`, and `acceptPickerRow` discard or reconstruct Bob's query,
  descriptor ranges, and `cursor_after`. Reopening a closed `((foo))` chip can seed
  `foo))` by slicing the draft instead of using Bob's `query`.
- The grouped UI does not retain hidden-row dimming or Done/Cancelled capsules, the
  ID-less locator lacks the specified Add ID label, the detail link is plain secondary
  text despite a colored helper, and shared section headers can incorrectly say "Not in
  a Pomodoro" for this source.

The lander reviewed post-start CLI commits `8d7309ff`, `fe56228f`, `2631a1c9`,
`66ddba93`, `c0f94390`, `41338bcf`, and Mac `d41bb61`. Task types use the existing
discovery/scanning path, and the mention scanner consumes the plain block links this
picker inserts. Trailing `@@@` template rewrite and template completion gating need
coexistence regressions with the picker fixes. Review any additional drift before final
closeout.

## Implementation

1. **Repair Bob's unclosed field boundary.** In `src/native/capture_task_wikilinks.rs`,
   validate the query up to the caret without letting unrelated suffix punctuation
   suppress an otherwise valid unclosed field. Preserve the specified
   non-whitespace-fragment replacement boundary and the full closer-inclusive range for
   a valid closed token. Keep code, existing wikilinks, whole-item actions, Work Log
   children, triple parentheses, template headers, and non-character-boundary offsets
   protected. Add meaningful unit/CLI regression cases for a caret before Markdown
   links, parentheses, and code in the suffix, plus multibyte and closed-token variants.
   Extend the splice-validity coverage to assert both suffix preservation and successful
   dry-run capture for valid source drafts.

2. **Retain and use Bob's picker contract in the Mac.** Thread the decoded wikilink
   picker descriptor and query through picker state, chip state, Escape/reopen, and
   prompt restoration. Apply Bob's `marker_range` and `trigger_removal_range` verbatim
   with range/draft validation, and honor the selected candidate's `cursor_after` on
   ordinary acceptance. Continue splicing `wikilink_replacement` after ID assignment;
   its caret is after the returned link. Do not synthesize backend grammar/ranges as a
   compatibility fallback for missing/invalid wikilink descriptor data.

   For a full-pool refetch at the opener, validate the unchanged draft and same wikilink
   token, but do not require equality of end offsets that Bob intentionally makes
   caret-dependent. Preserve the original Bob-authored insertion/removal ranges and
   original query while using the empty-query response's complete candidate pool. Never
   replace the original insertion range with the shorter refetch range. Closed-token
   chip reopen must use Bob's query without closing parentheses. Clearing or changing
   the local filter must reveal all eligible tasks without additional per-keystroke
   subprocesses. Preserve stale-response and stale-draft guards.

3. **Complete the Mac tests and presentation.** Correct initializer ordering in both new
   macOS test files, then fix any other e7 compile/test errors exposed by actual macOS
   verification. Generate or reconcile fake-bob JSON against a freshly built bob-cli and
   a synthetic vault; ensure UTF-8 ranges and `cursor_after` values are realistic rather
   than unrelated constants. Extend `CaptureWikilinkTaskPanelTests` with the missing
   behavioral cases: caret/delete/backspace chip instead of auto-open;
   second-parenthesis burst auto-open; closed and multiword chip reopen; full-pool
   filter clearing; descriptor-controlled Backspace and highlight; returned caret;
   prompt failure and Escape/filter restoration; missing replacement compatibility;
   Command-/Control-Return follow-ups; and `--allow-closed` for both Done and Cancelled
   ID-less rows. Use bounded async waits, not untimed continuations.

   Finish the approved UI details in `WikilinkTaskPickerPresentation.swift` and shared
   picker views, scoped so existing pickers retain their behavior: hidden rows subdued,
   grouped completed rows labeled, ID-less Add ID affordance, link-colored detail text,
   and headers appropriate to task wikilinks. Reuse available Bob metadata for note
   icons; do not infer vault semantics in Swift. Verify the remaining approved keys and
   accessibility behavior while exercising the real renders. Format only changed lines.

4. **Verify integration and complete the missing acceptance.** Add focused regression
   coverage showing the trailing `@@@` rewrite and existing `@@template:` gate coexist
   with `((` and keep draft-global byte ranges correct after rewrites. Verify the
   inserted ordinary `[[target#^id]]` remains compatible with the landed mention scanner
   and type-aware capture path; reuse existing behavior instead of adding a second
   parser or writer. Update only documentation made inaccurate by the repair.

   Run bob-cli `just check` (prefer `sase tool run check`) and `just docs-check`. Do not
   run `just check-full`. On a host with Swift run the Mac repo's `just portable-check`;
   use an isolated scratch copy on athena if needed. Run `Scripts/macos-ci-check` in the
   Mac repo for the current working tree using `/sase_monitor` for the long CI wait.
   This script tests a temporary snapshot and does not depend on this tale's own
   eventual commit. Recheck all required steps even after a lint/test failure. Fix all
   e7-caused failures. Follow the existing task triage policy for independently caused
   failures without turning them into extra epic scope.

   Best-effort compile a scratch copy on the Mac with `Scripts/bundle.sh` and
   `just typecheck-tests` if it is reachable, following its AGENTS.md; remove the
   scratch copy afterward. Offline hardware does not waive macOS CI. Download the exact
   successful test run's `render-fixtures` artifact, view all five wikilink states at
   760/620 points in light/dark (20 images), and fix alignment, truncation, selection,
   locator colors, and contrast defects. Repeat affected checks/renders after code
   changes. Record the successful CI run and the visual review result in an epic note.

## Final step: close bob-cli-e7 in this coder's turn

The original follow-up triage is complete. `bob-cli-e7.1` note #2 and `bob-cli-e7.2`
note #2 describe one defect: seven stale CLI theme/type assertions caused by active
`bob-cli-e6`. Source commit `c0f94390` (e6.3) repairs all seven; the lander recorded
corroborating source evidence on e6 and every disposition on e7. No task was created:
this was active-epic work, and closed `bob-cli-de` concerned a different earlier
catalog-count bug. Include these outcomes in the final close note, and confirm the
repaired assertions through the required check. There were no other proposals.

After implementation and passing required verification:

- Re-read `bob-cli-e7` and every descendant/note and confirm all are complete; read
  `plan:202610/wikilink_task_picker.md` through `sase artifact read` and confirm this
  tale fulfills its remaining requirements. Recheck any commits landed since this audit
  in bob-cli and bob-mac-capture and integrate any direct conflict or duplicate. Do not
  wait for this work's own commit.
- Run `sase bead epic-symbols bob-cli-e7`. Resolve every listed exemption (wire up,
  privatize, add an appropriate non-test pragma, or delete under Symvision policy), or
  re-key it only to a still-open later bead that demonstrably needs it. The audit found
  none; recheck rather than assuming.
- Run `sase bead close bob-cli-e7 --note "<verification>"`. The note must name repaired
  defects, compliance with history=all/link_form=shortest and the thin-client rule,
  source/commit and drift review, actual check/CI/render results, and the follow-up
  dispositions above. If rejected for symbols, fix them and retry. If named phases are
  incomplete, finish/reopen them. Never force a successful nested landing or force
  merely to make close pass. Only deliberate cancellation/supersession may use the
  corresponding `--force --reason ... --resolution canceled|superseded` path.
- After closing run `just symvision` if the recipe is available. Neither repo currently
  defines it; explicitly record absence if still unavailable.
- Set `status: done` in the frontmatter of the epic's plan, canonical ref
  `plan:202610/wikilink_task_picker.md`, in the `plans` repo opened with `/sase_repo`.
  Resolve its current path using `sase artifact path`/the PLAN line from
  `sase bead read`, never a hard-coded workspace path.
- Run `sase bead read bob-cli-e7 -r "Need the parent link"`. This audit found no
  `parent_bead`, so no ancestor close is expected. If the current record differs, follow
  the original landing prompt's normal, non-forced ancestor readiness/closeout rules and
  stop at any incomplete parent with a note.
- Finish with `/sase_final`, declaring all modified repositories. This is when the
  coder's implementation is committed; do not defer epic closeout until after that
  commit or its CI.
