---
tier: tale
title: Finish review-attention regression coverage and land bob-cli-n8
goal:
  Prove real-queue attention parity and upkeep-only Mac marks, finish source docs, and
  close bob-cli-n8.
size: medium
proposed_by: bbugyi200.athena.bob-cli-n8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-n8
create_time: 2026-11-18 11:32:57
status: wip
---

- **PARENT:**
  [202611/review_attention_follows_walk.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/review_attention_follows_walk.md)
- **BEAD:**
  [bob-cli-n8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-n8/README.md)

# Finish review-attention regression coverage and land bob-cli-n8

## Outcome and boundaries

Finish the remaining acceptance work from
`plan:202611/review_attention_follows_walk.md`, then close **bob-cli-n8** in this tale's
implementation turn. There is no separate tale lander and no original lander waiting to
resume. One coding agent can finish this bounded test/docs work.

The accepted decisions remain fixed: `s` is lit exactly when the available review walk
has rows, its tier is the first nonempty of PRE, NEW, PROJECTS, PENDING, NEXT,
RECURRING, TICKLER, ROTTEN, POST; POST shares PRE cyan, ROTTEN is orange, and the Mac
count uses `counts.walk`, falling back to `commitments` only when absent. Keep schema
version 1, the three label forms, and the existing w/b behavior. No memory changes, new
decisions, release version edits, or CHANGELOG edits are needed.

Use `/sase_repo` to open `bob-plugins` and `bob-mac-capture`; read their AGENTS.md. Use
audited bead/artifact reads for the original plan and the n8 landing note.

## Evidence already established by the lander

- All four n8 phases are closed and every phase note was reviewed. bob-cli commits
  `aea85529` and `91ff49f8`, plugin `c7e6d3e`, and Mac `1e20997e` implement the chosen
  product behavior. The decision and glossary updates are in place.
- The plugin attention-vector, walker-footer, and selector suites pass 98/98 on
  `b2a6d2b`. The vector suite's `reviewMemo` constructs `counts.walk` directly from its
  own `queue.length`, so its equality assertion does not test the production memo
  builder. The parity fixtures omit individual PROJECTS/PENDING/NEXT/RECURRING cases.
  The original plan explicitly requires a real memo-builder regression and every tier
  alone.
- Mac decoding tests prove walk-count preference/fallback and tone tests check the new
  hex values, but `WalkAttentionPresentationTests.swift` has no POST-only or ROTTEN-only
  report regression. `CaptureAgendaPresentation.swift` still calls the mark's count
  "commitments". `docs/idle-agenda.md` describes the new count but does not explicitly
  say ROTTEN and POST also light the mark.
- GitHub run `44337649084` passed on precommit snapshot
  `dc39311c435c837c82296c590686590fb48ad6af`; that is a CI snapshot, not the landed
  commit. The landed commit is `1e20997e84022cbcd45ffe48e67c6ef317609924`; its master
  run `44344402326` was still running at audit time. Do not report that run as green
  without inspecting its result. Original phase evidence covers the Mac scratch
  bundle/typecheck. No live Obsidian or Mac session was inspected by the lander.
- Post-start changes were reviewed: bob-cli `f0857be8` changes subprocess test fixtures;
  plugin `b2a6d2b` replaces two wall-clock gates with CPU-work gates; Mac
  `504531ef`/`c6e53611` adds the HEADLINE label and media for a separate agenda row.
  Preserve all of these. The headline row and title-row attention should coexist.
- `just check` (ToolRun `f85981b1d68a198f26b7c0dc3ccf4fe4`) stopped at the existing
  rustfmt-only closure in `tests/cli/albums_create.rs:1149`, introduced by `2f80442b`.
  bob-cli-n7's closeout already includes that repair. No n8 failure was inferred from
  it. No `just check-full` was run or is authorized by this plan.
- Follow-up triage is finished and recorded on n8: n8.2 note #1 corroborated
  `bob-cli-io` (+17) and `bob-cli-m1` (+3). Its keep-moving/off-clock observations
  predate the n7.4 CPU-work fix, so explanatory notes were added to closed tasks
  `bob-cli-ki`/`bob-cli-ju`; new tasks and reopening were declined. No new task is
  needed for these reports. Keep that disposition in the final epic close note.
- `sase bead epic-symbols bob-cli-n8` found no entries, and bob-cli has no `symvision`
  recipe at audit time. Recheck at closeout. n8 has no `parent_bead`.

## 1. Exercise the production plugin memo-to-attention path

Extend the existing attention-vector suite (and the freshness memo suite only if that
keeps the harness simpler) using `LedgerToolsPlugin`, `makeFreshnessTask`,
`makeFreshnessApp`, and a pinned local date from `scripts/ledger-tools-harness.cjs`.
`scripts/test-ledger-tools-freshness-review-model.cjs` shows plugin setup/teardown;
`src/230-plugin-freshness-api.js` owns `freshnessEnsureMemo`, and
`src/110-freshness-queue.js` owns `freshnessQueue` and `freshnessCounts`.

Build actual task fixtures and call the production memo builder. Cover each of the nine
tiers alone, the S1-S7 mixes from `docs/walks.md`, and the empty queue. Assert the
actual memo's walk count equals the actual queue length, the queue's tier order agrees
with the expected fixture, and the push is nonnull iff the review footer is non-clear.
For a nonempty queue assert the first tier and the exact label, including
singular/plural/zero commitments. Verify both selected-s and off-s behavior; only the
off-s label appends `; click to show`.

Keep a separate unavailable/indexing case: it must clear attention; unavailable is not
the same as an available empty walk. Retain the defensive no-known-tier case. Avoid
computing both expected counts and actual counts from one synthetic queue. Existing
tests of CSS and selector ink already pass; keep them.

