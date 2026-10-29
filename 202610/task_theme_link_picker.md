---
tier: epic
title: Task-aware Link to today picker
goal: Complete bob-cli-dm.6 so the picker preselects and explains a suggested theme,
  remembers deliberate exceptions in guarded task edits, and supports link once.
phases:
- id: suggestion-model
  title: Task-aware picker inputs and shared theme preselection
  size: medium
  depends_on: []
  description: 'suggestion-model: add task context and the synchronous suggestion
    snapshot to both picker entry points, implement the shared preselection rule without
    changing Today order or typed-query ranking, and run the copied preselect vectors
    through the model.'
- id: picker-intent
  title: Stable selection, explanation, and link-once intent
  size: medium
  depends_on:
  - suggestion-model
  description: 'picker-intent: preserve selection by key across asynchronous refreshes,
    render suggestion reasons and the live field-change footer, add explicit Alt+Enter
    link once, and carry themeIntent through the v1 picker API with interaction tests.'
- id: guarded-theme-write
  title: Remember deliberate choices in the guarded task edit
  size: medium
  depends_on:
  - picker-intent
  description: 'guarded-theme-write: revalidate the shown suggestion, apply the shared
    field writer inside the existing task-line plan without changing freshness eligibility,
    report successful and partial outcomes accurately, and finish docs and runtime
    tests.'
proposed_by: bbugyi200.apollo.bob-cli-dm.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-dm.6
create_time: 2026-10-28 20:08:11
status: wip
bead_id: bob-cli-dm.6.1
---

- **PROMPT:** [prompts/202610/task_theme_link_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_theme_link_picker.md)
- **PARENT:** [202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_suggestions.md)
- **BEAD:** [bob-cli-dm.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.6.1.md)

# Task-aware Link to today picker

This is a child epic for **bob-cli-dm.6**. It completes the `link-picker` phase of
`plan:202610/task_theme_suggestions.md`. Its land agent verifies the three phases
together and closes bob-cli-dm.6 after this child epic's work has landed. Phase workers
close only their own assigned child phase. Nobody working this child epic closes the
outer epic bob-cli-dm or another outer phase.

## Outcome and scope

Ctrl+Shift+Enter opens a task-aware picker. Empty search selects the suggested Pomodoro
in place, or offers a new one when permitted. The menu explains the suggestion and
previews a deliberate task-theme change. Enter on the untouched default links without
storing a field; a deliberate exception is remembered in the same task-line edit.
Alt+Enter always links once without changing the field.

All product changes belong to the linked **bob-plugins** repository, primarily
`plugins/block-id-prompt/`, its tests, and its documentation. Open it with `/sase_repo`
and read its `AGENTS.md` before work. Edit source fragments and regenerate `main.js`;
never hand-edit generated code. Every fragment must stay at or below 1000 lines. New
helper fragments must be registered in `src/fragments.json`, and mixins/exports wired in
160/170 when needed.

Do not change the task-theme resolver or storage contract, the Task Card or successor
writers (outer phase dm.7), capture/Swift code, or vault defaults (outer phase dm.9).
There are no memory edits. The outer decisions are final: `ledger_theme_pills=yes` is
already landed; `seed_plex_chore=yes` belongs to dm.9; both memory decisions are `no`.

## Evidence and contracts

Read the parent with
`sase bead read bob-cli-dm.6 -r "Need phase scope and final decisions"` and the design
with
`sase artifact read plan:202610/task_theme_suggestions.md "Need link-picker contract"`.
The current implementation contract is **PT13** in bob-cli's `docs/pomodoro-themes.md`;
the parent plan's PT12 numbering predates a separate configured-vocabulary addition.
Preserve PT12 vocabulary behavior, configured spelling, warnings, and color rules while
adding PT13.

At planning time:

- bob-cli HEAD was `41cc4acc`; bob-plugins HEAD was `bc245fc`. Both working trees were
  clean. Dependencies dm.2, dm.3, and dm.5 were closed; dm.6 had no prior-work notes.
  Recheck current code before implementing.
- Ledger Tools already provides `api.themes.suggestFor`, `taskThemeOf`,
  `planThemeChange`, `setTaskTheme`, `clearTaskTheme`, and `isKnownTheme` under themes
  API v1. Root Ledger Tools API v3 has `caps()` and `planBudget()`.
- `suggestFor({path, line, text?})` takes a **one-based** line and full Markdown note
  text, not a single task line. Block ID Prompt source lines are zero-based. The
  suggestion returns directly or is null; it is not a response envelope.
