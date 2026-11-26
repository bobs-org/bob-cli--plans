---
tier: epic
title: Property Card Cabinets
goal: 'Child areas, sub-projects, notebooks, albums, and album records open as collapsible
  cabinets at the bottom of their note''s Property Card. Each cabinet shows exactly
  the data its H2 section showed. Album Schema and bank Setup move into the same chrome
  as a restyled Properties cabinet, and `_` becomes a numbered, cancel-safe cabinet
  pick. The stored catalogs stay bob-owned Markdown, and nothing in bob-cli, the release
  train, or Bob Mac Capture changes. The work lands dark beside bob-cli-qj, bob-cli-qm,
  and bob-cli-qp and switches on only after the sibling go-lives it would race.

  '
decisions:
  golive_waits_for_qp:
    ask: Hold cabinets' go-live for bob-cli-qp.8 (Ref Cards) as well as bob-cli-qm.7?
    default: true
    why: qp.7/qp.8 assert family DOM and edit glossary:property-card; waiting avoids
      racing them
    answer: true
  memory_cabinet_term:
    ask: Add the glossary:cabinet strand at go-live?
    memory:
    - glossary:cabinet
    default: true
    requested: As a part of this change we should add an excellent new glossary term
      for "cabinet".
    answer: true
  memory_card_strands:
    ask: Update the property-card, area-card, and project-card strands for cabinets
      and the `_` pick?
    memory:
    - glossary:property-card
    - glossary:area-card
    - glossary:project-card
    default: true
    corrects: '`_` folds the card''s own section'
    answer: true
  memory_decision_record:
    ask: Add a decision record stating that cabinets absorb bob-owned catalogs at
      render time and never rewrite them?
    memory:
    - decisions:cabinets-absorb-catalogs-at-render-time
    default: false
    answer: true
phases:
- id: kit
  title: Cabinet gate, model, session state, provider registry, and the view collector
  depends_on: []
  size: medium
  description: 'kit: add the cabinet-owned `cabinetsPreview` gate. Add the frozen
    kind table with canonical ranks, roles, and defaults; bottom-up numbering; session
    state per (path, kind) with rename/delete carry; the content-provider registry;
    the view-scoped cabinet collector; a role-aware `_` fold facade; and one toggle
    router. Also add the go-live dependency edges onto the gating sibling go-lives
    and COORDINATION notes.'
- id: footer
  title: Walker-footer overlay push/pop API
  depends_on: []
  size: small
  description: 'footer: add a LIFO overlay API on the ledger walker-footer status
    item. Live walk painters defer while an overlay is up, and popping repaints the
    current state. A hidden footer unhides for the overlay''s lifetime, and a view-local
    fallback strip covers a hidden status bar. Both surfaces get aria-live, and the
    API never writes the vault or spawns bob.'
- id: absorption
  title: Catalog recognition, ownership claims, render-time hiding, and the body renderer
  depends_on: []
  size: medium
  description: 'absorption: spike Vim motion over hidden ranges in real Obsidian first.
    Then build a region.rs-mirroring recognizer and fingerprinted ownership claims,
    a Live Preview StateField hide (fallback ladder), the strict-inside reveal, Reading
    section tagging, a stand-down helper for inline painters, the navigation reveal
    hook, and a generic catalog body renderer.'
- id: chrome
  title: One cabinet chrome, delegation, the stack, and the CSS region
  depends_on:
  - kit
  size: medium
  description: 'chrome: build the shared cabinet header and drawer. Delegate fold
    rows (Schema, Setup, Outline) and the Properties foot to it under the gate, and
    assemble one ordered stack per card with Properties last. Mount content providers
    lazily, add the pick-badge painter, flip album Schema to closed-when-healthy,
    fix the bank foot keycap, and own the new Cabinets CSS region.'
