---
tier: tale
size: medium
title: Finish and land the Ctrl+W window prefix epic (bob-cli-m0)
goal: 'Prove every Ctrl+W window move in real Obsidian, make sure a move never leaves
  an empty split behind, re-arm a Ctrl+W pressed while an action runs, make the full
  athena harness green, ship nav 2.83.2, and close epic bob-cli-m0.

  '
proposed_by: bbugyi200.apollo.bob-cli-m0.land
proposed_with: claude/opus
bead: bob-cli-m0
status: done
---

- **PARENT:**
  [202611/obsidian_ctrl_w_window_prefix.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/obsidian_ctrl_w_window_prefix.md)
- **BEAD:**
  [bob-cli-m0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-m0/README.md)

# Plan: finish and land the Ctrl+W window prefix epic (bob-cli-m0)

## Context

Epic `bob-cli-m0` ("Ctrl+W window prefix for Obsidian vertical splits",
`plan:202611/obsidian_ctrl_w_window_prefix.md`) has all six phases closed (`m0.1` spike
through `m0.6` finish). Its product code is in the **`bob-plugins`** linked repo, plugin
`plugins/bob-navigation-hotkeys`. The epic commits are a2550d6, 615aaea, 4678586,
1a9e304, c725370, and 0e16e30. The vault runs nav **2.83.1**, built from the clean
commit 0e16e30.

The land agent's review found the following epic work still unfinished. This tale
finishes it and then closes the epic. Read first:

- `sase bead read bob-cli-m0 -r "<why>"` — the epic, its DECISIONS, and the land-triage
  note. Its PLAN path is the epic plan file. Read that plan's "Design contract" section
  and the "Phase: actions" section.
- `sase bead read bob-cli-m0.1 -r "<why>"` — `SPIKE VERDICT:` (exact move, create, and
  collapse call sequences, plus harness caveats: `iterateRootLeaves` stops on a truthy
  callback return, `selectTab` resets `scrollTop`, and `duplicateLeaf` views mount
  asynchronously).
- `sase bead read bob-cli-m0.6 -r "<why>"` — acceptance notes and the stale spike probe
  report.
- Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and
  read its `AGENTS.md`. Edit only `src/` fragments, regenerate `main.js` with
  `npm run build`, and keep each hand-edited fragment at or under 1000 lines. Finish
  with `bob plugins sync`.

Epic DECISIONS remain binding:

- `linux_ctrl_w = keep`: never touch the vault `hotkeys.json`.
- `window_prefix_record = no`: edit no SASE memory note.

Relevant files in `plugins/bob-navigation-hotkeys/src/`:

- `266-window-prefix-model.js`: the pure planners `planWindowPrefixKey`,
  `normalizeWindowPrefixKeydown`, `getWindowPrefixArmingVerdict`, and
  `buildWindowPrefixViewModel`.
- `267-window-prefix-actions.js`: the live snapshot adapter, plus
  `executeWindowPrefixPlan` and `executeWindowPrefixMove`.
- `473-window-hint-view.js`: the painter.
- `687-plugin-window-prefix.js`: the state machine.
- `688-plugin-window-prefix-commands.js`: the palette commands and the underline. Its
  `runWindowPrefixPaletteKey(key)` runs snapshot → plan → execute for any key, including
  `v`.

Harness files:

- `scripts/obsidian-window-prefix-e2e.cjs`: about 2,480 lines, with groups `spike`,
  `prefix`, and `visual`; `actions` is still a skipped placeholder.
- `scripts/obsidian-window-prefix-e2e-lib.cjs`: owns `DECISION_VECTORS`, the signature
  grammar with `*`, `'`, and `!`, the fixtures, and `GROUPS`.

Unit tests are the `scripts/test-navigation-window-*.cjs` files, all listed in
`test:unit`.

## Work

### 1. A move must never leave an empty split behind (`267-window-prefix-actions.js`)

`executeWindowPrefixMove` relies on Obsidian auto-collapsing a tab group that
`removeChild` empties, but nothing checks that it did. Its invariant computes
`sourceCollapsed` from leaf-derived groups (`windowPrefixLayoutGroups(afterLeaves)`). An
emptied `WorkspaceTabs` that is still attached to its parent split therefore "passes"
the check, and the empty split stays on screen. The unit fakes always auto-collapse
(`scripts/test-navigation-window-prefix-actions.cjs`), so this has never been tested.

- Decide collapse from the live tree. The source group's immediate parent is the last
  link of the `sourceChain` captured before the move. The source counts as collapsed
  only when it has no children **and** that parent no longer contains it (and
  `sourceGroup.parent` is unset or different).
