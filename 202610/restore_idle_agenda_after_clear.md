---
tier: tale
title: Restore the idle Pomodoro agenda after clearing capture input
goal: Clearing capture input immediately restores the cached agenda and prevents obsolete
  preview responses from hiding it again.
size: medium
proposed_by: bbugyi200.apollo.6g.w1.w0
status: done
---

# Restore the idle Pomodoro agenda when capture input is cleared

## Outcome and scope

When the user deletes the capture draft, including leaving only whitespace or newlines,
Bob Mac Capture must immediately show its cached idle agenda again: the current Pomodoro
and all future Pomodoros. No previous preview summary, action label, pending-list
notice, or draft error may keep the agenda hidden or return after an asynchronous
operation finishes. Background agenda revalidation continues through the existing store.

This is one bounded Mac-app state-lifecycle fix and its regression coverage, implemented
by one coding agent. Medium size accounts for the two asynchronous preview paths and the
macOS model tests. No CLI, JSON schema, capture grammar, agenda layout, or memory
changes are needed.

## Repository and required context

Work in `bob-mac-capture`, opened from the current host checkout with
`sase repo open bob-mac-capture -r "Fix idle agenda restoration after clearing capture input"`.
Read any instructions reported by that command. During planning the configured linked
checkout was unavailable on this host; opening
`sase repo open gh:bobs-org/bob-mac-capture -r "Inspect idle agenda clear-input regression; linked primary checkout is absent"`
succeeded. If the same condition occurs, use that supported fallback and only the path
printed by SASE. Do not hardcode another agent's checkout path.

Planning inspected app revision `26aac7d`. Recheck the relevant methods on the
implementation checkout before applying the fix, preserving any newer agenda features.
The supplied screenshot, `~/tmp/screenshots/20261010_125548.png`, shows an empty editor
with its placeholder, a leftover `Preview → ...` summary of a Pomodoro start, a Ready
status, and a disabled Start button, with no agenda.

Read the governing decisions through `/sase_memory_read`:

- `decisions:idle-capture-shows-ledger-agenda`: cached content returns immediately on an
  empty draft, refresh happens in the background, bob supplies all facts, and the fixed
  eye line and folding behavior stay intact.
- `decisions:mac-capture-is-a-thin-client`: Swift owns presentation and process
  orchestration; it must not read or parse the vault.

Relevant app files:

