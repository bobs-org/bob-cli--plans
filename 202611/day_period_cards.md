---
tier: tale
size: medium
title: 'Day and period cards: the intention card always mounts in the title slot as
  the note''s title block, with a leaf tile, state pill, docked rail, and set-intentions
  ghost (bob-cli-o6.5)'
goal: On desktop Obsidian, every canonical daily, week, quarter, and year note opens
  on a day or period card mounted in Obsidian's title slot. The card shows a path-computed
  title, a leaf tile, a species kicker, a state pill, and a docked rail with a children
  strip on periods. The recognized H1, nav line, and child-link line are hidden at
  render time and revealed when the cursor touches them. Current and future subjects
  with no intentions show a `＋ Set intentions` ghost. Content bands keep their gates.
  Every failure restores native rendering, and no note bytes change.
proposed_by: bbugyi200.athena.bob-cli-o6.5
proposed_with: claude/opus
bead: bob-cli-o6.5
status: done
---

- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)
- **BEAD:**
  [bob-cli-o6.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.5.md)

# Day and period cards (phase `bob-cli-o6.5`)

This tale implements phase `day-period-cards` of epic `bob-cli-o6`
(`plan:202611/property_card_title_block.md`). Read that plan's **Design contract** first
(`sase bead read bob-cli-o6.5 -r "<why>"` prints its path). It is the spec for every
visual and behavioral rule below. This tale decides the open adapter questions; where it
is silent, the contract wins.

The epic's DECISIONS are final. This phase edits **no memory files**: the glossary
strands belong to the later `glossary` phase. Do not create beads. Record discovered
follow-up work as
`sase bead note bob-cli-o6.5 'PROPOSED FOLLOW-UP: <summary — detail>'`.

## Inputs you must read first

- `sase bead read bob-cli-o6.1 -r "<why>"`. Note #1 is the verdict, **TITLE SLOT: GO**:
  Live Preview and Reading both mount in the slot, and no surface falls back. It also
  documents the primitive API:
  `propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost })`.
  - The call returns a release handle, or `null`, which means fall back to the body.
  - `onLost(reason)` fires with `denied`, `path-changed`, `disconnected`, `unstable`, or
    `insert-failed`.
  - The Live Preview scope is `view.editor.cm.dom` (the `.cm-editor`), and `view` is the
    CodeMirror `EditorView`.
  - The Reading scope is the `.markdown-preview-view` child of
    `previewMode.containerEl`, and `view` is `previewMode.renderer`. Pass the renderer:
    the primitive calls its `updateHeader()` so scroll restore stays exact.
- `sase bead read bob-cli-o6.2 -r "<why>"`. Note #1 lists the identity-kit API, all on
  `module.exports.helpers` (`src/374-exports-property-card.js`):
  - species: `propertyCardSpecies`, `propertyCardApplyIdentity`;
  - builders: `propertyCardBuildGlyphTile`, `propertyCardBuildTitleBlock`,
    `propertyCardBuildRail`;
  - ranges and recognizers: `propertyCardTitleSourceRange`,
    `propertyCardDayTitleRecognizer`, `propertyCardPeriodTitleRecognizer`;
  - ownership: `propertyCardOwnTitleInEditor`, `propertyCardOwnTitleInReading`,
    `propertyCardTitleReleaseAll`;
  - the host opt-in `titleOwnership: true`.

  Editor ownership returns `null` in Source mode and when the field is missing.
  Re-owning with the same path and key returns the same handle. Reading ownership bumps
  the generation on every call, so keep the latest handle and do not re-own on every
  paint.

- `sase bead read bob-cli-o3 -r "<why>"`. That is the ghost proposal this phase absorbs.
- The research report, if your workspace has it:
  `sase artifact read research:202611/property_card_title_block/property_card_title_block.md "<why>"`.
  If it does not resolve, the epic plan's design contract is enough.
- `sase memory read docs.md -r "<why>"` before touching `docs/`.

## Where the work happens

All code lives in the **bob-plugins** linked repo, plugin `plugins/bob-ledger-tools`.
Open it with `sase repo open bob-plugins -r "<why>"` and use only the printed path. Read
its `AGENTS.md`. Its rules apply here:

- Edit `src/` fragments only, then regenerate `main.js` with `npm run build`. Never
  hand-edit `main.js`.
- Keep each hand-edited fragment at or below 1000 lines.
- Fragments share one script scope, so guard calls into other fragments with
  `typeof fn === "function"`.
