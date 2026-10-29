---
tier: epic
title: Task Card theme editing and successor retargeting
goal:
  Task Cards explain and edit task themes, and deliberate successor retargets remember
  the correct task theme without changing ordinary placement or freshness behavior.
phases:
  - id: card-theme-model
    title: Model and display Task Card themes
    size: medium
    depends_on: []
    description:
      "card-theme-model: extract crowded Task Card helpers, add the theme projection and
      header/property row, and test suggestion, inheritance, repair, and task scope
      handling."
  - id: card-theme-edit
    title: Edit task themes through the Task Card
    size: medium
    depends_on:
      - card-theme-model
    description:
      "card-theme-edit: add the t stage and Ctrl+D use-default with API-backed field
      setters and guarded local, counted, and linked task edits that preserve status,
      links, and freshness."
  - id: successor-theme-context
    title: Carry each successor's context and retarget intent
    size: medium
    depends_on:
      - card-theme-edit
    description:
      "successor-theme-context: carry task identity and full note context into each
      successor retarget picker, preserve validated themeIntent through both
      normalizers, and retain cancellation and older-plugin behavior."
  - id: successor-theme-write
    title: Remember guarded successor retarget choices
    size: medium
    depends_on:
      - successor-theme-context
    description:
      "successor-theme-write: apply eligible retarget theme changes inside the successor
      task edit, preserve default-placement and close/move/roll behavior, and document
      and verify the complete contract."
proposed_by: bbugyi200.apollo.bob-cli-dm.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-dm.7
create_time: 2026-10-28 22:50:52
status: wip
---

- **PROMPT:**
  [prompts/202610/task_card_successor_themes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_card_successor_themes.md)
- **PARENT:**
  [202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_suggestions.md)

# Task Card theme editing and successor retargeting

## Assignment and boundaries

Implement the unfinished `card-successor` phase, bead `bob-cli-dm.7`, of
`plan:202610/task_theme_suggestions.md`. This is a child epic of that assigned phase.
Its phases each fit one coding agent; the child epic's land agent verifies the combined
result and closes `bob-cli-dm.7` after its own phases are complete. Ordinary child phase
workers close only their own assigned bead. No worker or land agent in this child epic
may close `bob-cli-dm` or any other ancestor plan bead. Do not hand-set bead status or
reopen an in-progress phase.

All implementation belongs in the linked `bob-plugins` source repository. Use
`/sase_repo` and `sase repo open bob-plugins -r "<specific reason>"`, then read its
`AGENTS.md`; use only the returned checkout. Never edit deployed plugin source in the
vault. Paths below are relative to bob-plugins unless explicitly marked bob-cli. Read
the governing plan with `sase artifact read`, and read reference memory through
`/sase_memory_read`. Consult `docs.md` before changing plugin documentation.

The parent epic's reviewer decisions remain final: `ledger_theme_pills=yes`,
`seed_plex_chore=yes`, `memory_decision_record=no`, and `memory_glossary_term=no`. The
ledger gate already shipped; seeding belongs to dm.9. **No memory changes** are
authorized here. No CLI, release version, CHANGELOG, Mac client, vault defaults, or
unrelated task behavior changes are in scope. Do not create task beads. Record
discovered work on the assigned child phase, and relay it to dm.7 as
`PROPOSED FOLLOW-UP: <summary — evidence>` for the outer epic's land agent.

## Evidence and why an epic

The audited read of dm.7 showed `in_progress`, no previous work notes, and no
`--epic-symbol` entries. The prerequisite dm.6 is closed; its linked child epic landed.
The plugin tree inspected at `a9066b8` is clean and includes `25d864e` (theme API),
`97341c4` (preselection), `ea583df` (intent), and `0f74920` (writes). Do not redo those
changes; recheck HEAD and the phase notes at implementation time.

The current gaps are concrete:

- `310-task-card-previews.js` has no theme row/key and is 985 lines;
  `350-picker-task-card.js` has no theme stage/write dispatch and is 990 lines.
- NAV `165-pomodoro-theme-picker.js` already provides catalog access, ranking,
  components, hue, and subscription lifecycle. Reuse it without importing its Pomodoro
  creation or Recent-hide gestures into task editing.
