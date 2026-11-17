---
tier: tale
title: Complete Record Card typed stages and Tab fill-out
goal:
  Finish bob-cli-ms.6 with all typed field editors, safe preview and save-and-advance
  behavior, approved browse keys, tests, and a verified plugin deployment.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ms.6
proposed_with: codex/gpt-6-astra
bead: bob-cli-ms.6
create_time: 2026-11-17 18:05:03
status: wip
---

- **PARENT:**
  [202611/record_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card.md)
- **BEAD:**
  [bob-cli-ms.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.6.md)

# Complete Record Card typed stages and Tab fill-out

## Scope and completion

Complete assigned phase **bob-cli-ms.6**, the `record-stages` phase of
`plan:202611/record_card.md`. One coding agent can finish this medium tale: the guarded
writer, per-record read, session, modal, choices, and test harness have landed. The
seven editors share one stage controller and preview path. The phase has no prior
implementation attempt with a recorded unfinished remainder; its existing note supplies
mandatory co-landing requirements.

The result is an inline editor for every record field kind, with bob-owned validation,
current-preview-only typed saves, safe Tab/Shift+Tab fill-out, retained drafts after
stale reads, and the missing approved browse keys. Verify and deploy the result, then
close only bob-cli-ms.6. Do not close bob-cli-ms or another ancestor. Do not create
beads. Record discoveries as `PROPOSED FOLLOW-UP:` notes on bob-cli-ms.6 for its parent
land agent.

## Authority and starting evidence

Read the current phase with
`sase bead read bob-cli-ms.6 -r "Need current phase scope, decisions, and notes"`. Read
these artifacts through `sase artifact read`:

- `plan:202611/record_card.md`: shared contract and `record-stages` scope.
- `research:202611/record_card/record_card.md`: normative design, subject to the epic's
  settled overrides.
- `plan:202611/record_card_field_jump_co_landing.md`: the phase note fixes the answers
  to `record_card_digits=pick`, `climb_keeps_field=yes`, and `scoped_force=yes`; do not
  re-ask them.

The epic's final decisions remain `inline_title=hide` and `decision_record=no`. No
memory files may be edited. Phase 7 owns activating the Properties/inline-title hide,
sheet entry points, record `==`, navigation-hotkeys, and Command Mode. Keep those
behaviors unchanged in this phase. Identity fields, schema mutation, record renaming,
note appends, and multi-field saves remain outside this tale.

Open bob-plugins with `/sase_repo` and use only the returned checkout. Read its
AGENTS.md. Before implementation, refresh history and phase notes and preserve newly
landed work. The inspected bob-plugins base was **c1da817**, Ledger Tools **1.131.0**,
containing closed phase ms.5. Its relevant state is:

- `242n-plugin-record-card-model.js` dispatches only choices; other kinds have a YAML
  fallback. It captures only scalar draft/filter/caret state.
- `242o-plugin-record-card-dom.js` paints choices and rebuilds the stage on paint.
- `242p-plugin-record-card.js` is 794 lines and owns the modal, direct writes, refresh,
  close keys, and input handling. It has no typed-stage implementation.
- `242l-plugin-record-update-service.js` and `242m-plugin-record-update-session.js`
  already expose preview, apply, direct apply, Undo, and optimistic sheet updates.
- `242aa-plugin-guarded-update-session.js` provides a 250ms debounced `setDraft`,
  `previewNow`, `apply`, `applyOps`, `getState`, and `rebase`. `canApply` matches
  revision, expected fingerprint, identity, and vault generation. **Rebase clears the
  preview but does not schedule another one.**
- `fields[].values` from `records show TYPE STEM` is the full ranked suggestion list;
  `top_values` is capped at five. Invalid properties have `value: null` and preserve the
  input in `raw`. Update reports instead expose planned properties, `warnings`,
  `consequence`, and typed errors.
- The co-landing gaps are confirmed: non-choice digits silently return; browse Shift+J/K
  is absent; `recordSchema()` passes only `filePath`; the album entry chain drops a
  requested field. Record Card CSS is already isolated correctly.
- Baseline verification passed all **33 tests** in `test-ledger-tools-record-card.cjs`
  and `test-ledger-tools-record-update.cjs`. `sase bead epic-symbols bob-cli-ms.6`
  reported no entries at planning time.

