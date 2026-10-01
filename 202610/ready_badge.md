---
tier: tale
title: A shared READY badge for daily notes and the dashboard
goal:
  Daily files and dash share a live READY backlog badge that turns red above a
  configurable limit defaulting to 100.
size: medium
proposed_by: bbugyi200.athena.0uq
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0uq](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0uq.md)
- **COMMITS:**
  - [8957f4a](https://github.com/bobs-org/bob-cli/commit/8957f4a7720291ce4f7ef53e53e06611a5edf382)
    — feat(plan): add max_ready soft cap for READY backlog (default 100)

# A shared READY badge for daily notes and the dashboard

## Outcome

Add READY to the daily note's existing live `bob-plan` badge row, immediately after NEXT
and before the theme names. Daily notes and `dash.md` use the same READY component and
show `READY 87/100`, for example. The normal accent is the dashboard's existing blue;
the badge turns red only when its count exceeds its configurable limit. Exactly 100 is
allowed by the default limit of 100.

This is a **soft limit**: it communicates backlog pressure without refusing capture,
changing task statuses, or removing tasks. Clicking the badge opens `dash#READY Tasks`,
providing a direct path from noticing the count to reviewing the queue.

One agent can implement this bounded feature across the plugin, its dashboard caller,
and the existing shared configuration. A tale of size `medium` fits; independent epic
phases would add coordination without useful separation.

## What the exploration established

- `bob-plugins/plugins/bob-ledger-tools/main.js` already owns daily `bob-plan`
  rendering, the live Tasks-cache integration, the ledger-derived Today cache, and the
  version 3 public `api`. Its CSS already follows the dashboard's chip geometry. The
  current daily row has PLAN, TODAY, PENDING, NEXT, then themes.
- The vault's `dash.md` owns the dashboard's DataviewJS chip row. READY is currently an
  uncapped blue chip with separate label and value spans. Its READY count and Tasks
  section exclude tasks already in Today. PENDING and NEXT use whole-lane meters,
  including Today; READY deliberately differs because it is the backlog.
- Both `_templates/daily.md` and the current daily note already contain an empty
  `bob-plan` block. Updating the processor automatically upgrades existing daily files
  containing that block and all future daily files. No daily-file migration or template
  edit is needed.
- Limits come from `plan:` in the Bob YAML config. JavaScript exposes camelCase caps;
  Rust parses the same snake_case configuration in `src/native/config/plan.rs`. The
  chezmoi source is `home/dot_config/bob/config.yml`. That source still contained legacy
  `max_now` when inspected, so reread it before editing and preserve concurrent lane-cap
  work rather than treating this feature as a migration of other settings.
- `src/native/dataview/tasks/mod.rs::READY_QUERY` is the freshness universe. It includes
  Today and is therefore not, by itself, the dashboard READY backlog.

Honor the accepted sticky-lane and ledger-Today decisions. The prior dashboard contract
was reviewed through the audited read of `plan:202609/retire_now_sticky_lanes.md`. This
feature adds an observation and a limit; it does not revise those decisions.

## Product and data contract

### The count

Count the global READY backlog shown by `dash.md`, regardless of which daily file hosts
the badge. Use Obsidian Tasks objects and their configured status **type `TODO`**,
including custom TODO symbols, rather than assuming `[ ]`.

The predicate must match the dashboard READY section's effective filters:

1. Exclude completed, cancelled, and non-task entries.
2. Apply the existing dashboard note-wide visibility rules: template exclusion,
   `dash.md` self-exclusion, hidden-tag handling, and the Tasks global query's
   `_conflicts` exclusion. Obtain the Tasks list through `getTasks()` rather than
   scanning Markdown independently. The installed Tasks implementation returns its raw
   task cache, not the result of `globalQuery`: apply the deployed `_conflicts` path
   exclusion explicitly. The cache is indexed with the global `#task` filter. Preserve
   this distinction in the fixtures.
3. Exclude dependency-blocked tasks using `task.isBlocked(allTasks)` with the **full**
   Tasks list, not an already filtered subset.
4. Include unscheduled tasks and tasks scheduled on or before the current local day;
   exclude future-scheduled tasks. Use the existing Tasks-date adapter.
5. Exclude tasks for which the existing ledger API's `isToday(task)` is true.

Reuse existing low-level status, path, date, and blocked adapters. Check differences
before reusing `planLaneVisible`: its hide-subtag and case behavior may be broader than
the dashboard's current explicit `!task.tags.includes("#hide")` rule. Preserve the
effective dashboard query semantics, including hidden-tag behavior, and pin them with
fixtures. Do not silently change other dashboard sections to make a new counter agree.
Self-exclusion always refers to `dash.md`, never the hosting daily note; ordinary ready
tasks living in a daily note remain eligible.

The badge on an older daily file is a **live current backlog**, not a historical
snapshot. Its tooltip explicitly says this. Today exclusion and scheduled-date
eligibility use the current day even when the PLAN portion describes that older file's
ledger.

### The setting

```yaml
plan:
  max_ready: 100 # soft limit for dashboard READY tasks, excluding Today
```

Use `plan.max_ready` alongside the existing lane caps, with `maxReady` in the plugin's
`caps()` result. Missing config, a missing key, and a null value default to 100. Valid
overrides are integers from 1 through `u32::MAX`; reject zero, negative values,
fractions, strings, booleans, and overflow consistently in Rust and JavaScript for this
new key. On mobile, use the existing built-in-default behavior when desktop
configuration is unavailable.

Follow the established invalid-config policy: Rust's `bob plan` reports a clear
configuration error and exits 2; non-strict consumers and the plugin use the full
default plan configuration with the existing warning mechanism. Include the fallback
explanation in the READY tooltip on the dashboard, too. Do not create a second Obsidian
setting, per-note override, or separate persisted count.

Rust must validate this shared setting and expose `caps.max_ready` additively in its
existing plan report. Keep JSON schema version 2. This feature does not add a native
READY count, a new lint, capture enforcement, or a tmux meter: its count is the Obsidian
dashboard backlog and its feedback is the shared badge.

### Appearance and interaction

Use a single plugin-owned READY renderer and stylesheet on both surfaces. Match the
dashboard: a subdued small-caps READY label, a stronger blue tabular-numeral value, a
2px accent start border, a softly tinted background, a 7px corner radius, and the
existing compact padding and row gap. Red replaces the accent, border, and tint when
over; keep text readable in both light and dark themes. Preserve the existing
dashboard's other chips and the daily row's wrapping behavior.

Show `READY 0/100`, `READY 100/100`, or red `READY 101/100`. There is no intermediate
warning color. Use a muted `READY –` when Tasks data, required Today state, or the count
is unavailable; zero is reserved for a successfully evaluated empty queue. The limit
remains available in that placeholder's tooltip.

The tooltip and accessible label describe the count, limit, current-backlog scope, Today
exclusion, and destination. When over, state the excess in words, for example
`101 ready tasks; limit 100; 1 over the limit. Open READY Tasks in dash.` Color is
supported by the visible fraction and accessible text. Keep the tooltip concise; do not
put implementation details in the note UI.

Support mouse activation, Enter/Space, Ctrl/Cmd activation in a new leaf, visible
keyboard focus, and Obsidian's link-hover preview, using the current source path as
navigation context. Respect reduced motion if retaining the dashboard's hover lift. Do
not add an arrow to this internal navigation link.

## Implementation steps

1. **Open and inspect the authorized sources.** Use `/sase_repo` to open `bob-plugins`,
   `chezmoi`, and `gh:bobs-org/bob`, with task-specific reasons. Read each opened
   `AGENTS.md` and inspect status before edits. Use only the returned checkout paths for
   source reads/writes; never edit deployed custom plugin files. Recheck the READY
   query, config, and plugin API against current heads. Read `obsidian.md` and the
   applicable task-lane/Today decision memories through `/sase_memory_read`. No
   memory-file changes are part of this plan.

2. **Extend the existing cap contract.** Add `max_ready`, its default and getter, to
   Rust `PlanConfig`, the parser, and `PlanCaps::caps_of`. Add `maxReady` to the
   plugin's `defaultPlanCaps`, normalization, and YAML loading. Add the documented
   `max_ready: 100` entry to the chezmoi Bob config and adjust its comment only as
   needed; preserve existing settings and any concurrently landed cap migration. Update
   `docs/plan.md` with the backlog definition, boundary, config example, additive caps
   JSON field, and the daily/dashboard surface contract.

3. **Own READY computation and rendering in Bob Ledger Tools.** Introduce a pure,
   testable backlog-budget helper using the predicate above. Add synchronous, guarded
   `api.readyBudget()` returning `{ count, cap, over }`, where `count` is `null` when
   unavailable and `over` is false in that case. Add
   `api.renderReadyBadge(parent, { sourcePath, component })` for the dashboard; it uses
   the shared element renderer and registers lifecycle-owned live updates. These are
   additive API version 3 members; keep every existing member unchanged. The daily block
   uses the same element renderer, model, and CSS internally. Include READY in the daily
   row's accessible description without changing what the ledger's PLAN status means.

4. **Give both surfaces the same live state.** Extend the existing debounced refresh
   path for Tasks cache updates, Today rebuilds, layout readiness, and local-day
   rollover. A Ready task being linked or unlinked from Today must refresh both badges
   even if its checkbox has not yet reconciled. Read the caps afresh on refresh, and use
   the existing one-minute interval to detect a changed cap or day, so saving the config
   is reflected within 60 seconds without a plugin reload or new per-badge poller. Date
   changes must refresh READY even when no daily note exists and the Today key set
   remains empty.

   Distinguish a valid empty Tasks list from unavailable data. The installed Tasks API
   also exposes `getState()`: its states are `Cold`, `Initializing`, and `Warm`. When
   that method is available, show the placeholder until `Warm`; a cold cache returns
   `[]` and must not be mistaken for an empty backlog. Keep compatibility with hosts
   that expose only `getTasks()`. Wait for the initial current-day Today build before
   claiming a count. A missing current daily file can settle to a valid empty Today set;
   it does not suppress READY. A failed dependency/count evaluation produces an
   unavailable badge rather than an apparently healthy partial result. Resolve READY
   synchronously from current state, and guard asynchronous daily-block paints against
   older reads overwriting newer renders. Reuse a batch state for the open READY
   widgets.

   Daily views keep their existing render-child lifecycle. Dashboard widgets attach to
   `dv.component` (the installed Dataview API exposes this lifetime owner). Unregister
   on unload, replace a component's old widget on Dataview rerender, prune detached
   nodes, and clear timers/views on plugin unload. Reopening a note must not accumulate
   widgets, listeners, or timers.

5. **Adapt only the dashboard READY slot.** In the opened vault's `dash.md`, render
   READY through the new API at its existing position between NEXT and BLOCKED, passing
   `dv.current().file.path` and `dv.component`. Avoid letting the generic chip loop
   render a second READY badge. The normal path uses no inline READY counting or
   READY-specific style copy. Keep a small compatibility fallback for an older/unloaded
   ledger plugin: use its existing inline READY count when evaluable, `caps().maxReady`
   when valid or 100 otherwise, and the existing chip style with an over-limit state.
   Missing Tasks yields a placeholder. Guard all optional API calls so deployment
   ordering cannot break the rest of the dash. Preserve the task queries, frontmatter,
   other badges, and surrounding notes.

6. **Document and deploy the completed feature.** Update the plugin README's row,
   daily-block description, and public API documentation; bump Bob Ledger Tools'
   manifest minor version from the implementation's current head. Do not rewrite
   historical daily files: the existing `bob-plan` blocks gain READY at render time.
   Install the validated bob-cli build using the established workflow if needed for the
   updated native config/caps contract. Run
   `bob plugins sync --no-pull --repo "<opened bob-plugins path>" --plugin bob-ledger-tools`
   and inspect its outcome; this deployment is required by that repo's instructions.
   Apply the chezmoi config from its source of truth; follow its required
   `chezmoi update -a --force` after its change is committed, reviewing the pending
   apply diff first. Reconcile the edited vault checkout with
   `BOB_DIR="<opened vault path>" bob vault-sync run`, then use `bob vault-sync run` and
   `bob vault-sync status --json` for the normal installed vault. Use the sync tool's
   commit/reconcile path, preserving pre-existing work. Never manually run git commit or
   copy custom plugin sources into the vault.

## Verification and acceptance

Extend `scripts/test-ledger-tools-plan-budget.cjs` or add a focused READY test file
registered in `package.json`. Use meaningful fixtures and component/event stubs:

- Missing/null/default configuration, a custom positive cap, and invalid numeric
  types/range; Rust and JavaScript agree for the new field. Old configurations and
  legacy `max_now` still load.
- Counts at 0, 99, 100, and 101, plus the equivalent custom-cap boundary. Only a strict
  excess turns red; `over` is independent of the PLAN ledger status.
- Custom TODO symbols; other statuses; dependencies referencing tasks outside the
  filtered queue; past/today/future schedules; dashboard self-exclusion; tasks in
  ordinary daily files; template/conflict/hidden-tag cases; and Today exclusion. Pin the
  queue predicate against the effective dashboard READY query with shared fixture data,
  including the hide-subtag distinction identified above.
- Both surfaces use the same READY element structure, classes, fraction, over state,
  tooltip, and destination. Verify keyboard and modifier-key navigation.
- Null/throwing Tasks access, Cold/Initializing/Warm cache transitions, valid empty
  Tasks, initial Today loading, absent daily note/Pomodoros section, and an older ledger
  API produce the specified UI without breaking the rest of the dashboard.
- Task-cache updates and Today-only changes update both open widgets. A config change
  and local-midnight transition update without note recreation, including an empty Today
  set. Old asynchronous paints cannot restore stale READY state. Rerender/unload/reopen
  does not retain duplicate widgets or live registrations.

Update Rust config and plan-report tests for the new field, including JSON objects that
assert exact cap shapes. Run `npm test` and `npm run validate` in bob-plugins, and
`just all` in bob-cli after the changes. Use `/sase_monitor` for long-running checks
when needed. Do not mutate real task statuses or populate the real vault with
boundary-test tasks.

Execute the dashboard DataviewJS block in a stubbed host using the opened vault file as
input to catch syntax, duplicate-slot, and fallback regressions. Headless `bob query`
can confirm that the unchanged READY Tasks block still parses, but cannot prove its
Obsidian Today filtering or visual rendering. Confirm live API READY counts match the
dashboard READY section when Obsidian is available.

Visually verify daily and dash together in Reading View and Live Preview, light and dark
themes, and a narrow pane: label/value balance, wrapping, red state, placeholder state,
focus, hover, and link destination. Prefer screenshots from the actual host; if no
Obsidian UI is accessible, report that limitation and give Bryan the short remaining
visual checklist without claiming it was tested. The feature is complete only after
automated checks and deployment succeed, with any actual visual-check limitation
explicitly stated.
