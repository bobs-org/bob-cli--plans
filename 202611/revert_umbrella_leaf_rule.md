---
tier: tale
title: "Revert the k7 umbrella rule: a leaf project is again a sub-project"
goal:
  Keep-moving (offers and ⏭ marks) returns to the pre-k7 leaf rule — a leaf project is
  a recognised project that is not its own root parent — across Rust, docs, the glossary
  strand, the cycler, and the ledger-tools mirror, while keeping the whole keep-moving
  marks feature and the `_generated/` ambiguity fix. The corrected
  `job_email_recruiters` (now a child of `job_2026`) stays the screenshot leaf.
size: medium
decisions:
  leaf_rule:
    ask:
      Should keep-moving return to the pre-k7 leaf rule (a leaf project is not its own
      root parent)?
    choices:
      revert:
        Top-level, root:true, orphan, and unresolved-parent projects stop being leaves
        (pre-k7)
      keep_umbrella:
        The k7 rule stands; only stale contract text about job_email_recruiters is
        refreshed
    default: revert
    why:
      The rule chased job_email_recruiters' mis-parented frontmatter, which Bryan has
      since corrected
    answer: revert
  memory_glossary:
    ask: Under leaf_rule = revert, restore the pre-k7 Leaf Project glossary strand?
    memory:
      - glossary:leaf-project
    default: true
    corrects:
      "A standalone [[top-level-project]], a `root: true` project with no sub-projects,
      and an orphan project are leaves."
    answer: true
proposed_by: bbugyi200.athena.1bl
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 13:36:00
status: wip
---

# Revert the k7 umbrella leaf rule to the sub-project rule

## Context

Epic `bob-cli-k7` (plan `202611/keep_moving_link_marks.md`) redefined **Leaf Project**
from "a project note that is not its own root parent" to "a recognised project that is
not an umbrella" (a root project whose `^prj` carries a `🧩 **Sub-projects:**` marker
listing ≥ 1 child). Under the new rule, standalone top-level projects, `root: true`
projects without listed children, orphans, and unresolved-parent projects all became
leaves, so they now get keep-moving offers and `⏭ N` marks.

That redefinition was built on a wrong premise. The screenshot example,
`job_email_recruiters.md`, had `parent: "[[job]]"` (the `job` **area**) only because
Bryan mis-parented it; he meant `parent: "[[job_2026]]"` and has since corrected the
vault (verified today: its parent is `job_2026`, a project). Under the pre-k7 rule the
corrected note is an ordinary sub-project leaf, so the ⏭ 4 screenshot case works with
no rule change. The sibling plan `~/.sase/plans/202611/opt_in_project_roots.md`, which
chased the same mis-parenting from the root-parent side, was rejected; R2 root-parent
semantics were never changed by k7 and stay untouched here.

Behavioral consequence of the revert: the six standalone top-level projects currently in
the vault (`job_update_cv`, `job_apply_to_databricks`, `dev_plex_hack_patch`,
`vacation_sc_26`, `needs_attn_tasks`, `gkeep_gdocs_inbox_dump`) return to never-leaf: no
keep-moving offers, no ⏭ mark. Making any of them a leaf is a one-line `parent` edit in
the vault, Bryan's to make; this plan never writes the vault.

What k7 built that **stays**, under either decision:

- The entire keep-moving marks feature: `docs/keep-moving-marks.md`, the 141d/141e model
  and render fragments, the 271g/271h mixins, CSS, gates G0–G8, the count rule,
  tooltips, the parity invariant (KB24), and the off-clock chip at side 4. Only the
  G2/K2 `LEAF(N)` definition reverts.
- The `_generated/` ambiguity fix (`projectParentPathIsIndexed` filtering in parent-stem
  resolution, LP16). `bob-cli-k5` was closed on it and stays closed. Under the pre-k7
  rule this fix matters more, not less: an ambiguous parent stops the walk and wrongly
  makes a sub-project its own root, i.e. a non-leaf.
- `bob-cli-ke` (the scratch-Obsidian picker stall) is unaffected in substance; do not
  touch it.

> [!decision] leaf_rule = keep_umbrella No code, test, or memory changes. Only append a
> dated note to the Live verification section of `docs/keep-moving-marks.md` stating
> that `job_email_recruiters` is now a sub-project of `job_2026`, so the recorded
> audit's "standalone top-level project" premise no longer describes the live vault (the
> mark itself still renders, since a sub-project is a leaf under both rules). Run
> `just docs-check` and stop; every step below is skipped.

All remaining steps assume `leaf_rule = revert`.

## Ground truth to read first

