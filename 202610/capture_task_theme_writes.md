---
tier: tale
title: Remember deliberate capture theme choices
goal:
  Complete bob-cli-dm.4 with atomic task-theme writes, matching previews and output,
  documentation, and regression coverage.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-dm.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-dm.4
create_time: 2026-10-28 18:44:18
status: wip
---

- **PARENT:**
  [202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_suggestions.md)
- **BEAD:**
  [bob-cli-dm.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.4.md)

# Remember deliberate capture theme choices

Implement phase `bob-cli-dm.4` of `plan:202610/task_theme_suggestions.md`. An explicitly
named capture destination should remember a task's deliberate exception to its suggested
theme, or clear the exception when returning to the inherited theme. The task line and
ledger changes must commit together, and preview/output must explain the same change.

## Scope and current evidence

The phase is already assigned and `in_progress`; do not set its status by hand. The
planning read found no previous-work notes. The prerequisites have landed: `ac13ab54`
adds capture/ref-today suggestions, `13b24303` extends rename coverage, and `fc594281`
repairs the shared field helpers. No capture planner currently calls
`plan_theme_change`, and capture results have no `task_theme` property. This is one
bounded integration over existing rule and field helpers, suitable for one coding agent.
Re-read the bead and inspect the checkout before coding to avoid duplicating any
subsequent work.

All implementation is in bob-cli. Do not change plugins, Swift, the real vault,
Cargo.toml's version, or CHANGELOG.md. This is additive API behavior; keep the existing
schema version. The parent decisions are final: ledger theme pills and Plex seeding are
enabled in their own phases; both proposed memory additions were rejected. Do not edit
any memory note or re-ask these decisions.

Read the parent through `sase artifact read` and the phase through
`sase bead read bob-cli-dm.4 -r "Need capture-write scope and current notes"`. Read
required memory through `/sase_memory_read`, including `docs.md` before documentation
edits. The parent calls the task contract PT12, but the current repository places task
suggestions/write rules in **PT13** of `docs/pomodoro-themes.md`; use the existing PT13
anchors and preserve published section identities.

## Observable behavior

Apply the already implemented `task_theme::plan_theme_change` rule to these capture
forms when they explicitly name a destination:

- New body-bearing task: `<text> @route:id#NAME`, including link-and-start.
- Existing task: whole-item `@route:id#NAME` and `^route:id#NAME`.
- The equivalent `=name` shorthand for the supported new/existing link-and-start forms;
  use the parser's normalized representation, not another text parser.
- Ensure Next relocation: `@route+id#NAME`.

Use the resolved destination entry's full name after lookup, prefix/slug matching,
literal-name selection, or creation. Do not store the typed selector as the theme. The
rule is:

1. A capture without a deliberate name does not change the field.
2. A destination must have exactly one non-exempt theme component. Unnamed, all-exempt,
   and multi-component non-exempt names do not change it; `PLAN + BUILD` selects BUILD
   when PLAN is exempt.
3. No suggestion means no field change, including a new task in a note with no inherited
   default. Unknown and hidden suggestions still count as suggestions.
4. Choosing the inherited key removes every task theme field.
5. Otherwise set the chosen component in catalog spelling, unless exactly one valid
   existing field already has that key.

The suggestion for a new task comes only from its destination note via
`suggest_for_new_task`; existing tasks use `suggest_for_task` on their staged pre-edit
line. For a task-source suggestion, inherited is its nested inherited value (possibly
null); for reading/review/note suggestions, inherited is the suggestion itself.
Duplicate/invalid fields use the existing repair semantics.

Guards retain precedence: a refused or consumed link, blocked new task, or refused
project task gets no field edit. Existing eligible prerequisite-drop behavior continues
normally and may write the theme after the link proceeds. A successful explicit named
choice may change a field even when the link is already current. Ordinary toggles,
unlink/release, close, whole-session start, unnamed linking, successor placement, and
ref-today must not become theme writers.

## Implementation

### 1. Add shared staged theme planning

Use `src/native/task_theme.rs` (`parse_task_theme`, `plan_theme_change`,
`set_task_theme`, `clear_task_theme`) and `src/native/pomodoro_themes/suggest.rs`; do
not duplicate their rules or field scanning. Add a small capture-local helper/module for
suggestion context, theme-change planning, and reporting rather than duplicating this
across forms.

Resolve note facts from the current `CaptureBatchPlanner` snapshot, with disk fallback
for untouched ancestors. `HomeSource`, `HomeResolver`, `NoteFacts`, and
`CaptureBatchPlanner::staged_snapshot/current_contents` supply the relevant interfaces.
Honor staged creation/deletion and use vault-relative paths for home resolution. Refresh
resolver caches between items so later captures observe earlier task and note changes.
Avoid an unconditional full-vault scan per item; reuse the catalog/config at an
appropriate batch scope and follow the existing configuration path and warning
conventions. Preserve structured suggestion warnings without duplicating them for the
same note within the batch.

Capture the suggestion and field state before this item's theme edit, using its current
residence. Resolve the destination through the existing link/start planner, then apply
the theme change to the latest task-line post-image after status, schedule, freshness,
and dependency edits. Stage it through the existing planner; do not write files
separately or add a second commit. Keep all unrelated bytes, line endings, child
content, and block identity intact. A theme edit must not alter freshness eligibility or
status decisions.

### 2. Connect the capture paths

