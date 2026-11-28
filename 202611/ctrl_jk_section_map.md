---
tier: epic
title: Ctrl+J / Ctrl+K Section Map
goal: 'Every successful Ctrl+J / Ctrl+K section jump in Obsidian briefly shows a passive,
  beautiful Section Map in the jumping pane: the note''s headings, where you are,
  where you came from, where the next press lands, and when a press wrapped. The map
  is built from the same scanner and jump function the keys use, so it never disagrees
  with them, and it never takes focus, keys, or clicks.

  '
decisions:
  peek_lifetime:
    ask: How long should the unbound "Show section map" peek stay on screen?
    choices:
      pinned: Stays until your next key, edit, cursor move, or pane change; no fade
        timer
      linger: Fades 2 s after showing, exactly like a jump (the research default)
    default: pinned
    why: A deliberate peek is for reading, and any key still dismisses it at once
    answer: pinned
  section_map_record:
    ask: Add a decisions strand saying section jumps show a passive, gesture-scoped
      Section Map?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: model
  title: Shared heading scanner and Section Map model
  depends_on: []
  size: medium
  description: 'model: refactor the section-header scanner into one walker that the
    keys and the map share, then add the pure Section Map model (labels, nesting depth,
    windowing, chips, counter, aria sentence) with unit and parity property tests.'
- id: card
  title: Section Map card painter, placement, and styles
  depends_on:
  - model
  size: medium
  description: 'card: build the pure DOM painter with keyed in-place reconciliation,
    the margin/overlay/compact placement function, the .bob-section-map styles, and
    a headless-Chromium preview whose screenshots are reviewed against the visual
    spec.'
- id: wiring
  title: Jump integration, lifecycle controller, landing flash, and peek
  depends_on:
  - model
  - card
  size: medium
  description: 'wiring: move the section jump into a new fragment, fix the double
    scroll, add the landing flash, the hold/linger/fade lifecycle with activity dismissal,
    the doc-keyed heading cache, the aria-live sentence, and the unbound peek command.'
- id: ship
  title: Real-Obsidian acceptance, docs, and deploy
  depends_on:
  - wiring
  size: medium
  description: 'ship: prove the map end to end in real Obsidian over CDP (both key
    routes, lifecycle, dismissal, wrap, windowing, compact, unchanged note bytes),
    document it in the navigation plugin reference, and deploy with bob plugins sync.'
proposed_by: bbugyi200.athena.1hg
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 15:01:41
status: done
bead_id: bob-cli-rc
---

- **PROMPT:** [prompts/202611/ctrl_jk_section_map.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/ctrl_jk_section_map.md)
- **BEAD:** [bob-cli-rc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-rc/README.md)

# Plan: Ctrl+J / Ctrl+K Section Map

## Context

Bryan jumps between Markdown headings with Ctrl+J / Ctrl+K and loses his place: the
landing pins the heading to the top of the viewport (scrolling the old context away),
the target list is flat across H1–H6, wrap-around at either end is silent, and nothing
shows position or what comes next.

The consolidated research report is the design brief. Every worker reads it first:

```bash
sase artifact read research:202611/ctrl_jk_section_hud/ctrl_jk_section_hud.md "Section Map design brief"
```

The research sidecar also holds the rendered mockup
(`202611/ctrl_jk_section_hud/ctrl_jk_section_map_mockup__cld.png`; panels A, B, and C
are the target look). Bryan accepted all ten requirement adjustments in that report.
This plan carries them out and adds the design decisions below. Where this plan and the
report differ, this plan wins.

### Where the work lives

- All code is in the **bob-plugins** linked repo, plugin
  `plugins/bob-navigation-hotkeys`. Open it with your `/sase_repo` skill
  (`sase repo open bob-plugins -r "<why>"`). Read its `AGENTS.md` before editing.
- That plugin uses the committed fragment build. Edit `src/` fragments, register new
  ones in `src/fragments.json` (array order is source order), run `npm run build`, and
  never hand-edit `main.js`. Every hand-edited fragment must stay at or under 1000
  lines. `610-plugin-jumps-and-counted-keys.js` is at 962 lines and `700-exports.js` at
  999, so new code goes in **new fragments**, and new test exports go in a new
  `747-section-map-exports.js` (pattern: `746-snooze-card-exports.js`).
