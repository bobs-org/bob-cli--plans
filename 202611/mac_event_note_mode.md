---
tier: epic
title: Complete Bob Mac Capture Event note mode
goal: 'Complete bob-cli-ji.7: open a private event note from any capture draft with
  Command-E, the status menu, or a Calendar row; choose an event and scope; edit with
  Bob''s highlights and a faithful hatched time rail; save, clear, and undo through
  bob gcal note JSON while preserving capture text and undo history. Ship the documentation
  and verify the exact Swift tree with macOS CI.'
phases:
- id: contract
  title: Complete event-note fixtures and the Swift process contract
  depends_on: []
  size: medium
  description: 'contract: produce full normalized bob-cli JSON fixtures, expose structured
    effective buffer reasons and effective fields on show, then add typed CaptureCore
    responses and isolated gcal-note client methods using decodeReport. Copy the fixtures
    into fake-bob and cover exit-2 reports, compatibility, argv, and overlapping UTF-8
    spans. Read the Contract section.'
- id: session
  title: Event-note session, request ordering, and mutation undo
  depends_on:
  - contract
  size: medium
  description: 'session: implement a dedicated event-note model with list/filter/select,
    explicit scope, separate per-scope drafts, debounced preview, serialized save/clear/undo,
    capability state, and stale-response rejection. Test its complete lifecycle with
    bounded fake-bob operations before panel wiring. Read the Session section.'
- id: rail
  title: Shared hatched meeting and buffer rail
  depends_on:
  - contract
  size: medium
  description: 'rail: extract the agenda capsule and buffer-wing rendering into a
    reusable view, draw real diagonal hatching, and add the event-note rail driven
    by Bob''s structured times, hue, and reasons. Preserve agenda styles and sizing;
    add geometry and light/dark render coverage. Read the Rail section.'
- id: picker
  title: Mode entry points, capture preservation, and event picker
  depends_on:
  - session
  size: medium
  description: 'picker: wire Command-E, Event Note status menu, and Calendar-row URL
    entry into a surface inside the capture panel with a grouped keyboard picker.
    Keep the actual capture editor and its undo manager alive; route focus, sizing,
    Escape, capability, and panel switches without capture writes. Cover the entry
    and restoration paths in hosted tests. Read Picker and entry.'
- id: editor
  title: Highlighted editor, live preview, and save controls
  depends_on:
  - picker
  - rail
  size: medium
  description: 'editor: add the fixed identity header, recurring scope control, multiline
    AppKit editor, UTF-8 highlight/error overlays, shared rail, Lands as block, status
    and footer keys. Wire save, clear, and last-mutation undo to the session model.
    Complete hosted end-to-end and render tests, including draft/undo restoration
    and unsupported Bob. Read Editor and acceptance.'
- id: docs
  title: Document the shipped mode and assemble completion evidence
  depends_on:
  - editor
  size: small
  description: 'docs: add the Mac event-notes page and nav entry, short keyboard/protocol/
    idle-agenda links, and On your Mac in the bob-cli guide. Correct the mode''s pending
    wording, run mounted docs gates, and collect phase and exact-SHA CI evidence for
    the child epic''s land agent. Do not close bob-cli-ji.7 or any ancestor from this
    phase. Read Documentation and Landing.'
proposed_by: bbugyi200.apollo.bob-cli-ji.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ji.7
create_time: 2026-11-10 10:29:43
status: done
bead_id: bob-cli-ji.7.1
---

- **PROMPT:** [prompts/202611/mac_event_note_mode.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_event_note_mode.md)
- **PARENT:** [202611/private_gcal_event_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/private_gcal_event_notes.md)
- **BEAD:** [bob-cli-ji.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ji/bob-cli-ji.7.1.md)

# Complete Bob Mac Capture Event note mode

## Scope and settled decisions

This is a child epic for the assigned phase **bob-cli-ji.7**, under **bob-cli-ji**. It
implements the Mac section of `plan:202611/private_gcal_event_notes.md`, supported by
`research:202611/private_notes_for_unowned_gcal_events/private_notes_for_unowned_gcal_events.md`.
Read those artifacts with `sase artifact read` when consulting them. The original phase
had no earlier implementation/remainder notes at planning time.