- bob-cli commit `b2c3697e` (k7.1) — the diff whose semantics this plan reverts. The
  pre-images at `b2c3697e^` are the authoritative revert targets for
  `src/native/task_complete/keep_moving.rs` (`is_leaf_project`),
  `docs/task-dependencies.md` §12.15–§12.17, `docs/projects.md` "Leaf projects",
  `docs/pomodoro-homes.md` (~lines 77–88), `docs/reference/json-outputs.md` (~211), and
  `sase/memory/glossary/leaf-project.md`.
- bob-plugins commit `87b28e1` (k7.2) and the pre-image of `successorLeafProject` at
  `87b28e1^` in `plugins/task-status-cycler/src/089c-successor-keep-moving.js`. The
  post-image's `successorKeepMovingResolveParent` (with `projectParentPathIsIndexed`) is
  kept.
- `docs/keep-moving-marks.md` (the display contract: gates, KB/KT/KP vectors, Live
  verification) and the §12.16 KM6/KM33 rows plus the §12.17 LP table as they stand.
- `plugins/bob-ledger-tools/src/141d-keep-moving-marks-model.js`
  (`keepMovingMarkLeafProject`, `keepMovingMarkHasSubprojectMarker`) and its tests.
- Reference memory to read with `/sase_memory_read` before editing: `docs.md` (docs/ and
  generated-glossary rules). Read bob-plugins' `AGENTS.md` before working there.

## Steps

### 1. Rust (bob-cli)

- Restore the pre-k7 `is_leaf_project` body in
  `src/native/task_complete/keep_moving.rs`: the `project_surface(...).is_none()` gate,
  then `home.kind == Project && home.key == path && home.root_parent != home.key` — no
  marker shortcut, resolver walk always runs. Restore the pre-k7 doc comment.
- Remove `subprojects_listed` from `ProjectSurface` in `src/native/projects/surface.rs`
  and its asserts in `src/native/projects/tests/surface.rs`. Grep first;
  `is_leaf_project` is its only consumer today. If another consumer has appeared since,
  leave the field and say so in the commit body.
- `src/native/task_complete/tests/keep_moving_tests.rs`: re-score every LP assert to the
  old rule and keep the k7-era vector ids so test names stay stable across all three
  codebases: LP5, LP6, LP7, LP14, LP15, LP17 → not leaf (roots; the marker never changes
  leafness); LP13 → leaf, with its fixture reparented to the corrected arrangement
  (`job_email_recruiters.md` → `job_2026.md` project → `job` area); LP12 (cycle) stays
  leaf; LP16 stays leaf and now exercises the walk. KM6 reverts to "N is a root project
  → nothing". Rewrite the KM33 dry-run test (do not delete it) to the corrected
  arrangement: a sub-project leaf with 4 Next residents yields a 4-row `keep_moving`
  group exactly as KM1. Restore pre-k7 `CountingHomeSource` expectations (the ancestor
  walk runs again for marker-less projects).
- Run the repo's standard Rust checks per the justfile.

### 2. Docs (bob-cli)

- `docs/task-dependencies.md`: restore the pre-k7 §12.15 Leaf-project Vocabulary bullet
  and K2 line; drop the ROOT/UMBRELLA/LEAF block and the "What changes" / "Source of
  truth" notes; restore the pre-k7 G4 gate note (no "umbrella check on held text").
  §12.16: revert KM6; rewrite KM33 to the corrected sub-project fixture as in step 1.
  §12.17: restore the pre-k7 LP table, then re-add LP13–LP17 re-scored under the old
  rule (as in step 1) so every vector id keeps one meaning everywhere. Keep the anchor
  `#1215-keep-moving-offers-leaf-projects` unchanged.
