---
tier: tale
title: Finish the highlights scan lock-window landing
goal:
  Repair optimistic scan validation, intake collisions, and recovery receipts, then
  close bob-cli-mr.3.1 and its parent phase.
size: medium
proposed_by: bbugyi200.athena.bob-cli-mr.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-mr.3.1
status: done
---

- **PARENT:**
  [202611/highlights_scan_lock_window.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/highlights_scan_lock_window.md)
- **BEAD:**
  [bob-cli-mr.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mr/bob-cli-mr.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-mr.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-mr.3.1.land.md)
- **COMMITS:**
  - [bfbedda](https://github.com/bobs-org/bob-mac-capture/commit/bfbedda918ab3884fcba3b16a6206437f3b913fa)
    — docs(refs): correct last-scan receipt wording for narrow lock window

# Finish the highlights scan lock-window landing

## Assignment and fixed scope

Finish only the remaining work in epic **bob-cli-mr.3.1**, then finish its landing in
this same coding turn. This tale has no separate land agent. Its parent is phase
**bob-cli-mr.3** of epic **bob-cli-mr**; after landing mr.3.1, close only mr.3 and leave
mr to its waiting land agent. Do not wait for a future commit SHA, push, or CI job
before these closes: verified final code and closeout belong in the same turn, with
commits settled through `/sase_final` afterward. Never use `--force` for a successful
nested landing.

Read `sase bead read bob-cli-mr.3.1 -r "Need landing findings and follow-up triage"`,
its children .1/.2/.3 and all their notes, and the approved original plan via
`sase artifact read plan:202611/highlights_scan_lock_window.md "Need governing scan invariants"`.
Read the governing memories through `/sase_memory_read`:
`decisions:references-are-ingested-at-birth`,
`decisions:open-pdf-writes-are-coordinated`, `sase_beads.md`, and `docs.md`. The
ancestor's final decision is **scan_lock = narrow**. Preserve the existing shared
human/JSON lifecycle and the lock-free no-work behavior; do not redesign them. No
release-version, changelog, accepted-decision, real-vault, deployment, or cron edits. No
public CLI option or JSON schema change is needed.

The audit baseline is bob-cli **6c409ab2**. The three phase commits are 1e8f77af,
3236ceef, and 6c409ab2. Later non-epic commits 99005975 and 48a3116d add Record Card
reads/writes and completion test deadlines; their source does not need to consume the
scan machinery. Recheck drift before finishing, but do not repeat that unrelated work.

The landing audit rebuilt the current CLI and ran
`cargo test --test cli highlights::scan_interleave -- --test-threads=4`: all **18 tests
passed**. Two additional isolated probes against that rebuilt binary confirmed gaps
beyond that suite: (a) adding `lib/blogs/second.pdf` at the `planning` handshake returns
exit 0 with `summary.pdfs = 1` and creates only the first reference; (b) queued
`clash.pdf` and `clash.PDF` produce `output_collision` in dry-run, but write mode moves
both PDFs, creates one reference, then returns a per-PDF preimage error. Thus the latter
is already a partial publication, not just a theoretical guard omission. The
late-recovery receipt gap is established by source inspection; its reproducer is part of
this tale's regression work.

Audited reproduction artifacts are attached to mr.3.1. Read them with
`sase artifact read <ref> "Need landing reproductions"`:
`file:explicit:c5c3f5771f7e66352b19542a` is the isolated Python probe, and
`file:explicit:7aa8ba853d55aee2765444f5` is its output from the freshly built CLI. Build
the current CLI first and pass its actual Cargo target binary path to the probe; the
repository's `target/debug/bob` can be an older binary when Cargo uses another target
directory. Both probes operate only on temporary fixture vaults.

## 1. Bind validation evidence to the inputs actually planned

In `src/native/highlights_ref/scan_plan.rs`, `prepare_from_inputs` currently builds the
locator, plans all PDFs, finalizes/reserves, and only then calls
`capture_scan_evidence`. The subsequent validation compares fresh files to those
post-planning bytes. A source that changes during planning can therefore attach newer
evidence to an older plan. In particular, discovery precedes `plan_pdfs`, but
`evidence.lib_pdfs` is first read afterward: a PDF arriving while the `planning`
handshake is paused is present in both evidence snapshots while absent from
`prepared.pdfs`. The validator accepts the incomplete plan. The current `pdf_hashes`
comparison also repopulates values from the old plans on both sides; retain a real
comparison against the bytes used by each plan.

Implement a coherent optimistic read set before/after the pure planning pass. A
conservative content-and-membership snapshot of explicitly enumerated relevant trees is
acceptable, or carry exact read-time preimages from readers. It must cover:

- Library/xlib membership, including newly added/removed/renamed PDFs, sidecars,
  TextBundle text and source assets, audio companions, and prospective intake targets.
- PDF bytes actually parsed, existing/missing ref notes, routed and reading
  destinations, shared locator and ID-occupancy inputs, parent/alias/routing/config
  inputs, and the archive/capture metadata consulted by holds and reading/annotation
  finalization.
- Inputs that do not appear in the resulting plan because an earlier version was absent,
  made the plan a no-op, or caused a per-PDF hold/failure. Do not rely only on `Debug`
  output of the locator or only on the plan's selected output paths to cover parent
  resolution and archive ID occupancy.

Detect changes during preparation as well as after it. Keep expensive parsing and
rendering outside the mutation locks; do not repair this by planning the whole library
again while locked. Preserve one invocation date, one hook execution, the bounded retry
policy, safe refusal on persistent churn, and finalization reset semantics. Do not hash
unrelated vault media or introduce a persistent cache. Propagate relevant snapshot
errors instead of treating unreadable membership as empty/unchanged.

Add deterministic regressions for a PDF created during the existing `planning`
handshake, and same-length sidecar/ref/config or parent input edits after their reads
but before evidence capture. Add a private test seam if necessary, without a production
flag. Cover a newly created input that the old plan did not select. Every case must
replan with current data or fail safely; it must not accept old work with new evidence.
Retain the tests proving unchanged scans succeed under held locks and unrelated existing
PDFs are never reparsed inside the lock window.

## 2. Restore collision checks over the combined intake/library set

`prepare_from_inputs` validates note-output and planned-asset collisions, but writing
preparation includes only existing library PDFs. `integrate_intake_plans` calls
`plan_pdfs` for new intake and combines/finalizes outcomes without either collision
validator. This drops the former collision guard for intake-only initial libraries and
for conflicts between existing and new PDFs.

Before the first move, validate prospective note outputs for the existing library plus
all queued destination PDFs under the mutation locks. Recheck the combined final plans
for asset collisions before any plan executes. Keep source/destination non-clobber
checks and per-PDF partial-failure accounting. Do not introduce an unlock gap between
intake and note publication, and do not add a retry after any owned move. Preserve the
existing `output_collision` code, paths, and human/JSON error behavior. If a failure is
only discoverable after intake planning, use the existing protected failure/cleanup
semantics; never erase intervening data.

On a case-sensitive fixture, queue `xlib/blogs/clash.pdf` and `xlib/blogs/clash.PDF`,
which both map to `ref/blogs/clash.md`. Verify writing scan rejects the collision as
dry-run does and never overwrites one reference with the other. Also test a
queued/existing collision and conflicting assets across the combined set. Use the
existing guard helpers rather than duplicate their policy.

## 3. Publish recovery-only results at every recovery boundary

`recover_pending_overwrite` publishes a receipt while its locks are held. The recovery
branch in `lock_validate_intake_and_execute` instead recovers, drops both locks, and
rebuilds without publishing. If the rebuild becomes no-work or fails, actual recovery
mutations leave the sentinel receipt untouched. Make both recovery entry points settle
their result while serialized. Reuse a common helper where useful, and never publish an
abandoned validation retry as if it mutated.

Add a bounded handshake regression: start with work, pause after preparation, plant a
valid overwrite intent that restores/removes that work, then resume. Recovery must occur
under both locks and replace the sentinel receipt before the lock-free no-work return.
Also cover a post-recovery replan failure and keep human/JSON direct output correct.
Existing prelock failures and genuine no-work scans must still leave receipts untouched.
Preserve coordinated `.forMerging` marker writes, note-only PDF checks, dirty holds,
notification rules, and receipt/stdout identity for the final writing result.

## 4. Finish documentation and verify the final implementation

Open **bob-mac-capture** through `/sase_repo` and read its AGENTS.md before edits. Its
`docs/refs.md` currently says, near lines 53–58, that every writing scan rewrites
`last-scan.json`. Correct the source page (not bob-cli's generated mount): only scans
that perform recovery or enter the relevant mutation/receipt path publish a receipt;
lock-free no-work and prelock failures leave the previous receipt and notification
alone. Keep the direct command result distinct from the serialized receipt when recovery
alone runs. Check existing bob-cli contracts in `docs/highlights-ref-sync.md`,
`docs/vault-git-sync.md`, and `docs/ref.md` for any additional necessary clarification.
No Swift changes are expected.

Run focused tests selecting actual cases in scan_plan, scan_interleave, full-scan
human/JSON, marker writes, overwrite recovery, annotation/reading-task allocation,
create ingestion, and intake-only paths. Run **just check**, never `just check-full`,
for the final code. Run **just docs-check** with linked source checkouts opened through
`/sase_repo`. Use `/sase_monitor` for long verification, carrying this tale's complete
closeout instructions to the verification successor when needed; do not lose the final
landing step. The prior phase's prepared completion failed to recognize Rust stages, so
do not assume a prepared host-completion intent can replace manual landing work.

Known unrelated docs failure: `docs/reference/json-outputs.md` is 6089 words against
6000 at the audit baseline. The same page is 5999 at b0bddbb9/1e8f77af, 6058 after
99005975, and 6089 after 48a3116d; this was routed as `DISCOVERED ISSUE` to active epic
**bob-cli-ms**. If it remains, reproduce it on the clean current base and record that
attribution; do not raise the cap or call it remaining scan work. Verify changed docs
and later docs gates separately if that known lint issue prevents reaching them. Any new
failure from these repairs must be fixed before closing.

## 5. Finish this epic's closeout in the coding turn

The lander already completed `/sase_new_task` triage and recorded it on mr.3.1. Carry
these outcomes into the close note, rather than asking another lander to triage again:

- mr.3.1.1 #1 and mr.3.1.2 #1: duplicate generated-decision drift proposals declined
  because 48a3116d fixed the drift; `generate_reference(check=True)` returned `[]`.
- mr.3.1.3 #1: JSON word budget belongs to causal active epic bob-cli-ms, which has the
  independent evidence note. No duplicate task existed in all-status searches and the
  last-week sweep, so no new task was created.
- mr.3.1.3 #2: Mac receipt wording was caused by this epic and is completed above, not
  deferred to a new task.

Re-read all mr.3.1 descendants and notes, check their closed/done resolutions and linked
plan readiness, and review commits since 6c409ab2 for drift that needs integration.
Record exact passing tests and any independently attributed baseline failures. Run
`sase bead epic-symbols bob-cli-mr.3.1`; resolve every listed entry by wiring,
privatizing, adding a justified non-test pragma, or deletion. Re-key only to a
still-open later bead that truly needs the exemption. At audit time there are no entries
and no `symvision` recipe in this repository's justfile; check the current tree rather
than assuming that.

Then close normally:

`sase bead close bob-cli-mr.3.1 --note "Verified all three original phases and the landing repairs: coherent optimistic input evidence, combined intake collision guards, serialized recovery receipts, correct Mac receipt docs, preserved lock-free no-work and bob_sync.lock then scan.lock order, unchanged coordinated writes, and later-commit integration. [Insert actual check results and the three follow-up outcomes above.]"`

Replace the bracketed text with real results. If close refuses leftover epic symbols,
finish the cleanup and retry. If a descendant remains incomplete, finish it normally;
never force merely to get success. After close, run `just symvision` if the recipe is
available and confirm the whitelist is clean. Open the plans repo via `/sase_repo`, then
set only `status: done` in the frontmatter of the epic's linked plan:
**plan:202611/highlights_scan_lock_window.md**, using the current PLAN path reported by
`sase bead read` (never a prior agent's workspace path).

Next inspect the current parent link with
`sase bead read bob-cli-mr.3.1 -r "Need the parent link"`. For the verified parent phase
**bob-cli-mr.3**, re-read it and verify this finished child implements the approved
`scan_lock = narrow` work. Run `sase bead epic-symbols bob-cli-mr.3` and retire any
entries. Record evidence recommending resolution of original task **bob-cli-mo** to
mr.3's parent lander (do not close mo here, per the approved child plan), and close only
the phase:

`sase bead close bob-cli-mr.3 --note "Child bob-cli-mr.3.1 is complete: verified unlocked library planning and no-work scans, bounded validated mutation locks, paired intake publication, collision/recovery repairs, docs, and [actual tests]. This fulfills scan_lock=narrow; recommend resolving bob-cli-mo using this evidence. Containing epic bob-cli-mr remains for its waiting land agent."`

Again insert actual results and never use `--force`. Leave **bob-cli-mr** open. Submit
`/sase_final` as the last action with commit decisions for every repository changed,
including linked documentation and plan status. Do not postpone closure until those
commits exist; the host commits after the coding turn ends.
