---
tier: epic
title: Bob Mac Capture relation successors
goal:
  Bob Mac Capture presents starts-after offers and authoring cues from Bob's JSON,
  preserves older payload behavior, and completes bob-cli-fs.10 with verified Mac
  presentation and real-Bob fixtures.
phases:
  - id: protocol
    title: Decode relation successors and generate real-Bob fixtures
    size: medium
    depends_on: []
    description:
      "protocol: add tolerant CaptureCore models for relation offers, held rows, check
      state, Bob-authored Pull-In copy, and authoring metadata; generate reproducible
      scratch-vault fixtures and test every payload location and older responses."
  - id: chooser
    title: Render Bob-authored Pull-In groups and select relation offers
    size: medium
    depends_on:
      - protocol
    description:
      "chooser: make the Pull-In choice model and card consume Bob's group order and
      copy, gather relation offers from every capture location, preserve opaque
      selection keys, and cover relation-only and mixed choices with model and design
      tests."
  - id: reporting
    title: Present relation previews, held notices, notifications, and badges
    size: medium
    depends_on:
      - chooser
    description:
      "reporting: wire relation offers and held/check state into passive capture
      previews, completion summaries, notification counts, and block badges; test all
      nested and item-level paths while preserving existing dependency and keep-moving
      copy."
  - id: authoring
    title: Show relation offer authoring cues and finish the Mac contract docs
    size: medium
    depends_on:
      - reporting
    description:
      "authoring: present Bob's offers-source capsule, consequence message, amber
      caveats, and inverse hint without closing or dimming the source; finish protocol
      and preview docs and run the complete portable and best-effort macOS verification
      gates."
proposed_by: bbugyi200.athena.bob-cli-fs.10
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fs.10
create_time: 2026-11-02 19:46:04
status: wip
---

- **PROMPT:**
  [prompts/202611/mac_relation_successors.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_relation_successors.md)
- **PARENT:**
  [202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/starts_after_relation.md)

# Bob Mac Capture relation successors

## Purpose and ownership

Complete the Mac implementation assigned to `bob-cli-fs.10`, phase `mac` of
`plan:202611/starts_after_relation.md`. This is a child epic of that phase. Its
implementation phases change **bob-mac-capture only**; bob-cli supplies the already
landed JSON contract and fixture binary. The child epic's land agent is responsible for
verifying the aggregate result and closing `bob-cli-fs.10`. Child phase workers close
only their own assigned beads. Nobody in this child epic closes `bob-cli-fs` or another
ancestor plan bead.

Each phase fits one coding agent. The dependencies are deliberately serial because the
chooser, preview, and authoring changes share `CapturePanelModel.swift` and
`CapturePanelView.swift`; the first phase owns the common wire models. There are no new
reviewer choices. The parent's decisions remain final: only the three specified glossary
terms were authorized for memory changes, and decision records were not. This Mac child
epic needs **no memory edits**.

Before editing, use `/sase_repo` to open `bob-mac-capture`, use only its returned path,
and read its `AGENTS.md`. Access any other repository through `/sase_repo`; read parent
plan artifacts with `sase artifact read`. Do not change the live vault, installed app,
Bob configuration, Cargo version, or CHANGELOG.

## What is already complete

Inspection on November 2, 2026 found no earlier work notes on `bob-cli-fs.10`; it is in
progress, with no `--epic-symbol` entries. Its Rust prerequisites `.3`, `.4`, and `.5`
are closed. The additional launch gate `bob-cli-fb.5` is closed, with keep-moving
support and a passing portable check recorded. Do not redo that work.

The inspected bob-cli HEAD was `71907dab`; its `capture/output.rs` already exposes all
relation payloads, and `capture/pull_in_copy.rs` authors group and top-level copy. The
inspected Mac HEAD was `a58e4f0`, with a clean tree. It has keep-moving support, but
searches found no `relation_successors`, `relation_held`, `relation_check`, or
`offers_source` handling. Recheck the current tree before implementing so concurrent
landed work is preserved.