- Plugin methods come from mixin classes installed in `690-install-plugin-mixins.js`,
  which throws on duplicate method names. Moving `jumpToSectionHeader` therefore means
  deleting it from the 610 mixin.
- No bob-cli change, no JSON contract, and no release-train involvement.
- Every phase ends with the repo's gate (`npm run build`, `npm run build:check`,
  `npm run docs:gen`, `npm run docs:inventory:check`, `npm test`) plus
  `bob plugins sync`. New test files go in the explicit `test:unit` list in
  `package.json`. Follow the version-bump convention of recent bob-navigation-hotkeys
  commits (manifest, `docs/index.md`, plugin page).

### Governing decisions

- `decisions:q-and-ctrl-bracket-mirror-esc`: any panel that Esc closes must also close
  on `q` (consumed outside text controls) and `Ctrl+[`. The Section Map is strictly
  passive: no focus, no key handling, no consumed keys, `pointer-events: none`. That
  keeps it outside this rule, so it never steals `q` from Vim macro recording. Activity
  dismissal covers Esc without binding it. If a later version adds focus or clicks, it
  then owes the full panel key contract.
- `decisions:successor-jump-parks-the-walk`: other surfaces own Ctrl+J too (the
  successor-picker pin, schema and bank field jumps, the insert-mode newline). The map
  hooks the end of the **commands** `jump-to-{next,prev}-section-header` and never
  listens for the literal Ctrl+J / Ctrl+K chord, so those owners are untouched.

### Facts verified in bob-plugins@6827a275

- Commands: `490-plugin-lifecycle.js:427-436` registers both jump commands as
  `editorCallback: (editor) => this.jumpToSectionHeader(editor, ±1)` with no default
  hotkey. The vault binds them twice: vimrc `nnoremap <C-j> :bob_next_header<CR>` (exmap
  → `obcommand`) for Vim normal mode, and `hotkeys.json` Ctrl+J / Ctrl+K for insert and
  non-Vim contexts. Ctrl is the real Ctrl key on every platform.
- Scanner: `getSectionHeaderLines` / `getSectionHeaderJumpLine` in
  `130-fences-and-status.js:13-81` (840 lines), with `SECTION_HEADER_RE` in
  `010-requires-and-config.js:20`. Targets are ATX H1–H6 (up to three leading spaces; a
  bare `#` counts), skipping frontmatter and fenced code. Next is the first heading
  strictly after the cursor and previous is the last strictly before it. Both wrap.
- Jump: `jumpToSectionHeader` in `610-plugin-jumps-and-counted-keys.js:178-205` calls
  `setEditorCursor` (which itself calls `editor.scrollIntoView`) and then
  `scrollEditorLineToTop`. That is the double-scroll frame.
  `setEditorCursorWithoutScroll` already exists (`170-open-task-jump.js:673`).
- Landing flash: `bobOpenNotesLandingFlashEffect`/`Field` and
  `flashBobOpenNotesLandingLine` in `676-plugin-open-notes-uri.js` (CSS
  `.cm-line.bob-landing-flash` in `styles.css:129-150`, 1.2 s, reduced-motion variant).
  The flash reads the global `window.matchMedia`, not the editor's own window.
- Existing precedents to copy: `473-window-hint-view.js` (a pure painter that reuses one
  DOM tree and never throws), `scripts/preview-window-prefix.cjs` (a headless Chromium
  preview of the production painter plus `styles.css`),
  `scripts/test-navigation-window-hint-view.cjs` (a fake-DOM painter test), the
  `scripts/obsidian-window-prefix-e2e*.cjs` runners, the shared e2e scratch helper, and
  the multi-window listener targets in `687-plugin-window-prefix.js`.
- The plugin has no settings tab, and v1 adds none.

### Not in v1 (deferred, per the research)

- Headings hidden by bob-ledger-tools decorations or by folding (needs its own decision,
  because fixing it changes which lines Ctrl+J visits).