- If the plan expects a collapse and the emptied source group is still attached,
  collapse it explicitly. Call `parent.removeChild(sourceGroup)`, then
  `recomputeChildrenDimensions` on the parent if present, then
  `workspace.onLayoutChange()`, and check again. A split that the move empties collapses
  per the design contract, so explicit collapse beats rollback. If it still will not
  detach, throw so the existing rollback restores the layout and shows the failure
  notice.
- If the plan does not expect a collapse, keep the existing `sourceStillHoldsTabs`
  check.
- Unit tests in `test-navigation-window-prefix-actions.cjs`:
  - (a) A fake `WorkspaceTabs` whose `removeChild` does **not** auto-collapse: a
    last-tab join and a middle-out move still end with no empty group attached to the
    root tree, and the result reports `sourceCollapsed: true`.
  - (b) A fake whose parent `removeChild` throws: the move rolls back to the original
    split and index, and the result reports `restored: true`.

### 2. A Ctrl+W pressed while an action runs must re-arm (`687-plugin-window-prefix.js`)

The design contract says: "A Ctrl+W pressed while an action is still running waits for
it, then plans from a fresh snapshot." Today `handleWindowPrefixSuffix` consumes that
Ctrl+W (`preventDefault`) and enqueues `tryArmWindowPrefix(event)`. That re-arm then
fails the `default-prevented` verdict that the prefix itself caused. The Ctrl+W is lost,
and the following suffix key is swallowed too. This shows up most after `v`, which waits
up to 1.5 s for the duplicate view to mount.

- When the running-state suffix is a Ctrl+W (`ctrlKey` and a resolved key of `w`),
  enqueue a re-arm that runs after the current action. Evaluate the arming context from
  the live state at that moment, and treat `defaultPrevented` as false only because the
  prefix consumed this event itself (it is in `windowPrefixHandledEvents`). Arm only if
  the verdict allows it.
- Suffix keys typed after that queued re-arm already enqueue behind it, so they dispatch
  against the new arm. Keep that behavior.
- Extend the serialization test in `scripts/test-navigation-window-prefix.cjs`:
  1. Arm, press `l`, and hold the gated action.
  2. Press Ctrl+W, then `l`, and release the gate.
  3. Assert that both `l` actions ran, one at a time (max concurrency 1), each planned
     from a fresh snapshot, and that the scope is popped at the end.

### 3. Add the missing `--only actions` real-Obsidian group (actions-phase deliverable)

The actions phase was supposed to add an `--only actions` group, but it never landed.
`runE2e` still records `actions/reserved` as skipped. Because of that, last-tab-out and
middle-out collapse, create-right, pinned placement, `v`, and losslessness through the
executor have never been proven in real Obsidian.

- Add the runner, ideally in a new `scripts/obsidian-window-prefix-e2e-actions.cjs`
  module that reuses the main harness's helpers (export them as needed). Wire it into
  `runE2e`, the `--help` text, the header comment, and the `GROUPS` comment in the lib,
  all of which still call `actions`/`visual` "reserved". Extend `--self-test` to cover
  the new group's argument parsing and its vector-to-key table.
- Launch with `wplib.macHotkeys()` (Ctrl+W unbound). Drive the executor through the
  plugin instance:
  `app.plugins.plugins["bob-navigation-hotkeys"].runWindowPrefixPaletteKey(key)`.
- For every `wplib.DECISION_VECTORS` entry, build `from` with `buildFromSignature`, run
  its key, and compare `readLive` → `signatureFromLive` to `to`. Use these keys:
  - `L`: create-right, join-right, last-tab-out, pinned-stays-pinned,
    unpinned-after-pins.
  - `H`: create-left, join-left, middle-out, edge-sole-tab.
  - `H` and then `L`: only-tab.
  - `v`: split.
  - `l`, then `h` at the edge, then `w` twice: focus.
- For the collapse vectors, also assert that the DOM under `.workspace-split.mod-root`
  holds exactly the expected number of `.workspace-tabs` and none of them is empty.
- Losslessness through the executor for `L` and `H`: type into the editor, move it, and
  press `u` to undo back to the exact prior text. Cursor, `scrollTop`, and pin state
  must also survive.
- After a move, call `requestSaveLayout`, then `reloadAndReconnect`; the signature must
  be unchanged.
- A reading-view leaf and the fixture PDF or PNG leaf must keep their view type and
  still render after `L` and `H`.
- After every case, the `app.keymap` scope stack equals its starting state
  (`keymapFinger` / `sameKeymap`).
- If a vector fails in real Obsidian, fix the executor, not the vector. The decision
  contract is authoritative.

### 4. Fix the stale spike probe (caused by the epic)

`spike/keyboard-unbound-capture` (finding 4a) still assumes a bare window capture
listener sees Ctrl+W first and that the event is not `defaultPrevented`. Since `m0.5`,
the shipped prefix consumes Ctrl+W at window capture, so the probe records `events=[]`
and fails. That is the spike's only failure (14/15).

