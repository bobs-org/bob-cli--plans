---
tier: tale
title: Links stage ranks Blocked matches by score
goal:
  Ctrl+Shift+L and Ctrl+Shift+Alt+L render through the relation planner, so Blocked
  tasks rank by match score.
size: small
decisions:
  blocked_ties:
    ask:
      Should equal-score Blocked rows tie with Ready rows in the Links stages instead of
      sorting after them?
    choices:
      peer:
        Ready and Blocked share one tie lane; In Progress and Next still lead every tie
      lane:
        Equal-score Blocked rows keep sorting after Ready rows, as the shared order does
        today
    default: peer
    why: Relations never block, so Blocked status says nothing about link relevance
    answer: peer
proposed_by: bbugyi200.athena.156
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 09:40:24
status: wip
---

# Plan: Links stage stops sinking Blocked matches (Ctrl+Shift+L / Ctrl+Shift+Alt+L)

## Verdict

The suspicion is **confirmed** for the Links stage (`Ctrl+Shift+L`, also reached via the
Task Card's `l`) and the Linked-from stage (`Ctrl+Shift+Alt+L`, `⇧L`). The search ranker
is not the problem. The stage renders through the wrong planner.

### Root cause

`bob-plugins/plugins/bob-navigation-hotkeys/src/410-picker-filter-and-render.js`,
`getFilteredItems()`: every keystroke and every render calls `renderResults()` →
`getFilteredItems()`. For a vault-wide local-task stage that method dispatches only two
ways: the Blocks stage goes to `planBlocksStageItemsFromVault`, and **everything else**
goes to `planDependencyStageView`, the Depends-on planner. It has no relation-stage
branch, so the Links and Linked-from stages render through the Depends-on planner.
`planDependencyStageView` (`src/300-dependency-stage.js`) intentionally moves every
`candidate.blocked` (`[?]`) row into a trailing **BLOCKED** section beneath all RESULTS,
whatever its match score (`docs/task-dependencies.md` §6.3). That is correct for the
Depends-on stage. It is wrong for relations, which never block.

The relation-aware planner (`planRelationStageItemsFromVault` → `planRelationStageView`
/ `planLinkedFromStageView` in `src/304-relation-stage-view.js`) is built and seeded
correctly by `showRelationStage`, `relationStageListOptions`,
`refreshRelationStageTodayLinks`, and `applyVaultStageItems` (which already dispatches
three ways). Its rows land in `this.items`, which `getFilteredItems()` then ignores, so
they never reach the screen. The existing relation tests assert on `picker.items` or
call the pure planners directly, so none of them exercise the render path. That is why
this slipped through.

### Evidence (live vault, read-only reproduction)

A throwaway script loaded the built `main.js` helpers, built the relation candidate pool
from `~/bob` (502 open Blocked tasks against roughly 100 Ready, Next, and In Progress
tasks), and compared `getFilteredItems()` with `planRelationStageItemsFromVault()` on a
picker with `relationStage = true`:

| query         | rendered today (`getFilteredItems`)                                                                                                              | relation planner (intended)                                        |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| `xprompt tag` | first Blocked row at index 43; sections RESULTS/BLOCKED; closed `done/` rows fill the top                                                        | Blocked "Make it possible to invoke an xprompt by tag!" at index 0 |
| `artifact`    | **no Blocked row at all** in the 61 rendered rows. All of them sit behind closed and archived matches and the 60-row "type to search N more" cap | Blocked rows at index 0–8                                          |
| _(empty)_     | same-note Blocked rows pushed to a BLOCKED section at index 32                                                                                   | same-note Blocked rows at index 7 onward                           |

The misrouting has more visible effects, all fixed by the same change. Closed and
`done/` rows mix into RESULTS instead of the designed CLOSED section. The 🍅 today and ⛓
prerequisite badges never render. Archived rows lose their `relationKind` styling. The
owner row is not disabled as "this task". The Linked-from stage skips
`planLinkedFromStageView`'s previous-daily and archive exclusions.

A smaller, secondary demotion remains even inside the correct planner.
`compareRelationStageCanonical` breaks exact-score ties through
`compareDependencyStageCanonical`, whose lane order puts `?` after Ready (lane 3). The
`blocked_ties` decision covers it.

## Changes (all in the `bob-plugins` linked repo unless noted)

Open `bob-plugins` with `/sase_repo` before reading or editing it. Read its `AGENTS.md`.
Edit only `src/` fragments; regenerate `main.js` with `npm run build`; keep every
hand-edited fragment at or below 1000 lines.

### 1. One stage-planner dispatch shared by render and refresh

- In `plugins/bob-navigation-hotkeys/src/400-picker-local-task.js`, add one picker
  method, for example `planVaultStageItemsForQuery(stage, query)`. The name must be
  unique across every picker mixin, because `installBulletPropertyPickerMixins` throws
  on duplicate method names. The method holds the three-way dispatch that
  `applyVaultStageItems` already does:
  - `this.blocksStage` → `planBlocksStageItemsFromVault(stage, query)`
  - `this.relationStage` → `planRelationStageItemsFromVault(stage, query)` (this covers
    both directions, because that function routes on `stage.direction`)
  - otherwise →
    `planDependencyStageView({ current, candidates, query, dependent, edges, linkedKeys, sourceValueSets })`
- Make `applyVaultStageItems` call it after its existing `vaultStage` reassignment
  (blocks decoration stays where it is).
- In `src/410-picker-filter-and-render.js` `getFilteredItems()`, replace the two-way
  branch with `this.planVaultStageItemsForQuery(this.vaultStage, this.getQuery())`. Keep
  the existing `try … catch → super.getFilteredItems()` fallback, and update the stale
  comment so it names all three stages.
- Do not change `planDependencyStageView`, `planBlocksStageItemsFromVault`, the
  Depends-on BLOCKED section, or `dependencyStageRank`.

### 2. Tie order for relation stages (decision `blocked_ties`)

> [!decision] blocked_ties = peer In `src/304-relation-stage-view.js`, make both
> relation planners break ties with a relation-specific canonical order in which `?`
> shares Ready's lane. Otherwise it is identical to `compareDependencyStageCanonical`:
> owner note in document order first, then visible before `#hide`, then In Progress,
> Next, Ready/Blocked together, then other statuses, then path, then line. Either add an
> optional fourth `options` argument to `compareDependencyStageCanonical`, such as
> `{ blockedWithReady: true }`, that only the relation callers pass, or add a small
> relation-local comparator. Do not change the Depends-on default, which
> `docs/task-dependencies.md` §6.3 and the DK vectors pin. Apply it in
> `compareRelationStageCanonical`, which keeps its archived and closed sinking, and in
> both `planLinkedFromStageView` sort calls.

> [!decision] blocked_ties = lane Leave the tie comparators unchanged. Section 1 alone
> removes the BLOCKED section and the score-blind sinking. Equal-score Blocked rows keep
> sorting after Ready rows, as the shared canonical order does today.

### 3. Regression tests

Add `scripts/test-navigation-relation-stage-render.cjs` and append it to the `test`
script list in `package.json`. The existing
`scripts/test-navigation-relation-links-stage.cjs` is already about 1900 lines. Reuse
its fixture pattern: `makePlugin(notes)`, `TestEditor`, and `makeFlipPicker`, around
lines 1685–1728, build a real `BulletPropertyPickerModal` prototype picker with
`relationStage = true` and an `inputEl.value` query. Copy the minimal helpers you need;
do not import from another test file. Assert on `picker.getFilteredItems()`, the render
path, after `picker.showRelationStage(dir)`:

1. **Score wins over status (the reported bug).** Use a Blocked `[?]` task in another
   note whose text starts with the query word (field prefix) and Ready tasks that match
   it only as a later word or a substring. For query `widget`, the Blocked row is the
   first RESULTS row, and no row has `stageSection === "blocked"`.
2. **Blocked rows are not hidden behind the cap.** With 70 Ready or closed matches of
   equal or lower score and one strong Blocked match, the Blocked row is among the
   rendered rows, not folded into `type to search N more`.
3. **Render matches the relation planner in both directions.** For `"to"` and `"from"`,
   with an empty query and a typed one, `getFilteredItems()` deep-equals
   `planRelationStageItemsFromVault(picker.vaultStage, query)`. Closed and `done/` rows
   appear only in the CLOSED section, after RESULTS.
4. **Other stages unchanged.** A Depends-on stage picker (`relationStage` false,
   `blocksStage` false) still renders Blocked candidates in the BLOCKED section through
   `getFilteredItems()`. A Blocks stage picker still renders
   `planBlocksStageItemsFromVault` output.
   `scripts/navigation-dependencies-stage-harness.cjs` shows the Depends-on fixture
   shape.
5. **Tie order** (decision-dependent). Use equal-score candidates in two other notes: In
   Progress `[/]`, Next `[*]`, Ready `[ ]`, and Blocked `[?]`. With `peer`, In Progress
   and Next lead, then Ready and Blocked interleave by path, then line, and a Blocked
   row on an earlier path precedes a Ready row on a later path. With `lane`, Ready
   precedes Blocked. In both cases, run the Depends-on canonical order on the same
   candidates and assert it is unchanged.

Existing suites must still pass unchanged, especially
`test-navigation-relation-links-stage.cjs`, `test-relation-index-linked-from-stage.cjs`,
`test-navigation-dependencies-stage-*.cjs`, and `test-navigation-blocks-stage-view.cjs`.
If an existing test asserted the buggy render shape, fix the test to the documented
relation contract and say so in the commit message.

### 4. Docs

- `bob-plugins/docs/task-relations.md`, in the Links stage paragraph that describes
  CURRENT, RESULTS, and CLOSED: add one sentence saying RESULTS rank open tasks of every
  status by match score, so Blocked `[?]` tasks are never set apart in their own
  section. With `peer`, also say that exact-score ties put In Progress and Next first,
  then Ready and Blocked together.
- bob-cli `docs/task-relations.md` §15 LG table: add
  `LG17 | Links / Linked-from stage query that a Blocked task matches best | the Blocked row ranks by score inside RESULTS; the stage never shows a BLOCKED section`.
  With `peer`, append
  `; equal scores order In Progress, Next, then Ready and Blocked together`. This repo
  is the primary workspace checkout. Do not edit `Cargo.toml` or `CHANGELOG.md`.

### 5. Build, verify, deploy

In `bob-plugins`:

- `npm run build`, then `npm test` (it runs `build:check` and the docs inventory check
  first), then `npm run docs:check`.
- `bob plugins sync` to deploy to the vault. The repo's `AGENTS.md` requires it after
  any change.

In bob-cli, run `just docs-check`.

Repeat the live-vault check read-only, with a scratch Node script outside both repos
that never writes to `~/bob`. Load `plugins/bob-navigation-hotkeys/main.js` with stubs
for `obsidian` and `@codemirror/view`, the same stubs the `scripts/test-*.cjs` files
use. Build candidates with `helpers.createDependencySnapshotMap` and
`helpers.collectVaultRelationCandidatesFromSnapshots`. Wrap them with
`helpers.decorateRelationVaultStage`. Set the result as the `vaultStage` of an
`Object.create(helpers.BulletPropertyPickerModal.prototype)` picker with
`stage = "value"`, `relationStage = true`, and
`selectedPropertyItem.property.values = "local_task_id"`. Confirm that for `artifact`
and `xprompt tag`, `getFilteredItems()` now equals
`planRelationStageItemsFromVault(...)`, a Blocked row is first, and no `blocked` section
exists.

## Out of scope

- The Depends-on stage (`Ctrl+Shift+B`, Task Card `b`) and the Blocks stage
  (`Ctrl+Shift+Alt+B`, `⇧B`) keep their documented trailing BLOCKED section with 🔒
  badges (`docs/task-dependencies.md` §6.3 and §15). That is designed behavior, not this
  bug. Changing it is a separate design decision.
- The shared ranker tiers (`dependencyStageRank`, `capture_link_tasks.rs::rank`) and the
  DK vectors.
- Pomodoro Task Link pickers, which never offer Blocked tasks by design, and Bob Mac
  Capture link search.
