---
tier: epic
title: Complete the core note property editor
goal: Any Markdown note supports guarded keyboard-first property editing, add/delete,
  undo/redo, ownership routing, and YAML return through the verified =+ editor.
phases:
- id: props-runtime
  title: Guarded property transport, history, and overlays
  size: medium
  depends_on: []
  description: 'props-runtime: implement the props v1 client, guarded session, bounded
    per-path undo/redo, and confirmed-write overlays with deterministic tests.'
- id: editor-shell
  title: Property modal, role routing, and discrete edits
  size: medium
  depends_on:
  - props-runtime
  description: 'editor-shell: build cache-first grouped rows, role routing, navigation
    and close ladder, checkbox/delete/history actions, and degraded states.'
- id: value-stages
  title: Shared typed stages, date-time, and tags
  size: medium
  depends_on:
  - editor-shell
  description: 'value-stages: reuse record helpers for typed edits, add datetime and
    tags, wire prose completion, and verify preview/apply races and record regressions.'
- id: add-flow
  title: Property name completion and positioned creation
  size: medium
  depends_on:
  - value-stages
  description: 'add-flow: implement positioned insertion, vault name/type/count completion,
    literal-name override, and atomic first-property creation through typed stages.'
- id: entrypoints-yaml
  title: Public entry points and YAML return
  size: medium
  depends_on:
  - add-flow
  description: 'entrypoints-yaml: wire the plus chord, palette and Command Mode, drawer
    actions, public API, species hand-offs, and the props YAML excursion with Done
    return.'
- id: acceptance-docs
  title: Real Obsidian acceptance and editor documentation
  size: medium
  depends_on:
  - entrypoints-yaml
  description: 'acceptance-docs: add real-Obsidian acceptance and offline self-tests,
    document the core editor, and record verified evidence for closing bob-cli-pf.4.'
proposed_by: bbugyi200.apollo.bob-cli-pf.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pf.4
create_time: 2026-11-22 23:01:39
status: done
bead_id: bob-cli-pf.4.1
---

- **PROMPT:** [prompts/202611/property_editor_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_editor_core.md)
- **PARENT:** [202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md)
- **BEAD:** [bob-cli-pf.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.4.1.md)

# Complete the core note property editor

Implement the `editor-core` scope of `bob-cli-pf.4` as a child epic. On any Markdown
note, `=+` opens a keyboard-first property editor. Loose properties are edited through
`bob props`; card properties hand off to their owning editor; identity and derived
properties explain their ownership. Add, delete, undo, redo, and returning from YAML
work without losing a draft or unrelated bytes.

This is a child of the assigned phase, not a replacement for the parent epic
`bob-cli-pf`. Approval automation creates the child work; do not manually create beads.
The child land agent completes and closes `bob-cli-pf.4` after verifying all six phases.
Phase workers close only their own assigned child phase. Nobody working this plan may
close `bob-cli-pf` or another ancestor plan bead.

## Scope and existing evidence

Read `plan:202611/card_owned_frontmatter.md` through `sase artifact read`. Its sections
4, 5, and 7 and its `editor-core` phase are authoritative. The assigned phase had no
previous implementation notes at planning time. Its prerequisites `bob-cli-pf.1` and
`bob-cli-pf.2` are closed; inspection found their code, but no property editor or
`props_v1` consumer in bob-ledger-tools.

The bob-cli prerequisite landed as `3e480e4a`
(`feat(props): add bob props show and update commands`). `docs/props.md`,
`src/native/props/{show,protocol,roles}.rs`, and `tests/cli/props.rs` specify the actual
wire contract. In bob-plugins, `175283d3`, `d3827858`, `c8deb534`, and `bb57281a`
include drawer entry points, documentation, real-app coverage, and follow-up fixes.
Recheck current history and the phase notes before implementing; preserve work that has
landed since.

All product implementation belongs in the linked `bob-plugins` repository. Open it
through `/sase_repo`, use the printed checkout, and read its AGENTS.md. Paths below are
relative to that repository unless explicitly labelled bob-cli. Edit fragments, register
their order in `src/fragments.json`, and regenerate `main.js` with `npm run build`. Each
hand-edited fragment must stay at or below 1000 lines. Use new `298…` property-editor
fragments and an exports fragment (the parent suggests
`378-exports-property-editor.js`); check name availability.

Confirmed reusable seams:

- `242aa-plugin-guarded-update-session.js`: `createGuardedUpdateSession`, direct apply,
  preview revision/generation guards, save-before-apply, and lock behavior. Do not grow
  this already large fragment.
- `242l`/`242m`: record transport and session patterns. Their record report decoder and
  single undo entry are not the props protocol or the required stack.
- `242n`, `242q`, `242r`, `242s`, `242sa`: record model, typed stages, save lifecycle,
  and prose integration. Extract genuinely shared helpers instead of copying a second
  record editor or subclassing its record-specific assumptions.
- `albumSchemaCardModalBase`, its regions layout, and `bobCardWireCloseKeys`: modal
  chassis, hint strip, All actions menu, and shared Esc aliases.
- `240n`/`240o`/`240p`/`240r`: source-ordered inventory, drawer DOM, state, and
  adapters. `propertyCardFootSetEditorHook` already accepts an opener;
  `propertyCardFootOpenEditor(path, key, mode)` calls it as `(path, {key, mode})`.
- `248m-plugin-schema-double-equals.js`:
  `registerPropertyCardSecondKeyArm({key: "+", canAct, run})` is implemented but
  unregistered. Preserve its existing capture gate and `==`/`=-` behavior.
- `242e-plugin-album-yaml-excursion.js` and `242pa-plugin-record-card-entrypoints.js`:
  leaf-bound source excursion and Done restoration. Unknown kinds currently normalize to
  album.

The parent's decisions are final: bank gets a foot row; drawer state is per note for the
session; only the two named card glossary strands may eventually change; no decision
record is authorized. This child changes no memory. Whole-container Properties takeover
and those glossary changes belong to `bob-cli-pf.6`. Rename, reorder, selection,
clipboard paste/copy, filtering, kind conversion, and the YAML **value stage** belong to
`bob-cli-pf.5`. Prepare extension seams for them, but do not ship inert controls
advertising unimplemented actions. YAML excursion/repair is part of this child;
map/opaque loose values retain that escape until the later YAML stage lands.

## Shared implementation contract

Use `openPropertyEditor({filePath, key, mode})` as the common entry, with
`mode: "browse" | "edit" | "add"`; expose the same options through the plugin API for
sibling plugins and later takeover. Resolve only existing Markdown files. An omitted
path means the active note; an explicit path always wins. Keys and paths survive report
reconciliation; duplicate rows use a source-line identity so they remain separately
visible. No frontmatter is written on open.

Keep a property-specific controller boundary with operations for reading, `setDraft`,
preview/apply, immediate operation batches, undo/redo, rebase, and close. Later parity
code must be able to submit a batch as one history entry. It must not reconstruct YAML,
use `processFrontMatter`, or write vault bytes in JavaScript. Preserve native Properties
behavior until the separate takeover.

Use bob's `note_kind`, property `kind`, `role`, `owner`, `editor`, `reason`, source
lines, and fingerprint as authority. Cache values may paint first but cannot authorize a
mutation before a validated current report. Keep null, empty string, empty list, false,
zero, and absence distinct. Do not pass UI-only stage names such as URL, Link, or
Suggest as unsupported wire kinds: those write strings with the supported props kinds.
Preserve exact inverse payloads, including optional `prop.restore.framing`, for
byte-exact undo.

All phases run their focused tests, `npm run build`, `npm test`, and `bob plugins sync`
after plugin changes. Run long commands through `/sase_monitor` with sufficient
continuation instructions. Follow the linked repo's deployment conflict procedure if
sync refuses an overwrite; do not force. Keep additions to manifests, test lists,
lifecycle, API mixins, and CSS additive. Do not alter Cargo versions, the changelog, or
the props protocol.

## Phase props-runtime

Build the property service and session before any public entry point exists.

1. Call `bob props show NOTE -f json` and `bob props update NOTE -f json -i - [-d]`
   through `bobSpawnJson`. Decode schema 1, top-level `properties`/`fingerprint`,
   boolean `changed`, `fingerprint_before/after`, `inverse_ops`, `changes`, and typed
   errors. Unlike record reports, invalid/duplicate YAML may have a null fingerprint.
   Accept recovery show reports for rendering, never for writes.
2. Detect `props_v1` from show capabilities. Recognized unknown-subcommand output from
   old bob means capability unavailable; do not mislabel every exit 2, I/O failure,
   malformed JSON, or unknown schema as an old binary. Keep these failures actionable
   and read-only.
