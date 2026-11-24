---
tier: epic
title: Area and project settings in the Property Editor
goal:
  Complete bob-cli-pl.6 with contract-driven settings, safe set and clear actions,
  project task handoffs, and verified entry points in the existing Property Editor.
phases:
  - id: preview-contract
    title: Return the resulting settings contract from property previews and writes
    size: small
    depends_on: []
    description:
      "preview-contract: expose the candidate contract in successful update JSON,
      document the additive field, and test clear/apply/undo sources without changing
      validation or writes."
  - id: settings-rows
    title: Extend the Property Editor with contract groups and stable virtual rows
    size: medium
    depends_on:
      - preview-contract
    description:
      "settings-rows: add a per-kind row provider, render stable settings and project
      virtual rows, preserve selection and clipboard behavior, gate unsupported actions,
      and implement capability fallback."
  - id: setting-actions
    title:
      Implement simple setting stages and clear-to-inherit through the shared session
    size: medium
    depends_on:
      - settings-rows
    description:
      "setting-actions: add the five simple controls, temporary generic stages, typed
      set/clear operations, effects and warnings, and fresh contract reconciliation
      after apply and undo."
  - id: settings-entrypoints
    title: Route area and project settings entry points and project task handoffs
    size: medium
    depends_on:
      - setting-actions
    description:
      "settings-entrypoints: register area/project in the existing registry, route card
      actions by setting id, implement project-task navigation and handoffs, and
      document the API and bare-plus keys."
  - id: settings-acceptance
    title: Verify settings behavior and visuals in real Obsidian
    size: medium
    depends_on:
      - settings-entrypoints
    description:
      "settings-acceptance: extend the scratch-vault harness, exercise settings and
      generic parity with current bob, inspect screenshots, fix phase-specific
      regressions, and record deployment and verification evidence."
proposed_by: bbugyi200.apollo.bob-cli-pl.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pl.6
create_time: 2026-11-24 15:49:19
status: wip
---

- **PROMPT:**
  [prompts/202611/area_project_settings.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/area_project_settings.md)
- **PARENT:**
  [202611/area_project_cards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/area_project_cards.md)

# Area and project settings in the existing Property Editor

Implement the complete `editor-settings` scope of **bob-cli-pl.6**, a phase of
`plan:202611/area_project_cards.md`. This is a child epic of that assigned phase. The
host's approval flow creates and links its beads; do not manually create them. The child
epic's land agent completes and closes **bob-cli-pl.6** after its own phases and this
child epic are complete. Child phase workers close only their assigned phase. Nobody in
this child epic closes **bob-cli-pl**.

The result is one Property Editor: area/project settings appear before ordinary
properties, missing settings explain their effective values and sources, setting
gestures use Bob's guarded writer, and `==` or Settings lands on the settings group. The
existing bare `+` opener remains the general property entry point.

## Evidence and scope

The planning read of bob-cli-pl.6 found it in progress with no previous work notes or
child plans. Its prerequisites bob-cli-pl.3 and bob-cli-pl.4 are closed.
`sase bead read bob-cli-pf` confirmed the foundation epic closed as done, including the
user-directed bare `-`/`+` rebind. Its note #3 records that this supersedes the older
`=-`/`=+` wording in the area/project epic. Use bare `+` in code, tests, and user
documentation; `==` is unchanged. Do not restore the retired chords.

Inspected bob-plugins through `/sase_repo`, at `6150d7e6`: local home-card model, DOM,
host, and foot are present in `249b` through `249e`, landed by `5717d518` and
`b5f2f6c1`, with subsequent acceptance commits. No contract settings provider, typed
settings stages, or area/project registry entries are implemented there.
`homeCardOpenSetting` currently forwards a raw key to the generic browse modal.
Inspected bob-cli at `2b3817fe`: typed setting operations and `props_contract_v1`
already exist. Do not redo either prerequisite.

The source APIs that this plan builds on are:

- bob-cli `src/native/props/{cli,show,plan,protocol}.rs` and
  `contract/{read,settings,effects}.rs`; `tests/cli/props_contract.rs` and
  `docs/props.md` define the wire contract.
- bob-plugins `plugins/bob-ledger-tools/src/298*`: protocol decoding, guarded
  session/history, row model, modal, typed stages, entry points, write safety,
  selection/filter, rename/reorder, clipboard, kind change, and YAML return. Later
  fragments wrap earlier modal methods; integrate with the final dispatch chain rather
  than patching a method that a later mixin replaces.
