---
tier: tale
size: medium
title: Finish Link to Today interaction safety and land bob-cli-8f
goal:
  Repair the remaining Link to Today defects, verify the integrated CLI and Mac app, and
  close epic bob-cli-8f in this coder turn.
proposed_by: bbugyi200.apollo.bob-cli-8f.land
bead: bob-cli-8f
create_time: 2026-10-16 13:31:04
status: wip
---

- **PARENT:**
  [202610/refs_link_to_today.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/refs_link_to_today.md)
- **BEAD:**
  [bob-cli-8f](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8f/README.md)

# Remaining work and landing for bob-cli-8f

This is one bounded repair tale, not a new feature or another epic. Its coder owns the
landing: no land agent resumes after this tale. Close the epic in the same turn as the
final code, before host finalization commits that code. Do not wait for this work's
commit SHA, push, or CI run before closing.

## Audited starting point

The land audit read epic `bob-cli-8f`, every note on its three closed children, and
`plan:202610/refs_link_to_today.md`. The implementation is in bob-cli commit `a3c27210`
and bob-mac-capture commits `ef6522b` and `3a6db2b`. Reviewed heads were bob-cli
`280321a3` and bob-mac-capture `3a6db2b`, both on master. A final fetch found CLI
`ff97c620` (articles vault CLI), which was reviewed and fast-forwarded; its
command/completion and justfile changes preserve ref-today and the earlier coverage
fixes. The Mac head was unchanged. The epic has no parent bead at audit time. Its linked
plan is `202610/refs_link_to_today.md` inside the plans repository and remains
`status: wip`.

The final decision is **after_link = stay**: preserve reference selection, keep the
panel open, show its Today pill immediately, and anchor the toast above the footer. Bob
remains the sole authority for vault semantics; Swift selects from Bob's results and
submits selectors, never capture grammar or vault writes. Do not change task-lane,
dependency-drop, schedule, guard, or capture grammar rules.

Already implemented: CLI resolver/refusals, four selectors, shared capture planner and
atomic commit, typed errors, additive JSON, help/docs, parity tests; portable
response/card/presentation models and independent preview/write process lanes;
card/menu/key/footer/toast/notification integration. Keyboard hints are already in
`RefsFooter`: do not duplicate them in the card. A model-owned optimistic overlay is
acceptable; no architectural rewrite into the library is required.

The CLI phase reported its focused suites green with unrelated baseline failures. The
Mac phase's initial missing build verification was completed by its note #2: release
bundle and all test targets typechecked; portable tests had passed. These are
prior-agent evidence, not checks run by this lander. Reverify changed code.

## 1. Open repositories and establish the current baseline

Use `/sase_repo` to open `bob-mac-capture` and `plans`, using only printed paths, and
read applicable AGENTS.md files. Read the original design using
`sase artifact read plan:202610/refs_link_to_today.md "Need approved landing scope"`.
Use `/sase_memory_read` for `decisions:mac-capture-is-a-thin-client` and read
`tailnet.md` before using remote Swift hosts. Do not edit memory.

Re-read `bob-cli-8f` and its three children with `sase bead read`, including the
lander's LAND AUDIT / FOLLOW-UP TRIAGE and LAND INTEGRATION / REMAINING WORK notes.
Check new commits since the audited heads, including the current base branch if there is
now a PR. Preserve unrelated work.

The earlier integration review covered all CLI commits through `ff97c620` since the
epic's first commit, plus work since epic creation: snapshot/transclusion/project
metadata, article heading selection, capture-template completion/output repairs, and Mac
saved-template protocol/checkout, settings imports, signing and Accessibility. These
preserve the shared capture item envelope and independent Refs lanes and need no feature
migration. In particular, retain the fixes in `f2ced0c5` and do not reintroduce a second
capture or task-link implementation.

## 2. Correct the CLI destination after moving to a duplicate headline

In `src/native/capture_task_toggle/ledger.rs`, `destination_after_found_move` looks up
an entry by digest after moving a task-link subtree. For entry-ref selectors, multiple
identical open headlines fall back to `planned.clone()`, whose line number is from
before the move. An unnamed entry is a valid target; two unnamed placeholders
legitimately have identical digests.

Add a regression first. With this daily note (and the usual Ready v2 reference fixture),
select the **second** unnamed row using its exact `-P` listing ref:

```markdown
## Pomodoros

- [ ] () — SOURCE
  - [[sase#^ref-attention]]
- [ ] ()
- [ ] ()
```

The selected headline starts at line 5, moves to line 4 after the source child is
removed, and its link is at line 5. The current fallback reports destination line 5.
Correct the endpoint using the known selected entry and subtree movement mapping, not a
digest-only lookup or another first-match heuristic. Preserve `PomodoroScan::by_ref`
stale/ambiguous refusal semantics and capture's existing Named/Implicit behavior. Cover
source-before-destination, source-after-destination, and a multi-line moved subtree with
duplicate unnamed headlines. Check both the mutated ledger and the JSON destination line
against its actual headline; retain the original parity and no-write-on-refusal tests.