Current extension points in the Mac repository:

- `Sources/CaptureCore/CaptureModels.swift`: tolerant Codable models, initializer
  defaults, completion/span/candidate models, and Pull-In metadata.
- `Sources/CaptureCore/CaptureSuccessorOfferRows.swift`: gathers unblocked and
  keep-moving rows from task completion, Pomodoro close, and nested relation closes.
- `Sources/BobMacCapture/CapturePullInChoice.swift`: sections currently key only on
  `(group, project)` and sort through a hard-coded rank table. Context and row meta can
  currently fall through to `Unblocked by …` and `trigger: …`.
- `Sources/BobMacCapture/CapturePullInChoiceView.swift`: group glyphs and copy are local
  switches; caption height is currently special-cased for keep-moving.
- `CapturePanelModel.makePullInPreviewSnapshot`: manually gathers offers and must also
  include item-level toggle offers.
- `CapturePanelView.pullInPreviewSummary`, `CaptureSuccessorPresentation.swift`,
  `CaptureTaskCompletePresentation.swift`, `CapturePomodoroClosePresentation.swift`, and
  `NotificationService.pulledInSummary`: passive and post-submit presentation.
- `CaptureRelationCloseEffect.swift`, the cancel/complete presentation wrappers, and the
  panel's relation-label prompt/slot: authoring cues.

## Binding contract

Bob remains the authority for eligibility, destinations, labels, verdicts, mutations,
and group copy. Swift never scans the vault, parses LINKS, resolves release state,
normalizes relation labels, or computes successor eligibility. All schema versions
remain 1 and the change is additive. Preserve older JSON behavior when new fields are
absent.

Consult the current implementations and contracts in bob-cli:
`src/native/capture/output.rs`, `src/native/capture/pull_in_copy.rs`,
`src/native/capture_complete/model.rs`, `src/native/capture_complete/successor.rs`,
`src/native/task_complete/relation_successors/copy.rs`,
`tests/fixtures/relation_successor/vectors.json`, `docs/task-dependencies.md`
§§12.5–12.7 and 12.18–12.19, and `docs/task-relations.md`.

### Wire shapes

- `relation_successors`, `relation_held`, and `relation_check` occur on `task_complete`,
  `pomodoro_close`, nested `relation_completes[].task_complete`, and the capture item
  itself for toggles. Missing arrays decode as `[]`; optional metadata stays optional.
  Follow the existing per-field tolerant decoding style, with defaults on public
  initializers.
- Successor rows reuse `TaskCompleteUnblocked`. They gain optional
  `relation: {label, key, event, after: [{note_path, block_id, text}]}`. This metadata
  can also occur on another group's row as a second reason.
- Offers gain optional `after`, an opaque target identity string joining the row to a
  relation group. Offer `key` remains opaque and is never reconstructed.
- Held rows contain task identity, text, status, reason, optional `waits_on` and
  `waits_for` target arrays, optional `scheduled` and `project`, and relation metadata.
  Reasons are `already_planned`, `already_started`, `blocked`, `hidden`, and
  `waits_for`. Check state is `checked` or `unavailable`.
- `pull_in.copy` has `{title, capsule, glyph}`. Each `pull_in.groups[]` entry has
  optional `project`, `after: {key, note_path, block_id, text}`, `label`, `event`, and
  `folded`, plus `copy: {title, why?, tag?, fold?, glyph}`. New fold counters are
  `already_started` and `waits_for`; retain all existing counters. Missing `folded` is
  valid for hand-off and project-finished groups.
- `capture-complete.relation_successor` contains source and target endpoints, optional
  `verdict`, `message`, and `hint`. **Inverse hints have no verdict** in the current
  Rust emitter. `offers_source` is an additive Boolean on spans and completion
  candidates. Missing flags mean false.
