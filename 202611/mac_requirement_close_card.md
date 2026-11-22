---
tier: epic
title: Requirements on the Mac Pull-In card
goal: Complete bob-cli-on.15 so the Mac Pull-In card presents nested requirement groups,
  stages their verdicts safely, and submits them through the existing capability-gated
  bob contract with compiled and tested native UI.
phases:
- id: protocol
  title: Decode requirement cards and encode gated verdict arguments
  size: medium
  depends_on: []
  description: 'protocol: add typed requirement capability, group, row, consequence,
    and receipt models in bob-mac-capture, canonical collection helpers, and capability-gated
    BobProcessClient arguments. Generate synthetic fixtures from current bob and cover
    old payloads and requirement-only responses. Follow the protocol section and shared
    verification rules below; do not activate the card yet.'
- id: outcomes
  title: Stage requirement verdicts and preserve branch identity
  size: medium
  depends_on:
  - protocol
  description: 'outcomes: extend the portable Pull-In outcomes and the choice model
    with typed requirement rows, Met/Drop/Carry/Carry-and-pull verdicts, overlap refusal,
    branch folding, owner grouping, navigation-only pins, refresh, and commit projections.
    Follow the outcomes section and test its transition matrix, including default-Met
    commits and Shift-Return downgrading.'
- id: pipeline
  title: Preserve requirement choices through preview and submission
  size: medium
  depends_on:
  - outcomes
  description: 'pipeline: thread requirement capability and staged choices through
    CapturePanelModel snapshots, guarded dispatch, every refusal/retry and inbox-route
    path, all offer collectors, preview call sites, and notification receipts. Requirement
    groups alone must open the card. Follow the pipeline section and test stale recovery,
    executable identity changes, and post-commit jump behavior.'
- id: card_ui
  title: Render requirement branches and wire their keyboard actions
  size: medium
  depends_on:
  - pipeline
  description: 'card_ui: render requirement sections first with bob-authored copy,
    depth glyphs, branch folds, consequences, scope summaries and accessible states.
    Wire Ctrl+X, Ctrl+Shift+X, Ctrl+Shift+T, Space/Tab and Ctrl+J through the existing
    router, controller and reason/recall UI. Preserve ordinary offer behavior, text
    focus, mark-all exclusions and Esc semantics; follow the card_ui section.'
- id: integration
  title: Verify complete Mac requirement flows and publish their documentation
  size: medium
  depends_on:
  - card_ui
  description: 'integration: exercise the full fake-bob and current-bob synthetic-vault
    acceptance matrix, finish cross-surface regression coverage and requirement receipts,
    update Mac keyboard/preview/protocol docs and the bob-cli requirement contract,
    and run portable, documentation, scratch-Mac and exact-SHA macOS gates. Record
    completion evidence for the child epic land agent; do not close ancestors.'
proposed_by: bbugyi200.athena.bob-cli-on.15
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-on.15
create_time: 2026-11-22 14:56:47
status: wip
bead_id: bob-cli-on.15.1
---

- **PROMPT:** [prompts/202611/mac_requirement_close_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_requirement_close_card.md)
- **PARENT:** [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)
- **BEAD:** [bob-cli-on.15.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.15.1.md)

# Requirements on the Mac Pull-In card

This is a child implementation epic for **bob-cli-on.15**. Closing a task with pending
requirements must show those requirements at the top of the Mac Pull-In card. The user
can accept them as met, drop a branch with an optional reason, make a branch its own
task, or make it a task and pull it in. All writes still belong to one guarded
`bob capture` call.

The five phases are sequential. They separate wire compatibility, branch selection,
process orchestration, native UI, and integrated verification so each phase fits one
coding agent. Each worker closes only its own assigned phase. The child epic's land
agent verifies the combined result and then closes **bob-cli-on.15**; it must not close
**bob-cli-on** or any other ancestor. Individual workers prepare evidence for that land
action and never interpret it as permission to close their ancestors.

## Evidence and fixed scope

Read these through `sase artifact read` when needed:

- `plan:202611/task_requirements.md`, especially §7 and `mac-close-card`.
- `research:202611/task_reqs_design/task_reqs_design.md`.
- `research:202611/task_reqs_nested_and_notes_impact.md`; its nesting and Mac Esc
  clarifications supersede the original report.

