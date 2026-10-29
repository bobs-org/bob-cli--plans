---
tier: tale
size: medium
title: Repair task-theme picker refresh and guarded writes, then finish landing
goal:
  Fix the three verified gaps in bob-cli-dm.6.1 and complete its landing and parent
  phase bob-cli-dm.6 in the same coding turn.
proposed_by: bbugyi200.apollo.bob-cli-dm.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dm.6.1
create_time: 2026-10-28 21:42:47
status: wip
---

- **PARENT:**
  [202610/task_theme_link_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_link_picker.md)
- **BEAD:**
  [bob-cli-dm.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.6.1.md)

# Repair task-theme picker and finish landing

This tale contains only remaining work for epic **bob-cli-dm.6.1**, whose parent is
phase **bob-cli-dm.6** in outer epic **bob-cli-dm**. Implement the repairs, verify them,
and perform the closeout below yourself. There is no later land agent for this tale. Do
not wait for this turn's commit, SHA, push, or CI before closing: SASE commits after the
turn ends. Leave outer epic bob-cli-dm to its waiting land agent.

## Context and fixed decisions

Read with audited commands:

```sh
sase bead read bob-cli-dm.6.1 -r "Need landing audit and remaining gaps"
sase bead read bob-cli-dm.6 -r "Need parent phase scope and final decisions"
sase artifact read plan:202610/task_theme_link_picker.md "Need picker contract and epic closeout path"
sase artifact read plan:202610/task_theme_suggestions.md "Need inherited task-theme rules"
```

Open **bob-plugins** with `/sase_repo` and read its `AGENTS.md`. Product changes are
confined to `plugins/block-id-prompt/src/`, its generated `main.js`, and relevant tests.
Use Ledger Tools' existing public APIs and pure field helpers. Keep source fragments at
or below 1000 lines, registering any new fragment in `src/fragments.json`.

The implementation contract is **PT13** in bob-cli's `docs/pomodoro-themes.md`; retain
PT12 configured vocabulary, catalog spelling, warnings, and tint behavior. The inherited
decisions remain ledger_theme_pills=yes, seed_plex_chore=yes (belongs to dm.9), and both
memory decisions=no. Make no memory changes, task-theme storage/resolver redesign, Task
Card or successor writer changes, capture/Swift changes, or vault-default edits.

The lander read every note on dm.6.1 and all three closed phases, plus parent dm.6 and
both plans. Their three implementation commits in bob-plugins are **97341c4**,
**ea583df**, and **0f74920**. On 0f74920, `npm test` passed **4131/4131** and the shared
`scripts/fixtures/pomodoro_themes/vectors.json` was byte-identical to bob-cli's fixture.
`npm run validate` also passed (8/8 plugins; generated code and inventory current).
Existing tests miss the failures below; some refresh tests only check that an already
correct default remains selected.

Integration audit through bob-cli **cba7c47d** and bob-plugins **0f74920**:

- No unrelated plugin commits followed the child epic's baseline **bc245fc**.
- Bob-cli changes after the planning baseline **41cc4acc** are relation completion
  (**387a4fc2**), manual heading parsing (**4005c613**), theme rename/split verifier
  preservation (**687375eb**), local install profile (**38f15d3e**), and reverse
  relation rename (**cba7c47d**). The verifier change retains task-field coverage and
  introduces no picker API change; no adaptation was needed. Recheck any later drift
  before closing.

**Follow-up triage is complete and recorded on dm.6.1.** Neither this epic, its three
children, nor dm.6 had a `PROPOSED FOLLOW-UP:` entry. There were no distinct unrelated
discoveries, new task beads, corroborations, or declined proposals. The defects below
are caused by this epic and must be fixed as epic work. Carry these outcomes into the
close note. If genuinely new unrelated discoveries arise, use `/sase_new_task` before
recording a new task and identify the proposing bead in its evidence.

## 1. Refresh catalog knowledge without changing the task suggestion's meaning

Relevant code: `076-pomodoro-theme-picker.js` (`safeBlockPomodoroThemePickerSnapshot`,
`withPomodoroThemeHistoryRows`, `startBlockPomodoroThemePicker`) and
`085-pomodoro-link-picker-modal.js` (`updateQuery`, `shownSuggestion`, `initialKey`).
The snapshot freezes `theme.known` and `choice` at open; fetch/onChange callbacks only
rebuild rows. `updateQuery` always preserves a surviving key even before interaction.

