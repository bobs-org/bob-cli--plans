---
tier: epic
title: Priority marks - render the task priority field as a signal-bar icon
goal: 'In Obsidian, every canonical `[priority:: …]` task field (Live Preview, reading
  view, embeds, hover previews, Dataview task views, and Tasks query results) is shown
  as one compact signal-bar glyph that reads the priority at a glance. The stored
  Markdown never changes, the cursor reveals the raw field for editing, and broken
  priority fields get a visible repair flag. The Task Card and priority notices use
  the same glyph, so you learn it where you pick a priority.

  '
phases:
- id: ledger-marks
  title: Priority marks in bob-ledger-tools
  depends_on: []
  size: medium
  description: 'ledger-marks: build the display-only priority mark in bob-ledger-tools.
    That covers the canonical-field parser, the lenient ladder-config reader, the
    mark model and tooltip, a single CSS-mask glyph set, the Live Preview decoration,
    the rendered-view post-processor, CSS-only Tasks-result replacement, the repair
    flag, the session toggle, and the additive `api.priorityMarks` v1 namespace. Also
    write the bob-cli display contract with conformance vectors, then test, build,
    and sync.

    '
- id: card-glyph
  title: Task Card and priority notices reuse the glyph
  depends_on:
  - ledger-marks
  size: small
  description: 'card-glyph: bob-navigation-hotkeys renders the shared mark through
    `api.priorityMarks` v1 in the Task Card level chips and in the priority notice
    header, and falls back to today''s rendering when the api is absent. Then test,
    document, build, and sync.'
proposed_by: bbugyi200.athena.0xf
create_time: 2026-10-06 13:50:50
status: done
bead_id: bob-cli-4p
---

- **PROMPT:** [prompts/202610/priority_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/priority_marks.md)
- **BEAD:** [bob-cli-4p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4p/README.md)

# Plan: Priority marks

## Why

A prioritized task currently renders its priority in three ways, and none of them looks
good:

- **Live Preview and reading view.** A Dataview property pill shows `PRIORITY | medium`.
  The vault snippet `dataview-properties.css` already shrinks `dependsOn` to a glyph and
  hides `id`, but `priority` still gets the full pill.
- **Tasks query results** (`dash.md`, `blocked.md`, `rotten.md`, …). Tasks 8.4.0 always
  renders the priority component with its emoji serializer, even in Dataview task
  format. The result is a stray `⏫`/`🔼`/`🔽`/`⏬` inside
  `<span class="task-priority" data-task-priority="high|medium|…"><span>EMOJI</span></span>`
  (the inner span holds the emoji, possibly with a leading space).
- **Raw text** wherever Dataview does not pretty-render.

The vault has about 430 `[priority:: …]` fields, all with the values `high`, `medium`,
`low`, or `lowest`. About 82 are written without a space (`[priority::high]`), which
Tasks accepts. Bob's semantics (`docs/projects.md` → "Priority property and scheduled
rolls"):

| Bob label | Stored value | Meaning                             |
| --------- | ------------ | ----------------------------------- |
| P0        | no field     | Highest: do it now. No roll.        |
| P1        | `high`       | Rolls 2–7 days ahead                |
| P2        | `medium`     | 8–30 days                           |
| P3        | `low`        | 31–90 days                          |
| P4        | `lowest`     | 91–365 days. Decay past P4 cancels. |

`highest` is a valid Tasks value but is not on the default ladder. Labels and windows
come from the `properties[]` entry `{name: priority, values: priority, levels: [...]}`
in `~/.config/bob/config.yml`.

## Design (decided; phases implement it as written)

### Principles (borrowed from the freshness mark, `docs/freshness.md` §11)

- **Display-only.** `[priority:: value]` stays the only stored form. Nothing writes the
  mark, and the Rust and Tasks semantics are unchanged.
- **Shape carries meaning; color whispers.**
- **Truthful or neutral.** When the ladder config is unknown, the tooltip omits P-labels
  instead of guessing.
- **Reversible.** The cursor or a click reveals the raw field. Source mode shows raw
  text. A session toggle restores today's pills and emoji. Without bob-ledger-tools the
  vault looks exactly as it does today.
- **One glyph definition** shared by every surface.

### The glyph: a signal staircase

