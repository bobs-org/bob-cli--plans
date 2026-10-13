---
tier: tale
title: Bob Mac Capture successor choice before submit
goal:
  Complete bob-cli-7c.7 with grouped offers, explicit keyboard selection, safe guarded
  submission, and Swift verification.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7c.7
bead: bob-cli-7c.7
create_time: 2026-10-13 14:20:07
status: wip
---

- **BEAD:**
  [bob-cli-7c.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/bob-cli-7c.7.md)

# Bob Mac Capture: choose successor offers before committing a capture

Implement assigned phase `bob-cli-7c.7` of `bob-cli-7c`. Return on a capture draft with
successor offers opens one in-panel choice step; confirming passes opaque offer keys to
the existing atomic `bob capture`. Esc returns to the unchanged draft, and Shift-Return
commits the close with no successor picks.

## Scope and authority

- Read
  `sase bead read bob-cli-7c.7 -r "Need the assigned phase and final epic decisions"`
  before implementing. The phase is already assigned and in progress; never set its
  status by hand. At planning time it had no earlier implementation or remaining-work
  notes, and dependency `bob-cli-7c.3` was closed.
- Read the binding design through
  `sase artifact read plan:202610/successor_offers.md "Need the approved mac_choice design"`.
  The accepted report is
  `research:202610/successor_pull_in_offers/successor_pull_in_offers.md`, also read
  through `sase artifact read`. The epic and current Rust contract take precedence over
  earlier sketches in that report.
- Work in the linked `bob-mac-capture` checkout obtained through `/sase_repo`:
  `sase repo open bob-mac-capture -r "Implement bob-cli-7c.7 Mac successor choice"`.
  Read its `AGENTS.md`. All Mac paths below are relative to that returned checkout.
  Rust/doc paths explicitly labelled bob-cli are relative to the primary checkout.
- This is bounded single-agent integration work: the Rust planner, keys, selection flag,
  write-free refusal, and contract already exist. Reuse the app's inline prompt, process
  client, preview, presentation, and fake-bob testing patterns. Add focused choice
  types/view/tests rather than refactoring the large panel model or porting capture
  grammar or eligibility logic to Swift.
- Honor the epic's accepted decisions without asking again. This phase edits no SASE
  memory. Decision-record and glossary work belongs to `bob-cli-7c.8`.
- No destination override on the Mac, new CLI grammar, plugin changes, vault writes
  outside bob, auto-selection, auto-submit for a lone offer, or installation into
  Bryan's running Mac app is part of this phase.

## Verified implementation seams

`Sources/CaptureCore/CaptureModels.swift` defines `CaptureCommandSuccess`,
`CaptureCommandFailure`, `TaskCompleteUnblocked`, and `TaskCompleteStillBlocked`. Its
additive fields already use `decodeIfPresent` and compatible initializer defaults.
`BobProcessClient.capture` builds argv before the `--` delimiter; `decodeCaptureResult`
preserves structured failure JSON even on nonzero exit. `isUnsupportedOptionFailure` is
an existing helper used by the agenda endpoint.

`Sources/BobMacCapture/CapturePanelModel.swift` has `submit`, `completeSubmit`,
`completePreview`, `startLivePreview`, `inlinePromptVisible`, `clearInlinePrompts`,
`clearEditorInputLockIfFree`, and `editorTextDidChange`. Only successful submission
clears the draft and dismisses the panel. Live previews have analysis-generation guards;
explicit previews/submissions have request IDs. Batch normalization loses the enclosing
response unless its metadata is retained separately.

`CaptureKeyCommandRouter.swift` prioritizes prompts/pickers ahead of editor commands;
`CapturePanelController.swift` constructs the routing context and dispatches every
command. `CapturePanelView.swift` hosts the prompt cards, `completePreviewItem`, and
`closePreviewItem`. `CaptureSuccessorPresentation.swift` shares successor row and
notification copy between completion and close cards.

## 1. Decode the implemented Rust contract and carry choices through the client

Use bob-cli `docs/task-dependencies.md` §§12.5–12.6, 12.12, 12.14, `docs/capture.md`,
`src/native/capture/output.rs`, `src/native/capture/pull_in.rs`, and
`tests/cli/capture/successor_offers.rs` as the wire-format authority.

Add small Codable/Equatable types, with Sendable where their owners require it:

- `offer`: `key`, `n`, `group`, optional `project`, `destination` (`day_file`,
  `entry_name`, `entry_role`, `fallback`), `selected`, `block_id_needed` on
  `TaskCompleteUnblocked`.
