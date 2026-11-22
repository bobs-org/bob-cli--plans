---
tier: epic
title: Obsidian requirement close card and carry-now
goal: Complete bob-cli-on.11 with guarded requirement carry, successor picker v6,
  ordered verdict writes, one Undo, and final jump landing across managed Done gestures.
phases:
- id: carry_api
  title: Carry a requirement through the navigation split API
  size: medium
  depends_on: []
  description: 'carry_api: extend the Task Split twin through SP50, publish taskSplit
    v1, and route Ctrl+Shift+T on requirement lines through guarded carry with jump
    history.'
- id: req_model
  title: Port requirement close snapshots and verdict validation
  size: medium
  depends_on: []
  description: 'req_model: add pure requirement group, identity, choice, fold, and
    stale-validation twins pinned to RV1-RV19, with copies for navigation and the
    cycler.'
- id: picker_v6
  title: Render and stage requirement verdicts in the successor picker
  size: medium
  depends_on:
  - req_model
  description: 'picker_v6: add requirement groups, contextual keys and reasons, branch
    folds, navigation-only pins, and reqs results to the navigation successor picker
    v6.'
- id: req_apply
  title: Apply requirement carry and drop verdicts with guarded Undo
  size: medium
  depends_on:
  - carry_api
  - req_model
  description: 'req_apply: build the cycler carry/drop service using the navigation
    carry API and existing requirement writers, preserving preimages, receipts, move
    maps, and one guarded Undo.'
- id: req_delivery
  title: Place carried tasks and resolve final requirement landings
  size: medium
  depends_on:
  - req_apply
  description: 'req_delivery: adapt successor placement for carried tasks without
    freshness stamping and resolve requirement pins through composed move maps and
    final note routing.'
- id: close_wiring
  title: Connect managed Done paths and verify the complete card flow
  size: medium
  depends_on:
  - picker_v6
  - req_delivery
  description: 'close_wiring: capture pre-close scope on all managed Done paths, integrate
    one requirement-first card and ordered apply, resolve jumps last, finish contract
    docs, and verify both deployed plugins.'
proposed_by: bbugyi200.athena.bob-cli-on.11
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-on.11
create_time: 2026-11-22 14:59:29
status: wip
bead_id: bob-cli-on.11.1
---

- **PROMPT:** [prompts/202611/obsidian_requirement_close_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_requirement_close_card.md)
- **PARENT:** [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)
- **BEAD:** [bob-cli-on.11.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.11.1.md)

# Obsidian requirement close card and carry-now

This child epic implements **bob-cli-on.11**, the `obsidian-close-card` phase of
`plan:202611/task_requirements.md`. Its land agent verifies and closes **bob-cli-on.11
only**, after its own child phases settle. No worker closes `bob-cli-on` or another
ancestor. Let SASE create and manage the child epic and phase beads through plan
approval; do not create them manually.

## What has landed and why this is an epic

The assigned phase was `in_progress` with no previous implementation notes when read on
2026-11-22. Its prerequisites `bob-cli-on.8` and `bob-cli-on.10` are closed. The
inspected bob-cli tree at `96a23f50` already contains Rust carry, requirement close
collection, verdicts, write planning, placement, landing, and the capture protocol.
Reuse those contracts, rather than implementing them again.

The opened bob-plugins tree at `2b3b6b6` includes the requirement recognizer/layout/
writer twins, gesture guards, drop/restore/toggle, typing assist, and real-Obsidian
requirement scenarios. It does **not** contain this phase's implementation:

- `256-task-split-plan.js` still declares SP1-SP31 parity; no `api.taskSplit` exists.
- `206f-successor-picker-api.js` advertises v5 and discards a result with no ordinary
  picks, closes, or jump. `206a` does not whitelist `requirements`.
- The cycler has offer-close and jump integration but no requirement close service.
- `256` has 972 lines, `257` 876, and cycler `089i` 986. Extend through new fragments.

