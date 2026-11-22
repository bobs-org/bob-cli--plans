---
tier: epic
title: Requirements on the capture successor card
goal:
  "Complete bob-cli-on.8: managed capture Done closes audit each owner's pending
  requirements, apply explicit drop/carry/pull-in choices atomically, reject stale or
  overlapping choices, and report the final navigation and outcomes to clients."
phases:
  - id: rows
    title: Pre-close requirement groups and snapshot identity
    depends_on: []
    size: medium
    description:
      "rows: collect each managed closing owner's pre-close requirement forest,
      implement snapshot-bound row keys and canonical group copy, and establish the RV
      fixture schema and row-building vectors."
  - id: choices
    title: Requirement verdict resolution and refusal rules
    depends_on:
      - rows
    size: medium
    description:
      "choices: implement the pure Met, Drop, Carry, and Carry-Pulled selection model,
      shared card numbering, overlap and snapshot validation, and requirement-aware
      mark-all exclusion with RV selection vectors."
  - id: writes
    title: Atomic carry and drop planning
    depends_on:
      - choices
    size: medium
    description:
      "writes: adapt carry_req and drop_req to guarded staged capture snapshots, compose
      line movement across disjoint branches, and return exact outcome receipts without
      adding another commit path."
  - id: placement
    title: Carried-task pull-in and final jump resolution
    depends_on:
      - writes
    size: medium
    description:
      "placement: plan carried-task links at the existing successor destination, raise
      only pulled tasks to Next without stamping freshness, and resolve pinned
      requirement landings through carry and routing move maps."
  - id: capture
    title: CLI, capability gate, JSON, and human output
    depends_on:
      - placement
    size: medium
    description:
      "capture: connect the requirement pipeline to every managed capture Done entry,
      add carry and req-choice arguments, publish groups and receipts, preserve
      old-client defaults, and cover the public CLI flow end to end."
  - id: contract
    title: Published contract and cross-feature regression evidence
    depends_on:
      - capture
    size: medium
    description:
      "contract: finish task-dependencies sections 12.25 and 12.26 and the JSON
      reference, complete RV and compatibility coverage, run the canonical code and docs
      gates, and prepare completion evidence for bob-cli-on.8's land agent."
proposed_by: bbugyi200.athena.bob-cli-on.8
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-on.8
create_time: 2026-11-22 07:22:28
status: wip
---

- **PROMPT:**
  [prompts/202611/requirement_close_engine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/requirement_close_engine.md)
- **PARENT:**
  [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)

# Requirements on the capture successor card

## Scope and evidence

This is a child epic for the `close-engine` phase, **bob-cli-on.8**, of
`plan:202611/task_requirements.md`. Deliver that phase only. The endorsed sources are
`research:202611/task_reqs_design/task_reqs_design.md` and
`research:202611/task_reqs_nested_and_notes_impact.md`; the amendment and the parent
epic's design of record supersede the original report's single-level restriction. Read
them through `sase artifact read`.

The assigned bead is in progress with no previous-work notes. Inspection at `2f4fef6d`
found the prerequisites already landed: `task_reqs`, `task_block_layout::writers`,
capture grammar 29 and requirement authoring, and Task Split carry (`22391208`). The
capture close path has no requirement verdict engine, `--carry`, `--req-choice`,
requirement groups, or RV fixtures. Do not redo the prerequisite work. Recheck the
latest tree and bead notes when each phase starts. `sase bead epic-symbols bob-cli-on.8`
currently reports none.

Six bounded implementation phases are appropriate because the existing close flow spans
tree completion, staged batch planning, successor selection, routing, and final
navigation. All dependencies are explicit and sequential because the phases share those
interfaces. Each phase has a concrete test boundary and fits one coding agent; none
needs another planning turn by default.

Honor the parent DECISIONS without asking again: `notes_drawer_birth` is
`with_first_note`, and `seed_review` is `yes`. The approved memory changes belong to the
parent's memory phase. **This child epic edits no memory, plugin code, Mac code, live
vault, Cargo version, or CHANGELOG.** It publishes the Rust contract consumed by
bob-cli-on.11 and bob-cli-on.15. No task beads may be created: workers put
`PROPOSED FOLLOW-UP:` notes on their assigned beads.

## Behavior that every phase must preserve

- Only a managed **Done** closing set is audited: the root and actual embedded or nested
  real tasks closed with it, deduplicated in closing order. Skip recurring owners.
  Cancels, parked/kept/deferred links, automated closes, reconcile, raw edits, and
  another card's offer-close verdict never create another requirement audit. A `^prj`
  owner is eligible under the same rule.
