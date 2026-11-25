---
tier: epic
title: Mac meeting notes filing and Meetings library scope
goal: Complete bob-cli-pw.8 with guarded clipboard meeting-notes filing from Capture
  and its agenda, a first-class Meetings kind in Bob Refs, and verified macOS CI.
phases:
- id: meet-protocol
  title: Meeting notes JSON client and portable session
  size: medium
  depends_on: []
  description: 'meet-protocol: add the ref meet response models, guarded clipboard
    preview/file client, capability detection, portable session state machine, synthetic
    contract fixtures, and protocol documentation in bob-mac-capture.'
- id: meetings-library
  title: First-class Meetings kind in Bob Refs
  size: small
  depends_on: []
  description: 'meetings-library: add RefKind.meeting, the Meetings scope on Command-6,
    presentation and search coverage, synthetic tests, and the Bob Refs documentation
    in bob-mac-capture.'
- id: meet-panel
  title: Meeting Notes sheet and agenda entry points
  size: medium
  depends_on:
  - meet-protocol
  - meetings-library
  description: 'meet-panel: integrate the portable session into a native Meeting Notes
    mode, Command-M, the status menu, and focusable agenda Notes actions; preserve
    the capture draft, implement choice and receipt states, and finish UI tests and
    user docs.'
proposed_by: bbugyi200.apollo.bob-cli-pw.8
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pw.8
create_time: 2026-11-24 23:55:00
status: wip
bead_id: bob-cli-pw.8.1
---

- **PROMPT:** [prompts/202611/mac_meeting_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_meeting_notes.md)
- **PARENT:** [202611/meeting_notes_event_refs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/meeting_notes_event_refs.md)
- **BEAD:** [bob-cli-pw.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/bob-cli-pw.8.1.md)

# Mac meeting notes: finish bob-cli-pw.8

## Outcome and ownership

Complete assigned phase `bob-cli-pw.8` of `plan:202611/meeting_notes_event_refs.md`
through a child epic. Copying meeting notes, opening Capture, pressing Command-M, and
pressing Return should file the previewed notes through `bob ref meet`. An agenda Notes
action selects the event first. Bob Refs recognizes the result as a Meeting and offers a
Meetings filter.

The original phase has no previous-work notes. At exploration, bob-cli HEAD `e4c96b05`
already contains the public command, its preparation/ingest/overwrite routes, and the
additive agenda contract. The linked Mac repository at `7d369ae8` has Event Note mode
but no Meeting Notes session or `RefKind.meeting`. Check current history before editing
and retain any work that has landed since these anchors.

This is an epic because the typed process/session layer, native UI integration, and
reference-library presentation are separate coding units. Each phase above has a bounded
implementation and test surface. The first two have no dependency on each other; the
panel phase follows both and owns the combined keyboard docs. The child epic land agent
owns final integration evidence and completion of `bob-cli-pw.8`; individual child
workers close only their assigned beads. Nobody working this child epic is authorized to
close `bob-cli-pw` or any higher ancestor.

## Fixed constraints

- Read the assigned bead and original epic through the audited SASE commands. Its
  accepted decisions remain `scaffolding=rehang`, `recap_scope=full`, and both memory
  decisions `no`. This Mac work does not perform first-use vault operations, and edits
  no memory files.
- Open `bob-mac-capture` with `/sase_repo` and use its returned path. Read its
  `AGENTS.md`. All Mac source, fixtures, and mounted docs belong there. No manual
  repository cloning or edits in another person's checkout.
- bob owns meeting resolution, normalization, parent eligibility, file identity, SHA
  calculation, and every vault mutation. Swift renders JSON and orchestrates processes.
  Do not parse notes, infer matching scores, write vault files, or add a second
  implementation of the backend rules.
- All public code, fixtures, screenshots, docs, and commit messages use invented
  meetings and people. No private vault data or clipboard contents in artifacts or logs.
  Do not reproduce the original epic's real first-use examples.
- Keep Event Note mode's private-calendar-note behavior distinct. Preserve the existing
  capture draft and first responder when returning from Meeting Notes.
- No new CLI flags, changes to the CLI contract, AI enrichment, source adapters,
  transcript editing, or new global hotkey. Command-M is local to Capture.

## Verified implementation anchors

In bob-cli, read the actual contract before generating Swift fixtures:

- `src/native/highlights_ref/meet/model.rs` and `meet/output.rs` define the reports.
- `src/native/highlights_ref/meet_cli.rs` adds filed/queued receipts and exit codes.
- `src/native/meetings/agenda.rs` supplies `event_ref` and `meeting_notes` for started,
  noncancelled task rows with an ID.
- `docs/meeting-notes.md` and `docs/reference/json-outputs.md` document the shipped
  behavior; `tests/cli/highlights/meet_cli.rs` supplies synthetic CLI precedents.

In bob-mac-capture:

- `CaptureCore/GcalNoteSession.swift`, `GcalNoteModels.swift`, and the corresponding
  process protocol and tests are the portable lifecycle precedent.
- `CaptureCore/BobProcessClient.swift` has `decodeReport`, which decodes structured JSON
  even on nonzero exits, per-lane cancellation, and executable-identity-based
  unsupported-command caching. Ordinary `decode` rejects nonzero exits too early.
- `BobMacCapture/GcalNotePanelModel.swift`, `CapturePanelController.swift`, and
  `CapturePanelView.swift` demonstrate overlay presentation, sizing, and focus.
- `BobMacCapture/AppDelegate.swift` builds the status menu and refresh callbacks;
  `CaptureKeyCommandRouter.swift` owns local keyboard dispatch.
- `CaptureCore/CaptureAgendaModels.swift` and `CaptureAgendaPresentation.swift` carry
  calendar rows into `CaptureAgendaCalendarRowData`;
  `BobMacCapture/CaptureAgendaView.swift` renders the existing Event Note action.
- `RefsCore/RefsModel.swift`, `RefsRanking.swift`, and the caption/inspector helpers own
  kind and scope logic. App presentation is under `BobMacCapture/Refs/`.
- Command-M is currently unused in the capture router and keyboard contract. Refs scopes
  currently occupy Command-1 through Command-5; Command-6 is free.

Paths beginning with a Swift module name above are relative to `Sources/`. Recheck
bindings before integrating; preserve existing command behavior.

## Phase meet-protocol

Create focused `CaptureCore/MeetingNotes*.swift` files for models, an injectable process
protocol, and a main-actor session, following the GcalNote organization. Keep tests
portable without AppKit or a real clipboard.

### Transport and decoding

1. Add methods for list (`ref meet -l -f json`), clipboard preview
   (`ref meet [EVENT] -c -d -f json`), and filing (same selection/options, omit `-d`,
   add `--expect-sha256 HEX`). Use argv arrays, never a shell, and preserve the normal
   configured bob executable/vault environment. A supplied EVENT remains one opaque
   token. Expose parent, explicit status, and overwrite options.
2. Decode schema 1, `ok`, list meetings, preview/receipt payloads, warnings, and
   structured errors. Use the shipped wire shape: meeting `ref`, `block_id`, and `root`
   can be null; candidates contain `{meeting, detail, evidence}`; `needs_parent` uses
   **`parent_candidates`**, whose exact structure comes from `ParentChoice`; `existing`
   names note/task/status. Do not model the earlier design's shorthand `candidates` as
   the only parent-choice field.
3. Represent `would_file`, `filed`, and `queued` distinctly. Preserve the queue
   path/reason/detail, optional commit, pages, and warnings. Unknown optional fields
   remain forward compatible; a different schema version refuses explicitly.
4. Use dedicated list/preview/write lanes. A dismissed or superseded preview may be
   cancelled; an already started write must not be cancelled by another panel request or
   replayed automatically. Allow a bounded filing timeout appropriate to PDF generation
   rather than treating a normal render as a quick preview.
5. Detect an old bob only from the established exit-2 unrecognized-subcommand signature
   for `ref meet`. Cache that result by executable identity and clear it on
   replacement/recheck. Valid JSON exit-2 choice errors are supported behavior.

### Session behavior

- On entry run list and clipboard preview; use the supplied event token for the agenda
  path and no token for auto-match. Keep list and preview results independent so a slow
  list cannot replace a ready preview. Use request generations to ignore stale results
  after selection, option changes, leaving, or re-entry.
- Expose listing/changing-meeting, loading, preview-ready, choosing-parent,
  confirming-replace, writing, filed, queued, empty, unsupported, and recoverable error
  states. A change of event, parent, status, or overwrite choice invalidates the old
  preview and hash and obtains a fresh preview before enabling filing.