- Counted `3<C-j>` jumps.
- `<C-o>` burst history.
- Click-to-jump or a focusable map.
- Numbered row leaps.
- Reading-view parity.
- Mobile.
- A display preference or off switch.
- Any change to scanner scope (HTML-comment headings, Setext, callouts).

## Design

This section is the shared spec. Every phase builds against it.

### What it looks like

```text
┌────────────────────────────────────────────┐
│ § SECTIONS              ↻ wrapped   5 / 12 │  header
│    Bob Mac Capture                         │
│    │ Goals                                 │  depth guides
│    │ Design                                │
│  ○ │   Capture grammar                 ⌃K  │  ring = section you just left
│ ┃▒▒│▒▒▒Live preview▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│  pill + 3 px rail = you are here
│    │   Completion                      ⌃J  │  chips = where the next press lands
│    │ Open questions                        │
│      ⋯ 3 more                              │  exact gap count (windowed notes)
└────────────────────────────────────────────┘
```

Compact strip (narrow panes), docked at the bottom of the pane:
`⌃K Capture grammar  ·  ┃ Live preview 5/12  ·  Completion ⌃J`

### Content rules

- **Header:** an accent `§` and a small-caps `SECTIONS` label. On the right: a
  `↻ wrapped` pill when this press wrapped, then the counter `5 / 12` (tabular numerals;
  the current number is strong, `/ 12` is faint). With no current section the counter
  reads `– / 12`. A single-heading note shows `only section` in place of the wrap pill.
- **Rows:** one per heading in document order. Indent comes from real nesting depth (an
  outline stack, so H1 → H3 indents one step), clamped at 4 steps. The label cleaner
  applies these rules in order:
  - Strip the closing `#` run, `%%comments%%`, and a trailing ` ^block-id`.
  - Unwrap `[[target|alias]]` to `alias`, and `[[target]]` to `target` without `.md`.
  - Unwrap `[text](url)` to `text`. Turn `![alt](…)` / `![[…]]` into `alt` or nothing.
  - Remove paired emphasis delimiters (`**`, `__`, `*`, `_`, `~~`, `==`) and backticks,
    but never intraword underscores.
  - Strip inline HTML tags and collapse whitespace.
  - Keep `#tags` and emoji, because they are landmarks.
  - If nothing is left, show _(untitled)_.

  All text goes in through `textContent` only.

- **Identity:** a row's key is its heading **line**, never its text, so duplicate
  `## Notes` rows stay distinct.
- **Current row:** the section that contains the cursor (for a jump, that is the landed
  heading). It gets an accent pill, a 3 px accent rail, and semibold text. It is never
  truncated and may wrap to 2 lines.
- **From row** (jumps only): a hollow accent ring on the section the cursor was in
  before this press, drawn only when it differs from the current row. Ctrl+K from a
  section body lands on that section's own heading, so no ring appears.
- **Chips:** `⌃K` / `⌃J` keycaps on the rows that `getSectionHeaderJumpLine(±1)` returns
  from the cursor. They are always computed by the real jump picker, never by index ± 1.
  A chip whose press would wrap shows accent ink plus `↻`. A chip whose press would not
  move (the target is the cursor line) is omitted. A row can carry both chips (`⌃K`
  first). Use `⌃J` / `⌃K` on every platform, because the bindings are Ctrl everywhere.
- **Windowing:** up to `SECTION_MAP_FULL_ROWS = 12` headings (or fewer if the pane
  height allows fewer) show the full outline; that covers 97% of real notes. Above that,
  the map shows a contiguous window of `SECTION_MAP_WINDOW_ROWS = 8`, sized by
  direction:

  | Direction | Rows before current | Rows after current |
  | --------- | ------------------- | ------------------ |
  | Down (+1) | 2                   | 5                  |
  | Up (−1)   | 5                   | 2                  |
  | Peek      | 3                   | 4                  |

  The window slides to stay inside the note, and starts at 0 when there is no current
  row. On top of the window, the map pins:
  - up to 2 ancestors of the current row from outside the window (the outermost and the
    innermost);
  - any chip target outside the window (wrap targets), so both chips are always visible.

  Gaps between segments become `⋯ N more` rows with exact counts. The result is
  deterministic from (headings, cursor, direction).