- `248m-plugin-schema-double-equals.js`: `registerPropertyCard` and the gated `==`
  registry; `297d` demonstrates an extension and teardown.
- `249d-plugin-home-card-host.js`: `homeCardOpenSetting` and the card/chip callback; the
  foot kit routes actual property keys through the existing editor hook.
- bob-navigation-hotkeys exposes `openBulletPropertyPicker(editor, options)`; inspect
  its current contract before using it for a `^prj` handoff. Do not invent an
  `api.taskCard.open` method.
- bob-plugins `docs/property-editor.md`, `docs/develop/architecture.md`,
  `scripts/obsidian-property-editor-e2e*.cjs`, and the home-card acceptance harness
  provide the current interaction and verification conventions.

One necessary refinement crosses into bob-cli: `PlanOutcome.report` already holds the
candidate's settings contract, but `emit_update` drops it. Existing `effects` give
values and scope, not the candidate inherited source label. The current modal merges
update properties into an old show report, so retaining that old `contract` would also
leave ghost/declared state stale. Return the existing candidate contract additively; do
not recreate inheritance in JS or change the settings schema version.

Five bounded phases keep this below a single large rewrite. Their dependency chain is
deliberately serial because they extend the same modal methods and fragment order. Each
implementation phase delivers its own meaningful regression coverage; acceptance owns
the real application run and visual evidence.

## Fixed design and boundaries

Honor the enclosing epic's final decisions: `card_structure_counts = no`; memory
authorization is limited to `glossary:area-card`, `glossary:project-card`, and
`glossary:property-card`. This child epic changes **no memory**; those edits belong to
the enclosing takeover phase. Do not re-ask these choices.

This work does not implement the swatch/type/clock policy controls of bob-cli-pl.7, the
restricted parent picker and catalog refresh of bob-cli-pl.8, or the native Properties
takeover and bank-card mounting of bob-cli-pl.9. Supply a documented stage dispatch seam
for those successors. Until they land, reuse generic value stages for those settings,
with Bob performing validation. Preserve the bank hybrid: `+` can show AREA SETTINGS,
`bank` still hands off to its bank editor, and `==` on bank.md keeps opening that
editor. The bank card's title/foot are unchanged.

All settings writes use the existing property update session. No second modal, history
stack, JS YAML writer, `processFrontMatter`, invented effective-value resolver, or
frontmatter task-schedule writer. Keep native Properties visibility as landed. Painting
and navigation cause no mutations.

## Shared row and action contract

Introduce a small internal extension seam, with explicit settings/virtual/source entry
types, used by rendering, visible-row navigation, filtering, selection, actions, and
initial focus. Preserve the original source inventory for generic operations and source
positions. Do not put synthetic rows into `report.properties`. Use stable setting IDs
(independent of source lines) and separate IDs for virtual project rows. Preserve focus
when a ghost becomes declared or a clear removes keys. The displayed property count
remains the actual source property count.

On a valid current `props_contract_v1` report, groups are:

1. AREA SETTINGS or PROJECT SETTINGS in **report contract order**. Area/bank: Summary,
   Lives under, Hue, Default type, Clock, Ready cap, Review, Capture names. Project:
   Lives under, Hue, Default type, Clock, Stats root, Ready cap, Review, Capture names.
   Each shows label, YAML keys, effective/declared value, and `set` only if declared.
   Merge aliases (`theme`) and the two clock keys into their single setting row; those
   source entries do not appear again as loose rows.
2. ON THE `^prj` LINE on projects: Goal, Starts, Depends on, from `contract.prj`. These
   are navigation-only virtual rows. An absent or archived task yields an honest
   unavailable state, never a fabricated line or new task.
3. PROPERTIES: off-contract loose properties in source order with the full generic
   editor behavior. Preserve created, aliases, tags, cssclasses, priority, unknown
   fields, null/false/empty distinctions, and duplicate-key recovery.
4. Identity / lifecycle: locked `type`, and project `status` (derived from the
   checkbox). The latter navigates to `^prj` on Enter. Preserve Bob's reasons and
   ownership gates on every mutation route, not just Enter.
