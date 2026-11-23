---
tier: tale
title: Finish reference refresh convergence and land bob-cli-pj.2.1
goal:
  Reference scans converge safely through replacement plans and lock retries, with the
  epic and parent phase verified and closed.
size: medium
proposed_by: bbugyi200.athena.bob-cli-pj.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pj.2.1
create_time: 2026-11-22 21:12:36
status: wip
---

- **PARENT:**
  [202611/refs_incremental_coherence.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/refs_incremental_coherence.md)
- **BEAD:**
  [bob-cli-pj.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pj/bob-cli-pj.2.1.md)

# Finish reference refresh convergence and land bob-cli-pj.2.1

## Scope and ownership

Repair the remaining correctness and verification gaps in
`plan:202611/refs_incremental_coherence.md`, then finish its landing in this same coding
turn. This is one medium tale: the implementation changes are confined to the existing
scan evidence/convergence code and its tests. It needs no child epic, client changes,
new public contract, or memory edits.

The epic is **bob-cli-pj.2.1**, its direct parent is phase **bob-cli-pj.2**, and the
containing epic **bob-cli-pj** has its own waiting land agent. Close this epic and then
only its parent phase when verified; do not close bob-cli-pj or mark that outer epic's
plan done. The approved `stall_after = two_hours` decision is final and unchanged. Do
not edit release versions or CHANGELOG.md.

The landing review baseline is bob-cli `29d7fec7`. All three phases are closed:
`063a8768` records dependencies/classifies changes, `33f6d08d` implements preparation
refresh, and `29d7fec7` implements lock-window refresh. The lander read every child note
and both linked plans, inspected these commits and actual source, and found the issues
below by code inspection. It has **not** run new Rust regressions or claimed a fresh
`just check` pass. Add regressions before repairing these paths, and verify the combined
tree yourself.

## Remaining defects

1. **A lock refresh can return before converging.** In
   `scan_apply.rs::lock_validate_intake_and_execute`, the Refresh branch calls
   `refresh_prepared_once`, discards the returned `ScanChange` as `_`, and checks
   `prepared_scan_needs_mutation` immediately. If the replacement is Failed or Held, or
   otherwise eliminates writes, it returns `no_work_applied` even when that refresh
   detected another change. Concrete regression recipe: adapt
   `failed_subset_plan_is_replaced_by_later_recovery_without_stale_drafts` to trigger
   its first sidecar edit at `after_prepare`; corrupt the sole PDF at `before_refresh`,
   then restore its original bytes at `before_refresh_evidence`. The returned Failed
   outcome has no writable work, while the returned change requires another PDF refresh.
   The current lock branch skips that recovery. It can similarly discard a global or
   destination-asset result before returning no-work.
2. **Replacement dependencies are absent from the refresh comparison.**
   `refresh_prepared_once` captures `pre.observations` and `pre.owners` from the old
   outcomes. After planning replacements it compares pre/post without incorporating
   dependencies those replacements actually consulted. The classifier uses
   `old.observations`, and its Markdown-content branch uses only `old.owners`. Initial
   preparation explicitly augments its pre evidence from the outcomes; refresh does not.
   A sidecar edit can make a formerly non-consuming plan start consulting vault links.
   If an existing unowned note gains a relevant link after that read but before refresh
   evidence capture, the old flags classify it as finalization-only. A later
   empty-subset pass can adopt the edited bytes without rerendering the missing
   redirect/stub. The same lifecycle problem applies to new parent/lineage reads and
   incomplete failures.
3. **Comparison-only ownership leaks into the accepted baseline.**
   `complete_comparison_evidence` merges previous states and owners into `post`, and a
   stable refresh adopts that augmented `post` directly. Superseded routed/reading
   ownership can therefore remain attached to a replaced PDF for the rest of the
   invocation. Its comment promises stable evidence from new outcomes only, but the
   implementation keeps the old union. Old dependencies are necessary for the
   comparison, not as permanent ownership after a stable replacement.
4. **Several promised acceptance paths lack end-to-end evidence.** The new classifier
   tests manually construct flags and candidates, while the new interleave tests cover
   sidecars, daily deduplication, preparation failure recovery, and separate
   preparation/lock churn. They do not prove real shared reader acquisition,
   locator/finalization ownership changes, or one budget spanning both preparation and
   locks. Also, phase three left four `#[allow(dead_code)]` attributes on the now
   test-only legacy validation adapter and string predicate instead of retiring that
   transitional path.

