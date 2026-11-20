---
tier: epic
title:
  "Collection cards adopt the title block: album, record, and bank cards wear their
  species, and album and record cards own the note title from the title slot"
goal:
  On desktop Obsidian, every album note and record note opens on its card in the title
  slot above native Properties, and that card is the note's only visible title. The
  album card paints its title from local data at once, hides its redundant H1 at render
  time, and folds its schema to one line. The record card carries the RECORD kicker, a
  sheet tile, and paper material, with its walk rail docked at its bottom. The bank
  Setup card gains the BANK kicker and tile. The Record Card and album card editor
  headers show the species tile and word. Every failure restores native rendering.
parent_bead: bob-cli-o6.4
phases:
  - id: album-card
    title:
      Album card becomes the title block and owns the album title from the title slot
    depends_on: []
    size: large
    description:
      "album-card: in bob-ledger-tools, give notebookHeaderNoteTitle the album title
      rule, paint the album card's title block (stack tile, ALBUM kicker with owner
      links, title, summary subtitle, actions) from local data before the report, fold
      the schema to one line by default, mount the card in the title slot with title
      ownership through a reusable adopter glue fragment (body fallback otherwise),
      update unit, preview, and album e2e checks, update docs, bump, deploy, and run the
      album e2e on athena."
  - id: record-card
    title:
      Record card wears the RECORD kicker and sheet tile, mounts in the title slot, and
      migrates its title selectors onto the kit
    depends_on:
      - album-card
    size: large
    description:
      "record-card: in bob-ledger-tools, rebuild the record sheet header as the kit
      title block (RECORD kicker with album link and noun, sheet tile, paper material),
      mount it in the title slot through the album-card adopter glue with the walk rail
      docked inside and the first-line walk line hidden by a record title recognizer
      with reveal, move the inline-title and owned-row hiding onto kit classes, update
      unit, preview, and record e2e checks (never the record-writer harness), update
      docs, bump, deploy, and run the record e2e scripts on athena."
  - id: bank-and-editors
    title: Bank Setup card and the two card editors show the species tile and word
    depends_on: []
    size: medium
    description:
      'bank-and-editors: in bob-ledger-tools, add an additive kit identity badge (tile
      plus species word), put it on the bank Setup card head row with
      data-card-kind="bank" and an unchanged "Bank setup" aria-label, and on the Record
      Card and album card editor (Schema Card) modal headers, update unit, preview, and
      bank e2e checks and the docs header sentences, bump, deploy, and run the bank e2e
      on athena.'
proposed_by: bbugyi200.athena.bob-cli-o6.4
proposed_with: claude/opus
create_time: 2026-11-20 13:42:57
status: wip
---

- **PROMPT:**
  [prompts/202611/collection_cards_title_block.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/collection_cards_title_block.md)
- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)

# Collection cards adopt the title block

## Scope and completion

This child epic completes **bob-cli-o6.4** (`collection-cards`), a phase of the epic
**bob-cli-o6** ("Property cards become the note title block"). The phase is `large` and
had no earlier work or notes when this plan was written, so it splits into three phases
that each fit one coding agent:

- `album-card` builds the reusable adopter glue (slot mount plus title ownership plus
  body fallback) with the album card as its first adopter.
- `record-card` reuses that glue for the record sheet, so it runs after `album-card`.
- `bank-and-editors` touches different fragments and runs in parallel from the start.

The child epic's land agent closes **bob-cli-o6.4** after its phases close. Phase
workers close only their own beads. Nobody in this epic closes **bob-cli-o6**, its other
phases (`bob-cli-o6.5`, `bob-cli-o6.6`, `bob-cli-o6.7`), or any `bob-cli-ms` bead.
Discoveries go on the worker's own bead as `PROPOSED FOLLOW-UP:` notes; never create
beads.

The outer epic's DECISIONS are final. `memory_glossary_cards = yes` belongs to the outer
`glossary` phase (`bob-cli-o6.7`), not to this epic, and
`memory_decision_title_block = no`. **No phase of this epic edits any memory file.**