Four rising, pill-ended bars sit in a 16-unit box. Filled bars show the priority, and a
faint track shows the bars that are left. This is the same silhouette the nav priority
notice already uses through Lucide `signal-high`/`-medium`/`-low`/`-zero` (4/3/2/1
filled positions for P1–P4), so the mapping is already familiar. The fill drains as a
task decays P1 → P4, echoing the freshness lease ring that drains with age.

| Stored value | Default label           | Glyph                                                                             |
| ------------ | ----------------------- | --------------------------------------------------------------------------------- |
| `high`       | P1                      | 4 of 4 bars filled                                                                |
| `medium`     | P2                      | 3 of 4                                                                            |
| `low`        | P3                      | 2 of 4                                                                            |
| `lowest`     | P4                      | 1 of 4                                                                            |
| `highest`    | (off-ladder by default) | "urgent": a solid rounded square with a knocked-out `!`, no track, same footprint |
| no field     | P0                      | nothing: there is no field to replace, and P0 is the absence of deferral          |

The glyph is chosen by **stored Tasks value**, not ladder position. This makes it robust
without config and matches what Tasks and Dataview see. Labels and windows in the
tooltip come from the ladder.

Starting geometry (viewBox `0 0 16 16`, black fills, used as masks):

- Bars, as `rect`s with `width 2.4` and `rx 1.2`: `x 1.1 y 10 h 4`, `x 4.9 y 7 h 7`,
  `x 8.7 y 4 h 10`, `x 12.5 y 1 h 13` (shared bottom at y 14).
- Urgent: a rounded square `x 1.5..14.5`, `y 1..14`, `rx 3.2`, with the `!` knocked out
  (`fill-rule="evenodd"` or an SVG `<mask>`). The stem is a rect
  `x 7.05 y 3.2 w 1.9 h 5.6 rx 0.95` and the dot is a circle `cx 8 cy 11.1 r 1.1`.
- Tuning by ±0.3 units during implementation is fine. Record the final numbers in the
  contract doc.

### Color and tone (monochrome by design)

Task checkboxes already use red (Blocked — most prioritized tasks are Blocked until
their scheduled date), yellow (In Progress), and green (Next). The freshness `due`
capsule owns the only loud orange. Nav already has two hue ramps that disagree: the Task
Card goes red→blue and the notices go orange→purple. A per-level hue on hundreds of
lines would be noise, so:

- **Ink.** Filled bars use `--text-muted`. The track is the same ink at 26% (the
  freshness track opacity). Urgent uses `--text-normal`, so its weight carries the
  emphasis.
- **Resting.** On closed tasks (`x`, `X`, `-`) the ink is `--text-faint` at opacity
  0.75. This is derived purely in CSS from the nearest `li.task-list-item` /
  `.HyperMD-task-line` `[data-task]` ancestor, so it works on every surface. Closed must
  win over any per-level color.
- **Per-level theme hooks** (unset by default):
  `--bob-priority-color-{highest,high,medium,low,lowest}`. A user snippet can tint
  levels without touching the plugin.
- **Metrics** match `.bob-fresh-mark`:
  - interface font at 0.8em, inline-flex, padding `0.06em 0.14em`, margin `0 0.08em`,
    pill radius, `vertical-align: 0.05em`, opacity 0.85
  - hover: opacity 1 plus a 10% ink capsule
  - glyph box 1.08em square
  - `[data-fold-space="true"]` adds `margin-inline-start: 0.3em`
  - transitions are disabled under `prefers-reduced-motion`

### One glyph definition: CSS masks

`plugins/bob-ledger-tools/styles.css` defines these SVG data-URI custom properties once,
on `body`:

- `--bob-priority-glyph-track`
- `--bob-priority-glyph-fill-1` … `-4`
- `--bob-priority-glyph-urgent`

A glyph host draws the track with `::before` and the fill with `::after`. Both are
absolutely positioned, use `-webkit-mask-image` and `mask-image`, and set
`background-color` from the ink variable. The vault's `dependsOn` glyph already proves
masks work on every Obsidian platform. There are two kinds of glyph host:

1. `.bob-priority-mark[data-priority="…"] .bob-priority-mark-glyph`, emitted by the
   plugin's JS.
2. `body.bob-priority-marks .plugin-tasks-list-item .task-priority[data-task-priority="…"]`.
   This one is CSS-only, for Tasks results. The inner emoji span is _visually hidden_
   (clip pattern, not `display: none`) so screen readers keep it, and the host gets the
   glyph box plus `margin-inline-start: 0.3em` to replace the lost leading space.

