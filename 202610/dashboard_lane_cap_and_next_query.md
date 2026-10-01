---
tier: tale
title: Restore dashboard lane caps and repair the NEXT Tasks query
goal:
  Show PENDING/NEXT dashboard badges as section/cap again, make the NEXT Tasks block
  parse in Obsidian Tasks 8.4.0, and stop bob-cli's headless Tasks engine from accepting
  query syntax that the real plugin rejects.
size: medium
proposed_by: bbugyi200.apollo.40.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.40.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.40.f0.md)
- **COMMITS:**
  - [5d24c98](https://github.com/bobs-org/bob-cli/commit/5d24c98857b1d16dd1d4161da72272f272ad92bc)
    — fix(tasks): reject native-only status.symbol in user Tasks queries

# Restore dashboard lane caps and repair the NEXT Tasks query

## Outcome and scope

The `202610/dashboard_badge_parity` change (bob-ledger-tools 1.14.0, vault `dash.md`
revision `2357e5eb`, bob-cli `eba9cbc`) introduced two regressions on the Bob dashboard:

1. The PENDING and NEXT badges show only the section count (`NEXT 24`) instead of
   `section/cap` (`NEXT 24/15`).
2. The `### NEXT Tasks` block fails in Obsidian with
   `Tasks query: do not understand query` / `Problem line: "status.symbol is *"`.

Restore the cap fraction on both lane badges while keeping the section count from the
approved parity work as the numerator. Replace the invalid NEXT filter with Tasks 8.4.0
syntax that selects exactly the `[*]` tasks the badge counts. Close the verification gap
that let the invalid line through: bob-cli's native Tasks engine accepts
`status.symbol`, which the real Tasks plugin does not support.

This is a medium tale. One coding agent can make three small, connected changes: the
bob-plugins badge text, the vault `dash.md` query and fallback, and a bob-cli
parser-strictness fix plus documentation. The changes have no independent parallel work
that would justify an epic.

## Diagnosis (verified while planning; nothing was modified)

### Regression 1: the lane badges lost their cap

The cap was removed on purpose by the previous plan. Its step 2 said "the cap fraction
belongs in that explanation", meaning the tooltip. Bryan has now rejected that
presentation. The badge must show `section/cap`. The code removes the cap in two places:

- **bob-plugins `plugins/bob-ledger-tools/main.js`**:
  - `dashboardLaneBadgeModel()` returns `text: \`${label} ${section}\``.
  - `paintDashboardLaneElement()` calls the shared `setReadyAnchorContent()`. That
    helper already writes `count/cap` through `readyBadgeValueText()`. The function then
    overwrites the value span with `` `${model.section}` ``. A comment there says "the
    value span already shows the section count". That overwrite is what removes `/cap`
    on the normal rendering path.
- **Vault `dash.md` fallback `renderChip()`**: this is the older-plugin and
  unavailable-API path. For lanes it uses `isLane ? String(raw) : ...`, so the fallback
  chip also drops the cap.
- **Tests and docs that lock in the behavior**: in bob-plugins
  `scripts/test-ledger-tools-dashboard-parity.cjs`, the test "dashboard tooltips
  distinguish section pressure from whole-lane caps" expects `"PENDING 1"` and
  `"NEXT 11"`. Bob-cli `docs/plan.md` (the "Lanes (NEXT and PENDING)" section) documents
  `PENDING 49` with the cap only in the tooltip. The plugin README table says
  "whole-lane pressure in the tooltip".

The live vault runs bob-ledger-tools **1.14.0**, and live `~/bob/dash.md` is at
`2357e5eb`. Both match the sources above.

### Regression 2: the NEXT Tasks block does not parse

- Vault commit `2357e5eb` changed the NEXT block from `status.name includes Next` to
  `status.symbol is *`. The previous plan asked for an explicit symbol-`*` selector.
- Obsidian Tasks **8.4.0**, installed at
  `.obsidian/plugins/obsidian-tasks-plugin/main.js`, defines only the `status.name` and
  `status.type` status filter fields. It has no `status.symbol` filter. The screenshot
  shows the exact upstream error. Tasks' documented way to match a status symbol is the
  scripting property inside `filter by function`, e.g.
  `filter by function task.status.symbol === "*"`.
- The vault's Tasks status config maps `*` to name `Next`, type `ON_HOLD`. Type
  `ON_HOLD` is shared with `?` Blocked, so `status.type` cannot select NEXT.
- **Why nothing caught it:** bob-cli's native Tasks engine
  (`src/native/dataview/tasks/parse.rs`, `parse_text_filter`) accepts `status.symbol is`
  and `status.symbol is not` as a **non-upstream extension**. Bob-cli's own internal
  lane constants depend on it: `NEXT_QUERY` / `PENDING_QUERY` in
  `src/native/dataview/tasks/mod.rs` use `status.symbol is *` / `status.symbol is /`.
  `docs/plan.md` presents `status.symbol is *` as the NEXT query. Those are the likely
  source of the copied line. Because of the extension, this command reports
  `error: null` for every block, including the broken NEXT block:

  ```bash
  bob query --tasks-note dash.md -f json
  ```

  The same command does report Tasks' error for genuinely unknown lines (for example,
  `status.foo is *` gives `do not understand query` / `Problem line: ...`). Separately,
  the plugin parity harness checks a simulated `querySection()`, not dash.md's actual
  query text.