This is six bounded coding units, not one medium tale: carry writes, the pure
snapshot/choice model, interactive picker presentation, multi-file apply, placement and
landing, and managed Done integration. The first two can run independently. Picker and
apply can run after their listed dependencies; delivery follows apply, and the final
wiring phase is serial and owns integration. All phase dependencies are declared above.
Before editing, refresh through the normal repository workflow and check whether any
scoped work has since landed.

## Authority and fixed behavior

Read the parent plan and its two endorsed sources through audited artifact reads:

- `research:202611/task_reqs_design/task_reqs_design.md`
- `research:202611/task_reqs_nested_and_notes_impact.md`

The amendment permits nesting; the parent plan resolves the remaining choices. The
landed contract is `docs/task-requirements.md` §§6-7 and `docs/task-dependencies.md`
§§12.13, 12.25-12.26. Rust and its SP/RV fixtures govern exact bytes, row identity,
copy, and refusals where old planning prose is abbreviated.

The parent decisions remain final: `notes_drawer_birth = with_first_note`,
`seed_review = yes`, and the three approved memory decisions. **This child epic edits no
memory.** Migration, promotion, Mac UI, and the parent's memory phase remain outside
this scope. The `NOTES` drawer appears only when it receives a note.

A requirement is a non-`#task` checkbox whose structural parent is its real task or
another requirement. Notes, drawers, fences, quotes, and nested real tasks interrupt
ownership. Requirements can nest to the existing depth bound. Pending requirements are
Met with a Done owner without any requirement completion write. Explicit legacy `[x]`,
dropped `[-]`, carried `[>]`, and excluded descendants keep their semantics.

Carry creates one task for a disjoint selected branch, moves the entire subtree, reuses
a trailing anchor, rebases indentation, leaves a `[>]` link stub, and adds the existing
owner mention backlink. No inheritance or dependency is invented. The task goes after
the owner's block, or through existing daily-note routing. Attached notes stay with
their branch and follow the task-level NOTES rule.

One post-close successor card audits all eligible non-recurring owners in the successful
managed Done set, including embedded real-task closes. Requirements come first, in owner
close order and then document order, and alone can open the card. Never open a recursive
audit for offer-close verdicts, cancels, parked events, automated closes, reconcile, or
raw edits. Enumerate from actual pre-close snapshots; reading only the now-Done text
would hide the Pending rows.

Row keys are opaque snapshot-bound identities, not labels or line numbers alone. Follow
the Rust owner key, sibling path, clean-text digest, and full owner-block preimage.
Reorder, re-indent, and edited duplicate labels refuse stale choices.

## Cross-phase interfaces

Agree these seams in code and tests; use ordinary repo-relative paths throughout.

1. Navigation `api.taskSplit` v1 provides
   `carryReq({notePath, line, expectedLine}) -> Promise<result>`. `line` and returned
   task `line` are **1-based**, matching SP/RV; convert once at the editor boundary.
   Success includes `{ok: true, task: {path, line, blockId, text}, moveMap, restores}`.
   Refusal includes `{ok: false, code, message}` and any honest partial-write recovery
   detail. `moveMap` identifies source and destination paths and 1-based line moves;
   `restores` contains exact before/after snapshots and `created` flags for every
   changed file, including repaired inbound links. These additive fields support the
   card's batch Undo; the required `task` and `moveMap` contract is unchanged. Card
   callers additionally supply `expectedOwnerPreimage`, an optional exact current
   owner-block guard checked before planning and held through persistence.
   `expectedLine` alone cannot catch a changed descendant between the cycler's preflight
   and the API's read; do not silently accept a newer block in that gap. The API itself
   does not navigate or emit an independent Undo notice. The standalone Ctrl+Shift+T
   command owns its notice/navigation; the card owns the batch receipt.
2. The pure requirement model accepts the source notes and ordered successful owner
   identities, returns canonical RV owner snapshots and requirement groups, and
   validates choices against those snapshots. Use identical, tested choice/fold helpers
   in nav and cycler; copy shared fragments with a parity test, as the existing
   requirement twins do. The cycler owns collection; nav consumes already collected
   groups and never derives owner state from rendered text.
