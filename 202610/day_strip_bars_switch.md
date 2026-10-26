---
tier: tale
title: "Day strip bars switch: both lanes, root parents only, or themes only"
goal:
  One quiet switch on the Day strip cycles both bars, root parents only, and themes
  only, as one view for every daily note.
size: medium
decisions:
  remember:
    ask: Should the bars choice survive an Obsidian restart on this device?
    choices:
      device: Remember it on this device, so every restart opens in the last choice
      session:
        Keep it for the plugin session only, like Toggle Day stats; restarts open with
        both bars
    default: device
    why:
      It is a reading preference, not a temporary hide; forgetting it each restart feels
      broken
    answer: device
  memory_glossary:
    ask:
      Should the Day Strip glossary strand say its two lanes show together or one at a
      time?
    memory:
      - glossary:day-strip
    corrects:
      "Its contents: five LABEL-value facts, a [[root-parent]] legend with `so far ·
      HH:MM` over the root-parent bar, and a [[pomodoro-theme]] legend over the
      full-width theme bar that separates it from the badges."
    default: true
    answer: true
proposed_by: bbugyi200.athena.14l.f1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 13:05:22
status: wip
---

# Day strip bars switch: both lanes, root parents only, or themes only

## Goal

The Day strip on a canonical daily note ends with two lanes that split the same `worked`
minutes two ways: the **root-parent** lane (a `●` legend over its bar) and the **theme**
lane (a `■` legend over its bar). Add one small switch that cycles the strip between
three views:

1. **Both** (today's look, and the default after upgrade),
2. **Root parents only**,
3. **Themes only**.

The choice is **one global view**. Pressing the switch on any strip repaints every open
strip at once (split panes, popout windows, `bob-day-stats` fences), and every daily
note you open next paints in that view from its first frame, skeleton included. The
switch must be:

- **intuitive:** it speaks the strip's own mark language (a `●` dot for root parents and
  a `■` rounded square for themes, filled when shown and hollow when hidden). It never
  moves between presses, and its tooltip names the current view and the next one;
- **reliable:** the view is read from memory, never from disk, during a paint. Every
  switch repaints from the cache without spawning `bob`. The strip falls back honestly
  when the chosen lane has no data that day, and keyboard focus survives the repaint;
- **beautiful:** the switch is quiet chrome in the legend's own size and faint ink, and
  the lane that appears fades in softly. In both view the strip keeps its exact geometry
  and height budgets.

No Rust change and no new `bob` config key are needed.

## Design

### The three views

```text
both (default)
WORKED    REVIEWED   DONE   CREATED   REVIEW CLOSED
4h 50m    12         7      9         ◐
● GTD 2h 30m   ● vibe 1h 55m   ● bob 25m             so far · 14:05   ●■
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
■ VIBE 1h 55m   ■ BOB 25m   ■ GTD 2h 30m
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

root parents only
● GTD 2h 30m   ● vibe 1h 55m   ● bob 25m             so far · 14:05   ●□
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

themes only
■ VIBE 1h 55m   ■ BOB 25m   ■ GTD 2h 30m             so far · 14:05   ○■
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

- **Cycle order:** both → root parents → themes → both. Each press walks one step down
  the both-view stack (top lane alone, then bottom lane alone), then returns to both.
- **The top lane row is the anchor.** It is always the first lane row under the facts,
  and it carries the meta (`so far · HH:MM`, `refresh failed`) and the switch. In
  themes-only view the themes row takes the top-row position, spacing (`.35em` below the
  facts), meta, and switch, so the switch keeps the same box across all three views. You
  can keep clicking without moving the mouse.
- **Spacing:** the `.6em` inter-lane gap belongs to a lane row that follows a bar, not
  to the themes lane as such. A single-lane strip is facts, one legend row, and one bar,
  and that bar is the strip's last line and the separator above the badges.
- **Hidden lanes are not painted.** They are not hidden with CSS. The fitter measures
  only present legends, and accessible text describes only what is shown. Switching is a
  synchronous repaint from the cached report.

### The switch

