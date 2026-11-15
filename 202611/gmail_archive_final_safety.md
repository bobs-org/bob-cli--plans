---
tier: tale
title: Finish Gmail archive merge traversal and confirmation identity
goal:
  Preserve every effective merge target, stabilize confirmations and journal
  compatibility, and close bob-cli-lk with verified evidence.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-lk.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-lk
status: done
---

- **PARENT:**
  [202611/gmail_archive_merge_and_reply_reopen.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_merge_and_reply_reopen.md)
- **BEAD:**
  [bob-cli-lk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lk/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-lk.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-lk.land.md)
- **COMMITS:**
  - [49231b3](https://github.com/bobs-org/bob-cli/commit/49231b374a483d53bbc33018836577d940d0aa0c)
    — fix(gmail): keep branching merge terminals and line-shift-proof confirms

# Finish Gmail archive merge traversal and confirmation identity

## Objective and landing responsibility

Complete the remaining work in **bob-cli-lk**, then close that epic in this coding turn.
This is one medium tale, not another phased epic. There is no later land agent for this
tale. Its final step owns the epic close, whitelist check, and plan status update; never
defer them until this work has a commit SHA, push, or CI result.

The approved epic is `plan:202611/gmail_archive_merge_and_reply_reopen.md`. The linked
plan file is `202611/gmail_archive_merge_and_reply_reopen.md` in the `plans` sidecar;
open that repository with `/sase_repo` before editing. The audited epic has no parent
bead. Re-read its parent link before final closure in case it changed.

## Audit already completed

The lander read bob-cli-lk, every note on lk.1/lk.2/lk.3, the closed nested epic lk.3.1
and all four phases and their notes, both approved plan artifacts, all seven
implementation commits, and current source. The integrated tip is `92f5d1c8`, fetched
and fast-forwarded after the nested lander's finalizer committed its repairs.
Implementation commits: `47fac4b0`, `7ffc8251`, `b112a618`, `78f9bdc9`, `88fb9cb4`,
`ff272bc6`, and `92f5d1c8`.

The implemented decisions stay final: newest effective closed linker; `mark_unblocked`
with WRITE skips; exact-message REOPEN LOG; only the authorized Gmail Task glossary
correction. Recovery now consults exact pre/postimage receipts, rechecks fresh turn
markers, settles markers after durable writes, uses ledger Today membership, and reports
the triggering sender. Do not redo that completed work.

Integration review covered `9574e618` (landed after epic creation, before its first
commit), then non-epic `b3dc64a0`, `e221958e`, `643b3697`, `257fec08`, `13fdf78d`,
`719dbd45`, and `92813d08`. The archive epoch fix is preserved; Areas catalogs use the
shared structural-section machinery and need no Gmail-specific wiring; Bank reporting
has no Gmail consumer. Shared JSON reference edits coexist, except for the small wording
correction below. bob-plugins `f68dbdc` was opened and inspected:
`291g-plugin-gmail-sync.js` validates the schema/ok envelope without enumerating
actions, and `291h-gmail-close-notice.js` matches task locators without an action
filter. No plugin edit or deployment is required by this tale.

All remaining defects below are in this epic's acceptance scope. They are not new task
beads. Synthetic CLI probes on `92f5d1c8` reproduced all three: branching target loss,
line-shift confirmation mismatch, and legacy checksum rejection on actual id-less
open/close events. No production source was edited for these probes. Recheck post-audit
commits for relevant drift before editing.

Baseline `sase tool run check` (the `just check` command) passed at `92f5d1c8`: ToolRun
`437730ce997247c921fa316472fa3e92`, exit 0, 7,044 library tests and 3,662 CLI tests plus
every remaining suite. Three library tests were ignored by the existing suite. These
uncovered paths are absent from that coverage. The lander did not rerun docs
generation/check or `check-full`; the nested landing's successful docs gates remain
attributed evidence until this tale reruns the required gates.

The reproducible probe script is `file:explicit:7c9a1b8c8dc0ec3c318feaa6` and its
results are `file:explicit:b03c85385f54e56455009b2e`; consume them with
`sase artifact read <ref> "Need landing regression evidence"`. The script asserts the
current defects for diagnosis; convert these into corrected-behavior Rust regressions,
rather than committing tests that pin the bugs.

## 1. Preserve every terminal target in a branching merge chain

In `src/native/gmail/sync/archive_scan.rs`, `effective_linkers` visits every direct
merge entry, but recursive `follow_merge` returns only one `ArchiveLinkTask`. Both
recursive branches repeatedly assign `last = Some(follow_merge(...))` and discard every
terminal except the last. Multiple merge relations are a supported input: relation
gestures on an already-closed source write another relation, and the scanner explicitly
collects all entries.

Minimal synthetic shape (only S links the saved mail note):

```markdown
- [-] #task S [[mail/2026/lease]] ^src
  - 🪢 **LINKS:** merged into [[job#^mid]]
- [-] #task Mid ^mid
  - 🪢 **LINKS:** merged into [[job#^open]], merged into [[job#^closed]]
- [ ] #task Open ^open
- [x] #task Closed ^closed
```

The current recursion drops `open`, making the thread archivable and eligible for
closed-thread reply reopening even though an effective linker is open. Swapping entry
order changes the outcome.

Refactor the traversal to collect every terminal effective linker, deduplicated by the
existing `(path, line)` identity, while retaining shared relation resolution, path-local
cycle detection, the eight-hop cap, section tombstones, and typed unresolved holds. A
shared terminal reached through two branches is not a cycle. Unresolved branches must
remain fail-closed even when another branch resolves. Handle intermediates both present
and absent in the original direct-mail linker set. Do not solve this by arbitrarily
choosing one relation or rejecting ordinary branching graphs.

Add meaningful regression coverage for:

- open plus closed terminals in either relation order: both survive, Watching, not
  archivable, not in `eligible_closed_threads`;
- two distinct closed terminals and diamond deduplication;
- one unresolved or cyclic branch beside a resolved branch;
- existing single-chain, section tombstone, custom label, done/, and depth-cap behavior
  staying intact;
- the full eligible set feeding newest-linker reply selection and digest evidence.

## 2. Remove indirect line-number dependence from confirmation

Phase lk.2 fixed `DigestTask` and `fresh_matches_close`, but
`archive_state.rs::close_from_evidence` still hashes `task_values(&tasks)`, including
each locator's `line`, into the close generation. `ArchiveJournal::project` computes
that generation for a newly closed task with a prior open observation, and
`archive_preview.rs::plan_digest` binds it. A read-only preview does not persist the
close. Shifting lines before confirmation therefore changes the generation and digest
even with a block ID. Backlog previews and already-journaled closes do not exercise this
path, which is why the existing phase tests missed it.

Make the semantic confirmation identity independent of report/history line locations
throughout this projection-to-live path. Preserve generation boundaries, one-shot
consumption, terminal receipt ownership, frozen saved-message sets, and old journal
parsing. Use one canonical task-identity representation where practical: path and block
ID, otherwise row hash with multiplicity/occurrence, plus status. Keep line numbers in
stored history and report locators. Existing historical generations/receipts must remain
readable and consumed; do not rewrite journals or remove generation protection from the
digest just to make the test pass.

There is a related mixed-version hole: lk.2 added `row_hash` to `task_values`, which
also feeds persisted open-observation IDs and close-generation hashes. The pre-lk.2
reader (`7ffc8251^`) ignores unknown locator fields, then **recomputes** those hashes
using `task_values` without `row_hash`; it therefore rejects new id-less observations
and closes. Ignoring an unknown JSON key alone does not prove compatibility. The
approved epic explicitly required old readers to accept the new events. Preserve the old
journal hash contract when adding richer task identity (or use the approved unhashed,
optional metadata placement), and keep this already-emitted interim event format
readable too. Prefer a separate semantic confirmation witness over changing the
persisted generation format. Keep that witness bound to the lifecycle/epoch facts that
distinguish reopen/reclose generations.

Cover at least:

1. Record an open observation, close the task without running a live sync, preview with
   `-d`, insert unrelated lines above its task, then confirm the original digest with
   live `-c`. It must confirm and archive. Use synthetic fake gog; a burst-limit gate
   can force confirmation for fresh closes.
2. The same flow with an id-less task and duplicate identical id-less rows.
3. Status flips, task addition/removal, merge endpoint changes, and saved-ID changes
   still invalidate the confirmation.
4. Legacy locators and existing close-generation/receipt replay, reopen/reclose,
   receipt-label ID resolution (`9574e618`), backlog confirmation, and journaled close
   line shifts remain valid.
5. Feed new id-less open and closed events through the actual pre-lk.2 parsing/hash
   semantics and prove acceptance, while current readers accept legacy and the interim
   row-hash-bound events. Include an id-bearing control. Do not claim compatibility
   merely from a locator parser unit test.

Read and update task **bob-cli-ll** with the verified resolution; it tracks this same
defect and was only partially fixed by lk.2. Close it normally only after its acceptance
is covered, and append a correction/completion note to **bob-cli-kx** so its historical
digest-fix report includes this final path.

## 3. Finish verification and the shared documentation correction

The Gmail row in `docs/reference/json-outputs.md` currently says `recent_archive` counts
"since local midnight"; its established contract is **since previous local midnight**,
as the producer, tests, and detailed Gmail docs specify. Restore that qualification
while preserving the Bank/Areas additions and the reference word budget. Read `docs.md`
and `cli_rules.md` through `/sase_memory_read` before changing these user-visible
contracts. No memory edit, new CLI option, or version change is needed.

Run focused regressions first, then `just check`, `just docs-gen`, and
`just docs-check`. Do not run `just check-full`. Use `/sase_monitor` if a named run
escalates or a long verification needs handoff; a passing file-change gate is
`just check`. Any unrelated infrastructure issue follows `/sase_new_task`, not a new
requirement for the Gmail feature. Never exercise the real Gmail account or vault:
synthetic fixtures only. Every remote read remains readonly; this tale adds no Gmail
mutation. No sender/subject/body/task prose enters persisted journals or recovery state.
Preserve inert rendering and JSON schema 1.

## 4. Close bob-cli-lk in this same turn

First re-read bob-cli-lk and its descendants, checking that all named phases and the
linked nested plan are done and that this tale resolved all three reproduced defects.
Read any new notes and relevant post-audit commits. Do not wait for this tale's own SHA,
push, or CI to close the epic.

Follow-up triage is already complete and recorded in the epic's LAND TRIAGE note:

- lk.1 #2: duplicate bob-cli-it; +1 recorded (6 reports), no new task.
- lk.1 #3: duplicate bob-cli-g8; +1 recorded (6 reports), no new task.
- lk.1 #4: duplicate bob-cli-f2; +1 recorded (3 reports), no new task.
- lk.3.1.4 #1: bob-cli-lo was created by the nested lander, fixed in `92f5d1c8`,
  docs-checked and closed. Do not file it again.
- lk.3.1.4 #3: the nested lander already corroborated bob-cli-kl (+1, 12 reports).
  Decline another task or repeat +1 for that same observation. `92813d08` later added a
  spawn delay; kl remains the independent flake tracker.

Repeat these outcomes and why duplicates were declined in the close note, together with
verified implementation, inherited decisions, audit/integration commits, repair
regressions, final gates, and bob-cli-ll's disposition.

Run `sase bead epic-symbols bob-cli-lk`. Resolve every listed exemption by wiring,
privatizing, a justified non-test pragma, or deletion; re-key only when a still-open
later bead genuinely needs it. The last audit found no entries. Then close normally:

```text
sase bead close bob-cli-lk --note "<the concrete verification and triage above>"
```

If closure reports leftover symbols, fix them and retry. If named phases remain
unfinished, complete or reopen them. Never force a successful nested landing, and never
use `--force` merely to make the command succeed. Cancellation/supersession requires a
deliberate non-success resolution and reason.

After close, run `just symvision` if the recipe is available (the audited justfile has
no symvision recipe). Through the opened `plans` sidecar, set `status: done` in
`202611/gmail_archive_merge_and_reply_reopen.md` frontmatter. Re-read
`sase bead read bob-cli-lk -r "Need the parent link"`; the audited parent is absent, so
finish normally. If an unexpected parent link now exists, stop ancestor closure, record
the changed linkage and blocker, and report it rather than guessing.

Finish through `/sase_final` with commit decisions for every changed repository. This
tale has no separate lander; do not leave bob-cli-lk or its plan status for a future
agent after successful verification.
