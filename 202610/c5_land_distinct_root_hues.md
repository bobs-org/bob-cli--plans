---
tier: tale
size: medium
title:
  Distinct colours per root parent in the working-time bars, then land epic bob-cli-c5
goal: "Every root parent shown in the Day strip's working-time bar (and the stats dash
  Root parents bar it hands off to) gets its own colour, the dead rollup stub is gone,
  and epic bob-cli-c5 is closed with its plan marked done.

  "
proposed_by: bbugyi200.athena.bob-cli-c5.land
proposed_with: claude/opus
bead: bob-cli-c5
create_time: 2026-10-25 12:34:24
status: wip
---

- **PARENT:**
  [202610/day_strip_above_badges_root_parents.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/day_strip_above_badges_root_parents.md)
- **BEAD:**
  [bob-cli-c5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c5/README.md)

# Plan: distinct root-parent colours, then land epic `bob-cli-c5`

## Context

Epic `bob-cli-c5` (plan `plan:202610/day_strip_above_badges_root_parents.md`) moved the
Day strip above the `bob-plan` badges and grouped its full-width working-time bar by
**root parent**. All five phases (`bob-cli-c5.1`–`.5`) are closed. The land agent
verified them against the source:

- **bob-cli** commits `0cf45b8c`, `5078e8f8`, `24fcc300`, `c2ef18c6`;
- **bob-plugins** commits `998a7f9`, `4150279`;
- the vault heading cleanup: 27 `bob-plan` notes plus the template now read a bare
  `## Pomodoros`;
- the four glossary strands, and the beads `bob-cli-bt` (closed) and `bob-cli-bs`
  (noted);
- the plugin vectors are byte-identical to bob-cli's;
- no `--epic-symbol` entries exist.

The commits that landed since the epic started (ref-doctor, gkeep, capture-complete,
popout mark gates, and Task Card close) do not touch this feature.

Two items remain, and both are epic work.

1. **Bryan's epic note #1.** "make sure that we use colors that are distinct for each
   root parent shown in the working time bar".
   - The shipped contract coloured each root with its area's hue (R8). Same-area roots
     therefore share a colour. On a real day, `bob`, `dev_plex_hack_patch`, and two more
     `dev` roots were all blue, and only `GTD review` was orange.
   - Bryan's note overrides that part of the epic's design. "Colour means area" is
     retired for root bars. Each root parent shown in one bar gets its own colour.
2. **A dead stub.** `pomodoroHomeRollupValue` in bob-plugins
   `plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js` has no callers. The epic
   retired `rollup:` everywhere else.

Triage is already recorded on the epic:

- the `bob-cli-c5.3` follow-up became task `bob-cli-c7`;
- `bob-cli-c6` is a separate task, not caused by the epic.

Do **not** create beads for them, and do not fix them here.

## Repos and rules

- **bob-plugins.** Open it with `sase repo open bob-plugins -r "<reason>"` and read its
  `AGENTS.md`.
  - Edit `plugins/bob-ledger-tools/src/` fragments only, then run `npm run build`.
  - Never hand-edit `main.js`.
  - Keep every hand-edited fragment at or below 1000 lines.
  - Run `bob plugins sync` after the gates pass.
- **bob-cli** (this repo): docs only.
  - Read the `docs.md` reference memory first.
  - The file-change gate is `just check` plus `just docs-check`.
  - Do not run `just check-full`.
  - Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
- No memory edits are needed. The four glossary strands never mention hues.

## The colour rule (build exactly this)

Bob's catalog keeps reporting each home's area hue (R8). Bob's JSON and the `bob stats`
terminal output do not change. The plugin derives one display colour per root in a bar
from those reported hues, as follows.

Add `statsPanelDistinctHues(rows, report)` to `src/239-plugin-stats-panel-models.js`,
next to `statsPanelHomeHue`. Export it in `src/350-exports.js` for tests. It returns a
`Map` from each `row.key` to a hue in `STATS_PANEL_HOME_HUES`, or `"gray"`:

1. **Gray buckets stay gray.** A key starting with `@` for which
   `statsPanelHomeHue(report, key)` returns `"gray"` maps to `"gray"`. That covers
   `@unlinked`, `@unfiled`, and other non-review buckets. `@review:*` buckets are
   coloured rows.
