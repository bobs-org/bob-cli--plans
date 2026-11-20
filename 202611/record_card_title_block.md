---
tier: tale
title: Record card owns the note title from the title slot
goal:
  Complete bob-cli-o6.4.1.2 with local record identity, kit title ownership, a docked
  walk rail, verified lifecycle behavior, documentation, and deployment.
size: medium
proposed_by: bbugyi200.athena.bob-cli-o6.4.1.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-o6.4.1.2
create_time: 2026-11-20 15:21:00
status: wip
---

- **PARENT:**
  [202611/collection_cards_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/collection_cards_title_block.md)
- **BEAD:**
  [bob-cli-o6.4.1.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.4.1.2.md)

# Record card owns the note title from the title slot

## Scope and current evidence

Complete **bob-cli-o6.4.1.2**, the `record-card` phase of
`plan:202611/collection_cards_title_block.md`. Only close this phase after its work is
verified. Keep every ancestor open. Record discoveries as `PROPOSED FOLLOW-UP:` notes on
this bead; create no task beads. No memory edits are authorized here. The outer epic's
glossary changes belong to bob-cli-o6.7; its decision-record choice is no.

This is one medium coding tale because the identity kit, title-slot primitive, title
ownership field, and reusable adoption controller are already implemented. The remaining
change is one record adapter and its existing acceptance probes. The phase has no
earlier remaining-work notes. Recheck before implementation.

Verified in bob-plugins at `a68e29b` (bob-ledger-tools 1.147.0):

- Album adoption landed in that commit; bank/editor badges and day/period adoption also
  landed. Do not redo those changes.
- `240k-plugin-property-card-title-adopt.js` provides `propertyCardAdoptTitle`,
  `propertyCardAdoptPatch`, heading demotion, per-scope release, and Reading/Live view
  helpers. Exports are in `374c`; `240j`/`374b` are already occupied.
- The record still has its old title row, ancestor crumb nav, report-gated model, body
  widget, and row-gated inline-title CSS. Its host does not opt into title ownership.
  Therefore this phase is not already implemented.
- `242i-plugin-record-sheet-model.js` has 816 lines and `242k-plugin-record-sheet.js`
  has 810. Extract new record-specific helpers into new fragments as needed rather than
  passing the 1000-line limit.
- `ADOPTED_KINDS` in the preview script already contains **bank and album**.
- `bob-cli-ms` and `bob-cli-ms.8` remain in progress. Do not edit
  `scripts/obsidian-record-writer-e2e.cjs` or its `test-` wrapper, even if a gate
  reports their known failure.
- `sase bead epic-symbols bob-cli-o6.4.1.2` currently has no entries. Recheck at close.

Read the assigned bead and these artifacts through audited commands:

- `plan:202611/collection_cards_title_block.md`: shared rules and record phase.
- `plan:202611/property_card_title_block.md`: binding design contract, especially local
  title rendering, ownership, placement, interaction, and accessibility.
- Beads `bob-cli-o6.1`, `bob-cli-o6.2`, and `bob-cli-o6.4.1.1`: title-slot GO in both
  modes, kit API, and album integration evidence.

The outer plan's research reference
`research:202611/property_card_title_block/property_card_title_block.md` returned
`missing` during planning. The approved outer and collection plans were read and contain
the complete binding contract; do not invent requirements from the absent report.

Use `/sase_repo` to open bob-plugins, then read its `AGENTS.md`,
`docs/develop/architecture.md` (title kit and slot), and `docs/develop/deployment.md`.
Paths below are relative to that repository. Read `docs.md` through `/sase_memory_read`
before documentation changes. Edit source fragments, register new
fragments/exports/tests, and regenerate `main.js` with the build. Kit extensions must be
additive and backward compatible.

## Required result

On a canonical desktop record, Reading and Live Preview show the card before Properties
in the title slot. It has a sheet tile containing the album emoji (fallback
`file-text`), `RECORD · <Album title link> · <noun>`, the record title, and the kit's
paper material. The accessible name is `Record: <title>`.

The local title appears before a report is available. When the report arrives, the noun
and field body join inside the same mounted root. Title ownership is independent of
Properties-row ownership. Fields, ghosts, collapse/fold behavior, created footer,
subscription band, editor commands, and writer semantics retain their behavior. A
recognized first-body walk line is hidden by the title kit; its rail stays docked inside
the card, including while raw source is revealed. No rendering, folding, or revealing
changes note bytes.

