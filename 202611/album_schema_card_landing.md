---
tier: tale
title: Finish the Schema Card and land bob-cli-kr.4.1
goal:
  Repair remaining editor defects, integrate current album reports, prove real Obsidian
  behavior, and close the child epic and its parent phase.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-kr.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kr.4.1
create_time: 2026-11-12 18:07:40
status: wip
---

- **PARENT:**
  [202611/album_schema_card_editor.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_schema_card_editor.md)
- **BEAD:**
  [bob-cli-kr.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kr/bob-cli-kr.4.1.md)

# Finish the Schema Card and land bob-cli-kr.4.1

Complete the remaining work of **bob-cli-kr.4.1**, then close that epic and its parent
phase **bob-cli-kr.4** in this same implementation turn. This tale has no land agent and
no later automatic resumption of the interrupted landing. Its last step owns the
concrete closeout below. Do not wait for this work's commit SHA, push, or CI result
before that closeout; code and close declarations can land together through
`/sase_final`.

This is one medium implementation: bounded repairs to an existing card/session and
acceptance harness, one additive presentation integration, and verification. Do not
rebuild the Rust writer or introduce another schema store. No new CLI options, schema
grammar, migrations, or memory changes.

## Governing context and completed audit

Read through the audited commands:

- `sase bead read bob-cli-kr.4.1 -r "Need landing audit and follow-up outcomes"`
- `sase bead read bob-cli-kr.4 -r "Need parent scope and final decisions"`
- `sase artifact read plan:202611/album_schema_card_editor.md "Need the remaining editor acceptance contract"`
- `sase artifact read plan:202611/album_schema_render_and_edit.md "Need inherited decisions and parent phase contract"`

The final decisions remain: one confirmed gesture per update envelope/scoped commit;
mobile cached read-only summary/chips with a YAML hint and no write affordances;
Ctrl+Shift+P falls back to the card only on an album when there is no Task Card target;
**no memory file changes**. Preserve the actual task, Task Link, dependency, relation,
counted-command, and active-picker priorities.

The lander reviewed every note on all four closed children, their plugin commits
`da5cdda`, `a57994a`, `6050124`, `a572162`, and the actual transport, session, card,
entrypoint, YAML-excursion, writer, and acceptance sources. The backend already owns
schema validation, counts, keys, diffs, refusals, and inverse metadata. The focused
editor suites passed 58/58 before the latest header integration. The lander has triaged
every phase-proposed follow-up; the epic notes contain all outcomes. Do not create
duplicates or defer these epic defects into tasks.

Audited primary base: `b7f74b58`. Audited plugin base advanced from `a572162` to
`3ef045b`, then `b6f3dd9` during landing. The latter two commits fix the three stale
walkMode assertions, the task-tag onload test, navigation reference budget, false
ancestry warnings, and duplicate headers. The combined task-tag, bracket-repeat,
period-link, and album-header suites pass **98/98** on `b6f3dd9`. Preserve those changes
and recheck newer upstream drift before editing.

Open bob-plugins using `/sase_repo`; use only the returned path and read its
`AGENTS.md`. Paths below are relative to that repository unless marked bob-cli. Edit
source fragments, regenerate bundles with `npm run build`, and keep authored fragments
at or below 1000 lines. Register added test files in `package.json`.

## 1. Repair gesture dispatch and keep the typed draft through repaint

In `241y-plugin-album-schema-card.js` and `242c-plugin-album-schema-edit.js`:

- Put modifier-aware reorder handling before plain j/k navigation. On the audited tree,
  `schemaOnKeyDown({key:'j',altKey:true,...})` changes selection from `field:one` to
  `field:two` and leaves `draftOps: []`; Alt+K has the same precedence bug. Exercise
  actual key events for fields and choices, not just direct `schemaMoveSelected` calls.
  Plain navigation must remain read-only.
