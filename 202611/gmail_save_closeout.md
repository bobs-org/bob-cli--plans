---
tier: tale
title: Repair Gmail save transactions and finish the nested landing
goal:
  Complete save recovery, image persistence, identity parity, and structural previews,
  then close bob-cli-hg.6.1 and parent phase bob-cli-hg.6.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-hg.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-hg.6.1
create_time: 2026-11-06 05:28:43
status: wip
---

- **PARENT:**
  [202611/gmail_save_list.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_save_list.md)
- **BEAD:**
  [bob-cli-hg.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hg/bob-cli-hg.6.1.md)

# Repair Gmail save transaction gaps and finish the nested landing

## Outcome and ownership

Finish the remaining work found while landing **bob-cli-hg.6.1**, then close that epic
and its parent phase **bob-cli-hg.6** in this coding turn. This is one bounded repair of
the Gmail save transaction, its preview, and its acceptance tests. The four original
phases are already closed; do not restart their implementation. The containing epic
**bob-cli-hg** belongs to its waiting land agent and must stay open. This tale has no
land agent or later landing continuation.

Read `sase bead read bob-cli-hg.6.1 -r "Need landing findings and follow-up triage"`,
its current notes, and the approved artifact `plan:202611/gmail_save_list.md`. The
linked epic plan file is `202611/gmail_save_list.md` inside the plans repository opened
through `/sase_repo` (`sase repo open plans -r "Complete Gmail epic plan"`). Read that
artifact through `sase artifact read`, not direct filesystem reads.

Keep the accepted decisions: default full body, default subject-and-who link label, and
**no SASE memory edits**. The only authorized outer memory change belongs to another
phase. Gmail stays read-only, network calls finish before the vault lock, all tests use
synthetic mail and isolated temporary vaults, and no real mailbox or vault is used.
Never edit versions or CHANGELOG.md.

## Landing evidence and remaining defects

The lander reviewed all four children and every note plus commits `cdc40030`,
`c0d877be`, `b70a8fbf`, and `fcc4efa5`, their current source, and the outer plan's
sections 3, 4.1–4.8, and 5.6. The separate commits since the first child commit are
`6af313b4`, `248a2bfa`, and `5c119a06`: capture merge acceptance, shared merge
eligibility, and successor verdict repairs. They do not change Gmail's staged
writer/lock/commit APIs and need no Gmail-specific integration. `5c119a06` repairs the
capture verdict failures reported by phase .1. Recheck newer drift at startup.

1. **Recovery loses image provenance.** In `src/native/gmail/save.rs::run_inner`, retry
   notes that are byte-unchanged but dirty are committed first. Only afterward
   `git_head_text` reads HEAD and `newly_referenced_origins` compares with it. HEAD
   already includes every new embed, so no attachment origin is emitted. The current
   failed-commit test contains no image and checks only the Gmail journal. The retry
   also reports `created: false, added: 0` in that journal after a failed first save,
   rather than the creation that just became durable.
2. **Imports are based on the fresh rendering, not the merged note.**
   `plan_prepared_saves` keeps all `PreparedThread.imports` even when append-only merge
   preserves an old card and does not use those new embeds. For example, save an
   image-bearing message with `-B headers`, then save it with `-B full` without `-R`:
   old cards remain headers-only, but the image is installed with a successful
   "unchanged" result although no saved card references it. Destination selection must
   also reserve other imports in the same batch, not only existing files.
3. **Account equality drifts across layers.** The resolver/index compare accounts
   ignoring ASCII case, but `note::merge::merge` compares them byte-for-byte. An
   existing note saved with `you@example.com` is found but refused after the configured
   account becomes `YOU@example.com`. Preserve its identity and path; case changes must
   not create another note or require rerender.
4. **The promised structural diff is absent.** `print_report` calls a one-line status a
   structural preview, but neither human nor JSON dry-run output contains the planned
   structural changes, including for `-R`. The approved child plan requires a reviewable
   diff with message prose and snippets redacted.
