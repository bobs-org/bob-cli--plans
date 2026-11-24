---
tier: epic
title: Property cards own the frontmatter — foot row, =- drawer, created stamp, =+
  editor, takeover
goal: 'On every note a property card mounts on, the card shows the whole frontmatter:
  a quiet foot row with a concise created stamp and a collapsed read-only Properties
  drawer toggled by =-. A keyboard-first =+ property editor, backed by a byte-preserving
  `bob props` writer, does at least everything Obsidian''s native Properties can on
  any Markdown note. Obsidian''s native Properties block is hidden on card notes,
  and every failure path brings it back.

  '
phases:
- id: foot-drawer
  title: Card foot row, drawer, created stamp, and =- (bob-plugins)
  depends_on: []
  size: large
  description: 'foot-drawer: kit created formatter, inventory model, foot row and
    read-only drawer adopted by every card (record fold and footer migrated), =- arm
    on the generalized = gate, palette and Command Mode toggle, tests, e2e, docs.'
- id: props-cli
  title: Generic bob props show|update (bob-cli)
  depends_on: []
  size: large
  description: 'props-cli: new bob props group on yaml_spans.rs with fingerprinted
    span ops, role table, typed refusals, byte-exact inverse ops, props_v1 capability,
    catalog and completion entries, golden byte-preservation tests, docs/props.md
    contract.'
- id: owns-frontmatter
  title: Whole-container ownership primitive and focus spike (bob-plugins)
  depends_on: []
  size: medium
  description: 'owns-frontmatter: unwired propertyCardOwnFrontmatter kit primitive
    (StateField, Reading class, CSS variables plus direct-child fallback, inert and
    focus guard) with unit tests and the S1 real-Obsidian go/no-go probe recording
    the Add file property command id.'
- id: editor-core
  title: The =+ property editor (bob-plugins)
  depends_on:
  - foot-drawer
  - props-cli
  size: large
  description: 'editor-core: keyboard-first modal on the record chassis for any note:
    grouped rows with route marks, value stages incl. checkbox, date-time, tags, add
    flow with vault completion, delete, undo/redo, y excursion, == hand-off, degraded
    modes, entry points, tests, docs.'
- id: editor-parity
  title: Rename, reorder, multi-select, copy/paste, filter, kind change, YAML stage
    (bob-plugins)
  depends_on:
  - editor-core
  size: large
  description: 'editor-parity: completes native Properties parity and beyond in =+
    with rename, reorder, multi-select batch ops, YAML copy and overwrite-previewed
    paste, filter, feature-detected kind and vault type change, and a validated YAML
    stage.'
- id: takeover
  title: Hide native Properties on card notes (bob-plugins, plus memory if approved)
  depends_on:
  - foot-drawer
  - owns-frontmatter
  - editor-parity
  size: large
  description: 'takeover: wires whole-container ownership into album, record, and
    intention cards behind eligibility and a cached props_v1 probe, reroutes Mod+;
    and native focus helpers to =+, runs the fail-open e2e matrix and visual walkthrough,
    updates docs and approved memory.'