- Read requirements from the owner's **pre-close** snapshot. Pending includes flagged
  unknown marks as recognized by `task_reqs`. Excluded descendants are not Pending.
  Manual `[x]` is Met for that row only. Requirement recognition remains structural;
  ordinary checklists and nested real tasks stay distinct.
- Put one `requirements` group per eligible owner before all task-offer groups. List
  Pending rows in document order at their actual depth. Use the capped glyph run from
  the parent design: two, three, then four hashes for depth 3+.
- Default and explicit Met write no requirement bytes. Drop writes only the selected
  root's `[-]` and optional Cancel Log. Carry creates exactly one real task and a `[>]`
  stub for each disjoint selected branch; notes, anchors, and descendant history travel
  using the existing Task Split rules. No dependency is invented. NOTES appears only
  when notes are filed into it.
- Carry plus pull-in raises the new task to Next and links it at the ordinary successor
  destination; never give this capture-created task a freshness stamp. Carry without
  pull-in leaves it unlinked. Mark-all never selects req rows.
- A jump pin is navigation only. It resolves after all planned writes and routing: the
  new task for a carried root, the mapped line for a descendant moved by an ancestor, or
  the requirement's final line otherwise. Never find the landing by label alone.
  Preserve the existing one-pin and walk-park contract; bob-cli returns the landing,
  while clients own focus and walk state.
- Capture has one all-or-nothing commit. A stale row, overlapping structural choices,
  missing destination, or failed guard must leave every vault file unchanged. Do not
  weaken disk-preimage or scanned-file-set guards.

## Shared implementation decisions

### Rows and identity

Use a dedicated module under `src/native/task_complete/` for the pure row, group,
verdict, and copy model, and small modules under `src/native/capture/` for adapters.
Extend existing functions through narrow interfaces rather than growing `plan.rs` or
`output.rs` with the whole engine.

The existing `task_reqs::req_row_key(owner_key, path, text)` hashes only the clean row
text. A path plus that hash cannot detect swapping two identically labelled branches
with different children across separate preview and submit processes. Bind the **owner
key** to the complete owner-block preimage:

```text
task_key = existing canonical note/task identity
owner.key = task_key + "@" + sha256(exact pre-close owner-block bytes)
row.key = owner.key + "/req/" + sibling_path + "~" + sha256(clean_text)[0..8]
```

The owner object also carries the canonical `task_key`, note path, block ID when
present, 1-based source line, and display text. An idless owner's canonical identity
uses the existing deterministic line/digest identity conventions; do not mint IDs for a
preview. Compare the complete supplied row key to freshly enumerated rows, not merely
the suffix accepted by `carry_req`. Clients treat keys as opaque. No new preview-token
flag or persistent state file is needed. Retain the full preimage and original
coordinates internally for validation against the staged planner; internal capture edits
must be mapped explicitly.

Numeric picks address the current command's probe. Clients holding a card must submit
its opaque keys, including for the jump pin; fresh numeric picks cannot promise to
identify an earlier preview. Document this distinction and test a two-process
preview/edit/submit scenario. Same-key preimage checking is required again between the
probe and final batch and at disk commit.

### Wire shape and compatibility

Keep the existing task offer schema and `group` discriminator. Requirements use an
additive group with `group: "requirements"`, `kind: "requirements"`, `owner`, `folded`,
canonical `copy`, and `rows`. Each requirement row contains an `offer` envelope, `req`,
and `consequences`; it is never inserted into `unblocked[]` as if it were a recovered
task. Introduce an internal card-row view for shared numbering and selection, leaving
real-task scanners unchanged.

`req` contains `owner`, sibling `path`, exact `depth`, `parent_key` or null, clean
`text`, 1-based `line`, pre-close `state`, and `pending_descendants`. `consequences`
contains bob-authored `met`, `drop`, `carry`, `carry_pulled` copy. `pull_in.reqs` is
`{supported: true, pending: <pending row count>}`. Top-level `req_outcomes[]` parallels
the existing `offer_closes[]`: include owner/row identities, verdict, optional reason,
carried task identity, whether it was pulled, and the final bob-authored receipt. Do not
repurpose the existing capture item's `requirements[]`, which reports authored
requirement writes.

