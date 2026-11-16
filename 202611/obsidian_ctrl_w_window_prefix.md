---
tier: epic
title: Ctrl+W window prefix for Obsidian vertical splits
goal: 'In Bob''s Obsidian, Ctrl+W followed by a key focuses, moves, and creates main-area
  splits from the keyboard. h/l focus the neighbouring split, H/L move the current
  tab into, out of, or across splits (creating and collapsing splits as needed) without
  losing its editor state, and a calm, theme-native hint shows exactly what each key
  will do.

  '
decisions:
  linux_ctrl_w:
    ask: How should Ctrl+W behave on Linux, where Obsidian binds it to Close tab by
      default?
    choices:
      keep: Linux keeps Ctrl+W as Close tab; the prefix arms where Ctrl+W is unbound
        (your Mac)
      rebind: Rebind vault Close tab to Meta+W; Mac unchanged, Linux closes with Super+W
        and gets the prefix
    default: keep
    why: Your Mac is the daily driver, and a vault hotkey edit syncs to every host
    answer: keep
  window_prefix_record:
    ask: Record an accepted decisions strand for the Ctrl+W window prefix once it
      ships?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: spike
  title: Real-Obsidian layout and keyboard spike
  depends_on: []
  size: medium
  description: 'spike: build the window-prefix CDP e2e harness and prove lossless
    cross-group tab moves, split create/collapse, and keyboard precedence in real
    Obsidian.'
- id: model
  title: Pure layout model, planners, key table, and view model
  depends_on: []
  size: medium
  description: 'model: add Obsidian-free fragments that turn a layout snapshot into
    focus/move/split plans, key normalization, arming verdicts, and the hint view
    model.'
- id: actions
  title: Live layout adapter and lossless executor
  depends_on:
  - spike
  - model
  size: medium
  description: 'actions: build the live snapshot adapter and executor that focuses,
    joins, creates, collapses, and duplicates splits with invariants, rollback, and
    an API canary.'
- id: hint
  title: Ring, card, minimap, and motion painter
  depends_on:
  - model
  size: medium
  description: 'hint: paint the armed ring, key-cap badge, hint card with minimap,
    nudges, pulses, and receipts from the view model, with a light/dark preview harness.'
- id: prefix
  title: Ctrl+W prefix state machine
  depends_on:
  - actions
  - hint
  size: medium
  description: 'prefix: wire the capture listener, armed keymap scope, arming gate,
    dispatch, cancel paths, and two-tier hint timing into a reliable Ctrl+W prefix.'
- id: finish
  title: Commands, focused-split underline, docs, acceptance, rollout
  depends_on:
  - prefix
  size: medium
  description: 'finish: add palette commands and the focused-split underline, document
    the window keys, run the full real-Obsidian acceptance, and sync.'
proposed_by: bbugyi200.apollo.e9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 20:01:42
status: done
bead_id: bob-cli-m0
---

- **PROMPT:** [prompts/202611/obsidian_ctrl_w_window_prefix.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_ctrl_w_window_prefix.md)
- **BEAD:** [bob-cli-m0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-m0/README.md)

# Plan: Ctrl+W window prefix for Obsidian vertical splits

## Context

Bryan keeps a task note beside its draft in two side-by-side Obsidian tab groups (see
`~/tmp/screenshots/20261115_171818.png`: the left group holds pinned `20261115`, `dash`,
and active `sase_blog`; the right group holds `sase_blog_contents`). Bob has tab verbs
that act within one group, but nothing focuses, fills, creates, or collapses a split
from the keyboard, and the two tab bars look alike, so it is hard to see which split has
focus. The vault already tracks this work as `bob.md` `^ctrl-w-keymaps`, which blocks
`^split-tabs`.

The consolidated research
`research:202611/obsidian_ctrl_w_window_prefix/obsidian_ctrl_w_window_prefix.md` is the
design basis, and Bryan accepted every requirement it recommends. Read it with
`sase artifact read` before starting any phase. This plan keeps its key table, its
decision contract, its two-tier hint, and its "spike first, never ship a lossy move"
rule. Verification while planning added two keyboard facts the research did not have.
The spike phase must confirm both:

1. **Obsidian's keymap runs first.** Since Obsidian 1.14, the keymap is a window capture
   listener registered before plugins load (see the comment above
   `registerCountedLaneToggleInputListeners` in `610-plugin-jumps-and-counted-keys.js`).
   A bound hotkey therefore runs its command before any plugin listener. It calls
   `stopPropagation`, not `stopImmediatePropagation`, so a plugin's window capture
   listener still runs afterwards and sees `event.defaultPrevented`. On Linux, Obsidian
   binds Ctrl+W to Close tab by default, so that tab closes before the prefix sees the
   key. On macOS, Ctrl+W is unbound (Close tab is Cmd+W), and the vault's
   `.obsidian/hotkeys.json` binds nothing to Ctrl+W.
2. **Suffix chords can collide with vault hotkeys.** The vault binds `Ctrl+L` to
   `mrj-jump-to-link:activate-jump-to-link`. Without extra work, a Vim user who types
   `Ctrl+W Ctrl+L` (Ctrl still held) would trigger jump-to-link. This plan has the armed
   prefix push an Obsidian `Scope` onto `app.keymap`, which is how modals take the
   keyboard. That routes every suffix key through Bob's handler before any hotkey can
   run.