3. Picker v6 preserves `picks`, `closes`, and `jump` and adds
   `reqs: [{key, verdict, reason?}]`, with verdicts `met | drop | carry | carry_pulled`.
   Canonical keys remain unchanged. Default Met rows need no entry; a req-only explicit
   result must survive every API filter. Keep explicit Met distinct from unstaged
   default when validating conflicts. Folded descendant choices are retained for
   unfolding but excluded from the final active structural selection while their
   ancestor owns the branch, matching RV.
4. The apply service consumes validated choices, the pre-close audit snapshots, exact
   expected **post-close** live snapshots, and existing destination/close context. It
   returns actual applied outcomes, failures/refusals, merged restores, created-task
   identities, and move information. It does not open another picker or navigate. Final
   wiring owns the one receipt and final landing.

Retain the current post-close timing. The initiating managed Done may already have
written the owner before the card opens. Store both the original audit identity and the
exact post-close expected bytes, including known close bookkeeping; never pretend those
hashes are equal or ignore arbitrary owner-line edits to pass a guard. When the gesture
changes its own text later, compose that known change explicitly. External edits
anywhere in the guarded owner block refuse `req_close_stale`.

## Phase carry_api

Work in bob-plugins navigation only, with bob-cli docs for the API/gesture contract.
Starting points are `256-task-split-plan.js`, `257-task-split-apply.js`,
`653-plugin-task-split.js`, `654-task-undo-notice.js`, `709-task-split-exports.js`, the
lifecycle/API assembly, and existing RQ/RL/RW twins in `694`-`698`.

- Port the landed `src/native/task_split/{plan,carry}.rs` requirement behavior into new
  split fragments. Include candidates at any depth, state/exclusion refusals, overlap
  refusal, branch extraction, anchor repairs, daily placement, `[>]` stubs, NOTES
  filing, and the descendant move map. Preserve ordinary note/drawer split.
- Copy the full `tests/fixtures/task_split/vectors.json` and its provenance into
  navigation's fixture directory. Pass SP1-SP50 without changing expected data to
  accommodate JS. Cover a still-open requirement marker under an already-Done owner: the
  post-close audit must be able to carry it without reopening its owner.
- Extract reusable guarded persistence from `653`; the existing cross-note path writes
  only destination/source, so explicitly persist **all** plan postimages, including
  anchor repair files. Guard editor-first live text, destination existence, and file
  creation; persist destination before source stub. Reuse rollback and
  `restoreGuardedSnapshots`, preserving recovery copies when rollback cannot safely
  remove them. Return truthful success/failure and exact snapshots.
- Install `api.taskSplit` v1 with the interface above. Missing/stale target and
  unavailable writer are typed refusals, not writes. No broad new parsing API.
- Route Ctrl+Shift+T on requirement rows to this API; leave the ordinary picker/count/
  visual behavior intact except for Rust-defined requirement selection/refusals. Land on
  the new task and record Ctrl+O history exactly once.
- Extend split-vector and gesture tests with same-note, daily/cross-note, nested,
  duplicate-label, stale expected line, and failed-write/Undo cases. Test the public API
  rather than merely exporting a helper. Update navigation's reference and API docs.
  Build, test, and deploy navigation per the common gate below.

## Phase req_model

Work in new pure fragments, their manifests/exports, and fixture runners. Avoid editing
runtime successor orchestration in this phase.

- Port `src/native/task_complete/req_close.rs` and the relevant interactive/final choice
  logic in `req_verdicts.rs`. Reuse RQ; do not introduce another recognizer.
- Collect only Pending rows for each successful non-recurring closing owner, including
  unanchored owners and nested real tasks with distinct identities. Preserve exact
  preimages, sibling paths, hashes, clean text, depth, parent key, descendant counts,
  canonical consequences, historical folds, and owner close order.
- Match RV canonical numbering (requirements first), mark-all exclusion, active choices,
  branch fold/unfold, on-row overlap refusal, duplicate owner/key refusal, pin-only
  targets, and stale validation. A pin under an ancestor carry is legal and never
  creates another carry.