5. A collapsed Bob-managed group with its count: `done_tasks`, loose `template`, legacy
   project `scheduled`, retired `task_count`/`open_task_count`, and other derived
   fields. Expanding retains source rows, writer/reason labels, and YAML access.
   Template remains editable; retired counts remain deletable and explain that nothing
   maintains them. `done_tasks` offers its archive link. Bank's card row retains the
   existing card-key handoff group.

Ghosts are faint and italic but legible; use Bob's effective label and help. Missing
required `parent` is a warning, not an inherited value. Ready cap and Review say
`Use Bob setting (...)`, never parent inheritance. `task_type: off` says
`Stop parent defaults`; grey is an explicit mute. Include issues and warnings without
treating warnings as refusals.

Settings and virtual rows cannot be renamed, reordered, or kind-converted. Enforce that
in keyboard, menu, and direct action paths with explanatory text. Generic loose rows
retain those actions. Selecting or filtering synthetic rows must not turn them into raw
YAML keys or make an absent value writable accidentally.

Copy declared settings as their **actual source keys and bytes** (both keys of a
shadowed clock, or legacy theme where present), using the existing clipboard machinery.
Never serialize a ghost's effective default as a declaration. A ghost copy is a no-op
with a teaching message; virtual task rows are not properties. Mixed selections must not
silently drop ineligible rows. Preserve existing paste validation and atomicity: pasted
mappings still preview through Bob. A successful paste/add/rename onto a contract key
must refresh the settings rows as well.

For batch deletion, declared settings map to distinct `setting.clear` operations,
ordinary loose properties to `prop.unset`, ghosts contribute no operation, and
protected/required/virtual selections refuse as a whole with a reason. One accepted
batch is one history item; a ghost alone says `Already using the inherited value.`
Required parent clear remains refused by Bob and is explained in the UI.

## Phase preview-contract

1. Add optional `contract` to successful `bob props update -d -f json` and apply
   responses from `view.report.contract`, including semantic no-ops and restore updates.
   Omit it for non-contract notes. Reuse the computed value: no new reads, locks,
   mutations, or resolver logic. The same field reports candidate data in preview and
   confirmed data in apply. Keep old fields and schema version intact.
2. Extend CLI tests to verify declared-to-ghost after clearing color and residence
   settings, exact inherited/config source labels, apply parity, and inverse-op
   restoration. Include bank hybrid and ordinary-note omission. Prove dry-run leaves
   disk bytes unchanged and existing inverse operations restore them.
3. Document the additive field in `docs/props.md` and regenerate/check docs using
   repository instructions. No CLI option or version changes are needed.

Validation: focused native props and CLI props tests, then required repository checks
(`just check` and docs checks as applicable). A change to response JSON is additive, so
no breaking-contract marker or release/version edit is warranted.

## Phase settings-rows

1. Add contract shape checking/capability helpers without rejecting otherwise usable
   older `props_v1` reports. Decode and retain `contract`, `effects`, and `warnings`
   additively. Missing/invalid extension data must never authorize a typed operation.
   Ordinary-note behavior remains unchanged.
2. Implement the extension seam and all groups above through the existing modal. Include
   area/project species badges in the editor heading using the shared kit. Route
   rendering, selected/marked rows, filter matching, source-line lookup, and
   reconciliation through the same entry model. Match settings by labels, keys, values,
   and effective labels. Preserve focus and marks predictably through filter/fold
   changes and report refreshes; hidden folded rows are not actionable.
3. Add read-only/unsupported-action policies, managed folding, source-safe copy, and
   paste integration. Settings activation calls a stage resolver seam; until the next
   phase supplies actions, explain unavailable actions rather than issuing generic
   writes against synthetic IDs.
4. Capability fallback: without `props_contract_v1`, render the landed generic
   properties and `Update bob for typed settings`. Without `props_v1`, retain read-only
   cache rows and YAML recovery. Malformed YAML, duplicates, missing fingerprints, and
   unresolved ownership retain their existing safety behavior.

Add focused model/DOM and action tests from real-shaped contract fixtures for area,
project, bank, legacy theme, shadowed clock, missing parent, malformed contract, old
Bob, and damaged frontmatter. Cover stable focus, actual property counts, clipboard
source bytes, filtering/folding, and generic row parity. Register new tests in
`package.json` and new fragments in source order.

## Phase setting-actions

