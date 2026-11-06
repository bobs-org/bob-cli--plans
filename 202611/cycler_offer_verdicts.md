---
tier: epic
title: Complete cycler successor-card verdicts
goal:
  Obsidian successor cards safely apply Done and Cancel alongside pull-ins, recover
  dependents without another offer round, report exact guarded Undo, and advance the
  review walk once without landing on a resolved task.
phases:
  - id: contract_parity
    title: Repair the shared offer-close contract and executable vectors
    size: medium
    depends_on: []
    description:
      "contract_parity: align the landed Rust descriptor copy and shared SV fixtures
      with the approved parent design, make the engine vectors executable, and preserve
      existing close bytes and recovery policy."
  - id: descriptors
    title: Add cycler close descriptors and normalize staged choices
    size: medium
    depends_on:
      - contract_parity
    description:
      "descriptors: copy the canonical vectors, implement pure eligibility and
      consequence simulation for all five offer groups, and normalize disjoint picks and
      closes without enabling verdict controls yet."
  - id: verdict_apply
    title: Apply guarded verdicts and recover dependents with Undo snapshots
    size: medium
    depends_on:
      - descriptors
    description:
      "verdict_apply: implement the internal Now verdict transaction, per-row stale
      refusal, root-only Done and Cancel, joint recover-only cascades, truthful
      receipts, and exact guarded Undo snapshots; leave gesture wiring dormant."
  - id: gesture_integration
    title: Connect verdicts to gestures, notices, reopen, and the review walk
    size: medium
    depends_on:
      - verdict_apply
    description:
      "gesture_integration: enable descriptors for compatible nav, revalidate
      verdict-only and mixed submissions, run the verdict step after picks, deliver one
      receipt with Undo, preserve reopen behavior, and exclude resolved rows from the
      single walk advance; update docs and verify the complete feature."
proposed_by: bbugyi200.athena.bob-cli-hn.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hn.6
create_time: 2026-11-05 20:37:45
status: wip
---

- **PROMPT:**
  [prompts/202611/cycler_offer_verdicts.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/cycler_offer_verdicts.md)
- **PARENT:**
  [202611/successor_card_verdicts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/successor_card_verdicts.md)

# Cycler successor-card verdicts for bob-cli-hn.6

Implement the existing phase `bob-cli-hn.6` of `plan:202611/successor_card_verdicts.md`.
Completing or parking a task can open one successor card. That card can stage Done and
Cancel for existing offered tasks, alongside ordinary pull-ins. Applying it closes only
those roots, recovers their dependents without another picker, and reports exactly what
happened with guarded Undo. Closing the project's last task and marking its offered
`^prj` Done is the first end-to-end demonstration.

This is a child epic because contract parity, pure description, guarded mutation, and
gesture/walk integration have distinct invariants and test boundaries. Each phase is
bounded implementation work for one coding agent. The dependency chain is intentional:
no verdict controls become available before the entire apply path is ready. This plan
creates no additional product decisions.

## Authority, scope, and landed work

Read the parent design through `sase artifact read`, the assigned bead through
`sase bead read`, and the relevant decision strands through `/sase_memory_read`. The
parent design's shared rules, its cycler phase, and its final DECISIONS remain binding.
This plan resolves implementation details, not the accepted UX policy.

The planning audit found no earlier-work notes on `bob-cli-hn.6`; it is `in_progress`,
and `sase bead epic-symbols bob-cli-hn.6` reported no entries. There is no reason to
reopen or manually change its status. Recheck landed work before implementing,
particularly if the branch has advanced since this audit.

Already landed and to be reused:

- bob-cli `53f9c2c7`: `src/native/task_complete/offer_close.rs`, its tests, and
  `tests/fixtures/offer_close/vectors.json`; parent phases hn.1 and hn.2 are closed.
- bob-plugins `8a16ebe`: nav successor picker v4, including staged verdict UI.
- bob-plugins `996fc83`: nav cancel v1 and notice v4, including guarded Undo; parent
  phase hn.5 is closed. The inspected plugin HEAD was `710feae`.
- A read-only harness comparison of the existing cycler `completeTaskLineText` and nav
  `api.cancel.planCancel` against SV1–SV7 passed all seven exact source postimages.
  Preserve those bytes and reuse those writers.

