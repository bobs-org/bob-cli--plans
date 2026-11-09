---
tier: tale
title: Finish Record Card completion presentation and land the epic
goal: Make Record Card choices visible and reliable, verify the repaired tree, and
  close bob-cli-ir.10.1 and parent phase bob-cli-ir.10.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ir.10.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ir.10.1
status: done
---

- **PARENT:**
  [202611/mac_record_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_record_card.md)
- **BEAD:**
  [bob-cli-ir.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.10.1.md)

# Finish Record Card completion presentation and land bob-cli-ir.10.1

Repair the bounded presentation defects found while landing `bob-cli-ir.10.1`, then
finish that epic and its parent phase `bob-cli-ir.10`. This is one coder's work in
`bob-mac-capture`, with real-Bob fixture support as needed. There is no later land agent
for this tale: its coder owns the closeout below. Do not close the containing epic
`bob-cli-ir`.

## Binding context and completed audit

Read these through their audited interfaces:

- `sase bead read bob-cli-ir.10.1 -r "Need the landing audit and follow-up triage"`.
- `sase artifact read plan:202611/mac_record_card.md "Need the original acceptance contract"`.
- `sase artifact read plan:202611/record_notes.md "Need the parent phase contract"`.
- `sase bead read bob-cli-ir.10 -r "Need the parent phase and decisions"`.
- `sase memory read decisions:mac-capture-is-a-thin-client -r "Need the client boundary"`.

Open `bob-mac-capture` using `/sase_repo` and read its `AGENTS.md`. All Mac paths below
are relative to that returned checkout, and bob-cli paths are relative to the primary
checkout. Do not use another agent's checkout. Before changing mounted documentation,
read `docs.md` through `/sase_memory_read`.

The lander read the epic and every note on all four closed phases:

| Phase               | Source evidence          | Audit result                                                                                                                                                      |
| ------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob-cli-ir.10.1.1` | bob-cli `5e49fa5e`       | Named `-F/--record-field`, next-missing `-n`, `record.edit`, assist version 1, read-only errors and apply/reparse tests exist.                                    |
| `bob-cli-ir.10.1.2` | Mac `1e6fb18`            | Optional record decoding, process argv, portable presentation/focus, cached routes, real-Bob fixture generator and old-Bob tests exist.                           |
| `bob-cli-ir.10.1.3` | Mac `7933c7e`            | Scoped completion requests, native undo, keyboard routes, mixed-draft submit blocking and dismissal guards exist; surface integration needs the repairs below.    |
| `bob-cli-ir.10.1.4` | Mac `8c13ac4`, `4bca5da` | Card, semantic tints, accessibility identities and renders exist. Compilation/test fixes in its notes are present. The More alignment proposal remains epic work. |

The successful macOS run is
https://github.com/bobs-org/bob-mac-capture/actions/runs/42611020479. The lander checked
its Build, Test, Bundle, Launch and Install steps. Its snapshot SHA
`1e507e9cf9d982f8fb52e439a651437d4635dc97` has tree
`b9ee8b3024615983df9c95ade77cbdb4d90bdceb`, exactly the tree at audited Mac HEAD
`4bca5da078afd182d3bf38b5eac1219db338008d`. This resolves phase .3's original unverified
macOS note; it does not verify future edits.

Audited bob-cli HEAD and origin/master were `941ee724`. Since the child epic began, the
separate record-update work landed as `8cc4a754`, `12641768`, `96196052`, `1c4939b9`,
`a729a8fd` and `941ee724`. It preserves named-field completion and extends `record.edit`
with safe current-value prefills. It also adds `record_target` responses at a create
route, exposing the invisible-selection defect below. `070d2aac` adds sized-close
parsing; the Record Card must continue consuming Bob's semantic output without
implementing that grammar. No unrelated Mac commits landed after `e5b5889` through the
audited Mac HEAD. Recheck drift before finishing.

No memory changes are authorized by this tale. Preserve schema version 1,
`CaptureGrammarGate.minimumRevision`, Cargo versions and CHANGELOG.md. The accepted
outer decisions remain `seed_records=yes`, `memory_glossary_capture=yes`,
`memory_glossary_record_note=no`, and `memory_decision_record_types=no`; config,
seeding, and the authorized Capture memory change belong to other outer phases.
Property-sheet previews, full revisit/current-before-after cards, dedicated rich target
presentation and Similar-records UI remain the existing outer phases' work.

## 1. Make rendered completion and keyboard acceptance agree

The concrete defect spans these existing paths:

- `CapturePanelView.auxiliaryRegion` chooses `RecordCardView` before the generic
  completion list whenever `recordPresentation` exists.
- `RecordCardView.candidates(for:)` returns rows only for `record_value`, so a
  `record_field` response is hidden. The checked-in `record-complete-field.json` fixture
  offers six such rows for a fresh `- ` bullet.
- `CapturePanelModel.recordCompletionListVisible` nevertheless treats every current
  nonempty record completion as visible. The key router then lets Tab/Return accept its
  selected candidate.
- Latest Bob's `capture_complete/records.rs::target_probe` also returns `record_target`
  at the caret at the end of `Ramen Nagi @eat/` when the folder has existing notes.
  Those hidden candidates replace the route with an existing record, changing create
  intent to update intent when accepted.
- The value view does not read `selectedCompletionIndex`; `inlineValues` shows only
  `candidates.prefix(6)` and a noninteractive ellipsis. Keyboard navigation can select
  invisible values and gives no visible indication of which value Return will use.

Give every accept-enabled completion context a visible, accessible selection. Keep the
card and completion UI in the existing picker slot; reuse the existing generic list for
contexts the card does not render, or integrate that list within the card. This generic
fallback is the required compatibility repair for `record_target`; do not build the
later rich revisit surface here. Render the field-name rows from Bob and apply their
exact replacements/continuations. Ensure Tab's advertised first-missing behavior cannot
silently accept a hidden existing-record target.

For record values, show the selected candidate distinctly and keep it visible as
selection changes, including candidate seven and later. A bounded scrolling list or a
selection-following window is appropriate; an ellipsis must not hide the active choice.
Expose selection to VoiceOver and preserve stable identity across a same-draft refresh.
Only Bob's candidate/edit bytes may enter the draft. Preserve the request generation,
item, field, cursor, selection and client guards, and revalidate before acceptance.

## 2. Repair the existing card layout and footer

Resolve phase `bob-cli-ir.10.1.4` note #4: required/set rows and expanded optional
Visited/Url rows must share one state/label/value alignment. The current row HStacks
have different intrinsic widths: only empty optional values contribute a Spacer, and the
preview uses a centered VStack while production uses a leading LazyVStack. Use a
consistent full-width row layout in both render and production paths. Test the actual
production layout as well as the render helper at minimum and wide widths.

`RecordCardView.footerIsRefusal` currently returns true for _any_ nonempty
`model.statusText`, while `applyParse` sets ordinary success to `Ready`. `footerCopy`
likewise lets `Ready` replace the missing-field and keyboard hints. Distinguish current
errors/refusals from ordinary status and show the useful `presentation.footerHint` when
there is no blocking message. Clear or supersede stale field messages when a newer parse
resolves them; keep current Bob diagnostics and whole-draft blockers authoritative.
Verify complete/incomplete/invalid states and light/dark/increased-contrast appearances.
Do not use color as the only state cue.

Finish the original editor-route requirement too: `CaptureEditorPalette` and
`CompletionRowContent` currently describe the emoji as an adornment on the Record Card
only; the editor route receives just a background wash. The binding card-surface
contract calls for the record emoji on the route chip itself. Add a presentation-only
adornment anchored to Bob's owning-item `record_route` span, using the editor's drawing
or overlay layer. Preserve the exact underlying string, UTF-8/UTF-16 offsets, copy,
selection, undo and caret behavior; do not insert an emoji or attachment character into
the editable text. Keep absent metadata neutral and cover two record types in one draft.

## 3. Cover real interaction and intervening Bob changes

Extend the existing Record Card tests and isolated fixture generator rather than
introducing a second schema/parser. Add current-Bob fixtures only where necessary;
record their revision/inputs/argv. Never generate fixtures from the live vault. Keep
older fixtures as compatibility coverage where useful.

Use bounded/cancellable panel tests with responses matching the actual draft and cursor
at each step. The original panel suite's single fixed completion fixture is insufficient
evidence for a full chain. Required regressions:

- Start a new record, accept status then category then location, follow each Bob
  continuation, end with no value list, and submit once. Assert zero mutation calls
  during assistance (exclude `capture --dry-run` from that count).
- Start a fresh `- ` field-name menu; the selected field is visible and accepting it
  uses Bob's returned insertion. Cover backward missing-field navigation too.
- Exercise seven or more value suggestions, keyboard selection, Return/Tab acceptance,
  visual/accessible selected state and scrolling. Assert the accepted text matches the
  displayed selected row.
- Feed current Bob's `record_target` response at a create route. No hidden candidate can
  turn the draft into an update. Test the visible generic fallback and preservation of
  current-value `record.edit` responses without adding full revisit UI.
- Exercise empty suggested-text entry through named field plus `record.edit`, ordinary
  filtering while typing, optional/default editing, and undo/redo. Metadata refreshes
  must not edit the draft.
- Check the `@eat#` New restaurant acceptance and `//` literal escape, a blank-title
  route, mixed-item submit blocking, old Bob without assist version, and stale replies
  after selection/item switches and Escape. Add focused missing regressions; retain
  existing coverage rather than duplicating every existing test.
