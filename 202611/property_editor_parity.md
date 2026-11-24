---
tier: epic
title: Complete Property Editor native parity
goal: The =+ Property Editor supports keyboard rename, reorder, selection, copy/paste,
  filtering, kind conversion, and validated YAML stages through the existing guarded
  writer, with real-Obsidian evidence and deployed plugins completing bob-cli-pf.5.
phases:
- id: selection-filter
  title: Add property selection, filtering, and batch delete
  size: medium
  depends_on: []
  description: 'selection-filter: add the marks gutter, stable selection and visual
    ranges, key/value filtering, and one-batch delete to the landed property editor;
    preserve the close ladder and guarded write lifecycle.'
- id: rename-reorder
  title: Add guarded property rename and reorder
  size: medium
  depends_on:
  - selection-filter
  description: 'rename-reorder: implement the inline rename stage and source-order
    moves with attached comments, stable focus, filter lockout, undo, and reduced-motion-aware
    feedback.'
- id: source-fragments
  title: Extract lossless property source and clipboard fragments
  size: medium
  depends_on:
  - rename-reorder
  description: 'source-fragments: build a read-only, source-ordered YAML fragment
    adapter for exact property copying, source-seeded YAML stages, and validated clipboard
    mappings without changing bob''s writer contract.'
- id: yaml-stage
  title: Add the validated multiline YAML value stage
  size: small
  depends_on:
  - source-fragments
  description: 'yaml-stage: add source-seeded YAML value editing through prop.set_yaml
    and the existing preview latch; retain opaque-key refusals and the source excursion.'
- id: clipboard
  title: Add YAML copy and overwrite-previewed paste
  size: medium
  depends_on:
  - yaml-stage
  description: 'clipboard: wire keyboard and menu copy/paste, show ordered new/replaced/skipped
    keys, and apply the exact approved editable subset in one guarded undoable batch.'
- id: kind-change
  title: Add local kind conversion and explicit vault type changes
  size: medium
  depends_on:
  - clipboard
  description: 'kind-change: preview and apply loose-value conversions; feature-detect
    Obsidian''s type manager and require a separate confirmation naming affected notes
    before changing a vault-wide property type.'
- id: acceptance
  title: Verify property editor parity in Obsidian and deploy
  size: medium
  depends_on:
  - kind-change
  description: 'acceptance: extend the existing scratch-vault real-Obsidian runner
    for all parity actions, review visual evidence, update the native Properties parity
    reference, run required gates, and sync the plugins.'
proposed_by: bbugyi200.apollo.bob-cli-pf.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pf.5
create_time: 2026-11-23 12:39:49
status: done
bead_id: bob-cli-pf.5.1
---

- **PROMPT:** [prompts/202611/property_editor_parity.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_editor_parity.md)
- **PARENT:** [202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md)
- **BEAD:** [bob-cli-pf.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.5.1.md)

# Complete native Properties parity in the Property Editor

This is a child epic for **bob-cli-pf.5**, the `editor-parity` phase of
`plan:202611/card_owned_frontmatter.md`. Attach the approved epic beneath that phase.
Finish its remaining implementation and verification; its delegated land agent then
closes **bob-cli-pf.5 only**, after all child work is complete. Neither child phase
workers nor this epic's land agent may close `bob-cli-pf` or any other ancestor.
Individual phase workers close only their own assigned bead.

## Scope and established evidence

The assigned phase was `in_progress` with no earlier implementation notes or child plans
when inspected. `bob-cli-pf.4` is closed, with its core-editor child epic also closed.
The linked `bob-plugins` checkout was clean at `5c9b4fbf` during planning. The editor,
service, typed stages, add flow, history, YAML excursion, entry points, tests, and
real-Obsidian harness are already present. Do not redo them. In particular,
`docs/property-editor.md` explicitly leaves rename, reorder, selection, clipboard,
filtering, kind changes, and inline YAML to this phase.
`298g-plugin-property-editor-keys.js` does not map these actions, and
`298h-plugin-property-editor-stages.js` excludes maps and opaque values.

