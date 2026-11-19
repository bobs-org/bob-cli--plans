---
tier: epic
title: Extract the shared property-card kit
goal:
  Album, record, and bank cards share property ownership, editor registration, host
  lifecycle, and chrome while preserving their behavior and appearance; the verified and
  deployed kit unblocks the intention-card phase.
phases:
  - id: ownership
    title: Shared property ownership and card editor registry
    size: medium
    depends_on: []
    description:
      "ownership: capture the unchanged album, record, and bank acceptance baseline,
      generalize selective property hiding into propertyCardOwnKeys, and replace the
      closed double-equals dispatch with registerPropertyCard. Migrate all three cards,
      preserving existing assertions, gates, command identities, and editor behavior."
  - id: host
    title: Shared Reading and Live Preview host lifecycle
    size: medium
    depends_on:
      - ownership
    description:
      "host: extract createPropertyCardHost and move album, record, and bank lifecycle
      plumbing onto it. Preserve species models, anchors, caches, record-walk
      integration, Source-mode bank lint, embed display behavior, and cleanup."
  - id: chrome
    title: Shared property-card chrome and precise documentation
    size: medium
    depends_on:
      - host
    description:
      "chrome: consolidate property-card chrome and hue tokens without changing pixels,
      align the three plugin references around card versus card editor, and document the
      extension API for the intention-card phase. Compare final screenshots to the
      original baseline, run the complete acceptance gates, and deploy with bob plugins
      sync. Prepare completion evidence for the land agent; do not close bob-cli-nh.6 or
      bob-cli-nh from a child phase."
proposed_by: bbugyi200.athena.bob-cli-nh.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-nh.6
create_time: 2026-11-19 09:44:33
status: wip
---

- **PROMPT:**
  [prompts/202611/property_card_kit.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_card_kit.md)
- **PARENT:**
  [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)

# Plan: Extract the property-card kit without changing the existing cards

Complete the `kit` scope of **bob-cli-nh.6**, from `plan:202611/period_intentions.md`,
so the following intention-card phase can use shared property ownership, editor
dispatch, host lifecycle, and chrome. This is a child epic of the assigned phase, not a
replacement for the intentions epic.

The implementation is in the linked **bob-plugins** repository. Each worker must open it
with `sase repo open bob-plugins -r "Implement the property-card kit phase"` and use the
returned checkout. Read its `AGENTS.md`, `docs/develop/architecture.md`, and
`docs/develop/validation.md`. Paths below are relative to bob-plugins unless explicitly
identified as bob-cli paths.

## Evidence and sizing

The phase was in progress with no prior-work notes when read. No implementation of
`propertyCardOwnKeys`, `registerPropertyCard`, `createPropertyCardHost`, or
`.bob-property-card` was present in the inspected checkout. The source base was
bob-plugins `36d4801` and bob-cli `5b4c2636`; recheck history before changing code,
because other phases can land while this proposal is reviewed.

The accepted research was read through
`sase artifact read research:202611/period_intentions_card/period_intentions_card.md`.
Its initially missing local copy became available after opening the research sidecar.
The parent plan remains the authority for this phase's exact scope.

Three hosts currently occupy 2,501 lines: `241v-plugin-album-header.js` (620),
`242k-plugin-record-sheet.js` (928), and `248c-plugin-bank-setup-hosts.js` (953). They
share mounting machinery but have distinct behaviors that must survive: record
decoration caching, frontmatter-only fallback mounts, subscription ownership, the record
walk rail, and bank Source-mode lint. Property ownership and the registry form a bounded
first change; lifecycle migration is a separate bounded refactor; CSS and documentation
finish with visual acceptance. All phases are medium-sized direct implementation work.
Dependencies are serial because they edit the same fragment manifest, host integration
points, generated bundle, and styles.

## Scope and fixed decisions

- Keep existing card models, data services, writers, editor stages, palette names,
  command IDs, and documentation file URLs. No new production card or CLI command.
- Preserve bank → album → canonical record matching order, every existing Vim refusal,
  the pending `=` cancellation, and single event consumption.
- Property ownership is desktop-only, Reading or Live Preview only, and restricted to
  the exact successfully rendered card's native Properties container. Source, embeds,
  hover popovers, failed mounts, and teardown must expose native fields.
- Preserve current display behavior separately from hiding. The record test
  `Reading paints one owned sheet, ignores nested embeds, and cleans up on unload`
  explicitly expects a rendered embedded record sheet with **no hide ownership**. Do not
  delete that display or weaken the assertion to implement the general embed exclusion.
  An adapter controls whether its display may render in an embed; property ownership
  never may. A new host defaults to excluding embeds.
