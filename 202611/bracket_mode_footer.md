---
tier: tale
title: "Bracket mode footer: always show what [[ / ]] repeat"
goal:
  Obsidian's desktop status bar always shows a concise, beautiful [[ s w n ]] indicator
  with the active bracket-repeat walk lit, updates the moment the mode changes,
  acknowledges ]] / [[ presses, and lets a click on a letter switch modes without
  walking or stealing editor focus.
size: medium
proposed_by: bbugyi200.athena.16u
proposed_with: claude/opus
create_time: 2026-11-01 18:16:25
status: wip
---

# Plan: Bracket mode footer: always show what `[[` / `]]` repeat

## Goal

Make the Obsidian status bar always show which walk `]]` / `[[` will repeat, using a
small, native-looking indicator. Today the mode is invisible. Bryan only finds out what
`]]` does by pressing it, or by seeing
`Nothing to repeat yet — press ]s, ]w, or ]n first`.

All work lands in the linked **`bob-plugins`** repo, in the `bob-navigation-hotkeys`
plugin. Open it with `/sase_repo` (`sase repo open bob-plugins -r "<reason>"`), use the
printed path for every read and write, and read its `AGENTS.md` first. bob-cli,
bob-mac-capture, and the vault vimrc do not change.

## Background (current behavior)

- `plugins/bob-navigation-hotkeys/src/546-plugin-bracket-repeat.js` owns the state.
  `rememberBracketRepeatType(type)` takes `s`, `w`, or `n`. It sets
  `this.bracketRepeatType` and writes Obsidian local storage key
  `bob-navigation-hotkeys-bracket-repeat` only when the value changes.
  `readBracketRepeatType()` restores the value from storage lazily.
  `repeatBracketWalk(direction)` delegates synchronously to `jumpToDueTask` (`s`),
  `jumpToLaneTask` (`w`), or `openLabeledBodyLink` (`n`). With no type, it shows
  `BRACKET_REPEAT_EMPTY_NOTICE`.
- `src/490-plugin-lifecycle.js` sets the type from the family command callbacks only:
  - `s`: next/prev/first/last due task, `Ctrl+Alt+J/K`, and the Ledger Tools
    review-footer click.
  - `w`: lane-task and lane commands.
  - `n`: `open-next-link` and `open-prev-link`. The vault vimrc maps `]]` / `[[` to
    `repeat-next-bracket-walk` / `repeat-prev-bracket-walk`.
- `docs/bob-navigation-hotkeys.md` § "Bracket repeat" is the user contract.
  `scripts/test-navigation-bracket-repeat.cjs` pins the behavior. Both must keep passing
  unchanged, except for the doc additions below.
- Precedents for status-bar items (copy their patterns, not their code):
  - Ledger Tools review footer: `240-plugin-freshness-status.js`, CSS `.bob-freshness`.
  - Keep inbox item: `292–294-keep-inbox-*.js`, CSS `.bob-keep-inbox`.
  - Command Mode status chip: `bob-command-mode/src/120-status-chip.js`. They share
    these rules:
  - Desktop only (`Platform`), and a no-op without `addStatusBarItem`.
  - A pure view model plus an idempotent painter that reuses one DOM tree.
  - Tooltips go through `setTooltip(node, text, { placement: "top" })`, with a `title`
    fallback.
  - The indicator never throws.
- Visual precedent: `scripts/preview-lane-walk-notice.cjs` and
  `scripts/preview-review-footer.cjs` render production model, painter, and CSS in
  headless Chromium, in light and dark, and save PNGs.

## Design

### What it looks like

The new status-bar item reads as the two repeat keys framing the three families they can
repeat. The active family sits on a soft accent "thumb":

```
review   [[ ⟨s⟩ w  n ]]
lanes    [[  s ⟨w⟩ n ]]
links    [[  s  w ⟨n⟩]]
unset    [[  s  w  n ]]      no thumb; tooltip says "press ]s, ]w, or ]n"
```

`⟨x⟩` marks the thumb. Why this form:

- The letters are the exact second keys Bryan types (`]s`, `]w`, `]n`). The lit letter
  therefore reads directly as "`]]` currently means `]s`". Showing all three letters
  also teaches the alternatives in about 11 characters.
- Position plus the filled thumb carries the meaning, so the indicator never depends on
  color alone.
- It uses one theme-native accent (`--interactive-accent`), not one hue per family. The
  review footer already uses every tier hue (NEXT green, ROTTEN orange, …), and a green
  `w` beside a green NEXT group would imply a link that does not exist.
- It uses monospace glyphs, so the letters read as keys. Fixed segment widths also let
  the thumb slide by pure CSS.

### Visual spec (bob-navigation-hotkeys `styles.css`, new section)

These values are the starting point. Tune them against the preview screenshots until the
indicator looks crisp. The preview is the acceptance bar.

- Root `.bob-bracket-mode`:
  - Layout: `display: inline-flex; align-items: center; gap: ~0.3em; flex: 0 0 auto`.
    The item never shrinks. The review footer is the item that yields width.
  - Type: `font-family: var(--font-monospace)`,
    `font-size: var(--font-ui-smaller, 12px)`, `line-height: 1`, `user-select: none`.
  - Custom properties: `--bob-bm-seg` (segment width, about `1.3em`) and
    `--bob-bracket-index` (0, 1, or 2).
  - Do not add Obsidian's `mod-clickable`. Only the letters are interactive.
- Keys `.bob-bracket-mode-key.is-backward` (`[[`) and `.is-forward` (`]]`):
  - `color: var(--text-faint)`.
  - Slight negative letter-spacing so each pair reads as one glyph.
  - `aria-hidden`, because they are decorative.
- Track `.bob-bracket-mode-track`: `position: relative; display: inline-flex`.
- Thumb `.bob-bracket-mode-thumb`:
  - Placement: absolutely positioned behind the letters, `width: var(--bob-bm-seg)`,
    vertically centered with a little block overhang,
    `border-radius: var(--radius-s, 4px)`.
  - Fill: `background: color-mix(in srgb, var(--interactive-accent) ~20%, transparent)`.
  - Ring: `inset 0 0 0 1px` shadow at about 38% accent.
  - Motion: `transform: translateX(calc(var(--bob-bracket-index) * var(--bob-bm-seg)))`
    with `transition: transform 180ms cubic-bezier(.2,.8,.2,1), opacity 140ms ease-out`.
  - In `[data-bracket-mode="none"]` the thumb has `opacity: 0`.
- Segments `button.bob-bracket-mode-seg` (letters `s`, `w`, `n`):
  - Reset: no native button chrome (`appearance: none`, transparent, no border or
    shadow).
  - Size: `width: var(--bob-bm-seg)`, height about `1.35em`, `font: inherit`.
  - Color: `var(--text-muted)`, and `z-index` above the thumb.
  - Hover: `--text-normal` ink only.
  - Active (`.is-active`): `--text-normal`, weight 600, `cursor: default`.
  - `:focus-visible`: 2px `--interactive-accent` outline.
- Press feedback (driven by `data-bracket-flash` on the root):
  - `forward`: the `]]` key shows accent ink.
  - `backward`: the `[[` key shows accent ink.
  - `empty`: all three letters show `--text-normal`.
  - A static color rule holds while the attribute is present. Under
    `@media (prefers-reduced-motion: no-preference)`, add a short color fade-out
    keyframe (about 420ms for keys, about 600ms for `empty`).
- `.is-instant` suppresses the thumb's transform transition, so the thumb fades in at
  its letter instead of sliding.
- `@media (prefers-reduced-motion: reduce)`: no transform transition and no keyframes.
  The static flash color still shows for the flash lifetime.
- `@media (forced-colors: active)`:
  - Thumb: `background: Highlight`, `forced-color-adjust: none`.
  - Active letter: `HighlightText`.
  - Keys: `CanvasText`.
- Must work in light and dark themes with Obsidian variables only. Never clip within the
  status bar height.

### DOM and accessibility

The item is created once and reused forever. Never rebuild it, or the slide transition
breaks.

```
<div class="status-bar-item … bob-bracket-mode" data-bracket-mode="s"
     style="--bob-bracket-index: 0" role="group"
     aria-label="Bracket repeat mode: review walk">
  <span class="bob-bracket-mode-key is-backward" aria-hidden="true">[[</span>
  <span class="bob-bracket-mode-track">
    <span class="bob-bracket-mode-thumb" aria-hidden="true"></span>
    <button class="bob-bracket-mode-seg is-active" data-bracket-type="s"
            aria-pressed="true" aria-label="Review walk (]s / [s)">s</button>
    <button class="bob-bracket-mode-seg" data-bracket-type="w"
            aria-pressed="false" aria-label="Lane walk (]w / [w)">w</button>
    <button class="bob-bracket-mode-seg" data-bracket-type="n"
            aria-pressed="false" aria-label="Links (]n / [n)">n</button>
  </span>
  <span class="bob-bracket-mode-key is-forward" aria-hidden="true">]]</span>
</div>
```

### Copy (single-sourced in one frozen family table)

| type | letter | name        | keys      | first tooltip line                                |
| ---- | ------ | ----------- | --------- | ------------------------------------------------- |
| `s`  | s      | review walk | `]s / [s` | `]] / [[ repeat the review walk (]s / [s)`        |
| `w`  | w      | lane walk   | `]w / [w` | `]] / [[ repeat the lane walk (]w / [w)`          |
| `n`  | n      | links       | `]n / [n` | `]] / [[ open the next / previous link (]n / [n)` |
| none | —      | not set     | —         | `BRACKET_REPEAT_EMPTY_NOTICE`, reused verbatim    |

- Second tooltip line, active state:
  `s review · w lanes · n links — click a letter to switch`.
- Second tooltip line, none state: `… — click a letter to choose`.
- Root `aria-label`: `Bracket repeat mode: <name>`, or `Bracket repeat mode: not set`.

### Behavior

1. **Always visible on desktop.**
   - Mount the item in `onload` after the commands are registered.
   - After the mount succeeds, call `readBracketRepeatType()` so the stored mode is lit
     at startup, before any key press. With no status bar (mobile,
     `Platform.isDesktopApp === false`, or no `addStatusBarItem`), return before reading
     storage.
   - Mount at most once.
2. **Mode changes slide the thumb.**
   - `rememberBracketRepeatType` repaints only when the value actually changes.
   - The first paint, and the move from `none` to a family, apply `is-instant` so the
     thumb fades in at its letter. Remove `is-instant` on the next frame
     (`requestAnimationFrame`, with a `setTimeout` 0 fallback).
   - Every later change slides.
   - The painter skips all DOM writes when the paint key (the mode) is unchanged.
3. **`]]` / `[[` acknowledge the press.**
   - `repeatBracketWalk` delegates to the walk exactly as today. Delegation stays
     synchronous, so the Vim count is still consumed by the walk.
   - After delegating, it flashes `forward` or `backward` in a best-effort try/catch and
     returns the delegate's result unchanged.
   - A cold press still shows the existing notice and returns `false`, and additionally
     flashes `empty`.
   - Restarting a flash means: clear the timer, remove the attribute, read `offsetWidth`
     to force a reflow, set the attribute again, and start one timer that removes it
     (about 450ms for keys, about 650ms for `empty`).
4. **Clicking a letter selects that family.**
   - A click calls `rememberBracketRepeatType(type)`. That persists the choice and
     slides the thumb.
   - It never runs a walk, shows a notice, writes the vault, or writes a stats-journal
     event.
   - A `mousedown` handler calls `preventDefault()`, so the click never steals editor
     focus and the very next `]]` works.
   - Clicking the active letter is a no-op.
   - Wire listeners once with `this.registerDomEvent` when available, falling back to
     `addEventListener`.
5. **Never in the way.**
   - Every indicator call from `rememberBracketRepeatType` and `repeatBracketWalk` sits
     in its own try/catch. Indicator failures never change walk results, notices,
     storage, or timing.
   - On `onunload`, clear the flash timer and the pending `is-instant` frame. Obsidian
     removes the item itself.
   - No new commands, settings, hotkeys, notices, or vault writes.

## Implementation steps

1. **New fragment `plugins/bob-navigation-hotkeys/src/547-bracket-mode-indicator.js`.**
   List it in `src/fragments.json` immediately after `546-plugin-bracket-repeat.js`.
   Keep it at or below 1000 lines. Contents:
   - `BRACKET_MODE_FAMILIES`: a frozen, ordered `s, w, n` table with letter, name, keys,
     tooltip line, and segment aria-label.
   - `BRACKET_MODE_FLASH_MS`: frozen durations.
   - Pure `buildBracketModeView(type)`. It returns a frozen
     `{ mode, index, ariaLabel, tooltip, segments }`. Invalid or `null` input gives
     `mode: "none"`, `index: -1`.
   - `ensureBracketModeDom(el)`. It uses Obsidian's `createSpan` / `createEl` element
     helpers and returns `{ back, track, thumb, segments, forward }`. It is idempotent:
     it is cached on the element, and a second call returns the same parts.
   - `paintBracketModeIndicator(el, parts, view, previousMode)`. It sets
     `data-bracket-mode` and `--bob-bracket-index` (via `style.setProperty`, with a
     `setAttribute("style")` fallback). It toggles `is-active` / `aria-pressed`, applies
     the root `aria-label`, and sets the tooltip. On a fresh appearance it handles
     `is-instant`.
   - Mixin `BobNavigationHotkeysBracketModeIndicatorMixin` with these methods:
     - `setupBracketModeIndicator()`
     - `refreshBracketModeIndicator()`
     - `flashBracketModeIndicator(kind)`
     - `selectBracketModeFromIndicator(type)`
     - `teardownBracketModeIndicator()`
   - Read `Platform`, `setTooltip`, and `setIcon` (if used) from the existing `obsidian`
     module binding defensively, as `677-plugin-ref-pdf-open.js` does with
     `obsidian.Platform`.
2. **Wire state changes in `546-plugin-bracket-repeat.js`.**
   - In `rememberBracketRepeatType`, inside the "value changed" branch and after the
     memory and storage update, call `this.refreshBracketModeIndicator()` in a
     try/catch.
   - In `repeatBracketWalk`, call `flashBracketModeIndicator` as specified in
     Behavior 3.
   - Do not change `planBracketRepeat`, the storage key, or the notice text.
3. **Lifecycle in `490-plugin-lifecycle.js`.**
   - Initialize `bracketModeEl`, `bracketModeParts`, `bracketModePaintKey`,
     `bracketModeFlashTimer`, and `bracketModeInstantFrame` next to
     `this.bracketRepeatType = undefined`.
   - Call `this.setupBracketModeIndicator()` after the command registrations.
   - Call `this.teardownBracketModeIndicator()` from `onunload`.
4. **Install and export.**
   - Add the mixin to `690-install-plugin-mixins.js` right after
     `BobNavigationHotkeysBracketRepeatMixin`. The duplicate-method guard must stay
     green.
   - Export `BRACKET_MODE_FAMILIES`, `BRACKET_MODE_FLASH_MS`, `buildBracketModeView`,
     `ensureBracketModeDom`, and `paintBracketModeIndicator` from `700-exports.js`,
     beside the existing bracket-repeat helpers.
5. **CSS.** Add a `/* --- Bracket mode indicator (status bar) --- */` section to
   `plugins/bob-navigation-hotkeys/styles.css`, per the Visual spec.
6. **Unit tests: new `scripts/test-navigation-bracket-mode-indicator.cjs`.**
   - Add it to the `package.json` `test` list right after
     `scripts/test-navigation-bracket-repeat.cjs`.
   - Use the same obsidian stub style as the bracket-repeat test. Reuse or extend
     `createFragmentNode` from `scripts/navigation-hotkeys-harness.cjs` for a fake
     element that supports `createSpan`, `createEl`, attributes, classes,
     `style.setProperty`, and listeners.
   - Cover:
     - The view model for `s`, `w`, `n`, `null`, and garbage: mode, index, aria labels,
       and both tooltip lines. The none tooltip equals `BRACKET_REPEAT_EMPTY_NOTICE`
       plus the choose line. The family table is frozen and ordered.
     - Desktop mount: exactly one item, three letter buttons, and
       `data-bracket-mode="none"` on a cold device.
     - Mobile, `isDesktopApp === false`, or missing `addStatusBarItem`: no item, no
       throw, and `bracketRepeatType` stays `undefined`.
     - A second setup does not add a second item.
     - Startup restore: stored `w` paints `w` / index 1 with no key press. An invalid
       stored value paints `none`.
     - Every family command callback repaints to its family, and the DOM nodes are
       identical objects before and after. A same-family command does no DOM writes,
       because the paint key is unchanged.
     - `is-instant` is applied on first appearance and on `none` → family, but not on
       family → family.
     - Repeat forward and backward set `data-bracket-flash` to `forward` / `backward`,
       still delegate with the original args, and return the delegate's result. A cold
       repeat sets `empty`, shows the existing notice, and returns `false`. The flash
       timer clears the attribute.
     - A throwing indicator (for example, a painter or flash that throws) never changes
       delegation, return values, storage, or notices.
     - Clicking a letter sets the type and persists it, and no walk method is called.
       The `mousedown` handler calls `preventDefault`.
     - `onunload` clears pending timers.
   - `scripts/test-navigation-bracket-repeat.cjs` must pass unmodified.
7. **Visual preview: new `scripts/preview-bracket-mode.cjs`.**
   - Model it on `scripts/preview-lane-walk-notice.cjs`. Load the built bundle's
     helpers, use the Obsidian-Element shim, inline the production `styles.css`, and
     drive headless Chromium over CDP (`findChromium` honors `CHROMIUM_PATH`;
     `/usr/bin/google-chrome` and the Playwright cache exist on athena). Use
     `--out DIR`, defaulting to `/tmp/bob-bracket-mode-preview`.
   - Render, in light and dark themes, a mock status bar (`--status-bar-background`,
     item padding, the vim `🟢` prefix, and a short `⟳ Review 3/12 · NEXT 1/4` text
     stand-in on the left) holding these states:
     - `none`, `s`, `w`, and `n`.
     - `flash=forward` and `flash=backward` with reduced motion emulated, so the static
       flash color is visible.
     - `flash=empty`.
     - A forced-colors frame, if CDP emulation supports it (`Emulation.setEmulatedMedia`
       `forced-colors`).
   - Save one PNG per theme, plus an `index.html`.
8. **Docs and manifest.**
   - Add a short "Footer indicator" subsection to the "Bracket repeat" section of
     `docs/bob-navigation-hotkeys.md`. Cover what `[[ s w n ]]` shows, the unset state,
     the slide and the press acknowledgement, click-to-switch (no walk, keeps editor
     focus), desktop only, and that it never writes notes.
   - Update the sentence "Only the family command callbacks record a type" so it also
     names a click on a footer letter.
   - Append one sentence to the `manifest.json` description, for example: "The status
     bar always shows the bracket-repeat mode as `[[ s w n ]]` with the active walk lit;
     click a letter to switch."
   - Bump the manifest minor version from its current value (2.65.0 → 2.66.0 at plan
     time).
   - Run `npm run docs:gen` to regenerate the inventory and description rows. Never
     hand-edit generated rows or `main.js`.
9. **Build, verify, deploy** (all from the bob-plugins checkout):
   - `npm run build` and `npm run build:check`.
   - `node --test scripts/test-navigation-bracket-repeat.cjs scripts/test-navigation-bracket-mode-indicator.cjs`.
   - `npm run docs:inventory:check`, `npm test`, and `npm run validate`.
   - `node scripts/preview-bracket-mode.cjs --out /tmp/bob-bracket-mode-preview`. Open
     every PNG and check the alignment, centering, contrast, and spacing criteria below.
     Iterate on the CSS until they hold, then rerun the preview.
   - Best-effort real-Obsidian load check:
     `xvfb-run -a node scripts/obsidian-startup-smoke.cjs -e /snap/bin/obsidian`. It
     uses a scratch vault and never opens the live vault. If it cannot launch, say so in
     the final report instead of blocking.
   - Run `bob plugins sync` to deploy to the vault, as the bob-plugins `AGENTS.md`
     requires.

## Acceptance criteria

- **Startup.** On desktop Obsidian the status bar always shows `[[ s w n ]]` from
  startup. The stored mode is lit before any key press. A fresh device shows no thumb,
  and its tooltip carries the existing empty-notice text.
- **Family keys.** `]s`, `[s`, `]S`, `[S`, `Ctrl+Alt+J/K`, and the review footer click
  light `s`. `]w`, `[w`, `]W`, and `[W` light `w`. `]n` and `[n` light `n`. The thumb
  slides between letters. Its first appearance fades in place.
- **Repeat keys.** `]]` briefly lights the `]]` glyph and `[[` lights the `[[` glyph. A
  cold press lights the three letters and still shows
  `Nothing to repeat yet — press ]s, ]w, or ]n first`.
- **Click to switch.** Clicking a letter selects that family and persists it per device.
  The click does not walk and does not take focus from the editor, and the next `]]`
  repeats the chosen family.
- **Walks unchanged.** Walk behavior, notices, storage semantics, Vim count parity, and
  synchronous delegation are unchanged. No vault, stats-journal, or freshness writes
  come from the indicator.
- **Platforms and unload.** On mobile there is no item and no error. Unload leaves no
  timers behind.
- **Looks.** Preview PNGs in light and dark show:
  - Glyphs sit on one baseline, and the thumb is vertically centered within the
    status-bar height.
  - The active letter has clear contrast on the thumb, and inactive letters stay
    legible.
  - Nothing is clipped or misaligned next to the mock review footer text.
  - Forced colors and reduced motion behave as specified.
- **Green suite.** `npm test`, `npm run validate`, and `npm run docs:inventory:check`
  pass. `bob plugins sync` has deployed the build.

## Out of scope

- Changing what `]]` / `[[` do, the storage key, or per-device scope.
- Changing the Ledger Tools review footer or its `]s next` hint.
- Changing the vault vimrc.
- Showing walk progress in this item. The walk toasts and the review footer own that.
- Mobile UI.
