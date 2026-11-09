---
tier: tale
size: medium
title: Retire interrupted close-alias replay and land bob-cli-is
goal:
  Keep buffered edits in their original editor and drop buffered submits when replay is
  interrupted, correct the remaining syntax-reference omission, and close epic
  bob-cli-is.
proposed_by: bbugyi200.apollo.bob-cli-is.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-is
status: done
---

- **PARENT:**
  [202611/close_outcome_x_s_typing_aliases.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/close_outcome_x_s_typing_aliases.md)
- **BEAD:**
  [bob-cli-is](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-is/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-is.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-is.land.md)
- **COMMITS:**
  - [3cd53b4](https://github.com/bobs-org/bob-cli--plans/commit/3cd53b415342703102e9411b2c696761a950ed62)
    — chore(plans): mark close-outcome alias epic and replay landing done

# Finish the close-outcome alias landing

This is the remaining work from the landing audit of **bob-cli-is**, not a new feature.
One coding agent can complete it. This tale has no land agent: its coder must perform
the epic closeout in the final step below. Do not wait for another lander or require
this tale's eventual commit SHA, push, or post-push CI result before closing the epic.

## Context already verified

Read `sase bead read bob-cli-is -r "Need landing findings and final decisions"` and the
approved `plan:202611/close_outcome_x_s_typing_aliases.md` through `sase artifact read`.
The original epic has three closed phases, is.1–is.3, and currently no parent bead. Its
**final decision is no memory edits**, including no new decision record. Preserve that
decision, grammar version 19, rewrite schema 1, and the release-train policy; do not
edit package versions or CHANGELOG.md.

The lander reviewed every epic/child note and the actual implementations:

- bob-cli `23217c1d`: pre-key close-slot recognition, shared close lexer, structured
  competing-wildcard exception, one-byte `x`→`!` / `s`→`*` edits, literal CLI/parser
  behavior, and binding-contract unit/CLI tests.
- bob-mac-capture `03392a5`, `2a39c2f`, `0f6753b`: bounded hold, lexical send filter,
  capability probe with its separate process lane, native edits/undo, ordered burst
  replay, comma assist, and probe lifecycle repairs.
- Independently verified all required macOS jobs/steps and exact SHA metadata:
  `03392a55eb87577ac8bd1b69c381949cc9e2024b` in run **42519302906**, and
  `0f6753bf429b9b1b86454fb8d95a17ce965a8f1c` in run **42534561890**, both green. These
  establish the existing implementation baseline, not verification of this repair.
- Reviewed non-epic drift through primary `361974f4` plus fetched base `8427bd96`
  (Gmail-only), and Mac `e7ddfc6`. Gmail/records changes need no alias integration.
  Primary `0b8b913f` and Mac `e7ddfc6` add dot-route aliases and share rewrite/gesture
  routing. Keep their fast `..`, Space/Escape restore, and native undo behavior.
- Active epic **bob-cli-iu** already owns separate dot-arm modifier/pre-reply
  invalidation repairs and the Mac mounted-doc word-budget overages; its notes #1–2
  identify the exact gaps and pending repair tale. Read those notes to coordinate shared
  controller edits, and integrate its repairs if they land. Do not duplicate that work
  or file another task for its already-recorded documentation issue.
- Step-3 follow-up triage is finished and recorded on bob-cli-is: the sole
  `PROPOSED FOLLOW-UP`, is.1 note #1, was **declined as already fixed** by `32c241af`
  (the exact two needless returns and `and_then(Some(...))` lint sites). No new task or
  corroboration is warranted. The epic's probe/count/agenda CI note is resolved by
  `0f6753b` and its green run. Do not reopen those issues based on old evidence.

Open bob-mac-capture with `/sase_repo` and read its AGENTS.md. Access every repository
only through its own checkout or the path returned by `sase repo open`. Paths below are
repository-relative. Do not edit mounted Mac docs in bob-cli.

## 1. Reproduce and repair interruption during replay

The source-audited defect is in bob-mac-capture's
`Sources/BobMacCapture/CapturePanelController.swift`:

- `endTypeAheadHold` finishes `CaptureTypeAheadHold`, then
  `replayTypeAheadKeys(..., acrossTurns: true)` transfers keys into
  `typeAheadReplayQueue`.
- `scheduleNextTypeAheadReplayKey` delays each delivery and waits for the preceding
  comma-assist parse. During that interval the original hold is inactive.
- `flushTypeAheadHoldBeforeFocusOrDismissal` drains only the original hold. It does not
  retire `typeAheadReplayQueue`, cancel `typeAheadReplayTask`, or reset the replay
  flags. The task is otherwise canceled only in `deinit`, while the panel/controller are
  retained across dismissal.
- Mouse-down, first-responder loss, resign-key, close, and dismissal call that
  incomplete flush. Client replacement invalidates the model's rewrite, but once
  `meaningChangingRewriteHoldGeneration` is nil it cannot notify the controller of the
  still-scheduled replay. Both direct `processClient` assignment and `setProcessClient`
  need coverage.
- `routeKeyDown` currently appends all live keys to the replay queue before checking
  non-bufferable commands. A command that should flush first can therefore trail a
  queued Return. `dispatchReplayedKey` has no original-editor/client lifecycle guard;
  its Return reaches `submitAfterTypeAheadReplay` after the old context has gone away.

Extend `Tests/BobMacCaptureTests/CaptureCloseAliasTests.swift`. Its existing
`testCaretFocusClientReplacementAndDismissalFlushBeforeActing` interrupts while Bob is
still blocked; add tests that release Bob, pause replay after the reply (and separately
after one delivered edit), then interrupt before the remaining keys run. Use an injected
replay scheduler or bounded test gate, not a race against the current 16 ms delay or
wall-clock sleeps. Prove the new tests fail on the original behavior.

Use one coherent lifecycle for the active hold and its pending replay tail:

- An interruption must cancel scheduled delivery and invalidate any task already waiting
  for comma-assist parse, so an old task cannot restart or modify a new queue.
- Drain pending edit keys once, in order, into the original draft before a focus or
  dismissal transition; preserve their timestamps and drop every pending Return or
  Command-Return. Retain the existing dropped-submit status. An already-applied Bob
  rewrite remains applied. Never deliver the tail into another field/client.
- If the original editor is already unavailable, preserve its draft text safely without
  dispatching its key events to the new responder. Keep grammar in Bob.
- Flush non-bufferable commands before acting on them, including Command-Z/V, Escape,
  Tab, arrows, and function keys. Normal live text during uninterrupted replay still
  joins the tail in order. Ensure flushing cannot recursively launch another unowned
  hold or leave replay flags stuck.
- Notify the controller on actual process-client replacement even after the rewrite hold
  has ended. Cover both replacement APIs; do not confuse a same-object Recheck with
  replacement or break the identity-cached capability probe.
- Preserve chained alias replay and comma-assist ordering. A second alias must own the
  remaining tail, and timeout/failure/cap flushes must still drop queued submits. Do not
  add a second capture grammar or global suppression of editor assists.

Exercise `xs1` plus Return/Command-Return, focus to another field, click/caret move,
resign-key, retained close/dismissal, direct/setter client replacement, and a command
flush while replay is suspended. Assert draft preservation, no stale capture process, no
edits in the new field, no later resurrection after releasing the test gate, and
successful subsequent fresh typing. Keep the old success, timeout, no-change, old-Bob,
undo/redo, timestamp, and comma-assist tests green. Include an ordered replay regression
involving the shared dot-route or equals/hash gesture path so the interleaved dot work
remains integrated.

## 2. Correct the omitted reference wording

Read `docs.md` through `/sase_memory_read` before editing. In bob-cli
`docs/reference/capture-syntax.md`, the `x-close` subsection currently says “`xx`,
`x-ray`, and capital `X` never close.” Scope that statement to literal grammar, paste,
`capture-parse`, and `bob capture`, and explain that Mac editor typing turns `xx` into
`x!` and `xs` into `x*` in eligible close slots. Link the existing sigil-alias contract.
Check adjacent overloaded-marker wording for the same omission without rewriting
unrelated documentation or inventing grammar-level aliases.

## 3. Verify the repaired tree before closeout

- Recheck drift since the audit SHAs and integrate any changes that affect the same
  replay/gesture paths. Re-read new epic or child notes before deciding readiness.
- Run `just check` in bob-cli and `just docs-check` for the reference correction. Never
  run `just check-full`; a check-full-only failure is infrastructure, not remaining epic
  work. If docs-check still reports only iu's recorded Mac page-budget overages (6088,
  6060, and 6014 words at e7ddfc6 for capture-panel/keyboard/protocol), distinguish that
  existing active-epic issue from a regression in this correction. Record the exact
  current result and its ownership; do not conceal it or broaden this tale into a
  duplicate iu repair. Resolve any new failure this tale causes.