- Parent DECISIONS are final: `day_summary_cap=sixty`, both specified glossary decisions
  accepted, and decision-record writes declined. This child epic needs **no memory
  edits**; the parent vocabulary phase owns its authorized glossary changes. Do not add
  or edit either declined decision record.
- The future intention-card UI, fields, update protocol, YAML excursion kind, commands,
  and navigation fallback belong to bob-cli-nh.7. Provide its extension points without
  implementing its features here.

## Common implementation and verification contract

Edit `plugins/bob-ledger-tools/src/` and its `fragments.json`, never its generated
`main.js` directly. Fragments share a script scope and each hand-edited fragment must
remain at most 1,000 lines. Install new mixins through the existing duplicate-safe
installer, export test helpers through the established exports, and include new test
files in the explicit `package.json` `test:unit` list.

Retain existing helper and plugin method names as thin adapters where existing tests or
other plugin components use them. The common implementation must own the duplicated
behavior; do not leave three copied implementations behind the facade. Existing
assertions stay unchanged. In particular the selective Properties tests assert current
record classes and exact bounded selectors: keep compatible aliases and facade methods
while adding shared APIs. New tests prove the generalization and failure behavior rather
than repeating implementation details.

For each phase: build generated sources, run focused tests, then `npm test` and
`npm run validate` before landing. Run `npm run docs:gen` if registered inventory facts
changed, followed by the checks. Keep implementation checkpoints independently green in
the parent's order: ownership, registry, host, chrome, docs. Use host finalization for
commits; do not make manual Git commits outside the authorized SASE commit workflow. Run
`bob plugins sync` after changes as the linked repo requires; follow its deployment
conflict recovery instructions without forcing an overwrite.

Long builds and Obsidian suites must use `/sase_monitor`; wait for the staging command
to exit and finalize the turn according to that skill. A subsequent worker reads the
results before declaring verification complete.

### Baseline and acceptance evidence

Before the first production edit, record the actual source SHAs and run the unmodified
album, record, and bank suites in isolated scratch vaults on athena:

```bash
BOB_BIN=<verified-bob-binary> xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs \
  -e /snap/obsidian/current/app/obsidian -r <album-report> -o <album-shots>
BOB_CLI_DIR=<verified-bob-cli-checkout> BOB_BIN=<that-checkout>/target/debug/bob \
  xvfb-run -a node scripts/obsidian-record-card-e2e.cjs \
  -e /snap/obsidian/current/app/obsidian -r <record-report> -o <record-shots>
BOB_BIN=<verified-bob-binary> xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs \
  -e /snap/obsidian/current/app/obsidian -r <bank-report> -o <bank-shots>
```

Use the clean source/binary provenance contract in `docs/develop/validation.md` for the
full record suite; never substitute `--scope sheet` or `--self-test` for acceptance.
Repositories outside the current checkout must first be opened through `/sase_repo`, not
discovered or cloned independently. Keep fixture/config/data and the verified binary
fixed for paired comparisons where possible. Source revisions required by concurrently
landed changes must be recorded explicitly.

Preserve reports and screenshot baselines as durable indexed artifacts through
`sase artifact create` and record their refs in child-phase notes. Following workers
consume them through `sase artifact read`, not private transient filesystem paths. Do
not rely on a prior agent's temporary directory surviving. Run the suites again after
behavioral refactors and on the final integrated tree, inspect each report and
screenshot, and preserve the final evidence the same way.

Use stable fixtures for visual comparison: the same font, viewport, theme, date, data,
expansion state, scroll position, and reduced-motion setting. Cover all three cards in
light and dark at desktop width and 375 px, expanded and collapsed, including the
record's docked rail. The chrome phase may add a focused `preview-property-cards.cjs`
harness if the existing CDP captures do not provide those stable paired states. It must
render the actual DOM builders and styles. Compare corresponding card pixels exactly; do
not accept a new golden merely because it resembles the baseline. Exclude external
clock/cursor noise by making the fixture deterministic, not by masking differences
inside the card.

If a gate fails identically on the unchanged clean base, record the exact command,
source SHA, same assertion/error, and any already-tracking task as a
`PROPOSED FOLLOW-UP:` note on the assigned phase. Such a reproduced base failure does
not keep a completed phase open. A new regression must be fixed. Missing acceptance
evidence must be reported as missing, not as a pass.

## Phase ownership: selective hiding and the registry

1. Capture the baseline above and the initial `npm test` result. Read the existing
   selective-hide, double-equals, album-header, and bank-setup tests before editing.