No JS touches Tasks' DOM, so re-renders, sorting, and `hide priority` all keep working
for free.

### Canonical field (mirrors the Tasks Dataview regex)

A canonical field matches `^[\[(] *priority:: *(highest|high|medium|low|lowest) *[\])]`:

- the opening and closing brackets must be a matching pair
- the key is exactly `priority`
- the value is lowercase

Eligibility in each surface:

- **Live Preview.** `freshnessTaskStatus(line) !== null` (quote-aware task line). The
  line holds exactly one `/priority\s*::/gi` occurrence, and it is canonical. It is not
  inside code (`freshnessMarkPosInCode`).
- **Rendered views.** A text node holds exactly one `priority::` occurrence, and it is
  canonical. The node's nearest `li` ancestor has class `task-list-item`. The node has
  no `code`, `pre`, `.dataview.inline-field`, `.bob-priority-mark`, or `.bob-fresh-mark`
  ancestor.

Anything else is left alone (a Dataview pill or raw text).

### Repair flag

While marks are on, any leftover `priority` Dataview pill in Live Preview is
non-canonical. That covers uppercase values, `P2`, `urgent`, duplicates, mismatched
brackets, and priority fields on non-task lines. Tasks will not read any of these, and a
trailing one also hides earlier fields from Tasks. CSS flags it with full opacity and a
dashed orange border, matching the freshness repair flag. The rule is scoped to
`body.bob-priority-marks .markdown-source-view.is-live-preview` and matches
`data-dv-norm-key="priority"` on either `.inline-field-key` or
`.inline-field-standalone-value`.

### Tooltip

The mark carries `role="img"`, an `aria-label`, and `data-tooltip-position="top"`. Lines
are joined with `\n`. The text must **never contain `::`**, because Dataview's
reading-view pass (sortOrder 100) re-scans `innerHTML` for inline fields. Value names
are capitalized (`High`).

- **On the ladder:** `P2 · Medium priority` ⏎ `Rolls 8–30 days ahead` ⏎
  `Ctrl+Shift+P to change`.
  - Equal bounds read `Rolls 5 days ahead`.
  - 1–1 reads `Rolls 1 day ahead`.
- **Off the ladder (ladder known):** `Highest priority` ⏎ `Not on the P1–P4 ladder` ⏎
  `Ctrl+Shift+P to change`. The text uses the first and last ladder labels, or a single
  label when the ladder has one level.
- **Ladder unknown** (missing, unreadable, or invalid config, no matching entry, or
  mobile): `Medium priority` ⏎ `Ctrl+Shift+P to change`.

### Interaction and surfaces

**Live Preview:**

- A `Prec.highest` ViewPlugin emits `Decoration.replace` with a `PriorityMarkWidget`.
- **Space folding.** When the character before the opening bracket is a space, the range
  starts at that space and the widget restores the gap. This is how the mark beats
  Dataview's pill widget by position.
- **Reveal.** The mark hides while any selection range overlaps the field span (the
  inclusive rule).
- **Mousedown** places the cursor at the field start and focuses the editor. It never
  writes; `Ctrl+Shift+P` stays the one way to change priority.
- **`eq()`** compares a model key, so unchanged marks never flicker.
- **Rebuild** on doc, viewport, or selection changes, Live Preview or file switches, and
  a `priorityMarksRefresh` StateEffect.
- No marks in source mode.

**Rendered views:** `registerMarkdownPostProcessor(..., 50)` runs before Dataview's
inline-field pass at 100, so it sees raw text. It splits the text node into before /
mark / after. This covers reading view, embeds, hover previews, Dataview `TASK` views,
and Tasks descriptions that still carry a non-trailing field.

**Tasks query results:** handled by the CSS-only host above. There is no tooltip there
(accepted). The `li` already carries `data-task-priority`.

**Session toggle:**

- The command "Toggle task priority marks" (id `toggle-priority-marks`) is session-only
  and on by default.
- It flips `body.bob-priority-marks`, which also gates the Tasks CSS and the repair
  flag.
- It dispatches the refresh effect to every markdown editor and triggers the existing
  `TODAY_RELOAD_EVENT`, so Tasks results re-render.
- It shows the Notice `Priority marks on` / `Priority marks off`.
- When off, JS creates no marks.

### Rejected alternatives (record these in the contract doc)