The parent's choices are final: `auto_sync = yes`, `mac_chord = cmd_e`,
`memory_record = yes`, `memory_glossary = no`. Command-E works with a nonempty capture
draft. Bob performs the writer-host sync; the Mac must not issue a second sync after
saving. The authorized decision records have already landed. This child epic changes
**no memory files** and asks no replacement decisions.

Read `decisions:mac-capture-is-a-thin-client` and
`decisions:private-event-notes-beat-google-per-field` through `sase memory read`. The
app owns presentation and process orchestration. Bob alone owns parsing, normalization,
validation, precedence, root selection, occupied time, task rendering, and journal/vault
writes. It never RSVPs or writes to Google.

Every phase starts in bob-cli and opens `bob-mac-capture` through `/sase_repo`; read the
linked checkout's `AGENTS.md` and use only the returned checkout. Paths below are
relative to the named repository, never to a numbered workspace. No deployment into
Bryan's installed application or live vault is needed.

## Existing work and integration facts

The CLI selection/read contract landed in `a2809a1d`, write/preview in `cf7c0605`, and
the decision/guide in `8f2b4925`. The inspected Mac HEAD was `6b6052c`. Recheck history
and phase notes before implementing; preserve any equivalent work that lands meanwhile.
There are no Event note models, views, client methods, or fake-bob cases in that Mac
tree.

Important existing boundaries:

- Bob's command implementation is `src/native/gcal/note.rs` and
  `src/native/gcal/note/write.rs`; tests are `tests/gcal_note_cli.rs`. The eight files
  under `tests/fixtures/gcal_note/json/` are currently **subset assertions**, not
  complete responses. They cannot be used verbatim as UI fixtures.
- `show` currently places text in `event.note.text` and uses string-array
  `event.note.lines`. Writes put typed line objects in top-level `note.lines` and text
  in `note.text`. `show` lacks top-level `input`, `spans`, and `effective`. A blank
  preview is a structured `empty` validation error; clearing uses `-c`, never a guessed
  empty write.
- Actual `effective.root.hue` is a nullable string. Meeting/occupied bounds are also in
  `preview`; the parent plan's illustrative JSON is not the decoder schema. Effective
  buffer reasons are currently available inside Rust's layered
  `candidate.event.description.buffer_reasons`, but not as structured JSON.
- Note spans overlap: `buffer` and its nested `reason_sign` are both returned.
  `CaptureCore/CaptureTextRanges.swift`'s `validatedSpanRanges` rejects overlapping
  arrays, so a whole note span array cannot go through it unchanged.
- `CaptureAgendaModels.swift` decodes `calendarURL`, but
  `CaptureAgendaPresentation.swift`'s `CaptureAgendaCalendarRowData` drops it. Calendar
  rows currently have a join action, not an event-note focus target.
- `CaptureAgendaView.swift` owns private `railEventCapsule`, `railWingView`, and
  geometry helpers. Wings currently use a 0.35-opacity fill. Its presentation already
  has normalized wing geometry, hue, off-clock/Maybe, and active state.
- `CapturePanelView`, `CapturePanelController`, and `CaptureKeyCommandRouter` own
  measured sizing, first responder, and layered key routing. Replacing the capture
  NSTextView while showing another mode can destroy its native undo.
- Capture's existing Control-E is line motion. Command-E is not claimed there; the
  optional Highlights Command-E hotkey applies only when Highlights is frontmost and
  must keep that scope. `AppDelegate.makeStatusMenu()` is the menu entry point;
  `BobPanelCoordinator` keeps Capture and Refs mutually exclusive.

The graph separates portable contract/state, shared drawing, and panel work so each
phase fits one coder. `rail` and `session` can proceed independently after `contract`;
`picker` and `rail` have separate production-file ownership. `editor` joins them. No
phase is a placeholder for an unbounded integration effort.

## Contract — phase contract

1. In bob-cli, add nullable `before_reason` and `after_reason` strings to
   `effective.buffer`, rendered by Bob from the **winning layered** reasons. Include the
   same `effective` object in show responses. Use the existing inline-to-display text
   facilities; do not introduce another description parser. These additive schema-1
   fields let a newly opened, empty editor display the effective rail and inherited
   reasons without parsing Markdown in Swift. They belong only to `gcal note` responses,
   never general list/sync JSON. Test occurrence/series/Google inheritance,
   explicit-zero and no-reason suppression, and unchanged row projection. Update
   `docs/gcal-notes.md`'s JSON contract in this phase after reading docs memory.
