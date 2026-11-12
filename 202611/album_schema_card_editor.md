---
tier: epic
title: Editable album Schema Card
goal:
  Complete bob-cli-kr.4 with previewed per-gesture schema edits, guarded Undo, album and
  record entry points, validated YAML return, and deployed integration proof.
phases:
  - id: update-session
    title: Guarded schema update service and edit session
    size: medium
    depends_on: []
    description:
      "update-session: extend the shared album subprocess service with typed update
      results and a tested preview/apply/Undo session, editor-save ordering, and
      sync-only recovery."
  - id: editable-card
    title: Schema Card gesture stages and consequence UI
    size: medium
    depends_on:
      - update-session
    description:
      "editable-card: connect the existing Schema Card to the update session, implement
      every v1 gesture and typed field editor, and preserve focus, accessibility, and
      mobile degradation."
  - id: schema-entrypoints
    title: Album and record entry points and YAML return
    size: medium
    depends_on:
      - editable-card
    description:
      "schema-entrypoints: add palette and Command Mode surface entry points, the
      album-only Task Card chord fallback, and a pane-scoped YAML excursion with
      validated Done."
  - id: editor-acceptance
    title: Schema editor integration and real Obsidian acceptance
    size: medium
    depends_on:
      - schema-entrypoints
    description:
      "editor-acceptance: extend the existing scratch-vault album harness to prove real
      writer preview/apply/Undo, stale-save refusal, source validation, and keyboard and
      visual behavior; finish reference docs and deployment evidence."
proposed_by: bbugyi200.apollo.bob-cli-kr.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kr.4
create_time: 2026-11-12 13:46:47
status: wip
---

- **PROMPT:**
  [prompts/202611/album_schema_card_editor.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_schema_card_editor.md)
- **PARENT:**
  [202611/album_schema_render_and_edit.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_schema_render_and_edit.md)

# Editable album Schema Card

## Scope and ownership

Complete **bob-cli-kr.4** by turning its landed read-only Schema Card into the
keyboard-first editor defined by `plan:202611/album_schema_render_and_edit.md`, section
`card-editor`. This is a child epic under that assigned phase, not a replacement for the
outer album-schema epic. No implementation files have changed during planning. The
assigned phase had no earlier work notes or unfinished implementation when read; its
writer and header dependencies have landed.

Four serial medium phases separate the subprocess/session correctness work, the gesture
UI, cross-plugin entry points and source lifecycle, and executable proof. They share
fragments, generated bundles, and deployment state, so their ordering is intentional.
Each phase must be directly implementable by one coding agent. The child epic's land
agent integrates these deliveries and completes **bob-cli-kr.4** through its prescribed
completion workflow; phase workers close only their assigned child phase. Nobody here
closes **bob-cli-kr**, **bob-cli-k3**, or their unrelated descendants. Do not manually
change the assigned bead's status.

Final inherited decisions, with no new choices to approve:

- One confirmed gesture is one `bob albums update` envelope and one scoped commit.
- Mobile retains read-only cached summary/chips and the YAML hint; no editor or
  subprocess write affordance is exposed there.
- Ctrl+Shift+P opens the Schema Card on an album only where the Task Card has no target.
  Actual tasks and their existing property routes retain priority.
- No memory file may change, including `decisions:album-schemas-edited-through-bob`.

Out of scope: a YAML parser/writer or schema validator in JavaScript, any second schema
store, per-record value editing, key renames/merges/migrations, changes to existing
field kind/list shape or noun, new CLI options, release/version changes, and revisions
to the outer epic's decisions. The existing writer supplies refusals and the source-mode
alternative for unsupported changes.

## Evidence to reuse

Read these artifacts with `sase artifact read`, never directly from a sidecar:

- `plan:202611/album_schema_render_and_edit.md`: governing contract and decisions.
- `research:202611/album_record_schema_ux/album_record_schema_ux.md`: normative
  recommendation where the governing plan does not override it.
- `plan:202611/album_and_record_notes.md`: album/record ownership and schema contract.
- `plan:202611/album_plugin_ui.md`: shared report service and presentation lifecycle.

The research artifact initially failed to resolve in the planning checkout; opening
`bob-cli--research` with `/sase_repo` refreshed it and the audited read succeeded. Use
the same recovery if a successor sees a stale sidecar.

