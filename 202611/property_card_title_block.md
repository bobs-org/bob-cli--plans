---
tier: epic
title:
  "Property cards become the note title block: one title, a clear species, and a docked
  rail on album, record, day, and period notes"
goal:
  On desktop Obsidian, every album note, record note, canonical daily note, and week,
  quarter, and year note opens on its property card, and that card is the note's only
  visible title. The card leads with a species glyph tile and a kicker word (ALBUM,
  RECORD, DAY, WEEK, QUARTER, YEAR), so an album, a record, and a day can never be
  mistaken for each other, even in grayscale. The redundant H1, inline title, and nav
  line are hidden at render time, never deleted (except album H1s, which move into a
  bob-owned `title:` key). Every failure restores native rendering.
phases:
  - id: title-slot
    title: Title-slot spike and kit mount primitive (go/no-go)
    depends_on: []
    size: large
    description:
      "title-slot: build the kit primitive that mounts a property card in Obsidian's
      title slot (after the hidden inline title, before native Properties) in Live
      Preview and Reading, prove it with a real-Obsidian CDP spike on athena across
      switches, splits, re-renders, settings, and teardown, record a GO or NO-GO verdict
      with evidence on this phase bead, and deploy the unused primitive."
  - id: identity-kit
    title:
      "Identity kit: species tile, kicker, title row, rail, tokens, and title ownership"
    depends_on: []
    size: large
    description:
      "identity-kit: add the shared species registry, glyph tiles, kicker/title/subtitle
      row, docked rail with optional children strip, data-card-kind tokens and the
      temporal accent, and generalized title ownership (inline-title hide, Live Preview
      block hide with reveal-on-selection-touch, Reading source hide) to the
      bob-ledger-tools kit, with unit tests and a preview matrix; no card adopts it yet."
  - id: album-title-cli
    title: Album `title:` key, H1-less album create, and `bob albums tidy`
    depends_on: []
    size: large
    description:
      "album-title-cli: in bob-cli, make frontmatter `title:` the first title source for
      album notes in every reader, make bob albums create write `title:` and no H1, and
      add bob albums tidy (dry-run by default) that backfills `title:` from the H1 and
      removes the now-redundant H1 under record-tidy keep rules, with registry,
      completion, docs, and no-churn sync tests. No live-vault apply in this phase."
  - id: collection-cards
    title: Album, record, and bank cards adopt the title block
    depends_on:
      - title-slot
      - identity-kit
    size: large
    description:
      "collection-cards: the album card and record card adopt the identity kit and the
      title slot (or body fallback per the title-slot verdict), the album card renders
      its title from local data and hides its redundant H1, the album folds its schema
      to one line, the bank Setup card gains the BANK kicker, the Record Card and album
      card editor headers carry the species, docs follow, and the plugins deploy."
  - id: day-period-cards
    title: Day and period cards always mount as the title block with a docked rail
    depends_on:
      - title-slot
      - identity-kit
    size: large
    description:
      "day-period-cards: the intention card becomes the day card and period card: it
      always mounts on canonical daily, week, quarter, and year notes with a
      path-computed title, leaf tile, temporal accent, state pill, docked nav rail with
      children strip, and a set-intentions ghost on current and future subjects;
      recognized H1, nav, and child-link lines hide with reveal; content bands stay
      gated; docs, editor header, and deploy follow."
  - id: obsidian-proof
    title: Real-Obsidian proof, preview matrix, live album tidy, and final deploy
    depends_on:
      - collection-cards
      - day-period-cards
      - album-title-cli
    size: large
    description:
      "obsidian-proof: add and run a real-Obsidian title-block walkthrough on athena
      (ownership, reveal, fallback, heading links, Outline, byte neutrality, grayscale,
      forced colors), render the full preview matrix, fix what it finds, run the live
      bob albums tidy only when every host runs a bob that reads album `title:`, pass
      every gate, and deploy the final tree."
  - id: glossary
    title: Glossary updates the reviewer accepted
    depends_on:
      - obsidian-proof
    size: small
    description:
      "glossary: apply only the memory decisions answered yes (glossary strands and the
      optional decision record) so memory matches the shipped cards, then run sase
      memory init; if every memory decision is no, record that on this bead and change
      nothing."
decisions:
  memory_glossary_cards:
    ask:
      Update the Intention Card, Property Card, and Record Card glossary strands to
      match the shipped cards?
    memory:
      - glossary:intention-card
      - glossary:property-card
      - glossary:record-card
    default: true
    corrects:
      paints at the top of a canonical daily or period note in Reading view and Live
      Preview when any of `intentions`, `summary`, or `reflection` is nonempty.
    answer: true
  memory_decision_title_block:
    ask:
      Add a decisions strand that property cards own the title block and title bytes
      stay per species?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.1eh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 11:49:14
