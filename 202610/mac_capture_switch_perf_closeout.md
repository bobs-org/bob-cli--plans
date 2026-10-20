---
tier: tale
title: Complete panel-switch verification and land bob-cli-a8
goal: Finish refresh and caret coverage, record performance evidence, and close bob-cli-a8
  with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-a8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a8
status: done
---

- **PARENT:**
  [202610/mac_capture_panel_switch_and_perf.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/mac_capture_panel_switch_and_perf.md)
- **BEAD:**
  [bob-cli-a8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a8/README.md)

# Finish panel-switch verification and land bob-cli-a8

Complete only the remaining work in epic `bob-cli-a8`, then close it in this same coding
turn. This tale has no land agent. Its own commit is made after the turn ends: do not
put closeout after obtaining this work's commit SHA, pushing it, or waiting for its CI
run.

## Context and fixed decisions

Read `sase bead read bob-cli-a8 -r "Need the landing audit and closeout requirements"`
and its two landing notes. The original plan is
`plan:202610/mac_capture_panel_switch_and_perf.md`; read it through
`sase artifact read`. All three phases are closed. The implementation in linked
`bob-mac-capture` is:

- `e99cec6`: panel switching, routing, model state, coordinator, and docs (`a8.1`).
- `554c69e`: Refs ranking, prepared-cache identity, inspector and Today no-op guards,
  coalesced library publications, and refresh window (`a8.3`).
- `1d5735a`: Capture refresh window, no-op guards, editor/filter lookups, fuzzy
  matching, and signposts (`a8.2`).

Use `/sase_repo` to open `bob-mac-capture`, then read its `AGENTS.md`. All Swift paths
below are relative to that opened repository. Do not edit installed applications or
Bryan's Mac checkout. The existing decisions are final:

- `open_surfaces=switch`: Ctrl+Tab and Ctrl+Shift+Tab work through open surfaces; hide
  performs their existing cleanup. IME composition passes through, repeats are consumed,
  and task-goto dispatch remains protected.
- `arrival_reset=keep`: `.panelSwitch` restores the Refs view, clears pending arrivals,
  and disables arrival-driven resets for that presentation. Ordinary shows keep their
  existing arrival behavior.
- `show_refresh_window=coalesce`: show-triggered targets, templates, and Refs Today
  refreshes use the five-second window. Required non-show triggers stay unthrottled. The
  idle agenda must still revalidate on every show.

The app remains a thin client of bob. No grammar, vault reads, or semantic decisions
move into Swift. No memory changes are needed.

## Verified findings and follow-up dispositions

Audited evidence is `file:explicit:78b480232df324bc6ff25313`. Read it through
`sase artifact read` if inspecting the comparison. Current macOS CI run `39511067813` at
`1d5735a` compiled and ran the combined tree. Compared with pre-epic run `39499997621`
at `785508c`, its newly failing node is:

`Tests/BobMacCaptureTests/RefsPanelModelTests.swift::testOpenRefreshesTodayWithoutRefreshingSnapshot`

The test still waits for a second Today process after only one second. That conflicts
with the new five-second window; the production throttle must remain.

The lander completed follow-up triage before this proposal:

- `bob-cli-a8.1` note 1, the sole `PROPOSED FOLLOW-UP`, is declined as currently
  unreproducible. On bob-cli `787d62cb`, both named Material tests passed with the
  project environment:
  `uv run --project tools/docs --group material pytest -q tools/docs/tests/test_templates.py -k 'test_templates_in_material or test_component_chips_in_material'`
  (2 passed, 3 deselected). Phase `a8.3` also reported its later full docs-check pass.
  Do not file a speculative docs task.
- Four unrelated failures reproduce before the first epic commit and at current HEAD.
  They are now ready CI tasks: `bob-cli-ad` (third-@ template-start completion timeout),
  `bob-cli-ae` (grown-list fixture marks the old first row), `bob-cli-af` (duplicate
  visibility callback on hide), and `bob-cli-ag` (Tabs next-show preselection). Existing
  `7h`, `8e`, and `8j` concern different nodes; related links preserve context. Do not
  duplicate these tasks or require their fixes for this epic's closeout.

The original plan's hosted caret round trip and ranking before/after numbers are still
missing. Capture targets uses `Date()` directly in AppDelegate, and its window tests
only exercise the stateless predicate, leaving the promised injected clock and refresh
reason coverage incomplete.

## 1. Repair and exercise the refresh contract

