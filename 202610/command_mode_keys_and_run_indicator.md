---
tier: tale
title: "Command mode: Ctrl+[ escape, reliable Ctrl+N/P, and a footer run indicator"
goal:
  Ctrl+[ acts exactly like Esc in every command-mode stage, Ctrl+N/Ctrl+P walk the
  completion menu without losing focus or scrolling the selection out of view, and
  Obsidian's status bar shows a small, polished bob run indicator while commands run.
size: medium
decisions:
  indicator_label:
    ask: What should the footer run indicator show beside its spinner?
    choices:
      brand:
        Shows `bob 0:42`; the command name moves to the hover tooltip; narrow and steady
      title:
        Shows the run's title and clock, like today's chip; up to ~200px wide, shifts
        between runs
      icon: Spinner only (plus ×2 when several run); time and name appear only on hover
    default: brand
    why: Says bob is busy, and for how long, without crowding the status bar
    answer: brand
  indicator_runs:
    ask:
      Should the indicator count every run the command-mode engine owns, not only `:`
      sheet runs?
    choices:
      all: Sheet, palette, toast-action, and sibling-plugin API runs all light it
      sheet: Only runs started from the `:` sheet; palette and API runs stay invisible
    default: all
    why:
      One engine, one honest busy signal; a writer queued behind a hidden run would look
      stuck
    answer: all
proposed_by: bbugyi200.apollo.al
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 08:25:13
status: wip
---

# Command mode: `Ctrl+[` escape, reliable `Ctrl+N`/`Ctrl+P`, and a footer run indicator

## Goal

Polish Bob Command Mode (the `:` bottom sheet in Obsidian, plugin `bob-command-mode`) so
that:

1. `Ctrl+[` behaves exactly like `Esc` everywhere in command mode.
2. `Ctrl+N` / `Ctrl+P` reliably walk the completion menu: every press moves exactly one
   row, the highlight never leaves view, and typing keeps working afterwards.
3. Obsidian's footer (status bar) shows a small, beautiful indicator while a bob command
   runs in the background, and closes the loop gracefully when the command finishes.

All code lives in the **bob-plugins** linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<reason>"`), read its `AGENTS.md`, and work only in the
printed checkout. Rules from that file that matter here:

- `bob-command-mode` is a fragment plugin: edit `plugins/bob-command-mode/src/*.js`, add
  any new fragment to `src/fragments.json` (source order matters), run `npm run build`,
  and never hand-edit the generated `main.js`.
- Each hand-edited fragment must stay at or below 1000 lines. `170-sheet.js` (869) and
  `175-args-vim.js` (870) are close to the limit, so put new logic in a new fragment and
  keep edits there small.
- New `scripts/test-*.cjs` files must be added to the explicit list in the
  `package.json` `test` script.
- Run `bob plugins sync` after changes (see Validation).

## Why it misbehaves today (root causes)

Read these first. The fix targets causes, not symptoms.

1. **Focus is dropped on every selection move.** `CommandModeSheetModal.select()`
   (`src/170-sheet.js`) calls `render()`. That empties `contentEl` and recreates the
   `<input>`, but nothing refocuses it. In real Obsidian, focus falls to `<body>`, so
   the _next_ `Ctrl+N`/`Ctrl+P`, arrow, `Enter`, or typed key never reaches the
   `modalEl` keydown listener. That is the "unreliable" feel: the first press works,
   then the sheet goes dead. The Vim stage uses the same `select()`. The test harness
   hides this: `pressKey` in `scripts/modal-harness.cjs` falls back to `contentEl` when
   focus has left the modal.
2. **Other re-renders drop focus too:** the async clipboard re-render in `onOpen`
   (whenever the clipboard holds a URL), the `cmRefreshVimrcLint` re-render
   (`175-args-vim.js`), `activateRow` → confirm stage, and `enterOutput`.
3. **The selection is never scrolled.** `scrollSelectionIntoView()` is an empty stub, so
   the highlight walks off the 45vh list.
4. **Every re-render resets context.** The caret jumps to the end of the query (editing
   mid-query is broken), and the list's `scrollTop` resets to 0.
5. **Argument-mode order is wrong.** `moveArgumentCompletion` walks `argCompletions` in
   candidate order, but rows render grouped (`argCompletionGroups`). When bob's groups
   interleave, the highlight jumps around. Completion rows are never scrolled into view.
6. **Chord matching is brittle.** `cmSheetListAction` and the args/Vim handlers test
   `event.key === "n"`/`"p"`, so Caps Lock (`"N"`) and non-Latin layouts miss. `Ctrl+[`
   is not handled anywhere, and the pager maps every Ctrl chord to `none`.
7. **The status chip (`src/120-status-chip.js`) is text-first.** It rotates a `⟳` text
   glyph (`::before`), which wobbles because the glyph is off-center. It rewrites
   `textContent` every 250 ms, sets both `title` and `aria-label` (two tooltips in
   Obsidian), vanishes abruptly, and ignores runs adopted after a plugin reload
   (`runner.activeRecords()` only lists `live`).

## Design

### Keys: one alias layer

`Ctrl+[` **is** Escape and `Ctrl+N` / `Ctrl+P` **are** `↓` / `↑`, translated once at the
entry point. Every existing Escape and arrow branch then serves both chords, with no
parallel behavior to drift.

- `cmCommandModeKeyAlias(event)` returns `"Escape"`, `"ArrowDown"`, `"ArrowUp"`, or
  `null`:
  - `Ctrl+[`: `ctrlKey` and no Alt/Meta/Shift, with `event.code === "BracketLeft"` or
    `event.key === "["`. This matches `isCtrlLeftBracketKeydown` in
    bob-navigation-hotkeys.
  - `Ctrl+N` / `Ctrl+P`: Ctrl only, with `key.toLowerCase()` of `n`/`p` or `code`
    `KeyN`/`KeyP`.
  - `null` while `event.isComposing`.
- The aliased event is a plain object. It has the alias `key`, all modifiers false, and
  the original `target`. Its `preventDefault`/`stopPropagation` forward to the real
  event, and `defaultPrevented` reads the real one. The test harness and Obsidian's
  Escape handling then behave identically for `Esc` and `Ctrl+[`.
- Sheet (`handleSheetKeydown`, wrapped after the args-vim override): apply the Escape
  alias in every stage and the arrow aliases only in `list`, `args`, and `vim`. The
  results:
  - list: `Ctrl+[` closes the sheet.
  - args and vim: back to the list.
  - confirm card: back.
  - output: back to the list. In pager search or stop-confirm it cancels, as Esc does.
- Standalone output pager (`CommandModePagerModal.prototype.handlePagerKeydown`): apply
  the Escape alias.
- Record every handled event in a `WeakSet` so no key is ever handled twice (the modal
  listener plus the scope fallback below).

### Keys: never lose focus, always show the selection

- **Focus-preserving render.** Wrap the final `CommandModeSheetModal.prototype.render`.
  - Before rendering, capture whether the input was focused, its
    `selectionStart`/`selectionEnd`/`value`, and the list's `scrollTop` with the current
    stage and query.
  - After rendering, check whether focus is on a live element inside the sheet. If not,
    move it to the stage's home: the query input for `list`/`args`/`vim`/`confirm`, or
    the sheet itself for `output` (`modalEl` with `tabindex="-1"`, as the standalone
    pager does in `150-pager.js`).
  - Restore the caret when the input value is unchanged; otherwise put it at the end.
  - Restore `scrollTop` when the stage and query are unchanged; otherwise start at the
    top.
  - Finally, ensure the selection is visible.

  This one wrapper fixes causes 1, 2, and 4 for every current and future render path.

- **Real `scrollSelectionIntoView()`** for list and Vim rows, plus the same for argument
  completion rows.
  - Scroll only the list container. Never call an unbounded `scrollIntoView` that could
    nudge the modal or the workspace.
  - Compute the union rect of the selected row, its about line, and its section heading
    when the row is the first in its group. Adjust `listEl.scrollTop` by the minimum
    needed, keeping a one-row margin (a Vim `scrolloff=1` feel), and clamp the margin
    when the list is short.
  - When the selection is the first row, scroll fully to the top so the first heading
    shows.
  - Fall back to `scrollIntoView({ block: "nearest" })` only when rects are unavailable.
  - Use instant scrolling, not smooth: held-key auto-repeat must keep up.
- To support this, make small edits in `170-sheet.js` and `175-args-vim.js` that keep
  element handles: `this.listEl`, the existing `this.rowEls`, the section heading
  elements keyed by their first row, and the about-line element. In argument mode keep
  `this.argListEl` and `this.argRowEls`.
- **Argument mode walks what you see.** `moveArgumentCompletion` and Tab's no-selection
  default use `argCompletionGroups.flatMap((g) => g.rows)` (rendered order), not
  `argCompletions`. Wrapping at both ends is unchanged.
- **Clicks don't steal focus.** List rows and completion rows `preventDefault()` on
  `mousedown`, following the block-id-prompt pomodoro picker. Clicking still activates
  the row.
- **Scope fallback, belt and braces.** On open, if `this.scope?.register` exists,
  register a catch-all handler with `(null, null)`, as the image-zoom modal in
  bob-navigation-hotkeys does.
  - Act only when the key's target is outside the sheet's `modalEl`, meaning focus fell
    to `<body>` (for example, the user clicked the sheet chrome or a tail refresh
    rebuilt a focused node).
  - In that case, move focus home and route the event through `handleSheetKeydown`.
  - Return `false` only when the key was handled. Never act on keys targeted inside the
    sheet.
- Full re-render is kept. A list of about 100 rows renders in a few milliseconds, so
  held-key auto-repeat stays smooth. Do not restructure the stage architecture.

Footer hint text (`cmSheetFooterHints` and the args/Vim footers) should advertise the
chords, with `esc` left as is:

- list:
  `⌃N/⌃P select · ↵ run · ⇥ args · ⇧↵ preview · ⌃↵ run now · ? help · :: vim · esc close`
- args: `⌃N/⌃P completion · ⇥ accept · ↵ run · ⇧↵ preview · ⌃↵ run now · esc back`
- vim: `⌃N/⌃P mapping · ↵ run Vim ex · esc back`

Add `scroll-margin-block` to `.bob-cmd-row` and `.bob-cmd-completion-row` only if the
manual margin math is not used. Pick one mechanism, not both.

### Footer run indicator (redesign of the status-bar chip)

Keep the existing status bar item (`addStatusBarItem`, class `bob-cmd-chip`) and its 400
ms reveal, so fast reads never flash. Add Obsidian's `mod-clickable` for native hover
styling. Rebuild it as an icon-led indicator.

**Anatomy.** Create three children once and only update them afterwards:

- `.bob-cmd-chip-mark` (`aria-hidden`): an 11px CSS-drawn ring, 1.5px stroke, with a
  faint track (accent at about 25% via `color-mix`) and an accent arc
  (`--interactive-accent`) spinning at 0.9s linear. The ring is drawn in CSS, not as a
  font glyph or icon dependency, so it stays perfectly centered and crisp at any DPI.
- `.bob-cmd-chip-label`: `bob` in semibold `--text-normal`, 11px `--font-interface`. Add
  `×N` in `--text-muted` when more than one run is active.
- `.bob-cmd-chip-clock`: elapsed time of the longest active run (`cmFormatClock`), in
  `font-variant-numeric: tabular-nums` and `--text-muted`, so the width never jitters.

> [!decision] indicator_label = title The label shows the newest run's title, ellipsized
> at about 200px, instead of `bob`. With several runs it falls back to `bob ×N`. The
> clock is unchanged.

> [!decision] indicator_label = icon Hide the label and clock. Show only `×N` beside the
> ring when several runs are active. The elapsed time moves to the first tooltip line.

**States.** Exactly one tone class applies on the root. Among visible active runs,
overdue takes priority over running, and running over queued.

| State                                   | Look                                                                                                                                                                                                                      |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `is-running`                            | Accent arc spinning                                                                                                                                                                                                       |
| `is-queued` (every visible run queued)  | Dashed `--text-faint` ring, no spin; the clock shows the wait                                                                                                                                                             |
| `is-overdue` (a writer past its budget) | Arc and clock in `--color-orange` (fallback `#d89a36`); the spin slows to 1.8s ("laboring"); never red, since it has not failed                                                                                           |
| `is-settled` + `is-<outcome>`           | The ring becomes the outcome glyph from `CM_TOAST_META` (`✓ ◐ ✗ ? ■`, shared with the toasts) in the toast palette (green, orange, red, red, muted); the clock shows the final duration; holds `CM_CHIP_SETTLE_MS` = 1600 |
| `+ is-leaving`                          | The last `CM_CHIP_LEAVE_MS` = 300 of the settle window: opacity fades to 0 over 240 ms, then hidden                                                                                                                       |

**Settle rules.**

- Only a run that was actually shown (alive at least 400 ms) settles. Fast reads never
  flash.
- If other runs are still active, there is no settle; the running state continues.
- If a new run appears during a settle, it replaces the settle immediately.
- The 300 ms leave window is longer than the 250 ms refresh tick, so at least one tick
  always lands in it and the CSS transition plays.

**Motion.**

- Appearing: a 160 ms fade plus a 2px rise, via a CSS animation on the root that replays
  whenever `display` returns from `none`.
- Settle: the glyph pops in with a 160 ms scale-in from 0.6.
- `prefers-reduced-motion`: no spin (a static three-quarter ring still reads "busy"), no
  rise or scale, and an instant hide instead of the fade.

**Tooltip.**

- Use Obsidian's native tooltip, `aria-label` plus `data-tooltip-position="top"`, with
  multi-line content. Remove the `title` attribute.
- Header line, for example `bob · 2 running`.
- For each run, newest first, capped at 4 with `…and N more`:
  - `Title · 0:42`
  - `bob <argv without the binary path>`
  - optional `Last: <progress line>` (existing `commandModeProgressText`)
  - a suffix of `queued`, `past its ~3 min budget`, or `adopted after reload`
- Final line: `Click to open output`.
- While settled, show `Title · Completed in 0:42` (using the toast label) and the click
  line.

**Interaction.**

- Click, `Enter`, and `Space` open the indicator's current target run: the newest active
  run, or the settled run during a settle.
- The plugin stores the render result's `runId` (for example
  `this.commandModeChipRunId`). The existing status keydown handler uses it instead of
  recomputing from `activeRecords()`.