status: wip
---

- **PROMPT:**
  [prompts/202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_card_title_block.md)

# Property cards become the note title block

## Why this epic exists

Bryan asked to finish unifying the property-card family: stop rendering the H1 on album
notes and daily files, render the title in the note's card the way record notes already
do, and make it obvious from the title whether a note is an album, a record, or a daily
file. Today an album shows three titles (the filename inline title `eat`, the H1
`# Restaurants`, and a crumb ending in `Restaurants`), and albums, records, and day
cards share one rounded, area-washed chrome, so they look alike.

Bryan accepted every requirement in the consolidated research report except memory
edits, which only the `memory_*` decisions in this plan can authorize. Bryan asked the
planner to lead the design. The design contract below makes the calls the research left
open, and every phase implements against that contract.

Read first, through audited interfaces:

- `sase artifact read research:202611/property_card_title_block/property_card_title_block.md "<why>"`:
  the consolidated report. Its adjustments A1–A9, resolutions 1–4, and decision defaults
  1–6 are accepted requirements. Default 7 (a decision record) is only the
  `memory_decision_title_block` decision here.
- `sase bead read bob-cli-ms -r "<why>"` and `sase bead read bob-cli-ms.8 -r "<why>"`:
  the record precedent, covering the title moving into the sheet, H1-less records,
  `bob records tidy`, the Properties-hide selectors, and the real-Obsidian proof.
- `sase bead read bob-cli-nh -r "<why>"`: the property-card kit, the intention card, and
  `bob periods update`.
- `sase bead read bob-cli-o3 -r "<why>"`: the empty-day ghost proposal.
  `day-period-cards` absorbs it.
- Before touching docs or CLI help, run `sase memory read docs.md` and
  `sase memory read cli_rules.md`. Before checking other hosts, run
  `sase memory read tailnet.md`.

## Design contract

### The rule

**The card is the note's title block. Hue says where, form says what, and words say
both.** Hue keeps its one Bob meaning, the root area or theme
(`decisions:theme-hue-is-config-pin-exempt-area-hash`). Species never gets a hue of its
own. The glyph tile's shape and the kicker word carry species, and surface material
reinforces it.

### One skeleton for every card

```text
╭──────────────────────────────────────────────────────────────────────╮
│ ┌────┐  ALBUM · Body                               ＋ Restaurant  ⊞  ▾ │ kicker row + actions
│ │ 🍽 │  Restaurants                                                  │ title: role=heading, aria-level=1
│ └────┘  Restaurants Kelly and I have tried, liked, or want to try.   │ subtitle (muted)
│ ──────────────────────────────────────────────────────────────────── │
│ species body (schema line · record fields · intention bands)         │
│ ──────────────────────────────────────────────────────────────────── │
│ ‹ Thu 19                      ↑ W47 · Nov 16 – 22              Sat 21 › │ docked rail (where nav exists)
╰──────────────────────────────────────────────────────────────────────╯
```

- **Glyph tile.** A rounded square about 2.4em, `aria-hidden`. It spans the kicker and
  title rows on the left and collapses to 1.8em at narrow widths.
- **Kicker row.** UI font at 0.72–0.76em, weight 650, letter-spacing 0.08em, uppercase.
  The species word comes first in the card's ink, then `·` and muted context, which may
  be links. Actions sit right-aligned on the same row.
- **Title.** A `div` with `role="heading"` and `aria-level="1"` that uses the record
  sheet's proven `--inline-title-*` / `--h1-*` variables. Long titles wrap; they never
  truncate.
- **Subtitle.** One muted line at UI-small size. It is omitted when empty.
- **Rail.** A hairline separator, then `‹ prev`, `↑ parent`, and `next ›` from the
  note's own nav line. Week, quarter, and year cards add a second row of children chips.
- **Shared rules.** The approved 8px radius, a 2px inline-start accent stripe, about
  16px of padding, and Obsidian theme variables everywhere. No shadows, gradients,
  banners, or nested boxes.

### The species