- Recognize block-diff `reason: "relation"` and task block role `"relation_successor"`
  without weakening unknown-value fallback behavior.

### Presentation rules

The card and its sections render Bob's supplied title, capsule, why, tag, fold, and
glyph. The context line uses the supplied copy rather than deriving an unblocking story.
Existing per-group Swift copy stays only as the no-copy fallback for older Bob versions.
Bob's `pull_in.groups` order is authoritative.

Map glyph keys to SF Symbols: `hand_off` → `arrow.turn.down.right`, `project` →
`folder`, `project_created` → `folder.badge.plus`, `project_finished` →
`checkmark.circle`, `keep_moving` → `forward.end.fill`, `relation` → `link`. Unknown
keys get a neutral fallback such as `circle`, never `lock.open`.

Relation-only group/title copy is `Starts after “T”`, with why
`“T” is parked · Pending, off today's plan` or `“T” is done`, and tag
`your links · not dependents`. Its accent is neutral, with no new hex colors. Use lane
names. Do not leak `Unblocked by`, `trigger: relation`, raw status-symbol transitions,
or an unlock glyph into relation presentation. Preserve the existing selection/focus
behavior: no marks on open, explicit Return/Skip submits Bob keys, and no capture write
merely from opening the chooser.

Muted row meta names multiple triggering targets (`after “T1” and “T2”`, or
`after “T1” and k more`) and a relation second reason (`· label “T”`). Use decoded
metadata and the parent's copy vectors; these are display formatting, not another
planner. The same distinction applies to held/notification lines where the wire contract
supplies structured data rather than a dedicated copy field.

Held-only presentation is muted and appears when no chooser opens. Never show
`already_planned` held rows. Show the documented waits-for, already-started,
dependency/schedule/project-blocked, and hidden reasons. `relation_check: "unavailable"`
must produce a muted uncertainty line, using `relation links` when the response does not
identify a single label. No "nothing picked" notice.

An `offers` authoring effect is neither a close nor a source-dimming effect. Its capsule
is SF `link` with `offers`; display Bob's message unchanged, color
`already_parked`/`already_closed` caveats amber, and show inverse hints muted even when
`verdict` is absent. Cancel and complete behavior remains intact.

## Phase protocol

1. Add the additive models above and wire them through every owning struct's decoder,
   CodingKeys, encoder where applicable, and defaulted initializer. Small dedicated
   model files are preferable to unrelated refactoring of the large
   `CaptureModels.swift`.
2. Extend `CaptureSuccessorOfferRows.swift` with a single tested way to gather relation
   offers, held rows, and check results from all four locations. Keep existing
   unblocked/keep-moving helpers semantically unchanged. Preserve deterministic
   traversal and do not count batch root summaries twice.
3. Add `Scripts/generate-relation-successor-fixtures.sh`, following
   `generate-project-note-offer-fixtures.sh` and `generate-keep-moving-fixtures.sh`. Use
   an explicit `BOB_BIN` freshly built from current bob-cli master in an opened checkout
   with an isolated target/build directory. Run any long build through `/sase_monitor`.
   The generator uses isolated vault/config/state/cache, a pinned clock, and path
   normalization only; it never patches semantic output.
4. Produce real fixtures for parked `x*`, Done `!`, Pending toggle-unlink, held-only,
   mixed `x*!2`, nested complete-label successor offers, selected rows, and authoring
   (`offers`, `already_parked`, `already_closed`, inverse hint, spans/candidates with
   `offers_source`). Seed every mixed-case input explicitly so the fixture genuinely
   contains both dependency and relation offers. Add selected/unselected cases needed
   for notification and block-badge tests.
5. Exercise real required-choice refusal, `-p none`, and opaque-key selection on
   separate scratch vaults. Store fixture provenance, generator command, and Bob source
   SHA. Extend `Tests/Fixtures/fake-bob` only where panel/client tests need fixture
   responses. Supplement with hand-built decoder cases for malformed, absent, empty, and
   unavailable data that real Bob cannot deterministically emit.