The parent plan supplies the complete approved behavior. Its referenced research
artifact `research:202611/card_owned_frontmatter/card_owned_frontmatter.md` returned
`missing` during the audited read; this plan relies on the approved parent design and
the landed implementation, not on inaccessible research content.

All parent decisions remain final: `bank_foot=yes`, `drawer_memory=note`,
`memory_glossary_cards=yes` only for the two named glossary strands, and
`memory_decision_record=no`. This child epic makes **no memory changes**; the authorized
glossary work remains with the parent's takeover phase. It does not implement takeover,
backfill `created`, or add new CLI commands.

Use `/sase_repo` to open `bob-plugins` before reading or editing it, and obey its
`AGENTS.md`. All implementation paths below are relative to that repository unless
prefixed `bob-cli:`. Re-check current source and phase notes before implementing, since
other epics may have advanced the base.

## Architecture and invariants

Extend the existing editor rather than introduce another modal or writer:

- `plugins/bob-ledger-tools/src/298e-plugin-property-editor-model.js` owns role routing
  and groups; `298f-plugin-property-editor-modal.js` owns the modal;
  `298g-plugin-property-editor-keys.js` owns browse-key intents.
- `298h` and `298i` implement value draft parsing, retained inputs, previews, and
  lifecycle. `298k` and `298l` implement key validation and adding. `298m` owns entry
  points and YAML return. `298n` and `298o` contain the landed write-safety and
  row-fidelity fixes.
- `298-plugin-property-protocol.js` and `298a` through `298d` provide the existing
  `createPropertyUpdateSession`, exact inverse-op history, and confirmed overlays.
  `379-exports-property-editor.js` exposes test helpers.
- The current modal fragment is 908 lines. Put new behavior in focused source fragments
  and extract small shared helpers as needed. Every hand-edited fragment stays at or
  below 1,000 lines. Register fragments in order in `src/fragments.json`, then generate
  `main.js` with `npm run build`.
- Tests already use `scripts/ledger-tools-harness.cjs` and
  `scripts/test-ledger-tools-property-editor-*.cjs`. Extend those patterns and register
  new suites explicitly in `package.json`'s `test:unit`.

All note mutations use the existing `bob props update` protocol. The contract is
`bob-cli:docs/props.md`; relevant Rust code is under `src/native/props/`. Use
`prop.rename {key,to}`, `prop.move {key,before|after}`, `prop.unset`,
`prop.set {key,value,kind?}`, and `prop.set_yaml {key,yaml,before?|after?}`. No
`processFrontMatter`, direct vault writes, whole-file restoration, or client YAML
serialization may become a second note writer.

Every mutation requires a current capable report, a valid fingerprint, and Bob's fresh
ownership checks. Card, identity, derived, unresolved, and recovery rows retain their
existing routes and refusals. Rendering, filtering, selection, and copying never write a
note. New prospective keys are checked by Bob; do not duplicate Rust's ownership
classification in JavaScript.

Typed stages use the existing 250 ms dry-run and exact-draft Enter latch. Discrete
operations use the session's direct operation path. Save every open pane before apply.
Preserve drafts through stale refusals, require a new preview, and never automatically
repeat a write with an unknown outcome. Batch edits produce one history entry, not one
per key. A move can change bytes without changing the semantic fingerprint; use
`changed` and the returned inverse operations, not hash inequality, to record it.
Refresh all affected editor/card/drawer surfaces through the existing confirmed overlay.