All plugin source paths below are relative to bob-plugins. Keep edited fragments at or
below 1000 lines, edit `src/` rather than generated `main.js`, and register new tests in
the existing one-line `package.json` test:unit list without losing other epics' entries.

## 1. Finish the approved browse behavior

Implement only the missing co-landing items before adding stages:

- On scalar choices, digits retain schema-order pick-and-save. On other browse rows,
  teach `Digits pick a choice on choice fields`, without moving or writing.
- Browse Shift+J/K selects the adjacent schema field including ghosts, clamps instead of
  wrapping, and announces `<Label>, field N of M`. At a boundary use `schemaCardBump`
  from `241xb` and announce `No next field` or `No previous field`. Repeated events
  produce no announcement/bump; respect reduced motion. These keys are inert in stages,
  text controls, menus, and composition. Keep them out of the hint strip, but document
  them and provide equivalent All actions controls.
- `recordSchema()` captures the selected field before closing and passes
  `{filePath, field}` to `openAlbumSchemaEntry`. Thread that optional field through
  `242d-plugin-album-entrypoints.js` and `openAlbumSchemaCard` to the modal's initial
  `selectedKey = field:<key>`, with `stats` fallback when absent/unknown. Existing album
  callers remain compatible. Keep changes in the 973-line `241y` minimal.
- Continue using `bob-record-card-*`, never `bob-album-schema-dialog` or its body class.
  Reuse the existing bump/live-region utilities rather than schema layout.

## 2. Add one typed-stage model and renderer

Use new cohesive fragments around the reserved `242q-plugin-record-card-stages.js` seam,
split into pure model, DOM, and controller pieces as needed. Add them in
`src/fragments.json` after their dependencies and extend the test exports. Keep `242p`
as the modal shell with small dispatch hooks; do not fork the album stack.

Determine the stage from the **list flag first**, then scalar kind and `suggest`. Seed
from the authoritative property value, or its raw value when invalid, never from
formatted labels or a schema default. Do not silently erase an invalid value when
opening its stage. Preserve explicit empty, zero, list members, and raw text as distinct
states. Track a stage identity, field key, draft revision, original value, raw draft,
list chips/item draft, picker selection, and focus/caret identity.

| Stage        | Required behavior                                                                                                                                                                                                                                                                                                                                                             |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Text         | Auto-growing single-value textarea. Preserve pasted text, including invalid newlines, and show bob's refusal; never strip a newline into a different saved value. Enter is save, not newline insertion.                                                                                                                                                                       |
| Suggest text | Accessible combobox over `fields[].values`, retaining bob's rank and counts while filtering. Explicitly choosing an existing scalar suggestion directly applies its exact value. Free typing goes through preview, including bob's `record_value_near` warning. Typing alone never implicitly chooses a suggestion.                                                           |
| Date         | Text input accepting `today`, `t`, `yesterday`, and ISO. Derive the parsed echo from bob's successful preview, not a second date parser. Browse `t` keeps its existing direct apply.                                                                                                                                                                                          |
| Number       | Text/numeric-inputmode control with schema min/max hints. Preserve incomplete or invalid tokens for bob to refuse. The envelope accepts numeric strings, so no lossy `parseFloat`, empty-to-zero, NaN, or clamping.                                                                                                                                                           |
| URL          | Text/URL control with bob's validation and hint; do not normalize or auto-prefix an invalid URL.                                                                                                                                                                                                                                                                              |
| Link         | Inline vault note suggestions, accepting a bare name or a complete wikilink. Use Obsidian's metadata cache and vault paths for suggestions and resolution relative to the record; preserve explicit aliases/headings/block anchors. Suggestion selection fills the draft, which previews before save. No nested modal, invented target on a miss, or cross-plugin dependency. |
| List         | Structured chip editor with a separate item input. Enter on a nonempty item adds it to the draft; Enter on an empty item saves the list. Empty-input Backspace removes the last chip; expose chip edit/remove controls. Editing/replacing a chip preserves order and literal commas. Submit a JSON array, never a comma-joined string.                                        |

For list fields, a choice/suggestion pick edits the list draft instead of emitting a
scalar field.set. On Tab, include a nonempty pending list item before preview and save
so no typed input is lost. Scalar kind rules still govern each item.