In `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, replace the stale
two-immediate- opens expectation with deterministic clock-driven coverage through
`RefsPanelModel.refreshForOpen()`:

1. A first open starts one Today refresh while a fresh snapshot is reused.
2. A second open inside five seconds adds no Today process or snapshot pass.
3. Advancing the injected clock to the boundary/outside the window permits another Today
   refresh, still reusing the fresh snapshot.
4. An explicit non-show Today refresh inside the window still runs.

Reuse the `RefsLibrary(now:)` seam and fake-bob argv/generation observations. Await
settled work with bounded conditions. Do not replace useful assertions with arbitrary
long sleeps, increase timeouts to conceal the changed contract, or disable the test.
Update stale comments saying Today refreshes on every open in
`RefsPanelModel.refreshForOpen` and `RefsLibrary`.

Finish the targets clock seam in `AppDelegate.refreshTargetsWhenPossible(reason:)`. Use
one injected current instant for both the skip decision and recorded start time. Keep
the production default as `Date.init`, and preserve the shared interval already defined
by `CaptureShowRefreshWindow` / `RefsLibrary.pullThrottleInterval`. Prefer a small
testable orchestration seam to a broad AppDelegate refactor. Exercise the actual
reason-to-refresh decision with a fake client or refresh spy, not just interval math:
first show, skipped show inside the window, show at/after its boundary, and launch,
watcher, recheck, and failure-recovery reason bypasses. Existing production call sites
must use the tested seam. Do not invent a new background recovery loop.

Keep templates' queued non-show follow-up and the agenda's per-show refresh unchanged.
Retain existing no-op publication and fuzzy oracle tests.

## 2. Complete hosted state-preservation coverage

Add the original plan's missing hosted Capture round-trip test to the existing
`Tests/BobMacCaptureTests` controller/hosted harness. Use the real controller-created
editor, a nonempty draft with a caret away from the end, a template origin, and an
error. Drive the coordinator from Capture to Refs and back, or faithfully invoke its
real close-retaining-draft and show paths. Assert after bounded layout/focus settling:

- Only the destination panel is visible at each hop.
- Draft, template origin, error, and model selection survive.
- The backing `NSTextView.selectedRange()` is the original non-end caret after return.

The existing model-only test is insufficient for the last assertion. Reuse existing
hosted infrastructure, clean up panels on every exit, and never await an untimed gate.
If this exposes an epic-caused selection/focus defect, fix the narrow production path
and keep the regression test. Preserve the deferred main-queue switch and stale-origin
guard. Existing router and coordinator tests already cover repeats, modifiers, modal
surfaces, copy-return, and deferred callbacks; retain them.

## 3. Finish measurements and check integration drift

Record comparable before/after ranking measurements, as required by the original plan.
Use the existing `swift run refs-rank` harness, a deterministic synthetic snapshot of at
least 10,000 items, fixed `--now` and `--tz`, and the same build configuration/input on
baseline `e99cec6` (before Refs perf) and the repaired tree. Include browse and search
cases, warm up, take several measurements, report medians and method, and compare ranked
output for equality. Exclude compilation from timing. Scratch copies must be built from
the checkout opened through `/sase_repo`; do not fetch repository files by other routes.
Keep benchmark artifacts outside tracked product source unless they are useful reusable
tooling.

The prior Mac SSH attempt timed out. Retry best-effort through the tailnet instructions.
If reachable, use the added signposts to measure Capture prepare/dismiss, switch, and
template refresh on comparable before/after scratch builds, recording scenario and
numbers. If unreachable, explicitly record that Mac timings could not be measured; do
not fabricate results or block forever. Portable ranking measurements are still
required.

Review commits that landed since `1d5735a` in the linked repo and since `787d62cb` in
bob-cli before finalizing. The lander already reviewed concurrent `785508c` (Highlights
Command-T entry) and `a9a5c9b` (gold underline rendering): they use the shared panel
coordinator and preserve raw annotation equality, respectively; no additional production
integration was found necessary. Check that the final tests and cache paths preserve
both. The new capture-related app work belongs to active epic `bob-cli-ab`; integrate
any changes that have actually landed and conflict with this work, without waiting for
unlanded phases. Short Pomodoro spelling still comes from bob's process contracts; its
separate agenda presentation enhancement is already task `bob-cli-a9`.

Store concise measurement/verification evidence with `sase artifact create` and append a
note to `bob-cli-a8` with the results, integration review, and any limitations.

## 4. Verify the final files

- Run `just portable-check` and `just format-lint` in `bob-mac-capture` on a Swift host.
  This compiles/tests CaptureCore and RefsCore; it does not validate AppKit targets.
- For changed `Sources/BobMacCapture*` / `Tests/BobMacCapture*`, make a best-effort Mac
  scratch copy excluding `.git` and `.build`, run `Scripts/bundle.sh` and
  `just typecheck-tests`, and delete that scratch copy. Never install or use Bryan's
  checkout. The CLT-only Mac lacks XCTest; do not use `--build-tests` there. If it is
  offline, explicitly report the new macOS targets as unverified, as AGENTS.md requires.
- Run the relevant hosted tests when an XCTest-capable Mac is available. Attribute any
  baseline failures to `ad/ae/af/ag` with evidence; repair any new epic-caused failure.
  Do not wait for this turn's own future commit or CI to close the epic.
- Run the primary bob-cli `just check` file-change gate. Do not run `just check-full`.
  The linked Mac repo currently has no `check` recipe; its prescribed equivalent is
  `portable-check` plus the Mac checks above. Use `/sase_monitor` for long commands and
  carry the entire remaining closeout in its continuation.
- Run linked `just docs-check` against the primary bob-cli checkout with `BOB_CLI_DIR`
  set. Open any needed mounted repo through `/sase_repo` and provide its explicit docs
  override so the wrapper does not locate or clone another checkout. Read `docs.md`
  memory first if documentation edits are needed. Distinguish any confirmed independent
  infrastructure issue from remaining feature work.

## 5. Close bob-cli-a8 and its plan in this turn

This is the tale's final implementation step; do not defer it to another lander.

1. Re-read `bob-cli-a8` and every child with
   `sase bead read ... -r "Need final descendant readiness and all notes"`. Confirm all
   descendants remain closed, every new note is addressed, and the original plan and
   fixed decisions are satisfied. Carry the follow-up dispositions above into the close
   note. If a new distinct out-of-scope issue appears, apply `/sase_new_task` and record
   its disposition too.
2. Run `sase bead epic-symbols bob-cli-a8`. Resolve every entry by wiring it up,
   privatizing it, adding a justified non-test pragma, or removing it under the
   Symvision epic-whitelist policy. Re-key a Justfile entry only if a specifically
   identified still-open later bead still needs it. The lander's result was empty;
   verify again. Do not leave stale exemptions for a later agent.
3. Append the final verification evidence and exact check outcomes to the epic, then
   close normally, using a note such as:

   ```sh
   sase bead close bob-cli-a8 --note "Verified all three phases, their notes, and e99cec6/554c69e/1d5735a against the switch/keep/coalesce decisions; completed the refresh-clock and hosted-caret coverage and performance evidence recorded in the final landing note. Reviewed and integrated post-epic drift. The a8.1 Material-test proposal is no longer reproducible; independent baseline Mac failures are tracked by ad/ae/af/ag. Final verification results and Mac availability limits are recorded above. No epic-symbol exemptions remain."
   ```

   Make the note accurate for the actual results. If close rejects leftover symbols,
   finish cleanup and retry. If named phases are incomplete, finish or reopen them.
   Never use `--force` merely to make closure succeed or advance a successful nested
   landing. Use forced canceled/superseded resolution only for a deliberate abandonment
   with an explicit reason, not for this successful repair path.

4. After closing, run `just symvision` wherever the recipe is available and record its
   outcome. Neither inspected repo currently supplies it; if still absent, explicitly
   record unavailability together with the empty epic-symbol audit.
5. Open the plans sidecar with `/sase_repo`, use audited artifact access for the
   original plan, and set `status: done` in the frontmatter of
   `202610/mac_capture_panel_switch_and_perf.md`, preserving its other metadata.
6. Run `sase bead read bob-cli-a8 -r "Need the parent link"` after closing. This epic
   currently has no `parent_bead`, so finish normally. If the linkage changed, do not
   guess: a phase parent is closed normally only after verifying its scope; leave its
   containing epic for its lander. A plan parent requires re-reading its previous
   landing note, all descendants/notes, linked plan, and post-child drift, plus
   rerunning readiness checks, symbol cleanup, normal close, available symvision, and
   plan status update. Repeat only through fully complete direct plan parents. At the
   first incomplete or ambiguous ancestor, record its blocker and report it.

Use `/sase_final` as the last action before the normal final response, declaring every
repository changed in this tale. The host commits after the turn; that is compatible
with closing the epic now.