## Implementation

1. Make one converged refresh operation serve the lock consumer as well as the two
   unlocked consumers. Reuse or narrowly expose `settle_prepared_scan` (or an equivalent
   small wrapper); do not invent another retry budget. Consume every returned change,
   including empty-subset, global, and destination failure results, before a no-work
   return. For actual work, reacquire locks in maintenance/scan order and validate
   again. A genuinely stable no-work result may return without reacquiring.

   Preserve the shared allowance of four refresh passes and one full rebuild, including
   recovery/intake rebuilds; preserve pinned date, hooks once, recovery-before-hooks,
   error codes/messages, and receipt ownership. Hard failures observed after dropping
   locks remain prelock failures with no abandoned-retry receipt. Destination changes
   observed while holding locks keep the locked `scan_failed` receipt path. Never retry
   after an owned intake move. Collision validation remains mandatory before acceptance.
   Keep the explicit conservative rule that any observed `ref_dir` input change causes a
   full rebuild, including an existing reference-note edit.

2. Separate live outcome evidence from comparison-only evidence. Compare the last
   trusted bytes against a fresh pre-snapshot before replacing anything, retaining newly
   arriving edits and destination/global precedence. Associate the pass's pre bytes with
   the dependency flags/owners actually used by retained and newly produced outcomes for
   its pre/post comparison. Preserve prior dependencies long enough to detect
   removed/changed inputs, including destinations and Planned-to-Failed/Held
   transitions. Treat incomplete failures conservatively. Do not solve this by assigning
   every Markdown file to every PDF.

   On convergence, retain only current outcome dependencies and matching per-PDF
   candidate baselines; comparison-only owners must not leak into later selection. On
   another mismatch, retain enough evidence to classify the new drafts correctly on the
   next pass. Continue replacing both draft maps, clearing stale entries, and finalizing
   all plans in canonical order. Keep direct disk hashes and planner-hash agreement
   checks.

3. Retire the production-unused `PreparedValidation`, `validate_prepared_scan`,
   `scan_change_to_validation`, and `is_destination_asset_stale` compatibility layer.
   Prefer updating tests to assert `ScanChange` and the typed error constructors
   directly; narrowly test-only helpers are also acceptable. Remove the new broad
   dead-code allowances. Keep public behavior unchanged.

## Focused regression coverage

Use the existing bounded debug handshakes and exact `locked=0 count=N` plan log. Assert
convergence and current output for finite changes; do not use an
either-success-or-refusal assertion where success is required.

- The sole-PDF lock-window Failed-to-recovered scenario above must import the current
  sidecar without an abandoned draft, with the expected extra subset pass and no refresh
  under either writer lock. Include a stable no-work control, and a no-work refresh that
  detects a global or destination change so those results cannot be discarded.
- Start with an existing reference whose initial plan does not consult vault links, then
  change its sidecar so a refresh does. During that refresh, add a link to the relevant
  existing block in an already-existing unowned Markdown note. Verify a further targeted
  parse and the redirect/stub in final output; an unrelated second PDF must stay
  retained. Reuse the real underline or section fixtures in `tests/underlines.rs` and
  `tests/section_edges.rs` under `src/native/highlights_ref/`. Cover a real root-parent
  consumer, including a root owned by one PDF and consulted by another. This also checks
  the Markdown owner/shared-read union with current observations.
- Prove current ownership replaces obsolete ownership after convergence, including a
  newly owned Markdown target and a later edit to a former target. Keep separate
  assertions for cleanup after Failed/Held replacement.
- Complete the original matrix with targeted checks for one real locator candidate/line
  shift versus unrelated follow-up/orphan changes, and archive ID occupancy changing
  finalization without unnecessary PDF parses. Reuse existing fixtures and assertions
  where they already prove part of a row.
- Combine preparation churn and lock-window churn in one invocation: consume some
  refreshes before the first lock, the rest afterward, then the one full rebuild. Bound
  total plan calls by initial + four subsets + one rebuild; exhaust to `inputs_changed`
  without an intermediate retry receipt. Prove that a maintenance writer can acquire its
  lock during a refresh, not only during initial planning. Retain global rebuild and
  mixed destination/global fail-closed precedence coverage.

## Integration and verification