- `status.symbol` appears in no other vault note's Tasks block, and no bob-plugins query
  uses it. Within bob-cli, only the two internal constants and one parser unit test
  (`"status.symbol is not x"` in `parses_every_filter_family_and_boolean_combinations`)
  use it.
- Both candidate replacement lines were run through bob-cli's native engine and
  currently select the same 6 tasks:

  ```text
  filter by function task.status.symbol === "*"
  status.name includes Next
  ```

## Governing contracts and repositories

Before implementation, read `obsidian.md` with `sase memory read`, plus the decision
records `decisions:task-lanes-are-sticky`, `decisions:today-is-read-from-the-ledger`,
and `decisions:ready-is-freshness-gated`, each with a specific reason. This fix changes
how badges are presented, not lane semantics: sticky lanes, ledger-derived TODAY,
freshness-gated READY, and whole-lane budgets all stay as they are. No durable memory
change is part of this plan.

Use `/sase_repo` to open `bob-plugins` and `gh:bobs-org/bob`, the vault. Use only the
printed checkout paths, and read each repo's AGENTS.md and `git status` before editing.
Where the vault's AGENTS.md conflicts with `obsidian.md` (it still says Obsidian Sync is
active), follow `obsidian.md`'s git-sync instructions. Keep all unrelated user or sync
changes intact.

## Implementation

### 1. bob-plugins: show `section/cap` on the lane badges (bob-ledger-tools 1.14.1)

- In `dashboardLaneBadgeModel()`, set the available text to
  `` `${label} ${section}/${cap}` `` (e.g. `PENDING 49/10`, `NEXT 24/15`). Keep the
  unavailable text `` `${label} –` ``. Unavailable must never render as `0/cap`.
- In `paintDashboardLaneElement()`, delete the value-span overwrite. The shared
  `setReadyAnchorContent()` already writes `readyBadgeValueText({count: section, cap})`,
  which is `section/cap` or `–`. Keep the label-span rewrite (READY → PENDING/NEXT) and
  update its comment. The rendered value span must equal the model text's value part
  across repeated refreshes, and no extra spans may accumulate.
- **Numerator:** the section count stays the numerator (TODAY and `dash.md` excluded),
  as approved in the parity plan. The badge must still match its Tasks section.