Repositories: primary bob-cli only for the narrowly defined contract repair in phase
`contract_parity`; linked bob-plugins for all cycler/nav implementation. Open
bob-plugins with `/sase_repo`, use only the returned checkout, and read its `AGENTS.md`.
Edit source fragments, build generated `main.js`, and register every new test script in
the repository-root `package.json`. Hand-edited fragments have a 1000-line cap. The
current cycler `137-plugin-successor-project.js` has 980 lines,
`085b-successor-offers.js` 966, and nav `536-plugin-review-advance.js` 938: extract
cohesive helpers before expanding them.

No memory edits belong to this child epic. The parent's allowed edit to
`decisions:unblocked-tasks-are-offered` remains the parent land agent's work; hn.1 and
hn.2 already record its authorization collision. The rejected new decision strand and
glossary edits remain rejected. Do not bump the public cycler API to v5 or implement
project-promotion wiring: hn.7 owns those. Do not change Mac code, capture flags, or the
task-merge card.

## Fixed implementation contract

1. **Engine ownership.** The cycler computes separate `complete` and `cancel` booleans
   (equal in v1), optional `why`, `completeConsequence`, `cancelConsequence`,
   `scopeNote`, and integer `unblocks`. It uses the shared approved copy and
   recover-only simulation. Presenters only render descriptors. Cover hand_off, project,
   project_finished, keep_moving, and relation rows. Muted rows have no verdict slot.
   Exclude new, recurring, non-open, unresolvable, and writer-refused rows. Explicitly
   reject non-open statuses even though `completeTaskLineText` returns
   `ok: true, unchanged: true` for some of them.
2. **Capability.** Attach `model.close = {supported: true}` and row descriptors only
   with nav `api.successors.version >= 4` and `api.cancel.version >= 1`, callable
   methods, and the integrated apply implementation. Existing v3 and missing-cancel
   behavior is unchanged. Keep cycler API version 4 until hn.7.
3. **Identity.** Use existing offer keys and their digest/occurrence codec, never a new
   key algorithm. A block ID is not permission to ignore changed task text. ID-less
   closes preserve the ID-less line; picks alone may mint IDs. Preserve exact card
   preimages, digest multiplicity, and stable identity across harmless line movement.
   Ambiguous duplicates are refused, never rebound to a neighbour.
4. **Ordering.** Within the existing reference-mutation queue: show one card,
   reread/replan, apply picks and ordinary predecessor recovery, apply verdicts,
   remember only ordinary pull-in take-backs, present the result, then finish the
   original gesture/walk continuation. Verdict-only submissions must enter the fresh
   reread path, which currently runs only when `selectedKeys.length > 0`.
5. **Expected lines.** A verdict compares against the line the pick pass was expected to
   leave: its verified successful postimage when that pass edited the row, otherwise the
   card preimage. Never accept an arbitrary freshly read line as the new expected line.
   Verify the fresh offer set and today's live Task Links again. A failed recovery/pick
   write cannot authorize a later close.
6. **Writes.** Done calls `completeTaskLineText(line, {date})`, root only, without Work
   Log prompts. Cancel calls nav's pure `planCancel`, with one trimmed reason line and
   Task Card's four Cancel Log cases. Fields such as `fresh`, `keeps`, and `unblocked`
   on the root remain untouched. No verdict writes a ledger link.
7. **Recovery.** After successful closes, simulate/apply the actual joint closed set
   over fresh dependency data. Recover via existing dependency/project rules and
   `rewriteRecoveredTaskLine`, honoring every WRITE skip. No picker, relation offers,
   keep-moving offers, pull-ins, or ID minting. A shared dependent is counted and
   reported once, even when two closes are jointly required.
8. **Concurrency.** Inside the queue, call only internal `...Now` functions or pure
   helpers. Never call queued `recoverBlockedDependents`, `finalizeClosedTasks`, or nav
   `runPostCancelEffects`. Preserve all editor and vault write guards. Obsidian is
   best-effort per row; unrelated valid rows must still apply when one is stale. Failed
   writes and recoveries remain visible.
9. **Undo.** Coalesce `{path, before, after}` across verdict and recovery writes.
   `before` is the exact state after ordinary picks/predecessor recovery and before the
   verdict step; `after` is what this step actually wrote. Undo keeps the predecessor
   closed and pull-ins intact. Use nav's existing all-or-nothing guarded restore,
   refusing if any touched postimage changed. Do not capture unrelated concurrent edits
   into an Undo preimage.