Normal CLI previews and req-aware clients use one number space with requirements first.
For an old client invoking `--require-pull-in-choice` without `--req-choice`, use the
legacy task-offer projection and numbering, advertise the capability in `pull_in.reqs`,
and leave requirements at Met. Such a client must never accidentally drop a req when it
submits a task-offer number. A new client discovers the capability, re-previews with
`--req-choice`, and submits keys from that preview. Publish and test this handshake
before downstream clients rely on it. With both gate flags, Pending reqs alone demand a
choice. `-p none` or explicit req verdicts acknowledge the card; a jump alone is not an
acknowledgement. In the req-aware projection `offer_count` counts actionable card rows,
`picked_count` counts actual pull-in links, and `reqs.pending` counts all pre-close
Pending reqs, including subsequently folded descendants.

### Choices and copy

Req choices are Met, Drop, Carry, and Carry plus pull-in. Repeating the same interactive
verdict returns to Met; changing it replaces the verdict. The CLI accepts final choices
rather than replaying keystrokes. Preserve ordinary offer duplicate/conflict rules, make
identical repeated carry picks idempotent, and reject contradictory choices with
`offer_verdict_conflict`. Treat `-k` plus `-p` on the same req as redundant conflicting
final choices; `-p` alone already means carry plus pull-in.

An ancestor Drop or Carry owns its branch. Explicit descendant verdicts conflict as
`req_verdict_overlap`; a navigation-only descendant pin is allowed. Default Met
descendants are not explicit conflicting choices. Staging a structural verdict folds
descendant rows into `· {k} nested go with it`; restoring Met unfolds them with their
staged choices retained. Folding is display state, not silently discarded selections: a
hidden conflicting choice must be refused before submit. Resolve overlap before planning
any writes.

Freeze canonical strings in RV vectors, including:

- `Requirements · “{S}”`; `{n} reqs` or `{n} reqs in {b} branches`;
  `Met when it closes, unless you say otherwise`.
- `· {d} dropped earlier · {c} carried earlier · {m} met earlier`, omitting zero
  segments; count explicit historical states, not excluded descendants as met. Branch
  counts describe the pending requirement forest.
- `“{ancestor}” already takes this branch — restage it as met to choose here`.
- `Why drop it? (optional)`;
  `⌃X met · ⌃⇧X drop · ⌃⇧T own task · space pull in · ⌃J jump`; `reqs ✓4 ⊘1 →1` as the
  summary pattern.
- `↗ jumps to the requirement — ⌃⇧T makes it a task`.
- `✓ {n} requirements met with “{S}”`, `⊘ Dropped “{T}”` plus optional `· “{reason}”`,
  and `→ “{T}” is its own task now` plus optional `· pulled in`.
- Human default-close line `⌗⌗ {n} requirements met with “{S}”` and hint
  `-k N own task · -X N[=why] drop · -p N pull in`; dry runs use would forms.
- Accessibility describes the semantic state and exact depth, for example
  `requirement, level 2, will be dropped`.

## rows — pre-close collection and RV foundations

1. Read `task_reqs.rs`, `task_complete/tree.rs`, the managed entry points in
   `capture/task_complete.rs`, and `capture/pomodoro_close.rs::plan_close_successors`.
   The latter also serves past/chain close callers. Inventory every managed Done path
   from `docs/task-dependencies.md` §12.1, including relation-driven explicit
   completions, and distinguish offer-close recursion exclusions.
2. Collect immutable per-owner forests at the same snapshot from which each successful
   open-to-Done transition is planned. Attach them to the close outcome or per-item
   internal result before the original text is discarded. The close order is explicit;
   do not infer it from a sorted identity set. Ensure the old dependency/link-unblocked
   pre-gates cannot suppress a req-only audit. Preserve the no-extra-vault-scan path for
   req-less closes.
3. Implement groups, historical folds, scope counts, parent links, owner-block digests,
   and pending-descendant counts. Respect depth refusal rather than truncating. Nothing
   in this phase writes req state or advertises an enabled public req capability.
4. Create `tests/fixtures/req_close/vectors.json` with a README saying bob-cli is
   authoritative and clients copy bytes. Give cases stable RV identifiers, a schema
   revision, family, inputs, and expected outputs. Add the Rust fixture loader. Test
   three depths, mixed indentation, notes interrupting ancestry, nested real-task
   owners, recurring exclusion, resolved folds, flagged marks, no pending rows, multiple
   closing owners, duplicate labels and block hashes.

## choices — selection and validation

1. Add pure verdict requests and validated selections using the rows contract. Preserve
   owner and row provenance with each request. Cover toggle/fold transitions in portable
   RV cases so JS can reproduce them later.