- Create any new CodeMirror `StateEffect` lazily, never at module scope. The
  freshness-mark suite asserts a single eager `StateEffect.define()`.
- Deploy with `bob plugins sync -p bob-ledger-tools`. Never pass `-F` on your own;
  follow the refusal procedure in `AGENTS.md`.

Sibling phase `bob-cli-o6.4` (`collection-cards`: album, record, and bank) runs in
parallel. It edits the same `styles.css`, `src/fragments.json`, `package.json`
`test:unit`, manifest version, and possibly kit fragments `240*`. To stay clear of it:

- Keep this phase's code in intention-card fragments (`296*`, `297*`, and the new ones
  below). Touch kit fragments only for additive, backward-compatible fixes, and say so
  in the commit message.
- Put new CSS in one new block right after the existing `.bob-intention-card*` rules,
  headed `/* Day and period cards (bob-cli-o6 day-period-cards) */`. Do not edit the kit
  section.
- Before you start, and again before you deploy, run `git fetch` and rebase onto
  `origin/master`. If anything landed:
  - keep both sides' `fragments.json` and `test:unit` entries;
  - regenerate `main.js` with `npm run build` instead of merging it;
  - take the next free manifest minor version (currently `1.145.0`, so `1.146.0` unless
    taken).

## Behavior decisions this tale makes

- **Placement.** With the GO verdict, the title slot is the primary mount in both modes.
  The body mount from today stays as the fallback. It is used only when the primitive
  returns `null`, or `onLost` reports `unstable` or `insert-failed`.
  - In fallback the card does **not** own the title: no inline-title hide and no H1,
    nav, or child-link hiding. The note shows today's layout plus the card, and its
    title is never missing.
  - Fallback is sticky per scope and path. It clears on a path change or a mode change.
- **Title ownership** (slot mode only) uses the kit recognizers:
  - `propertyCardDayTitleRecognizer(subject, { alias, today })` for days;
  - `propertyCardPeriodTitleRecognizer(subject, { alias, today })` for periods;
  - `alias` is the note's `aliases` (or `alias`) frontmatter from `metadataCache`.
  - Live Preview owns through
    `propertyCardOwnTitleInEditor(view, { path, recognizer })`, which the kit makes
    idempotent.
  - Reading owns through `propertyCardOwnTitleInReading(scope, { path, recognizer })`,
    called once per mount and again only when the path or the recognizer `key` changes.
    Keep the handle.
- **Rail ⇔ hidden nav.** The rail renders only when the recognized source title range
  (`propertyCardTitleSourceRange(source, recognizer)`) contains a recognized nav line.
  The children strip renders only when that range contains a recognized child-links
  line.
  - So a rail never duplicates a visible nav line, and a note whose H1 is not recognized
    (legacy H1-less dailies, `2025.md`, the `prev | day | next` June 2026 shape) shows
    no rail and hides nothing.
  - Targets always come from the nav line's links.
- **Rail labels.**
  - Days: prev and next show `<Wkd> <d>` (`Thu 19`, `Sat 21`), with short labels `Thu`
    and `Sat`. The parent is the week's short label (`W47`). When a two-link daily nav
    (`prev · next`) has no parent link, the rail parent uses the subject's week path.
  - Periods: use the nav link's label with the leading `‹ ` / `↑ ` and trailing ` ›`
    stripped (`W46`, `2026-Q4`, `W48`; `Q3`, `2026`, `2027-Q1`; `2025`, `All time`,
    `2027`). Short labels equal the wide ones.
  - Each item gets an `ariaLabel` that names the full subject (for example
    `Previous day: Thu, Nov 19, 2026` or `Parent week: 2026-W47`).
- **Children strip.** Chip labels and targets come from the child-links line. The chip
  whose subject contains `today` (same day, the week containing today, or the quarter
  containing today) is `current`.
- **Unresolved targets** are marked with
  `intentionCardDomTargetExists(plugin, target, sourcePath).ok === false`, for kicker
  parts, rail items, and chips.
