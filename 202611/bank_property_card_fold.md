---
tier: epic
title:
  "Bank as a proper property card: folded Setup, hidden native Properties, and the `_`
  fold key"
goal: "The bank card on bank.md owns the note top as a title-slot property card whose
  Setup folds by default (problems stay visible), native Obsidian Properties hide behind
  the fail-open coordinator, and a guarded bare `_` folds the card's own section on
  bank, album, record, area, and project cards.

  "
decisions:
  family_fold_look:
    ask:
      When `_` reaches record, area, and project cards, should their folds keep the
      compact kicker chevron?
    choices:
      compact:
        Chevron stays in the kicker tools and gains `_`, tooltip, and ids; card heights
        stay pixel-stable
      row:
        Chevron becomes the full-width fold row bank and album use; each card grows by
        one line
    default: compact
    why:
      Those folds hide a whole card body, not a named section; a row adds height, not
      meaning
    answer: compact
  memory_glossary_cards:
    ask:
      Correct the glossary Property Card and Bank Card strands once bank owns its title
      and Properties?
    memory:
      - glossary:property-card
      - glossary:bank-card
    default: true
    corrects: Bank keeps its native block.
    answer: true
  memory_card_keys_record:
    ask:
      Add a decisions-web record fixing the card key family (`-`/`_` show, `==`/`+`
      edit)?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: fold-core
    title: Shared fold kit, the `_` key, the command, and album Schema
    depends_on: []
    size: medium
    description:
      "fold-core: add the 240s fold kit (state helper, node-based registry, fold row),
      species foldDefault, the guarded `_` arm and toggle-card-details command plus
      Command Mode row, and move album Schema onto the kit with tests, e2e, docs, and
      the vimrc comment."
  - id: bank-fold
    title: Setup collapses by default and the folded card stays honest
    depends_on:
      - fold-core
    size: medium
    description:
      "bank-fold: bank Setup resolves through the fold helper (collapsed when healthy,
      self-opening and still toggleable when broken), gains the shared fold row with an
      attention chip, and fixes the gap tone, rule numbering, ids, and stale-notice
      placement."
  - id: bank-title
    title: The bank card moves into the title slot and owns the title
    depends_on:
      - bank-fold
    size: medium
    description:
      "bank-title: new 248cb title mixin adopts the title slot with an emoji-splitting
      identity model and bank recognizer, retires the badge, moves the edit action into
      the kicker as ⊞ Setup, adopts the real hue walk, and releases cleanly in Source
      mode."
  - id: bank-takeover
    title: Hide native Properties on bank.md behind the fail-open gate
    depends_on:
      - bank-title
    size: medium
    description:
      "bank-takeover: spike bank.md YAML parity in real Obsidian, add bank to the 240q
      coordinator, acquire and release ownership from the bank host, promote the
      bank-native e2e to full takeover verification, flip docs, and apply the
      decision-gated glossary corrections."
  - id: fold-family
    title: The `_` key reaches record, area, and project folds
    depends_on:
      - fold-core
    size: medium
    description:
      "fold-family: register record, area, and project folds with the kit (look per
      family_fold_look), give the foot's `-` disclosure the same hover keycap, and cover
      each kind in unit and e2e tests and docs."
  - id: bank-polish
    title: Area strip, screenshot review, and passive-rendering proof
    depends_on:
      - bank-takeover
      - fold-family
    size: small
    description:
      "bank-polish: render the authored-only area strip on the bank card, prove folding
      is passive, review light/dark wide/narrow screenshots across the family, sweep
      docs, and write the optional key-family decision record."
proposed_by: bbugyi200.athena.1g2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 12:49:28
status: wip
---

- **PROMPT:**
  [prompts/202611/bank_property_card_fold.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_property_card_fold.md)

# Bank as a proper property card: folded Setup, hidden native Properties, and the `_` fold key

## Goal

Make the bank card on `~/bob/bank.md` a first-class member of the property-card family,
like the album, record, and day/period cards:

1. **Setup folds by default.** A healthy bank card rests as one calm line. Problems
   still show on that line, and a broken config opens itself.
2. **`_` folds the card's own section.** In Live Preview Vim normal mode, a bare `_`
   toggles bank **Setup** and album **Schema**. Album Schema stays expanded by default.
   The same key then reaches record, area, and project folds, so the keyboard reads the
   same on every card:

   | Key  | Job                                                        | On-screen twin         |
   | ---- | ---------------------------------------------------------- | ---------------------- |
   | `_`  | Show/hide this card's own section (Setup, Schema, details) | The fold row / chevron |
   | `-`  | Show/hide the complete Properties drawer                   | The foot disclosure    |
   | `==` | Open the species editor                                    | `⊞ Setup` / `⊞ Schema` |
   | `+`  | Open the Property Editor (AREA SETTINGS on `bank.md`)      | Foot affordances       |

   The minus key shows and hides; the equals key edits.

3. **The card owns the note top.** The bank card moves into the title slot and owns the
   title (`[🏦] BANK · Cash` / `Bank` / summary subtitle). The whole-frontmatter
   coordinator then hides Obsidian's native Properties block, behind the same fail-open
   evidence gate the other owned species use.

Source of requirements: research report
`research:202611/bank_property_card_fold/bank_property_card_fold.md`. Bryan accepted all
of its recommendations. That includes the staged universal `_`, the title-slot move as a
hard prerequisite of the hide, and the baseline bug fixes. Its memory suggestions are
only the decisions above.