Inspected bob-cli base: `d71a3bf9`, including public writer commit `859c1571`.
`src/native/albums/update/{protocol,ops,plan,spans,run}.rs` and `docs/records.md`
already provide the nine v1 operations, semantic fingerprint, typed refusals,
consequences, inverse operations, scoped commit, and post-sync warning. Do not rebuild
this backend.

Inspected bob-plugins base: `e0053cd`, including `f714fee` (header reshape).
`bob-cli-k3.7.1` is closed, verified through `sase bead read`, so its plugin fragments
are available to extend. `bob-cli-kr.3` is also closed with 13 passing header tests.
Recheck current source and assigned-bead notes on every implementation launch.

Relevant bob-plugins paths:

- `plugins/bob-ledger-tools/src/241l-plugin-album-data.js` and
  `241m-plugin-album-data-service.js`: list/sync transport, report generation,
  subscriptions, debounce, and teardown.
- `236-plugin-stats-data.js`: `bobSpawnJson` uses `execFile`, timeout, maxBuffer,
  `BOB_DIR`, and JSON decoding even on numeric nonzero exits. It currently ignores the
  returned child handle and has no stdin delivery. Decoder failures retain only generic
  error fields, so update refusals need deliberate structured handling.
- `241u-plugin-album-header-model.js`: the presentation projection currently drops
  `schema_fingerprint` and field `default`; it must not be the edit model as-is.
- `241v-plugin-album-header.js` and `241w`/`241x` schema model/DOM fragments: source
  identity, existing exact-line YAML jump, header lifecycle, and chip tokens.
- `241y-plugin-album-schema-card.js`: the existing read-only modal, with selected field
  index and navigation/YAML keys. Extend it rather than adding another card.
- `plugins/bob-command-mode/src/160-sheet-rank.js`: `CM_SHEET_SURFACE_ROWS` and
  `cmSheetSurfaceRows`; `170-sheet.js` executes these through the registered Obsidian
  command. `080-api.js` is a CLI runner API, not a surface-registration API.
- `plugins/bob-command-mode/src/090-plugin.js`: `commandModeSaveAllMarkdown` is the
  precedent for awaiting Markdown view saves before invoking a writer.
- `plugins/bob-navigation-hotkeys/src/490-plugin-lifecycle.js` registers
  `set-bullet-property`; `550-plugin-cancel-and-counted-property.js` resolves its Task
  Card targets. Route the existing command, not competing global key handlers.
- `scripts/test-ledger-tools-albums-data.cjs`, `test-ledger-tools-album-header.cjs`, and
  the navigation/Command Mode fixtures are the unit-test precedents.
- `scripts/obsidian-album-ui-e2e.cjs`, its `-lib.cjs`, and
  `test-obsidian-album-ui-e2e.cjs` supply the scratch-vault/CDP and offline harness.
- `docs/albums.md` owns album behavior; `docs/develop/deployment.md` owns rollout.

## Shared implementation contract

Open bob-plugins through `/sase_repo` and read its `AGENTS.md` before editing. All paths
below are repository-relative. Edit source fragments and their `src/fragments.json`
ordering, then generate `main.js`; no hand-edited bundle. Keep every authored fragment
at or below 1000 lines, extracting cohesive session, stage, and source-excursion
fragments as needed. Wire mixins through the existing class assembly and expose test
helpers through the album export fragment. Add every new test to the explicit
`package.json` test list.

The model contains album path/stem, semantic fingerprint, report generation, stable
field keys/choice values, typed values with property-presence information, and
server-supplied counts. The Rust report is authoritative. Preserve `false`, `0`, empty
string/list, and absence separately. Choice labels can already include emoji; preserve
the writer's documented split/combine convention, and send only the component being
changed to avoid stripping an existing icon accidentally.

The wire request is `{schema_version: 1, expect: <fingerprint>, ops: [...]}` through
stdin to `bob albums update STEM -f json -i -`, adding `-d` for preview. Labels,
defaults, and inverse metadata never become shell text. Positions are zero-based and
moves index the list after removing the selected item. A choice-kind field addition
includes its required nonempty initial choices in the same gesture. Added fields are
optional; making them required is a later explicit gesture.