2. **Sort.** Order the coloured rows by `value` descending, then `label` ascending, then
   `key` ascending. Because of this sort, the result does not depend on input order.
3. **Pass 1 (keep the area hue).** In that order, each row whose preferred hue
   (`statsPanelHomeHue(report, key)`) is in `STATS_PANEL_HOME_HUES` and still unclaimed
   takes it. The biggest root of each area therefore keeps the area's colour.
4. **Pass 2 (spread collisions).** In the same order, each row still without a hue takes
   the first unclaimed hue scanning `STATS_PANEL_HOME_HUES` cyclically.
   - The scan starts just after its preferred hue's index, or at index 0 when the row
     has no palette hue.
5. **Fallback.** If all 8 hues are already claimed, the row keeps its preferred hue (or
   `"gray"`). More than 8 coloured roots in one bar cannot all differ; this must never
   throw.

Worked example (the `sameHueRoots` fixture):

- the input is `sase` 75 (blue), `GTD review` 50 (orange), `bob` 45 (blue), `gtd` 15
  (orange), `dev` 10 (blue), and `unlinked` 5;
- the result is sase **blue**, GTD review **orange**, bob **aqua**, gtd **yellow**, dev
  **magenta**, unlinked **gray**.

**Order is unchanged.** Segments still sort with `statsPanelHomeAreaRows` (area
hue-slot, then label, buckets last), so same-area roots stay adjacent and now differ in
colour.

## Steps

1. **Model helper.** Implement `statsPanelDistinctHues` as specified above.
2. **Day strip** (`src/239-plugin-day-strip-model.js`, `dayStripMix`)
   - Build the hue map from `ordered` and set each segment's `hue` from it, for both the
     `root_parent` dimension and the old-bob `area` fallback.
   - The legend dots read the same `segment.hue`, so they always match their segments.
     Change nothing else in the mix.
