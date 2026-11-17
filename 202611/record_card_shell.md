---
tier: tale
title: Record Card shell, choices, and discrete actions
goal:
  Complete bob-cli-ms.5 with a guarded keyboard-first record modal, choice edits,
  discrete actions, Undo, and YAML return.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ms.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-ms.5
create_time: 2026-11-17 16:33:11
status: wip
---

- **PARENT:**
  [202611/record_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card.md)
- **BEAD:**
  [bob-cli-ms.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.5.md)

# Record Card shell, choices, and discrete actions

## Objective and ownership

Complete assigned phase **bob-cli-ms.5**, `record-card`, of **bob-cli-ms**. A desktop
user can open **Edit record**, browse all schema fields, pick a choice, set a date to
today, clear an optional field, undo a save, visit YAML and return, or climb to the
album Schema Card. Every value write uses the already-landed guarded record session.

This is one bounded implementation agent's work: the sheet, service, protocol, and
session already exist. The phase had no earlier work/remainder notes when inspected. Do
not reimplement those dependencies. Typed editors and their save-and-advance chain
belong to **bob-cli-ms.6**; sheet edit affordances, `==`, Ctrl+Shift+P, Command Mode,
and switching on Properties/inline-title hiding belong to the following entry phase.
Leave useful stage extension points for ms.6, but do not ship a second writer or a
generic typed editor as a shortcut.

The implementing agent completes and closes **only bob-cli-ms.5** after verification. Do
not close bob-cli-ms or any ancestor. Never manually set the assigned phase status.
Record discoveries with `sase bead note bob-cli-ms.5 'PROPOSED FOLLOW-UP: ...'`; do not
create task beads.

## Authority, decisions, and verified starting point

Read the phase through `sase bead read bob-cli-ms.5 -r "Need current scope and notes"`.
Read these artifacts through `sase artifact read`, not direct sidecar file reads:

- `plan:202611/record_card.md`: settled design, shared Record Card contract, and the
  `record-card` phase are authoritative.
- `research:202611/record_card/record_card.md`: normative design where the parent plan
  does not override it.
- `plan:202611/double_equals_schema_keymap.md`: shipped entry behavior, changed only by
  the later entry phase.

Honor the final epic decisions: `inline_title = hide` is implemented by the later entry
phase; `decision_record = no` prohibits the proposed decision note. **No memory file may
be edited.** Read `docs.md` and `decisions:q-and-ctrl-bracket-mirror-esc` through
`/sase_memory_read` before working in their domains. All three close keys must share one
Esc dispatch.

Open bob-plugins using `/sase_repo`, use only its returned checkout, and read its
`AGENTS.md`. Refresh history and inspect dirty state before edits. Inspected versions:

- bob-cli `6c409ab2`, containing `99005975` per-record show and `48a3116d` guarded
  records-update envelope.
- bob-plugins `f49e175`, Ledger Tools **1.130.0**, containing the read-only sheet and
  completed ms.4 session. No `openRecordCard` or Record Card implementation exists.
- ms.4 is closed. Its completion notes document session tests and the real-bob writer
  matrix, plus independent suite/deployment limitations listed below.
- `sase bead epic-symbols bob-cli-ms.5` currently reports no entries. Recheck before
  closure rather than relying on this snapshot.

Relevant existing code, relative to bob-plugins:

| Code                                                                                                | Reuse                                                                                                    |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `plugins/bob-ledger-tools/src/242i-plugin-record-sheet-model.js`                                    | Canonical gate, field/value models, lint, labels and formatting.                                         |
| `242j-plugin-record-sheet-dom.js`, `242k-plugin-record-sheet.js` in the same directory              | Safe text/link/chip renderers; `recordSheetGateFor`, `recordSheetModelFor`, YAML helpers and overlay.    |
| `242l-plugin-record-update-service.js`                                                              | `apiRecordShow`, `apiRecordUpdatePreview/Apply`, capability check, Undo store and own-write suppression. |
| `242m-plugin-record-update-session.js`                                                              | `createRecordUpdateSession`, `recordUpdateServiceDeps`, `applyOps`, `undo`, rebase and close.            |
| `241y-plugin-album-schema-card-dom.js`, `241y-plugin-album-schema-card.js`, `241yc/241yd` fragments | `albumSchemaCardModalBase`, card chrome, stable DOM regions, modal scope and lifecycle patterns.         |
| `241xc-plugin-card-close-keys.js`                                                                   | `bobCardWireCloseKeys`, `bobCardTypesText`, matching unwire.                                             |
| `242d-plugin-album-entrypoints.js`, `242e-plugin-album-yaml-excursion.js`                           | Parent Schema entry and shared guarded YAML excursion.                                                   |
| `248l-plugin-bank-setup-card-entrypoints.js`                                                        | Existing alternate-kind YAML return implementation.                                                      |