- Keep `role="button"` and `tabindex="0"` while visible. Do not add `aria-live`, because
  the clock ticks every second.

**Implementation shape.**

- Split model from paint:
  - `cmStatusChipModel(records, nowMs, state, progressFor, options)` is pure and returns
    `{ visible, tone, outcome, label, count, clock, tooltip, runId }`.
  - `cmRenderStatusChip(element, records, nowMs, onOpen, progressFor, state)` keeps its
    current signature, with `state` optional for the preview script and old callers. It
    paints idempotently: create children on first paint, then touch only changed text,
    classes, and attributes.
  - Never recreate the mark. Recreating it restarts the CSS spin and stutters every 250
    ms.
- Settle state is plugin-owned:
  `{ shown: Set<runId>, settle: null | { runId, title, outcome, durationMs, atMs } }`.
  - Each paint adds visible runs to `shown`.
  - `commandModeFireEvent` (`090-plugin.js`) sets `settle` when `CM_RUN_FINISHED_EVENT`
    fires for a shown run. The payload already carries `runId` and `outcome`; read the
    title and duration from the record.
- Add the constants `CM_CHIP_REVEAL_MS` (400), `CM_CHIP_SETTLE_MS`, and
  `CM_CHIP_LEAVE_MS` to `010-load.js`.
