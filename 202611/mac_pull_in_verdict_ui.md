---
tier: tale
title: Mac Pull-In card Done and Cancel verdicts
goal: Complete bob-cli-hn.9 with staged verdict UI, preserved refusal choices, and
  verified macOS integration.
size: medium
proposed_by: bbugyi200.athena.bob-cli-hn.9
proposed_with: codex/gpt-6-astra
bead: bob-cli-hn.9
status: done
---

- **PARENT:**
  [202611/successor_card_verdicts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/successor_card_verdicts.md)
- **BEAD:**
  [bob-cli-hn.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hn/bob-cli-hn.9.md)

# Mac Pull-In card: stage Done and Cancel verdicts

Complete assigned phase **bob-cli-hn.9** of `plan:202611/successor_card_verdicts.md`.
The Mac Pull-In card must let a user mark an existing offer Done or Cancel, review the
consequences, and apply those verdicts alongside any pull-ins in one capture. Nothing
writes while choosing. A refused submission returns to the choices intact.

## Scope and evidence

This is a medium tale because the engine, JSON, process arguments, and portable outcome
model already exist. One agent can finish the bounded Mac integration, rendering,
regression tests, documentation, and verification. This is the first implementation plan
for this phase; its bead had no prior-work or remaining-work notes when inspected. Its
status is already `in_progress`; do not set it by hand.

Use `/sase_repo` to open **bob-mac-capture** and read its `AGENTS.md` before working.
All product changes belong there. Read these audited sources before implementation:

- `sase bead read bob-cli-hn.9 -r "Verify assigned scope and final epic decisions"`.
- `sase artifact read plan:202611/successor_card_verdicts.md "Read binding Mac card design"`.
- `sase artifact read research:202611/successor_offer_verdicts/successor_offer_verdicts.md "Read accepted A1–A14 requirements"`.
- The primary bob-cli checkout's `docs/task-dependencies.md` §12.22, plus the `docs.md`
  and `decisions:mac-capture-is-a-thin-client` memory through `/sase_memory_read`.

The parent plan is binding where it is more specific than other sources. Its reviewer
decisions are final: `memory_verdict_record=no`, `memory_offers_supersede=yes` for
`decisions:unblocked-tasks-are-offered` only, and `memory_glossary=no`. This UI phase
requires no memory edits; the authorized memory work belongs to the contract phase. Do
not edit any memory in this tale.

Verified starting points:

- Mac commit `8ab7c4e` (`feat(capture): add offer-close model support`) implements
  closed prerequisite **bob-cli-hn.8**. Its note reports 2,194 portable tests passing
  with one skipped. Confirm this remains in the checkout; do not redo its decoding,
  fixtures, or argv work.
- `Sources/CaptureCore/CapturePullInOutcomes.swift` provides exclusive marks/verdicts,
  descriptor eligibility, normalized reasons, counts, Esc arming, key reseeding, and
  `argvSelection()`. It currently advances by Bob's offer number, while the card
  navigates grouped/filter-visible rows. Its `primaryLabel` and `skipLabel` currently
  say `Apply choices` / `Apply closes`, which differ from the accepted Mac labels below.
  Make the small integration adjustments and update its tests.
- `CapturePullInChoice.swift` still stores raw `markedKeys` and `highlightedKey`.
  Preserve its grouping, filtering, folding, stable offer keys, and plan meter.
- `CapturePanelModel.swift` owns opening, submitting, and reopening cards.
  `CaptureInboxRouteSubmitContext` saves `pullInChoiceForBack` and pull-in selection,
  but does not yet carry verdict arguments. `dispatchCapture` does not pass them.
- `CaptureKeyCommandRouter.swift` has no Pull-In X binding. Dispatch and focus
  integration live in `CapturePanelController.swift`; the row list and footer live in
  `CapturePullInChoiceView.swift`.
- `openMoveChoice` and `CaptureMoveChoiceView.detailStrip` are the reseed and
  consequence-strip precedents. Reuse relation complete/cancel symbols and palette.