All stages expand beneath their field. Use labelled controls, proper combobox and
listbox relationships, visible focus, and text nodes for report/user strings. Provide
Save, Save and next/previous, Cancel, and picker/list controls in the card's action
surface. Update contextual hints and remove the typed-stage YAML fallback for supported
kinds. Keep YAML available through the existing action.

## 3. Wire preview, save, and fill-out through the existing session

One changed field produces one operation. Explicit empty scalar/list optionals produce
`field.unset`; a deliberately cleared required field refuses with the existing teaching
pattern and never applies. Defaults never refill cleared values. Use local checks only
for presentation/empty-state affordances; bob remains the validation authority for
values and guarded writes.

For typed edits, call `session.setDraft(ops)` on actual draft changes, retain the 250ms
debounce, and display the current preview's consequence and warning messages. Paint
refusal message/hint verbatim. Do not use `applyOps` to bypass the preview for typed
drafts. Existing scalar suggestion picks join choices/today/clear/Undo on the discrete
direct-apply route.

Enter remains inert while its current typed preview is pending or refused; a successful
current preview enables exactly one `session.apply()`. Guard rapid presses and repeated
keydown events. Do not let an old preview or Enter from an earlier stage save newly
typed text.

Tab/Shift+Tab consumes the key in every stage, including choices and comboboxes:

1. If unchanged, open the adjacent field's stage with no preview or write. Use the
   existing schema-order wrapping navigation for Tab; Shift+J/K alone clamps.
2. If changed, remember one advance direction with the current stage/draft identity.
   Wait for its current successful preview, apply once, then advance only on success
   (including a bob-reported no-op). Choices and explicit existing scalar suggestion
   picks may use their direct route.
3. Refusal or stale ends the pending advance and keeps the current stage open. Editing
   again, canceling, changing field, closing, or starting composition invalidates the
   pending action. Repeated Tab does not skip fields or queue saves.
4. A new stage starts from the applied/read state, not metadataCache's older value.
   Preserve the existing optimistic overlay and per-record Undo machinery.

Factor common apply settlement out of `recordApplyOps` as needed so typed apply and
direct apply share saving/saved/refusal/stale handling, report consumption, Undo, and
overlay updates. Preserve the current fingerprint and pane-save guard. Any necessary
change to the shared guarded session must be small and covered by album/bank
regressions; prefer Record Card/session-wrapper hooks.

## 4. Make refresh, retries, and input focus reliable

Retain the complete stage draft, chip state, selected candidate, active control,
caret/selection, and field identity across a record/album refresh and stale refusal.
Invalidate queued saves before rebasing. Re-read/rebase to a current fingerprint, show
`This record changed elsewhere — refreshed; your draft is kept · ↵ retry`, and allow an
explicit retry through a fresh preview. Both preview-time and apply-time stale refusals
need this path: the wrapper currently settles stale apply results but does not settle
stale preview results automatically.

Rebase alone is insufficient: deliberately schedule a new preview of retained ops when
retrying. A refreshed schema that removes or changes the field must stop the old stage's
write intent and teach the user; never apply its old draft to another field. Capability
loss similarly disables pending writes while keeping the safe read-only/YAML route.

