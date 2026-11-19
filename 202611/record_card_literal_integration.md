---
tier: tale
size: medium
title: Finish literal YAML integration and land the Record Card proof epic
goal:
  Stored literal lists agree across record reads, warnings and album grouping, and epic
  bob-cli-ms.8.3.1 plus parent phase bob-cli-ms.8.3 are verified and closed.
proposed_by: bbugyi200.apollo.bob-cli-ms.8.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ms.8.3.1
status: done
---

- **PARENT:**
  [202611/record_card_obsidian_proof.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card_obsidian_proof.md)
- **BEAD:**
  [bob-cli-ms.8.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.8.3.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ms.8.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ms.8.3.1.land.md)
- **COMMITS:**
  - [5b4c263](https://github.com/bobs-org/bob-cli/commit/5b4c26360efe9d90d20b8371ac53a05bef7492b6)
    — fix(records): validate stored literal lists without CLI comma splitting

# Finish literal YAML integration and land the Record Card proof epic

## Remaining work and verified starting point

The land audit of **bob-cli-ms.8.3.1** found one bounded integration defect. Finish it
and this landing in the same tale; there is no later land agent for this tale. Do not
create a child epic or defer closeout until this work's commit, push, SHA, or CI result
exists.

Read the epic and its latest landing/triage notes through `sase bead read`. Read
`plan:202611/record_card_obsidian_proof.md`,
`plan:202611/record_card_title_and_proof.md`, and the evidence below through
`sase artifact read`. Open bob-plugins and plans through `/sase_repo` before accessing
their files. Use the paths returned, never a sibling workspace.

The lander reviewed every child note (7 on bob-cli-ms.8.3.1.1, 22 on .2, 6 on .3), the
approved contracts, commits and source. Both current upstream heads were fetched:
bob-cli **e07a26bf**, bob-plugins **36d4801**. There was no upstream drift. The plugin
acceptance archive has **126 indexed evidence files with matching hashes**, and **all
1,412 source-manifest entries match current committed plugin files**. All 25 reviewed
screenshot hashes match. Its real Obsidian report has **144 passed assertions, zero
skips/failures**, and its non-force deployment/smokes verify all 31 managed hashes. The
landed Properties repair already integrates record-walk commits 281ee8a/e7d82af,
including failed-build fail-open behavior and CodeMirror focus ownership.

Do not redo the title/H1 migration or rebuild the UI. Binding decisions remain
`inline_title = hide`, `decision_record = no`, and `first_cleanup = now` (already
completed). The later explicit selective-Properties requirement stands: hide
title/schema rows represented by the sheet; retain native identity/other rows and Add
property. Preserve Record Card Shift+J/K and Schema Card Ctrl+J/K. No memory edits,
Cargo.toml version edits, or CHANGELOG edits.

## Reproduced integration defect

Evidence: **file:explicit:2031403c6a2f28b860534cfa**; reproducible script
**file:explicit:510e2d5881cc4678f20c4a39** (substitute the newly built binary path,
keeping its scratch vault isolated). This is an independent scratch-vault reproduction
using the accepted e07a26bf binary SHA256
**0c43504dad27a160cc4ce311e090d2c9e591d29f93834f08014699904734b639**. The fixture has an
album with no status field and these fields:

```yaml
category: { list: true, values: { "Science, fiction": Fiction, other: Other } }
rating: { values: number }
```

Its canonical record has `category: ["Science, fiction"]` and `rating: 1`.

1. `bob records show items one -f json` succeeds: category is set, has one literal
   value, display Fiction, and no findings.
2. `bob albums list -f json` incorrectly returns empty groups and `needs_status: 1`.
3. A guarded `records update items one -i - -f json` setting only rating to 2 succeeds
   but emits `record_invalid_value` on the untouched category. The generated album
   callout files the record under Needs a status, despite the returned properties
   displaying Fiction.

Root cause: e07a26bf fixed stored-YAML validation in records
show/check/current-value/Undo paths, but other consumers still apply CLI comma splitting
to values already delimited by YAML. The shared grouping helper feeds album reports,
generated indexes and the newer record-walk ordering, making this integration part of
the epic.

## Implement the narrow repair

1. Add meaningful regressions that reproduce the disagreement before changing code.
   Cover a one-item list-choice containing a comma used as the album grouping field, and
   editing a different field while that list is untouched. The accepted literal value
   must produce its intended group, no false validation warning, and an index consistent
   with the read/update reports. Include guarded apply and Undo boundaries: no unrelated
   record bytes change, the generated callout remains correct, and literal members
   survive. Retain a control proving CLI `-s` comma-separated input still works.
2. In `src/native/albums/grouping.rs::group_value`, validate indexed YAML values with
   the existing `validate_literal_values` helper. In
   `src/native/records/render.rs::collect_warnings`, validate parsed YAML with the same
   literal semantics, removing the unnecessary CLI-input state construction. Inspect
   `src/native/albums/update/ops.rs::count_newly_invalid`, which likewise constructs
   `SuppliedValues` from indexed values; align that stored-value path with
   `SuppliedLiteralValues` without altering authored CLI input semantics. Audit nearby
   production uses of these validators and change only demonstrated stored-YAML
   consumers. Do not introduce another parser or broadly refactor validation.
3. Check the shared grouping's effects on album index and record-walk order with focused
   tests. Preserve schema v1, existing scalar/list constraints, writer guards,
   refusal/Undo semantics, and the intentional Needs a status behavior for actually
   invalid/missing or multi-choice grouping values. This is not the separately tracked
   index_by label-order feature or a schema migration.

## Verification and deployment

Use the existing tests under `tests/cli/records.rs`, album tests and grouping/render
unit modules. Run the new regression cases and relevant record/album/walk tests,
formatting, then **`just check`**. Never run `just check-full`. Use `/sase_monitor` for
long tests, builds and remote work; wait for the staging command to exit, then finish
through `/sase_final` when it reports Staged.

Keep candidate and baseline Cargo **target and intermediate build directories
separate**. A prior phase proved that sharing either can silently substitute the
baseline binary. Record the exact source/base plus diff hash, output directories and
binary hash; the existing Obsidian harness supports a pinned dirty-source provenance
manifest, so verification must not require this tale's future commit SHA.

The existing full-gate exception policy remains narrow: twelve date-sensitive tick
failures are tracked by bob-cli-nf, and plugin full-suite timing/held-lock exceptions
are already documented on the epic. Exact pristine comparisons and fixed-date controls
are in **file:explicit:1989925b303385554af4b7ee**; final acceptance is
**file:explicit:6c29e2ed7ad7cd9f4e9a88db**. Inspect new gate results; fix all failures
caused by this repair. Never relabel a nonzero gate as green, fix an unrelated known-red
just to close, or substitute a control for `just check`.

Build the repaired Bob and repeat the scratch reproduction, requiring correct
read/group/index/warning results and byte-safe Undo. Refresh upstream and integrate any
new overlapping changes before final verification. Run the existing full real-Obsidian
Record Card harness against that exact binary in an owned isolated Athena scratch
vault/profile; the existing plugin source can remain unchanged. Reuse the documented
harness and provenance mechanisms, not old screenshots as fresh proof. Inspect every
newly generated screenshot and publish exact reports, commands, hashes and exceptions.
Read tailnet memory before SSH. If plugin files require an evidenced repair, obey the
linked AGENTS.md, rebuild fragments, run its required gates, and deploy with explicit
opened repo and --no-pull, never force. Otherwise preserve and confirm the accepted
plugin deployment hashes. Make the repaired Bob available on Apollo through the
repository's supported installation workflow without changing its release version;
report the Mac install/checklist separately. Verify `record_values_v1` remains
available.

The final live coverage audit has 193 records/1,989 authored keys: 192 reads and 1,982
keys pass; one unchanged parent mismatch leaves seven uncovered. The data repair is now
**bob-cli-ng**. This documented baseline rejection does not require an unrelated
live-vault mutation here. Keep it explicit, not allowlisted or called fully covered.

## Follow-up triage is already complete

The lander recorded these outcomes on bob-cli-ms.8.3.1 before proposing this tale:

- Phase .1 note 4 and phase .2 note 11: corroborated existing **bob-cli-lp** with +1 and
  file:explicit:ada6f67bcfefb9e4ca2f7bdb.
- Phase .1 note 4 lint proposal: declined as already fixed by **bob-cli-nb.1 /
  bb864f88**; current `albums/report.rs` uses `then_some`, and later exact-source Clippy
  completed.
- Phase .2 note 18 and phase .3 note 3: corroborated existing **bob-cli-nf** with +1 and
  file:explicit:1989925b303385554af4b7ee; both pristine/candidate failure bodies and
  19/19 fixed-date controls agree.
- Phase .3 note 2: filed **bob-cli-ng**, small, for the single existing parent-link data
  defect, with file:explicit:925b70926af44650172dea5b. It is separate from the epic's
  implementation, with no identified duplicate or causal active epic after the required
  searches/sweeps.

Do not file duplicate tasks. New unrelated discoveries use `/sase_new_task`; anything
caused by this repair remains this tale's work. Preserve the known timing/held-lock
trackers and deferred features without re-reporting them merely for recurring.

## Final step: finish this epic and its parent phase

This step is part of this tale's own work, performed after verification in the same turn
as its final code, **before host finalizers create the commit**. There is no gate on
this work's SHA, push or CI. Do not wait for a separate lander to resume.

1. Re-read **bob-cli-ms.8.3.1** and its three children, confirming no new
   notes/descendants invalidate the reviewed evidence. Check latest upstream drift and
   the linked plan readiness. Carry the verification, preserved exceptions and every
   follow-up outcome into the close note.
2. Run `sase bead epic-symbols bob-cli-ms.8.3.1`. Resolve every entry (wire, privatize,
   valid non-test pragma or delete) or re-key only to a still-open later bead that needs
   it. The land audit found zero entries, but rerun at closeout.
3. Ensure this tale's own bead is complete and close it normally first if it is an open
   descendant that would block its parent. Then close the epic normally with
   `sase bead close bob-cli-ms.8.3.1 --note "<source/integration verification, new literal-value regressions, UI/deploy evidence, gate exceptions, and lp/nf/ng/declined-lint triage>"`.
   Never use `--force` to advance a successful nested landing. If rejected for remaining
   descendants/symbols, finish them and retry normally; do not change resolution merely
   to bypass the guard.
4. Run `just symvision` after closing if available. At the land audit `just --list` had
   no symvision recipe, so report unavailable honestly if still absent. Set
   **`status: done`** in **`202611/record_card_obsidian_proof.md` within the opened
   plans repository**, the PLAN file for **plan:202611/record_card_obsidian_proof.md**.
   Do not mark the containing parent epic's plan done.
5. Run `sase bead read bob-cli-ms.8.3.1 -r "Need the parent link"` and re-read
   **bob-cli-ms.8.3**. It is a **phase**, whose required work is the real Obsidian
   walkthrough, DOM-backed Properties repair and read-only coverage evidence. Confirm
   this child plan and the repaired integration satisfy that scope. Run
   `sase bead epic-symbols bob-cli-ms.8.3` and resolve/re-key any leftovers. Close only
   that phase normally with
   `sase bead close bob-cli-ms.8.3 --note "<verified child epic completion, coverage attribution, final UI/deployment evidence and integration repair>"`.
   Leave **bob-cli-ms.8** and **bob-cli-ms** open for their already-waiting land agents.
   If new scope or ambiguity prevents closing the phase, record the concrete blocker
   there and report it.
6. Mark this tale's plan done as appropriate and finish through `/sase_final`, declaring
   every modified repository. Final user report states the closed beads or exact
   remaining blocker, the honest gate/coverage limitations, and the Mac reminder: run
   **`just install-all`**, then try `==` on bead_lifecycle, a status digit pick, a claim
   edit and Undo, and inspect the sheet title. Do not claim the human Mac check was
   performed.