Use the existing tests in `src/native/capture_task_toggle/tests.rs` and
`tests/cli/ref_library/today.rs`. Avoid unrelated ledger refactoring.

## 3. Make listing, preview, reload and write lifetimes independent

Work in `Sources/BobMacCapture/Refs/RefsPanelModel.swift` and the portable
`Sources/RefsCore/RefsTodayLinkCard.swift` as needed.

- Today listing calls currently use the card-open sequence only. A fast submit during
  initial loading sets `.running`, but `failTodayLinkListing` can later call
  `card.fail`, change it to `.failed`, and release all write guards while the write
  subprocess is still active. Give listing/reload requests their own identity and reject
  obsolete successes **and errors**, including after submit, close, or a newer reload.
  Do not globally ignore `card.fail` while running: genuine write failures must still
  reach the user.
- Preview tasks currently start a process on every edit. Add the planned roughly 90 ms
  cancellable debounce. Snapshot `previewRequest` before suspension, keep its
  selector/generation checks, cancel only obsolete read/debounce work, and preserve the
  separate non-cancelling write lane. A stale transport failure must not overwrite a
  newer preview or a write. Row-click and Return submissions must not wait for a
  preview.
- A `stale_pomodoro` write currently enters an editable `.failed` state and starts a
  reload without marking its lifetime. Invalidate old choices immediately and prohibit
  submitting them until fresh rows arrive. Preserve the typed query; distinguish
  reloading from the initial-loading automatic fast path, so Return during reload cannot
  accidentally choose an automatic destination. Permit cancellation of a read-only
  reload. Drop out-of-order reload responses.
- Track enough request ownership that closing/reopening a card cannot allow old work to
  touch the new card, and a hidden-panel write still finishes and notifies exactly once.
  Keep the write guard until that write settles.

Use deterministic controlled continuations or the existing fake-fetcher seams in model
tests. Exercise a delayed initial listing failure after immediate submit, old preview
errors after submit, rapid A-to-B-to-A edits, stale reload followed by edits/Return,
cancellation/reopen, and hidden-panel completion. Verify exactly one write and the
expected selectors without wall-clock-dependent test races.

## 4. Refresh task details after success and restore failed-card interaction

`finishTodayLink` currently calls only
`library.refreshTodayAndWait(reason: .captureSuccess)` and never discards inspector
cache. The pill can update while the task's mark/status/dependencies in the library and
inspector remain stale. After a successful write (including a guard that changes
prerequisites), explicitly refresh the reference snapshot as well as Today and
invalidate the selected reference's inspector cache using
`inspectorLoader.discard(id:)`. Use a dedicated Today-link refresh reason if that fits
the existing reason enum. Coalesce with existing refresh machinery, preserve the
selected row and stay-open toast, and do not rely on a watcher to refresh a root task
note. Keep the optimistic pill until authoritative post-write Today data arrives; a
failed refresh must not erase a valid overlay or report success for a newer write on the
same reference. A guarded outcome must not invent a pill.

Add model tests with changed task data in the returned snapshot and a primed inspector
cache. Verify fresh task details, fresh Today data, no stale cache reuse, guarded
results, failed refresh, and repeated links. Use both snapshot and Today generations
where needed to distinguish old completions.

On a genuine write failure, advance the focus generation or otherwise explicitly refocus
the now-enabled card field. The existing field wrapper only requests focus when its
focus sequence changes, and the current failure path never changes it.
`todayLinkRowTapped` also accepts only `.ready`; allow a valid row click to clear a
recoverable `.failed` state and submit, while preserving running/refused guards. Add the
documented beep for an unselectable ready-state Return/Command-N (e.g. multiple running
Pomodoros with no choice), using a testable feedback seam. IME, loading-with-query, and
running-state ignored keys should remain ignored.

## 5. Complete focused coverage and the requested render matrix

The existing Mac model test covers only one Today happy path. Extend it to the cases
above and the original open-card exclusivity, synthetic-tab guard, Esc, running guard,
hidden success/failure notifications, inline failure, and old-bob refusal. Keep portable
key-map coverage including Ctrl-N navigation, Command-N with Caps Lock and IME, and
exact-name Shift-Return.

`RefsTodayLinkCardDesignTests.swift` currently renders only ready and linked. Use its
existing RenderFixtureWriter to render light/dark loading, ready, typed create, multiple
running, no open entries, guard preview, refused, running, and failed cards, plus
linked/moved/already/guarded toasts. Use fixed time for running progress so fixtures are
reproducible. Verify narrow/wide card placement and failure focus with the existing
hosted-view test seams where appropriate.

