---
tier: tale
title: Match the dashboard READY badge to its neighboring badges
goal:
  The shared READY badge uses dashboard label and value styling and retains it through
  live updates.
size: small
proposed_by: bbugyi200.apollo.3x
create_time: 2026-10-01 10:58:00
status: wip
---

# Match the dashboard READY badge to its neighboring badges

## Goal and scope

Make READY in `~/bob/dash.md` use the same badge styling as PENDING, NEXT, BLOCKED,
REVIEW, and TODAY: a small muted label, an accent-colored numeric value, matching
padding, height, baseline, spacing, border, background, and interaction feedback. The
reference is `~/tmp/screenshots/20261001_104952.png`, where READY displays `210/100` in
oversized dark text despite its red over-limit border and background.

This is a focused presentation fix in the linked `bob-plugins` repository. One coding
agent can implement it directly; no epic phases are warranted. Retain the shared READY
renderer used by the dashboard and daily `bob-plan` blocks. No vault note edit is
needed: the existing dashboard call will receive the corrected badge after plugin
deployment and reload.

## Evidence and root cause

- The Tasks DataviewJS block in `~/bob/dash.md` renders neighboring chips with separate
  `.task-count-label` and `.task-count-value` spans. The label uses `var(--text-muted)`,
  `0.78em`, small caps, weight `650`, and `0.045em` letter spacing. The value uses the
  chip accent, `0.9em`, tabular numerals, and weight `750`.
- READY alone normally comes from
  `app.plugins.plugins["bob-ledger-tools"].api.renderReadyBadge(bar, options)`. The
  dashboard intentionally omits it from the generic chip loop and has a correctly styled
  fallback when that API is unavailable.
- In `bob-plugins`, `plugins/bob-ledger-tools/main.js` defines `paintReadyElement()`,
  which creates a `.bob-plan-chip.bob-plan-ready` anchor with a single
  `text: model.text` value. Its CSS styles the entire text uniformly and applies
  `var(--text-normal)` rather than giving the label and value their own styles. Some
  font properties also live on the daily `.bob-plan` wrapper, which the dashboard does
  not use.
- `refreshReadyBadges()` subsequently calls `setText(model.text)` or sets the anchor's
  `textContent`. Merely adding spans at initial render would lose the fix on the first
  live update.
- `readyBadgeModel()` already provides `count`, `cap`, `placeholder`, `over`, text,
  tooltip, and accessibility label. Counting and budget logic need no changes.
  `docs/plan.md` in bob-cli defines the READY contract.
- Baseline verification:
  `node --test <opened bob-plugins path>/scripts/test-ledger-tools-ready-badge.cjs`
  passes all 20 tests.

## Implementation

1. Open the linked source through the audited repository workflow:
   `sase repo open bob-plugins --reason "Fix shared READY badge typography to match the dashboard"`.
   Use only the returned repository path and read its `AGENTS.md` before editing.
   Inspect the current dashboard block and screenshot again to account for intervening
   changes. Never edit the installed plugin copies under `~/bob/.obsidian/plugins/` as
   source files.

2. In `plugins/bob-ledger-tools/main.js`, give the shared READY anchor separate label
   and value spans with stable classes, for example `.bob-plan-ready-label` and
   `.bob-plan-ready-value`. Render label `READY` and value
   `${model.count}/${model.cap}`, or `–` when unavailable, from the existing model
   fields. Keep `readyBadgeModel().text` and all public API signatures compatible. Keep
   the current anchor classes, destination, tooltip, accessibility label, keyboard
   handling, modifier-click behavior, hover preview, and lifecycle ownership.

3. Use one internal content-update routine for initial rendering and
   `refreshReadyBadges()`. Update the child spans and the model-dependent title,
   accessibility label, and state classes without replacing the anchor or flattening its
   content to text. Preserve its event listeners and widget registration. Repeated
   updates must leave exactly one label and one value; transitions between available,
   over-limit, and unavailable states must preserve that structure. The daily renderer
   must continue using the same READY markup as the dashboard.

4. In `plugins/bob-ledger-tools/styles.css`, style those spans to match the dashboard's
   label and value properties listed above. Scope any needed overrides to
   `.bob-plan-ready` so parent small-caps, weight, and letter spacing do not distort the
   value. Give READY the dashboard's explicit interface font and `1.3` line height on
   both hosting surfaces. Match the existing dashboard chip geometry (`0.38em` gap,
   padding `0.28em 0.65em 0.3em 0.6em`, `7px` radius, `1px` border with `2px` accent
   start border, and `14%` accent background) and its hover/focus feedback. Preserve the
   plugin's reduced-motion handling. Use the existing `--bob-plan-accent` variable for
   the value as well as the border and background: blue normally, red only when
   `count > cap`. Keep unavailable styling and theme variables; avoid fixed screenshot
   colors or global changes to other badges.

5. Adapt the DOM stubs and existing element-contract assertions in
   `scripts/test-ledger-tools-ready-badge.cjs` to represent child elements and parent
   relationships. Add focused regression coverage for the live update failure mode
   below. Follow the repository's per-plugin release convention: increment
   bob-ledger-tools' patch version in its manifest and update the matching README
   version entry. Update the READY renderer comment/documentation to describe the
   structured label and value.

## Validation and acceptance

- Exercise initial rendering and live refresh with zero, below-cap, exactly-at-cap,
  over-cap, and unavailable budgets, including transitions back from over-cap and
  unavailable. Assert the displayed value, stable child structure, red-state boundary,
  accessibility text, and tooltip. Ensure refresh retains the same anchor and working
  navigation handlers; repeated refreshes must not add spans or handlers. Preserve
  existing count, cap, Today exclusion, daily/dashboard sharing, and navigation tests.
- Run the focused READY suite first. Then run `npm test` and `npm run validate` against
  the opened bob-plugins repository using `npm --prefix <opened bob-plugins path> ...`.
  Review the final diff for unrelated changes and whitespace errors. Investigate
  failures before attributing them to this change; no bob-cli Rust behavior is being
  edited.
- Deploy only the changed plugin from the opened source path. Preview with
  `bob plugins sync --no-pull --repo <opened bob-plugins path> --plugin bob-ledger-tools --bob-dir ~/bob --dry-run`,
  inspect the diff, then run the same command without `--dry-run`. `--no-pull` and the
  explicit repo path ensure deployment uses the edited checkout. Honor dirty-file skips
  rather than forcing an overwrite. Verify the plugin is reported as synced with a
  read-only plugin listing.
- Reload bob-ledger-tools in Obsidian if UI access is available, then check `dash.md` in
  the view shown in the screenshot and in Reading view. READY should sit between NEXT
  and BLOCKED exactly once, use the same muted small label and accent-colored value as
  its neighbors, and stay visually consistent after a live count/cap refresh. Check
  under-cap blue, over-cap red, focus/hover, and light/dark themes without changing live
  task data or the user's caps just to manufacture test states. Fixtures can cover
  states the live vault does not currently show. Check a daily `bob-plan` block for the
  shared badge's appearance and navigation too. If Obsidian cannot be accessed, report
  that visual verification and any reload remain outstanding; automated checks do not
  establish visual parity.

The task is complete when the corrected plugin is deployed, automated checks pass, and
the available visual checks support matching the dashboard badges. Report any
unavailable UI validation explicitly. Preserve the dashboard fallback and task queries,
the shared live backlog semantics, soft-cap rules, and all unrelated daily chips.