- **Changing storage to Tasks emoji or P-codes.** This breaks the Tasks Dataview format.
  It also rewrites bob-cli parsers and writers, capture, and the Task Card, plus about
  430 vault lines.
- **A CSS-only restyle of the Dataview pill (like `dependsOn`).** CSS cannot read the
  value text, so it cannot pick a glyph per level.
- **Lucide `signal-*` icons inline.** They are stroke-only with no track, they are hard
  to read at 0.8em, and P4 becomes a lone dot.
- **A per-level hue ramp.** It clashes with the status colors, and the existing ramps
  already disagree.
- **The P-label text beside the glyph.** It doubles the width on every line, and the
  label is one hover away and shown on the Task Card.
- **Marks on P0 tasks.** That would put a glyph on nearly every line with nothing to
  replace.
- **Click opens the Task Card.** Display surfaces never act.
- **JS mutation of Tasks' DOM or a MutationObserver.** This is fragile across re-renders
  and costly, and CSS on Tasks' own `data-task-priority` is exact.

### Non-goals

- `projects.base` `priority_badge` (numeric project-note frontmatter priority, a Bases
  text formula)
- bob CLI/TUI and Bob Mac Capture output
- decay-card level rows
- any storage, parser, or config-schema change
- memory notes

## Phase `ledger-marks`: Priority marks in bob-ledger-tools

Work in the `bob-plugins` linked repo, opened with `/sase_repo`, and read its
`AGENTS.md`. bob-ledger-tools uses the fragment build. Edit `src/`, never `main.js`, and
keep each fragment at or under 1000 lines. Mirror the freshness-mark code paths
(`130-freshness-marks.js`, `160-widgets-and-row.js` `FreshnessMarkWidget`,
`250-`/`260-plugin-freshness-mark-*.js`) and their defensive never-throw style. Do not
refactor the freshness code.

1. **Pure module.** Add a new fragment `src/135-priority-marks.js` and list it in
   `src/fragments.json` after `130-freshness-marks.js`. It holds:
   - `PRIORITY_MARK_VALUES`
   - `priorityMarkSource(lineText)`, which returns
     `{ value, fieldStart, fieldEnd, foldSpace }` or `null` and applies the Live Preview
     single-occurrence and canonical rules
   - `priorityMarkSourceInText(text)` for rendered nodes
   - `coercePriorityLadder(parsedYaml)`, which is lenient:
     - take the first `properties[]` entry with `name === "priority"` and
       `values === "priority"`
     - keep levels with a non-empty string label, a scalar value, and integer
       `0 <= min_days <= max_days`
     - skip bad levels and never show Notices (nav owns config validation)
     - return a frozen list or `null`
   - `priorityMarkModel(value, ladder)`, which returns a frozen
     `{ value, glyph: "bars"|"urgent", filled, label, tooltip, key }` or `null`
   - `buildPriorityMarkElement(doc, model, { foldSpace, decorative, inheritColor })`. It
     produces `span.bob-priority-mark[data-priority][data-fold-space]` containing
     `span.bob-priority-mark-glyph`. `decorative` means `aria-hidden="true"` with no
     label. `inheritColor` sets `data-inherit-color="true"`, so the ink becomes
     `currentColor`.
   - `PriorityMarkWidget` (guarded on `WidgetType`, like `FreshnessMarkWidget`)
2. **Mixin.** Add a new fragment `src/265-plugin-priority-marks.js` (after
   `260-plugin-freshness-mark-render.js`) with `BobLedgerToolsPriorityMarksMixin`:
   - `setupPriorityMarks()`: the body class, the toggle command, the editor extension,
     and the post-processor at 50
   - `createPriorityMarkExtension()` and `buildPriorityMarkDecorations(view)`
   - `priorityMarkShouldRebuild(u)`
   - `renderPriorityMarksIn(el, ctx)`
   - `togglePriorityMarks()` and `refreshPriorityMarkEditors()`
   - `priorityLadderSnapshot(now)`: reads `planConfigPath()` through the same optional
     `fs` / `parseYaml` / `Platform.isDesktopApp` guards as `loadFreshnessConfig`, with
     a `{ path, statKey, checkedAt, ladder }` cache stat-checked at most every 60 s,
     like `freshnessConfigSnapshot`
   - `priorityMarksApi()`