Build settings actions on the existing stage renderer, List controls, and guarded
session; new fragments stay below the repository's 1000-line limit. The main modal
already has 933 lines, so do not inflate it past that limit. Document the stage
resolver/options so later policy and parent phases replace only their own handlers.

- **Summary:** one-line text, live `N/140`; Bob refuses multiline and warns over 140
  without blocking a valid save.
- **Ready cap:** count stepper, 1–999 or `off`; `o` toggles off in the control's
  non-text key context. Legacy false initializes as off and new writes use `off`. A live
  effect line uses the existing local noteReady count (`6 ready · cap 8 → room for 2`);
  unavailable counts remain unavailable. Never store a count.
- **Review:** days control, 1–365, with `Applies to tasks living in this note only`.
- **Capture names:** reuse the ordered List stage, preserve order, show Bob's
  duplicate/blank refusals and cross-note collision warnings inline.
- **Stats root:** Enter performs one undoable setting set to true or clear to absence,
  never root:false. Respect disabled on top-level projects. Explain that past and future
  time regroup here and leaf-project successor offers change.
- **Temporary stages:** parent uses the existing link-capable generic stage; color and
  task type use generic text, and clock reuses the appropriate generic value controls.
  Translate a logical clock value into the strict mode object when using
  `setting.set clock`; never submit a scalar there. When editing an actual clock key
  through generic controls, use its real `prop.*` op and Bob's validation. No bespoke
  policy picker, parent ranking, or catalog sync yet.

Enter on a ghost opens an override draft seeded from reported effective data where
meaningful; opening never writes. Dedicated settings save through `setting.set`. Ctrl+D
clears all keys for a logical setting with `setting.clear`. The guarded dry-run must
succeed before apply. Render its candidate contract's effective label and
`effects[].message` so a clear explains what will apply and where it comes from; render
warnings inline. Preserve the existing one-gesture clear/toggle grammar rather than
adding a second confirmation modal.

Consume a confirmed post-apply contract after set, clear, generic property edits, paste,
undo, and redo. Never merge new properties with an old contract and call it current. On
a supported older response without the additive field, invalidate the contract and
re-read `props show`; show pending until it is refreshed. A candidate preview is not
confirmed state and cannot leak into a card or cache overlay.

