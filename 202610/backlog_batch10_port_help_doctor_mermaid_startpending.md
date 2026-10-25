---
tier: epic
title: 'Backlog batch 10: port-dedupe ports, capture help, ref doctor, Mermaid pdftotext,
  and StartPending recheck'
goal: Close five backlog tasks by keeping explicit non-default ports in URL identity,
  correcting capture help, deciding ref doctor scope, making Mermaid PDF tests pass
  without pdftotext, and rechecking the StartPending flake for closeable non-reproduction.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6
phases:
- id: port-dedupe-ports
  title: Keep explicit non-default ports in URL dedupe keys and display URLs
  size: small
  depends_on: []
  description: 'port-dedupe-ports: resolve bob-cli-c3 by keeping url.port() in dedupe_key_for
    and display_url with unit tests for non-default, spelled-out default, and absent
    ports.'
- id: capture-help-accuracy
  title: Correct capture TEXT help to match the unknown-flag guard
  size: xsmall
  depends_on: []
  description: 'capture-help-accuracy: resolve bob-cli-c4 by wording TEXT help to
    say only the flag where TEXT begins is rejected, with docs-check green.'
- id: ref-doctor-identity
  title: Decide and implement bob ref doctor for id and duplicate checks
  size: medium
  depends_on: []
  description: 'ref-doctor-identity: resolve bob-cli-b8 by making a local implement
    or decline decision first, then flagging id/stem mismatches and duplicate ids
    with live-vault repair or closing as canceled with the decision recorded.'
- id: mermaid-pdf-pdftotext
  title: Make Mermaid PDF tests pass when pdftotext is absent
  size: small
  depends_on: []
  description: 'mermaid-pdf-pdftotext: resolve bob-cli-9i by handling the missing
    pdftotext path in the diagram PDF tests so clean trees pass without the binary.'
- id: mac-startpending-recheck
  title: Recheck the Mac StartPending preview flake for closeable non-reproduction
  size: small
  depends_on: []
  description: 'mac-startpending-recheck: resolve bob-cli-4k by rechecking macOS CI
    history including 461f46c fourth pass and closing as done with evidence if non-reproducing,
    otherwise corroborating the flake.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.land
proposed_with: muse/muse-spark-1.3-contributor
create_time: 2026-10-25 10:04:05
status: wip
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch10_port_help_doctor_mermaid_startpending.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch10_port_help_doctor_mermaid_startpending.md)
- **PARENT:** [202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.md)

# Backlog batch 10: port-dedupe ports, capture help, ref doctor, Mermaid pdftotext, and StartPending recheck

This epic continues the bounded backlog campaign after batch 9 closed `bob-cli-bl` and
epic `bob-cli-an.6.6.6.5.6.5.5.6`. Fresh enumeration shows 126 unresolved tasks. Assign
exactly these five existing task beads, reused never cloned:

- `bob-cli-c3` (small, bug): URL dedupe keys drop explicit non-default ports.
- `bob-cli-c4` (xsmall, bug): capture TEXT help overstates the unknown-flag guard.
- `bob-cli-b8` (medium, feature): ref doctor for id/stem mismatches and duplicates.
- `bob-cli-9i` (small, ci): Mermaid PDF tests fail when pdftotext is absent.
- `bob-cli-4k` (large bead, small recheck phase, flake): StartPending preview timeout,
  fourth consecutive pass on `461f46c`.

## Phase details

### port-dedupe-ports

Baseline in `src/native/highlights_ref/clip_url.rs` (`dedupe_key_for`) and
`src/native/url_routing/intent.rs` (`display_url`). Fix the dead `port_or_known_default`
comparison by keeping `url.port()`, which already strips default ports. Add unit tests
for a non-default port, a spelled-out default port, and no port. Check stored dedupe
keys and existing reference notes for key-shape dependence before changing it. Related
bead `bob-cli-7u` stays green.

### capture-help-accuracy

Baseline in `src/native/capture/cli.rs` (`text_arg` help) and
`src/native/capture/mod.rs` (`find_unknown_long_flag`). Narrow the help to what the
guard enforces: an unknown `--flag` where TEXT would begin is rejected, later tokens
stay literal. Do not widen the guard without its own decision. Read `docs` memory before
editing help and run `just docs-check`.

### ref-doctor-identity

Make a local implement or decline decision first, per the continuation contract. If
implementing, add `bob ref doctor` diagnostics for id/stem mismatches and
case-insensitive duplicate ids, with repair through `bob ref rename` on the live vault.
If declining, close `bob-cli-b8` as canceled with the decision recorded. Never treat the
historical follow-up suggestion as authorization.

### mermaid-pdf-pdftotext

Baseline in `native::highlights_ref::create::tests` diagram PDF tests. Handle the
missing-`pdftotext` path so clean trees pass without the binary (skip with reason, gate,
or text-tolerant assertion that still fails real regressions). Verify with the focused
lib diagram tests.

### mac-startpending-recheck

Recheck `bob-cli-4k` against macOS CI history including run `40195734261` (`461f46c`),
the fourth consecutive pass. If non-reproducing, close as done with the evidence chain.
Otherwise add independent recurrence evidence to the bead and leave it open for a future
batch.

## Acceptance

- `bob-cli-c3`, `bob-cli-c4`, `bob-cli-b8`, `bob-cli-9i`, and `bob-cli-4k` are each
  closed normally (done, or canceled with a recorded decision for `bob-cli-b8` if
  declined) with verification evidence.
- Epic symbols are clean, the batch-10 plan file says `status: done`, and all changed
  repositories are committed via `/sase_git_commit`.
- Batch 11 passes validation and is proposed through `sase plan propose`, with its
  reference recorded on the batch-10 epic bead.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include flakes `bob-cli-5o`, `bob-cli-b9`, `bob-cli-67`, and `bob-cli-2m`;
   small deterministic plugin bugs such as `bob-cli-4e`, `bob-cli-8z`, `bob-cli-9q`, and
   `bob-cli-bn`; and Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x`.
   `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic owns the PTY
   completion drivers. This is guidance, not a preassigned batch, and concurrent work
   may obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.

The successor epic must also carry the working rules batch 9 used. Phase workers leave
target tasks open for the lander and record baseline, fix, tests, and limitations on
their phase. Use `/sase_repo` before touching another repository, and
`sase artifact read` for artifacts. No live vault edits, live feed publishes, or
credential handling. Build bob through `$CARGO_TARGET_DIR`. Follow the daily release
train: no version or `CHANGELOG.md` edits. Run `just check`, never `just check-full`.
The lander must commit every changed repository, including the plans sidecar status
edit, with `/sase_git_commit` before proposing its own successor, and that plan
explicitly authorizes the skill for those obligations.