- Copy `tests/fixtures/req_close/vectors.json` byte for byte into the consuming plugins
  with READMEs stating "update bob-cli first, then copy". Provide runners for RV1-RV19
  and twin-copy parity; keep all later RV cases intact for later phases.
- Canonical model functions should remain usable with injected snapshots in node tests.
  Register manifests/exports without advertising live requirement support. Build, test,
  and deploy only changed plugins.

## Phase picker_v6

Work in navigation `206*`, new requirement fragments, scoped CSS, and picker tests. The
`req_model` helpers supply behavior; do not create alternate overlap semantics.

- Extend `normalizeSuccessorPickerModel` to recognize canonical `requirements` groups
  and preserve `owner`, `copy`, `folded`, `offer`, `req`, and `consequences`. Adapt the
  ordinary row shell at one explicit boundary. Requirement-only models are valid even
  without ordinary task contexts, destinations, or close descriptors.
- Render the canonical header/scope/caption, historical fold, depth indentation,
  requirement hash run capped at four, owner's ink when available, exact-depth
  accessible labels, and consequence strip. Structural choices show
  `· {k} nested go with it`; restaging Met restores visible descendants/choices.
  Preserve stable selection and filtering when rows fold or unfold.
- Implement the row-specific keys: Ctrl+X Met; Ctrl+Shift+X Drop with
  `Why drop it? (optional)` and Log Recall; Ctrl+Shift+T Carry; Space/Tab Carry plus
  pull-in (Tab advances); Ctrl+J navigation-only pin. Repeating a verdict returns to
  Met. Mark-all never touches requirement rows. Show the canonical ancestor conflict on
  the row rather than silently clearing it.
- Use the existing prose/reason field host and its focus/Escape handling. Keep any
  recently landed task-link prompt support. Footer follows the highlighted row and shows
  `reqs ✓N ⊘N →N`. Default Met is not a staged edit; explicit choices use the current
  two-step Escape behavior. Esc discards choices, with the owner already Done.
- Return `reqs` through modal settlement and both API sanitizers. Preserve req-only and
  pin-only results; ordinary result shapes remain compatible. Bump the API to 6 and
  correct the stale v4 header. A v6 picker without the new cycler remains safe.
- Extend `test-navigation-successor-picker.cjs` and
  `test-navigation-successor-verdicts.cjs`, adding focused requirement DOM/state tests
  as needed. Cover all key transitions, mark-all, folded choices, reason focus/recall,
  one-pin behavior, unload, rapid Enter guard, mixed rows, and req-only settlement.
  Update navigation/API reference docs, build/test, and sync navigation.

## Phase req_apply

Work in new cycler write/apply fragments and the small navigation receipt/ Undo adapter
needed to consume their result. Leave managed Done call-site wiring to `close_wiring`.
Exercise the service through a fake vault and real nav API adapter.

- Use `req_model` to preflight every active choice and the full expected owner postimage
  before starting. Require the caller's completed destination preflight for
  carry-plus-pull-in before mutation; `req_delivery` supplies that adapter. Until that
  adapter is present, refuse carry-plus-pull-in rather than applying a partial choice.
  Keep older nav APIs safe: unavailable `taskSplit` refuses carry, and never substitutes
  ordinary split.
- Carry disjoint branches bottom to top through `api.taskSplit.carryReq`; carry
  destination persists before source stub. After each successful operation, compose the
  expected texts and line maps for remaining choices, including multiple owners in one
  file. Apply drops with the existing RW twin and reason/Cancel Log rules. Met writes
  nothing. Refuse wrong branches after a file shifts or a duplicate label changes;
  revalidation is by snapshot/structure, never label search.
- Return canonical `req_outcomes` and aggregated Met receipts alongside existing
  offer-close receipts. Compose snapshot restores per path from earliest before to final
  after, including created notes, repaired links, and daily edits. The caller can merge
  this with ordinary verdict Undo into one guarded action. Keep the owner Done when
  undoing card verdicts; reopening remains the existing separate gesture. Refuse the
  whole Undo before writes if any recorded postimage changed.
