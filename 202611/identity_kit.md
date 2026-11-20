---
tier: tale
size: medium
title:
  "Identity kit: species registry, title block and rail builders, card tokens, and title
  ownership in the bob-ledger-tools kit"
goal:
  The bob-ledger-tools property-card kit exports a species registry, glyph tile, title
  block, and docked rail builders. It also ships data-card-kind tokens with the temporal
  day accent, plus title ownership that hides and reveals the title in Live Preview and
  Reading and always fails open. All of this is unit-tested and preview-rendered, while
  every existing card stays pixel-identical.
proposed_by: bbugyi200.athena.bob-cli-o6.2
proposed_with: claude/opus
bead: bob-cli-o6.2
status: done
---

- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)
- **BEAD:**
  [bob-cli-o6.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-o6.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-o6.2.md)
- **COMMITS:**
  - [6278b78](https://github.com/bobs-org/bob-plugins/commit/6278b78c8f0987071d189245dd40eca426c55e12)
    — feat(ledger-tools): add property-card identity kit and title ownership

# Identity kit (phase `bob-cli-o6.2`)

This tale implements phase `identity-kit` of epic `bob-cli-o6`
(`plan:202611/property_card_title_block.md`). Read that plan's **Design contract** first
(`sase bead read bob-cli-o6.2 -r "<why>"` prints its path). It is the spec for every
visual and behavioral rule below. This phase builds only the kit. **No card adopts it
yet**: the album, record, bank, and intention cards must render pixel-identically before
and after this change, with one intended exception, the dead `purple` walk-rail rule
fixed in step 5.

The epic's DECISIONS are final. This phase edits **no memory files**. Do not create
beads. Record discovered follow-up work as
`sase bead note bob-cli-o6.2 'PROPOSED FOLLOW-UP: <summary — detail>'`.

## Where the work happens

All code lives in the **bob-plugins** linked repo, plugin `plugins/bob-ledger-tools`.
Open it with `sase repo open bob-plugins -r "<why>"` and use only the printed path. Read
its `AGENTS.md` before starting. Its rules apply here:

- Edit `src/` fragments only, then regenerate `main.js` with `npm run build`. Never
  hand-edit `main.js`.
- Keep each hand-edited fragment at or below 1000 lines.
- Fragments share one script scope, so function declarations are hoisted across
  fragments. Calls into later fragments (for example the `intentionCard*` helpers in
  `296a`) work at runtime. Guard them with `typeof fn === "function"` so a missing
  helper fails open.
- Deploy with `bob plugins sync`. Never pass `-F` on your own. Follow the refusal
  procedure in `AGENTS.md`.

Sibling phase `bob-cli-o6.1` (`title-slot`) runs in parallel. It adds
`src/240e-plugin-property-card-title-slot.js` after `240d` in `src/fragments.json`, a
test entry in `package.json`, and a manifest minor bump. This tale's new fragments are
named `240f`–`240i` so their file names cannot collide with it. Before you start, and
again before you deploy, run `git fetch` and fast-forward or rebase onto
`origin/master`. If `title-slot` (or any other phase) has landed:

- keep both sides' `fragments.json` and `test:unit` entries;
- regenerate `main.js` with `npm run build` instead of merging it;
- take the next free manifest minor version.

CodeMirror effects must stay lazy. `src/010-load-and-constants.js` notes that the
freshness-mark suite asserts a single eager `StateEffect.define()` at module load.
Create every new `StateEffect` and `StateField` lazily, on first use, never at module
scope.

## Steps

### 1. Species registry: `src/240f-plugin-property-card-species.js`

- `PROPERTY_CARD_KIND_ATTR = "data-card-kind"` and
  `PROPERTY_CARD_KINDS = ["album", "record", "day", "week", "quarter", "year", "bank"]`.
- `propertyCardSpecies(kind)` returns a frozen descriptor, or `null` for an unknown
  kind. A descriptor holds:
  - `kind`;
  - `word`: `Album`, `Record`, `Day`, `Week`, `Quarter`, `Year`, or `Bank`. CSS
    uppercases it, so screen readers still read a word;
  - `tile`: `stack` for album, `sheet` for record, `leaf` for day, week, quarter, and
    year, and `plain` for bank;
  - `icon`, the fallback Obsidian icon: `library` for album, `file-text` for record,
    `calendar` for day, `calendar-range` for the three periods, and `landmark` for bank;
  - `band`, the leaf default: `WEEK`, `QTR`, or `YEAR`. A day's band is the weekday,
    which the adapter supplies;
  - `temporal`: `true` for day, week, quarter, and year.
- `propertyCardAccessibleName(kind, title)` returns `"<Word>: <title>"` (for example
  `Album: Restaurants` or `Day: Fri, Nov 20, 2026`). With an empty title it returns just
  the word. With an unknown kind it returns the title alone.
- `propertyCardApplyIdentity(card, { kind, title })` sets `data-card-kind`, sets
  `aria-label` to the accessible name, and sets `role="group"` only when the card has no
  role already. It returns `false` and changes nothing for an unknown kind or a missing
  card.

### 2. Builders: `src/240g-plugin-property-card-title-block-dom.js`

Every builder takes `doc` (an `ownerDocument`) and a spec. It returns `null`, never
throws, when `doc` is missing or the kind is unknown. Text enters only through
`textContent` or text nodes, never `innerHTML`. Icons go through Obsidian's `setIcon`,
captured guardedly at load the way `src/293-keep-inbox-render.js` does. When `setIcon`
is unavailable (tests), leave the glyph empty and set `data-icon`.

- **`propertyCardBuildGlyphTile(doc, { kind, emoji?, icon?, band?, value? })`**
  - Returns `span.bob-property-card-tile` with `data-tile-form` set to the species' tile
    form and `aria-hidden="true"`.
  - `stack`, `sheet`, and `plain` tiles hold one `span.bob-property-card-tile-glyph`. It
    shows the emoji text if one is given, else the icon.
  - A `leaf` tile holds `span.bob-property-card-tile-band` (the band text, defaulting to
    the species band) over `span.bob-property-card-tile-value`.
- **`propertyCardBuildTitleBlock(doc, spec)`**
  - The spec is `{ kind, title, kicker = [], pill?, subtitle?, tile = {}, onLink? }`.
    Each kicker part is `{ text, target?, unresolved? }`. `pill` is
    `{ text, tone: "solid" | "outline" }`.
  - It returns `{ element, tile, kicker, heading, pill, subtitle, actions }`, or `null`
    if the kind is unknown or the title is empty.
  - The structure is:

    ```text
    div.bob-property-card-title-block[data-card-kind]
      span.bob-property-card-tile                       (from the tile builder)
      div.bob-property-card-title-main
        div.bob-property-card-kicker-row
          span.bob-property-card-kicker
            span.bob-property-card-kicker-species      ("Album")
            ( span.bob-property-card-kicker-sep[aria-hidden] "·"  +  part )*
          span.bob-property-card-actions               (always present, empty)
        div.bob-property-card-title-row
          div.bob-property-card-title[role=heading][aria-level=1]
          span.bob-property-card-pill[data-pill-tone]   (only when pill.text)
        div.bob-property-card-subtitle                 (only when nonempty)
    ```

  - A part with a `target` renders as
    `a.bob-property-card-kicker-link[data-href][href]`, with `is-unresolved` when
    `unresolved` is set. Clicking it calls `preventDefault()` and then
    `onLink(target, event)`, so adapters can honor modifier keys the way
    `recordWalkOpenLink` in `242v` does. Do **not** add Obsidian's `internal-link`
    class, which would attach Obsidian's own global click handling.
  - A part without a target renders as `span.bob-property-card-kicker-part`.

- **`propertyCardBuildRail(doc, { prev?, parent?, next?, children?, label?, onNavigate? })`**
  - Each item is `{ label, shortLabel?, target, unresolved?, ariaLabel? }`. Each child
    is `{ label, target, current?, unresolved? }`.
  - It returns `null` when there is no item and no child. Otherwise it returns
    `{ element, items: { prev, parent, next }, children: [] }`.
  - The structure is:
    - `nav.bob-property-card-rail[aria-label]` (default `Note navigation`);
    - `div.bob-property-card-rail-row`, a three-column grid. It holds
      `a.bob-property-card-rail-item[data-rail=prev|parent|next]` with `‹ ` / `↑ ` /
      ` ›` glyphs in `aria-hidden` spans, plus `span.bob-property-card-rail-label-wide`
      and `span.bob-property-card-rail-label-narrow` (the narrow label falls back to the
      wide one). A missing item becomes an empty `span.bob-property-card-rail-gap`, so
      columns stay put.
    - `div.bob-property-card-rail-children[role=list]`, only when there are children. It
      holds `a.bob-property-card-rail-chip[role=listitem]`, and the current chip gets
      `is-current` and `aria-current="true"`.
  - A click calls `preventDefault()` and then
    `onNavigate(direction | "child", item, event)`.

### 3. Title source range and recognizers: `src/240h-plugin-property-card-title-range.js`

Keep this fragment pure, with no DOM and no CodeMirror.

- **`propertyCardCleanHeadingText(raw)`**
  - Strips the ATX marker and any closing `#`s, unescapes backslash escapes, turns
    `[[t|l]]` into `l` and `[[t]]` into `t`, collapses whitespace, and trims.
- **`propertyCardTitleSourceRangeFromLines(lineAt, lineCount, recognizer)`**, the core.
  - `lineAt(i)` returns `{ text, from, to }` for 0-based line `i`, with document
    offsets. Scanning stops early, so Live Preview never stringifies a large note.
  - Skip YAML frontmatter (`---` closed by `---` or `...`). Unterminated frontmatter
    returns `null`. A leading `﻿` on line 0 is ignored when testing for the opening
    `---`, and offsets still count it.
  - The **first non-blank body line** must start the block, so a fenced, late, or
    after-prose H1 is never recognized. If that line is an ATX H1 (`/^ {0,3}#[ \t]+\S/`)
    and `recognizer.heading(cleaned, rawLine)` returns `true`, the block starts there.
  - When `recognizer.requireHeading === false` (heading-less species such as records), a
    first line that `recognizer.follower` accepts also starts the block.
  - Anything else returns `null`.
  - After the start, absorb blank lines and every line that
    `recognizer.follower(trimmedLine, index)` accepts. Stop at the first other non-blank
    line.
  - Returns:
    - `{ from, to, fromLine, toLine, headingLine, title, nextLine }`.
    - `to` is the end of the **last recognized non-blank line**, so trailing blank lines
      are never hidden.
    - `headingLine` is `-1` when there is no heading.
    - `nextLine` is the first line after the block.
- **`propertyCardTitleSourceRange(text, recognizer)`** wraps the core for strings, with
  `\n`, `\r\n`, or `\r` separators and correct offsets.
- **`propertyCardSelectionTouches(selection, range)`**
  - Uses the inclusive `recordWalkSelectionTouches` rule: some range satisfies
    `min <= range.to && max >= range.from`.
  - It works on CodeMirror selections and on plain `{ ranges: [{ from, to }] }`.
- **Ready-made recognizers.** Each is a plain object with a stable `key` string. Title
  ownership deduplicates by `key`.
  - **`propertyCardAlbumTitleRecognizer(title)`**
    - `key` is `album:<title>`.
    - The heading matches when its cleaned text equals the whitespace-collapsed, trimmed
      title.
    - It has no followers.
  - **`propertyCardDayTitleRecognizer(subject, { alias, today })`**
    - `key` is `day:<date>`.
    - The heading matches the cleaned first alias, or one of this date's shapes,
      compared case-insensitively: `YYYY-MM-DD ddd`, `Day: YYYY-MM-DD ddd`, and
      `day = YYYY-MM-DD.ddd`. The weekday is `Mon`…`Sun`, computed from the date.
    - The follower is `intentionCardIsNavigationLine(line, subject, today)`.
  - **`propertyCardPeriodTitleRecognizer(subject, { alias, today })`**
    - `key` is `<kind>:<id>`.
    - The heading matches the cleaned first alias. bob's scaffold H1 is the period
      title, which is also the alias, for example `2026-W47 · Nov 16 – 22`.
    - The follower is the nav line or `intentionCardIsChildLinksLine`.
  - Recognizers whose subject is missing or whose `intentionCard*` helper is missing
    return a recognizer that never matches.

### 4. Title ownership: `src/240i-plugin-property-card-title-ownership.js`

Constants:

- `PROPERTY_CARD_OWNS_TITLE_CLASS = "bob-property-card-owns-title"`
- `PROPERTY_CARD_TITLE_SOURCE_CLASS = "bob-property-card-title-source"`
- `PROPERTY_CARD_TITLE_SOURCE_ATTR = "data-bob-title-source"`, whose value is the owner
  path

**Scope denial.** `propertyCardTitleScopeDenied(scope)` returns
`propertyCardDeniedScope` (`240b`), and also denies any scope inside `.canvas-node`,
`.canvas-node-content`, or `.print`. That keeps embeds, hover popovers, canvas nodes,
print/export, mobile, and non-view scopes native.

**Live Preview.** Write the logic as pure, exported functions so the stubbed tests can
drive it. The CodeMirror objects are thin wrappers around them.

- `propertyCardTitleFieldCreate()` returns
  `{ owner: null, range: null, revealed: false }`.
- `propertyCardTitleFieldReduce(value, tr, deps)` takes `tr`, which needs `effects`,
  `docChanged`, `state` (`doc`, `selection`), and `startState`.
  - Apply ownership effects. `{ own: { path, generation, recognizer } }` sets the owner.
    `{ release: generation }` clears the owner only when the generation matches.
  - **Drop the owner** when the state is not Live Preview
    (`state.field(editorLivePreviewField, false) === false`) or when
    `state.field(editorInfoField, false)?.file?.path` differs from the owner path.
  - Recompute `range` through `propertyCardTitleSourceRangeFromLines` over `doc.line()`
    only when the owner changed or the doc changed.
  - Recompute `revealed` on every transaction with `propertyCardSelectionTouches`.
- `propertyCardTitleFieldDecorations(value)` returns one
  `Decoration.replace({ block: true })` from `range.from` to `range.to` when there is an
  owner and a range and the range is not revealed. Otherwise it returns
  `Decoration.none`.
- `propertyCardTitleFieldAttributes(value)` returns
  `{ class: PROPERTY_CARD_OWNS_TITLE_CLASS }` whenever there is an owner, whether or not
  a range was recognized (the card still owns the inline title). Otherwise it returns
  `null`.
- `propertyCardTitleOwnershipExtension()` lazily defines the effect and the
  `StateField`. It provides `EditorView.decorations.from(field, …)` and
  `EditorView.editorAttributes.from(field, …)`. `editorAttributes` lands on
  `.cm-editor`, so the scope class drops in the same transaction as the owner. It
  returns `null`, and the caller registers nothing, when any CodeMirror piece is
  missing.
- `propertyCardOwnTitleInEditor(view, { path, recognizer, defer? })` returns a
  generation-bound, idempotent release handle, or `null` when denied.
  - The **ownership effect is deferred**, because dispatching inside an update is
    illegal. Dispatch through `queueMicrotask` (fall back to `Promise.resolve().then`),
    so it lands before the next paint. `defer` is injectable for tests.
  - Re-owning with the same `path` and `recognizer.key` returns the existing handle and
    dispatches nothing. Adapters call this on every decoration rebuild, so this prevents
    a dispatch loop.
  - The handle's release is also deferred, and it is a no-op once a newer generation
    owns the view.
  - Track owned views in a module `Set` so `propertyCardTitleReleaseAll()` (used by host
    dispose) can release them all. Guard every dispatch with `try`, because the view may
    be gone.

**Reading.** `propertyCardOwnTitleInReading(scope, { path, recognizer })` returns a
generation-bound release handle, or `null` when the scope is denied or is not a
`.markdown-preview-view`.

- It adds the scope class, then tags the recognized elements with the source class and
  attribute:
  - Walk the sizer's direct section children in order, skipping `.mod-header`,
    `.mod-frontmatter`, `.markdown-preview-pusher`, `.mod-footer`, and empty sections.
  - The first content section's first element must be an `h1` the recognizer accepts
    (or, for a heading-less recognizer, a follower `p`).
  - After it, tag every following `p` whose reconstructed line (or every
    `<br>`-separated line) the follower accepts.
  - Reconstruct a line with `propertyCardReadingLineText(p)`: text nodes stay as text,
    `a[data-href]` becomes `[[data-href|textContent]]`, and `br` becomes `\n`.
- **Tag only the `h1` and `p` elements, never a section wrapper.** The intention card
  mounts inside the H1 section. Never tag anything inside an embed, a node that fails
  `propertyCardNodeBelongs`, or a `.bob-property-card` subtree.
- Like `propertyCardOwnKeys` (`240b`), keep a `MutationObserver` on the scope that
  re-tags when lazily rendered sections arrive.
- Release (handle, `propertyCardTitleReleaseAll`, or a newer owner on the same scope)
  disconnects the observer, untags every element, and removes the scope class. An old
  generation's release never clears a newer owner.

**Unified entry and host opt-in.**

- `propertyCardOwnTitle(scope, { path, recognizer, view })` dispatches to the editor
  path when `scope` is `.cm-editor` and `view` is given. It dispatches to the Reading
  path when `scope` is a `.markdown-preview-view`. Otherwise it returns `null`.
- Additive host change in `240c`/`240d`: a new option `titleOwnership: true` makes
  `setup()` register `propertyCardTitleOwnershipExtension()` **once per plugin
  instance**. A flag on the plugin makes a second opted-in host a no-op. `dispose()`
  calls `propertyCardTitleReleaseAll()`.
- Hosts that omit the option (album, record, bank, and intention today) register nothing
  new and behave exactly as before.
- Document the option in the options comment above `createPropertyCardHost`.

### 5. CSS: a new section at the end of `styles.css`

Add a section headed
`/* Property-card identity kit (bob-cli-o6 identity-kit): start. */` and close it with a
matching end comment.

- **Scoping.** Every new rule must be scoped to a new class or to
  `.bob-property-card[data-card-kind]`. No adapter sets that attribute yet, so the
  current cards keep their exact pixels.
- **Title ownership.**
  - `.cm-editor.bob-property-card-owns-title > .cm-scroller > .cm-sizer > .inline-title`
    and the Reading
    `.markdown-preview-view.bob-property-card-owns-title > .markdown-preview-sizer > .mod-header > .inline-title`
    get `display: none !important`.
  - `.markdown-preview-view.bob-property-card-owns-title .bob-property-card-title-source`
    gets `display: none !important`.
  - Leave the record-specific `bob-record-sheet-hides-properties` selectors untouched,
    as aliases until `collection-cards` migrates them. Add a comment saying so.
- **Card kinds.**
  - `.bob-property-card[data-card-kind]` gets
    `container: bob-property-card / inline-size`. Say in the comment that cards which
    opt in must be block-width; the day card drops `fit-content` when it adopts the kit.
  - Album: 8% tint, and the title block gets a heavier
    `border-block-end: 1px solid color-mix(in srgb, var(--bob-property-card-accent) 35%, var(--background-modifier-border))`.
  - Record: 4% tint and the hairline border.
  - Day, week, quarter, and year: set
    `--bob-day-accent: color-mix(in srgb, var(--color-cyan, #00bfbc) 50%, var(--text-normal))`
    and map `--bob-property-card-accent` to it, ignoring `data-root-hue`. Tints are 6%,
    8%, 10%, and 12%.
  - Bank: unchanged chrome.
- **Title block.** Use a grid of `auto minmax(0, 1fr)` with about `0.75em` column gap.
  - Kicker: `650 0.74em/1.35 var(--font-interface)`, `letter-spacing: 0.08em`,
    uppercase. The species word takes `--bob-property-card-accent`. The context and
    separators are `--text-muted`.
  - Actions: pushed right with `margin-inline-start: auto`.
  - Title: reuse the record sheet's `--inline-title-*` / `--h1-*` variable chain from
    `.bob-record-sheet-title`, with `overflow-wrap: anywhere` and no truncation.
  - Pill: `solid` and `outline` tones in the card accent.
  - Subtitle: `--text-muted` at `--font-ui-small`.
- **Tiles.** A 2.4em rounded square (6px radius) that becomes 1.8em in the narrow
  container.
  - `stack`: two offset sheet edges, drawn with `::before`/`::after` borders.
  - `sheet`: a folded top-right corner.
  - `leaf`: an accent band in `--bob-property-card-accent`, with band text in
    `--background-primary`, over a large value.
  - `plain`: a bordered tile.
  - No shadows, gradients, banners, or nested boxes anywhere.
- **Rail.** A hairline `border-block-start`, then a three-column row of prev / centred
  parent / next.
  - The children strip wraps chips. `is-current` uses the accent outline.
  - Links show unresolved styling for `is-unresolved`.
  - Every link, chip, and button gets
    `:focus-visible { outline: 2px solid var(--interactive-accent); outline-offset: 2px }`.
- **Narrow container**, `@container bob-property-card (max-width: 375px)`:
  - actions wrap below the title and the tile shrinks;
  - rail labels switch from wide to narrow and chips wrap;
  - the species word stays visible.
- **Forced colors.** Under `@media (forced-colors: active)`, the card border, stripe,
  tile border, pill border, and leaf band use `CanvasText` (with `forced-color-adjust`
  where needed), and the species word stays visible.
- **Reduced motion.** Under `@media (prefers-reduced-motion: reduce)`, all kit
  transitions and animations are off.
- **Fix the dead rule.** `.bob-record-walk-rail[data-root-hue="purple"]` becomes
  `[data-root-hue="violet"]` with `var(--bob-area-violet, var(--text-muted))`.

### 6. Exports

Append every new public function and constant above to
`src/374-exports-property-card.js`. That includes the pure field functions and the
recognizers. Add the four new fragments to `src/fragments.json` immediately after
`240d-plugin-property-card-host-editor.js` (after `240e-…` if title-slot has landed).
Then run `npm run build`.

### 7. Unit tests, both added to `package.json` `test:unit` next to `test-ledger-tools-property-card-host.cjs`

Stub `obsidian` and CodeMirror the way
`scripts/test-ledger-tools-property-card-host.cjs` does. Reuse a FakeNode-style DOM from
an existing ledger-tools test, adding `addEventListener`, `textContent`, and
`nextElementSibling` as needed.

**`scripts/test-ledger-tools-property-card-identity.cjs`**

- **Registry:** all seven kinds, frozen descriptors, an unknown kind returning `null`,
  accessible names, and `propertyCardApplyIdentity` (kind attribute, label, role
  preserved when one is already set).
- **Tiles:** every form, emoji versus icon fallback with `data-icon`, `aria-hidden`, and
  leaf band and value.
- **Title block:**
  - kicker species word first, separators `aria-hidden`, and link parts;
  - a click calls `preventDefault`, then `onLink(target, event)`;
  - `is-unresolved`, and no `internal-link` class;
  - exactly one `role=heading` with `aria-level=1`;
  - pill tones, subtitle omitted when empty, actions slot present;
  - markup-looking titles stay literal text;
  - `null` for an unknown kind, an empty title, or a missing `doc`.
- **Rail:** `null` when empty, gap placeholders, wide and narrow labels, a children
  strip with a current chip, and `onNavigate` directions.
- **Preview cases:** `buildCases` from step 8 builds every fixture offline.

**`scripts/test-ledger-tools-property-card-title-ownership.cjs`**

- **Range vectors.**
  - Album: frontmatter, then `# Restaurants`, then an intro gives the H1 line only. A
    title mismatch, prose before, an H1 inside a fence, a late H1, unterminated
    frontmatter, and an empty body give `null`. A BOM and `\r\n` give correct offsets.
  - Day: every shape (`# 2026-11-20 Fri`, `# Day: 2023-07-19 Wed`,
    `# day = 2024-01-01.Mon`, and an alias match). Both nav forms (`prev`/`next` and
    `prev · W47 · next`) are absorbed across blank lines. Trailing blanks are excluded.
    H1-less 2024/2025 shapes give `null`. A body line like `day::2025-01-01 Wed` is
    never hidden.
  - Period: week, quarter, and year with the scaffold alias, nav, and child-links lines.
    A legacy H1-less year gives `null`.
  - A heading-less follower range for `requireHeading: false`.
- **Reveal boundaries:**
  - a cursor at `from` (first line) reveals;
  - a cursor at `to` (end of the last line) reveals;
  - `to + 1` (the line after) does not;
  - the end of the frontmatter / `from - 1` does not;
  - multiple selections where one touches reveal;
  - a selection spanning the range reveals.
- **Reducer:**
  - an own effect sets the owner and range;
  - a stale-generation release is ignored, and the current one clears;
  - a Source-mode state drops the owner;
  - a file-path change drops the owner;
  - a doc change recomputes the range;
  - a selection-only change keeps the range and toggles `revealed`;
  - decorations and attributes follow the owner, live mode, and reveal.
- **Deferred ownership:**
  - nothing dispatches synchronously;
  - the same key dispatches once;
  - a new key gets a new generation;
  - an old handle cannot release a newer owner;
  - `propertyCardTitleReleaseAll` releases every view;
  - a denied embed scope returns `null`.
- **Reading:**
  - tags the `h1` and nav `p` (and `<br>`-joined nav plus child lines);
  - leaves the section wrapper and a card inside it untouched;
  - ignores embed content;
  - re-tags a late section through a fake `MutationObserver`;
  - release untags and removes the class;
  - generation safety holds.
- **Host opt-in:** two hosts with `titleOwnership: true` register the extension once. A
  host without it registers nothing new.

### 8. Preview matrix: `scripts/preview-title-block.cjs`

This is a sibling of `scripts/preview-property-cards.cjs`. Additively export that
script's CDP helpers (`CdpClient`, `freePort`, `cdpTarget`, `sleep`,
`previewDocument`/`PreviewElement`) if it does not already, and reuse them. Keep both
scripts working.