- `Sources/BobMacCapture/CapturePanelModel.swift`
- `Sources/BobMacCapture/CapturePanelView.swift`
- `Sources/BobMacCapture/CaptureAgendaStore.swift`
- `Sources/CaptureCore/CaptureAgendaVisibility.swift`
- `Tests/BobMacCaptureTests/CaptureAgendaModelTests.swift`
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`
- `Tests/Fixtures/fake-bob`, existing agenda and capture fixtures
- `README.md`, Idle agenda section; `justfile`; `.github/workflows/ci.yml`

## Diagnosis

The production text binding calls `editorTextDidChange` when the draft characters
change. Its blank branch sets `previewState = .idle`, clears a subset of parse and
completion state, invalidates analysis/rewrite work, and calls `noteAgendaDraftCleared`.
It does **not** clear `previewResult`, `previewResults`, `previewGlobalDestination`, or
previous errors. It also omits picker cleanup and pending close/start-list state.

`destinationSummary` uses `previewResults` independently of `previewState`.
`auxiliaryOwnedByOther` treats that summary (or an error or picker chip) as an owner, so
`agendaVisible` remains false even though the editor is blank and its agenda plan is
ready. `primaryActionTitle` also derives from those cached preview results, explaining
the screenshot's leftover Start label.

`resetAnalysisState` already performs most of the required cleanup for Discard, stash
restore, and empty-panel presentation. The blank-edit path duplicates only part of it.
The shared reset itself does not currently reset `closePendingText` or
`closePendingAction`.

Live analysis and live-preview responses are protected by `analysisGeneration`; rewrite
responses have a separate generation. Preserve those invalidations. Explicit `preview()`
instead publishes through `activeRequestID`, shared with submit, and checks no analysis
generation. The editor remains editable during explicit Preview. A response or failure
from that request can therefore restore obsolete preview/error state after a user clears
the text unless that preview request is retired. This is a code-path finding; the macOS
reproduction and tests have not run on the planning host.

## Implementation

1. **Add a regression that exercises the actual blank-edit transition.** Extend the
   existing agenda model test harness with a capture process client and zero debounce.
   Use its real agenda store, existing fake-bob fixtures, and the pinned fixture date
   `2026-08-28`. Wait for a real snapshot, not merely a non-nil loading plan. Produce a
   settled session-start preview through `editorTextDidChange`, verify the preview
   summary and Start action, then set the draft to empty and call the same edit callback
   at cursor offset zero. Assert immediately, before awaiting refresh, that the cached
   agenda is visible and undimmed, the preview state is idle, preview data and
   destination summary are empty, errors are absent, and the action title is Capture.
   The current implementation should fail these assertions.

2. **Make blank edits use one complete reset of draft analysis.** Reuse
   `resetAnalysisState` in the blank branch, extending it only as needed to clear
   pending-list text/action along with its existing preview, error, completion, picker,
   prompt, rewrite, and close-comma state. Retain the blank branch's priority-roll reset
   and call `noteAgendaDraftCleared` after cleanup so the hold releases and the normal
   store refresh follows. Keep the operation synchronous on the main actor: rendering
   cached content must not wait for a subprocess, a timer, or a fresh snapshot
   publication.

   Preserve the actual text, selection, and native undo history; whitespace counts as
   blank without rewriting it. Keep normal nonempty analysis and completion-acceptance
   suppression behavior. Clearing obsolete picker state must also leave the editor
   unlocked when no prompt owns it. Preserve the agenda cache, enabled setting, planner,
   budget, and current stale/day guards. Keep legitimate stash presentation separate
   from draft analysis cleanup.

3. **Retire obsolete explicit previews without affecting real submissions.** On
   clearing/resetting a draft, invalidate the explicit preview's request identity and
   release `isPreviewing` if an explicit preview is in flight. Reuse the existing
   request-ID checks so both success and failure callbacks become harmless; keep any
   helper narrowly scoped to preview requests. Because submit shares `activeRequestID`,
   never indiscriminately clear it while a real capture is in flight. A canceled
   response must not clear the busy flag of a newer request. Preserve successful capture
   summaries, announcements, notifications, and panel dismissal: user blank edits and
   successful submission are different lifecycle events, and the existing
   success-summary-on-reopen behavior must continue to pass its tests.

4. **Cover the behavioral boundaries with focused model tests.** Keep assertions on
   user-visible state and request behavior; do not merely duplicate assignments from the
   reset method. Reuse fixtures and fake-bob's recording/delay support, extending that
   support only if needed to reliably hold a particular stage of an asynchronous
   request.

   | Starting condition                                                             | After clearing                                                                                  |
   | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
   | Settled start preview; empty or whitespace/newline-only text                   | Immediate cached agenda; no Preview summary or Start label                                      |
   | Failed preview carrying an error/code                                          | Error owner disappears and agenda returns                                                       |
   | Pending close/start-list preview                                               | Pending notice clears and the next valid draft can submit normally                              |
   | Dismissed picker chip tied to old text                                         | Chip/suppression no longer blocks idle content                                                  |
   | Live analysis/preview or rewrite still running                                 | Late results cannot refill draft UI or hide agenda                                              |
   | Explicit Preview still running, with success or failure                        | Request retires; late result cannot refill preview/error state or interfere with the next draft |
   | Agenda refresh returns identical bytes or fails with a current cached snapshot | Agenda remains visible; existing stale behavior applies                                         |
   | Agenda disabled or bob lacks agenda support                                    | Cleanup still happens; agenda remains hidden according to existing policy                       |

   For late-result tests, ensure the relevant request really started before clearing and
   observe completion/termination or a controlled release before checking the final
   state. Avoid tests that only cancel the debounce before the old request exists.
   Include clear-then-type-again coverage. Exercise Discard/reopen and retain/reopen
   through existing tests to catch shared-reset regressions. Verify clearing never
   invokes a mutating capture and an empty Return remains inert.

5. **Document and review the final behavior.** The README already promises immediate
   restoration on clear. Keep that contract; add at most a concise clarification about
   retiring old draft previews if useful. Review the final diff for unrelated
   layout/contract changes and confirm the fix belongs to the app model rather than a
   weakened agenda visibility predicate or a forced store reload.

## Validation and acceptance

Run the new regression on the unfixed implementation when a macOS runner is available,
then rerun it with the fix. On macOS 26 with the repository's Xcode toolchain wrapper,
run:

```sh
./Scripts/xcode-swift.sh test --filter CaptureAgendaModelTests
./Scripts/xcode-swift.sh test --filter CapturePanelModelTests
just format-lint
just build
just test
```

The full suite includes agenda store/hold, picker, preview, submit/stash, and eye-line
coverage. Use the repository's macOS CI workflow for validation when the implementation
host cannot run AppKit tests. Linux CaptureCore-only tests are not proof of this fix;
the planning host has no `swift` executable. Use `/sase_monitor` for any long CI or test
wait required by the SASE workflow, and report unavailable Mac verification honestly.

On a Mac, open the panel with a populated Now/Next/Later agenda, type a valid
session-start draft and wait for preview, then select all and delete. Repeat by
backspacing the last character, leaving whitespace, clearing a failed draft, and
clearing while explicit Preview runs. Each time the cached agenda must return without
closing/reopening the panel, the input line must stay at its existing eye line with
focus retained, and late responses must not replace it. Type another draft and verify
preview/action behavior resumes normally. Also check that disabling Agenda still
suppresses it and successful capture still dismisses normally. Use test fixtures or a
disposable vault for any write-path smoke test.

Acceptance requires the immediate restoration regression and asynchronous clear
regressions to pass, no stale preview/error/action on a blank editor, existing
submission and agenda visibility contracts preserved, and successful macOS
build/test/format validation or an explicit outstanding verification status if that
environment is unavailable.