- **Fisheye:** current, chip-target, and from rows use `--text-normal`. Other rows use
  `--text-muted`, fading with heading-index distance (opacity floor 0.55). Gap rows use
  `--text-faint`.
- **Edge states:**
  - One heading: `1 / 1 · only section` and no chips (an honest no-op, no wrap pill).
  - Zero headings: keep today's `Notice`, and no card.
  - Cursor above the first heading (peek): no current row.

### Placement

- The map is mounted in the jumping `MarkdownView`'s own content element and built with
  that element's `ownerDocument`, so pop-out windows work.
- There is one map plugin-wide. It belongs to the pane that ran the command and moves
  when another pane jumps.
- Placement is chosen once per visible sequence from rects, by a pure function, and
  recomputed only on pane resize or when the map moves to another pane. It never chases
  the caret, and the card never moves between presses.

The three modes, in order of preference:

1. **Margin:** used when the right gutter beside the readable-line-width text column is
   at least `220 + 2 × 16` px. The card sits in that gutter, centered, with width
   `min(300, gutter − 32)`. Its top is 12 px below the editor scroller's top, which is
   level with the landed heading. It covers no text.
2. **Overlay:** used when there is no gutter. The card sits at the top-right of the
   scroller with a 12 px inset (clear of the scrollbar), width
   `clamp(220px, 30%, 300px)`. It is translucent with backdrop blur. Top-right is right
   because the landed heading is pinned top-left.
3. **Compact:** used when the pane is narrower than 480 px or shorter than about 5 rows.
   A one-line pill strip docks 12 px above the pane's bottom edge, so it never covers
   the landed heading at the top.

- `maxRows` comes from the available height and feeds the model.
- If a visible Obsidian `Notice` intersects the card, nudge the card's top below it
  once, at placement time.
- Z-order: above CodeMirror's text, caret, and selection layers. Below modals, menus,
  notices, and tooltips.

### Lifecycle

| State           | Entered by                            | Leaves when                                                              |
| --------------- | ------------------------------------- | ------------------------------------------------------------------------ |
| Hidden          | start, dismissal, end of fade         | a successful jump or peek → Holding / Lingering / Pinned                 |
| Holding         | a jump while Ctrl is physically down  | Ctrl keyup, window blur, or the 6 s cap → Lingering                      |
| Lingering       | Ctrl released, or a jump with Ctrl up | 2.0 s with no jump → Fading; another jump → Holding or Lingering         |
| Pinned (peek)   | the peek command, per `peek_lifetime` | only activity dismissal                                                  |
| Fading (180 ms) | linger expiry                         | fade done → Hidden; another jump → visible again with no entrance replay |

- **Activity dismissal** (from any visible state, with a 90 ms opacity-only fade):
  - a document change in the map's editor;
  - a selection change whose head line differs from the map's anchor line (the landing
    line, or the peek cursor line);
  - a non-modifier `keydown` (capture phase, on every window) that is not followed by a
    section jump or peek in the same dispatch — checked after the event's dispatch
    completes, so held Ctrl+J auto-repeat never flickers;
  - a `pointerdown` anywhere;
  - `active-leaf-change` to another leaf, `file-open` in the map's leaf, a layout change
    that detaches the host, or plugin unload.
- **Not a dismissal:** mouse-wheel scrolling (the map describes the cursor, not the
  viewport), and window blur (blur only ends Holding).
- Every jump bumps a generation. Timers from older generations are no-ops. Navigation is
  never debounced; only painting is coalesced to the latest model per animation frame.
- Timing constants (`SECTION_MAP_LINGER_MS = 2000`, `SECTION_MAP_HOLD_CAP_MS = 6000`,
  `SECTION_MAP_FADE_MS = 180`, `SECTION_MAP_DISMISS_MS = 90`) are named constants, meant
  to be tuned after a week of real use.

### Visual and motion spec

Use Obsidian theme variables only, so the map is correct in light and dark with no new
palette.