## Implementation

### 1. Separate local identity from the report-backed body

In `242i`/`242k`, add a pure local record identity model and compose it with the
existing body model. Keep the body gate and its `hideEligible`/`editable` decisions
intact. Do not make editor entry points or subscription ownership succeed merely because
a title exists.

Local identity requires a canonical record and resolvable canonical album using metadata
and existing path/parent checks. Preserve native rendering for malformed metadata,
mixed/non-record types, invalid parent/folder shapes, and mobile. Report absence alone
must not suppress an otherwise valid local identity. The title keeps its existing
precedence: trimmed frontmatter title, first H1, then file stem. Resolve the album title
through the landed `notebookHeaderNoteTitle` album rule and read its emoji from
metadata, so a stale report cannot change the kicker title. Do not add area-owner crumbs
to the record kicker.

Compose a pending identity-only model with no owned property keys and no report
editability. Preserve the broken-schema warning and album-YAML button when that body
state is available. Report arrival, loss, and schema changes must not cause the title to
disappear, acquire a different source, or remount its root.

### 2. Build the kit title block and preserve record hooks

Update `242j-plugin-record-sheet-dom.js` to call `propertyCardApplyIdentity` and
`propertyCardBuildTitleBlock`. Keep `.bob-record-sheet`, `data-record-sheet`,
`data-record-source`, field row keys, and focus hooks. Add `.bob-record-sheet-title` to
the kit heading. Put the existing edit/collapse tools in the actions slot; keep their
handlers and accessible names.

The kicker contains only the album link and, when available, the noun after the kit's
RECORD word. Preserve modifier-aware opening, middle-click, and hover previews using
existing handlers. The tile uses the album emoji or the kit icon. Remove the old crumb
nav and its dead CSS. Collapsing keeps the kicker, title, tools, chip summary, and rail;
the summary no longer duplicates crumbs. Broken schema warnings follow the title block.

Use the kit record material, full-width layout, wrapped long titles and narrow actions,
focus rings, forced-colors rules, and reduced-motion behavior. Avoid old title-row CSS
overriding the kit grid at 375px.

### 3. Adopt the slot and own the source title range

Opt `recordPropertyCardHost()` into `titleOwnership: true`. Reuse
`propertyCardAdoptTitle(scope, node, {path, kind: "record", view, recognizer, currentPath, onLost})`;
do not build another mount controller. Follow the landed album adapter for cached roots,
deferred Live Preview mount, in-place patching, focus restoration, onLost refresh, and
release. In Reading pass the actual preview renderer from `propertyCardReadingViewFor`;
Live Preview uses its `.cm-editor` and EditorView.

The current `paintRecordSheetReading` removes/rebuilds on every paint. Change it to
retain the root when path/scope match, patch only on meaningful changes, and restore
keyed focus if children change. The Live Preview cache likewise keeps the same root
across report arrival, fold changes, and selection-only updates. Update the
frontmatter-only Reading fallback so it mounts the title without waiting for a body
postprocessor or report.

Add a record-specific recognizer in a new record fragment, with `requireHeading: false`,
a matching legacy H1, and a follower accepting only a valid current/stale record walk
line. Reuse `recordWalkClassify`/`recordWalkModel` grammar and retain whole-source
damage checks: duplicate or damaged nav stays authored content. **Implementation trap:**
`recordWalkClassify` requires frontmatter; passing it a bare paragraph yields
`no_frontmatter`. Build the recognizer from the classified whole record or a
deliberately framed candidate, not a naked line. Reading uses
`propertyCardReadingLineText` to reconstruct `[[href|text]]`; make that representation
match the same walk grammar. Give the recognizer a stable key that changes when its
title/walk inputs change, because the adoption helper uses it for re-owning.

Use the kit range and inclusive selection rule: recognize only the first nonblank body
walk line, or one following the matching first H1 with optional blanks. Do not hide a
fenced, damaged, late, or after-prose line, or a different H1. Live Preview reveals on
any selection touching the range and hides it again on leave; the line after the range
is outside it. Never overlap a kit range with the old walk replacement decoration.