- **Identity** (pure, computed from the subject):

  | Kind    | Title                                            | Kicker parts after the species word  | Subtitle                                | Tile band / value |
  | ------- | ------------------------------------------------ | ------------------------------------ | --------------------------------------- | ----------------- |
  | day     | `Fri, Nov 20, 2026` (`<Wkd>, <Mon> <d>, <yyyy>`) | `W47` → week path, then plain `2026` | none                                    | `FRI` / `20`      |
  | week    | `2026-W47`                                       | `2026-Q4` → quarter path             | `Nov 16 – 22, 2026`                     | `WEEK` / `47`     |
  | quarter | `2026-Q4`                                        | `2026` → year path                   | `W40–W53 · Sep 28, 2026 – Jan 3, 2027`  | `QTR` / `Q4`      |
  | year    | `2026`                                           | `All time` → `all_time`              | `Dec 29, 2025 – Jan 3, 2027 · 53 weeks` | `YEAR` / `2026`   |
  - Subtitle ranges use `propertyCardPeriodRangeLabel(start, end)` (bob's spaced en
    dash). When start and end fall in the same year, append `, <yyyy>`. A cross-year
    range already carries both years.
  - The quarter's week span and the year's week count come from `subject.label`.
  - Kicker targets are link paths without `.md`, as the kit expects.

- **State pill.**
  - Days: `TODAY` (solid) when `subject.id === today`; `YESTERDAY` or `TOMORROW`
    (outline) at one day's distance (date ordinals); nothing else.
  - Periods: `THIS WEEK`, `THIS QUARTER`, or `THIS YEAR` (solid) when
    `subject.state === "current"`; nothing else.
  - The existing midnight timer repaints the pill (the signature includes `today`).
- **Ghost.** One quiet dashed row, `＋ Set intentions`, with a `==` keycap hint.
  - It shows when `subject.state` is `current` or `future` and the note has no
    intentions. Summary or reflection may exist. Past subjects never show it.
  - It is a `<button type="button">`, so Enter and Space work natively. Click or Enter
    calls `plugin.openIntentionCard({ filePath: path, initialStage: "add" })`, which
    opens the card editor on the intention stage. `==` itself is unchanged: it already
    opens the card editor, with Add selected on an empty open subject.
  - The ghost owns no Properties rows.
- **Owned Properties rows.** The existing seven keys are owned only when the model has
  content (`intentions`, `summary`, or `reflection` nonempty), exactly as before. An
  identity-only card owns no rows, so an empty note keeps every native row.
- **Links.** Kicker links, the parent link, and chips open through one helper,
  `intentionCardOpenLink(target, sourcePath, event)`.
  - It calls `openLinkText(target, sourcePath, paneType)`. `paneType` comes from
    Obsidian's `Keymap.isModEvent(event)`, captured guardedly from `require("obsidian")`
    the way `src/142b-plan-today-roots.js` does. If that is unavailable, use
    `ctrl/meta/shift`. A middle click (`auxclick`, button 1) opens a tab.
  - Rail `prev` and `next`, unmodified, mirror `recordWalkOpen` in
    `src/242v-plugin-record-walk-rail.js`. When the
    `bob-navigation-hotkeys:open-prev-link` / `open-next-link` command exists and the
    source leaf shows the source path, set that leaf active and run the command, so jump
    history matches `[n` / `]n`. Otherwise use `intentionCardOpenLink`.
  - Read the navigation-hotkeys implementation once to confirm that these commands
    follow the note's own nav line for daily (`prev`/`next`) and period (`‹`/`›`)
    labels. If a shape disagrees, use the link fallback for it and note why in a
    comment.

## Steps

### 1. Model: `src/296a-plugin-intention-card-model.js`

- **`intentionCardModelFor(path, frontmatter, today, parentFrontmatter)`** always
  returns a model for a canonical subject whose fields validate. Remove the "no nonempty
  field returns `null`" gate. Set `hasContent` to whether any of `intentions`,
  `summary`, or `reflection` is nonempty.
  - Malformed fields (`fields.ok === false`) still return `null` (fail open).
  - Non-canonical paths and `all_time` still return `null`.
  - Check `intentionCardOverlayModel` in `297d` and the overlay wrapper in
    `296c`/`297d`: a first add or last remove must flip `hasContent`, and nothing may
    re-null the model. The applied overlay still never paints a pending preview.
- **`intentionCardPlacement(source, subject, today)`** returns a placement for notes
  without an unfenced H1:
  `{ headingLine: -1, insertBeforeLine: <first non-blank body line after the frontmatter, or lines.length>, skippedLines: 0 }`.
  - Unterminated frontmatter still returns `null`.
  - Notes with an H1 keep today's result exactly.
  - Update the existing test that expects `null` for a fenced-only H1. It now gets the
    first body line.