- `buildCases(H, doc)` builds, with the real kit builders, a title block plus rail
  inside `div.bob-property-card[data-card-kind][data-root-hue]` for these fixtures:
  - album: 🍽, `Restaurants`, kicker `Body`, a summary subtitle, and actions
    `＋ Restaurant` / `⊞`;
  - album with no emoji (library icon);
  - record: kicker `Restaurants · Restaurant`;
  - today's day (FRI/20, `Fri, Nov 20, 2026`, `TODAY` pill, rail
    `Thu 19 · W47 · Sat 21`);
  - a past day (no pill);
  - week, with seven day chips, one current, and `THIS WEEK`;
  - quarter, with week chips;
  - year, with quarter chips;
  - bank;
  - a very long wrapping title.

  Export `buildCases` for the unit test.

- **Captures:**
  - light and dark × 1280px and 375px for every case;
  - a grayscale pass, with `filter: grayscale(1)` on `html`, of album, record, and day
    with **identical titles**, to prove kicker and tile alone tell them apart;
  - a forced-colors pass through
    `Emulation.setEmulatedMedia({ features: [{ name: "forced-colors", value: "active" }] })`.
  - Write each PNG, `report.json`, and an `index.html` grid to `--out`.
- **In-browser assertions**, recorded in the report and failing the run:
  - exactly one `[aria-level="1"]` per card;
  - kicker species-word contrast against the computed card background is at least 4.5:1;
  - stripe contrast against the page background is at least 3:1, in light and dark.

  Resolve colors through `getComputedStyle` on a probe element.