Avoid rebuilding the active input during ordinary status-only paints. Preserve focus and
selection when a larger repaint is necessary, and never replace an IME composition node.
Stage keys run before browse keys: `j/k/t/u/y/=/digits` type in text controls; textarea
arrows keep their editing behavior; picker arrows navigate only that picker. Respect
`isComposing`, keyCode 229, and composition lifecycle. Esc and Ctrl+[ cancel through the
shared ladder; q does so outside text controls. Read
`decisions:q-and-ctrl-bracket-mirror-esc` through `/sase_memory_read` before changing
that dispatch. Cancel/close clears timers, preview drafts, queued Tab intent, and
listeners. Late reads/previews cannot trigger a write or reopen a canceled stage. An
already-started apply may settle; ignore its late UI callbacks.

## 5. Tests and docs

Add `scripts/test-ledger-tools-record-card-stages.cjs` and register it explicitly.
Reuse/extract the existing fake DOM/modal harness rather than bypassing event wiring.
Drive fake timers and controllable preview/apply promises; assert the actual envelope
operations and request counts. Cover:

- All stage kinds, invalid stored raw values, zero, schema defaults, optional clear,
  required refusal, date parsed echo, bounds, URL/newline refusal.
- Full ranked suggestions including beyond the first five, direct exact picks, free-text
  preview and near-duplicate warning; link bare names, aliases/anchors, ambiguous
  same-stem paths, and unresolved input.
- List add/edit/remove, literal comma items, list-of-choice precedence, Tab with an
  unfinished item, empty-input Enter/Backspace, and empty optional/required lists.
- Mixed-kind forward/backward Tab chains, wrap, unchanged navigation with zero writes,
  one changed-field write, refusal stopping advance, and success/no-op.
- Enter before preview, out-of-order previews, edit-during-preview, repeated Enter/Tab,
  cancel/close during preview, stale during preview/apply, rebase/retry, and a
  schema/capability change invalidating old intent.
- Caret and focus on asynchronous paints, no input replacement during composition, no
  browse letters in controls, close-key parity, and teardown.
- Co-landing digits, clamped Shift+J/K with repeat/reduced-motion behavior, and `=` on
  Status opening the Schema Card on Status; Shift+J then reaches the next schema field;
  an unknown field falls back to Album details. Assert CSS isolation.

Keep existing record-sheet/update/card, album entrypoints/card/field-jump/YAML, bank
session, and close-key tests green. Extend the existing real-bob scratch-vault
record-writer tests only where needed to prove typed-list/invalid-value behavior; do not
turn the tests into live-vault mutations.

After reading bob-cli's `docs.md` memory, complete **bob-plugins** `docs/record-card.md`
with stages, save/advance, refusals/retry, and key updates. Include exactly the approved
distinction: "Digits here pick a choice; on the album Schema Card they jump to field N."
Do not prematurely describe phase 7 as shipped. Use that source page, not the bob-cli
mounted copy. Keep capped front-door pages short with links and never raise a word
budget. Bump Ledger Tools to the next free minor version and regenerate build and
documentation inventory.

## 6. Verify, deploy, and close the assigned phase

Run `npm run build`, the focused tests, `npm run docs:gen`, `npm test`,
`npm run validate`, and `BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check` from
bob-plugins. Run bob-cli's `just check` through `/sase_monitor` as required by the epic.
Use `/sase_monitor` before long checks and finish each staged handoff through
`/sase_final`; never leave a running subprocess at turn end.

Use the repository's `docs/develop/validation.md` and deployment smoke workflow. Run an
isolated Obsidian stage walkthrough where prerequisites exist, covering a typed field,
suggestions, a list, mixed Tab navigation, stale draft/focus, and narrow light/dark
rendering. The parent land agent still owns the full integrated phase-7 CDP proof.
Distinguish offline self-tests from real Obsidian evidence and record a concrete
prerequisite limitation when a real run is unavailable.

Immediately before **each** sync fetch and integrate `origin/master`, preserve both
epics' package/test edits, rebuild generated main.js, and reverify affected checks.
Deploy the final integrated tree with `bob plugins sync --repo "$PWD" --no-pull`. Never
hand-merge main.js or edit the vault plugin sources directly. Follow AGENTS.md and
`docs/develop/deployment.md` for a refusal. The phase note's scoped force authorization
applies only to the three named plugins and only after all its proofs hold: the stamped
source exists and is clean, its HEAD is an ancestor of origin/master, its author's phase
is closed, and the forced intermediate deploy is clean current origin/master. Then
restore local changes, rebuild and sync without force; note old stamp and backup
directory. This is not general permission to inspect sibling checkouts outside
`/sase_repo` or bypass an unlanded active deploy. Complete the deployment check and
record its exact verdict.

Do not assume ms.5's recorded docs budget failure or timing flakes still exist. If a
gate fails identically on the clean current base, record the reproduction and any
existing tracking bead in a `PROPOSED FOLLOW-UP:` note on bob-cli-ms.6; that baseline
failure alone does not prevent closure. Fix new failures.

Record the implemented behavior, checks, live version, deployment result, and remaining
parent integration proof on bob-cli-ms.6. Re-run `sase bead epic-symbols bob-cli-ms.6`;
resolve each entry or re-key it to an appropriate still-open bead. Then run
`sase bead close bob-cli-ms.6 --note "<implemented scope and verified evidence>"`. Leave
all ancestors and sibling phases alone. End through `/sase_final`, committing every
changed repository via the host declaration.
