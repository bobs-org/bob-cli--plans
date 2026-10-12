---
tier: tale
title: Keep PROJECTS / REFERENCES dash chips purple on first paint
goal:
  The dash Browse-row PROJECTS and REFERENCES chips render with their own purple
  collection class from the very first paint, never the READY blue class, and the first
  paint and the live refresh always produce the same class.
size: small
proposed_by: bbugyi200.apollo.6v
create_time: 2026-10-12 05:08:36
status: wip
---

# Plan: Fix PROJECTS / REFERENCES dash chips painting blue (READY class) on first render

## Problem

On `~/bob/dash.md`, the Browse-row **PROJECTS** and **REFERENCES** chips sometimes
render blue (the READY accent) when the note first loads, then flip to purple (their
intended `--interactive-accent`) a moment later.

## Root cause (confirmed by reading the code)

All code lives in the linked **`bob-plugins`** repo (open it with
`sase repo open bob-plugins -r "<why>"` and work only in the printed path), plugin
`plugins/bob-ledger-tools`. The deployed vault copy is byte-identical to the repo, so
the bug is live.

1. `dash.md` renders these chips through `api.dashboardCollections.renderChip`, which
   reaches `paintDashboardCollectionElement` in `src/200-plugin-dashboard.js`. It
   correctly creates the anchor with class `bob-plan-chip bob-plan-${normalized}`, but
   then fills the label/value spans via the shared READY helper:

   ```js
   setReadyAnchorContent(anchor, {...}, { kind: "ready", label: model.label });
   ```

2. `setReadyAnchorContent` (`src/070-ready-and-review.js`, ~line 521) is not
   content-only: it ends by **overwriting the anchor's whole class attribute**
   (`anchor.setAttribute("class", cls)`) with `bob-plan-chip bob-plan-${kind}...`.
   Because the caller passes `kind: "ready"` — and the helper's kind whitelist only
   accepts `pending | next | crowded | ready`, so even passing `"projects"` would fall
   back to `"ready"` — the first paint becomes `bob-plan-chip bob-plan-ready`.
   `styles.css` maps `.bob-plan-ready` to `--color-blue`, and the
   `.bob-plan-projects-arrow` / value colors inherit that accent, so the whole chip is
   blue. The `bob-plan-projects` / `bob-plan-references` rule (purple
   `--interactive-accent`) never matches.

3. The chip only becomes purple when `refreshDashboardCollectionChips` runs, because
   that path rewrites the class directly to
   `bob-plan-chip bob-plan-${widget.kind}[ bob-plan-unavailable]` without going through
   `setReadyAnchorContent`. That refresh is triggered (150 ms debounce) by metadataCache
   `changed`/`resolved`/`deleted`, vault `create`/`delete`/`rename`/`modify` events, and
   the 60 s interval tick (`src/170-plugin-lifecycle.js` ~lines 420–545).

This explains the "sometimes": every fresh Dataview render of the dash block paints the
chips blue; they stay blue until the next vault/metadata event or the next minute tick,
which in an active vault is often near-instant and in an idle one can take up to 60 s.
Dataview re-renders after edits also re-paint blue after the plugin's faster refresh has
already run, so the blue state can recur. The class overwrite in `setReadyAnchorContent`
predates the collections feature (commit `570f40d`), so the collections chips have had
this bug since they were added (`f292382`); the existing test never asserted the class.

## Fix

Make the first paint produce exactly the same class as the refresh path, using the
established "caller keeps its chip-kind class" pattern already used for
PENDING/NEXT/CROWDED.

