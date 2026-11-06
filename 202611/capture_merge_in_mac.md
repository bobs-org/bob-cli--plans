---
tier: epic
title: Complete Bob Mac Capture merge-in bands and pickers
goal:
  Bob Mac Capture safely previews and submits new-task merge-in captures with source
  pickers, explicit rewrites, compact landing groups, and verified macOS behavior,
  completing bob-cli-hl.5.
phases:
  - id: wire
    title: Decode merge-in contracts and record fixture responses
    size: medium
    depends_on: []
    description:
      "wire: add tolerant merge_in, diagnostic rewrite, merge_in_pending, block-line
      origin, and related-link decoding. Record reproducible synthetic Bob responses and
      portable decoding tests, preserving existing transfer contracts."
  - id: editor
    title: Integrate merge-in source pickers and explicit diagnostic rewrites
    size: medium
    depends_on:
      - wire
    description:
      "editor: add a distinct merge-in source picker role, Bob-authored continuations
      and Add block ID splicing, span colors, revision-safe pending state, and a generic
      explicit diagnostic rewrite action. Keep incomplete captures unsubmitable."
  - id: band
    title: Render merge-in bands and capture outcome copy
    size: medium
    depends_on:
      - editor
    description:
      "band: build CaptureMergeInPresentation and render complete, pending, and refusal
      states inside the new-task card. Reuse transfer rows and Pull-In controls,
      preserve Similar tasks, and implement footer, status, notification, and settled
      accessibility copy."
  - id: landing
    title: Fold merge landing groups and remove duplicate target blocks
    size: medium
    depends_on:
      - band
    description:
      "landing: fold contiguous merge-target rows by Bob's origin identity for both
      arrows, preserve authored children and source tombstones, and show each created
      target's exact block once. Verify keyboard disclosure and narrow layouts."
  - id: rollout
    title: Raise the grammar gate and verify the integrated Mac experience
    size: medium
    depends_on:
      - landing
    description:
      "rollout: require the shipped merge-in grammar revision, update affected gate
      fixtures, document preview and picker behavior, complete integrated regression and
      rendered-layout coverage, and provide exact-SHA macOS verification evidence."
proposed_by: bbugyi200.athena.bob-cli-hl.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hl.5
create_time: 2026-11-06 01:47:44
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_merge_in_mac.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_merge_in_mac.md)
- **PARENT:**
  [202611/capture_merge_in_arrow.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_merge_in_arrow.md)

# Complete the Bob Mac Capture phase of capture merge-in

## Scope and current evidence

This is a child epic for **bob-cli-hl.5**, implementing the Mac phase of
`plan:202611/capture_merge_in_arrow.md`. Read that artifact through `sase artifact read`
before implementation. The accepted parent design remains the authority; this plan
divides its Mac work into bounded coding phases and changes no product decisions.

At planning time bob-cli HEAD was `248a2bfa`: grammar 15, atomic execution, pending
payloads, picker filtering, rewrite toggles, related-source exclusion, and acceptance
tests have landed. The assigned phase is in progress, has no earlier work notes, and has
no `--epic-symbol` entries. Do not redo the CLI phases.

Opening `bob-mac-capture` through `/sase_repo` resolved master `6f4e279`. Its tree was
clean. It already has `CaptureTransferPresentation`, transfer pending/refusal cards,
merge-target/source task-block roles, Pull-In handling, and transfer fixtures. It has no
merge-in implementation; `CaptureGrammarGate.minimumRevision` is still 5.
`CaptureDiagnostic` lacks `rewrite`, the parse models lack `merge_in`, the failure model
lacks `merge_in_pending`, and block rows lack `origin`. Existing
`CaptureTransferTaskDescriptor` already decodes `created`; retain and test it instead of
introducing a competing descriptor.

Five medium phases are appropriate because wire compatibility, editor state and
splicing, card rendering, landing-row folding, and release gating each have separate
failure modes and substantial tests. Dependencies are deliberately sequential: several
phases touch the same large panel model/view. Each phase includes its own compilation
and focused verification; the final phase handles integration.

## Rules inherited from the parent