An update result has `schema_version: 1`, `command: "update"`, `ok`, `dry_run`,
`stem/path`, `changed`, before/after fingerprints, consequence, per-op counts, derived
keys, diffs, inverse ops, commit, sync, warnings, and nullable typed error. Exit 2 can
be a valid refusal document; exit 1 may also carry a typed failure. Transport/decode
success and `report.ok` must be distinct. Unsupported versions, malformed output,
missing backend, and timeout disable Apply with an actionable message; never treat
failure as an empty successful schema or preview.

Keep schema validation, diff generation, usage accounting, key derivation, and refusal
decisions in bob. UI input typing and request construction are not a second schema
validator. All report/error/user-supplied text is inserted as text.

## update-session

Build the shared mutation path and a DOM-independent, dependency-injected session
controller. The visible card remains read-only until the next phase wires it.

1. Add update argv and response helpers alongside `241l`/`241m`, extending the existing
   spawn abstraction with optional JSON stdin delivery/closure and safe handling of
   stdin errors. Keep existing list/stats/other callers compatible. Decode valid refusal
   envelopes without losing `error.affected`, hint, path/line, or operation identity;
   validate wire shape, not schema semantics. Use a writer timeout appropriate for the
   existing guarded sync path rather than blindly inheriting the list timeout, and
   retain bounded output and no-shell execution.
2. Expose preview, apply, and sync-only retry through the shared album service.
   Serialize applies across cards and coordinate them with album auto-sync: no
   overlapping apply/sync; no duplicate sync/commit caused by this writer's own metadata
   events. Preserve genuinely pending unrelated invalidations. Refresh after a write
   with the existing generation-aware mechanism, ensuring an older in-flight list cannot
   become the post-write report. Preserve stale last-good data on refresh failure, but
   keep editing disabled until a fresh report arrives.
3. Session states distinguish browsing, draft, preview-pending, preview-ready, applying,
   refused/stale, and saved-with-sync-warning. A debounce (around 250 ms) submits one
   preview after typing settles; navigation and reopening are reads. Key every response
   by session epoch, draft revision, album identity, fingerprint, and relevant report
   generation. Ignore superseded responses. A change to the draft or relevant vault
   generation invalidates its Apply eligibility immediately.
4. Before apply await supported saves of open Markdown editors, including all panes
   showing the album. A save failure refuses apply; missing required save support is
   taught, not silently assumed successful. Retain the original `expect` through save
   and apply so a Source edit cannot be silently rebased. If saving invalidates the
   preview, show a new preview or a stale refusal and require confirmation again. Double
   Enter and multiple entry points share one apply guard.
5. Apply only the exact envelope whose latest dry run succeeded. On success return fresh
   model/selection identity, changed/no-op status, and Undo availability. Preserve
   opaque `inverse_ops`, including restoration metadata, with the returned
   `after_fingerprint`; never reconstruct inverse YAML or use git reset/revert. `u`
   stages the inverse on the same preview-and-Enter path. Keep at least the last
   successful gesture undoable; a no-op must not overwrite it. External changes or newly
   used values can refuse Undo through the same backend checks.
6. For `stale_schema`/`stale_note`, refresh and expose the changed-schema explanation;
   retain the draft as unconfirmed and never automatically retry its mutation. For
   successful local commit plus failed post-sync, keep the committed result and Undo and
   offer **Retry sync** using `bob vault-sync` only, never `albums sync` or replay of
   the mutation. Unknown outcomes after a timeout require refresh and explicit
   reconciliation, not blind retry. Closing/unloading cancels pending preview
   timers/subscriptions and ignores late UI work without claiming to cancel an
   already-running committed transaction.
7. Add focused service/session tests with injected process and timer fakes: stdin
   fidelity including shell-looking labels, numeric exit/refusal decoding, timeout,
   duplicate apply, auto-sync coordination, generation races, save ordering/failure,
   stale preview suppression, no-op, inverse metadata replay, and sync-only retry.
   Assert cancelled/refused sessions never invoke apply or a plugin vault writer.

## editable-card

Connect the existing header opener/modal to the session, carrying raw report fields
without losing typed defaults or the fingerprint. Keep counts and chips from the
existing report-backed renderer.

1. Make album stats, field rows, individual choice chips, and Add field keyboard targets
   identified by stable keys, not transient array indices. Up/down moves rows;
   left/right or Tab reaches ordered choices and explicit controls. Keep selection/focus
   through previews and successful repaints; if a selected item is removed, select its
   nearest remaining neighbor.
