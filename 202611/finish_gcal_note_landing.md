---
tier: tale
title: Finish private event note boundary cases and land bob-cli-ji.8
goal:
  Repair the remaining note command defects, synchronize Mac fixtures, and complete the
  epic and ready parent closeouts.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ji.8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ji.8
status: done
---

- **PARENT:**
  [202611/finish_private_gcal_event_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/finish_private_gcal_event_notes.md)
- **BEAD:**
  [bob-cli-ji.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ji/bob-cli-ji.8.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ji.8.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ji.8.land.md)
- **COMMITS:**
  - [05ec825](https://github.com/bobs-org/bob-cli/commit/05ec8253f144ed6e5947f659a17eb50dea45d251)
    — fix(gcal): finish private note landing boundary cases

# Finish private event note boundary cases and land bob-cli-ji.8

## Scope and evidence

Finish the bounded defects found by the bob-cli-ji.8 land audit. One coder owns this
tale, verification, and the epic closeout below; no land agent resumes after it. Do not
repeat the four completed phases or redesign the journal ordering.

Read `sase bead read bob-cli-ji.8 -r "Need landing audit and follow-up outcomes"`, all
four closed phase beads with their notes, and these audited plan artifacts:

- `sase artifact read plan:202611/finish_private_gcal_event_notes.md "Need remaining-work context"`
- `sase artifact read plan:202611/private_gcal_event_notes.md "Need parent design and decisions"`

The audited bob-cli base is `ba7a1c90`. The four phase commits are `54e650d1`,
`9660397d`, `b706f559`, and `ba7a1c90`. A fresh origin fetch found no non-epic commits
after the first ji.8 commit and no base divergence. Recheck drift before landing. Source
and tests substantiate the account-aware fold, honest held rows, pre-note
exact-selection clone, provenance pruning, alias stripping, read modes, span/suggestion
contract, adoption, and fragile hint. The gaps below remain.

Preserve the accepted decisions: `auto_sync=yes` on the writer host; Google stays
read-only; private notes do not alter event inclusion or create daily notes; schema
version 1 stays compatible with Bob Mac Capture. Do not edit memory, Cargo.toml's
version, or CHANGELOG.md. Do not introduce epic-symbol exemptions. Read the docs
reference memory before changing help or docs, and cli_rules if adding an option becomes
necessary (none is expected).

## 1. Honor the first explicit argument separator

In `src/native/gcal/note.rs::rewrite_note_tail`, `in_text` is checked before the
`token == "--"` branch. Once a directive started implicit TEXT, an explicit `--` is
retained as text and subsequent flags are still extracted as command options. An
executable probe compiling the unmodified current functions with rustc gave:

```text
input:  [-i, e1, @family, --, -d]
actual: [-i, e1, -d, --, @family, --]
```

Recognize the first explicit separator in either state. Drop that delimiter from the
note text, stop option extraction there, and preserve every following word verbatim.
Options extracted before it remain options. Emit exactly one clap separator; a second
literal `--` after the delimiter remains text. Cover implicit TEXT followed by `--`,
options on either side, attached-value options, and the existing explicit-before-TEXT
behavior in the pre-pass unit vectors and a fake-gog CLI regression that asserts the
actual saved/preview text and mutation count.

## 2. Resolve edit/adopt scope once, and keep previews read-only

`present_candidate` calls `current_scope_text`, which prompts via `resolve_show_scope`;
that selected scope is discarded. `write_one` later asks again or `choose_clear_scope`
picks whichever scope currently has a note. A user can seed the editor from All events
and save into This event, or an empty editor can clear the other scope. Resolve once and
carry the selected scope through the existing-note refusal, seed, validation retry loop,
preview, save, and clear. Preserve the existing explicit-scope requirement off a TTY.

`-e -d` passes `inner` argument validation. An empty editor result then sets
`prepared.clear=true`; the clear branch in `write_one` unconditionally appends and
syncs. Make empty edit previews non-mutating, just like nonempty edit previews. Keep the
public direct `-c -d` rejection; it is already correct. An empty edit without a saved
note must return `unchanged` for either scope. Currently recurring explicit scopes can
fail in `choose_clear_scope` before reaching that branch. Explicit `-c` with no note
must retain its existing error.

Add fake-editor regressions in `tests/gcal_note_cli.rs`: one prompt only when choosing
series, correct series seed and write with event text unchanged, empty series edit with
only an event note leaves that event note intact, both empty scopes report unchanged
without an append, and `-e -d` empty/nonempty leaves both journal bytes and an existing
rendered daily note unchanged. Exercise `-A`'s existing-note guard under the same
resolved scope. Use BOB_GCAL_ASSUME_TTY and the existing fake gog/editor helpers; never
use the live vault or Google.

## 3. Complete doctor remedies and live-master clears

In `src/native/gcal/doctor.rs::notes_check`, `fixes.into_iter().next()` discards every
remedy except the first although multiple issue kinds are reported. Preserve one
first-offender remedy for each present issue kind in human output and JSON. Keep the
existing `fix` string contract, for example with newline-separated fixes; do not replace
its type. Tests must combine orphan, unresolved-route, foreign, and malformed records
and assert every applicable remedy survives.

For an unresolved series route, doctor currently prints
`bob gcal note -e -s series -i <master> -C <calendar>`. A bare series master has no
recurringEventId, so this command fails `resolve_show_scope`. Supply a remedy that works
with the named identity: an exact occurrence edit when available, or the
already-supported explicit series-key clear. Do not invent an occurrence, add an
unbounded lookup, or silently mutate from doctor. Run the actual printed command against
the fixture, not just a substring assertion. Preserve the 25-lookup cap and
outage-versus-not-found distinction.

`run_show` also routes `-c -s series -i <live-master>` to `clear_by_key`. That helper
appends a tombstone but never runs writer-host immediate sync and reports that the event
is gone even when the master is live. This violates the accepted auto_sync decision.
Distinguish a live master from a genuine orphan; resolve its affected occurrence days
using the existing bounded selection machinery and reuse the normal save/clear
application path. Clearing a live series must update already existing, rendered affected
daily notes immediately on the writer host, preserve hand edits, and never create future
notes. On another writer host or a sync failure, preserve the saved clear and report the
established honest status. True missing-master clears must remain possible by key. Add a
test with a live master, today's noted occurrence and rendered task, clear by master ID,
and assert the actual row has reverted to Google values; cover off-writer and lookup
failures. Document any newly exposed status in `docs/gcal-notes.md`, including the
by-key `cleared` status already emitted but absent from its exhaustive list.

## 4. Refresh the affected Mac contract fixtures

The land instruction requires integrating consumers of the completed feature. Although
the earlier repair epic scoped phases to bob-cli and proposed this as a follow-up, these
stale copies are caused by the epic and are retained here.

Use `/sase_repo` to open `bob-mac-capture`; read its AGENTS.md and use only the path the
command prints. At Mac `e638795`, fifteen JSON copies under `Tests/Fixtures/gcal-note/`
differ from bob-cli's `tests/fixtures/gcal_note/json/`: applied, both_scopes, cleared,
google_reasons, inherited_reasons, invalid, preview, preview_match, saved,
saved_elsewhere, saved_sync_failed, series_scope, show, show_preview_match, unchanged.
The proposed list of eighteen overstated the changes: ambiguous, list_all,
waiting_states are already equal. Compare fresh trees instead of trusting that list.

Regenerate bob-cli fixtures only when an actual contract correction requires it, using
`BOB_UPDATE_GCAL_NOTE_FIXTURES=1 cargo test --test gcal_note_cli`, and inspect each
diff. Copy authoritative JSON bytes to the Mac fixture set and ensure all eighteen
agree. Existing portable consumers are
`Tests/CaptureCoreTests/GcalNoteContractTests.swift` and
`GcalNotePresentationTests.swift`; keep their meaningful assertions. Scope the
integration to fixture data and portable tests, without changing AppKit code. Never
hand-edit JSON merely to preserve a stale assertion.

## 5. Verify the repaired work

Run the focused Rust gcal-note, gcal-notes, doctor/auth and affected list tests, then
the canonical `just check`. Do not run `just check-full`. A separate full-check
infrastructure failure cannot be treated as remaining epic implementation work. Run
`just docs-gen` when help changes and check docs for new failures. Existing
mounted-source word-budget failures remain bob-cli-jz/h9; do not increase them.

In the Mac checkout run `just portable-check` and the gcal-note tests (the portable
suite includes them). If Swift is unavailable locally, use the repo's documented
tailnet/scratch workflow after reading required tailnet memory. Do not skip compilation
of any changed Swift. Before landing Mac changes, run `Scripts/macos-ci-check` against
the uncommitted tree, using its supported scratch verification so its result is
available before closeout; observe its command's actual exit status. Follow the repo's
monitor requirement after any push. Do not make closeout depend on the SHA, push, or CI
of this tale's own final commit. Use `/sase_monitor` for long commands, with
continuation instructions that preserve this tale's responsibility to finish the
landing.

Follow-up triage is already complete and recorded on bob-cli-ji.8:

- ji.8.2 note #1 timestamp-tie proposal declined: `(ts, host, path, line)` is the
  explicitly approved deterministic order, not an unimplemented promise.
- ji.8.3 note #1 and ji.8.4 note #1 are the same fixture proposal, retained in step 4.
- ji.8.3 note #2 corroborated existing bob-cli-jz and bob-cli-h9. Independent current
  canonical word counts at Mac e638795 were capture-panel 6036, capture-previews 6088,
  keyboard 6060, protocol 6149 (cap 6000). The plugin page is already in jz; this lander
  did not recount it. No new unrelated task was warranted.

## 6. Close this tale and epic, then recheck parent bob-cli-ji

This is the final implementation step, in the same turn as the last code changes; do not
wait for a final commit SHA. Record the regression results, fresh drift review, Mac
verification and all follow-up dispositions in the close notes. If this tale is a
tracked open descendant, close its assigned bead normally once its work is verified so
it does not block the epic's descendant check. Do this only for a distinct descendant
tale ID, never as an early close of bob-cli-ji.8 itself.

Run `sase bead epic-symbols bob-cli-ji.8`. Resolve every entry by wiring, privatizing,
adding an appropriate non-test pragma, or removing the symbol. Re-key an exemption only
to a still-open later bead that actually needs it. Then run:

```sh
sase bead close bob-cli-ji.8 --note "<verified repairs, all child notes and decisions, integration/drift, tests, and follow-up outcomes>"
```

Run `just symvision` if available (none existed at ba7a1c90). Use `/sase_repo` to open
`plans` before modifying its files; read the artifact through
`sase artifact read plan:202611/finish_private_gcal_event_notes.md "Need completed plan status"`
and set `status: done` in that plan's frontmatter. Use the PLAN location returned by an
audited bead read, not an assumed checkout path. If closing reports leftover symbols,
finish the cleanup and retry. Never use `--force` merely to succeed or to advance a
successful nested landing. Finish incomplete phases instead; cancel or supersede only
with an explicit, justified resolution, not for completed work.

After this epic closes, run `sase bead read bob-cli-ji.8 -r "Need the parent link"`. Its
direct parent at audit time is plan bead **bob-cli-ji**, linked to
`plan:202611/private_gcal_event_notes.md`. Re-read that parent, its prior landing notes
#4/#5, every descendant and all notes (including nested Mac plan ji.7.1), the linked
plan, and post-child drift in bob-cli and the Mac. Re-run descendant and linked-plan
readiness checks (`sase plan links validate`, attributing any unrelated legacy failures
rather than sweeping them into this scope). Confirm accepted auto_sync=yes, Cmd+E,
authorized decision records and no glossary strand remain satisfied. Prior parent fixes
4c1f7dde/85e618d8 addressed clippy/catalog/doctor-list failures; the earlier parent
follow-up outcomes remain in note #4.

If bob-cli-ji is fully complete, retire its `sase bead epic-symbols bob-cli-ji` entries,
close it normally with
`sase bead close bob-cli-ji --note "<parent descendants, notes, linked plan, drift, decisions and validation rechecked>"`,
run `just symvision` if available, and set its linked plan's `status: done`. Repeat only
through directly parented plan ancestors that are fully ready. If a direct ancestor is a
phase, verify and close only that phase, leaving its containing epic to its waiting land
agent. Stop at the first incomplete or ambiguous parent, append a specific blocker note
there, and report it; never force an ancestor closed. Finish through `/sase_final`,
committing all changed repositories declaratively.