- `trigger` and `hide_removed` on unblocked rows; `trigger` and `project` on
  still-blocked rows. Preserve unknown strings for neutral presentation.
- Top-level optional `pull_in` on successful responses: `supported`, `offer_count`,
  nullable `choice`, `picked_count`, optional `groups`, each with `group`, `project`,
  and `folded {hidden, already_planned, sub_projects}`.
- Optional `pull_in` and `offers: [TaskCompleteUnblocked]` on failures. The
  `pull_in_choice_required` refusal is exit 1 with `ok: false`, an error string,
  capability metadata, and these top-level offer rows. Do not expect its rows inside a
  normal capture item. The current refusal has no `still_blocked` array; do not invent
  muted rows that the response does not provide.

Decode additions with `decodeIfPresent` and preserve legacy payloads, current schema
handling, encoding round trips, and source-compatible initializer defaults. A missing
capability is unknown/legacy, never evidence of supported selection. An absent or empty
offer key must never become a selectable row or a positional fallback.

Extend `BobProcessClient.capture` with defaulted selection and guard parameters.
Represent an explicit selection separately from no decision: ordered keys versus `none`,
never offer numbers or locally derived keys. Add `--pull-in <comma-joined keys|none>`
and `--require-pull-in-choice` before `--`, preserving the draft as one argv element,
clipboard behavior, lanes, cancellation, and priority-roll seed. An explicit selection
requires a capability advertisement from the matching preview or refusal. Live dry-runs
remain write-free and never themselves demand a choice.

Implement the old-binary guard fallback in the process client, at the boundary with raw
exit status and stderr available. Retry exactly once, without the guard, only when clap
reports that exact guard flag as an unexpected argument before any write (exit 2). Do
not retry structured refusals, a rejected `--pull-in`, other unknown arguments,
timeouts, arbitrary errors, or malformed JSON as legacy compatibility. Keep the same
draft, seed, and other flags on retry. Cache unsupported guard capability for the
resolved executable identity; invalidate it when the client/executable changes,
including an installed binary replacement. Use the existing state queue for mutable
client state. Show a concise capability note when this legacy path is known; do not
imply the new choice behavior is available on that bob.

## 2. Add the choice state and gate every submission path

Add a dedicated choice-state type with the draft snapshot, process-client identity,
original `openAfterCapture` intent, priority seed, offered rows, group metadata,
highlighted key, empty marked-key set, and local presentation/filter state. Aggregate
offers from both task-completion and Pomodoro-close results across the whole draft,
deduplicating by opaque key and ordering by bob's `n`. Bob owns net eligibility and
numbering; the app only groups and renders supplied rows.

Retain the enclosing preview's pull-in metadata together with its exact draft and
client/generation provenance for both live and explicit previews. Clear this state when
the draft, process client, or preview validity changes. A trimmed pending-close preview
cannot authorize submission of its untrimmed draft.

Split submission orchestration from the actual process dispatch so confirming a choice
continues with explicit picks without re-entering the public inline-prompt guard or
reopening the chooser:

1. Keep the existing no-draft, pending-close, inline-prompt, and in-flight guards.
2. With a current supported preview and offers, show the choice step and do not launch a
   writing capture yet. Mark nothing, including for a single offer.
3. Without explicit picks, send the guard unless this binary is known unsupported. This
   covers absent/stale previews, a legacy preview, and an offer appearing after a
   zero-offer preview. A guard success with no offers goes through ordinary success.
4. On `pull_in_choice_required`, end the in-flight state and open the choice step from
   the refusal's capability and offer rows. Keep the draft/panel; do not send a failure
   notification or dismiss the panel for this expected result. Reject malformed choice
   refusals visibly without silently committing with none.
5. Confirm with marked keys, or the highlighted selectable key if none are marked.
   Shift-Return explicitly passes `none`, regardless of marks. A successful capture
   alone clears the draft, notifies, optionally opens notes, and hides the panel.
6. Esc cancels only this step, restores editor focus and the original draft, and
   performs no capture write. Draft edits, replacement, clearing, stash transitions,
   panel lifecycle reset, and process-client changes invalidate the choice and its
   picks. Late preview/refusal callbacks must not resurrect it or replace its list.
7. A stale key or other write-free failure keeps the draft and error visible; refresh
   offers before another attempt. Never fall back to linking by number or silently retry
   with an empty selection.