2. Generalize `242ka-plugin-record-sheet-properties.js` into
   `propertyCardOwnKeys(scope, keys)` with an explicit, idempotent release handle. The
   two-argument API owns one current card mount in one native view scope; adapters
   attach it only after a successful mount and release it before replacing that mount.
   Track mount identity so an old unload/release cannot clear a newer mount. New owned
   keys replace that owner's old set rather than accumulating.
3. Preserve the proven direct container traversal:
   `.cm-editor > .cm-scroller > .cm-sizer > .metadata-container`, or
   `.markdown-preview-view > .markdown-preview-sizer > .mod-header > .metadata-container`.
   Tag only `.metadata-property` rows whose exact `data-property-key` is owned. Never
   hide the container, unrelated keys, or Add property. Never traverse into nested
   containers, embedded notes, or hover previews.
4. Retain MutationObserver updates for replacement rows and renamed keys, watching
   `childList`, subtree, and `data-property-key` changes but not the classes the painter
   writes. Clear old tags, disconnect observers, and empty tracking on release, gate
   loss, source/mode/path change, render failure, and plugin unload. A null builder
   result or exception must roll back pending ownership.
5. Record adapters supply schema row and ghost keys plus `title` only when
   `coverage.title === "sheet"`, retaining `hideEligible` and the record-specific
   inline-title suppression. Album supplies exactly `record`; bank supplies exactly
   `bank`. Replace the unbounded album/bank surface CSS hiding rules with owned-row
   rules. Preserve the record compatibility classes required by current tests.
6. Add `registerPropertyCard({kind, match({path, frontmatter}), open})`, with an ordered
   registry initialized per plugin instance and disposed on unload. Reject duplicate
   kinds rather than duplicating key handlers. Built-ins register in bank → album →
   record order and retain the exact current predicates and open arguments (`sourcePath`
   for bank, `filePath` for album/record). Match failures fail open without consuming a
   Vim key.
7. Refactor `248m-plugin-schema-double-equals.js`: keep `schemaDoubleEqualsTargetKind`
   as a compatibility facade, and have runtime matching and opening use the registered
   entry instead of separate kind switches. The gate must accept a resolved registered
   kind, not arbitrary truthy strings. Preserve every existing gate reason and refusal
   ordering, the capture-phase handler, WeakSet deduplication, and `<Esc>` cancellation
   of the pending operator before opening. Registering a synthetic fourth kind in tests
   must work without changing the gate or dispatcher. Unsupported notes still get Vim
   reindent.
8. Preserve relation editing through the normal-mode/Live-Preview guards. The relation
   equals decision applies to capture text; do not introduce a text-regex refusal that
   incorrectly disables a valid normal-mode card gesture on a note containing relation
   text. Test that insert/composition and non-editor input remain untouched. Keep all
   card close-key semantics unchanged.

Add focused tests for two panes of the same path, stale releases, replacement Properties
DOM, key renames, denied scopes, null/throwing mounts, and complete cleanup for all
three adapters. Extend `scripts/test-ledger-tools-schema-double-equals.cjs` with
ordering, fourth-kind, duplicate/unregister/reload, throwing matcher, and once-only open
coverage while retaining all existing assertions. Re-run the three real e2e suites,
record evidence, sync the validated plugin, and close only this child phase.

## Phase host: one lifecycle implementation, three adapters

Create `createPropertyCardHost({...})` in small source fragments before the adapters
that consume it. Return an explicit controller with setup/dispose, refresh,
scheduleRefresh, Reading mounting, and editor extension hooks. Document the final
options rather than forcing every species into one model format.

The common controller owns:

- Reading post-processor registration, batching per path, then deduplication per actual
  view surface so two panes both paint;
- binding generations and `MarkdownRenderChild` cleanup, detached-then-attached roots,
  stale-chunk unload immunity, and late view remounts;
- one coalesced 100 ms refresh timer, per-path batch timers, refresh effects,
  `layout-change` / `file-open` scheduling, and cleanup;
- `Prec.highest` ViewPlugin installation, widget lifecycle, path/mode change detection,
  refresh-effect dispatch, and an `editorAttributes` facet that restores owned classes
  when CodeMirror replaces the editor class list;
- the handoff to shared owned-key hiding after successful rendering, plus rollback and
  release on every failed/removed mount;
- per-adapter display/embed eligibility while keeping shared hiding unconditionally
  disabled inside embeds and hover popovers.

Adapter callbacks retain their existing model, gate, builder, signature, anchor,
subscription/data refresh, focus restoration, and optional post-paint behavior. Existing
wrapper methods may delegate to the controller for callers/tests. Move **all three**
hosts onto it; remove obsolete duplicated scheduling and registration loops. Preserve
Reading post-processor priorities (album/record 49, bank 47) and avoid multiple
registrations when setup is called again.

