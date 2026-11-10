---
tier: tale
title: Repair notebook UI lifecycle and finish bob-cli-j8.6.1
goal:
  Notebook cards and headers refresh correctly in both Obsidian views, and verified
  repairs close bob-cli-j8.6.1 and parent phase bob-cli-j8.6.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.6.1
status: done
---

- **PARENT:**
  [202611/notebook_catalog_ui.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/notebook_catalog_ui.md)
- **BEAD:**
  [bob-cli-j8.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.6.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-j8.6.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-j8.6.1.land.md)
- **COMMITS:**
  - [e2abff0](https://github.com/bobs-org/bob-cli--plans/commit/e2abff08c97d314b215c515c46e2ae26c9e64c07)
    — chore(plans): mark notebook catalog UI plans done

# Repair notebook UI lifecycle and finish bob-cli-j8.6.1

Finish the bounded defects found while landing **bob-cli-j8.6.1**. This tale's coder
owns the landing through closing that epic and its parent phase **bob-cli-j8.6**. There
is no later land agent for this tale. Do not close the containing epic **bob-cli-j8**,
which has its own waiting land agent and other active phases.

This is one medium implementation: the existing notebook data/model/render fragments and
their tests are already present. Repair their lifecycle and state integration; do not
rebuild the feature or expand notebook capture/migration scope. No new memory decisions
or CLI contracts are needed.

## Binding context and current evidence

Use `/sase_repo` to open **bob-plugins** before reading or modifying it, and read its
AGENTS.md. Paths below are relative to that linked repository unless marked bob-cli.
Edit source fragments, regenerate `main.js`, keep each hand-edited fragment within 1000
lines, and preserve concurrent plugin metadata.

Read with the audited commands:

```bash
sase bead read bob-cli-j8.6.1 -r "Need landing audit, follow-up outcomes, and current readiness"
sase bead read bob-cli-j8.6.1.1 -r "Need data phase notes"
sase bead read bob-cli-j8.6.1.2 -r "Need catalog phase notes"
sase bead read bob-cli-j8.6.1.3 -r "Need header phase notes and triaged proposals"
sase bead read bob-cli-j8.6 -r "Need parent phase scope and final decisions"
sase artifact read plan:202611/notebook_catalog_ui.md "Need approved UI contract"
sase artifact read plan:202611/bob_notebooks.md "Need governing plugins-notebook-catalog contract"
sase artifact read file:explicit:bc690a37fdcaf7a084746ce3 "Need landing reproduction evidence and triage outcomes"
```

The implementation is in plugin commits **db5accf**, **56b3604**, and **32a8545**. At
landing audit, 53 notebook data/catalog/header/create-API tests passed, as did
build:check and validate (9/9 plugins). Those tests missed the live-widget defects
below. Both existing stylesheet-contract tests fail on the literal hex added by 56b3604.
`sase bead epic-symbols bob-cli-j8.6.1` was empty.

The lander reviewed every child note and all concurrent work since epic creation
(2026-11-09T20:15:01Z). In particular, **fe7097f** added `api.notebooks.create` and
Create notebook here during this epic and is already in its implementation base.
Preserve that API and prove it integrates with the shared UI cache. Plugin base
**f06c7e8** subsequently changed only Navigation's merge-copy wording/summary keys;
preserve it. Bob-cli **2327590f** is the audited primary base after fast-forwarding
archive-health changes; notebook schema remains version 1. Earlier concurrent archive
pairing, capture parsing/composition, merge-copy, and capture-tip changes do not need a
new catalog consumer. Recheck later drift before completing.

The parent DECISIONS for capture collapse and narrowly authorized memory changes remain
final and outside this repair. Do not edit memory, bob-cli's Cargo version, or
CHANGELOG.

## 1. Repair mounted headers and view ownership

Work in `241e-plugin-notebook-header-model.js`, `241f-plugin-notebook-header.js`, and
`241g-plugin-notebook-header-live.js`.

- **Keep CodeMirror-owned DOM alive.** `notebookHeaderPaintEditorDom` currently deletes
  every header without `data-cm-widget=true`, while `HeaderWidget.toDOM` never supplies
  that attribute. Mounting the built widget then refreshing removes it (child count 1 to
  0). Make CodeMirror the owner of its header and stop manual cleanup from deleting that
  DOM. Update and unload must leave exactly one header on each eligible surface and none
  in Source mode or after retyping/deletion.
- **Make widget equality reflect rendered content.** The current signature stores only
  crumb and child paths. It compares equal after a child-title change and after toggling
  child chips off, although the DOM should change. Include all rendered labels, targets,
  hue, chips visibility, and relevant ancestry state, or use an equivalent explicit
  update strategy. Preserve focus across a refresh.
- **Keep Reading headers within their source note.** `closest('.markdown-preview-view')`
  makes an embedded notebook paint after the outer note's H1, and processing the outer
  non-notebook removes it. Use the nearest note/embed rendering boundary, retain source
  paths per mounted surface, and avoid registering/repainting a whole-note header
  independently for every section. Support two notebooks embedded in one note and
  simultaneous panes. Clean up owned headers on render-child/plugin unload, without
  removing a sibling surface's header.
- Keep placement below the note's own H1, or at the first content position after
  frontmatter when no H1 exists. Do not cover unrelated text or alter note bytes.
- Complete the approved ancestry behavior using report parent identities for notebooks
  and metadata/link resolution from owner through root. The current model jumps directly
  from root to owner. Preserve intermediate project ancestors, bounded cycle handling,
  honest missing parents, and deterministic child order. Every real crumb, including the
  current notebook, must link to its actual note; only ellipsis/unresolved context is
  inert. Do not turn display labels into paths.

Add meaningful tests to `scripts/test-ledger-tools-notebook-header.cjs`. Its current
Live Preview test only sets the path to null; add a mounted live widget, actual
update/refresh sequence, equality transitions, and embed boundary behavior. Test title
edits, setting and hue changes without document edits, pane reuse, unload, retyping,
deep non-root project ancestry, H1-less notes, and keyboard focus.

## 2. Make catalog states actionable and restore color contracts

Work in 241b/241c/241d catalog fragments and `styles.css`, using the existing shared
241a report cache and single-flight sync.

- Carry `stale`, `unavailable`, fetch error, and sync-refusal state into rendered
  surfaces. Today the paint options discard them, and `notebookCatalogSyncModel` returns
  null both for a failed first fetch and for a stale last-good `current` catalog. Keep
  last-good counts visibly stale, explain unavailable data without inventing zero
  counts, and provide a keyboard-accessible report retry. A report retry must use the
  list API; keep the explicit catalog-writing sync action distinct. Render concise
  damaged/sync-failure feedback and clear it only after authoritative success. Avoid
  retry loops and subprocesses per row/paint.
- Missing catalogs must remain actionable when the host has neither a Notebooks H2 nor
  an H1. Use a rendered-only header/content anchor in both views. Never insert Markdown
  as a UI fallback. Preserve the chips toggle's approved scope.
- `notebookHostHueName` currently returns `resolved.root_hue`. Use the host's effective
  ancestry hue for its catalog, including explicit grey mute. Repro:
  `dev(area) -> root(project, blue) -> sub(project, orange)` returns blue for sub;
  changing sub to grey still returns blue. Reuse existing home color helpers and palette
  tokens, with neutral fallback. Keep root-specific header context truthful.
- Remove `#c98500` from the add-summary fallback; use existing theme/palette tokens. Do
  not weaken the stylesheet tests or add an exception. Cover nested rows with long
  labels, count/age/add-summary together, and 375 px width so the current absolute chips
  and accumulated right padding cannot overlap or overflow.
- Preserve actual wikilinks and editing. Scope rendered actions, focus restoration, and
  cleanup to the right source surface. The add-summary action must focus the opened
  notebook's summary when supported, otherwise give the existing visible Properties
  fallback; it must not focus another pane's summary through a global document search.

Extend `scripts/test-ledger-tools-notebook-catalog.cjs` to exercise first-failure, stale
retained current data, retry/recovery, refusal, H1-less anchors, local-color override
and mute, and both Reading and Live Preview. Preserve existing generation,
single-flight, source/fenced-example, and zero-write tests.

## 3. Finish settings and creation integration

- The asynchronous initial chips-setting load currently refreshes only headers. Apply a
  loaded false value to already-mounted catalogs too. Gate late load/refresh results on
  the live lifecycle epoch. Keep unrelated persisted keys and immediate toggle behavior;
  automatic structural sync remains active when chips are hidden.
- Reuse the public notebook creation API introduced by fe7097f. Add an integration test
  showing successful creation and its create/metadata events update a mounted
  report/header/catalog through the shared cache, without duplicate caches, queued
  self-write loops, or overlapping sync commands. If the existing event path is
  sufficient, retain it; fix only a demonstrated invalidation gap.
- Preserve read-only styling and real-link navigation on hosts where bob cannot spawn.
  Exercise the desktop-unavailable case and make any unavailable live data explicit. Use
  existing metadata for read-only ancestry where needed; do not create a second notebook
  counting/scanning engine.

## 4. Verify, preserve concurrent work, and deploy

Read `docs/develop/validation.md` and `docs/develop/deployment.md` in bob-plugins.
Update the focused tests in the explicit package test list when adding files. Regenerate
bundles with `npm run build`. Update user docs/manifest only where needed for the final
behavior, retaining concurrent descriptions; use the bob-cli docs memory before changing
mounted docs. Regenerate inventory when required.

Run focused notebook/create tests, both repaired stylesheet-contract nodes,
`npm run build:check`, full `npm test`, and `npm run validate`. Run the mounted docs
gate with `BOB_CLI_DIR` set to the actual current primary checkout. Primary file
verification is `just check`; do not run `just check-full`. If checks reveal new
epic-caused issues, finish them before close. Compare unrelated failures against an
unchanged base and retain precise node/results; do not turn them into epic work.

Use `/sase_monitor` for long checks and carry this concrete final closeout in its
continuation. Wait for the monitor staging command to exit and finish that turn via
`/sase_final`; never leave a live subprocess when ending a turn. Verification and
closeout must not depend on this repair's own future commit, push, SHA, or CI run.

Create an isolated real-Obsidian notebook fixture covering nested catalogs, missing
summary, stale/damaged state, children, long text, and embeds. Verify Reading and Live
Preview in light/dark at desktop and 375 px, including native links, source editing,
focus, settings, and a refresh after mounting. Capture actual evidence. The audit host
lacked both an executable `/snap/bin/obsidian` and `xvfb-run`;
`node scripts/obsidian-startup-smoke.cjs --report /tmp/j861-obsidian-smoke.json` exited
2 with an explicit unrun verdict. Use an existing authorized capable host if needed
(read its applicable access/vault memory first). If prerequisites remain unavailable,
record the exact limitation/blocker; do not call an offline harness a real-Obsidian pass
or claim unverified acceptance.

After changing bob-plugins, deploy the edited checkout with
`bob plugins sync --repo "$PWD" --no-pull`, as its AGENTS.md requires. Respect deploy
protection; do not silently force unrelated vault work. Verify managed-file hashes and
run the documented isolated post-sync smoke, reporting its actual verdict. Include
generated bundles and all authored changes in the SASE final declaration.

## 5. Finish this epic and its parent phase in the same completion turn

This is the tale's final step, not a handoff back to this lander. Complete it once the
repaired scope and required evidence are ready, before the final commit declaration. Do
not wait for this tale's commit, push, exact SHA, or CI result.

The lander already completed follow-up triage. Preserve these outcomes in the epic close
note, with attribution to **bob-cli-j8.6.1.3**:

- Proposal #1: no new task; the #c98500 failures were caused by phase .2 and are
  repaired here.
- Proposal #2: existing **bob-cli-fz** corroborated (+1, total +6 at audit), using the
  proposing phase's 90-second motion-test evidence. No new task, no fresh browser
  reproduction claimed by the lander.
- Proposal #3: declined as resolved/non-reproducing. `bob-docs gen --check` passed on
  bob-cli 076fdc48 after intervening glossary generation in 3f8c49d4/076fdc48.
- No new task beads were created. The lander's FOLLOW-UP TRIAGE note records the
  same-type/all-type searches, last-week task sweep, and active-epic causal review.

Re-read every current child note and confirm no new unfinished descendant or linked plan
remains. Review commits that landed since this audit in both repositories and integrate
any actual notebook conflict. Then:

1. Run `sase bead epic-symbols bob-cli-j8.6.1`. For **every** entry, resolve the symbol
   by wiring it, privatizing it, using an appropriate non-test pragma, or deleting it
   under Symvision policy. Re-key a Justfile exemption only if a named, still-open later
   bead genuinely still needs it. No stale entries may remain.
2. Close normally:
   `sase bead close bob-cli-j8.6.1 --note "<actual source/commit audit, repaired behavior, verification and deployment results, integration review, and all follow-up outcomes above>"`.
   If rejected, finish the named blockers and retry. Never force merely to succeed, and
   never use force to advance this successful nested landing. A deliberate
   canceled/superseded outcome would require its actual reason and is not the expected
   path.
3. Run `just symvision` when that recipe is available (check both changed repos); record
   its absence if still unavailable. At audit neither repo had that recipe.
4. Open the **plans** sidecar through `/sase_repo`; the epic's audited PLAN is
   **plan:202611/notebook_catalog_ui.md**, file **202611/notebook_catalog_ui.md** under
   the returned sidecar root (normally
   `sase/repos/plans/202611/notebook_catalog_ui.md`). Read it via `sase artifact read`
   and set only its frontmatter **`status: done`**, preserving other metadata.
5. Run `sase bead read bob-cli-j8.6.1 -r "Need the parent link"`. Its parent is **phase
   bob-cli-j8.6**. Re-read that phase and verify this child has completed the entire
   approved `plugins-notebook-catalog` scope. Retire any entries from
   `sase bead epic-symbols bob-cli-j8.6`, then close **only that parent phase** with
   `sase bead close bob-cli-j8.6 --note "<verified child scope, UI/cache/sync/settings evidence and integration>"`.
   Leave **bob-cli-j8** and its plan open to their already-waiting land agent. If that
   parent has acquired new incomplete/ambiguous work, leave it open, record a blocker
   note, and report the blocker. Never force the nested close.
6. Finish through `/sase_final`, declaring all changed repositories, including the plans
   sidecar. Report actual closure, checks, deployment, and any explicit limitation. Do
   not claim completion while acceptance remains unverified.