| Species                        | Glyph tile (form)                                                                                                  | Kicker                                     | Title (one source each, never parsed from a hidden H1)         | Subtitle                                | Material                                                                 | Rail                                           |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ | -------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------------ | ---------------------------------------------- |
| Album                          | **Stack**: the tile with two offset sheet edges behind it; shows the album `emoji`, else Obsidian's `library` icon | `ALBUM · <owner crumb>`                    | frontmatter `title:` → first H1 → first alias → humanized stem | frontmatter `summary`                   | "Collection cover": area-hue wash about 8%, slightly heavier header band | none                                           |
| Record                         | **Sheet**: one tile with a folded top-right corner; shows the album emoji, else `file-text`                        | `RECORD · <Album title link> · <noun>`     | frontmatter `title:` (unchanged)                               | none (crumb moved to kicker)            | "Paper": wash 3–4%, hairline border, area stripe                         | existing walk rail                             |
| Day                            | **Calendar leaf**: an accent band with the weekday (`FRI`) over a large date (`20`)                                | `DAY · W47 · 2026` (W47 links to the week) | from the path: `Fri, Nov 20, 2026`                             | none                                    | Temporal accent `--bob-day-accent`, no area hue                          | `‹ Thu 19 · ↑ W47 · Sat 21 ›`                  |
| Week                           | Leaf with band `WEEK` over `47`                                                                                    | `WEEK · 2026-Q4`                           | `2026-W47`                                                     | `Nov 16 – 22, 2026`                     | Same accent, slightly deeper tint                                        | `‹ W46 · ↑ 2026-Q4 · W48 ›` + day chips        |
| Quarter                        | Leaf with band `QTR` over `Q4`                                                                                     | `QUARTER · 2026`                           | `2026-Q4`                                                      | `W40–W53 · Sep 28, 2026 – Jan 3, 2027`  | Deeper still                                                             | `‹ Q3 · ↑ 2026 · 2027-Q1 ›` + week chips       |
| Year                           | Leaf with band `YEAR` over `2026`                                                                                  | `YEAR · All time`                          | `2026`                                                         | `Dec 29, 2025 – Jan 3, 2027 · 53 weeks` | Deepest                                                                  | `‹ 2025 · ↑ All time · 2027 ›` + quarter chips |
| Bank (Setup card on `bank.md`) | Existing bank mark in a plain tile                                                                                 | `BANK`                                     | Setup heading (unchanged)                                      | unchanged                               | unchanged                                                                | none                                           |

The album emoji is content, so it appears on both the album and its records. The tile
shape (stack versus sheet) and the kicker word carry species. With a uniform theme in
grayscale and identical titles, the kicker word and tile shape alone must tell the
species apart.

**State pill** (accent pill after the title): `TODAY` on today's daily note, and
`YESTERDAY` or `TOMORROW` in outline style. Current periods get `THIS WEEK`,
`THIS QUARTER`, or `THIS YEAR`. Nothing else gets a pill. The midnight rollover
(existing timer) repaints it.

**Tokens.** Each card gets a `data-card-kind="album|record|day|week|quarter|year|bank"`
attribute on `.bob-property-card`. `--bob-day-accent` is derived from the theme's cyan
(`var(--color-cyan, #00bfbc)`), mixed toward `--text-normal` until kicker text meets
4.5:1 in light and dark. Week, quarter, and year deepen only its wash. This fixes the
current day-card fallback to `--interactive-accent`, which collides with violet-area
notes. Fix the dead `.bob-record-walk-rail[data-root-hue="purple"]` rule (it should be
`violet`) in the kit phase.

### States

- **Album, report pending or broken.** The title block renders from local data at once
  (no mount flash). The body shows the existing broken or invalid-schema states. Owned
  Properties rows (`record`) hide only when the report-backed body is painted, as today.
- **Album body.** The schema chip rows fold to one line by default:
  `▸ Schema · 5 fields · grouped by Location · dated notes · 104 restaurants`. The
  existing per-session toggle expands it. `＋ <noun>` and `⊞ Schema` move into the
  kicker row's actions.
- **Record.** Fields, ghosts, the fold, and the created footer are unchanged. The crumb
  moves into the kicker. Collapsed records keep kicker, title, and the chip summary
  line.
- **Day or period with no fields.** The card still mounts with kicker, title, pill, and
  rail. On current and future subjects with no intentions, one quiet dashed row reads
  `＋ Set intentions` with a `==` keycap hint. Click, Enter, or `==` opens the card
  editor at the intentions stage. Past subjects never show it, and it owns no Properties
  rows. The intention, summary, and reflection bands and the recent-closed pending line
  keep their existing content gates.
- **Daily without a recognized H1** (the H1-less 2024 and 2025 shapes and the legacy
  2026 shape). The card mounts with the path-computed title and hides nothing in the
  body. An unrecognized H1 stays visible as authored content. Legacy body lines such as
  `day::2025-01-01 Wed` stay too.

### Title ownership: hide, reveal, restore

A card owns the title only while all of these hold: desktop Obsidian, Live Preview or
Reading, the owning leaf's own view (never an embed, hover popover, or canvas node), and
a successfully mounted title row. While it owns:

1. The view scope (`.cm-editor` or `.markdown-preview-view`) carries
   `bob-property-card-owns-title`. Kit CSS hides that view's `.inline-title` through
   direct-child selectors, generalizing the proven record selectors in `styles.css`.
2. The **recognized source title block** is hidden. It is the first body H1 when it is
   redundant, plus the blank lines and recognized nav or child-link lines that follow:
   - Album: the first H1 before any prose whose cleaned text equals the card title.
   - Day: an H1 equal to the first alias or to this date's known shapes
     (`YYYY-MM-DD ddd`, `Day: YYYY-MM-DD ddd`, `day = YYYY-MM-DD.ddd`).
   - Period: an H1 equal to the first alias or to the scaffold label.
   - Nav and child-link lines: exactly the lines `intentionCardIsNavigationLine` and
     `intentionCardIsChildLinksLine` accept. Record walk lines use `recordWalkModel`.
3. **Live Preview** uses one kit `StateField` that supplies
   `Decoration.replace({ block: true })` over that range. A `ViewPlugin` cannot replace
   line breaks, and dispatching an effect inside an update is illegal, so the ownership
   effect is deferred. When any selection range touches the hidden range (inclusive, the
   `recordWalkSelectionTouches` rule), the raw Markdown shows for editing. Opening a
   note with the cursor in the frontmatter or at its restored position outside the range
   must not reveal it.
4. **Reading** hides the owning view's recognized `h1` and nav and child-link `p`
   elements with a kit class. It never hides a section wrapper that also hosts a card,
   and never anything inside embeds.
5. **No bytes change.** Headings stay in the metadata cache, so `[[note#heading]]`,
   Outline jumps (which reveal by selection), search, and `[n`/`]n` keep working.

Ownership drops on teardown, a mode switch to Source, file change, a failed mount or
repaint, plugin disable, or unload. Dropping it removes the scope class, clears the
StateField through its effect, and removes Reading classes, so native rendering (inline
title, H1, nav line, Properties) returns in the same frame. Mobile, Source mode, embeds,
hover popovers, canvas nodes, and print/export keep native rendering. In the pane,
exactly one `aria-level="1"` node exists while the card owns the title.

**Title ownership is separate from row ownership** (A9). The title row renders only from
local data: path, `metadataCache` frontmatter, headings, aliases, and the parent chain.
It never waits for a bob report. The owned-Properties hide and editability keep their
existing per-species gates.

### Placement

- **Title slot (if `title-slot` says GO).** The card is the first child after the hidden
  inline title and before native Properties: inside `.cm-sizer` in Live Preview and
  inside `.mod-header` in Reading. A MutationObserver re-inserts it when Obsidian
  re-renders the header, and a ResizeObserver calls `view.requestMeasure()` after the
  card's height changes. The hidden body title range then collapses to nothing.
- **Body fallback (NO-GO, or a surface where the slot is unavailable).** The card sits
  at the top of the body, where the hidden range was. In Live Preview it is a block
  widget placed just before the hidden range (side −1), so a reveal shows the raw lines
  under a still-visible card. Notes without a recognized H1 place it at the first body
  line. Properties stay above it, as today. The worst outcome is today's layout, never a
  missing title.

### Accessibility, sizing, and motion

- The card's accessible name leads with the species word (`Album: Restaurants`,
  `Day: Fri, Nov 20, 2026`). Glyphs are `aria-hidden`.
- Text contrast is at least 4.5:1 and the accent stripe at least 3:1, in light and dark.
  Focus rings are visible on every link and button.
- Under `forced-colors: active`, borders and the stripe fall back to `CanvasText`, and
  the species word stays. `prefers-reduced-motion` disables any transition.
- At content width of 375px or less, actions wrap below the title, the tile shrinks, the
  rail compacts to `‹ Thu · ↑ W47 · Sat ›`, children chips wrap, and the species word
  never disappears.

### Interaction

- `==`, `Ctrl+Shift+P`, the dispatcher order, every card editor's fields, and every
  keymap are unchanged.
- Kicker links and rail links behave exactly like the Markdown links they mirror,
  including modifiers for tab and split. Unresolved targets look unresolved. When
  `bob-navigation-hotkeys` is present and the pane is active, rail prev and next execute
  its `open-prev-link` / `open-next-link` commands, so jump history matches `[n`/`]n`,
  as the record walk rail already does.
- **Card editors mirror the identity** (A8). The Record Card, the album card editor
  (Schema Card), and the day/period editor each show the same tile and species word in
  their modal header, so the `=` climb from a record to its album visibly changes
  species.

