---
tier: epic
title: Ctrl+Shift+P Task Card with fast actions and full property-panel parity
goal: 'Replace the property picker''s first screen with a beautiful, reliable Task
  Card that reduces common priority actions to one key after opening, preserves every
  existing editing capability and stored side effect, and protects the October freshness
  trial with a staged rollout.

  '
phases:
- id: repair-refresh
  title: Repair refresh rendering and establish a real modal harness
  depends_on: []
  size: small
  description: 'repair-refresh: fix the confirmed malformed footer crash, verify the
    decay card''s Less often path, and ship the independent patch before the trial.

    '
- id: card-model
  title: Plan card actions and frozen priority previews
  depends_on:
  - repair-refresh
  size: medium
  description: 'card-model: add pure context-aware card and key models with independent
    per-target priority previews, availability reasons, and table tests.

    '
- id: card-view
  title: Render the compact Task Card and its accessible visual states
  depends_on:
  - card-model
  size: medium
  description: 'card-view: add the task header, recommendation timeline, priority
    strip, stable action rows, theme-native styles, and permanent classic search mode.

    '
- id: card-dispatch
  title: Connect safe keyboard actions and synchronous linked-task shells
  depends_on:
  - card-view
  size: medium
  description: 'card-dispatch: route card gestures through existing stages and writers,
    guarantee immediate focus and safe asynchronous resolution, and add a pilot setting.

    '
- id: schedule-input
  title: Add concise date input and inline scheduling reasons
  depends_on:
  - card-dispatch
  size: medium
  description: 'schedule-input: extend date input with bare day offsets, unsigned
    units, weekdays, exact previews, inline reasons, and Shift+Enter reason skipping.

    '
- id: schedule-review
  title: Combine scheduling reason and Work Log review
  depends_on:
  - schedule-input
  size: medium
  description: 'schedule-review: replace serial reason and work-summary prompts with
    one optional review while preserving every existing per-target log rule and writer.

    '
- id: parity-rollout
  title: Verify full parity and prepare the dated default rollout
  depends_on:
  - schedule-review
  size: medium
  description: 'parity-rollout: complete interaction and writer regression coverage,
    validate performance and themes, align the decay-card cancel alias, and release
    version 2 with a default activation boundary of October 19.

    '
- id: docs-hints
  title: Document the new actions, compatibility paths, and rollback
  depends_on:
  - parity-rollout
  size: small
  description: 'docs-hints: update plugin and CLI documentation, date-aware ready
    hints, and the Schedule Log glossary, then verify deployment from the source repository.'
proposed_by: bbugyi200.apollo.research.05.linker.w0
create_time: 2026-10-03 16:27:18
status: wip
bead_id: bob-cli-42
---

- **PROMPT:** [prompts/202610/ctrl_shift_p_task_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ctrl_shift_p_task_card.md)
- **BEAD:** [bob-cli-42](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-42/README.md)

# Ctrl+Shift+P Task Card

## Outcome and scope

Build a new first screen inside `BulletPropertyPickerModal` in **bob-plugins**. Keep
`bob-navigation-hotkeys:set-bullet-property`, the existing vault binding, session
discovery, configuration, property stages, planners, and mutation writers. Classic
filtered properties remain permanently available as search mode. The main speed
improvement is a deliberate P-level pick: opening the panel and pressing `2` sets P2 and
its displayed date, typically two gestures instead of six.

The baseline is the user-endorsed, audited research artifact
`research:202610/ctrl_shift_p_task_card/ctrl_shift_p_task_card.md`. Adopt its
recommendations, including default-on reason capture, the habit-safe key map, unchanged
same-level priority-pick semantics, the trial boundary, and the early refresh repair.
Name the surface **Task Card** and the palette entry **Task card (set properties)**. No
further product decisions need to be sent back to Bryan.

An epic is appropriate because the card model, visual treatment, input lifecycle, and
scheduling review each need a bounded implementation and acceptance boundary. All phases
are small or medium direct implementation work. Dependencies are serial because these
phases touch the same `main.js`, CSS, and modal tests; do not launch them concurrently
against that file. This plan does not authorize code changes until SASE plan approval.

## Repository and evidence map