### 2. Pure identity and rail model: new `src/296ab-plugin-intention-card-identity.js`

Add it to `src/fragments.json` right after `296a`. Keep it pure: no DOM and no plugin.

- `intentionCardIdentity(subject)` returns
  `{ kind, title, kicker: [{ text, target? }], subtitle, tile: { band, value }, pill }`
  per the table above. It returns `null` for a missing subject.
- `intentionCardStatePill(subject)` returns `{ text, tone }` or `null`.
- `intentionCardTitleRecognizer(subject, frontmatter, today)` returns the kit day or
  period recognizer with the alias. It returns `null` when the kit is missing.
- `intentionCardRailModel(source, subject, recognizer, today)` returns
  `{ range, prev, parent, next, children }` or `null`.
  - Compute the range with `propertyCardTitleSourceRange`.
  - Walk lines `range.fromLine + 1 .. range.toLine`. Take the first line that
    `intentionCardIsNavigationLine` accepts as the nav line, and the first line that
    `intentionCardIsChildLinksLine` accepts as the children line.
  - Use `intentionCardLineLinks` for targets and labels, and `intentionCardLinkSubject`
    for subject identity.
  - Return `null` when the range has no nav line. Children are `[]` when there is no
    children line.
  - Items carry `{ label, shortLabel, target, ariaLabel }`; children carry
    `{ label, target, current }`. Unresolved flags are added later, in the DOM step,
    because they need the vault.
- `intentionCardShowsGhost(model)`: `model.subject.state` is `current` or `future`, and
  `model.intentions.length === 0`.
- Export all of these in `src/375-exports-intention-card.js`.

### 3. Plugin model wrapper: `src/296c-plugin-intention-card-host.js`

- `intentionCardModelFor(path, sourceText)` (the mixin method) attaches the following to
  the returned `model`:
  - `identity` (`intentionCardIdentity(subject)`);
  - `recognizer` (built from the confirmed or overlay frontmatter, with the alias from
    `metadataCache`);
  - `rail` (from the same source text it already reads);
  - `ghost`;
  - `today`.
- It returns `{ model, source, placement, subject, recognizer }`.
- A model `signature` (JSON of path, today, the display fields, identity, rail, ghost,
  and hasContent) drives repaint dedupe. Keep the recognizer object out of the JSON and
  use its `key`.
- Host options:
  - add `titleOwnership: true`;
  - add `"active-leaf-change"` to `workspaceEvents`;
  - add `onRefresh: () => plugin.intentionCardSweepTitleSlots()`.

### 4. Card DOM: `src/296b-plugin-intention-card-dom.js`

`intentionCardBuildDom(model, plugin, doc)` builds, in order:

```text
section.bob-property-card.bob-intention-card[data-card-kind][data-intention-source]
       [data-intention-state=closed|current|future][role=group][aria-label="Day: Fri, Nov 20, 2026"]
  div.bob-property-card-title-block            ← propertyCardBuildTitleBlock (kit)
  div.bob-intention-card-parent                 (unchanged: ↑ parent label + parent intentions)
  div.bob-intention-card-intentions             (unchanged gates)
  section.bob-intention-card-summary            (unchanged gates)
  section.bob-intention-card-reflection         (unchanged gates)
  div.bob-intention-card-missing                (unchanged gates)
  button.bob-intention-card-ghost               (only when model.ghost)
    span.bob-intention-card-ghost-label "＋ Set intentions"
    kbd.bob-intention-card-ghost-keys "==" [aria-hidden]
  nav.bob-property-card-rail                    ← propertyCardBuildRail (kit), only when model.rail
```

- Apply `propertyCardApplyIdentity(card, { kind, title })` (it keeps `role="group"`),
  and drop the old `"<kind> intention card"` label.
- Build the title block with `kind`, `title`, `kicker`, `pill`, `subtitle`,
  `tile: { band, value }`, and
  `onLink: (target, event) => plugin.intentionCardOpenLink( target, model.path, event)`.
  Mark each kicker part `unresolved` before building.
  - If the kit builder returns `null` (kit missing), fall back to the old
    `div.bob-intention-card-kicker` text `DAY · <label>`, so a card is never titleless.
- Build the rail with `label` `Day navigation`, `Week navigation`, and so on, and
  `onNavigate: (direction, item, event) => plugin.intentionCardRailNavigate(direction, item, model.path, event)`.
  Mark items and chips `unresolved` first.