### Where the work happens

- All product code lives in the **`bob-plugins`** linked repo, in the plugin
  `plugins/bob-navigation-hotkeys`. Open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`. Never edit
  `~/bob/.obsidian/plugins/` directly.
- The plugin uses the committed fragment build. Edit only `src/` fragments, keep each
  hand-edited fragment at or under 1000 lines, and add new fragments to
  `src/fragments.json`. Regenerate `main.js` with `npm run build`, and run
  `npm run build:check` and `npm test` (new test files are added explicitly to
  `test:unit` in `package.json`). Finish every phase with `bob plugins sync`, following
  `AGENTS.md` if the sync refuses to run. `700-exports.js` is already at 1000 lines, so
  test exports go in a new `720-window-prefix-exports.js`, following
  `717-copy-card-exports.js`.
- Plugin methods are mixin classes installed by `690-install-plugin-mixins.js`. Add each
  new mixin class to that list. Mixin fragments must appear before `690` in
  `fragments.json`.
- **Real-Obsidian checks run on athena**, which has Obsidian 1.15.1 at
  `/snap/bin/obsidian` and `xvfb-run`. apollo has neither. From another host, copy the
  working tree (without `.git` and `node_modules`) to a fresh `mktemp -d` directory on
  athena over the `athena` Tailscale SSH alias. Run
  `xvfb-run -a node scripts/obsidian-window-prefix-e2e.cjs -e /snap/bin/obsidian -r <dir>/report.json -o <dir>/shots`
  there, copy the report and screenshots back, and review them. The harness only ever
  opens an owned scratch vault and profile, never `~/bob`. A missing host is reported as
  a limitation, never as a pass.
- Headless-Chromium previews (`scripts/preview-*.cjs`) run on apollo using the
  Playwright Chromium in `~/.cache/ms-playwright`.
- Parallel phases (`spike`/`model`, then `actions`/`hint`) all touch `fragments.json`,
  `package.json`, and the generated `main.js`. On rebase, resolve the list conflicts by
  keeping both sides, take either side of `main.js`, and rerun `npm run build`.
- Leave the nav `manifest.json` version alone until `prefix`, which bumps the minor
  version. `finish` bumps the patch version if it changes plugin code. Do not edit
  `bob.md` tasks; Bryan closes `^ctrl-w-keymaps` himself.

## Design contract (every phase builds to this)

### Vocabulary

A **tab** is a `WorkspaceLeaf`. A **split** (the word used in all user-facing copy) is a
main-area tab group (`WorkspaceTabs`) in the current window's `workspace.rootSplit`. A
**column** is the set of splits that share one horizontal span. In v1 layouts each
column holds one split.

### Key table (v1)

| Keys after `Ctrl+W`     | Action                            | Behavior                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------- | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `h` / `←` · `l` / `→`   | Focus split left / right          | Geometric nearest neighbour; **never wraps**; at an edge it is a visible no-op (edge bump plus a dimmed row). Focuses the destination split's selected tab. With two splits this is exactly "go to the other split".                                                                                                                                                            |
| `w` · a second `Ctrl+W` | Focus next split                  | Reading order (left→right, top→bottom), **wraps**. With two splits this toggles.                                                                                                                                                                                                                                                                                                |
| `H` · `L`               | Move the current tab left / right | **Join** the adjacent split; or **create** a split at that edge when other tabs stay behind; a split that the move empties **collapses**. The lone tab of the only split stays put: a no-op that suggests `v`. Focus follows the tab. Deliberately unlike Vim's `<C-w>L`, which moves a whole window. Here the unit is the tab, as in VS Code's "move editor into right group". |
| `v`                     | Split right, same note            | `workspace.duplicateLeaf(leaf, 'split', 'vertical')`, focusing the new split, like Vim's `<C-w>v`.                                                                                                                                                                                                                                                                              |
| `Esc` · `Ctrl+[` · `q`  | Cancel                            | Layout unchanged.                                                                                                                                                                                                                                                                                                                                                               |

Held Ctrl is accepted on every suffix, so `Ctrl+W Ctrl+L` equals `Ctrl+W l`.
`j k J K s c o p x = < >` and digits are reserved for later. No key in this family ever
closes a note (layout-only rule). Window motions never record a Bob `Ctrl+O` jump.

### Decision contract

Notation: `*` marks the focused tab, `'` marks the selected tab of an unfocused split
(where it matters), and `!` marks a pinned tab. Each line is an acceptance vector for
both the unit planner tests and the real-Obsidian e2e. Signatures in tests use this
exact notation.

```text
Create right:         [ A  B* ]                   L → [ A' ] | [ B* ]
Create left:          [ A  B* ]                   H → [ B* ] | [ A' ]
Join right:           [ A  B* ] | [ C'  D ]       L → [ A' ] | [ C  B*  D ]
Join left:            [ A'  B ] | [ C*  D ]       H → [ A  C*  B ] | [ D' ]
Last tab out:         [ A* ] | [ C  B' ]          L → [ C  B  A* ]              (left split closes)
Middle out:           [ A' ] | [ B* ] | [ C' ]    H → [ A  B* ] | [ C' ]         (middle split closes)
Only tab:             [ A* ]                      H or L → unchanged; "only tab · v splits"
Edge, sole tab:       [ A* ] | [ B' ]             H → unchanged; "already leftmost"
Pinned stays pinned:  [ P!*  B ] | [ Q!  C' ]     L → [ B' ] | [ Q!  P!*  C ]
Unpinned after pins:  [ A*  B ] | [ Q!'  R!  C ]  L → [ B' ] | [ Q!  R!  A*  C ]
Focus:                [ A* ] | [ B' ] | [ C' ]    l → B focused; h → unchanged + left-edge bump; w twice (two arms) → C focused
Split:                [ A* ]                      v → [ A' ] | [ A* ]           (same note, new split focused)
```

Insertion rule: a moved unpinned tab goes at `max(destSelected + 1, destPinnedCount)`,
and a moved pinned tab goes at `destPinnedCount`, keeping its pin. A source split that
survives a move-out shows its most recently used remaining tab
(`workspace.getMostRecentLeaf(sourceGroup)`), not an index neighbour. Widths are never
auto-equalized; a split that closes gives its width to its neighbours, the same way
Obsidian handles a close.

### Arming gate and key handling

`Ctrl+W` (physical Ctrl, never `Mod`; no Shift, Alt, or Meta; not `repeat`; not
`isComposing`; **not `defaultPrevented`**) arms the prefix only in the main window, only
when the active leaf is in `workspace.rootSplit`, and only when one of these holds:

- focus is in that leaf's Markdown CodeMirror editor with Vim mode `normal` or `visual`,
  no Vim operator or count pending, and an empty `inputState.keyBuffer`. Read the mode
  from the _focused_ editor with `resolveVimCodeMirror` + `getCurrentVimMode`
  (`isVimNormalModeEditor` rejects visual mode, so it is not enough on its own), and
  reuse the pending-input parsing in `090-local-task-ids.js`.
- focus is non-editable (not an input, textarea, or contenteditable) and not inside a
  modal, prompt, menu, suggestion popup, hover popover, Command Mode sheet, or Bob card.
  This covers reading view, PDF, image, canvas background, graph, and empty tabs.

Everything else passes through untouched: insert or replace mode (Ctrl+W still deletes a
word), the `:` line, search fields, sidebars, popout windows, and any Ctrl+W that an
Obsidian hotkey already handled. That last case is why Linux keeps Close tab under the
`keep` decision. Arming calls `preventDefault` + `stopImmediatePropagation` so Vim never
sees the Ctrl+W.

While the prefix is armed:

- **Keyboard ownership.** Push one `Scope` with a catch-all
  `scope.register(null, null, handler)` onto `app.keymap`. Obsidian's keymap then routes
  each suffix to Bob before any hotkey (Ctrl+L, Linux Ctrl+[ go-back) and stops it
  reaching CodeMirror. The same handler is also reachable from the plugin's own window
  and document capture listeners as a fallback, deduped with a `WeakSet` of handled
  events.
- **Pop on every exit.** Every exit path pops that scope: action, cancel, blur, error,
  and unload. A leaked catch-all scope would swallow the keyboard, so `disarm()` is the
  single exit and runs in `finally`.
- **Key normalization.** Ignore modifier-only keydowns (pressing Shift for `H` must not
  cancel), `repeat`, and composition. Read letters from `event.key`, falling back to
  `event.code` (`KeyH` plus `shiftKey`) when Ctrl turns `key` into a control character.
  Meta or Alt chords cancel the prefix and pass through (`Cmd+S` still saves). Esc,
  Ctrl+[, and q cancel. A mapped key runs its plan. A mapped key whose plan is blocked
  keeps the prefix armed, reveals the card at once, nudges its row, and states the
  reason. An unmapped printable key is consumed, so it never edits the note or runs a
  Vim command; it keeps the prefix armed and shows "‹k› isn't a window key". Other
  non-printable keys cancel and pass through.
- **One action per arm, run in sequence.** A Ctrl+W pressed while an action is still
  running waits for it, then plans from a fresh snapshot. At arm time, record the source
  leaf and the snapshot. Before executing, re-snapshot. If the source leaf is no longer
  active or no longer in the same split, cancel with "Layout changed — try again" rather
  than act on a different tab.
- **Cancel triggers.** Esc, Ctrl+[, q, a mousedown outside the card, window blur or
  `visibilitychange`, focus moving into a modal, a prompt, or any element outside the
  armed context, and any `layout-change` or `active-leaf-change` the prefix did not
  cause. No silent timeout is needed, because the card makes the armed state visible.

### The hint — calm, truthful, theme-native

The card is the feature. If the verbs work but the hint is a `Notice`, the feature has
failed. Every color comes from Obsidian theme variables (`--interactive-accent`,
`--text-on-accent`, `--background-primary`, `--background-secondary`,
`--background-modifier-border`, `--text-normal/muted/faint`, `--shadow-s/l`,
`--color-orange`), so light, dark, and CSS snippets all work. The design follows an 8px
spacing grid, uses the interface font everywhere except key caps (monospace), uses no
emoji, and draws icons with Lucide through `setIcon` (`arrow-left`, `arrow-right`,
`arrow-left-right`, `columns-2`), falling back to ← → ⇄ ◫ when `setIcon` is absent
(tests and previews).

**Tier 1 — armed, 0 ms.** A fixed, pointer-transparent ring overlays the focused split's
`containerEl` rect, so the split's own layout never shifts. It has a 10px radius, a 2px
inset `--interactive-accent` stroke, a 3px outer halo at 18% accent, and fades in over
90ms. A key-cap badge `⌃W` (`Ctrl W` off macOS) sits in the ring's top-right inner
corner: an accent pill with `--text-on-accent` text, 11px semibold monospace, and
`--shadow-s`. The editor keeps focus and caret, and nothing dims. Fluent `Ctrl+W l`
(about 150–250 ms) shows only this tier.

**Tier 2 — card, after a 300 ms pause** (`WINDOW_PREFIX_HINT_DELAY_MS`, no setting), or
at once when a blocked or unmapped key is pressed, or at 0 ms under
`prefers-reduced-motion`. The card is non-modal, never takes focus, and is
`position: fixed`. It is centered on the _main area_ (the `rootSplit` rect, so sidebars
never push it off-center), 16px above the area's bottom edge, with
`width: min(520px, areaWidth − 32px)`. It uses a 12px radius, a 1px border,
`--shadow-l`, `--background-primary`, padding `12px 14px 10px`, and
`z-index: var(--layer-popover)`. Wireframe for the screenshot layout, with focus on the
left split:

```text
╭────────────────────────────────────────────────────────────────╮
│ [⌃W]  Window                                    2 splits  [esc] │
│                                                                │
│ ┆H┆ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓ ┌─────────────────────┐      │
│     ┃ sase_blog               3  ┃ │ sase_blog_contents [l]│     │
│     ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛ └─────────────────────┘      │
│                                                                │
│ FOCUS                            TAB                           │
│ [h]  left edge                   [H]  ← new split              │
│ [l]  → sase_blog_contents        [L]  → join sase_blog_contents│
│ [w]  ⇄ sase_blog_contents        [v]  ◫ split right            │
│                                                                │
│ esc · q cancel                                                 │
╰────────────────────────────────────────────────────────────────╯
```

- **Header.** An accent key cap, then the title "Window" at weight 650. On the right, a
  muted "N splits" / "1 split" and an `esc` key cap.
- **Minimap** (44px tall). One pill per split, with flex-grow proportional to its real
  width and a minimum of 56px. Splits stacked in one column are drawn stacked. Each pill
  shows the selected tab's display title, ellipsized, plus a small muted tab-count
  bubble. The active pill gets an accent border, a 12% accent tint, and bold text.
  Neighbour pills carry a tiny `h` or `l` key cap where focus would land, which teaches
  the spatial mapping at a glance. A dashed 40px ghost slot labelled `H` or `L` marks an
  edge where a move would _create_ a split. With more than four columns, show the active
  split ±1 plus `+N` counters at the ends.
- **Key grid.** Two columns under small-caps headers "Focus" and "Tab" (10.5px, 0.06em
  tracking, `--text-faint`). Rows pair lowercase with uppercase (`h|H`, `l|L`, `w|v`),
  so Shift visibly means "take the tab along". Each cell has a 22px `bob-key-card-key`
  cap, a glyph, and a target title (`--text-normal`, ellipsized). An unavailable cell
  keeps its place at 45% opacity, with the reason in faint italics. A layout-changing
  consequence shows as a small amber chip, "closes split"
  (`color-mix(--color-orange 16%)` background, `--color-orange` text). It is
  information, not alarm, because nothing is lost.
- **Footer.** Faint "esc · q cancel", plus an `aria-live="polite"` message line for
  nudges ("Already the leftmost split", "j isn't a window key"). The card has
  `role="status"` and announces a one-line summary of the keys when it appears.

**Copy table** (the model phase owns these strings; polish the wording, keep the
meaning):

| Key       | Available                                                | Unavailable reason                                                                    |
| --------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `h` / `l` | `← {title}` / `→ {title}`                                | `left edge` / `right edge`; `no other split`                                          |
| `w`       | `⇄ {title}`                                              | `no other split`                                                                      |
| `H` / `L` | `← join {title}` (+ `closes split` chip) · `← new split` | `already leftmost` / `already rightmost`; `only tab · v splits`; `layout API changed` |
| `v`       | `◫ split right`                                          | `no tab to split`                                                                     |

`{title}` is the destination split's selected tab (`leaf.getDisplayText()`, or
`empty tab` when there is none).

**Motion** (all with an ease-out curve). The card fades in and rises 6px→0 over 120 ms,
and fades out over 80 ms. After a focus or move, the ring jumps to the destination and
pulses once (220 ms, matching the yank-flash timing) before fading. An edge bump is a
160 ms inner glow on the bumped side of the ring. A row nudge is a 160 ms, 3px
horizontal shake while its reason brightens. Under `prefers-reduced-motion` these become
plain fades, and the card appears at 0 ms.

**Receipts.** Successful focus and `v` show no receipt, because the screen already
changed. `H` and `L` show one compact pill in the card's slot for 1.6 s (`aria-live`),
for example `Moved “sase_blog” → sase_blog_contents · left split closed` or
`Moved “sase_blog” → new right split`.

**Persistent focused-split underline** (research A8). Whenever the main area has two or
more visible splits, the active split's tab-header strip carries a quiet inset 2px
underline at 75% accent. It uses CSS on `.workspace-tabs.mod-active` if the spike
confirms that class exists. Otherwise, toggle a `bob-wp-active-split` class on
`active-leaf-change`, gated by a `bob-wp-multi-split` body class.

### Reliability rules

- **Same leaf, lossless.** A move reparents the _same_ leaf object, so undo history,
  cursor, selection, scroll, folds, pin, view mode, and back/forward history all
  survive. Clone-and-reopen (`setViewState` into a new leaf, or `openFile`) is
  forbidden, because it silently loses undo.
- **Invariants after every move.** The leaf's parent is the target. The leaf appears in
  the workspace exactly once. The source has either collapsed or still holds tabs. The
  active leaf is the moved leaf. The total content-leaf count is unchanged. On any
  failure, restore the leaf to its original split and index, refocus it, and show one
  notice. Never leave an empty split behind.
- **Upgrade canary.** Feature-detect the internals at load and again before each move
  (`children` arrays, `insertChild`, `removeChild`, `selectTab`, `createLeafBySplit`,
  `recomputeChildrenDimensions`). If any is missing, the `H`/`L` rows dim with "layout
  API changed" and no move runs. Fail visibly; never move a tab in an unknown way.
- **Edge creation without collapse races.**
  `createLeafBySplit(leaf, 'vertical', before)` makes the new split with a placeholder
  leaf. Move the real tab in, then detach only that placeholder, so the new split is
  never empty mid-operation.
- After any layout change, call `workspace.requestSaveLayout()`, so the result survives
  a reload through `workspace.json`.

### Out of scope for v1 (reserved, designed later)

`j`/`k`/`J`/`K` and horizontal splits, `p` (previous split), `=` and `<`/`>` resizing,
counts, `x` exchange, `c`/`o` (close or merge — still an open product question),
numbered splits, sidebars, popout windows, and hover editors.

## Phase: spike — Real-Obsidian layout and keyboard spike

**Goal:** settle the unknowns with evidence from real Obsidian before any product code
depends on them, and leave behind the e2e harness that later phases extend.

Build `scripts/obsidian-window-prefix-e2e.cjs`, a `-lib.cjs`, and an offline
`scripts/test-obsidian-window-prefix-e2e.cjs` (`--self-test`; added to `test:unit`).
Model them on `obsidian-copy-card-e2e*.cjs` and `obsidian-vim-yank-flash-e2e.cjs`, built
on `obsidian-startup-smoke-lib.cjs`. The harness must:

- run an owned scratch vault and profile with all repo plugins installed, Vim mode on,
  at least six fixture notes (enough for the decision-contract vectors), a tiny PDF or
  image, and a `hotkeys.json` that mirrors the Mac. That file unbinds Ctrl+W from
  `workspace:close` (`[]`) and binds `Ctrl+L` to a harmless command, so suffix-collision
  probes are real.
- offer helpers that later phases reuse: build a layout from a signature such as
  `[A B*] | [C]`; read the live layout back as the same signature; send trusted keys
  through CDP `Input.dispatchKeyEvent`; type into an editor and read its text; reload
  the app; switch light and dark themes; emulate reduced motion; take screenshots; and
  run named groups with `--only`.

Its `--only spike` group answers, with each finding recorded in the JSON report:

1. Moving a tab between splits with `removeChild` / `insertChild` / `selectTab` (plus
   `recomputeChildrenDimensions`, `onLayoutChange`, `setActiveLeaf`,
   `requestSaveLayout`) keeps the same leaf and editor. Type, move, then `u` undoes.
   Check that cursor, scroll, folds, pin, and back/forward history survive, and that
   reading-view and PDF/image leaves still render after the move.
2. Whether removing a split's last tab collapses that split by itself, or what explicit
   call does, and that the width is redistributed and the layout survives an app reload.
3. Whether `createLeafBySplit(leaf, 'vertical', true)` places the new split on the left,
   and where `duplicateLeaf(leaf, 'split', 'vertical')` lands and focuses.
4. Keyboard precedence:
   - (a) with Ctrl+W unbound, a window capture listener sees Ctrl+W before CodeMirror,
     not `defaultPrevented`;
   - (b) with Obsidian's default Linux binding, Close tab runs first and the listener
     sees `defaultPrevented`;
   - (c) a pushed catch-all `Scope` intercepts a bound suffix chord (Ctrl+L, Linux
     Ctrl+[) before its command runs, and keeps plain letters from Vim;
   - (d) `app.keymap.popScope` removes a scope that is no longer on top (a modal opened
     above it);
   - (e) insert-mode Ctrl+W still deletes a word when the listener passes through;
   - (f) for the record only, whether `app.scope.register(['Ctrl'], 'w', …)` would
     preempt a bound Close tab.
5. Facts for later phases: whether the bundled Vim still has its idle `<C-w>` entry; the
   native focus-split command IDs; whether `.workspace-tabs.mod-active` exists; whether
   `getMostRecentLeaf(group)` returns that group's most recent tab; and how deferred
   (unloaded) background leaves behave when focused or moved (`loadIfDeferred`).

**Done when:** the harness and self-test are committed, `npm test` passes, the real run
on athena completes, and a `SPIKE VERDICT:` note on this phase's bead records the
findings. The verdict states whether the move is lossless, gives the exact working move
and create call sequences, collapse behavior, and placement facts, and lists the
keyboard-precedence answers (a)–(f). Later phases read that note (`sase bead show`).
This phase changes no plugin fragments. If reparenting turns out unsafe, say so plainly
in the verdict; `actions` then follows its contingency.

## Phase: model — Pure layout model, planners, key table, and view model

**Goal:** encode the whole design contract as pure, Obsidian-free functions, so dispatch
and hint read one table and can never disagree.

Add `471-window-layout-model.js` and `472-window-prefix-keys.js` (names are a guide;
keep each at or under 1000 lines), registered in `fragments.json` near
`470-keydown-and-freshness.js`, with exports in a new `720-window-prefix-exports.js`.

- **Snapshot (input).**
  `{ groups: [{ id, rect: {left, top, right, bottom}, tabs: [{ id, title, pinned, viewType }], selected, recency }], activeGroupId, activeTabId, capabilities: { move, moveReason }, platform: "mac"|"other" }`.
  Drop zero-area groups. Cluster groups into columns by horizontal overlap.
- **Geometry.** The left neighbour is the group whose right edge is closest to the
  active group's left edge, among groups with any vertical overlap. Break ties by
  distance, then vertical overlap, then recency; the right neighbour mirrors this. Next
  and cycle order is reading order, wrapping.
- **Planners.** `planWindowAction(snapshot, action)` for `focus-left`, `focus-right`,
  `focus-next`, `move-left`, `move-right`, and `split-right` returns
  `{ action, kind: "focus"|"join"|"create"|"duplicate"|"blocked", sourceGroupId, targetGroupId, tabId, insertIndex, side, collapsesSource, reason }`.
  It implements the decision contract, the insertion rule, every blocked reason, and the
  `capabilities.move === false` canary.
- **Keys.** `WINDOW_PREFIX_KEYS`: a frozen table of
  `{ key, aliases, action, column: "focus"|"tab" }`. Add
  `resolveWindowPrefixKey(eventLike)`, which returns
  `{ type: "action", action } | { type: "cancel" } | { type: "ignore" } | { type: "cancel-pass" } | { type: "unmapped", label }`.
  It covers held Ctrl, Shift-only keydowns, Meta/Alt pass-through, repeat, IME, arrows,
  a second Ctrl+W, Esc, Ctrl+[, and q.
- **Arming verdict.** `windowPrefixArmVerdict(context)` decides over a plain descriptor
  `{ key: {ctrl, meta, alt, shift, code, key, repeat, composing, defaultPrevented}, window: "main"|"popout", leafArea: "main"|"sidebar"|"none", focus: "editor"|"text-input"|"overlay"|"view", vimMode, vimPending }`
  and returns `{ arm, reason }`.
- **View model.** `buildWindowPrefixModel(snapshot)` returns
  `{ title, splitCount, minimap: { columns, ghosts, overflow }, rows: [{ key, column, glyph, target, available, reason, consequence, plan }], summary }`,
  using the copy table above. Each row's `plan` is _the same object_ `planWindowAction`
  returns for that key.
- **Signatures.** `formatWindowLayoutSignature(snapshot)` and its parser, for the
  `[A B*] | [C]` notation that tests and the e2e harness share.

**Tests:** add `scripts/test-navigation-window-layout-model.cjs` and
`scripts/test-navigation-window-prefix-keys.cjs` (both in `test:unit`). They cover every
decision-contract vector plus: one, two, three, and six columns; a nested horizontal
split; uneven widths; pinned prefixes on both sides; a lone tab at either edge; a
missing API capability; the full key-normalization matrix; arming verdicts for each
focus context; and a parity test asserting that every row's `plan` deep-equals
`planWindowAction` for its key across all fixtures. No version bump. Run `npm run build`
and `npm test`, then sync.

## Phase: actions — Live layout adapter and lossless executor

**Goal:** turn the model's plans into real workspace changes that are lossless,
verified, and reversible.

First read the `spike` bead's `SPIKE VERDICT:` note and use its call sequences verbatim.
Add a `BobNavigationHotkeysWindowActionsMixin` in `686-plugin-window-actions.js`,
installed in `690`.

- **`snapshotWindowLayout()`.** Builds the model snapshot from
  `workspace.iterateRootLeaves`. Keep only leaves whose root is `rootSplit` in the main
  window, group them by `leaf.parent`, and take rects from
  `parent.containerEl.getBoundingClientRect()`. Read pin state, display text, and
  recency, and fill `capabilities` from the canary. Never trust `getLeavesOfType` order,
  and never use `getAdjacentLeafInDirection`. Keep a map from snapshot ids back to live
  `WorkspaceTabs` and `WorkspaceLeaf` objects.
- **`runWindowAction(action, {expectedSourceLeaf})`.** Revalidates, plans, and executes:
  - focus: use `setActiveLeaf(leaf, {focus: true})`, reusing the existing focus helpers
    in `630-plugin-vim-link-and-leaves.js`, and load deferred leaves first.
  - join: reparent, select the most recent remaining tab in the source, and collapse the
    source if it emptied.
  - create: placeholder split, move the tab in, detach the placeholder.
  - duplicate: `duplicateLeaf`, then focus the new split.

  Then run the invariant check, roll back on failure, and call `requestSaveLayout`. Runs
  happen one at a time through a promise lock. It never calls `recordJump`. It returns
  `{ ok, kind, plan, receipt, destinationRect, bumpSide, reason, selfCaused: true }` for
  `prefix` to present, and shows no UI of its own.

- **Canary.** Run the canary at load and again before each move. Missing internals set
  `capabilities.move = false` with the reason "layout API changed".
- **Contingency.** If the spike verdict says lossless reparenting is not possible, ship
  focus, next, and split only. In that case `capabilities.move` is false with the reason
  "lossless move unavailable", and you file a follow-up task bead through
  `/sase_new_task` to revisit it. Never implement a clone-based move.

**Tests:** add `scripts/test-navigation-window-actions.cjs`, using fake
`WorkspaceTabs`/`WorkspaceLeaf` objects that record calls: every decision-contract
vector, rollback when `insertChild` throws (the tab ends up back at its original index),
placeholder cleanup when create fails mid-way, canary blocking, a stale
`expectedSourceLeaf` producing "layout changed", and serialized concurrent calls. Extend
the e2e harness with `--only actions`. That group drives `runWindowAction` directly
through the plugin instance against the real decision-contract vectors (checked by
layout signature), proves undo, cursor, scroll, and pin survive `L` and `H`, proves the
layout survives a reload, and checks that a reading-view leaf and a PDF or image leaf
survive moves. Run it on athena. No version bump. Sync.

## Phase: hint — Ring, card, minimap, and motion painter

**Goal:** make the hint beautiful, exactly as specified in "The hint" above, as a pure
painter that is easy to preview and test.

Add `473-window-hint-view.js` (split it into a second fragment if it grows past 1000
lines). It defines a `createWindowHintHost(doc, { setIcon })` that renders only from the
model's view model plus rects. It has no workspace access, reuses one DOM tree
idempotently, and never throws, because the painter is cosmetic. Its API:
`showArmed(model, ringRect, areaRect)`, `showCard(model)`, `nudge(key, message)`,
`bump(side)`, `pulse(rect)`, `receipt(text)`, `message(text)`, `hide()`, and
`destroy()`. It also respects `prefers-reduced-motion` (via `matchMedia` when present).
Add a `/* Window prefix */` section to `styles.css` with the ring, badge, card, minimap,
key grid, chips, receipt, motion keyframes, the reduced-motion overrides, and the
persistent focused-split underline rules. Reuse `.bob-key-card-key` for the caps.

**Preview harness.** Add `scripts/preview-window-prefix.cjs`, modeled on
`preview-bracket-mode.cjs`. It renders the production painter with `styles.css` and
fixture workspace DOM in headless Chromium, in light and dark, for these states:

1. tier-1 only, two splits;
2. the screenshot layout's card;
3. one split with three tabs (ghost slot);
4. one split with one tab (only `v`);
5. three splits, middle focused;
6. six splits (overflow counters);
7. a blocked `h` nudge at the left edge;
8. `L` from a sole tab with the "closes split" chip;
9. the `layout API changed` dimming;
10. the receipt pill;
11. a 420px-wide main area;
12. reduced motion.

Look at every PNG yourself and iterate until the result is calm, aligned, legible, and
balanced in both themes. Check that the ring never covers the tab titles, that
ellipsized titles keep their counters, and that dimmed rows still read clearly.

**Tests:** add `scripts/test-navigation-window-hint-view.cjs`, using `modal-harness.cjs`
`ElementStub`. It covers row availability and reasons, the consequence chip, minimap
pills and ghosts, overflow, aria attributes, idempotent repaint, `destroy` cleanup, and
`setIcon` fallback glyphs. No version bump. Sync.

## Phase: prefix — Ctrl+W prefix state machine

**Goal:** make `Ctrl+W <key>` dependable in every context listed in "Arming gate and key
handling", with the two-tier hint timing.

Add `BobNavigationHotkeysWindowPrefixMixin` in `687-plugin-window-prefix.js`, installed
in `690`, and called from `onload` in `490-plugin-lifecycle.js` next to the other
`register*InputListeners()` calls. It contains:

- **Live context.** Window and document capture listeners, deduped with a `WeakSet`.
  Build the arming context from the live event, the focused editor, and the DOM, and
  call `windowPrefixArmVerdict`.
- **States.** `idle → armed → carded → running → idle`, with a single `disarm()` that
  clears timers, pops the keymap `Scope`, hides the painter, and drops the source-leaf
  capture.
- **Hint timing.** `WINDOW_PREFIX_HINT_DELAY_MS = 300`, applied as described in the hint
  section.
- **Dispatch.** `resolveWindowPrefixKey` → `runWindowAction` → painter feedback: pulse
  or receipt on success; bump, nudge, or message on blocked and unmapped keys.
- **Cancel and invalidation.** Every cancel trigger from the design contract.
  `active-leaf-change` and `layout-change` events produced by the prefix's own action
  are ignored.
- **Library guards.** Guard the `Scope` and `Platform` requires, so existing test stubs
  that lack them keep loading.
- **Scope contingency.** If spike finding 4(c) shows that a pushed `Scope` does not
  preempt bound hotkeys, keep the capture-listener path. A suffix that a hotkey already
  handled (`defaultPrevented`) then cancels the prefix quietly, and plain-letter
  suffixes are unaffected. The docs must name the collision (Ctrl+L in this vault).

**Tests:** add `scripts/test-navigation-window-prefix.cjs`, driving the state machine
with fake events, timers, keymap, and actions. It covers:

- arming in each context and passing through in each excluded one;
- yielding to a `defaultPrevented` Ctrl+W;
- Shift-only keydowns not cancelling, and held-Ctrl suffixes;
- a second Ctrl+W cycling;
- Meta and Alt passing through;
- blocked keys staying armed, and unmapped printable keys being consumed;
- the scope popped on every exit path, including a thrown action;
- a stale source cancelling; serialization;
- no card painted for a fluent sequence, and the card at 300 ms.

Extend the e2e harness with `--only prefix`, which checks:

- `Ctrl+W l`, `h`, `L`, and `H` end-to-end on the screenshot layout;
- `Ctrl+W Ctrl+W` toggles;
- `Ctrl+W Ctrl+L` focuses right and does not run the Ctrl+L hotkey;
- Esc leaves the layout signature and note text byte-identical;
- an unmapped `j` changes nothing;
- insert-mode Ctrl+W deletes a word;
- an open Task Card or Copy card passes Ctrl+W through;
- reading view and an empty tab arm;
- a pending `d` operator passes through;
- with the default Linux binding restored, Ctrl+W closes the tab and the prefix stays
  idle;
- after every case, the `app.keymap` scope stack equals its starting state.

Capture screenshots of the ring and of the card at 300 ms. Run on athena. Bump the nav
manifest minor version. Sync.

## Phase: finish — Commands, focused-split underline, docs, acceptance, rollout

**Goal:** round the feature out, document it, prove it end to end, and ship it.

- **Palette commands.** Add commands with no default hotkeys, so every verb can be
  remapped and reached without the prefix:
  - `window-focus-left` (Focus split left)
  - `window-focus-right` (Focus split right)
  - `window-focus-next` (Focus next split)
  - `window-move-tab-left` (Move tab to left split)
  - `window-move-tab-right` (Move tab to right split)
  - `window-keys` (Show window keys), which opens the prefix with the card shown at once

  They route through `runWindowAction` and the same painter for receipts and pulses. A
  blocked command shows its reason as a notice. The `v` verb gets no Bob command,
  because Obsidian's native Split right already covers it.

- **Focused-split underline.** Implement it per the spike's `mod-active` finding,
  including the `bob-wp-multi-split` body class, updated on `layout-change`, and cleanup
  on unload.
- **Docs** (read the `docs.md` reference memory first):
  - a new `docs/window-keys.md` page, "Window keys (Ctrl+W)", in `docs/nav.yml`. It
    covers the key table, the decision-contract pictures, where Ctrl+W arms and where it
    passes through, the one-line "H/L move a tab, not a Vim window" note, the Linux
    behavior, v1 scope, and palette commands;
  - a bullet in `docs/bob-navigation-hotkeys.md` under "Notes, tabs, and keyboard
    layers" that links the page;
  - one sentence added to the nav manifest description.

  Then run `npm run docs:gen`, `npm run docs:inventory:check`, and
  `BOB_CLI_DIR=<this bob-cli checkout> npm run docs:check`.

- **Vault.** Add one comment line to `~/bob/obsidian_vimrc.md`'s vimrc block:
  `" <C-w> belongs to bob-navigation-hotkeys' window prefix — a <C-w> map here never fires`.

  > [!decision] linux_ctrl_w = rebind Also set
  > `"workspace:close": [{"modifiers": ["Meta"], "key": "W"}]` in
  > `~/bob/.obsidian/hotkeys.json`, keeping every other entry byte-identical. Cmd+W on
  > macOS is unchanged; on Linux, Close tab moves to Super+W and Ctrl+W arms the prefix.
  > Note this in the docs page.

  > [!decision] linux_ctrl_w = keep Leave `hotkeys.json` alone. The docs say that on
  > Linux, Obsidian's Close tab keeps Ctrl+W unless it is rebound, and show the one-line
  > rebind.

- **Acceptance.** Run the full harness on athena with every group, plus a `visual`
  group: screenshots of the armed ring, the card, the receipt, and the underline in
  light and dark at one, two, and three splits. Review every screenshot and fix any
  polish issue you find. Bump the patch version if plugin code changed, then run the
  final `bob plugins sync`. Record the report path and verdict in the bead notes.

  > [!decision] window_prefix_record After shipping, use `/sase_memory_write` to add an
  > accepted strand to the bob-cli `decisions` web: "Ctrl+W Is A Layout-Only Window
  > Prefix". It applies to bob-plugins. Its claim: `Ctrl+W` arms only where unbound and
  > outside text entry; `h`/`l` are directional and never wrap; `w` and a second
  > `Ctrl+W` cycle; `H`/`L` move the live tab, creating and collapsing splits, never
  > cloning it; no key closes a note. Its evidence: the research ref, this plan, and the
  > landing commits. Then run `sase memory init`.