- Leave status unspecified initially so bob configuration wins. Display bob's effective
  status; selecting Next or Read passes the explicit override. Preserve a configured
  Ready/WIP status until the user chooses otherwise.
- Filing is allowed only from a current `would_file` preview with its exact event
  identity, selected options, and returned `notes.sha256`. It must explicitly pin the
  preview's EVENT, even when preview auto-matched. Missing identity or hash cannot
  enable Return. Repeated Return starts only one write.
- `ambiguous_meeting`/`no_meeting` offer Bob's candidates or the recent-meeting list;
  rows without an event token cannot be selected. `needs_parent` offers Bob's parent
  candidates. `already_filed` exposes Open existing and an explicit Replace choice; only
  that choice permits `-O`, followed by a new preview and Return.
- `notes_changed` clears the old approval and re-previews with a notice. It never
  automatically files the newly copied text. `no_notes`/`notes_unreadable` expose a calm
  paste/copy-and-retry state. Other errors preserve enough state for retry.
- Filed and queued are separate terminal receipts. The session notifies the app to
  refresh its agenda/library once; queued never claims a note exists or offers an Open
  note action for an uncreated path. Explain the supplied queue next step.
- Leaving discards session-only data and cancels reads, while a write in flight settles
  without issuing another write. Test the chosen close-during-write policy and its
  refresh notification; do not leave a hidden mutation result unobserved.

Add `Tests/Fixtures/meeting-notes/` with synthetic list, auto/explicit preview, choices,
filed, queued, changed-notes, empty, and malformed/schema cases. Record fixture
provenance and generator instructions. Exercise real CLI JSON from a temporary synthetic
vault when practical; validate against the landed structures. Tests must cover exact
argv/hash/selection replay, choice errors on exit 2, capability caching, stale
completions, single-flight writes, retry/leave behavior, and receipt distinctions. Async
test gates must have timeouts.

Update Mac `docs/protocol.md` with the optional command, schema, error/exit handling,
hash guard, and older-bob behavior. Pass the portable and documentation gates.

## Phase meetings-library

Add `RefKind.meeting` for `ref_type: meetings`: label `Meeting`, symbol `person.2.fill`,
and a distinct existing-system-color tint that does not reuse the Now status pink.
Include `meeting`, `meetings`, and `notes` in kind search words. Update every exhaustive
kind/scope switch, captions, inspector presentation, and fixtures that enumerate kinds.
Keep unknown reference types working.

Add `RefScope.meetings = 6`, its visible scope control, and Command-6 dispatch; retain
the existing numeric scope values and persisted selection compatibility. The scope must
compose with existing search/newest/tabs behavior. Add portable
kind/search/filter/caption tests and app key-router/presentation tests. Update
`docs/refs.md`; the subsequent panel phase updates the combined keyboard table.

Pass `just portable-check`, `just docs-check`, and the Mac compilation/CI gates below
because this phase changes app-target switches and keyboard handling.

## Phase meet-panel

### Sheet and entry points

Build `MeetingNotesPanelModel` and views around the landed portable session. Wire them
into AppDelegate and the capture panel controller/view, using a mutually exclusive
overlay with Event Note mode, existing focus restoration and panel sizing. Command-M,
**File Meeting Notes…** in the status menu, and the agenda Notes action all enter this
same sheet. Retain the capture draft underneath. Prevent menu/keys from opening a second
mode or launching capture submission during filing.

Present the meeting title, date/span, destination and effective reading status; show
auto-match evidence, Change meeting with filed rows marked, TL;DR, next steps, section
headings/counts, transcript disposition, and warnings from Bob. Include the Next/Read
control and the session's parent, replacement, empty, error, and unsupported states. Use
existing visual tokens, accessible labels, finite-height scrolling, and
keyboard-selectable choices; do not expose subprocess details as routine UI. A filed
receipt offers Open note using existing Obsidian navigation. A queued receipt stays
visibly queued with its next step.