- **Surface:** `color-mix(in srgb, var(--background-secondary) 92%, transparent)` with
  `backdrop-filter: blur(10px)`. Border is 1 px `--background-modifier-border`, radius
  `--radius-l`, shadow `--shadow-s`, padding 8 px.
- **Rows:** 24 px tall in `--font-ui-smaller`. Indent is 12 px per depth step, with 1 px
  depth guides in `--background-modifier-border`.
- **Current-row pill:** one absolutely positioned element behind the rows, filled with
  `color-mix(in srgb, var(--interactive-accent) 16%, transparent)`, with a 3 px
  `--interactive-accent` rail and radius 6 px.
- **From ring:** a 7 px hollow ring with a 1.5 px accent border, in the left gutter.
- **Chips:** right-aligned keycaps in monospace at about 10.5 px. Border 1 px
  `--background-modifier-border` on `--background-primary`, ink `--text-muted`. Wrap
  chips use accent border and ink plus `↻`.
- **Untitled rows:** italic `--text-faint`.
- **Motion:**
  - entrance only from Hidden: 90 ms ease-out, opacity plus 4 px drop;
  - the pill slides from the old row to the new row with a 140 ms
    `cubic-bezier(.2,.7,.2,1)` transform, retargeting smoothly under auto-repeat;
  - on a wrap, or when either row is not painted, a 120 ms pill crossfade replaces the
    slide (a full-list slide would read as a scroll), and the `↻ wrapped` pill pops in
    120 ms;
  - exit 180 ms.
- **Not allowed:** per-press entrance replay, countdown or progress bars, and
  smooth-scrolling the editor.
