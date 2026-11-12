---
tier: tale
title: Repair bank import integrity and finish its landing
goal:
  Reject corrupt imports, preserve authored notes and registry links, complete journal
  readiness, and close bob-cli-kd.4.1 and bob-cli-kd.4 normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kd.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kd.4.1
create_time: 2026-11-11 21:09:08
status: done
---

- **PARENT:**
  [202611/bank_import.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_import.md)
- **BEAD:**
  [bob-cli-kd.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kd/bob-cli-kd.4.1.md)

# Finish the bank import landing

Repair the remaining defects found while landing `bob-cli-kd.4.1`, then finish that epic
and its directly enclosing phase `bob-cli-kd.4`. This is one bounded coding task in the
existing bank modules, their tests, and narrow vault setup. There is no subsequent land
agent for this tale: its final step owns the closeout.

## Scope and evidence

Read `sase bead read bob-cli-kd.4.1 -r "Need the landing audit and follow-up outcomes"`
and its four closed children before working. Read the approved design through
`sase artifact read plan:202611/bank_import.md "Need the import acceptance contract"`.
The original plan file to mark done is `202611/bank_import.md` in the plans sidecar,
resolved through the bead's PLAN entry or `SASE_SDD_PLANS_DIR`.

The review inspected the implementation and commits `bffbf36b`, `ee2d2cbf`, `d9f0539f`,
and `b8c71181`, plus non-epic drift through `0d2f433d` (also the fetched base-branch
head). Existing bank tests pass 92/92 and shared transaction tests pass 13/13. Those
tests do not cover the failures below. Reproduction script:
`file:explicit:f1a42eafbd40c70d5d0a8f74`; results:
`file:explicit:0f0772d7cfd9fab573bd4a19`. Read these with `sase artifact read`, not by
opening its artifact-store path. It uses synthetic exports and temporary vaults only.
Its CLI probes used the installed `bob 0.5.0`; the source review confirmed the same
paths in the current checkout. Reproduce with the newly built binary while adding
regressions.

Confirmed remaining behavior:

1. An OFX file cut off after a transaction's complete leaf fields is accepted without
   closing transaction/list/statement containers. Omitting the first `</STMTTRN>` in a
   two-transaction file silently consumes the second transaction inside the first.
   `source::block_inners` returns the rest of the input when a close is missing, and can
   pair an opener with the next transaction's close.
2. Two valid files for checking with identical FITID `synthetic-1`, but amounts `-10.00`
   and `-20.00`, succeed with one new row and one seen row. The planner's `batch_keys`
   and existing-journal shortcuts skip immutable-money validation.
3. Two unjournaled rows with the expected eight-character bank ID cause import to add a
   third row under the twelve-character ID. The existing
   `duplicate_ids_are_reported_and_left_alone` test explicitly expects this incorrect
   behavior. The contract says an already-present ID is seen and warns against selecting
   an arbitrary duplicate.
4. Inserting Nov 12 after a Nov 11 bank row and its indented child places the new
   heading before Nov 11's trailing prose and plain checklist line. Their bytes survive
   but their day changes. `month::day_end_line` only includes the last row and its
   immediate indented children. The parent's existing note #1 describes the same issue.
   Post-start `03a0aba2` fixed an `ensure` cursor after children, not this whole-section
   ownership problem.
5. A valid restaurant registry mapping `eat -> restaurants` finds Test Cafe but produces
   `[[eat/test_cafe|🍽️]]` instead of its real note path. In addition, `plan_import`
   guards the literal `eat` directory using names collected from the configured
   directory, so apply can refuse an unchanged valid registry.
6. An acknowledged row taught as “My preferred cafe” from `SQ *TEST CAFE` is not learned
   for a new `TST*TEST CAFE` transaction, even though both clean to the same merchant.
   `identity::merchant_key` masks and folds raw text without cleaning it, contrary to
   the agreed cleaned-and-masked merchant key.
7. Child `.4` note #2 reports that the live vault has neither the bank journal ignore
   allowlist nor union attribute. This is remaining import setup, not a new task. Its
   predicted partial commit is inaccurate: `ob::commit_paths` explicitly stages the
   journal, and transaction failure rolls rows and append back. The real consequence is
   a failed first import. Improve the actionable refusal and complete the narrow
   settings work.