Reproduced: today's only open entry is BUILD, the note suggests WRITE, and the catalog
initially reports WRITE unknown. After the catalog reports WRITE known and a refresh
runs, the untouched picker still selects BUILD with
`choice={kind:"fallback", code:"unknown_theme"}` and no suggested-create row.

Separate the fixed semantic suggestion (key/source/from/chain/inherited/repair) from
refreshable catalog knowledge and decoration. Continue calling `suggestFor` only once at
open and again in the guarded write path, never on menu refresh. Recompute menu
eligibility/preselection from the existing semantic snapshot and current safe catalog
knowledge; retain the existing ledger/budget/creation rules and optional-API fallback.
Refresh catalog spelling and applicable decoration without silently adopting a changed
task default. Keep the returned `themeIntent.shown` semantically faithful to what was
presented; decorative/catalog changes must not defeat the writer's semantic comparison.

On an empty query **before interaction**, adopt the newly eligible suggested row and
update `initialKey` along with selection. Enter/double tap on that row remains
non-deliberate. **After arrows or typing**, preserve the selected stable key if it
survives and retain the settled initial key; use the existing deterministic fallback
when the row disappears. Clearing a query must not reset interaction. Nonempty query
refreshes preserve the user's position and never override PT7 with preselection.

Extend the modal/runtime tests with actual deferred `fetch` and `onChange` transitions:
unknown-to-known suggested creation, eligible existing match, before and after arrows,
typing then clearing, removed rows, and callbacks after close. Prove no additional
resolver call, correct initial key, unchanged Today ordering, and non-deliberate Enter
on a revised untouched default. Include a catalog spelling other than canonical
uppercase.

## 2. Keep typed Recent candidates and annotate the final visible row

In `withPomodoroThemeHistoryRows`, the code unconditionally excludes the suggestion from
Recent whenever the _empty-query snapshot_ says `choice.kind === "create"`. However, it
only creates the synthetic suggested row when the current query is empty. It also marks
suggestion matches before constructing Recent rows, so a surviving Recent match can lose
its reason when the generic new row is removed.

Reproduced with open BUILD, known suggested catalog theme `{key:"write",name:"Write"}`,
creation allowed, and query `wri`:

```text
Without suggestion: Recent Write, New WRI
With suggestion:    New WRI
```

Exclude the suggested theme from Recent only when the corresponding synthetic create row
is actually in this result set. Select/annotate the best matching suggestion row after
composing the final visible rows (including Recent), or otherwise preserve the same
invariant. At most one visible matching row carries the reason. Keep empty-query
suggested-create deduplication and Today order intact. For nonempty queries, use
precisely the existing PT7 ordering/first selectable default, with suggestion decoration
only.

Add model/view regressions comparing suggestions on/off for a partial query such as
`wri`, exact `write`, and a fallback where the visible match is a Recent row. Assert row
order, first selection, exactly one reason, and catalog spelling/preserveSpelling
through the committed choice. Existing hidden-theme and Ctrl+D behavior must continue
passing.

## 3. Skip theme writes when current exempt keys are unavailable

In `078-pomodoro-theme-write.js`, `pomodoroThemeWriteExemptKeys` converts missing,
throwing, or malformed `caps()` results into `[]`. That is indistinguishable from a
valid configuration with no exempt themes. Reproduced using the **real** Ledger Tools
`pomodoroThemePlanChange` helper: deliberate choice of PLAN, shown note suggestion
WRITE, and `caps()` throwing produces `{action:"set",name:"PLAN"}` instead of leaving
the field untouched.

Return a distinct unavailable result for missing/throwing/malformed caps, including
Promise results and malformed `exempt` elements; skip only the optional theme edit when
that result occurs. Continue the guarded link normally. A genuine `exempt: []` is valid
and must retain its existing semantics. Read current caps at commit; do not fall back to
a stale picker snapshot. Preserve current-field parsing, suggestion revalidation,
resolved destination naming, setters, freshness eligibility, and truthful
success/partial notices.