Run from the assigned bob-cli checkout. Before accessing bob-plugins, use `/sase_repo`
and `sase repo open bob-plugins -r "Implement the approved Task Card phase"`. Use only
the returned checkout path and read its `AGENTS.md`. Never implement in installed vault
copies. Research artifacts must be consumed with `sase artifact read`. No work depends
on a numbered checkout path or on this planner's local directory.

Relevant bob-plugins files and symbols, verified against navigation-hotkeys 1.71.0:

- `plugins/bob-navigation-hotkeys/main.js`: `FilteredPickerModal`,
  `BulletPropertyPickerModal`, `showPropertyStage`, `showValueStage`,
  `showRefreshValueStage`, `applySelectedValue`, `maybeOfferPriorityWorkLog`,
  `offerSchedulingWorkLogOrDispatch`, `openBulletPropertyPicker`, `openLinkPicker`,
  `resolveLinkPickerTargets`, `FreshnessDecayCardModal`,
  `openFreshnessDecayLessOftenPicker`, `parseBulletPropertyTypedDate`.
- Property/session adapters: `createBulletPropertyItems`, counted and linked variants,
  `describeLaneRow`, `describeRefreshRow`, `describeCancelTaskRow`,
  `withDependencyPropertyPills`, and `groupLinkPickerTargetsByNote`.
- Priority preview machinery: `rollPriorityScheduledDateWithOffset`, the cached
  recommendation and counted/link batch planners, and
  `planFreshnessDecayExplicitLevelPicks` as the explicit-level reference.
- Mutation hooks: `setBulletPriorityValue`, `setCountedBulletPriorityValue`,
  `applyLinkPickerPriorityValue`, normal property set/delete hooks,
  `deleteProjectNoteScheduledValue`, `applyLaneToggleFromPicker`, refresh hooks, cancel
  hooks, and the dependency stage's existing guarded writers.
- `plugins/bob-navigation-hotkeys/styles.css`, `manifest.json`, `README.md`,
  `package.json`, and `scripts/test-navigation-*.cjs`. Test exports currently use
  `module.exports.helpers`; retain that established convention.

Relevant bob-cli files: `docs/projects.md`, `docs/task-dependencies.md`,
`docs/freshness.md`, `docs/plan.md`, `docs/plugins.md`,
`src/native/note_ready/render.rs`, and the Schedule Log glossary strand.

Two defects were independently verified during planning. `bob-cli-3z` is an existing
ready task, not a new follow-up: the actual refresh-stage options passed into the actual
`renderFooter` method reproduce a TypeError because the hint is a string. The same stage
is reused by Less often. `openLinkPicker` awaits target reads before constructing its
modal; immediate subsequent keys can therefore reach the editor. Recheck current code
and the audited bead before repairing either to avoid competing with an already-landed
fix. Do not create a duplicate task or modify bead state merely to plan this work.

Before governed changes, use `/sase_memory_read` for the sticky-lane, dependency,
freshness, approved-decay, and ledger-Today decision records. Their behavior stands. No
new CLI command, option, config.yml schema, task field, status, or build system is
introduced.

## Interaction contract

The modal opens in **card mode**, focused synchronously on its action-list control, with
Schedule selected. Every actionable key is printed next to its outcome. Card
accelerators are inactive in search, text fields, reason prompts, block-ID prompts, and
nested pickers. Search uses the existing rows, filtering, and handlers.