### 9. Docs

In bob-plugins `docs/develop/architecture.md`, add a short `### Title block kit`
subsection (about 150–200 words) at the end of **Property-card extension API**. It
covers:

- the species registry and `data-card-kind`;
- the three builders and their injected link handlers;
- `--bob-day-accent` and the block-width requirement;
- `propertyCardOwnTitle` (Live Preview `StateField` with reveal-on-touch, Reading tags),
  the `titleOwnership` host option, and each release trigger;
- that title ownership is separate from row ownership;
- the two new test entry points and the preview script.

Behavior-only, concise wording. No user-facing doc changes in this phase.

## Verification (all required before closing)

Run these from the bob-plugins checkout:

1. `npm run build`, then `npm run build:check`.
2. `npm test`. All units and perf must pass. If a failure also reproduces on a clean
   `origin/master` checkout, record it as a `PROPOSED FOLLOW-UP:` (cite any tracking
   bead) instead of blocking.
3. `npm run docs:check`.
4. Pixel neutrality for existing cards:
   `xvfb-run -a node scripts/preview-property-cards.cjs -o <tmp>/pc -b <origin/master sha before your change>`
   must report every pair pixel-identical. If a fixture uses a violet hue on a record
   walk rail, only that intended `purple` → `violet` difference may change. Name it in
   the bead note.

   Also run `xvfb-run -a node scripts/preview-intention-card.cjs -o <tmp>/ic` and
   confirm it is unchanged.

