---
tier: epic
title: Ctrl+W N — numbered tab hints and an in-place tab jump
goal: "In Bob's Obsidian, every main-area tab carries a quiet numeral (1–9, in reading
  order across splits) beside its title, and Ctrl+W followed by that digit focuses
  exactly that tab where it lives — selected inside its own split, never moved, cloned,
  or reopened — while the numerals brighten into accent key caps when the Ctrl+W card
  appears and the destination numeral flashes once to confirm the jump.

  "
phases:
  - id: model
    title: Tab catalog, digit planner, and main-area enumeration
    depends_on: []
    size: medium
    description: "model: add the pure tab catalog, digit normalization, focus-tab
      planner, numeral visibility verdict, and view-model tab data, plus read-only
      main-area and hotkey readers that do not need a main-area active leaf.

      "
  - id: actions
    title: In-place tab-jump executor
    depends_on:
      - model
    size: medium
    description: "actions: execute focus-tab plans losslessly (load if deferred, session
      check, setActiveLeaf, invariants, restore), fold the selectedTab fix through the
      move executor, and prove it on real Obsidian with a new e2e tabs group.

      "
  - id: paint
    title: Tab numerals, card tabs row, and previews
    depends_on:
      - model
    size: medium
    description: "paint: build the dormant tab-numeral stamper and CSS (idle, lit, here,
      flash, stacked) and the card's pill ranges and tabs row, with a light/dark preview
      harness and unit tests.

      "
  - id: prefix
    title: Digits in the Ctrl+W prefix, palette commands, go live
    depends_on:
      - actions
      - paint
    size: medium
    description: "prefix: wire digits into the Ctrl+W state machine with an
      armed-catalog freeze, light the numerals with the card, flash on success, add Jump
      to tab 1–9 commands, flip the dormant seams, bump the minor version, and prove it
      on real Obsidian.

      "
  - id: finish
    title: Docs, visual acceptance, and polish
    depends_on:
      - prefix
    size: medium
    description:
      "finish: document digits and numerals on the Window keys page, run real-Obsidian
      visual acceptance in light and dark, fix every polish issue, and sync."
proposed_by: bbugyi200.apollo.research.4y.linker.w0
proposed_with: claude/opus
create_time: 2026-11-16 11:00:59
status: wip
---

- **PROMPT:**
  [prompts/202611/obsidian_ctrl_w_numbered_tab_jump.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_ctrl_w_numbered_tab_jump.md)

# Plan: Ctrl+W N — numbered tab hints and an in-place tab jump

## Context

Epic `bob-cli-m0` shipped the `Ctrl+W` window prefix in `bob-navigation-hotkeys`:
`h`/`l`/`w` focus splits, `H`/`L` move a tab, `v` splits, with an armed ring, a card
after a 300 ms pause, and palette commands. Its plan
(`plan:202611/obsidian_ctrl_w_window_prefix.md`) reserved digits for later; today an
armed digit is consumed as "‹N› isn't a window key".

This epic gives digits to tabs. The design basis is the consolidated research
`research:202611/obsidian_ctrl_w_numbered_tab_jump/obsidian_ctrl_w_numbered_tab_jump.md`,
and Bryan accepted every requirement it recommends. Read it with `sase artifact read`
before starting any phase. Its sibling
`research:202611/obsidian_ctrl_w_numbered_tab_jump/obsidian_ctrl_w_numbered_tab_jump__cld.md`
holds the real-Obsidian probe (and mock screenshots) that the executor and the numeral
technique rely on. Also read the m0 plan: its arming gate, key handling, hint, and
reliability rules all still hold. This plan only adds to them, and where it refines the
research it says so (see "Design refinements over the research").

Facts this plan builds on, verified in real Obsidian 1.15.1 by the research probe and by
the m0 spike verdict (`sase bead read bob-cli-m0.1`):

1. `workspace.setActiveLeaf(leaf, {focus: true})` on an unselected tab in another split
   selects it inside its own split (`leaf.parent` unchanged), focuses it, and leaves the
   source split's selected tab alone. A background tab's scroll survives plain tab
   switching.
2. `leaf.loadIfDeferred` exists. After an app restart, background tabs are deferred, and
   they are the common jump target.
3. `leaf.tabHeaderEl` exists per leaf, and header DOM order equals `group.children`
   order. A `data-` attribute on `.workspace-tab-header-inner` survives rename,
   `openFile`, `onLayoutChange()`, and pin toggles.
   `.workspace-tab-header-inner::before` is unused by Obsidian, and Markdown tabs hide
   their icon slot.
4. Inserting a 16 px numeral shifts each title by about 21 px. Numerals must therefore
   be persistent; inserting them per gesture would jolt the whole tab bar on every
   `Ctrl+W`.
5. `WorkspaceTabs.selectedTab` is `undefined` in 1.15.1 and `currentTab` is a numeric
   index, so the selected tab is `group.children[group.currentTab]`. The shipped
   `267-window-prefix-actions.js` still reads `selectedTab` in its selected-leaf
   resolver, move executor, rollback, and diagnostics, so a failed join's rollback never
   reselects the destination's previous tab. This epic fixes it (the issue is recorded
   as a `DISCOVERED ISSUE` note on `bob-cli-m0`; no task bead exists).
6. `snapshotWindowPrefixLayout()` returns `null` whenever the active leaf is outside the
   main area (focus in a sidebar). Numerals therefore need an enumeration that does not
   depend on the active leaf.
7. A pushed catch-all keymap `Scope` intercepts bound chords (spike finding 4c), so
   `Ctrl+W Ctrl+3` reaches Bob before Linux's native `Mod+3` "Go to tab #3".
