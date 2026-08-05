---
tier: tale
title: Renumber picker priority levels to P1-P3 and add a P4 someday level
goal:
  "The `Ctrl+Shift+P` priority property offers P1/P2/P3 for the levels previously labeled P2/P3/P4 (same written values
  and same day windows), plus a new P4 level that writes `[priority:: lowest]` and rolls a `scheduled` date 91-365 days
  out, with the notice icon and accent color extended to the fourth level and all docs restating that a task with no
  priority field is an implicit P0."
proposed_by: bbugyi200.athena.t1
create_time: 2026-08-05 13:32:49
status: done
---

# Plan: Renumber picker priority levels to P1-P3 and add a P4 someday level

## Repositories

This tale spans three repositories. Open the two non-primary ones through the `/sase_repo` skill and use the printed
path for every read and write:

- `sase repo open chezmoi -r "<reason>"` — owns `home/dot_config/bob/config.yml`, the source of the deployed
  `~/.config/bob/config.yml` that the plugin reads. After committing there, run `chezmoi update -a --force` so the
  change reaches `~/.config/`.
- `sase repo open bob-plugins -r "<reason>"` — owns `plugins/bob-navigation-hotkeys/main.js`, `styles.css`,
  `manifest.json`, `scripts/test-navigation-hotkeys.cjs`, and `README.md`.
- The primary `bob-cli` repo owns `docs/projects.md`.

Test the plugin from the `bob-plugins` checkout with `npm test` (runs `node --test scripts/*.cjs`) and
`npm run validate` (manifest check). Deploy with `bob plugins sync -p bob-navigation-hotkeys -r "$PWD"`; the explicit
`-r "$PWD"` source root is required in SASE workspaces, because the default repo path does not exist there.

## Problem

`~/.config/bob/config.yml` currently configures three priority levels:

| Picker label | Written field         | Random `scheduled` window |
| ------------ | --------------------- | ------------------------- |
| `P2`         | `[priority:: high]`   | 2-7 days from today       |
| `P3`         | `[priority:: medium]` | 8-30 days from today      |
| `P4`         | `[priority:: low]`    | 31-90 days from today     |

The numbering was chosen so that a task with no priority field reads as an implicit P1. Bryan wants the implicit "no
priority field" state to be **P0** (do it now, highest priority) instead, which frees the whole P1-P4 range for explicit
levels. Two changes follow:

1. Renumber the three existing levels down by one: P2 → P1, P3 → P2, P4 → P3. Each level keeps its written value and its
   day window; only the picker label moves.
2. Add a fourth level, P4, that behaves exactly like the others and rolls a random `scheduled` date 91-365 days out.

## Design decisions

**1. Relabel in place; no vault data migration.** The picker label never reaches the Markdown — the writers store
`level.value` (`[priority:: high]`), and the label only appears in the picker rows, the picker subtitle (`current: P2`),
and the transient priority notice. Every existing `[priority:: high|medium|low]` task in the vault therefore keeps its
exact bytes and simply displays one number lower. Do **not** rewrite vault tasks in this plan.

**2. New P4 writes `[priority:: lowest]`.** In the vault's Dataview task format, `priority` is a reserved Obsidian Tasks
key whose legal values are `highest`, `high`, `medium`, `low`, and `lowest`. Tasks-format parsers read trailing inline
fields right to left and stop at the first unrecognized field, so an invented value would hide every `[scheduled:: ]`,
`[id:: ]`, and `[dependsOn:: ]` field to its left from Obsidian Tasks, `bob query`, and the dash. That constraint is
already documented in the config comment and in `docs/projects.md:237-242`; keep it. With `high`/`medium`/`low` taken by
P1-P3, `lowest` is the only remaining name below `low`, and it is already accepted by the `bob-cli` trailing-field
parser (`src/native/task_status_hooks.rs:2095-2098` matches `"highest" | "high" | "medium" | "low" | "lowest"`), so no
`bob-cli` code change is needed. `highest` stays unused and reserved.

**3. The fourth level needs two small plugin changes to "work like the others."** Everything else about priority levels
is config-driven and already handles an arbitrary level count (validation, picker rows, roll, counted writes,
date-picker roll suggestion, notice text). Two spots are hard-coded to three levels and would silently degrade:

- `getPriorityLevelIconName()` (`main.js:10926-10937`) maps index 0/1/2 to `signal-high`/`signal-medium`/`signal-low`
  and falls back to `signal` — which is the **full-strength** four-bar Lucide icon. A P4 would show a stronger-looking
  icon than P1. Index 3 (and the fallback) must become `signal-zero`.