- `choosePomodoroLinkTarget` currently passes only displayText/status;
  `pickPomodoroLinkDestination` ignores its request. The modal's `updateQuery` always
  resets selection, including catalog/history refreshes.
- `planTargetTaskUpdate` has no theme option. Its `lineRewritten` flag controls
  freshness and must not start counting theme-only changes.
- Fragments 075 and 120 have 986 and 980 lines. Use new small fragments for model and
  write helpers; leave these files as thin call sites.
- The seven focused suites listed below passed **178/178** before changes.
  `sase bead epic-symbols bob-cli-dm.6` found no entries; recheck at closure.

The task-theme API signature is:

```text
planThemeChange({field, suggestion, inherited, destinationName,
                 exemptKeys, deliberate})
  -> {action: "none"} | {action: "clear"} | {action: "set", name, key}
```

`taskThemeOf` returns none/set/repair states. The pure change planner returns a
canonical name; a caller must resolve catalog spelling before emitting a set. Both
setters return strings and do not write files. A suggestion from a task field carries
`inherited` (including null); other suggestions are themselves the inherited suggestion.
Serialized ThemeRefs omit `known` when true.

## Phase suggestion-model

Work in 075/076, 122, 126 and new small helper fragments as needed. Keep this phase
read-only with respect to task fields.

1. Pass complete task context through both picker entry points. The direct gesture uses
   `source.sourcePath`, `source.line + 1`, and the live editor's full `getValue()`.
   Public `api.pomodoroLinkPicker.pick(request)` accepts optional
   `task: {path, line, text, displayText, status}` with a documented one-based line,
   matching `themes.suggestFor`. Calls without task retain the generic successor header
   and old behavior. Obtain one suggestion at open; do not repeatedly run the resolver
   on menu refreshes. Missing/older/throwing optional APIs degrade to no suggestion and
   leave the picker usable.
2. Introduce a pure preselection helper mirroring
   `src/native/pomodoro_themes/preselect.rs` in bob-cli. Consume safe snapshots of
   suggestion, components, caps (`exempt`, `strict`, `maxThemes`), plan meter, and
   ledger keys. Use the public Ledger Tools APIs, not config parsing or a new
   subprocess. Factor common budget projection setup so public picks with a task get the
   same creation/meter rules as direct picks.
3. For an empty query, first reject an unknown suggestion. Otherwise prefer a running
   exact open entry, then first exact in ledger order, then first containing. Exact
   means its distinct component keys, removing exempt keys other than S itself, equal
   `{S}`. Keep Today rows in ledger order, with the selected row marked in place. A
   containing row keeps its complete name.
4. With no matching entry, create a suggested row only for a known theme, with an
   existing Pomodoros section, at most one timed open entry, and a permitted strict-cap
   projection. Exempt or already-counted components must not falsely consume another
   theme slot. Reuse the creation anchor and meter; display `＋ New [S] Pomodoro`,
   preserve catalog spelling in its eventual choice, and remove its duplicate from
   Recent. Hidden themes are still eligible, including when absent from Recent.
5. Otherwise retain today's default and expose `unknown_theme`, `creation_blocked`, or
   `strict_cap` in the model. Annotate the best visible matching row when possible,
   without inventing a selectable unknown row. For nonempty queries, preserve existing
   PT7 ordering and first selectable selection; annotate at most one matching row
   without moving it. Apply the suggestion logic even when the current history helper's
   early return skips Recent because creation is disallowed.
6. Copy bob-cli `tests/fixtures/pomodoro_themes/vectors.json` byte-for-byte to
   bob-plugins `scripts/fixtures/pomodoro_themes/vectors.json`. Run all `preselect`
   vectors through the pure helper, including `already_queued` as a supported vector
   input. Production Link to today does not gain a new queued-link fallback: its
   existing toggle handles already-linked tasks.

Add model tests for typed queries, creation-blocked existing matches, hidden themes,
suggested-create Recent deduplication, catalog spelling, non-default exempt keys, exempt
S itself, and strict-cap accounting. Add runtime/API tests for the one-based/full-text
conversion, missing Ledger Tools, and optional task requests. Document the additive
request shape without claiming that remembering is already wired. Run the phase
verification below and sync.

## Phase picker-intent

Depends on suggestion-model. Work mainly in 076/085/126, styles, and new view helpers.
The field writer stays deferred until guarded-theme-write.