Relevant code is `capture/plan.rs`, `capture/pomodoro_link.rs`,
`capture/ensure_next.rs`, `capture/task_toggle.rs`, and the detail types in
`capture/batch.rs`. New tasks use `plan_capture_with_pomodoro_link`; whole-item links
use `plan_existing_task_pomodoro_link_item` / `plan_pomodoro_link_capture`; Ensure Next
has its own guarded relocation path. Thread deliberate intent/context explicitly where
shared planners need it. Do not infer intent merely from a named destination, because
default destinations and ref-today can also be named.

`capture/existing_task_link.rs::link_existing_task` is shared with ref-today: pass an
explicit never-deliberate context there, or keep the writer in a capture-only caller
with an equally explicit boundary. Verify ref-today writes no field even when its user
selects another named destination.

Ensure the final task line, per-item output, and task-block previews include the field
edit. Apply it before batch block trackers finish, and never overwrite edits already
staged by dependencies or relations. New tasks must report their edited headline, and
existing tasks must retain their genuine previous headline.
`plan_capture_batch_with_config_path` applies inbox routing after all items: compute
suggestions before that routing step and preserve the edit through any subsequent move.
An inbox task with no suggestion must not acquire a field from the eventual
destination's default during the same capture.

Remove obsolete dead-code allowances/comments on shared helpers now used in production;
leave legitimately unused future-phase APIs alone. Planning-time
`sase bead epic-symbols bob-cli-dm.4` reported no entries, but recheck at completion.

### 3. Report changes consistently

Add an optional `task_theme` to `CaptureItemResult` in `capture/output.rs`:

```json
{
  "action": "set",
  "theme": "ThemeRef object, or null for cleared",
  "previous": "previous valid ThemeRef object, or null",
  "inherited": "ThemeSuggestion object, or null"
}
```

The actual values must be typed objects/null, not the explanatory strings above. Use
`action: "set" | "cleared"`; omit the entire property when nothing changed. Repair
states have no single valid previous theme, so report `previous: null`. Use existing
`ThemeRef` construction and catalog display spelling. Preserve the legacy top-level
first item and ordered `captures` array behavior. Update every result constructor with
the correct optional value, defaulting to absent for unrelated capture kinds.

After the ledger line, human output adds one theme row:

```text
theme  task theme → FIX · sase_blog suggests WRITE
theme  task theme cleared · sase_blog suggests WRITE
```

Dry runs use `would` (for example, `would set task theme → FIX` and
`would clear task theme`). Use the established inheritance copy for reading, daily
review, and no-default cases. Keep ANSI styling consistent with existing theme
rendering, and ensure the multi-item output gets the row per changed item. Dry-run JSON
reports the same planned change as an actual capture and never writes.

### 4. Document shipped behavior

Update `docs/capture.md` in Pomodoro-linked tasks, linking/starting existing tasks, Task
Link toggle/Ensure Next, and the JSON reference. Include a set and clear example,
no-suggestion/guard behavior, dry-run output, and links to the current PT13 write rule.
In `docs/pomodoro-themes.md`, update the task-theme section's stale claim that capture
commits do not yet write and document the additive `task_theme` shape. Do not claim
other pending epic surfaces have shipped. No new CLI option is required.

## Verification

Add focused integration cases in a new capture test module registered by
`tests/cli/capture/mod.rs`, reusing existing fixtures/helpers. Parameterize the
supported grammar families against set, clear, no-op, composite, exempt, unnamed, no
suggestion, and guarded destinations wherever the grammar permits each case. Use a
bare/default selector to cover unnamed/non-deliberate cases. Include:

- Exact, prefix/slug, literal, and created destinations; resolved spelling wins.
- Existing matching key is a byte-preserving no-op; paren/duplicate/invalid fields
  repair through the common helpers. Unknown/hidden suggestions remain eligible.
- Reading inheritance can set an exception and clear back to READ; configured exemptions
  and `PLAN + BUILD` follow the shared rule.
- Successful dependency drop still links and remembers; consumed project/blocked guards
  and refusals leave theme bytes untouched.
- A two-item batch targeting the same task first sets FIX and then chooses its inherited
  BUILD: the second item sees FIX as previous, clears it, and leaves no field. Compare
  both JSON item summaries with final vault bytes.
- A later item failure leaves task and ledger files unchanged; use the existing
  transaction/commit-test harness for a write failure or note-drift regression where
  appropriate rather than implementing a new transaction mechanism.
- Dry-run and write agree on task lines and `task_theme`; dry-run preserves files. Human
  set/clear output and first-item/bulk JSON are covered.
- Inbox routing uses the source residence, ref-today never writes, and unnamed
  links/toggles/close paths preserve existing fields. Freshness behavior is the same as
  the equivalent capture without a theme change.

Run the new tests and relevant capture/ref-today/shared theme-vector regressions, then
`just all` and `just docs-check` as required by the parent phase. Use `/sase_monitor`
for long commands and await the handoff command's exit; a coding turn must not end with
an unmanaged command still running. No tests were run in this planning turn because no
implementation changed.

## Completion

Record only discovered follow-ups as
`sase bead note bob-cli-dm.4 'PROPOSED FOLLOW-UP: <summary — evidence>'`; do not create
task beads. A check failure reproduced identically on the clean base does not prevent
closing this phase: record the reproduction and any existing tracking bead. Do not
modify unrelated code just to silence baseline failures.

Before closing, run `sase bead epic-symbols bob-cli-dm.4`. Resolve remaining entries or
re-key their justfile lines to an appropriate still-open owner; do not leave references
to this phase after closure. Close only this assigned phase with
`sase bead close bob-cli-dm.4 --note "<implemented behavior and verification>"`. Never
close `bob-cli-dm` or an ancestor plan bead. Leave this phase open while this plan is
being reviewed; completion belongs to the implementing agent.