- Format only changed Swift lines. Run the linked Mac repo's `just portable-check` on a
  Swift host (athena if necessary). Read tailnet memory before remote access. Compile a
  scratch Mac copy, excluding `.build` and `.git`, with `Scripts/bundle.sh` and
  `just typecheck-tests`, then remove that scratch copy. Never use Bryan's checkout or
  install the app. If Mac access fails, report it; do not claim compilation.
- Run `Scripts/macos-ci-check` in bob-mac-capture against the final working tree;
  require the full macOS gate, including execution of the new AppKit regressions. Use
  `/sase_monitor` for long commands and carry precise next steps in its prompt. Do not
  substitute portable tests for AppKit compilation or CI execution.
- With precommit CI green, complete step 4 in the same coding turn as the final verified
  changes. Stage any required post-finalization exact-SHA push-CI watch through
  `/sase_monitor`; that watch verifies the push and handles failures, but **must not be
  a dependency of this tale's epic closeout**. Finish with `/sase_final` so all edited
  repositories are committed by the host.

## 4. Close out bob-cli-is yourself

There is no separate tale land agent and the interrupted lander will not resume. After
implementation and precommit verification, perform these steps concretely:

1. Re-read bob-cli-is and every child/note, verify all descendants and linked plans are
   complete, and carry the audit and follow-up outcomes above into the close note. If
   this tale has a linked bead under bob-cli-is, close that completed tale bead normally
   first so the epic has no open descendant; do not force a successful nested landing.
   Resolve any genuinely unfinished child rather than forcing it.