### Naming

In docs and UI text, the cards are the **album card**, **record card**, **day card**,
**period card** (week, quarter, year), and **bank card**. "Card editor" names what `==`
opens. "Schema card" stays only as the album card editor's former name. Code identifiers
and CSS classes (`intentionCard*`, `.bob-intention-card`) stay unchanged to avoid churn.
`data-card-kind` carries the new vocabulary. The plugin docs page keeps its published
URL (`docs/intention-card.md`) and is retitled "Day and period cards" (formerly the
intention card).

### Bytes, per species

| Species        | This epic                                                                                                                                                                                               |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Record         | Already H1-less. No change.                                                                                                                                                                             |
| Album          | `title:` becomes the first title source. New albums are written with `title:` and no H1. `bob albums tidy` backfills `title:` and removes the redundant H1. Live apply is gated (see `obsidian-proof`). |
| Day and period | No byte, Templater, or scaffold change. The H1 and nav line stay for mobile, Source mode, embeds, agents, and `[n`/`]n`.                                                                                |

## Phases

### `title-slot`: title-slot spike and mount primitive (go/no-go)

bob-plugins, `plugins/bob-ledger-tools`.

- Add a kit fragment (for example `240e-plugin-property-card-title-slot.js`, listed in
  `src/fragments.json` after `240d`) exporting a small API:
  `propertyCardTitleSlotFor(scope)` resolves the Live Preview
  `.cm-editor > .cm-scroller > .cm-sizer` or the Reading
  `.markdown-preview-view > .markdown-preview-sizer > .mod-header`, and denies embeds,
  popovers, canvas nodes, and mobile.
  `propertyCardMountInTitleSlot(scope, node, { path, kind, view })` inserts the node
  before `.metadata-container`, or before `.cm-contentContainer` / at the end of
  `.mod-header` when Properties are hidden. It re-inserts on header re-render, calls
  `requestMeasure` on resize, and returns an idempotent generation-bound release handle
  in the style of `propertyCardOwnKeys`. It returns `null` wherever it cannot mount,
  which is the adapter's signal to fall back to the body.
- Add `scripts/obsidian-title-slot-spike.cjs` (CDP, isolated profile and vault in the
  style of `obsidian-record-card-e2e.cjs`) plus an offline wrapper test in `npm test`.
  It mounts a probe card through the primitive and measures:
  - survival across file switch, Live Preview ↔ Reading ↔ Source, split panes, the same
    file in two panes, metadata edits that re-render Properties, and window resize;
  - the Properties visibility setting (visible, hidden, source), inline title on and
    off, readable line length on and off, and light and dark themes;
  - with a tall card and with the card growing after mount: click-to-cursor accuracy on
    body lines, `coordsAtPos`, scroll restoration on reopen, and Vim `gg`/`G`;
  - teardown through plugin disable and unload, with no orphan nodes.
- Write the verdict as a bead note `TITLE SLOT: GO` or `TITLE SLOT: NO-GO` with the
  report and screenshot artifacts (`sase artifact create --bead`). A partial GO, such as
  Reading only, must name the surfaces that fall back. The adopt phases read this note.
- Unit tests cover the primitive on the fake DOM (insertion order for each Properties
  setting, re-insertion, release generations, denial cases). Run `npm run build`,
  `npm test`, and `npm run docs:check`, bump the plugin version, commit, and run
  `bob plugins sync`. Nothing user-visible changes yet.

### `identity-kit`: species, tiles, rail, tokens, and title ownership

bob-plugins, `plugins/bob-ledger-tools`. Runs in parallel with `title-slot`. Build the
kit side of the design contract. No adapter adopts it in this phase.

- Add a species registry and DOM builders (new kit fragments after `240d`):
  `propertyCardSpecies(kind)` (word, accessible prefix, tile form, fallback icon),
  `propertyCardBuildGlyphTile`, `propertyCardBuildTitleBlock` (tile, kicker parts with
  optional links, title, state pill, subtitle, actions slot), and
  `propertyCardBuildRail` (prev, parent, and next items plus an optional children strip,
  link handlers injected). Text always enters as text nodes. Obsidian icons go through
  `setIcon`, guarded for tests.
- Add a CSS section for `data-card-kind`, tile forms (stack, sheet, leaf), kicker,
  title, pill, subtitle, rail and children strip, materials, `--bob-day-accent` and its
  scale tints, narrow-width rules, `forced-colors`, and reduced motion. The current
  three cards' pixels must not change until they opt in, because new rules are scoped to
  the new classes.