Extend `inlinePromptVisible`, `clearInlinePrompts`, editor locking/unlocking, completion
suppression, agenda/auxiliary ownership, draft-change cancellation, accessibility
announcements, and focus restoration consistently with existing prompts. Preserve
`isSubmitting`/request-ID protection so repeated confirmation cannot dispatch two
writes, and consume Return repeats rather than letting one held key both open and commit
the choice.

## 3. Render the preview and chooser, and wire the actual key monitor

Keep preview cards informational. On both `!` and close cards, distinguish offered rows
from merely recovered rows, retaining bob's global `offer N` number. Group under
Hand-offs, Project ‹note›, and Project finished. Show the destination and `needs ^id`
where supplied; the non-empty project's `^prj` is its group header, not a selectable
resident. Add `↩ choose what to pull in` only when offers exist.

Create a focused SwiftUI choice card within the existing auxiliary region with a bounded
scroll viewport, clear highlight and check marks, and the same group/row presentation.
Every offer remains reachable for 1, 6, 11, and 20+ rows. Include a local text filter
and expandable project groups for long lists; opening a group or filtering cannot
discard hidden marks or rewrite keys. Typing a filter reveals matching rows in collapsed
groups. No selection cap. Muted resident rows show their blocked reason and are never
selectable; folded count lines show hidden, already planned, and sub-project counts. Use
the contract's wording for sub-projects:
`N sub-projects stay blocked until the next reconcile`.

If the enclosing result has a plan-link meter, reuse the existing budget presentation to
show the projected count for the effective picks; it is a warning, never a local
refusal. Do not read the vault to fill missing budget or group metadata.

Add routing context and commands to `CaptureKeyCommandRouter` and the actual
`CapturePanelController` monitor/dispatcher. The step owns keys ahead of completion, the
ordinary picker, and editor commands:

| Key                   | Choice action                                                      |
| --------------------- | ------------------------------------------------------------------ |
| Up/Down               | Move between visible selectable offers, scrolling to the highlight |
| Space                 | Toggle the highlighted offer                                       |
| Command-A             | Mark every selectable offer in the entire choice set               |
| Return / keypad Enter | Submit marked offers, otherwise the highlighted offer              |
| Shift-Return          | Submit with `--pull-in none`                                       |
| Escape                | Return to the unchanged draft, with no write                       |

Keep text-filter typing in its own field without letting Space or Command-A mutate the
capture draft. Make the footer state the effective pull-in count and explain Return,
Shift-Return, and Esc. If filtering leaves no highlight and no marks, ordinary Return
does not accidentally choose none. Preserve the original Command-Return
open-after-success intent through the choice; modified Return inside the step must not
escape to a generic editor submit/newline path. Check panel sizing/visibility and
accessible labels/focus as well as model methods.

Update `CaptureSuccessorPresentation.swift` with `project_member` → `project member`,
`not_selected` → `not pulled in`, `no_destination` → `no open Pomodoro`, and meaningful
still-blocked `project` wording using the supplied project/date/reason data. Report what
was pulled in on the existing single success notification; recovered-only captures
retain the existing unblocked summary. Keep `breaker` and `not_planned_today` copy for
legacy payloads. Display `trigger`/`hide_removed` accurately for surfaced project rows
without falsely describing an unpicked row as linked. Update the Mac README's
capture/key behavior to describe the new step and legacy capability note; the primary
repository contract is already written.

## 4. Regression coverage

Extend `Tests/Fixtures/fake-bob` with current-contract fixtures and explicit legacy
fixtures. Keep existing default scenarios stable where possible; use opt-in fixture
modes for delayed previews, offers, refusal, unknown-guard retry, and stale-key error.
Record exact argv and capture invocation counts. Add focused test files rather than
putting the feature's entire test matrix into the large general panel test file.

- Core decoding: both close forms, batch-level `pull_in`, zero offers, project folds,
  muted rows, `trigger`/`hide_removed`, missing additive fields from older bob, refusal
  `offers`, omitted `not_linked` on linked rows, and unknown strings.
- Client: flags before `--`, a draft with spaces/newlines, keys containing `#^` and
  digest/duplicate suffixes round-tripped unchanged, `none`, guard success/refusal,
  exactly one safe legacy retry, cached rejection, executable change, and no retries on
  unrelated errors or stale selections. Preserve live-preview `--dry-run --no-clip`.