- Open `bob-mac-capture` with `/sase_repo` and use only its returned checkout. Read its
  `AGENTS.md`. Paths below are relative to that repository unless prefixed `bob-cli:`.
  Never edit a live installation or Bryan's vault.
- Bob owns syntax, source eligibility, merge effects, replacement strings, byte ranges,
  and consequence text. Swift owns presentation and request lifetime. Read
  `decisions:mac-capture-is-a-thin-client` and
  `decisions:similar-tasks-are-a-settled-heads-up` with `/sase_memory_read`.
- `<` creates an ordinary task or Pomodoro-linked task and merges existing sources into
  it in one Bob transaction. The item keeps `task` or `pomodoro_task`; never route it
  through the whole-card `CaptureTransferPresentation` path. A label-driven merge on
  such an item receives the same band through its `merges[]` payload.
- Parent decision `pull_in_title_count = yes` is final: counts refer to sources that
  actually lost live Task Links. Preserve Bob's authored Pull-In copy and offer keys.
  Offers remain explicit, unselected by default, and part of the existing aggregate
  capture flow. Preserve the newly landed Done/Cancel choice behavior.
- This child epic edits **no memory**. The parent authorized only `glossary:task-merge`,
  which belongs to its separate memory phase, and rejected
  `decisions:capture-merge-in-arrow`. Do not re-ask those decisions.
- No Cargo version or changelog edits, grammar redesign, CLI release, installation, or
  unrelated refactoring. Use Conventional Commits; this child does not need an
  additional breaking-change marker for the already shipped CLI grammar break.
- Do not create follow-up beads. Record `PROPOSED FOLLOW-UP:` on the assigned child
  phase; the land agent triages. Check existing notes/history and landed changes at the
  start of every phase. Never hand-set bead status.

## Interface and presentation contract

Use `bob-cli:docs/capture.md` (merge-in and JSON sections), `bob-cli:docs/task-merge.md`
(MI17/MI18), and actual synthetic responses from the current CLI as the wire
specification. Schema version remains 1.

1. Parse responses expose
   `merge_in: {arrow: {start,end}, sources: [{start,end, marker,route,block_id}]}` on
   the appropriate top-level/item structures. Pending clauses retain their task mode and
   report `merge_in_source` with no error diagnostic. Spans are `merge_in_arrow`,
   `merge_in_source`, and the existing interactive placeholder for the unfinished token.
2. A diagnostic may carry `rewrite: {start,end,replacement,label}`. Ranges are half-open
   UTF-8 offsets into that exact draft. Only an explicit button action applies a
   rewrite. Bob supplies `Use >`, `Move before <`, or `Add spaces`; the app must not
   construct replacements or change Return's meaning.
3. An `incomplete_merge_in` dry-run failure may carry
   `merge_in_pending: {item,need,sources[]}`. Item is one-based. Sources contain
   resolved title/path/line/status and mergeability, or Bob's source-specific error.
   Preserve complete sources while the next one is being typed. Pending is calm; an
   ineligible source shows its reason inline and cannot be submitted.
4. Successful task items already carry `merges[]`, offers, and repaired files.
   `task_blocks` identify created targets with `merge_target` and sources with
   `merge_source`. A line's optional `origin` is the exact `<relative_path>#^<block_id>`
   matching a merge's source. Preserve these bytes; never join by title or infer
   grouping from indentation.
5. Completion uses vault-wide `task_parent` for `< +query`, note-scoped tasks for
   `< @route+`, and routes for `< @`. Bob filters mergeability, excludes typed
   sources/aliases, supplies row details, and returns continuation `" +"`. ID-less rows
   use the existing `capture-task-id --route` flow and its `parent_replacement`. Header:
   `⤵ Merge into “T”`; footer: `↩ pick · , pick another`.
6. `capture-related` marks sources `link: "merge_in"`; they stay out of matches.
   Preserve Bob's left-side action anchors and the existing settled Similar-tasks
   timing. Never calculate exclusions or move the arrow in Swift.

The complete band sits inside `standardPreviewItem`, below the new-task header and above
Similar tasks. It says `⤵ Merging in N tasks`, lists title, secondary note/line, textual
status transition, and moved-note count (`1 note`, `2 notes`, or `title only`). Show
every source up to three; above three show a keyboard-accessible `Show all N`. Always
show the count and every blocking error or meaning-changing warning. De-duplicate
consequences by `(kind, keys)` and display their Bob-authored text verbatim;
`prerequisites_left` has a warning marker. Reuse the existing merge purple and transfer
source/status/consequence/Pull-In components.