1. Track selection by stable row key and distinguish user input from refresh. Store the
   settled initial key used to determine deliberate intent. While untouched,
   asynchronously arriving catalog/history data may revise the default and its initial
   key; after navigation or typing it may not reset the user's position. Preserve the
   same key through refresh if it survives. If it disappears, choose a deterministic
   remaining selectable row without clearing the user's interaction state. Actual query
   changes still apply PT7 first-row selection. Clearing a query does not erase the fact
   that the user interacted. Do not rerun `suggestFor` for these refreshes; catalog
   knowledge and display decoration can refresh separately from task semantics.
2. Render `Suggested · this task`, `Suggested · reading task`,
   `Suggested · daily review`, or `Suggested · from ● <stem>`. The dot uses `from.hue`
   (the place's hue), while the theme pill uses the theme hue. Reuse PT6 tint/style
   tokens. Include textual accessible labels such as
   `Pomodoro WRITE, suggested, from sase_blog`; color is never the only cue.
3. Add a second footer line refreshed on every selection/query/refresh change. Use
   `taskThemeOf` plus `planThemeChange` against the shown snapshot to preview the actual
   pending set/clear. Show it only for an effective edit:
   `this task becomes FIX · sase_blog suggests WRITE · ⌥↵ link once`, or
   `task theme cleared · sase_blog suggests WRITE · ⌥↵ link once`.
   Reading/review/no-inherited phrases are `reading tasks suggest READ`,
   `daily review suggests PLAN`, and `no default here`.
4. Bind Alt+Enter explicitly before the generic Enter/Shift branch. It commits the
   selected row with `linkOnce=true`, `deliberate=false`. Plain Enter, row click, and
   Ctrl+Shift+Enter commit the selected row; deliberate is true if its key differs from
   the settled initial key or the query is nonempty. Shift+Enter retains
   named-create/exact-existing behavior and is deliberate when it commits a named
   intent. Moving away and returning to the original key with empty search is not
   deliberate. An untouched double tap is not deliberate. Cancellation writes nothing.
   Ctrl+D on the suggested create row is consumed without hide/unhide calls, including a
   hidden suggestion.
5. Every task-aware committed choice carries
   `themeIntent: {deliberate, linkOnce, shown}`. `shown` is the suggestion actually
   presented (or null), preserved independently of subsequent API changes. Extend
   `normalizePomodoroLinkPickerChoice` to retain it along with `preserveSpelling`;
   absent intent must keep the legacy return shape. Treat malformed optional intent
   conservatively and never let it enable a write. API version remains 1. dm.7 will
   consume this contract without further changing this picker.

Test keyboard, click, no-op, accessibility, and footer behavior with the existing modal
harness. Exercise deferred fetch and onChange refreshes both before and after
arrows/typing, query clearing, removed rows, and a late callback after close. Verify the
initial key does not silently follow user navigation. Update API docs for themeIntent
and the manifest description to `↵ = suggested, else current/next`. Generate the
inventory. Run the phase verification and sync.

## Phase guarded-theme-write

Depends on picker-intent. Keep helper logic outside the nearly-full 120 fragment. Main
integration points are 070, 120, 124/130 and a new helper.

1. In `applyPomodoroTaskLink`, retain all existing stale-line, blocked-task,
   project-task, inbox-route, daily-note, and destination guards. After the destination
   is successfully planned and before any write, use the resolved destination name from
   `pomodoroPlan`, never the query, slug, or display time/title. Recompute `suggestFor`
   from the current original residence and full live task note with the correct
   one-based line number.
2. Compare its write-relevant identity to `themeIntent.shown`: suggested key, source,
   declaring note/chain, inherited identity, and repair state. Ignore purely decorative
   color and catalog-loaded metadata when deciding whether the semantic suggestion
   changed. If it changed or disappeared, link as usual, skip the theme edit, and end
   the outcome with `· task theme unchanged (suggestion changed)`. Catch optional API
   failures conservatively. Never write using stale displayed inheritance.
3. Apply the existing pure write rule with the current field state and exempt keys. No
   deliberate choice, linkOnce, no shown suggestion, an unnamed or all-exempt
   destination, or more than one non-exempt component means no edit. A sole X equal to
   inherited clears every theme field; otherwise set X with catalog spelling unless the
   valid field already has that key. Invalid or duplicate fields repair only on an
   eligible deliberate write. Use the Ledger Tools setters for exact placement; do not
   duplicate their parser.
4. Pass a narrowly scoped `themeEdit` into `planTargetTaskUpdate`. Apply it after lane
   changes and before block-ID append/fresh stamp in the existing whole-line
   replacement. Return effective theme-change metadata for notices and include it in
   `hasChanges`. Preserve the old `lineRewritten` freshness predicate: a theme-only edit
   must not cause a freshness stamp. A lane/ID/ schedule edit still stamps exactly as
   before and preserves canonical field placement, CRLF, blockquote prefixes, and
   trailing block ID.
5. Same-note task and ledger edits remain one guarded combined operation and one undo
   unit; do not add a second field-only write. Cross-note ordering remains task then
   daily note. On a failed daily write after saving a field, explicitly say the task
   theme was saved/cleared and the link failed. A failed task write must never be
   reported as a saved theme.
6. Successful edits use `Linked under FIX · task theme FIX` or
   `Linked under WRITE · task theme cleared`, retaining relevant lane, schedule-log,
   budget, cleanup, and vocabulary-warning information. Integrate with
   `formatPomodoroLinkOutcome`, partial failures, review-walk continuation, and routed
   outcomes so every path reports the same truth. Do not advance the walk more than the
   existing successful-link behavior.
7. Keep the gesture order picker → block-ID prompt → inbox route → write. Routing must
   never introduce the destination default into the shown suggestion or remember a theme
   on the originating inbox task. Cover an inbox with no shown suggestion and a route to
   a note with a default. Preserve theme fields through unlink, close, move, and other
   never-writer callers of the generic planner; only the deliberate link path passes
   themeEdit.

Extend target-update, plan-budget/freshness, picker-runtime, link-runtime, and API
tests. Include a theme-only Next task, set and clear with lane/schedule/ID changes,
same-note combined edit, duplicate/paren repair, stale inherited default with unchanged
task line, Alt+Enter, absent APIs, blocked/refused links, cancellation, and fault
injection where the task save succeeds but the daily save fails. Check write counts and
exact before/after bytes as well as notices. Include an API consumer round-trip proving
themeIntent survives normalization for dm.7.

Finish `docs/block-id-prompt.md` (Linking tasks to today: defaults, reason, footer,
Alt+Enter, deliberate rule, stale suggestion, inbox behavior) and
`docs/develop/plugin-apis.md`. Describe only shipped behavior. Regenerate inventory, run
verification, and sync.

## Verification and landing

After each phase run `npm run build`, `npm test`, and `npm run validate` from the opened
plugin repo. `npm test` verifies generated code and inventory before the Node suites.
Run `npm run docs:gen` when manifest/docs facts change. For changed docs, run
`npm run docs:check` with `BOB_CLI_DIR` set to the current host bob-cli checkout, so the
docs script uses the authorized checkout rather than cloning its own cache. The focused
baseline command is:

```sh
node --test \
  scripts/test-block-id-prompt-pomodoro-targets.cjs \
  scripts/test-block-id-prompt-pomodoro-picker-view.cjs \
  scripts/test-block-id-prompt-pomodoro-picker-runtime.cjs \
  scripts/test-block-id-prompt-pomodoro-link-runtime.cjs \
  scripts/test-block-id-prompt-target-update.cjs \
  scripts/test-block-id-prompt-pomodoro-link-picker-api.cjs \
  scripts/test-block-id-prompt-plan-budget-and-freshness.cjs
```

Add new suites to `npm test` when introducing separate test files. Verify the fixture
copy is byte-identical to bob-cli. After every modifying phase run
`bob plugins sync --repo <opened-plugin-repo> --no-pull -p block-id-prompt` and report
the actual result. Do not deploy from an unrelated checkout or hand-edit vault plugin
files. If changing another plugin becomes necessary, include it explicitly in sync and
explain why.

The land agent rechecks the parent phase requirements against the finished tree, runs
the complete plugin and docs gates, verifies sync, and checks the
suggested/default/query/Alt+Enter/one-undo behavior in Obsidian when a usable session is
available. If no UI is available, report that manual verification was not performed; do
not claim the Node harness proves the desktop undo UI. Use `/sase_monitor` for commands
that need a long-running handoff and wait for the handoff command itself to exit.

Do not hand-set bead status or create follow-up beads. Record unrelated discoveries on
bob-cli-dm.6 as `PROPOSED FOLLOW-UP: <summary — evidence>`. For a failing gate,
reproduce on the clean base before classifying it as pre-existing. An identical
clean-base failure is documented with evidence (and an existing task ID if known) and
does not hold this phase open.

Before closing any assigned phase, run `sase bead epic-symbols <assigned-id>` and
resolve or re-key all remaining entries to a still-open appropriate bead. After child
work is complete, the land agent also runs `sase bead epic-symbols bob-cli-dm.6`,
handles leftovers, and uses
`sase bead close bob-cli-dm.6 --note "<implementation and verified checks>"`. Close no
outer ancestor. Follow `/sase_final` for the repository declarations; the linked plugin
repo's changes are part of the declaration.