10. **Reopen and walk.** Same-day predecessor reopen removes only untouched ordinary
    pull-ins, leaving verdict roots closed. A walk gesture advances once from its
    original anchor and skips every successfully resolved row, even with a stale Tasks
    cache. Refused rows remain eligible for the walk. Preserve PRE/POST boundary
    behavior, gesture hold, jump history, and lock.

## Phase contract_parity

Work in bob-cli. Correct the shared inputs before copying them to JS. This is a repair
of the approved contract, limited to descriptor/copy/vector consistency; the existing
root-write and recovery behavior stays intact.

The audit found concrete mismatches:

- The parent requires readable `why` sentences; the landed Rust descriptor exposes
  internal strings such as `recurring` and `new_task` instead.
- The landed scope note says `embedded task` for both ordinary children and project
  residents. The approved table distinguishes open subtasks from open project tasks. The
  project consequence omits the approved status transition.
- Rust computes `unblocks` but does not append the approved unblock/tickler consequence
  suffixes. The landed docs table is also a shortened version of the parent's canonical
  table.
- SV9 and SV10 expect `[unblocked:: today]` but their dependent source lines have no
  `[fresh::]`. The established WRITE policy skips those lines; the Rust unit test
  correctly supplies `[fresh:: 2026-11-01]`.

Read `docs.md` and the recovery decision first. Align `docs/task-dependencies.md`
sections 12.22–12.23, the Rust descriptor builder, and the canonical JSON fixture with
the parent's full canonical sentences, singular/plural handling, root scope, and
all-or-nothing Undo description. Preserve machine refusal codes in `OfferCloseRefusal`;
separate classification from human `why` so `plan_offer_close` does not infer a refusal
enum by matching translated/display text. Keep the CLI's existing machine error
contracts intact.

Make fixtures executable assertions in
`src/native/task_complete/tests/offer_close_tests.rs`, instead of a catalogue that tests
can silently disagree with. Add the missing fresh field to the stamped recovery
fixtures; separately pin an unstamped NEW dependent and other WRITE skips. Add focused
vector coverage for authored Cancel with a newly created log, plural scope/recovery,
exact refusal sentences, and two roots jointly unblocking one dependent. Retain SV
identifiers; append cases without renumbering existing cases. Clearly distinguish
machine refusal classification from descriptor `why` in fixture fields. Test per-row
simulation with one root only and joint apply with both roots. Existing Done/Cancel
postimages must continue matching the seven already verified JS cases.

Do not rewrite unrelated capture planning or user choices. If capture's ongoing hn.3
work has already corrected these fields, use that landed work and avoid duplicate edits.
Validate focused Rust offer-close tests, lint, and docs checks; run the repository gate
required by the current checkout. Record failures that reproduce on the clean base as
directed below. Deliver a canonical fixture commit whose bytes the next phase can copy.

## Phase descriptors

Work in bob-plugins. Copy `tests/fixtures/offer_close/vectors.json` byte for byte from
bob-cli into `plugins/task-status-cycler/scripts/fixtures/offer_close/vectors.json`; add
a README identifying the actual canonical source commit. Follow
`scripts/test-task-status-cycler-relation-successors.cjs` for canonical-byte equality
when the primary checkout is available. Never hardcode a numbered workspace path. Use
the same vectors for exact copy and task-byte assertions.

Add pure helpers in new ordered fragments, for example `089g-successor-offer-close.js`
and `089h-successor-offer-close-copy.js`, with exports for tests. Reuse existing
parsing, task status names, normalized display text, project membership, live-link
detection, and offer digest utilities. Build a reusable immutable snapshot/context once
per gesture, overlay ordinary planned recovery, and simulate each candidate
independently using the existing dependency planner. Avoid a vault scan per candidate.
Include dependency-index and project-resident semantics; unavailable dependency data
must not be treated as proof that a blocked row is writable.

Provide internal entry points for description and validation that the next phase and
future hn.7 wrappers can reuse. Preserve the public v4 API. A suggested context contains
rows keyed by offer key, card identities/preimages, note snapshots, parsed
tasks/dependency index, date, daily snapshot, and freshness writer. Keep data passed to
the modal separate from private write authority.

Normalize `{picks, closes}` in one helper, retaining valid outcomes `done` and
`cancelled`, trimming/collapsing cancel reasons, deduplicating keys, and excluding every
accepted close key from picks. Reject conflicting or malformed close instructions
conservatively rather than silently choosing a destructive outcome. Preserve the
existing `normalizeSuccessorPicks(result)` array contract for current callers/tests
through a wrapper if a new choice normalizer is used. `null` remains no choice;
`{picks: [], closes: [...]}` remains meaningful.