The original CLI phase attached seven real envelopes, omitting the requested
`no_reading_task`, `stale_pomodoro`, and `reading_task_closed` fixtures. Produce these
three from the repaired CLI against scratch vaults, scrub paths, add them alongside
`Tests/Fixtures/refs-today-*.json`, update the provenance README, and use them in
refusal/reload coverage. Never touch the real vault.

## 6. Verify and reconcile any new drift

Run `just check` in bob-cli (prefer the named `sase tool run check` wrapper). Also
exercise the specific ref-today and capture-ledger regressions if the gate cannot reach
them due to a confirmed unrelated failure. **Do not run `just check-full`.** A green
check with a red check-full is infrastructure work, not unfinished epic scope. Handle
long verification via `/sase_monitor`; retain this tale's remaining steps in the monitor
continuation.

Follow bob-mac-capture AGENTS.md: run `just portable-check` on a Swift host (athena if
unavailable locally); compile a scratch copy on the Mac with `Scripts/bundle.sh` and
`just typecheck-tests`, excluding `.git` and `.build` from rsync and removing the
scratch copy afterward. Never use Bryan's checkout or `just install` there. Run the
focused model/router/design tests where a real XCTest environment is available. The CLT
Mac only typechecks tests using the stub; do not claim it executed them. If the Mac is
unreachable, explicitly report that the changed macOS targets are unverified, per its
AGENTS.md. Never direct another agent to skip compiling Swift.

Check formatting without whole-file indentation churn (the missing formatter
configuration is separately tracked by `bob-cli-8k`). Run `git diff --check` in modified
repos. Do not chase already tracked unrelated Mac failures `bob-cli-61`, `bob-cli-67`,
`bob-cli-7h`, `bob-cli-7p`, `bob-cli-8j` or template failures. Attribute any newly
observed unrelated failure through `/sase_new_task`; epic-caused failures stay in this
tale and must be repaired. Rerun appropriate checks after fixes and review new base
commits before closeout.

## 7. Finish the epic landing in this turn

The lander already completed `/sase_new_task` triage and recorded these outcomes on
`bob-cli-8f`; include them in the final close note:

- `bob-cli-8f.1` note #3 proposed completion-kind and canonical-command coverage
  repairs. Both were implemented by subsequent `f2ced0c5` under `bob-cli-8b`: three
  missing kinds and five missing enum entries. Declined new tasks as fixed.
- `bob-cli-8f.1` note #4 proposed the missing `titlesec.sty` test dependency. It belongs
  to active `bob-cli-87` / phase `.4` and was already in that epic's note #1. The lander
  appended corroboration naming the proposing bead and the current package
  lookup/test-gating evidence. No duplicate task was created.
- No proposals came from `.2` or `.3`. The defects repaired by this tale are epic work,
  not deferred follow-ups. Card hints were found in RefsFooter, so their supposed
  absence was declined as a defect.

Re-read the epic and descendants, verify every child remains closed and every land-audit
issue above is resolved, and record actual verification results and remaining unrelated
baseline limitations honestly. The selected stay-open decision must still hold.

Run `sase bead epic-symbols bob-cli-8f`. Resolve every listed entry by wiring,
privatizing, a justified non-test pragma, or deleting per Symvision policy; only re-key
it when a still-open later bead actually needs the exemption. The audit found no
entries, but check again. Then close normally:

```bash
sase bead close bob-cli-8f --note "<scope and child-note verification; repaired defects; post-start integration review; actual checks; every follow-up outcome above>"
```

If rejected for symbols, finish cleanup and retry. If rejected for incomplete named
phases/descendants, finish or reopen those beads and resolve the work. Never use
`--force` to make a successful landing advance. Only deliberately canceled/superseded
work can use `--force --reason ... --resolution canceled|superseded`; it is not expected
here.

After closing, run `just symvision` where available (both audited justfiles lack that
recipe; detect availability and record this rather than inventing a pass). Open the
plans repo via `/sase_repo` and set `status: done` in its
`202610/refs_link_to_today.md`, the original epic's PLAN file. Use
`sase artifact path plan:202610/refs_link_to_today.md` to resolve that exact path if
necessary. Do not merely mark this repair tale done.

Finally run `sase bead read bob-cli-8f -r "Need the parent link"`. No parent was present
at audit time, so normal completion is expected. If one has since been added, follow the
original landing rule: verify and normally close only a parent phase, leaving its epic
to its waiting lander; for a parent plan, audit its prior landing note, all
descendants/notes, linked plan and post-child drift, rerun readiness checks, retire
symbols, close normally, run available symvision, mark its plan done, and repeat through
complete directly parented plans. Stop and note any incomplete or ambiguous ancestor
without forcing it closed.

Use `/sase_final` for the normal completion declaration for every changed repo. Report
the epic closure, verification limits, and that the Mac's installed bob must include
`ref today` for Command-N to work. Do not wait for this tale's own post-turn commit or
CI to perform any of these closeout actions.