- Ledger Tools `279-plugin-task-theme-api.js` already exports `suggestFor`,
  `taskThemeOf`, `setTaskTheme`, `clearTaskTheme`, and `planThemeChange`. The setters
  are pure string transforms; the caller owns guarded persistence.
- `suggestFor` takes a vault-relative path, **one-based line**, and **full note text**,
  not just a task line. Its optional text must be the matching note.
- `planThemeChange` requires a shown suggestion. Task Card explicit editing must bypass
  that one restriction without weakening ordinary linking.
- NAV `206-successor-picker-modal.js` calls `api.pick()` without a task, applies one
  returned destination to all marked rows, and discards `themeIntent`.
- Cycler `085b-successor-offers.js` builds display-only rows and independently strips
  intent in `normalizeSuccessorPickDestination`.
- Cycler `089b-successor-verdicts.js` owns the picked task's consolidated line edit;
  `137-plugin-successor-project.js` supplies snapshots, rereads after picking, replans,
  and applies note writes before daily insertions. These are required integration points
  in addition to the older epic's named 085/087 files.
- `085-successor-plan.js` is 997 lines, `087-successor-place.js` 922, and NAV 206
  is 918. Extract cohesive helpers before adding behavior. Keep every handwritten
  fragment at or below 1,000 lines and register it in `src/fragments.json`.

The authoritative bob-cli documentation is now **PT13** in `docs/pomodoro-themes.md`;
PT12 is the configured vocabulary section. Use PT13 links despite the older parent
plan's PT12 wording. The research artifact cited by the parent plan returned `missing`
during planning; this plan relies on the full approved epic and landed code/contracts,
not on unavailable research content.

Planning-time baseline: `node --test --test-reporter=dot` passed for
`scripts/test-navigation-task-card-model.cjs`,
`scripts/test-navigation-successor-picker.cjs`, and
`scripts/test-task-status-cycler-successor-offers.cjs` on that unchanged plugin tree.
Full build, documentation, deployment, and implementation regressions remain the coding
phases' verification obligations.

Four phases separate projection, modal writes, cross-plugin transport, and successor
transaction integration. The explicit serial graph avoids concurrent changes to
navigation's fragment manifest, exports, generated entrypoint, and documentation
inventory. No phase requires exploratory design of another phase's contract.

## Shared behavior and compatibility

Consume the existing suggestion result: task override, reading task, daily review,
nearest eligible note default, otherwise none. For source `task`, inherited is its
`inherited` member; for other non-null suggestions, the suggestion itself is the
inherited result. A null result is legitimate. Inspect `taskThemeOf` separately so
invalid/duplicate fields can still show `Needs repair` when no default exists.

Use catalog spelling and normalized keys, existing hue APIs, and PT6 pill styling. Task
fields never add catalog/history entries or change theme color. Unknown themes are
visibly marked and never automatically selected; hidden themes remain valid suggestions.
Do not infer fields from Dataview's inherited `task.theme`.

The existing link write rule stays intact: only deliberate choices with a shown
suggestion can write; the resolved destination must have exactly one non-exempt
component, `PLAN + BUILD` means BUILD, and unnamed/all-exempt/composite destinations do
not write. Selecting the inherited key clears; selecting another sets. Unchanged
semantic fields retain their exact bytes. `linkOnce=true` always wins over deliberate.

Only the explicit Task Card editor permits editing without a shown suggestion. Implement
this as a small local planner over the existing parser/setters rather than fabricating a
shown suggestion or globally relaxing the shared link rule. Selecting **Use default**
always clears all theme fields, including an invalid or duplicate field and a default of
none; selecting the inherited theme also clears. Other theme choices follow the parent
write rule, including single-component validity and exempt handling. The default action
must remain usable for an exempt inherited theme.

Feature-detect every new optional API method. With missing/older Ledger Tools, do not
guess suggestions or write fields; retain the card's existing actions and show an
unavailable reason for theme editing. Older destination pickers still retarget normally
without field writes. Keep existing namespace versions and return shapes compatible by
adding optional properties only.

## Phase card-theme-model