- The ghost button:
  - `aria-label` is `Set intentions`;
  - its click handler calls `preventDefault()` and `stopPropagation()`, then
    `plugin.intentionCardOpenGhost(model.path)`;
  - `mousedown` calls `preventDefault()` so CodeMirror never takes the click.
- Keep `intentionCardDomRelease` and the render-component logic unchanged.

### 5. Title-slot controller: new `src/296d-plugin-intention-card-title-slot.js`

Add it to `fragments.json` right after `296c`. Add a mixin,
`BobLedgerToolsIntentionCardTitleSlotMixin`, to `src/310-install-methods.js` next to
`BobLedgerToolsIntentionCardMixin`.

State is `this.intentionCardSlots`, a `Map` from scope to an entry:
`{ scope, mode: "live" | "reading", path, signature, node, slotRelease, titleRelease, titleKey, fallback, generation }`.

- **`intentionCardSlotMount(scope, { mode, view, path, prepared, currentPath })`**
  1. If an entry exists with the same path, the same signature, `!fallback`, and
     `propertyCardTitleSlotMountedNode(scope) === entry.node`, re-assert ownership only
     and return the node. In Live Preview, re-asserting means the idempotent editor own.
     In Reading, re-own only if the path or `titleKey` changed.
  2. Otherwise, build the card and call
     `propertyCardMountInTitleSlot(scope, node, { path, kind, view, currentPath, onLost })`.
     The primitive detaches a previous node for the same scope; call
     `intentionCardDomRelease(old)`.
  3. If the call returns `null`, record a fallback entry, release title ownership, and
     return `null`. The caller paints the body fallback.
  4. On success, own the title (see the decisions above). Own the keys only if
     `model.hasContent`, otherwise release them. Use the existing
     `intentionCardOwn(scope, path)` / `intentionCardRelease(scope, path)`.
- **`onLost(reason)`** works only if the entry generation still matches:
  - delete the entry, release its title handle and keys, and release its render
    component;
  - mark fallback for that scope and path when `reason` is `unstable` or
    `insert-failed`;
  - then `this.intentionCardHost().scheduleRefresh()`.
- **Live Preview** (`buildIntentionCardDecorations`):
  - Compute `prepared` as today.
  - If the view's entry is a fallback for this path, return today's widget decoration.
    Change only `intentionCardLiveAnchor` for no-H1 placements: put the widget at the
    start of `insertBeforeLine` with `side: -1`, or at the end of the document with
    `side: 1` when there is no body line.
  - Otherwise, schedule `intentionCardSlotMountLive(view, path)` with `queueMicrotask`.
    Dedupe it per view with a pending flag. Return `Decoration.none`.
  - The microtask re-checks that the view is connected, in Live Preview, and on the same
    path (`intentionCardLivePath(view)`). It recomputes `prepared` from
    `view.state.doc.toString()` and mounts, with
    `currentPath: () => view.state.field(editorInfoField, false)?.file?.path || ""`.
  - If the mount returns `null`, dispatch the host refresh effect to that one view (also
    in a microtask, never inside an update), so the decorations rebuild with the
    fallback widget.
  - When `prepared` is `null` or the path is gone, release this view's entry and its
    editor title handle as well as the keys.
- **Reading** (`paintIntentionCardReading(el, sourcePath)`):
  - Resolve root, scope, and gates as today. Find the leaf whose
    `view.previewMode.containerEl` contains the scope, and use its
    `previewMode.renderer` as `view` and `() => leaf.view.file?.path` as `currentPath`.
  - When the scope's entry is a fallback for this path, run today's body anchor path:
    `intentionCardDomAnchor`, extended so a note without a recognized H1 prepends the
    card inside the first content section. No title ownership.
  - Otherwise, call `intentionCardSlotMount` (mode `reading`). If it returns `null`,
    take the body path.
- **`removeIntentionCardReading(root, scope, path)`** and
  **`intentionCardCleanEditor(view)`** also release this scope's slot entry (when the
  path matches, or always for clean) and its title handle.
- **`intentionCardSweepTitleSlots()`** runs from the host's `onRefresh`:
  - Prune entries whose scope is disconnected.
  - For each `markdown` leaf, find its mode: `preview` means `reading`; a source view
    with `getState().source === true` means `source`; anything else is `live`.
  - Release entries on that leaf's scopes that are not the wanted scope for the current
    mode, or whose path differs from `leaf.view.file?.path`. This frees the hidden
    Reading card while editing, so a pane has exactly one `aria-level="1"`.
  - For a Reading-mode leaf on a displayable path with no entry, call
    `paintIntentionCardReading` on its sizer. This covers notes whose post-processor
    never fired.
  - The sweep is O(leaves) and never throws.