All code changes ship in the **bob-plugins** linked repo (`bob-ledger-tools`, plus one
`bob-command-mode` row). Open it with `/sase_repo` (`sase repo open bob-plugins`). There
are no bob-cli code changes, no vault migration, and zero edits to `~/bob/bank.md`. The
only vault edit is one comment line in `~/bob/obsidian_vimrc.md`. The only bob-cli edits
are the decision-gated memory strands and their generated docs glossary.

## Ground truth (verified in bob-plugins master before planning)

All paths are under `plugins/bob-ledger-tools/src/` unless noted. Line numbers are
approximate. Re-check them before editing.

- **Bank card today**: `248a-plugin-bank-setup-model.js` (pure model),
  `248b-plugin-bank-setup-dom.js` (DOM), `248c-plugin-bank-setup-hosts.js` (host via
  `createPropertyCardHost({ kind: "bank" })`, ~800 lines),
  `248ca-plugin-bank-setup-foot.js` (shared foot adapter).
  - The card mounts in the body after the first H1, as a CodeMirror widget in Live
    Preview and via `insertBefore` in Reading. It has no `titleOwnership`. Native
    Properties stay visible, with only the `bank` row hidden per row
    (`propertyCardOwnBankKeys`, `240b`).
  - `BANK_SETUP_DEFAULT_EXPANDED = true`. The session-only Map is
    `plugin.bankSetupExpanded`.
  - **Invalid-state lock bug**: `bankSetupModel` hardcodes
    `expanded: state === "invalid" ? true : expanded`, so toggling an invalid card flips
    the Map and the repaint ignores it.
  - **Collapsed cards hide problems**: findings, the stale "Updating" note, and account
    status paint only inside the body, after `if (!model.expanded) return root`.
  - **Dangling `aria-controls`**: the toggle points at `"bob-bank-setup-body"`, but the
    body only has that string as a class, never an id.
  - **Double-numbered rules**: `bankSetupPaintRules` emits an `<ol>` and an index span,
    and CSS never sets `list-style: none`.
  - **`gap` status has no tone**: `bankSetupStatusTone` maps only balanced/stale/delta.
    `gap` falls to muted `·`.
  - Responsive bank CSS uses viewport `@media (max-width: 480px)` blocks (`styles.css`
    ~10650). These blocks can't see a narrow split pane.
  - A stale comment in `248c` says the card mounts "exactly where the album header
    mounts". Album moved to the title slot long ago.
  - `bankCardRootHue` (`241m`) reads only `bank.md`'s own `color` and falls back to a
    literal `"magenta"`. Album uses the real walk, `notebookHostHueName`.
- **Album card**: host `241v-plugin-album-header.js` (`titleOwnership: true`,
  `propertyCardAdoptTitle`, frontmatter coordinator). The schema fold DOM is in
  `241x-plugin-album-schema-dom.js`: a `button.bob-album-schema-summary` whose text is
  `▾ Schema · N fields · …`. The chip card id is `bob-album-schema-chips-${stem}`, which
  duplicates across two panes. `albumHeaderDefaultExpanded()` (`241w`) always returns
  true. There is no keymap.
- **Key dispatcher**: `248m-plugin-schema-double-equals.js` handles one capture-phase
  keydown on window and document, with WeakSet dedup and `stopImmediatePropagation`.
  - `handleSchemaDoubleEqualsKeydown` filters to `=`, `-`, `+`.
  - `propertyCardBareKeyGate` accepts only `"-"`/`"+"`. It falls through on modified,
    repeat, composing, default-prevented, not-content, not-live-preview, no-vim,
    not-normal, operator-pending, key-buffer, counted, register, and an arm that can't
    act.
  - The `-` arm uses `propertyCardCanToggleProperties(view)`
    (`240p-plugin-property-card-foot-state.js`): a connected, path-matching mount inside
    the supplied view, resolved via `propertyCardFootViewPath`.
  - The command `toggle-card-properties` is registered here. Its Command Mode row is in
    `plugins/bob-command-mode/src/160-sheet-rank.js` (`CM_SHEET_SURFACE_ROWS`, section
    `vault`).
