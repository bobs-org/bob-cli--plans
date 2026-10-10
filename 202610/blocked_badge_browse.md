---
tier: tale
title: Move the dashboard BLOCKED badge to Browse
goal:
  Place BLOCKED after PROJECTS and REFERENCES in the live dashboard Browse row, with
  matching styling and preserved count and navigation behavior.
size: small
proposed_by: bbugyi200.apollo.6b
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.6b](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.6b.md)
- **COMMITS:**
  - [6c1969d](https://github.com/bobs-org/bob-cli/commit/6c1969de4744ccdf9e4b974bc088d32aecc6e364)
    — docs(dashboard): move BLOCKED into the Browse row

# Move the dashboard BLOCKED badge to Browse

## Outcome

Move the existing BLOCKED badge in Bryan's live `~/bob/dash.md` from Review to Browse
and match the appearance of the PROJECTS and REFERENCES badges. The final reading and
keyboard order is:

| Row    | Badges                          |
| ------ | ------------------------------- |
| Work   | TODAY · PENDING · NEXT · READY  |
| Review | NEW · ROTTEN · CROWDED          |
| Browse | PROJECTS · REFERENCES · BLOCKED |

Appending BLOCKED preserves the existing order of the two Browse links. BLOCKED remains
an informational task count with no cap, links to `blocked.md`, and keeps its existing
count calculation and unavailable `–` state.

This is a `tale` with implementation size `small`: one agent can make the bounded
dashboard presentation change and update its documentation. No new CLI options, plugin
API, data model, task semantics, or memory changes are needed.

## Verified source and access

Planning inspected the vault through `/sase_repo` using
`sase repo open gh:bobs-org/bob`, and inspected the linked `bob-plugins` repository
using `sase repo open bob-plugins`. Open these through that skill when needed and use
their returned paths; do not assume a previous agent's checkout location. Read the
opened repository's `AGENTS.md` and inspect its status before editing. The inspected
vault and plugin checkouts were clean.

The inspected vault snapshot has the following relevant code in the first `dataviewjs`
block of `dash.md`:

- `counts.blocked` is calculated from the Tasks plugin, retaining the existing
  visibility and blocked-task predicates; its initial unavailable value is `–`.
- `chipsAfterReady` contains a BLOCKED descriptor with `target: "blocked"`,
  `external: true`, and the detail `Informational only; there is no limit.`
- The Review rendering section finds `blockedItem` and calls
  `renderChip(blockedItem, reviewBadges)` between ROTTEN and CROWDED.
- The Browse rendering loop renders PROJECTS and REFERENCES through
  `ledgerApi.dashboardCollections.renderChip`, with inline unavailable fallbacks.
- `.task-count-widget .task-count-blocked` explicitly sets
  `--task-count-accent: var(--text-muted, var(--interactive-accent))`.
- `renderChip` already creates separate label, value, and decorative `↗` spans, a
  focusable link, click/Enter/Space activation, modifier-click navigation, and an
  Obsidian hover preview.

The visual reference is `plugins/bob-ledger-tools/styles.css` in `bob-plugins`:
`.bob-plan-chip`, `.bob-plan-projects`, `.bob-plan-references`, their label/value rules,
and their arrow rules. The collection renderer is in
`plugins/bob-ledger-tools/src/200-plugin-dashboard.js`. Its collection namespace only
supports projects and references; moving BLOCKED does not require extending that
namespace. The existing inline fallback chips already use the same general badge shape
and typography.

Recheck the current dashboard before implementation because the vault continues to
change between planning and approval. The requested result is the live vault dashboard,
not merely a modified isolated checkout. Follow the repository finalization and
documented vault delivery/sync workflow in `docs/vault-git-sync.md`, preserve unrelated
user changes, and verify the delivered note. Apply the narrow dashboard change rather
than replacing the whole live note with a possibly older snapshot.

## Implementation

1. **Move the existing rendering call.** Remove the BLOCKED rendering call from the
   Review section. Render its existing descriptor once into `browseBadges` after the
   PROJECTS/REFERENCES loop. Keep that call outside the collection API success/fallback
   branches so BLOCKED remains present with an older, missing, or throwing ledger
   plugin. Update the grouping comments to match the final row order. Keep Work and the
   task-query sections intact.

2. **Match the Browse presentation within `dash.md`.** Keep using `renderChip` and its
   existing interactions. Change BLOCKED's accent to `var(--interactive-accent)`,
   matching both the normal collection badges and their inline fallbacks. Retain the
   shared rounded shape, padding, border, tinted background, muted small-caps label, and
   accent-colored tabular count. Check the small remaining differences between the two
   renderer families: the generic inline arrow currently uses `0.78em`/weight `700`,
   whereas the normal collection arrow inherits the badge text size/weight; inline chips
   also lift on hover, whereas normal collection chips do not. Add narrowly scoped
   BLOCKED rules as needed to match the normal Browse arrows and hover behavior,
   including the inherited badge-size arrow and weight `650`. Preserve visible keyboard
   focus, wrapping, and reduced-motion behavior. Do not apply a warning class or
   count-based color to BLOCKED. Keep the existing descriptor, count logic, navigation
   target, tooltip semantics, and accessible label. Avoid changing shared plugin CSS or
   restyling the other dashboard rows.

3. **Update the two documentation contracts in bob-cli.** In `docs/dashboard.md`, change
   the row table, explain Browse as navigation to supporting task/collection pages, and
   describe BLOCKED as cap-free with the same informational accent as its Browse
   neighbors. In `docs/plan.md`, update the `dash.md` row of the Surfaces table with the
   new grouping and appearance; clarify that `dashboardCollections` still supplies only
   PROJECTS/REFERENCES. Preserve the documented count scopes. No plugin code or
   deployment is expected.

## Validation and acceptance

This is a reversible presentation edit; do not add a permanent test harness or run the
full Rust suite. Use focused checks:

1. Inspect the final diff for one BLOCKED rendering call into Browse, none into Review,
   accurate comments/docs, and no changes to the count predicates, badge destination,
   task queries, or other badge behavior. Run `git diff --check` in each changed
   repository. Parse the extracted `dataviewjs` body as an async JavaScript function
   without executing it to catch syntax errors, since the block contains top-level
   `await`.
2. Render the dashboard in Obsidian, when available, and confirm exactly ten badges with
   the final row order above. Compare BLOCKED beside PROJECTS and REFERENCES in
   light/dark themes and a narrow pane: accent, label/value typography, arrow,
   border/background, spacing, hover, focus, and wrapping. The label and count must
   remain legible and the row must not overflow.
3. Verify that click, Enter, Space, and Ctrl/Cmd-click still open Blocked Tasks, and the
   hover preview and accessible label still identify that destination. Confirm the
   displayed count agrees with the unchanged calculation. Zero and large counts must
   retain the informational accent; unavailable data must retain `–` without a cap or
   warning color. Exercise unavailable ledger/Tasks inputs in an isolated render if
   practical, without changing the user's tasks or persistent plugin settings solely for
   validation.
4. Verify that the delivered `~/bob/dash.md` contains the approved change. If Obsidian
   rendering cannot be exercised from the implementation environment, report that
   specific visual-check limitation instead of claiming it passed.

Completion means the live dashboard places BLOCKED last in Browse, styled like its
neighbors, with its previous behavior preserved and both docs consistent.