- Choice/model: zero writes on open/Esc; zero pre-checks; single highlighted default;
  all/none/subset; 1/6/11/20 offers; one chooser across a batch; the `sase_foo` project
  fixture with two residents and a header-only `^prj`; disabled rows/folded counts;
  filter/collapse preserve marks; a zero-result filter; client/draft invalidation;
  delayed preview/refusal; no-preview guard path; zero-offer preview race; stale keys;
  no duplicate submit; successful dismissal only; open-after intent; unrelated captures.
- Router/controller: every key above, numeric keypad Enter, prompt priority, repeats,
  Shift-Return not inserting a newline, Command-A not selecting the draft, and Esc not
  closing/stashing the panel. Verify dispatcher/context wiring, not just enum mapping.
- Presentation/notification: numbering/group order/destinations/id-needed, all new
  reason copy, still-blocked project reason, legacy copy, and one pulled-in
  notification.
- Add or extend a render fixture for the grouped choice card with enough rows to scroll.
  Ensure active rows and the footer remain visible at supported panel sizes.

Cite the relevant SO vectors in tests (notably SO1–SO6, SO27, SO29, and fan-out). Rust
already verifies the vault writes for these vectors; Mac tests verify the exact selected
keys and single-commit behavior, not a second Swift eligibility engine.

## 5. Compile, test, publish evidence, and close only the assigned phase

The linked repo's current `AGENTS.md` supplements the epic's older CI-only assumption.
Never land uncompiled Swift and never treat a formatting pass as compilation.

1. Run `just portable-check` in the linked checkout. It parses/lints every Swift file
   and builds/tests portable CaptureCore/RefsCore. This planning host has neither
   `swift` nor `swift-format`; if unchanged, use a scratch copy on `athena`, following
   `sase memory read tailnet.md -r "Need remote Swift verification hosts"`. Copy this
   opened checkout, excluding `.build` and `.git`; never edit another machine's existing
   checkout. Clean up only the scratch directory you created.
2. Because this change touches `Sources/BobMacCapture` and its tests, attempt a scratch
   copy on SSH alias `mac` with bounded connection timeouts. Run
   `./Scripts/bundle.sh --identity -` and
   `./Scripts/xcode-swift.sh build --build-tests`, plus the relevant/full test suite
   when available. Remove the scratch copy afterward. Do not run `just install` or touch
   the user's installed app or checkout. If the Mac is offline, record that this local
   macOS verification was unavailable and obtain CI compilation evidence.
3. Arrange normal SASE finalization/publication of the linked-repo changes so the
   changed commit gets the existing `.github/workflows/ci.yml` run. If local macOS
   compilation is unavailable, push a review branch and use a pull-request CI run before
   landing; do not land uncompiled app targets on master. Do not invoke raw
   `git commit`; use the applicable SASE workflow. Confirm CI for the actual changed
   commit, not an earlier green run: formatting, Build, Test, Bundle, signature, launch,
   and install gates. A red lint parse step can skip all later stages; inspect those
   stages too. Use `/sase_monitor` for remote builds/CI waits and put remaining
   verification and closure work in its `--next`. Wait for the monitor-start handoff
   command itself to exit before ending the turn.
4. Fix failures caused by this work and repeat the affected checks. If a failure
   reproduces identically on an isolated clean base with the same toolchain/command,
   record the failure, base commit, comparison, and any existing tracking bead as a
   `PROPOSED FOLLOW-UP:` note on `bob-cli-7c.7`. The user's clean-base exception means
   such a failure alone does not keep the phase open. Report any macOS targets that
   remain unverified honestly; do not claim skipped steps passed.
5. Record what changed, portable/macOS/CI evidence and commit/run identifiers, and any
   limits on `bob-cli-7c.7`. Do not create follow-up beads. Use
   `sase bead note bob-cli-7c.7 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`
   for discovered out-of-scope work; the epic land agent triages it.
6. Immediately before closing, run `sase bead epic-symbols bob-cli-7c.7` again. It
   reported no entries during planning. If entries now exist, resolve them or re-key
   each justified exception to a still-open parent/later phase and verify the result.
7. Close only `bob-cli-7c.7` using
   `sase bead close bob-cli-7c.7 --note "<implemented behavior and actual verification>"`.
   Do not close `bob-cli-7c`, `bob-cli-7c.8`, or any ancestor plan bead. Use
   `/sase_final` as the last action before a normal ending response; successful SASE
   plan/monitor handoffs use their own mechanical completion path.