5. **Transaction acceptance is incomplete.** `tests/gmail_shell.rs` currently has only
   six save tests including the failed-commit recovery test. Preparation unit tests do
   not exercise the real save orchestration for images, mixed batches, denied
   writers/policy, or deterministic preimage/write races. Add direct coverage of these
   connected boundaries while fixing the above defects.

All four behavioral defects above were reproduced against the unchanged fcc4efa5 binary
with isolated temporary vaults and a synthetic fake-gog thread. The portable reproducer
is `file:explicit:e7aea0b4b642b917e1e2d9f7`; read it through `sase artifact read` and
use a local build of bob, never the real mailbox. Its observed results are
`file:explicit:18705a7c72a5c9d067cf6be1`.

## Implementation

1. Extend the existing fake gog/harness with deterministic synthetic thread state
   (including new/deleted messages and attachment-backed HTML with CID images). Reuse
   the narrowly allowed Discovery read and prohibit the cache-writing Gmail attachment
   command. Add failing regressions for the defects above before adjusting the
   transaction. Do not introduce a general mocking framework.

2. In the locked planner, derive retained imports and origin descriptors from the final
   managed cards actually persisted by the append-only merge. Preserve old cards and
   image bytes, including cards absent from Gmail. Omit new imports for discarded fresh
   versions of old cards; explicit `-R` may replace those cards. Reserve destinations
   across the whole batch and retries, verifying hashes and retaining safe,
   deterministic collision handling. Do not overwrite a different existing asset. Keep
   proposed bytes in memory and retain the staged-save API that later link/pull phases
   can combine with their own edits in one commit.

3. Capture the relevant pre-commit HEAD/preimages and recovery event facts before
   `ob::commit_paths`. Derive pending origins from those preimages and the final
   persisted note, then append only after a successful scoped commit (or durable non-Git
   write). A failed commit emits no success journal; its retry emits the missing Gmail
   and attachment events exactly once, with meaningful created and added counts, and a
   subsequent no-op emits none. No new body cache, persistent transaction framework, or
   blanket Git staging. Preserve unrelated staged and unstaged paths. Compare normalized
   account identity consistently in the merge boundary while preserving existing
   authored metadata and path.

4. Add a safe structural-preview value derived from the same planned text/files as
   execution. Human `-d` and `-d -R` show actual additions/removals or before/after
   structural rows: note paths, owned metadata changes, message/card identifiers,
   folding changes, and image paths/counts where relevant. Redact all message prose and
   snippets _before_ producing a diff; never diff raw note bodies into diagnostics. JSON
   may grow an additive preview field and must keep its existing schema, results, and
   single-input note/link fields. `-q` still preserves requested JSON. A true no-op
   preview must say so. Dry-run creates no lock, notes, images, hubs, warning, journals,
   commits, or persistent caches.

5. Complete focused acceptance tests around the real orchestrator:
   - image save → failed commit → successful retry → byte-identical no-op, with exact
     event counts, origins, hashes and no body/snippet sentinels outside notes;
   - headers-to-full without rerender versus explicit rerender, append after a deleted
     historical message, same-name/different-byte and duplicate-hash images, no orphan
     imports and no note referencing a missing newly imported asset;
   - repeated URL/ID/note aliases coalesce to one identity while preserving result
     association; a later refused input in a mixed batch leaves all vault bytes alone;
   - writer mismatch exits 2 without writes, dry-run on a non-writer is allowed,
     never_save address/domain/label refusals write nothing, and `-B headers` performs
     zero attachment downloads;
   - one deterministic `NoteChanged` replans successfully and a second race refuses; a
     text-apply failure cleans only unchanged images created by that attempt, preserving
     existing assets/concurrent edits; no Gmail fetch occurs while locked;
   - scoped commits preserve an unrelated staged path as well as an unstaged path;
   - human and JSON previews are structural, useful for rerender, and body-free;
   - same-account case variation refreshes the existing note successfully.

   Prefer small test-only seams around the writer/commit boundary if necessary; do not
   use timing sleeps or add public fault-injection options. Retain existing resolver,
   renderer, list and doctor tests rather than duplicating their corpora. Remove
   obsolete broad dead-code allowances only when consumers now justify it; keep future
   outer-phase APIs with clear ownership.