Work primarily in NAV `310-task-card-previews.js`, `320-task-card-header.js`,
`330-task-card-view.js`, `350-picker-task-card.js`, and new focused fragments. Split
oversized model/key helpers and modal methods by responsibility, preserving load order
and mixin installation (`460-install-picker-mixins.js`) where relevant. Do not hand-edit
`main.js`.

1. Add a pure theme projection plus a runtime adapter supplying the matching live note
   snapshot, theme API, and existing target identities. Resolve the actual task behind a
   Task Link; never use the daily link bullet as its task. Deduplicate counted/linked
   targets using the current Task Card session machinery.
2. Render the header line and a selectable Theme property row:
   - `Theme [FIX] this task · sase_blog suggests WRITE`;
   - `Theme [WRITE] from ● sase_blog`;
   - `Theme none · t to set`. Use the established reading/review/no-default phrases,
     declaring-note hue dot, unknown tooltip/dotted underline, and repair styling. Color
     must not be the only signal. Batch cards identify their scope and mixed values;
     never imply that a value or default applies to all targets when it differs.
3. Define the row's `propertyName: theme`, deletable state, shortcut label `t`, and
   immutable projection contract for the next phase. Keep write dispatch gated until a
   theme edit implementation is present; this phase must not advertise an active handler
   that does not exist.
4. Reuse the existing modal harness and task-card model/view suites. Cover explicit
   versus inherited, none, reading/review labels, unknown, duplicate/invalid with and
   without inheritance, missing API, linked target identity, mixed batches, and
   render-without-writes. Verify source extraction preserves existing key behavior.

Completion: the theme projection and view are tested, existing Task Card suites pass,
source builds and line caps pass. The next phase owns enabling the editor and user
gesture documentation. Follow the per-phase build and deployment rules below.

## Phase card-theme-edit

Add a focused theme-stage model/view/mixin alongside the Task Card fragments. Reuse NAV
165 for theme inventory, normalized components, catalog spelling, hue, Recent ranking,
and subscription cleanup. Flatten Today names into distinct valid component choices; a
composite Pomodoro name is not itself a task-theme value.

1. Enable `t` and Enter on the Theme row. The stage offers Use default first with its
   inherited reason, then Today, Recent, and a validated typed new name. Deduplicate
   rows by normalized key; preserve typed query/selection across asynchronous catalog
   refreshes. For mixed batches, Use default clears each target to its own inherited
   default and the UI says so. Unknown inherited names remain available for explicit
   selection without automatic preselection.