6. Extend `CapturePullInDecodingTests`, `CaptureSuccessorOfferRowsTests`, and
   relation-effect decoding tests. Check mixed payload preservation and all four array
   locations, top/group copy, missing folds, and inverse null verdict. Run the common
   verification below before closing this child phase.

Done when real fixtures are reproducible, current and old payloads decode, and the
aggregate helpers retain relation rows/check state from every path. This phase does not
change UI behavior.

## Phase chooser

1. Extend choice row/section metadata and identity to include `after`. Build sections in
   the supplied group order, joining on `(group, project, after)`. Rust can emit
   distinct labels for the same target: retain the supplied label as an additional
   discriminator in that case rather than merging Bob groups. Use exact supplied
   metadata; do not implement label normalization in Swift. Preserve deterministic
   legacy ordering only when group metadata is absent.
2. Use supplied group and top-level copy for the section headings, capsule, context,
   contrast tags, folded line, and glyph. A supplied copy object with an absent `why` or
   `fold` does not authorize synthesizing a legacy string. Keep every existing no-copy
   fallback, including new-project and keep-moving looks. Group metadata may not create
   extra empty sections in a per-item subset.
3. Generalize caption/fold height accounting to the rendered content, covering relation
   captions and new fold counters. Preserve fixed height while filtering or collapsing;
   include multi-target/second-reason meta in accessibility text.
4. Gather relation rows in `makePullInPreviewSnapshot` via the protocol helpers,
   including item-level toggles and nested complete-label closes. Deduplicate by Bob's
   offer key as today, preserving canonical numbering and request freshness. Ensure
   preview and required-choice-refusal recovery both open the chooser. Selected keys
   round-trip unmodified through `--pull-in`; Shift-Return/Skip remains an explicit none
   choice.
5. Extend `CapturePullInChoiceTests`, `CapturePullInChoiceModelTests`,
   `CapturePullInChoiceDesignTests`, and client flow tests. Cover relation-only, mixed
   order, distinct targets, same target with distinct labels, unknown glyph, literal Bob
   copy overriding local strings, older no-copy payloads, stale keys, filtering/collapse
   sizing, and all source locations. Use the real fixtures. Compile and run the
   applicable checks before closing this child phase.

Done when the parent card mock is represented in design/model tests and the real
offer/refusal fixtures reach a chooser whose explicit choice submits Bob's keys.

## Phase reporting

1. Reuse the chooser's group presentation in `pullInPreviewSummary`; include relation
   rows from task completion, Pomodoro close, nested relation completion, and top-level
   toggles. Show `↩ choose what to pull in`, Bob's caption/tag/fold, and muted
   suffix/second-reason meta without displaying unrelated batch groups.
2. Add pure CaptureCore presentation helpers for held/check output, reused by preview
   and post-capture presentation. Handle all held reasons and uncertainty, suppress
   `already_planned`, and avoid duplicating folded diagnostics when the chooser is
   active. Respect the no-picker condition and silent none choice.
3. Extend `SuccessorRows.notificationLine`, callers in task/Pomodoro completion, and
   `NotificationService.pulledInSummary` to count relation picks exactly once across all
   payload locations. Relation-only notifications read `🪢 Pulled in n · label “T”`;
   mixed dependency notices retain their existing wording while counting relation picks.
   Preserve keep-moving breakdowns and present relation context without double-counting
   a hand-off's second reason.
4. Give relation block-diff lines the SF `link` badge with
   `.help("Offered by your relation link")`. Route `relation_successor` task blocks
   through the appropriate existing task rendering; do not mistake them for unrelated or
   unknown blocks.
5. Extend `CaptureSuccessorPresentationTests`, `NotificationServiceTests`, and relevant
   preview/block tests. Include held-only and unavailable payloads, selected and
   unselected rows, mixed reasons, multiple captures, duplicate keys, toggle unlink,
   nested relation completes, and no-copy legacy cases. Compile and run the applicable
   checks before closing this child phase.