The lander fetched origin/master, which still matched `29d7fec7`. Since the epic's
creation it reviewed `8c019eb4` Gmail whitespace, `fdd94725` property operations,
`9d0cbfbe` requirement docs, `7f104e52` Pomodoro mention history, and `3e480e4a`
property commands. None duplicates or conflicts with the scan feature. Property writes
use the maintenance lock and are observed through the existing Markdown/root/locator
evidence; no new consumer API is needed. Repeat the drift review for commits after this
baseline before closing, and integrate anything newly relevant.

Run formatting and the meaningful focused regressions, then `just check` and
`just docs-check`. Do not run `just check-full`. Use `/sase_monitor` for long checks
according to that skill and keep this tale responsible for the remaining closeout. Do
not leave completion waiting for this turn's own commit, SHA, push, or CI result. Read
docs memory before changing any docs; only adjust the existing coherence section if its
wording needs correction.

Follow-up triage is already finished and recorded on bob-cli-pj.2.1:

- Phase bob-cli-pj.2.1.3 note #1's three Mac budget errors are independently reproduced
  in `file:explicit:f0ee94709e52d2aa9074a8ad`. Existing trackers bob-cli-jz
  (capture-previews/keyboard) and bob-cli-h9 (protocol) received corroboration. Mac
  `d76a4849` from active epic bob-cli-on.15.1 introduced every overage (5998→6082,
  5974→6036, 5999→6107; cap 6000); a `DISCOVERED ISSUE` on that epic assigns repair
  there. They are baseline failures for this refs epic, not unfinished scan work.
- The plugin-apis 8963/8000 report is a nonfatal develop-budget warning. Existing
  headroom task bob-cli-ob received corroboration; active bob-cli-on.11.1 received
  evidence of its API additions. A new CI-failure task was declined because the warning
  is advisory and remediation already has a tracker. No duplicate tasks were created.
- No other child proposed a follow-up. Phase .2's fake-curl transient error passed its
  isolated rerun; the lander did not independently reproduce it.

If docs still fails identically on these unrelated baselines, retain the evidence and
follow the original epic's explicit baseline-failure exception. Fix every failure caused
by this repair. Do not claim an unrun gate passed.

## Final step: finish the epic and parent phase

This tale has no land agent of its own. After the code and verification above are
complete, do all of the following in this coding turn:

1. Read bob-cli-pj.2.1 and all descendants with `sase bead read -r ...`; confirm each
   phase resolution is done, all notes are addressed, and the linked plan requirements
   and decisions are satisfied. Record the concrete fixes, gate results, post-review
   drift, and every follow-up disposition in the close note. If an actually incomplete
   descendant is named, finish or reopen it deliberately; do not force a successful
   nested landing.
2. Run `sase bead epic-symbols bob-cli-pj.2.1`. Resolve every listed exemption by
   wiring, privatizing, an appropriate non-test pragma, or deletion under Symvision
   policy. Re-key only when a still-open later bead needs the exemption. The lander
   found no entries at `29d7fec7`; recheck now.
3. Close normally with
   `sase bead close bob-cli-pj.2.1 --note "<specific verification, integration, and triage evidence>"`.
   If leftover symbols reject close, clean them and close again. Never use `--force`
   merely to succeed or advance this successful nested landing.
4. Run `just symvision` if available (there was no recipe at the review baseline), then
   set `status: done` in the frontmatter of the epic's PLAN:
   `plan:202611/refs_incremental_coherence.md`. Open the plans repository via
   `/sase_repo`, read the artifact through `sase artifact read`, and use its current
   resolved path; do not assume a workspace path.
5. Read bob-cli-pj.2.1 again for its parent link. Its parent is phase **bob-cli-pj.2**:
   reread that phase and `plan:202611/tick_contention_noise.md`, verify that the
   completed child satisfies `refs-incremental-coherence`, and run
   `sase bead epic-symbols bob-cli-pj.2` with the same cleanup rule. Close only that
   phase using
   `sase bead close bob-cli-pj.2 --note "<three-site convergence, dependency, budget, safety, and verification evidence>"`.
   Leave containing epic **bob-cli-pj** to its already-waiting land agent.
6. Finish through `/sase_final`, declaring every repository changed. The closeout occurs
   before host commit; do not wait for the repair commit to exist. If a real unresolved
   defect prevents close, record the blocker on bob-cli-pj.2.1 and bob-cli-pj.2 and
   report it accurately instead of forcing.