Honor the final parent decisions: live configuration remains checking only;
subscriptions are outside this task; only the outer wrap-up owns the approved glossary
edits. Make no SASE memory, version, or changelog changes. Balance proof, coverage
verdicts, twins, `import -F`, plugin UI, and real-transaction ingestion remain outside
this import epic. Preserve published parser/journal APIs consumed by `bob-cli-kd.5`.

## Implementation

### 1. Refuse corrupt source and identity evidence

In `src/native/bank/source.rs`, distinguish optional SGML leaf closes from required
aggregate boundaries. Validate the containers actually consumed by the parser so
missing/misnested transaction, transaction-list, and statement closes cannot silently
truncate or combine records. Keep OFX 1.x SGML and OFX 2.x XML, header encodings,
entities, empty statements, and posted calendar dates working. No malformed source may
reach the maintenance/sync lifecycle. Diagnostics name the file and field/record, never
its raw FITID or account number.

In `import_plan.rs`, validate each alias/full-key identity's immutable posted date and
cents across all batch files and against the folded journal before the seen shortcuts
produce effects. Equal repeated observations remain seen; inconsistent ones refuse the
whole batch with exit 2 and no planned row/journal writes. Source description changes
alone must not rewrite prior rows or become money conflicts. Use typed refusal data and
update the CLI mapping accordingly.

When an unjournaled matching ID appears on multiple rows, refuse the ambiguous batch
with a bounded diagnostic rather than creating another financial row or inventing
recovery provenance. Leave both existing rows byte-identical. Preserve single-row crash
recovery and legitimate collisions with a _different known full key_, including
eight-to-twelve extension and refusal of a twelve-character collision. Replace the test
that currently blesses a third duplicate row.

### 2. Preserve authored sections and integrate the current records registry

Repair missing-day insertion in `bank/month.rs` using the whole earlier day's section
boundary, respecting the next heading and fenced content. Do not reuse a “last row plus
children” location as a day boundary. Keep row append and ensure cursor behavior correct
for their different purposes; retain the child-preserving fix from `03a0aba2`. Assert
section ownership, not just substring survival, with trailing prose, a blank-separated
note, an ordinary checklist, an unfinished money-row stub, BOM/CRLF, and insertion
before an existing later day. Test through both ensure and import because they share
this helper.

In `import_run.rs` and `import_plan.rs`, carry the resolved restaurant note path and
directory through the snapshot and planner instead of rebuilding them under literal
`eat/`. Reuse `RecordContext`, `RecordCatalog`, and the latest shared read dependency
APIs. Guard membership in the directory actually read. Prove an unchanged nonlegacy
directory applies successfully and a newly added/changed restaurant input refuses the
stale plan. Keep exact-title ambiguity and rule/learned-link precedence intact. Do not
mutate restaurant records or their new generated indexes/catalogs.

Normalize merchant keys from conservative cleaned, masked descriptions. Continue
learning from current acknowledged row text, never waiting rows. Preserve old
append-only journal bytes: derive a comparable normalized merchant from an old event's
stored masked description when learning, instead of requiring a journal migration or
rewriting history. Test decoration changes such as `SQ *` versus `TST*`, preserve rule
precedence, and prove distinct merchants remain distinct.

### 3. Finish journal readiness and required transaction proof

Use the shared `vault_git::check_ignore_quiet` helper in the real import apply path to
refuse an ignored journal destination before installing any month rows, with the exact
`.gitignore` remediation in the hint. Do not run Git from the dry run planner, force-add
ignored data, or silently rewrite a user's Git policy on every import. Add a
temporary-Git-vault regression demonstrating the refusal leaves row/journal bytes
unchanged and the configured case commits both paths.

Recheck child `.4`'s setup finding. The only required vault-policy edits are the narrow
`.bob/bank/*.jsonl` allowlist and union merge rule documented in
`docs/vault-git-sync.md#bank-import-journal`; preserve all existing rules and make the
edit idempotent. Verify effective Git rules, including any ancestor-directory ignore
rules, using `check-ignore` and `check-attr`. Read `obsidian.md` through
`/sase_memory_read` and use `/sase_repo` before accessing a repository outside the
checkout. The lander's attempt to open inventory name `bob` was refused as unknown; do
not assume that name resolves or bypass the repository access rules. Resolve the actual
registered/provider reference through supported SASE inventory or configuration, then
use only the broker-returned path. A prepared change in that repository belongs in the
same final declaration. Verify the prepared tree; do not make closeout wait on this
tale's eventual commit SHA, push, or CI result. If the required repository cannot be
accessed, record the concrete blocker and leave the epic open rather than claim live
setup was completed.