Do not attach supported descriptors to live gestures in this phase. Pure model
enrichment may be tested with injected compatible APIs. Add and register
`scripts/test-task-status-cycler-successor-verdicts.cjs` covering all relevant pure SV
cases, all five groups, missing/old APIs, status/recurrence guards, ID-less identity and
duplicate ambiguity, scope, consequence copy, and choices. Keep existing v3-shaped
models deeply equal. Build, run focused tests and the standard plugin gates, then
`bob plugins sync`.

## Phase verdict_apply

Work in bob-plugins. Add an internal `applyOfferClosesNow`-style method and supporting
pure planner, installed through `200-install-methods.js` and ordered in
`src/fragments.json`. Test it directly before connecting gestures. Its input carries
normalized closes, the original card identities, verified expected post-pick lines, and
the fresh planning context. Its result carries `offer_closes`, `offer_close_refused`,
`undo: {restores}`, successful resolved identity refs for the walk, and explicit
recovery/write failures.

Revalidate offer membership, exact expected line, digest multiplicity, eligibility, and
live links per row. Resolve positions before writing. Plan multiple same-note
Done/Cancel operations together, accounting for Cancel Log insertions that shift later
rows. Do not feed multiline cancel results into the existing single-line
`applySuccessorNoteEdits` interface. Extract or extend a guarded note writer for planned
whole-note/splice changes while retaining editor-first reads and `vault.process` guards.
Preserve LF/CRLF, final newlines, untouched child bytes, and user edits. A stale row in
a note must not prevent another unchanged target in that note from being considered.

After actual successful root writes, generalize the current `cancelledClose`
recover-only path into an explicit internal mode usable for both outcomes. Disable
relation and keep-moving generation, linking, prompts, and reopen receipt storage. Keep
outcome-specific project semantics; do not pretend Done is Cancel merely to enter the
current recovery branch. Rebuild/overlay fresh task documents after roots close so joint
dependencies resolve correctly. Recover only from successful writes, reporting actual
recoveries and preserving WRITE skip rules. Use the current `rewriteRecoveredTaskLine`
and dependency rules rather than inventing a second recovery engine.

Record all actual note writes in the snapshot accumulator. If a file changes between
root and recovery writes, guard/refuse the later change and report it; never overwrite
it or manufacture a successful recovery receipt. Snapshot records must remain safe for
the existing nav restore guard. Attribute each joint recovery deterministically once
across the receipt rows; descriptor counts remain independent single-row predictions.
Include authored reasons in Cancel receipt copy, with no blank-reason clause.

Add registered integration tests for mixed Done/Cancel in one note, all four Cancel Log
cases, unchanged root fields and children, no minted IDs, stale or deleted rows, new
live links, duplicate ID-less rows, editor and vault paths, failed note writes, joint
recovery, project recovery, freshness skips, and no second offer round. Run the worker
through `enqueueTaskReferenceMutation` with a bounded test timeout so a nested-queue
deadlock fails clearly. Exercise nav's real restore helper: exact Undo of roots plus
recovery, refusal on one changed file without partial restoration, and preservation of
earlier pick/predecessor writes. Build, run focused and standard plugin gates, and sync.

## Phase gesture_integration

Work in bob-plugins. Refactor cohesive pieces out of `137-plugin-successor-project.js`
to make headroom, retaining its existing pick/apply behavior. Wire the prepared
descriptors into `buildSuccessorPickerModel` only when the full compatibility gate
passes. Retain original offer/preimage data across `pick()`. Normalize choices and make
the fresh-read/replan branch run for either picks or closes. Exclude close keys from
pick selection and destination overrides. Preserve ordinary no-choice recovery, picker
cancellation, and old-nav fallback.

After `applySuccessorNoteEditsWithUnblockedGuard` and daily insertions, invoke the new
verdict worker in the same queued mutation, before `rememberSuccessorPass`. Pass only
verified post-pick expectations. Remove successful verdict roots from ordinary
receipt/take-back collections, including the `trigger === "project_empty"` special case
in `136-plugin-successor-receipt.js`. Do not present the same task as both an unpicked
recovery and a completed row.

Extend `successorNoticeForModel` and `presentSuccessorModel` in `135` (using new
fragments if useful) for closes-only, mixed, and refusal-only results. Forward the v4
fields and Undo to nav v4; with an incompatible/missing notice API show canonical text
that includes every close/refusal and actual stamped recovery. An old notice renderer
must not silently discard verdict results. Keep partial write/recovery failures
explicit. Reuse nav's existing Undo action and fix only the small copy/forwarding seams
needed to match the canonical receipt table.