Done when every relation result is visible in its intended surface, selected counts
agree with Bob, and non-dependency rows carry no dependency explanation.

## Phase authoring

1. Extend `RelationSourceEffect` and the shared relation-effect presentation with
   `offers`. Ensure this cannot use closing/dimming behavior even if shared APIs still
   have historical close-oriented names. Prefer an explicit style for neutral offer,
   amber caveat, and muted hint over overloading `isClosing`.
2. Carry `relation_successor` through every completion/prompt refresh and selected or
   fresh-typed candidate path. Show `offers_source` capsules on candidate rows and
   authored spans where existing cancel/complete cues appear. Render Bob's message/hint
   unchanged; never derive a verdict from task status in Swift.
3. Cover direct and inverse direction, new and existing source, released caveats,
   hint-only/null-verdict responses, unavailable/absent metadata, and cancel/complete
   regressions in the portable relation-effect tests and `CaptureRelationPanelTests`.
   New async tests must have bounded waits.
4. Update `docs/protocol.md` for every decoded addition and fallback, and
   `docs/capture-previews.md` for the choice, neutral relation group, held/check
   notices, notification/badge, and authoring cues. Preserve published page paths.
5. Run the full verification below and record evidence for the child land agent. This
   phase closes only its own assigned child phase, not `bob-cli-fs.10`.

Done when authoring fixtures render all four indication forms, old closing-label
behavior remains covered, docs pass, and aggregate verification is recorded.

## Common verification and landing

Every implementation phase compiles its Swift changes. Run focused tests during
development, then `just portable-check` in the opened Mac repository. It parse-lints all
Swift and builds/tests CaptureCore and RefsCore on Linux. Use athena if the current host
lacks Swift. Format only changed lines to the repository style.

For phases touching `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, also attempt the
required macOS scratch compile: use the `mac` SSH alias with a bounded connection
timeout, create a unique temporary copy, rsync excluding `.build` and `.git`, run
`Scripts/bundle.sh` and `just typecheck-tests`, and remove only that scratch copy. Never
use Bryan's checkout or `just install`. Read `tailnet.md` through the memory skill
before remote work. Mac Command Line Tools do not include XCTest, so do not substitute
`swift build --build-tests` there; CI runs the actual macOS tests. If the Mac is
offline, record the failed attempt and explicitly report the macOS targets as
unverified; this does not waive the portable compile requirement.

Run `just docs-check` after docs changes. Validate the fixture generator with `bash -n`
and run it against the recorded Bob build. Do not rerun an expensive fixture build if
unchanged inputs and retained provenance already cover it. Use `/sase_monitor` for long
builds/checks and wait for its handoff command to exit.

Before final completion the child land agent verifies that all four child phases landed,
reviews the combined diff for the parent's acceptance matrix, and runs the full
`just portable-check` and `just docs-check` on the combined tree. Record the actual
test/build results and Mac verification limits. Do not claim macOS test execution from
Linux tests or stub-based typechecking.

If a check failure reproduces identically on the clean base, record it as
`PROPOSED FOLLOW-UP:` on the assigned bead with reproduction evidence and any existing
task ID; it does not keep completed scope open. Workers never create task beads. If a
wire-contract defect outside this Mac scope is found, record that evidence instead of
silently recreating server semantics in Swift.

Before closing each assigned child phase, run its `sase bead epic-symbols` check and
resolve/re-key leftovers. After the entire Mac scope is verified, the child epic's land
agent runs `sase bead epic-symbols bob-cli-fs.10` and resolves any leftovers, then
closes **only the parent phase** with
`sase bead close bob-cli-fs.10 --note "<combined implementation and verification evidence>"`.
That is the expressly delegated phase completion; it is not authorization to close
`bob-cli-fs` or any other ancestor plan. Follow the host workflow for the child epic's
own landing declaration.