1. **`plugins/bob-ledger-tools/src/070-ready-and-review.js` — `setReadyAnchorContent`:**
   add `"projects"` and `"references"` to the `kind` whitelist (the
   `rawKind === "pending" || ...` check). Update the adjacent comment (the "bob-cli-3d
   fix" note) to list the collection kinds too. No other behavior changes: collection
   models carry `count: null`, `cap: null`, `over: false` and no `tone`, so the
   resulting class is `bob-plan-chip bob-plan-<kind>` plus `bob-plan-unavailable` for
   placeholders — identical to what `refreshDashboardCollectionChips` writes.

2. **`plugins/bob-ledger-tools/src/200-plugin-dashboard.js` —
   `paintDashboardCollectionElement`:** pass `{ kind: normalized, label: model.label }`
   instead of `{ kind: "ready", label: model.label }`.

3. **Keep paint and refresh from drifting again:** add a tiny pure helper next to
   `dashboardCollectionChipModel` in `src/340-dashboard-views-and-plan-block.js`, e.g.
   `dashboardCollectionChipClass(kind, model)` returning
   `bob-plan-chip bob-plan-<kind>[ bob-plan-unavailable]`, export it from
   `src/350-exports.js` helpers, and use it for both the `createEl` class in
   `paintDashboardCollectionElement` and the `setAttribute("class", ...)` in
   `refreshDashboardCollectionChips`. (Keep `normalized`/`widget.kind` restricted to
   `projects | references` as today.)

4. **Do not** change `styles.css`, `dash.md`, or any READY/PENDING/NEXT/CROWDED
   behavior. The purple (`--interactive-accent`) accent on
   `.bob-plan-projects, .bob-plan-references` is the intended look.

5. Rebuild the generated entrypoint with `npm run build` (never hand-edit
   `plugins/bob-ledger-tools/main.js`; keep each edited fragment ≤ 1000 lines).

## Tests

In `scripts/test-ledger-tools-dashboard-collections.cjs` (already in the `npm test`
list):

- New test, e.g. "renderChip keeps the collection kind class on first paint and
  refresh": for both `projects` and `references`, render via
  `plugin.dashboardCollectionsRenderChip(host, { kind, sourcePath: "dash.md", component })`
  and assert the anchor's class (the stub records it in `el.attrs.class`; also check
  `el.cls`, which `setReadyAnchorContent` assigns on stubs) is exactly
  `bob-plan-chip bob-plan-<kind>` and contains no `bob-plan-ready`.
- Cover the placeholder case (e.g. `plugin.dashboardCollectionsBase = null` → "base
  definitions loading") and assert `bob-plan-chip bob-plan-<kind> bob-plan-unavailable`.
- Assert parity: after `plugin.refreshDashboardCollectionChips(new Date())`, the class
  is unchanged from the first paint. The existing `stubHost()` leaves `el.parentNode`
  null and refresh skips widgets without a parent that has `createEl`, so set
  `el.parentNode = host` in the stub (or in the new test) so refresh actually visits the
  chip; ensure the existing "renderChip links, labels, and cleans up per component" test
  still passes.
- A direct unit check of the exported helper `setReadyAnchorContent` with
  `{ kind: "projects" }` / `{ kind: "references" }` producing the matching class, and of
  the new `dashboardCollectionChipClass` helper.

## Version and deploy

- Bump `plugins/bob-ledger-tools/manifest.json` patch version (currently `1.48.0` →
  `1.48.1`) per the repo's `fix(...)` convention, and update the Bob Ledger Tools row's
  version column in the repo `README.md` to match (it is currently stale at `1.43.0`).
- Run, from the `bob-plugins` checkout: `npm run build`, then `npm test` and
  `npm run validate`; all must pass.
- Run `bob plugins sync` to deploy to the vault (required after any change in that
  repo).
- Commit the change in the `bob-plugins` repo (e.g.
  `fix(ledger-tools): keep PROJECTS/REFERENCES chip class on first paint (1.48.1)`).

## Verification

- Automated: the new test fails before the fix (class is `bob-plan-chip bob-plan-ready`)
  and passes after.
- Manual (for Bryan after reloading the `bob-ledger-tools` plugin or restarting
  Obsidian): open `dash.md` fresh — PROJECTS and REFERENCES are purple immediately, the
  same as after any later refresh, and the arrow/value colors match; READY stays blue;
  PENDING/NEXT/CROWDED/BLOCKED/STATS are unchanged.