- Store the stage's typed values, property-presence toggles, selected input, and
  caret/selection independently of its DOM. `paintSchemaCard` currently rebuilds inputs
  from the original report. A probe changing a label to `New draft`, then calling
  `schemaAlbumState`, observes the label revert to `One`. Report refresh, pending/result
  paint, and navigation must not erase an unfinished gesture or steal typing focus. Keep
  explicit stale handling rather than silently rebasing.
- Wire input/change events to the existing roughly 250 ms preview debounce. Currently
  the card registers only click/keydown/focusin; typing then waiting 350 ms produces
  zero previews. The session debounces only after the user explicitly submits the stage.
  Meet the approved contract: a settled draft previews automatically, every edit
  immediately invalidates Apply, and Enter applies only a successful current preview
  whose consequence was shown. Keep the stage usable while previews arrive and preserve
  false/zero/empty/list/unset distinctions. Composition and input keystrokes must not
  trigger hotkeys or an accidental mutation. One event has one dispatch path.
- Fix removal of initial choices in a new choice field. The current code removes a row
  then renumbers the old static NodeList (including the removed element), and leaves
  remove-button indices stale. Rebuild surviving row identities and actions from the
  live list, preserving values. Test removing the first and middle of three choices,
  editing survivors, then removing another survivor; the preview must receive exactly
  the intended ordered labels and emoji.

Add meaningful event-level regressions using a DOM fixture that understands the
selectors being exercised. Tests must cover unsolicited refresh while typing, debounce
coalescing, pending/refused/stale preview gating, and no double apply. Keep edits inert
during an in-flight save/apply; a newly typed draft must not allow an obsolete captured
envelope to apply after the save await.

## 2. Retain Undo's original guard and the pane identity of YAML Done

In `242b-plugin-album-update-session.js`, use the saved Undo `afterFingerprint` with its
opaque `inverseOps`. It is stored but currently ignored by `stageUndo`: after applying
fp1 -> fp2, `rebase('external-fp3')`, `stageUndo()`, and preview, the request
incorrectly expects external-fp3. It must keep fp2 so an external edit is refused by bob
rather than silently accepted as the inverse's baseline. The card's explicit
stale/rebase route must not silently rebase an inverse either. Test this with the
service/session and actual writer; no-op applies must still retain the prior Undo, and
post-sync recovery stays `bob vault-sync` only.

In `242e-plugin-album-yaml-excursion.js`, capture the excursion identity and recheck it,
the leaf's current file, and teardown state after every awaited save, report, and view
restoration. Currently Done retains a local record across awaits and can restore/reopen
an old album after navigation cleared that excursion. Concurrent Done clicks need one
completion guard. A stale or unavailable report cannot authorize return. Keep schema
errors in Source at their diagnostic line, allow warning-only schemas, and test
navigation/leaf close/unload during a delayed save or report. No vault mutation is added
here.

## 3. Complete the album-only chord and singular-label integration

In Navigation Hotkeys `551-plugin-album-schema-fallback.js` and the existing
property-command route, remove the unapproved bullet-only restriction where the Task
Card has no target: the album fallback should work on a blank line, title, or other idle
album row too. The shared resolver can return `record-parent`; require a direct album
result for this chord so a record note does not acquire the album-only overload. Palette
and Command Mode should continue to resolve record notes to their owning album. Test the
real command routing, including recognized targets whose own operation refuses, and
missing plugin/mobile. Do not add a competing key registration.

Integrate bob-cli `b04120b0` and plugin `4d987b7`: `bob albums list` now supplies the
display `singular`, and the owner catalog already consumes it. Carry it through
`241u-plugin-album-header-model.js` and its signature and consume it in
`241w-plugin-album-schema-model.js` for the header and Schema Card stats. The audited
card still reports `1 tv_show` for `{noun:'tv_show', singular:'TV show'}`. Reuse the
catalog's additive old-bob fallback, without deriving another display noun in
JavaScript. Verify 0/1/many counts and old JSON without `singular`.