- Add title ownership as a kit fragment:
  - the `bob-property-card-owns-title` scope class and inline-title CSS, with the
    record-specific selectors left as aliases until `collection-cards` migrates them;
  - the Live Preview `StateField` and deferred ownership effect, with a pure range
    function `propertyCardTitleSourceRange(text, recognizer)` and reveal-on-touch;
  - Reading helpers that tag and untag recognized `h1` and `p` elements per scope;
  - release paths for every loss-of-ownership trigger in the contract. Register the
    extension once through the host.
- Fix the dead `purple` walk-rail rule.
- Extend `scripts/preview-property-cards.cjs`, or add a sibling preview, so it renders
  each species' title block from fixtures in light and dark at 375px and wide, plus
  grayscale and forced-colors captures. Unit tests cover the registry, builders,
  recognizer ranges (album, every day shape, period, legacy no-H1, fenced and late H1s),
  reveal boundaries (cursor on the first line, last line, line after, multiple
  selections), and release generations.
- Run `npm run build`, `npm test`, and `npm run docs:check`, add a short "title block
  kit" section to `docs/develop/architecture.md`, bump, commit, and `bob plugins sync`.

### `album-title-cli`: album `title:` key and `bob albums tidy`

bob-cli. Runs in parallel from the start. Read `cli_rules.md` and `docs.md` first.

- **Readers.** For album-typed notes, frontmatter `title:` (a trimmed nonempty string)
  wins over H1, alias, and stem in every album title reader: `src/native/albums/mod.rs`
  (the report `title`), owner titles in `src/native/albums/registry.rs`, album titles
  seen from records in `src/native/records/context.rs`, `[!albums]` catalog rows, walk
  lines (`↑ <Album title>`), and capture or notebook paths that title albums. Leave
  non-album `derive_title` behavior unchanged.
- **Create.** `render_new_album` writes `title:` in canonical key order (`parent`,
  `type`, `title`, `summary`, optional `emoji`, `created`, `record`) and no H1, like
  H1-less records. The intro line stays.
- **Tidy.** Add `bob albums tidy`, mirroring `bob records tidy`'s CLI shape, dry-run
  default, apply flag, JSON report, guarded transaction, and one commit. For each album
  without `title:`, it writes `title:` from the cleaned first H1 (or the derived title).
  It then removes the H1 when it equals `title:` and is the first body block. It keeps
  and reports an H1 that differs, follows prose, or sits in a fence. It is idempotent.
- **Registrations.** Register the command in all three places that broke the `ms` and
  `nh` landings: `src/native/catalog/table.rs` (entry or exclusion),
  `src/native/completion/kinds.rs` (a decision for every value arg), and the runner and
  canonical-path list (`src/runner.rs`, `src/native/completion/tree.rs`). Follow
  `bob --help` section ordering.
- **Tests.** Fixture vaults cover the reader precedence, create output, tidy dry-run,
  apply, keep rules, and idempotency. Most important: running `bob albums sync` before
  and after tidy produces byte-identical record walk lines, `[!records]` indexes, and
  owner catalogs, so tidy causes no churn.
- **Docs.** Update `docs/records.md` (the album file contract and title source),
  `docs/reference/json-outputs.md` only if a JSON field changes (respect its word
  budget), and the generated CLI docs (`just docs-gen`). Gates: `just fmt`,
  `just check`, and `just docs-check`. Then `just install` on athena. Never edit the
  Cargo version or `CHANGELOG.md` (`decisions:bob-cli-releases-ride-a-daily-train`). Do
  **not** apply tidy to the live vault in this phase.

### `collection-cards`: album, record, and bank cards adopt the title block

bob-plugins. Read the `title-slot` verdict note and the `identity-kit` exports first.
Before starting, `sase bead read bob-cli-ms` to confirm the record epic is closed. If
its remaining landing tale is still running, do not edit the record-writer harness it
repairs.

- **Album card** (`241u`, `241v`, and the schema header fragments):
  - Build the title block. The title comes from local data: `notebookHeaderNoteTitle`
    gains the album rule (frontmatter `title:` first), so crumbs, record kickers, and
    the card agree before and after album tidy.
  - Emoji goes in the stack tile, `summary` in the subtitle, the owner crumb in the
    kicker, and actions in the kicker row. The schema folds to one line by default.
  - Mount in the title slot or the body fallback, and own the title with the redundant
    H1 hidden.
  - The title block paints immediately, and the report-backed body joins when the report
    arrives.
- **Record card** (`242i`–`242k`, `242v`): the kicker (`RECORD · <Album> · <noun>`), the
  sheet tile, and paper material. The title-slot or fallback mount keeps the walk rail
  docked at the card's bottom. In slot mode, the walk line is hidden by the kit range
  with reveal. Migrate the record-specific inline-title and owned-row selectors onto the
  kit classes while keeping every current record e2e assertion green.
