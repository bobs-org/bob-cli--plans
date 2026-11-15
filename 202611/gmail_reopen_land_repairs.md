---
tier: tale
size: medium
title: Repair Gmail reopen recovery and task identity, then finish the landing
goal:
  Finish the remaining correctness work in bob-cli-lk.3.1 and close that epic and its
  parent phase bob-cli-lk.3 normally.
proposed_by: bbugyi200.apollo.bob-cli-lk.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-lk.3.1
create_time: 2026-11-14 20:24:13
status: wip
---

- **PARENT:**
  [202611/gmail_closed_reply_reopen.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_closed_reply_reopen.md)
- **BEAD:**
  [bob-cli-lk.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lk/bob-cli-lk.3.1.md)

# Remaining Gmail reopen work

This is the lander's remaining-work tale for `bob-cli-lk.3.1`, whose approved plan is
`plan:202611/gmail_closed_reply_reopen.md`. One coding agent can complete these bounded
repairs in the existing preparation, planner, and transaction seams. Implement them and
finish the landing in this same workstream; no lander resumes after this tale. Do not
create another epic or delegate the closeout.

## Starting evidence and constraints

The audited primary tree is `ff272bc6758d643caebe1fa607fd1ecc14fabb0c`, master equal to
origin/master. The four child phases are closed: `.1` b112a618, `.2` 78f9bdc9, `.3`
88fb9cb4, `.4` ff272bc6. Their source, all notes, linked plan, and parent phase were
read. Basic search, selection, restoration, hold handling, reports, config, and
documentation are implemented. Do not reimplement the feature or claim that their status
proves the uncovered failure boundaries passed. The lander's findings below are
source-audit findings, not runtime reproductions. Start by adding failing synthetic
regression tests for them. Detailed evidence is
`file:explicit:755288a4b63f0c4483ea2fd7`; read it through `sase artifact read` when
needed.

Use `sase bead read bob-cli-lk.3.1 -r "Need current landing notes and descendants"`,
read its children and notes, and
`sase artifact read plan:202611/gmail_closed_reply_reopen.md "Need the approved reopen contract"`
for the complete acceptance matrix. The land audit and follow-up outcomes are recorded
on the epic. Any linked repo must be opened through `/sase_repo`; plan artifacts must be
read through `sase artifact read`.

Inherited final decisions remain: select only the newest effective closed linker;
resurface Ready results with `mark_unblocked` and existing WRITE skips; record the exact
message-card link under direct-child REOPEN LOG; the authorized Gmail Task glossary
correction is already done. Do not add memory edits or decisions. Keep schema 1,
existing turn effects, archive saved-ID epochs and digest v2. Never change the package
version or CHANGELOG. All fixtures use fake gog and temporary vaults; never run against
Bryan's real account or vault. Durable state remains body-free. No remote reads under
the vault maintenance lock.

## 1. Make reopen recovery authoritative before planning

Own `src/native/gmail/sync/{file,command,reopen}.rs` and focused tests.

Current defects:

- `RecoveryReopenTarget.postimage_sha256` is written but never read. `build_attempt`
  receives watch recovery but not reopen recovery, and rejects all non-archivable
  evidence, including an already-open recovered target. A crash after durable task
  writes but before the `pending_tasks=false` checkpoint leaves a receipt that can be
  deferred forever. A restored original done/ locator no longer exists.
- Marker-only receipt IDs are removed from the retry set but may still be selected
  through activity search. If Bryan recloses and changes/removes the log before marker
  replay, the same reply can reopen it again. `WatchJournal` is reloaded under the lock
  but reopen candidates never revalidate the marker before editing.

Pass typed reopen recovery into the under-lock planning/replay path. Resolve each exact
account/thread/message receipt before new candidate work, separating unperformed writes
from an owned durable transaction and marker-only settlement. Consume the recorded
post-restore identity and ownership hashes (extend body-free per-transaction/per-file
evidence when needed to cover mail, restored source/destination, and inbound repairs). A
prewrite intent alone is not proof of successful application; a log alone is not proof
either. Complete the existing scoped commit obligations before the turn marker. Preserve
independently edited/reclosed task bytes and never replay an already-applied reopen.
Recovery of owned writes must not require closed membership, the original done/ row, or
new Gmail hydration. If application is unproven or partial, retain evidence and defer
safely without consuming M; do not silently clear it. Keep existing filing/watch
recovery compatible.