The outer epic `bob-cli-kr` already knows about this integration in note #2; record its
completion there so its lander does not duplicate it. The warning and duplicate-header
repairs in outer note #3 landed in `b6f3dd9`; preserve them. Other outer header
diagnostics and the broader Properties serializer proof stay with that outer lander, not
this bounded child landing.

## 4. Make acceptance exercise the real editor and run it

Repair and extend `scripts/obsidian-album-ui-e2e.cjs` and its existing helper,
writer-matrix, and offline-wrapper files. The added Schema Card walk has never run
successfully and contains concrete invalid assumptions:

- It reuses `first` after ArrowDown repaints/removes that element, then dispatches Enter
  on the detached node. Reacquire the focused connected control after repaint, or use
  CDP keyboard dispatch against the actual focused element.
- Clicking an `add-choice` target only selects it in the implementation, yet the harness
  expects a form immediately. Exercise the actual documented selection and edit gesture.
  Cover Command Mode through its surface execution path and Task Card priority through
  the registered command, not a direct fallback helper call alone.
- Its supposed two-pane delayed-save race uses `vault.modify`, opens no second Source
  pane, then expects exactly one writer attempt even though generation invalidation can
  remove Apply before any attempt. Create the real second pane with an unsaved schema
  edit and a controlled supported-save delay. Assert a stale refusal or invalidated
  preview, unchanged gesture bytes/commit count, retained unconfirmed draft, and no
  blind retry. Cover backend stale refusal separately where the write actually reaches
  bob.
- The `staleSession` ledgerEval body omits `return`, so its evidence is undefined.
  Correct it and make every required verdict assert observed state rather than a
  constant true or merely that a helper exists. Verify current choice keys, record
  paths, and titles against the actual scratch fixture.
- The standalone writer matrix silently exits zero when bob is missing and its wrapper
  accepts that as success. For required acceptance, pass an explicit compatible **built
  current bob** via `BOB_BIN`, record its provenance, and distinguish unrun/skipped from
  pass. Existing unrelated `bob-cli-l8` tracks stale-binary autodetection; do not create
  another task or broaden this tale into that generic discovery repair.

Reuse the Git-backed scratch vault and test remote; never test edits in the personal
vault. Prove current dry-run consequence/derived keys, one commit per apply, previewed
guarded Undo, no-op/cancel/refusal byte stability, label/key and record-byte
preservation, used-removal full counts, typed defaults/unset, double-Enter,
local-success/sync-only retry, album/record entry, and YAML invalid/repair/Done. Retain
the inherited mobile/Properties cleanup fixtures.

Apollo has no `obsidian` or `xvfb-run` and currently has Node 24.15.0. Do not repeat the
previous phases' conclusion that this makes proof optional. Athena has a working
isolated Obsidian route demonstrated by the later header rollout: read
`file:explicit:1e158a38d48049a8df1636ba` through `sase artifact read` for its hashes and
proven host route. Read tailnet reference memory before remote access, and open any
repository through `/sase_repo` before using it. Test an exact snapshot of this turn's
working files on the capable host in a scratch location; do not require a commit/push
first. Use the established Node/app/Xvfb prerequisites and bounded CDP harness, using
`BOB_BIN=<verified current binary> xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs -e /snap/bin/obsidian -r /tmp/album-schema-landing.json -o /tmp/album-schema-landing-shots`
on the capable host. A genuinely unavailable host remains an explicit blocker, never a
pass.

Capture and inspect light/dark screenshots at approximately 375 px for choice and typed
forms, consequences, refusals, wrapping, labels, and focus after repaint. Preserve
structured reports, selected binary/source hashes, commands, and inspected screenshots
through the artifact workflow and link them in the epic note. The earlier 45-check
header walkthrough is useful baseline evidence, not proof of the later Schema Card
editor walk.