- `renderPriorityNoticeFragment()` (`main.js:11243-11249`) only emits `is-level-N` for `levelIndex <= 2`, so a P4 notice
  would fall back to the generic `--text-accent` border instead of a level color.

**4. P4 notice color is `var(--color-purple)`.** The existing ramp is index 0 orange, 1 yellow, 2 blue
(`styles.css:900-910`); purple continues it as the coldest/most distant step and stays inside the stylesheet invariant
test that forbids literal `#hex`/`rgb()`/`hsl()` in the priority-notice section
(`scripts/test-navigation-hotkeys.cjs:4102-4118`).

**5. Relative-date phrasing stays "in N days."** `formatRelativeDayOffset()` (`main.js:10785-10803`) will render a P4
roll as e.g. `in 203 days`, and a counted P4 session as `in 91-365 days`. Adding month/year phrasing is a separate
concern and is deliberately out of scope; do not touch that helper.

**6. Sorting is unchanged and out of scope.** Obsidian Tasks sorts `Highest > High > Medium > Normal > Low > Lowest`,
and a P0 task (no field) is `Normal` to Tasks, so P0 sorts between P2 and P3 in any `sort by priority` query. That is
pre-existing behavior under the old numbering (implicit P1 was also `Normal`) and this plan does not change it. Note it
in the commit message; do not try to fix it by remapping values.

## Implementation

### 1. chezmoi — `home/dot_config/bob/config.yml`

Replace the `levels:` list and fix the implicit-level comment. The final `priority` entry reads:

```yaml
# Choosing a level writes `[priority:: <value>]` and rolls a random date into the
# property named by `schedules`, drawn inclusively from that level's day window.
# A task with no priority field is implicitly P0: do it now, no roll.
#
# `value` is what lands in the note and MUST stay one of Obsidian Tasks' priority
# names (highest, high, medium, low, lowest). Tasks-format parsers read trailing
# inline fields right to left and stop at the first one they do not recognize, so an
# invented value such as `P2` would also hide every `[scheduled:: ]`, `[id:: ]`, and
# `[dependsOn:: ]` field to its left from Obsidian Tasks, `bob query`, and the dash.
# `label` is what the picker shows.
- name: priority
  values: priority
  schedules: scheduled
  levels:
    - label: P1
      value: high
      min_days: 2
      max_days: 7
    - label: P2
      value: medium
      min_days: 8
      max_days: 30
    - label: P3
      value: low
      min_days: 31
      max_days: 90
    - label: P4
      value: lowest
      min_days: 91
      max_days: 365
```

Only the `implicitly P1` → `implicitly P0` line and the `levels:` list change. Keep the `such as P2` sentence as-is: it
is an example of an _invented_ value, and the point that a label must never be written as a value still holds.

### 2. `plugins/bob-navigation-hotkeys/main.js` — two edits

- `getPriorityLevelIconName()` (`:10926`):

  ```js
  function getPriorityLevelIconName(levelIndex) {
    if (levelIndex === 0) {
      return "signal-high";
    }
    if (levelIndex === 1) {
      return "signal-medium";
    }
    if (levelIndex === 2) {
      return "signal-low";
    }
    return "signal-zero";
  }
  ```

  The fallback changes too, so any future fifth level degrades downward instead of jumping back to full strength.

- `renderPriorityNoticeFragment()` (`:11243`): widen the level-class gate to the fourth level.

  ```js
  const levelClass =
    Number.isInteger(model.levelIndex) && model.levelIndex >= 0 && model.levelIndex <= 3
      ? ` is-level-${model.levelIndex}`
      : "";
  ```

No other `main.js` change is required. `normalizePriorityLevelIndex()`, `createBulletPropertyValueItems()`,
`rollPriorityScheduledDate()`, `createPriorityRollDateItem()`, `getPriorityRollLevel()`, the single-task writer, and the
counted writer are all driven by `property.levels` / `levelsByValue` and already handle four levels. Confirm this by
reading them rather than by assuming.

### 3. `plugins/bob-navigation-hotkeys/styles.css`

Add the fourth level color immediately after `.bob-nh-notice.is-level-2` (`:908-910`), inside the existing priority
notice section so it stays covered by the theme-token invariant test:

```css
.bob-nh-notice.is-level-3 {
  --bob-nh-level-color: var(--color-purple);
}
```

