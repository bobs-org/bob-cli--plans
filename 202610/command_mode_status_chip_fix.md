---
tier: tale
title: Fix the invisible Bob Command Mode run indicator
goal:
  Bob Command Mode's status-bar run indicator appears for every command-mode run lasting
  over 400 ms.
size: medium
decisions:
  sheet_running_line:
    ask:
      Should the command sheet also show a running line while it waits on a report,
      preview, or help run?
    choices:
      sheet_line: Sheet footer shows a spinner and a running line until the output opens
      chip_only: Only the status-bar chip is fixed; report runs stay silent in the sheet
    default: sheet_line
    why: Report runs keep the sheet open over the status bar, which can hide the chip
    answer: sheet_line
proposed_by: bbugyi200.apollo.ax
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 13:46:24
status: wip
---

# Fix the invisible Bob Command Mode status-bar run indicator

## Problem

Running a bob command through Bob Command Mode (the `:` sheet, the palette, toast
actions, or the sibling-plugin API) never shows the status-bar run indicator. Commit
`b13d634` ("feat(command-mode): Ctrl-[ escape, reliable Ctrl-N/P, and brand run
indicator") was meant to add it. `docs/bob-command-mode.md` § Feedback describes it: a
spinning ring, `bob`, and a clock, shown after a run has lasted 400 ms, with a short
outcome glyph when the run settles.

All work is in the **`bob-plugins` linked repo**. Open it with the `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`) and use only the path that command prints.
Read its `AGENTS.md` first. bob-cli itself does not change.

## Root cause (confirmed)

`cmChipEnsureChildren` in `plugins/bob-command-mode/src/120-status-chip.js` checks
whether each anatomy node is attached:

```js
const present = (element.children || []).includes(node);
```

In Obsidian, `element.children` is a live DOM `HTMLCollection`. It is iterable and
indexed but has **no `includes` method**. The first paint of a visible model (any run
that has been active for 400 ms or more, or a settle) throws
`TypeError: (element.children || []).includes is not a function`. The throw happens
before `cmRenderStatusChip` reaches `element.style.display = ""`, so the chip stays
`display: none` forever. `commandModeRefreshStatusChip` (in `src/090-plugin.js`) runs
from a 250 ms `setInterval` with no guard, so the same uncaught error recurs about four
times a second while a run is active. The settle paint fails the same way, so the
outcome glyph never shows either.

The tests miss this because `ElementStub` in `scripts/modal-harness.cjs` models
`children` as a plain `Array`, so `.includes` works there. A probe that passed an
element whose `children` is an HTMLCollection-like object (indexed, `length`, `item()`,
iterable, no Array methods) reproduced the failure exactly: it threw that `TypeError`
and `style.display` stayed `"none"`. The pre-`b13d634` chip only set `textContent`, so
this is a regression from the brand rewrite.
`scripts/preview-command-mode.template.html` calls the same function in a real browser
and was broken the same way; the fix repairs it without changing the template.

## Changes

### 1. Make the chip paint DOM-safe (`src/120-status-chip.js`)

- In `cmChipEnsureChildren`, never call Array methods on `element.children` directly.
  Read a snapshot with `Array.from(element.children || [])`, which works for both a real
  `HTMLCollection` and the stub's array. Use it for both the class lookup (`find`) and
  the "already attached?" check before appending. A snapshot taken before appending is
  correct: mark, label, and clock are distinct nodes, so appending one cannot change
  whether another is present.
- Keep the existing contract: anatomy order mark → label → clock, appended exactly once
  and never reordered, and repeated paints reuse the same mark node, so the spin is
  never restarted.
- Audit the rest of `120-status-chip.js` for any other Array-only use of a DOM
  collection or a stub-only property. As of planning, `.includes` on `children` is the
  only one; `child.classes || []` already falls back safely on real nodes.

### 2. Add tests that use real DOM semantics (`scripts/test-command-mode-status-chip.cjs`)

Add a small DOM-faithful fake element local to this test file. Do not change the shared
`ElementStub`; many other suites depend on its array `children`. The fake needs:

- a `children` getter returning a fresh HTMLCollection-like object (numeric indices,
  `length`, `item(i)`, `Symbol.iterator`, **no** Array methods);
- a string `className`, plus a `classList` (`add`, `remove`, `toggle`, `contains`) kept
  in sync with it;
- a `style` object, `setAttribute` / `getAttribute` / `removeAttribute`, `appendChild`,
  and `textContent`;
- `addClass` and `addEventListener`, so the plugin path below can use it as a status-bar
  item.

While the new tests run, install a `global.document.createElement` that returns this
fake, then restore the previous `document`. Follow the `installDocument` pattern in
`scripts/command-mode-harness.cjs`.

New cases:

1. **First visible paint on a DOM-like element**: a run started 5 s ago paints without
   throwing. `style.display === ""`, and the children are exactly `bob-cmd-chip-mark`,
   `bob-cmd-chip-label`, `bob-cmd-chip-clock` in that order. The label is `bob`, the
   clock is `0:05`, `is-running` is set, and pre-existing classes such as
   `status-bar-item` and `bob-cmd-chip` are kept.
