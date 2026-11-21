---
tier: tale
size: medium
title:
  Record card wears the RECORD kicker and sheet tile, owns the note title from the title
  slot, and migrates its title selectors onto the kit
goal:
  On desktop Obsidian, opening a record such as eat/<restaurant>.md shows the record
  card in the title slot above native Properties. The card has a sheet tile, the kicker
  RECORD · <Album title link> · <noun>, the record title, and paper material, with the
  walk rail docked at its bottom. The card title paints at once from local data, the
  first-line walk line (or a legacy H1 equal to the title) is hidden at render time and
  revealed by the cursor, and every failure restores native rendering. The plan ends
  with bob-ledger-tools bumped, deployed, verified in real Obsidian, and bead
  bob-cli-o6.4.1.2 closed.
proposed_by: bbugyi200.athena.bob-cli-o6.4.1.2
proposed_with: claude/opus
bead: bob-cli-o6.4.1.2
status: done
---

- **BEAD:**
  [bob-cli-o6.4.1.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.4.1.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-o6.4.1.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-o6.4.1.2.md)
- **COMMITS:**
  - [b8b7baf](https://github.com/bobs-org/bob-plugins/commit/b8b7baf440e888cd94c948f5339cd37527928079)
    — feat(ledger-tools): record card owns the title block from the title slot

# Record card: title block in the title slot

## Scope and status

This tale completes **bob-cli-o6.4.1.2**, the `record-card` phase of the child epic
`plan:202611/collection_cards_title_block.md` (bead `bob-cli-o6.4.1`). Close only this
bead. Never close `bob-cli-o6.4.1`, `bob-cli-o6.4`, `bob-cli-o6`, or any `bob-cli-ms`
bead. Create no beads. Record discovered work with
`sase bead note bob-cli-o6.4.1.2 'PROPOSED FOLLOW-UP: <summary — detail>'`. No phase of
this epic edits any memory file.

An earlier plan for this phase (`plan:202611/record_card_title_block.md`) was approved,
but its coding run produced no output and left nothing behind. bob-plugins
`origin/master` has no record-card commit, and the bead has no notes. This plan replaces
that one. Start by confirming nothing has landed since
(`git log origin/master -- plugins/bob-ledger-tools` in bob-plugins, plus
`sase bead read bob-cli-o6.4.1.2`).

## Read first

Read through the audited interfaces:

- `sase artifact read plan:202611/property_card_title_block.md "<why>"` is the binding
  design contract. Read the sections "One skeleton for every card", "The species"
  (Record row), "States", "Title ownership: hide, reveal, restore", "Placement",
  "Accessibility, sizing, and motion", "Interaction", and "Naming".
- `sase artifact read plan:202611/collection_cards_title_block.md "<why>"` gives the
  shared rules and the `record-card` phase.
- `sase bead read bob-cli-o6.4.1.1 -r "<why>"`, note #6, gives the album adoption
  evidence and the glue API. Notes #2–#5 list known failures that also happen on a clean
  tree.
- Recheck `sase bead read bob-cli-ms` and `bob-cli-ms.8`. Both were in progress at
  planning time. While either is open, **do not edit**
  `scripts/obsidian-record-writer-e2e.cjs` or
  `scripts/test-obsidian-record-writer-e2e.cjs`. Running them is fine.

Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`). Read its
`AGENTS.md`, `docs/develop/architecture.md` ("Title block kit", "Title slot", "Adopting
the title block"), and `docs/develop/deployment.md`. Paths below are relative to
`plugins/bob-ledger-tools/` unless they start with `scripts/` or `docs/`.

## What exists today (verified at bob-plugins `63cd4ee`, bob-ledger-tools 1.147.0)

**The kit and glue have landed and are in use.**

- Adopter glue: `src/240k-plugin-property-card-title-adopt.js`, exported in `374c`.
  - `propertyCardAdoptTitle(scope, node, { path, kind, view, recognizer, currentPath, onLost })`
    returns `{ mode: "slot" | "fallback", release }`.
  - Helpers: `propertyCardAdoptedEntry`, `propertyCardAdoptRelease(scope, path)`,
    `propertyCardAdoptPatch(target, fresh)`, `propertyCardSyncTitleHeading`,
    `propertyCardReadingViewFor(app, scope)`, `propertyCardLiveCurrentPath(view)`.
  - Live Preview mounts and owns in a microtask. Reading mounts synchronously.
  - The glue promotes the kit heading only while it owns the title. It demotes it
    otherwise.
- Range and recognizers: `src/240h-…`.
  - A recognizer is
    `{ key, requireHeading, heading(cleaned, raw), follower(trimmedLine, index) }`.
  - The range starts at the first non-blank body line. With `requireHeading: false`, a
    follower may start it.
- Ownership: `src/240i-…`.
  - Live Preview: a StateField that block-replaces the range and reveals it on an
    inclusive selection touch.
  - Reading: tags `h1` and follower `p` with `bob-property-card-title-source`. Follower
    text comes from `propertyCardReadingLineText`, which renders links as
    `[[data-href|text]]`.
- Builders: `src/240g-…`.
  `propertyCardBuildTitleBlock(doc, { kind, title, kicker, tile, onLink })` returns
  `{ element, heading, actions, … }`.
- Species: `src/240f-…`. Record is the `sheet` tile with the `file-text` icon, word
  `Record`. Accessible names come from `propertyCardApplyIdentity`.

**The album card is the reference adopter.** See `src/241v-plugin-album-header.js`:

- `paintAlbumHeaderReading`: retains the node, patches it on signature change, releases
  the adoption before the body sweep, and falls back to body placement.
- `buildAlbumHeaderDecorations` and `albumHeaderPatchLive`: deferred patch, no card
  widget in slot mode, today's widget in fallback.
- `albumHeaderCleanEditor`: releases the adoption.

**The record card has not adopted anything.**

- Host (`src/242k-plugin-record-sheet.js`, 810 lines): no `titleOwnership`.
- `paintRecordSheetReading` removes and rebuilds the sheet on every paint. It inserts
  the sheet before the first H1.
- The model (`src/242i-…`, 816 lines) is report-gated:
  - `recordSheetGate` fails with `no-report` when no report exists.
  - `albumResolveSchemaTargetForFile` also needs the report to accept a record parent.
- DOM (`src/242j-…`): `header > .bob-record-sheet-title-row` and the crumb
  `nav.bob-record-sheet-nav`. The collapsed `.bob-record-sheet-summary` repeats the
  crumbs. The aria-label is `<title> record details`.
- Walk (`src/242t-…`, `242u-…`, `242v-…`).
  - Live Preview (`buildRecordSheetDecorations` in `242v`): a first-body walk line
    becomes one `.bob-record-walk-dock` replace widget holding the sheet and the rail.
    Otherwise the sheet is a widget at the anchor and the rail replaces the line in
    place. Touching the line removes the rail and shows the raw line.
  - Reading (`recordWalkPaintReading`): appends the rail inside the sheet
    (`has-record-walk`) and hides the paragraph with `bob-record-walk-source`.
- The walk model's album title without a report is `albumFrontmatter.title` or the
  humanized stem. It ignores the album rule in `notebookHeaderNoteTitle`.
- CSS (`styles.css`, around lines 11260–11330 and 11440–11475):
  - inline-title hiding keyed on `bob-record-sheet-hides-properties`;
  - owned-row hiding keyed on that marker plus `bob-record-sheet-owned-property`;
  - title-row, nav, crumb, and summary rules.

  The kit CSS (around 11786–11820) already has the owns-title rules,
  `.bob-property-card-title-source`, and `[data-card-kind="record"]` paper material. Its
  comment still promises the record migration.

- Owned rows: `src/240b-…` tags both `bob-property-card-owned-property` and the legacy
  `bob-record-sheet-owned-property`. The legacy class is read only by
  `scripts/test-ledger-tools-record-sheet.cjs`,
  `scripts/test-ledger-tools-property-card-ownership.cjs`, and the unload-leak probe in
  `scripts/obsidian-record-card-e2e.cjs` (around line 1366).
- Many callers rely on the report-gated `recordSheetModelFor`, `recordSheetGateFor`, and
  `recordSheetOwnsPath`:
  - Record Card editor: `242p`, `242pa`, `242s`;
  - subscription header: `241hg`;
  - exports: `370-exports-record-sheet.js`;
  - e2e probes and unit tests.

  **Keep those three APIs' semantics unchanged.**

- Selector consumers outside the record fragments resolve the card by class within the
  active leaf (`scripts/obsidian-record-card-workflow*.cjs`, `241hf`
  `closest(".bob-record-sheet")`), so they keep working when the card lives in
  `.cm-sizer` or `.mod-header`. No other plugin queries record-sheet selectors. Re-grep
  before moving the card.
- `scripts/preview-property-cards.cjs` `ADOPTED_KINDS` holds `bank` and `album`.
- `src/310-install-methods.js` installs mixins. `src/370-exports-record-sheet.js` (43
  lines) exports record helpers for tests.

Re-verify each fact before relying on it, because sibling work keeps landing.

## Required result

On a canonical desktop record, in Live Preview and in Reading:

- **Placement.** The card is in the title slot, after the hidden inline title and before
  native Properties.
- **Title block.**
  - Sheet tile: the album emoji, else `file-text`.
  - Kicker: `RECORD · <Album title link> · <noun>`. The noun appears once the report
    arrives.
  - Title: frontmatter `title:`, then the first H1, then the stem. This is today's
    order.
  - Paper material. Accessible name `Record: <title>`.
  - Exactly one `aria-level="1"` node in the pane.
- **Local paint.** The title block paints from local data before any report. The noun
  and the report-backed body (fields, ghosts, fold, created footer, subscription band,
  broken warning, and YAML button) join inside the same mounted node, with no remount.
- **Title ownership is independent of row ownership** (contract A9). The inline title is
  hidden whenever the card owns the title. The Properties-row hide keeps its
  `hideEligible` gate.
- **Walk line.** A walk line that is the first non-blank body line, or directly follows
  a legacy H1 equal to the title, is hidden at render time. Touching it in Live Preview
  reveals the raw line while the card and its rail stay in the slot. The rail sits
  inside the card's bottom in both modes. A walk line that is not first keeps today's
  behavior.
- **Native fallback.** Source mode, embeds, hover popovers, canvas, mobile, a failed
  mount, and teardown keep or restore native rendering. No note bytes change.
- **Interaction unchanged.** `==`, `Ctrl+Shift+P`, every keymap, the Record Card editor,
  and writer semantics stay the same.

## Implementation

Edit `src/` fragments only. Keep each hand-edited fragment at or below 1000 lines. Put
new record code in new fragments, for example `242kb-plugin-record-sheet-title.js` and
`242w-plugin-record-title-recognizer.js`; pick free names. Register each new fragment in
`src/fragments.json` at a sensible position, each new mixin in
`src/310-install-methods.js`, and each new test-facing helper in
`370-exports-record-sheet.js` (or a new `370*` fragment). Rebuild with `npm run build`.

Any kit change must be additive and backward compatible; say so in the commit. Removing
the legacy owned-row class (step 7) is the one sanctioned kit cleanup.

### 1. Local identity and the composed card model

Add
`recordSheetIdentityModel(path, { frontmatter, metadataCache, vault, desktop, heading })`.
It is pure and uses no report. It returns
`{ kind: "record", path, stem, title, albumPath, albumStem, albumTitle, emoji }` or
`null`.

It returns `null` unless every report-free gate check passes:

- desktop;
- the frontmatter is an object;
- the type is canonical record (`recordSheetIsCanonicalRecordType`);
- the parent is one unambiguous link that resolves to an existing note other than itself
  (reuse `albumSchemaEntryParentTarget` and `albumHeaderResolveFile`, or extract the
  report-free part of `albumResolveSchemaTargetForFile` additively);
- the parent's cached type includes album (`recordSheetParentIsAlbum`);
- `recordSheetFolderKind(path, albumStem) === "ok"`.

Fields:

- `title`: trimmed frontmatter `title`, then `heading`, then the stem.
- `albumTitle`: `notebookHeaderNoteTitle(albumPath, …)`, which already has the album
  rule.
- `emoji`: the album's frontmatter `emoji` when it is a plain string.

Add a plugin method `recordSheetCardModelFor(path, sourceText)` that returns
`{ identity, body, title }` or `null`:

- `identity` is required.
- `body` is today's `recordSheetModelFor(path, sourceText)`, which may be `null`.
- The title always comes from `identity`, so it never flips when the report arrives or
  changes.

Keep `recordSheetModelFor`, `recordSheetGateFor`, `recordSheetOwnsPath`, and the Record
Card entry points unchanged. A title alone never enables editing, row hiding, or
subscription ownership.

Fill `widget.albumPath` and the editor cache's `albumPath` from the identity, so edits
to the album's metadata (title, emoji) repaint the kicker through
`recordSheetWatchesParent`.

Align the walk model's album title with the album rule: when there is no report entry,
`recordWalkModel` uses `notebookHeaderNoteTitle(albumPath, …)` instead of
`albumFrontmatter.title` or the humanized stem. The rail and the kicker then agree
before the report, and the canonical walk line matches what bob writes for albums that
have not been tidied. Update any walk unit expectation that encoded the old fallback,
and list it in the close note.

### 2. Title block DOM (`recordSheetBuildEl` in `242j`)

`recordSheetBuildEl(model, options, doc)` must still accept a bare body model (the
preview and existing tests pass one). Derive the identity from the body when no identity
is given, as `albumHeaderCardFrom` does.

The root keeps:

- `section.bob-record-sheet.bob-property-card`;
- `data-record-sheet`, `data-record-source`, `data-root-hue`, `data-record-state`;
- the row and focus hooks.

Build it in this order:

1. Call `propertyCardApplyIdentity(root, { kind: "record", title })`, so the aria-label
   is `Record: <title>`. Keep `role="group"`.
2. The first child is `propertyCardBuildTitleBlock`:
   - **Tile:** `{ emoji: identity.emoji }`. The sheet form and the `file-text` fallback
     come from the species.
   - **Kicker:**
     - First, the album title as a link to `albumPath`. It opens through the existing
       `openPath` handler, keeping modifiers and middle-click (match the album card's
       `albumHeaderLinkPane`). It shows a hover preview through `recordSheetBindHover`.
       Give it the class `bob-record-sheet-crumb`.
     - Then the noun as plain text, only when the body is present.
     - Area or owner crumbs leave the record card, per the contract.
   - **Title:** add the class `bob-record-sheet-title` to the kit heading.
3. Move `div.bob-record-sheet-tools` (edit, collapse) into the block's `actions` slot.
   Keep its handlers, labels, and `data-record-sheet-focus` keys. The edit button still
   appears only when `editable`.
4. Delete `recordSheetRenderTitleRow` and `recordSheetRenderCrumbs`, and the
   `header.bob-record-sheet-header` wrapper if nothing still needs it. Grep first.

Body states:

- **Collapsed:** the title block, then `.bob-record-sheet-summary` with the chips and
  field count. The summary no longer holds crumbs.
- **Broken:** the title block, then today's warning and `Open album YAML` button.
- **Pending** (identity, no body): the title block only.

### 3. Record title recognizer (new record fragment)

Add `recordSheetTitleRecognizer({ path, title, walk, sourceText })`:

- `requireHeading: false`.
- `heading(cleaned)` accepts an H1 whose cleaned text equals the record title, which
  covers legacy records.
- `follower(line)` accepts only the classified walk line, and only when `walk.lineState`
  is `current` or `stale`:
  - **Live Preview form:** the trimmed source line at `walk.lineIndex`. Compare it
    exactly.
  - **Reading form:** the `[[href|text]]` text from `propertyCardReadingLineText`. Parse
    it with `recordWalkParseLinkRun(line, albumStem)` plus `recordWalkIsWalkLine`.
    Require the same targets as `walk.writtenLinks`, in order and without `.md`, and the
    same display labels (`recordWalkDisplayLabel`).

  **Trap:** `recordWalkClassify` needs frontmatter. Never classify a bare line. Use the
  model already classified from the whole source.

- `key` changes whenever the title, the walk line text, or the line state changes, for
  example `record:<path>:<title>:<lineState>:<line>`. The glue re-owns only when the key
  changes.
- With no title and no walk line, it is the never-recognizer
  (`propertyCardNeverTitleRecognizer("record")`).

The kit then hides the walk line only when it is the first non-blank body line or
follows the recognized H1. A fenced, damaged, duplicate, late, or after-prose line, and
an H1 that differs from the title, all stay visible.

### 4. Reading mount (`paintRecordSheetReading`, moved or split into the new fragment)

Follow `paintAlbumHeaderReading`:

1. Use `propertyCardReadingViewFor(this.app, scope)`. A stale binding whose leaf now
   shows another file paints nothing and releases.
2. Reuse the adopted slot node for this path, or the existing body node. Rebuild only
   when the card signature changes, and patch it in place with `propertyCardAdoptPatch`.
   The signature covers the identity, `recordSheetSignature(body)`, collapse, fold, hue,
   overlay, subscription, and walk/rail state.
3. Restore keyed focus (`data-record-sheet-focus`).
4. Adopt with
   `propertyCardAdoptTitle(scope, node, { path, kind: "record", view: reading.view, currentPath: reading.currentPath, recognizer, onLost: () => this.scheduleRecordSheetRefresh() })`.
5. On `fallback`, keep today's body placement (before the first non-code H1, else the
   root start).

**Sweep trap:** `.mod-header` sits inside the Reading root. Before any sweep, release
the adoption for that scope and path (`propertyCardAdoptRelease`). Exclude the slot node
from "stray" and body-node sweeps, as `albumHeaderBodyNodes` does. Never let
`recordWalkRemoveReading` strip the rail out of a retained slot card on an ordinary
repaint.

**Rail placement:**

- The rail goes inside the card at its bottom (`has-record-walk`) whenever a matching
  walk paragraph exists. This is today's Reading behavior.
- When the card owns the title and the paragraph is the recognized title-source range,
  the kit tag hides it. Do **not** also add `bob-record-walk-source`, so the paragraph
  returns the moment ownership drops.
- Any other matching paragraph keeps today's `bob-record-walk-source` handling.

**Unchanged:** the subscription band, the Properties-row hide (`hideEligible` and
`recordSheetSetNativeProperties`), the `bob-record-sheet-hides-properties` scope marker,
and the frontmatter-only view fallback (`mountRecordSheetFrontmatterOnlyViews`). Make
sure a pending, report-less model also mounts through that path.

### 5. Live Preview mount (`buildRecordSheetDecorations` in `242v`)

Opt the host into `titleOwnership: true`. Then, following `buildAlbumHeaderDecorations`
and `albumHeaderPatchLive`:

1. Build the card model from the live text. Reuse the adopted node for this path, and
   patch it in a deferred callback only when the card signature changes. Selection-only
   updates must never rebuild or remount the card.
2. Adopt with `view.dom` as the scope, `view` as the view, `recognizer`, and
   `currentPath: () => propertyCardLiveCurrentPath(view)`.
3. **Slot mode:**
   - Return no card widget.
   - If the walk rail exists and `propertyCardTitleSourceRange(text, recognizer)` covers
     `walk.lineIndex`, the rail is part of the card's bottom (`has-record-walk`). Add
     **no** record decoration on that line: the kit StateField owns its hide and reveal,
     and the rail stays during reveal.
   - Otherwise, a non-first walk line keeps today's in-place rail replacement and
     today's reveal rule (`recordWalkSelectionTouches`).
   - Never let a record decoration overlap the kit range.
4. **Fallback** (embeds, no slot, a remembered failure): keep today's code path
   unchanged (the dock widget, sheet widget, and in-place rail), with the kit heading
   demoted.
5. Keep the editor cache keyed so a cached result can never hide rows for a failed
   build. `recordSheetSetEditorHide` and the row gate stay as they are.

### 6. Lifecycle

Release the adoption, matching scope and path only, in each of these places. This
mirrors `albumHeaderReleaseAdoption` and the other album paths.

- `recordSheetRemoveReading`;
- the Live Preview "no path / not Live Preview / no card / build failed" branches;
- `recordSheetCleanEditor`;
- `recordSheetSetEditorScope(view, false)`.

The glue's `onLost` triggers a refresh, which remounts or falls back.

Never clear another pane's adoption, or another species' adoption on the same scope
after a file switch. Host dispose already runs `propertyCardTitleReleaseAll` only on
plugin unload. Re-verify that, and do not change it.

### 7. Selector migration and CSS

- Delete the two record-specific inline-title rules keyed on
  `bob-record-sheet-hides-properties`. Only the kit's `bob-property-card-owns-title`
  hides the inline title now.
- Delete the record owned-row rule keyed on `bob-record-sheet-owned-property`. Owned
  rows hide through the shared `bob-property-card-owned-property` rule.
- In `240b`, remove the `PROPERTY_CARD_LEGACY_RECORD_OWNED_CLASS` tagging and untagging,
  but only after updating every consumer in the same change:
  - `scripts/test-ledger-tools-record-sheet.cjs`;
  - `scripts/test-ledger-tools-property-card-ownership.cjs`;
  - the unload-leak probe in `scripts/obsidian-record-card-e2e.cjs`, which should check
    `bob-property-card-owned-property` and `bob-property-card-owns-title` leaks instead.

  Grep again first.

- Keep the `bob-record-sheet-hides-properties` scope marker, its embed
  `.metadata-container` rule, and its readers.
- Update the kit CSS comment that promised this migration.
- Remove the dead title-row, nav, and crumb rules and the old 375px title-row overrides.
- Add record rules only where the kit falls short:
  - the actions slot holding the tools;
  - the folded summary line under the block;
  - the rail flush at the card's bottom inside the slot (today's `has-record-walk`
    geometry, so the existing "rail docks flush" check still holds);
  - focus rings on the kicker link and tools;
  - `forced-colors` and reduced motion, per the contract.
- The fallback dock styles stay.

## Tests (unit, offline)

Update `scripts/test-ledger-tools-record-sheet.cjs`. For slot mode, the fake Live
Preview needs
`.markdown-source-view.is-live-preview > .cm-editor > .cm-scroller > .cm-sizer > .cm-contentContainer`
and an installed title field. Copy the fixtures in
`scripts/test-ledger-tools-property-card-title-adopt.cjs` and
`scripts/test-ledger-tools-intention-card-title-block.cjs`, so slot ownership is really
exercised and not an accidental fallback. Cover:

- DOM:
  - kicker, tile form, title class and text, tools in the actions slot;
  - aria-label `Record: <title>`;
  - collapsed summary without crumbs;
  - broken warning after the title block.
- Local identity:
  - a pending card before any report;
  - the body joining the same node with no remount;
  - report loss;
  - the album title rule (album `title:`, then H1, alias, stem);
  - the emoji fallback;
  - rejections: mixed type, outside folder, nested folder, missing, ambiguous, or cyclic
    parent, and a non-album parent.
- Slot and fallback:
  - slot ownership in both modes; the fallback dock path unchanged;
  - one promoted heading when owned, demoted otherwise;
  - `onLost("path-changed")` remounts;
  - release on Source mode, file switch, and failed paint;
  - two panes on the same file stay isolated.
- CSS literal checks now point at the kit selectors. The owned rows use the shared
  class.

Add recognizer tests:

- the walk line first;
- after a legacy H1 equal to the title;
- stale;
- not first;
- fenced;
- damaged or duplicate;
- an H1 that differs from the title;
- the Reading `[[href|text]]` form, including escaped labels;
- key changes.

Add reveal boundary tests through `propertyCardTitleFieldReduce`: the cursor on the
line, on the line after, and multiple selections.

Also update:

- `scripts/test-ledger-tools-record-walk.cjs` (the album-title fallback);
- `scripts/test-ledger-tools-property-card-ownership.cjs` (no legacy class);
- `scripts/test-ledger-tools-property-card-host.cjs`, if the extension count changes;
- `scripts/test-obsidian-record-card-e2e.cjs` and
  `scripts/test-obsidian-record-walk-e2e.cjs` (the offline wrappers).

## Real-Obsidian e2e and previews

**`scripts/obsidian-record-card-e2e-lib.cjs`.** Split
`assertNativeVisibility(probe, shouldHide)` into explicit, separately passed
expectations: `{ titleOwned, rowsHidden }`.

- `titleOwned` drives the inline-title check.
- `rowsHidden` drives the schema-row check.
- Neither is derived from the observed DOM.

Update its self-test. Then update `scripts/obsidian-record-card-e2e.cjs`:

- **Healthy and collapsed owning views:** title owned, rows hidden.
- **Broken schema** (`broken/record.md`): the card mounts with the title, title owned,
  rows visible. This is a changed expectation; today the inline title shows.
- **No sheet at all:** Source mode, `bank/2026-11.md`, `eat/legacy-type.md`,
  `misfiled.md`, `ordinary.md`, parent and type changes, and plugin disable. Native
  title and rows stay visible. Re-enabling the plugin restores both hides.
- **Order:** replace `orderBeforeFirstHeading` with "card in the title slot (`.cm-sizer`
  / `.mod-header` direct child) and before native Properties".
- **New identity checks:** `data-card-kind="record"`, the sheet tile, `Record: <title>`,
  the kicker album link and noun, and one level-one heading in the owning view.
- **New behavior checks:** title before the report and the body joining the same node
  (use the delayed-report technique from the album e2e); Live Preview reveal of a legacy
  H1 or walk line; an embed card that is demoted and does not own the title.
- Keep every other assertion. List every changed expectation in the close note.

**`scripts/obsidian-record-walk-e2e.cjs`:**

- The rail is inside the slot card at its bottom in both modes.
- The first-line walk paragraph is hidden through the kit class
  (`p.bob-property-card-title-source` in Reading), not `bob-record-walk-source`.
- The Live Preview cursor reveal shows the raw line, and the **same** card and rail stay
  mounted. This replaces "rail returns after cursor leaves".
- Moved-line in-place behavior, counts, wrap, narrow layout, navigation commands, the
  missing target, and the frontmatter-only no-rail check stay.

`scripts/obsidian-record-card-workflows.cjs` is imported by `--scope full`, so keep its
hooks working; it should need no edits. Do not touch the record-writer harness.

**Runs.** On athena, under `xvfb-run -a` with `TMPDIR=/tmp` (to avoid the Chromium
socket path limit), a current `BOB_BIN`, and `BOB_CLI_DIR` set to a bob-cli checkout:

```bash
TMPDIR=/tmp xvfb-run -a node scripts/obsidian-record-card-e2e.cjs --scope full -e /snap/obsidian/current/app/obsidian -r /tmp/record-card-title.json -o /tmp/record-card-title-shots
TMPDIR=/tmp xvfb-run -a node scripts/obsidian-record-walk-e2e.cjs -e /snap/obsidian/current/app/obsidian -r /tmp/record-walk-title.json -o /tmp/record-walk-title-shots
TMPDIR=/tmp xvfb-run -a node scripts/preview-property-cards.cjs -e /snap/obsidian/current/app/obsidian -o /tmp/record-title-preview
```

Check each script's `--help` for exact flags. Hand any run longer than a few minutes to
`/sase_monitor`, and never end the turn while one is still running.

**Previews.** In `scripts/preview-property-cards.cjs`, add
`record: { word: "Record", tile: "sheet", headings: 1, glyph: <the fixture's album emoji> }`
to `ADOPTED_KINDS`. Keep `bank` and `album`. Render light and dark, at 375px and wide,
for the expanded, collapsed, and broken states. Open and inspect every record capture.

**Artifacts.** Before registering anything, read `sase_artifacts.md` through
`/sase_memory_read`. Then register the e2e reports, the preview index and report, and
the key screenshots (Reading, Live Preview, reveal, 375px) with
`sase artifact create --bead bob-cli-o6.4.1.2`. Cite them in the close note. If athena
cannot run Obsidian, record the exact reason, keep the offline wrappers green, and add a
`PROPOSED FOLLOW-UP:`.

## Docs

Before editing docs, read `docs.md` through `/sase_memory_read`. Update
`docs/record-card.md`, describing behavior only and staying within its word budget:

- the record card in the title slot;
- the RECORD kicker with the album link and noun, and the sheet tile;
- the title painting before the report;
- title ownership being independent of row hiding;
- the legacy H1 and first-line walk-line hide with cursor reveal;
- the rail docked inside the card, including during reveal.

Use the naming: record card, album card, card editor. Add a short record line to the
"Adopting the title block" note in `docs/develop/architecture.md` only if the record
adopter adds something new there.

## Gates, integration, deploy

From the bob-plugins checkout:

```bash
npm run build && npm run build:check
node --test scripts/test-ledger-tools-record-sheet.cjs scripts/test-ledger-tools-record-walk.cjs scripts/test-ledger-tools-property-card-ownership.cjs scripts/test-ledger-tools-property-card-title-adopt.cjs scripts/test-ledger-tools-property-card-host.cjs scripts/test-obsidian-record-card-e2e.cjs scripts/test-obsidian-record-walk-e2e.cjs scripts/test-command-mode-record-surface.cjs scripts/test-ledger-tools-record-card.cjs
npm test
BOB_CLI_DIR=<bob-cli checkout> npm run docs:check
```

Rerun any failure in isolation. Known flakes under load, recorded on `bob-cli-o6.1`,
`bob-cli-o6.2`, and `bob-cli-o6.4.1.1`:

- the `bob-cli-io` frees-marks budget;
- the event-link 25 ms budgets;
- the command-mode sheet-sequence timeout.

Known clean-tree failures:

- bob-cli `docs/reference/vault-format.md` over its word budget in `docs:check`;
- the album-schema-writer two-commit self-test;
- the album UI `schema-field-jump-ctrl-j` check.

A failure that reproduces identically on a clean `origin/master` gets a
`PROPOSED FOLLOW-UP:` (citing any tracking bead) and does not block this phase.

**Integration.**

1. Fetch and merge `origin/master`, keeping both sides' `src/fragments.json` and
   `package.json` `test:unit` entries.
2. Regenerate `main.js`; never merge it.
3. Take the next free minor version in `manifest.json` (1.148.0 unless it is taken), run
   `npm run docs:gen`, and rebuild.
4. Rerun the gates on the merged tree.

**Deploy** only bob-ledger-tools, from the opened checkout: a dry run first, then the
real sync.

```bash
bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools --dry-run
bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools
```

Never pass `-F` on your own. If sync refuses, follow "When sync refuses an overwrite" in
`docs/develop/deployment.md` and `AGENTS.md`. After the sync, confirm the vault's
`main.js`, `styles.css`, and `manifest.json` match the build byte for byte.

Commits go through SASE finalization. Declare the bob-plugins changes through
`/sase_final` with a commit message such as
`feat(ledger-tools): record card owns the title block from the title slot`. The message
notes that the kit change (dropping the legacy owned-row class) is a sanctioned
migration and that the other kit use is additive.

## Close

1. Run `sase bead epic-symbols bob-cli-o6.4.1.2` (empty at planning time) and resolve
   any leftovers.
2. Close only this bead with `sase bead close bob-cli-o6.4.1.2 --note "<…>"`. The note
   lists:
   - the commit and version;
   - the gate results, citing any known flakes;
   - each changed e2e and unit expectation;
   - the artifact refs for the e2e reports, previews, and screenshots;
   - the selector-consumer audit result;
   - the deploy byte-identity check.

Submitting this plan alone completes nothing.