Rewrite it to assert the shipped behavior with the Mac hotkeys (Ctrl+W unbound):

- Ctrl+W in Vim normal mode arms the prefix. Use `prefixDebug`: `armed`, and a
  `lastVerdict` that is allowed with `defaultPrevented: false`, which proves Obsidian's
  keymap left the unbound key alone.
- The leaf count, editor text, and Vim mode are unchanged.
- `Escape` cancels.

Keep the finding's report key (`findings.keyboard.a`).

### 5. Model test gaps from the model phase's test spec

In `scripts/test-navigation-window-prefix-model.cjs`, add:

- six columns (focus left, right, and next across all of them, and a view model with six
  splits);
- a stacked pair of splits sharing one column (nearest neighbour with vertical overlap,
  and the tie-break order: gap, then vertical overlap, then recency);
- uneven widths (`widthRatio` and nearest-edge geometry);
- a parity test across every fixture in the file, including the decision-contract
  fixtures: each `buildWindowPrefixViewModel(snapshot).rows[i].plan` deep-equals
  `planWindowPrefixKey(snapshot, row.key)`.

Note: the layout-signature formatter and parser live in the e2e lib rather than the
model fragment. That deviation from the epic plan's naming is accepted; do not move
them.

### 6. Build, verify, and ship

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.83.1 to **2.83.2**.
- Run `npm run build`, `npm run build:check`, and the window-prefix unit tests (all
  green).
- Run `npm test`. Known clean-base failures are tracked elsewhere: `bob-cli-m8`
  (block-marks dates), `bob-cli-km`, `bob-cli-ki`, `bob-cli-ju`, and `bob-cli-io` (perf
  budgets under load), `bob-cli-3w` (stage ranker), `bob-cli-f8` (task-wikilink
  budgets), and `bob-cli-fz` (headless Chromium hangs). Introduce no new failure, and
  confirm any other failure reproduces on clean `origin/master` before calling it
  pre-existing.
- Docs: update `docs/window-keys.md` only if behavior wording changes. Run
  `npm run docs:gen`, then `npm run docs:inventory:check`, then
  `BOB_CLI_DIR=<a bob-cli checkout> npm run docs:check`.
- Real-Obsidian acceptance on **athena**, which has Obsidian 1.15.1 at
  `/snap/bin/obsidian` and `xvfb-run`; apollo has neither:
  1. Copy the bob-plugins working tree, without `.git` and `node_modules`, to a fresh
     `mktemp -d` directory on athena over the `athena` Tailscale SSH alias.
  2. Run
     `xvfb-run -a node scripts/obsidian-window-prefix-e2e.cjs -e /snap/bin/obsidian --only <group> -r <dir>/report-<group>.json -o <dir>/shots-<group>`
     for each of `spike`, `actions`, `prefix`, and `visual`.
  3. Copy the reports back and review them. **Every group must pass every case**: spike
     15/15, actions all, prefix all, and visual all.
  4. Review the actions screenshots, if you take any.

  A missing host is a limitation to report, never a pass.

- Run `bob plugins sync` from a clean, committed tree. The vault deploy currently stamps
  clean commit 0e16e30, so a normal sync from a descendant commit needs no `-F`. Follow
  `AGENTS.md` if it refuses.

## Closeout (final step — closes the epic in this same turn)

Do this after the code, tests, athena acceptance, and sync are all done. It must not
wait on this work's own commit SHA, push, or CI.

1. Run `sase bead epic-symbols bob-cli-m0`. For each listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it.
   Re-key the entry only if a still-open later bead needs the exemption. At land time
   there were none.
2. Close the epic: `sase bead close bob-cli-m0 --note "<verification>"`. The note
   records:
   - the athena report results for all four groups;
   - the collapse hardening and Ctrl+W re-arm fixes, with their tests;
   - nav 2.83.2 synced;
   - that the land agent already verified the six phases against their commits;
   - that the land agent found no integration changes needed from the commits that
     landed during the epic: bank, catalog-scroll, Ctrl+Enter-from-last-PRE, capture
     `//`, and docs. The new bank setup card is an Obsidian modal, so the prefix already
     refuses to arm over it.
   - a pointer to the land-triage note on the epic for the follow-up outcomes.

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   epic symbols, clean them up and close again.

3. Run `just symvision` in bob-cli if the recipe exists. At land time the bob-cli
   `justfile` had none, so skip it and say so in the final response.
4. Set `status: done` in the frontmatter of the epic plan file, the PLAN path shown by
   `sase bead read bob-cli-m0` (`plan:202611/obsidian_ctrl_w_window_prefix.md`),
   replacing `status: wip`.
5. `bob-cli-m0` has no `parent_bead`, so the landing ends here.