- **Dispose:** in `intentionCardOnDisposeStart`, release every entry: slot handle, title
  handle, keys, and render component. The host's `titleOwnership` dispose then calls
  `propertyCardTitleReleaseAll()`.
- **Commit-mount paths:** replace the unconditional `commitMount(scope, path, true)` in
  the fallback widget and Reading body paths with
  `intentionCardCommitKeys(scope, path, model)`, which owns only when `hasContent`.
- **Actions:**
  - `intentionCardOpenLink`, as in the decisions above;
  - `intentionCardRailNavigate(direction, item, sourcePath, event)`: prev and next
    through the navigation command when eligible, everything else through
    `intentionCardOpenLink`;
  - `intentionCardOpenGhost(path)`:
    `openIntentionCard({ filePath: path, initialStage: "add" })`.

### 6. Card editor header: `src/297cc-plugin-intention-card-editor-paint.js`

In `intentionCardEditorPaintAll`, before `h2.bob-intention-editor-title`, add
`div.bob-intention-editor-identity[data-card-kind=<kind>]`. It holds:

- `propertyCardBuildGlyphTile(doc, { kind, band, value })`, from the subject of
  `options.subjectPath` (`intentionCardSubjectForPath`) and `intentionCardIdentity`;
- `span.bob-property-card-kicker-species` with the species word.

Keep the `h2` text and every field, region, and key unchanged. The row repaints when
`[`, `]`, or `=` changes the subject. If the kit is missing, omit the row.

### 7. CSS: one new block after the `.bob-intention-card*` rules in `styles.css`

- `.bob-property-card.bob-intention-card[data-card-kind]` gets `display: grid` and
  `width: auto`. It is block-width (the kit needs it for its container queries), and it
  must outrank the title-slot `display: block` rule.
- Zero the rail's top margin inside the card grid
  (`.bob-intention-card > .bob-property-card-rail { margin-block-start: 0; }`). The grid
  gap already spaces it.
- `.bob-intention-card-ghost`:
  - a full-width row with a dashed `1px` border in `--background-modifier-border`, an
    8px radius, and transparent background;
  - `--text-muted` text, which turns `--text-normal` on hover;
  - a visible `:focus-visible` outline;
  - the `kbd` keycap in the Obsidian `kbd` style;
  - text contrast of at least 4.5:1.
- `.bob-intention-editor-identity` is an inline-flex row. It defines `--bob-day-accent`
  (the same formula as the kit), `--bob-property-card-accent`, and
  `--bob-property-card-ink` locally, so the tile and word match the card without card
  chrome.
- Add `forced-colors` rules (the ghost border is `CanvasText`) and reduced-motion rules
  (no transitions).
- Remove nothing from the kit section. Record `preview-intention-card` before/after
  captures and confirm that intentional changes are the only differences.

### 8. Tests

- **Update** `scripts/test-ledger-tools-intention-card.cjs`, `-acceptance.cjs`,
  `-overlay.cjs`, and any other intention-card suite that asserts the old DOM, the
  `null` empty model, or no-H1 `null` placement:
  - an empty canonical note now yields an identity-only model and card, with no owned
    rows;
  - the overlay test's "removing the last one unmounts it" becomes "bands disappear and
    the identity-only card stays";
  - the DOM assertions use the title block, data-card-kind, and the new aria-label.