- The Rust `unknown_pick_error` in `src/native/capture/pull_in.rs` still returns a usage
  error with `stale, unknown, or out-of-range --pull-in pick:` and prose rows, without
  structured offers. Unlike the three new verdict refusals, that legacy response needs a
  read-only preview refresh before reseeding; never parse its printed rows into offer
  identities.
- `sase bead epic-symbols bob-cli-hn.9` currently reports no entries. Repeat at close.

## 1. Use the portable outcomes as the card's source of truth

Replace the raw selection storage with `CapturePullInOutcomes` in
`CapturePullInChoiceState`. Derived compatibility accessors are fine for existing
views/tests, but must not maintain competing marks or verdict dictionaries.

Keep Leave as absence of either mark or verdict. Space/pointer Pull In replaces
Done/Cancel; Done replaces Pull In/Cancel; Cancel replaces Pull In/Done only after its
reason is confirmed. Pressing a row's current verdict chord clears it to Leave.
Command-A marks Leave/Pull-In rows and preserves every Done/Cancel row. Existing Space
movement stays unchanged.

On keyboard staging, advance to the next **visible selectable** row in the card's
grouped order and clamp at the end. Clearing stays on the acted-on row. Do not use
Bob-number order for visual movement; preserve that order for argv. Extend the portable
API with explicit navigation input or adapt its highlight in the card; test the case
where these orders differ. Filters and collapsed groups retain hidden choices. An
unmarked hidden highlight never supplies a fallback pick.

An explicit mark wins over the highlight. Once any verdict exists, Return has no
highlight fallback. Plan-budget projections count pull-ins only. The primary action is
enabled with any effective pull-in or verdict, including a verdict-only note pull. Use
the approved labels everywhere, updating portable label tests:

| State         | Primary                   | Shift-Return                |
| ------------- | ------------------------- | --------------------------- |
| Pull-ins only | `Pull In {a}`             | Existing skip behavior      |
| Verdicts only | `Close {n}`               | `Skip pull-ins · Close {n}` |
| Both          | `Pull In {a} · Close {n}` | `Skip pull-ins · Close {n}` |

Footer counts use `pull in {a} · ✓ {b} done · ⊘ {c} cancel`, omitting zero segments.
Keep current zero-choice and note-pull-only behavior when there are no verdicts.

## 2. Wire row actions, reason focus, and discard behavior

Add model actions `toggleOfferDone`, `beginOfferCancel`, `confirmOfferCancel`, and
`restoreOfferCancel`, accepting a row key where pointer/accessibility actions need one.
Use Bob's `offer.close` fields for eligibility, `why`, consequences, scope, and unblock
counts. Never infer these from task text, status, or project shape.