2. Implement the complete gesture set with an explicit edit stage and consequence: Enter
   edits the selected label; stats edit label/plural/dated_notes; `a` adds an optional
   field with kind defaulting to text, list/suggest controls, and initial choices where
   needed; `c` adds a label/emoji after the selected choice; `!` and `s` stage
   required/suggest toggles; Alt+J/K stage field or choice moves; Ctrl+D stages removal;
   `y` jumps to Source; `u` previews Undo; Escape cancels a stage or closes from
   browsing. None of these implicitly bypasses the consequence step.
3. Field editing exposes label, default, and numeric bounds with typed controls and
   explicit Unset actions. Cover the existing kinds, including boolean and list
   defaults; no `value || default` coercions. Choice editing changes label and/or emoji
   while stored value stays fixed. Add stages show bob's derived keys before creation.
   Schema constraints, including status defaults and cross-album clashes, remain backend
   refusals with useful hints.
4. Paint bob's consequence and affected/newly-invalid counts before Enter can apply.
   Pending/error/older responses cannot enable it. Surface forbidden noun/kind/key edits
   as read-only identities with the YAML/migration route, not editable inputs that imply
   support. Used removals still reach dry run to teach the full-count refusal. Keep the
   card open after success with a visible Undo action and toast; Undo previews before
   Enter. Handle stale refresh and saved-local sync retry.
5. Use one key dispatch path per event; typing in inputs must not trigger navigation
   letters or toggles, and IME composition must not apply. Support visible button
   equivalents, accessible names, required labels, busy/disabled states, and an
   appropriate live region for consequences/refusals. Reuse owner hue, Obsidian theme
   variables, tabular counts, and wrapping around 375 px. Retain mobile read-only
   behavior and all scoped Properties restoration fixtures.
6. Add DOM/editor tests for every gesture/envelope mapping, typed absence vs
   empty/false/zero, choice insertion/move order, empty and invalid albums,
   keyboard-only use, escaped text, focus retention, no double-dispatch, stale and
   refused stages, and no Apply before successful current preview. Update
   `docs/albums.md` for the now-working card without claiming the later entry points.

## schema-entrypoints

1. Register **Edit album schema** in Ledger Tools with one shared album resolver. Direct
   album paths resolve through the report; record notes resolve their actual parent link
   through Obsidian metadata and then a report album identity. Handle
   scalar/list/wikilink forms and legacy record types through existing helpers. Missing,
   cyclic, ambiguous, or non-album parents teach the failure rather than guessing a
   directory/title. Header and palette use the same opener; no extra subprocess service.
   Opening an invalid album still shows lint and the YAML path.
2. Add the registered command to Command Mode's `CM_SHEET_SURFACE_ROWS` in the Vault
   group. Use existing availability checks and `runSurface`; keep the CLI catalog's
   intentional exclusion of raw `albums update` intact. No new Rust CLI command or
   alternate stdin runner is needed. Test palette and sheet from album, record,
   unrelated note, missing plugin, and mobile contexts.
3. Add the album fallback at the existing property-command target decision after
   task/Task Link/dependency/relation targeting has had priority. Preserve counts,
   active-picker guards, Vim mappings, and existing targets on other notes. Never
   swallow a recognized Task Card target merely because its own operation refuses. Test
   a real task row in an album, its managed child lines, a non-task album row, and
   unrelated notes. Avoid a second unconditional Ctrl+Shift+P registration.
4. Extend `albumHeaderJumpYaml` with a per-leaf excursion record (album path, original
   view state, selected key, session identity), not a global source-mode lock. Render a
   small accessible **Done** affordance for that album's Source surface; Source mode
   lacks the regular header, so this must have an explicit mount/cleanup lifecycle
   outside the note text. `y` still closes the card and focuses the exact field/issue
   line.
5. Done awaits editor save, forces and awaits a current album report, and only returns
   to the prior view/reopens the card when bob reports a valid schema. Schema errors
   keep Source mode and focus/paint the diagnostic line with message and suggestions.
   Unavailable/stale reports cannot approve Done. Non-schema warnings alone must not
   trap the user. Users can freely navigate elsewhere; changing file, closing the leaf,
   or unloading clears the excursion and its DOM. No automatic schema repair or Markdown
   edits. Test two panes, failed save, deleted/type-changed album, warning-only report,
   repair then Done, and teardown.
