---
tier: tale
title: Prove scan refresh dependencies and finish the tick contention landing
goal:
  Real scan interleavings prove the remaining dependency and ownership cases, gates
  pass, and bob-cli-pj is closed with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-pj.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pj
status: done
---

- **PARENT:**
  [202611/tick_contention_noise.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/tick_contention_noise.md)
- **BEAD:**
  [bob-cli-pj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-pj.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-pj.land.md)
- **COMMITS:**
  - [08b2765](https://github.com/bobs-org/bob-cli--plans/commit/08b2765eb6028a786a812a20753e4b8b89ec2127)
    — docs(plans): mark tick contention plan complete

# Prove the remaining scan refresh cases and land bob-cli-pj

## Scope

Complete the missing acceptance coverage for epic **bob-cli-pj**, then close that epic
in this coding turn. This is one medium tale: the remaining work fits one agent in the
existing scan fixtures, production refresh functions, and focused regressions. Fix any
epic defect those tests expose. No new child epic, new public contract, client
migration, performance cache, or memory change is planned.

The original epic plan is `plan:202611/tick_contention_noise.md`. Its final
`stall_after = two_hours` decision remains unchanged. Its direct phases `bob-cli-pj.1`
and `bob-cli-pj.2` are closed. Phase .2 delegated to closed epic `bob-cli-pj.2.1`, whose
three phases are also closed. Its plans are `plan:202611/refs_incremental_coherence.md`
and the repair tale `plan:202611/refs_refresh_landing.md`. **bob-cli-pj has no parent
bead.**

This tale owns the remaining outer landing; nothing resumes another lander after it. Do
not wait for this work's own commit, push, SHA, or CI to close the epic. Closeout before
the host's final commit is normal. Do not change Cargo versions or CHANGELOG.md, and do
not run `just check-full`.

## Verified baseline and precise gap

The outer lander fetched and fast-forwarded to **15225a6a**. It read every bead and note
in the tree, all three linked plans, epic commits `badcd30c`, `063a8768`, `33f6d08d`,
`29d7fec7`, and `15225a6a`, and the relevant source and tests. No new Rust or docs gate
was run in that audit.

The following is implemented and must be retained:

- Tick maps vault/scan contention to waiting, bounds refs retries by cadence, escalates
  at two hours, retains pending derive work on waits, and preserves hard failures.
  `inputs_changed` and destination-asset `scan_failed` remain distinct.
- All three scan stale sites use typed classification and converge outside writer locks.
  One invocation shares four refresh passes and one full rebuild. The pinned date, lock
  order, recovery/intake rules, hooks, collision checks, and receipt ownership remain
  intact.
- The last repair consumes returned changes before a lock-window no-work return, unions
  live observations/owners into the comparison baseline, adopts clean live evidence on
  convergence, and removes the legacy dead-code validation adapter.
- Existing tests cover daily finalization, sidecar subsets, Failed recovery, lock-window
  no-work/global/destination outcomes, and a combined preparation/lock budget with an
  actual maintenance-lock acquisition during refresh.

The repair tale explicitly required additional real-reader and replacement tests but did
not deliver them. `scan_plan.rs` tests
`live_vault_link_acquisition_selects_unowned_markdown` and
`clean_baseline_drops_obsolete_owners` construct evidence by hand. They never exercise a
real replacement plan or stable evidence adoption. The parent-reader case in
`same_length_edits_before_evidence_replan` combines root edits with config/ref/library
changes and permits refusal; it does not prove targeted parent refresh. Real locator and
archive-occupancy cases are also missing. These are remaining approved acceptance work,
not optional follow-up tasks. No additional runtime defect is asserted by the outer
source audit.

## Complete the missing behavioral tests

Use real PDF/sidecar/note fixtures and the production scan preparation, refresh, and
apply flow. Prefer `tests/cli/highlights/scan_interleave.rs` with existing bounded
`Handshake`, `spawn_scan`, and `BOB_REF_SCAN_TEST_PLAN_LOG`. Narrow unit integration
tests may supplement it when they call actual production planning and refresh; assigning
flags, candidate vectors, or owner maps by hand cannot satisfy these cases. Assert
current output and exact selected PDF paths/counts, not merely success or a manually
constructed classification. Finite edits must converge.

1. **Acquire a real vault-link dependency during refresh.** Start with an existing
   published reference whose initial plan does not consult links, an unrelated second
   PDF, and an already-existing unowned `2026/day.md`. Change the first sidecar so the
   replacement plan absorbs a published underline or section block and now consults
   vault links. At `before_refresh_evidence`, after the real link read, add a link to
   the old block in the daily file. Require another targeted parse of only the first PDF
   and the correct redirect/stub in the final note. Reuse the fixture semantics in
   `src/native/highlights_ref/tests/underlines.rs::absorbed_identity_and_link_aware_stub_are_reversible`
   or `tests/section_edges.rs::declaring_sticky_redirect_stub_appears_when_linked` (the
   latter path is relative to `src/native/highlights_ref/`). The renderer-only tests are
   fixture examples, not substitutes for the scan test. Keep Markdown membership stable
   so a global rebuild cannot hide a missing consumer refresh.

2. **Exercise a real parent resolver and owned/shared union.** Build a birth/reopen
   fixture that actually calls the `plan_reading_task` resolver in `sync.rs`, with the
   root note also owned as another PDF's planned write target. Edit that existing root
   after its read, using an alias/status change that affects the observed parent result.
   Assert the owning and parent-consuming PDFs are both selected, while a control PDF
   without those dependencies is retained. Verify the resulting route/action or
   appropriate current-data refusal at the per-PDF level; do not accept a stale route or
   a top-level churn error for a finite edit. Avoid mixing this with config, `ref_dir`,
   or membership edits.

3. **Replace ownership through actual refresh convergence.** Use a routed or reading
   destination that changes from an existing former note to an existing new note. Drive
   the actual refresh to stable adoption, then edit the former note in the same
   invocation before apply. Assert that obsolete ownership alone does not reparse the
   replaced PDF. In a separate controlled edit to the new target, require that PDF to
   replan and preserve the edit in output. Avoid incidental broad parent/link
   dependencies when testing former-owner removal. Inspect the actual adopted evidence
   in a narrow production-flow test if needed to disambiguate legitimate shared reads.
   Include production cleanup assertions for a replacement that becomes Failed or Held,
   reusing the existing Failed recovery fixture rather than duplicating its
   already-covered draft behavior.

4. **Exercise real locator and finalization inputs.** Move or shift one PDF's actual
   located task within existing Markdown and assert that only its PDF is reparsed and
   the current line is used. An unrelated follow-up/orphan change should refresh shared
   finalization without reparsing other successful PDFs. Separately change ID occupancy
   in an existing archive note, keeping note membership and per-PDF inputs stable;
   require collision-free final preview IDs without another PDF parse. Use current
   locator/archive fixtures and the `reserve_preview_ids` production path. Existing
   daily duplicate suppression proves only one part of this requirement.

Check that the new tests would catch the original missing live-dependency union and
comparison-owner leakage, using focused temporary mutations if helpful, and restore all
such mutations. Do not add redundant tests for the already-proven mixed budget/no-work
cases. Fix production defects exposed by the new tests in the existing implementation
without weakening the expected behavior.

Preserve destination-asset precedence and the deliberate rule that observed `ref_dir`
changes are global, even when a reference note has one owner. Preserve PDF disk/planner
hash agreement, conservative incomplete failures, canonical outcome order, both draft
maps, and the finite shared budget. Never refresh after an owned intake move. Every
ordinary refresh plan-log line must have `locked=0`.

## Integration, gates, and follow-up disposition

The outer lander reviewed all non-epic changes since creation, including props
inventory/span/CLI (`e98c7fe9`, `fdd94725`, `3e480e4a`), Gmail HTML whitespace
(`8c019eb4`), dedicated-link history (`7f104e52`), and requirement documentation
(`c17dd80a`, `9d0cbfbe`, `bf005290`, `ef640fde`). Property updates already acquire the
maintenance lock, reread/replan, and write atomically. Existing Markdown, root, and
locator evidence covers them. Daily-history changes use finalization and actual
shared-read dependencies. No duplicated implementation or consumer migration was found.
Fetch origin, review new drift after `15225a6a`, and integrate any relevant changes
before closing.

Run focused scan/interleave regressions and tick tests, then `just check` and
`just docs-check`. Use `/sase_monitor` for long commands under its documented rules;
`sase tool run check` is the named wrapper for `just check`. Never claim an unrun gate
passed. The nested lander reported its prior check run
`9d1be27cee08c4bf935c9c3a581e780c` passed; that does not replace validation of these new
tests/fixes. Do not run `just check-full`.

Follow-up triage is complete and recorded on bob-cli-pj by this lander using
`/sase_new_task`. Carry these outcomes into the final close note:

- The sole `PROPOSED FOLLOW-UP:` is bob-cli-pj.2.1.3 note #1. Its Mac docs budget
  failures duplicate **bob-cli-jz** (capture-previews/keyboard) and **bob-cli-h9**
  (protocol). Owning epic **bob-cli-on.15.1** has since closed and reports Mac
  **c0959905** fixed them: 5976/5995/6000 words, cap 6000, with docs-check passing. The
  outer lander added attribution notes to the existing trackers, not new +1s or tasks;
  it did not independently reproduce a post-fix failure. Open linked repositories
  through `/sase_repo` if fresh docs mounting needs them.
- Plugin API documentation at 8963/8000 was an advisory develop warning. Existing
  **bob-cli-ob** owns headroom, with causal growth already recorded on active
  **bob-cli-on.11.1**. The outer lander recorded the disposition on both. A new
  CI-failure task was declined because the warning is advisory and remediation is
  tracked. Do not copy the obsolete assertion that the Mac overages still fail.
- Phase bob-cli-pj.2.1.2's fake-curl transient passed its isolated rerun; the outer
  lander observed no independent recurrence. No measured performance follow-up or other
  distinct proposal was present.

If a gate still has an unrelated baseline failure, establish its current cause and use
the original nested plan's baseline-failure exception, with accurate evidence. Fix every
epic-caused failure. For genuinely new unrelated work, follow `/sase_new_task`; do not
refile the proposals already settled above. No docs change is expected; read docs memory
first if a contract correction becomes necessary.

## Final step: close this epic completely

This tale must finish the outer landing itself after its code and gates:

1. Read `bob-cli-pj` and all descendants and notes with `sase bead read -r ...`. Verify
   their completed code, this remaining coverage, and all linked plan requirements.
   Record exact new test names/results, drift integration, and the triage outcomes above
   in the close note. Keep the already-completed child work; do not invent another child
   landing. If a genuinely incomplete descendant is identified, finish or reopen it
   deliberately.
2. Run `sase bead epic-symbols bob-cli-pj`. Resolve every entry by wiring, privatizing,
   a valid non-test pragma, or deleting it per Symvision policy. Re-key only to a
   still-open later bead that actually needs the exemption. Baseline 15225a6a has no
   entries and no `symvision` recipe; recheck availability.
3. Close normally with
   `sase bead close bob-cli-pj --note "<specific verification, integration, and follow-up dispositions>"`.
   If symbols reject close, clean them and retry. If incomplete phases reject it, finish
   or reopen them. Never force to make close succeed or to advance a successful nested
   landing; cancellation/supersession requires a deliberate documented reason and
   matching non-done resolution.
4. Run `just symvision` if available after closing. Then open the plans repository with
   `/sase_repo`, read the artifact through `sase artifact read`, and set `status: done`
   in the frontmatter of **plan:202611/tick_contention_noise.md** using the current
   resolved PLAN path. Do not leave that update to another agent.
5. Re-read `bob-cli-pj` for its parent link. The audited link is null, so no ancestor
   closes are needed. If the link unexpectedly changed, follow the land prompt's
   parent-phase/parent-plan verification rules before closing any ancestor.
6. Finish through `/sase_final`, declaring every changed repository. Do not wait for
   this turn's own commit or CI. If a real unresolved defect prevents closing, record it
   on bob-cli-pj and report it rather than forcing completion.
