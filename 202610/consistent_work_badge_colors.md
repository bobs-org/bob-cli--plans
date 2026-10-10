---
tier: tale
title: Consistent daily and dashboard Work badge colors
goal:
  Apply the same six count-to-limit colors to TODAY, PENDING, NEXT, and READY, with a
  TODAY theme-overflow override.
size: medium
proposed_by: bbugyi200.apollo.6c.w1
create_time: 2026-10-10 10:45:25
status: wip
---

# Consistent colors for daily and dashboard Work badges

## Outcome and scope

Give TODAY, PENDING, NEXT, and READY the same count-to-limit color scheme in daily-note
`bob-plan` blocks and the Work row at the top of `dash.md`. This is one bounded change
for one implementation agent: plugin presentation, the dashboard's inline
integration/fallbacks, tests, documentation, and plugin deployment. A medium tale is
appropriate; separate implementation phases are unnecessary.

The requested precedence is the contract. For an available nonnegative integer count `n`
and positive integer limit `L`:

| Condition                | Color  |
| ------------------------ | ------ |
| `n === 0`                | grey   |
| `0 < n < L / 2`          | blue   |
| `L / 2 <= n < 3 * L / 4` | green  |
| `3 * L / 4 <= n < L`     | yellow |
| `n === L`                | orange |
| `n > L`                  | red    |

Use exact comparisons, not rounded displayed percentages. Check zero, strict excess, and
equality before descending through 75% and 50%. Small and odd caps may naturally skip
bands. An unavailable count remains `–` with neutral styling and unavailable
accessibility text; it is never converted into numeric zero.

TODAY uses `budget.links.count / budget.links.cap`, with one higher-priority override:
`budget.themes.count > budget.themes.cap` makes it red, including when the link count is
zero. Themes exactly at their cap do not affect the link-based color. Both existing
fractions remain displayed. Missing ledger data or a missing Pomodoros section remains
unavailable, rather than a synthetic `0/limit`.

The four badges keep their existing count sources:

- PENDING/NEXT: the live dashboard **section** count excluding TODAY, compared with that
  lane's configured cap. Whole-lane pressure remains separately labeled in
  tooltips/accessibility text and daily lint lines.
- READY: the current freshness-gated backlog and `max_ready`, with the existing
  whole-lane breakdown in its tooltip.
- TODAY: the existing distinct counted Task Links and theme counts from the ledger
  budget, including its existing exemptions and deduplication. It is not the number of
  resolved tasks in the dashboard TODAY section.
- Older daily notes retain their own TODAY ledger but use current live
  PENDING/NEXT/READY counts, as they do today.

Keep task membership, limits, ledger parsing, warnings, enforcement, chip text,
navigation, layout, and Review/Browse badge policies intact. Exact-cap orange is a
presentation state, not a new lint or an `over` condition. This does not change
CLI/tmux/Mac Capture colors, task-status colors, per-note Ready chips, or review
semantics. No new CLI options, configuration, or memory edits are needed.

## Repository access and evidence

From the active bob-cli workspace, read applicable instructions and use `/sase_repo` to
open these repositories; use only the returned paths:

```sh
sase repo open bob-plugins -r 'Implement the approved daily and dashboard Work badge color plan'
sase repo open gh:bobs-org/bob -r 'Update the dashboard Work badge integration and fallbacks'
```

Read each returned `AGENTS.md`, inspect status before edits, and preserve unrelated
changes. Use `/sase_memory_read` for `obsidian.md`, `glossary:Task Link`,
`glossary:Pomodoro`, `decisions:today-is-read-from-the-ledger`, and
`decisions:ready-is-freshness-gated` before relying on those contracts. The current
Obsidian memory and bob-cli `docs/vault-git-sync.md` document Git sync; the vault AGENTS
file still mentions the retired Obsidian Sync and is not the sync runbook.

Inspected implementation locations, relative to their own repositories:

| Repository  | Locations and significance                                                                                                                                                                                            |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob-plugins | `plugins/bob-ledger-tools/src/070-ready-and-review.js`: `dashboardLaneBadgeModel`, `readyBadgeModel`, and `setReadyAnchorContent`; the latter rewrites all classes on refresh.                                        |
| bob-plugins | `src/180-plugin-plan-and-ready.js` under that plugin: lane/READY paint and refresh paths, currently driven by `model.over`.                                                                                           |
| bob-plugins | `src/290-plugin-today-and-location.js`: `paintPlanBlock`, currently coloring TODAY from aggregate budget status.                                                                                                      |
| bob-plugins | `src/340-dashboard-views-and-plan-block.js`: `planBlockModel`; `src/050-plan-budget.js`: ledger budget; `src/170-plugin-lifecycle.js`: public API; `src/350-exports.js`: test helpers.                                |
| bob-plugins | `plugins/bob-ledger-tools/styles.css`: badge accents, dashboard value spans, and daily inherited text styles. Existing colors are keyed to lane identity.                                                             |
| Bob vault   | `dash.md`, first `dataviewjs` block: TODAY budget/inline chip; plugin PENDING/NEXT/READY renderers; generic lane and separate READY fallbacks; inline CSS. TODAY currently uses cyan while the daily chip uses green. |
| bob-cli     | `docs/dashboard.md` and `docs/plan.md` explicitly describe the superseded normal-at-cap/no-intermediate-color behavior.                                                                                               |

All plugin `src/` paths above are under `plugins/bob-ledger-tools/`. It uses
`src/fragments.json`; edit source fragments and run `npm run build`, never edit
generated `main.js` directly. Every hand-edited fragment must remain at most 1000 lines.
In particular, `340-dashboard-views-and-plan-block.js` is already 958 lines; put the
policy in a small new pure-helper fragment if needed.

## Implementation

1. **Centralize the presentation policy in bob-ledger-tools.** Add a pure
   `workBadgeTone(count, cap)` returning `grey|blue|green|yellow|orange|red` or an
   unavailable result for invalid/missing input, plus `todayBadgeTone(budget)` applying
   the ledger-availability check and theme-overflow override. Do not coerce `null`,
   strings, NaN, negative counts, or nonpositive caps to zero. Existing cap loaders
   retain their established fallback/default behavior. Export the helpers for tests and
   expose an additive, feature-detectable public
   `api.workBadges = { version: 1, tone, todayTone }` namespace, backed by these same
   functions. Keep the top-level API at v3 and all budget return contracts compatible. A
   shared class-building routine should carry tone alongside the existing strict-excess
   and unavailable markers.

2. **Wire every plugin paint and update path.** Add the tone to lane/READY view models
   using their displayed counts. Apply it in `paintDashboardLaneElement`,
   `refreshDashboardLaneBadges`, `paintReadyElement`, and `setReadyAnchorContent` so a
   refresh cannot strip it or accumulate stale classes. Update daily TODAY rendering
   using `todayBadgeTone`, preferably via its existing view model. Keep `over`,
   `sectionOver`, `laneOver`, lint emission, and aggregate ledger status semantics
   unchanged. Preserve the existing widget lifecycle, stale-paint guards, listeners,
   label/value spans, and keyboard/hover behavior; no new polling mechanism is needed.

3. **Integrate the real dashboard, including compatibility fallbacks.** Retain the
   structured TODAY budget until rendering instead of deriving color from the formatted
   text. Gate its availability on the actual Pomodoros section. Use `api.workBadges`
   when available and a small local equivalent when an older/unloaded plugin lacks it.
   Apply this policy to inline TODAY, the generic PENDING/NEXT fallback chips, and the
   separate READY fallback. Prefer plugin renderers for PENDING/NEXT/READY when the
   Work-color capability is present; with an older plugin lacking it, use the inline
   Work fallbacks even if old renderers exist, so they cannot reintroduce identity-based
   colors on refresh. A present budget API returning unavailable must retain a
   placeholder instead of substituting legacy counts; preserve valid fallback count/cap
   behavior. Keep fallback policy testable against the plugin's policy and preserve one
   badge per Work item, Work order, destinations, tooltips, and other rows.

4. **Use matching color tokens in both CSS owners.** Introduce explicit Work tone
   classes (for example `bob-work-tone-green`) for the plugin chips and inline dashboard
   chips. Use the same mappings on both surfaces: muted grey, `--color-blue`,
   `--color-green`, `--color-yellow`, `--color-orange`, and `--color-red`, with
   identical literal fallbacks. These classes override the old per-lane accent rules and
   any Work excess marker at sufficient specificity. Do not derive these colors from
   `--task-status-*`, which can give different meanings to the same hue. Scope them to
   Work badges so shared `bob-plan-over`/`bob-plan-warn` rules elsewhere retain their
   behavior. The existing accent-driven border/background and dashboard numeric values
   use the tone; preserve readable daily text and the existing label typography. Neutral
   unavailable styling must remain distinct in text/semantics from a successfully
   evaluated grey zero.