- Adopted runs, so the indicator never lies after a reload:
  - The runner (`060-runner.js`) keeps adopted unfinished records in a map while their
    adoption poll runs, and removes them when finalized or found finished.
  - It clears the map in `dispose()`.
  - `activeRecords()` returns live plus adopted runs, deduplicated by id, with adopted
    ones flagged. Today its only callers are the chip paths in `090-plugin.js`.
- Run scope follows the `indicator_runs` decision.

> [!decision] indicator_runs = sheet Filter the records passed to the model to those
> whose `trigger` starts with `sheet` (`sheet`, `sheet-recent`, `sheet-preview`,
> `sheet-help`, `sheet-apply`, `sheet-rerun`), and record settles only for those runs.

## Implementation steps

1. **New fragment `src/176-sheet-keys.js`**, listed right after `175-args-vim.js` in
   `fragments.json`. It contains the alias helper, the aliased-event builder, the
   handled-event `WeakSet`, the sheet keydown wrapper, the pager keydown wrapper, the
   focus-preserving `render` wrapper, the real `scrollSelectionIntoView`, the
   list-scroll helper, the `onOpen`/`onClose` wrappers for the scope fallback, and the
   row `mousedown` guard (wrap `renderRow`). Start with a one-line header comment in the
   style of its siblings.