Model an active reason edit with its key, text, and saved prior row outcome. Opening
Cancel focuses an inline single-line field; it does not capture. Return confirms and
advances; blank is valid. Blur confirms the text once. Esc or Control-[ restores the
exact previous outcome and focus without arming the card's discard warning. Normalize
with the existing cancel argument type: trim and collapse newlines. Ensure a blur caused
by confirmation or restoration cannot double-apply or advance. The same Cancel chord on
an already staged Cancel row clears it regardless of its stored reason. Done never
prompts for a log.

Extend the router/context/controller together:

- Exact Control-X stages Done; exact Control-Shift-X begins Cancel. Consume repeated
  verdict presses without acting. Command-X remains native Cut, including the filter and
  reason editor. Unrelated modifier combinations retain native behavior.
- Add `pullInReasonFocused` and a reason-field `CapturePanelFocusTarget`, with a stable
  accessibility identifier and actual first-responder detection. In that context, Return
  confirms and Esc/Control-[ restores; other keys stay native. Preserve
  input-composition guards so confirmation does not consume IME editing.
- Outside the reason field, the first Esc/Control-[ with verdicts arms
  `Esc again to discard {N} close(s)`; the second discards and returns through the
  existing back path without writing. Repeats must not turn one held Escape into
  discard. Any other key disarms, even native filter typing and Cut that produce no
  routed command. Pull-in marks alone keep the existing single-Escape behavior.
- Capability is `pull_in.close.supported`. Close hints/actions are offered only when it
  is true and at least one visible row is eligible. An unavailable chord beeps; an
  ineligible row also displays and announces Bob's `why`. Muted rows have no verdict
  slot. Old Bob and all-new project cards keep their existing presentation.

## 3. Preserve outcomes through submit, routing, and refusal

Use the existing `CapturePullInOfferVerdicts` and `BobProcessClient.capture`
`offerVerdicts` / `offerCloseCapabilityAdvertised` parameters. Submit opaque keys only.
Done uses a comma list; Cancel uses repeated key/reason arguments through the existing
builder. Never emit them without the matching preview's capability proof.

Return submits picks and verdicts once. Shift-Return sends explicit `.none` for pull-ins
and retains every verdict, including on note-pull-only drafts. Control-Return submits
both and preserves the open-notes intent. Keep the draft, process identity, vault
identity, generation, priority seed, and Move selection guards already used by this
flow.

Carry the outcomes and capability through `dispatchCapture`,
`CaptureInboxRouteSubmitContext`, route confirmations, refusal handling, and route Back.
Entering and leaving inbox routing must neither lose reasons nor resurrect a highlight
fallback. Do not emit route answers for card-resolved rows. Reuse Bob's route candidates
and conflict response rather than deriving vault behavior locally.

For `offer_close_stale`, `offer_close_refused`, `offer_verdict_conflict`, and stale
pull-in rejection, reopen through `openPullInChoice` with the remembered outcomes.
Refresh row data/descriptors from Bob, then reseed by key; retain surviving marks,
verdicts, reasons, and a surviving highlight. Retain filter/fold state where valid. The
existing `reseed(validKeys:)` only removes keys, so extend it or rebuild its row
snapshot explicitly rather than leaving old eligibility/consequence data attached. Do
not silently discard a still-present staged verdict merely because Bob now refuses it:
keep the judgment visible with Bob's error until the user changes it; Bob remains the
final write validator. Removed keys are dropped and counted once. Show the refusal and
announce how many staged choices were lost.

When a write-free stale-pick response lacks structured rows, perform at most one fresh
dry-run of the same guarded draft/seed to obtain authoritative rows. Recognize the
existing stale-pick error narrowly; do not classify every usage error as stale. Never
retry a write automatically, and never treat a timeout/unknown write outcome as a
write-free refusal. A failed refresh keeps the error and remembered choices available; a
refresh with no offers shows an empty recovery state and a route back to the draft, with
no enabled commit. Ignore refreshes after the draft/client or generation changes.
Preserve the original refusal while reseeding.

## 4. Render the staged result accessibly

Use `RelationSourceEffect` symbols and `CaptureEditorPalette` relation colours. Done
replaces the mark slot with `checkmark.circle` and the destination capsule with
`✓ Done`; Cancel uses `circle.slash` and `⊘ Cancel`. Titles become secondary without
strikethrough. Eligible rows show muted `unblocks {k}` detail when positive. Cancel
detail shows the quoted reason, or faint `no reason`. The inline field says
`Why cancel? (optional)` with `↵ · Esc back`. Keep it outside the row's toggle Button
hit target so text interaction does not toggle Pull In.

Add one consequence line above the footer in the Move card's detail-strip style. It
persists after advancing and shows the latest staged verdict's Bob-authored consequence
and scope. Highlighting a staged row shows that row's consequence; clearing its verdict
clears that message. Esc warning (amber) and refusal styling temporarily take
precedence. Announce changes politely using the existing model announcement channel
without duplicate announcements.

Add gated footer hints `⌃X Done · ⌃⇧X Cancel`, updated action labels/counts, and correct
accessibility instructions for verdict-only/mixed commits. Provide row context-menu and
VoiceOver actions `Mark Done`, `Mark Cancelled…`, `Pull In`, and `Leave`, sharing the
model actions and targeting the acted-on row. Pointer actions do not advance the
highlight. Accessible suffixes are `will be completed`, `will be cancelled: {reason}`,
or `will be cancelled, no reason`.

Use the existing 0.15-second snappy crossfade convention, disabled under Reduce Motion.
Do not reorder rows or jump focus during rendering. Account for the inline field/status
strip in panel sizing; verify minimum width, long reasons, filtering, and light/dark
appearances using the existing render-fixture tests.

## 5. Regression tests and docs

Extend the existing test suites rather than duplicating the engine or argv tests:

- `CapturePullInChoiceTests`: modifier/repeat/reason routing, native Cut, exact footer
  labels, exclusive outcomes, visible-order advancement, hidden choices, bulk marking,
  no fallback with verdicts, capability gates, and pick-only budget.
- `CapturePullInChoiceModelTests`: no capture during staging; prior-outcome restore and
  blur confirmation once; mixed/verdict-only commits; Shift/Control-Return;
  Esc/Control-[ arming, repeat, and disarm; each refusal with preserved choices and
  lost-count announcement; missing-payload stale-pick refresh and no write retry;
  timeout behavior and obsolete async response rejection.
- `CaptureInboxRouteChoiceModelTests`: pull-in/verdict/reason preservation through route
  confirmation and Back, with unchanged draft, seed, and open intent.
- `CapturePullInChoiceDesignTests`: Done, Cancel with reason, active reason editor,
  mixed counts, refusal/Esc strip, and reduced-motion-friendly rendering at supported
  widths and both appearances. Check context-menu/VoiceOver semantics as well.
- Update `CapturePullInOutcomesTests` for any label, navigation, and refresh API
  adjustments. Reuse phase 8's real-Bob fixtures and fake client harness.

Async tests must use bounded waits; never introduce an untimed continuation. Update
`docs/capture-previews.md` (Successor offers), `docs/keyboard.md` (Successor choices),
and `docs/protocol.md` (argv, compatibility, and refusal reopening). Document capability
fallback, optional reason and two-step Escape, root-only verdicts, recover-only
cascades, and no automatic write retry. Keep the existing documentation structure and
published URLs.

## 6. Verify and finish only this phase

Run `just portable-check` and `just docs-check` from the opened Mac checkout. Format
changed lines only. Portable checks cannot compile AppKit/SwiftUI targets. Read
`tailnet.md` through `/sase_memory_read` before remote access. Per the Mac `AGENTS.md`,
use a disposable scratch copy on the Mac (rsync excluding `.build` and `.git`), run
`Scripts/bundle.sh` and `just typecheck-tests`, and remove that copy. Do not use Bryan's
checkout or `just install`. If the Mac is offline, explicitly report that the local
macOS scratch targets were unverified; still obtain the CI gate before declaring the Mac
implementation verified.

Run the repository's actual `Scripts/macos-ci-check` (capital S). It pushes a throwaway
snapshot branch and waits for the exact-SHA workflow, including macOS lint, Build, Test,
Bundle, signature, launch, and installation checks. Hand long commands to
`/sase_monitor` before starting them and wait for the monitor-start command itself to
exit. Record the exact SHA and successful run URL. Fix failures and verify the changed
tree again. Any subsequent push requires watching its own exact-SHA macOS run to green
with `/sase_monitor`; do not substitute a prior SHA's success. Finish repository commits
through `/sase_final` as instructed by its context, without manually invoking git
commit.

If a check fails identically on the clean base tree, record evidence as
`sase bead note bob-cli-hn.9 'PROPOSED FOLLOW-UP: <summary — reproduction and existing task ID if known>'`
and follow the phase's clean-base exception rather than leaving it open for that
unrelated failure. Record other discovered out-of-scope work the same way; do not create
task beads.

After implementing and verifying the assigned scope, run
`sase bead epic-symbols bob-cli-hn.9`. Resolve any remaining entries or re-key them to a
still-open parent/later phase before closing. Close **only bob-cli-hn.9** with
`sase bead close bob-cli-hn.9 --note "<implementation and verified test/CI evidence>"`.
Do not close bob-cli-hn or any ancestor plan bead. If a host notice attaches the worker
to a closed bead, verify whether the work landed before reopening anything; use
`sase bead open bob-cli-hn.9` only if the work is genuinely unfinished.