- **Bank Setup card** (`248b`, `248c`): `data-card-kind="bank"`, tile, and `BANK` kicker
  only. No title move.
- **Editors:** the Record Card header (`242o`) and the album card editor header (`241y`
  and its shell) show the tile and species word. Field semantics, regions, and keys are
  unchanged.
- Update unit and e2e fixtures that assert DOM shape, extend the preview matrix to the
  real builders, and update `docs/albums.md` and `docs/record-card.md` (behavior only,
  within word budgets). Run `npm run build`, `npm test`, and `npm run docs:check`, bump,
  commit, and `bob plugins sync`. Run the existing real-Obsidian record and album e2e
  scripts on athena and attach their reports.

### `day-period-cards`: day and period cards always mount, with a docked rail

bob-plugins. Read the `title-slot` verdict note and the `identity-kit` exports first.

- **Always mount** (`296a`–`296c`):
  - `intentionCardModelFor` keeps returning a model for every canonical subject. Empty
    fields produce an identity-only model, but malformed frontmatter still fails open to
    native rendering, exactly as today.
  - `intentionCardPlacement` and its Live Preview and Reading anchors accept notes with
    no recognized H1 by mounting at the first body line.
  - Owned keys stay the existing seven.
- **Title block:**
  - Path-computed titles reuse `intentionCardDayLabel`, `intentionCardDateRange`, and
    the subject `label` parts.
  - Kicker links go to the parent subject. Add leaf tiles, the state pill (driven by the
    existing `state`, `today`, and midnight timer), and the temporal accent with scale
    tints.
  - The existing kicker text (`DAY · <label>`) becomes the new kicker. The
    parent-intentions line stays in the body.
- **Rail:** built from the recognized nav line's targets and labels (daily
  `prev · W47 · next` and period `‹ · ↑ · ›` grammars both). Periods add a children
  strip from the child-links line, with today's child highlighted. The recognized H1,
  nav, and child-link lines hide through the kit range with reveal. Rail navigation
  follows the interaction contract.
- **Ghost:** `＋ Set intentions` on current and future subjects with no intentions,
  opening the editor at the intentions stage. This absorbs `bob-cli-o3`; record that in
  a bead note so the land agent can close o3 as superseded.
- **Editor header** (`297c*`): tile and species word.
- Verify by hand-building fixtures for every daily shape in the cld census
  (`# Day: 2023-07-19 Wed`, `# day = 2024-01-01.Mon`, no-H1 2024/2025, legacy 2026,
  current 2026), empty and populated days, today, yesterday, and future, week, quarter,
  and year (including the legacy H1-less `2025.md`), and `all_time` (no card).
- Update the unit, acceptance, and e2e scripts (`obsidian-intention-card-e2e*.cjs`) for
  the new DOM, and retitle and update `docs/intention-card.md` as "Day and period cards"
  (URL kept, "formerly the intention card"). Run `npm run build`, `npm test`, and
  `npm run docs:check`, bump, commit, and `bob plugins sync`. Run the intention-card e2e
  on athena and attach the report.

### `obsidian-proof`: real-Obsidian proof, preview matrix, live album tidy, final deploy

- Add `scripts/obsidian-title-block-e2e.cjs` (an isolated CDP profile and vault, with an
  offline wrapper in `npm test`) and run it on athena with xvfb. It must prove:
  - one visible title per species, and exactly one `aria-level="1"` per pane;
  - inline title, H1, and nav hidden only in the owning view;
  - reveal on Vim `gg`/`j`/`k` across the boundary, on `/` search into the H1, and on
    Outline jumps, plus undo after editing a revealed H1;
  - `[[note#heading]]` links resolve and scroll;
  - Source mode, embeds, hover popovers, a canvas node, a forced failed mount, and
    plugin disable each restore native rendering in the same frame;
  - note bytes are unchanged after open, fold, reveal, and close (hash every fixture);
  - empty and legacy dailies show a title;
  - grayscale and `forced-colors` screenshots tell all species apart with identical
    titles.
- Render the full preview matrix (light and dark × 375px and wide × album, record,
  today, past day, empty day, legacy daily, week, quarter, year, bank, and broken
  report) and inspect every image. Fix every defect found, in the owning fragment.