Out of scope here and owned by `bob-cli-o6.6`: the new real-Obsidian title-block
walkthrough (`obsidian-title-block-e2e.cjs`), the full cross-species preview matrix,
grayscale and forced-colors proofs, and the live `bob albums tidy`. Day and period cards
are `bob-cli-o6.5`, which runs in parallel in the same plugin (see "Shared rules").

## Read first, through audited interfaces

- `sase artifact read plan:202611/property_card_title_block.md "<why>"`: the outer
  epic's **design contract**. Its sections "The rule", "One skeleton for every card",
  "The species" (Album, Record, and Bank rows), "States", "Title ownership: hide,
  reveal, restore", "Placement", "Accessibility, sizing, and motion", "Interaction", and
  "Naming" are binding requirements for every phase here.
- `sase bead read bob-cli-o6.1 -r "<why>"`, note #1: **TITLE SLOT: GO** for both Live
  Preview and Reading, with no surface falling back. It also gives the primitive's API
  and adopter notes:
  - `propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost })`
    returns a release handle, or `null`, which means use the body fallback.
  - In Live Preview, `scope` is the leaf's `.cm-editor` (`view.editor.cm.dom`). In
    Reading it is the `.markdown-preview-view` child of `previewMode.containerEl`.
  - `view` is the CodeMirror `EditorView` in Live Preview and `previewMode.renderer` in
    Reading.
  - Expect `onLost("path-changed")` on every file switch within a leaf, and remount from
    `onLost`.
- `sase bead read bob-cli-o6.2 -r "<why>"`, note #1: the identity kit API (fragments
  `240f`–`240i`):
  - species: `propertyCardSpecies`, `propertyCardApplyIdentity`;
  - builders: `propertyCardBuildGlyphTile`, `propertyCardBuildTitleBlock`,
    `propertyCardBuildRail`;
  - range and recognizers: `propertyCardTitleSourceRange*`,
    `propertyCardAlbumTitleRecognizer`;
  - ownership: `propertyCardOwnTitle`, `propertyCardOwnTitleInEditor`,
    `propertyCardOwnTitleInReading`, `propertyCardTitleReleaseAll`;
  - the `createPropertyCardHost({ titleOwnership: true })` opt-in, which registers the
    kit StateField once per plugin.

  Ownership handles are release functions carrying `.generation`. Editor ownership
  returns `null` in Source mode or when the field is not installed. Reading ownership
  bumps the generation on every call, so keep the latest handle.