- **Over state:** red still uses the **whole lane** (`lane > cap`), consistent with
  `bob plan`, the daily `bob-plan` chips, and navigation notices. In the rare case where
  section ≤ cap < lane, the badge can read, for example, `NEXT 15/15` in red. The
  existing tooltip and aria text already explain this:
  `15 in this section; whole lane 16/15; 1 in TODAY · 1 over the limit`. Keep that
  tooltip and aria wording. Also add the fraction to the aria label so screen readers
  hear the same `n of cap` the badge shows.
- Update the parity suite assertions to `"PENDING 1/10"` and `"NEXT 11/10"`. Add these
  cases:
  - The painted value span reads `section/cap` after the first render and after a
    refresh.
  - An unavailable budget renders `–` with no `/cap`.
  - The section ≤ cap < lane edge case is red and its tooltip names the whole-lane
    excess.
- Bump `plugins/bob-ledger-tools/manifest.json` to 1.14.1 and update the version and
  description in the root README table, replacing "whole-lane pressure in the tooltip"
  with a `section/cap` description. Follow any other local version conventions the
  1.14.0 commit touched.
- Run the four focused ledger-tools suites plus
  `scripts/test-ledger-tools-dashboard-parity.cjs`. Then run `npm test` and
  `npm run validate`.

### 2. Vault `dash.md`: valid NEXT filter and a fallback chip with its cap

- Replace `status.symbol is *` in the `### NEXT Tasks` block with
  `filter by function task.status.symbol === "*"`.
  - This is the upstream Tasks 8.4.0 way to select a status symbol. It matches the
    plugin's `dashboardLaneStatusMatches()` (symbol `*`) and bob-cli's `NEXT_QUERY`
    exactly.
  - It does not depend on status display names. `status.name includes Next` would also
    match a future status whose name merely contains "next", which the badge would not
    count. Restoring the old `status.name includes Next` line is the alternative
    reviewers may prefer: it is equivalent under today's status config.
  - Leave the `isToday` filter and `sort by priority` lines unchanged.
- In the inline `renderChip()` fallback, render lanes as `` `${raw}/${item.cap}` `` when
  `raw` is a number, and `–` otherwise. Keep the existing whole-lane `item.over` and the
  `laneDetail` tooltip and aria text. Do not touch the other chips or READY.
- Make no other dash.md edits. Change only the NEXT block, not the PENDING, READY, NEW,
  or TODAY query blocks.

### 3. bob-cli: reject `status.symbol` on user-authored Tasks queries

Make bob-cli's headless engine report the same error as Obsidian for the line that
broke, so `bob query --tasks-note` can be trusted as a parse check. Keep the internal
lane constants working unchanged.

- Add an explicit parse dialect, or an equivalent narrow flag, to `parse::parse` in
  `src/native/dataview/tasks/parse.rs`. Thread it to `parse_text_filter`, including
  through boolean combinations.
  - **Upstream Tasks dialect:** used by the user-facing entry points `run` (for
    `--tasks` / `--tasks-file`) and `run_note` (for `--tasks-note`) in
    `src/native/dataview/tasks/mod.rs`. Any `status.symbol …` filter fails with the
    existing `do not understand query` / `Problem line: "<line>"` error, at top level or
    inside boolean expressions. This applies to query, global-query, and query-file
    defaults sources alike, because Obsidian rejects all three.
  - **Native dialect:** used only by the internal callers `query_matching_descriptions`
    and the rich-row query in the same file (the `NEXT_QUERY` / `PENDING_QUERY` /
    `READY_QUERY` / `OPEN_QUERY` users). It keeps the extension, so `bob plan` lane
    budgets, freshness, and capture behavior do not change and pay no QuickJS cost.
- Add documentation comments on the extension and on the constants stating that
  `status.symbol is` is native-internal syntax and not valid in Obsidian Tasks blocks.