Adapt `242v-plugin-record-walk-rail.js` so a title-owned first-body walk rail is built
inside the card bottom with `has-record-walk` in both modes and remains there during
reveal. Use the kit source class instead of the old Reading walk class for this owned
range. A walk line outside the title range keeps its existing in-place replacement
behavior. Preserve rail-only operation when no sheet can mount, singleton/count/syncing
states, and command/modifier navigation.

When adoption reports fallback, retain today's body widget/dock path and demote the kit
heading. Source mode, embeds, hover/canvas, denied/failed mounts, path changes, failed
paint, teardown, and unload release ownership and restore native content promptly.
Release only the matching scope/path/generation; do not clear another pane or another
card species. The kit heading is level one only while ownership is held. Test two panes
showing the same file.

### 4. Migrate selectors and audit consumers

Delete the record-specific inline-title CSS tied to `bob-record-sheet-hides-properties`.
Only `bob-property-card-owns-title` controls inline-title hiding. Keep the record scope
marker for row consumers; preserve `hideEligible` and the native Properties container,
identity/unrelated keys, and Add property visibility.

Use `bob-property-card-owned-property` for individual owned rows. Remove legacy
`bob-record-sheet-owned-property` tagging in `240b` and its CSS only after updating all
consumers, including `test-ledger-tools-property-card-ownership.cjs`, record sheet
tests, and the e2e unload-leak probe. This is a compatible kit cleanup for the migrating
adapter, not a change to ownership semantics for other species.

Search all plugins and scripts for affected selectors before moving the card. Verify
navigation hotkeys, command mode, `248m` dispatch, `242pa` entrypoints, walker footer,
and subscription surfaces (`241hf`) still resolve the card from the owning view when it
is outside `.cm-content`. Keep stable field/editor hooks; change only assumptions about
location or the explicitly retired selectors.

## Verification and expectations

Run focused unit tests as changes land, then the repository gates. Extend the existing
record-sheet/walk suites or add a focused identity/recognizer suite. Fixtures for Live
Preview must contain `.cm-contentContainer` and install the title field, so they
exercise real slot ownership rather than accidental fallback.

Cover local identity before reports, body attachment to the identical root, report loss,
invalid schema with title still owned but rows visible, collapsed and broken DOM, album
title precedence, emoji fallback, and unchanged field/body semantics. Cover first-line
and after-H1 walk ranges, stale lines, late and fenced lines, damaged/duplicate lines,
Reading reconstructed links, inclusive reveal boundaries, multiple selections,
frontmatter-only notes, Source transitions, embeds, failed mount, file switches, stale
releases, and two-pane isolation. Preserve fallback docking assertions alongside the new
slot assertions.

Update `scripts/obsidian-record-card-e2e-lib.cjs` so `assertNativeVisibility` takes
separate explicit expectations for **title ownership** and **row hiding**; neither
expectation may be derived from the observed DOM. Update its self-test and offline
wrapper. In `obsidian-record-card-e2e.cjs`:

- Healthy and collapsed owning views: inline title hidden and eligible rows hidden.
- Broken schema or missing report with valid local identity: inline title hidden, native
  rows visible. Preserve the existing body/editor safety checks.
- Source, non-record/invalid-parent notes, embeds, failed ownership, and plugin disable:
  native title and rows restored according to that surface's settings.
- Replace `orderBeforeFirstHeading` with slot membership and order before Properties;
  assert species identity, sheet tile, accessible name, and one level-one card heading
  while owned. Preserve authored nonredundant headings.
- Add delayed-report proof, no remount, raw reveal, lifecycle cleanup, and unchanged
  passive fixture hashes. Record changed assertions in the close note.

Update record-walk e2e to prove the rail is inside the slot card, first walk paragraph
uses kit hiding, and Live Preview reveal keeps the same card and rail. Preserve
moved-line behavior, count/footer agreement, navigation, and missing target checks. Run
the full record-card e2e: `--scope full` also runs `obsidian-record-card-workflows.cjs`;
it is an imported workflow module, not a separate CLI. Its keyboard, YAML-jump, editing,
conflict, and guarded-write checks must still pass. Do not weaken them or touch the
reserved record-writer harness.

