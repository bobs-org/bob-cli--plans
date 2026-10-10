---
tier: tale
size: small
title: Move the Dashboard Review row above Work
goal: Put the Review badge row above Work on the live dashboard so morning review
  is the first navigation row, with Browse still last and every badge's membership
  unchanged.
proposed_by: bbugyi200.apollo.6e
status: done
---

# Move the Dashboard Review row above Work

## Outcome

On the live Dashboard, `~/bob/dash.md`, the badge navigation directly under `# Dash`
should read and tab in this order:

| Row    | Badges, left to right           |
| ------ | ------------------------------- |
| Review | NEW · ROTTEN · CROWDED          |
| Work   | TODAY · PENDING · NEXT · READY  |
| Browse | PROJECTS · REFERENCES · BLOCKED |

Bryan starts the day with the GTD morning review, so Review belongs above Work. Browse
stays last. All ten badges still appear once. Counts, colors, caps, destinations,
tooltips, and accessible names stay as they are.

This is a `tale` of size `small`. One agent can reorder the rows and update the docs
that state that order. There is no new command, option, plugin API, task state, or
memory change, and no reviewer choice.

## What owns the order

The three rows are ordinary DOM children of `.dashboard-nav`. That container is a column
flex with no CSS `order`. `makeRow` appends a row when it is called, so call order is
both the visual order and the keyboard order. Each chip is a `tabindex="0"` link, and
there is no roving tabindex. Later render calls only fill the host they are given
(`reviewBadges`, `workBadges`, or `browseBadges`); they do not move rows.

The live note currently creates the hosts in this order:

```javascript
const workBadges = makeRow("Work", "Work");
const reviewBadges = makeRow("Review", "Review");
const browseBadges = makeRow("Browse", "Browse");
```

The same creation order is what `docs/dashboard.md` calls the "exact reading and
keyboard order." `plan:202610/dashboard_child_pages.md` introduced that order.
`plan:202610/blocked_badge_browse.md` later moved BLOCKED from Review to Browse and kept
Work, then Review, then Browse. Those plans stay historical. This tale changes the row
order they recorded because Bryan asked for Review first.

`bob-plugins` paints chips into the hosts the note creates. It does not choose row
order. The `]s` walk, lane derivation, freshness gating, and the `## Tasks` sections are
separate from this navigation. Accepted decisions that mention `dash.md` govern those
behaviors. Leave the decision records alone.

`sase repo open gh:bobs-org/bob` prepares an external vault clone. That clone is not the
vault Obsidian and `bob vault-sync` use. `dash.md` happened to match `~/bob/dash.md` at
planning time, while the clone's `HEAD` did not match the live vault's `HEAD`. Edit the
live note. Recheck it immediately before editing, because `bob vault-sync` keeps
publishing other vault writes.

## Implementation

1. Inspect `git -C ~/bob status --short` before editing. Keep every unrelated dirty path
   as it is. Read the vault `AGENTS.md` from the opened `gh:bobs-org/bob` checkout
   before editing. The live file to change is `~/bob/dash.md`.

2. In the first `dataviewjs` block, create the row hosts in Review, Work, Browse order:

   ```javascript
   const reviewBadges = makeRow("Review", "Review");
   const workBadges = makeRow("Work", "Work");
   const browseBadges = makeRow("Browse", "Browse");
   ```

   Update the grouped-navigation comment just above the chip arrays so it lists Review
   (`NEW · ROTTEN · CROWDED`), then Work (`TODAY · PENDING · NEXT · READY`), then Browse
   (`PROJECTS · REFERENCES · BLOCKED`). Leave the render sections where they are. Work
   still receives TODAY, then PENDING, NEXT, and READY. Review still receives NEW,
   ROTTEN, and CROWDED. Browse still receives PROJECTS and REFERENCES through
   `dashboardCollections`, then the existing BLOCKED `renderChip` after that loop,
   including when the collection renderer is missing or throws. `renderChip`'s
   missing-host fallback may keep using `reviewBadges`.

3. Update the bob-cli sentences that state this order, and only those sentences:
   - `docs/dashboard.md`: the opening order becomes Review, Work, Browse. Put the Review
     table row above the Work row. In the Work-color section, refer to the Work row on
     `dash.md` so the sentence describes the row that shares the daily `bob-plan`
     colors.
   - `docs/plan.md`: in the `dash.md` Surfaces cell, change the group phrase
     `Work / Review / Browse` to `Review / Work / Browse`. Keep the membership sentences
     (`Work is TODAY · …`, `Review is NEW · …`, `Browse is …`) and the `## Tasks` order
     `TODAY → NEW → PENDING → NEXT → READY`.
   - `docs/freshness.md`: in the `dash.md` surfaces cell, use the same group phrase
     `Review / Work / Browse`. Leave the chip membership list and the task-section order
     as they are.
   - `docs/README.md`: the dashboard guide blurb becomes `Review / Work / Browse`.

   Do not edit the old plan files, decision records, plugin sources, or the `## Tasks`
   queries.

4. Publish the vault edit the way the vault instructs. The vault `AGENTS.md` requires
   `/sase_git_commit` before ending a turn that changed files under `~/bob`. Stage only
   `dash.md`. If `bob vault-sync` has already committed that file alone, do not add an
   empty second commit. Follow `docs/vault-git-sync.md`: no `reset --hard`, no
   force-push, and no `-X ours` or `-X theirs`. Bob-cli doc edits stay in this project's
   normal finalization. Do not commit unrelated vault files.

## Validation and acceptance

This is a presentation reorder. Do not add a test harness and do not run the full Rust
suite.

1. Reread `~/bob/dash.md` and confirm the three `makeRow` calls are Review, Work,
   Browse, and that each badge is still appended to its current host. `## Tasks` remains
   TODAY, NEW, PENDING, NEXT, READY.
2. Extract the first `dataviewjs` body and syntax-check it as an async function, without
   executing it. The block uses top-level `await`, `dv`, `app`, and `window`.
3. Run `git diff --check` in `~/bob` and in the bob-cli checkout. Search both trees for
   `Work, Review, Browse`, `Work / Review / Browse`, and `Work row at the top`. Those
   order claims should be gone. Membership sentences that name Work first inside its own
   row may remain.
4. The delivered dashboard has exactly ten badges in the three rows above. BLOCKED still
   follows PROJECTS and REFERENCES, still links to `blocked.md`, and still uses its
   informational accent. When Obsidian is available, confirm the row order, tab order,
   and a narrow pane in Reading view or Live Preview. When it is not, say that the
   visual check was not run. A source check alone is enough to accept the reorder.