3. **Stats dash Root parents view** (`src/239-plugin-stats-homes-render.js`)
   - In `statsPanelRootTable`, compute the map from `rootRows` (the span or pinned
     point's roots) and give every root row its mapped hue. Because the strip and the
     dash call `statsPanelHomeRows` for the same point, the hand-off shows the same
     colours as the strip.
   - The root bar renders `row.hue || "gray"`. Today `statsPanelHomeAreaBar` forces any
     `@` key to gray, which greys `GTD review` in the dash while the strip shows it
     coloured. Remove that override; `statsPanelHomeHue` already greys `@unlinked` and
     `@unfiled`.
   - Root rows (dot, label, mini bar) use the mapped hue.
   - Expanded sub-project leaves use their root's mapped hue.
   - Area group header rows keep the area hue.
   - Leave the old-bob `Areas & projects` fallback unchanged, apart from the shared
     area-bar change above.
4. **Dead stub.** Delete `pomodoroHomeRollupValue` from
   `src/142-pomodoro-home-chips.js`. First confirm with a grep over `src/` and
   `scripts/` that nothing references it.
5. **Fixtures and preview** (bob-plugins)
   - Relabel the `sameHueRoots` fixture as "same-area roots get distinct hues" in:
     - `plugins/bob-ledger-tools/scripts/fixtures/stats/day-strip-fixtures.cjs`;
     - the fixtures `README.md`;
     - `scripts/preview-day-strip.template.html`;
     - the case list in `scripts/preview-day-strip.cjs`.
   - Add preview check **(m)**: in every ready cell with ≤ 8 non-gray segments, the
     non-gray bar segments carry pairwise distinct `bob-day-strip-hue-*` classes, and
     each legend dot's hue class equals its segment's.
   - Run `node scripts/preview-day-strip.cjs` and `node scripts/preview-stats-block.cjs`
     with zero failures. Look at the same-hue and many-roots cells in light and dark
     themes, and check that the colours read as distinct and calm.
   - Attach the final same-hue PNGs to the epic with
     `sase artifact create -p <png> -l "<label>" --bead bob-cli-c5`.
6. **Tests** (bob-plugins `scripts/`)
   - Unit tests for `statsPanelDistinctHues`:
     - the worked example above;
     - input-order invariance;
     - two roots from different areas keep their own area hues;
     - a 9-coloured-root input returns 9 entries without throwing;
     - gray buckets stay gray.
   - `test-ledger-tools-day-strip.cjs`:
     - update "time mix reuses Stats area ordering and hues…" so the order is unchanged
       but the hues come from the map;
     - add a painter assertion that `sameHueRoots` bar segments carry distinct hue
       classes and legend dots match them.
   - `test-ledger-tools-stats-panels.cjs`:
     - the root bar gives same-area roots distinct hue classes;
     - `GTD review` is coloured, not gray, in the root bar;
     - root rows and their sub-project leaves share their root's mapped hue;
     - area headers keep the area hue.
     - Update existing assertions only where they encoded the shared hue.
7. **Docs and manifest**
   - bob-plugins: bump `plugins/bob-ledger-tools/manifest.json` from `1.70.0` to
     `1.71.0`. Then run `npm run docs:gen` and `npm run docs:check`.
   - bob-cli `docs/stats.md`, Day strip:
     - Replace "Colour means area; a segment means a root parent, so same-area roots
       share a hue and are told apart by order, gaps, and labels." with the rule above,
       in one or two sentences. Every root parent in the bar gets its own colour: the
       biggest root of each area keeps the reported area hue, and the other same-hue
       roots take the next free palette hue. `unlinked` and `unfiled` stay gray.
     - Adjust "hues all come from the report" to say the strip derives its colours from
       the report's area hues.
   - bob-cli `docs/stats.md`, dash bullet (~line 797): the root bar uses those same
     distinct root colours; root rows and their sub-project leaves match them; area
     headers keep the area hue.
   - bob-cli `docs/pomodoro-homes.md` R2: replace "same-area roots share a hue" with
     "bob reports each root's area hue (R8); bars give every root they show a distinct
     colour (see stats.md)".
   - Run `just docs-check`.
8. **Gates**
   - bob-plugins: `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`,
     and both preview scripts with zero failures. Then run `bob plugins sync`.
   - bob-cli: `just check` and `just docs-check`.

## Final step: close out epic `bob-cli-c5`

Do this in the same turn as the code, after the gates pass. It needs no commit SHA,
push, or CI result from this work.

1. Run `sase bead epic-symbols bob-cli-c5`. It reported none at planning time.
   - For any entry it now lists, resolve the symbol: wire it up, privatize it, add a
     non-test pragma, or delete it per the Symvision epic-whitelist policy.
   - Re-key the Justfile line only if a still-open later bead needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-c5 --note "Land verified: phases c5.1-c5.5 closed and checked against source (bob-cli 0cf45b8c rollup retired, residence_chain, leftover-key test; 5078e8f8 root_parent dimension with H16, catalog, render order, perf within 10%; 24fcc300 docs/stats.md contract; c2ef18c6 four glossary strands; bob-plugins 998a7f9 strip as first bob-plan child with full-width root bar and area fallback; 4150279 Root parents dash, sub-project disclosure, roots hand-off, JS rollup mirror removed, vectors byte-identical). Vault: template plus 27 bob-plan notes read a bare ## Pomodoros. bob-cli-bt closed, bob-cli-bs noted. Post-epic commits (ref-doctor, gkeep, capture-complete, popout mark gates, Task Card close) do not touch the feature; Bob Mac Capture has no rollup references. Lander tale: Bryan's note #1 done - statsPanelDistinctHues gives every root parent in the Day strip bar and the dash root bar its own colour (biggest root keeps the area hue, collisions take the next free palette hue, unlinked/unfiled gray), docs updated, ledger-tools 1.71.0 synced; dead pomodoroHomeRollupValue stub removed. Follow-ups: c5.3 FakeNode assert OOM -> bob-cli-c7 (reproduced); bob-cli-c6 stays a separate pre-existing task. No epic symbols."
   ```

   - Adjust the note only so that it states what actually happened, including any gate
     you could not run.
   - Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/day_strip_above_badges_root_parents.md`. The PLAN line of
   `sase bead read bob-cli-c5 -r "Need the plan path for closeout"` gives its path.
   Change nothing else in that file.
5. The epic has no `parent_bead`, so nothing further is needed.