Add `record` to preview `ADOPTED_KINDS`, preserving bank and album. Assert its species
word, sheet tile, heading, and accessible name; render and inspect light and dark at
narrow and wide widths, expanded/collapsed/broken states. Keep any still-unadopted
species' existing pixel checks. The full cross-species title-block walkthrough and
grayscale proof belong to bob-cli-o6.6.

Commands from the opened plugin repo (set `BOB_CLI_DIR` to the current bob-cli checkout
and `BOB_BIN` to the current installed binary):

```bash
npm run build
npm run build:check
node --test scripts/test-ledger-tools-record-sheet.cjs scripts/test-ledger-tools-record-walk.cjs scripts/test-ledger-tools-property-card-ownership.cjs scripts/test-ledger-tools-property-card-title-adopt.cjs scripts/test-obsidian-record-card-e2e.cjs scripts/test-obsidian-record-walk-e2e.cjs
npm test
npm run docs:check
TMPDIR=/tmp xvfb-run -a node scripts/obsidian-record-card-e2e.cjs --scope full -e /snap/obsidian/current/app/obsidian -r /tmp/record-title-card.json -o /tmp/record-title-card-shots
TMPDIR=/tmp xvfb-run -a node scripts/obsidian-record-walk-e2e.cjs -e /snap/obsidian/current/app/obsidian -r /tmp/record-title-walk.json -o /tmp/record-title-walk-shots
TMPDIR=/tmp xvfb-run -a node scripts/preview-property-cards.cjs -e /snap/obsidian/current/app/obsidian -b a68e29b -o /tmp/record-title-preview
```

Also run the focused editor/dispatcher tests affected by the selector audit and any new
tests. Use `/sase_monitor` for long runs; never end a turn with a native background
command still running. `TMPDIR=/tmp` avoids the already-observed Chromium socket-path
limit. Offline self-tests are not real-Obsidian evidence. Register reports, preview
index, and key screenshots with `sase artifact create --bead bob-cli-o6.4.1.2`, and
inspect the screenshots before claiming success.

Rerun failures in isolation and compare suspected unrelated failures against a clean
base. The album prerequisite documented timing flakes, a bob-cli docs word budget
failure, and the schema-writer two-commit failure; these are evidence to investigate,
not automatic permission to ignore new failures. Identical clean-base failures get
`PROPOSED FOLLOW-UP:` notes with existing task IDs when known and do not keep the phase
open. If real Obsidian cannot run, record the exact cause, keep offline checks green,
and append the required follow-up per the parent plan.

## Docs, integration, deploy, and close

Update `docs/record-card.md` to describe the title slot, species kicker/tile,
local-first identity, independent row hiding, recognized H1/walk hide and reveal, and
persistent docked rail during reveal. Preserve its published URL and word budget. No CLI
contract, Cargo version, CHANGELOG, or memory changes.

Before final integration and sync, fetch and incorporate `origin/master` using the
repository workflow. Keep all sibling fragments and test registrations; regenerate
rather than merge `main.js`. Bump bob-ledger-tools to the next available minor version,
run `npm run docs:gen`, rebuild, and verify the final merged tree.

Deploy only this plugin using the explicitly opened repository:

```bash
bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools --dry-run
bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools
```

Follow `docs/develop/deployment.md` for protected deployments, startup smoke
verification, and byte checks of deployed `main.js`, `styles.css`, and `manifest.json`.
Never force an unrelated/unlanded deploy away. Honor SASE's host-owned commit flow: no
manual git commit. Same-checkout dirty sync is supported; if a clean landed prerequisite
is needed, use the documented clean-sync recipe and SASE finalization/monitor
continuation, not a promise of a future manual step. Declare all changed repositories
via `/sase_final`, with the linked plugin commit describing record adoption and any
additive kit changes.

Before closing, run `sase bead epic-symbols bob-cli-o6.4.1.2` and resolve or re-key any
leftovers to a still-open owner. Close only this bead with
`sase bead close bob-cli-o6.4.1.2 --note "<verified behavior, gates, changed e2e expectations, artifacts, version and deployment evidence>"`.
Never close an ancestor. Completion requires the record implementation, applicable
verification, docs, and deploy evidence described above; submitting this plan alone does
not complete the phase.