Return files only the current preview. Return on a candidate selects and previews it; it
does not also file. Esc, q outside text controls, and Ctrl+[ dispatch the same one-level
back/close action. q remains typeable in filter controls and Ctrl+[ never types. Honor
IME composition, prevent key-repeat writes, and restore the capture focus. Route
shortcuts before AppKit can treat Command-M as minimize.

Hide all new entry points when the process client knows `ref meet` is unsupported; an
initial unsupported attempt shows a clear update message and returns safely. Recheck Bob
or an executable change restores the ability to probe support.

### Agenda

Decode the optional `event_ref` and `meeting_notes` fields using `decodeIfPresent` and
carry them into calendar presentation. Missing fields from an older bob add no action.
Show a focusable trailing **Notes** beside the existing Event Note action when a started
task has an event token and `state: missing`. The detailed approved design includes both
current and ended started rows; ended rows in the Earlier expansion must work. Future,
cancelled, filed, unknown-state, and ID-less rows offer no filing action. Do not infer
tokens from titles or calendar URLs. Keep existing calendar actions and layout intact;
refresh after the session's receipt so a filed row loses its Notes action.

### Tests and docs

Add old/new agenda decoding and eligibility tests plus focused app tests for all three
entry points, draft/focus restoration, mutually exclusive modes, key aliases, text/IME
exceptions, parent and replacement choices, repeat Return, queued receipts, and
capability hiding. Reuse synthetic rendering/test harnesses for a short manual or
captured visual inspection of preview, choices, and receipt at small and normal panel
sizes; use only invented content.

Add Mac `docs/meeting-notes.md`, update `docs/idle-agenda.md` and `docs/keyboard.md`
(including Command-6 from the library phase), and link the flow from the appropriate
existing docs page. Read docs memory before edits. If the parent CLI guide needs a
correction to link the now-shipped Mac flow, keep it narrow in
`docs/guides/file-meeting-notes.md`; preserve published URLs. Run documentation gates in
every repo whose docs change.

## Verification required for every implementation phase

Follow the linked repo's `AGENTS.md`: no Swift lands without compilation. Format only
changed lines. Run `just portable-check` on a host with Swift. If unavailable locally,
read tailnet memory and run it on an isolated permitted scratch copy on athena.
App-target changes additionally require a best-effort isolated scratch build on mac,
excluding `.git`/`.build`, with `Scripts/bundle.sh` and `just typecheck-tests`, followed
by scratch cleanup. Do not use Bryan's checkout, install the app, or attempt XCTest on
the Command Line Tools-only Mac.

Before landing Swift, run the linked repository's **`Scripts/macos-ci-check`** via
`/sase_monitor`; this script snapshots to a temporary CI branch and verifies the exact
SHA, including macOS build, tests, bundle, launch and install gates. Monitor every
subsequent push's exact-SHA CI to green as required. If scratch Mac access fails, report
it; passing Linux tests never substitute for macOS CI. Never finish the scope or close
`bob-cli-pw.8` with unverified new macOS failures. Long commands must be staged with
`/sase_monitor` and completed through `/sase_final`, not left running when a provider
turn ends.

Use targeted tests during development, then the required full gates once the tree is
ready. Record commands, tested SHAs, and CI links in the phase notes. A failure
reproduced identically on the clean base is recorded as `PROPOSED FOLLOW-UP:` on the
assigned phase, with evidence and an existing tracking bead when known; the user
explicitly permits closing with that documented base failure. Do not create task beads
or silently waive newly introduced failures.

## Child epic land completion

Review the combined implementation against the original mac-surfaces phase and this
plan. Verify the final combined Mac tree, docs, and exact-SHA macOS CI; inspect
synthetic evidence that auto-match and agenda preselection both reach a guarded filing,
clipboard replacement requires a fresh preview, queued differs from filed, old bob
leaves ordinary Capture working, and Meetings search/scope works. Confirm no real
meeting data entered tracked files and no forbidden memory changed.

Before closure, run `sase bead epic-symbols bob-cli-pw.8`; resolve any entries or re-key
them to a genuinely still-open owner. There were none during planning. Follow the same
symbol check for each assigned child before its closure. The child land agent should
first settle the child epic and its descendants through its authorized host workflow so
they do not block the original phase's descendant guard. It should then close **only the
original phase** with
`sase bead close bob-cli-pw.8 --note "<implementation, tests, exact-SHA CI evidence>"`
once its work is complete. Never close the original parent `bob-cli-pw` or any higher
ancestor.