The **landed envelope uses `field`, not `key`**:
`{op: "field.set", field: "status", value: "ready"}` and
`{op: "field.unset", field: "url"}`. Follow the parent plan and bob-cli
`src/native/records/protocol.rs`, rather than the older research example.
`apiRecordShow` returns a transport result wrapping `report`; check both the transport
and report outcome. Its report contains schema `fields` (choice counts and full
suggestions), and `record.properties`, `findings`, lines and fingerprint.

## Implementation

### 1. Pure model and stage boundary

Add `242n-plugin-record-card-model.js` with pure helpers for targets, opening selection,
navigation, choices filtering, contextual hints, All actions, status text and the Esc
ladder. Field identity is the schema key, never a painted array index.

Build one selectable target for **every** declared field in declaration order, including
a separate ghost target for each empty optional. Identity and other-property rows are
read-only. Prefer the explicitly requested field when valid, then the first missing
required field, then the first field. Empty schemas are safe and leave YAML, Schema and
close available. Preserve the selected key on repaint; if a schema edit removes it,
clamp its former index to the remaining targets.

Keep card state explicit: selected field, stage kind/field, choice filter and selected
choice, menu state, session snapshot, latest per-record report, read sequence, pending
apply, and closed/composing flags. Supply a single stage-open dispatch and shared
stage/draft preservation boundary that ms.6 can extend. In this phase choices are the
implemented editor. Other field kinds still render and support applicable discrete
actions and YAML; their Enter path should explain that YAML is the available edit route
until their typed stage exists. Do not make a no-op look like a save.

### 2. Modal shell, entry point, and read lifecycle

Add `242o-plugin-record-card-dom.js` and `242p-plugin-record-card.js`; split additional
cohesive fragments if needed to keep every edited fragment at or below 1000 lines.
Extend `albumSchemaCardModalBase()` and reuse `bob-key-card` chrome and record-sheet
value renderers. Do not copy the album card's schema editing stack.

Provide `openRecordCard({filePath, field, stage})`, resolving an omitted path from the
active Markdown file. Reuse the strict record-sheet gate, not `albumSchemaEntryKindOf`'s
loose fallback. Preserve teaching Notices for desktop, missing report/album, bad parent
and capability failures. The palette command on a non-record says
`Open a record note to edit its fields.` Healthy canonical records without
`record_values_v1` may open read-only with `Update bob to edit here`; they cannot create
a writable session. Broken schemas open a read-only warning card with the album
diagnostic and album-YAML route. Other ineligible notes never get a guessed record
target.

Paint cached sheet rows immediately, with title `<album emoji> <record title>`, subtitle
`<Album> › <Noun> · <path>`, and status `Reading record…`. For an eligible capable
record, make one `apiRecordShow(type, stem)` request on opening; create the session only
after a valid current fingerprint arrives. Old-bob and broken-schema paths must remain
usable read-only even if show cannot run or refuses. Read failures paint the decoder's
message/hint and never impersonate an empty writable record.

Initialize `createRecordUpdateSession` with record/album paths, type, stem, fingerprint
and `recordUpdateLastUndo(recordPath)`, using `recordUpdateServiceDeps`. Consume its
onChange snapshot and existing onApplied overlay/Undo behavior. Do not duplicate pane
saves, echo suppression, invalidation or Undo persistence in the modal.

Subscribe to relevant record/album vault modifications and album report changes.
Coalesce duplicate events, re-read the record, and rebase against the latest read while
keeping selection, open stage/filter/draft, focus and caret. Mark outstanding reads with
a monotonic request/close token so out-of-order responses cannot replace newer data.
Recheck the gate when type, parent, capability or schema changes. Close/unload disposes
listeners, subscriptions, pending latches and the session; late callbacks must never
repaint, reopen, or enable writes. Own-write events must not overwrite the successful
apply response with stale metadata or trigger duplicate writes.

### 3. Browse keys, choices and discrete writes

Use one action dispatcher for keyboard, pointer controls and All actions. Outside text
controls implement:

| Gesture                        | Behavior                                                                                        |
| ------------------------------ | ----------------------------------------------------------------------------------------------- |
| `j`/`k`, arrows, Tab/Shift+Tab | Next/previous field, including ghost fields; follow the existing card's wrap convention.        |
| Home/End                       | First/last field.                                                                               |
| Enter                          | Open the selected field's inline stage.                                                         |
| `1`–`9` on choices             | Apply the corresponding schema-order choice directly.                                           |
| `t` on date                    | Directly set today using bob's accepted date input, avoiding UTC/local-day conversion mistakes. |
| Ctrl+D                         | Unset an optional field; required fields teach why clearing is refused.                         |
| `u` / Ctrl+Z                   | Apply the last per-record inverse operation through `session.undo()`.                           |
| `y` / Ctrl+E                   | Close and take the guarded record-YAML excursion.                                               |
| `=`                            | Close and call `openAlbumSchemaEntry({filePath: recordPath})`.                                  |