- The apply service is best-effort Obsidian persistence: surface actual failures, never
  report an unapplied carry/drop/link as successful, and never delete the only surviving
  copy after partial failure. A refused structural choice invalidates dependent
  placement and its corresponding created-task landing.
- Run RV20-RV28 through deterministic plan/apply adapters, comparing bytes, copy, and
  identity. Add plugin-specific fault tests for dirty editors, destination/repair-file
  changes, partial writes, composed Undo, and created-file restoration. Rust atomic
  transaction assertions remain Rust's; plugin failure tests assert its explicitly
  best-effort recovery contract.
- Build/test and sync changed plugins. Publish the apply-service seam and exact
  snapshots it requires in the phase's completion note for final wiring.

## Phase req_delivery

Work in new cycler placement/landing fragments using the apply result from `req_apply`,
plus focused placement and landing tests. Do not change managed Done call sites yet.

- Port `req_placement.rs` and `req_landing.rs` around existing successor insertion
  helpers. Provide the required preflight adapter before any carry-plus-pull-in write;
  no valid destination is a refusal, never a silent downgrade. Revalidate destination
  text/existence at persistence and report honest partial failure if it changes later.
- Carry-plus-pull-in links the created task at the existing successor destination, in
  card order, raises it to Next, and never stamps it fresh. Carry alone remains
  unlinked. Preserve slot/inherit/continuation/fallback behavior, destination overrides,
  and ordinary offer semantics; do not add a destination flag.
- Compose moves across sibling carries, drops/Cancel Logs, same-file owners, daily
  routing, and later inbox-route renames. Resolve pins by identity: selected carry ->
  created task, ancestor carry -> moved descendant line, otherwise -> the original
  requirement's final line. Equal labels never substitute for provenance. Pin-only
  results write nothing. The resolver returns a landing; it never navigates itself.
- Merge daily edits and post-placement task changes into the apply service's exact
  snapshots. A later successful edit to a file keeps its earliest before and final
  after; one guarded Undo restores the whole card batch. Failed placement cannot claim
  `pulled: true`; failed carry cannot create a fabricated task landing.
- Run RV29-RV43 through the adapters, including all destination types, several carries
  in card order, no-destination refusal, descendant and same-label pins, drop logs, and
  route renames. Add dirty-destination and placement-failure tests around the real
  service boundary. Build/test and sync changed plugins, then record the orchestration
  interface for `close_wiring`.

## Phase close_wiring

Own the final integration in cycler `130`, `136`, `137*`, `160`, `170`, and jump callers
plus navigation notice/API adapters. Add fragments before growing near-cap files. Find
every caller afresh; the names below are a starting map, not the audit.

- Audit `finalizeClosedTasks -> planAndApplySuccessorsNow`, direct/embedded Ctrl+Enter
  closes, status-cycle batches, Pomodoro task-link closes, API callers, and
  `137c-successor-jump.js`. Capture original owner blocks **before** each initiating
  mutation; attach only owners actually closed, in successful close order, with exact
  expected post-close snapshots. Carry this context through queued reference
  finalization without re-reading it as a pre-close snapshot.
- Attach groups only for `api.successors.version >= 6` and eligible managed Done. Ensure
  requirement-only closes bypass existing early dependency/offer gates such as missing
  task dependency IDs or no normal offers. Keep one card for the entire gesture.
  Older/missing pickers leave requirements implicitly Met and retain the previous
  ordinary-offer behavior.
- Merge requirement groups first into the existing picker model. Route `reqs` to the
  apply service. Respect the established post-close timing, then sequence requirement
  carries, drops, remaining close bookkeeping, pull-in links, and final jump; keep
  recovery and ordinary offers intact. No new audit for tasks closed by the card's
  ordinary offer-close verdicts. Refresh identities/maps after structural edits so
  existing offer writes cannot target pre-carry line offsets.