| Gesture on the card                            | Exact outcome                                                                                                                                                                             |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `1`–`4`                                        | Pick the configured level and its frozen displayed date; qualifying Next/Pending targets still receive the optional Work Log prompt.                                                      |
| `0`                                            | Clear priority to implicit P0; retain scheduled date and existing delete-property side effects.                                                                                           |
| `Ctrl+Enter`                                   | Apply the existing cached recommendation, including decay, terminal cancellation, and composed batch behavior. Preserve the existing Cmd+Enter alias.                                     |
| `Ctrl+R`                                       | Explicitly regenerate recommendation and strip previews; no write.                                                                                                                        |
| `Enter`                                        | Open the selected action; Schedule is the default. Selection alone never writes.                                                                                                          |
| `b`                                            | Open the existing Depends on stage, labelled Blocked by on the card.                                                                                                                      |
| `f`                                            | Open the repaired Refresh every stage, labelled Review every on the card.                                                                                                                 |
| `x`                                            | Open the existing optional cancel-reason stage. Never cancel on the card key alone.                                                                                                       |
| `Alt+N`                                        | Commit to Next or release to Ready through the existing lane flow, including the Pending Work Log prompt.                                                                                 |
| `Ctrl+D`                                       | Delete the selected row's property through the existing deletion flow. Default selection is Schedule; this is the two-gesture unschedule path. Non-property rows do not become deletable. |
| Up/Down, `Ctrl+N`/`Ctrl+P`                     | Move among action rows without writing; disabled rows remain visible and announce their reason.                                                                                           |
| `/`                                            | Open classic search with an empty query.                                                                                                                                                  |
| Any other unmodified printable character       | Open classic search, seed exactly that character, and continue normal typing.                                                                                                             |
| Backspace with an empty input, or visible Back | Discard pending, uncommitted changes and return to the card in the same session. Nonempty input edits text normally.                                                                      |
| Escape, close button, or dismissal             | Close and discard all uncommitted state. No write.                                                                                                                                        |

Keep `p`, `s`, `d`, `c`, `r`, `l`, `n`, and `o` unbound on the card so existing `pr…`,
`sc…`, `dep…`, `can…`, `ref…`, `com…`, and `rel…` filter habits remain safe. After the
first unbound character, every character, including digits and `b/f/x`, belongs to the
search input. No timer swallows typing after a committed action closes. Opening chords,
held-key repeats, and IME composition never approve a write.

There are two intentional, visible legacy differences from the research: bare Enter on
an unprioritized task opens Schedule instead of committing its lane, and Ctrl+D with
default focus now clears Schedule instead of being a no-op on the lane row. Both must be
explained in the help and pilot notes. Alt+N remains the fastest lane gesture.
Same-level `2` on P2 remains a deliberate re-pick that resets the roll streak;
Ctrl+Enter remains a roll that can advance the decay ladder.

Configured additional priority levels remain visible and reachable through taps,
navigation, and classic search. Assign unique single-digit accelerators through 9 when
applicable; never reinterpret a multi-digit sequence as a silent write. Zero is reserved
for clearing. A missing or incompatible configured property produces a disabled row with
a reason, never a guessed writer or a hard-coded default property.

## Visual design

Use a calm, compact card in the same family as the approved-decay card. Its five zones
are: a precise task header; the recommendation banner; the priority strip; the stable
action list; a small footer. A representative composition is:

```text
Write the onboarding guide                                  [close]
Work_ship · NEXT · P2 · Tue 2026-10-06 · in 3d · 1 prerequisite

Ctrl+Enter   Roll P2 → Wed 2026-10-21 · in 18d · step 1/2
             today ───── [8d ──────── ▲ ───────── 30d]   Ctrl+R

  1 P1         2 P2 •          3 P3         4 P4         0 P0
  exact date   exact date      exact date  exact date   keeps date
               re-pick · resets streak

> Enter   Schedule…                             current date
  b       Blocked by…                           1 of 1 open
  f       Review every…                         7d · default
  Alt+N   Release to Ready                       Next → Ready
  ───────────────────────────────────────────────────────────
  x       Cancel task…                          optional reason

type to search · Ctrl+D clear selected property · Esc close
```

Dates in this sketch are illustrative; actual display is always generated from the
frozen plan. Clean task text of fields, `#task`, and trailing IDs; clamp to two lines
with access to the full text. Header metadata uses the target's note and actual lane,
priority, schedule, dependency counts, and effective review interval. Missing
capabilities read as unavailable, not as fabricated values. Counted sessions show the
exact N+1 scope, clamping, mixed values, and a target disclosure. Task Links say
`via Task Link` and name target notes. Lifecycle tasks say `project`.

Recommendation banners are conditional. Distinguish roll, decay, and terminal cancel in
words; terminal cancel uses danger styling. Batches display action counts,
skipped/unavailable counts, and date spans; disclose individual effects. The timeline
uses configured bounds, the frozen date tick, and an outlined current-date tick. It does
not conceal out-of-window dates or imply a single batch date.