- Restore pre-k7 wording (from `b2c3697e^`) in `docs/projects.md` "Leaf projects",
  `docs/pomodoro-homes.md` (a `root: true` project is never a leaf, so it never gets
  keep-moving offers), and `docs/reference/json-outputs.md` ("a subproject with a
  different root parent; it may itself have subprojects").
- `docs/keep-moving-marks.md`: point the "Which links" G2 gate at the reverted §12.15
  definition; rewrite the KB fixture preamble so N = `job_email_recruiters.md` (project
  → `job_2026` project, an umbrella under the `job` area); KB6 becomes "N is a root
  project (LP2, LP3, LP5, LP14, LP15)"; KB17's `_generated` stem vector now collides
  with the parent stem `job_2026`. Keep KT/KP untouched. Do not rewrite the existing
  Live verification record; append a dated addendum from this plan's own verification
  (below) noting the rule revert and the vault reparenting.
- Run `just docs-gen`, then `just docs-check` until the only failures are the five
  budget errors tracked by `bob-cli-jz` / `bob-cli-h9`.

### 3. Memory (bob-cli, only as `memory_glossary` allows)

- Restore the pre-k7 body and `summary` of `sase/memory/glossary/leaf-project.md`
  (pre-image at `b2c3697e^`) through the `/sase_memory_write` procedure. Then run
  `sase memory init` and re-run `just docs-gen` (the generated
  `docs/reference/glossary.md` follows the strand).

### 4. Cycler (bob-plugins task-status-cycler)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`.

- `src/089c-successor-keep-moving.js`: restore the pre-k7 `successorLeafProject`
  walk-only body (project type, no `root: true`, parent walk: area ⇒ false, project ⇒
  true, missing/unresolved/ambiguous parent ⇒ false, bounded cycle ⇒ true) — but keep
  the post-k7 `successorKeepMovingResolveParent` with its `projectParentPathIsIndexed`
  filtering. Drop the marker-count path and the held-text parameter:
  `successorLeafProject(app, path)` again. Revert the held-text threading through both
  `leafProject` call sites in `src/085-successor-plan.js`, keeping `options.leafProject`
  injectable. In `src/089-successor-project.js`, keep the shared listed-children helper
  only if `projectMarkerHasOpenChild` still uses it; otherwise restore the pre-k7 shape.
- `scripts/test-task-status-cycler-keep-moving.cjs`: re-score LP1–LP17 as in step 1,
  revert KM6, rewrite the KM33 end-to-end case to the corrected sub-project fixture
  (Ctrl+Enter on the live link yields the 4-row `Keep ‹N› moving` group), and keep the
  LP16 `_generated` case.
- `docs/task-status-cycler.md`: keep-moving paragraph and description row back to the
  sub-project wording. `npm run docs:check`.

### 5. Ledger-tools mirror (bob-plugins bob-ledger-tools)

- `src/141d-keep-moving-marks-model.js`: rewrite `keepMovingMarkLeafProject` as the
  root-rule port mirroring the reverted `successorLeafProject` (same answers on every LP
  vector), keeping the `ctx`-injected accessors and `_generated`-safe resolution. Remove
  `keepMovingMarkHasSubprojectMarker` and the marker-count branch if nothing else uses
  them.
- Re-score fixtures in `scripts/test-ledger-tools-keep-moving-marks.cjs` (LP and KB
  vectors), `scripts/test-keep-moving-marks-parity.cjs` (LP parity plus KB24 identity
  sets against the reverted cycler planner),
  `scripts/test-ledger-tools-keep-moving-marks-surface.cjs`, the lander's KM33
  real-wiring test, and the e2e fixtures in
  `scripts/obsidian-keep-moving-marks-e2e-lib.cjs` /
  `obsidian-keep-moving-marks-e2e.cjs`: everywhere N was a top-level project, give it a
  project parent (umbrella under an area) so it stays a leaf; add one negative fixture
  where N's parent is an area and no mark renders.
- `docs/bob-ledger-tools.md`: any "leaf project" wording back to the sub-project sense.
  The page sits at 5995 of its 6000-word budget — keep the edit word-neutral or
  shrinking.
- Bump both plugin versions per repo convention. `npm run build`, `npm test`,
  `npm run docs:check`, then `bob plugins sync` (follow AGENTS.md's refusal procedure if
  sync refuses).

### 6. Ship

- bob-cli: as of planning, no release tag contains `b2c3697e` (latest is v0.5.0,
  2026-11-07). If that is still true when this lands, commit as
  `fix(keep-moving): restore the sub-project leaf rule` with no `!`; if a release
  containing `b2c3697e` has shipped by then, mark the commit `!` (keep-moving groups
  disappear for standalone top-level projects). Never touch `Cargo.toml`'s version or
  `CHANGELOG.md` (`decisions:bob-cli-releases-ride-a-daily-train`).
- bob-plugins: separate commits per repo convention.

## Verification

- bob-cli `just check` green (the known `capture_link_tasks` failure tracked on
  `bob-cli-k3.4.1` excepted if still present). bob-plugins `npm test` green and
  `bob plugins sync` done.
- LP1–LP17 agree across Rust, the cycler, and the ledger port; KB24 parity holds.
- Scratch-vault audit (never `~/bob` itself, as in k7.6): clone the vault repo
  `--no-checkout` at current `HEAD` and point the built `bob` at it.
  - `bob capture --dry-run -f json '!job_email_recruiters:<an open block id>'` still
    returns a `keep_moving` group (leaf via its `job_2026` project parent).
  - The same dry-run against an open task of a standalone top-level project with a live
    link staged in the scratch day note (for example `job_update_cv`) returns **no**
    `keep_moving` group.
  - In scratch Obsidian with the synced plugins, the recruiter link renders `⏭ N` with
    N matching the dry-run rows, and the standalone top-level project link renders no
    mark.
- Append the dated results to the Live verification section of
  `docs/keep-moving-marks.md`, then re-run `just docs-check`.