- Render More alignment, selected values (including overflow), field-name/fallback
  lists, ordinary versus error footer states, and the editor's route emoji adornment.
  Check Unicode/multiline selection and copy offsets remain unchanged. Inspect the
  generated images.

Keep ordinary capture, snippets, indentation, modifier keys and task-picker behavior
green. Update `docs/keyboard.md` or `docs/capture-previews.md` only as needed to
describe the final visible behavior. Recheck the source and commits that landed since
the audit; incorporate compatible preview-worker edits without undoing them.

## 4. Verify the repaired tree before closeout

Run `just portable-check` in the Mac checkout. Run `just docs-check` there for mounted
docs changes and `just check-scripts` for fixture-tool changes. In bob-cli, use
`just check` for file-change verification; do not run `just check-full`. A
`just check-full` failure alongside a passing `just check`, if independently reported,
is a test-infrastructure issue rather than unfinished epic work.

Never land uncompiled Swift. For AppKit/UI source or test changes, read tailnet memory
and attempt the documented disposable Mac scratch build: rsync without `.git`/`.build`,
run `Scripts/bundle.sh` and `just typecheck-tests`, then remove the scratch copy. Never
build in Bryan's checkout or install the app. If Mac scratch access is unavailable,
report it; macOS CI compilation remains required.

Run `Scripts/macos-ci-check` through `/sase_monitor` for a throwaway exact-tree
snapshot. Use the monitor continuation to finish this same tale, inspect test/render
evidence, and perform step 5 after green. Keep the staging command running until it
exits, then finish the turn through `/sase_final`. Do not wait for a staged successor in
the staging turn. Fix failures caused by the changes. The final closeout must not depend
on the final code commit's future SHA, push or CI: the precommit snapshot gate verifies
the working tree first, so close the epic in the same finishing turn as the final code.
After any independently required push, watch its exact-SHA macOS run via `/sase_monitor`
as instructed by the linked repo.

## 5. Finish this landing, including the parent phase

The lander completed follow-up triage and recorded it on `bob-cli-ir.10.1`: the only
proposal, `.4` note #4 about More alignment, was declined as a separate task because the
epic introduced it. It is fixed by this tale. There were no distinct external
follow-ups, duplicates to corroborate, or new task beads. Carry that outcome into the
epic close note. If genuinely new unrelated work is discovered, use `/sase_new_task` and
record its outcome; defects caused by this epic must be repaired before closing.

After checks pass, re-read `bob-cli-ir.10.1`, all four children and their notes, and
`bob-cli-ir.10`. Confirm every outstanding audit finding is resolved and no new child
work or conflicting drift appeared. Do not reopen completed phases solely to record
these fixes. Close any genuinely completed new descendant before its containing epic if
the host created one for this work; never bypass descendant readiness.

1. Run `sase bead epic-symbols bob-cli-ir.10.1`. Resolve each listed entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting it under Symvision
   policy; re-key only when an identified still-open later bead actually needs the
   exemption. The audit found none, but check again.
2. Close normally with
   `sase bead close bob-cli-ir.10.1 --note "<verified source, child-note outcomes, drift integration, repair tests, Mac snapshot SHA/CI URL, and follow-up triage>"`.
   If symbols block the close, fix them and retry. If an unfinished phase blocks it,
   complete/reopen it deliberately; never force a successful nested landing or use force
   merely to make the close succeed.
3. Run `just symvision` if available in the affected repositories, and explicitly record
   when the recipe is unavailable (it was absent from both audited justfiles).
4. Open the plans sidecar through `/sase_repo`. Read the artifact with
   `sase artifact read plan:202611/mac_record_card.md "Need completed epic plan for status update"`
   and set only its frontmatter `status: done`. Use its resolved PLAN path, not a
   hard-coded workspace path; include the edited plans repo in `/sase_final`.
5. Re-read `sase bead read bob-cli-ir.10.1 -r "Need the parent link"`. Its concrete
   parent is phase `bob-cli-ir.10`. Verify this child satisfies that phase's Mac Record
   Card and guided-entry scope, and run `sase bead epic-symbols bob-cli-ir.10`.
   Retire/re-key its entries under the same rules, then close only that parent phase
   normally with
   `sase bead close bob-cli-ir.10 --note "<child repairs, parent acceptance coverage, final bob-cli/Mac revisions and exact-tree macOS CI evidence>"`.
   Leave containing epic `bob-cli-ir` and `plan:202611/record_notes.md` for its waiting
   land agent. If the parent is incomplete or ambiguous, record a precise blocker note
   there and report it; do not force it closed.

Finish through `/sase_final`, declaring the final code and plan-status edits together.
Report exactly which beads closed and any parent blocker. No later lander or postcommit
step is needed to complete this tale's authorized closeout.