Retain revision/fingerprint matching, save-every-pane-before-apply, stale draft
retention, close-during-write guards, unknown-outcome behavior, and inverse-op history.
All close keys use `bobCardWireCloseKeys`: Esc, q outside text controls, and Ctrl+[
everywhere back out one layer without committing.

Tests exercise setting op shapes, boundary values and refusals, preview warning display,
clear ghost/no-op, mixed batch refusal, exact inverse undo, repeated Enter, stale
previews, fresh contract reconciliation, no writes on open/paint/cancel, and ordinary
Text/List/YAML stages. Cover legacy clock/theme source preservation and ensure
loose-property paste still reaches Bob's validation.

## Phase settings-entrypoints

1. Extend `openPropertyEditor` with an optional setting id/first-settings focus request.
   Keep existing path/key/selection/mode callers compatible. Requests survive the
   initial async report and work for ghosts. Map keys such as `theme`, `offclock`, and
   `offclock_types` onto their logical setting. On degraded reports, select the
   corresponding real property when present, otherwise a safe first row.
2. Register area and project through the existing `registerPropertyCard` lifecycle,
   after bank/album/record and the temporal extension, before later extensions. Use
   current classifiers and exclusion rules. Keep the existing `==` Vim gate, bank
   precedence, and teardown. `==` and Settings select the first settings row; a policy
   chip selects its setting. Opening never toggles a checkbox. Browse `=` inside an
   area/project editor focuses settings without recursively reopening.
3. Project virtual rows hand off to the actual current `^prj`: navigate to the correct
   note and validate its source line before using the existing Task Card method; fall
   back to the cursor when unavailable. `contract.prj.line` is line **text**, not a
   numeric index. Resolve it against current source or the existing recognized-task
   helper. Never land on a different task after a source change. Project status Enter
   performs navigation only. Missing/archived `^prj` reports absence without creating
   anything. Preserve the originating pane where possible.
4. Update `docs/property-editor.md` with groups, ghosts, simple stages, managed and task
   routes, capability fallback, and the current bare-plus/== keys. Update the relevant
   hotkey reference and document the narrow extension seam in developer docs so
   bob-cli-pl.7/.8 can extend it without another modal fork.

Tests cover registry precedence, repeated setup/unload, first-settings/ghost/chip focus,
key alias mapping, bank `==`, ordinary/album/record/day behavior, missing nav plugin,
stale task lines, terminal projects, and preserved pending-write guards. No navigation
route writes task fields or frontmatter dates.

## Phase settings-acceptance

Extend the existing Property Editor e2e using a separate settings scenario module and
fixtures if needed; do not grow its already-large main script into a second monolith.
Keep an offline self-test registered in `npm test`. Use an isolated scratch vault and
`BOB_BIN` plus `BOB_CLI_DIR` identifying a current bob-cli build that includes the
preview-contract change. Do not test mutations in Bryan's vault.

Required real-Obsidian evidence:

- Bare `+`, `==`, Settings, and each chip focus the correct area/project setting; bank
  `==` still opens bank, and bank `+` shows area settings.
- Ghost → ready cap override → clear → undo/redo preserves focus, source labels,
  one-operation history, and exact original bytes including unrelated keys/body.
- Invalid values stay refused with Bob's message; long summary and alias collisions warn
  without blocking. Review remains by residence. Root toggles write true or absence and
  are disabled on top-level projects.
- Goal/Starts/Depends on reach the correct Task Card or cursor fallback; status only
  navigates. No new frontmatter scheduled/due/start/deadline appears.
- Managed fold, legacy-count deletion, generic created/aliases/tags editing,
  selection/filter, declared-setting copy, paste validation, and YAML return work.
- Degrade independently for a Bob with props_v1 but no contract and for one with no
  props_v1. Repeated render/resize/open/cancel writes nothing. Stale preview and
  close-key behavior stay safe with text controls and IME composition.
- Inspect light/dark × narrow/wide screenshots of populated settings and simple stages:
  selected/ghost/locked rows legible, effects visible, no clipping, focus rings intact.
  Use existing styles, forced-colors and reduced-motion conventions.

Run the harness and inspect its report and screenshots; offline self-tests alone do not
satisfy the application acceptance. Follow existing deployment/validation docs to use a
machine with desktop Obsidian if this host lacks it. Read tailnet and Obsidian memory
first if that path requires remote host or vault work. Use `/sase_monitor` for long
commands and carry the exact acceptance result forward. Record commands, source SHAs,
binary provenance, checks, screenshots, and any limitations on the phase; make evidence
durable using `/sase_artifact` where needed.

Finish user documentation to match tested behavior and update the validation guide.
Correct regressions in this epic's scope before considering acceptance complete.

## Repository checks, deployment, and completion

Every worker first reads its assigned bead and the enclosing final decisions, checks
what has landed, confirms bob-cli-pf closed, and opens bob-plugins with `/sase_repo`
before reads or edits there. Read its AGENTS.md. Never mention a numbered workspace path
in follow-up plans. Read reference memory through `/sase_memory_read`; read sidecar
artifacts only through `sase artifact read`.

For each plugin-changing phase: edit source fragments, build generated main.js with
`npm run build`, run focused tests plus `npm test`, `npm run build:check`, and required
validation/docs checks, then run `bob plugins sync`. Follow the repo's overwrite refusal
recovery procedure without force. No manual commits: finish via `/sase_final`. Long
checks use `/sase_monitor`, never an unowned background process.

Before closing any assigned child phase, run `sase bead epic-symbols` for that bead and
resolve or re-key remaining suppressions to a still-open owner. Record discovered
out-of-scope work as `PROPOSED FOLLOW-UP:` notes on that assigned phase; do not create
task beads. Identical failures proven on the clean base are documented follow-ups, not a
reason to strand the phase. Record how the base reproduction was obtained.

The child land agent reviews the combined implementation against every requirement
above, verifies all child phases and remaining symbols are settled, records the actual
API seam for bob-cli-pl.7/.8, and closes the child epic through its normal landing
workflow. Then run `sase bead epic-symbols bob-cli-pl.6` and close only
`bob-cli-pl.6 --note "<verified behavior and evidence>"` when its scope is complete. Do
not close bob-cli-pl or any other enclosing plan. This plan proposal itself is
intermediate work and must leave bob-cli-pl.6 in progress.