- Resolve the one jump only after every write/route settles: carried row -> created
  task; descendant -> composed move map; otherwise -> the same requirement's final line.
  Do not use the task-only fallback for a requirement landing. Preserve Ctrl+O and
  `decisions:successor-jump-parks-the-walk`: a pinned walk close parks at the closed
  row's neighbour, with no intermediate advance or second landing.
- Add requirement outcomes to the existing receipt and one guarded Undo. A req-only card
  must produce its receipt without `unblocked` or `offer_closes` rows. Esc writes no
  requirement bytes and leaves the owner Done; ordinary reopen still works.
- Cover the actual public paths with integration tests: no dependency ID, req-only,
  mixed offers, embedded multi-owner close, cross-note daily carry, same-file multiple
  carries, ancestor carry plus descendant pin, duplicate labels/stale indentation,
  missing/older API, recurring/cancel/automated/offer-close exclusion, and guarded Undo
  after a late user edit. Reuse/extend the real-Obsidian requirement e2e harness and
  record any environment restriction honestly.
- Finish `docs/task-requirements.md` §§6-7 and its obsolete interim Ctrl+Shift+T text,
  `docs/task-dependencies.md` §12.13 and the v6 shipping statement, and bob-plugins
  navigation/cycler references plus `docs/develop/plugin-apis.md`. Describe the exported
  result/line units and same-day reopen/Undo behavior precisely.
- Run the final common gate, deploy both plugins, and leave evidence for the land agent.
  This coding phase closes only its own child phase bead.

## Common verification and deployment

Open bob-plugins with `/sase_repo` before touching it and read its AGENTS.md. Edit
`src/` and `fragments.json`, never generated `main.js`; every hand-edited fragment stays
at or below 1000 lines. Add new test files to the explicit `test:unit` list. Do not edit
Cargo.toml version or CHANGELOG.md. Contract breaks follow the existing release-train
`!` convention; no new CLI flags are needed here.

Each implementation phase runs its focused vector/behavior tests, `npm run build`, and
`npm test` in bob-plugins. Run `npm run docs:gen` when inventory changes and
`npm run docs:check` for plugin docs. For bob-cli doc edits, read `docs.md` memory, run
`just docs-gen`, `just docs-check`, and the repository's required check gate. Final
integration runs `just check` plus the plugin gates on the combined tree. Long checks
use `/sase_monitor` as required by the single-turn environment.

After plugin changes, deploy only the changed IDs:
`bob plugins sync -p bob-navigation-hotkeys` and/or
`bob plugins sync -p task-status-cycler`. Never force. If sync protects a newer
checkout, follow the opened repo's documented reconciliation procedure. Do not edit the
vault's installed plugin sources. This epic performs no bulk vault migration.

The known separate task `bob-cli-p8` concerns project offer-close scope counting of
`#ref` rows in `089h`. Preserve adjacent work; do not expand this epic to solve it. Any
newly discovered unrelated work goes only into a
`PROPOSED FOLLOW-UP: <summary — evidence>` note on the assigned phase bead, never a new
task bead. A check failure reproduced identically on the clean base is recorded with the
existing task ID when known and does not leave a completed coding phase open.

## Land-agent acceptance and closure

The child epic's land agent checks every SP/RV family, the public carry API, every
eligible Done path, exact staleness guards, no req completion writes, and one final
jump/Undo using the combined implementations. Confirm both plugins deployed and document
any actual test limitation. Do not claim live five-task testing unless it was performed;
use the existing isolated Obsidian fixture vault for automated runs.

Before closing each child phase, run `sase bead epic-symbols <its-id>` and resolve or
re-key leftovers to an open owner. Before completing the assigned outer phase, the land
agent runs `sase bead epic-symbols bob-cli-on.11` again (there were no entries at
planning time), then closes `bob-cli-on.11` with a note naming verification and deploy
evidence. That is this child epic's completion obligation; never close `bob-cli-on` or
any other ancestor. This planner leaves bob-cli-on.11 in progress until the child
implementation and verification finish.