3. **Wiring.**
   - Add `priorityMarksRefresh` (a StateEffect, guarded) in `010-load-and-constants.js`.
   - Register the mixin in `310-install-methods.js`.
   - In `170-plugin-lifecycle.js`:
     - initialize `this.priorityMarksEnabled = true` and
       `this.priorityLadderCache = null`
     - call `setupPriorityMarks()` next to `setupFreshnessMarks()`
     - clear state in `onunload`
     - add the additive, frozen api namespace
       `priorityMarks: { version: 1, model(value), render(host, value, options) }`
       (synchronous and never throwing; `render` appends to `host` and returns the
       element or `null` for non-Tasks values). The top-level api stays v3.
   - Export the pure helpers in `350-exports.js` `helpers`.
4. **CSS.** Add a "Priority marks" block to `plugins/bob-ledger-tools/styles.css`
   exactly as described under Design: the mask variables once, both glyph hosts, tones,
   theme hooks, resting, fold space, hover, reduced motion, the Tasks host with its
   visually-hidden emoji, and the Live Preview repair flag.
5. **Tests.** Add `scripts/test-ledger-tools-priority-marks.cjs`, append it to the
   `package.json` `test` list, and use stubs in the style of
   `test-ledger-tools-freshness-mark-surfaces.cjs`. Cover:
   - the parser accepts and rejects the conformance vectors below verbatim
   - the model and tooltip, including that no output contains `::`
   - element structure and the `decorative` / `inheritColor` options
   - Live Preview decorations: range with and without space folding, selection reveal,
     no marks in source mode, inside code, on non-task lines, or when toggled off, and a
     line with both `[fresh:: …]` and `[priority:: …]` gets two disjoint decorations
   - post-processor node splitting and exclusions (non-task `li`, code, existing marks,
     duplicates)
   - ladder coercion: valid, partial, missing file, mobile, parse error, and cache reuse
   - api namespace shape and never-throw behavior
   - a CSS contract check that parses `styles.css` and asserts that each of the five
     values has a rule on both hosts and that each mask variable is defined exactly once
6. **Contract doc (bob-cli primary repo).** In `docs/projects.md`, add
   `### Priority marks` after "Priority property and scheduled rolls", plus its Contents
   entry. Mark it as the authoritative display contract, in the style of
   `docs/freshness.md` §11. It should cover the principles, the glyph table with the
   final geometry, tones, eligibility, the repair flag, the tooltip grammar, surfaces
   and interaction, the toggle, the rejected alternatives, the conformance vectors
   below, and a **Live verification (Bryan, in Obsidian)** checklist:
   - P1–P4 and highest look right in light and dark themes
   - the cursor or a click reveals the raw field
   - `dash.md` Tasks results show glyphs, not emoji
   - a hand-broken `[priority:: High]` shows the dashed repair pill
   - the toggle restores the pills and emoji and then turns the marks back on
   - closed tasks look resting
   - fresh marks and priority marks sit together cleanly
   - Metadata Menu does not double-decorate
   - mobile (iOS) renders the glyphs
7. **Plugin docs and metadata.**
   - Bump the bob-ledger-tools minor version in `manifest.json`.
   - Extend its description with "render `[priority::]` fields as signal-bar priority
     marks".
   - Update `README.md`: the plugin table row (version, feature, and `priorityMarks`
     namespace v1 in the api list), a short paragraph next to the freshness-mark
     paragraph, and the test-layout listing.
8. **Verify and deploy.** Run `npm run build`, `npm test`, and `npm run validate` in
   bob-plugins, then run `bob plugins sync`. Commit both repos through the normal final
   flow. Report that the live verification checklist is still pending for Bryan.

**Conformance vectors** (default ladder: P1 `high` 2–7, P2 `medium` 8–30, P3 `low`
31–90, P4 `lowest` 91–365; ⏎ separates tooltip lines):

