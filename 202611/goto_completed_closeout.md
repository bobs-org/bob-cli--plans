---
tier: tale
title: Finish Go To completed-task state and close bob-cli-h3
goal:
  Keep Go To scope and task snapshots consistent across concurrent fetches, errors, and
  chip reopen, then verify and close bob-cli-h3.
size: medium
proposed_by: bbugyi200.athena.bob-cli-h3.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-h3
status: done
---

- **PARENT:**
  [202611/goto_completed_toggle.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/goto_completed_toggle.md)
- **BEAD:**
  [bob-cli-h3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h3/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-h3.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-h3.land.md)
- **COMMITS:**
  - [70c0839](https://github.com/bobs-org/bob-mac-capture/commit/70c0839736427bfc664a70ab83db4810a9351654)
    — fix(capture): synchronize Go To scope snapshots

# Finish the Go To completed-task toggle and close bob-cli-h3

Repair the asynchronous scope/snapshot mismatch in Bob Mac Capture, verify the
completed-task feature against current master, and close epic `bob-cli-h3` in this same
coding turn. This tale has no land agent after it. Its coder owns the entire closeout
below, before host finalization commits the code.

## Scope and verified baseline

Read `sase bead read bob-cli-h3 -r "Need landing findings and final decisions"` and
`sase artifact read plan:202611/goto_completed_toggle.md "Need the original feature contract"`.
Both phase beads (`bob-cli-h3.1`, `bob-cli-h3.2`) are closed. The lander reviewed every
phase note, the Rust implementation in `f54a352c`, the Mac implementation in `78df1c9`,
and the original plan. Keep the final decisions unchanged:

- Completed rows come from the whole vault, including archives, reference notes, and
  daily notes; canceled tasks are included.
- Each fresh `/` draft starts open-only. Escape to a chip and reopen preserves that
  session's choice.
- Bob supplies all task facts through `capture-complete -C/--closed-tasks`; Mac Capture
  remains a thin client. No grammar, status, vault scan, or vault write is implemented
  in Swift.

Rust's vault-wide catalog, newest-close-date sorting, open-first query results, optional
`closed`/`note_kind` fields, and unchanged unflagged candidates are already implemented
and tested. The Mac process-client flag, unsupported-option mapping, Command-X routing,
Completed group, `note_path|ref` identity, closed dates, hints, and archive navigation
are already implemented. Preserve them.

Current review baseline: bob-cli `cf2e3c9f` and bob-mac-capture `78df1c9`, both equal
their respective `origin/master` at review time. Reviewed intervening changes:
`a6c55cbb` fixes the glossary frontmatter, `ce8f53b1` advances grammar to 12, and
`cf2e3c9f` adds move-in handling in the capture guard. All coexist with Go To.
`bbb5856f`, which landed after epic creation but before its first code commit, creates
done daily calendar rows with `[completion:: ...]`; Go To already reads these through
its shared vault catalog. No additional integration code was needed. Recheck drift from
these baselines before closing.

Use `/sase_repo` to open `bob-mac-capture`, read its `AGENTS.md`, and use only the
returned checkout path. The changes below belong in that linked repository. There are no
authorized memory edits in this repair.

## 1. Make Go To scope and snapshot settle together

The defects are in `Sources/BobMacCapture/CapturePanelModel.swift`:

- `toggleGotoClosedTasks` writes `picker.includesClosedTasks` before the request
  succeeds. `cancelPicker` invalidates the generation, but saves that new bit with the
  old candidates into `CapturePickerChipState`. For a bare `/`, `openPickerFromChip`
  sets `needsRefetch` false and reuses those candidates. Press Command-X and immediately
  Escape/reopen during a delayed fetch: the capsule says completed is on but all rows
  are open. The inverse retains completed rows while saying off. Waiting for the
  canceled request cannot repair it.
- A failed latest request restores `!includesClosedTasks`, which is the previous desired
  state, not necessarily the last applied snapshot's scope. Starting with open rows,
  press on then off before either fetch settles and fail the off fetch: the first result
  is discarded, but the error branch flips on over open-only rows. The inverse sequence
  can leave completed rows while off.
- The chip refetch path uses `try?` and draft/range checks rather than the Go To request
  generation. Integrate that path into the same scope/error/session rules, so an older
  chip fetch cannot replace a later picker or toggle result.

Keep one explicit distinction between the last successfully applied scope and the latest
desired scope. Implement the small transition state as a production-used, portable value
in CaptureCore, with focused tests there; keep process calls and AppKit presentation in
the panel. The state must handle these rules:

1. Each Command-X flips the latest desired choice, even while another request is
   pending. A fresh request gets a generation/session identity. Repeated keys remain
   consumed by the existing router.
2. Until success, retain the last usable candidate snapshot and its applied scope. Keep
   pending intent separate, so success, errors, cancellation, and chip reuse never infer
   snapshot scope from the inverse of a requested boolean. Keep the displayed settled
   capsule and footer consistent with the applied snapshot; any pending indication
   should be brief and use the existing status surface.
3. Apply only the latest response for the same Go To session, unchanged draft and
   replacement range, and `context == "task_goto"`. Apply candidates, scope, selection,
   and row budget together. Preserve the current filter and selected row if its key
   survives; otherwise choose the first visible row. Preserve the existing
   showing/hiding accessibility announcements after successful application.
4. Escape records both the desired choice and enough snapshot provenance to know whether
   cached candidates fulfill it. Invalidate the departing request. If the desired choice
   has not been fetched, reopening must fetch it even for a bare `/`. A settled
   bare-slash chip should retain its existing no-refetch behavior. Both toggle-on and
   toggle-off must work across dismissal while pending.
5. A latest fetch error restores the last applied scope and candidates and presents the
   error. An unsupported `--closed-tasks` response leaves open-only behavior with
   `Update bob to show completed tasks`, using the existing warning/status surfaces.
   Apply equivalent handling to a required chip refetch; do not silently swallow its
   failure. Stale successes and stale errors have no UI effects.
6. Reset requested/applied state when the Go To session is discarded, accepted, or
   replaced. A genuinely new `/` session starts off even after the previous one
   successfully showed completed rows. Existing other picker behavior stays intact.

Relevant files: `CapturePanelModel.swift`, `CapturePickerState.swift`, and the
production state value and tests in CaptureCore. Update the header/hint wiring only as
needed to distinguish applied state from pending intent. Do not add a persisted setting
or duplicate task catalog. Read the thin-client decision using `/sase_memory_read`
before changing this flow.

## 2. Add regressions that cover the actual missing paths

Keep and extend `Tests/BobMacCaptureTests/CaptureTaskGotoPanelTests.swift`. Its existing
chip test waits for 11 candidates before escaping; its overlap test only covers a
successful last request. Add production-state tests runnable on Linux and panel
integration tests for:

- Delayed on fetch, Escape before completion, reopen bare `/`, eventually show completed
  rows with on state. Repeat from a settled on state with a delayed off fetch and ensure
  completed rows disappear.
- On/off before settlement, fail the latest request, keep the original open rows and off
  state; reverse from a settled on snapshot and preserve on state/rows.
- An older success or failure delivered after a newer result, after Escape/reopen, or
  after abandoning and recreating an identical `/` draft cannot affect the new session.
  Cover delayed chip refetch as well as toggle refetch.
- Unsupported-option failure during a pending choice carried through a chip shows the
  update hint and settles coherently. Normal fetch errors retain usable rows and report
  the failure.
- Successfully toggle on, discard the draft, then auto-open a fresh `/`: it starts off.
  Assert selection/filter preservation on success and error, and selection fallback when
  turning off removes the selected completed row.

Use bounded fixture delays/request-specific failures in `Tests/Fixtures/fake-bob` or the
repository's existing controlled request helpers. A global exit override that prevents
the initial picker from opening does not exercise these races. Assert public panel
behavior; the portable tests must exercise the state actually used by the panel, not a
parallel reproduction of its implementation. No untimed continuations or unbounded
waits.

## 3. Verify and integrate the repair

Format only changed Swift lines using the repository's `.swift-format`. Run
`just portable-check` in the linked Mac repository. This compiles/tests CaptureCore and
RefsCore; it does not run the AppKit panel tests. Best-effort compile a scratch rsync
copy on the Mac with `Scripts/bundle.sh` and `just typecheck-tests`, following that
repository's `AGENTS.md` and build documentation. Read tailnet memory before connecting.
Never build in Bryan's checkout or install the app. If the Mac is offline, state
explicitly that native macOS targets remain unverified, as the original approved plan
allows. Do not characterize portable success as panel test execution.

Run bob-cli's canonical `just check` (through `sase tool run check` when configured)
against the integrated tree. Do not run `just check-full`. If verification takes long
enough to need a handoff, use `/sase_monitor` with explicit continuation to finish this
tale and the epic closeout; never end with a live unmonitored command. Do not wait for a
commit SHA, a push, or CI for this turn's own commit before closing.

Recheck commits after the baseline in both repositories and their base branches.
Preserve grammar 12 and the move-in/transfer changes already integrated. Investigate any
new conflicts or consumers of Go To scope state. If public behavior docs need an update,
read docs memory and update the owning Mac docs, then run `just docs-check`. The
existing completed-task contract remains the desired behavior.

Both original phase follow-up proposals have already been triaged by the lander with
`/sase_new_task`; carry these outcomes into the final close note:

- `bob-cli-h3.1` note #2, missing frontmatter in glossary `task-merge`: declined as
  resolved by `a6c55cbb` (`bob-cli-go.13`). The current documentation check gets past
  memory parsing; `go` and `go.13` also record the repair. No task was created.
- `bob-cli-h3.2` note #1, generated `docs/develop/decisions.md` drift: reproduced on
  `cf2e3c9f` by `just docs-check` (lint passes; generated drift exits 1 and names only
  that page). The new source is standalone gcal commit `bbb5856f`, unrelated to h3.
  Existing task `bob-cli-b6` received `+1 --verified-after-close` with phase
  attribution. No duplicate task was created. Same-type and broad searches, the
  last-week task sweep, and all active epic scopes found no other causal owner.

The docs drift is tracked external work; it does not justify leaving the repaired Go To
epic open. If a new, genuinely unrelated problem appears, use `/sase_new_task` and
record its outcome. Any h3-caused issue remains part of this tale.

## 4. Close bob-cli-h3 in this coding turn

This is the tale's final work step, not a handoff to another lander. The code can still
be awaiting the host's commit when these commands run.

1. Re-read `bob-cli-h3` and both phase beads with audited `sase bead read` calls.
   Confirm all descendants are complete, all original notes and new landing findings are
   addressed, and the original linked plan's requirements remain satisfied. Add the
   repair/verification evidence and both follow-up outcomes to the close note. The
   original epic had no parent (`parent_id: null`, empty ancestors) at the landing
   review.
2. Run `sase bead epic-symbols bob-cli-h3`. The lander found no entries, but rerun it
   now. Resolve every entry keyed to this epic or a phase by wiring, privatizing,
   deleting, or a justified non-test pragma per policy; re-key only if a still-open
   later bead actually needs the exemption. Check both affected repositories when
   applicable. Never leave a closing bead's exemption behind.
3. Close normally with
   `sase bead close bob-cli-h3 --note "<reviewed scope, decisions, commits/drift, repaired races, verification results/limits, and both follow-up outcomes>"`.
   If outstanding symbols reject the close, fix them and retry. If new descendants or
   named phases are incomplete, finish their work and normal closure first. Never force
   a successful nested landing or force merely to make close succeed;
   canceled/superseded work requires a deliberate reason and the corresponding
   resolution, not a workaround for these required fixes.
4. After closing, run `just symvision` wherever that recipe is available. Neither
   inspected justfile currently defines it; if it is still absent, record that
   limitation and the successful empty `epic-symbols` result instead of inventing a
   pass.
5. Open the plans repository through `/sase_repo`. Read the original plan through
   `sase artifact read plan:202611/goto_completed_toggle.md "Need completed epic plan status"`
   and set `status: done` in the frontmatter of its `202611/goto_completed_toggle.md`
   file at the PLAN path returned by the current bead read. Preserve all other metadata
   and decisions. Do not hand-edit bead lifecycle fields.
6. Read `sase bead read bob-cli-h3 -r "Need the parent link"` again. It had no parent,
   so normally nothing more closes. If that link has changed, a parent phase closes
   normally only after verifying its required work; leave its containing epic to its
   waiting lander. For a direct parent plan, review its previous landing note, every
   descendant/note, linked plan readiness, and post-child drift, retire its epic
   symbols, then close normally, run available symvision, and mark its plan done; repeat
   only through fully complete direct plan ancestors. At the first incomplete or
   ambiguous parent, record a blocker note and report it.

Finish through `/sase_final` with commit decisions for all edited repositories,
including the linked Mac repository and the plans sidecar. Report the epic closure,
actual verification results, and any native macOS limitation.
