---
tier: tale
title: Finish the Mac meeting notes landing
goal:
  Repair and verify the remaining Mac meeting notes flows, then close bob-cli-pw.8.1 and
  its parent phase bob-cli-pw.8.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-pw.8.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pw.8.1
create_time: 2026-11-25 04:34:28
status: wip
---

- **PARENT:**
  [202611/mac_meeting_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_meeting_notes.md)
- **BEAD:**
  [bob-cli-pw.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/bob-cli-pw.8.1.md)

# Finish the Mac meeting notes landing

Finish the remaining native integration defects in **bob-cli-pw.8.1**, verify the
combined feature, and close that epic and its parent phase **bob-cli-pw.8**. This is one
bounded repair to the existing meeting-notes session, native wrapper, and views; one
coding agent can implement it directly. Do not rebuild the backend or create another
phased epic. This tale has no land agent: its coder owns the final closeout.

## Context and constraints

Read `bob-cli-pw.8.1` and its three children with `sase bead read`, including the
lander's `LANDING AUDIT` and `FOLLOW-UP TRIAGE COMPLETE` notes. Read
`plan:202611/mac_meeting_notes.md` and the original Mac phase in
`plan:202611/meeting_notes_event_refs.md` through `sase artifact read`. Open
`bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`. Swift paths below are
relative to that returned repository. The original epic plan to mark done at closeout is
**plan:202611/mac_meeting_notes.md**, repository-relative path
**202611/mac_meeting_notes.md** in the plans repository opened through `/sase_repo`.
Never substitute this repair tale's file for that original epic plan.

Keep the accepted decisions: `scaffolding=rehang`, `recap_scope=full`, both memory
decisions `no`. This work neither performs first-use vault operations nor edits memory.
Bob remains the sole parser, resolver, hash authority, and vault writer. Use synthetic
meetings only, preserve the capture draft, and preserve Event Note mode as a distinct
feature. No new CLI options, global hotkeys, source adapters, or AI enrichment. Read the
thin-client and escape-alias decisions through `/sase_memory_read` before changing their
governed behavior.

Audit baseline: bob-cli `bcf0566b`, Mac `284417d7`, plugins `c1819664`. Every Mac commit
from `c8692ed3` through `284417d7` belongs to this epic; bob-cli's only commit since
this epic began is its guide link `bcf0566b`. Both implementation checkouts matched
fetched `origin/master`; there was no unrelated later implementation to merge at audit
time. Repeat that drift review before final verification, integrating any newly landed
conflicting behavior. Keep the already-correct JSON transport, event/hash/options
replay, exit-2 choice decoding, configured status defaults, single-flight writes, and
Meetings kind/Command-6 scope.

The previous macOS run
https://github.com/bobs-org/bob-mac-capture/actions/runs/45580348007 passed every
required job step. Its snapshot `3f4fcf361976aac2f1c69c168564bc94f40697b4` and Mac HEAD
`284417d7` have identical tree `4263a2ce0ef40a450eefa9e67ae2f6b613ee520b`. That is valid
baseline evidence, not verification of these repairs. Earlier memberwise-init and
status-menu assertion failures were fixed; the old Refs query assertion was fixed by
`36b3e4aa`. Do not duplicate those changes.

## 1. Make the visible chooser and keyboard state agree

Repair `Sources/CaptureCore/MeetingNotesSession.swift`,
`Sources/BobMacCapture/MeetingNotesPanelModel.swift`, `MeetingNotesPanelView.swift`, and
`CaptureKeyCommandRouter.swift` together:

- The preview's **Change meeting** currently calls `retryList()`, which never
  transitions to `.choosingMeeting`. Add an explicit chooser transition and wire the
  action to it. Show the current recent list while loading fresh results. Leaving the
  chooser returns one level to a valid preview or an intentional exit, without filing or
  retaining approval for a changed selection. Late read results must not replace newer
  choices.
- `ForEach(model.pickerMeetings, id: \.path)` collides for multiple meetings in the same
  daily note. Use Bob's event reference for selectable-row identity, with a distinct
  stable presentation identity for tokenless rows; never invent an EVENT. Make tokenless
  rows visibly disabled and skip them for keyboard selection. Filtering must keep
  selection valid and visible. Test two meetings with the same path and different event
  refs, plus an ID-less row.
- The router currently sends every non-fileable Return to `openSelectedMeeting()`,
  including parent and replacement screens. Arrow keys also always move meeting
  selection. Route selection/activation by the visible state: meeting choice, parent
  choice, explicit replacement review, preview, error, and receipt. Parent choices must
  be usable from the keyboard. Return on a choice only changes the selection/options and
  fetches a new preview; a later deliberate Return files that preview. A repeated Return
  never starts a second write or implicitly approves replacement.
