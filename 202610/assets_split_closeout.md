---
tier: tale
title: Complete assets package checks and close the split epic
goal:
  Finish package failure coverage and release guidance, verify the integrated split, and
  close bob-cli-da.4.1 and parent phase bob-cli-da.4.
size: medium
proposed_by: bbugyi200.athena.bob-cli-da.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-da.4.1
create_time: 2026-10-27 18:12:54
status: wip
---

- **PARENT:**
  [202610/embedded_assets_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/embedded_assets_split.md)
- **BEAD:**
  [bob-cli-da.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-da/bob-cli-da.4.1.md)

# Finish the embedded-assets split landing

## Objective and close boundary

Finish the remaining work found while landing `bob-cli-da.4.1`, then close that epic and
its directly owning phase `bob-cli-da.4` in this implementation turn. This is one
bounded coding task, with no land agent to resume after it. The final step below is part
of implementation, before the host commits this turn's work. Do not wait for this work's
commit SHA, push, or CI result to perform closeout.

The approved scope is `plan:202610/embedded_assets_split.md`, whose linked plan path is
`202610/embedded_assets_split.md` within the plans repository. Its parent is phase
`bob-cli-da.4`, not a plan bead. Leave containing epic `bob-cli-da`, its plan
`plan:202610/backlog_batch_b2.md`, and original task `bob-cli-ai` for the
already-waiting parent lander. Do not inherit the containing epic's successor
backlog-planning work.

The final `assets_split = split_now` decision is already implemented. Preserve the
independent assets version, current root version, release-plz pin 0.3.173, rehearsal
setting, schedule, unique tags, and existing installed asset contract. Release-plz owns
future version/changelog changes. No publishing is required.

## Verified implementation to retain

At landing review, master and origin/master were both `3cc63511`. The tree was clean.
Read the epic and its two closed children with audited `sase bead read` and the approved
plan with `sase artifact read` before implementation; their notes contain the complete
verification and follow-up triage.

- `b63e5a04` moved the payloads into `bob-obsidian-assets`, added the workspace
  dependency, updated Rust embedding/Python source fallbacks/docs generators, and made
  Cargo packaging verify both packages together.
- `3cc63511` integrated independent package versions with the daily train, previews,
  summaries, root binary selection, and release documentation.
- All 24 relocated payload, notice, and provenance files were compared byte-for-byte
  with `b63e5a04^`; every comparison passed and no old-path copies remain. All 27
  installed support-asset paths and executable flags are unchanged. The internal
  print-font facade retains the 11-reader/13-total sharing checks.
- Post-start drift `f6287847` (PLAN/daily home), `6b0a30bf` (Mermaid extraction), and
  `a6a359b8` (theme split) was reviewed. The Mermaid change preserves the assets
  fallback paths. The other changes need no assets integration.
- Fresh `cargo fmt --all -- --check` and `just check-release-tools` passed; the latter
  ran 108 tests. The phases reported source and isolated installed adapter passes,
  shared-font tests, full package verification, docs gates, and root-only/assets-change
  rehearsals with the pinned release-plz 0.3.173.
- Last phase measurements were root 3,217,425 bytes (3.07 MiB, 4.93 MiB below the 8 MiB
  warning) and assets 5,004,764 bytes (4.77 MiB, 3.23 MiB below the warning), versus a
  pre-split baseline of 8,220,034 bytes. Refresh measurements during the final
  integrated package check.
- Neither `sase bead epic-symbols bob-cli-da.4.1` nor the corresponding query for
  `bob-cli-da.4` found exemptions. This checkout currently has no `just symvision`
  recipe; check again at closeout.

Recheck commits that arrive after `3cc63511` for relevant drift, especially any touching
the files below. Preserve the sibling Mermaid extraction behavior.

## 1. Complete the package failure-path coverage

The approved assets phase explicitly required missing-archive and independent
archive-size failure cases. `tools/release/test_package_check.py` currently has no
missing-archive test. Its `test_size_failure_applies_to_either_archive` only calls
`size_level`; it cannot catch a main-loop bug that ignores one package's failure or
reports the wrong archive.

Replace that ineffective case with meaningful tests of `package_check.main`. Reuse or
extract the existing temporary-archive/fake-Cargo fixture instead of duplicating its
setup. Keep both package listings otherwise valid, mock Cargo execution rather than
running builds in these unit tests, and exercise:

- Each package in turn has no archive after a successful packaging command: main returns
  nonzero and identifies the missing archive.
- Each package in turn exceeds `FAIL_BYTES` while the other is under its budget: main
  returns nonzero and identifies the failing package. Prove the second package is
  checked independently, rather than stopping after a healthy root archive.
- A package above `WARN_BYTES` but at or below `FAIL_BYTES` still succeeds with a
  warning; retain the existing strict threshold-boundary assertions.

Use small valid temporary tar archives with scoped size/threshold control to avoid
allocating megabytes just to test the gate. Keep meaningful package and diagnostic
assertions. If these cases expose an implementation defect, repair that defect in
`tools/release/package_check.py` within the same task. Preserve full verification by
default, `--workspace --locked --allow-dirty`, the optional local `--no-verify` mode,
and per-package notices/payload ownership checks.

## 2. Correct directly affected release guidance