- **Add** `scripts/test-ledger-tools-intention-card-title-block.cjs` to `package.json`
  `test:unit`, next to the other intention-card suites. Reuse the FakeNode DOM and the
  stubs of `scripts/test-ledger-tools-property-card-title-slot.cjs` /
  `scripts/intention-card-test-helpers.cjs`. Cover:
  - **Identity** for day, week, quarter, and year: titles, kicker parts and targets,
    subtitles (same-month, cross-month, cross-year week, same-year quarter, Q4 across a
    year, an ISO year starting in the prior December), tile band and value, and pills
    (today, yesterday, tomorrow, past, future, current and non-current periods).
  - **Census fixtures** from the epic. Each asserts the hidden range, the rail, and
    `hidden: none` where expected:
    - `# Day: 2023-07-19 Wed`;
    - `# day = 2024-01-01.Mon` followed by `# pomodoro = 15` (only the first H1 is
      hidden);
    - H1-less `day::2025-03-01 Sat` and `day::2026-01-15 Thu`;
    - the June 2026 shape `# 2026-06-01 Mon` with the unrecognized `prev | day | next`
      line (H1 hidden, no rail, nav visible);
    - the current 2026 shape with a three-link nav;
    - a two-link `prev · next` daily (rail parent from the subject);
    - week, quarter, and year with nav and children lines;
    - the legacy H1-less `2025.md` (card, no rail);
    - `all_time.md` (no model).
  - **Rail:** labels, short labels, aria labels, the current chip for today's day, week,
    and quarter, and unresolved marking through a stub vault.
  - **Ghost:** shown on today, tomorrow, and a current week with no intentions; not
    shown on a past empty day, on a populated day, or on malformed notes (no model).
    Clicking it calls `openIntentionCard` with `initialStage: "add"`.
  - **DOM:** exactly one `[aria-level="1"]` in the card; child order as in step 4; the
    kit-missing fallback still shows a title.
  - **Slot controller** on a fake `.cm-editor > .cm-scroller > .cm-sizer` and a fake
    Reading `.markdown-preview-view > .markdown-preview-sizer > .mod-header`:
    - the mount lands before `.metadata-container`;
    - a repaint with the same signature does not re-insert;
    - a changed signature replaces the node and releases the old render component;
    - `null` from the primitive gives a fallback entry, and the body path runs without
      title ownership;
    - `onLost("unstable")` marks fallback, and `onLost("path-changed")` remounts on
      refresh;
    - keys are owned only with content;
    - the sweep releases a Reading entry when the leaf switches to Live Preview, and
      mounts a Reading-mode leaf with no entry;
    - dispose leaves zero entries, zero slot nodes, and no
      `bob-property-card-owns-title` class.
  - **Link routing:** `Keymap.isModEvent` pane types, middle click opens a tab, and rail
    prev and next run the nav-hotkeys command only for the matching active-capable leaf.
  - **Editor header:** the tile and species word for each kind, and the repaint on
    subject change.
- **Real-Obsidian e2e:** update `scripts/obsidian-intention-card-e2e.cjs`,
  `-e2e-lib.cjs`, `-workflows.cjs`, and `-landing-workflows.cjs` for the new DOM. Keep
  the offline wrapper `scripts/test-obsidian-intention-card-e2e.cjs` in step.
  - Replace the placement checks with title-slot checks in Live Preview and Reading:
    - the card's parent is `.cm-sizer` / `.mod-header`, before `.metadata-container`;
    - the inline title is hidden;
    - the recognized H1 and nav line are not visible;
    - exactly one visible `[aria-level="1"]` in the pane, the card title;
    - the rail shows prev, parent, and next with the fixture targets.
  - Replace `display-empty-fails-open` with `display-empty-identity-card`. In both
    modes: the card shows its title and rail, all Properties stay visible, and nothing
    is owned. A past empty day has no ghost.
  - Keep `display-malformed-fails-open`, the Source, embed, unmount, failure-recovery,
    and plugin-disable checks. Additionally assert that native rendering returns: the
    inline title and H1 are visible again, and there are zero
    `bob-property-card-owns-title` classes.
  - Update the landing workflow's last-remove check: the card stays and the intention
    bands disappear before metadata.
  - Add checks:
    - `display-reveal-on-cursor`: in Live Preview, placing the cursor on the H1 line
      shows the raw H1 and nav. Moving it to the `^gtd` line hides them again.
    - `display-ghost-opens-editor`: today with no intentions uses a new empty
      `today + 3` future fixture; clicking the ghost opens the card editor on the
      intention stage.
    - `display-rail-navigates`: clicking `next ›` on a fixture with an existing next
      note opens it in the same leaf.
    - `display-period-children`: the week shows day chips with today's chip
      `is-current`.
    - `display-legacy-daily`: a new H1-less `day::` fixture at an unused past date shows
      the title, no rail, and its body line visible.
    - `display-bytes-neutral`: hash every fixture before and after open, reveal, and
      close.
  - Add each new fixture only at a path that no existing check requires to stay missing,
    and extend the lib's collision and missing guards.
  - Update `REQUIRED_CHECKS`. Keep the screenshots: light, dark, and 375px, in Live
    Preview and Reading, for today, empty, week, and the legacy daily.