- **Reduced motion** (read from the host window's `matchMedia`): opacity only, with no
  transforms.
- **Landing flash:** the existing accent `.bob-landing-flash` line decoration on the
  landed heading line, so the page and the map share one mark.

### Accessibility

- A visually hidden `role="status"` / `aria-live="polite"` / `aria-atomic="true"`
  element carries one coalesced sentence once rapid presses settle (about 400 ms after
  the last jump). Example: "Section 5 of 12, Live preview; next Completion; previous
  Capture grammar", plus "; wrapped to the top" when the press wrapped.
- The painted rows are `aria-hidden`.
- Current-row identity is carried by rail, weight, and pill, never by color alone.
- The peek command is the untimed route.

### Reliability rules

1. **Jump first, map second.** The map runs in `try/catch` after the cursor and scroll
   succeed. A map failure never blocks, alters, or undoes a jump, and never becomes a
   toast.
2. **One scan per press, one source of truth.** The same heading list picks the target
   and builds the model, and chips call the real picker. A parity property test enforces
   this.
3. **No stale map.** Any document change hides the map. Heading entries are cached per
   editor in a `WeakMap` keyed by the immutable CM6 `state.doc` (never by path or
   mtime). Generic editors without CM rescan.
4. **Pane-local singleton with full teardown.** Every listener, observer, timer, and
   node is released through `register*` on unload. Access to `editor.cm` stays
   feature-detected. No vault writes, settings writes, or network.
5. **Scanner scope is unchanged.** The map shows exactly the set of lines the keys
   visit.

## Phase: model

**Goal:** one heading walker shared by the keys and the map, plus a pure, DOM-free,
Obsidian-free Section Map model.

1. **Scanner refactor** (`130-fences-and-status.js`):
   - Extract `scanSectionHeadings(lines)`, which returns `[{ line, level, text }]` from
     the exact walk `getSectionHeaderLines` does today.
   - `getSectionHeaderLines(lines)` becomes
     `scanSectionHeadings(lines).map((h) => h.line)`.
   - Split `getSectionHeaderJumpLine(lines, cursorLine, direction)` so it delegates to a
     new `pickSectionHeaderJumpLine(headerLines, cursorLine, direction)`.
   - Behavior must not change.
2. **New `131-section-map-model.js`:**
   - `cleanSectionHeadingLabel(text)`, following the content rules.
   - `getSectionHeadingEntries(lines)`, which adds `depth` (outline stack, clamped at
     4), `label`, and `untitled` to the scanned headings.
   - `findSectionIndexAtLine(entries, line)`.
   - `buildSectionMapModel({ entries, cursorLine, fromLine, direction, wrapped, source, maxRows })`,
     which returns the frozen view model below.
   - Named constants for the row budgets.
3. **Exports:** create `747-section-map-exports.js` and expose the new helpers to the
   test harness.

View-model contract (the `card` and `wiring` phases depend on it; keep it stable):

```js
{
  version: 1,
  source: "jump" | "peek",
  direction: 1 | -1 | 0,
  total, currentIndex /* -1 when none */,
  counter: { current: "5" | "–", total: "12" },
  wrapped, only, windowed,
  rows: [
    { kind: "heading", key: `h:${line}`, line, index, depth, label, untitled,
      current, from, pinned: "ancestor" | "wrap" | null, distance,
      chips: [{ direction: -1 | 1, wrap }] },
    { kind: "gap", key: `gap:${firstIndex}-${lastIndex}`, count },
  ],
  prev: { line, index, label, wrap } | null,
  next: { line, index, label, wrap } | null,
  ariaLabel,
}
```

**Tests:**

- New `scripts/test-navigation-section-map-model.cjs`:
  - label-cleaning table (every rule, odd vault shapes, an `<img onerror>` label stays
    inert text);
  - depth with skipped levels;
  - counter and edge states (0, 1, and 2 headings, cursor above the first heading);
  - from-ring rules (including Ctrl+K from a body);
  - chip wrap and omission;
  - windowing for 42 headings in each direction, with pinned ancestors, pinned wrap
    targets, and exact gap counts;
  - duplicate labels;
  - determinism;
  - aria sentences.
- New `scripts/test-navigation-section-map-parity.cjs`:
  - a seeded-PRNG property test (no new dependencies) over random documents with
    frontmatter, fences (backtick and tilde, longer closers), indented and bare `#`
    headings, and heading-shaped lines in fences;
  - for every cursor line (plus −1 and past the end), the model's chip targets equal
    `getSectionHeaderJumpLine(lines, cursor, ±1)`;
  - `getSectionHeaderLines` equals a frozen copy of the pre-refactor implementation kept
    in the test as an oracle.
- `scripts/test-navigation-hotkeys-section-nav.cjs` stays green unchanged.

**Done when:** the gate passes, the code is synced, and the model needs no DOM.

## Phase: card

**Goal:** a pure painter and placement function that make the model look like the mockup
in light and dark, at every width.

1. **New `474-section-map-view.js`** (after `473-window-hint-view.js`; split it into
   `474a-…` if it approaches the cap). Copy 473's helper style: it works with Obsidian's
   `createEl` and a fake DOM, and it never throws.
   - `computeSectionMapPlacement({ host, scroller, textColumn, notices })` returns
     `{ mode: "margin" | "overlay" | "compact", left, top, width, maxRows }` according
     to the placement rules.
   - `renderSectionMapCard(state, model, placement, { reducedMotion })` returns the new
     `state`.
   - Rendering reconciles rows in place, keyed by `row.key`: it reuses nodes, reorders
     them with `insertBefore`, and removes stale ones.
   - It positions the single pill element by transform, and chooses slide versus
     crossfade (wrap, or the old/new row not painted).
   - It toggles the entrance class only from hidden, and renders the compact strip.
   - It writes text only through `textContent`/`setText`.
   - It sets `aria-hidden` on the visual rows and owns the live-region node (the
     controller sets its text).
2. **`styles.css`:** a `.bob-section-map` block that implements the visual and motion
   spec, uses theme variables only, sets `pointer-events: none`, and adds a
   `prefers-reduced-motion` variant next to the existing flash rules.
3. **New `scripts/test-navigation-section-map-view.cjs`** (fake DOM; pattern:
   `test-navigation-window-hint-view.cjs`). It covers:
   - placement mode boundaries;
   - node identity kept across renders for the same keys;
   - pill target and slide versus crossfade;
   - chip and ring classes;
   - gap rows;
   - compact content;
   - entrance not replayed on update;
   - inert hostile labels;
   - the painter never throwing on malformed models.
4. **New `scripts/preview-section-map.cjs`** (pattern: `preview-window-prefix.cjs`). It
   renders the production painter, real models from the `model` phase, and `styles.css`
   in headless Chromium with stubbed Obsidian theme variables. Scenes:
   - wide margin mode with ring and chips (dark);
   - a 42-heading windowed note in overlay mode;
   - a wrap on a 3-heading daily note (light);
   - the compact strip;
   - the only-section state;
   - odd and untitled labels;
   - a mid-slide frame;
   - reduced motion.

   Look at every PNG yourself and iterate until it matches this spec and the research
   mockup: aligned baselines, no clipped chips, a readable fisheye, a calm surface in
   both themes. Document the command in `docs/develop/validation.md` beside the
   window-prefix preview.

**Done when:** the gate passes, the code is synced, and the preview screenshots match
the spec. The painter is not wired to commands yet.

## Phase: wiring

**Goal:** the keys show the map, and the lifecycle is exactly the table above.

1. **New `611-section-map-lifecycle.js`:** a pure state machine with injected clock and
   timer functions. It has the states from the table, a generation guard, the hold cap,
   linger, fade, dismissal, and the peek behavior below. No DOM.
2. **New `612-section-map-controller.js`** (mixin):
   - host resolution from the command's `ctx` (a `MarkdownView` whose editor is this
     editor, otherwise the active `MarkdownView` with the same editor, otherwise no
     map);
   - mount and move of the singleton card, and `ResizeObserver` re-placement;
   - a passive Ctrl tracker (capture `keydown`/`keyup` plus `blur` on every window,
     reusing the window-prefix multi-window target pattern);
   - the activity-dismissal listeners;
   - one `EditorView.updateListener` registered through `registerEditorExtension`, which
     handles doc changes and foreign selection changes. The jump's own selection update
     must never dismiss: arm the anchor line before dispatching the cursor;
   - the per-frame paint coalescing and the debounced aria sentence;
   - the `WeakMap` heading cache keyed by `state.doc`;
   - full teardown.
   - Skip the map on `Platform.isMobile`; the jump itself is unchanged there.
