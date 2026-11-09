---
tier: tale
title: Mac new-record preview and success summary
goal:
  Complete bob-cli-ir.11 with a faithful note preview, inline record refusals, committed
  success summaries, and verified macOS behavior.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ir.11
proposed_with: codex/gpt-6-astra
bead: bob-cli-ir.11
create_time: 2026-11-09 09:49:23
status: wip
---

- **PARENT:**
  [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:**
  [bob-cli-ir.11](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.11.md)

# Mac new-record preview and success summary

Complete phase `bob-cli-ir.11` of `plan:202611/record_notes.md`. A complete record draft
must preview the note Bob will write as labeled properties and rendered body, offer the
exact Markdown in a disclosure, and report the record after successful capture. An
incomplete record refusal stays inline with Bob's ready-to-copy draft. Implement the
feature and verify it before closing only `bob-cli-ir.11`.

## Scope and evidence

Read the assigned bead with
`sase bead read bob-cli-ir.11 -r "Need current phase scope, decisions, and notes"`
before implementation. It was in progress with no earlier-work notes at planning time.
Do not change its status manually or reopen it. Its prerequisites `bob-cli-ir.5` and
`bob-cli-ir.10` are closed. Check current history before repeating work.

The inspected bob-cli HEAD was `7fe12b33`; the linked bob-mac-capture HEAD was
`eded3b9`. The latter includes portable record decoding, the Record Card, completion and
Return handling, field styling, and subsequent fixes. Preserve those implementations.
This plan adds the missing preview/result presentation; it does not rebuild guided
entry.

Open `bob-mac-capture` with `/sase_repo` and use only the returned checkout. Read its
`AGENTS.md`. Before documentation changes, read `docs.md` through `/sase_memory_read`.
Read `decisions:mac-capture-is-a-thin-client` and the parent plan with
`sase artifact read`. The parent also requires the research sections Schema, Capture
syntax, Making input easy, and Reliability in
`research:202611/record_type_capture/record_type_capture.md`; the parent plan wins
wherever they differ. If that research initially resolves missing, open the configured
research sidecar through `/sase_repo` and retry the audited artifact read.

Final parent decisions are `seed_records=yes`, `memory_glossary_capture=yes`,
`memory_glossary_record_note=no`, and `memory_decision_record_types=no`. None requires
memory or vault changes in this phase. Do not edit memory, schema configuration, live
vault data, release versions, or changelogs. Record update cards, existing-record rows,
and Similar records belong to `bob-cli-ir.12`.

One coding agent can finish this medium tale: the planning and note-writing engine
already works, the required data mostly exists, and the UI uses the existing preview,
status, notification, and fixture paths. The small additive Bob change below supplies
committed metadata rather than introducing another resolver or a Swift dependency on
cached parse state.

## 1. Expose the existing result data and decode it

In bob-cli, `src/native/capture/output.rs` already defines `NoteFileJson` and
`NoteFilePropertyJson`; `CaptureResult::from_items` gathers one `note_files[]` entry per
record operation in item order. Both dry run and commit return these entries. Each has
`path`, `action`, `record_type`, ordered `properties`, `body`, and exact `markdown`;
update entries can also have `lines`. A property contains `key`, `label`, optional typed
`value`, `display`, and `source`, with optional `before`/`before_display` for updates.

The committed label, emoji, and authored note count currently live only in
`CaptureItemResult.record_note`, which is skipped during JSON serialization. Expose an
additive optional per-item `record_note` summary using those existing values: `action`,
`label`, optional `emoji`, `path`, `notes`, and `properties`. The straightforward
implementation is to serialize `RecordNoteSummary` and mark its internal `markdown` and
`note_file` members skipped, replacing the item member's unconditional skip with
omission when absent. An equivalent small explicit wire DTO is acceptable if it keeps
internal fields private. Do not duplicate the full Markdown in this summary. Ordinary
items must omit it. Existing summary construction in `src/native/capture/record_note.rs`
remains the authority for defaults, labels, and note counts.

This applies to single and mixed successes and dry runs; the same optional object can
describe updates for compatibility, although this phase presents the new-record case.
Preserve `note_files` as a top-level collection, the existing first-item mirror, and
`normalizedCaptures` semantics. No grammar revision or schema-version bump is needed for
this additive data.

In CaptureCore, add focused record-result wire types, preferably in a new
`CaptureRecordPreviewPresentation.swift`, and integrate them into
`CaptureCommandSuccess` in `CaptureModels.swift`:

- Decode `note_files` with an empty default, including its existing update fields for
  forward use, and `record_note` as optional per-item metadata.
- Reuse `CaptureRecordValue` for scalar/list/null property values. Render Bob's
  `display` preferentially, fall back to a readable canonical value, and retain
  meaningful zero/false values. Ignore absent optional values.
- Make additive decoding tolerant like `CaptureRecordPresentation.swift`: absent, null,
  malformed, or unknown optional metadata cannot invalidate an otherwise usable legacy
  response. An unusable entry gets the existing generic preview instead of an invented
  empty note.
- Decode the already-emitted record refusal details on `CaptureCommandFailure`: `draft`,
  `missing[]`, and `invalid[]` entries with `field`, `code`, and `message`. They are
  top-level JSON keys. Missing-field refusals may have no `code`; do not require a
  nonexistent `record_incomplete` code or classify them by English error text. Recognize
  the structured ready-to-copy payload, with record context as needed.

Document the added success object and existing refusal shape in
`docs/reference/json-outputs.md`. Extend `tests/cli/capture/record.rs` to prove single
and mixed summary shape, omission on ordinary items, and consistency between dry-run and
committed fields. Keep the existing test that compares `note_files[].markdown` with the
actual bytes written to the scratch vault.

## 2. Render the new note in the existing preview stack

Add a focused SwiftUI `RecordNotePreviewView` rather than growing the already large
`CapturePanelView.swift` with all layout details. Wire it through
`PreviewPane.previewContent` / `previewItem` for `kind == "record_note"` and the
corresponding `action == "create"` entry.

Match top-level file entries to record items in order using Bob's path/action identity.
Consume each once; never use only the first file or a path-only dictionary that
collapses multiple operations on the same path. Mixed task and record drafts retain
their original item order and task/block previews. Update items keep their existing
presentation for phase 12. When record metadata is absent, preserve the generic
fallback.

The card presents, in order:

1. A compact new-record heading and destination path, with Bob's emoji and label when
   available.
2. A property sheet in Bob's order. Show the human labels and display values; mark
   default values with a visible `default` badge and expose typed/default provenance to
   accessibility. Render wikilink values as link chips using the existing display
   conventions, including aliases and block anchors. Lists wrap; plain values remain
   readable. Do not infer a record schema or look up links in the vault.
3. The supplied `body`, including its title, Notes heading, dated bullets, nested
   bullets, and authored sections. Render these as note content, not a task checkbox or
   an all-monospace blob. Reuse `TaskDisplayText` and existing inline rendering for
   links/code where suitable. A small presentation-only renderer for Bob's Markdown body
   is in scope; parsing capture grammar, frontmatter, or reconstructing note content is
   not.
4. A collapsed **View Markdown** disclosure. Its selectable monospaced content is the
   original `markdown` string, including YAML, indentation, Unicode, blank lines, and
   trailing newline. Do not trim or reserialize it, and preserve the exact original
   string for any copy action.

Use the preview area's existing scrolling and natural-height policy. Long paths, labels,
values, and nested notes must stay reachable at 620px minimum and 760px standard panel
widths. Keep disclosure state keyed to the current record identity so response refreshes
do not flip it or transfer it to another item. Preserve keyboard focus and give the
disclosure, default badges, and link chips useful VoiceOver labels. Respect Reduce
Motion and contrast.

## 3. Keep refusals inline and show committed success

Introduce a small portable refusal presentation and inline card from Bob's structured
failure payload. Show its error message, missing/invalid field details, and selectable
ready-to-copy draft. Never silently apply that example draft: it can contain ellipses
and choice hints. The user retains the entire original draft, including other items in a
mixed capture.

Wire live-preview, explicit-preview, and submit failure paths consistently in
`CapturePanelModel.swift`, with presentation in the existing Record Card or PreviewPane
slot. Retain the current quiet guided-entry behavior: parse needs can suppress live dry
run, and Return on an incomplete record focuses the missing field without attempting a
write. Do not add redundant subprocesses just to fetch an incomplete refusal while the
current Record Card already shows that state. When Bob actually returns the structured
refusal, display it inline without a modal and clear an obsolete successful note
preview. Clear refusal state on edits, discard, success, or an unrelated failure. Accept
replies only under the existing draft/generation/client checks.

Use one portable success presentation derived from the committed per-item `record_note`
object for panel status, accessibility, and `NotificationService.swift`. For example:

```text
✓ 1/1  🍽️ new restaurant  eat/ramen_nagi.md
        💚 Liked · Ramen · Clark · with Kelly · 1 note
```

Build the detail from Bob's ordered nonempty property displays and note count, not from
the cleared editor or a prior parse. Use readable list/link labels; `with` is
appropriate for the `companions` property, while arbitrary configured fields remain
generic. Handle zero/one/many notes and absent emoji. Preserve batch accounting and
include each record once in mixed success notifications. Reuse existing success
notification/open-target behavior so the message remains useful when the panel hides
after capture. Never report creation on dry run or failure. Older Bob falls back to
current result copy.

## 4. Fixtures, integration checks, and documentation

Extend `Scripts/generate-record-fixtures.sh` and its provenance README, or add a
companion generator that follows its scratch-vault and explicit `BOB_BIN` conventions.
Generate real `bob capture --dry-run -f json`, commit, and refusal fixtures from the
implementation's Bob binary; pin the Bob commit and all inputs. Never hand-edit
purported real output. Keep existing parse/completion fixtures stable unless a justified
contract change alters them.

Include a restaurant with choice glyphs, a default companion, aliased links,
Unicode/quotes/colons, and nested notes; a podcast with no authored notes; two record
items; a mixed task/record draft; missing and invalid fields; and an explicit empty
optional property. Use isolated temporary config, vault, cache/state directories, and
fixed `BOB_NOW`/timezone. Compatibility fixtures may be deliberately synthetic and must
be labeled as such.

Meaningful coverage:

- CaptureCore decode/presentation tests for real new and mixed responses, defaults and
  source labels, omitted metadata, unknown values, malformed additive payloads, and
  exact Markdown preservation.
- Model tests using the existing fake Bob harness for explicit/live previews, committed
  summary after the draft clears, mixed batches, inline refusal with the draft intact,
  and delayed stale failure/success replies. Use bounded waits; do not introduce untimed
  continuations.
- Hosted rendering through `RenderFixtureWriter` at 620px and 760px, light and dark,
  including collapsed/expanded Markdown, long values, multiple files, and the refusal
  card. Inspect the resulting images for clipping, wrapping, readable link/default
  presentation, and panel scroll behavior.
- Notification tests for record-only and mixed success, missing emoji, and legacy
  fallback. Keep existing guided Return and generic task tests green.

Update the linked repo's `docs/capture-previews.md` and `docs/protocol.md` to describe
new-note preview, source badges, exact Markdown, inline refusal, success copy, and
additive compatibility. Do not modify the mounted copies in bob-cli or promise phase-12
update/Similar UI.

## 5. Verification, Mac latency evidence, and completion

Run focused tests first, then required repository gates. In bob-cli run the record
capture integration tests, the default `just` gate, and `just docs-check` for its
contract change. In bob-mac-capture run `just portable-check`, script checks for changed
generators, and `just docs-check`. Format only changed Swift lines. Never land Swift
that has not compiled.

Follow the linked repo's macOS rules: read `tailnet.md` through `/sase_memory_read`,
prepare a disposable Mac scratch copy with rsync excluding `.build` and `.git`, then run
`Scripts/bundle.sh` and `just typecheck-tests`. Never use Bryan's checkout or run
`just install`. The Mac's Command Line Tools lack XCTest, so `--build-tests` is not a
valid substitute there. CI runs the actual macOS tests. Remove only the scratch copy
created for this work.

Measure typing-to-preview latency on macOS with a real Bob subprocess and a synthetic
scratch vault/config. Time a production editor/model edit through acceptance and layout
of that generation's ready preview, including debounce, parse, dry run, and rendering.
Use existing `CaptureSignpost` instrumentation plus a narrowly scoped development
harness or opt-in hosted benchmark; do not report subprocess-only duration as end-to-end
latency. Compare a complete record draft with a similarly sized ordinary task draft
under the same setup. Warm both, alternate at least 20 measured edits per case, and
record median and p95 milliseconds, sample count, Mac/OS, app and Bob SHAs, and
measurement boundaries on `bob-cli-ir.11`. Log timings and synthetic identifiers only.
Investigate a material regression before calling the feature complete.

If the personal Mac is unreachable, use an available macOS CI runner for compilation,
rendering, and the same real-Bob benchmark. Report any remaining unverified requirement
honestly; portable results or timings from athena do not satisfy the Mac measurement
requirement.

Before landing Swift, run the linked `Scripts/macos-ci-check` and observe its exact-SHA
macOS result. Use `/sase_monitor` for the long check and exact-SHA CI after every push,
as required by the repository; wait for the monitor handoff command itself to exit and
finish the turn through `/sase_final`. Do not end a turn with an unmonitored running
check. Capture run URLs and verification results in bead notes. No manual git commits
unless separately authorized; declare changes in both repositories through
`/sase_final`.

Only this phase may be closed. Before closure, run
`sase bead epic-symbols bob-cli-ir.11`; resolve each entry or re-key its Justfile line
to a still-open bead. There were none at planning time. Then run
`sase bead close bob-cli-ir.11 --note "<implementation, test/CI, layout, and Mac latency evidence>"`
once the scope is complete. Do not close `bob-cli-ir` or any ancestor plan bead.

Never create discovered task beads from this phase. Record each as
`sase bead note bob-cli-ir.11 'PROPOSED FOLLOW-UP: <summary — evidence>'`. For a check
that fails identically on a clean base, document the comparison and any existing tracker
ID in such a note; that unrelated failure alone must not prevent closing this phase
after its work is verified.