- **Preview:** extend `scripts/preview-intention-card.cjs` fixtures with empty today
  (ghost), a past empty day, a future day, a week with children, a quarter, a year, and
  a legacy daily. Render light and dark at 1280 and 375px through the real
  `intentionCardBuildDom` with the kit CSS. Inspect every PNG yourself and fix what you
  find in the owning fragment.

### 9. Docs

- Retitle `docs/intention-card.md` to **"Day and period cards"** and keep its path and
  URL. Open with one sentence saying it was formerly the intention card. Rewrite only
  what changed:
  - the card always mounts as the note's title block in the title slot;
  - the title, kicker, tile, pill, and docked rail with the children strip;
  - render-time hiding of the recognized H1, nav, and child-link lines, with reveal on
    cursor touch;
  - the ghost on current and future subjects;
  - bands stay content-gated, and rows are owned only with content;
  - fallback and fail-open rules.

  Keep the editor sections, and update its header description.

- Update the nav title in `docs/nav.yml` and the link text in
  `docs/bob-ledger-tools.md`. Run `npm run docs:gen` if `docs:inventory:check` asks for
  it.
- Use the naming from the contract: **day card**, **period card**, and **card editor**.
  Code identifiers stay `intentionCard*`.

### 10. Gates, version, deploy, and evidence

1. `npm run build`, then `npm run build:check`, `npm test`, and `npm run docs:check`
   (with `BOB_CLI_DIR` pointing at the bob-cli workspace checkout if the script needs
   it).
   - If a timing-budget test fails, rerun it alone and on a clean `origin/master`
     worktree. If it fails the same way on the clean tree, it is a known load flake:
     cite `bob-cli-io` or the earlier phase notes, and do not let it block.
2. Bump `plugins/bob-ledger-tools/manifest.json` to the next free minor version.
3. Rebase onto `origin/master` again (see the coordination rules), rebuild, and rerun
   the affected suites.
4. `bob plugins sync -p bob-ledger-tools`: a dry run, then the real run. Confirm that
   the vault `main.js`, `styles.css`, and `manifest.json` are byte-identical to the
   build.
5. Run the updated real-Obsidian intention-card e2e on this host (athena, under xvfb,
   per the script's `--help`) against the final build. Every required check must pass.
   Inspect its screenshots.
6. Attach the evidence with `sase artifact create --bead bob-cli-o6.5`: the e2e report,
   the preview index or report, and the screenshots you relied on.
7. Bead notes on `bob-cli-o6.5`:
   - `bob-cli-o3 SUPERSEDED: the set-intentions ghost shipped in day-period-cards (current and future subjects, opens the card editor on the intention stage); the land agent may close bob-cli-o3 as superseded after the obsidian-proof phase confirms it.`
   - A short summary of what shipped, which surfaces fell back (none expected), the
     version, and the gates.
   - `PROPOSED FOLLOW-UP:` notes for anything out of scope that you found.
8. `sase bead epic-symbols bob-cli-o6.5`. Resolve each symbol, or re-key it to a
   still-open bead. Then close **only** `bob-cli-o6.5` with a `--note` stating what you
   verified. Never close the epic or any other bead.
9. The bob-plugins commit
   (`feat(ledger-tools): day and period cards own the title block`) lands through
   `/sase_final` at the end of the turn.

## Acceptance

- In both Live Preview and Reading:
  - today's daily, a past empty daily, a legacy H1-less daily, the week, the quarter,
    and the year each show exactly one visible title, the card's;
  - the card is in the title slot above Properties;
  - the kicker reads `DAY` / `WEEK` / `QUARTER` / `YEAR`;
  - the leaf tile and temporal accent show, and no area hue;
  - the docked rail matches the hidden nav line, and periods show children chips with
    today's chip current.
- Placing the cursor on a hidden line reveals the raw Markdown. Moving away hides it
  again.
- Current and future subjects without intentions show `＋ Set intentions`. Click or
  Enter opens the card editor on the intention stage. Past subjects never show it.
- Bands, the parent line, and the recent-closed missing band behave exactly as before.
  Rows are owned only when the card has content.
- Source mode, embeds, hover popovers, malformed frontmatter, mobile, plugin disable,
  and unload restore native rendering. Note bytes never change.
- `==`, `Ctrl+Shift+P`, every card-editor key, and `bob periods update` behave as
  before. The editor header shows the tile and species word.
- All gates pass, or fail only on cited load-only flakes that also fail on a clean tree.
  The vault deploy matches the build, and the e2e report is attached.