For pending input show the new task's parsed header and
`⤵ Merge tasks into this one — type + to pick`, plus the completed sources Bob has
resolved. Do not simulate a successful write or invent minted IDs. Refusals show Bob's
message and one button for each available rewrite. State, disclosure data, actions, and
announcements must stay attached to the matching draft/request revision; editing,
switching executables, or resetting the panel must invalidate stale actions and prevent
late responses from restoring old content.

When a created merge-target task block exists, keep the standard item header but
suppress its duplicate verbatim `previewBlockLines`. Render the exact target block once.
Fold each contiguous run with the same `origin` into `⤵ <source title> · N notes ▸`,
using Bob's moved-note count. Expansion shows all original rows in order. Apply the same
folding to `>` merge targets. Authored children and rows without `origin` remain intact;
source tombstones retain the existing cards and moved-lines folds.

Copy for a single task item that merges:

| Surface            | Required behavior                                                                                                                         |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Primary action     | `Capture & merge N`; mixed/bulk drafts keep `Capture`                                                                                     |
| Status             | `Would create “T” and merge N tasks into it · foo.md`                                                                                     |
| Notification title | `Created “T” · merged N tasks`                                                                                                            |
| Notification body  | One source per line: title, previous → Cancelled, moved-note count                                                                        |
| VoiceOver          | New-task identity followed by source summaries, including spoken status transitions; announce only a settled change, without moving focus |
| Grammar gate       | Minimum 15, or the actual shipped merge-in revision if superseded; `Update bob to merge tasks with <`                                     |

Use grammatical singular forms for one source. Preserve established primary-action
precedence for active choice cards and preserve one aggregate submit.

## Phase: wire

Limit this phase to additive models, reproducible fixtures, and portable tests.

- Extend `Sources/CaptureCore/CaptureModels.swift` parse response/item,
  `CaptureDiagnostic`, failure envelope, and block-line types. Add focused model files
  where appropriate. Add the `merge_in` related-link kind in
  `CaptureRelatedModels.swift` without losing unknown-value compatibility.
- Reuse transfer descriptor/error shapes where they really match, adding a distinct
  merge-in pending envelope. New optional fields use tolerant `decodeIfPresent` or
  existing lossy-array conventions; missing/malformed optional metadata cannot break
  unrelated captures. Preserve source order and UTF-8 offsets exactly.
- Follow `Scripts/generate-transfer-preview-fixtures.sh` with a deterministic merge-in
  fixture generator using an explicit `BOB_BIN`, isolated configuration, fixed clock,
  and disposable vaults. Keep generated paths stable and record the CLI commit used. Do
  not use installed-binary assumptions or private vault data.
- Cover complete one/two/five sources, duplicate and long titles, zero notes,
  dependent/consequence/offer cases, blocked target, pending second source, source
  refusal, each rewrite, Unicode, mixed bulk, label merge, origins for both arrows, and
  parser/completion/related responses. Several dimensions can share a fixture. Wire
  script syntax checks into the existing script checks if needed.
- Portable tests prove additive compatibility, malformed-row recovery, target `created`,
  origin preservation, rewrite byte ranges, and pending source errors. Run
  `just portable-check` and relevant generator/script checks. No gate bump yet.

## Phase: editor

Use `CapturePanelModel.swift`, `CapturePickerState.swift`, picker presentation/view
helpers, editor highlighting, and existing UTF-8 splice/undo helpers.

- Add a distinct merge-in source role from Bob's parse/completion metadata. Scope it to
  the active item and source slot, including task-list and mixed bulk input. Keep
  existing `>` source/destination behavior separate. Route, vault-task, and note-task
  pickers display Bob's header context/details and comma continuation.
- Extend Add block ID handling to splice `parent_replacement`, keep the typed left side
  and prior sources, and continue only from a matching request snapshot.
  Cancellation/failure leaves the draft unchanged. Reuse the current Bob rewrite lane
  for `+` marker toggles; `:`/`^` after a source ID remains Bob's decision.