2. **`170-sheet.js`** (small edits): keep `this.listEl`, the section heading elements,
   and the about element in `renderList`/`renderRow`. Leave `select()` and
   `refocusInput()` behavior to the new wrappers.
3. **`175-args-vim.js`** (small edits):
   - Keep `this.argListEl` and `this.argRowEls` in `renderArguments`, and add the
     completion-row `mousedown` guard.
   - Change `moveArgumentCompletion` and Tab's default pick to rendered order.
   - Update the args and Vim footer hint strings.
4. **`160-sheet-rank.js`**: update the list hint in `cmSheetFooterHints`. Leave
   `cmSheetListAction`'s own `n`/`p` checks in place; the alias layer makes them
   reliable.
5. **`120-status-chip.js`**: rewrite as model plus idempotent paint, per the design.
6. **`090-plugin.js`**:
   - Status item: add `mod-clickable`, set the initial attributes, use the stored chip
     target run for `Enter`/`Space`, and pass the settle state into each paint.
   - Record settles in `commandModeFireEvent`.
   - Apply the `indicator_runs` filter if `sheet` is chosen.
7. **`060-runner.js`**: track adopted unfinished records, and have `activeRecords()`
   include them.
8. **`010-load.js`**: add the chip constants. **`990-exports.js`**: export the new
   helpers and constants for tests.