All stages use `bobCardWireCloseKeys` and the existing Esc dispatcher. The ladder
remains menu → stage → filter → modal. `q` types only in the panel's text controls;
Ctrl+[ always follows Esc. Clipboard and ordinary text undo in inputs retain their
native behavior. IME, composition, repeat-triggered writes, pending apply, closed
modals, and unload cannot launch duplicate or late work.

## Phase selection-filter

Implement the selection and visible-row model before adding more commands. Keep the
complete source-ordered inventory distinct from the grouped visible rows. Rendering,
focus traversal, range selection, and filter matches must agree on the visible row
order. Use source occurrence identity when duplicates are displayed, and reconcile
successful rename/move/refresh by a unique key when line numbers change. Never transfer
a mark to a different duplicate.

- Space toggles a mark on the current row and consumes the native button activation. `V`
  enters a visual range; `j`/`k` extend from its anchor. Pressing `V` again exits range
  extension while retaining the marks. Shift+↑/↓ extends a range. Ordinary navigation
  outside range mode only moves focus. A visible, accessible marks gutter distinguishes
  focus from marks.
- `/` opens a labelled incremental filter over keys and the complete readable value
  text, including list/map content, rather than clipped row text. Enter or ↓ returns
  focus to the matching rows. An empty match set has a clear message and a stable
  keyboard target. Esc clears the active filter layer before closing the modal,
  including after focus has returned to rows.
- Filter edits reset marks and the range anchor, so hidden rows cannot be deleted by
  accident. Announce that reset when marks existed. Navigation and range extension
  operate only on visible rows. Opening an empty filter does not silently clear marks
  until its query changes.
- Ctrl+D / Mod+Backspace deletes marked rows, or the current row if none are marked, in
  one ordered batch. For a mixed protected/loose selection, refuse the batch with named
  locked rows rather than partially deleting it. Copy may later include read-only rows.
  Undo/redo remains one user action.
- While any filter query is applied, expose a disabled reorder action with the
  instruction to clear the filter. Carry that state to phase two.
- Update the hint strip and All actions entries for the features added here; add concise
  keyboard and selection/filter behavior to the reference now.

Test row order versus visual order, duplicate identities, range reversal, boundaries,
focus after deletion/refresh, zero matches, marks reset, protected batch refusal, one
inverse batch, close-key layering, typing guards, and no writes/spawns from
selection/filter paint. Retain the existing core suites.

Acceptance: a keyboard user can mark a range, filter and return to rows, delete a safe
marked set, and restore it with one undo without affecting hidden or protected keys.

## Phase rename-reorder

Add `r` for an inline key-name stage seeded with the current key. Reuse the existing key
validator, but exclude the current key from the duplicate check. Preview `prop.rename`;
preserve the typed draft on refusals. A destination that would be card-owned, identity,
or derived must be refused by Bob even when it does not exist. A same-name rename
remains a no-op. Successful rename moves focus and any mark to the renamed key without
losing other marks.

Add Alt+J/K and Alt+↓/↑ to move the focused property relative to its adjacent entry in
the full source inventory. Do not derive neighbors from the grouped or filtered DOM.
Reordering moves only the focused property; the marks remain for batch delete/copy. A
protected row cannot itself be moved here; a protected neighbor may be a position
reference. Boundaries are no-ops. Keep the selected key visible, announce its new source
position, and retain grouped presentation. Disable moves while a filter query is active,
a stage is open, or a write is pending. Do not queue held-key writes.

Use `prop.move` so leading comments travel with the entry. Add the parent design's 120
ms row-slide feedback using the existing style language; disable it under reduced motion
and cancel stale animation callbacks on close/unload. Add both commands to the All
actions menu and update their docs.

Test duplicate and protected rename destinations, quoted/Unicode/integer-like names,
focus after rename, group boundaries, filter lockout, exact operation neighbors, comment
movement and one-step undo. Verify moves still enter history when their before/after
semantic fingerprints agree. Extend DOM tests for keyboard modifiers, reduced motion,
and late preview/apply races.

Acceptance: rename changes only the key token and reorder moves the entire entry through
Bob; focus, marks, untouched values, and undo remain correct.

## Phase source-fragments

Provide one pure, read-only YAML source adapter plus a thin asynchronous snapshot reader
for both later clipboard and YAML-stage consumers. This is necessary because
`props show` exposes `line`/`end_line` but not raw ranges; multiple flow-map entries can
share a line. A raw slice based solely on those line numbers is insufficient. The
existing `240n-plugin-property-card-inventory.js` scanner is a useful starting point,
but currently splits lines and does not preserve byte spans or root-flow entry
boundaries.

Return ordered entries with normalized key/occurrence, exact key-entry span, attached
leading trivia, and the value fragment suitable for `prop.set_yaml`. Keep original line
endings, quotes, inline comments, block scalar contents, and integer-like key order. For
block mappings, copying entries retains their original key/value bytes. For root flow
mappings, preserve each selected raw entry and assemble a valid flow mapping using only
the necessary braces and commas. Do not copy delimiters, note body, or another entry by
accident.

Use Obsidian's `parseYaml` for semantic validation and a small structural source scanner
for ordered boundaries; never treat `Object.keys(parsed)` as source order or use a
line-only regular expression as a YAML parser. Support the native property shapes,
quoted keys, multiline scalars, nested maps/lists, and root block/flow mappings.
Cross-check parsed mapping keys against the source entries. Reject ambiguous boundaries
and unsupported YAML features explicitly instead of guessing or silently normalizing
them. Treat `__proto__` and similar names as data using Maps or null-prototype records.

The clipboard decoder accepts one YAML mapping, optionally wrapped in one matched
frontmatter delimiter pair; returns ordered value fragments; and rejects
malformed/multi-document input, duplicate keys, non-mapping roots, fence/body injection,
and non-string/non-scalar property names. Anchors, aliases, merge keys, custom tags, and
unresolved cross-entry references must remain explicit unsupported/refused cases, not
flattened values. Parsing clipboard content never executes or writes it.

Keep source extraction distinct from paste decoding: when an opaque property's raw span
is unambiguous, retain it for exact copying and inspection in the YAML stage. A readable
raw span does not authorize writing or imply that pasting it into another note is safe.
Never substitute a JSON-rendered value for it.

The snapshot reader uses current note source and checks that path, source content, and
report inventory still correspond before returning spans. Read source around the fresh
report and retry a bounded number of times if it changes. Handle open unsaved panes
explicitly: do not silently copy stale disk text as their latest contents; either use a
uniquely authoritative editor snapshot with matching inventory or request saving before
retrying. A read action must not itself save the note. Cancel late reads on path change
or unload. Future applies still use Bob's fingerprint and session guards.

Expose helpers through the existing test export mechanism, register a focused suite, and
document the adapter's result/error contract for the next phases. Test BOM, LF/CRLF, no
final newline, leading comments, empty/null/false/zero, coercible text, quoted colons,
block-scalar chomping, nested and flow values, root flow maps, numeric-looking keys,
Unicode, duplicates, unsafe YAML, concurrent source changes, and unsaved-pane refusal.
Assert byte slices as well as parsed values. Do not broaden the CLI report or add a
second writer.

Acceptance: later phases can obtain validated ordered source fragments and clipboard
values without reconstructing a note from metadata or losing source distinctions.
Unsupported input returns an actionable refusal.

## Phase yaml-stage

Extend the existing typed-stage controller with a multiline YAML value stage seeded from
the source-fragment adapter. Loose map values open it on Enter; loose opaque values may
be inspected in it with a clear diagnostic. Provide a menu route for editing another
loose value as YAML without stealing `y`, which continues to mean the existing
Source-mode excursion.

Build only `prop.set_yaml {key,yaml}` and validate through the existing debounced Bob
dry-run. Show Bob's error and hint verbatim. Preserve multiline editing, caret,
composition, and native text undo; use an explicit Save button and Mod+Enter to submit
the multiline draft, with the current preview latch. Plain Enter inserts a newline. Wire
cancellation into the existing stage layer; it must not close the whole modal or apply a
draft.

The parent design's opaque-stage request does not override its byte-safety rules: Bob
still refuses existing anchors/aliases/merge keys/custom tags with `prop_opaque`.
Display that refusal, retain the draft, and keep `y`/Source repair reachable. Do not
bypass it with `prop.restore`, direct writes, or a flattened parsed value. Card maps
continue to route to `==`.

Test successful loose-map edits, malformed drafts, sibling/document injection, null
versus empty values, opaque refusal, current-preview submission, stale retention,
cancellation, and map undo/redo. Add the stage's controls and limits to
`docs/property-editor.md`.

Acceptance: editable nested values gain an in-modal validated YAML stage, while every
existing ownership and opacity guarantee remains enforced.

## Phase clipboard

Wire Mod+C and Mod+V in browse mode and add matching All actions entries. Inside text
inputs, leave clipboard events to native text editing.

Copy marked rows, or the focused row when none are marked, from the validated source
snapshot. Order them by file order even when the visible groups differ. Read-only keys
may be copied. Report clipboard unavailability/denial without changing the note or
claiming success. An ambiguous/unsafe source selection must refuse with a YAML escape
rather than emit fabricated YAML.

Paste first reads the clipboard and decodes it through the source adapter. Open a stage
showing every incoming key in source order with a status: **New**, **Replace**
(including old and proposed values), or **Skipped** with the ownership/opacity reason.
Existing card, identity, derived, and unresolved keys are skipped. Use Bob's read-only
dry-run checks to classify prospective keys that are not in the current report; do not
infer their writability from absence. Only ownership/opacity refusals become skips.
Parse errors, invalid values, whole-note recovery, and transport errors block the
proposal.

Construct one `prop.set_yaml` batch for the displayed editable subset. Preserve existing
positions for replacements and append new keys in clipboard order. Preview the complete
final batch against the current fingerprint. Enter applies only that exact visible
preview, as one undoable transaction. When no keys are writable, show why and disable
apply. A stale report/role change invalidates the preview, retains the clipboard draft,
refreshes the listing, and requires explicit confirmation again; do not silently shrink
an already confirmed batch. Clipboard text changing externally after preview cannot
alter the proposal.

Test source-exact copying, protected copying, permission failures, mark/current
fallback, ordered new/replaced/skipped plans, new protected-name rejection, malformed
YAML, all-skipped input, race invalidation, cancellation, single-batch undo/redo, and no
live-document mutation during parsing or preview. Update the clipboard reference with
these visible behaviors.

Acceptance: copying emits source-based YAML and pasting always shows the exact overwrite
set before one guarded write; protected keys are never overwritten.

## Phase kind-change

Add `t` and a menu action for a loose property's conversion stage. Show current
kind/value, chosen destination, and converted preview. Use the existing typed parsers
for text, number, checkbox, date, datetime, list, tags, and aliases, and the YAML stage
for map/exotic shapes. Define a tested conversion table: safe scalar conversions, single
text value to a one-item list, and validated date/datetime tokens are supported;
ambiguous or destructive conversions need editable explicit input, not guessed values.
Preserve null/empty/false/zero distinctions, list order, and key-specific tags/aliases
behavior. Do not claim that a JSON string can force Bob to report a different inferred
kind when its wire contract cannot represent that distinction.

Note-local conversion uses `prop.set` with the supported `kind`, or `prop.set_yaml`
where appropriate, and the standard preview/apply/history flow. It never implicitly
changes the vault's property type. Display the warning when converting `created` to
datetime: Obsidian's datetime widget can strip offsets. Bob remains responsible for
preserving stored offset style.

Provide a separate **Change vault property type** action within the stage.
Feature-detect `app.metadataTypeManager` and the actual callable API in the installed
Obsidian; probe it in the disposable acceptance vault, not Bryan's live type catalog.
Encapsulate the supported call in a narrow adapter and test missing, throwing, and
asynchronous implementations. Compute the number of distinct vault notes containing the
key from available metadata APIs, with a cache-iteration fallback, and show the key,
proposed type, and count in an explicit confirmation line. Recheck the count and API
before the confirmed action. If a reliable count or usable type API is unavailable,
explain how to use All Properties instead and perform no global change. Do not write
`.obsidian/types.json` directly or guess an internal method signature.

Global type mutation is separate from note-local history: do not record a misleading
property undo token for it. Explain that `u` undoes note values, not the vault type
setting. Keep local conversion and global confirmation as separate submissions so a
partial failure is reported accurately, without automatic retry or rollback of unrelated
notes. Changing the vault type does not rewrite other notes' values.

Test conversion previews and invalid input, protected-row denial, actual op shapes,
unchanged vault type after local conversion, the created warning, precise use counts,
confirmation cancellation, unsupported managers, failed setters, and successful global
changes confined to the scratch vault. Update reference copy and the action/hint
inventory.

Acceptance: users can change a loose value's kind with a preview, and any vault-wide
type change is separately confirmed with its scope named.

## Phase acceptance

Extend the existing `scripts/obsidian-property-editor-e2e.cjs` family, splitting support
modules as necessary, and keep its offline wrapper registered in `npm test`. Extend
`scripts/fixtures/fake-bob.cjs` only as needed for offline protocol coverage; mocks are
not evidence of real UI acceptance.

Run the real harness with a binary built from the current bob-cli source, using its
existing `BOB_BIN` / `BOB_CLI_DIR` provenance gate and a disposable vault. Keep the
earlier core assertions. Add trusted keyboard-driven cases for rename, reorder with
comments and undo, mark/range delete plus undo, source-exact copy, overwrite paste with
new/replaced/skipped keys, filtering, kind conversion, YAML-map save and refusal, and
the supported/missing type manager branches. Probe the actual type-manager API and
record that result. Check disk bytes after writes and inverse operations; compare
untouched body and properties, not just the displayed values.

Add interaction checks for a filtered empty result, no reorder under an active filter,
native clipboard within inputs, the menu/stage/filter/modal close ladder, mixed
protected selections, two-pane unsaved edits, stale paste previews, and unload during
reads/previews. Record plugin and Bob provenance, writer invocations, assertions, and
console errors in the existing report. Keep clipboard mutation and global type changes
confined to the harness's scratch process/vault, restoring a pre-test clipboard if the
host exposes it.

Capture and inspect light/dark and narrow/wide views of marked rows, filter, rename,
YAML, paste preview, and conversion/global warning. Check keyboard focus, overflow,
selected-versus-marked contrast, and reduced-motion reorder. Reuse the current Obsidian
button-reset/layout guards. Do not claim the offline self-test as real-Obsidian
acceptance. Read the tailnet memory before using another machine; retain
reports/screenshots via `/sase_artifact`.

Complete `docs/property-editor.md` with the exact keymap and a parity table against
native Properties (add, edit, rename, delete, reorder, select, copy/paste, kinds, and
global types), plus the YAML-stage capability and protected/opaque limitations. Update
`docs/develop/validation.md` and affected hotkey references. Remove the paragraph that
defers these completed features.

Acceptance: the complete native in-document operation set is usable from the keyboard,
and the real scratch-vault run plus reviewed screenshots demonstrate the promised
behavior and refusal paths.

## Verification, deployment, and completion

Each implementation phase runs its meaningful focused tests, `npm run build`,
`npm run build:check`, `npm test`, and `npm run docs:check` for its changed docs, then
`bob plugins sync` from the opened linked checkout as required by its instructions.
Regenerate the documentation inventory when its source changes. Do not hand-edit
generated bundles or force over another checkout's deployed build. Follow
`docs/develop/deployment.md` if sync reports a protected build. Use `/sase_monitor` for
long checks or remote runs and wait for the staging command itself to exit before
finalizing.

The acceptance phase and child epic land agent verify the combined tree, all seven phase
results, deployed plugin state, and source-linked evidence. Before closing any assigned
phase, run `sase bead epic-symbols <assigned-id>` and resolve or re-key any remaining
symbols to a genuinely open owner. The land agent repeats
`sase bead epic-symbols bob-cli-pf.5` before closing the delegated phase with a note
naming the verified behaviors and evidence. No symbols were attached to `bob-cli-pf.5`
at planning time.

Phase workers never create task beads or close ancestors. Record discovered out-of-scope
work as `PROPOSED FOLLOW-UP:` notes on their assigned bead for land-agent triage. A
failure reproduced identically on the clean base is recorded with evidence and any
existing tracking bead, and does not strand a completed phase. Missing required parity
behavior or unrun acceptance remains unfinished work. Every turn ends through
`/sase_final` with its appropriate commit or staged-handoff declaration.