Walk landings currently use `successorNoticeTarget: "return"`, which returns plain text
before rendering nav's rich notice. Preserve a single walk advance while ensuring the
verdict receipt still exposes `Undo closes` on this route. Pass structured receipt data
through the finalization/continuation result or show the v4 receipt once and let the
walk display only its landing tail. Do not duplicate the close summary or lose Undo
merely because a walk is active.

Pass successfully resolved identity refs to the existing `handledRefs`/answered key
exclusion mechanism in nav `536-plugin-review-advance.js`. Translate offer identities
using the original review queue's own key function; offer keys and walk keys are not
interchangeable. Preserve anchor position at the original landing, rather than moving it
to the last resolved row and skipping intervening unanswered work. If a new
exclusion-only outcome member is needed, make it additive and preserve older caller
behavior. Ensure a stale queue cannot land on any resolved root, and do not suppress
refused rows. Retain same-day reopen semantics, including project-finished rows and
mixed pull-ins/verdicts.

Extend the registered verdict suite and existing review-advance/wiring tests with SV14
and SV15 plus these end-to-end cases:

- The first demo: last project task Done, offered `^prj` Done, one commit and one
  receipt; the project and predecessor stay closed after take-back.
- Mixed pull-in, Done, and reasoned Cancel, both normal Done and a parked relation offer
  gesture; exactly one picker and recover-only secondary effects.
- Verdict-only commit, no highlighted-row fallback, no selection, Esc, unload, malformed
  results, v3 picker, and v4 picker without cancel v1.
- A card-open edit refuses only that row; same-note insertions and recovery do not
  incorrectly mark verified expected postimages stale.
- A stale walk queue with the next row resolved, multiple nonadjacent resolved rows, and
  one refused row: advance once to the first remaining row, preserving checklist
  boundaries, jump history, and the receipt's Undo action.
- Same-day predecessor reopen restores only untouched picked rows. Undo closes restores
  only verdict/recovery changes, including when picks share the note.

Update `docs/task-status-cycler.md` under “Successor offers and take-backs”, its
generated inventory inputs where needed, and `docs/task-relations.md` where it describes
successor choices. Explain capability fallback, root-only verdicts, recover-only
cascades, refusal, Undo, and reopen. Public v5 promotion APIs remain the next parent
phase's work. Run build, the full test suite, docs check, and `bob plugins sync`.

## Verification and completion ownership

Every worker runs focused tests for its actual change and required repository checks.
For bob-plugins the gate is `npm run build`, `npm test`, and `npm run docs:check`;
generated entrypoints and any generated docs belong in the change. Always run
`bob plugins sync` after plugin repository edits. Use `/sase_monitor` for long gates and
follow `/sase_final` for committing every changed repository. No manual git commits,
version edits, or release work.

Previously reported failures are evidence to compare, not permission to skip
verification. hn.2 records clean-base clipboard-registration failures tracked by
bob-cli-hp and BOB_GMAIL documentation failures noted on hn.1/hl.1. hn.5's theme fixture
failures may already be repaired by plugin commit `710feae`. Reproduce any current
unrelated failure on a clean base and record the exact test, SHA, and existing tracker
in a `PROPOSED FOLLOW-UP:` note on the worker's assigned bead. Do not create task beads;
the parent land agent triages notes. Identical clean-base failures do not keep completed
phases open.

Before closing any assigned child phase, run `sase bead epic-symbols` for that phase and
resolve or re-key remaining entries to an open owner. Phase workers close only their own
assigned phase, with a verification note. They never close this child epic,
`bob-cli-hn.6`, or `bob-cli-hn`.

The child epic's land agent checks all four deliverables together, confirms the
canonical fixture bytes match both repositories, confirms the complete gesture and walk
tests and sync evidence, and checks `sase bead epic-symbols bob-cli-hn.6`. Its
completion responsibility includes the original phase `bob-cli-hn.6`, once the child
work is landed and its descendants are closed, using
`sase bead close bob-cli-hn.6 --note "<verified scope and gates>"` under the original
request's land-agent authorization. It must not close `bob-cli-hn` or any ancestor plan
bead. Leave concise internal helper/API notes for hn.7 so its promotion wrappers reuse
this engine rather than implementing another one.