- `handleEscape()` currently re-previews the same unresolved parent/ambiguity or
  already-filed refusal, trapping the user in that state. Implement an actual one-level
  back/close transition. Esc, q outside this sheet's text controls, and Ctrl+[ must use
  that same action. Honor IME composition and keep Ctrl+[ from inserting text.
  Text-control detection must refer to the sheet, not the hidden capture editor.
- The already-filed screen must offer **Open existing** using the existing Obsidian
  navigation helper, alongside explicit replacement review and keeping the existing
  note. Only explicit replacement can set overwrite, and it still requires a fresh
  guarded preview before filing.

Use the existing portable session boundaries where possible. Expose only the minimal
native test injection needed to exercise real model/controller paths.

## 2. Preserve write ownership, visibility, draft, and refresh

The portable session deliberately defers `leave()` during `.writing` and retains a
terminal receipt. The native model currently sets `isPresented=false` first, and the
controller can then open another mode. Its `setProcessClient` also replaces the old
session after calling this deferred leave. Align those layers:

- Keep the active writing sheet and session owned until settlement, and keep its
  receipt/error observable. Back, Escape aliases, menu entry, Capture hide/toggle, panel
  switching, and Event Note entry must not hide or replace an active write. Defer
  process-client reconfiguration until settlement or explicitly retain and observe the
  original session; never cancel or replay that write.
- Keep mode changes mutually exclusive. Block meeting entry while a capture submit or
  Event Note mutation is active, including menu and agenda paths, not just the Command-M
  router. Re-entry while writing must not reset or duplicate the operation.
- Save and restore the original capture responder/caret. While the sheet is visible,
  ordinary keys passed through by its router must not mutate the covered capture editor.
  Add native tests with a nonempty draft and actual responder transitions.
- AppDelegate currently installs two callbacks that each refresh the agenda and neither
  explicitly invalidates the reference library. Use one receipt handler that refreshes
  the agenda and invalidates/refreshes Refs through its existing lifecycle, once per
  filed or queued receipt. Do not depend on filesystem watcher timing. Queued remains a
  distinct receipt with no Open action for an uncreated note; filing errors and
  clipboard mismatch do not falsely announce a receipt.

Test leave during a delayed write, successful and failed settlement, reconfiguration,
and repeated entry. Session/model ownership and both refresh destinations must be
observable in the tests. Give every async test gate a real timeout, including the
current `MeetingNotesResponseGate` continuation in the portable tests.

## 3. Correct agenda eligibility and capability gating

`CaptureAgendaEventRow.offersMeetingNotesAction` currently rejects
`statusSymbol == "x"`, although Bob's shipped task settings define x/X as DONE and `-`
as CANCELLED. The current cancellation test combines x with CANCELLED and misses this
regression. Use Bob's returned semantics: current and ended started tasks, including
x/DONE, offer Notes when the exact event token and explicit missing-notes state are
present. Cancelled, future, filed, unknown-state, malformed/missing-state, and ID-less
rows must not offer it. Decode an absent `meeting_notes.state` conservatively instead of
silently making it `missing`. Check the actual backend contract in
`src/native/meetings/agenda.rs` and `src/native/capture_pomodoros_agenda.rs`; do not add
client-side time or token inference. Verify the real Earlier expansion carries the
action through presentation and view.

Capability hiding currently affects only the status-menu item. Thread the known
unsupported state to the agenda action and Command-M/controller entry, and restore
probing after Bob is rechecked or its executable identity changes. The initial
unsupported attempt should show the update message and leave ordinary Capture usable.
Preserve normal behavior when the capability is merely unknown and when valid JSON
exit-2 choice reports are received.

## 4. Render the actual preview and receipt fields

`MeetingNotesPresentation.transcriptDisposition` currently reads `notes.shape`, so it
displays the input format as the transcript disposition. Read `notes.transcript` and
cover kept/omitted/none and absent fields with fixtures. Preserve the returned outline,
counts, destination, effective status, match evidence, and warnings. Include receipt
commit warnings when present; do not claim queued files already exist. Keep new
user-facing text at the product level.

Add synthetic visual coverage for preview, chooser, replacement/parent choice, and
filed/queued receipts at small and normal panel sizes. Inspect the resulting images, not
only their creation status. Use the repository's existing native rendering harness and
`/sase_artifact` for durable synthetic evidence when useful.

## 5. Fix this epic's documentation regressions

The phase notes' baseline measurements were wrong. Using bob-cli's canonical
`bob_docs.markdown.count_words` on `git show 7d369ae8:<path>` gives:

| Mac source         | Before this epic | Current audit tree | Reference cap |
| ------------------ | ---------------: | -----------------: | ------------: |
| `docs/protocol.md` |             5997 |               6308 |          6000 |
| `docs/keyboard.md` |             5978 |               6019 |          6000 |

Both failures are epic-owned. Read docs memory, then tighten the new protocol/key prose
and remove duplication with the existing `docs/meeting-notes.md` so both pages are below
cap. Keep published paths, anchors, and exact command/key contracts. Update
meeting-notes documentation to match the repaired chooser, write-close policy,
existing-note action, and capability behavior. Do not raise budgets or suppress lint.

The lander already finished all step-3 follow-up triage; outcomes are in the epic's
`FOLLOW-UP TRIAGE COMPLETE` note. All three phase proposals were consolidated: protocol
-> existing **bob-cli-h9**, keyboard/navigation -> **bob-cli-jz**, plugin API headroom
-> **bob-cli-ob**, each corroborated with independent measurements. The Mac regressions
still belong to this tale. Plugin sources were untouched by this epic: at plugins
`c1819664`, navigation is 6029/6000 and plugin-apis is 9287/8000. The latter is an
advisory develop-page warning. No new duplicate tasks or separate plugin-API CI failure
should be created. Existing unrelated plugin headroom work is outside this tale. Record
any changed disposition in the epic before closure.

## 6. Verify before closing, without a dependency on the final commit

Add meaningful regressions through the native model/controller/view paths; the current
new app tests cover only command enum routing and menu ordering. Exercise all three
entry points, same-day chooser rows, parent/replacement keyboard actions, escape
layering, draft/focus preservation, write settlement, capability transitions, one-time
agenda/Refs refresh, and ended DONE agenda rows. Retain and run existing
contract/session and Meetings search/scope tests.

Follow the Mac repo's compilation rules. Run `just portable-check` where Swift is
available (on an isolated athena scratch copy through the tailnet workflow if needed).
For app-target edits attempt a separate Mac scratch `Scripts/bundle.sh` and
`just typecheck-tests`, excluding `.git`/`.build`, then clean it up. Never install into
or use Bryan's live checkout. Record inability to reach the Mac if it occurs. Run the
complete `Scripts/macos-ci-check` precommit snapshot gate through `/sase_monitor`; its
exact snapshot must pass the macOS build, tests, bundle, signature, launch, and install
steps before closure. This can verify uncommitted final code. Do not create a
post-commit SHA/push/CI dependency before the closeout below.

Run **`just check`** in bob-cli for file-change verification and the mandatory
`just docs-check` gates for changed docs in their owning repos, using the correct linked
mounts. Never run `just check-full`. Compare any failures with a clean base: the known
unrelated plugin budget failure/advisory above is already triaged, but the two Mac
budget errors must be gone and any new failure caused here must be fixed. If
`just check` passes and some other actor reports a `just check-full` failure, that is
test-infrastructure work, not unfinished epic scope.

Use `/sase_monitor` for long commands, wait for its staging command to exit, and end
those turns through `/sase_final`. Resume to consume the completed results. Record
tested trees, commands, CI URL, visual inspection, base-only exceptions, and remaining
drift review in the epic note. Never claim newly edited Swift verified by the old run.

## 7. Finish this epic and its parent phase in the coding turn

After repairs and verification, reread `bob-cli-pw.8.1` and every child/note and confirm
the linked original plan is complete. If this tale itself has been materialized as an
unfinished descendant, finish its verified work and close that assigned tale normally
first so it does not block its own epic. Do not force-close anything, and do not wait
for this work's final host commit, push, or post-commit CI.

1. Run `sase bead epic-symbols bob-cli-pw.8.1`. Resolve every listed symbol by wiring,
   privatizing, adding a justified non-test pragma, or deleting it under the Symvision
   policy. Re-key a Justfile exemption only to a genuinely still-open later bead that
   needs it. There were no entries at this audit, but recheck now.
2. Close normally:
   `sase bead close bob-cli-pw.8.1 --note "<original implementation audit, repaired native flows, integration review, tested tree and green macOS CI, docs and check results, and follow-up outcomes h9/jz/ob>"`.
   If rejected, resolve the named leftovers and retry normally. Never use `--force`
   merely to advance a successful nested landing.
3. Run `just symvision` where the recipe is available and confirm the whitelist is
   clean; if no recipe exists, record that explicitly. Open the plans repo with
   `/sase_repo`, resolve the original `plan:202611/mac_meeting_notes.md`, and set only
   its frontmatter `status: done` after the close.
4. Read `sase bead read bob-cli-pw.8.1 -r "Need the parent link"`, then
   `sase bead read bob-cli-pw.8 -r "Verify completed child fulfills the parent Mac phase"`.
   The parent is the phase **bob-cli-pw.8**, not a plan. Recheck its mac-surfaces
   contract and `sase bead epic-symbols bob-cli-pw.8`, resolve any entries, and close
   only that phase normally:
   `sase bead close bob-cli-pw.8 --note "<verified combined Meeting Notes and Meetings scope, guarded filing, agenda actions, draft/focus behavior, and green macOS snapshot evidence>"`.
   Leave the containing epic **bob-cli-pw** open for its waiting land agent; do not
   change its plan status. If the phase is unexpectedly incomplete, record the concrete
   blocker there and report it instead of forcing the close.
5. Finish through `/sase_final`, declaring all changed repositories for the host to
   commit and choosing bead actions from verified scope completeness. Report the
   epic/parent phase outcomes and any explicitly accepted unrelated gate failures.