3. **New `613-plugin-section-jump.js`** (mixin):
   - Move `jumpToSectionHeader(editor, direction, ctx)` here and delete it from 610.
   - Order of work: one scan; then the picker; then `wrapped` (only when there are two
     or more headings and the target is not strictly past the cursor in the travel
     direction); then arm the controller; then `setEditorCursorWithoutScroll` and
     `scrollEditorLineToTop`; then the landing flash; then the map, inside `try/catch`.
   - Keep the old `editor.scrollIntoView` path only as the fallback when
     `scrollEditorLineToTop` returns false.
   - Zero headings and no editor keep today's exact Notices and return values.
   - Add `showSectionMap(editor, ctx)` for the peek. It never moves the cursor, and it
     shows the Notice "No section headers" when the note has none.
4. **Commands** (`490-plugin-lifecycle.js`):
   - Pass `ctx` through:
     `editorCallback: (editor, ctx) => this.jumpToSectionHeader(editor, ±1, ctx)`.
   - Register `show-section-map` ("Show section map", no default hotkey).
   - Add the new mixins to `690-install-plugin-mixins.js` and the fragments to
     `fragments.json`.
5. **Landing flash:** generalize `flashBobOpenNotesLandingLine` into a shared
   `flashEditorLandingLine(editor, line)` that reads reduced motion from the editor's
   own window. `obsidian://bob-open-notes` keeps identical behavior through it.

> [!decision] peek_lifetime = pinned The peek enters **Pinned**. It has no timer and
> leaves only through activity dismissal (any key, edit, foreign cursor move,
> pointerdown, or pane/file change). A jump made from a pinned peek follows the normal
> jump lifecycle.

> [!decision] peek_lifetime = linger The peek enters **Lingering**, or **Holding** if
> Ctrl is down, exactly like a jump, and fades 2.0 s later.

**Tests:**