Use exact modifier matching so unrelated OS/editor chords stay untouched. IME
composition (`isComposing`, keyCode 229 and local composition state) never dispatches a
write. In text inputs, `j`, `k`, digits, `t`, `u`, `y`, `=` and `q` type normally;
Ctrl+Z remains native text undo there. Esc and Ctrl+[ still back out.

The choices stage is inline under its field. Show a filter input and list with emoji
labels, muted album usage counts, current-value ring/text and numeric badges. Filter by
label/value without changing the stored choice identity. Keep schema-order digit
assignments stable when filtering and show only badges that still map to their row. Use
j/k outside the filter input and arrows/Enter for list selection; digits typed in the
filter remain text. All choices, including positions beyond nine, are reachable by
navigation and click. Empty results and the current choice are explicit states. Never
invent a choice or a new schema key from filter text.

Each choice pick, browse digit, today and optional clear calls `session.applyOps`
exactly once with one field operation and the current fingerprint. Undo calls the
existing session method. Discrete actions do not trigger a dry-run preview. Required
clear refuses without dispatch, e.g. `Status is required — pick one of 6 choices`; bob
remains the final validation authority for every submitted operation.

Latch in-flight gestures and repeated Enter so a double press/keydown+click sequence
cannot save twice or re-open a stage on the same press. Stay open after success, return
to browse on the same field, consume the applied properties immediately, and show
`✓ Saved · u undo`. No-op results must not manufacture a save or replace Undo. On stale,
refresh/rebase and retain the intended choice/draft for **explicit** retry; do not
auto-replay. Keep bob's refusal message and hint verbatim. Represent stale as
`This record changed elsewhere — refreshed; your draft is kept · ↵ retry`.

### 4. Close ladder, actions, feedback and accessibility

Use `bobCardWireCloseKeys` and override `onEscapeKey`, routing Esc, Ctrl+[ and eligible
q through exactly one handler: close All actions first, then cancel the stage, then
close the card. Respect in-flight apply safety and prevent a key from also reaching the
editor behind the modal. q is never a hint badge or another action. Unwire on close.
Exercise both Obsidian modal-scope and DOM routes so the default modal Esc cannot bypass
stage cancellation.

The status region supports reading, save progress, current session consequence, saved,
typed transport/validation refusal and stale retry. Build contextual hints and the
`⋯ All actions` buttons from the same action model; every keyboard action has a
pointer-accessible equivalent, appropriate disabled reasons and visible focus. Add a
polite live region announcing field selection and save/refusal outcomes without
announcing every repaint. Preserve focus during updates. Render untrusted labels, values
and diagnostics as text or through the existing safe renderers. Add only scoped Record
Card CSS, with stacked labels/usable actions at 375px, light/dark support, visible
current/selected states and reduced-motion behavior.

### 5. Record YAML excursion and plugin wiring

Extend the shared `242e` excursion's kind handling to retain `record` rather than
coercing it to `album`. Add the record Done dispatch alongside the bank branch; follow
the bank excursion's guarded save, view-state restore and completion-promise patterns.
Keep record path, parent album path and selected field distinct.

For `y`/Ctrl+E, close the card first, capture the original leaf/view state, open Source
on the per-record property's **1-based record line**, and mount Done. An absent key uses
the report's frontmatter boundary/fallback, never the album schema field line. For a
broken schema, expose the parent album's diagnostic/YAML route explicitly. Done must
save before restoring, reject save failures, ignore a replaced/closed leaf, restore the
previous view, then reopen Record Card on the same field. Record findings are displayed
by the reopened card and **never block Done**. Do not route record Done through the
album `albumYamlDoneDecision` schema-errors guard. Preserve all album and bank excursion
behavior.

Register a Record Card mixin/lifecycle alongside the sheet and writer in
`310-install-methods.js`, `170-plugin-lifecycle.js` and its teardown file. Add the
**Edit record** palette command (`edit-record`). Register source fragments in
`src/fragments.json` in dependency order and expose the minimal helpers/modal hooks
needed by tests via a final exports fragment. Re-read and increment Ledger Tools'
current minor version. Build the generated `main.js`; never edit it by hand.

Update the plugin-owned `docs/record-card.md` to describe the implemented card, palette
entry, browse/choices key table, read-only states, Undo, and YAML return. Keep clear
which typed stages and entry gestures are still pending in this staged rollout. Refresh
generated inventories; do not hand-edit the mounted copy in bob-cli. No Rust API or
release-version change is expected.

## Verification and delivery

Add `scripts/test-ledger-tools-record-card.cjs` to the explicit `package.json`
`test:unit` list. Follow existing Obsidian/DOM stubs and fake timers. Use behavior
assertions, including emitted envelope calls and focus/lifecycle effects:

- Opening selection precedence, schema order including ghosts, empty schema, selection
  retention/removal, and browse key/modifier matrix.
- Immediate cached rows; exactly one initial read; writes disabled before its valid
  fingerprint; transport/report failure, missing capability, broken schema, strict
  canonical gate, non-record Notice and mobile refusal.
- Choices filtering/counts/ring, stable badges, more than nine choices, empty results,
  click/Enter/digit applying one operation with no preview; today and optional clear;
  required-clear refusal; saved/no-op/stale/refusal feedback.
- Double Enter and repeated events, action parity, in-flight guards, IME, input typing,
  native input Undo, and no navigation-letter interception.
- Undo using the stored inverse and fingerprint, including close/reopen persistence.
- Modify refresh with retained selection/filter/draft, out-of-order reads, own-write
  refresh, changed schema/capability, close/unload while reading/applying, and teardown.
- Menu → stage → card for Esc/q/Ctrl+[ through scope and DOM routes, including focus
  outside the modal and q inside controls. Assert the editor behind receives no action.
- `=` closes before parent-schema dispatch; YAML jumps to the record line; Done saves,
  restores and reopens the same field despite record findings; save failure, duplicate
  Done and leaf changes fail safely. Existing album/bank YAML tests stay green.
- Safe DOM text, keyboard focus, labelled actions and polite live announcements.

Run focused card tests and the existing record-sheet, record-update, album/bank session,
card-close and YAML excursion suites. Run the existing record writer self-test with a
verified current `BOB_BIN`; distinguish an offline fixture check or skipped binary from
a real writer pass. Extend that matrix only if a new integration concern needs coverage;
do not rewrite it.

From bob-plugins run `npm run build`, `npm run docs:gen`, `npm test`,
`npm run validate`, and `BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check`. Run
`just docs-gen` for any generated primary documentation changes and the parent- required
`just check` through `/sase_monitor`. Long suites likewise use the monitor skill; wait
for the staging command itself to exit and finish the turn through `/sase_final`, never
leave an ordinary running subprocess behind.

Run `bob plugins sync --repo "$PWD" --no-pull` from the opened plugin checkout after
changes. Follow its deployment guide's isolated smoke/deployment checks, recording exact
outcomes and prerequisites; do not claim a real-Obsidian pass from fake DOM tests. The
epic land agent owns the complete seven-phase CDP walkthrough. If visual checks can run
now, inspect the card at narrow/wide sizes in light/dark and record screenshots;
otherwise make the missing visual evidence explicit for that land agent.

ms.4 reported these limitations; they are leads to verify, **not assumed waivers**:

- Mounted docs word-budget failure: `docs/reference/json-outputs.md`, 6089/6000, also
  cited by bob-cli-mr.3.1.
- Timing failures in event-link-times (bob-cli-km), frees-marks (bob-cli-io),
  keep-moving-marks-surface (bob-cli-ki), off-clock-link-chips (bob-cli-ju).
- Album writer self-test: held-lock apply counted two commits instead of one.
- A protected, abandoned dirty Ledger Tools 1.129.0 deployment, source `b0a9c89`,
  previously prevented ms.4's 1.130.0 sync. Recheck whether its work has since landed;
  follow `AGENTS.md` and `docs/develop/deployment.md`, never force on assumption. If
  still abandoned and unlanded, the required approval must name the deployed version and
  features being replaced, with the implementation fully prepared first.

For a check failure, fix regressions from this work. If it reproduces identically on the
clean starting tree, record exact evidence and any existing tracking bead as a
`PROPOSED FOLLOW-UP:` on ms.5; such failures alone do not keep the phase open.

Before closure, inspect the final diffs, ensure generated outputs are current, and run
`sase bead epic-symbols bob-cli-ms.5`. Resolve leftovers or re-key them to a still-open
appropriate bead. Record implementation, exact checks, deployment outcome and the
stage-extension interface on ms.5. Close only with
`sase bead close bob-cli-ms.5 --note "<what was implemented and verified>"` once this
phase is complete. End through `/sase_final`, declaring every changed checkout and never
committing manually. Preserve outstanding deployment/visual limitations honestly in the
phase evidence and final response.