2. Upgrade the eight named JSON fixtures (`list_all`, `show`, `preview`, `saved`,
   `unchanged`, `cleared`, `invalid`, `ambiguous`) to complete reproducible responses
   from fake-gog integration cases. Normalize only unstable paths, timestamps, and URLs
   consistently, retaining valid realistic values. Assert the normalized whole object
   equals the fixture. Add focused full fixtures for recurring scope, both scopes,
   inherited reasons, waiting reply/Maybe, applied, saved elsewhere, and
   saved-but-sync-failed as needed for UI tests. Never use live Google or personal event
   content.
3. Add `CaptureCore/GcalNoteModels.swift` (or equivalently focused files) with
   `SchemaVersioned` envelopes for listings, show/write/preview, and structured
   errors/candidates. Model event identity, organizer, RSVP, summaries, scope, typed
   note lines, highlights, ranged errors, warnings, effective data, landing preview, and
   status. Keep show and write note representations distinct. Decode optional/additive
   fields safely and reject schema mismatches. Unknown display strings should degrade
   neutrally without inventing semantics.
4. Add focused `BobProcessClient` methods for list, show/select, preview, save, and
   clear. All use argument arrays and `-f json` with `decodeReport`. List is
   `gcal note -l -a -f json`. Selection after a picker uses `-i ID -C CAL` and
   `-s event|series`; URL entry passes the original URL as QUERY and lets Bob resolve
   it. Never decode `eid` in Swift. Pass full multiline text as one argument after `--`,
   including leading dashes and Unicode.
5. Use separate `gcal-note-list`, `gcal-note-show`, `gcal-note-preview`, and
   `gcal-note-write` lanes. Preview cancels its predecessor. Mutations use
   `cancelsPreviousInLane: false` and the model serializes them; previews cannot cancel
   saves or capture submits. Retain finite timeouts. Add narrow unsupported
   `gcal`/`note` subcommand detection cached by `identityToken`, following the existing
   optional-command patterns. Authentication, network, malformed JSON, and invalid-note
   errors must not become "old bob".
6. Copy complete contract fixtures into a dedicated `Tests/Fixtures/gcal-note/`
   directory in the Mac repo, recording their source and regeneration route. Add a
   `gcal note` fake-bob case supporting per-operation outputs, a temporary synthetic
   note store for save/clear/undo, argv recording, delays, and explicit errors. It must
   not touch a real vault or depend on a sibling checkout.
7. Add portable decoding/client tests for every fixture and exit-2 JSON, schema
   mismatch, empty/malformed output, unknown optional fields, identities, lane
   isolation, Unicode/newlines/option-like text, and capability reset. Supply a small
   span adapter that validates each returned range through existing UTF-8 helpers
   (singleton `validatedSpanRanges` calls or nonoverlapping paint layers). Keep the
   existing global validator strict. Paint broad styles before nested sign/error
   overlays, and ignore unsafe ranges without crashing.

Acceptance: Bob-generated full fixtures decode in Swift without weakened required
identities; no client grammar is added; all modified Rust and Swift compile and the
appropriate gates below pass.

## Session — phase session

Add a dedicated `@MainActor` event-note model in a new Mac file rather than expanding
`CapturePanelModel` with another full state machine. It owns the event-note lifecycle
and can be exercised without displaying the capture panel.

- States cover inactive, loading/list, resolving/show, editing, preview pending,
  writing, unsupported, and recoverable error. Local filtering never submits or chooses
  an event. Selection is an opaque calendar/event identity; group rows by Bob's day and
  order by the returned start. Never match mutations by title.
- Keep a generation plus selected identity, scope, and input text on each read request.
  Publish only if all still match. Cancel read lanes and debounce tasks on leave; late
  replies cannot reopen a mode, recolor a new draft, or replace another scope. Preview
  is debounced about 150 ms and the previous process is canceled. A show response loads
  scope-specific text; the layered summary is never substituted for that text.
- Recurring events expose This event and All events explicitly. Load/save each with
  `-s`; keep unsaved text separately per selected identity and scope within the open
  mode so switching scope does not overwrite it. One-off events expose only This event.
  Preserve invalid drafts and Bob's errors for correction.