The read-only live query already found exactly one matching chore in `gtd_daily`, with
`#gtd #pre`, `repeat = every week on Monday when done`, and scheduled `2026-11-16`.
Preserve it; do not capture a duplicate or alter the daily chore. Use the `bob_query`
skill if a fresh read is needed. Do not import real bank data or put raw exports in the
vault/repo.

The original import acceptance also calls for tests through the command's shared
lifecycle. Existing tests mostly exercise `plan_import` and `apply_and_commit` directly.
Add the missing bounded cases needed to establish the actual command: malformed source
never invokes sync, multi-file conflict leaves no writes, lock refusal,
plan-after-pre-sync, exact dry-run/apply bytes at a fixed clock, and post-sync failure
retains the committed rows and journal with a recovery warning. Reuse existing temporary
vault/sync seams. Retain append/verify/commit rollback, stale journal/membership,
replay, and row-only crash-recovery regressions. No new maintenance-lock implementation
or callback-owned journal append.

### 4. Verify, then finish this landing in the same coding turn

Read docs memory before editing user-facing help/docs. Make only contract changes needed
for the repaired refusal and Git readiness behavior. Run focused bank and host-catalog
tests, notebook/album transaction regressions for shared helpers, `just docs-gen` and
`just docs-check` when changing docs, and the required file-change gate
**`just check`**. Never run `just check-full`. Use `/sase_monitor` for long commands and
retain the closeout instructions in any verification handoff. A baseline infrastructure
failure is triaged separately, not turned into another unfinished feature. Do not claim
a pass that was not observed.

Re-read current child statuses/notes and review commits landed since `0d2f433d`,
especially bank balance, records registry, and `host_catalog::txn` changes. Keep the
newer album catalog dependency composition; its `ReadDependencies::extend` must still
carry byte, frontmatter, and both directory-dependency kinds. Confirm the published
source/journal/planner interfaces remain available for `kd.5`.

All original `PROPOSED FOLLOW-UP` entries have already been triaged by the lander:
`.1 #1` and `.2 #2` lint issue fixed by `6e3d8837`, no new task; `.2 #1` and `.3 #1`
completion failures corroborated on `bob-cli-g8` and active `bob-cli-3j`, separate
date-aging claim declined because the helper pins `BOB_NOW`; `.4 #1` docs anchors
corroborated on `bob-cli-kk`, mounted plugin link recorded on active `bob-cli-kb`;
`.4 #2` journal setup retained here. Carry these outcomes into the close note; do not
file duplicates. The epic's audit note records the complete rationale.

Finally, perform these actions yourself, after verification and without waiting for this
work's own commit, push, or CI:

1. Run `sase bead epic-symbols bob-cli-kd.4.1`. Resolve every returned entry by wiring,
   privatizing, deleting, or an appropriate non-test pragma; re-key only when a named
   still-open later bead truly needs it. Both this epic and `kd.4` had no entries at the
   landing audit. Recheck rather than assume that persists.
2. Close normally with
   `sase bead close bob-cli-kd.4.1 --note "<actual source, integration, probe, test, journal-readiness, and follow-up verification>"`.
   If rejected for symbols or unfinished phases, finish them and retry. Never use
   `--force` to advance a successful nested landing. Canceled/superseded work requires a
   deliberate reason and cannot substitute for a successful repair.
3. Run `just symvision` if the recipe exists (it was absent at audit). Open the plans
   sidecar through `/sase_repo`, read the original plan through its audited artifact
   reference, and set `status: done` in `202611/bank_import.md` frontmatter.
4. Run `sase bead read bob-cli-kd.4.1 -r "Need the parent link"`. Its parent is phase
   **`bob-cli-kd.4`**, not the outer plan. Re-read that phase, verify this child
   fulfilled its import scope and inherited decisions, retire/re-key any
   `sase bead epic-symbols bob-cli-kd.4` entries, then close only that phase with
   `sase bead close bob-cli-kd.4 --note "<verified child import scope and checks>"`.
   Leave **`bob-cli-kd`** and `bank_daily_acknowledgment.md` to the outer waiting land
   agent. If this phase has gained incomplete work, note the blocker there and report it
   instead of forcing closure.
5. Finish through `/sase_final`, declaring every changed repository, including the plans
   sidecar and any prepared vault-policy change. No manual Git commit.