The implementation contract is bob-cli `docs/task-dependencies.md` §§12.24–12.26 and
`docs/task-requirements.md`. Rust's `tests/fixtures/req_close/vectors.json` and README
pin the RV cases and copy. Use the shipped snapshot-bound keys, not the original
research's obsolete text-hash-only identity example.

At planning time bob-cli HEAD is `96a23f50`: the close contract landed in `7073b7c8` and
its wire/vector documentation in `96a23f50`. The Mac HEAD is `8562f811`; marker editing
(`e1a55464`) and arbitrary even-depth indentation (`8562f811`) already landed.
`bob-cli-on.8` and `.14` are closed. The assigned `.15` has no prior implementation
notes and remains in progress. Its epic-symbol query currently reports none. Recheck
current trees and notes before changing code.

The Mac currently decodes neither requirement groups nor requirement receipts.
`CapturePullInGroup` has metadata but no typed requirement rows; `CapturePullInOutcomes`
stores only task-offer Done/Cancel verdicts. Its default highlight fallback and
`confirmPullInChoice` require special care: a highlighted requirement defaults to Met,
and an all-default requirement card must still commit. `PullInSectionIdentity` lacks an
owner discriminator and would merge multiple requirement owners.
`makePullInPreviewSnapshot` and several view/notification paths collect task offers
independently, so simply extending one decoder is insufficient.

The parent decisions are final: `notes_drawer_birth = with_first_note`,
`seed_review = yes`, and the three approved memory decisions remain with the parent's
memory phase. This child epic edits no memory. It does not implement vault migration,
requirement recognition/writes, Obsidian behavior, or a new glyph design. Use the
accepted capped hash run already used by Mac marker rendering.

## Shared implementation and verification rules

- Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`. Repository paths
  below are relative to the indicated repository; do not use another agent's checkout.
  Read the required reference memory through `/sase_memory_read`, including `docs.md`
  for documentation changes and `decisions:mac-capture-is-a-thin-client` and
  `decisions:successor-jump-parks-the-walk` for their governing contracts.
- Bob supplies ownership, depth, eligibility, ordering, destinations, consequences,
  snapshots, receipts, and final jump landing. Swift stages user choices and presents
  that data; it never reads or mutates vault Markdown or ports Rust's requirement
  recognizer. Treat owner and row keys as opaque.
- Add focused files instead of expanding giant model/view files unnecessarily. Keep
  existing initializers source compatible with defaults. A missing optional feature must
  not make a legacy command response undecodable. A malformed new row must not discard
  unrelated task offers or become a selectable ordinary task.
- Every phase runs its relevant tests and `just portable-check` in the Mac repo. Format
  only touched Swift lines. Changes to `Sources/BobMacCapture*` or
  `Tests/BobMacCapture*` also require a best-effort Mac scratch build: read tailnet
  memory, rsync a scratch copy excluding `.git` and `.build`, run `Scripts/bundle.sh`
  and `just typecheck-tests`, and remove the scratch copy. Never use Bryan's checkout or
  install there. Linux portable tests cannot verify AppKit/SwiftUI targets; do not
  report them as doing so.
- Never land uncompiled Swift. Run `Scripts/macos-ci-check` from the Mac repo before
  landing Swift, using `/sase_monitor` for the long command. After every push, watch
  that exact SHA's macOS run to green through `/sase_monitor`. A scratch build is not a
  substitute for CI test execution. If the Mac is offline, explicitly record the scratch
  limitation and still obtain macOS CI.
- Use `sase monitor start` before long commands; wait for the staging command to exit,
  then finish through `/sase_final`. Do not leave an ordinary subprocess running at turn
  end. Use `/sase_final` to commit every changed repository.
- For docs run `just docs-check` in the owning repo. The final integration phase also
  runs bob-cli `just docs-gen` and `just docs-check`, as the parent requires. Do not
  edit release versions or changelogs.
- Record out-of-scope work as `PROPOSED FOLLOW-UP:` on the worker's assigned phase,
  never create task beads. A failure identical on the clean base is documented with its
  evidence and existing tracking bead, if any; it does not leave a finished phase open.
  Query `sase bead epic-symbols <assigned-id>` before closing; resolve entries or re-key
  them to an appropriate still-open bead.

## protocol

Own the portable wire and process boundary in `Sources/CaptureCore`:
`CaptureModels.swift`, a new focused requirement-model file,
`CaptureSuccessorOfferRows.swift`, and `BobProcessClient.swift`.

1. Decode `pull_in.reqs {supported,pending}`, group `kind` (fallback `group`), owner
   identity, group copy (`scope`, `history`, footer included), requirement rows,
   consequences, and `req_outcomes`. Use distinct requirement row models: requirement
   rows lack required task status fields and cannot simply be decoded as
   `TaskCompleteUnblocked`. Expose their offer key/number, owner key, structured sibling
   path, parent key, exact depth, pending-descendant count and text. Decode final
   carried-task identities and bob-authored receipt messages.
2. Add a canonical requirement collector that reads `pull_in.groups[].rows` from
   normalized success/batch and refusal-current payloads as applicable. Preserve Bob's
   group order and deduplicate mirrored root/item data by complete opaque identity.
   Requirements must never appear in `allUnblockedRows` as recovered tasks. Test actual
   wire placement instead of inventing nested fields.
3. Add a typed requirement submission value and a separate positive capability proof to
   `BobProcessClient.capture`. Known support adds `--req-choice` to previews, guarded
   submits, and final verdict submissions. Support is tied to the same executable
   identity and preview as existing capabilities.
4. Encode Met via no structural flag (or explicit `--complete-offer` for a staged
   explicit Met), Drop via repeated `--cancel-offer key[=reason]`, Carry via `--carry`,
   and carry-plus-pull via `--pull-in`. **Never send both `--carry` and `--pull-in` for
   the same requirement**: Rust refuses that combination. Combine ordinary task verdicts
   and requirement verdicts without losing their separate capability checks. Preserve
   opaque keys and one argv value per reason; never shell-interpolate. An explicit
   no-pull choice uses `--pull-in none`.
5. Old Bob must receive no new flags. Staged requirement actions without support fail
   locally rather than being discarded or retried as an ordinary capture. Do not strip
   `--req-choice` in a compatibility retry while retaining req keys.

Tests extend the decoding, client argv, and offer-row suites. Use a current bob binary
from this epic (build it when necessary), never assume the daily release already
includes the contract. Follow `Scripts/generate-offer-close-fixtures.sh` to generate
deterministic requirement-only, mixed, legacy, write and stale fixtures from a temporary
synthetic vault; record command and source provenance. Copy RV JSON byte for byte into
the Mac fixtures with a README directing changes to bob-cli first. Exercise
client-relevant rows/choices/copy cases, not vault-write algorithms in Swift. No card
activation belongs to this phase.

## outcomes

Own `CapturePullInOutcomes.swift`, focused portable requirement selection helpers, and
`CapturePullInChoice.swift`'s model. Introduce a typed row abstraction with task and
requirement cases, preserving convenient legacy task-only initializers. Do not
manufacture task status, block IDs, or destinations for requirements.

1. Add requirement Met, Drop(reason), Carry, and Carry-plus-pull selections. Default Met
   is distinct from the ordinary task-offer highlight-to-pull fallback. Repeating a
   verdict returns to Met; another verdict replaces it. Ctrl+X on a requirement restores
   Met and never means task Done. Mark-all/toggle-all touch eligible ordinary task
   offers only and preserve requirement choices.
2. Use Bob's parent metadata and structured sibling paths within one owner to check
   branch ancestry; a manually met intermediate parent may be absent from listed pending
   rows. Compare path components (1 is not an ancestor of 10), never parse row keys or
   infer ancestry from text/indentation. Refuse overlaps in either staging order,
   retaining all previously staged choices. A rejected ancestor action must not silently
   hide an already chosen descendant.
3. A successfully staged Drop/Carry folds descendant display rows into
   `· {k} nested go with it`; Met unfolds them. Filtering, group collapse and refresh
   must not erase selection state. A navigation-only descendant jump pin remains valid
   while its ancestor is carried or dropped. At most one pin exists.
4. Include owner identity in section identity and row matching. Preserve closing owner
   order, Bob's row numbers, canonical group copy and historical folds. Requirement
   sections precede ordinary groups; history is not actionable scope.
5. Compute the row-aware semantic summary using RV rules: descendants excluded by staged
   structural choices are not counted as met. All-default requirement cards are
   committable. Return never silently pulls the highlighted requirement. Provide a
   commit projection that downgrades carry-plus-pull to Carry for Shift-Return, keeps
   Drop/ordinary verdicts and the jump pin, and selects no ordinary pull-ins. Do not
   mutate staged UI choices merely to project arguments.
6. Refresh by exact complete keys. Preserve surviving marks, verdicts, reasons,
   highlight and pin; report removed choices. A changed owner preimage means old keys
   cannot transfer by label or sibling path. Revalidate any surviving branch choices.
   Default Met does not arm Esc; explicit structural choices do. Escape in a reason
   editor restores the previous choice without arming card discard.

Tests cover each verdict transition, reason cancellation, both overlap orders, disjoint
owners/branches, absent intermediate pending parents, fold/unfold and filter
interactions, mark-all, all-default commit, Shift-Return, duplicate labels, stale
refresh, hidden descendant pins and legacy offer-only behavior. Use the RV choice/copy
expectations. Compile the native choice-model tests as required.

## pipeline

Own `CapturePanelModel.swift`'s request contexts and orchestration plus the shared
collector call sites in `CapturePanelView.swift` and `NotificationService.swift`.

1. Carry requirement groups and their capability through `CapturePullInPreviewSnapshot`,
   `makePullInPreviewSnapshot`, card creation, fresh refusals, and the
   submit/inbox-route contexts. Once a normal dry-run preview proves `reqs.supported`,
   the guarded submission must include `--req-choice`: the legacy
   `--require-pull-in-choice` handshake alone deliberately omits requirement groups. A
   requirement-only close must open the pre-write card.
2. Audit every use of `allUnblockedRows`, `allKeepMovingRows`, relation/note-pull
   collectors, group rows and unique-offer helpers. Consolidate collection where
   practical so task close, current/past Pomodoro close, complete-label relations, mixed
   drafts and batch root mirrors all preserve requirements exactly once. Recurring or
   ineligible closes remain governed by Bob's returned groups.
3. Route card commits through the typed outcome projection. Preserve choices,
   capabilities, seed, snapshot, executable identity, and pin through move-in,
   inbox-route selection/back, refusal refresh, and retry. Invalidate on draft,
   generation or executable changes rather than submitting old keys to a new bob.
4. Handle `req_close_stale`, `req_verdict_overlap` and `offer_verdict_conflict` without
   hiding the draft or resubmitting defaults. Decode the `current` recovery payload and
   report lost choices on refreshed cards. Keep the existing refusal behavior for
   ordinary offer-close paths.
5. Return with untouched requirements commits default Met. Shift-Return invokes the
   no-pull projection. Esc retains the complete draft and writes nothing; an edited
   requirement participates in the existing two-step discard flow. Native view actions
   added by the next phase call these same model methods.
6. On success render `req_outcomes.message` alongside task receipts, including
   requirement-only captures, and avoid duplicating mirrored batch receipts or counting
   requirements as recovered tasks. Use Bob's final `jump.landing` only, after the whole
   write succeeds; it wins over open-notes navigation. A null landing has no text-based
   fallback.

Test fake-process transcripts for initial support discovery, legacy refusal then capable
request, requirement-only commit, mixed verdict flags, stale-current refresh, executable
replacement, inbox-route back/submit and carried-descendant jump. Exercise preview,
success, and notification call sites explicitly.

## card_ui

Own `CapturePullInChoiceView.swift`, presentation helpers, the Pull-In branch of
`CaptureKeyCommandRouter.swift`, `CapturePanelController.swift` dispatch, and small
model action wrappers needed by those commands.

1. Draw one requirements section per owner first, with Bob's title, scope, caption,
   historical line, and indented rows. Reuse the existing requirement glyph drawing:
   depth 1 is `⌗⌗`, depth 2 `⌗⌗⌗`, depth 3+ `⌗⌗⌗⌗`; exact depth belongs in accessible
   text. Use supplied owner ink if available, otherwise the established neutral
   requirement teal. Do not add a native tickable checkbox.
2. Show the fold line, current verdict/consequence, optional drop reason, and navigation
   pin. Use Bob's supplied consequences instead of composing task status/destination
   badges for requirements. The footer follows the highlighted row and shows the
   canonical requirement keys and `reqs ✓… ⊘… →…` summary.
3. Wire Ctrl+X → Met, Ctrl+Shift+X → Drop reason, Ctrl+Shift+T → Carry, Space →
   Carry-plus-pull, Tab → same then advance, and Ctrl+J → pin. Repeated keys retain
   existing consumption guards. Ordinary rows retain their current keys; the new Carry
   chord acts only on requirement rows.
4. Reuse the inline reason editor, Log Recall, and LINK band with
   `Why drop it? (optional)`. Enter stages, Escape restores the previous verdict;
   editing focus must not leak shortcuts into card submit or pin actions. Ctrl+X must
   not override native text editing where the existing router protects it. Preserve
   q/Ctrl+[ handling through the existing Escape dispatch.
5. Put overlap feedback on the attempted row, naming the ancestor with the frozen RV
   copy. The reverse staging order must identify the conflicting existing descendant
   without clearing it. Pointer and keyboard actions share validation.
   Filtering/collapsing cannot make a hidden structural conflict committable.
6. Speak semantic state and depth, e.g. `requirement, level 3, will be dropped`, plus
   owner context. Expose fold count and pin state without reading hash strings.

Extend router, choice, design/presentation, reason/recall and accessibility tests.
Include deep branches, multiple owners with the same title, long labels/reasons,
requirements mixed with ordinary offers, and a pinned hidden descendant. Add a synthetic
panel preview/screenshot through existing infrastructure if available; inspect the
resulting layout. Compile native source and all touched test targets.

## integration

Finish and run this acceptance matrix using deterministic fixtures and a disposable
synthetic vault, never Bryan's live vault:

| Scenario                                      | Evidence required                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------- |
| Requirements only, untouched, Return          | Card opens; owner closes; no requirement completion bytes are written        |
| Mixed owners and ordinary offers              | Requirements first, no merged owners or duplicate rows, shared Bob numbering |
| Drop with blank or recalled reason            | Exact gated argv and Bob receipt; staged edit can be canceled                |
| Carry a leaf or nested branch                 | One task for each disjoint chosen branch; UI trusts Bob's result             |
| Carry-plus-pull and Shift-Return              | Normal commit pulls; Shift-Return carries without pulling and retains pin    |
| Ancestor/descendant choices in either order   | Visible refusal, no lost staged choice and no process submission             |
| Pin descendant then carry ancestor            | One navigation after success to Bob's moved landing                          |
| Duplicate labels edited/reordered/reindented  | `req_close_stale`; fresh keys replace old ones without label matching        |
| Inbox routing, back, retry, executable change | Choices survive valid continuations and invalidate on changed identity       |
| Esc and reason-field Esc                      | No capture write; draft retained; prior verdict restored appropriately       |
| Old Bob, req-less or recurring close          | Legacy behavior and no unsupported flag leakage                              |
| Receipts and notifications                    | Requirement outcomes shown once without inflating ordinary task counts       |

The earlier phases own their focused tests; this phase fills only integration gaps,
updates generated fixture provenance, and confirms that the copied RV file still matches
bob-cli byte for byte. If testing reveals a substantial implementation remainder,
describe it precisely on the phase for appropriate replanning instead of claiming an
untested flow works.

Update the Mac-owned `docs/keyboard.md` (Successor choices), `docs/capture-previews.md`,
and `docs/protocol.md`. Update bob-cli `docs/task-requirements.md` to describe the
shipped Mac behavior and link to the wire contract rather than duplicating it. Keep the
already implemented marker and nesting docs intact. Read `docs.md` before editing and
run both repos' applicable documentation gates. The feature adds client support to an
existing CLI contract; do not introduce or rename public bob options.

Record passing portable, scratch build/typecheck, and exact-SHA macOS CI evidence,
fixture provenance, relevant doc gates, and any verified clean-base failures on this
child epic's phases for its land agent. The land agent rechecks the combined tree,
resolves epic symbols, and closes only its assigned child epic through the normal
lifecycle plus **bob-cli-on.15** when that parent's full scope is verified. The outer
epic **bob-cli-on** remains for its own land agent.