2. Implement a common numbering view: req groups first in closing order, req rows in
   source order, then ordinary offers in their existing relative order. A probe number
   maps to a key once; final numbering does not shift after carries or drops. No-req
   commands preserve current numbers and output.
3. Resolve carry, complete, cancel-with-reason, pull-in, and jump selectors to their row
   kind. Mark-all selects ordinary eligible offers only. Apply same-row conflicts and
   ancestor overlaps; malformed, stale, reordered, re-indented, duplicate-owner, and
   disappeared identities refuse safely. `req_close_stale` includes fresh groups/current
   numbered rows for recovery.
4. Add RV tests for every verdict pair, siblings, multi-owner disjoint picks,
   dropped/excluded ancestry, same-text branch reorder, ancestor structural choice with
   descendant pin, and ordinary-offer/req mixed numbering. This phase supplies internal
   APIs; do not expose public mutation flags before the write and placement paths exist.

## writes — staged carry/drop adapter

1. Reuse `task_split::carry_req`, `ReqRef`, `CarryOutcome.plan`, and its line moves. The
   helper permits open marks beneath a newly closed owner; no need to reopen that owner
   or stamp a req. Reuse `task_block_layout::writers::drop_req`, including its
   expected-line guard and standard Cancel Log. Never implement alternate carry or log
   writers.
2. Add a req write pass beside the existing offer-verdict pass. Use
   `capture/relations_merge.rs::load_staged_snapshot` and its file-set/read guards,
   loading the full vault only when a carry needs link repair. Apply every returned
   split postimage through `CaptureBatchPlanner`; never write directly to disk. Drops
   need only their guarded note text.
3. Validate against the frozen pre-close forest before mutation. Then apply disjoint
   carries bottom-to-top, drops next, and owner-close bookkeeping before later link
   placement. Compose coordinates after each edit and preserve changed-file preimages.
   Existing close code may stage its own status first; retain equivalent final ordering
   without making a carried task part of a subsequent blanket close. Known internal
   transformations must not be mistaken for external staleness, nor accepted by label
   matching.
4. Return outcomes plus moved-line and created-task identities for placement and
   reporting. Excluded/carried descendants must not be counted as Met. Preserve repair
   of external anchors, daily-owner destination rules, NOTES, line endings, and
   independently dropped/carried history.
5. Extend RV with exact bytes for depth-1 and nested leaf/branch carries, disjoint mixed
   carry/drop, notes with child reqs, anchor reuse and repair, cross-note daily carry,
   no-op Met, and optional Cancel Log. Add transaction tests that inject stale
   source/destination/repair-file changes and file-set changes; all touched and
   untouched files must retain their original bytes on refusal. No separate commit or
   partial-success path is allowed.

## placement — pull-in and jump

1. Reuse `task_complete::successors::insert_successor_links` and §12.3 anchor
   resolution, including ancestor, continuation, and implicit anchors. Preserve any
   existing destination override interface; do not invent a new public CLI destination
   flag. Require a valid destination for carry-pulled: refuse the whole batch when
   unavailable instead of silently downgrading.
2. Link the created task once, raise it to Next, and classify it as created in this
   capture so existing freshness logic does not stamp it. Ensure type agreement, inbox
   routing, ledger retirement, and later batch items use the final task identity and do
   not remove or duplicate its link.
3. Extend jump planning with req-aware final identity/line resolution. Compose carry
   descendant moves, preceding drop-log insertions, other disjoint carries, and
   inbox-route moves. An ambiguous or lost pinned target refuses before commit; it must
   never land on a same-text neighbor or a req stub when the carried task was requested.
4. Add RV and focused planner tests for every landing kind, a pinned child moving with
   an ancestor, mixed task-offer/req cards, multiple carries in one note, routing, daily
   destinations, no destination, and proof that jump alone changes neither verdict nor
   bytes. Verify carry-pulled has Next and no `[fresh::]`/`[unblocked::]` stamp;
   ordinary existing offers keep their established freshness behavior.

## capture — public integration

1. Wire the completed modules through `CaptureRequest`,
   `plan_with_pull_in`/`plan_with_offer_verdicts`, `plan_capture_batch`, per-item
   closing results, and `CaptureResult`. Refactor the repeated probe/final paths only as
   needed for one coherent selection flow. Revalidate across probe/final and the net
   post-pass; later draft items must not redirect a req verdict or cause it to execute
   twice.