### 4. `scripts/test-navigation-hotkeys.cjs`

Most edits are mechanical label renames driven by the fixture. Rename first, then run the suite and fix fallout.

- **Fixture** `createPriorityPickerConfig()` (`:3655-3671`): mirror the deployed config exactly — `P1`/`high`/2-7,
  `P2`/`medium`/8-30, `P3`/`low`/31-90, `P4`/`lowest`/91-365.
- **Config-validation fixtures** `:752-753` (whitespace trimming, `" P2 "`/`" high "` and `"P3"`/`"medium"`), `:779-784`
  (normalized levels and `levelsByValue`), `:801` (`label: "P2"`), and `:857` (duplicate-value case, `label: "P3"`):
  renumber to `P1`/`P2` so the suite reads consistently with the deployed config. These assert trimming/duplicate
  detection, not specific numbers, so the values and windows stay as they are.
- **Counted current-label assertions** `:1149` (`currentLabel` for `high`) becomes `"P1"`; `:1169` (`currentLabels` for
  `high` + `low`) becomes `["P1", "P3"]`.
- **Notice model / picker / roll assertions** at `:3869`, `:3881`, `:3914`, `:3948`, `:4257-4267`, `:4281`, `:4310`,
  `:4340-4349`, `:4372`, `:4390`, `:4526-4531`, and `:4659`: every `P2` referring to the `high` level becomes `P1`. The
  rolled dates, ISO strings, weekdays, and relative spans in these assertions are unchanged, because the `high` window
  is still 2-7 days and `random: () => 0` still rolls `minDays`.
- **Value-item list** `:4205-4235`: renumber the three existing entries and append a fourth expected row —
  `label: "P4"`, `value: "lowest"`, `detail: "lowest · in 91–365 days"`, `searchText: "P4 lowest 91–365 days"`,
  `current: false`, `priorityLevel: property.levels[3]`. Keep the en dash (`–`) used by the existing rows. The
  `currentLabel` assertion at `:4244` (for `[priority:: medium]`) becomes `"P2"`.
- **Icon table** `:3826-3830`: keep 0/1/2, and change index `3` and index `9` to `"signal-zero"`.

New coverage to add (the behavior this plan introduces is otherwise unpinned):

- **Fourth-level notice class.** Extend or mirror the renderer test at `:3983-4010`: build a model with
  `level: property.levels[3]`, `levelIndex: 3`, and assert the card classes include `is-level-3` and that
  `model.iconName === "signal-zero"`.
- **Wide-window roll bounds.** Call `helpers.rollPriorityScheduledDate(property.levels[3], new Date(2026, 7, 3), r)`
  with `r = () => 0` and with `r = () => 0.999999`, asserting `2026-11-02` (base + 91) and `2027-08-03` (base + 365).
  This pins that `span = maxDays - minDays + 1 = 275` and the `clampNumber` guard keep the roll inside the window for a
  window much wider than the existing ones.
- **End-to-end P4 write.** Add a `choosePriorityLevel(harness, "P4")` case alongside the existing P2-now-P1 test at
  `:4248`: with `baseDate: new Date(2026, 7, 3)` and `random: () => 0`, assert the line becomes
  `- [?] #task One [priority:: lowest] [scheduled:: 2026-11-02] ^one` (future date ⇒ Blocked, same as today) and that
  the notice text is `priority → P4 (lowest); scheduled → 2026-11-02 · Mon · in 91 days; marked task Blocked`. Verify
  the exact notice string by running the test rather than trusting this transcription.

### 5. Version bump

Per repo convention every deployed plugin change bumps the manifest and the README table in the same commit. This adds a
level, so bump the minor version: `plugins/bob-navigation-hotkeys/manifest.json:4` and the version cell of the Bob
Navigation Hotkeys row in `README.md:16` both go from `1.15.3` to `1.16.0`. The README description cell already says
"priority levels that roll scheduled dates" and needs no wording change. Mention the bump in the commit message,
matching `816b6fb`.

### 6. `bob-cli` — `docs/projects.md`

Rewrite the "Priority property and scheduled rolls" section (`:225-268`) for the new numbering. The table becomes:

```markdown
| Picker label | Written field         | Random `scheduled` window |
| ------------ | --------------------- | ------------------------- |
| `P1`         | `[priority:: high]`   | 2-7 days from today       |
| `P2`         | `[priority:: medium]` | 8-30 days from today      |
| `P3`         | `[priority:: low]`    | 31-90 days from today     |
| `P4`         | `[priority:: lowest]` | 91-365 days from today    |
```