- Color `merge_in_arrow` merge purple and `merge_in_source` in the route family.
- Add revision-bound pending/refusal state derived from the current parse and dry-run
  failure. Ensure the quiet picker shortcut does not bypass the dry run needed to
  resolve already picked sources. Clear this state on ordinary success, reset, or a
  different draft. Disable submission for incomplete/refused input.
- Expose a generic explicit rewrite action that validates the original snapshot and
  UTF-8 boundaries, applies Bob's replacement through the normal undoable edit path, and
  re-runs analysis. The next phase supplies its visible buttons.
- Add bounded asynchronous hosted tests for comma chaining, ID-less success and
  cancellation, all rewrite labels/Unicode ranges, undo, stale responses/actions, source
  eligibility copy, Similar-tasks edits before the arrow, and unchanged transfer
  behavior. Follow `CaptureTransferPendingTests` and parent-task tests.

## Phase: band

- Add `Sources/CaptureCore/CaptureMergeInPresentation.swift`, built from nonempty
  `merges[]` only for task-creating kinds. Expose source rows, consequences, offers,
  summary/notification/accessibility copy, and pending/refusal presentation using the
  previous phase's data. Pure tests pin count/pluralization, stable source identity,
  duplicate-title handling, consequence de-duplication, and blocked/no-offer cases. Do
  not reconstruct any merge effect.
- Extract narrowly reusable row views from `CaptureTransferCardView.swift` as needed;
  preserve existing transfer snapshots and behavior. Integrate a merge-in band in
  `CapturePanelView.swift`'s standard card without replacing its title, type, schedule,
  destination, or Similar-tasks band. Add source disclosure, warning rows, existing
  Pull-In content, and the pending/refusal views and rewrite buttons.
- Wire primary action, both live/committed status paths, `NotificationService`, and
  settled VoiceOver announcements. A source-less pending draft must not flash a red
  failure; source errors remain visible. Return retains normal picker and capture
  behavior; only a rewrite button rewrites.
- Add hosted tests for mixed bulk, label-driven merge, existing choice-card precedence,
  source exclusion from Similar tasks, one aggregate submission, notification and
  VoiceOver strings, and stable settled announcements. Add render tests for
  complete/pending/refused bands at the established 760/620 widths in light/dark
  appearances, with long titles, five sources, and warnings.

## Phase: landing

- Propagate optional origin through `CaptureBlockDiffRow` and
  `CaptureTaskBlockPresentation`. Extend fold presentation with stable source identity
  and Bob's source title/count. Group only contiguous same-origin runs; repeated titles
  and separate runs must never be coalesced by text.
- Integrate keyboard and VoiceOver disclosure in task-block views for both `<` and `>`
  targets, preserving row bytes, indentation, order, badges, existing unchanged folds,
  and source tombstone folds. Missing or unknown origin metadata must degrade to
  readable unfurled rows without guessed titles or lost content.
- In `standardPreviewItem`, recognize a matching created `merge_target` block and
  suppress only the duplicated verbatim preview. Preserve the task header, merge band,
  and all nonduplicated metadata. Handle auto-minted IDs, a target created by an earlier
  item, same-note merges, and ordinary captures.
- Test title-only groups, authored children/drawers before source payload, duplicate
  source titles, disjoint origin runs, no-origin compatibility, both arrows,
  expansion/reset behavior, and exactly one target block. Extend rendered tests and
  inspect light/dark narrow-width results for visible warnings and usable disclosures.

## Phase: rollout

- Raise `CaptureGrammarGate.minimumRevision` to the shipped CLI merge-in revision (15 in
  the inspected tree), using the exact update copy above. Update gate tests and
  synthetic fixtures that intentionally represent a compatible executable. Keep
  deliberately old/missing grammar fixtures as negative tests; do not weaken
  compatibility checks to make the test suite pass.
- Verify a current compatible parser permits the aggregate capture and an old, missing,
  or stale executable/parser cannot submit. Preserve executable-change invalidation and
  non-merge regression coverage.