Priority segments show their key, configured label, and exact ISO date with weekday and
relative distance. Batches show a span plus a per-target disclosure. The current
priority is an accent ring plus text; mixed priorities are explicitly mixed. Zero says
`clear · keeps date`. Fixed action order is Schedule, Blocked by, Review every, lane,
divider, Cancel, then a More section for configured custom properties. Rows that are
unavailable stay in place with a short reason. Banner presence may change but action
rows never reorder by property definition or task state.

Add `bob-task-card-modal` overrides to this modal only: width `min(600px, 92vw)`,
content height bounded at about 75vh, scrolling within the content. Keep this width
through date, refresh, search, and review; the vault-wide dependency stage may widen.
Leave generic `bob-cnp-modal` dimensions and other picker defaults intact. Factor only
the card visual tokens into `bob-key-card-*` CSS/classes shared with the decay card;
keep source files and the plain-JavaScript build unchanged.

Use Obsidian theme variables: an approximately 12% accent-tinted recommendation with a
3px leading bar, muted secondary metadata, existing lane-chip colors, low-tint
red/orange/yellow/blue P-levels, and `--text-error` for destructive outcomes. Numeric
dates use tabular figures. Label zero-distance dates as today and past dates as overdue
with an explicit distance, using the existing warning/error tokens so the meaning never
depends on color. Use existing icons, no bitmap asset or network font. Support
light/dark themes, narrow windows, touch targets, wrapped keycaps, and
`prefers-reduced-motion`; only a short hover/focus fade is needed.