Add focused and runtime fault-injection tests for missing/throwing/malformed caps after
a valid picker intent was created. Use the real Ledger Tools pure parser/planner/setters
where practical instead of another copy of the write rule. Assert the link completes,
the task field is byte-for-byte unchanged, no saved-theme claim appears, and ordinary
lane/ID edits retain their old freshness behavior. Test an actual empty exempt array, a
custom exempt name, and PLAN + FIX with valid caps to prove legitimate writes survive.

## 4. Verify, deploy, and finish this landing in the same turn

Run the meaningful new regressions, then the complete plugin gates from the opened repo:

```sh
npm run build
npm test
npm run validate
```

Run `npm run docs:check` with `BOB_CLI_DIR` set to the active authorized bob-cli
checkout; this avoids the docs script cloning a different repo. Read the docs memory
first if documentation needs correction, and regenerate inventory if manifest/docs facts
change. Verify the fixture copy remains byte-identical. For bob-cli file-change
verification use `just check`, never `just check-full`. A check-full infrastructure
failure is not epic work. Use `/sase_monitor` if a long command needs a handoff, with
all remaining closeout instructions in its continuation; do not end a turn with any
inline command still running.

Deploy the generated plugin with
`bob plugins sync --repo <opened-bob-plugins-path> --no-pull -p block-id-prompt` and
report the actual result. Check suggested/default/query/Alt+Enter/one-undo behavior in
Obsidian only if a usable session is available. Otherwise explicitly state that
desktop/manual undo verification was not performed; the Node harness is not proof of the
desktop UI.

Before closure, reread dm.6.1 and all children to catch new notes, verify descendant
readiness and the linked plan's requirements, and review commits after the audit SHAs
above in both repositories. Finish any new epic-caused conflict. Keep dm.7's public
`themeIntent` v1 contract unchanged.

Then complete these steps, without waiting for this turn's commit:

1. If this tale has its own assigned bead that is an open descendant of dm.6.1, close
   that bead normally after its implementation and checks pass, so it cannot block the
   epic close. Resolve any exemptions keyed to it first. Do not use force.
2. Run `sase bead epic-symbols bob-cli-dm.6.1`. Resolve every listed exemption (wire,
   privatize, non-test pragma, or delete under the Symvision policy), or re-key it only
   to a still-open later bead that actually needs it. None existed at the initial audit.
3. Close normally:
   `sase bead close bob-cli-dm.6.1 --note "<verified original three phases, repair regressions, final test results, deployment/manual limits, integration through the current SHAs, inherited DECISIONS, and the follow-up triage outcomes above>"`.
   If rejected, finish the named work/exemptions and retry; never use `--force` merely
   to succeed or to advance a successful nested landing.
4. Run `just symvision` when that recipe is available; otherwise report its absence (the
   audited bob-cli checkout has no such recipe). Open the plans sidecar through
   `/sase_repo` and set `status: done` in **plan:202610/task_theme_link_picker.md**,
   whose known PLAN path is `sase/repos/plans/202610/task_theme_link_picker.md` relative
   to the host checkout. Use the current audited bead/artifact resolution if its path
   changed. Do not directly read sidecar artifact bodies; use `sase artifact read` for
   context.
5. Read `sase bead read bob-cli-dm.6.1 -r "Need the parent link"` and verify that parent
   **bob-cli-dm.6** remains a phase with this child plan fulfilling its link-picker
   requirements. Recheck its notes and descendants. Run
   `sase bead epic-symbols bob-cli-dm.6`, retire or legitimately re-key every entry,
   then
   `sase bead close bob-cli-dm.6 --note "<verified complete task-aware preselection, stable intent/reason/footer, link-once, guarded remembering and freshness, complete plugin/docs checks, deployment result and manual limits>"`
   normally. Run available Symvision after the parent exemption cleanup as well. Leave
   **bob-cli-dm** and its outer plan **plan:202610/task_theme_suggestions.md** open for
   their waiting land agent. If this parent phase is incomplete or ambiguous, note the
   exact blocker on dm.6 and report it instead of forcing closure.
6. Use `/sase_final` as the final action, declaring every repository edited by this turn
   (including bob-plugins and the plans sidecar). Report the actual closed beads,
   checks, sync result, and any unresolved parent blocker concisely.
