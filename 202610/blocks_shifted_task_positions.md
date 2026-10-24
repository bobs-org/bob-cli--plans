---
tier: tale
title: Follow task positions through generated BLOCKS edits
goal:
  Review settlement and counted dependency edits follow their tasks through BLOCKS
  insertions and removals, with correct stamps and one undo per gesture.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.2
status: done
---

- **PARENT:**
  [202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.2.md)
- **COMMITS:**
  - [42f86c8](https://github.com/bobs-org/bob-plugins/commit/42f86c8381c4bdf21d51e1ee7d915314285749e2)
    — fix(navigation): follow task positions through generated BLOCKS edits

# Follow task positions through generated BLOCKS edits

Implement phase `bob-cli-an.6.6.6.5.6.2` (blocks-shift-indexes) from
`plan:202610/backlog_batch6_color_quote_walk_blocks_shift_promotion_race_mac_panel_ci.md`.
This phase supplies the fixes and evidence for `bob-cli-94` and `bob-cli-95`. Close only
the assigned phase; the parent lander owns those two task beads and every ancestor. Do
not set runtime-owned statuses by hand or create additional beads.

## Scope and evidence

This is one medium implementation pass: one existing line-coordinate mechanism, three
counted planning loops, and the Task Card's review outcome reader. The phase has no
earlier notes recording unfinished work. Inspection and in-memory Node reproductions on
bob-plugins `d2699df`, bob-navigation-hotkeys `2.56.2`, confirm that the fixes have not
landed. The linked checkout was clean. No product files were changed during planning.
`sase bead epic-symbols bob-cli-an.6.6.6.5.6.2` reported no entries; check again before
closing.

The parent epic's only finalized decision, `color_grouped_quote_id = remint`, belongs to
another phase. This work does not change that decision or any memory note. Preserve the
existing review and dependency contracts, particularly
`decisions:answering-advances-the-walk`, its move exception in
`decisions:task-move-never-advances-the-walk`, and
`decisions:task-deps-are-depends-on-links`.

Use `/sase_repo` to open `bob-plugins` before accessing it and read its `AGENTS.md`. All
paths below are relative to that repo unless identified otherwise. Edit source
fragments, never generated `main.js`; each hand-edited fragment must remain at most 1000
lines. Read the bead and design through their audited commands again if the
implementation starts on a newer tree. Preserve concurrent unrelated changes. The next
parent phase, promotion-anchor-race, also edits this plugin and depends on this phase;
this task must supply a rebuilt entrypoint and deployment evidence.

Confirmed baseline fixtures:

```js
const content = [
  "- [ ] #task Target ^target",
  "- [ ] #task First ^first",
  "- [ ] #task Second ^second",
].join("\n");
const session = {
  explicit: true,
  targets: [
    { line: 1, rawLine: "- [ ] #task First ^first" },
    { line: 2, rawLine: "- [ ] #task Second ^second" },
  ],
};
```

Using `scripts/navigation-dependencies-stage-harness.cjs`:

- `helpers.planCountedLocalTaskDependency(content, session, {line: 0, rawLine: content.split("\n")[0], existingIdField: null, existingBlockId: "target"}, "Tasks.md", {blocksLinesEnabled: true, vaultFiles: ["Tasks.md"], resolveLinkpath: () => "Tasks.md"})`
  returns `valid:false` and "Could not plan dependency navigation for every source
  task". The identical call with BLOCKS disabled succeeds.
- For the first two fixture lines, construct a stage modal on First, capture its review
  origin/index/before-line, and apply the real `planDependencyEdit` adding Target using
  `helpers.applyEditorContentTransaction`. The plan's `parentLineNext` is 2.
  `settleReviewOrigin` instead passes the generated `\t- 🧱 **BLOCKS:** [[#^first]]` at
  index 1 to `continueReviewWalkAfter`, not
  `- [?] #task First [dependsOn:: Tasks__target] ^first` at index 2.
- The real `applyVaultCountedDependencyRef` adapter accepts a same-note Target snapshot.
  With the three-line fixture and a freshness-stamp spy, it attempts to stamp First at
  line 2 while processing Second, then refuses the next source with `not-task`,
  committing zero undo groups. A genuine cross-note control, with the prerequisite in
  Other.md, succeeds and stamps both intended dependents. Keep coverage for both adapter
  usages; do not claim the ordinary cross-note control fails on the baseline.

## Implementation

### 1. Track counted sources through each dependency plan

Use the established pattern in `src/084-blocks-stage-plan.js`: keep immutable source
snapshots for stale checks and reporting, plus separate mutable current line positions.
After each successful plan, map every tracked source through `plan.lineEdits` using
`mapLineThroughLineEdits` from `src/083-blocks-dependency-plan.js`. Those edits are
sequential coordinates, including both Depends-On structure and same-note BLOCKS
insertions/deletions; do not reinterpret them as absolute original coordinates or
replace them with a net line-count delta. A deleted/invalid source position refuses
before the editor commit. Do not mutate `session.targets` or reorder their public
snapshot identity.

Apply this to these three loops:

- `src/270-counted-batches.js::planCountedLocalTaskDependency`.
- `src/420-picker-counted-dependency.js::removeCountedDependencyWithoutInboxRoute`.
- `src/420-picker-counted-dependency.js::applyVaultCountedDependencyRefWithoutInboxRoute`
  (the loop behind `applyVaultCountedDependencyRef`).

Use the current mapped source line for planning, task/project classification, and
working-copy reads. Use that plan's `parentLineNext` for freshness stamping before the
next plan or cleanup changes positions again. Remap both already-processed and pending
sources so the final first-source position remains available. Preserve all-or-nothing
source planning, source-count notices, identity preparation, recovery/unblocked
behavior, subtree handling, BLOCKS follow-ups, and one editor transaction per gesture.
Replace the incorrect comment that bottom-up planning cannot shift pending sources.

Return the mapped first-source `cursorLine` from the local planner. Its writer in
`src/570-plugin-link-roll-and-schedule.js` explicitly supplies this position to the
transaction. If its subsequent `prepareDependencyPomodoroCleanup` fold changes
positions, map that cursor through the pre-cleanup/final-content alignment too. A fixed
planner must not still park the cursor on a generated BLOCKS line. Keep the existing
cursor character clamping and one-undo behavior. The writer's fragment is already 996
lines at the inspected base: put reusable position-mapping logic in the existing editor
helper fragment or extract a cohesive fragment rather than exceeding the 1000-line
limit.

### 2. Resolve the reviewed row in the committed content

Keep `reviewLineIndex`, `reviewBeforeLine`, the origin, and counted `handledRefs` in
their original coordinates: they identify rows in the pre-gesture review queue. Use a
separate mapped position only to read `afterLine`.

In `src/340-picker-core.js`, retain the original editor content for a captured review
origin when the card opens. Add a small shared modal reader near `settleReviewOrigin` in
`src/350-picker-task-card.js`. It aligns that snapshot with the editor's current content
using the existing `alignEditorLines` machinery in `src/181-editor-line-diff.js`, then
reads the original reviewed task's matched new line. That alignment already ignores
dependency fields, freshness fields, checkbox changes, and added block IDs, while
preserving line ordering. It also sees the final content after cleanup, rather than
stopping at an intermediate `parentLineNext`.

Use an identified alignment pair for the reviewed row, not the generic cursor mapper's
deleted-line clamping to a neighboring row. If no reviewed task can be identified,
settle conservatively with no resolving outcome and release the origin lock; never
substitute a BLOCKS/Depends-On row or whichever task is under the cursor. Support
unchanged content and missing editor/snapshot data safely. Existing tests that construct
modal stubs should supply realistic capture context where necessary.

Have `settleReviewOrigin` use this reader. Also reuse it for the identical card
`afterLine` read in the failed inbox-route fallback in
`src/455-picker-inbox-route-gate.js`; successful route outcomes keep their existing
explicit handling. This shares outcome reading without changing inbox routing or moving
any tasks as part of this fix.

Keep settlement idempotent and preserve the decision record: a resolving committed
answer from a landing advances once; dismissal, no-op, refusal, non-resolving edit, and
off-landing work stay. The original before-line and handled refs must not acquire
post-write line numbers. No change to the public review API, queue semantics, or the
Ctrl+Shift+M exception is needed.

### 3. Add behavioral regressions using the real harnesses

Extend existing scripts instead of adding an isolated mock implementation. Relevant
homes are `test-navigation-hotkeys-dependencies.cjs`,
`test-navigation-dependencies-stage-writes.cjs`,
`test-navigation-dependencies-stage-stale-guards.cjs`, `test-navigation-stamps.cjs`, and
`test-navigation-review-advance-gestures.cjs`.

Cover this matrix with compact fixtures:

- Two counted sources below a prerequisite: add with BLOCKS enabled and disabled; both
  source rows receive the dependency and appropriate stamp, generated rows and the
  prerequisite do not receive source stamps, and the input session stays unchanged. With
  BLOCKS enabled, the reverse row lists both dependents once.
- Toggle/remove with BLOCKS above the sources, including removal of the last reverse row
  and a partial CURRENT removal where one source is already unlinked. Check the right
  remaining sources, counts, and deletion mapping.
- Runtime counted add/remove: exactly one undo group, correct final cursor on the first
  source, no partial source commit if a source or preimage becomes stale. Retain the
  existing prerequisite-below case and existing blockquote guards.
- The vault-counted adapter's same-note above-source insertion and removal, plus a real
  external prerequisite control: stamp the intended dependent at its post-plan line and
  preserve cross-note follow-up behavior.
- A real review landing followed by a same-note dependency commit inserting BLOCKS above
  it: the captured outcome names the task, and the walk advances exactly once to the
  next remaining item. Repeat settlement to prove idempotence. Add a deletion-above case
  and a shifted non-resolving case that stays. Verify original handled refs,
  cancellation/refusal, and the shared inbox fallback. Use both explicit task block IDs
  and a task that starts without one where the existing planner supports it. Avoid tests
  that only assert an internal helper's index; include actual outcome/landing and
  written-content assertions.

Keep dates deterministic through the harness's existing date/stamper seams. These tests
validate in-memory or isolated fixtures, never live task-note edits.

## Validation and deployment

1. Reproduce the two task defects before implementing, then run the focused scripts
   after `npm run build`. Include the review predicate suite, editor-line-diff suite,
   dependency stage write/stale guards, counted dependency suite, and stamps suite. Run
   each relevant new regression against the old implementation when practical to
   demonstrate that it detects the defect.
2. Increment only the navigation plugin's patch version from the current integrated
   manifest version (baseline 2.56.2); do not overwrite a concurrent version bump.
   Regenerate its source-built entrypoint and the version-bearing plugin inventory using
   `npm run build` and `npm run docs:gen`. Read the docs reference memory before
   changing generated docs. Do not edit bob-cli package versions or `CHANGELOG.md`.
3. Run `npm test` and `npm run validate` in bob-plugins; they include the generated
   entrypoint check. Run `npm run docs:check` for the regenerated inventory. Respect the
   1000-line fragment ceiling, extracting a cohesive helper fragment and updating
   `src/fragments.json` only if necessary. Use `/sase_monitor` before a long check that
   requires handoff, and wait for the monitor command itself to exit.
4. Deploy the tested checkout through `bob plugins sync`, specifying the path printed by
   `sase repo open`, with `--no-pull --plugin bob-navigation-hotkeys`. Preview with
   `--dry-run --format json`, then sync and inspect the report; exit zero alone does not
   prove deployment because dirty vault files can be skipped. Verify synced state with
   `bob plugins list` against the same repo, and record the version and deployment
   result. Do not hand-edit the vault plugin or use an unreviewed force overwrite. The
   parent phase already authorizes normal plugin deployment.

If an unrelated check fails, reproduce it on the clean implementation base. An identical
clean-base failure does not keep this assigned phase open: append a
`PROPOSED FOLLOW-UP:` note on this phase with evidence and an existing tracking bead if
known, then complete this phase's validation and closure. Do not create task beads or
absorb unrelated fixes. Never report a red check as passing.

## Completion

Record the baseline, implementation, focused/full-check results, exact plugin version,
and verified sync in notes on this phase and supporting evidence notes on `bob-cli-94`
and `bob-cli-95`. Leave those task beads open for the parent epic's lander. Before
closing, run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.2`; resolve any owned entries
or re-key genuinely deferred entries to a still-open owner. Then run:

```sh
sase bead close bob-cli-an.6.6.6.5.6.2 --note "<implemented behavior, verification, and deployed version>"
```

Do not close an ancestor or start the parent's next backlog epic. Use `/sase_final` as
the last action before the implementation turn's normal final response, declaring the
bob-plugins changes and any primary-repo obligations so the host can commit them.