2. Add public `-k, --carry <PICKS>` and hidden internal `--req-choice`. Keep public
   options alphabetical by their actual long names: carry belongs after cancel-offer,
   despite the parent plan's contradictory placement sentence. Extend `-x`,
   `-X N[=why]`, `-p`, and `-j` by row kind. Update flag guards, parsing, completion
   metadata, help examples, and dry-run behavior. No new capture subcommand and no
   public flag without a short alias.
3. Publish the groups, rows, capability counts, receipts, current-row refusal payloads,
   and jump result described above. Use optional/defaulted additions on existing
   structures; ordinary output remains compatible. Suppress req row participation in
   legacy-client selection as specified. Enable `pull_in.reqs.supported` only now that
   the whole path works.
4. Render requirements first in human previews with correct indentation, capped glyphs,
   folded history/branch lines, numbers, colored copy, and outcome receipts. Ensure
   receipts reflect the actual committed outcome or clearly describe a dry run.
5. Add CLI tests under `tests/cli/capture/` and register modules in `mod.rs`. Cover
   direct task Done, embedded/multi-owner Done, supported Pomodoro and relation
   completion paths, req-only choice-required, explicit acknowledgement, legacy-client
   fallback, mixed offers, keyed and numbered flags, `-p all`, conflicting choices,
   stale keyed replay in a second process, and dry runs. Include cancels, no-req tasks,
   offer-close recursion, and ordinary offer numbering as negative/regression cases.
6. Run `just docs-gen` for the CLI surface and update the JSON reference enough to
   publish the exact usable protocol. Add a note on this child phase with the contract
   locations and fixture revision for the parent land agent to pass to the dependent
   client phases. Use `!` in the commit subject for the changed CLI/JSON contract; the
   host finalizer performs commits.

## contract — acceptance and documentation

1. Update `docs/task-dependencies.md` §§12.5, 12.13 (picker v6 contract), and 12.14; add
   §12.25 Requirements on the successor card and §12.26 RV vectors. Link these from
   `docs/task-requirements.md` §7 and its fixture roster. Finish
   `docs/reference/json-outputs.md` and generated CLI docs. Describe plugin v6 as a
   consumer contract until its separate phase ships; do not claim this child epic
   deployed either client.
2. Make the field types, optional omissions, discriminator compatibility, row-key
   opacity, preimage binding, legacy handshake, owner order, numbering, folded counts,
   conflicting choices, refusal codes, and exact receipt/jump shapes explicit. Include a
   JSON example of a req-only close and a mixed card with a carried branch. Defaults
   never stamp requirement completion.
3. Complete the portable RV inventory: row building, all write bytes at multiple depths,
   selection state/folding, conflicts, stale duplicated text, re-indent, jump move maps,
   and every canonical string. No network, real vault, or UI is needed. Existing
   RQ/RL/RW/SP fixtures stay authoritative for their respective primitives rather than
   being duplicated wholesale.
4. Run focused requirement/close tests and relevant existing successor suites
   (`successor_offers`, `successor_verdicts`, `successor_jump`, relation successors,
   compatibility, and Pomodoro successor/selection tests). Then run
   `cargo fmt --all -- --check`, the canonical `just check`, `just docs-gen`, and
   `just docs-check`. Use `/sase_monitor` for long commands and wait for its staging
   command to exit; never finish with a raw running subprocess. Capture the final
   verification evidence and exact tree revision.
5. A failure reproduced identically on a clean base is recorded as `PROPOSED FOLLOW-UP:`
   with any existing tracking ID; it does not prevent completion of this phase. Fix new
   failures caused by this work. Do not create new task beads or claim checks passed
   when they did not.

## Landing and bead ownership

Each worker closes only its assigned child phase after checking its own
`sase bead epic-symbols` and resolving or re-keying all temporary exemptions. Do not add
permanent dead-code suppressions to bridge these sequential phases. If an interim
exemption is necessary, key it to a still-open integrating phase or child epic, with
removal in the consuming phase.

The **child epic's land agent** owns the final combined review and the completion
handoff for **bob-cli-on.8**. After its child phases and its own epic settlement are
complete, verify the whole assigned parent-phase scope, run
`sase bead epic-symbols bob-cli-on.8`, remove/re-key any remaining exemptions, and close
**bob-cli-on.8 only** with a note containing the implementation and test evidence under
the land agent's host authority. Individual child workers do not close that ancestor.
Never close `bob-cli-on` or any ancestor plan bead. The parent epic's land agent owns
broader client integration, migration, memory, and live-vault acceptance. Keep
bob-cli-on.8 in progress during this proposal; there is no manual reopen or status
update.
