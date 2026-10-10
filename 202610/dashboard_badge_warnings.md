---
tier: tale
title: Make dashboard badge warnings match their counts
goal:
  Make BLOCKED informational and align NEXT and PENDING warning colors with their
  displayed counts.
size: medium
decisions:
  lane_warning_scope:
    ask:
      Should dashboard NEXT and PENDING turn red only when their displayed section count
      exceeds the cap?
    choices:
      section:
        "Yes: color section/cap; retain whole-lane pressure in tooltips and daily/CLI
        warnings."
      whole_lane:
        "No: show and color whole-lane count/cap; move the section count into the
        tooltip."
    default: section
    why:
      Makes NEXT 10/15 non-red as requested and keeps badge counts aligned with their
      sections.
    answer: section
proposed_by: bbugyi200.apollo.68
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.68](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.68.md)
- **COMMITS:**
  - [5b1e5b5](https://github.com/bobs-org/bob-cli/commit/5b1e5b5a11bdbe7fe3b3b55748962d15b6792f95)
    — docs(dashboard): document section warning policy

# Make dashboard badge warnings trustworthy

## Outcome and scope

Bryan wants the badges at the top of `dash.md` to provide a useful final morning-review
check. BLOCKED has no cap and must always be informational. For capped dashboard lane
badges, the count being colored must be the count shown next to the cap. A count exactly
at the cap is acceptable; only a strict excess is red. Unknown data remains visibly
unavailable, not a healthy zero.

This is one medium tale: bounded changes to the Bob vault dashboard, the shared
bob-ledger-tools dashboard renderer, regression coverage, and contract documentation.
One coding agent can implement and validate it together. No new commands, configuration
keys, task statuses, freshness rules, or memory changes are needed.

## Findings and evidence

- The supplied screenshot is `~/tmp/screenshots/20261010_085532.png`: NEXT reads `10/15`
  in red and BLOCKED reads `378` in red.
- The vault's `dash.md` defines `.task-count-widget .task-count-blocked` with
  `--task-count-accent: var(--task-status-blocked, var(--color-red, ...))`. This is an
  unconditional category color, not an overflow warning. BLOCKED's chip descriptor has
  no cap or over-limit condition.
- In bob-plugins, `plugins/bob-ledger-tools/src/070-ready-and-review.js`,
  `dashboardLaneBudgetFromTasks()` returns section counts excluding TODAY and whole-lane
  counts including TODAY. Its `over` field describes the whole lane.
  `dashboardLaneBadgeModel()` displays `section/cap` but computes its own `over` from
  `lane > cap`.
- `plugins/bob-ledger-tools/src/180-plugin-plan-and-ready.js` uses the model's `over` to
  apply `bob-plan-over` during both initial painting and refresh. `styles.css` makes
  this red. The normal NEXT accent is already non-red.
- The inline fallback in `dash.md` repeats the mismatch: it displays section counts but
  copies `pendingSection.over` / `nextSection.over`, or their whole-lane budget
  equivalents, into its warning state.
- A read-only `bob plan --bob-dir <opened-vault-checkout> --format json` on the
  inspected 2026-10-10 snapshot reported NEXT `17/15`, with seven Next tasks among
  TODAY's eleven tasks. Thus `17 - 7 = 10` explains the screenshot. PENDING was `8/10`,
  with four in TODAY and four in its dashboard section. These are snapshot observations,
  not fixed live acceptance counts.
- bob-cli `docs/plan.md`, “Lanes (NEXT and PENDING),” explicitly documents red
  section/cap badges driven by whole-lane excess. The current
  `scripts/test-ledger-tools-dashboard-parity.cjs` likewise asserts this; all twelve
  existing tests passed during planning. This is an intentional presentation-policy
  correction, not a broken cap configuration or task count.

The applicable decisions were read through SASE memory: sticky lanes, freshness-gated
READY, and the tiered review walk. They continue to govern task state and review
membership. The dashboard is not a new review-completion engine, and this work must not
auto-complete the morning review.

## Settled behavior and reviewer choice

BLOCKED always uses a neutral theme color, including zero, large counts, and unavailable
data. Its count and `blocked` destination remain intact. Its tooltip/accessibility text
should say it is informational and has no limit. Do not change the global blocked-task
color or the shared red overflow class.

> [!decision] lane_warning_scope = section Keep NEXT/PENDING section counts as the
> visible numerators. Color each by `section > cap`, including when the whole lane
> exceeds the cap. The screenshot case becomes a normal-colored `NEXT 10/15`. Keep
> whole-lane counts, TODAY counts, and clearly labeled whole-lane excess in
> tooltips/accessibility text. Existing daily badges, CLI warnings, and navigation
> notices still enforce whole-lane caps; a non-red dashboard does not claim those
> independent checks are clear. This is the proposed default matching Bryan's requested
> behavior.

> [!decision] lane_warning_scope = whole_lane Retain the whole-lane warning and make its
> count visible on the badge.

If `whole_lane` is selected, preserve `lane > cap` coloring and change the visible
NEXT/PENDING numerator to `lane`. The screenshot case becomes red `NEXT 17/15`, with “10
in this section; 7 in TODAY” in its tooltip. Keep the destination sections exclusive of
TODAY. This alternative preserves the existing whole-lane warning policy while making
its reason visible without hovering. Implement only the selected behavior, not a new
runtime toggle.

## Implementation

1. **Open and inspect the source checkouts.** Use `sase repo open bob-plugins` and
   `sase repo open gh:bobs-org/bob` with task-specific reasons; use only the returned
   paths for repository reads and writes. Read their `AGENTS.md` and inspect each
   checkout's status before editing. Recheck `dash.md` and the current plugin fragments
   because other work can land after planning. The vault instructions' Obsidian Sync
   description is stale: the audited `obsidian.md` reference and bob-cli
   `docs/vault-git-sync.md` describe the current git-based vault sync. Preserve all
   unrelated changes.

2. **Correct the shared dashboard lane presentation.** Edit the source fragment
   `070-ready-and-review.js`, particularly `dashboardLaneBadgeModel()`, to derive the
   display count and warning from the selected scope. Preserve the existing budget API
   shape and whole-lane meaning of `dashboardLaneBudget(...).over`, `nextBudget()`, and
   `pendingBudget()`; do not globally change `laneBudgetFromTasks()`. Keep
   section/lane/TODAY values available to callers. Distinguish section excess from
   whole-lane excess in tooltip and aria text; neither should be described ambiguously
   as simply “over the limit.” Use a single model decision for initial render and live
   refresh. For `whole_lane`, update both the `setReadyAnchorContent()` count passed by
   `paintDashboardLaneElement()` and the visible fraction written by
   `refreshDashboardLaneBadges()` in `180-plugin-plan-and-ready.js`. For `section`,
   those existing count paths can remain unchanged. Keep placeholders, configured caps,
   links, focus/keyboard behavior, widget reuse, and cleanup working as before.

3. **Fix the vault dashboard and its fallback.** Edit `dash.md` in the opened vault
   checkout. Set the BLOCKED accent to a neutral theme token such as `var(--text-muted)`
   and add its no-limit explanation without changing other badges' warning styles.
   Update the PENDING/NEXT inline fallback to use the selected numerator and compare
   that same count with the effective cap. Under the default, do not copy budget `over`
   into section warning state; calculate strict numeric `count > cap` after resolving
   the displayed count. Under `whole_lane`, obtain the visible total from the existing
   whole-lane budget or the legacy Tasks calculation before TODAY exclusion; never label
   a section count as a whole-lane total. Keep current API-cap preference and documented
   default caps. Ensure an unavailable current API result stays `–`; a stale whole-lane
   `over: true` must not make a placeholder red. When using an older/unloaded plugin,
   retain guarded legacy counting and make its numeric comparison truthful. When the
   shared renderer throws or returns no node, the fallback should still agree with the
   selected contract and produce only one badge. Leave task query bodies, row order,
   counts, and navigation targets intact.

4. **Update coverage and documentation.** Adapt the existing dashboard parity tests to
   the selected warning contract. Preserve the independent query/count parity checks and
   the whole-lane API assertions. Add the meaningful boundary and refresh cases below.
   Update bob-cli `docs/dashboard.md` and `docs/plan.md`, and bob-plugins `README.md`,
   to explain BLOCKED's informational status and the selected lane warning policy,
   removing claims that conflict with it. Use the plugin repository's normal
   manifest/version convention for this release. Edit only source fragments, then run
   `npm run build` to regenerate committed `main.js`; never hand-edit generated
   entrypoints. Keep each hand-edited fragment within the repository's 1000-line limit.

## Validation and acceptance

Exercise both lanes using synthetic data, so acceptance does not depend on Bryan's task
counts staying at the screenshot values:

| Case                                                 | Default section behavior                         | Whole-lane alternative |
| ---------------------------------------------------- | ------------------------------------------------ | ---------------------- |
| NEXT section 10, lane 17, TODAY 7, cap 15            | `10/15`, non-red                                 | `17/15`, red           |
| NEXT section 15, lane 16, TODAY 1, cap 15            | `15/15`, non-red                                 | `16/15`, red           |
| NEXT section 16, lane 16, cap 15                     | `16/15`, red                                     | `16/15`, red           |
| NEXT section 15, lane 15, cap 15                     | `15/15`, non-red                                 | `15/15`, non-red       |
| PENDING section 4, lane 11, TODAY 7, cap 10          | `4/10`, non-red                                  | `11/10`, red           |
| PENDING section 10/11, lane equal to section, cap 10 | at cap non-red; above cap red                    | same                   |
| Custom cap, zero count, and unavailable data         | selected count; strict comparison; `–` never red | same                   |
| BLOCKED 0, 378, and a larger count                   | neutral and no cap                               | neutral and no cap     |

- Test the model and actual painted/refreshed anchor classes, visible fractions, title,
  and aria labels. Cross the cap in both directions on a reused widget and confirm the
  warning class is removed again. In the default case, varying TODAY/whole-lane pressure
  without changing the displayed section must not flip its warning. Verify the unchanged
  whole-lane APIs still report excess in the 17/15 and 11/10 examples.
- Exercise the edited Dataview block itself in a small temporary DOM/API harness or an
  available Obsidian session: normal renderer, renderer failure, older API, unloaded
  plugin, and unavailable data. Cover both below-cap and above-cap fallback values and
  BLOCKED's neutral accent. Do not satisfy this check with a separately rewritten model
  or only a CSS string assertion. Preserve Enter/Space/click navigation and the existing
  ten-badge grouping.
- In bob-plugins, run `npm run build:check`, the focused dashboard parity, plan-budget,
  ready-badge, and dashboard-collections test files via `node --test`, and
  `npm run validate`. Run the repository's required full `npm test` once for the final
  source tree. Use `/sase_monitor` if a command needs a background handoff. There are no
  Rust behavior changes requiring new Rust tests.
- Confirm no changes to task lines, cap settings, TODAY membership, READY gating,
  NEW/ROTTEN/CROWDED warnings, or collection semantics. A real red warning must still
  appear where the existing rules require it.

## Deployment and completion

Deploy the built plugin from the opened checkout using
`bob plugins sync --no-pull --repo <opened-plugin-checkout> --plugin bob-ledger-tools`,
first with `--dry-run --format json`, then without `--dry-run`. Inspect copy and skip
results; exit zero alone does not prove deployment. Follow the repository's required
sync workflow without editing the installed plugin copy directly or deploying from a
different checkout.

Include the vault `dash.md` change and plugin changes in their respective SASE
finalization obligations. Publish the vault change through the normal authorized
repository finalization and existing vault git-sync flow; an external checkout change
alone does not update the live dashboard. Do not copy an entire stale dashboard over
concurrent user edits. Follow current vault deployment instructions and preserve
unrelated notes.

After deployment, verify the live dashboard in Live Preview and Reading view, using both
light and dark appearance if an Obsidian session is available. BLOCKED must be neutral
and the chosen NEXT/PENDING fraction and color must agree; confirm tooltips and
navigation. Do not mutate real tasks just to force over-limit scenarios. If GUI
verification or vault propagation cannot be observed in the coding environment, report
that precise limitation and the remaining live check instead of claiming the screenshot
is fixed.

The final implementation report should state the selected policy, the 17-total/7-TODAY
explanation, tests passed, and deployment/visual-verification status. No implementation
or deployment is part of this planning turn.