9. **`styles.css`**:
   - Replace the `.bob-cmd-chip::before` glyph block with the new chip anatomy, states,
     motion, and reduced-motion rules. Reuse the toast palette variables and fallbacks
     already in the file.
   - Add list-scroll styling only if chosen in the scrolling step.
10. **Preview** (`scripts/preview-command-mode.template.html` and the case list or
    report text in `scripts/preview-command-mode.cjs`):
    - Render chip cases inside a mock status-bar strip: one run, two runs, queued,
      overdue, settled completed, and settled failed, in light and dark.
    - The preview loads only `020`, `110`, `120`, `140`, and `150`, so the chip may
      depend only on those fragments plus `010` constants it inlines or receives. Either
      keep it self-contained or add the needed fragment to the preview loader.
11. `npm run build`.

## Tests

Use `scripts/command-mode-harness.cjs` and `scripts/modal-harness.cjs`. Add
`scripts/test-command-mode-sheet-keys.cjs` and
`scripts/test-command-mode-status-chip.cjs` (register both in `package.json`), and
update the existing chip test in `test-command-mode-outcomes-api.cjs` to the new model.

**Keys and focus:**

- Three `Ctrl+N` presses move three rows. `harness.activeElement === modal.inputEl`
  after every press, `Ctrl+P` walks back, and both wrap. Repeat in the Vim stage.
- Variants: `{ key: "N", ctrlKey: true }` (Caps Lock) and `{ key: "т", code: "KeyN" }`
  navigate. `Ctrl+Shift+N` and `Ctrl+Alt+N` do not.
- Focus and caret survive each re-render path:
  - typing (input event), with the caret kept mid-query
  - the async clipboard re-render after open
  - `cmRefreshVimrcLint`'s re-render
  - entering and leaving args and Vim
  - the confirm stage
  - output (focus goes to the sheet, not the input)
- Scrolling:
  - Give list and row stubs deterministic `getBoundingClientRect`, `scrollTop`, and
    `clientHeight` values (extend `ElementStub` additively, or set them per test).
  - Assert the selected row, its about line, and its section heading end within the list
    viewport with the one-row margin.
  - Assert `scrollTop` is preserved across a same-query re-render and reset on a query
    change.
  - Assert wrapping to the first row scrolls to the top.
- Argument mode: with interleaved completion groups, `Ctrl+N` follows rendered order;
  Tab with no selection accepts the visually first row; the selected completion is
  scrolled into view.
- `Ctrl+[` matches Escape in every stage:
  - list: closes
  - args: back to the list with the base-path query
  - Vim: back
  - confirm: back
  - output: back to the list
  - output search editing: cancels
  - standalone pager: identical to `Escape`

  `Ctrl+Shift+[` and `Ctrl+Alt+[` are not aliased.

- Scope fallback: attach a scope stub (`register`/`unregister` recording handlers) to
  the sheet before `open()`. Do not change `ModalStub` globally; other plugins' tests
  share it.
  - A `Ctrl+N` delivered to the catch-all with `target = body` moves the selection once
    and refocuses the input.
  - The same key with a target inside the sheet is ignored by the fallback.
  - An event routed through both paths moves the selection only once.
- A row `mousedown` is `defaultPrevented`.