Use a labelled modal dialog with its real focus trap and a visible close button. The
action list exposes listbox/options and active selection, with shortcuts and disabled
reasons. The strip exposes the research's radiogroup/current-value semantics, keeping
keyboard focus distinct from the committed value: moving focus never writes; only an
explicit activation commits. Search has combobox/listbox relationships and
`aria-activedescendant`. Both review fields have labels. Scope character shortcuts to
the focused card widget, never a text input or the whole application. Announce loading,
errors, and updated previews without a stream of redundant announcements. Return focus
to the invoking editor or the correct surviving task location on close. These focus and
search requirements follow the
[W3C dialog pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/),
[combobox pattern](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/), and
[focused-widget character shortcuts guidance](https://www.w3.org/WAI/WCAG22/Understanding/character-key-shortcuts.html).

## Safety and preservation contract

The card model is read-only. Build it from existing property/session describers and
preview planners. Store one immutable preview set per open/re-roll generation: one
recommendation plus one independently rolled date per level per distinct target.
Injected local base date and random make tests deterministic. Paint, filter, navigate,
switch modes, and review without consuming randomness. Explicit Ctrl+R regenerates both
sets together. Stale refusal rebuilds a fresh set but requires a new approval; never
fall through to a fresh roll inside the writer.

Pass preview values into existing supported hooks: `precomputedRoll` for single targets,
`precomputedRollByLine` and the companion scheduled-value map for counted targets, and
`precomputedByPath` for linked targets. The latter calls `applyLinkPickerPriorityValue`,
not an inline property setter. Extend the existing priority Work Log adapter to accept
already-planned rolls; today it rolls again or bypasses preview injection when no Work
Log target qualifies. Test both branches. Recommendations stay separate from
explicit-level picks and retain their cached ladder behavior. Clear priority uses the
appropriate existing single/count/link delete hook and retains scheduled dates.

Use modal scope bindings and synchronous focus, with DOM-event de-duplication so one
physical key is dispatched once. Card actions share the existing single-flight guard.
Hold it through any async commit, disable repeated approval, and clear it correctly when
transitioning to an uncommitted prompt. Use a lifecycle token so delayed focus, loads,
and errors cannot resurrect a dismissed modal or act on a replaced session. Escape
cancels pending work before commitment; after a commit has started, never imply that it
rolled back. Closing a modal never cancels a writer midway through cleanup.

For Task Links, validate discoverable source context synchronously and open the same
modal's loading shell before the first await. Set the active-picker reference before
reading targets. Loading swallows accelerators, letters, navigation, paste, and repeat
events so nothing reaches Vim; Escape closes. Do not buffer approval keys for replay.
After resolving, guard the source snapshot and modal generation, install the resolved
session, compute previews, render, and focus synchronously. Failure remains visible with
close/retry and zero writes. A source change, explicit-count replacement, missing
target, or unload invalidates the old continuation. Existing direct dependency entries
and Less often still skip the card and initialize their stage before painting.

Preserve the following matrix, in classic and card entry paths:

| Domain              | Required behavior                                                                                                                                                                                                                                        |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Targeting           | Single task, plain bullet, closed task where supported, `^prj`, counted N+1 tasks with existing skips/clamps, single and counted sibling Task Links bounded by their Pomodoro. Selection/prose refusals remain.                                          |
| Priority            | Configured Tasks-native values, independent per-target rolls, deterministic existing reasons, P0 retains schedule, same-level pick resets streak, capture `p:N` parity.                                                                                  |
| Recommendations     | Same-level roll, decay, terminal cancellation, frozen dates, existing unavailable/recurring guards, batch mix, Ctrl+R behavior.                                                                                                                          |
| Schedule            | All existing presets and typed forms; future date marks Blocked and prunes today's open-Pomodoro links; historical links remain; `^prj` YAML propagation and selective unscheduling remain.                                                              |
| Logs                | Schedule/Work/Cancel grammar, ordering, indentation, opt-in markers, blank-input rules, dice/leaf reasons, and streak classification remain byte-equivalent for equivalent actions.                                                                      |
| Lane                | Sticky Next/Pending; only explicit release lowers them; live-link cleanup and optional Pending-release Work Log remain; Blocked lane action is unavailable.                                                                                              |
| Refresh             | Existing presets, default clearing, typed 1–365, lane-aware interval display, and `setRefreshLine` stamping; unavailable APIs do not invent intervals or write.                                                                                          |
| Freshness           | Existing approved stamp placement and keeps reset rules; cancellation never stamps.                                                                                                                                                                      |
| Dependencies        | Vault-wide CURRENT/RESULTS/BLOCKED search, marks, ID prompts, post-batch cycles, self/stale guards, plain managed Depends-On links, legacy folding, derived fields, recovery, and Ctrl+D removal. Counted local tasks retain add-to-all/remove-from-all. |
| Linked dependencies | Preserve the existing first-linked-task-only editor and explicitly label `first of N linked tasks`; no silent bulk dependency write.                                                                                                                     |
| Cancellation        | Reason review, recurring refusal, Cancelled symbol/date, Cancel Log, pruning and Blocked-dependent recovery. Closed targets remain unavailable where the current writer refuses them.                                                                    |
| Transactions        | One undoable local/count edit. Cross-note preflight, rollback, and cleanup warnings remain. Never promise a global Ctrl+Z from the daily note for target-note writes.                                                                                    |
| Compatibility       | Arbitrary configured list properties and delete-property flows remain in More/search; direct Depends-On-line, chip/API, palette dependency, and decay Less often entries remain direct stages. Other navigation/move pickers remain unaffected.          |

On plain bullets, Schedule, configured priority, and custom properties use existing
capabilities; lane/review/cancel dim. Dependencies remain available only where the
current property/stage can support them, including defined legacy fields; show an honest
reason for an add operation that still requires a task. No recommendation for closed
tasks. Missing freshness APIs dim Review every while existing property writes retain
their current no-stamp fallback. Use human-readable effect summaries, never raw Markdown
as the main explanation. Preserve existing date/scope receipt notices.

## Scheduling input and combined review

Extend the existing date parser conservatively, using its local-calendar arithmetic:
bare `N` means N days from the frozen base day (`0` today, `1` tomorrow); `Nd`, `Nw`,
and `Nm` work without `+`; `mon` through `sun` mean the next occurrence strictly after
today. Keep ISO, M/D and M-D, `+Nd/w/m`, presets, and existing next-year/month-end
rules. Reject invalid dates, negative/overflow offsets, and ambiguous input without
writing. Do not replace existing fuzzy preset searches with a new natural-language
parser.

Add a pure typed-schedule resolver returning `{date, reason, valid, error}`. An inline
reason is the text after a complete recognized date token and whitespace; its remainder
is kept as the reason, not parsed as more commands. Do not steal ordinary preset filter
text. Show the resolved weekday, ISO date, relative distance, and explicit year at
rollover. Freeze this parsed action before opening any review.

With the card enabled, selecting a date/preset with Enter and no supplied reason opens
one review: Reason focused, optional Work summary when any explicitly targeted
Next/Pending task qualifies, qualifying task count, and the human-readable effects.
Label the two fields distinctly. Enter commits from a single-line field; Tab reaches the
second field and controls. Back returns without writing; Escape dismisses. Use the
existing reason-normalization, eligibility, and dispatch adapters rather than another
scheduling writer.

`3 waiting on API` followed by Enter skips the reason-only question and dispatches
directly if no Work summary is applicable. With qualifying Next/Pending targets, one
Work summary review is still offered, with the supplied reason displayed. Shift+Enter
skips the reason by the existing blank-reason rule; it does not silently skip an
applicable Work Log opportunity. With no eligible Work target it commits in four
gestures: open, Enter, `3`, Shift+Enter. In the combined review, empty fields plus Enter
are the cheap explicit way to skip both. Blank reason logs `🤷 no reason given` only for
targets already keeping a Schedule Log; blank summary creates no Work Log. Per-target
opt-in/eligibility still applies in mixed and cross-note batches.

Priority picks and recommendations retain their known deterministic reason and only
offer a Work summary when eligible. Do not add a reason prompt to them. Cancellation and
Pending-lane release retain their own existing reason/Work flows. When `taskCard` is
explicitly disabled, the old date and serial-prompt behavior remains the rollback;
search reached from an enabled card uses the new scheduling affordances.

## Implementation phases and acceptance

### repair-refresh

Replace the refresh hint string with structured `{keys, label}` hints whose Escape label
says close/dismiss. Harden `renderFooter` to skip malformed hints safely while retaining
valid hints. Add a lightweight DOM/modal/scope stub harness that actually calls
`open → onOpen → renderAll → renderFooter → renderResults`; existing tests' `open()`
stub omits this and hid the crash. Put shared harness code under `scripts/` without
reorganizing production modules, and export through existing helpers only as needed.

Test the refresh presets, typed interval/default, malformed footer, and the actual Less
often picker with 90+ day starting intervals through rendering. Confirm that visible
rows match the active items and Escape writes nothing. Run targeted stamps and
decision-card tests, `npm run validate`, then `npm test`. Ship an independent patch
release (next available 1.71.x patch), update the version table, and run the required
`bob plugins sync` from the opened source before 2026-10-05. This repair must not wait
for completion of the redesign. If the deadline has passed, deploy the repair
immediately without enabling the card. Reuse/coordinate `bob-cli-3z` through the audited
bead workflow; do not start a competing task worker.

### card-model

Implement `planTaskCard(context)` and `resolveTaskCardKey(model, event)` plus a pure
preview builder with injected date/random. Produce stable row IDs, labels, availability
reasons, real scope counts, mixed metadata, recommendation/batch effects, and explicitly
typed intents. Resolve using actual configured properties and existing describers. Plan
explicit level previews separately from the recommendation. Local models need no vault
scan; linked models use already resolved targets.

Add table tests across session type × lane/status × configuration/API availability ×
key/modifiers; include same-level picks, priority clearing, unconfigured values, extra
levels, disabled actions, repeats, composition, Ctrl/Cmd recommendation aliases, search
fallback, and unrelated modified keys. Assert expected current model values against
existing planner outputs, not implementation snapshots. Prove distinct targets get
independent rolls and duplicate links share one target preview.

### card-view

Add the card renderer and mode/stage boundaries inside the current modal. Keep
`showPropertyStage` as the classic search implementation and add `showTaskCard` plus an
explicit search transition. Render the specified header, banner/timeline, strip, stable
rows, disabled reasons, More properties, footer, disclosures, close/Back controls, and
empty/error states. Use pure model data; no render-time roll or write. Card mode is
opt-in internally until the dispatch phase connects it.

Apply scoped compact styles across this modal's flow, widen only dependencies, and share
limited visual tokens with the decay card. Render every state through the harness,
including no recommendation, cancellation recommendation, mixed batches, long task text,
missing APIs, custom properties, and narrow layouts. Confirm generic child-note,
task-move, and Pomodoro pickers keep their existing classes and dimensions.

### card-dispatch

Connect the complete key map and taps to existing methods and writers with frozen
options. Extend `maybeOfferPriorityWorkLog` or its immediate dispatch adapter to accept
precomputed rolls in every eligibility branch; validate actual written dates/logs
against displayed values. Wire priority clearing and row-property deletion across
single/count/link/`^prj` paths without a new mutation implementation. Preserve
dependency direct-entry and custom-property paths.

Implement synchronous focus, modal scope dispatch, event de-duplication, input guards,
single-flight commits, Back semantics, and the async Task Link shell in the existing
modal. Test immediate `2` after `open()` without flushing timers; delayed target reads
with `x` cannot reach an editor spy; no pending digit is replayed; close/reopen, counted
replacement, source-change refusal, resolution failure, and unmount do not revive old
sessions. Simulate double key/click, held repeats, and composition without duplicate
writes. Prove old filter strings still reach their original stages.

Introduce a local plugin `taskCard` preference via Obsidian `loadData/saveData` and a
small settings control. Existing onload has no persisted settings: initialize command
registration and picker guards safely, fail closed to classic until settings load,
preserve unrelated saved keys, and handle unreadable data. At this phase absent means
classic/off. Explicit true opts into the pilot; false forces classic. Do not touch
hotkeys.json or bob config.yml. Run the relevant navigation/dependency/stamps/roll tests
and synchronize from source with the flag off.

### schedule-input

Add the conservative grammar and typed reason resolver, dynamic preview row, inline
reason dispatch, and Shift+Enter blank-reason path. Scope changes to enabled Task Card
flows. Retain existing preset/filter behavior, pinned roll, Ctrl+R, and Ctrl/Cmd+Enter.
Unit tests cover `0`, `1`, `3 reason`, unsigned and signed units, weekdays on the same
weekday, invalid/leap dates, end-of-month offsets, December rollover, DST in a real
local timezone, malformed/overflow input, and full reason preservation. Exercise the
DOM/input path with exact preview and committed result, including mixed Task Links.

### schedule-review

Add one uncommitted review state that composes existing schedule reason and Work Log
payloads. Route bare dates, presets, inline reasons, and reason-skip intents through it
when needed; priority/recommendation and lane/cancel retain their appropriate existing
prompt semantics. Deduplicate eligible Work targets by note plus identity, not line
number alone. Freeze the callback and target snapshot, then pass both log payloads
through the current mutation adapter once.

Test no-log/existing-log targets, blank/typed reasons, blank/typed summary, Ready vs
Next/Pending, projects with propagated children, counted mixed targets, linked notes
with equal line numbers, back/escape at both fields, and a target changed during review.
Assert date, log grammar, eligibility, pruning, stamp, and transaction count against
existing equivalent paths; nothing is written before confirmation.

### parity-rollout

Expand end-to-end modal tests to cover the preservation matrix, entering every existing
stage through renderAll at least once: card, search, date, configured priority/list,
refresh, dependencies, block ID, cancel reason, lane release, legacy schedule/work
prompts, and combined review. Retain all existing test suites; adjust only deliberate
stage-one/default-mode expectations and add explicit classic-path coverage. Include
cross-note stale/preflight failure and rollback/cleanup receipt tests; do not infer
global undo from a stub.

Run targeted new suites, `npm run validate`, and `npm test` once after integration;
repeat only for further changes/failures. Exercise the installed panel in real Obsidian
if an interactive environment is available. Record screenshots of light/dark/narrow card
and combined review, and timings from real vault use: useful warm paint about 100ms or
less, search update about 50ms or less. Model paint is bounded by configured levels ×
selected targets and performs no vault-wide scan. If GUI access is absent, record that
limitation clearly and supply a precise manual acceptance checklist; headless harness
results are not a claim of visual verification. Use disposable test notes for mutation
checks and remove them after verifying.

Align the decay card with `x` as an alias for existing `d` Drop, retaining `d` and all
existing shortcuts; add its visible key hint at the default rollout boundary. The
optional `f` alias is omitted in v1. Enter in the decay card still never cancels.

Release completed navigation-hotkeys as **2.0.0** (or the next compatible major if
upstream advanced), updating manifest/version docs. Protect the trial with a pure
default resolver: absent/null `taskCard` is false through local 2026-10-18 and true from
local 2026-10-19, evaluated only when a new panel opens. Explicit false always forces
classic; explicit true remains the user's pilot opt-in. The settings control offers
Automatic, Task Card, and Classic list and explains the activation date. Do not persist
a calculated default as false, which would prevent the later rollout; do not use a timer
or background vault mutation. Add boundary tests for October 18/19, loaded/failed
settings, explicit overrides, and an already-open modal. This delivers the agreed
default-off trial and later default-on migration without a sleeping phase or an
automatic UI change midway through a session. Keep the rollback setting for at least one
to two weeks after activation; do not remove it in this epic.

Pilot measurement is a simple 20–30-use manual tally: gestures per outcome, accidental
actions, lane-Enter habit changes, and whether reason/summary remain distinct. No
telemetry. Common acceptance targets: open + digit = 2 gestures on Ready, open + digit
plus blank Work summary confirmation = 3 on qualifying lanes, cancel = 3, dependencies =
2, counted four-task P2 = 3 including the count, explicit unschedule = 2. Preserve
Ctrl+Enter at 2. Investigate any failed parity or accidental-write case before enabling
the default. Real GUI approval problems can hold activation behind explicit false.

### docs-hints

Update bob-plugins README with the concise key table, screenshots when available,
scope/preview explanations, classic search, custom properties, same-level re-pick,
blank-log behavior, linked undo limits, deliberate Enter/Ctrl+D changes, pilot setting,
and the October 19 default boundary.

Update bob-cli `docs/projects.md`, `docs/task-dependencies.md` §6.1, `docs/freshness.md`
review outcomes, and `docs/plan.md` to document the card accelerators alongside classic
paths. Make `src/native/note_ready/render.rs` hint copy date-aware with the same October
19 boundary: before then keep the current generic sequence / defer / drop hint;
afterwards advertise `defer Ctrl+Shift+P 1–4`, `drop Ctrl+Shift+P x`,
`sequence Ctrl+Shift+P b`, plus search compatibility. Use the already-resolved scan day
where available, and inject the day into hint formatting for deterministic tests; do not
add a CLI option or make headless bob read plugin data.json. Explain that explicitly
choosing Classic uses the documented search path. Update relevant renderer assertions
and run the focused Rust/unit and note-ready CLI tests; run `cargo fmt --check` for Rust
edits.

The user endorsed the research's explicit Schedule Log glossary update. Under
`/sase_memory_write`, revise only `sase/memory/glossary/schedule-log.md` to name the
Task Card priority shortcuts, explicit Schedule path, and unchanged log semantics
alongside capture p:N; keep it short and link existing vocabulary where relevant. Run
`sase memory init` for generated instructions/shims; never hand-edit them or accepted
decision records. No other memory changes are included.

After any bob-plugins edits, its AGENTS requires synchronization. From the opened source
checkout use the existing `bob plugins sync --repo <plugins-root> --no-pull` with a dry
run first, then the real sync; repeat at the patch and final release boundaries as
appropriate. Verify installed version/file parity and reload the plugin if GUI access
exists. If a phase runs where the real vault is unavailable, test against a disposable
vault via existing --bob-dir and clearly record real deployment as outstanding; do not
silently claim it happened. Source changes in opened repositories must be declared
through `/sase_final`, not manual git commits.

## Completion and exclusions

The epic is complete when the refresh repair is deployed independently, the new front
door and scheduling review pass the existing and new parity suite, the compact design is
reviewable, default activation cannot precede October 19, and source/vault deployment
and docs accurately describe the shipped behavior. Record any genuine GUI/deployment
limitation and retain classic mode if acceptance is still outstanding.

No greenfield panel/plugin, global hotkey expansion, production module split, raw
Markdown diff UI, task-state redesign, note relocation, new tracking fields, auto review
stamps on open, or removed log prompt is part of this work. Search value results (`p2`,
`tomorrow`, `every 14`) and true bulk linked-dependency editing are the research's
optional future enhancements and remain outside this epic. Phase workers record
discovered unrelated work on their phase as PROPOSED FOLLOW-UP rather than silently
growing the redesign.