- **Tests:**
  - Update `parses_every_filter_family_and_boolean_combinations` so `status.symbol` is
    covered under the native dialect only.
  - Add unit tests showing the upstream dialect rejects `status.symbol is *`,
    `status.symbol is not x`, and `(status.symbol is *) OR (done)` with the Tasks error
    text.
  - Add a CLI/integration test in the existing style: a fixture note containing a
    `status.symbol is *` block yields that block's error from `--tasks-note`, while a
    `filter by function task.status.symbol === "*"` block succeeds and selects only
    `[*]` tasks.
  - Keep the existing plan-budget and lane tests green as proof that the internal
    constants are unaffected.
- Run `just all` (fmt, lint, test) in the bob-cli workspace.

### 4. Documentation

In bob-cli `docs/plan.md`, section "Lanes (NEXT and PENDING)":

- Describe the badges as `PENDING 49/10`: the section count over the whole-lane cap, red
  when the whole lane exceeds the cap, with the tooltip breakdown and the section ≤ cap
  < lane note.
- Mark the `status.symbol is *` block as bob-cli's native-internal lane query, not
  Obsidian Tasks syntax.
- State that the dashboard NEXT block uses
  `filter by function task.status.symbol === "*"`.
- Correct the inaccurate sentence "The PENDING query is identical with
  `status.type is IN_PROGRESS`". The constant uses `status.symbol is /`. Describe
  PENDING accurately for both the native constant and the dashboard block.

Elsewhere:

- In `docs/dataview.md`, add one sentence stating that user-supplied Tasks queries
  reject the native-only `status.symbol` filter, as Tasks 8.4.0 does.
- Do not edit accepted decision records.

## Deploy and verify

1. Deploy the plugin from the opened bob-plugins checkout:

   ```bash
   bob plugins sync --repo <opened-checkout> --no-pull --plugin bob-ledger-tools
   ```

   Confirm that `~/bob/.obsidian/plugins/bob-ledger-tools/manifest.json` reads 1.14.1.
   Never hand-patch the deployed `main.js`.

2. Deliver the `dash.md` change through the vault's documented git-sync workflow (per
   `obsidian.md`), keeping unrelated changes intact. Confirm that live `~/bob/dash.md`
   contains the new NEXT line.
3. Using the **rebuilt** bob-cli binary (install it per the repo's normal conventions,
   or invoke the built target directly), run:

   ```bash
   bob query --tasks-note dash.md -f json
   ```

   It must show `error: null` for all five blocks, and the NEXT block's task set must
   equal the plugin's symbol-`*` section (identities as path + line). Also confirm that
   the same binary rejects `bob query --tasks 'status.symbol is *' -o dash.md` with the
   Tasks error. This is a parse and filter check only; headless `bob query` has no live
   ledger-tools API.

4. If the live Obsidian runtime is reachable (see `tailnet.md`), reopen `dash.md` and
   check two things:
   - The NEXT Tasks block renders tasks with no error.
   - Each PENDING/NEXT badge reads `section/cap` and its numerator equals the task count
     shown in its section.

   If the runtime is not reachable, state that limitation explicitly. Do not present
   headless results as proof of UI parity.

5. Follow the SASE finalizer obligations for every modified repository (bob-cli,
   bob-plugins, vault). Do not make ad hoc manual commits.

## Acceptance criteria

- Live dashboard badges read `PENDING n/10` and `NEXT n/15`, with configured caps. The
  numerator equals the count in the corresponding Tasks section, and red still reflects
  the whole lane exceeding its cap. Unavailable data still shows `–`, never `0/cap`.
  READY is unchanged.
- The NEXT Tasks block renders in Obsidian Tasks 8.4.0 without a query error and lists
  exactly the non-TODAY `[*]` tasks that the badge counts.
- `bob query --tasks`, `--tasks-file`, and `--tasks-note` reject `status.symbol` filters
  with Tasks' own error. `bob plan` lane budgets and the other internal native-query
  users are unchanged.
- Focused and full bob-plugins tests, `npm run validate`, and bob-cli `just all` pass.
  The plugin is deployed as 1.14.1, docs match the shipped behavior, and live
  verification is completed or its limitation is recorded.