- A new blank editor initially shows the show response's effective preview; no empty
  save occurs. User-edited blank text gets Bob's validation result; the explicit Clear
  action remains the route for deleting a saved note.
- Save sends the current editor text and lets Bob validate again. A stale preview never
  authorizes a write. Allow one mutation at a time, consume repeat Return, and keep
  navigation/close deferred or disabled until its bounded result settles. Closing a
  panel must not cancel a started write through an unrelated `cancelAll` path. On
  transport failure retain the draft and mark the result uncertain when appropriate;
  reload before any retry, never blindly retry writes.
- `ok: true` with `saved`, `cleared`, or `unchanged` is success. `sync_failed` still
  means saved: show Bob's warning and recovery detail. No second sync. Return to the
  picker after success, keep a status banner, and reload the list so note chips reflect
  Bob. Refresh the existing agenda store after mutations through its normal refresh
  entry point, without a parallel vault read.
- Keep one last-successful-mutation undo record containing exact event/calendar, scope,
  and previous canonical text/absence. Failed/unchanged operations do not replace it.
  Command-Z while a record exists re-saves that prior text or clears that exact scope;
  consume it only after success. Undo does not recursively add another undo entry. It
  remains available from the picker and editor during the mode session and never uses
  capture's undo manager. Without a mutation record, the event editor may use its own
  text undo. No persistent undo journal.
- Read-only selection/preview errors, an empty list, no filter matches, and an
  unsupported binary are distinct states. Offer bounded retry for recoverable failures.
  Executable replacement/recheck resets unsupported state.

Tests must cover out-of-order previews and shows, switching event and scope, rapid
edits, writes surviving unrelated reads, repeat-submit prevention, save→picker→undo,
clear→undo, first-save→clear undo, both recurring scopes, unchanged, invalid,
saved-but-sync-failed, failure recovery, and mode exit. Use finite
expectations/timeouts; no untimed continuations or sleeps.

## Rail — phase rail

Extract focused shared capsule/wing drawing from `CaptureAgendaView.swift` into a new
component such as `EventTimeRail.swift`. Keep agenda layout computation in CaptureCore.
Preserve positions, height, Now marker, conflict/gap rendering, active outlines,
off-clock dashes, Maybe styling, and measured agenda row height. The changed wing fill
is a clipped diagonal hatch drawn with SwiftUI Path/Canvas, not a raster image or a
character string. It must remain visible in both appearances and increased contrast.

Add the event-note variant using the same visual primitives, with a larger solid meeting
capsule and proportionate wings. Convert only Bob-reported times to display geometry;
never compute buffers or clipping from the typed note. Handle `00:00`, `24:00`, zero
wings, narrow widths, and degenerate/missing data without negative widths or NaN. Show
before/after reason labels from the structured effective fields, the occupied/meeting
time sentence, root hue, and the explicit midnight-clipping label. Missing optional
reason fields on an older supporting Bob omit the reason label; do not scrape
`preview.bullets`.

Provide a concise VoiceOver summary including both spans and reasons; decorative
geometry is hidden. Test normalized geometry in portable code where practical. Add
synthetic render cases at full/minimum widths, light/dark, increased contrast, long
reasons, absent reasons, single/both/zero wings, midnight clipping, off-clock, Maybe,
and active buffers. Extend the existing agenda render tests to verify the shared drawing
did not disturb folding, row measurements, or eye-line layout. Inspect the produced
PNGs, not only the test exit code.

## Picker and entry — phase picker

Wire the model into the existing capture panel without a new window or capture sigil.
Add a dedicated picker view and lightweight model/controller delegates. Keep the real
capture editor mounted/retained while the mode covers it, with its text storage,
selection, scroll position, and undo manager intact. Suspend capture-specific editing
and callbacks while covered; do not save/stash/rewrite the capture draft to implement
this mode. Restore its first responder on exit. Use separate event-note metrics so
existing editor/agenda sizing stays valid.

Entry routes:

1. Route exact Command-E in `CaptureKeyCommandRouter` to enter from empty or nonempty
   drafts before ordinary completion/picker handling. Preserve IME and existing
   in-flight capture transaction boundaries; repeated keydown cannot open twice or
   cancel a submit. Control-E keeps line motion. Add mode-owned routing before capture
   routes, including Escape and Command-Z.