6. Update the album and relevant Navigation Hotkeys/Command Mode references, and
   regenerate command inventory. Keep the inherited mobile degradation unchanged.

## editor-acceptance

Extend the existing album scratch-vault/CDP harness rather than using the personal vault
for test edits. Use the actual bob writer with an isolated Git-backed vault and the
established test remote/sync setup. Add offline tests to `npm test`; they must not
invoke a live personal vault or silently stand in for real Obsidian proof.

Acceptance scenarios:

- Healthy grouped album, empty schema, invalid schema, an unrelated `record` property,
  record-note parent entry, and source-mode error/repair fixtures.
- Preview a new choice, inspect consequence/derived value, Enter once, and verify one
  scoped commit and rebuilt index. Relabel retains stored keys and record bytes. Undo
  previews and commits inverse ops, restoring schema/index bytes and preserving linear
  history. No-op/cancel/refusal leave vault bytes and commit count unchanged.
- Used removal reports complete counts, and opening/typing never writes. Real
  boolean/numeric/empty/unset edits preserve typed semantics through refresh.
- A two-pane delayed Source save between open/preview and Enter produces a stale
  refusal. No refreshed fingerprint silently blesses the old draft. Double Enter and
  repeat preview cannot duplicate a mutation.
- Post-sync failure keeps local success; retry invokes only vault sync and creates no
  second schema edit. Use a controlled temporary remote/failure in the fixture.
- Keyboard-only entry/edit/Undo/YAML/Done from album and record, Task Card priority on a
  task within an album, blocked Done with invalid YAML then successful repair.
- Inspect captured light/dark screenshots at approximately 375 px: choices, typed
  stages, consequences/refusals, wrapping, focus after repaint, and accessible control
  labels. Preserve baseline header/mobile/Properties cleanup tests.

Keep additions scoped to editor acceptance; the outer epic still owns its broader
Properties serializer round-trip proof and final cross-phase audit. Fix integration
defects found by this phase, and finish `docs/albums.md` to describe all entry points,
keymap, typed Unset, dry-run gating, Undo confirmation, stale recovery, sync-only retry,
YAML Done, and mobile behavior.

## Verification, deployment, and completion

Each implementation phase runs focused tests, `npm run build`, `npm test`,
`npm run validate`, and `BOB_CLI_DIR=<resolved primary checkout> npm run docs:check`
from the opened plugin repo. Run `npm run docs:gen` when command inventory changes. Read
`docs.md` through `/sase_memory_read` before documentation edits. Use `/sase_monitor`
for long checks, waiting for the handoff command itself to exit and finishing through
`/sase_final`; never end with an ordinary subprocess still running.

After any plugin changes, run `bob plugins sync --repo "$PWD" --no-pull` from the opened
linked checkout and follow `docs/develop/deployment.md` for mandatory smoke and
post-sync deployment verification. A conflicting deployment follows that doc's
fetch/ancestry recovery; no unapproved force overwrite. Do not claim skipped or
unavailable real-Obsidian checks passed. Retain exact commands and evidence refs.

Earlier header notes mention task-tag-marks (`bob-cli-kw`), timing flakes
(`bob-cli-ki`/`bob-cli-ju`), a docs anchor (`bob-cli-kk`), and missing Chromium. These
are starting leads, not blanket waivers: fix host prerequisites and reproduce any
surviving non-environment failure on the unchanged base before classifying it. Record
discoveries only with `sase bead note <own-phase> 'PROPOSED FOLLOW-UP: ...'`; never
create task beads. A check failure identical on clean base is recorded with evidence and
existing tracker where relevant, and does not strand a completed phase.

The child land agent reviews every promised gesture and the real-Obsidian evidence, runs
final plugin gates and the primary bob-cli `just check` gate, verifies deployment, and
records implemented behavior, exact results, and artifact references. It checks
`sase bead epic-symbols` for each bead it is authorized to complete, including
**bob-cli-kr.4**, resolving entries or re-keying them to a still-open owning bead. It
completes this child epic and **bob-cli-kr.4** only when implementation and proof are
complete; its note supplies evidence to the outer land agent and never closes
**bob-cli-kr**. Use `sase bead close bob-cli-kr.4 --note "<what was verified>"` through
the applicable assigned-bead completion workflow. No ancestor-close shortcut or force
close. Every root turn ends through `/sase_final`, with all edited repositories declared
and intermediate work keeping its assigned bead open.