6. Integrate the root help snapshot correction caused by outer phase hg.4: actual root
   help includes Gmail while `tests/fixtures/help/root-short.txt` and `root-long.txt`
   omit it. Read `docs.md` and `cli_rules.md` through the memory skill before changing
   help/docs. Refresh only the intended expectations from current CLI output. Update
   `docs/gmail.md` for the actual structural preview and recovery semantics, regenerate
   CLI references with `just docs-gen`, and avoid unrelated generated churn. No new CLI
   flags are needed.

## Verification and previously triaged work

Baseline verification on fcc4efa5: `just check` run `dffe775a90af653215b67f14325a396b`
exited 101; formatting and lint passed, lib 5371 passed with only the two clipboard
failures (2 ignored), CLI 3262 passed with only root-help drift, Gmail shell 24/24
passed, and other binaries passed. Catalog protocol and all successor-verdict tests now
pass. `just docs-check` failed only the known README digest assertion (446 passed, 1
skipped); all earlier documentation gates passed.

Run focused Gmail tests, then **`just check`** and **`just docs-check`** on the final
integrated tree. Never run `just check-full`. Use `/sase_monitor` only when needed for
long verification, with a continuation that performs this tale's remaining closeout; do
not make closeout depend on this work's own SHA, push, or CI run.

The lander triages every original PROPOSED FOLLOW-UP in a note on bob-cli-hg.6.1 before
proposing this tale. Carry those exact outcomes into the close note; do not file
duplicate tasks. Clipboard registration failures belong to bob-cli-hp; the README
contract-table digest belongs to bob-cli-hs. Neither is caused by this epic, and either
may remain red if independently confirmed unchanged. The old catalog 73-vs-72 assertion
is updated in the save/list commits; the old successor verdict assertions are repaired
by 5c119a06. The root-help drift belongs to the outer Gmail epic and is included in step
6 above. Fix any new failure caused by this tale, and record unrelated unchanged
failures accurately without calling the overall gate green.

## Final step: finish this landing in the same turn

After repairs and verification, reread bob-cli-hg.6.1 and all its children/notes and
confirm all promised remaining work above is complete. Recheck code/history drift since
the lander reviewed fcc4efa5. Include the reviewed commits, preserved DECISIONS,
repairs, actual gate results, and every original follow-up disposition in the close
note.

1. Run `sase bead epic-symbols bob-cli-hg.6.1`. Resolve each listed allowance by wiring,
   privatizing, adding a justified non-test pragma, or deleting it. Re-key only when a
   concrete still-open later bead owns the unused API. No stale entry may remain; the
   initial landing audit found none.
2. Close normally with
   `sase bead close bob-cli-hg.6.1 --note "<verification and follow-up outcomes>"`. If
   rejected for stale symbols, clean them and retry. If rejected for incomplete
   descendants, finish them. Never use `--force` merely to make the command succeed, and
   never use it to advance a successful nested landing. Any deliberate
   cancellation/supersession needs its own justified resolution, not a claim of
   completion.
3. Run `just symvision` if the recipe exists (it did not at the initial audit). If
   unavailable, record that fact, retaining the empty epic-symbols evidence. Set
   `status: done` in the frontmatter of the epic's linked plan
   `202611/gmail_save_list.md` in the plans repository opened through `/sase_repo`.
4. Run `sase bead read bob-cli-hg.6.1 -r "Need the parent link"` and confirm the parent
   is phase **bob-cli-hg.6**. Read that phase and verify its complete save/list scope is
   now met. Run `sase bead epic-symbols bob-cli-hg.6` and resolve or correctly re-key
   any entries. Close only the parent phase normally:
   `sase bead close bob-cli-hg.6 --note "<confirmed save/list contracts and results>"`.
   Leave **bob-cli-hg** and its `202611/bob_gmail.md` plan open for its waiting land
   agent. Do not use force. If the parent is incomplete or ambiguous, record the
   concrete blocker on it and report it rather than pretending it is complete.
5. Finish through `/sase_final`, declaring every changed repository including the plans
   sidecar. Closing the epic and parent in this turn before the host commits the final
   code is intentional and must not be deferred to a future agent.