- In bob-plugins: `AGENTS.md`, `docs/develop/architecture.md` (the "Title block kit" and
  "Title slot" sections), and `docs/develop/deployment.md` ("When sync refuses an
  overwrite").
- `sase bead read bob-cli-ms.8 -r "<why>"`, note #5: the Record Card epic's pending
  landing tale repairs `scripts/obsidian-record-writer-e2e.cjs`. Recheck `bob-cli-ms`
  and `bob-cli-ms.8` at the start of each phase. While either is open, **do not edit
  `scripts/obsidian-record-writer-e2e.cjs` or
  `scripts/test-obsidian-record-writer-e2e.cjs`**. Running them is fine. They assert no
  sheet DOM.

## What exists today (bob-plugins `c5f5a1b`, bob-ledger-tools 1.145.0)

Paths are relative to `plugins/bob-ledger-tools/` in the bob-plugins repo. Each phase
re-verifies these facts before relying on them, because other work keeps landing.

**The kit is landed but unused.** No adapter calls `propertyCardMountInTitleSlot`
(`src/240e-…`), `propertyCardOwnTitle*` (`src/240i-…`), `propertyCardBuildTitleBlock`,
`propertyCardBuildRail` (`src/240g-…`), or `propertyCardApplyIdentity` (`src/240f-…`).
The kit CSS section is at the end of `styles.css`:

- it hides `.inline-title` under `bob-property-card-owns-title` and hides
  `.bob-property-card-title-source`;
- it has `[data-card-kind="album"]` and `[data-card-kind="record"]` material rules;
- it has the stack, sheet, and plain tile forms;
- a comment says the record `hides-properties` selectors stay as aliases until this
  phase migrates them, but no alias selector exists.

**Album card.**

- Model: `src/241u-plugin-album-header-model.js` (`albumHeaderModel`,
  `albumHeaderOwnerChain`). The owner chain needs only `entry.parent`, so it can run
  from frontmatter.
- DOM: `src/241v-plugin-album-header.js` (`albumHeaderBuildEl`).
- Schema summary and chips: `src/241w-…`, `src/241x-…`.

The root is `section.bob-album-header.bob-property-card`, with
`aria-label="<title> album details"`. It holds:

- a crumb `nav.bob-album-header-nav`. The album title appears only as its current crumb.
- `div.bob-album-header-actions` (`＋ <noun>` copy and `⊞ Schema`);
- a schema summary button (`▾ Restaurant · 2 fields · …`). It defaults to expanded
  (`albumHeaderDefaultExpanded` returns `true`), and its per-session toggle is
  `plugin.albumHeaderExpanded`.
- the chip card.

Emoji and `summary` are never rendered. Until the `bob albums list` report arrives the
model is `null`, so nothing paints, and the H1 and inline title stay visible.

Placement:

- Reading inserts the card after the first non-code H1 (`albumHeaderReadingAnchor`).
- Live Preview adds an inline widget at the end of the first `# ` line through the
  host's `Prec.highest` ViewPlugin (`src/240d-…`, `buildAlbumHeaderDecorations`).
- The `record` Properties row is hidden through `propertyCardOwnAlbumKeys` after a
  report-backed paint.
- No code hides the album H1 or the inline title.

**`notebookHeaderNoteTitle`** (`src/241e-…`) applies frontmatter `title:` only to
`type: record` notes, then falls back to the first H1, the first alias, and the stem.
bob already prefers a valid album `title:` (one nonblank line) over the H1, alias, and
stem (bob-cli `src/native/albums/mod.rs`, landed by `bob-cli-o6.3`).

**Record sheet.**

- Model: `src/242i-…` (`recordSheetModel`). The title is frontmatter `title:`, then H1,
  then stem. Crumbs are the album owner chain, then the album, then the noun.
- DOM: `src/242j-…` (`recordSheetBuildEl`):
  `header.bob-record-sheet-header > .bob-record-sheet-title-row > [.bob-record-sheet-title[role=heading][aria-level=1], .bob-record-sheet-tools]`,
  plus `nav.bob-record-sheet-nav` crumbs. The collapsed form uses
  `.bob-record-sheet-summary`.
- Host: `src/242k-…`, `src/242ka-…` (owned keys).
- Walk: `src/242t-…`, `src/242u-…`, `src/242v-…`. When the walk line is the first body
  line, Live Preview replaces it with a `.bob-record-walk-dock` widget that holds the
  sheet and the rail. Reading appends the rail inside the sheet and hides the paragraph
  with `bob-record-walk-source`.
- Inline-title hiding is record-specific CSS keyed on
  `bob-record-sheet-hides-properties` (about `styles.css:11168-11175`), so it is tied to
  the Properties-row hide gate.

**Bank Setup card.** It is built in `src/248b-…` (`bankSetupBuildEl`) and mounted in
`src/248c-…`. Its root is `section.bob-bank-setup.bob-property-card` with
`aria-label="Bank setup"` exactly; the bank e2e asserts that. The head row is
`div.bob-bank-setup-head` with the `▾ Setup` toggle. 🏦 appears only in the collapsed
line.

**Editors.** Neither header carries a hue or `.bob-property-card`.

- Record Card header: `recordCardPaintHeader` in `src/242o-…`, with the title text from
  `recordCardTitle` in `src/242n-…` (`<emoji> <title>`).
- Album card editor (Schema Card) header: `albumSchemaCardPaintHeader` in `src/241yc-…`
  (958 lines), with the title from `albumSchemaCardTitle` in
  `src/241y-plugin-album-schema-card-dom.js`. `src/241y-plugin-album-schema-card.js` is
  at 989 of 1000 lines, so do not grow it.

**Previews.** `scripts/preview-property-cards.cjs` renders the real
`albumHeaderBuildEl`, `recordSheetBuildEl`, and `bankSetupBuildEl`. It **fails unless
every case is pixel-identical** to the baseline CSS (`-b`, default `36d4801`).
`scripts/preview-title-block.cjs` renders kit fixtures. Both drive Obsidian's Electron
over CDP and run under `xvfb-run -a`.

## Shared rules for every phase

- **Contract first.** Implement the outer design contract. Where this plan names a
  function or file, treat it as a pointer, not a script. Re-verify it against the
  current tree.
- **Fragments.** Edit `src/` fragments only, rebuild with `npm run build`, and keep each
  hand-edited fragment at or below 1000 lines.
  - Kit changes are **additive and backward compatible**. Say so in the commit.
  - Put new kit code in **new** fragments after `240i`, choosing the next free name when
    you implement, rather than growing `240g`, `240h`, or `240i`, which `bob-cli-o6.5`
    may also touch.
  - Export new helpers through a `374*` exports fragment.
- **Stable hooks.** Keep the root classes and data attributes other code queries:
  - `.bob-album-header`, `.bob-record-sheet`, and `.bob-bank-setup`;
  - `data-album-header`, `data-record-sheet`, and `data-record-sheet-focus`;
  - `.bob-record-sheet-row[data-record-key]`, `.bob-album-header-copy`, and
    `.bob-album-header-schema`;
  - `bob-record-sheet-hides-properties` as the scope's row-hide marker.

  Before moving a card out of the body, grep every plugin under `plugins/` and every
  `scripts/` file for each selector the change affects. Confirm every consumer still
  resolves the card once it lives in `.cm-sizer` or `.mod-header`. Consumers include
  `bob-navigation-hotkeys`, `bob-command-mode`, the walker footer, the `==` dispatcher
  in `src/248m-…`, and Command Mode surfaces.

- **Interaction unchanged.** `==`, `Ctrl+Shift+P`, the dispatcher order, every card
  editor's fields and regions, and every keymap stay the same. Kicker and rail links
  behave like the Markdown links they mirror, including modifier keys for tabs and
  splits.
- **Accessibility.** In a pane where the card owns the title there is exactly one
  `aria-level="1"` node. Any card that does **not** own the title (an embed, the body
  fallback, or a failed ownership) drops `role="heading"` and `aria-level` from its kit
  title (an additive kit helper), so it never doubles a visible H1. The accessible name
  leads with the species word (`propertyCardApplyIdentity`).
- **Parallel work in bob-plugins.** `bob-cli-o6.5` (day and period cards) and this
  epic's own phases edit the same plugin. Before every commit and every sync:
  - fetch and merge `origin/master`;
  - keep both sides' `src/fragments.json` and `package.json` `test:unit` entries;
  - regenerate `main.js` instead of merging it;
  - take the next free `manifest.json` minor version and rerun `npm run docs:gen`;
  - rebuild.

  If a sibling phase has already landed a helper you need (adopter glue, heading
  demotion, identity badge), reuse it rather than adding a second one.

- **Gates.** `npm run build`, `npm run build:check`, `npm test`, and
  `npm run docs:check` (set `BOB_CLI_DIR` to a bob-cli checkout). Known load-sensitive
  perf budgets and timeouts flake on athena under heavy load: the `bob-cli-io`
  frees-marks budget, the event-link 25 ms budgets, and the command-mode sheet-sequence
  timeout, recorded on `bob-cli-o6.1` and `bob-cli-o6.2`.
  - Rerun a failing test in isolation.
  - A failure that reproduces identically on a clean `origin/master` is recorded as a
    `PROPOSED FOLLOW-UP:` citing any tracking bead, and does not block the phase.
- **Real Obsidian.** Run the phase's existing e2e scripts on athena with `xvfb-run -a`
  and a current `BOB_BIN`. Hand any run longer than a few minutes to `/sase_monitor`.
  - Register each report and the key screenshots with
    `sase artifact create --bead <your bead>`. Read `sase_artifacts.md` through
    `/sase_memory_read` first.
  - Cite the artifacts in your closing note.
  - If athena cannot run Obsidian, record the run as skipped with the exact reason, keep
    the offline wrappers green, and leave a `PROPOSED FOLLOW-UP:`.
- **Docs.** Before touching docs, read `docs.md` with `/sase_memory_read`. Describe
  behavior only, keep each page under its word budget, and use the contract's naming:
  album card, record card, bank card, and card editor. "Schema card" survives only as
  the album card editor's former name.
- **Deploy.** Bump `plugins/bob-ledger-tools/manifest.json` (minor) and commit, then run
  `bob plugins sync -p bob-ledger-tools`: first a dry run, then the real sync. Never
  pass `-F` on your own. Follow the AGENTS.md refusal procedure. Confirm the vault's
  `main.js`, `styles.css`, and `manifest.json` match the build byte for byte.
- **No memory edits. No new beads.**

## Phase `album-card`

**Goal.** Opening `eat.md` on desktop shows one title: the album card's, in the title
slot above Properties. It paints immediately from local data. The H1 is hidden at render
time with reveal, and the schema is folded to one line.

**1. Title source.** Give `notebookHeaderNoteTitle` the album rule. For a note whose
`type` resolves to album, a frontmatter `title:` that is a string with one nonblank line
(trimmed) wins over the H1, alias, and stem, matching bob's precedence. An invalid
`title:` (not a string, blank, or multi-line) falls through, as in bob.

Grep the other album-title readers and align any that disagree:

- `src/124-links-walker-footer.js` around line 82;
- `src/291h-…` around line 17;
- `recordSheetModel` crumbs;
- `albumHeaderOwnerChain`.

Album crumbs, record kickers, and the card must then agree before and after
`bob albums tidy`.

**2. Local identity.** Add a pure model (for example `albumHeaderIdentityModel`) that
builds `{ kind: "album", path, title, emoji, summary, owners }` from the metadata cache
alone:

- `title` comes from the rule above;
- `emoji` is the frontmatter `emoji`;
- `summary` is the frontmatter `summary` when it is a plain string;
- `owners` comes from `albumHeaderOwnerChain` fed with the frontmatter `parent`.

It returns `null` unless the note is album-typed. The title block paints from this model
with no report.

When the report entry arrives, the report-backed body joins **inside the same mounted
node**, with no remount and no flash. The body holds the schema summary and chips, the
actions, the issues, and the ancestry note. The report's validated `summary` may refine
the subtitle, but the title never flips to a different source.

The `record` Properties row hides only once the report-backed body is painted, as today.
The broken and invalid-schema body states stay as today.

**3. Title block DOM** (`albumHeaderBuildEl`):

- Call `propertyCardApplyIdentity(root, { kind: "album", title })`. The aria-label
  becomes `Album: <title>`.
- The first child is `propertyCardBuildTitleBlock`:
  - a stack tile showing the album `emoji`, else the `library` icon;
  - kicker parts are the owner chain as links, opened with today's crumb link handler
    (modifiers kept). Keep the `bob-album-header-crumb` class on those links so the e2e
    probes still resolve.
  - the title;
  - the summary as the subtitle.
- Move `＋ <noun>` and `⊞ Schema` into the kicker row's actions slot, keeping their
  classes. Remove the crumb `nav`.
- Keep the root `section.bob-album-header.bob-property-card` and its data attributes.

**4. Schema fold.** A healthy schema's summary becomes one collapsed line by default:
`▸ Schema · 5 fields · grouped by Location · dated notes · 104 restaurants`. That is the
field count, then the grouping clause, then dated notes, then the record-count phrase.
The per-session toggle expands it.

Invalid and empty schemas keep today's expanded default and their text, so their issues
and empty-state copy stay visible. Grep other users of `albumHeaderSummaryText` (such as
the Schema Card) and keep their output unchanged, or update them deliberately.

**5. Adopter glue (reusable, new kit fragment).** One helper that adapters call with:

`(scope, node, { path, kind, view, recognizer, currentPath, onLost })`

It mounts `node` through `propertyCardMountInTitleSlot`, then owns the title through
`propertyCardOwnTitle(scope, { path, recognizer, view })`. It returns
`{ mode, release }`, where `mode` is `"slot"`, or `"fallback"` when the slot returns
`null`. It tracks the latest handles per scope.

Behavior:

- It remounts from `onLost` (`path-changed` and the other reasons).
- It drops both the slot and title ownership on release, teardown, a switch to Source
  mode, file change, and failed paint, so native rendering returns in the same frame.
- It demotes the kit heading whenever ownership is not held.
- Live Preview mounts run outside the CodeMirror update cycle (deferred, as the kit's
  ownership effect is). Never dispatch inside an update.

Check whether `bob-cli-o6.5` has already landed an equivalent helper. If it has, adopt
it and extend it additively instead. Write it so `record-card` can reuse it unchanged.

Also check that `propertyCardTitleReleaseAll()` in host dispose cannot strip another
host's ownership at runtime. If any host is disposed apart from plugin unload, scope the
release to that host additively.

**6. Mounting.**

- Opt the album host into `titleOwnership: true`.
- Live Preview: `buildAlbumHeaderDecorations` mounts the card through the glue and
  returns no card widget. Its recognizer is `propertyCardAlbumTitleRecognizer(title)`.
  When the glue reports `fallback` (embeds, or any surface without a slot), keep today's
  widget path unchanged.
- Reading: `paintAlbumHeaderReading` mounts into `.mod-header` through the glue. Its
  fallback is today's insert after the H1.
- An album with `title:` and no H1 (the shape after tidy) hides nothing in the body.
- An H1 that differs from the title, follows prose, or sits in a fence stays visible.

**7. CSS.**

- the album material ("collection cover" wash, heavier header band);
- the actions in the kicker row;
- the folded summary line;
- narrow-width rules (375px), focus rings, `forced-colors`, and reduced motion, per the
  contract.

Remove dead `.bob-album-header-nav` rules.

**8. Tests.**

- Update `scripts/test-ledger-tools-album-header.cjs` for the new DOM, the aria-label,
  the fold default, and the summary text.
- Add unit tests for the album title rule: a valid `title:`; blank, multi-line, and
  non-string `title:` falling through; then the H1, the alias, and the stem.
- Add unit tests for local paint before the report and for the body joining without a
  remount.
- Add unit tests for the glue on the fake DOM: slot success owns the title; slot `null`
  falls back; `onLost("path-changed")` remounts; release restores native rendering;
  generations are respected; the heading demotes; one `aria-level=1` exists.
- Update `scripts/test-ledger-tools-property-card-host.cjs` if the extension count
  changes.

**9. Album e2e.** Update `scripts/obsidian-album-ui-e2e-lib.cjs` and
`scripts/obsidian-album-ui-e2e.cjs`, plus `scripts/obsidian-album-properties-e2e.cjs` if
needed:

- the header-integrity probe, crumbs, and copy and schema buttons;
- the source-view font check, applied to the card root, not the new heading;
- the H1 hidden in the owning view and revealed when the cursor touches it;
- the `record` row hidden after the report.

Keep the offline wrappers in `npm test` green. Run the album e2e scripts on athena and
attach the reports.

**10. Previews.** In `scripts/preview-property-cards.cjs`, add an adopted-kinds set.
Adopted kinds leave the baseline pixel-identity gate and instead get structural
assertions: `data-card-kind`, one `aria-level=1`, the species word, and the tile form.
Add `album`, and keep light and dark × 375px and wide captures. Render and inspect every
album capture.

**11. Docs and deploy.**

- `docs/albums.md`: the album card's title block, the title source, the H1 hidden at
  render time and never deleted, the fold, and the actions.
- `docs/develop/architecture.md`: a short "Adopting the title block" note on the glue.

Then bump, run the gates, commit, and sync.

## Phase `record-card`

**Goal.** Opening a record such as `eat/<restaurant>.md` shows the record card in the
title slot:

- a sheet tile;
- `RECORD · <Album title link> · <noun>`;
- the title;
- paper material;
- the walk rail docked at the card's bottom;
- the first-line walk line hidden at render time with reveal.

Fields, ghosts, the fold, and the created footer are unchanged.

**1. Title block DOM** (`recordSheetBuildEl`):

- Replace the header's title row and crumb `nav` with `propertyCardBuildTitleBlock`:
  - a sheet tile with the album emoji, else `file-text`;
  - kicker parts: the album title link (today's crumb handler, modifiers kept), then the
    noun;
  - the title. Keep the `bob-record-sheet-title` class on the kit heading.
- Put `div.bob-record-sheet-tools` (edit, collapse) in the actions slot.
- Call `propertyCardApplyIdentity(root, { kind: "record", title })`. The aria-label
  becomes `Record: <title>`.
- Collapsed records keep the kicker, the title, and the chip summary line
  (`.bob-record-sheet-summary` without the crumb nav).
- Broken schemas show the title block, then today's warning and YAML button.
- Area owner crumbs leave the record card, per the contract.

**2. Local paint.** The title block paints from local data before any report:

- the title, from today's order;
- the album link and emoji, through the metadata cache and the `album-card` title rule.

The noun and the report-backed body join inside the same node when the report arrives.
The Properties-row hide keeps its existing gate (`hideEligible`).

**3. Mounting.**

- Opt the record host into `titleOwnership: true`. Mount through the `album-card` glue.
- The rail goes inside the card at the bottom (`has-record-walk`) in both modes.
- Add a record title recognizer as a new additive function in a record fragment:
  - `requireHeading: false`;
  - `heading` accepts an H1 equal to the record title (legacy records);
  - `follower` accepts the walk line that `recordWalkClassify` reports as `current` or
    `stale`.

  The kit range then covers that walk line only when it is the first non-blank body line
  (or directly follows the recognized H1). Selection touch reveals the raw line while
  the card and its rail stay in the slot.

- In Reading, feed the follower `propertyCardReadingLineText`'s `[[href|text]]` form, so
  the first-line walk paragraph hides through kit tags and returns when ownership drops.
- A walk line that is not first keeps today's in-place rail replacement.
- The body fallback keeps today's docked widget path unchanged.

**4. Selector migration.**

- The inline title hides through the kit's `bob-property-card-owns-title`. Title
  ownership is now independent of the row gate (contract A9).
- Delete the record-specific inline-title rules.
- Owned rows rely on the shared `bob-property-card-owned-property` rule. Drop the legacy
  `bob-record-sheet-owned-property` CSS and the tagging in `src/240b-…` only after grep
  shows no remaining consumer.
- Keep the `bob-record-sheet-hides-properties` scope marker for row hiding and for its
  readers.

**5. Tests.**

- Update `scripts/test-ledger-tools-record-sheet.cjs`:
  - the fake Live Preview needs `.cm-contentContainer` for the slot;
  - the title, summary, tools, and aria-label;
  - the CSS literal checks now point at kit selectors;
  - docking assertions move to slot mode, with the fallback kept;
  - the owned-row class.
- Add recognizer unit tests:
  - the walk line first;
  - after an H1;
  - stale;
  - not first;
  - fenced;
  - reveal boundaries: cursor on the line, the line after, and multiple selections.

**6. Record e2e.** In `scripts/obsidian-record-card-e2e-lib.cjs`, split
`assertNativeVisibility` into two expectations:

- **title ownership:** the inline title is hidden whenever the card owns the title in
  the owning view;
- **row hide:** the existing `shouldHide` gate.

Update the per-state expectations in `scripts/obsidian-record-card-e2e.cjs` to match,
listing each changed expectation in your closing note. Change `orderBeforeFirstHeading`
into "card in the slot before Properties".

In `scripts/obsidian-record-walk-e2e.cjs`: the rail sits inside the card bottom, the
walk paragraph is hidden, and the Live Preview raw line reveals.

The workflow scripts keep their hooks. Leave the record-writer harness untouched (see
"Read first"). Run the record card, record walk, and record workflow e2e scripts on
athena and attach the reports.

**7. Previews, docs, and deploy.**

- Add `record` to the adopted-kinds set in `preview-property-cards.cjs`, then render and
  inspect.
- `docs/record-card.md`: the card in the title slot, the kicker, title ownership and
  reveal, and the walk line hide.

Then bump, run the gates, commit, and sync.

## Phase `bank-and-editors`

**Goal.** The bank Setup card and both card editors name their species with the shared
tile and word. The `=` climb from a record's Record Card to its album's card editor
visibly changes species.

**1. Identity badge (additive kit, new fragment).** Add a helper such as
`propertyCardBuildIdentityBadge(doc, { kind, emoji?, icon? })`. It returns a
`span.bob-property-card-identity[data-card-kind]` holding the glyph tile and the species
word, uppercased by CSS like the kicker. Add CSS that gives the badge its own
`--bob-property-card-accent` and ink fallbacks, so it renders correctly outside
`.bob-property-card` (modal headers). Unit-test it.

**2. Bank Setup card** (`bankSetupBuildEl`):

- Set `data-card-kind="bank"` directly. Do **not** call `propertyCardApplyIdentity`: the
  aria-label stays exactly `Bank setup`, which already leads with the species word.
- Lead the head row with the badge: a plain tile with the 🏦 bank mark and the word
  Bank.
- No title move. The `▾ Setup` toggle, the body, and the collapsed line are unchanged.
- Make sure the kit's `.bob-property-card[data-card-kind]` rules (container query and
  ink) change nothing in the bank body. Add `[data-card-kind="bank"]` overrides if they
  do.

**3. Record Card header** (`recordCardPaintHeader`): add the badge with the sheet tile
showing the album emoji, and the word Record. The `h2` shows the title without the emoji
prefix, because the tile carries it. Change only the header display. Grep the other
users of `recordCardTitle` and keep their text. The subtitle is unchanged.

**4. Album card editor header** (`albumSchemaCardPaintHeader` in `241yc`): add the badge
with the stack tile showing the album emoji, and the word Album. The `h2` drops the
emoji. Keep `241yc` at or below 1000 lines, moving helpers out if needed, and do not
grow `241y-plugin-album-schema-card.js`.

**5. Tests.** Update:

- `scripts/test-ledger-tools-bank-setup*.cjs`;
- `scripts/test-ledger-tools-record-card.cjs` (header text);
- `scripts/test-ledger-tools-album-schema-card-shell.cjs` (header text);
- the Command Mode surface tests, if they are affected.

In `scripts/obsidian-bank-ui-e2e-lib.cjs` and `scripts/obsidian-bank-ui-e2e-setup.cjs`,
keep the `Bank setup` aria-label and the "header follows the bank H1" checks true. Run
the bank e2e on athena and attach the report.

**6. Previews, docs, and deploy.**

- Add `bank` to the adopted-kinds set in `preview-property-cards.cjs`. Introduce the set
  if `album-card` has not landed yet; on rebase keep both sides' kinds. Render and
  inspect the bank captures.
- Update the header sentences in `docs/albums.md` (Schema Card header) and
  `docs/record-card.md` (Record Card header), plus any bank docs sentence that describes
  the Setup head row.

Then bump, run the gates, commit, and sync.

## Landing this epic

The land agent:

- audits each phase against the outer contract's Album, Record, and Bank rows, the
  States and Title-ownership sections, and the editor-header rule;
- re-runs the bob-plugins gates on the integrated `origin/master`;
- renders `scripts/preview-property-cards.cjs` under `xvfb-run -a` and inspects every
  album, record, and bank capture;
- confirms that each phase attached its real-Obsidian e2e report, and reruns any e2e
  whose code changed after that report;
- confirms the vault deploy of `bob-ledger-tools` is byte-identical to the bob-plugins
  `origin/master` build;
- triages the phases' `PROPOSED FOLLOW-UP:` notes;
- closes this child epic.

It then runs `sase bead epic-symbols bob-cli-o6.4`, resolving or re-keying any entries
to a still-open bead. It closes **bob-cli-o6.4** with a note listing what it verified:
the commits, the version, the e2e and preview artifacts, and the gate results with any
cited known flakes.

It does **not** close `bob-cli-o6` or any other `bob-cli-o6` phase. The cross-species
title-block proof, the full preview matrix, and the live album tidy remain
`bob-cli-o6.6`'s work.