5. `xvfb-run -a node scripts/preview-title-block.cjs -o <tmp>/tb` must pass its
   assertions. **Open and inspect every PNG.** Fix any defect in the owning fragment or
   CSS and re-render. Attach the report and index as artifacts on `bob-cli-o6.2`, after
   reading `sase memory read sase_artifacts.md -r "<why>"` for the artifact commands.
6. Bump `plugins/bob-ledger-tools/manifest.json` to the next free minor version, run
   `npm run build`, then `bob plugins sync --dry-run -p bob-ledger-tools` and
   `bob plugins sync -p bob-ledger-tools` from the bob-plugins checkout. Follow
   `AGENTS.md` on refusal.
7. Nothing is user-visible yet. Optionally, open one album note in the real vault and
   confirm it renders exactly as before.

Close-out:

- Run `sase bead epic-symbols bob-cli-o6.2`. There were none at planning time.
- Leave a bead note listing the exported kit API names. The `collection-cards` and
  `day-period-cards` phases read it.
- Then run `sase bead close bob-cli-o6.2 --note "<what you verified>"`.
- Never close the epic `bob-cli-o6`.
- The bob-plugins changes are committed through the turn's `/sase_final` repository
  obligation. The commit message is
  `feat(ledger-tools): add property-card identity kit and title ownership`.

## Out of scope

- Adopting the kit in any card, or mounting in the title slot (`bob-cli-o6.1`).
- Migrating the record-specific selectors.
- Album `title:` in bob-cli.
- Any change to `==` dispatch, card editors, keymaps, or note bytes.
- Any memory edit.