- Update `docs/capture-previews.md` for bands, warnings, pending/refusals, origin
  disclosures, and copy; update the appropriate task-picker/keyboard pages for picker
  continuation and rewrite buttons. Do not grow `docs/protocol.md`, whose budget issue
  belongs elsewhere. Read documentation memory first and run `just docs-check` in both
  repositories using the current linked docs.
- Fill any remaining integrated coverage gaps from the matrix below, rebase on current
  linked master, and verify that concurrent past-Pomodoro and Pull-In Done/Cancel
  changes still work. Review render artifacts instead of assuming successful generation
  proves layout quality.
- Provide complete build/test/macOS gate evidence and outstanding baseline-only failures
  to this child epic's land agent. Do not install the app or close `bob-cli-hl.5` from
  this phase.

## Verification required for every coding phase

Never land Swift that has not compiled. Run `just portable-check` on this Linux host and
relevant focused tests. It cannot type-check AppKit/SwiftUI targets. For changes to
`Sources/BobMacCapture*` or `Tests/BobMacCapture*`, follow the linked repo's
`AGENTS.md`: best-effort scratch-copy build on the Mac, excluding `.git`/`.build`, then
`Scripts/bundle.sh` and `just typecheck-tests`, and remove that scratch copy. Read
`tailnet.md` before remote access. Never build in Bryan's checkout or install. If the
Mac is offline, report that scratch compilation was unavailable; still require the macOS
CI build/test gate for the actual changes.

Run `Scripts/macos-ci-check` from the linked repository before landing Swift. It
snapshots the exact tree to a temporary CI branch and reports exact-SHA macOS results.
Use `/sase_monitor` for this and other long commands, waiting for the monitor-start
handoff command itself to exit before finalizing. Watch every pushed SHA through the
required monitor workflow; a portable check is not a substitute. Keep formatting limited
to changed lines. If docs change, run `just docs-check`. If implementation discovers a
necessary bob-cli change, run its `just check` and `just docs-check` too, within the
parent contract.

Minimum integrated evidence:

| Area          | Cases                                                                                                                                            |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Band          | 1, 2, 5 sources; duplicate/long titles; 0/1/many notes; 0/1/many/shared dependents; warnings never hidden                                        |
| State         | Empty arrow, pending second source, unmergeable source, refusal rewrite, reset, stale parse/preview/picker/rewrite/announcement                  |
| Editing       | Vault/note/route pickers, ID-less Add block ID, comma continuation, Unicode and undo, task-list and mixed bulk                                   |
| Relationships | Explicit unselected Pull-In, Blocked and already-linked target, related-source exclusion, label merge, preserved Done/Cancel choices             |
| Landing       | Both arrows, auto-ID and created-target identity, authored notes/drawers, exact block once, origins and keyboard disclosure, existing tombstones |
| Compatibility | Missing/old/current/newer grammar, additive absent/malformed data, executable switch, normal captures and existing transfer regression           |
| UI            | Light/dark, 760/620 widths, five-source disclosure, visible warnings, keyboard operation, settled VoiceOver without focus changes                |

The parent plan mentions pre-existing Mac failures `bob-cli-h7`, `bob-cli-fu`,
`bob-cli-fp`, and docs budget `bob-cli-h9`; inspect current evidence before attributing
a failure to one. A failure reproduced identically on the clean base is recorded with
exact command/SHA/failure in `PROPOSED FOLLOW-UP:` and does not keep an assigned phase
open. Never silence a new failure or claim a red run is green.

## Completion ownership

Child phase agents close **only their own assigned phase**, after verification and
`sase bead epic-symbols <assigned-id>` has no leftovers. Resolve each exemption or
re-key it to a still-open owner before closing. Leave findings on that phase.

This child epic's **land agent** verifies the integrated scope, collects the exact
checks/render evidence, and performs the delegated completion of **bob-cli-hl.5**: run
`sase bead epic-symbols bob-cli-hl.5`, resolve any entries, then
`sase bead close bob-cli-hl.5 --note "<implemented behavior and concrete verification>"`
only when all child work is complete. It also follows the normal lifecycle for its own
child epic. It must not close **bob-cli-hl** or any other ancestor plan. A phase worker
treating this section as a checklist must leave the closure evidence for the land agent
rather than close an ancestor itself.
