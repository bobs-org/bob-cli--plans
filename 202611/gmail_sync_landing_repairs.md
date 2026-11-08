---
tier: tale
title: Repair Gmail sync landing gaps and close bob-cli-ii.3.1
goal:
  Complete the remaining sync integration and recovery work, verify it, and close the
  child epic and parent sync phase normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ii.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ii.3.1
status: done
---

- **PARENT:**
  [202611/gmail_sync_filing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_sync_filing.md)
- **BEAD:**
  [bob-cli-ii.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ii/bob-cli-ii.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ii.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ii.3.1.land.md)
- **COMMITS:**
  - [f904135](https://github.com/bobs-org/bob-cli/commit/f9041354c191eb8bb781abc000821675f9bb7cdb)
    — fix(gmail): repair sync landing gaps for filing, coordination, reports, and
    recovery

# Repair Gmail sync's remaining landing gaps and close bob-cli-ii.3.1

## Scope and authority

Finish the bounded integration defects found while landing `bob-cli-ii.3.1`, whose
parent is **phase `bob-cli-ii.3`**, then complete that landing in this tale. This is one
coding agent's work in the existing sync, save, test, and documentation boundaries. It
needs no new product decision or independently scheduled phase.

Read the landing and follow-up-triage notes with
`sase bead read bob-cli-ii.3.1 -r "Need the landing findings and triage outcomes"`. Read
its original plan with
`sase artifact read plan:202611/gmail_sync_filing.md "Need the remaining sync acceptance contract"`
and the governing outer contract with
`sase artifact read plan:202611/gmail_sync_reply_watch.md "Need inherited sync and reporting requirements"`.
The original epic plan to mark done at closeout is
**`plan:202611/gmail_sync_filing.md`**, not the outer reply-watch plan.

The original four phases are closed. Their implementation commits are `343b4d8f`
(membership), `39122cb2` (discovery), `d5c61ee5` (filing), and `5df6b9d5` (command). At
landing review, HEAD and freshly fetched `origin/master` were `5df6b9d5`; there were no
non-epic commits after the first of those four. Sibling turn-metadata commit `47341462`
landed between epic creation and its first child commit. Sync already uses its shared
note renderer/merge and mail.base upgrade: preserve that integration and its tests.
Check new drift again before editing and before closing.

The reviewer decisions are final: `cron_enabled=no`, `gmail_watch_record=no`,
`reply_hold_record=yes` (owned by the outer watch phase), and `inbox_glossary=yes`
(already implemented). This tale does not edit memory or implement the outer watch,
health, plugin, or cron phases. Keep Gmail read-only, tests offline, the release version
and changelog untouched, and `pull` as the same permanent hidden alias. Read `docs.md`
and `cli_rules.md` through `/sase_memory_read` before their corresponding edits.

## 1. Select only real new filings before applying the limit

`sync/command.rs` builds `Membership` and local adoption intents, but calls
`discover::select_candidates` with only the enrollment fold.
`sync/discover.rs::select_candidates` therefore spends `-l` slots on unenrolled threads
that already have local watching tasks. They are fetched as candidates before `file.rs`
finally adopts them.

Use the membership decision (or the already computed adoption keys) before the limit.
Enrolled and adoptable keys must not consume slots or receive new-candidate
metadata/body fetches. Preserve stable discovery order, label-free local adoption,
per-account identities, and `-l 0` behavior.

Regression: save/link thread A without an enrollment event, return A then new thread B
from discovery, and run `-l 1`. A is adopted locally and B is filed. Assert A receives
no candidate fetch, only one new row is counted, and replay creates neither another row
nor another adoption event. Repeat adoption with A absent from search and with `-l 0`.

## 2. Finish bounded request coordination and failure propagation

`discover_ids` invokes `search_page` directly, without the retry or pacing helpers.
`pace_before_request` can refuse the second healthy request at `-r 0` if the first
completed within 200 ms. Metadata exhaustion returns only `deferred`/warnings; command
orchestration ignores `deferred` and can report success after a real transient failure.
These are integration gaps between the existing helpers, not a request for another
transport.

Carry one request coordinator through paged search, label enumeration, candidate
metadata, hydration, policy metadata, and attachment reads. Reuse the shared read
interface and retry classification. Apply ordinary request pacing across stage
boundaries and bounded backoff to transient failures; avoid retrying a whole
body-preparation batch without pacing its underlying reads. Keep a single shared
remaining budget for guard contention, transient retries, and the vault lock. `-r 0`
gives each needed operation one attempt with no contention/backoff sleep; it must not
silently cap a healthy multi-thread run at one thread. Normal request pacing must not be
treated as a failed operation merely because the retry budget is zero.

Preserve successfully prepared candidates after per-thread failures or exhaustion, defer
remaining work, and propagate a body-free failure state to the report. A revoked/expired
grant discovered anywhere in preparation still aborts before vault writes. No network
operation may occur under the vault maintenance lock. Do not let a search-page failure
discard earlier discovered usable work unless it is a global authentication abort.

Use an injectable clock as well as a sleeper where timing governs the decision; the
current recording sleeper does not advance `Instant`, so it cannot reliably test
exhaustion. Pin transient search retries, pacing across stages, healthy two-thread
`-r 0`, partial successes, exhaustion, global authentication abort, and shared lock-wait
accounting. Keep paging, deduplication, repeated-token, and safety-cap tests; warn about
more results only when the cap actually leaves results outstanding.

## 3. Report durable outcomes and actionable failures

In `sync/report.rs`, both `pulled` and `watching_count` currently count any result with
status `problem`, including a failed metadata/body fetch with no saved note or task.
`watching_count` also seeds from the original scan, so a task closed during preparation
can remain in the summary. The `FilingFailure.kind/message` fields never reach warnings,
and human route rows omit the resolver's label, message, and hint.

Compute counts from typed successful/projected filings, adoption, and the current
enrolled watch membership, rather than inferring success from the public status string.
Route-problem rows that were actually filed still count; fetch/planning failures do not.
Use the revalidated apply snapshot for live membership and equivalent projected
membership for dry-run. Do not count a failed or deferred adoption as a newly enrolled
watch. Preserve schema 1 and its required fields and status spellings.

Surface filing failure kind/message in body-free warnings and keep useful per-thread
failure rows. Human and quiet output must show route label, resolver explanation, and
hint; distinguish a remote failure from a successfully filed route problem. Quiet JSON
remains a complete document. Keep the no-op header and inert mail title/who rendering.

Tests must cover one permanent fetch failure alone (`pulled=0`, `watching=0`, exit 1),
mixed success/failure counts, a first-seen route problem counting once, a planning
failure's diagnostic, dry-run parity, and a watcher closed/deleted between initial scan
and locked apply.

## 4. Complete transaction recovery without broadening commit scope

The current normal path uses shared save planning, capture's staged writer, one lock,
one scoped commit, and then journal appends. Keep it. The recovery gap is specific:
after a commit failure, replay sees the new task and adopts it. There are then no
prepared note plans; `commit_paths` includes only adoption note/task paths, while newly
written images and canonical auxiliary files are omitted, and the deferred save and
attachment-origin events are lost. The helper that recovers canonical hub paths is
called only when a note plan is changed.

Retain enough body-free transaction ownership evidence to finish the original scoped
commit and its deferred events before enrollment is recorded. Reuse shared save recovery
helpers; include owned images and canonical auxiliary paths while retaining their
existing ownership rules (custom hubs, unrelated staged dirt, and the vault AGENTS
warning are not blanket commit targets). Recheck owned paths and snapshots under the
lock; do not overwrite edits made after the failed attempt. Recovery must work after the
enrollment label disappears and in watch-only mode, without requiring the failed
transaction's SHA or a push. Preserve successful non-Git writes as the durability
boundary.

Add a temporary-Git-vault test with an imported image, canonical hub/base creation or
upgrade, unrelated dirty content, and a failing pre-commit hook. Assert no enrollment is
recorded on failure. Remove the hook and replay: exactly one task/card set, all owned
recovery paths committed, unrelated/custom content untouched, and the deferred
save/origin/enrollment events recorded without duplication. Include journal-append
failure then replay, image rollback on a failed text batch, and the bounded second
snapshot-conflict deferral. Existing save/link/compose recovery tests must continue
passing.

## 5. Finish the documented contract and acceptance coverage

The phone guide still describes forwarding to a plus address and applying only `bob`; it
never teaches the approved `bob` plus one `#/note` routing label on the original
conversation. Make that the primary procedure, explain the corresponding configured
prefixes, and keep any forwarding alternative clearly separate so it cannot imply
watching the original conversation. In `docs/gmail.md`, fix the claim that a missing
label disables sync (missing configuration defaults to `bob`; an explicitly empty label
disables it), and document adoption, durable enrollment after label removal/task
closure, and first-seen route failures. Complete the intended journal field reference in
`docs/reference/vault-format.md`. Add the missing watch-only `sync -l 0` help example
and regenerate CLI docs.

Remove the now-stale command-phase `allow(dead_code)` scaffolding throughout `sync/` and
on `save::write_prepared_batch`. Wire active APIs, remove redundant APIs/constants, and
keep any narrow future-consumer allowance only with an accurate reason tied to an actual
still-open outer phase. Do not use broad impl/module allowances to hide unused
implementation.

Review the original sync acceptance matrix against existing tests; the current
`tests/gmail_sync.rs` has only eight tests and its busy-guard test waits the default 120
seconds twice. Set explicit fast contention budgets, use deterministic fixtures, and
cover the gaps that exercise composition: legacy enrollment; close/delete/reopen/move
behavior; paged selection; guard and vault contention without writes; no-op/no-commit;
and alias equivalence. Test byte-identical stdout, stderr, exits, and relative vault
effects for `sync` versus `pull` in separate fixed-time fixtures, including live
success, dry-run, route problems, help, and setup failure. Reuse the already
comprehensive pure membership and routing vectors rather than duplicating them
mechanically.

Run focused sync tests and affected Gmail save/link/attach/compose/turn, catalog, and
completion tests. Run `just docs-gen` for the help edit, `just docs-check`, formatting,
and **`just check`** before closing. Do not run `just check-full`. Use `/sase_monitor`
for long verification as required; any continuation owns the same final closeout below.
A check-full-only failure is infrastructure, not remaining epic work.

## 6. Finish this landing in the coding turn

This tale has no land agent. Do not leave closeout for the agent that proposed it, and
do not order closeout after this work's commit SHA, push, or CI result. Passing local
verification and closing in the same turn as the code edits is the intended path;
`/sase_final` commits afterward.

The originating lander completed follow-up triage and recorded it on `bob-cli-ii.3.1`:
all four child notes contained **zero** `PROPOSED FOLLOW-UP:` entries, no unrelated
defect was established, no new task was warranted, and no proposal was declined. A prior
one-off guard test failure followed by passing reruns is covered by deterministic epic
verification, not filed as unrelated work. All defects above belong to this epic. If new
unrelated work is established, use `/sase_new_task` and record the outcome on the epic
before closing.

Complete these actions in order:

1. Reread `bob-cli-ii.3.1`, its four phase beads, all notes, and linked plan; verify
   this repair plus existing code meets their scope. Recheck drift against the current
   base branch, preserving any outer-epic work. If the host assigned this tale its own
   descendant bead, close that completed tale bead normally first so it cannot block its
   parent's close; also mark the tale's own plan done through the normal plan workflow.
2. Run `sase bead epic-symbols bob-cli-ii.3.1`. Resolve every entry by wiring,
   privatizing, an appropriate non-test pragma, or deletion; re-key only when a specific
   still-open later bead genuinely needs it. It was empty at review, but must be checked
   again.
3. Run
   `sase bead close bob-cli-ii.3.1 --note "<concrete source and commit review, drift integration, fixes, follow-up outcomes, and passing check/docs evidence>"`.
   Do not force a successful nested landing. If closure names unfinished work, finish or
   reopen it. Only intentionally canceled/superseded work may use
   `--force --reason ... --resolution canceled|superseded`; never force to make the
   command succeed.
4. Run `just symvision` if the recipe is available; the reviewed justfile had no such
   recipe, so record that fact if it remains absent. Open the plans sidecar with
   `/sase_repo`, use the PLAN path printed by `sase bead read`, and set `status: done`
   in the frontmatter of `plan:202611/gmail_sync_filing.md`. Read artifact content only
   through `sase artifact read`; do not edit bead-store status files.
5. Read `sase bead read bob-cli-ii.3.1 -r "Need the parent link"` and
   `sase bead read bob-cli-ii.3 -r "Verify the delegated sync phase is complete"`. The
   parent is phase **bob-cli-ii.3**. Verify its original sync scope, children, and
   notes; resolve/re-key entries from `sase bead epic-symbols bob-cli-ii.3`; then close
   **only that phase** normally with
   `sase bead close bob-cli-ii.3 --note "<verified delegated sync contract and checks>"`.
   Leave containing epic **bob-cli-ii** and its outer plan to its waiting land agent. If
   the parent is incomplete or ambiguous, note the precise blocker on it and report it
   instead of forcing closure.
6. Finish through `/sase_final`, declaring the code and every changed sidecar
   repository. Report actual closed bead IDs and verification; never claim the outer
   reply-watch epic is finished.