3. Wrap `createGuardedUpdateSession` with `prop_stale` and `report.error`. Typed
   previews debounce 250 ms. Enter commits only the matching successful preview; edits,
   selection changes, IME composition, close, and stale responses invalidate any pending
   Enter. Discrete operations use direct apply. Save every pane showing the target
   before apply; changes during save must revalidate. Track changes for arbitrary notes,
   not just canonical albums.
4. Keep the draft on stale refusal, reread and rebase, and require a current preview
   before retry. Lock waiting/cancel and unknown apply outcomes follow the shared
   controller; never blindly repeat an uncertain write.
5. Add undo/redo stacks keyed by normalized path, session-only and bounded to 20 entries
   each. Entries contain the inverse batch and its expected fingerprint. Undo's returned
   inverse becomes redo; redo's inverse becomes undo. New successful edits clear redo;
   no-op, failed, or stale operations do not consume history. Reopening preserves
   history; unload clears it. Rename/delete events migrate or release state
   consistently. Historical expectations must never be silently replaced with the
   current fingerprint.
6. Publish confirmed post-image overlays to the editor, card, and drawer for at most
   five seconds, retiring them when matching source/cache arrives, superseding external
   changes arrive, or their lease expires. Preserve deletions, ordering, nulls, and
   empty values; do not use the record overlay's null-as-unset conversion. Rendering
   itself must not spawn bob or write.

Test protocol fixtures from the real CLI shape; malformed and recovery reports;
revision/generation/save races; body-only changes; late responses after close;
lock/refusal handling; per-path isolation; history bounds, branching, stale refusal,
framing payload round-trip, and overlay retirement. Acceptance: the tested controller is
usable without the modal and preserves the props writer's guards and inverse bytes.

## Phase editor-shell

Build a property-specific modal on the shared chassis. Mount it internally for tests,
leaving public entry-point registration to entrypoints-yaml.

- Paint from cached metadata and confirmed source inventory immediately, then reconcile
  a `props show` report without stealing focus or losing selection. Header: species
  badge/paper or neutral page tile; `<KIND> · PROPERTIES`; title; muted
  `path · N properties`. Handle every wire note kind, including quarter, year, all_time,
  and bank. Reserve an extension slot for pf.5's filter.
- Group source-ordered rows into Loose, On the kind's card, Identity, and Bob-managed.
  Keep source order within groups. Reuse value/link/tag rendering and Obsidian
  property-type Lucide icons; use accessible rows, selected state, screen-reader
  announcements, narrow layouts, and existing colors.
- Roles gate every action, not only Enter. Card rows dispatch a typed hand-off intent;
  identity/derived rows are locked with bob's reason/writer and YAML escape. Loose
  `created` is editable. Unresolved ownership cannot become loose.
- Browse keys: j/k, arrows, Ctrl+N/P, g/G, Home/End. Enter/l opens a typed stage or
  immediately flips a loose checkbox. Ctrl+D/Mod+Backspace unsets a loose property;
  u/Ctrl+Z and Ctrl+R/Ctrl+Shift+Z call history. Keep native text undo and typing inside
  inputs separate from browse shortcuts. Expose `=` and y/Ctrl+E as actions for the
  later integration phase.