8. `iterateRootLeaves` stops when its callback returns a truthy value. Always use
   block-body callbacks.

### Where the work happens

- All product code lives in the **`bob-plugins`** linked repo, plugin
  `plugins/bob-navigation-hotkeys`. Open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`. Never edit
  `~/bob/.obsidian/plugins/` directly. There is no bob-cli, capture-grammar, Mac
  Capture, vimrc, or `hotkeys.json` change, and no phase edits `bob.md` tasks.
- The plugin uses the committed fragment build. Edit only `src/` fragments, keep each
  hand-edited fragment at or under 1000 lines, add new fragments to `src/fragments.json`
  (mixin fragments before `690-install-plugin-mixins.js`, with each new mixin class
  added to that file's list), and add new test files explicitly to `test:unit` in
  `package.json`. Regenerate with `npm run build`, then run `npm run build:check` and
  `npm test`.
- Current sizes: `687-plugin-window-prefix.js` 847 lines, `267-window-prefix-actions.js`
  818, `473-window-hint-view.js` 787, `266-window-prefix-model.js` 538. New behaviour
  therefore goes in new fragments, and the shipped ones get only small hooks. Suggested
  names (a guide, not a rule): `266a-window-tab-model.js`, `267a-window-tab-actions.js`,
  `689-plugin-window-tab-numbers.js`, `687a-plugin-window-tab-jump.js`, and test exports
  in a new `722-window-tab-exports.js`.
- Finish every phase with `bob plugins sync`, following `AGENTS.md` if the sync refuses.
- **Real-Obsidian checks run on athena**, which has Obsidian 1.15.1 at
  `/snap/bin/obsidian` and `xvfb-run`; apollo has neither. From another host, copy the
  working tree (without `.git` and `node_modules`) to a fresh `mktemp -d` directory on
  athena over the `athena` Tailscale SSH alias, run
  `xvfb-run -a node scripts/obsidian-window-prefix-e2e.cjs -e /snap/bin/obsidian --only <group> -r <dir>/report.json -o <dir>/shots`
  there, copy the report and screenshots back, and review them. The harness only ever
  opens its own scratch vault and profile. A missing host is reported as a limitation,
  never as a pass.
- Headless-Chromium previews (`scripts/preview-window-prefix.cjs`) run on apollo with
  the Playwright Chromium in `~/.cache/ms-playwright`.
- `actions` and `paint` run in parallel, and both touch `fragments.json`,
  `package.json`, and the generated `main.js`. On rebase, keep both sides of list
  conflicts, take either side of `main.js`, and rerun `npm run build`.
- Known red tests on clean master (see the land note on `bob-cli-m0`): the block-marks
  hardcoded-date guard (`bob-cli-m8`), four perf budgets under full-suite load,
  relation-chips, bracket-indicator, and highlights-hints. Reproduce any failure on a
  clean checkout before treating it as yours, and do not fix unrelated failures.
- Only `prefix` bumps the nav minor version; `finish` bumps the patch version if it
  changes plugin code.
- Record discovered work as `PROPOSED FOLLOW-UP:` notes on your own phase bead; do not
  create beads.

## Design contract (every phase builds to this)

### Principles

- **What you see is what you get.** One pure catalog function decides every tab's
  number. The tab bar, the card, the palette commands, and dispatch all read it. A stale
  mapping is refused, never substituted.
- **Nothing moves.** A digit only focuses. The tab stays in its split and keeps its
  place, editor, undo history, cursor, scroll, folds, pin, and view mode. `H`/`L` remain
  the only movers.
- **Quiet until asked.** Idle numerals sit in the tab title's own tone. They brighten
  only with the card. A fluent `Ctrl+W 3` paints just the shipped ring and one flash.

### Vocabulary

As in m0: a **tab** is a `WorkspaceLeaf`, and a **split** (the user-facing word) is a
main-area `WorkspaceTabs` in `workspace.rootSplit`. New terms: a **numeral** is the
digit painted in a tab header, and the **tab catalog** is the ordered list of every
main-area tab with its number.

### Numbering

- **Every main-area tab counts**: pinned, duplicate-file, PDF, image, Canvas, graph,
  empty, and deferred tabs. Identity is the leaf, never the file, so two tabs on one
  note are two targets.
- **Order:** splits in reading order — exactly the order `Ctrl+W w` cycles, from one
  shared function — then tabs left to right in `group.children` order (which equals
  header DOM order). Splits stacked in one column number top to bottom before the next
  column.
- **Tabs 1–9** get numbers. Tabs 10 and later get none, and the card says "N more
  unnumbered". `9` never means "last".
- **Numbers are positional.** Focus changes never renumber. Opening, closing, moving, or
  reordering tabs renumbers between gestures.
- **Excluded:** sidebars, popout windows, hover editors, and the phone layout, matching
  the shipped prefix boundary.

### Keys (added to the m0 key table)

| Keys after `Ctrl+W`                             | Action        | Behavior                                                                                                                                                                                              |
| ----------------------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `1`–`9` (top row or numpad; held Ctrl accepted) | Jump to tab N | Selects tab N inside its own split and focuses it, loading it first if deferred; never moves it. The ring moves to that split and pulses, and the tab's numeral flashes. No Bob `Ctrl+O` jump record. |
| digit of the focused tab                        | Success no-op | Ring pulse plus numeral flash; the prefix disarms.                                                                                                                                                    |
| digit with no such tab                          | Blocked       | Stays armed; the card appears at once; the tabs row nudges; "No tab 7 · 6 tabs" (the shipped blocked-key contract).                                                                                   |
| `0`, US `Shift+1`–`9` (`!`…`(`)                 | Reserved      | Unmapped with the shipped copy ("0 isn't a window key"). `⇧N` is the designed future "bring tab N here".                                                                                              |
| `Cmd`/`Alt` + digit                             | Pass-through  | Cancels the prefix, as today; `Ctrl+W Cmd+3` still runs Obsidian's group-local "Go to tab #3".                                                                                                        |

**Digit normalization.** A digit comes from `event.key` `"1"`–`"9"` regardless of Shift
(AZERTY digits are shifted). Only when held Ctrl turns `key` into an empty string or a
control character, fall back to `event.code` `Digit1`–`Digit9` / `Numpad1`–`Numpad9`,
and only when Shift is not held. NumLock-off numpad keys (`End`, `ArrowDown`, …) are not
digits. The repeat, IME, and modifier-only rules are unchanged.

### Decision contract

Notation as in m0 (`*` focused, `'` selected in an unfocused split, `!` pinned), with
superscripts for numerals. Each line is an acceptance vector for both the unit planner
tests and the real-Obsidian e2e, where the layout signature must stay byte-identical
except for the `*` and `'` marks.

```text
Numbers:            [ A¹  B²* ] | [ C³'  D⁴ ]
Across, selected:   [ A  B* ] | [ C'  D ]    3 → [ A  B' ] | [ C*  D ]
Across, unselected: [ A  B* ] | [ C'  D ]    4 → [ A  B' ] | [ C  D* ]
Same split:         [ A  B* ] | [ C' ]       1 → [ A*  B ] | [ C' ]
Pinned are tabs:    [ P!  B* ] | [ C' ]      1 → [ P!*  B ] | [ C' ]
Already here:       [ A  B* ] | [ C' ]       2 → unchanged; pulse + flash; disarm
No such tab:        [ A  B* ] | [ C' ]       7 → unchanged; stays armed; "No tab 7 · 3 tabs"
Past nine:          13 tabs in two splits    numerals 1–9 only; 4 → tab 4; card "4 more unnumbered"
Stale:              armed 3 = C, now 3 = X   3 → unchanged; "Tabs changed — try again"; disarm
```

### Plan shape

`planWindowPrefixKey(snapshot, "3", { armedCatalog })` returns, for a digit:

`{ ok, available, action: "focus-tab", key: "3", tabNumber: 3, sourceSplitId, sourceLeafId, targetSplitId, targetLeafId, crossSplit, alreadyHere, preview, consequence: null }`.

Blocked digits return `action: "noop"` with
`reason: "no-such-tab" | "tabs-changed" | "layout-api-changed"` and the copy-table
preview. `tabs-changed` also carries `disarm: true`. An invalid layout keeps the shipped
error shape.

### The numerals

| State               | When                                                                                         | Look                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Idle**            | Always, under the gates below                                                                | A bare digit in a fixed 16 px box before the title (the tab's hidden icon slot; on PDF, Canvas, and graph tabs, just before the icon): monospace, 10.5 px, weight 600, `tabular-nums`. Its colour inherits the tab title's own colour (Obsidian's `--tab-text-color*` variables: faint on background tabs, muted on each split's selected tab), so the numeral reads as part of the title. No box, no fill. |
| **Armed, 0–300 ms** | `Ctrl+W` pressed                                                                             | Unchanged. Only the shipped ring and `⌃W` badge appear, so fluent `Ctrl+W l` paints exactly what it paints today.                                                                                                                                                                                                                                                                                           |
| **Lit**             | With the card (300 ms pause, a blocked or unmapped key, reduced motion, or Show window keys) | Accent key caps: an `--interactive-accent` digit on a ~12 % accent tint, with a 1 px inset accent ring and a 4 px radius. The focused tab gets a hollow neutral cap (`--text-muted` digit, 1 px `--background-modifier-border-hover` ring) — "you are here". Only colour, background, and box-shadow transition (90 ms); nothing moves.                                                                     |
| **Flash**           | A digit is accepted (including the already-here digit)                                       | The destination cap fills solid `--interactive-accent` with `--text-on-accent` for 220 ms (the yank-flash timing), then returns to idle. Lit caps return to idle at the same moment, so the flash stands alone.                                                                                                                                                                                             |

- **Stacked tabs** (`mod-stacked` spines): the numeral stays upright on each spine and
  aligns with the spine's title.
- **Contrast:** lit caps reach at least 4.5:1 against the tab-bar background in
  Obsidian's default light and dark themes. Idle numerals match their own title's
  contrast. Use theme variables only, never hardcoded colours.
- **Reduced motion:** transitions are removed; the flash still shows for 220 ms as a
  plain state change.

**Technique.** Stamp `data-bob-tab="N"` on the leaf's `.workspace-tab-header-inner` and
paint it with CSS `::before { content: attr(data-bob-tab) }`, gated by the body class
`bob-tab-numbers`. The lit state is the body class `bob-tab-numbers-lit`, and the flash
is a class on the header inner. There are no injected nodes, so nothing can steal clicks
or drags and nothing can leak. For accessibility, set `aria-description` on
`.workspace-tab-header` to "Control W then N". Do not use `aria-keyshortcuts`: its
spaces separate _alternative_ shortcuts, so `Control+W 3` would announce two shortcuts.
If `::before` proves limiting in real Obsidian, fall back first to an injected
`order: -1` span inside the header inner, then to an absolute overlay on title rects. If
headers cannot be found (no `tabHeaderEl`), render no numerals; the card's tabs row adds
a faint "numbers hidden", and digits still work through the planner and the pill ranges.

**Visibility gates** (a pure verdict). Numerals show only when all of these hold:

1. not the phone layout (`Platform.isPhone`, guarded so test stubs load);
2. the main area holds at least two tabs;
3. `Ctrl+W` can arm on this host: no command's effective hotkey is exactly physical
   Ctrl+W (`Ctrl`, or `Mod` off macOS; no Shift, Alt, or Meta). Read hotkeys from
   `app.hotkeyManager`: a command's custom keys when present (even `[]`), otherwise its
   defaults. `getHighlightsCommandHotkey` in `677d-highlights-hints.js` is the house
   pattern. If hotkeys are unreadable, assume Ctrl+W is unbound on macOS and bound
   elsewhere.

**Lit implies shown.** While the card is shown, numerals show lit even when only gate 3
hid them (Linux reaching the card through Show window keys), so the card never names
numbers the tab bar does not show. Under the accepted `linux_ctrl_w = keep` decision,
Linux therefore shows no idle numerals, and they appear by themselves if Close tab is
ever rebound.

**Refresh.** One refresh, coalesced to an animation frame, runs on `layout-change`,
`active-leaf-change`, `file-open`, layout ready / plugin load, and whenever a `Ctrl+W`
keydown arrives already `defaultPrevented` (which re-checks gate 3). Arming runs a
synchronous refresh from the armed snapshot. The refresh:

- writes only attributes that changed;
- tracks the elements it stamped and clears any that left the catalog (tabs moved to a
  sidebar or popout, tabs past nine);
- never cancels an in-flight flash;
- never throws;
- never mutates the prefix's live id→object maps.

Unload removes every attribute, class, body class, timer, and subscription.

### The card

The card gains numeral ranges on its pills and one tabs row. Wireframe:

```text
╭────────────────────────────────────────────────────────────────╮
│ [Ctrl W]  Window                                  2 splits [esc]│
│ ┆H┆ ┏ sase_blog             1–3 ┓ ┌ WP-LONG              4–6 [l]┐ │
│ FOCUS                            TAB                            │
│ [h] ← left edge                  [H] ← new split                │
│ [l] → WP-LONG                    [L] → join WP-LONG             │
│ [w] ⇄ WP-LONG                    [v] ◫ split right              │
│ [1–6] #  jump to that tab · it stays in its split               │
│ esc · q cancel                                                  │
╰────────────────────────────────────────────────────────────────╯
```

- **Pills.** The tab-count bubble becomes the split's numeral range in monospace: `1–3`,
  a single `4`, and a faint `+k` when the split also holds k unnumbered tabs (`+k` alone
  when all of its tabs are past nine). Use `aria-label="tabs 1 to 3"`. A bare count
  beside a tab numbered with the same digit is ambiguous; the probe hit exactly that
  collision.
- **Tabs row.** One full-width row under the FOCUS/TAB grid, aligned to the grid's
  columns: a key cap sized to its content (`1–6`), the Lucide `hash` glyph (fallback
  `#`), "jump to that tab", and a faint aside "it stays in its split". With more than
  nine tabs it reads `1–9` · "jump to that tab" · faint "4 more unnumbered". With one
  tab in the whole main area it is a dimmed `1` · "only tab". A blocked digit nudges
  this row and writes its message to the existing `aria-live` line.
- The FOCUS/TAB grid is unchanged (no nine duplicate rows). The card's aria summary adds
  "1 to 6 jump to that tab".

### Motion and feedback

- **Fluent `Ctrl+W 3`** (under 300 ms): the shipped ring appears at 0 ms; on success it
  jumps to the destination split and pulses (220 ms) while the destination numeral
  flashes. No card and no receipt.
- **With the card shown:** the caps are lit. On success the caps return to idle, the
  flash and pulse play, and the card hides after the shipped 220 ms linger.
- **Blocked digit:** the card shows at once if it is not already up, the tabs row
  nudges, and the message line says "No tab 7 · 6 tabs". The prefix stays armed.
- **Stale mapping:** "Tabs changed — try again", then disarm with the shipped 900 ms
  linger.
- **Executor failure:** one Notice (copy table).
- Jumps show no receipt, because the screen already changed (m0's focus rule).
- Where only "lit implies shown" made the numerals visible (Linux through Show window
  keys), keep them shown until the flash ends, so the confirmation is never invisible.

### Copy table

The `model` phase owns these strings. Polish the wording but keep the meaning.

| Situation           | Copy                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------- |
| Tabs row            | `1–N` · "jump to that tab" · aside "it stays in its split"                              |
| More than nine tabs | `1–9` · "jump to that tab" · aside "K more unnumbered"                                  |
| One tab             | `1` (dimmed) · "only tab"                                                               |
| No such tab         | "No tab 7 · 6 tabs"; with one tab, "No tab 2 · only 1 tab"                              |
| Stale mapping       | "Tabs changed — try again"                                                              |
| Headers unavailable | aside "numbers hidden"                                                                  |
| Execution failed    | "Couldn't jump to tab N · focus restored" / "Couldn't jump to tab N · check the layout" |
| Palette names       | "Jump to tab 1" … "Jump to tab 9" (ids `window-jump-tab-1` … `window-jump-tab-9`)       |
| Pill aria labels    | "tabs 1 to 3", "tab 4", "4 more unnumbered"                                             |

### The jump executor

1. **Freeze at arm.** Keep the armed snapshot (already captured) and build the armed
   catalog from it. Restamp the numerals synchronously from that catalog, so the numbers
   on screen are the numbers dispatch uses.
2. **Hint is truth.** At dispatch, plan from a fresh snapshot together with the armed
   catalog. If the fresh catalog's tab N is not the armed catalog's tab N (compared by
   leaf identity), refuse with "Tabs changed — try again"; never select whichever tab
   now holds that number. The prefix already cancels on foreign `layout-change` and
   `active-leaf-change`; this check guards the races in between. Palette commands have
   no armed catalog and plan from a fresh snapshot.
3. **Load, then focus only if still current.** `await leaf.loadIfDeferred?.()`. Then,
   only if the session is still current (the arm generation is unchanged for the prefix;
   the active leaf is unchanged for a palette call), call
   `workspace.setActiveLeaf(leaf, {focus: true})`, retrying once after a frame like the
   shipped focus helper. A cancelled or superseded session returns
   `{ ok: false, reason: "superseded" }` and presents nothing.
4. **Invariants.** The active leaf is the target; the target's parent is unchanged; the
   target is its split's selected tab (`children[currentTab]`); for a cross-split jump,
   the source split's selected tab is unchanged; and the main-area leaf set is unchanged
   (no tab added, removed, or reparented). On failure, refocus the source leaf and show
   the one Notice.
5. **Never** call `openFile`, `getLeaf`, `duplicateLeaf`, `detach`, `insertChild`,
   `removeChild`, `requestSaveLayout`, or `recordJump`. Respect each view's own focus
   behaviour (no forced editor focus on PDF or Canvas).
   - Scroll: no save or restore unless the e2e proves a loss. If it does, restore with
     `leaf.getEphemeralState()` / `setEphemeralState()`; a hidden tab's `scrollTop`
     reads 0, so DOM capture cannot work.
   - Clipped tab strip: if the e2e shows Obsidian leaves the newly selected header
     clipped, add a guarded
     `tabHeaderEl.scrollIntoView({ block: "nearest", inline: "nearest" })`; otherwise do
     not.
6. **Return**
   `{ ok, action: "focus-tab", tabNumber, leafId, targetSplitId, crossSplit, alreadyHere }`
   for the presenter. The executor shows no UI except the failure Notice.

### Dormant until `prefix`

Every phase syncs to Bryan's live vault, so nothing user-visible appears before digits
work:

- `model` leaves `normalizeWindowPrefixKeydown` unchanged (digits stay unmapped from the
  keyboard) and only adds pure functions and read-only adapters.
- `actions` code is reachable only from tests and the e2e harness.
- `paint` renders the tabs row and pill ranges only when
  `createWindowHintHost(doc, { tabNumbers: true })` asks for them, and adds a stamper
  mixin that nothing starts yet.
- `prefix` flips all three seams together (digits in the normalizer, `tabNumbers: true`
  in `ensureWindowPrefixHintHost`, the stamper started from `onload`), along with the
  minor version bump.

### Design refinements over the research (decided while planning)

1. Hint-is-truth compares the armed and fresh catalogs by leaf identity instead of
   reading the DOM at dispatch. It gives the same guarantee and is pure and
   unit-testable; the e2e proves DOM parity separately.
2. Idle numerals inherit the tab title's own colour variables. The defaults equal the
   research's faint/muted, and themes stay coherent.
3. Lit implies shown, so the Show window keys card never names invisible numbers on
   Linux.
4. `aria-description` replaces `aria-keyshortcuts`, which cannot express a key sequence.
5. Palette commands are "Jump to tab N", matching the card copy and staying distinct
   from Obsidian's group-local "Go to tab #N".
6. The already-here digit flashes too, confirming "you are here".
7. Reading order's final tie-break becomes the layout-tree order instead of a string
   compare of ids (`split-10` sorts before `split-9`), so zero-area startup layouts
   number like the tree. `w` and digits keep sharing one function.
8. Dormant-until-`prefix` seams keep intermediate syncs from showing numbers that do
   nothing.

### Out of scope (reserved, designed later)

`⇧N` ("bring tab N here"), `0` (reserved; the candidates are an exact-number chooser,
`0 → number → Enter`, or "last tab"), numbering past nine, rebinding Obsidian's native
`Cmd+1…9` (a one-line `hotkeys.json` edit once "Jump to tab N" exists), sidebars, popout
windows, hover editors, and the phone layout.

## Phase: model — Tab catalog, digit planner, and main-area enumeration

**Goal:** one pure source of truth for tab numbers and digit plans, plus read-only
readers that work while focus sits in a sidebar.

**Pure functions** (a new fragment after `266`, exported for tests):

- `buildWindowPrefixTabCatalog(snapshot)`. Input: a main-area snapshot
  `{ splits: [{ id, rect, leaves: [{ id, label, pinned }], selectedLeafId }], activeLeafId }`,
  where `leaves` are in `children` order and `activeLeafId` may be `null`; it must not
  require an active leaf. Output:
  `{ ok, tabs: [{ number, leafId, splitId, label, pinned, selected, active }], byNumber, splits: [{ id, first, last, unnumbered }], total, numbered, unnumbered }`,
  where `number` is 1–9 or `null` and `byNumber` maps 1–9 to tabs. Split order comes
  from the shared reading-order function.
- `windowPrefixDigitFromKeydown(eventLike)` returns `"1"`–`"9"` or `null`, implementing
  "Digit normalization". Do not wire it into `normalizeWindowPrefixKeydown` yet.
- A digit branch in `planWindowPrefixKey(snapshot, key, options)` that routes
  `"1"`–`"9"` to `planWindowPrefixTabJump(snapshot, n, options)`, producing the plan
  shape and reasons in the contract, with `options.armedCatalog` driving the stale
  check.
- View-model additions in `buildWindowPrefixViewModel`: a `tabs` object (range, total,
  numbered, unnumbered, available, preview, aside) and a per-split `tabRange` (`first`,
  `last`, `unnumbered`, `label`, `ariaLabel`). The existing `rows` and `groups` stay
  unchanged.
- `windowPrefixCtrlWOwner(entries, { mac })`, which returns the owning command id or
  `null`, and
  `windowTabNumeralsVerdict({ tabCount, ctrlWOwner, hotkeysKnown, mac, phone, lit, headersAvailable })`,
  which returns `{ show, lit, reason }` per the gates and "lit implies shown".
- The copy-table strings and blocked-message helpers.
- The reading-order tie-break change (refinement 7). Existing `w` vectors must stay
  green.

**Read-only adapters** (in `267` or a new fragment):

- `snapshotWindowPrefixMainArea()` returns
  `{ splits, activeLeafId, leafById, splitById }`. `activeLeafId` is `null` when focus
  is outside the main area. Build it from `iterateRootLeaves` (block-body callback),
  restricted to `rootSplit` and grouped by parent. Order each split's leaves by
  `group.children` index (falling back to iteration order when `children` is missing)
  and take rects, labels, pins, and the selected leaf from the fixed resolver. It
  returns its own maps and never touches `this.windowPrefixLeafById` /
  `windowPrefixSplitById`.
- Refactor `snapshotWindowPrefixLayout()` to build on it, keeping the same output shape
  and capabilities. It still returns `null` when the active leaf is outside the main
  area, and it remains the only writer of the prefix's live maps.
- Fix `windowPrefixSelectedLeaf`: read `children[currentTab]` when `currentTab` is a
  number, then a leaf-valued `currentTab` / `selectedTab` for older builds, then
  `getMostRecentLeaf`, then the first leaf.
- `readWindowPrefixHotkeyEntries(app)` returns
  `{ known, entries: [{ commandId, modifiers, key }] }` from `app.hotkeyManager` (custom
  keys when present, otherwise defaults), never throwing.

**Tests:** add `scripts/test-navigation-window-tab-model.cjs` (in `test:unit`). It
covers:

- every decision-contract vector;
- one, two, and three splits; a stacked column; 13 tabs across two splits (numbers 1–9,
  four unnumbered, per-split ranges including `+k`); duplicate-file, pinned, and empty
  tabs; a single tab;
- the armed-versus-fresh stale refusal, already-here, and
  `capabilities.focus === false`;
- the full digit-normalization matrix: plain; held Ctrl with a control-character key and
  `code` `DigitN`; numpad; an AZERTY shifted digit; US `#`; `0`; Meta and Alt; repeat;
  NumLock-off `End` / `Numpad1`;
- the verdict matrix: phone; one tab; Ctrl+W owned by `workspace:close` through `Mod` on
  Linux and through `Ctrl` on macOS; `Mod+W` on macOS not an owner; unknown hotkeys on
  each platform; lit implies shown; headers unavailable;
- parity: for every fixture and N, `planWindowPrefixKey(snapshot, N).targetLeafId`
  equals `catalog.byNumber[N].leafId`, view-model ranges agree with the catalog, and the
  split order equals the `w` cycle order.

Extend the window-prefix actions tests with fakes for the adapters: sidebar focus yields
splits with a `null` active leaf; `children` order wins over iteration order; a numeric
`currentTab` resolves the selection; the prefix maps stay untouched; hotkey reading
handles custom `[]`, defaults, and a missing manager. All existing window-prefix suites
stay green. No version bump. Build, test, sync.

## Phase: actions — In-place tab-jump executor

**Goal:** turn a focus-tab plan into a verified, lossless focus change: the target tab
selected and focused where it lives.

- Route `focus-tab` from `executeWindowPrefixPlan` to a new executor (a new mixin
  fragment, installed in `690`) that implements "The jump executor" steps 3–6. Accept a
  session predicate as `armed.isCurrent` (default: always current). Return `superseded`
  without presenting when the predicate fails after the load. An already-here plan
  returns `ok` without calling `setActiveLeaf`. The internal re-plan in
  `executeWindowPrefixPlan` passes through `armed.armedCatalog` when present.
- Fold the `selectedTab` fix through the move executor. `sourceSelectedLeaf`,
  `targetSelectedLeaf`, the remainder-selection fallback, and the `selectedTabIsMoved`
  diagnostic all read through the fixed resolver, so a failed join's rollback reselects
  the destination's previous tab. Add a rollback unit test that proves it.
- **Tests:** add `scripts/test-navigation-window-tab-actions.cjs` (in `test:unit`),
  using fake `WorkspaceTabs` / `WorkspaceLeaf` objects that record calls. Cover:
  - every decision vector; the call log holds only `loadIfDeferred` and `setActiveLeaf`,
    never insert, remove, open, detach, or `requestSaveLayout`;
  - a deferred load awaited before focus;
  - a superseded session that never focuses late;
  - `setActiveLeaf` that fails to activate: the source is refocused and the Notice
    shows;
  - a changed parent or leaf count: restore plus Notice;
  - stale-source "layout changed" handling consistent with m0;
  - serialized concurrent calls.
- **e2e:** add a `tabs` group to `scripts/obsidian-window-prefix-e2e.cjs`, with its
  cases in a new `scripts/obsidian-window-prefix-e2e-tabs.cjs` module (like
  `-actions.cjs`). Add it to `GROUPS`, the help text, and the self-test. Drive jumps
  through the plugin instance with `runWindowPrefixPaletteKey("N")`, as the m0 actions
  group does (digits already plan through `planWindowPrefixKey` after `model`). Cases:
  - each decision vector, checked by layout signature (byte-identical except `*` and
    `'`);
  - a deferred background tab after an app reload loads and focuses with its file shown;
  - a long note scrolled to about line 120 in an unselected tab of the other split keeps
    its scroll after the jump;
  - typing in the target, jumping away and back, then `u` undoes;
  - a PDF or image tab focuses without forcing an editor;
  - a stacked-tabs group;
  - 13 tabs in a narrow split: jump to 9 and record whether its header is fully visible
    (this decides the `scrollIntoView` rule);
  - the source split's selection is unchanged after a cross-split jump.

  Run it on athena and record a `TAB JUMP E2E:` note on this phase's bead with the
  report summary and the scroll and clipping findings.

- No version bump. Build, test, sync.

## Phase: paint — Tab numerals, card tabs row, and previews

**Goal:** make the numerals and the card beautiful, exactly per "The numerals" and "The
card", as dormant, testable painters.

- **Stamper mixin** `BobNavigationHotkeysWindowTabNumbersMixin` (a new fragment before
  `690`, installed there) with:
  - `startWindowTabNumbers()` (subscriptions plus a first refresh; **not** called from
    `onload` in this phase), `stopWindowTabNumbers()`;
  - `scheduleWindowTabNumbersRefresh()` and `refreshWindowTabNumbersNow({ snapshot })`,
    implementing every "Refresh" rule on top of `snapshotWindowPrefixMainArea()`,
    `buildWindowPrefixTabCatalog()`, `readWindowPrefixHotkeyEntries()`, and
    `windowTabNumeralsVerdict()`;
  - `setWindowTabNumbersLit(on)` and `flashWindowTabNumber(leafId)`;
  - header resolution through `leaf.tabHeaderEl` and its `.workspace-tab-header-inner`.
- **CSS:** a `/* Window tab numerals */` block after `/* Window prefix */` in
  `styles.css`, covering idle, lit, here, flash, stacked spines, and reduced motion,
  with theme variables only. Measure Obsidian's header-inner gap so the numeral box plus
  gap reproduces the probe's ~21 px and the title baseline does not move. The "here" cap
  needs no extra state: the m0 spike confirmed `.workspace-tabs.mod-active`, so
  `.workspace-tabs.mod-active .workspace-tab-header.is-active` is the focused tab.
- **Card painter** (`473-window-hint-view.js`, or a helper fragment if it would pass
  1000 lines). Behind `options.tabNumbers === true`, pills show `tabRange` instead of
  the count bubble and the tabs row renders under the grid. `nudge()` with a digit key
  or `"tabs"` targets that row. Use the `hash` icon with a `#` fallback, and extend the
  aria summary. With the option off, the output is byte-identical to today, and a test
  proves it.
- **Preview harness:** extend `scripts/preview-window-prefix.cjs` so its fixture tab
  bars use Obsidian's real header structure (`.workspace-tab-header` >
  `.workspace-tab-header-inner` > icon and title, plus pin and close; `mod-active`,
  `is-active`, `mod-stacked`) and drive the real stamper on them. Render these states in
  light and dark:
  1. idle numerals, two splits;
  2. lit caps with the "here" cap;
  3. the flash;
  4. 13 tabs (crowded titles, legible numerals, tabs 10+ blank);
  5. stacked-tab spines;
  6. one split with two tabs;
  7. a single tab (no numerals);
  8. the card with pill ranges and the tabs row;
  9. the card with more than nine tabs, a `+k` pill, and "4 more unnumbered";
  10. the card with one tab and a dimmed tabs row;
  11. a blocked-digit nudge, "No tab 7 · 6 tabs";
  12. a 420 px narrow card;
  13. reduced motion.

  Compute lit-cap contrast in both themes and fail below 4.5:1. Look at every PNG
  yourself and iterate until the result is calm, aligned, legible, and balanced:
  numerals centred on the title's x-height, never crowding the pin or close icons, and
  caps the same size in the tab bar and on the card.

- **Tests:** add `scripts/test-navigation-window-tab-numbers.cjs` (in `test:unit`),
  using `modal-harness.cjs` `ElementStub`. Cover: stamps match the catalog; tabs past
  nine and tabs that moved away are cleared; only changed attributes are written; every
  gate (phone, single tab, Ctrl+W owned, unknown hotkeys per platform, lit implies
  shown); a flash survives a refresh and clears after 220 ms; idempotent repaint; full
  unload cleanup; missing headers never throw and report "numbers hidden". Extend
  `scripts/test-navigation-window-hint-view.cjs` for pill ranges (single and `+k`),
  tabs-row availability and asides, the nudge target, aria, and the option-off
  byte-identical output.
- No version bump. Build, test, sync.

## Phase: prefix — Digits in the Ctrl+W prefix, palette commands, go live

**Goal:** make `Ctrl+W N` dependable end to end, flip the dormant seams, and ship.

- **Keyboard.** `normalizeWindowPrefixKeydown` maps digits through
  `windowPrefixDigitFromKeydown` to `{ kind: "mapped", key: "N" }`, ahead of the
  unmapped branch. `0` and `#` stay unmapped.
- **Arm.** Capture `windowPrefixArmedTabCatalog` from the armed snapshot, refresh the
  stamper synchronously from it, and bump an arm-generation counter on every arm and
  disarm.
- **Dispatch.** `runWindowPrefixMappedKey` passes `{ armedCatalog }` to the planner, and
  `{ armedCatalog, isCurrent }` to `executeWindowPrefixPlan`. Present each outcome:
  - success (same-tab included): caps unlit, ring jump and pulse, numeral flash, disarm;
  - `superseded`: nothing;
  - `no-such-tab`: the shipped blocked path (stays armed, card now, tabs-row nudge,
    message);
  - `tabs-changed`: message, disarm, and the 900 ms linger;
  - failure: the executor's Notice, then disarm.

  Put the digit glue in a new fragment, so `687` stays at or under 1000 lines.

- **Lit.** `showWindowPrefixCardNow` lights the numerals; `hideWindowPrefixHint` and
  every disarm path (Esc, blur, mousedown, layout change, action, thrown action,
  teardown) unlight them.
- **Seams.** `ensureWindowPrefixHintHost` passes `tabNumbers: true`. `onload` (in
  `490-plugin-lifecycle.js`, next to `registerWindowPrefixCommands`) starts the stamper.
  The arming path schedules a numerals refresh when it sees a `Ctrl+W` that is already
  `defaultPrevented`.
- **Palette.** In `688-plugin-window-prefix-commands.js`, add `window-jump-tab-1` …
  `window-jump-tab-9` ("Jump to tab N", no default hotkeys) through
  `runWindowPrefixPaletteKey("N")`, with the active-leaf session predicate. Success
  shows the flash and pulse; a blocked command shows its copy-table reason as a Notice.
  Show window keys lights the numerals along with its card.
- **Tests.** Extend `scripts/test-navigation-window-prefix.cjs` (fake events, timers,
  keymap, actions, and stamper): a digit after arming runs focus-tab; held-Ctrl and
  numpad digits; `0` and `#` stay unmapped and consumed; `Cmd+3` passes through and
  cancels; a blocked digit stays armed with the card and nudge; a stale catalog refuses
  and disarms; a superseded load presents nothing; numerals light with the card and
  unlight on every exit path; a fluent digit (under 300 ms) never paints the card; the
  scope stack is restored. Extend `scripts/test-navigation-window-prefix-commands.cjs`:
  nine commands registered with no hotkeys, blocked Notices, and Show window keys
  lighting the numerals.
- **e2e** additions to `--only tabs`, with trusted CDP keys:
  - `Ctrl+W 4` across splits;
  - `Ctrl+W Ctrl+3` with focus in the right split lands on global tab 3 (not Obsidian's
    group-local Ctrl+3 target) and runs no native command;
  - `Ctrl+W` then Numpad 2;
  - `Ctrl+W 7` with six tabs stays armed with the card and message, then Esc leaves the
    layout signature and note text byte-identical;
  - `Ctrl+W 0` and `Ctrl+W Shift+3` change nothing;
  - the focused tab's digit disarms with no layout change;
  - numeral DOM parity: every numbered header's `data-bob-tab` equals the catalog, and
    tabs 10+ carry none;
  - opening a tab in the left split renumbers the right split on the next frame, and
    `H`/`L` renumber;
  - focus in a sidebar keeps the numerals; a single tab shows none;
  - with the default Linux binding restored, numerals hide and `Ctrl+W` closes the tab;
  - the lit class arrives with the card and is gone after Esc; the flash class sits on
    the destination for about 220 ms;
  - after every case, the `app.keymap` scope stack equals its starting state.

  Capture screenshots of idle, lit, and flash. Run on athena.

- Bump the nav manifest minor version, and add "`1`–`9` jump to a numbered tab" to the
  manifest description's `Ctrl+W` sentence. Build, test, sync. Record a `GO LIVE:` note
  with the e2e summary.

## Phase: finish — Docs, visual acceptance, and polish

**Goal:** document the feature, prove it looks right in real Obsidian, and ship the
polish.

- **Docs.** Read the `docs.md` reference memory first. In `docs/window-keys.md`:
  - add the digit rows to the key table;
  - add a "Numbered tabs" section: the numbering rule, idle/lit/flash, past nine, every
    tab counts, and "it stays in its split";
  - add the digit decision pictures;
  - extend the Linux note: numerals stay hidden while Close tab owns `Ctrl+W` and appear
    after the documented rebind;
  - note that Obsidian's native `Cmd+1…9` still count within the focused split, and that
    binding "Jump to tab N" to `Cmd+N` makes them follow the numerals;
  - note that if macOS's Mission Control "Switch to Desktop N" shortcuts (`Ctrl+N`) are
    enabled, they take held-Ctrl digits first, so release Ctrl before the digit;
  - add "Jump to tab 1 … 9" to the palette commands;
  - in Scope, replace "numbered splits" with the reserved `0` and `⇧N`.

  Update the `Ctrl+W` bullet in `docs/bob-navigation-hotkeys.md` if it lists keys. Then
  run `npm run docs:gen`, `npm run docs:inventory:check`, and
  `BOB_CLI_DIR=<a bob-cli checkout> npm run docs:check`.

- **Visual acceptance.** Extend the `visual` group (or add `tabs-visual`) with
  real-Obsidian screenshots in light and dark at one, two, and three splits: idle
  numerals, lit caps with the card, a flash mid-jump, 13 tabs, stacked tabs, the
  Linux-default hidden state, and the card with ranges and the tabs row. Review every
  screenshot against "The numerals" and "The card". Fix every polish issue you find
  (alignment, baseline, crowding near pin and close icons, contrast, cap sizes that
  differ between bar and card) in CSS or painter code, and rerun until clean. Then run
  `--only tabs`, `--only prefix`, and `--only actions` once more to show no regression.
- Bump the nav patch version if plugin code changed, run the final `bob plugins sync`,
  and record the report paths and an `ACCEPTANCE:` verdict note on this phase's bead.
