---
tier: tale
title: Match dashboard review badges to CROWDED
goal:
  Show NEW and ROTTEN in grey with a zero-count checkmark or red with a positive count,
  matching CROWDED on the dashboard Review row.
size: small
proposed_by: bbugyi200.apollo.6c
create_time: 2026-10-10 10:23:36
status: wip
---

# Match dashboard review badges to CROWDED

## Outcome

On the Review row of Bryan's `~/bob/dash.md`, NEW, ROTTEN, and CROWDED use the same
count-based visual language:

| Available count | Appearance                                                   | Example                                |
| --------------- | ------------------------------------------------------------ | -------------------------------------- |
| 0               | Muted grey accent and value, with a checkmark after the zero | `NEW 0 ✓`, `ROTTEN 0 ✓`, `CROWDED 0 ✓` |
| Greater than 0  | Red accent and value; no empty-state checkmark               | `NEW 3`, `ROTTEN 8`, `CROWDED 2`       |
| Unavailable     | Muted `–`, with no checkmark and no implied success          | `NEW –`                                |

The muted label typography, count formatting, link destinations, external-link arrows,
and Work / Review / Browse grouping remain intact. ROTTEN's color follows its displayed
count even when its oldest task has not reached the escalation threshold, and even when
today's upkeep budget has been met.

This is a `tale`, sized `small`: one coding agent can make a localized dashboard
presentation change and a short documentation update. It needs no phased work, new API,
configuration, CLI option, or durable-memory change.

## Evidence and ownership

Open repositories through `/sase_repo` and use only the paths it returns. Do not reuse
the planning agent's ephemeral checkout path:

```sh
sase repo open gh:bobs-org/bob -r "Implement dashboard review badge colors"
sase repo open bob-plugins -r "Verify the CROWDED badge styling reference"
```

Read each opened repository's `AGENTS.md` and inspect its status before editing. Read
`obsidian.md` through `/sase_memory_read` for vault conventions. Git vault sync is
current; the vault AGENTS description of Obsidian Sync is outdated.

The inspected vault `dash.md` uses the first `dataviewjs` block to build its navigation.
It reads `ledgerApi.freshness.reviewModel()` into `counts.new` and `counts.rotten`;
`newOver` follows `review.new > 0`, but `rottenOver` currently follows
`review.escalated`. The inline `renderChip` renders both NEW and ROTTEN on
`reviewBadges`. Their default CSS accents are yellow and orange respectively, and the
value formatter displays bare zeroes.

CROWDED normally uses `ledgerApi.noteReady.renderCrowdedChip`. In `bob-plugins`, the
reference implementation is:

- `plugins/bob-ledger-tools/src/340-dashboard-views-and-plan-block.js`,
  `noteReadyCrowdedChipModel`: `0 ✓` when empty, positive count otherwise.
- `plugins/bob-ledger-tools/styles.css`, `.bob-plan-crowded` and its `.bob-plan-over`
  rules: `var(--text-muted)` when calm and
  `var(--task-status-blocked, var(--color-red))` when positive.

The plugin also exposes `renderReviewChip`, but the inspected dashboard does not call
it. Changing that shared renderer alone would not fix these badges. Keep the dashboard's
existing inline rendering path; this task requires no plugin source changes, build, or
deployment. If the implementing checkout has changed, retrace the actual call sites
before editing.

## Implementation

1. In the vault's `dash.md`, keep using the shared review model for the counts. Change
   `rottenOver` to `review.rotten > 0`, retain `newOver`'s existing positive count
   check, and update the nearby comment to describe the new color rule. Keep the
   `available` guard and `–` defaults. Do not change the review model, freshness
   classification, escalation metadata, task queries, or budget logic.

2. In the inline stylesheet, give `.task-count-new` and `.task-count-rotten` the CROWDED
   muted accent, `var(--text-muted)`. Give the generic `.task-count-crowded` fallback
   that same muted default: it currently displays an unavailable dash with a red accent
   when the plugin renderer is absent. Ensure positive Review badges override the muted
   defaults with CROWDED's red theme-variable chain. Scope any added override to these
   Review badge classes so Work and Browse badges keep their existing colors. Check
   selector order and specificity for the accent, value, border, background, hover, and
   focus.

3. In `renderChip`, keep the raw count separate from the displayed value. For Review
   badge keys `new`, `rotten`, and `crowded`, display `0 ✓` only when the raw count is
   the number zero. Positive values stay numeric; unavailable `–` stays `–`. The generic
   CROWDED fallback currently has no numeric count source; preserve that behavior rather
   than inventing a zero or adding a new lookup. Retain all existing link and keyboard
   handlers. Use the raw count in any accessibility count wording, and give unavailable
   Review badges explicit unavailable wording rather than describing `–` as a task
   count.

4. Add a concise Review-badge paragraph to bob-cli's `docs/dashboard.md` under the badge
   warning guidance. Document grey plus `0 ✓`, red for every positive count, and
   unavailable `–`. Distinguish the Review count rule from the PENDING/NEXT cap rule
   already documented there.

## Verification and acceptance

This is a reversible inline presentation change; use syntax, diff, and visual checks
without adding a permanent test harness or changing real task data to manufacture badge
counts.

- Extract the edited first DataviewJS block into a temporary async-function wrapper and
  run `node --check` to catch JavaScript/template-string mistakes.
- Inspect the source against the three-state table above for each badge. In particular,
  verify `rotten = 1, escalated = false` is red, numeric zero alone gets the checkmark,
  and unavailable models cannot enter the zero branch.
- Verify the CSS cascade gives NEW/ROTTEN the same muted/red variables as CROWDED,
  including their numeric values and hover/focus accents. Confirm no yellow/orange
  default remains for those Review badges. The existing plugin CROWDED renderer remains
  the styling reference.
- If an Obsidian preview is available, reopen/refresh Dashboard in light and dark themes
  and a narrow pane. Check the visible Review row, click targets, Enter/Space
  navigation, and normal rerendering after counts change. Record which zero, positive,
  and unavailable states were actually observed; do not claim visual verification from a
  syntax check.
- Run `git diff --check` in each edited repository and review the scoped diffs. Task
  sections, Work/Browse styling, and plugin sources should have no changes.

## Delivery

The vault source change belongs to the opened `gh:bobs-org/bob` repository; the
documentation change belongs to bob-cli. Preserve unrelated work and declare both
changed repositories through `/sase_final` at implementation completion so SASE retains
the edits. Use the established repository publication and vault Git-sync workflow for
delivery to `~/bob/dash.md`; a change in an isolated checkout alone is not evidence that
Bryan's live dashboard has updated. Report the actual publication/sync and
visual-verification state explicitly. Never copy an entire checkout over the live vault
to deliver this one-note change.

Rollback is the inverse of the scoped dashboard presentation and documentation diffs,
preserving any intervening user edits. No task-data migration is involved.