2. Add `Event Note…` to `AppDelegate`'s status menu. Show through the existing
   coordinator, preserving a hidden capture draft and hiding Refs. An unknown capability
   is probed on use; confirmed unsupported hides the item until Bob changes/rechecks.
   Command-E can still show “Event notes need a newer bob”.
3. Carry `calendarURL` through Calendar row presentation. Add an accessible, focusable
   Event note row action and route Command-E when that row/action is focused to that
   URL. Preserve Join and the existing Command-J behavior. Outside a focused row,
   Command-E opens the picker. Missing URL offers the picker rather than guessing an
   event by its title; an ambiguous URL response shows Bob's candidates for explicit
   selection.

The picker has a quiet loading state, local filter field, day headers (Today, Tomorrow,
then formatted dates), and rows for time, title, organizer fallback to calendar,
Yes/Maybe/Awaiting reply chips, recurring marker, and noted summary in the root hue.
Selection moves with Control-N/P or Up/Down, scrolls into view, and only Return/click
opens an event. All state has text and accessibility labels. Escape from the picker
returns directly to the unchanged capture draft.

At this phase boundary, selection may show a tested identity/loaded-note detail surface;
the next phase replaces it with the full editor. Do not present disabled controls as a
finished editor or mark the original phase complete.

Hosted tests cover all three entry routes, nonempty capture text, selection and native
undo retention, focused Calendar URL dispatch, nil/ambiguous URLs, old-bob menu
visibility, unchanged Control-E/Command-J/Refs hotkeys, filtering, navigation, Escape,
and panel close/reopen/switch. Restrict production-file changes here to routing, menu,
panel integration, and Calendar row identity/action; the parallel rail phase owns the
existing rail drawing helpers.

## Editor and acceptance — phase editor

Add a dedicated multiline AppKit-backed editor with its own text storage and undo
manager. Show a fixed event identity card (day/time/title/organizer/RSVP), then the
scope control, editor, shared live rail, and dimmed Lands as block. For recurring scope
use **Control-Option-S** within this mode to toggle the segmented selection; verify it
is free in the mode router. Never use Tab for scope, because normal focus traversal must
remain usable.

Paint only matching `input.text` responses. Use the validated layered ranges from the
contract phase for route/root hue, clock, buffer, reason sign, reason, and prose; range
errors add red underlines plus their messages/suggestions. Unknown span kinds stay
neutral. Invalid UTF-8 boundaries cannot crash or highlight another character.
Repainting attributes must not register text undo, move the caret, cancel composition,
or rewrite canonical text into an active edit.

Show `preview.row` and `preview.bullets` as returned, with their path, without
rebuilding task fields in Swift. A null row (e.g. an invitation awaiting reply) needs an
explicit waiting/status explanation, not a fabricated task. Stale or invalid previews
cannot masquerade as the current input; dim/label any retained last valid preview while
validation is pending or fails. Display Bob's warnings.

Footer: `Only in Bob · This event` (or All events) and
`⏎ Save · ⇧⏎ New line · ⌘⌫ Clear · ⎋ Back`, plus discoverable scope/undo hints. Return
and keypad Return save; Shift-Return inserts a newline. Command-Delete clears the
selected saved scope. Do not clear an inherited series note when the selected occurrence
scope is absent. Escape from the editor returns to the picker while retaining that
session's per-scope draft; Escape again returns to capture. Saving/clearing returns to
the picker with Bob's saved/applied/waiting status and updated summary. Command-Z
invokes the session's last mutation undo even from the picker, never the hidden capture
editor. Buttons mirror key actions.

Add end-to-end hosted tests against the stateful fake for picker→editor→save→
picker→undo, clear/restore, first note, Unicode errors, recurring scope changes, rapid
stale responses, long text/scrolling, return-to-capture undo, old Bob, network and sync
failure, and repeat Return. Ensure event-note keystrokes never submit `bob capture`.
Test the response paths that omit optional effective fields, errors without preview, and
empty lists. Extend render fixtures to the complete picker, editor, invalid, saving,
saved/waiting, and unsupported states. Review light/dark full/minimum-width screenshots
and small-screen panel fitting.