Fix product code only if these regressions expose a real mismatch. For production plugin
edits, edit source fragments and regenerate main.js with `npm run build`. Never
hand-edit generated main.js. Tests-only changes need no plugin version bump.

## 2. Exercise upkeep-only Mac marks and finish the source docs

In `Tests/CaptureCoreTests/WalkAttentionPresentationTests.swift`, decode same-day JSON
reports for POST-only and ROTTEN-only states with `available: true`, `attention: true`,
positive `walk`, zero `commitments`, and the exact S2/S1 labels. Run those decoded
values through the real presentation. Assert one s mark with the nonzero walk count,
correct tier and complete label, present on the title row and in accessibility output.
Check the tone via `WalkAttentionTone`, with an explicit PRE/POST equality assertion in
the tone suite. Cover calm, unavailable, and wrong-day suppression using existing tests
where sufficient.

Add one bounded integration assertion using a real agenda fixture with a Daily Headline:
the headline survives while an upkeep-only s mark remains on the title row. This
protects integration with `504531ef`; leave its layout/rendering intact.

Correct the `CaptureAgendaWalkMark` comment in
`Sources/CaptureCore/CaptureAgendaPresentation.swift` to say walk landings. Update the
walk-attention paragraph of `docs/idle-agenda.md` to state that any landable review row
lights s, including ROTTEN upkeep and POST, with PRE/POST cyan and ROTTEN orange. Edit
mounted docs only in the source repo. Read bob-cli `docs.md` memory before regenerating
its docs. No memory modification is required.

## 3. Verify and account for drift

Refresh the local/base history before final verification and review new changes since
the audited commits for actual overlap. Incorporate any relevant consumer change without
undoing unrelated work. If the albums-create rustfmt repair is still absent, apply only
the reported one-line formatting correction (preserving the no-H1 assertion); if it has
landed, do nothing to that file.

Run the focused plugin attention/footer/selector and new production-memo tests,
`npm run build:check`, `npm run validate`, and `npm run docs:check`. Run `npm test` once
for the final plugin tree. Handle io/m1's already-triaged unrelated timing failures
using the known reports and focused controls; do not weaken thresholds or claim a red
full suite passed. Run `bob plugins sync` after any repo changes, following its
overwrite-protection procedure without `-F`.

Compile and test the changed Swift with `just portable-check` and run `just docs-check`
in bob-mac-capture. These changes belong to CaptureCore and its portable tests. If an
actual fix also touches macOS-only targets, additionally perform the required Mac
scratch bundle/typecheck; do not land uncompiled Swift. Run bob-cli `just docs-gen`,
`just docs-check`, and the canonical `just check`. Do not run `just check-full`. Use
`/sase_monitor` for long gates if needed and put the remaining local verification and
step 4 in its continuation.

The user's landing rule is explicit: step 4 must not depend on obtaining this work's own
commit SHA, pushing it, or waiting for its CI result. Finish local verification and
close n8 in the same turn as the final edits. Preserve the repo's CI monitoring
obligation by staging its `Scripts/macos-ci-check` after closeout, as described below;
do not create a self-commit dependency in this tale.

## 4. Complete the epic landing in this tale

Re-read bob-cli-n8, all descendant statuses/notes, and the linked original plan to
confirm the scoped gaps above are finished and no new blocker appeared. If this tale is
assigned its own descendant bead, close that completed bead normally first so the
ancestor's descendant check can succeed. Do not close unrelated phases or force a
successful nested landing.

Run `sase bead epic-symbols bob-cli-n8`. For every entry, resolve it (wire it up,
privatize it, add the policy-appropriate non-test pragma, or delete it), or re-key it
only to a still-open later bead that actually needs it. Do not defer cleanup. Then close
the epic normally:

```sh
sase bead close bob-cli-n8 --note "<verified source/commits, nine-tier real-memo parity, upkeep-only Mac marks, headline integration, test results, selected DECISIONS, and follow-up dispositions>"
```

Include the n8.2 follow-up outcomes (io/m1 corroborated; ki/ju fixed later and no new
task/reopen), the independent 98-test baseline, final verification evidence, and whether
either UI was checked live. If close rejects leftover symbols, clean them and retry. If
it identifies unfinished phases, finish/reopen those deliberately; never use `--force`
just to make it succeed. A canceled/superseded scope requires an actual deliberate
outcome and explicit reason, not a workaround.

After closing, run `just symvision` if the recipe now exists; otherwise record that it
is unavailable. Open the plans repo through `/sase_repo`, resolve
`plan:202611/review_attention_follows_walk.md` with `sase artifact path`, and set
`status: done` in that file's frontmatter. The plan path relative to the plans repo is
`202611/review_attention_follows_walk.md`.

Run `sase bead read bob-cli-n8 -r "Need the parent link"` after closing. The audited
epic has no parent, so finish normally. If that changed, follow the user's parent rules:
verify and close only a directly parented phase; for a plan ancestor recheck its landing
note, all descendants/notes, linked plan and drift, retire its symbols, close normally,
run available symvision, mark its plan done, and repeat only while each direct plan
ancestor is complete. Record any first incomplete/ambiguous parent's blocker instead of
forcing it.

After this closeout, stage `/sase_monitor` for the Mac repo's `Scripts/macos-ci-check`
with a continuation that records the CI outcome and repairs a real failure if needed.
This honors its verification workflow without blocking n8's close on this work's own
SHA/push/CI. Wait for the monitor start command itself to exit with Staged, then finish
through `/sase_final`, declaring every modified repository including the plans sidecar.
Do not wait for the staged monitor. Do not leave this epic awaiting another land agent.