| #    | Input                                                      | Expected                                                                               |
| ---- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| PM1  | `- [ ] #task A [priority:: high] ^a`                       | 4 bars · `P1 · High priority⏎Rolls 2–7 days ahead⏎Ctrl+Shift+P to change` · fold space |
| PM2  | `- [?] #task B [priority::medium] [scheduled::2026-10-22]` | 3 bars · `P2 · Medium priority⏎Rolls 8–30 days ahead⏎…`                                |
| PM3  | `- [ ] #task C (priority:: low)`                           | 2 bars · P3, 31–90                                                                     |
| PM4  | `- [ ] #task D [ priority:: lowest ]`                      | 1 bar · P4, 91–365                                                                     |
| PM5  | `- [ ] #task E [priority:: highest]`                       | urgent · `Highest priority⏎Not on the P1–P4 ladder⏎Ctrl+Shift+P to change`             |
| PM6  | PM2 with the ladder unknown                                | 3 bars · `Medium priority⏎Ctrl+Shift+P to change`                                      |
| PM7  | `- [ ] #task F [priority:: High]`                          | no mark (repair pill)                                                                  |
| PM8  | `- [ ] #task G [priority:: P2]`                            | no mark                                                                                |
| PM9  | `- [ ] #task H [priority:: high] [priority:: low]`         | no mark on either                                                                      |
| PM10 | `- [ ] #task I [priority:: high)`                          | no mark                                                                                |
| PM11 | `- plain bullet [priority:: high]`                         | no mark                                                                                |
| PM12 | ``- [ ] #task J `[priority:: high]` ``                     | untouched (code)                                                                       |
| PM13 | ladder `{P1, highest, 1–3}`, `[priority:: highest]`        | urgent · `P1 · Highest priority⏎Rolls 1–3 days ahead⏎…`                                |
| PM14 | ladder `{P1, high, 5–5}` / `{P1, high, 1–1}`               | `Rolls 5 days ahead` / `Rolls 1 day ahead`                                             |
| PM15 | `- [x] #task K [priority:: high]`                          | 4 bars, resting tone (CSS)                                                             |
| PM16 | `> - [ ] #task L [priority:: medium]`                      | 3 bars (quote-aware)                                                                   |
| PM17 | `- [ ] #task M x[priority:: low]`                          | 2 bars, no fold space                                                                  |

## Phase `card-glyph`: Task Card and priority notices reuse the glyph

Work in the `bob-plugins` linked repo, plugin `bob-navigation-hotkeys`, using the
fragment build. The glyph is decorative wherever a visible P-label already exists.

1. Add a guarded `getLedgerPriorityMarksApi(app)` helper, mirroring the lookup in
   `getCancelPlanBudgetChip`. It returns
   `app.plugins.plugins["bob-ledger-tools"].api.priorityMarks` only when `version >= 1`
   and `render` is a function, and `null` otherwise.
2. **Task Card** (`src/330-task-card-view.js`): in each configured-level chip, render
   `render(host, level.value, { decorative: true, inheritColor: true })` into a new
   `span.bob-task-card-level-glyph` at the start of `.bob-task-card-level-label`, before
   the label text. The P0 chip gets no glyph. Thread `app` from the existing Task Card
   options. When there is no api, or `render` returns `null`, nothing changes. Add small
   spacing and alignment CSS in `styles.css`.
3. **Priority notice** (`src/280-notices-and-dates.js`):
   - Add `levelValue` to the frozen model from `buildPriorityNoticeModel` and to the
     batch model, if the batch renders through the same fragment.
   - In `renderPriorityNoticeFragment`, render the mark (decorative, `inheritColor`, so
     the existing `--bob-nh-level-color` icon color still applies) into
     `.bob-nh-notice-icon`.
   - Otherwise keep `applyIcon(iconEl, model.iconName)`, and leave `iconName` unchanged.
   - Pass the api (or `app`) in from the existing `showPriorityNotice` call sites.
4. **Tests.** Extend `scripts/test-navigation-task-card-view.cjs` and
   `scripts/test-navigation-hotkeys-priority-notice.cjs`:
   - with a stub `priorityMarks` api, the glyph appears with the right value and the P0
     chip has none
   - without the api, or with `render` returning `null` or throwing, the output is
     unchanged and Lucide is used
5. **Docs and deploy.**
   - Bump the nav minor version and update its manifest description and the README row.
   - Add one sentence to the bob-cli `docs/projects.md` "Priority marks" section: the
     Task Card level strip and priority notices reuse the mark through
     `api.priorityMarks` v1.
   - Run `npm run build`, `npm test`, and `npm run validate`, then `bob plugins sync`.
   - Add "the Task Card shows the glyph beside P1–P4" to the live verification
     checklist.

## Done when

- Both phases' test suites, `npm run validate`, and the build staleness check pass.
- `bob plugins sync` has deployed both plugins.
- The contract doc and READMEs describe the shipped behavior.
- The live verification checklist is reported as pending for Bryan.