Record-specific responsibilities stay explicit: its frontmatter-only Reading view
fallback; source cache/in-flight reads; optimistic overlays and timers; collapse and
fold state; decoration reuse across body edits; selection-driven walk-line reveal;
metadata/day refresh; subscription-band ownership notifications; and rail mount/unmount
hooks in `242v-plugin-record-walk-rail.js`. Preserve current placement, including the
Reading card before an authored legacy H1. Late async results must not repaint an old
file or a disposed controller.

Bank-specific Source-mode lint remains active even though no property card mounts or
owns fields in Source mode. Keep its decoration producer as an adapter's additional
editor path or a separate species extension; do not put an unconditional Live-Preview
return above it. Preserve lint mapping through text edits and refreshed report
fingerprints. Album keeps its exact anchor, schema expansion, and focus.

Tests must prove lifecycle equivalence for the three adapters, including two panes, tab
return, mode changes, empty bodies, stale async/binding callbacks, teardown, failed
mount recovery, and record render-count/rail expectations. Add meaningful controller
tests to the explicit unit list. Run the focused card/rail suites, `npm test`,
`npm run validate`, and the three real e2e suites; compare reports to baseline, sync,
and close only this child phase.

## Phase chrome: shared pixels, terminology, and extension contract

Add `.bob-property-card` to the mounted album, record, and bank panel elements, keeping
their existing species classes. Editors and bank fence/category cards do not acquire
this class. Consolidate common border, radius, sizing, typography, theme variables, and
reduced-motion rules into the base. One `[data-root-hue]` ladder sets
`--bob-property-card-accent`; existing species accent variables map to it without
circular variable references. Retain species fallback colors and overrides used by
descendants.

Do not normalize intentional existing differences: album/bank backgrounds use 8% accent
while record uses 4%; record padding is `0.6em 0.75em` while album/bank use
`0.65em 0.8em`; bank uses `font-size: 0.9em` and `overflow-wrap: break-word`, while
album/record use the UI-small variable and `anywhere`. Preserve narrow-screen,
forced-color, focus, and record-rail border-radius overrides. Shared tokens may
parameterize those values but computed styles and pixels must stay identical.

Align `docs/record-card.md`, `docs/albums.md`, and `docs/bob-ledger-tools.md`: the
top-of-note panel is its **record/album/bank card**, and `==` opens its **card editor**.
Introduce former names once (record sheet, album header, Schema Card, Setup), then
consistently distinguish panels from editors. Retain palette names, button labels, IDs,
and URLs; preserve explicit anchors for any renamed headings with existing inbound
links. Keep other uses of "card" such as bank month fences and Task Card semantically
separate. Correct the nearby obsolete record-placement/native-Properties sentences
against the existing tested behavior, without redesigning that behavior or rewriting
unrelated references.

Add a concise property-card extension section to `docs/develop/architecture.md` covering
registry registration/disposal, matching order, the host adapter hooks, ownership
release, failure cleanup, scope restrictions, and CSS tokens. Explain how the later
intention adapter registers a fourth kind without editing the gate or the other three
hosts. Describe the actual final API and test entry points.

Generate/check docs inventory, build, and run `npm test`, `npm run validate`, plus
`BOB_CLI_DIR=<current-bob-cli-checkout> npm run docs:check` so the mounted-site gate
uses the intended source. Run final real acceptance and exact paired visual comparisons
from the original baseline. Publish the reports, screenshots, and comparison result as
artifacts; do not publish private live-vault values. Finish with `bob plugins sync`,
recording the deployed source/version and any resolved deployment conflict. Close only
this child phase.

## Child-epic land agent completion

Verify all three child phases landed together, their artifact-backed checks match the
final source, the extension API is documented, and deployment succeeded. Read the
original assigned bead again and reconcile every requirement against this plan and its
parent. Leave a concise API/evidence handoff on bob-cli-nh.6 for bob-cli-nh.7 to
consume; no direct messages to other users are needed.

Before the original phase can close, run `sase bead epic-symbols bob-cli-nh.6`. There
were no entries at planning time; resolve new entries or re-key legitimate later-phase
placeholders to a still-open owner. Check child phases similarly before their own
closes. Record discovered work as `PROPOSED FOLLOW-UP:` notes rather than creating task
beads. A reproduced clean-base failure is documented and does not hold completion open.

The requested child-epic land/completion flow must ultimately close **bob-cli-nh.6
only**, with a note naming the verified implementation, acceptance, visual parity, and
deploy evidence. Individual child workers never close it. The child epic's own
completion is host-managed. Never close **bob-cli-nh** or another ancestor
intentions-plan bead, never hand-set status, and never reopen the assigned phase absent
the explicit closed-at-launch host notice and genuinely unfinished work.