2. Run `sase bead epic-symbols bob-cli-is`. Resolve every listed exemption by wiring the
   symbol, privatizing it, adding a justified non-test pragma, or deleting it per
   Symvision policy. Re-key only to a still-open later bead that truly needs it. The
   audit found none, but check again. Never defer this cleanup to another agent.
3. Run
   `sase bead close bob-cli-is --note "<verified Rust and Mac scope; replay interruption repair and tests; intervening dot-route/Gmail/records review; verification results; is.1 follow-up declined because 32c241af already fixed it; epic probe/count/agenda note resolved; no memory edits; symbols retired>"`.
   Supply real results, not the placeholder. If symbols block the close, fix them and
   retry. Never force merely to make a close succeed. A deliberately canceled or
   superseded unfinished phase requires the user's specified
   `--force --reason ... --resolution canceled|superseded` semantics, not a successful
   landing disguised as cancellation.
4. Run `just symvision` if the recipe is available (neither audited justfile currently
   has it); explicitly record when unavailable. Open the plans repo with `/sase_repo`,
   resolve `plan:202611/close_outcome_x_s_typing_aliases.md`, and change only its
   frontmatter `status: wip` to `status: done`. Include that repository in finalization.
   Mark this repair tale done according to its ordinary lifecycle.
5. Run `sase bead read bob-cli-is -r "Need the parent link"` after closing. The audit
   confirmed **no parent_bead**, so finish normally if unchanged. If a link was added,
   follow the original landing contract: a parent phase closes normally only after its
   scope is verified, leaving its epic to its waiting lander; a parent plan needs all
   descendant/notes/linked-plan readiness and post-child drift rechecked, symbols
   retired, normal close, symvision, and its plan marked done, then repeat directly
   parented plan ancestors. Stop and note any incomplete or ambiguous parent.

Do not leave bob-cli-is open merely to wait for this tale's own commit or push. If
implementation or required verification still fails, keep it open and carry the specific
remaining work through an appropriate SASE monitor rather than claiming completion.