- **Existing folds without a key**: the record sheet collapse button is in `242j` (~line
  216, "Collapse record details") with its Map in `242k` (`recordSheetCollapsed`). The
  area/project home card collapse is in `249c` (`homeCardDomAppendCollapse`, "Collapse
  card details") with its Map in `249d` (`homeCardCollapsed`). Day/period cards have no
  fold.
- **Title kit**: `240e` slot, `240g` title block (`propertyCardBuildTitleBlock` →
  `{ element, tile, kicker, heading, pill, subtitle, actions }`), `240h` recognizers
  (`propertyCardAlbumTitleRecognizer`, `propertyCardHomeTitleRecognizer`), `240i` title
  ownership, `240k` adopter glue
  (`propertyCardAdoptTitle(scope, node, { path, kind, view, recognizer, currentPath, onLost, onMount })`
  → `{ mode: "slot" | "fallback" }`). The newest adopter pattern is
  `249d-plugin-home-card-host.js`. It shows `shouldRebuild` on `editorLivePreviewField`
  flips and `workspaceEvents`.
- **Species registry**: `240f` (`PROPERTY_CARD_SPECIES`; bank =
  `{ word: "Bank", tile: "plain", icon: "landmark" }`).
  `propertyCardAccessibleName(kind, title)` returns `"<Word>: <title>"`.
- **Frontmatter coordinator**: `240q-plugin-property-card-frontmatter-ownership.js`.
  - `propertyCardFrontmatterMountPath` (~511) knows only
    `data-album-source`/`data-record-source`/`data-intention-source`.
  - The eligibility list (~523) is
    `["album", "record", "day", "week", "quarter", "year"]`.
  - The capability-change refresh list (~700) is `scheduleAlbumHeaderRefresh`,
    `scheduleRecordSheetRefresh`, `scheduleIntentionCardRefresh`.
  - `propertyCardFrontmatterCacheEvidence` requires `parseYaml(source)` to equal the
    metadata cache.
- **Build and test conventions**: see the repo `AGENTS.md`.
  - Plugins build from `src/fragments.json` order. Fragments share one script scope, and
    each hand-edited fragment stays ≤ 1000 lines.
  - Mixins are composed in `310-install-methods.js`. Test exports live in
    `374-exports-property-card.js`.
  - Unit tests are `scripts/test-*.cjs` and must be listed in `package.json`
    `test:unit`.
  - Real-Obsidian e2e runs on athena with
    `xvfb-run -a node scripts/obsidian-*-e2e.cjs …`. See `docs/develop/validation.md`
    for the `BOB_BIN`/`BOB_CLI_DIR` recipes.
  - Every feature/fix commit bumps the plugin's `manifest.json` version.
  - Finish with `npm run build`, `npm test`, `npm run docs:gen && npm run docs:check`,
    then `bob plugins sync`.
- **Live `bank.md`**: frontmatter `parent: "[[cash]]"`, `type: "[[area]]"`, `summary`,
  `created` (offset timestamp), and nested `bank:` (1 account, ~21 flow- mapping
  categories with quoted digit hotkeys, 10 rules). Then `# 🏦 Bank`, prose, a `bob-bank`
  queue fence, and `## Albums`. `+` already opens typed AREA SETTINGS there. Home cards
  exclude `bank.md` (`HOME_CARD_TYPE_NOTES`).
- **Vault vimrc**: `~/bob/obsidian_vimrc.md` maps no `_`. It has a comment reserving
  `-`/`+` for the ledger-tools key gate.

## Design (shared by every phase)

### Interaction rules

- `_` uses the existing `248m` bare-key gate and nothing else: no document-wide hotkey,
  no vimrc mapping, no `=`-chord.
  - Match `event.key === "_"`. Shift is allowed because `_` is Shift+`-` on US layouts.
    Never match a physical key code.
  - Every existing fall-through stays: `2_`, `d_`, `"a_`, insert, visual, replace,
    Source mode, and Reading all keep Vim's first-non-blank motion.
  - `_` is consumed **only** when the active view has a live, toggleable card fold.
- `_` acts on the note's mounted card, not on the cursor position. It never targets a
  `bob-bank` fence, an album catalog row, an embedded card for another note, or a card
  in another pane.
- Folding never writes YAML, never spawns bob from a paint path, and never touches the
  `-` drawer state (or vice versa). Opening an editor never expands a fold.
- Fold state is **session-only**. It is kept in per-path Maps shared across panes and is
  never written to frontmatter, settings, or localStorage. Plugin reload resets it.
- **Focus.** `_` from the editor leaves focus in the editor, so a second `_` toggles
  back. Clicking or keyboard-activating the fold control keeps focus on that control
  across the repaint (existing `data-*-focus` restore).
- **Scrolling.** Expanding through `_` or the command reveals the fold row if it is
  off-screen (`block: "nearest"`), the way `-` reveals the foot. Collapsing never
  scrolls. No jump-history entry.
- **No motion.** Fold controls re-render on toggle, so transitions could never run
  reliably. Toggles are instant, and card height never animates.
- **Command.** "Toggle card details" (`toggle-card-details`, palette + Command Mode,
  next to "Toggle card properties"). It covers Reading view and non-Vim use, and is the
  remap/disable route for the bare key. It returns false and does nothing when the
  active view has no fold, matching the sibling command.

### Fold state: health picks the default, the user's choice always works

Replace every hardcoded default and the invalid lock with one pure kit helper:

```
propertyCardFoldResolve({ choice, problems, foldDefault }) -> open
  problems: "" when healthy; otherwise a stable signature of blocking
            problems (state + sorted error messages)
  choice:   { open, problems } recorded at toggle time, or undefined
  if problems == "":  return choice ? choice.open : foldDefault
  else:               return choice && choice.problems === problems ? choice.open : true
```

- A broken or missing config opens itself. A **new or different** problem re-surfaces
  once, even if the user collapsed earlier.
- An explicit toggle always works, including on a broken card, because the choice is
  stamped with the current `problems`.
- When a healthy paint finds a choice stamped with problems, it restamps it to `""` and
  keeps `open`. One session-Map write, no vault write. The next break then re-surfaces
  again.
- Warnings and account status never force the card open. They ride the attention chip.
- Bodyless states (`loading`, `unsupported`, `refresh-failed`, `mobile`, and `missing`
  without a starter) are **not foldable**. They render a static row without fold
  attributes, so `_` falls through to Vim.

`foldDefault` lives on the species registry (`240f`, one field per species): bank
`false`; album, record, area, project `true`; day/week/quarter/year have no fold. The
next default flip is then a one-line diff with one test.

### Fold registry: node-based, view-local, never stale

New kit fragment `240s-plugin-property-card-fold.js`, placed after `240r` in
`fragments.json`:

- `propertyCardRegisterFoldKind(plugin, kind, { toggle(path, view) })` returns an
  unregister function. The kind adapter flips its session choice and repaints.
- Each foldable control carries `data-card-fold="<kind>"`,
  `data-card-fold-source="<normalized path>"`, and `aria-expanded`.
- `propertyCardFoldTarget(view)` resolves the view path with `propertyCardFootViewPath`.
  It then finds the first connected, visible (non-zero rect), path-matching
  `[data-card-fold]` in `view.dom`/`containerEl`/`contentEl` whose kind is registered.
  - Title-slot cards live under `containerEl`, so they are found.
  - Embeds carry another path, and other panes aren't in the view, so both decline.
- `propertyCardCanToggleDetails(view)` and `propertyCardToggleDetails(view)` wrap
  `propertyCardFoldTarget`. When opening, `propertyCardToggleDetails` records the path
  in a pending-reveal Set. The next paint of an open fold row for that path consumes it
  and reveals the row.
- `propertyCardFoldNextId()` returns a session-unique id for `aria-controls` (like
  `propertyCardFootNextDisclosureId`). Two panes on one note never share ids.
- `propertyCardFoldKeyLive(plugin, surface)` is true only on desktop Live Preview with
  Obsidian Vim mode on (`app.vault.getConfig("vimMode") === true`).

### The fold row: one shared, beautiful control

Kit builder `propertyCardBuildFoldRow(doc, spec)` in `240s` returns
`{ element, bodyId }`. Bank and album use it.

```
spec: { kind, path, noun, summary, open, tone?: "error", attention?, extras?,
        keyLive, tooltip, onToggle(next) }
```

- **Element.** A real `<button type="button" class="bob-property-card-fold">` that spans
  the card width.
  - Attributes: `aria-expanded`, `aria-controls=<bodyId>`, `aria-keyshortcuts="_"`,
    `title=<tooltip>`, and the registry data attributes.
  - Content, in order: a chevron span (`▾` open / `▸` closed, aria-hidden), the **noun**
    in normal ink (weight 600), then the summary clause in muted interface text, joined
    with `·`. Optional extras come next (album's `walks by …` pill). The **attention
    chip** is pushed to the trailing edge. A `<kbd>` keycap `_` comes last.
  - The visible text grammar stays `▸ Setup · WF Checking · 21 categories · 10 rules`
    and `▾ Schema · 5 fields · …`. This keeps existing e2e regexes valid.
- **Attention chip.** At most one chip, a non-interactive span, so clicking anywhere on
  the row toggles. Show the most severe item first, with `+N` overflow. Use words plus
  an icon, never color alone (`✗` error tone, `⚠` warning tone). The chip renders in
  both states, so toggling never shifts the row. No "all good" chip ever: unknown or
  loading data never reads as healthy. Money never appears on this row.
- **Keycap.** Rendered only when `keyLive`. It is invisible at rest and fades to ~70%
  only on row hover or `:focus-visible`. The key teaches itself without resting noise.
- **Look.** No border or box. It is a transparent row on the card's existing tinted
  panel, with a subtle hover wash
  (`color-mix(in srgb, var(--bob-property-card-accent) 6%, transparent)`) at the card
  radius. Focus-visible uses the house 2px `--interactive-accent` outline.
  - Text wraps (`overflow-wrap: anywhere`, no nowrap ellipsis).
  - New CSS uses container queries or intrinsic wrapping, not viewport media queries.
  - Override Obsidian's `button:not(.clickable-icon)` box styling the way
    `.bob-album-header .bob-album-schema-summary` already does.
- **Body.** The body element always exists with `id=bodyId`. When collapsed it is an
  empty `hidden` shell, matching the foot drawer convention, so `aria-controls` never
  dangles and the collapsed body contributes no tab stops.
- **Compact variant.**
  `propertyCardDecorateFoldControl(button, { kind, path, open, bodyId, tooltip, keyLive })`
  applies the same attributes, tooltip, and keycap behavior to an existing chevron
  button (used by record/area/project).

Tooltips use the house style: `Toggle bank setup (_)`, `Toggle album schema (_)`,
`Toggle record details (_)`, `Toggle card details (_)`. The foot keeps
`Toggle properties (-)`.

### The resting bank card (end state)

```
╭──────────────────────────────────────────────────────────────────╮
│ [🏦]  BANK · Cash                                      ⊞ Setup  │  kicker: species word + parent crumb (real hue walk)
│       Bank                                                      │  owns the note title; `# 🏦 Bank` hides at render time
│       Every transaction and deposit, categorized in monthly…    │  subtitle = summary
│ ▸ Setup · WF Checking · 21 categories · 10 rules      ⚠ stale  │  shared fold row (+ hover keycap `_`)
│ ───────────────────────────────────────────────────────────────  │
│ ▸ Properties · 5                            created Nov 11, 2026│  shared foot (`-`), outside the fold
╰──────────────────────────────────────────────────────────────────╯
teaching prose · bob-bank queue · ## Albums                          ← now the first screen
```

The card has one identity stack, one disclosure line, and one foot: no leftover BANK
badge and no new hues. Collapsed, it should land near the height of an album card with
Schema folded.

## Phases

### Phase `fold-core`: shared fold kit, the `_` key, the command, and album Schema

Build the kit pieces from the Design section and prove them end-to-end on the album
card, whose Schema fold already exists.

1. **`240s-plugin-property-card-fold.js`.** Add it to `fragments.json` after `240r`, and
   export the new functions for tests in `374-exports-property-card.js`. Contents:
   `propertyCardFoldResolve`, the choice/restamp helpers, the registry,
   `propertyCardFoldTarget`, `propertyCardCanToggleDetails`,
   `propertyCardToggleDetails`, the pending-reveal Set, `propertyCardFoldNextId`,
   `propertyCardFoldKeyLive`, `propertyCardBuildFoldRow`, and
   `propertyCardDecorateFoldControl`. Add the matching CSS block to `styles.css` in the
   property-card kit section.
2. **`240f` species registry.** Add `foldDefault` per species as specified (record,
   area, and project included now, so later phases never touch `240f`).
3. **`248m` dispatcher.**
   - Add `"_"` to the keydown filter and to `propertyCardBareKeyGate`'s accepted keys.
   - Add an `_` arm:
     `{ reason: "details-toggle", canAct: () => propertyCardCanToggleDetails(view), run: () => propertyCardToggleDetails(view) }`.
   - Register `toggle-card-details` ("Toggle card details") next to
     `toggle-card-properties`, with the same `callback` shape and
     `schemaDoubleEqualsActive` guard.
   - Update the fragment header comment.
4. **`bob-command-mode`.** Add
   `{ plugin: "bob-ledger-tools", id: "toggle-card-details", title: "Toggle card details", section: "vault" }`
   right after the "Toggle card properties" row in `src/160-sheet-rank.js`. Rebuild and
   bump that plugin's manifest too.
5. **Album adoption** (`241v`, `241w`, `241x`).
   - Paint the Schema summary with `propertyCardBuildFoldRow`, keeping the legacy
     classes (`bob-album-schema-summary`, `data-schema-fold`, `is-error`) and the text
     grammar so e2e selectors survive.
   - Noun `Schema`. Tooltip `Toggle album schema (_)`. The `walks by` pill becomes an
     `extras` node.
   - Attention chip from `model.issues`: errors `✗ N problems`, otherwise warnings
     `⚠ N warnings`.
   - Replace `albumHeaderIsExpanded`/`albumHeaderDefaultExpanded` with
     `propertyCardFoldResolve`. Problems signature = the schema-invalid issues;
     `foldDefault` comes from the species registry, so album still defaults open.
   - The chip card id comes from `propertyCardFoldNextId()`, not the stem.
   - Register the `album` fold kind in the host's `onSetup` and unregister in
     `onDisposeStart`.
   - Pass `keyLive` (Live Preview surface only).
6. **Tests.**
   - New `scripts/test-ledger-tools-property-card-fold.cjs`, added to `test:unit`. It
     covers:
     - the fold-state matrix: species defaults, problems open, explicit choice wins, a
       different problem re-surfaces, healthy restamp, and repeated breaks;
     - registry view-locality: connected/visible/path-match accepts; another pane, an
       embed for another path, detached, disabled, or bodyless controls decline;
     - fold row DOM: `aria-controls` resolves to an existing element, ids are unique
       across two mounts of one path, tooltip, `aria-keyshortcuts`, keycap only when
       `keyLive`, chip in both states, no money text;
     - pending-reveal consumption.
   - Extend `scripts/test-ledger-tools-schema-double-equals.cjs` with a `_` gate matrix:
     every fall-through reason, Shift allowed, never consumed without a fold, and
     `-`/`+`/`==` behavior byte-identical. Also cover the new command (registered;
     returns false without a fold).
   - Extend the album header tests and the Command Mode surface-row test.
7. **E2E** (athena, real Obsidian). Extend `scripts/obsidian-album-ui-e2e.cjs` with a
   `_` round trip in Live Preview Vim normal mode: collapse, expand, focus stays in the
   editor, `2_` and `d_` remain Vim motions, and the command works in Reading. Run it
   with `BOB_BIN` per `docs/develop/validation.md`. If a real run is impossible, say so
   and keep the offline wrapper green.
8. **Docs.**
   - Turn `docs/bob-ledger-tools.md` "Property-card properties drawer" into a short
     key-family section that adds `_`, the command, and the Vim trade (bare `_` ≈ `^`,
     which still works).
   - In `docs/albums.md`, mention `_` where the Schema fold is described.
   - Regenerate the commands table (`npm run docs:gen`) and run `npm run docs:check`.
   - Vault: update the `~/bob/obsidian_vimrc.md` comment line to name `-`, `_`, and `+`
     as reserved for the ledger-tools key gate.
9. Bump the manifests, `npm run build`, `npm test`, then `bob plugins sync`.

**Done when:** `_` toggles album Schema in Live Preview Vim normal mode and through the
command in Reading. It falls through everywhere else, and the album looks and defaults
exactly as before apart from the keycap/tooltip and ids.

### Phase `bank-fold`: Setup collapses by default and the folded card stays honest

1. **`248a` model.**
   - Delete `BANK_SETUP_DEFAULT_EXPANDED` and the `state === "invalid"` override. The
     model takes a resolved `expanded` and also exposes:
     - `foldable`: valid, invalid, and missing-with-starter;
     - `problems`: `""` for valid; for invalid/missing, the state plus sorted error
       finding messages;
     - `attention`: the chip spec.
   - `bankSetupStatusTone("gap")` → `{ icon: "⚠", word: "gap", tone: "warn" }`.
   - The collapsed clause drops its leading `🏦` (the badge/tile carries it).
   - Signatures must include the attention spec.
2. **Attention order.** Invalid config errors (`✗ N problems`, or `✗ invalid` with zero
   located errors) → warning findings (`⚠ N warnings`) → the worst non-balanced account
   (`✗ delta`, then `⚠ stale`, then `⚠ gap`). `+N` counts the remaining items. The
   tooltip names the accounts and hints.
3. **`248c` host.**
   - `bankSetupModelFor` resolves `expanded` via `propertyCardFoldResolve` with the bank
     `foldDefault`. Choices live in `bankSetupExpanded` as `{ open, problems }`, with a
     restamp on healthy paints.
   - `bankSetupToggleExpanded` records a stamped choice.
   - Register the `bank` fold kind in `onSetup` and unregister in `onDisposeStart`.
4. **`248b` DOM.**
   - The head row keeps the species badge and the `⊞ Edit`/`↗ YAML` buttons. Below it
     sits the full-width `propertyCardBuildFoldRow` (noun `Setup`, tooltip
     `Toggle bank setup (_)`, `data-bank-setup-focus="toggle"` kept so focus restore
     works).
   - Bodyless states paint a static row with today's wording.
   - The stale notice moves **outside** the fold as one status line under the row, with
     Retry when an error is known: `Updating · last good values from 3:41 PM`. The
     fresh-state `last good` footnote stays inside the expanded body.
   - Fix the rules double numbering: the `<ol>` gets `list-style: none` and no padding,
     and the index span becomes a muted tabular-nums gutter.
   - Replace the bank `@media (max-width: 480px)` rules this card needs with intrinsic
     wrapping or container queries.
5. **Tests.**
   - Update `scripts/test-ledger-tools-bank-setup.cjs`: invalid is no longer forced but
     opens by default; an explicit collapse on invalid holds; a new finding re-surfaces
     it; healthy defaults collapsed; attention order and `+N`; `gap` tone; single rule
     numbering; unique ids and live `aria-controls`; stale line outside the fold;
     bodyless states carry no fold attributes.
   - Update the e2e fixtures and assertions in `scripts/obsidian-bank-ui-e2e*.cjs` that
     assume expanded-by-default. Add a `_` round trip on `bank.md`, plus an
     invalid-config case proving the toggle works.
6. **Docs.** Update the bank card paragraph in `docs/bob-ledger-tools.md`: collapsed by
   default when healthy, `_`, and the attention chip. Then bump, build, test, and
   `bob plugins sync`.

**Done when:** a healthy `bank.md` rests as one Setup line showing any warning or
account problem. A broken config opens itself and can still be folded, and `_` toggles
it.

### Phase `bank-title`: the bank card moves into the title slot and owns the title

`248c` is near the 1000-line ceiling. Put the new code in
`248cb-plugin-bank-setup-title.js` (mixin `BobLedgerToolsBankSetupTitleMixin`). Register
it in `fragments.json` after `248c` and in `310-install-methods.js`. Follow `241v`
(album) and `249d` (home) as the reference adopters.

1. **Host options** (`248c`).
   - Set `titleOwnership: true`.
   - Add `shouldRebuild` on `editorLivePreviewField` flips (copy `249d`).
   - Add `onResolveSurface` stray cleanup like album.
   - Delete the stale "exactly where the album header mounts" comment.
2. **Identity model** (pure, testable):
   `bankSetupIdentityModel(path, { metadataCache, vault, sourceText })`.
   - The title comes from the first body H1. A leading emoji grapheme cluster
     (Extended_Pictographic with VS16/ZWJ sequences) becomes the tile emoji and the
     remainder becomes the title: `# 🏦 Bank` → tile `🏦`, title `Bank`.
   - Fallbacks: tile `🏦`, title humanized from the stem.
   - Reuse an existing leading-emoji helper if one exists (for example near
     `242f-plugin-emoji-slot-model.js`). Otherwise add `propertyCardSplitLeadingEmoji`
     to the kit.
   - The kicker crumbs come from the parent chain, like album owners: `BANK · Cash`.
     Crumbs navigate, and Mod opens a tab.
   - The subtitle is the frontmatter `summary`.
3. **Recognizer.** Add `propertyCardBankTitleRecognizer(rawHeading)` to `240h`. It
   matches the first H1 whose cleaned text equals the raw heading (emoji included) and
   has no followers. The H1 hides at render time and reveals under a Live Preview cursor
   or selection, with zero bytes changed in `bank.md`.
4. **Accessible name.** In `240f`, when the title folds equal to the species word,
   `propertyCardAccessibleName` returns the word alone (`Bank`, not `Bank: Bank`). Add a
   unit test.
5. **Mount.**
   - Build the card with `propertyCardApplyIdentity` and `propertyCardBuildTitleBlock`
     (kind `bank`, tile emoji, kicker, title, subtitle).
   - Then append the fold row, the stale line, the body, and the foot.
   - Adopt with `propertyCardAdoptTitle` in Reading and Live Preview (deferred mount,
     `onLost` → reschedule). Keep today's body placement (H1 anchor / CM widget) as the
     fallback, with the kit's demoted heading.
   - Adoption runs on desktop only. Embeds, popovers, canvas, print, and mobile keep
     body placement. The BANK badge is retired: the tile and kicker carry identity.
6. **Header simplification.**
   - Move the edit action into the title block's actions slot, renamed `⊞ Setup`
     (`aria-label` "Edit bank setup", focus key `edit` kept). This matches album
     `⊞ Schema` and area `⊞ Settings`.
   - `↗ YAML` shows only in invalid, missing, unsupported, and mobile states, where
     Source repair is the primary action. A click on `⊞ Setup` never toggles the fold.
7. **Hue.** The card uses `notebookHostHueName` (settled cache), like album, so it
   follows `cash`'s hue walk. `bankCardRootHue` (`241m`) tries the same walk before its
   `"magenta"` literal, so fences match the card.
8. **Source mode.** Bank's `viewPath` still resolves in Source mode for lint. The
   Source/lint branch must explicitly release adoption
   (`propertyCardAdoptRelease(view.dom, path, "bank")`), title ownership, keys, and the
   foot, so a mode flip never leaves a hidden H1 or a stranded slot card.
9. **Foot.** `248ca` `where` marks become truthful:
   `{ type: "tile", parent: "kicker", summary: "subtitle", bank: "setup", created: "foot" }`.
   Mark only what actually painted. The per-row `bank` key hide stays.
10. **Density.** Move the `0.9em` font-size off the `.bob-bank-setup` root onto the
    Setup body, so the title block inherits full note-text sizing like album/area cards.
11. **Tests.**
    - New `scripts/test-ledger-tools-bank-setup-title.cjs`, added to `test:unit`. It
      covers: emoji split and fallbacks, recognizer match/miss, accessible name, the
      `⊞ Setup`/`↗ YAML` state matrix, hue source, foot `where` marks, Source-mode
      release, and the `shouldRebuild` flip.
    - Extend `scripts/obsidian-title-block-e2e.cjs` (or the bank UI e2e) with a bank
      case:
      - slot mount in LP and Reading;
      - inline title and `# 🏦 Bank` hidden, and the H1 revealed under the cursor;
      - no `[🏦] 🏦 Bank` doubling;
      - `j`/`k`/`gg` never trap around the card;
      - LP→Source→LP leaves no residue;
      - `_` and `-` still work from the slot.
12. **Docs.**
    - `docs/bob-ledger-tools.md` bank card paragraph: title slot, `⊞ Setup`.
    - `docs/develop/architecture.md` (~209): bank no longer "shows its species badge
      without owning the title".
    - `docs/property-editor.md` wherever it says the same.
    - Then bump, build, test, and `bob plugins sync`.

Native Properties stay visible in this phase. Area/project cards already sit in that
state, so it is a safe stopping point.

**Done when:** `bank.md` opens with the title-slot card as its only visible title, with
no duplicated emoji, and every fallback context still shows the body card.

### Phase `bank-takeover`: hide native Properties on bank.md behind the fail-open gate

1. **Parity spike first.**
   - Add a `bank.md` fixture to `scripts/obsidian-frontmatter-takeover-e2e-lib.cjs` that
     byte-copies the live note's frontmatter shapes: flow-mapping categories, quoted
     digit hotkeys, bare dates, an offset timestamp, and `[[...]]` strings.
   - Assert in real Obsidian that `propertyCardFrontmatterCacheEvidence(source, cache)`
     is true.
   - If it is false, diagnose. Fix only a genuine normalization defect in the shared
     stable-value comparison, with a unit test proving other species are unaffected.
     **Never loosen the gate.**
   - If it can't be fixed, stop after landing the fixture. Record a
     `PROPOSED FOLLOW-UP:` on the phase bead, because the hide would never engage.
2. **`240q`, three surgical edits.**
   - `propertyCardFrontmatterMountPath` maps kind `bank` → `data-bank-setup-source`.
   - Add `"bank"` to the eligibility list.
   - Add `scheduleBankSetupRefresh` to the capability-change refresh list.
3. **Bank host.**
   - Bind/release the coordinator in `onSetup`/`onDisposeStart`
     (`propertyCardFrontmatterCoordinatorBind(plugin, "bank")`).
   - Acquire after a slot mount through `propertyCardFrontmatterOwnMounted` (Reading and
     LP). Acquire on the deferred path for widget fallbacks
     (`propertyCardFrontmatterOwnMountedDeferred`), mirroring `241v`.
   - `isCurrent` re-derives the bank card model from the current source.
   - Release on removal, path change, Source/lint branch, mobile, and dispose.
   - If any evidence fails, the native block returns with the per-row `bank` hide as the
     fallback.
   - Invalid YAML loses the source/cache proof and restores native rendering. A valid
     YAML mapping whose `bank:` settings fail bob validation keeps ownership: the
     diagnostic card and the complete `-` inventory remain. Never re-implement bob's
     validation in JavaScript.
4. **E2E.** Promote the `bank-native` case in
   `scripts/obsidian-frontmatter-takeover-e2e.cjs` to full bank takeover verification:
   - hidden in LP and Reading;
   - Add-file-property routes to the editor, and the File Properties sidebar stays
     native;
   - invalid YAML, cache skew, old bob/absent capability, mobile, embed, hover, canvas,
     and print all fail open;
   - Source restores;
   - bank joins the `inert` + `aria-hidden` + `focusin` guard matrix (no hidden tab
     stops, no `k`/`gg` traps);
   - screenshots in light/dark × wide/narrow. Run it per `docs/develop/validation.md`
     "Frontmatter takeover", and record the report path in the phase bead.
5. **Docs.** `docs/property-editor.md` (~337: drop "Bank retains its native Properties
   block"), `docs/develop/architecture.md` (~233: drop "bank does not"), and the bank
   card paragraph in `docs/bob-ledger-tools.md`.
6. **Memory** (bob-cli repo; only if `memory_glossary_cards` is true; use
   `/sase_memory_write`).
   - Glossary strand `property-card`: bank joins the title-owning, whole-frontmatter
     species, and "Bank keeps its native block" goes. Remove the badge-without-title
     sentence and add `_` to the key sentence.
   - Glossary strand `bank-card`: the card owns the note top, Setup folds by default
     with `_`, and `⊞ Setup` replaces `⊞ Edit`.
   - Then run `sase memory init`, `just docs-gen` (regenerates
     `docs/reference/glossary.md`), and `just docs-check` in bob-cli.
7. Bump, build, test, and `bob plugins sync`.

**Done when:** in desktop LP and Reading, `bank.md` shows only the title-slot card above
its prose, with no native Properties block. Every uncertainty restores the native block.

### Phase `fold-family`: `_` reaches record, area, and project folds

> [!decision] family_fold_look = compact Keep each card's existing chevron button and
> call `propertyCardDecorateFoldControl` on it (tooltip, `aria-keyshortcuts`, data
> attributes, per-mount `aria-controls` to a body or `hidden` shell, hover keycap when
> `keyLive`). Layout and pixels stay unchanged.

> [!decision] family_fold_look = row Replace the chevron with `propertyCardBuildFoldRow`
> under each title block, using the noun `Details` on every one of these cards. The
> summary comes from the card's existing collapsed text. Update screenshots and any e2e
> height assertions.

1. **Record** (`242j`, `242k`). Register the `record` fold kind. Resolve `collapsed`
   through `propertyCardFoldResolve` with `problems: ""` and the species `foldDefault`.
   Tooltip `Toggle record details (_)`.
2. **Area/project** (`249c`, `249d`). Register the `area` and `project` fold kinds, and
   resolve through the helper the same way. Tooltip `Toggle card details (_)`.
3. **Foot parity** (`240o`, `240r`). The foot disclosure gains the same hover keycap
   (`-`) when the adapter passes `keyLive`. Both disclosures then teach their keys
   identically.
4. **Tests.**
   - Extend the record-sheet, home-card DOM/host, and property-card foot unit tests to
     cover registration, the `_` gate acting on each kind, the attributes, and the
     keycap.
   - Day/period notes: `_` falls through, `canAct` is false.
   - E2E: add `_` round trips to `scripts/obsidian-record-card-e2e.cjs` and
     `scripts/obsidian-home-cards-e2e.cjs`.
5. **Docs.** `docs/record-card.md`, `docs/area-and-project-cards.md`, and the key-family
   section in `docs/bob-ledger-tools.md`. Then bump, build, test, and
   `bob plugins sync`.

On generated-file rebase conflicts with the bank phases, rebuild (`npm run build`)
rather than hand-merging `main.js`, and take the next manifest version.

**Done when:** `_` folds every card that has a fold and falls through on every card that
doesn't.

### Phase `bank-polish`: area strip, screenshot review, and passive-rendering proof

1. **Area strip on the bank card.** Reuse `homeCardChips` (authored area policies) and
   `homeCardReadyModel`. Render the strip only when it has at least one authored policy
   chip or the READY model is over cap. A calm READY count alone does not render,
   because the bank queue is the daily surface. On today's `bank.md` the row is absent,
   but nothing disappears once Bryan sets a policy through `+`.
2. **Passive-rendering proof.** Add a unit test that toggles bank, album, record, and
   area folds many times and forces repeated refreshes. It must count zero vault writes
   and zero new paint-triggered bob subprocesses (bank config refreshes stay
   subscription-driven).
3. **Screenshot review.** Capture bank, album, record, and area cards in light/dark ×
   wide/narrow (an existing preview harness such as `scripts/preview-property-cards.cjs`
   or the e2e screenshot paths). Fix any visual drift from the Design section: row
   alignment, chip placement, keycap fade, wrapping in narrow splits, and the collapsed
   bank height near a folded album. Attach the shots to the phase bead.
4. **Docs sweep.** Make the docs read consistently about the key family. If
   `memory_card_keys_record` is true, use `/sase_memory_write` to add a new strand to
   the bob-cli decisions web (keyword "Card Keys Show And Edit"). Its claim is: on every
   property card, the minus key shows/hides (`-` Properties drawer, `_` the card's own
   section) and the equals key edits (`==` species editor, `+` Property Editor), and
   bare keys act only through the guarded `248m` gate. Cite this epic's plan and the
   research report. Then run `sase memory init`.
5. Bump, build, test, and `bob plugins sync`.

**Done when:** the card family looks and behaves as one system in screenshots, and
folding is provably passive.

## Risks and how the plan contains them

- **YAML parity on `bank.md`**: this is the only real unknown. `bank-takeover` spikes it
  first and fails open.
- **Title-slot height changes**: the collapsed default keeps the common path short,
  nothing animates, and the slot's scroll anchor holds reading position.
- **Bare-key creep**: `_` is the fourth bare punctuation key. The narrow gate, the
  never-consume-without-a-fold rule, the command remap route, and a doc line stating the
  Vim trade contain it.
- **Dual identity**: `+` showing AREA SETTINGS on a BANK card is correct. Docs say so,
  and `type: [[area]]` stays.
- **Mobile**: the degrade line is unchanged: no `_`, no hide, no title ownership.