Revalidate the fresh journal marker before applying a new candidate, suppressing
already-consumed M while allowing a genuinely newer reply on the same thread. Resolve
marker-only receipts locally even when the task is open or the log has been edited; do
not blindly suppress every message for a thread that has older recovery. Keep
cursor/retry settlement coupled to durable pending work. A checkpoint failure must not
strand an open task forever or advance past unrecoverable work.

Add deterministic failure-injection tests covering: prewrite receipt failure;
crash/checkpoint failure after live and restored postimages land; commit failure with
ownership retention; marker append failure; reclose plus log edit/deletion before
replay; consumed-marker change between preparation and lock; recovery outside the search
window; state-settlement failure. Assert exact-once marker/log, no second reopen, no
lost repair file, no premature cursor advancement, and byte preservation for user edits.
Reuse existing RecoveryIo/transaction test seams rather than sleeping to induce a race.
Current
`tests/gmail_sync.rs::failed_reply_marker_retries_without_reopening_a_reclosed_task`
only covers an already-checkpointed marker-only receipt with an intact log.

## 2. Keep selected task identities correct across the overlay

Own `src/native/gmail/sync/reopen_plan.rs` and its meaningful regression tests.

`locate_linker` trusts an id-less original line number whenever the current row parses
as a task. Earlier candidates can insert log lines or move blocks, making that number
point at an unrelated task. `located_at` falls back to the old line even when an
expected block ID is missing. Replace these unsafe fallbacks with checked identity
resolution on the evolving overlay: unique block ID, or selected-row identity/verified
relocation evidence for id-less rows. Refuse ambiguous/missing identities as transient;
never treat a different task at the old number as the selected one. Recheck supported
shape and close-date constraints on the resolved row. Update relocations through
edits/restores and re-resolve outcome task locators from final postimages, so a later
insertion above an earlier target cannot leave a wrong report line.

Preserve the existing newest/tie/date/merge-tombstone rules and allow multiple distinct
replies sharing one target. Use the existing archive stable-identity convention where
applicable: outer commit `7ffc8251` added block-ID or row-hash identity, and its
digest-v2 behavior must remain intact. Do not drop duplicate-row multiplicity or weaken
stale-evidence checks.

Tests must include two distinct id-less targets in the same note with unrelated closed
rows between them; insertion and restoration shifts; duplicate identical id-less rows;
disappeared block ID; two threads sharing a target; reverse-path candidate ordering that
inserts above a previously processed target; and done/ restore with collision rename
plus repaired inbound links. Assert no unrelated row changes and final reported
line/block IDs actually identify the touched targets.

## 3. Finish the small wiring defects and integrated acceptance

- In `file.rs::apply_reopen_plan`, derive `today_identities` from the real ledger/Today
  machinery (`plan_budget::today` and freshness scan are existing references), using the
  staged postimage/identity repairs as needed. The current set is every open Gmail
  watcher and cannot represent Today. Preserve NEW, out-of-scope, scheduled-tickler, and
  lane WRITE skips. Add a real command/planning integration vector for a Today-linked
  reopened target and a control that must receive `[unblocked::]`.
- Fill reopened report sender from M's sanitized `candidate.sender`, not the
  pre-hydration mail-index `who`. Keep canonical title/note behavior. Test Ada's saved
  thread receiving Bea's reply: REOPEN LOG and human/JSON reopened row must identify
  Bea; retain bounced and quiet behavior.
- Match complete exact message-card targets in `log_has_message_card`, including the
  delimiter after the ID, rather than prefix `contains`. A history card for message
  `abc1` must not consume message `abc`. Preserve fenced/nested exclusion and existing
  log/newline formatting.
- Remove obsolete preparation-only dead-code allowances from now-wired helpers, and
  delete the unused `plan_prepared_reopens` wrapper if it still has no caller. No
  blanket suppression for unfinished work.