**Indicator:**

- Hidden below 400 ms, and never settles for such runs.
- Running: one `.bob-cmd-chip-mark` element whose identity is unchanged across many
  paints; the label and clock follow the decision; `tabular` clock text; `aria-label`
  contains the title, `bob <argv>`, and `Last: …`; no `title` attribute; `mod-clickable`
  is set.
- Two runs: the `×2` count, the clock from the longest run, and click opening the
  newest.
- Queued-only gives `is-queued`. An overdue writer gives `is-overdue` and the budget
  tooltip.
- Settle: after a run-finished event for a shown run, `is-settled is-completed` with `✓`
  for 1600 ms, then `is-leaving`, then hidden. A failed run shows `✗` and `is-failed`. A
  new run during a settle shows running at once. Click and `Enter` during a settle open
  the settled run.
- Adopted unfinished runs appear in `activeRecords()` (flagged) and disappear once
  finalized.
- If `indicator_runs = sheet`: a palette-triggered run never lights the indicator.

## Documentation

Update `docs/bob-command-mode.md` in bob-plugins. Keep the term "status-bar chip" so the
bob-cli guide (`docs/guides/run-bob-from-obsidian.md`) stays accurate and needs no edit.

- **Keyboard and Vim:**
  - `Ctrl+[` is Escape everywhere in command mode: the sheet, the argument and Vim
    stages, the confirm card, and both output views.
  - `Ctrl+N` / `Ctrl+P` (any letter case or layout) move the selection, keep it in view,
    and leave the cursor in the query.
- **Feedback:** rewrite the chip paragraph to describe:
  - the ring, `bob`, and clock (per the decision), plus the count
  - amber overdue and dashed queued states
  - the brief outcome glyph and fade
  - the hover tooltip, and click/`Enter` opening the output
  - runs adopted after a reload
  - the scope per `indicator_runs`
- Run `npm run docs:check`. The generated command inventory does not change.

## Validation and deploy

- In the bob-plugins checkout, run `npm run build`, `npm test` (which includes
  `build:check` and the docs inventory check), `npm run validate`, and
  `npm run docs:check`. All must pass. Report failures verbatim.
- If a Chromium binary is available (`CHROMIUM_PATH` or `which`), run
  `node scripts/preview-command-mode.cjs --out <tmpdir>` and view both PNGs. Iterate on
  `styles.css` until every chip state is crisp, aligned, and legible in light and dark.
  If no Chromium exists on the host, say so in the final report; do not claim visual QA.
- Deploy as `AGENTS.md` requires. Run
  `bob plugins sync --repo <bob-plugins checkout> --no-pull --dry-run`, then the same
  command without `--dry-run`, and report the result. If the host lacks the Obsidian
  smoke prerequisites (`obsidian`, `xvfb-run`), record that as a limitation per
  `docs/develop/deployment.md`; do not claim a rollout pass.
- End the final report with this manual checklist for Bryan to run in real Obsidian:
  1. Press `:` and hold `Ctrl+N` through the whole list. The highlight stays in view,
     wraps, and typing works afterwards. Do the same with `Ctrl+P`, and in argument mode
     on a command with many completions.
  2. With a URL on the clipboard, press `:` and then `Ctrl+N`. It still moves.
  3. `Ctrl+[` and `Esc` behave identically in the list, args, Vim, confirm, sheet
     output, and the standalone pager.
  4. While the sheet is open, `Ctrl+P` never opens Obsidian's command palette and
     `Ctrl+N` never creates a note.
  5. Run `vault-sync run` from the sheet. After about 0.4 s the indicator shows a smooth
     spinner and ticking clock. Hover shows the command, click opens the output, and on
     finish a ✓ shows briefly and fades.

  If check 3 shows real `Esc` and `Ctrl+[` diverging in a nested stage (for example,
  Obsidian's own modal Escape closing the sheet first), file a bug task bead via
  `/sase_new_task` with the observed stage and behavior.

## Out of scope

- Keeping the query input persistent across renders, or restructuring stages. The
  focus-preserving wrapper makes full re-render safe.
- New chords beyond `Ctrl+[`, `Ctrl+N`, and `Ctrl+P`. No pager `j`/`k` changes.
- A persistent idle footer icon. The indicator exists only while bob is busy or
  settling.
- bob-cli changes. The guide stays accurate because the reference keeps the "status-bar
  chip" term.