decisions:
  bank_foot:
    ask: Should the bank card on bank.md also get the foot row, drawer, and created
      stamp?
    default: true
    why: You asked for created on all cards; bank.md still keeps native Properties
      (research D5)
    answer: true
  drawer_memory:
    ask: How should an opened =- drawer be remembered?
    choices:
      note: Per note for the session; walking ]n to another day shows it collapsed
        again
      pane: Per pane for the session; it stays open while walking ]n/[n through notes
    default: note
    why: Matches every existing card fold (record fold, schema chips, bank setup)
    answer: note
  memory_glossary_cards:
    ask: Correct the Property Card and Record Card glossary strands for whole-frontmatter
      ownership?
    memory:
    - glossary:property-card
    - glossary:record-card
    default: true
    corrects: '`parent`, `type`, `created`, legacy, and unrelated keys stay in Properties.'
    answer: true
  memory_decision_record:
    ask: Add a decision record that cards own in-document Properties, with one writer
      per key?
    memory:
    - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.5v.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 14:35:08
status: done
bead_id: bob-cli-pf
---

- **PROMPT:** [prompts/202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/card_owned_frontmatter.md)
- **BEAD:** [bob-cli-pf](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/README.md)

# Plan: Property cards own the frontmatter

This epic turns the research consensus in
`research:202611/card_owned_frontmatter/card_owned_frontmatter.md` into six phases.
Bryan accepted every requirement that research recommends, including adjustments R1–R7
and resolutions D1–D8. Memory edits are the only exception: they happen only through the
memory decisions above. Each phase below is self-contained. Read the research report
with
`sase artifact read research:202611/card_owned_frontmatter/card_owned_frontmatter.md "<why>"`
when a phase needs the reasoning behind a rule. Its sibling `__gem.md` §4.4 lists the
YAML byte-safety invariants this plan turns into `bob props` tests.

## The experience (design, owned by this plan)

### 1. The foot row: one quiet line at the bottom of every card

Each mounted property card ends in a **foot row**: one line, 1.6 em high, with a
hairline top border in the rail's ink. It sits **below** the nav rail (D7), so opening
the drawer never moves the Previous/Next arms during a walk.

```
‹ Thu 19            ↑ W47 · Nov 16 – 22            Sat 21 ›     ← existing rail
───────────────────────────────────────────────────────────────
▸ Properties · 14                                created 6:20 AM   ← foot row
```

- **Left:** a real `<button aria-expanded aria-controls>` disclosure reading
  `▸ Properties · N`, where N counts the top-level keys. The chevron rotates 90° over
  160 ms. Hovering shows a `=-` keycap tooltip ("Toggle properties =-").
- **Right:** the concise **created stamp** (§3), muted, with tabular numerals.
- **Lint:** when something is wrong, an amber `⚠ N` chip sits between the two. Causes
  are an invalid `created`, frontmatter that does not parse, an unclosed fence, or a
  duplicate key. The chip never opens the drawer by itself; its tooltip names the
  problems.
- The foot row stays rendered when the card body is collapsed (record and bank collapse)
  and on identity-only cards. That is how `created` stays visible "even when collapsed".
- Painting never spawns bob and never writes the vault. It reads the metadata cache plus
  the confirmed source bytes, like every other card surface.

### 2. The `=-` drawer: read-only, complete, grouped

`=-` (or a click on the disclosure, or **Bob: Toggle card properties**) opens a
read-only drawer under the foot row. The drawer is collapsed by default. Open state
lasts for the session and is never persisted.

> [!decision] drawer_memory = note State is a per-path session map in the kit, so a
> different note starts collapsed.

> [!decision] drawer_memory = pane State is keyed by the leaf/pane rather than the path:
> an open drawer stays open across `]n`/`[n` and note changes in that pane, and closes
> when the pane closes.

Content is the **complete inventory** in file order (D2). Membership and order come from
the confirmed source bytes, because JS objects reorder integer-like keys. Values come
from the metadata cache. Rows fall into three groups:

1. **Loose keys** (no label): keys the card renders nowhere else. Album-declared legacy
   keys keep the existing `legacy` chip.
2. **On this card**: keys the card renders. Each carries a faint, right-aligned _where_
   mark: `kicker`, `tile`, `title`, `field`, `intentions`, `summary`, `reflection`,
   `agent`, `schema`, or `foot`. This makes the card-to-YAML mapping learnable, which is
   what makes the takeover trustworthy.
3. **▸ Managed · N**: a closed sub-fold for bob-written and importer keys (`*_digest`,
   `*_drafted`, `zorg_*`, `generated_from_zorg*`). When an adapter declares a key "on
   this card", that wins over the managed pattern. Example: `summary_drafted` shows as
   `✦ agent` on day cards.

Value rendering matches the rest of the card:

- **Links:** wikilinks navigate, Mod-click opens a new tab, and hover shows Obsidian's
  page preview. URLs open externally.
- **Tags:** chips that open search.
- **Lists:** chips, with `+N` after six.
- **Scalars:** booleans show as `✓` or `✗`; null shows a muted `—`; long text clamps to
  two lines, with the full text in a tooltip.
- **Empty values stay distinguishable:** absent, null, empty string (`empty`), empty
  list (`empty list`), `false`, and `0` each look different.
- **Nested maps** (album `record:`, bank `bank:`): one summary row such as
  `record · 8 fields  ==`. Activating it opens that species' `==` editor. A map is never
  dumped as YAML.
- **Duplicate keys:** both source lines show, with an amber lint.
- **Unparseable frontmatter:** the drawer shows one teaching line, "Frontmatter doesn't
  parse — `y` opens YAML", and no rows.

Rows are focusable:

- `y` jumps to that key's YAML line. This generalizes the record sheet's
  `recordSheetBindYamlJump` into the kit.
- Enter, or a click on the key label, opens the `=+` editor on that key once the editor
  is registered. Clicking a value link still navigates.
- After `editor-core` lands, the drawer ends with a quiet action line: `＋ Add property`
  and `✎ Edit all  =+`.

### 3. The created stamp

One kit formatter is shared by all species. The species supplies the verb: record keeps
its existing copy `added`, and every other card says `created`.

| Situation                                             | Text                                                                                               |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Day card, created on the note's own date              | `created 6:43 AM` (the title already says the date)                                                |
| Temporal card (day/week/quarter/year), any other date | absolute: `created Nov 19, 9:41 PM` (no relative words on temporal cards; the title is the anchor) |
| Non-temporal card, today / yesterday                  | `created today, 7:55 AM` / `created yesterday, 9:41 PM`                                            |
| Non-temporal card, this year / earlier                | `created Jun 11` / `created Jun 11, 2025`                                                          |
| Date-only value                                       | never shows a time and never a fake `12:00 AM`                                                     |
| Missing on a temporal card                            | nothing (no nagging on 1,111 dailies and every period)                                             |
| Missing on album/record/bank                          | faint `no created date`                                                                            |
| Invalid                                               | amber `created ⚠`; the raw value and lint appear in the drawer                                    |

- The tooltip always shows the raw stored string with its offset, plus the age.
- Times render in the viewer's local zone. Date-only values are calendar dates, never
  shifted to UTC.
- The parser accepts `T` or a space as the separator, offsets `-0500`, `-05:00`,
  `+0000`, `Z`, or none, and YAML Date objects. This fixes the space-separated case that
  the current record regex (`242i:181-280`) mishandles.
- **The stamp never fabricates a value** from file times, git history, or the date a
  daily note represents.

### 4. The `=` chord family

`==` opens the card editor (existing), `=-` toggles the drawer, and `=+` opens the
property editor. Vim's `=` is the complete `indentAuto` operator, so a vimrc cannot bind
these chords. The existing capture-phase gate in `248m-plugin-schema-double-equals.js`
can.

- That gate is generalized into a **second-key arm table**. Each arm keeps today's
  snapshot checks: normal mode, Live Preview, a clean pending `=` with an empty key
  buffer, no count, no register, no IME, no repeat, and no Ctrl/Alt/Meta.
- Arms match on `event.key`, **never rejecting Shift**. On US layouts `+` is Shift+`=`.
- On acting, the gate prevents and stops the event, sends
  `Vim.handleKey(cm, "<Esc>", "mapping")` to cancel the pending operator, and runs the
  arm.

| Second key | Arm                                                           | Falls through to Vim when                                                         |
| ---------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `=`        | existing species registry                                     | no species matches (unchanged)                                                    |
| `-`        | toggle the drawer of the card mounted in this view            | no card with a foot row is mounted in this view                                   |
| `+`        | open the property editor on this note (any Markdown note, D4) | the arm is not registered yet (before `editor-core`) or there is no Markdown file |

Reading view and non-Vim use get palette commands (**Bob: Toggle card properties**,
**Bob: Edit note properties**), Command Mode rows (`CM_SHEET_SURFACE_ROWS` in
`bob-command-mode/src/160-sheet-rank.js`), and the clickable disclosure. There is no
Reading-view prefix window (R3).

When `=-` expands a drawer whose foot row is off-screen, it scrolls the foot row into
view (`block: "nearest"`, smooth unless `prefers-reduced-motion`) and leaves the cursor
alone. Collapsing never scrolls.

### 5. The `=+` property editor

The editor is a modal on the Record Card editor chassis: `albumSchemaCardModalBase`, a
regions layout, the guarded update session, the close ladder, and the hint strip plus
`⋯ All actions` menu. It wears its card's paper and identity badge, so `==`, `=-`, and
`=+` read as one object.

- **Header:**
  - The species tile, or a neutral page tile on notes without a card.
  - A kicker `<KIND> · PROPERTIES`. KIND comes from bob's `note_kind`: Day, Week,
    Record, Album, Area, Project, Notebook, Reference, or Note.
  - The note title.
  - A muted line `path · N properties`.
  - A `/ filter` pill on the right.
- **Rows:** in file order, grouped like the drawer: Loose / On the `<Kind>` card /
  Identity / Bob-managed. Each row has:
  - a type glyph that reuses **Obsidian's own property-type Lucide icons** (text, list,
    number, checkbox, calendar, clock, tags, aliases), so muscle memory transfers;
  - a monospace key;
  - the rendered value;
  - a right-aligned route mark: the type name, `→ ==`, or `locked · y YAML`.