- **Element:** `span.bob-day-strip-lanes` with `role="button"`, `tabindex="0"`,
  `data-view="both|roots|themes"`, `aria-label`, and `title`. It is never a `<button>`,
  so Obsidian's global button rules cannot touch the strip. It holds two `aria-hidden`
  marks:
  - `span.bob-day-strip-lanes-mark.bob-day-strip-lanes-mark-roots`, a circle;
  - `span.bob-day-strip-lanes-mark.bob-day-strip-lanes-mark-themes`, a rounded square;
  - each mark carries `data-on="true|false"`.
- **Placement:** the last child of the top lane row, after the meta when there is one,
  pushed to the right edge (`margin-left: auto`). The legend link keeps absorbing the
  free width.
- **Size:** `font-size: .85em` (the legend's size), so each mark has exactly the legend
  dot and swatch footprint (`.5em`). The marks sit `.25em` apart, with small padding
  around them for the hit box and `var(--radius-s, 4px)` corners. The switch must not
  make the row taller than its legend line, so both-view heights stay within today's 116
  px / 167 px budgets.
- **Ink:** quiet chrome, never a hue. At rest it uses `--text-faint`. On hover it uses
  `--text-muted` over `--background-modifier-hover`. `:focus-visible` gets a 2 px
  `--interactive-accent` outline at 1 px offset, like the facts and `Retry`. An on mark
  is filled with `currentColor`. An off mark is transparent with
  `box-shadow: inset 0 0 0 1px currentColor`, a hollow `○`/`□` that still keeps its
  footprint. Under `forced-colors`, marks use `ButtonText` and off marks keep a visible
  ring.
- **Why these marks:** each lane's legend already wears them (PT6, "names wear a tint;
  places wear a dot"). The switch's `●■` reads as "both lenses on", `●□` as "root
  parents only", and `○■` as "themes only", with no new icon to learn.
- **Copy** (fixed strip copy; `<roots>` is `root parents`, or `areas` when the roots
  lane uses the older-bob `area` fallback):

  | view   | `title`                                               | `aria-label`                                                     |
  | ------ | ----------------------------------------------------- | ---------------------------------------------------------------- |
  | both   | `Showing <roots> and themes · click for <roots> only` | `Day stats bars: <roots> and themes. Activate for <roots> only.` |
  | roots  | `Showing <roots> only · click for themes only`        | `Day stats bars: <roots> only. Activate for themes only.`        |
  | themes | `Showing themes only · click for both`                | `Day stats bars: themes only. Activate for <roots> and themes.`  |

- **Activation:** click, Enter, and Space advance the cycle once. `mousedown` calls
  `preventDefault`, so a click never steals focus from the editor and vim keeps working.
  Ctrl/Cmd and middle-click do nothing special (`auxclick` is ignored), and the switch
  never opens Stats.
- **Focus survives the repaint:** if the activated switch had keyboard focus, the new
  switch in the same strip is focused after the repaint
  (`focus({ preventScroll: true })`). Enter, Enter, Enter cycles all three views, and a
  screen reader announces each new label. A mouse press never moves focus.
- **Tab order:** facts, top legend, switch, then the second legend when both lanes are
  shown.

### When the switch shows, and honest fallbacks

A lane is **available** when its model is not omitted. The roots lane on a zero day
(empty track plus `no Pomodoros`) is available; the theme lane on that day is omitted.

- **Shown lanes**, in order:
  - both: every available lane (roots first);
  - roots: the roots lane if available, else the theme lane if available;
  - themes: the theme lane if available, else the roots lane if available;
  - with neither available, the roots row holds the meta alone, as today.
- **The switch is painted only when both lanes are available** (ready and stale states).
  On a zero-Pomodoro day, or a day with no theme parts, it is absent. Nothing is offered
  that would change nothing, and the shown lane always matches the choice whenever the
  switch is visible.
- **Skeleton:** it paints the lanes of the current view (two empty tracks for both, one
  for a single-lane view) plus an invisible, inert switch placeholder in the top row
  (`visibility: hidden`, no `tabindex`, no listeners). The skeleton height equals the
  ready height in that view.
- **Stale:** same as ready; the switch works, since switching is local.
- **Future, hidden, unavailable:** no lanes and no switch, as today.
- **Group `aria-label`:** includes the `Time by …` sentence of shown lanes only.

### One global view, and where it lives

- The plugin holds the view in memory (`this.dayStripLaneView`, default `both`). Every
  model call in the strip paint path (ready, skeleton, rollover) receives it as
  `context.laneView`. A paint never reads storage.
- **Switching** sets the view and then repaints every record in `dayStripViewsStore()`
  synchronously through `dayStripPaint(record)`. Paints are cache-first, never spawn,
  and keep the existing single-flight refresh policy untouched. The `paintedKey` gains
  the view and the shown lanes, so a switch always repaints and an unchanged view still
  dedupes.
- **Scope:** every strip on this device follows the one view, including explicit
  `bob-day-stats` fences. There is no per-note or per-fence view.

> [!decision] remember = device Persist the view per device, per vault, with Obsidian's
> local storage under the key `bob-ledger-tools-day-strip-bars`, holding
> `"both" | "roots" | "themes"`. Read it once in `setupDayStripHosts` through
> `this.app.loadLocalStorage(key)` (guarded with `typeof` checks; any other value is
> `both`). Write it through `this.app.saveLocalStorage(key, view)` on every change. A
> missing API or a throwing write keeps the in-memory view for the session silently,
> with no Notice. Per-device is deliberate: `bob-*` plugin folders are gitignored in the
> vault, and a view choice needs no cross-device agreement or vault commit. Local
> storage is synchronous, so the very first paint after a restart already uses the
> remembered view, with no flash of the wrong lanes.

> [!decision] remember = session Never read or write storage. The view lives in memory
> for the plugin session and starts as `both` on every load, like `Toggle Day stats`.
> Docs say "for the current plugin session".

### Command

Add `Cycle Day stats bars` (id `cycle-day-stats-bars`, no default hotkey), registered
next to `Toggle Day stats`. It advances the same cycle, repaints every strip, and shows
a Notice: `Day stats bars: root parents only`, `Day stats bars: themes only`, or
`Day stats bars: root parents and themes`. It works with no strip on screen, since it
changes the view the next strip paints. Use a literal `this.addCommand({...})` call so
the docs inventory scanner lists it.

### Motion

When a repaint on the same host and the same day changes the view, the newly shown
lane's row and bar get `bob-day-strip-lane-enter`: opacity 0 → 1 over 160 ms ease-out.
There is no transform, so layout never shifts beyond the inevitable height change. Data
repaints, day changes, first paints, skeleton → ready, and stale flips never animate.
Remember `{ day, view }` per host in a `WeakMap`, the same best-effort way
`dayStripMarkMemory` drives the check-draw, and write it only on ready and stale paints.
The existing `prefers-reduced-motion` block already disables the animation.

### Rejected alternatives

- **A segmented `Roots | Themes | Both` control:** it is explicit, but costs about 9em
  of the legend line on every day, and that line is the strip's scarcest width.
- **Clicking a legend to collapse its lane:** legends and bars already open Stats;
  overloading them breaks the established click contract.
- **Two independent per-lane switches:** two 6 px targets, and an impossible "both off"
  state to special-case.
- **Hiding lanes with CSS:** the fitter would measure zero-width legends, and accessible
  text would describe hidden data. A cached repaint is cheap and synchronous.
- **A vault-synced preference** (a `.bob/` journal or a git-tracked `data.json`): it
  costs a vault commit per click for a choice no other device needs, and journals are
  bob-owned writes.
- **A per-note or per-fence view, or a `bars:` fence option:** you asked for one state
  while browsing daily notes; no fence need exists yet.

## Implementation

Most work is in the linked `bob-plugins` repo (plugin `bob-ledger-tools`). Open it with
your `/sase_repo` skill (`sase repo open bob-plugins -r "<why>"`), read its `AGENTS.md`,
and use the printed path for every read and write. Edit `src/` fragments only, list any
new fragment in `src/fragments.json` (source order matters), keep every hand-edited
fragment at or below 1000 lines, regenerate `main.js` with `npm run build`, and never
hand-edit `main.js`. `src/239-plugin-day-strip-registry.js` is already at about 906
lines, so new plugin methods go in a new fragment.

### bob-plugins

1. **Lanes fragment** (new `src/239-plugin-day-strip-lanes.js`, listed in
   `src/fragments.json` right after `239-plugin-day-strip-theme-model.js`):
   - **Constants:** `DAY_STRIP_LANE_VIEWS = ["both", "roots", "themes"]` and
     `DAY_STRIP_LANES_STORAGE_KEY = "bob-ledger-tools-day-strip-bars"`.
   - **Pure helpers:** `dayStripLaneViewOf(value)` (sanitize; anything else is `both`),
     `dayStripNextLaneView(view)`, and
     `dayStripLaneLayout({ state, mix, themes, view })`. The layout helper returns
     `{ view, shown, switchable, next, title, ariaLabel }` per the rules and copy above,
     uses `areas` when `mix.dimension === "area"`, and never throws (on error: both,
     with today's shown lanes, not switchable).
   - **A `BobLedgerToolsDayStripLanesMixin` class**, added to
     `src/310-install-methods.js` next to the other day-strip mixins, with:
     - `dayStripLaneViewCurrent()`;
     - `dayStripLoadLaneView()` (the `remember` branch);
     - `dayStripSetLaneView(view, opts)`, which sanitizes, stores in memory, persists
       per the `remember` branch, repaints every record, and restores focus when
       `opts.focusFrom` names the strip element whose switch had focus;
     - `dayStripCycleLaneView(opts)`.

     Every method is best-effort and never throws into Obsidian.

2. **Model** (`src/239-plugin-day-strip-model.js`): `dayStripModel` adds a `lanes` field
   from `dayStripLaneLayout` on every state, using `context.laneView`. Hidden, future,
   unavailable, and model-error states get a not-switchable layout with no shown lanes.
   The skeleton gets the view's lanes. `dayStripAriaLabel` takes the shown lanes and
   appends only their sentences. Keep `mix` and `themes` exactly as they are; the layout
   only chooses among them.

3. **Painter** (`src/239-plugin-day-strip-render.js`):
   - Paint lanes from `model.lanes.shown` in order, keeping the flat child order (row,
     bar, row, bar) and `data-lane` attributes.
   - The first painted lane row gets the meta and, when `switchable` (or as the
     invisible inert placeholder in the skeleton), the switch. A themes row may now be
     first.
   - Add `dayStripRenderLanesSwitch(doc, model, handlers)` and an activation helper in
     the `dayStripActivate` style: click, Enter, and Space call
     `handlers.cycleLanes({ focused })`, where `focused` is true when the switch was
     `doc.activeElement`; `mousedown` calls `preventDefault`; no `auxclick` handling.
   - Add the per-host `{ day, view }` memory and the `bob-day-strip-lane-enter` class
     for entering lanes.
   - No `<button>` anywhere.

4. **Wiring** (`src/239-plugin-day-strip-registry.js`, `dayStripPaint`):
   - Pass `laneView: this.dayStripLaneViewCurrent()` to every `dayStripModel` call,
     skeleton paths included.
   - Add
     `handlers.cycleLanes = ({ focused }) => this.dayStripCycleLaneView({ focusFrom: focused ? record.el : null })`.
   - Add the view plus the shown-lanes list to the `paintedKey`.

   Also check `setupDayStripHosts` calls `dayStripLoadLaneView()` before any strip
   paints. Keep the registry fragment at or below 1000 lines.

5. **Command** (`src/239-plugin-day-strip-plan-host.js`, `setupDayStripPlanHost`):
   register `Cycle Day stats bars` (`cycle-day-stats-bars`) with the Notice copy above,
   next to `Toggle Day stats`.

6. **Styles** (`styles.css`):
   - Change the inter-lane rule from `.bob-day-strip-mix-row.bob-day-strip-lane-themes`
     to `.bob-day-strip-bar + .bob-day-strip-mix-row { margin-top: .6em }`. The shade
     rules that are scoped by `.bob-day-strip-lane-themes` keep working, because the
     themes row keeps that class.
   - Add the `.bob-day-strip-lanes` and `.bob-day-strip-lanes-mark*` rules (sizes, ink,
     hover, focus, off-ring), the skeleton `visibility: hidden` for the switch, the
     `bob-day-strip-lane-enter` keyframes, and a `forced-colors` rule.
   - Update the strip header comment: the last painted bar is the separator, and a
     switch on the top lane row chooses the lanes.

7. **Exports** (`src/350-exports.js`): `dayStripLaneViewOf`, `dayStripNextLaneView`,
   `dayStripLaneLayout`, and `DAY_STRIP_LANES_STORAGE_KEY`.

8. **Tests.** Add `scripts/test-ledger-tools-day-strip-lanes.cjs` (append it to the
   `test` script in `package.json`) using the existing FakeNode harness pattern from
   `scripts/test-ledger-tools-day-strip-themes.cjs`. For registry and persistence cases,
   reuse the `makeApp` pattern from `scripts/test-ledger-tools-day-strip-plan-host.cjs`,
   which already fakes `loadLocalStorage` / `saveLocalStorage` with a `Map`. Cover:
   - **Pure helpers:** sanitize; the cycle order; the layout matrix (both lanes
     available, themes omitted, roots omitted, neither, zero day, skeleton per view,
     unavailable/future/hidden); `areas` copy under the area fallback; exact
     `title`/`aria-label` strings.
   - **Painter, per view:** child order and `data-lane`; the first row carries the meta
     and the switch, the second row neither; a themes-first row works; the switch is
     absent when not switchable; its attributes (`role="button"`, `tabindex="0"`,
     `data-view`, marks with `data-on`); the skeleton placeholder has no `tabindex` and
     no listeners; no `<button>`.
   - **Activation:** click, Enter, and Space call `cycleLanes` once with the right
     `focused`; `mousedown` prevents default; `auxclick` and other keys do nothing.
   - **Group `aria-label`:** names only the shown lanes.
   - **Lane-enter class:** on a same-day view change only; not on data repaints, a day
     change, the first paint, or skeleton → ready.
   - **Registry:**
     - a switch repaints every registered record without a spawn (`dayStripSpawns`
       unchanged);
     - the `paintedKey` dedupes an unchanged view;
     - focus moves to the new switch when `focusFrom` is set;
     - `Cycle Day stats bars` is registered and shows the Notice copy.
   - **Persistence, `remember = device`:** setup loads a stored view; a stored invalid
     value loads as `both`; changes save under the key; a missing API or a throwing
     `saveLocalStorage` keeps the in-memory view without throwing.
   - **Persistence, `remember = session`:** storage is never touched, and a fresh load
     starts as `both`.

   Update existing day-strip tests only where they counted the top row's children or
   assumed the meta lives only on the roots row. The default view is `both`, so
   everything else must pass unchanged.

9. **Preview harness** (`scripts/preview-day-strip.cjs`,
   `scripts/preview-day-strip.template.html`):
   - Add a view axis. Render the existing matrix in `both`, plus these cases in `roots`
     and `themes`: `today`, `past`, `screenshotDay`, `themeCollision`, `noPomodoros`,
     `noThemeHues`, skeleton, and stale. Pass `laneView` in the model context.
   - **New check (t), switch anchor:** for each switchable fixture at each width, the
     switch's bounding box is identical (±0.5 px) across the three views.
   - **New check (u):** single-lane views paint exactly one lane row and one bar; the
     bar is the strip's last line box; the lane is the chosen one when available, else
     the fallback.
   - **New check (v):** skeleton height equals ready height per view (±0.5 px).
   - **New check (w):** both switch marks are square (±0.5 px) at the legend dot's size,
     the roots mark is round and the themes mark's radius is below half its width, an
     off mark computes a transparent background with a visible ring, and the switch box
     is at least 20×14 px and lies inside the top row.
   - **Existing checks:** (g) still finds no `button`; (j) applies to the last painted
     bar; (l) must still pass. If the switch costs `screenshotDay` a root at 410 px,
     tighten the switch's footprint first. Re-baseline that one expectation to the
     measured count only if the footprint is already minimal, and say so in the commit
     and docs.
   - **Interaction check:** forced `:focus-visible` on the switch shows an outline.
   - **Budgets:** both-view budgets stay at 116 px / 167 px; the switch must not grow
     the row. Measure the single-lane one-line (540/700 px) and wrapped (410 px) heights
     and add `DAY_STRIP_HEIGHT_ONE_LANE_*` budgets at measured plus 2 px.
   - Run `node scripts/preview-day-strip.cjs --out <tmpdir>` and look at the light and
     dark × 410/540/700 screenshots yourself (read the PNGs). Iterate until the switch
     reads as quiet chrome aligned to the bar's right end, and the single-lane strips
     look as calm as the two-lane one.

10. **Docs** (`docs/bob-ledger-tools.md`):
    - In the Day strip paragraph, say the switch at the right of the top lane cycles
      both bars → root parents only → themes only, for every daily note at once (per the
      `remember` branch).
    - Next to the `Toggle Day stats` paragraph, describe `Cycle Day stats bars`.
    - Run `npm run docs:gen` so the generated command inventory row appears.
    - If you touch the manifest description sentence ("…support the bob-day-stats fence,
      and provide the Toggle Day stats session command"), keep `manifest.json` and the
      page consistent.

11. **Release hygiene:**
    - Bump `plugins/bob-ledger-tools/manifest.json` to the next minor version over
      whatever HEAD carries; feature commits bump minor.
    - Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
    - Run `bob plugins sync` to deploy to the vault.

### bob-cli

12. **`docs/stats.md` → Day strip:**
    - **Opening paragraph:** "the last painted bar is the strip's last line".
    - **Layout paragraph:** the top lane row carries the meta and the switch. Replace
      "There are no history marks, no collapse control" with "no history marks", plus a
      bars-switch paragraph covering:
      - the three views and the cycle order;
      - marks, ink, and copy;
      - when the switch shows, and the fallbacks;
      - skeleton per view, the group `aria-label` covering shown lanes only, and the
        motion rule.
    - **Interaction:** the switch is a `span[role=button]` and never opens Stats; focus
      survives the repaint. Keep "no `<button>` elements".
    - **Layout:** the `.6em` gap follows a bar; add the measured single-lane height
      budgets.
    - **Controls:** the switch and `Cycle Day stats bars` set one view for every strip
      on the device (per the `remember` branch). `stats.day_strip` and
      `Toggle Day stats` still govern the whole strip.

### Memory

> [!decision] memory_glossary Use your `/sase_memory_write` skill. In the
> `glossary:day-strip` strand, replace the contents sentence with: "Its contents: five
> LABEL-value facts, then a [[root-parent]] lane and a [[pomodoro-theme]] lane, each a
> legend over a full-width bar, shown together or one at a time by the bars switch on
> the top lane row beside `so far · HH:MM`; the last bar separates it from the badges."
> Keep the rest of the strand. Then run `sase memory init` and `just docs-gen` (the
> glossary page is generated from the memory web).

> [!decision] memory_glossary = no Leave the strand unchanged. If no human reviewed the
> plan, follow the `/sase_memory_write` declined-change route (file a `memory` task
> bead).

Do not edit `CHANGELOG.md` or the `Cargo.toml` version (release train). The mounted
`docs/reference/plugins/*` pages in bob-cli are build-assembled and git-ignored; edit
only the bob-plugins source page.

## Verification

- **bob-plugins:**
  - `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` all pass;
  - the preview matrix reports no failures, including (t)–(w);
  - the screenshots look right in light and dark at 410/540/700 px in all three views;
  - `bob plugins sync` succeeds.
- **bob-cli:** `just docs-check` passes (after `just docs-gen` if memory changed).
- **Manual sanity in Obsidian** (after reloading the plugin):
  - open two daily notes side by side, press the switch in one, and both switch at once;
  - Tab to the switch and press Enter three times; it cycles and keeps focus;
  - `Cycle Day stats bars` from the palette shows its Notice;
  - under `remember = device`, the view survives an Obsidian reload.

## Out of scope

- No Stats page change: its Root parents / Themes toggle stays per Stats view.
- No per-note or per-fence view, no `bob` config key, no Rust or Stats JSON change, no
  Mac Capture change.