Read `docs.md` with `/sase_memory_read`. Update
`docs/develop/releases/how-the-release-train-works.md`, whose final operating costs
paragraph still describes splitting assets as a future design choice. Explain the
implemented two-package workspace and per-package budgets, linking to the existing
size-cap guide instead of repeating its size table. Align the guide's
bump/rehearsal/Trusted Publisher wording with the current release reference: both
package manifests and changelogs can be updated; assets have a distinct tag and no
GitHub Release; root releases select the binary build; live publication would require
setup for both crates. The current rehearsal/live switch and root daily guard do not
change.

In `docs/develop/releases/reference.md`, clarify the existing job-graph/binary prose
that currently says binaries run whenever JSON reported a release: they run only for a
`bob-obsidian` release (or explicit binaries retry), never an assets-only result. Keep
this a focused correction to existing pages, without new pages, generated
decision-record edits, or doc restructuring.

## 3. Verify the final integrated tree

Run the canonical `just check`, plus `just check-release-tools`,
`just check-web-clip-adapter`, `just docs-check`, and full `just package-check` (without
`--no-verify`). These are the approved epic's landing gates. Record actual results,
optional browser/font-generation skips, and both compressed sizes and headroom. Each
archive must remain below 8 MiB with useful headroom; full packaging must verify before
the assets dependency exists on crates.io.

Do not run `just check-full`. A passing `just check` and failing `check-full` would be
test infrastructure work, not unfinished epic scope. Do not repeat unchanged passing
suites without a new failure, edit, or unresolved concern. Use `/sase_monitor` for long
commands and carry the eventual result through closeout; do not end an ordinary turn
while commands are still running. Keep existing pinned release-plz rehearsal evidence
unless a release-tooling change invalidates it; these documentation and test repairs do
not by themselves require repeating the rehearsal.

Repair any epic-caused failure before closing. If an unrelated failure appears,
establish its provenance before triaging it with `/sase_new_task`; do not mislabel an
epic regression as follow-up work.

## 4. Finish this epic and parent phase in the same turn

The lander completed follow-up triage before proposing this tale and recorded it on
`bob-cli-da.4.1`. Preserve both outcomes in the close note:

1. `bob-cli-da.4.1.2` note #1 (stale release-train guide) was accepted as remaining epic
   work and is resolved by step 2; no separate task was filed.
2. `bob-cli-da.4.1.2` note #2 (successor daily-train decision record) was declined after
   reviewing `decisions:bob-cli-releases-ride-a-daily-train`. Its reopening trigger
   warranted reconsideration, but the approved design retained the scheduled train, root
   daily guard, exact-SHA CI, release-plz ownership, and root release/binary contract.
   No accepted policy was reversed; a trigger alone does not require superseding the
   immutable decision. The workspace details are documented in the approved plan and
   release reference.

No distinct non-epic follow-up was found. The lander searched memory and all-type tasks,
swept the last week, and reviewed active epics and original task `bob-cli-ai`; no new
task or duplicate corroboration was warranted.

Complete closeout in this order:

1. Reread `bob-cli-da.4.1`, its children and all newly added notes, and the linked plan.
   Confirm the repairs, final gates, current descendants, and any post-review drift are
   complete.
2. Run `sase bead epic-symbols bob-cli-da.4.1`. Resolve each listed exemption by wiring,
   privatizing, using a valid non-test pragma, or deleting the symbol under the
   Symvision policy; re-key a Justfile entry only when an explicitly still-open later
   bead needs it. Leave no entries keyed to this epic or a closing phase.
3. Run `sase bead close bob-cli-da.4.1 --note "<verification>"`. Write a concrete note
   covering the byte/installed-layout audit, source and commit review, drift
   integration, repaired test and documentation gaps, gate results/skips, before/after
   sizes, and both follow-up outcomes above.
4. Run `just symvision` after closing if the recipe is available. Otherwise record the
   absent recipe and the clean `epic-symbols` query. Use `/sase_repo` to open `plans`,
   audit-read `plan:202610/embedded_assets_split.md`, resolve its printed path within
   that opened repository, and set only its frontmatter `status: done`. Do not hand-edit
   generated bead-store files.
5. Run `sase bead read bob-cli-da.4.1 -r "Need the parent link"` and confirm the
   expected parent phase `bob-cli-da.4`. Reread that phase, verify this completed child
   supplies its assets-split requirements and final `assets_split = split_now` decision,
   then run `sase bead epic-symbols bob-cli-da.4` and resolve/re-key any entries under
   the same rule. Close only that parent phase normally with
   `sase bead close bob-cli-da.4 --note "<verified split, measurements, and gates>"`.
   Leave `bob-cli-da` and `bob-cli-ai` to the containing epic's lander and report that
   boundary clearly.

If a close rejects leftover exemptions, clean them and retry. If it rejects unfinished
descendants, complete the actual missing work; never use `--force` to advance this
successful nested landing. A genuine canceled/superseded outcome needs deliberate reason
and resolution rather than forced success. If the parent phase's scope is incomplete or
ambiguous, leave it open, record the concrete blocker there, and report it. Do not
broaden into a parent epic landing or successor backlog plan.

Finally use `/sase_final` for the normal end-of-turn declaration so the host commits
this work and the plan status edit. Closeout is not ordered after that commit: it
happens in this turn, before final declaration.