- **Status rule:**
  - `Checking value…` while a preview runs;
  - `✓ Saved · u undo` after a write;
  - bob's refusal message and hint verbatim, otherwise.
- **First frame:** paints instantly from the metadata cache, then reconciles with
  `bob props show` (the record card's paint-before-report idiom).
- **Routing (D8, one writer per key):** bob is the authority, using the role table in
  `props-cli`.

  | Role     | In the editor                                               |
  | -------- | ----------------------------------------------------------- |
  | derived  | locked; one teaching line names its writer; `y` opens YAML  |
  | card     | Enter hands off to that species' `==` editor on the field   |
  | identity | locked in v1 (`type`, record/album `parent`, album `title`) |
  | loose    | edited here through `bob props update`                      |

  `created` is a loose, editable key.

- **Keyboard grammar** (browse mode; `q` stays reserved per
  `decisions:q-and-ctrl-bracket-mirror-esc`):

  | Keys                                      | Action                                                                      | Phase  |
  | ----------------------------------------- | --------------------------------------------------------------------------- | ------ |
  | `j`/`k`, ↑/↓, Ctrl+N/P, `g`/`G`, Home/End | move                                                                        | core   |
  | Enter / `l`                               | edit value; on a checkbox row it flips immediately (no stage)               | core   |
  | `o` / `O` / Mod+;                         | add a property below / above / at end: name stage, then value stage         | core   |
  | Ctrl+D / Mod+Backspace                    | delete the property (undoable)                                              | core   |
  | `u` / Ctrl+Z, Ctrl+R / Ctrl+Shift+Z       | undo / redo (per-path stack)                                                | core   |
  | `y` / Ctrl+E                              | YAML excursion to this key's line, with the floating Done return            | core   |
  | `=`                                       | hand off to this note's `==` card editor (on the field when supported)      | core   |
  | Esc / `q` / Ctrl+[                        | close ladder: menu → stage → filter → close                                 | core   |
  | `r`                                       | rename the key (span-preserving)                                            | parity |
  | Alt+J / Alt+K (Alt+↓/↑)                   | move the property down / up; comments travel with it                        | parity |
  | Space, `V`, Shift+↑/↓                     | mark, visual range, extend selection                                        | parity |
  | Mod+C / Mod+V                             | copy marked rows as YAML / paste with overwrite preview                     | parity |
  | `/`                                       | filter keys and values                                                      | parity |
  | `t`                                       | change this value's kind; vault-wide type change behind an explicit warning | parity |

- **Stages:**
  - Reused from the record editor: Text (with the nav `[[`/`((` prose band), Suggest
    (existing values of this key across the vault), Number, Date, URL, Link, and List.
  - New: Checkbox (no stage), Date & time, Tags, and YAML.
  - Date & time accepts `now`, `9:41pm`, `fri`, `-3d`, and ISO. bob keeps the existing
    offset style.
  - Tags offers vault tag completion and writes values without `#`.
  - YAML is a validated stage for nested or exotic values; native Properties cannot edit
    these at all.
- **Add flow:**
  - The name stage completes over vault property names, showing each name's vault type
    and usage count (`metadataCache.getAllPropertyInfos()`, feature-detected).
  - Tab or Enter goes to the value stage that type picks. Shift+Enter keeps the typed
    name as a new text key.
  - Keys are validated locally and again by bob: not empty, not a duplicate, and a valid
    YAML key.
- **Write discipline:** the record session pattern exactly.
  - Discrete ops (flip, delete, reorder, undo) apply immediately with the current
    fingerprint.
  - Typed values get a 250 ms debounced dry-run, and Enter latches on a current preview.
  - Every open pane is saved before apply.
  - A stale refusal keeps the draft and rebases.
  - The optimistic overlay repaints the card and drawer for at most 5 s.
- **Degraded modes are honest:**
  - **bob without `props_v1`:** rows still render and writes are disabled. The status
    line teaches "Update bob to edit properties here — `y` opens YAML".
  - **Invalid YAML:** a recovery banner shows bob's parse error and line, and `y` jumps
    there.
  - **No frontmatter:** "No properties yet — `o` adds the first".

### 6. The takeover: hide native Properties completely, fail open always (R1, D6)

**Whole-container ownership** is a third kit ownership bit, parallel to title ownership:
`propertyCardOwnFrontmatter(scope, { path })` adds the scope class
`bob-property-card-owns-frontmatter`. It is released by generation-bound handles,
re-applied by observers, and restored in the same frame on Source toggle, path change,
mount loss, unload, or an exception.

- **Live Preview:** a lazily registered CodeMirror `StateField` plus `editorAttributes`,
  modeled on `240i`. Ownership drops in the same transaction as a Source toggle or a
  file change.
- **Reading view:** a generation-bound class on `.markdown-preview-view`.
- **CSS:**
  - Set Obsidian's own visibility variables (`--metadata-display-editing` /
    `--metadata-display-reading: none`) on the owned scope.
  - Add a direct-child `display: none !important` fallback on the exact native paths
    that `propertyCardNativeContainer` uses.
  - Never write a global `.metadata-container` rule. The node stays in the DOM because
    it is the title slot's anchor.
- **Focus safety:** `inert` plus `aria-hidden` on the container, and a `focusin` guard
  that returns focus to the editor, so Vim motions, `gg`, arrows at the top line, Tab,
  and screen readers never land in a hidden input.
- **Eligibility:** all of these must hold.
  - The card mounted successfully.
  - The frontmatter parses in the cache **and** the source has a closed `---` block.
  - Desktop, in Live Preview or Reading view.
  - Not an embed, popover, canvas, or print view.
  - **The installed bob reports `props_v1`.** This is probed once per plugin session per
    bob path, off the paint path, and cached. Until it is confirmed, today's per-row
    hiding stays, so version skew across machines can never strand you without an
    in-document editor.
- **Bank is excluded** (D5). bank.md keeps its native block and today's `bank` row
  ownership.
- **Native entry points never dead-end (R7):**
  - The **Add file property** command (Mod+;) opens `=+` in add mode on owned notes. It
    wraps the command so custom hotkeys and the palette are covered too, and restores it
    on unload.
  - Helpers that focus native property inputs (e.g. `notebookFocusSummaryProperty`)
    route through `=+` when the scope is owned.
  - The right-sidebar File Properties view is never touched. It is the permanent escape
    hatch.

> [!decision] bank_foot The bank card under `# 🏦 Bank` adopts the foot row, drawer, and
> created stamp like every other card. bank.md's native Properties block stays visible
> (D5), so on bank.md the drawer duplicates it by design until areas get a title-block
> card.

> [!decision] bank_foot = no The bank card is untouched in v1: no foot row, no `=-` on
> bank.md (`=-` falls through to Vim there). `=+` still works on bank.md as on any note.

### 7. Visual language (kit-wide; applies to every phase)

- **Type:**
  - Keys: monospace, 0.86 em, muted.
  - Values: the text font.
  - Group labels: the kicker's tracked small caps.
  - Numerals: tabular everywhere.
- **Layout:**
  - The drawer grid is `minmax(84px, max-content) 1fr auto` (key, value, where-mark).
  - Below about 420 px of container width, a container query stacks the key over the
    value.
- **Color:** no new colors.
  - Accent marks only the selected editor row and the active stage.
  - Amber is reserved for lint.
  - Where-marks and route marks use the faint text color.
- **Motion:**
  - The drawer animates `grid-template-rows` 0fr↔1fr over 160 ms, and the chevron
    rotates with it.
  - There is no motion under `prefers-reduced-motion`.
  - The drawer body is built only when opened.
- **Semantics:**
  - `<dl>` rows.
  - A real disclosure with `aria-expanded`.
  - The editor keeps the chassis's polite live region.
- **Verification:** every phase that paints verifies light and dark themes, narrow and
  wide panes, with real-Obsidian screenshots.

### 8. Write model: one writer per key

- **Painting is read-only and fail-open.**
- **Every write goes through bob with a fingerprint:**
  - Loose keys go through the new generic `bob props update`, built on
    `src/native/yaml_spans.rs`.
  - Card-owned keys go through their species editor (`records`, `albums`, `periods`, or
    `bank config update`).
  - Derived keys belong to bob and are not edited here.
- **`processFrontMatter` stays banned**
  (`bob-navigation-hotkeys/src/211-parent-frontmatter-plan.js`), and no phase adds a JS
  YAML writer (D1).

## Phase `foot-drawer`: card foot row, drawer, created stamp, and `=-` (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. All work is in
`plugins/bob-ledger-tools` plus one Command Mode row. Native Properties stays visible in
this phase. Temporary duplication is the safe half-state.

Build these new pure kit fragments, each under 1000 lines, with `374*` test exports. The
names are suggestions; keep the manifest order valid.

- **Created formatter** (`240m-…-created.js`): parse and format per §3, returning
  `{ text, tooltip, state: ok|missing|invalid, dateOnly }`. Pure, with the clock
  injectable.
- **Inventory model** (`240n-…-inventory.js`):
  - Scan top-level key order and duplicates from source text; fall back to
    metadata-cache order.
  - Group rows per §2 from adapter declarations
    `{ where: {key: mark}, nested: {key: opener} }` and the managed patterns.
  - Classify value shapes and lints, and return the count used by the foot row.
- **Foot row and drawer DOM** (`240o-…-foot-dom.js`): §1, §2, and §7.
  - Includes the kit `y` row-jump, generalized from `recordSheetBindYamlJump`
    (`242j:131`, `242k:497`).
  - Includes an `onOpenEditor(path, {key, mode})` hook, unset in this phase.
- **Drawer state and toggle API** (`240p-…-foot-state.js`): per the drawer_memory
  decision. `propertyCardToggleProperties(view)` returns whether it acted.

Adopt the foot row in every adapter, appended as the card's last element and outside any
collapsible body:

- **Album header:** after the schema card or mobile hint (`241x:153`, `241v`).
- **Record sheet:** after the walk rail (`242k:577-587`, Reading `242v:110-116`).
  - **Migrate** the `▸ N other properties` fold (`242j:295-345`, `242i:698-722`,
    `plugin.recordSheetFolds`) and the `added …` footer (`242j:438-442`) into the kit.
    The created stamp is then visible when the sheet is collapsed, which fixes today's
    gap.
  - Update `scripts/test-ledger-tools-record-sheet.cjs` (the fold and collapse cases
    near line 677).
- **Intention card (day/week/quarter/year):** after the rail (`296b:272-360`).
  Identity-only cards get a foot row too.
- **Bank Setup header** (`248b:417`, `248c`): per the bank_foot decision.

Each adapter declares its where-marks:

- **Record:** `parent`→kicker, `type`→tile, `title`→title when the sheet covers it,
  schema fields and ghosts→field, `created`→foot.
- **Intention:** `parent`→kicker, `type`→tile, `intentions`, `summary`, `reflection`,
  `summary_drafted`→agent, `created`→foot.
- **Album:** `type`→tile, `parent`→kicker, `emoji`→tile, `summary`→subtitle,
  `record`→schema (nested), `created`→foot.
- **Bank:** `bank`→card (nested).

Generalize the `=` gate (`248m`) into the second-key arm table of §4 and register the
`-` arm.

- Existing `==` behavior, registry API (`registerPropertyCard`), and all 29 tests must
  stay green.
- Add table tests for `-` and `+`, covering: Shift accepted for `+`, Ctrl/Alt/Meta
  rejected, count or register pending, insert/visual mode, Source mode, repeat, and IME.
- Add fall-through tests for no mounted card (`-`) and an unregistered arm (`+`).
- Add a non-US-layout case: unshifted `+`.

Also:

- **Commands:** add the palette command `toggle-card-properties` ("Toggle card
  properties") and the matching `CM_SHEET_SURFACE_ROWS` entry in bob-command-mode.
- **CSS:** add rules to `styles.css` scoped under the card roots, verified in light and
  dark themes.
- **Unit tests:** cover the formatter (every §3 row, every offset style, space
  separator, date-only, Date objects, invalid input, and local-zone rendering), the
  inventory (integer-like keys order, duplicates, managed overrides, nested maps,
  empty-value distinctions), and the DOM (aria, count/lint, collapse persistence,
  reduced motion).
- **Registration:** list every new test file in `package.json` `test:unit`.
- **Real-Obsidian e2e:** add `scripts/obsidian-card-foot-e2e.cjs` (with an offline
  `--self-test` wrapper in `npm test`) covering:
  - the foot row on all adopted species, including when the card is collapsed;
  - `=-` toggling in Live Preview with the cursor unchanged;
  - the off-screen expand scroll;
  - the `y` jump;
  - screenshots in light/dark and narrow/wide.
- **Docs:** update `docs/record-card.md`, `docs/intention-card.md`, `docs/albums.md`,
  `docs/bank.md` (if adopted), `docs/develop/architecture.md` (kit foot API, gate arm
  table), and the hotkey references. Run `npm run docs:check`.
- **Finish:** run `npm test`, then `bob plugins sync`.

Acceptance: every adopted card shows the foot row and stamp per §3, `=-` and the
disclosure toggle the drawer, `==` is unchanged, and no vault write or bob spawn happens
while painting.

## Phase `props-cli`: generic `bob props show|update` (bob-cli)

Before adding the command, read `sase/memory/cli_rules.md` and `sase/memory/docs.md`.
Releases ride the daily train (`decisions:bob-cli-releases-ride-a-daily-train`): never
edit the Cargo version or CHANGELOG, and use `feat(props):` commits. This is additive,
so there is no `!`.

**Command surface.** Add a `props` group to `src/runner.rs` in the Vault section
(alphabetical: `periods`, `props`, `query`, `records`). There is no bare default,
because `show` needs NOTE. It has two members:

- `bob props show NOTE [-f text|json]` (read-only).
- `bob props update NOTE -i FILE|- [-d] [-f text|json]` (guarded envelope writer).

For both:

- **NOTE** is a vault-relative Markdown path, with or without `.md`, or an absolute path
  inside the vault. Anything else is refused with a typed error.
- **Options** follow existing conventions (`-b` bob dir, `-d` dry run, `-f` format, `-i`
  input), sorted, each with a short alias.
- **Help** is excellent and has examples.
- **Text output** of `show` is a colored, aligned table of key, value, kind, and role.

Put the module at `src/native/props/` (`cli.rs`, `protocol.rs`, `roles.rs`, `plan.rs`,
`render.rs`, tests), reusing `yaml_spans.rs` and
`records::fingerprint::yaml_to_canonical_json`.

**`show -f json`** (`schema_version: 1`):

- top level: `ok`, `path`, `note_kind`, `capabilities: ["props_v1"]`, and `fingerprint`;
- `frontmatter: { state: ok|absent|invalid|unclosed, error?, start_line, end_line }`;
- `properties[]`, in file order, each with:
  - `key`, `line`, `end_line`;
  - `value` (JSON) and `kind`
    (text|number|checkbox|date|datetime|list|tags|aliases|map|null|opaque);
  - `style` (quoted, flow, datetime offset and separator style);
  - `role` (loose|card|identity|derived), `owner`, and `editor` (`==` for card keys);
  - `reason` (one teaching line for locked keys);
  - `duplicate` and `opaque` flags.

**Fingerprint:** `sha256` over the canonical JSON of the parsed mapping. Body edits and
formatting never stale it.

**`update` envelope:** `{ schema_version: 1, expect, ops: [...] }`, parsed strictly like
`records/protocol.rs` (duplicate keys, unknown fields, and wrong types are typed
refusals). Ops:

- `prop.set {key, value, kind?, before?|after?}` creates or replaces a value. Position
  applies to new keys; the default is the end. If the note has no frontmatter, it
  creates the block, keeping any BOM.
- `prop.unset {key}`
- `prop.rename {key, to}` changes only the key token. Value bytes and comments stay.
- `prop.move {key, before|after}` moves the full span with its attached leading comment
  lines.
- `prop.set_yaml {key, yaml, before?|after?}` takes a validated YAML value fragment for
  nested or exotic values.
- `prop.restore {key, raw, before?|after?}` restores exact bytes. It is used by
  `inverse_ops`, so undo is byte-identical.

**Serialization** (gem §4.4):

- Untouched keys keep their exact bytes.
- Quote whenever YAML would coerce or break the value: `[[wikilinks]]`; a leading
  `#@%&*!?|>-:` or backtick; `: ` or ` #` inside the value; bool/null/number/date
  lookalikes on text kinds; edge whitespace.
- Multiline text uses `|-`.
- Lists are written as block lists, except that a short single-line flow list stays
  flow.
- Datetime edits keep the existing value's offset and separator style.
- Tags are written without `#`.

**Safety:**

- **Refused for the whole note:**
  - Invalid or unclosed frontmatter: `prop_invalid_yaml`.
  - Duplicate keys: `prop_duplicate_key`, with a hint that `y` fixes it.
- **Refused per key:**
  - Anchors, aliases, merge keys, or custom tags make that key opaque: `prop_opaque`.
- **Each op's own checks** refuse with:
  - `prop_exists`
  - `prop_missing`
  - `prop_bad_key`
  - `prop_bad_value`
- **Before any write**, the candidate is verified by parsing it:
  - untouched keys must still be deep-equal;
  - each target must equal its expected value;
  - otherwise the result is `prop_verify_failed` and nothing is written.
- **Write path** (follow `periods/update.rs`):
  - take the vault lock with `ob::acquire_lock_waiting`, refusing with `lock_busy`;
  - re-read under the lock and re-check `expect`, refusing with `prop_stale`;
  - write atomically.
  - There is no git commit; vault-sync owns commits.

**Role table** (`roles.rs`, the authority, documented in the contract):

- **Note kinds:** reuse bob's existing classifiers (period/day subjects, canonical
  records, albums, bank hub, area, project, notebook, reference).
- **derived:**
  - `*_digest`, `*_drafted`, `review_*`, `zorg_*`, and `generated_from_zorg*`.
  - Period and day `parent`.
  - Reference `source_markdown`.
  - **Audit bob's frontmatter writers** to complete this list.
- **card:**
  - Record schema fields (owner `record`).
  - On period and day notes, `intentions`, `summary`, and `reflection` (owner
    `intentions`).
  - Album `record` (owner `album`).
  - Hub `bank` (owner `bank`).
- **identity:** `type`, record and album `parent`, and album `title`.
- **loose:** everything else, including `created`.
- **Refusals:**
  - `prop_owned` carries a hint naming the card editor (`==`).
  - `prop_derived` and `prop_identity` carry their reasons.

**Responses** include `fingerprint_before`, `fingerprint_after`, `inverse_ops`,
`changes` (a frontmatter line diff for previews), and the post-image `properties`.

**Catalog:**

- Add `props show` as a read row. Use a note-path slot if one exists; otherwise model it
  on the existing read rows.
- Add `props update` to `EXCLUDED` with the reason "JSON-input property edit owned by
  the Property Editor; run it from a terminal".
- Update completion so the catalog and completion tests pass.

**Tests:**

- **Golden byte-preservation fixtures:** comments; blank lines; CRLF; BOM; flow maps
  (`bank:`); flow lists; quoted and unquoted wikilinks; block scalars; integer-like
  keys; unicode keys and keys that need quoting; all offset styles; boolean-lookalike
  strings; empty values; no frontmatter; unclosed fence; duplicate keys; anchors and
  aliases.
- **Round trip:** applying every op and then its `inverse_ops` gives a byte-identical
  file.
- **Staleness:** body-only edits keep the fingerprint, and a stale `expect` refuses.
- **Locking:** under lock contention the command reports `lock_busy`.

**Docs:**

- Add a `docs/props.md` behavior contract with the JSON schemas, ops, role table, and
  refusal codes.
- Run `just docs-gen` and `just docs-check`.

Acceptance: `just` CI gates pass, every op works in dry-run and apply, and
`bob props show` on real vault notes of each kind reports sensible roles.

## Phase `owns-frontmatter`: whole-container ownership primitive and focus spike (bob-plugins)

Build the §6 ownership primitive as a kit fragment (e.g.
`240q-…-frontmatter-ownership.js`). Do not wire it to any adapter yet, so there is no
user-visible change. It includes:

- the StateField and editor attributes for Live Preview;
- the Reading-view generation-bound class;
- the CSS variables and the direct-child fallback;
- `inert` and `aria-hidden` plus the focusin guard;
- release on Source toggle, path change, release, unload, and exception;
- a `propertyCardFrontmatterReleaseAll()` registered in
  `170-plugin-lifecycle-teardown.js`.

Unit-test it next to `test-ledger-tools-title-ownership.cjs`: generation-bound release,
Source toggle in the same transaction, path change, embed and popover denial, mobile
denial, and idempotent re-own.

Then run **spike S1 (go/no-go)** as a real-Obsidian probe,
`scripts/obsidian-frontmatter-ownership-spike.cjs`, that owns a test note's container
directly and verifies:

- Vim `k` and `gg` and the Up arrow at line 0 never focus a hidden input, and typing
  continues in the body;
- Tab and Shift+Tab never enter the container;
- the title slot still positions against the hidden node;
- Mod+; while owned is observable, so `takeover` can wrap it — record the real command
  id;
- the Obsidian visibility variables apply in both modes.

Record the result and any required mitigation in the phase notes, and keep the probe
runnable. If a leak cannot be fixed in some mode, document the exact case so `takeover`
can keep per-row hiding there and flag it in the land report. Finish with `npm test` and
`bob plugins sync`.

## Phase `editor-core`: the `=+` property editor (bob-plugins)

Build §5's core rows: the modal, header, grouped rows, routing, browse keys, value
stages (including the new Checkbox, Date & time, and Tags), the add flow, delete,
undo/redo, the `y` excursion, the `=` hand-off, the close ladder, the degraded modes,
and the write discipline.

- **Fragments:** new ones (e.g. `298a…298h` plus `378-exports-property-editor.js`). Keep
  each one under 1000 lines.
- **Reuse, don't copy:** extract shared pieces of `242n`/`242q`/`242r`/`242s`/`242sa`
  into reusable helpers when reuse would otherwise mean copying. `242aa` is at 922
  lines, so do not grow it.
- **Service:** `bob props show NOTE -f json` and
  `bob props update NOTE -f json -i - [-d]` through `bobSpawnJson` and
  `createGuardedUpdateSession`.
  - The stale code is `prop_stale`; refusals come from `report.error`.
  - Detect capability from `capabilities` in the show report. An unknown subcommand
    (exit 2) means the bob is too old.
- **Undo:** a per-path stack of inverse-op batches, bounded at 20 and kept for the
  session, so undo survives reopening. Each entry carries its `expect`; a stale entry
  refuses with the hint.
- **YAML excursion:** add a `props` kind to the `242e` excursion. Today
  `albumYamlExcursionBegin` normalizes unknown kinds to `album` (`242e:183`), so extend
  it. Done reopens the editor on the same key.
- **Hand-off:** use the species open APIs (`openRecordCard`, `openIntentionCard`,
  `openAlbumSchemaEntry`, `openBankSetupCard`) with a field focus where one is
  supported, and add a `field` option where it is cheap.
- **Prose field:** register a new surface id (next free `S16`) in
  `scripts/fixtures/prose-fields.json` for the Text and List stages.
- **Entry points:**
  - the `+` gate arm (any Markdown note);
  - the palette command `edit-note-properties` ("Edit note properties") and its
    `CM_SHEET_SURFACE_ROWS` row;
  - the kit drawer hook, so a drawer row's Enter or label click opens on that key;
  - the drawer's `＋ Add property` / `✎ Edit all  =+` action line;
  - `api` exposure, so sibling plugins can open the editor.
- **Unit tests:**
  - model and routing;
  - key-action table;
  - stage parsing (date & time tokens, tags);
  - op building for every action;
  - undo-stack semantics;
  - degraded-mode states;
  - close ladder, including `q` typing inside inputs.
- **Real-Obsidian e2e:** `scripts/obsidian-property-editor-e2e.cjs` with `BOB_BIN`
  pointing at a bob built from current bob-cli master (extend
  `scripts/fixtures/fake-bob.cjs` for the offline self-test). Cover:
  - edit, add, delete, and undo on a project note and a day note;
  - hand-off from a record field;
  - a locked derived key;
  - an old-bob read-only mode;
  - invalid YAML;
  - byte preservation of untouched keys on disk;
  - light/dark and narrow/wide screenshots.
- **Docs:** add a new `docs/property-editor.md` reference (keymap, routing, degraded
  modes), and update the cross-links from the card docs and the hotkey docs. Run
  `npm run docs:check`.
- **Finish:** run `npm test`, then `bob plugins sync`.

Acceptance: on any note, `=+` can view every key, edit every loose key's value with a
fitting stage, add a key with vault name and type completion, delete, and undo or redo.
Card and derived keys route as in the table. Every write goes through `bob props`.

## Phase `editor-parity`: rename, reorder, multi-select, copy/paste, filter, kind change, YAML stage (bob-plugins)

Complete native parity and go beyond it, using the "parity" rows of §5's grammar.

- **Rename** (`r`): an inline key stage with a duplicate guard, through `prop.rename`.
- **Reorder** (Alt+J/K): through `prop.move`, with a 120 ms row slide and no motion
  under reduced-motion. Reorder is disabled while a filter is active.
- **Multi-select:**
  - Space marks a row, `V` starts a visual range extended by `j`/`k`, and Shift+↑/↓
    extends.
  - A marks gutter shows the selection.
  - Ctrl+D deletes all marked rows as one undoable batch.
- **Copy** (Mod+C): copies the marked rows, or the current row, as YAML using the source
  bytes.
- **Paste** (Mod+V): parses clipboard YAML and shows an overwrite preview listing which
  keys are new and which are replaced. Enter applies it as one batch of `prop.set_yaml`
  ops; locked and card keys are listed as skipped.
- **Filter** (`/`): an incremental filter over keys and rendered values. Enter or ↓
  returns to the rows, and Esc clears the filter before closing.
- **Kind change** (`t`):
  - Converts this value's kind (e.g. text→list, text→date) with a preview of the
    converted value.
  - The vault-wide property type changes only through a **feature-detected**
    `app.metadataTypeManager`, behind an explicit confirm line that names how many notes
    use the key.
  - Converting `created` to datetime adds a warning that Obsidian's datetime widget
    strips offsets.
  - Without the type manager, the editor teaches and defers to the All Properties
    sidebar.
- **YAML stage:** for map and opaque values, a validated multi-line stage. bob's dry-run
  is the validator, and the stage writes through `prop.set_yaml`.

Tests:

- **Unit:** the key-action table, selection model, paste-plan building, and conversion
  previews.
- **e2e:** extend the editor e2e with rename, reorder, a batch delete with undo, a paste
  overwrite, a filter, and a kind change.

Update `docs/property-editor.md`, adding a parity table against Obsidian's native
Properties. Finish with `npm test` and `bob plugins sync`.

Acceptance: everything native Properties can do in-document (add, edit, rename, delete,
reorder, copy, paste, multi-select, change type) is doable from the keyboard in `=+`.

## Phase `takeover`: hide native Properties on card notes (bob-plugins, plus memory if approved)

Wire `propertyCardOwnFrontmatter` into the album, record, and intention adapters after a
successful mount, under §6's eligibility rules.

- **Bank:** excluded, keeping its `bank` row ownership.
- **Per-row hiding (`240b`) stays** as the fallback layer. It is used:
  - while `props_v1` is unconfirmed;
  - in any mode the `owns-frontmatter` spike flagged.
- **Capability probe:** add the cached, off-paint-path `props_v1` probe.
- **Mod+;:** wrap the Add file property command, using the id recorded by the spike.
- **Focus helpers:** reroute native-focus helpers (audit every `.metadata-property`
  focus and `metadataEditor` use across the plugins) to `=+` when the scope is owned.
- **Identity-only day cards** become eligible.

Run a real-Obsidian fail-open matrix (`scripts/obsidian-frontmatter-takeover-e2e.cjs`,
with a self-test). It covers:

- hidden in Live Preview and Reading on day, week, record, and album notes;
- restored by:
  - Source toggle;
  - an embed;
  - a hover popover;
  - mobile emulation;
  - bad YAML;
  - an unclosed fence;
  - a path change in the same pane;
  - injected mount failure;
  - plugin disable and unload;
  - an absent capability;
- Mod+; opens `=+` in add mode;
- no focus leak (re-run the S1 checks);
- the File Properties sidebar still works;
- a full light/dark × narrow/wide screenshot walkthrough of day, week, record, album,
  bank, and a non-card note with `=+`.

Docs:

- `docs/record-card.md`, `docs/intention-card.md`, `docs/albums.md`,
  `docs/develop/architecture.md` (a third ownership bit and the eligibility rules), and
  `docs/develop/validation.md`.
- Retire the "native Properties and Add property stay visible" requirements in those
  pages.
- Note that Metadata Menu's Properties context menu is gone on owned notes.

Run `npm test`, then `bob plugins sync`.

> [!decision] memory_glossary_cards In bob-cli, use `/sase_memory_write` to rewrite two
> glossary strands, then run `sase memory init`.
>
> - **Property Card:** replace "owned-key property hiding" with what the kit now owns:
>   - the whole frontmatter (gated, fail-open; bank excluded);
>   - the foot row, created stamp, and `=-` drawer;
>   - the `=` chord family (`==`, `=-`, `=+`).
>
>   Keep the strand concise and link `docs/reference/plugins/property-editor.md`.
>
> - **Record Card:** say the card replaces native Properties when eligible and that
>   every key is in the drawer and `=+`. Drop "above native Properties" and the sentence
>   about keys staying in Properties.

> [!decision] memory_decision_record Add an accepted `decisions:` strand, "Property
> cards own in-document Properties; one writer per key". It records:
>
> - painting is read-only and fail-open;
> - loose keys are written via `bob props`, card keys via their `==` editor, and derived
>   keys are locked;
> - `processFrontMatter` and JS YAML writers are rejected;
> - evidence: this plan, the research report, and the commits;
> - it reopens if Obsidian exposes a byte-preserving frontmatter API.
>
> Then run `sase memory init`.

Acceptance: on eligible card notes, the native Properties block is gone in Live Preview
and Reading. Every failure path above restores it in the same frame. Nothing in-document
is ever a dead end.

## Cross-cutting rules for every phase

- Open bob-plugins with `/sase_repo` (it's a linked repo), read its `AGENTS.md`, and
  finish plugin changes with `npm test` and `bob plugins sync`. Never hand-edit a
  generated `main.js`.
- Phases that run in parallel (`foot-drawer`, `props-cli`, `owns-frontmatter`) should
  keep shared touch points additive: `src/fragments.json`, `package.json` `test:unit`,
  `styles.css`, and `170-plugin-lifecycle*.js`. Rebase cleanly.
- Every modal and panel follows `decisions:q-and-ctrl-bracket-mirror-esc` through
  `bobCardWireCloseKeys`.
- Every e2e that needs `bob props` uses a bob built from current bob-cli master via
  `BOB_BIN`. Plugins must behave honestly against a released bob that lacks `props_v1`.

## Out of scope (file task beads with `/sase_new_task` if still wanted)

- **Backfilling `created`** on records and periods. Research says never fabricate it; an
  opt-in migration from git history would be a separate piece of work.
- **One canonical offset style** for _new_ `created` writes across bob writers (`%z` vs
  `%:z`).
- **Title-block "note cards"** for area, project, notebook, and reference notes, which
  are the long-term end state for taking over those notes' Properties.
- **Reading-view `=` chords.**