## 5. Verify and deploy the completed tree

Read `docs.md` through `/sase_memory_read` before any documentation edit. Bring
`docs/albums.md` and affected key references into agreement with the fixed behavior;
keep the current navigation page under its word budget. Run the focused regressions,
`npm run build`, `npm run build:check`, `npm run docs:gen` when inventory changes,
`npm test`, `npm run validate`, and
`BOB_CLI_DIR=<resolved primary checkout> npm run docs:check`. Preserve only intended
generated changes. Classify every failure individually with clean-base evidence; the
earlier phase totals do not enumerate every failing node and are not a waiver. Existing
unrelated tasks and disposition are in the landing note.

Run primary bob-cli **`just check`**, using `/sase_monitor` for a long run (or the
named-tool escalation/join workflow). Do not run `just check-full`. The prior phase
never completed this gate. Fix failures caused by this epic; route genuinely unrelated
failures through `/sase_new_task` with exact evidence. Do not convert a passing check
plus a different infrastructure failure into additional feature scope.

Run the real isolated startup smoke, then the guarded plugin deployment from the opened
checkout with `bob plugins sync --repo "$PWD" --no-pull`. Follow
`docs/develop/deployment.md` for fetch/ancestry recovery of conflicting deploys; never
force another checkout's work away. The audit's latest inventory showed 11 synced, 0
drift after `b6f3dd9`, superseding phase .3's partial deployment. Verify this tale's
final managed bytes with the mandatory post-sync real smoke and before/after hash
checks, using the established cross-host snapshot route when necessary. A plain
`plugins list` zero exit is insufficient.

Long commands must finish or be handed to `/sase_monitor`, with the continuation owning
all unfinished steps including the closeout. Wait for staging commands themselves to
exit; never end a turn with an ordinary subprocess still running.

## 6. Close bob-cli-kr.4.1 and only its parent phase

This is the final implementation step, not a later landing request. Re-read the epic,
all four child notes, and `bob-cli-kr.4` and inspect any post-audit drift. All required
implementation and acceptance must now be complete. Incorporate the lander's recorded
follow-up outcomes into the close note and add exact new
verification/deployment/artifact evidence. Confirm inherited decisions.

1. Run `sase bead epic-symbols bob-cli-kr.4.1` and
   `sase bead epic-symbols bob-cli-kr.4`. Both were empty during audit, but recheck.
   Resolve every listed symbol by wiring, privatizing, a justified non-test pragma, or
   deletion under the Symvision policy; re-key only when a still-open later bead
   actually needs the exemption. Do not leave stale entries.
2. Close normally with
   `sase bead close bob-cli-kr.4.1 --note "<implementation, child-note, integration, follow-up, test, real-Obsidian, and deployment verification>"`.
   If closure reports unfinished descendants or symbols, finish them and retry. Never
   use `--force` to make success go through or advance nested landing.
3. Run `just symvision` if the recipe is available (the primary checkout had no such
   recipe at audit time); record availability and outcome. Open the plans repository
   through `/sase_repo`, resolve the epic's PLAN
   `plan:202611/album_schema_card_editor.md`, and set **`status: done`** in that file's
   frontmatter. Do not mark it done before successful epic closure.
4. Recheck that this completed child implements the parent phase's entire `card-editor`
   contract, then close only
   `sase bead close bob-cli-kr.4 --note "<child editor completion and acceptance evidence>"`.
   Leave containing epic **bob-cli-kr** open for its waiting land agent. Do not mark
   `plan:202611/album_schema_render_and_edit.md` done and do not close that containing
   epic. If the parent phase has gained unresolved or ambiguous requirements, record the
   concrete blocker there and report it instead.
5. Finish through `/sase_final`, declaring every edited repository, including the linked
   plugin and plans repositories. This same-turn finalization commits the final code and
   closeout; no dependency on its future SHA or CI.