Complete the original acceptance matrix with focused synthetic command-level coverage
for done/ restore and link repair in one transaction, via-merge target without a direct
mail link, archive off and `--no-archive`, `-l 0`, reopening before archive suppression,
and reply after archive releasing the owned receipt via existing `let_go: replied`.
Reuse sufficient existing tests; fill actual gaps, not duplicate every pure test at
every layer. Retain dry-run zero writes and reopen-off parity. Ensure saved IDs stay
frozen before live save writes and archive generations are unchanged.

Integration already inspected: non-epic commits after the first child are bank
docs/guidance `643b3697`/`257fec08` and Areas engine/planner `13fdf78d`/`719dbd45`;
neither needs Gmail feature wiring. Also reviewed concurrent outer archive identity
`7ffc8251` and bank docs `e221958e` since epic creation. bob-plugins at `f68dbdc`
accepts additive watch actions in `291g-plugin-gmail-sync.js` and consumes note/tasks in
`291h-gmail-close-notice.js`; no plugin edit needed. Recheck any later drift before
closure, especially changes in touched sync files.

Run formatting and `just check` after implementation. Do not run `just check-full`. Run
the focused reopen/restore/archive regression suites as needed; broaden only for
justified changes. The unrelated pipe-drain flake is tracked by bob-cli-kl and must not
become Gmail work. Documentation already describes the intended behavior: no user
contract change is planned. If docs need correction, read docs memory first and run docs
generation/check as required; the separate bank link defect must not be misclassified as
this epic's work. Use `/sase_monitor` for long gates, waiting for the staging command
itself to exit and ending through `/sase_final`; continuation of this tale retains all
closeout responsibilities.

## 4. Finish this epic's landing in the implementation turn

Follow-up triage is completed and recorded on bob-cli-lk.3.1. Carry every outcome into
the final close note; do not duplicate those tasks. Phase .4 note #3 was corroborated on
bob-cli-kl (+1); a new duplicate task was declined. Phase .4 note #1's bank link became
small CI task **bob-cli-lo**, related to bob-cli-l7 because the gcal-notes target is
independently fixed. Neither is caused by Gmail. No other child proposed follow-ups
exist. The full active-epic sweep found no causal owner for these unrelated failures;
the bank origin epic bob-cli-lm is closed.

After code and verification are complete, re-read current descendant statuses/notes,
linked plan, and commits since this audit. If the approved tale is represented by an
open descendant plan bead, close that fully implemented and verified tale normally
before closing its containing epic. Do not leave an open descendant and try to force the
epic closed.

1. Run `sase bead epic-symbols bob-cli-lk.3.1`. Resolve every entry by wiring,
   privatizing, permitted non-test pragma, or deleting; re-key only to a still-open
   later bead that demonstrably needs the exemption. The audit found none, but recheck
   now.
2. Close normally with
   `sase bead close bob-cli-lk.3.1 --note "<source/commit/decision verification, repair behavior, integration and test evidence, all follow-up outcomes>"`.
   If rejected for leftovers or uncompleted phases, finish that work and retry. Never
   use `--force` merely to succeed or to advance this successful nested landing.
3. Run `just symvision` if available (the audited justfile has no recipe). Open the
   plans repo through `/sase_repo`, resolve the linked plan path from `sase bead read`,
   and set `status: done` in `202611/gmail_closed_reply_reopen.md`. Include that
   repository's edit in the final declaration.
4. Inspect the parent link again with
   `sase bead read bob-cli-lk.3.1 -r "Need the parent link"`. Parent is phase
   **bob-cli-lk.3**. Read it, verify its reply-reopen scope was completed by this child,
   run `sase bead epic-symbols bob-cli-lk.3` and retire any entries, then close only
   that phase normally with
   `sase bead close bob-cli-lk.3 --note "<verified child implementation and gates>"`.
   Leave containing epic **bob-cli-lk** to its waiting land agent. If the parent is
   incomplete or ambiguous, record the concrete blocker on it and report it; do not
   force closure.
5. Finish through `/sase_final`, declaring every changed repository. The code, epic
   closure, and plan status may land in this same turn. Do not make closure depend on
   this work's future commit SHA, push, or CI result.