- **Live album tidy, gated.** Old bob derives album titles from the H1, so an old bob
  running `bob albums sync` after the H1 is gone would rewrite every record walk line to
  the humanized stem. Before applying, verify that every host that can run album sync
  (athena, apollo, and the Mac, reached per `tailnet.md`) has a bob that prefers album
  `title:`.
  - When all of them do, run `bob albums tidy` (dry-run, review, then apply) and
    `bob albums sync`, then confirm no walk, index, or catalog churn and that every
    album card still shows its title.
  - Otherwise, attach the dry-run report and leave a `PROPOSED FOLLOW-UP:` naming the
    blocking host and version.
- Run every gate: bob-plugins `npm run build:check`, `npm test`, and
  `npm run docs:check`; bob-cli `just check` and `just docs-check`. Deploy the final
  clean tree with `bob plugins sync`, and attach the e2e report, preview index, and gate
  logs as artifacts.

### `glossary`: accepted memory updates

Use `/sase_memory_write`, and apply only the memory decisions answered yes.

> [!decision] memory_glossary_cards Update three strands in the `glossary` web, then run
> `sase memory init`:
>
> - `glossary:intention-card`: change the keyword to "Day and Period Card" and keep the
>   slug, adding aliases `intention card`, `day card`, and `period card` next to
>   `intentions card`. Replace the nonempty-field mount rule with the shipped behavior:
>   the card always mounts as the title block, the bands stay content-gated, current and
>   future subjects show the ghost, the rail is docked, and the card fails open.
> - `glossary:property-card`: name the album, record, day, period, and bank cards, and
>   say that the kit owns the title block. That covers the species tile and kicker, a
>   single title, render-time hiding with reveal, and title ownership kept separate from
>   row ownership.
> - `glossary:record-card`: if the `title-slot` verdict was GO, say that the card paints
>   in the note's title slot above Properties. On NO-GO, leave the strand's placement
>   sentence unchanged.

> [!decision] memory_decision_title_block Add the strand
> `decisions:property-cards-own-the-title-block` in the decisions web format. Its claim:
> cards own the title block at render time, and title bytes stay per species (records
> H1-less, albums keep `title:` without an H1, days and periods keep their H1 and nav
> bytes). It cites this plan and the research report. Then run `sase memory init`.

> [!decision] memory_decision_title_block = no Write nothing in the decisions web.

If both memory decisions are no, add a bead note saying so and change nothing.

## Coordination rules for every phase

- bob-plugins phases follow `AGENTS.md` there: edit `src/` fragments only, rebuild
  `main.js`, keep each hand-edited fragment at or below 1000 lines, and deploy with
  `bob plugins sync` (never `-F` on your own; follow the refusal procedure).
- `title-slot` and `identity-kit`, then `collection-cards` and `day-period-cards`, may
  run in parallel. When rebasing, keep both sides' `fragments.json` and `test:unit`
  entries, regenerate `main.js` instead of merging it, take the next free manifest minor
  version, and fetch, merge, and build before every sync.
- Kit API changes after `identity-kit` lands are additive. A phase that needs a kit
  change makes it backward compatible and says so in its commit.
- Phase workers never create beads. They record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead.
- No memory file changes outside the `glossary` phase, and only for memory decisions
  answered yes.

## Acceptance (land agent)

- Open `eat.md`, `eat/<any record>.md`, today's daily, a past empty daily, a 2024 legacy
  daily, `2026/2026-W47.md`, `2026/2026-Q4.md`, and `2026.md` in Live Preview and
  Reading. Each shows exactly one title, in its card. The card's tile and kicker name
  the species. Album and record are told apart by kicker and tile with color removed.
  Days and periods use the temporal accent, never an area hue.
- Placement matches the `title-slot` verdict, and the fallback is today's body layout
  with the title still present.
- Every fail-open case in the contract restores native rendering, and no note bytes
  change from rendering, revealing, or folding.
- Album `title:` reads, create, and tidy are shipped. The live tidy was applied, or its
  version gate is recorded as a follow-up task with the blocking host.
- `bob-cli-o3` is closed as superseded by `day-period-cards`, after the ghost is
  verified.
- All gates are green, or red only on known, already-tracked flakes, each cited. The
  vault deploy is byte-identical to `origin/master` for every touched plugin.

## Non-goals

- No daily or period H1, nav-line, Templater, or scaffold byte changes. A later,
  separately accepted epic may stop emitting them once this hide has soaked.
- No notebook, area, project, bank-month, or `all_time` cards. No mobile title row.
- No title rename operation for albums or records (`bob-cli-n1`), and no album emoji
  writer (`bob-cli-ml`).
- No change to `==` dispatch, card-editor fields, keymaps, `bob periods update`, or any
  JSON contract beyond what `album-title-cli` adds.
- No memory edits except accepted memory decisions, applied in the `glossary` phase.