Then, in the prose:

- `:244-247`: "A task with no priority field is implicit P1. The picker therefore has no P1 row" becomes "A task with no
  priority field is implicit P0, the highest priority: do it now, with no rolled date. The picker therefore has no P0
  row". Keep the following two sentences about `Ctrl+D` and about a rolled date being an explicit commitment.
- `:249`: "Choosing P2, P3, or P4" becomes "Choosing P1, P2, P3, or P4".
- `:237-242` (the reserved-key rationale) stays as written; `[priority:: P2]` is still a valid example of an invented
  value.

Keep the file's existing ~79-column wrapping and re-wrap any paragraph you touch. No other `bob-cli` file changes: the
README does not mention P-levels, no CLI subcommand or option changes (so `sase/memory/cli_rules.md` is untouched), and
`src/native/task_status_hooks.rs` already accepts `lowest`.

## Verification

From the `bob-plugins` checkout:

```bash
npm test
npm run validate
```

From the `bob-cli` workspace (docs-only change, but keep the tree green):

```bash
just all   # or: cargo fmt --check && cargo clippy --all-targets --all-features && cargo test
```

Deploy, in this order, after the changes are committed in each repo:

```bash
# chezmoi checkout
chezmoi update -a --force
diff ~/.config/bob/config.yml home/dot_config/bob/config.yml   # must be identical

# bob-plugins checkout
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD"
```

If `sync` refuses a file as "dirty in vault; use -F/--force", inspect the vault's git state before forcing: confirm the
deployed file matches its committed baseline (`git show HEAD:plugins/bob-navigation-hotkeys/main.js` inside `~/bob`) so
a force-copy introduces only this change.

Then ask Bryan to reload Obsidian (plugin `main.js` and `styles.css` are only re-read on reload) and hand-check, in a
scratch note rather than a real one:

1. `Ctrl+Shift+P` → `priority` lists four rows: P1/P2/P3/P4 with `2–7d`, `8–30d`, `31–90d`, `91–365d` pills.
2. Choosing P4 writes `[priority:: lowest]` plus a `scheduled` date 91-365 days out, marks the task Blocked, and shows a
   purple-accented notice whose header icon is the empty-bars `signal-zero` glyph and whose pill reads `P4`.
3. Re-opening `Ctrl+Shift+P` on that task shows `current: P4`, and the `scheduled` stage pins a re-rollable `P4 roll`
   suggestion (`Ctrl+R` re-rolls it) inside the same window.
4. An existing `[priority:: high]` task now reports `current: P1` with no file change.

## Risks and edge cases

- **`signal-zero` must exist in Obsidian's bundled Lucide set.** It does in current Lucide, but it cannot be proven from
  this repo. Step 2 of the hand-check is the gate: if the notice header renders an empty icon slot, fall back to
  `signal-low` for index 3 (accepting that P3 and P4 then share a glyph) and update the icon-table test accordingly. Do
  not fall back to `signal`, which reads as the strongest level.
- **Deploy order matters.** The config is what creates the P4 row, and the plugin change is what colors and ices it
  correctly. Shipping only the config gives P4 a full-strength `signal` icon and an accent-colored notice; shipping only
  the plugin changes nothing user-visible. Land both before asking Bryan to reload.
- **Muscle memory.** Every explicit level shifts by one, so an old "P2 = do it this week" reflex now selects a
  medium/8-30-day roll. Nothing in the vault breaks — but call the renumbering out plainly in the commit message and, if
  the notice for the first few uses looks wrong to Bryan, it is the label that moved, not the window.
- **P0 has no picker row by design.** Clearing priority stays `Ctrl+D` on the `priority` row, and it deliberately does
  not remove or re-roll `scheduled`. Do not add a P0 row that writes an empty value; `normalizeBulletPriorityLevel()`
  rejects an empty `value` anyway (`main.js:428-437`).
- **Wide window, same roll code.** `rollPriorityScheduledDate()` is uniform-over-275-days for P4; no weekday weighting,
  so a P4 date can land on a weekend exactly like the other levels. Intentional.
- **Counted sessions.** `N<Ctrl+Shift+P>` with P4 rolls an independent date per task and reports a span
  (`in 91–365 days`). That path is level-agnostic and needs no change, but include one counted P4 assertion only if the
  existing counted tests are cheap to extend — otherwise the level-agnostic coverage already there is sufficient.