2. Add `t` to repeated-action suppression and wire Ctrl+D on the card's selected Theme
   row to Use default. This is field clearing, never Recent hide/unhide. Do not route it
   through generic property-config deletion: theme need not be in the configured
   property list. Back/Escape/Ctrl+[ and cancellation write nothing.
3. Plan per-target set/clear/no-op through the direct-edit rule above. Validate a typed
   name before commit, use catalog spelling when known, and apply only
   `api.themes.setTaskTheme`/`clearTaskTheme` to task lines. Preserve their exact field
   placement and no-op behavior; do not copy a field-rewriting regex.
4. Guard task identity and the exact shown line before writing; revalidate the inherited
   default used by a choice. A changed/missing task refuses rather than editing a
   neighbor or overwriting concurrent text. Use open editors first and the existing
   preimage-checked vault writer otherwise. Plan all batch lines before writing, retain
   existing local/count batch undo boundaries, and report cross-note partial failures
   honestly. No status change, Task Link change, ID mint, route operation,
   fresh/unblocked/keeps stamp, schedule/Work Log, or review-walk advance may be caused
   by a theme-only edit. Closed-task fields remain historical.
5. Tests exercise actual modal dispatch and persisted post-images: t, Enter, Ctrl+D,
   default including none/exempt, new theme with no prior suggestion, same-key
   byte-preserving no-op, field repair, invalid/composite input, cancellation, refresh
   stability, stale line, linked target, and mixed counted targets. Assert status,
   freshness, links, and unrelated bytes are unchanged.
6. Update `docs/task-card.md` gesture table/header/scope and relevant
   `docs/bob-navigation-hotkeys.md` behavior. Update inventory if facts change.

Completion: focused task-card model/view/schedule/review and new theme suites pass; the
full plugin checks and sync run under the rules below. Report any manual desktop
verification that was unavailable rather than claiming it.

## Phase successor-theme-context

Work in NAV 206 (extract a helper fragment as needed), cycler
`085b-successor-offers.js`, `137-plugin-successor-project.js`, and the corresponding
successor tests. The picker API from dm.6 already accepts a task and returns intent;
consume it, do not redesign it.

1. Extend selectable successor rows with optional validated task context sufficient for
   `api.pick({task: {path, line, text, displayText, status}})`. Build it from the actual
   successor and matching note snapshot; `line` is one-based and `text` is full note
   content. Keep source identity/preimage information for a safe refresh before opening
   the nested picker. Do not reconstruct identity from display text or parse the opaque
   offer key. Carry the new data through NAV's model normalizer.
2. A retarget is task-specific. For one highlighted row, pass that row's task. For
   multiple marked rows, run the task-aware pickers sequentially in displayed order, one
   per target, so each row's `shown` suggestion is actually shown for that task. Collect
   choices locally and publish overrides only after the entire retarget operation
   succeeds. A cancellation/read failure leaves the previous overrides, marks, and
   selection unchanged; reentry and modal-close guards still apply. Do not copy the
   first task's suggestion/intent to the other marked tasks. This retains marked-row
   retarget capability while making each decision truthful.
3. Preserve validated optional `themeIntent: {deliberate, linkOnce, shown}` through both
   NAV `normalizeSuccessorDestinationOverride` and cycler
   `normalizeSuccessorPickDestination`/`normalizeSuccessorPicks`, plus any intervening
   reconstruction. Match the existing block-id-prompt validator's semantics: malformed
   intent is dropped without rejecting a valid destination; linkOnce overrides
   deliberate. Preserve canonical-name intent such as `preserveSpelling` where used.
4. Keep no-task older row shapes and missing APIs compatible: ordinary retargeting still
   works but cannot acquire permission to edit a theme. The context transport and picker
   itself never mutate a task field or select successor offers.
5. Test actual request and return flow for existing/new destinations, two tasks with
   different defaults, a successor on a non-first note line, changed/missing source,
   intermediate cancel, modal close while awaiting, missing provider, old rows,
   malformed intent, Alt+Enter, and untouched preselection. Assert round-trip through
   both normalizers, not just one. Run the picker and cycler successor-offer suites.
6. Document additive row/intent transport in `docs/develop/plugin-apis.md` and the
   marked-row sequencing in `docs/bob-navigation-hotkeys.md`.

Completion: the intent reaches cycler with the correct task association and existing
retarget tests still pass. Theme writes remain for the final phase.

## Phase successor-theme-write

Use a small injectable pure planner for the theme decision, with a runtime adapter for
Ledger Tools and current snapshots. Integrate at `planSuccessors` and
`089b-successor-verdicts.js`, supplied by the replan path in 137; use 087's resolved
destination information rather than trusting display titles or typed slugs.

1. Apply only to an explicitly picked successor with a retarget override and valid
   deliberate intent. Default anchor/implicit placements, unpicked offers, no shown
   suggestion, link-once, and untouched preselection remain never-writers.
2. On the refreshed task/note snapshot, resolve the destination's actual open entry name
   (or validated new-entry name), rerun `suggestFor`, and compare semantic identity with
   `themeIntent.shown` including inherited theme, source, declaring note/chain, and
   repair state. Ignore decorative hue changes. A changed suggestion permits the
   existing link behavior but skips the field edit and reports why, following
   block-id-prompt's `078-pomodoro-theme-write.js` precedent.
3. Delegate the ordinary rule to `api.themes.planThemeChange`, then apply the API setter
   to the consolidated successor line. Put it after status/recovery changes and before
   block-ID/freshness suffix assembly where possible. Freshness eligibility and the
   stamp result must be the same as with no theme edit. Cover both ordinary and
   project-finished successors; do not accidentally alter recovery-only paths.
4. Destination, dependency, project, stale-task, and selection guards must succeed
   before a theme edit is eligible. The current runtime applies note edits before daily
   insertions, so preflight the chosen destination and consume the same resolved name in
   planning; do not persist a theme for an already-refused/moved/closed destination.
   Preserve existing editor/preimage protections and close-not-rolled- back semantics.
   If a later cross-note link write fails after the task write, report the saved theme
   and failed link accurately instead of claiming atomicity. Keep receipt/take-back
   checks consistent with the consolidated task post-image; authored theme state must
   not be silently stripped by reopen recovery.
5. Add integration tests through picker result -> normalize -> refreshed plan ->
   persisted edit, including set, clear, same-key no-op, repair, existing/new bucket,
   PLAN+BUILD, unnamed/all-exempt/composite, two independently retargeted successors, no
   suggestion, missing API, stale task/suggestion/destination, guard refusal, and
   task-save/daily-link failure. Assert no field edit for default placement or an offer
   merely appearing. Verify local daily-note successors too.
6. Add preservation regressions proving a distinctive `[theme:: FIX]` field (plus a
   paren form where supported) retains its exact bytes through Ctrl+Shift+M, Ctrl+Enter
   close, and roll/Alt+P paths. Other operation-specific bytes may change; compare field
   bytes and expected surrounding post-images, not the whole line as if a close did
   nothing. Ensure the tests exercise the real writers.
7. Update `docs/task-status-cycler.md` and final navigation/API docs: deliberate
   retarget can remember, ordinary successor placement never writes; retain the
   established guard/failure language. Register any new tests in `package.json`.

Completion: focused successor suites, new integration/preservation tests, full plugin
checks, and deployment pass or have accurately recorded clean-base failures.

## Verification, deployment, and land-agent completion

Every implementation phase must run `npm run build`, relevant `node --test` suites,
`npm test`, and `npm run validate` from its opened plugin checkout. Regenerate docs with
`npm run docs:gen` when needed and run `npm run docs:check` for changed docs. Build
output is committed source: generated `main.js` must match fragments. `npm test` already
checks build freshness and documentation inventory. If a command will outlive a provider
turn, use `/sase_monitor` before starting it and await the handoff command's exit; never
end a turn with an unmanaged command running.

The plugin AGENTS.md requires sync after repository changes. Deploy the changed plugins
from the checkout returned by `sase repo open` using
`bob plugins sync --repo <opened-repo> --no-pull -p <plugin-id>`; use separate
invocations for `bob-navigation-hotkeys` and `task-status-cycler` when both changed.
Record deploy results. Do not force past a provenance refusal; resolve it through the
normal finalization/deployment workflow and report the concrete blocker if necessary. Do
not deploy another checkout's code by relying on the default repository path.

The child epic land agent reviews all four phases together, confirms prerequisite
changes remain present, resolves integration conflicts, and runs the combined build,
test, manifest, and docs gates once on the final tree. Run the host `just check` if
available. Verify actual Obsidian interactions if a desktop is reachable: header and
pill states, t/Enter/default/Ctrl+D, cancellation, local undo, two distinct successor
retargets, Alt+Enter, and default-placement preservation. Otherwise explicitly record
manual UI checks as unverified. Deploy the final navigation/cycler builds.

A check that reproduces identically on the clean base is recorded with evidence as
`PROPOSED FOLLOW-UP:` (cite an existing tracking task if found) and does not leave a
phase open. Do not create follow-up beads in this child epic; the outer epic land agent
owns triage.

Before closing any assigned phase, run `sase bead epic-symbols <phase-id>`; resolve its
symbols or re-key each still-needed Justfile entry to an appropriate open bead. Before
the land agent closes dm.7, it must run `sase bead epic-symbols bob-cli-dm.7` again and
ensure no leftovers. Close the child phases and child epic only when their work is
verified, then close **only the delegated phase** with:

`sase bead close bob-cli-dm.7 --note "<implemented behavior, checks, deployment, and unverified UI details>"`

This land-only close of dm.7 is the explicit delegation requested by the user, not
permission for child phase workers to close their ancestors. Keep outer epic
`bob-cli-dm` open. Use `/sase_final` as the last action of every normal ending turn;
successful plan/monitor handoffs use their own lifecycle.