5. **Update documentation and release metadata.** Replace the old badge-color
   descriptions in bob-cli `docs/dashboard.md` and the PENDING/NEXT, READY, and surface
   sections of `docs/plan.md` with the threshold table, TODAY override, section-count
   rule, and unavailable behavior. Update bob-plugins `README.md` for the public
   namespace and color behavior; advance only bob-ledger-tools' manifest version and
   matching README version according to repository convention. Keep strict-excess lint
   and soft-limit descriptions accurate.

## Verification and acceptance

Extend the existing Node test harnesses `scripts/test-ledger-tools-plan-budget.cjs`,
`scripts/test-ledger-tools-ready-badge.cjs`, and
`scripts/test-ledger-tools-dashboard-parity.cjs`; a focused Work-color test file may
hold shared vectors. Test behavior at the real model/DOM update boundaries, not just
strings that mirror the helper implementation.

- Boundary vectors with cap 100: `0 grey`, `1/49 blue`, `50/74 green`, `75/99 yellow`,
  `100 orange`, `101 red`. Also cover default caps 10 and 15, small caps 1/2/3, a
  changed cap with unchanged count, and invalid/unavailable inputs. At cap 15, 7 is
  blue, 8 and 11 green, 12 and 14 yellow, 15 orange, and 16 red; there is no percentage
  rounding.
- TODAY: with a valid ledger, themes `3/3` and links `0/10`, `4/10`, `5/10`, `8/10`,
  `10/10`, `11/10` are respectively grey, blue, green, yellow, orange, red. Themes `4/3`
  are red even at `0/10` links. Missing section/API/read data produces `TODAY –`, while
  an existing empty Pomodoros section is grey. Unrelated lint warnings do not select the
  tone.
- PENDING/NEXT color depends on the section: NEXT section `10/15`, whole lane `17/15`,
  TODAY 7 is green; section `15/15` with a larger whole lane is orange. Whole-lane
  tooltip/lint information persists. READY uses its gated count even when the total lane
  is much larger. Cap equality still emits no cap lint.
- For lane and READY DOM renderers, verify first paint and transitions in both
  directions through colors, red-to-zero, unavailable-to-available, and
  available-to-unavailable; exactly one tone remains. Refresh retains spans, event
  handlers, accessibility information, and cleanup. Daily and dashboard models agree for
  the same inputs. Retain historical-daily and stale-paint coverage.
- Exercise the **actual first DataviewJS block from the edited `dash.md`** in an async
  DOM/API stub harness. Cover current API, missing color namespace, missing renderers,
  absent plugin, and renderer failures that enter fallbacks. Verify TODAY override and
  all Work fallback tones; ensure Review/Browse rules, labels, counts, links, and Work
  order remain intact. Supply the opened vault's dashboard path explicitly (for example
  `BOB_DASHBOARD_FILE`) to the harness; do not hardcode a home/workspace path or
  silently omit this integration run. Plugin-only tests should remain runnable without
  the vault checkout.

In the opened bob-plugins repository, run `npm run build`, the focused tests, the
dashboard integration harness against the edited checkout, `npm test`, and
`npm run validate`. Register any new plugin-only test file with `npm test`. The build
checks must validate both generated output and fragment line limits. Use
`git diff --check` in each changed repository. Rust runtime code is outside this change,
so a full Rust suite is not required for documentation-only edits.

Inspect daily and dashboard Work badges in both light and dark Obsidian themes when a
live UI is available, including yellow versus orange, zero grey, theme overflow,
hover/focus, narrow panes, and live count changes. Use temporary test inputs rather than
modifying real tasks to manufacture boundary counts. If no UI is available, report that
limitation explicitly and retain the automated render/CSS coverage; do not claim visual
validation.

## Deployment and completion

After checks, deploy the edited plugin checkout using the required source-repo workflow,
with the explicit path returned by `sase repo open`:

```sh
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools --dry-run
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools
```

Do not edit installed plugin assets directly or silently override dirty deployed files
with `--force`. Include the Bob vault `dash.md` change in the delivery; plugin sync
alone cannot deploy dashboard Markdown. Use the normal SASE repository finalization and
documented vault Git sync path for that change; continue to honor `/sase_repo` access
rules instead of copying into an unrelated checkout. Confirm delivery status separately
from plugin installation, and state any remaining vault-sync or Obsidian-reload
requirement accurately.

The final report should name the changed repositories, tests and deployment results, and
any unverified UI step. Account for all modified repositories in the required SASE final
declaration. This planning turn makes no implementation, vault-content, or deployment
changes before the plan is approved.