## Documentation — phase docs

Read docs memory. In bob-mac-capture add `docs/event-notes.md` and `docs/nav.yml` with
entry routes, filtering, identity/scope, all keys, rail meaning, saved versus applied,
clear/undo, and compatibility. Keep short links/entries in `keyboard.md`, `protocol.md`
(optional command and compatibility table), and `idle-agenda.md`; do not expand
already-large reference pages past their budget. Use the repository's synthetic
render-media workflow if publishing examples.

In bob-cli add “On your Mac” to `docs/guides/plan-events-you-dont-own.md` and remove
only the now-stale “Mac Capture Event note mode is follow-up” wording in
`docs/gcal-notes.md`. Preserve the current state of the independent editor/adopt phase;
do not claim it shipped based on this child epic. No new flags or help options are
planned. If implementation does change help, read `cli_rules.md` and regenerate CLI docs
with `just docs-gen`.

Run the mounted docs check against these exact checkouts: set `BOB_CLI_DIR` to the
current bob-cli root when using the Mac wrapper, or `BOB_DOCS_BOB_MAC_CAPTURE_DIR` to
the opened Mac checkout when invoking bob-cli's gate. Do not let the wrapper clone a
separate unmanaged repository.

## Verification and landing

Each coding phase runs its focused tests while developing, then the repository's
required gates before landing. No live Calendar access or live-vault mutation: Rust
tests use `GcalEnv`/fake gog; Swift tests use synthetic fake-bob fixtures.

- For Rust production/fixture changes: `cargo fmt --check`, the project's clippy gate,
  focused `cargo test --test gcal_note_cli`/gcal-note tests, then `just check`
  (including the full required suite). Run `just docs-check` for contract changes. Never
  edit the Cargo version or CHANGELOG.
- For every Swift-changing phase: `just portable-check` compiles/tests portable targets
  and parses all Swift. If Swift is unavailable, read tailnet memory and use a scratch
  copy on athena. AppKit changes additionally require a scratch Mac copy excluding
  `.git`/`.build`, `Scripts/bundle.sh`, and `just typecheck-tests`, best effort. Never
  use Bryan's checkout or `just install`. If Mac is offline, record that scratch
  verification is unavailable and rely on the required CI compilation/tests; do not
  claim those targets verified before CI passes.
- Before landing Swift run **`Scripts/macos-ci-check`** from the opened Mac repository.
  Use `/sase_monitor` for this long gate and wait for the staging command itself to
  exit. It snapshots the actual dirty tree and watches its exact SHA on a temporary CI
  branch. Record the SHA, URL, and all required step verdicts. After any further push,
  watch that exact SHA with `/sase_monitor`. Never substitute an unrelated earlier green
  run or stop at a successful lint. Later Swift changes invalidate the corresponding
  earlier verification.
- Inspect relevant render PNGs from the CI artifact; test hangs require the sampled
  xctest evidence and a bounded-test fix. Run docs gates for both affected trees as
  described above. Commit changed goldens with their owning repository.

Workers close only their assigned child phase, after `sase bead epic-symbols` and the
required checks. Never hand-change lifecycle status, create follow-up beads, or close
bob-cli-ji.7 from a child phase. Record discovered unrelated work as
`PROPOSED FOLLOW-UP:` notes on the assigned phase. An identical clean-base check failure
is documented with reproduction evidence and any existing bead, not used to leave a
completed phase open. Feature regressions must be fixed.

The **child epic's land agent** owns integration, final evidence, and completion of the
containing phase **bob-cli-ji.7**. It must confirm every acceptance item, review the
final cross-repo diff, verify no unfinished stubs remain, and check the final exact
Swift SHA's green macOS run. Summarize source commits, contract fixtures, render review,
docs results, and any clean-base-only failures on bob-cli-ji.7. Before closing it, run
`sase bead epic-symbols bob-cli-ji.7` and resolve/re-key any remaining entries to a
genuinely open owner. Close only the containing phase with
`sase bead close bob-cli-ji.7 --note "<verified evidence>"` when its work is complete;
never close original epic **bob-cli-ji** or another ancestor. Follow the runtime's own
child-epic settlement ordering for descendants. Each turn ends with `/sase_final`,
committing all changed repositories and using `keep` while the original assigned scope
is still incomplete.