- id: keys
  title: The `_` dispatch and numbered cabinet pick
  depends_on:
  - chrome
  - footer
  size: medium
  description: 'keys: route `_` to the details fold (zero extras), a direct toggle
    (one), or an armed pick (two or more). The pick uses a Scope plus a capture listener,
    and snapshots semantic kinds. It lights the on-card digits and the footer overlay.
    `__` toggles 1 and `-` toggles Properties; Esc, `q`, and Ctrl+[ cancel. Disarm
    triggers and palette parity are included.'
- id: host-cabinets
  title: Areas, Projects, Notebooks, and Albums cabinets on area and project cards
  depends_on:
  - chrome
  - absorption
  size: medium
  description: 'host-cabinets: register providers for the four owner catalogs. Each
    renders the stored callout through the body renderer and the existing painter,
    lifts title status into the header, and writes honest summaries. Missing and damaged
    states get an explicit Sync. Each claims absorption, stands inline painters and
    anchors down, and registers navigation reveal.'
- id: records-cabinet
  title: The album Records cabinet
  depends_on:
  - chrome
  - absorption
  size: medium
  description: 'records-cabinet: register the open-by-default Records provider for
    album cards. It covers the stored index and tally, group folds, the empty-album
    teaching line, and the subscriptions decoration, with absorption claims and painter
    stand-down. The phase also retargets the albums-catalog group jump into the cabinet
    and measures a 106-record album.'
- id: acceptance
  title: Real-Obsidian acceptance and screenshot review with the gate open
  depends_on:
  - keys
  - host-cabinets
  - records-cabinet
  size: medium
  description: 'acceptance: build a sibling scratch-vault e2e runner. It opens the
    cabinets gate and every sibling species gate, and covers every species in Live
    Preview and Reading, the full keyboard matrix, fail-open contexts, and the passive
    spy. It also checks content parity and screenshots light/dark, narrow/wide, reduced-motion,
    and forced-colors before a GO/NO-GO.'
- id: go-live
  title: Switch cabinets on, docs, approved memory, and clean deploy
  depends_on:
  - acceptance
  size: medium
  description: 'go-live: once the gating sibling go-lives have closed, delete the
    gate and update the docs in both repos. Write the new cabinets reference page
    and the approved memory (glossary:cabinet and the selected card strands and record).
    Then run the family passive proof, `bob plugins sync`, and a post-switch smoke
    that includes the prose Outline cabinet.'
proposed_by: bbugyi200.apollo.research.6m.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 16:08:58
status: wip
bead_id: bob-cli-qq
---

- **PROMPT:** [prompts/202611/property_card_cabinets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_card_cabinets.md)
- **BEAD:** [bob-cli-qq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-qq/README.md)

# Property Card Cabinets

## Why

Area, project, and album notes carry bob-owned catalogs under H2 sections: `## Areas`,
`## Projects`, `## Notebooks`, `## Albums`, and `## Records`. They push the note's own
work (`## Tasks`, prose) down the page. Their status is scattered, and each looks
different from the Property Card above it. Bryan wants these catalogs in **cabinets**:
named compartments at the bottom of the note's Property Card. Each cabinet holds the
same data its H2 section held and folds independently.

The card already has two such compartments in two styles. One is the Properties foot,
toggled with `-`, which becomes the **property cabinet**. The other is the full-width
fold row behind album **Schema** and bank **Setup**, toggled with `_`. This epic gives
both one chrome, adds the content cabinets above them, and turns `_` into a precise
numbered pick. Today `_` takes whichever fold it finds first.

This plan implements the consolidated recommendation in
`research:202611/property_card_cabinets/property_card_cabinets.md`. Bryan accepted
requirement adjustments A1–A12 from that report. They are restated in
[Accepted requirements](#accepted-requirements), and where this plan refines them, the
change is marked **Lead** below.

## Verified current state

All facts below were re-verified at planning time against bob-cli `master` (4168e201)
and bob-plugins `master` (6045168f). Fragment paths are relative to
`plugins/bob-ledger-tools/src/` in bob-plugins.

- **Folds.** `240s-plugin-property-card-fold.js` owns fold state helpers
  (`propertyCardFoldChoice`/`Resolve`/`Restamp`), the fold-kind registry
  (`propertyCardRegisterFoldKind`), and the full-width row (`propertyCardBuildFoldRow`).
  `propertyCardFoldTarget(view)` returns the **first** visible `[data-card-fold]` node
  for the view's path, which means `_` is DOM-order dependent once a card has two folds.
- **Fold-row users.** Album Schema (`241x-plugin-album-schema-dom.js:139`), bank Setup
  (`248b-plugin-bank-setup-dom.js:576`), and prose Outline
  (`249m-plugin-prose-note-card-dom.js:86`, kind `prose`) all call
  `propertyCardBuildFoldRow`. `249p` is qm.5's six-line Outline stub, now being written.
- **Compact details chevrons.** Area and project cards use
  `propertyCardDecorateFoldControl(…, { compact: true })`
  (`249c-plugin-home-card-dom.js:147`). Notebook, record, and ref cards use the same
  compact form. This is the card's own details fold, not a compartment.
- **Species fold defaults** live in `240f-plugin-property-card-species.js`. They are
  `album` `foldDefault: true` (Schema open), `bank` false, `prose` false, and
  `area`/`project`/`notebook`/`record` true (details expanded).
- **Properties foot.** `propertyCardFootBuildEl` (`240o:541`) builds the
  `▸ Properties · N` disclosure. Its lint chip and created stamp are siblings of the
  disclosure, its keycap sits outside the row, and its drawer animates with
  `grid-template-rows` (styles.css ≈11911). Every card appends it through
  `propertyCardFootAdapterAppend` (`240r:310`): home (`249e`), album (`241va`), bank
  (`248ca`), notebook (`249j`), prose (`249o`), and intention (`296da`). The one
  exception is the record sheet, which calls `propertyCardFootBuildEl` directly
  (`242kc:205`). Fold state lives in `240p`, which has rename carry-over and delete
  pruning (`propertyCardFootRenamePath`/`ReleasePath`). The bank foot never passes
  `keyLive`, so it lacks the `-` keycap.
- **Keys.** `248m-plugin-schema-double-equals.js` `handleSchemaDoubleEqualsKeydown`
  routes `=`/`-`/`_`/`+` through `propertyCardBareKeyGate` (Live Preview, Vim normal, no
  operator, count, or register, not composing, content target). `_` calls
  `propertyCardToggleDetails`. The palette commands `toggle-card-properties` and
  `toggle-card-details` are at `248m:445,461`. `bobCardCloseKey`
  (`241xc-plugin-card-close-keys.js`) classifies Esc, Ctrl+[, and `q`.
- **Numbered-prefix precedent.** The Ctrl+W window prefix in bob-navigation-hotkeys
  (`687-plugin-window-prefix.js`, model `266`, hint view `473`) pushes a catch-all
  Obsidian `Scope` and adds a capture-phase `keydown` listener. It disarms on
  blur/leaf/layout change, has no timeout, and uses the same three cancel keys.
- **Catalog painters.** Areas (`241aj`), projects (`241ae`), notebooks (`241d`), albums
  (`241q`, DOM `241p`), and records (`241t`, DOM `241s`) each register a Reading
  post-processor plus a Live Preview view plugin. They decorate Obsidian's rendered
  `[data-callout=…]` in place, and the LP plugins scan **all** of `view.dom`. Missing
  catalogs get standalone anchors at the heading or H1 (for example, `241ae:141–160`).
  The albums painter writes status into `.callout-title` (`241p:124`). Subscriptions
  decoration is `241hf`; the album group jump is `apiAlbumOpenGroup` (`241q`).
- **Ownership of the bytes.** bob-cli `src/native/host_catalog/region.rs` says that "Bob
  owns exactly the `> [!{callout}]` callout block; the position of the block, its
  heading, and every other byte belong to the note's author." Specs `AREAS`, `PROJECTS`,
  `NOTEBOOKS`, and `ALBUMS` make up `GENERATED_CATALOGS`. Records use
  `records_spec(head_title)` in `src/native/albums/index.rs` (`[!records] <tally>` under
  `## Records`). The HTML markers are legacy, notebooks-only. Other readers of those
  bytes include `SUBOPEN(N)` (`src/native/projects/surface.rs`), the capture
  generated-section refusals (`src/native/capture/sections.rs`), the album group jump,
  mobile, git, and agents.
- **Notebook notes never host `[!notebooks]`.** Only area and project notes do (see
  `glossary:notebook`). Records exist only on album notes. The root `area.md` is untyped
  and has no Property Card.
- **Walker footer.** One ledger status-bar item, `.bob-walker-footer`, created in
  `240-plugin-freshness-status.js` `setupFreshnessStatusBar`. It paints the `s`, `w`,
  `n`, and `b` panels routed by `122-walker-footer-host.js` and has no overlay concept.
- **Gates.** qp owns `refCardPreview` (`240u`), and qm owns `cardPreviewKinds`
  (`240t:214`, which qm.7 deletes). Cabinets reuse neither. Fragment slots `240u` and
  `240v` are now taken, so the research's `240u–240y` slot map is stale. This epic uses
  the free `240w*` family.
- **Real-Obsidian acceptance is isolated.** Every runner makes a unique scratch Git
  vault with an owned profile, process, and debug port, and never opens the live vault
  (bob-plugins `docs/develop/validation.md`).
- **Sibling epics, now.** bob-cli-qi is **closed** (all six phases). bob-cli-qj.3–5,
  bob-cli-qm.5–7, and bob-cli-qp.1 and qp.4–8 are in progress.

## Governing decisions

- `decisions:notebook-routes-write-notes`: bob owns each `## Notebooks` catalog's
  content and Bryan owns its position. The record rejected indexes "invisible to git,
  agents, and mobile". Cabinets therefore never move, rewrite, or remove stored bytes.
  They change only where the rendered catalog appears.
- `decisions:q-and-ctrl-bracket-mirror-esc`: the armed pick cancels with Esc, `q`, and
  Ctrl+`[` through one dispatch, and `q` is never a cabinet number or badge.
- `decisions:bob-cli-releases-ride-a-daily-train`: no bob-cli code or JSON contract
  changes, so nothing rides the train. A phase that finds it needs a bob-cli change
  stops and records a `PROPOSED FOLLOW-UP:` note.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture is untouched.
- `decisions:bank-hub-is-path-canonical-never-area`: the bank hub card gets Setup and
  Properties cabinets only, never Area catalogs.
- `decisions:prose-notes-are-owned-prose`: the Outline cabinet is the prose card's
  existing fold (both prose spellings are honored upstream). Cabinets never write prose.
- `decisions:decision-rules-are-read-at-planning-time`: this plan read the roster and
  cites the records above. No decision record governs card folds or `_` today.

## Accepted requirements

These come from the research's A1–A12, which Bryan accepted.

- **A1. Catalogs stay in Markdown, and cabinets absorb them at render time.** The
  callout is hidden in Live Preview and Reading only while a working cabinet stands in
  for it in that view. Its H2 is hidden only when the section holds nothing but the
  callout and whitespace. Source, mobile, embeds, hover, canvas, print, plugin-off,
  gate-closed, and any uncertain state show the stored section natively.
- **A2.** The pick cancels with Esc, Ctrl+`[`, **and** `q`.
- **A3/A4.** Hints show in two places: digit badges on the cabinet headers, and a footer
  overlay pushed and popped with an explicit restore. A view-local strip stands in when
  the status bar is hidden.
- **A5.** Bank keeps its noun "Setup". Prose Outline joins the stack.
- **A6.** Compact details chevrons are not cabinets and get no number. `_` falls back to
  them only when no cabinet sits above Properties. Otherwise they stay click-only.
- **A7.** While armed, `_` again toggles cabinet 1, and `-` toggles Properties and
  disarms.
- **A8.** The stack order is fixed and canonical, not taken from the note's H2 order:
  content first (Areas → Projects → Notebooks → Albums, or Records, or Outline), then
  config (Schema or Setup), then Properties last. Empty catalogs get no cabinet.
  Numbering counts contiguously upward from `1` just above Properties.
- **A9.** Defaults are set per kind (see [Defaults](#defaults-and-state)): Records open,
  the other content cabinets closed, and album Schema **closed when healthy**.
- **A10.** In docs and memory, "drawer" means only the Task Drawer, and the foot becomes
  the "Properties cabinet" (subject to the memory decisions). Code identifiers do not
  churn.
- **A11.** Untyped hosts (root `area.md`) keep their native catalogs.
- **A12.** Open/closed state is session-only per (normalized path, kind), with the
  foot's rename carry-over and delete pruning. It is never written to frontmatter.

## Design

### Anatomy

A **cabinet** is one header row over a body that opens in place. A card's cabinets live
in one **stack** at the bottom of the card. The **property cabinet** (Properties) is
always last. The **extra cabinets** sit above it in canonical order.

```text
  AREA · cash                                                    ▾
  Money in, money out.
  Ready 4/5 · Today 2
  ─────────────────────────────────────────────────────────────────
  ▸ Areas        3 · 2 nested                                   _4
  ▸ Projects     4 open · 1 blocked · 12 closed     ⚠ stale  ↻ Sync
  ▸ Notebooks    3
  ▸ Albums       2 · 41 records
  ▸ Properties   7                              created Nov 12    -
```

The `_4` and `-` keycaps appear on hover or focus (shown on two rows for illustration).
While the pick is armed, every extra shows its digit lit:

```text
  ▸ Areas        3 · 2 nested                                  [4]
  ▸ Projects     4 open · 1 blocked · 12 closed  ⚠ stale ↻ Sync [3]
  ▸ Notebooks    3                                             [2]
  ▸ Albums       2 · 41 records                                [1]
  ▸ Properties   7                              created Nov 12 [-]

  status bar:  ▤ Cabinets   1 Albums ▸ · 2 Notebooks ▸ · 3 Projects ▸ · 4 Areas ▸   - Properties   esc
```

An album at rest:

```text
  ALBUM · Places
  Restaurants — places worth returning to.
  ▾ Records      24 restaurants · by Status
      [stored index: groups, rows, facets, group folds — exactly as today]
  ▸ Schema       6 fields · walks by Status
  ▸ Properties   9                              created Nov 12
```

The card's face is not a cabinet: tile, title, crumbs, summary, readouts, and the
compact details chevron. Cabinets are independent disclosures. They are not an accordion
and not tabs, so several can be open at once.

### Kinds, order, and numbering

| Kind         | Noun       | Role       | Rank | Appears on                                      | Summary grammar                                                         |
| ------------ | ---------- | ---------- | ---- | ----------------------------------------------- | ----------------------------------------------------------------------- |
| `areas`      | Areas      | content    | 10   | area cards with an `[!areas]` region            | `N` direct, plus `· M nested` when deeper                               |
| `projects`   | Projects   | content    | 20   | area/project cards with `[!projects]`           | report tally `4 open · 1 blocked · 12 closed`, else `N`                 |
| `notebooks`  | Notebooks  | content    | 30   | area/project cards with `[!notebooks]`          | `N`                                                                     |
| `albums`     | Albums     | content    | 40   | area/project cards with `[!albums]`             | `N · M records` (M from the report, else omitted)                       |
| `records`    | Records    | content    | 50   | album cards (always, including the empty album) | the callout head tally plus grouping, e.g. `24 restaurants · by Status` |
| `outline`    | Outline    | content    | 60   | prose cards (fold kind `prose`)                 | the existing Outline summary                                            |
| `schema`     | Schema     | config     | 70   | album cards (fold kind `album`)                 | the existing Schema summary                                             |
| `setup`      | Setup      | config     | 80   | the bank card (fold kind `bank`)                | the existing Setup summary                                              |
| `properties` | Properties | properties | 100  | every card with a foot                          | `N`, with the created stamp as meta                                     |

Each header reads `Noun · count · qualifiers`. A missing report means the enrichment is
absent, never zero. Numbers are assigned from the bottom extra upward and are contiguous
over the cabinets present: on an album, `1` = Schema and `2` = Records; on a full area
host, `1` Albums, `2` Notebooks, `3` Projects, `4` Areas. At most eight extras can exist
today, so digits 1–9 always suffice.

### One chrome (every cabinet, Properties included)

The header is a flex row, `.bob-cabinet-head`. Its parts are:

- **The disclosure button**, `.bob-cabinet-toggle`, a real `<button>` with
  `aria-expanded`/`aria-controls`. It holds the chevron, the noun, and the summary,
  takes the free space, and truncates the summary with an ellipsis.
- **The trailing slot**, `.bob-cabinet-trail`. In order, it holds the faint meta (the
  created stamp), the attention chip, any action buttons (Sync, the Properties lint
  jump), and the keycap slot. Trailing controls are sibling buttons, never nested inside
  the disclosure.
- **Type.** `var(--font-ui-small)` and tabular numerals. The noun is `--text-normal` at
  weight 600, the summary `--text-muted`, and the meta `--text-faint`. Separators are
  `·`.
- **Geometry.** Header min-height 1.85em, padding 0.2em 0.35em, gap 0.4em, and radius
  `var(--radius-s)`. The body is indented 1.4em, so content aligns under the noun. Body
  padding is 0.25em top and 0.5em bottom.
- **Color.** The chevron takes the card accent (`--bob-property-card-accent`). Hover is
  a 6% accent wash, the same as today's fold row. Focus-visible is a 2px
  `--interactive-accent` outline with a 2px offset. There are no new palette values and
  no nested boxes.
- **The stack.** One hairline (`--background-modifier-border`) runs above the first
  cabinet, with hairlines between fronts at 60% opacity.
- **Motion.** The chevron rotates 90° in 160 ms, the drawer opens via
  `grid-template-rows: 0fr → 1fr` in 160 ms, and badges fade in 120 ms. All motion is
  off under `prefers-reduced-motion`.
- **Keycap slot.** It is inline and fixed at min-width 1.6em, so arming never reflows
  the row. This also fixes the foot's out-of-row keycap. At rest it is 0 opacity, and
  0.7 on hover or focus when `keyLive` (desktop, Live Preview, Vim). It shows `-` on
  Properties, `_` on the sole extra, and `_N` on each extra when there are two or more
  (**Lead**: the hover teaches the chord).
- **The armed state.** The stack gets `data-cabinet-pick="armed"` and a 1px inset ring
  at 35% accent. Each extra's keycap shows its digit at full opacity, with an 18% accent
  background, a 55% accent border, `--text-normal`, and weight 600. Properties shows `-`
  outlined. Badges show while armed even when not `keyLive`, for example after a palette
  arm.
- **Content bodies.** A content cabinet's body neutralizes the inner callout's chrome
  (no background, border, padding, or title row), so it reads as rows, not a box in a
  box. Everything the inline painter put in the callout title moves to the header's
  trailing slot.
- **Width.** A container query on the card handles narrow layouts. Below 30em the
  summary truncates first. Below 24em the meta hides and the attention chip collapses to
  its icon, with the word in the tooltip. Digits stay visible while armed.
- **Forced colors.** Text is CanvasText, badges get a ButtonText border, and focus uses
  Highlight.
- **No nested scrollbars.** Large bodies flow in the document, because nested scrollers
  trap Vim `j`/`k` and wheel scrolling. Bulk is handled by interior folds (record
  groups, `▸ Closed (N)`) and lazy bodies.

Cabinets come from **delegation**, so in-flight cards need no edits. Under the open
gate, `propertyCardBuildFoldRow` and `propertyCardFootBuildEl` build this chrome, and
fold kinds map to cabinet kinds (`album → schema`, `bank → setup`, `prose → outline`).
Schema, Setup, Outline, and Properties become cabinets **without** edits to the
fragments that own them. The prose card's Outline (qm.5, in flight) therefore needs no
adapter. **Lead**: this removes the research's qm.5 sync point.

### The `_` keymap

| Context / input                                                                                                                    | Result                                                                                                                                          |
| ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `-` anywhere on a card                                                                                                             | Toggle Properties. An armed pick is dismissed first.                                                                                            |
| `_` arming gate                                                                                                                    | Unchanged: today's `propertyCardBareKeyGate`. On failure, the key falls through to Vim's `_` motion.                                            |
| `_`, zero extras                                                                                                                   | The details fold (role `details`), else Vim `_`. Covers record, notebook, ref, leaf area/project, and day–year cards.                           |
| `_`, one extra                                                                                                                     | Toggle it at once. Bank Setup, prose Outline, and a single-catalog area all keep their one-key habit.                                           |
| `_`, two or more extras                                                                                                            | Arm the pick and toggle nothing yet. Snapshot the extras bottom-up as semantic kinds, never DOM positions.                                      |
| A valid digit while armed                                                                                                          | Toggle that cabinet and disarm. A cabinet that opens with its header offscreen scrolls into view (`block: "nearest"`). Closing never scrolls.   |
| `_` while armed                                                                                                                    | Toggle cabinet 1 (`__`). On an album that is Schema, the old habit at two keys.                                                                 |
| `-` while armed                                                                                                                    | Toggle Properties and disarm.                                                                                                                   |
| An out-of-range digit (including `0`)                                                                                              | Swallowed. Stay armed, and nudge the footer legend with "Choose 1–N". A digit never leaks into a Vim count.                                     |
| Esc, Ctrl+`[`, `q`                                                                                                                 | Cancel through `bobCardCloseKey` and swallow the key.                                                                                           |
| Meta/Alt/Ctrl combos (except Ctrl+`[`)                                                                                             | Cancel and pass through.                                                                                                                        |
| Any other key                                                                                                                      | Cancel and let the event continue. In Vim normal mode, a passed-through key cannot type.                                                        |
| Blur, visibility change, outside mousedown, active-leaf change, layout change, file open, mode change, card unmount, plugin unload | Disarm silently and release all listeners, badges, and overlays.                                                                                |
| A repaint mid-pick                                                                                                                 | The snapshot's numbering holds for kinds still present. If the picked kind vanished, cancel with a Notice: "Cabinets changed — press _ again". |
| Timeout                                                                                                                            | None, matching Ctrl+W and Vim operator-pending.                                                                                                 |

Reading view and non-Vim users get clickable headers, clickable footer chips, and the
`toggle-card-details` palette command. That command runs through the same dispatch, so
with two or more extras it arms the pick, and digits work in any mode while armed.

### Footer overlay

```text
▤ Cabinets   1 Albums ▸ · 2 Notebooks ▸ · 3 Projects ▾ · 4 Areas ▸    - Properties    esc
```

- **Order and chips.** Chips run `1..N` left to right. Each shows its keycap (the walker
  footer's answer-keycap styling, tinted with the card accent), its noun, and its
  current `▸`/`▾`, so the user sees what a digit will do. Chips are clickable.
- **Timing.** The overlay appears **150 ms** after arming, while the card badges light
  immediately. A fast `_1` never flashes the footer. **Lead**: this settles the
  research's optional delay.
- **Narrow bars.** On a narrow bar, `esc` sheds first, then nouns truncate. Digits never
  shed.
- **Announcements.** An `aria-live="polite"` region announces "Cabinets: 1 Albums, 2
  Notebooks, 3 Projects, 4 Areas. Press a number, or Escape to cancel."

### Defaults and state

| Cabinet                            | Fresh-session default                          | Resurfacing                                                                                                                       |
| ---------------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Areas, Projects, Notebooks, Albums | Closed                                         | Never auto-opens. Stale, missing, or damaged shows `⚠` plus an explicit `↻ Sync` on the closed header.                           |
| Records                            | Open (including the empty-album teaching line) | n/a                                                                                                                               |
| Schema                             | **Closed when healthy** (today: open)          | Opens once per new problem signature through `propertyCardFoldResolve`. An explicit close for the current signature is respected. |
| Setup, Outline                     | Unchanged (closed when healthy)                | Existing finding-driven resurfacing.                                                                                              |
| Properties                         | Closed (unchanged)                             | The `⚠ N` lint chip stays in the header.                                                                                         |

State is session-only per (normalized path, kind) and shared across panes showing the
same note, like the foot today. Per-mount DOM ids stay unique. Cleanup is bound to the
mount generation, so a stale renderer cannot unregister its replacement. Interior folds
(record groups, `▸ Closed (N)`, the projects tree) keep their existing per-pane maps.

### Absorption: hiding the stored sections

- **Recognize.** A pure function mirrors `region.rs` and `records_spec` exactly:
  case-insensitive callout head, optional fold sign, frontmatter and fences ignored, and
  the callout is the head line plus contiguous `>` lines. Only a region bob itself would
  recognize is adopted, so a heading whose text differs is never adopted. It returns
  `{kind, callout, heading|null, fingerprint, damage|null}`. `heading` is set only when
  the section (up to the next heading of level ≤2, or EOF) holds nothing else.
  Malformed, duplicate, or ambiguous regions are never hidden.
- **Claim.** A cabinet claims a region for one view only once its header is mounted and
  its content model is validated. The claim carries file/view identity, kind,
  fingerprint, mount generation, `reveal()`, and `release()`. **Invariant: a valid
  stored catalog is never hidden without a working cabinet for it in that view.** If a
  lazy body later fails to build, the claim is released at once, and the cabinet says
  "Couldn't render — showing the note's section".
- **Live Preview.** One StateField supplies the replace decorations directly, not per
  viewport. Claim changes dispatch a refresh effect. The hidden range covers whole
  lines: the callout, the heading when hideable, and one following blank line, so no
  empty gap remains.
- **Reveal and navigation.** Raw Markdown is revealed only when the selection lands
  **strictly inside** a hidden range (a Find hit, for example). Boundary contact never
  reveals. Heading navigation (`[[note#Projects]]`, the Outline pane, `#Records` links)
  calls the claim's `reveal()`, which opens the cabinet and scrolls its header into
  view. No motion may ever leave an invisible cursor.
- **Reading view.** Section elements are tagged source-aware, the way title ownership
  (`240i`) does, and re-scanned when claims change.
- **Painter stand-down.** Inline painters skip absorbed regions and anything inside
  `.bob-property-card-cabinets` (a cabinet paints its own body). Their missing-catalog
  anchors stand down wherever a cabinet for that kind is mounted.
- **Fail open everywhere.** Source mode, embeds, hover, canvas, print, mobile, a failed
  mount, stale evidence, malformed or duplicate regions, the closed gate, and plugin
  unload all show the stored section natively in the same frame.

### Special cases

- **Album group jump** (`apiAlbumOpenGroup`). It opens the target album, opens its
  Records cabinet, expands the group, and scrolls to it.
- **`subscriptions.md`.** Its Records body runs the `241hf` decoration: category
  headings, `$X/mo`, and `next ≈` chips.
- **Records order.** The Records index order is not the walk order. The stored grouping,
  labels, facets, and tally are kept verbatim, and nothing is rebuilt from `walk_by`.
- **Notebook cards** keep qm.4's child-notebook readout chips. Notebooks never host
  `[!notebooks]`, so a notebook card has zero extras.
- **Hosts without a card.** The root `area.md` and other untyped hosts keep native
  catalogs (A11).
- **Large catalogs** are perf fixtures. A 64-line Projects callout, a 106-record album,
  and a host with three catalogs mirror the largest live-vault shapes.

## Parallel delivery with bob-cli-qj, bob-cli-qm, and bob-cli-qp

**Coordination is needed, but only a little.** Cabinets restyle chrome that qm.6 and
qp.7 are asserting and screenshotting right now. They also edit memory and docs that
qm.7 and qp.8 own. Nothing else overlaps. The answer is a dark build, not a wait.

| Epic | Seam                                                                                                                                                                                        | Handling                                                                                                                                                                                        |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| qi   | None. It is closed.                                                                                                                                                                         | —                                                                                                                                                                                               |
| qj   | None. Its work is bob-cli Rust, Bob Prose, and the vault cutover, and cabinets never touch prose. Keeping the stored sections also leaves qj.5's extract/inline classification undisturbed. | —                                                                                                                                                                                               |
| qm   | qm.5 is filling `249p`/`249m` (Outline). qm.6 asserts and screenshots the prose and notebook cards, and qm.7 edits `glossary:property-card` and family docs.                                | No cabinet phase edits `249*` prose or notebook fragments. Outline becomes a cabinet by delegation. The closed `cabinetsPreview` gate keeps their DOM byte-identical. `go-live` waits for qm.7. |
| qp   | qp.4–6 build ref fragments (`249q–249v`). qp.7 runs acceptance, and qp.8 edits `glossary:property-card` and docs.                                                                           | Same: no ref fragment edits and a closed gate. `go-live` waits for qp.8 by default (`golive_waits_for_qp`).                                                                                     |

The working rules:

1. **Cabinet-owned gate.** `cabinetsPreview: true` in bob-ledger-tools plugin data, read
   once at load into a module flag before any card paints. With it absent, rendering and
   behavior are exactly today's. Every existing suite runs unchanged and green, and a
   parity test proves no `bob-cabinet*` class or attribute appears. It never reads
   `cardPreviewKinds` or `refCardPreview`.
2. **New fragments for new machinery** (`240w*`, `241al*`, `241tb*`, `122a*`), with
   **additive, gate-guarded hunks only** in shared files: `240s`, `240o`, `240p`,
   `240r`, `248m`, the footer paint path, the catalog painters, `styles.css` (a new
   `Cabinets` region at the end), `fragments.json`, `package.json`, and `374*` exports.
3. **External gate on go-live only.** `kit` adds bead dependency edges from this epic's
   `go-live` phase onto the gating sibling go-lives, and `go-live` step 0 re-verifies
   them, so the gate holds even if the scheduler ignores cross-epic edges.

   > [!decision] golive_waits_for_qp The gating beads are `bob-cli-qm.7` **and**
   > `bob-cli-qp.8`.

   > [!decision] golive_waits_for_qp = no The gating bead is `bob-cli-qm.7` only.
   > `go-live` then also adds a `COORDINATION:` note on `bob-cli-qp.8` saying that
   > `glossary:property-card`, the family docs, and the Properties chrome changed under
   > it. qp.8 already re-reads the strand and docs before editing.

4. **No calendar coupling for acceptance** (**Lead**: this corrects the research).
   Real-Obsidian runs use isolated scratch vaults, profiles, processes, and ports, so
   cabinets acceptance may run concurrently with qm.6/qp.7.
5. **Deploys** follow the `bob plugins sync` refusal protocol in bob-plugins
   `AGENTS.md`. With the gate closed, every intermediate deploy is invisible in the live
   vault.

## Rules every phase follows

- **Repos.**
  - All code lives in the linked bob-plugins repo. Open it with `/sase_repo`
    (`sase repo open bob-plugins -r "<why>"`), use only the printed path, and read its
    `AGENTS.md` first.
  - bob-cli is edited only by `go-live` (memory and generated docs).
  - Before any vault read, read the `obsidian.md` reference memory. Never open the live
    vault in an automated Obsidian run.
- **Fragments.**
  - Edit `src/` fragments only, keep each at or below 1000 lines, run `npm run build`,
    and never hand-edit `main.js`.
  - Prefixes are `propertyCardCabinet*` in JS and `.bob-cabinet*` /
    `.bob-property-card-cabinets` in CSS.
  - Add test exports following the `374*` pattern.
- **Rebase discipline.** On a conflict in `main.js`, `styles.css`, `fragments.json`,
  `package.json`, or a manifest version, take upstream, re-apply your additive lines,
  rebuild, and re-bump. Never hand-merge a bundle.
- **Per plugin change.**
  1. Bump the manifest per repo convention.
  2. Run `npm run build`, `npm run build:check`, the focused suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`, per the refusal protocol.

  Pre-existing `npm test` failures that reproduce on clean master are recorded, not
  fixed.

- **Passive invariant.** Painting, toggling, arming, picking, cancelling, and the footer
  overlay never write the vault or spawn bob. Only an explicit `↻ Sync` click runs bob's
  existing writer. Every phase that paints adds spy tests that expect zero writes and
  zero spawns.
- **Evidence.** Unit suites go in `scripts/test-ledger-tools-cabinets-*.cjs` against
  shared fixtures. Real-app evidence is retained with `/sase_artifact` against the phase
  bead.
- **Beads.** Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on
  your phase bead instead. Run `sase bead epic-symbols <phase>` before closing.
- **Long work.** Use `/sase_monitor` for builds, real-app runs, and gate waits, and
  finish through `/sase_final`.

## Phase `kit`

1. **Gate (`240w-plugin-cabinet-gate.js`).** Model it on `240u`: the key is
   `cabinetsPreview`, it is read once at load through the ledger-tools plugin-data
   loader, and it is remembered in a module flag `propertyCardCabinetsLive`. Load it in
   `onload` before any card host registers. Exports: `propertyCardCabinetsGateLive()`
   and a test setter. It is pure, never throws, and does no DOM work.
2. **Model (`240wa-plugin-cabinet-model.js`).**
   - A frozen `PROPERTY_CARD_CABINET_KINDS` table per
     [Kinds](#kinds-order-and-numbering), with `noun`, `role`, `rank`, and
     `defaultOpen`. `defaultOpen` is `true` for records, `"healthy-closed"` for
     schema/setup/outline, and `false` otherwise.
   - `propertyCardCabinetKindForFold(foldKind)` (`album → schema`, `bank → setup`,
     `prose → outline`, otherwise null).
   - `propertyCardCabinetOrder(kinds)` (canonical sort).
   - `propertyCardCabinetNumbering(extras)`, which is bottom-up, contiguous, capped at
     9, and never yields `q`.
3. **State (`240wb-plugin-cabinet-state.js`).** Session maps for content kinds keyed by
   (normalized path, kind):
   - `propertyCardCabinetIsOpen(path, kind, {problems})` and `…SetOpen` resolve through
     `propertyCardFoldChoice`/`Resolve`/`Restamp` and the kind default. Add
     `propertyCardCabinetDefaultOpen(kind, {problems})`.
   - Rename carry-over and delete pruning are wired beside `240p`'s, through the
     existing vault event binds in `propertyCardFootAdapterBind`.
   - Schema, Setup, Outline, and Properties keep their **existing** state owners. The
     kit only reads them through their fold registration or the foot state.
4. **Provider registry.** `propertyCardRegisterCabinetProvider(plugin, kind, provider)`
   returns an unregister function bound to the generation. The provider contract:
   - `applies({path, species, source, frontmatter}) → {summary, attention?, actions?} | null`,
     which must be synchronous over confirmed source and cached reports.
   - `buildBody({doc, path, view, component, container}) → Promise|void`, which is lazy
     and called on first open.
   - `onOpen` and `onClose`, both optional.
   - `paintKey(path)`, a string that drives body re-use.

   Ship a fake provider in tests only.

5. **Collector and facade.**
   - `propertyCardCabinetsInView(view)` returns
     `{path, extras: [{kind, number, open, element}], properties: {open, element}|null, details: target|null}`.
     It reads the visible `.bob-property-card-cabinets` stack for the view's path (same
     scoping and visibility rules as `propertyCardFoldTarget`) and validates DOM order
     against canonical ranks.
   - Under the open gate, `propertyCardDecorateFoldControl` stamps
     `data-card-fold-role="details"` on compact controls and `"cabinet"` on full-width
     rows, and `propertyCardFoldTarget(view)` returns **details-role** targets only.
     With the gate closed, both behave exactly as today.
6. **Toggle router.** `propertyCardCabinetToggle(view, kind)` routes as follows:
   - Properties goes to `propertyCardToggleProperties`.
   - Schema, Setup, and Outline go to their fold registration's `toggle(path, view)`,
     with the pending-reveal bookkeeping already in `240s`.
   - Content kinds go to the state map plus a repaint request for the mount.

   It returns false on a miss and never throws.

7. **External edges and notes.**
   - `sase bead read` this epic, then run
     `sase bead dep add <this epic's go-live phase> bob-cli-qm.7`, plus `… bob-cli-qp.8`
     when `golive_waits_for_qp` is yes. If a command refuses, record the refusal in a
     note; `go-live` step 0 still gates.
   - Add `COORDINATION:` notes with `sase bead note` on `bob-cli-qm.6`, `bob-cli-qm.7`,
     `bob-cli-qp.7`, and `bob-cli-qp.8`. They name the `240w*` gate and the guarded
     hunks in `240s`/`240o`/`240r`/`248m`/`styles.css`, state that rendering is
     unchanged while `cabinetsPreview` is absent, and ask that go-live cleanups keep
     those hunks.
8. **Tests.**
   - Closed-gate parity: every species builder's DOM contains no `bob-cabinet*` or
     `data-card-fold-role`, and `propertyCardFoldTarget` is unchanged.
   - Numbering and ordering tables, plus state resolve, restamp, rename, and delete.
   - Generation-bound provider unregister, and collector filtering (path, visibility,
     disabled, other panes).

## Phase `footer`

1. **API (`122a-walker-footer-overlay.js`).**
   `walkerFooterPushOverlay(plugin, spec) → {update(spec), pop()}`. The spec carries
   `id`, `label`, `icon` (lucide id with a glyph fallback), `accent`,
   `chips: [{key, text, state, onActivate}]`, `trailing`, `announce`, `nudge`, `view`,
   and `delayMs`. Overlays are LIFO, and only the top one paints.
2. **Yield and restore.**
   - While any overlay is live, every regular footer paint (s/w/n/b and legacy review)
     records "dirty" instead of painting. Hook the existing paint entry, not each
     painter.
   - On the last pop, repaint the current state through the normal refresh path. Never
     restore stashed HTML.
   - Unhide `.bob-freshness-hidden` for the overlay's lifetime and restore it on pop.
     Unload pops everything.
3. **Fallback strip.** When the status-bar item is not visible (a zero rect or a hidden
   ancestor), render the same chip DOM as a slim strip pinned to the bottom of
   `spec.view`'s content element. Remove it on pop.
4. **Look.**
   - Keycaps reuse the walker footer's answer-keycap styles, tinted by `accent`. Text
     uses existing footer variables.
   - Shedding order on a narrow bar: trailing faint text, then noun truncation. Keycaps
     never shed.
   - The CSS lives in a new `Walker footer overlay` styles region.
5. **Behavior.** `announce` goes to an `aria-live="polite"` node, and `nudge` flashes
   the legend once (no motion under reduced motion).
6. **Tests.**
   - Push/update/pop ordering and LIFO behavior.
   - Paints deferred and repainted exactly once on pop, with the walk-mode state
     preserved.
   - Hidden footer, the fallback strip, and unload.
   - Zero writes and spawns.
   - Extend `preview-walker-footer.cjs` with an overlay fixture for visual review.

## Phase `absorption`

1. **Spike first, in real Obsidian.** Use an isolated scratch vault and a minimal
   fixture. Hide a heading + callout range through a StateField with a synthetic claim.
   - **Check:** codemirror-vim `j`, `k`, `gg`, `G`, arrows, PageUp/PageDown, and mouse
     clicks around the range; `[[note#Projects]]` and Outline-pane navigation; and Find.
   - **Ladder:** (a) `Decoration.replace({block: true})`; (b) (a) plus
     `EditorView.atomicRanges`; (c) a fold-style inline replace spanning the lines,
     whose widget is a one-line stub "Projects · in the card above ↑" (CM6 folds already
     behave under Vim). Choose the first rung where no motion strands an invisible
     cursor and heading navigation can be intercepted.
   - Record `SPIKE RESULT:` on the phase bead with the chosen rung and evidence, and
     retain screenshots and logs with `/sase_artifact`.
2. **Recognizer (`240wf-plugin-cabinet-regions.js`).** Write
   `propertyCardCatalogRegions(text)` per
   [Absorption](#absorption-hiding-the-stored-sections), with a fixture table derived
   from `region.rs` and `albums/index.rs` tests:
   - CRLF, BOM, frontmatter, and fences;
   - the fold signs `-`/`+`;
   - the records head title;
   - a differing heading (never adopted);
   - authored prose in the section (callout only);
   - duplicates and malformed openers (damage, no hide);
   - legacy notebook markers;
   - EOF regions.
3. **Claims (`240wi-plugin-cabinet-claims.js`).**
   `propertyCardCatalogClaim({plugin, path, kind, surface, view, fingerprint, generation, reveal}) → release()`.
   Also add `propertyCardCatalogAbsorbed(viewOrEl, path, kind)` and
   `propertyCardCatalogClaimed(view, path, kind)` for stand-down. A claim is bound to
   its generation, released on unmount, file change, mode change, and unload, and
   ignored on a fingerprint mismatch.
4. **Live Preview (`240wg-plugin-cabinet-absorb-editor.js`).** Use the StateField (plus
   the spike's rung), a refresh effect, and the strict-inside reveal. On navigation into
   or onto a hidden range, call `reveal()`. Covered contexts: Source mode, embeds,
   hover, and canvas all get no decorations.
5. **Reading (`240wh-plugin-cabinet-absorb-reading.js`).** Use source-aware section
   tagging (class `bob-cabinet-absorbed`, `display: none`) modeled on `240i`, with a
   re-scan on claim change and print and export left untouched.
6. **Body renderer (`240wj-plugin-cabinet-catalog-body.js`).**
   `propertyCardCabinetRenderCatalog({plugin, doc, container, path, regionSource, component, paint})`:
   - Render the region's callout source with `MarkdownRenderer.render` under the mount's
     `Component`, so the rendered DOM matches what Obsidian would render.
   - Bind internal-link clicks and modifier-clicks with
     `app.workspace.openLinkText(…, path, …)`, and bind hover with the `240r` hover
     helper. Links inside a CodeMirror widget get no native handling.
   - Call `paint(calloutEl)` (the kind's painter), cache the result by `paintKey`, and
     re-attach it across card repaints. That keeps interior fold state and avoids
     flicker.
   - On a throw, release the claim and show the "Couldn't render" line.
7. **Tests.**
   - Recognizer fixtures.
   - Claim lifecycle and fingerprint rules.
   - StateField decorations against a CM6 test editor
     (`scripts/cm6-mapping-editor.cjs`): present only with a valid claim, absent in
     every fail-open context, the strict-inside reveal, and boundary contact.
   - Reading tagging.
   - The body renderer with a fake painter: link binding and cache re-use.
   - Zero writes and spawns.

## Phase `chrome`

1. **Header builder (`240wc-plugin-cabinet-dom.js`).**
   `propertyCardBuildCabinet(doc, spec) → {element, head, toggle, body, keycap}`, per
   [One chrome](#one-chrome-every-cabinet-properties-included). The spec carries `kind`,
   `path`, `noun`, `summary`, `meta`, `attention`, `actions`, `open`, `keyLive`,
   `restKey`, `bodyId`, and `onToggle`. Also add
   `propertyCardCabinetPaintPick(stack, numbering|null)` for badges and the armed state.
2. **Delegation (gate open only).**
   - `propertyCardBuildFoldRow` builds a cabinet head when its fold kind maps to a
     cabinet kind. It keeps `data-card-fold`, `aria-*`, `onToggle`, attention, and
     summary semantics, so existing callers and the `_` facade keep working.
   - `propertyCardFootBuildEl` builds the Properties cabinet. The lint chip moves into
     the trailing slot (still a button when an opener exists) and the stamp becomes
     meta. The `-` keycap sits inline.
   - The record sheet's direct call (`242kc`) inherits this unchanged.
3. **Stack (`240wd-plugin-cabinet-stack.js`).** `propertyCardFootAdapterAppend` (gate
   open) calls `propertyCardCabinetsAssemble(card, path, footEl, ctx)`, which:
   - creates `.bob-property-card-cabinets` (`role="group"`, `aria-label="Cabinets"`);
   - moves the card's config and Outline cabinet heads, each with its controlled body
     (`aria-controls`), into it;
   - asks the registry's providers for this path and species and mounts each applicable
     content cabinet (head now, body lazily on first open, via `buildBody` and
     `paintKey`);
   - orders everything canonically and appends Properties last.

   Repaints reuse body nodes when the `paintKey` is unchanged. Tests must cover every
   species whose fold row and foot are not siblings (the album header and bank setup
   containers).

4. **Defaults.**
   - Under the gate, album Schema uses
     `propertyCardCabinetDefaultOpen("schema", {problems})` (closed when healthy,
     resurfacing on a new signature). This is one guarded hunk where `241v`/`241u` read
     `foldDefault`.
   - The bank foot passes `keyLive` (the keycap fix).
   - The compact details chevron drops its keycap and the `(_)` tooltip suffix while its
     card has at least one extra.
5. **CSS.** Add a new `Cabinets` region at the end of `styles.css` with every rule of
   [One chrome](#one-chrome-every-cabinet-properties-included), scoped under
   `.bob-property-card`. Light, dark, reduced-motion, forced-colors, and the container
   queries are all required. Existing rules are untouched, and the cabinet look is
   reached by higher specificity and the new classes alone.
6. **Preview.** Extend `scripts/preview-property-cards.cjs` with cabinet fixtures:
   - an area host with four content cabinets, at rest and armed;
   - an album at rest;
   - the bank;
   - a prose note;
   - a record (zero extras);
   - narrow and wide widths.

   Render light and dark for visual review.

7. **Tests.**
   - Closed-gate parity again, plus the open-gate structure per species: order,
     Properties last, roles, a11y attributes, and unique ids per mount.
   - Lazy body: built once and re-used.
   - Focus returns to the head when collapsing a body that contains focus.
   - The Schema default flip and resurfacing.
   - Zero writes and spawns on paint and toggle.

## Phase `keys`

1. **Dispatch.** Add one guarded hunk in `248m`: under the open gate, the `_` arm calls
   `propertyCardCabinetUnderscore(view)` (`240we-plugin-cabinet-pick.js`) with the
   0/1/2+ routing from [The `_` keymap](#the-_-keymap). `canAct` is false only when none
   applies, so Vim `_` passes through. The `toggle-card-details` palette command uses
   the same dispatch. `-` keeps its arm but first dismisses a live pick.
2. **Pick state machine.**
   - **Arming.** While armed, push a catch-all Obsidian `Scope` and a window
     capture-phase `keydown` listener, modeled on `687-plugin-window-prefix.js` (copy
     the pattern, not the code).
   - **Event handling.** De-duplicate window and document delivery, ignore `repeat` and
     IME composition, and swallow handled keys with `preventDefault`, `stopPropagation`,
     and `stopImmediatePropagation`.
   - **Snapshot.** Capture the ordered extras as semantic kinds, the view, the path, and
     the mount generation.
   - **Disarm.** Every disarm trigger in the table releases everything.
3. **Badges and footer.**
   - On arm, call `propertyCardCabinetPaintPick` immediately and
     `walkerFooterPushOverlay` with the 150 ms delay, the card accent, chips 1..N plus
     `- Properties` and faint `esc`, and the announcement. Wire chip clicks to the same
     toggle.
   - An out-of-range digit calls `nudge` with "Choose 1–N".
   - On disarm, clear badges and pop the overlay.
4. **After a pick.** Run `propertyCardCabinetToggle`, scroll a newly opened offscreen
   head into view (`nearest`), and announce "Projects opened" or "Projects closed".
5. **Tests.**
   - The full keyboard matrix through the harness: 0/1/2+ extras, `__`, `-`, every
     cancel key, combos, other keys passing through, out-of-range digits, no digit
     reaching Vim, every disarm trigger, the mid-pick repaint, and the vanished kind.
   - Palette parity, and no listener left after disarm or unload.
   - Zero writes and spawns.

## Phase `host-cabinets`

Implement this in `241al-plugin-cabinet-host-catalogs.js` (split if over 1000 lines),
with stand-down hunks in `241ae`, `241aj`, `241d`, and `241q`.

1. **Providers.** Register `areas`, `projects`, `notebooks`, and `albums`, applying on
   area and project cards with confirmed source. Without confirmed source there are no
   content cabinets, so the native catalogs stay.
   - `applies` recognizes the region and computes the summary from the bytes (direct and
     nested counts) and the cached report through each painter's existing tally model.
     It also sets attention (stale, missing, or damaged) and an explicit `↻ Sync` action
     wired to the painter's existing sync gesture.
   - `buildBody` runs the body renderer with the painter, which is `areasPaintCallout`,
     `projectsPaintCallout`, the notebook painter, or `albumPaintCallout`. Each gets the
     same options as the inline Live Preview path, including the view's fold maps.
   - **Parity.** Nothing the inline painter shows is lost. Title-row status moves into
     the header, and the callout title row hides.
2. **Missing catalogs.** When the report has children but no stored region exists, the
   cabinet still mounts with `⚠ missing` and Sync. Its body is the painter's existing
   standalone status surface. There is no claim, because nothing is hidden.
3. **Claims and stand-down.**
   - Claim after the head mounts and the model validates.
   - In each painter, skip callouts inside `.bob-property-card-cabinets` and absorbed
     regions, and suppress standalone missing-catalog anchors where
     `propertyCardCatalogClaimed`/a mounted cabinet covers that kind.
   - Confirm that the report subscriptions still refresh with no inline callout
     painting.
4. **Navigation.** `reveal()` opens the cabinet and scrolls its head into view. This
   covers `[[note#Areas]]`, `#Projects`, `#Notebooks`, `#Albums`, and the Outline pane.
5. **Tests.**
   - Per-kind parity: the painted cabinet-body DOM text equals the closed-gate inline
     callout DOM text for the same fixture.
   - Summaries with and without a report, and the missing, stale, and damaged states.
   - Stand-down (no double paint, no stray anchors), the 64-line Projects fixture, and
     the three-catalog host.
   - The untyped root host stays native.
   - Zero writes and spawns, except a clicked Sync, which invokes exactly the existing
     writer.

## Phase `records-cabinet`

Implement this in `241tb-plugin-cabinet-records.js`, with hunks in `241t`, `241q`
(`apiAlbumOpenGroup`), and `241hf`.

1. **Provider.** Register `records` on album cards, always present and open by default.
   - The summary is the stored `[!records]` head tally plus grouping.
   - The body is the body renderer with `albumPaintRecordsCallout` under the same
     options and fold map as the inline path. This covers group folds, facets, date
     chips, and the empty-album teaching line.
   - For `subscriptions.md` the body also runs the `241hf` decoration.
   - Index order, labels, and facets stay verbatim.
2. **Claims and stand-down** follow `host-cabinets` step 3, for `241t`.
3. **Group jump.** `apiAlbumOpenGroup` opens the album, opens Records, expands the group
   through the existing `albumRecordsPendingExpand` path, and scrolls the group into
   view. With the gate closed, the jump behaves exactly as today.
4. **Perf.**
   - On a synthetic 106-record album, measure the card paint with Records open against
     today's inline records paint, and record both in a bead note.
   - Fail the phase if the cabinet paint exceeds 2× the inline paint or adds a long task
     over 100 ms.
5. **Tests.**
   - Parity with the inline index (including subscriptions), the empty album, and the
     group jump.
   - Records-open plus Schema-closed defaults, and `_` arming on albums (`1` Schema, `2`
     Records).
   - Zero writes and spawns.

## Phase `acceptance`

1. **Setup.**
   - Add a sibling runner, `scripts/obsidian-cabinets-e2e.cjs` with a `-lib`, that
     imports the existing card e2e libs. Run it against a scratch vault under `xvfb-run`
     on athena, per bob-plugins `docs/develop/validation.md`, and read the `tailnet.md`
     reference memory before using athena.
   - Plugin data sets `cabinetsPreview: true` plus every sibling gate still present on
     master (`cardPreviewKinds` for notebook/prose, `refCardPreview`), so the whole
     family is exercised.
   - Use synthetic fixtures shaped like the live vault:
     - a `cash`-like area with all four catalogs;
     - a `sase`-like project with a 64-line Projects callout;
     - a leaf project;
     - an `eat`-like album (106 records);
     - an empty album;
     - a subscriptions album;
     - the bank hub;
     - a prose note;
     - a notebook;
     - a record;
     - a ref, if qp's card is on master;
     - a day note;
     - an untyped root `area.md`.
2. **Matrix.**

   | Area           | Checks                                                                                                                                                                                                                                                                            |
   | -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Render         | Every species in Live Preview and Reading. Stack order, Properties last, summaries, attention, and Sync. Records open and Schema closed. The Setup and Outline unchanged defaults. The zero-extra `_` fallback.                                                                   |
   | Content parity | Cabinet bodies against the gate-closed inline catalogs (text and row counts), and the subscriptions decoration.                                                                                                                                                                   |
   | Keys           | The full [keymap](#the-_-keymap) table in Live Preview Vim normal mode, the palette arm in Reading, footer chips and clicks, the fallback strip with the status bar hidden, and the 150 ms delay (no flash on a fast `_1`).                                                       |
   | Absorption     | Hidden H2+callout. Kept heading with authored prose. Strict-inside reveal through Find. Vim `j`/`k`/`gg`/`G` and arrows with no invisible cursor. `[[note#Projects]]`, `#Records`, and the Outline pane open the cabinet. Toggling an offscreen cabinet never jumps the viewport. |
   | Fail open      | Source mode, embeds, hover, canvas, print/PDF export, a failed mount (forced throw), malformed and duplicate regions, the gate closed, and plugin disable. Each shows the stored section in the same frame.                                                                       |
   | Passive        | Spy on paint, toggle, arm, pick, cancel, and overlay: zero vault writes and zero bob spawns. Protected fixture hashes are unchanged.                                                                                                                                              |
   | Perf           | The 106-record album and the 64-line Projects host, within the `records-cabinet` budget.                                                                                                                                                                                          |

3. **Screenshots.** Light/dark × narrow (~420 px)/wide × LP/Reading for the area host at
   rest and armed, the album, the bank, prose, a record, and the footer overlay. Add
   reduced motion and forced colors for the area host. Copy the screenshots back and
   inspect every PNG.
4. **Docs.** Add a `Cabinets` runner section to `docs/develop/validation.md` (additive).
5. **Verdict.** Retain the report and screenshots with `/sase_artifact` and record
   GO/NO-GO on the bead. For NO-GO, fix in place if the fix is in scope. Otherwise
   record `PROPOSED FOLLOW-UP:` notes and stop. The phase closes only on GO.

## Phase `go-live`

0. **Gate.**
   - Confirm the gating beads are closed with `sase bead read`. Those are
     `bob-cli-qm.7`, plus `bob-cli-qp.8` when `golive_waits_for_qp` is yes. If one is
     open, poll the non-audited `sase bead list --status closed --type phase -f json`
     under `/sase_monitor` for up to 96 h. On timeout, stop without changes and note it.
   - A canceled sibling does not block, because there is then no race. Note it.
   - Re-read `glossary:property-card`, family docs, and `docs/develop/architecture.md`
     as those go-lives left them.
1. **Switch on.** Delete the `cabinetsPreview` gate, its plugin-data handling, and the
   closed-gate branches. Keep the delegation and fold-role behavior as the only path.
   Convert the closed-gate parity tests into regression tests of the live behavior.
2. **Passive proof.** Extend the family passive-rendering suite to cover cabinet paint,
   toggle, pick, and overlay, with zero writes and spawns.
3. **Docs (bob-plugins).**
   - Add a new reference page, `docs/cabinets.md`: anatomy, kinds and order, defaults,
     the `_` pick, the footer overlay, absorption and fail-open, and passive guarantees.
   - Update `nav.yml` and the generated inventory (`npm run docs:check`).
   - Update the family pages so `_` and the foot say "cabinet": area/project cards,
     albums, bank, prose card, notebook card, record card, ref card, and
     `bob-ledger-tools.md`. Also update `docs/develop/architecture.md`.
   - Docs say "Properties cabinet", and "drawer" means only the Task Drawer.
4. **Docs (bob-cli).** Grep `docs/` for stale fold or "Properties drawer" wording in
   bob-cli-owned pages and fix only what is now wrong. Run `just docs-gen` and
   `just docs-check`.
5. **Memory.** Use `/sase_memory_write` for every memory edit. Edit only the notes the
   accepted decisions cover, then run `sase memory init` and `just docs-gen`.

   > [!decision] memory_cabinet_term Add `glossary:cabinet`, refined to the shipped
   > behavior from this draft:
   >
   > **Cabinet** _(aka property cabinet, cabinet stack)_ — A named, independently
   > collapsible compartment at the bottom of a [[property-card]]. It has one header row
   > (chevron, noun, a one-line summary that still says what a closed cabinet holds, any
   > attention chip or stamp, and a keycap) over a body that opens in place. Cabinets
   > stack in one fixed order, with the **property cabinet** (Properties, toggled with
   > `-`) always last. Above it come content cabinets (Areas, Projects, Notebooks, and
   > Albums on an [[area-card]] or [[project-card]], Records on an [[album-card]],
   > Outline on a prose card), then configuration cabinets (Schema on an album card,
   > Setup on the [[bank-card]]). Empty catalogs get none. Records start open, the other
   > content cabinets start closed, and Schema and Setup open themselves for a new
   > problem.
   >
   > A content cabinet shows the same rows, grouping, and actions as the bob-owned
   > catalog callout it stands in for. The callout stays in the note, and only its
   > rendered section (plus an H2 that labels nothing else) hides while a working
   > cabinet is mounted. Source mode, mobile, embeds, hover, print, and anything
   > uncertain show the stored section instead.
   >
   > In Live Preview Vim normal mode, `_` toggles the only cabinet above Properties.
   > With two or more, it arms a numbered pick: digits count up from `1` just above
   > Properties and are lit on the headers and in the footer. A digit toggles that
   > cabinet, `_` again picks `1`, `-` picks Properties, and Esc, `q`, or Ctrl+`[`
   > cancels. A card with no cabinet above Properties keeps `_` for its own details
   > fold.
   >
   > Open or closed is session-only view state: never frontmatter, never a vault write,
   > and never a bob spawn. Not a [[task-drawer]], an Obsidian heading or callout fold,
   > the compact details chevron, or the stored H2 section itself.

   > [!decision] memory_card_strands In `glossary:property-card`, replace "`_` folds the
   > card's own section" and the foot's "Properties drawer" with the cabinet stack, `-`,
   > and the `_` routing. In `glossary:area-card` and `glossary:project-card`, replace
   > "`_` folds the Area's own details" and "`_` folds the card's own details": child
   > catalogs render as cabinets above the Properties cabinet, and `_` toggles or picks
   > them (the details fold only when none exist). Link [[cabinet]] from all three.

   No other memory note changes. `glossary:album-card` and `glossary:bank-card` stay
   true as written, so record a `PROPOSED FOLLOW-UP:` note on this phase bead proposing
   "Records cabinet", "Schema cabinet", "Setup cabinet", and "Properties cabinet"
   wording for them.

   > [!decision] memory_decision_record Add
   > `decisions:cabinets-absorb-catalogs-at-render-time`, which applies to bob-plugins
   > and the vault.
   >
   > - **Claim.** Cabinets render bob-owned catalogs at render time, the stored callouts
   >   stay byte-for-byte bob's, and nothing hides without a working cabinet.
   > - **Why.** The report's unanimous points 3–4 and the readers of the bytes.
   > - **Rejected alternatives.** Literal H2 removal, report-JSON bodies, nested
   >   scrollers, and a per-note H2 order.
   > - **Cost.** The Obsidian Outline still lists the stored headings.
   > - **Reopens when.** Bryan wants catalogs out of the Markdown, or mobile gains
   >   cards.
   > - **Evidence.** The research report and this epic's commits.

6. **Deploy and smoke.**
   - Run build, tests, and `bob plugins sync` (refusal protocol, no `-F`).
   - Run a post-switch smoke with the acceptance runner without the gate key. It must
     include the prose Outline cabinet over qm.5's landed body and a ref card.
   - Retain the evidence with `/sase_artifact`.

## Epic acceptance (land agent)

- Every phase is closed, `cabinetsPreview` no longer exists, and `bob plugins sync` is
  clean on master.
- `acceptance` recorded GO, with retained screenshots for every species, light/dark, and
  narrow/wide.
- The `go-live` smoke evidence shows the following:
  - the area host's four closed content cabinets with honest summaries;
  - an album with Records open over a closed Schema;
  - the bank's Setup and Properties in one chrome;
  - `_` arming with digits on the card and in the footer;
  - `__`, `-`, and Esc/`q`/Ctrl+`[` behaving per the table;
  - stored catalogs visible in Source mode.
- The passive suites are green with zero writes and spawns, and no bob-cli code changed.

## Out of scope

- Removing the stored H2 sections or changing the vault format. That would be a separate
  epic with migration, rewritten Rust readers, and a superseding decision.
- A root hub card for the untyped `area.md`.
- Persisted cabinet state across sessions.
- Virtualizing large bodies.
- Cabinets on mobile.
- Any Bob Mac Capture or bob-cli change.
- Rewriting the notebook card body.
- Renaming code identifiers that say "drawer" or "foot".

## Follow-ups to consider after go-live

- A device-local persistence option if re-opening cabinets becomes tedious.
- A root `area.md` hub card, so the last native catalog can join.
- Hiding absorbed headings from Obsidian's Outline pane, if that proves feasible without
  touching the bytes.
- An explicit picker if a species ever exceeds nine extras.