- New `scripts/test-navigation-section-map-lifecycle.cjs` (fake clock):
  - Holding → Lingering → Fading → Hidden timings, the 6 s cap, and blur;
  - a repeat during Fading restores without entrance;
  - an old-generation timer is a no-op;
  - each dismissal reason;
  - the peek per the decision.
- New `scripts/test-navigation-section-map-jump.cjs` (harness editor and CM fakes):
  - every case in the section-nav test lands identically;
  - no `editor.scrollIntoView` call when CM exists, and the fallback scroll when it does
    not;
  - `wrapped` correctness;
  - a flash effect on the landing line;
  - a throwing painter leaves the cursor, the return value, and the Notices unchanged;
  - the zero-heading Notice is unchanged;
  - the peek does not move the cursor;
  - the jump's own selection update keeps the map, a later different-line selection
    dismisses it, and a doc change dismisses it;
  - a non-modifier keydown with no jump dismisses after dispatch, while a keydown
    followed by a jump keeps the map;
  - mobile skips the map.
- `test-navigation-hotkeys-section-nav.cjs` and the open-notes flash tests stay green.

**Done when:** the gate passes and the plugin is synced. Real-Obsidian proof belongs to
the `ship` phase.

## Phase: ship

**Goal:** prove it in real Obsidian, then document and deploy it.

1. **New `scripts/obsidian-section-map-e2e.cjs`** (plus a `-lib` if needed), built on
   the shared e2e scratch helper and a scratch vault that never touches `~/bob`, with an
   offline self-test wrapper `scripts/test-obsidian-section-map-e2e.cjs` added to
   `npm test`. Drive real keys over CDP:
   - **Routes:** both the Vim normal-mode route and the insert / non-Vim `hotkeys.json`
     route. The fixture has no vimrc-support, so emulate the vault's
     `exmap … obcommand` + `nnoremap <C-j>` / `<C-k>` with the CodeMirror Vim API, and
     record that in the report.
   - **Lifecycle and dismissal:**
     - a single tap → card visible → gone about 2.2 s after Ctrl release;
     - held Ctrl+J auto-repeat updates the same card node in place with no flicker;
     - edit, `j`/`k`, Esc, and a click each dismiss, and Esc/`q` still do their normal
       Vim work;
     - a wheel scroll does not dismiss;
     - a pane switch dismisses.
   - **Content:**
     - a wrap shows `↻ wrapped`;
     - each chip's row equals where the next real press lands;
     - a 42-heading note windows correctly;
     - a narrow split shows the compact strip;
     - margin mode applies with readable line length on.
   - **Peek:** follows `peek_lifetime`.
   - **Safety:** note bytes are unchanged (hashes), the console has no exceptions, and
     screenshots are taken in light and dark.

   Review every screenshot. Fix and tune what the runner finds, including lifecycle
   constants only when a check proves them wrong. Document the run command in
   `docs/develop/validation.md`.

2. **Docs:**
   - In `docs/bob-navigation-hotkeys.md`, add a "Section map" subsection: what it shows,
     placement modes, lifetime and dismissal, passivity, peek, and what is out of scope.
   - Add the `show-section-map` row (no default hotkey) to the commands table.
   - Add one sentence to the manifest description.
   - Run `npm run docs:gen` and `npm run docs:inventory:check`.
3. **Deploy:** bump the version per convention, then run the full gate and
   `bob plugins sync`. On a sync refusal, follow the repo `AGENTS.md` procedure; never
   use `-F` on your own.

> [!decision] section_map_record Use your `/sase_memory_write` skill to add one
> decisions strand recording the claim "Section jumps show a passive, gesture-scoped
> Section Map built from the jump's own scanner; it never takes focus, keys, or clicks".
> Include the rejected alternatives from the research (toast, sidebar, tooltip, modal
> picker, stopping the wrap), the cost, what would reopen it, and evidence (this epic's
> commits and the research ref). Link it to `decisions:q-and-ctrl-bracket-mirror-esc`.
> Then republish memory.

**Done when:** the real-Obsidian run passes with reviewed screenshots, the docs and
inventory checks pass, and the deployed vault plugin version matches the manifest.