2. **Idempotent repaint**: a second paint keeps the identical mark node and still has
   three children. With two active runs the label is `bob ×2`.
3. **Settle then hide**: after a shown run finishes, the settle paint shows `is-settled`
   plus the outcome class and glyph. Once the settle window expires, the paint sets
   `style.display === "none"`.
4. **Plugin path**: build the plugin with the command-mode harness, as the existing
   plugin-level tests do (for example `scripts/test-command-mode-outcomes-api.cjs`).
   Make `addStatusBarItem` return the DOM-faithful fake, start the status chip, and make
   the runner report one running record that is 5 s old (stub
   `commandModeRunner.activeRecords` or start a real fake run, whichever the harness
   supports cleanly). Calling `plugin.commandModeRefreshStatusChip(now)` must not throw,
   must return `visible: true`, and must leave the status item displayed. Clear any
   interval the plugin starts, for example through `onunload()`.

Confirm the new DOM-faithful cases **fail on the unfixed code** (stash the fix, or run
them against `git show HEAD:...`) and pass with it. Keep the existing stub-based cases
unchanged.

### 3. Decision `sheet_running_line`

> [!decision] sheet_running_line = sheet_line In `src/170-sheet.js`, `runReport`, the
> preview branch of `runPreview` (rows with `hasPreview`), and `showHelp` all
> `await runner.run(...)`, which settles only when the run finishes. Until then the
> sheet stays on its list stage with no running feedback, while the bottom sheet (up to
> 920px wide, flush to the window bottom, above the status bar) can cover the chip.
>
> - Before awaiting, set a sheet field such as `this.waitingRun = { title }` and
>   `render()`. Clear it on **every** settle path (success, `!settled.runId` failure,
>   and thrown or rejected cases) before the next render or `enterOutput`.
> - In `renderFooter()`, when `this.waitingRun` is set and there is no warning `hint`,
>   render a footer hint containing a small spinning ring and `running <title>…`. Reuse
>   the existing `bob-cmd-spin` keyframes with a new sheet-scoped class in `styles.css`
>   (for example `.bob-cmd-sheet-hint .bob-cmd-sheet-spin`). Honour the existing
>   `prefers-reduced-motion` block. The text alone must still read as "running".
> - A warning hint keeps its current priority over the running line.
> - Add focused cases to `scripts/test-command-mode-sheet.cjs`, next to the existing
>   `runReport` / `showHelp` tests. Use a runner whose `run` returns a controllable
>   pending promise, assert that the footer shows the running line while pending, and
>   assert that the line is gone after settle (both on output and on a start failure
>   that sets `hint`).
> - Keep `170-sheet.js` at or under 1000 lines (it is about 873 now).
> - Update `docs/bob-command-mode.md` § Command sheet with one sentence: while a report,
>   preview, or help run is in flight, the sheet footer shows a running line until its
>   output opens.

> [!decision] sheet_running_line = chip_only Skip this section entirely. The sheet and
> its docs stay unchanged.

### 4. Docs

`docs/bob-command-mode.md` § Feedback already describes the intended chip behavior, so
no edit is needed for the chip fix itself. Do not document the 400 ms reveal as new; it
is existing behavior. Update the docs only as required by the decision above.

## Build, verify, deploy

Run these from the `bob-plugins` checkout:

1. `npm run build` (regenerates `plugins/bob-command-mode/main.js` from `src/`; never
   hand-edit `main.js`).
2. `node --test scripts/test-command-mode-status-chip.cjs scripts/test-command-mode-sheet.cjs scripts/test-command-mode-outcomes-api.cjs`
3. `npm test` (includes `build:check` and the docs inventory check), then
   `npm run validate` and `npm run docs:check`.
4. Optional: `node scripts/preview-command-mode.cjs` and check that the generated
   preview's chip cases now render (they previously threw in the browser).
5. `bob plugins sync` (required by the repo's `AGENTS.md` after any change), then
   confirm `bob plugins list` shows `bob-command-mode` deployed from the new head.
6. Commit the `bob-plugins` change through the normal SASE commit flow (a
   `fix(command-mode): ...` subject).

In the final report, tell Bryan two things. First, Obsidian on each host that runs it
(including the Mac) needs `bob plugins sync` there and a plugin reload or Obsidian
restart. Second, the chip deliberately stays hidden for runs that finish in under 400
ms, so a slower command is the reliable check: `vault-sync run`, or a report such as
`plan`.

## Out of scope

- The 400 ms reveal delay and the rule that sub-400 ms runs never settle-flash are the
  documented design; this plan does not change them.
- Do not touch the duplicate click wiring (`element.onclick` set in paint and the
  plugin's own `click` listener both open the output). It is harmless, because the
  second call reuses the open pager, and it is not part of this bug.