- Use one close ladder through `bobCardWireCloseKeys`: menu, stage, reserved filter
  layer, modal. Esc, q, and Ctrl+[ share it even when focus strays. q types in text
  controls; Ctrl+[ never types. Unload closes sessions, subscriptions, timers, and
  listeners and ignores in-flight results.
- Show `Checking value…`, `✓ Saved · u undo`, or bob's refusal message and hint
  verbatim. Old bob renders cached rows with writes disabled and
  `Update bob to edit properties here — y opens YAML`. Invalid/unclosed/ duplicate YAML
  displays recovery diagnostics and a source-line target. Empty notes show
  `No properties yet — o adds the first`.

Test row identity/order and all empty values, role checks on every mutation, selection
reconciliation, keys and focused inputs, close layering, first paint before async
report, recovery states, and teardown. Acceptance: shell, checkbox/delete/history, and
explicit routing intents are correct and share no unsafe record-only assumptions.

## Phase value-stages

Connect typed stages to the runtime. Extract shared rendering/parsing helpers from
record fragments when necessary, preserving record behavior and tests.

- Text with multiline support and the navigation plugin's `[[`/`((` prose band; Suggest
  from existing values of that key across the vault; Number; Date; URL; Link; and
  ordered List/chips. Feature-detect sibling completion APIs and retain plain text
  editing if unavailable. Scalar list values must not be silently coerced or flattened.
  Exotic values route to YAML until pf.5.
- Add Date & time accepting `now`, `9:41pm`, `fri`, `-3d`, and valid ISO. Reuse existing
  date parsing where applicable and inject clock/time zone for deterministic tests.
  Relative tokens resolve in the viewer's local calendar; time-only uses today, day-only
  keeps an existing time (midnight for a new datetime), and ISO with an explicit offset
  retains its instant. Pass validated strings to bob, which preserves the existing
  separator and offset spelling. Date-only values remain date-only unless the chosen new
  property's type is datetime; no vault-wide type mutation occurs. Reject
  invalid/ambiguous input with a visible preview rather than silently rolling dates.
- Add Tags with vault completion and stored values without leading `#`. Aliases use an
  ordered string-list stage. Preserve intentional null/empty distinctions through
  staging and submit; do not treat an empty input as deletion without an explicit delete
  action.
- Register the next unused prose surface id in `scripts/fixtures/prose-fields.json` (S16
  was free when inspected), covering Text and List stages, following the record
  surface's subject policy.
- Preserve caret/selection and composition state when repainting suggestions or
  previews. Cancel returns to browse without writing. Stage Enter obeys the session's
  current preview and guards against double commits.

Test typed operation construction, supported wire kinds, date tokens and offset styles,
tags, list/null/empty values, completion fallbacks, cancel, IME/caret retention, and
Enter-before-preview/changed-draft races. Run the existing record stage, prose, and
guarded-session suites after extraction. Acceptance: all core scalar and supported list
kinds are editable through `prop.set`; only intentionally deferred exotic/YAML values
use excursion.

## Phase add-flow

Wire o below, O above, and Mod+; at the end **inside this modal**. Wrapping Obsidian's
global Add file property command is pf.6's work.

The name stage uses feature-detected `metadataCache.getAllPropertyInfos()` for property
names, types, and usage counts. Fall back to a cache-derived name inventory/manual text
entry without inventing authoritative type information. Keep completion ordering stable.
Tab/Enter chooses the completion's type and opens its value stage; Shift+Enter keeps the
exact typed name as a new text key. Reject empty, duplicate, or invalid keys locally,
then let bob validate the full operation and prospective ownership. Accept valid
quoted/Unicode/numeric-looking names.

Store insertion anchors by source key rather than visual group index. New entries use
one `prop.set` with before/after, or no anchor for end. Do not create a placeholder key
before its value is accepted. Name/value cancellation writes nothing. If an anchor
disappears or a name becomes duplicate during editing, keep the draft and present the
current refusal rather than changing the destination. The first property uses the
empty-mapping fingerprint; undo removes newly created framing but keeps a pre-existing
empty block.

Test each placement across groups, unknown and known names/types/counts, API absence,
Shift+Enter override, unsafe/protected prospective names, duplicate and anchor races,
and no-frontmatter create/undo/redo. Acceptance: the full two-stage add flow is atomic
and usable on notes without cards or properties.

## Phase entrypoints-yaml

Register the common opener and clean up every registration on unload:

- `+` in the existing second-key arm table for any Markdown note. Preserve
  normal-mode/Live-Preview/count/register/buffer/IME/repeat/modifier guards and accept
  Shift for `+`. Cancel Vim's pending operator only when the arm acts.
- Palette `edit-note-properties`, name `Edit note properties`, plus the
  `CM_SHEET_SURFACE_ROWS` entry in bob-command-mode. Reading/non-Vim users use these and
  clicks; add no Reading-view prefix timer.
- The existing drawer opener hook: key-label click and Enter select/open that key; value
  links keep navigating. Add `＋ Add property` and `✎ Edit all =+` only while the hook
  is registered. Refresh already-open drawers on register and unregister. Keep per-note
  drawer state and bank behavior intact.
- Public plugin API exposing `openPropertyEditor` and its add/key options. Return useful
  failure results for missing/unloaded/non-Markdown targets.

Card-row Enter and browse `=` use the owning species APIs: `openRecordCard`,
`openIntentionCard`, `openAlbumSchemaEntry`, and `openBankSetupCard`, passing a field
where supported. On an ordinary note with no species editor, keep the property modal
open and explain the missing route. Do not send protected writes through props or
silently discard an uncommitted typed draft when an unrelated routing key is typed in a
text field.

Extend the existing YAML excursion with `kind: "props"`, its note path, originating
leaf/view state, and selected key. y/Ctrl+E switches to source at the reported key or
parse-error line and installs the floating Done button. Done saves all target panes,
restores that leaf's original mode, rereads props, and reopens on the same surviving key
(nearest row otherwise). Damaged YAML can reopen in recovery mode. Rename, deletion,
leaf switch/close, unload, and late callbacks must not reopen the wrong note. Preserve
album/record excursion behavior and never write through the excursion itself.

Test the real chord action table (including shifted +), command availability, drawer
link-vs-label actions, API targeting, field hand-offs, and excursion
save/restore/failure/cleanup. Acceptance: every documented core entry point reaches the
same working editor, including ordinary and invalid notes.

## Phase acceptance-docs

Add `scripts/obsidian-property-editor-e2e.cjs` with helper modules kept small, and an
offline harness self-test registered in `test:unit`. Reuse the isolated scratch
vault/profile/CDP conventions in `obsidian-card-foot-e2e.cjs` and
`obsidian-record-card-e2e.cjs`. Extend `scripts/fixtures/fake-bob.cjs` for deterministic
offline cases; fake-bob success does not count as real-app proof.

Build bob from the current bob-cli source containing props and pass explicit `BOB_BIN`
and `BOB_CLI_DIR`, recording the source SHA and binary provenance as the existing
harness does. Resolve the Cargo target directory rather than assuming its path. Use
scratch fixtures, never Bryan's personal vault. The real run must cover:

- edit/add/delete/undo/redo on project and day notes, with persisted disk assertions and
  unchanged key/comment/body bytes;
- a plain note with no frontmatter and complete first-add undo;
- record field hand-off and a derived/identity refusal with YAML access;
- old-bob read-only, invalid/unclosed/duplicate YAML recovery;
- checkbox, datetime/offset, tags, prose/list editing, and name completion;
- two panes showing a note, unsaved editor saves, stale drafts, reopening history, YAML
  Done restoring mode/key, close keys, and unload cleanup;
- `=+`, palette, Command Mode, drawer row/add/all, and sibling API paths;
- light/dark by narrow/wide screenshots, reviewed for clipping, focus, and
  selected/locked row legibility.

Read docs memory before documentation changes. Add `docs/property-editor.md` with the
core keymap, role routing, write/status behavior, history limit, degraded modes, and
YAML escape; state which parity actions arrive in pf.5. Cross-link it from
`docs/record-card.md`, `docs/intention-card.md`, `docs/albums.md`,
`docs/bob-ledger-tools.md`, and `docs/bob-navigation-hotkeys.md` where relevant. Update
validation instructions and generated inventory/navigation through the repository's
normal tools. Do not claim native Properties are hidden yet.

Run `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`, the offline
self-test and real-app suite, then `bob plugins sync`. Record exact commands,
source/binary provenance, screenshots/results as audited artifacts, and material
limitations in the phase note. Acceptance: the app behavior and disk bytes substantiate
every core requirement; docs describe the shipped scope.

## Land and completion

The child land agent reads every phase's evidence, resolves integration conflicts, and
verifies the complete pf.4 acceptance without redoing unchanged passing tests. Keep
`bob-cli-pf.5`'s extension seams clear: operation batches, row selection, stage
registration, filter slot, and path-scoped history.

Before closing any assigned child phase, run `sase bead epic-symbols` for that phase and
resolve entries or re-key them to a still-open relevant bead. Before closing
`bob-cli-pf.4`, run `sase bead epic-symbols bob-cli-pf.4` and do the same. Record the
completed coverage and remaining pf.5/pf.6 boundary, then close only `bob-cli-pf.4` with
`sase bead close bob-cli-pf.4 --note "<verified evidence>"`. The child epic's own
lifecycle is handled by its normal land workflow; this does not authorize closing the
parent epic `bob-cli-pf`.

Never set claimed/in_progress status manually or reopen an assigned phase. Record
discoveries as `PROPOSED FOLLOW-UP:` notes on the assigned phase; create no task beads.
A check failure reproduced identically on the clean base is recorded with that evidence
and any existing tracker id; it does not keep an otherwise completed phase open. End
every root turn through `/sase_final`.
