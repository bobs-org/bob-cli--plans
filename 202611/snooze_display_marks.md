---
tier: epic
title: Snooze display marks and plan chip
goal: 'Complete bob-cli-q5.5 with live snoozed definition marks, woken-link pills,
  Snoozed row and heading paint, an independent plan chip, and verified light/dark
  Obsidian evidence without changing note bytes or Today semantics.

  '
phases:
- id: display-contract
  title: Snooze display contract and canonical conformance vectors
  size: small
  description: 'display-contract: publish docs/snooze-marks.md and machine-readable
    SM vectors in bob-cli before plugin implementation.'
  depends_on: []
- id: definition-marks
  title: Snoozed task definition marks and refresh lifecycle
  size: medium
  description: 'definition-marks: implement alarm-clock definition marks, shared display
    helpers, toggles, navigation, and current-state refresh in Live Preview and Reading
    view.'
  depends_on:
  - display-contract
- id: woken-pills
  title: Woken Task Link pills with history navigation
  size: medium
  description: 'woken-pills: render same-note wake history before eligible open ledger
    links, with fresh and settled styling, exact identity resolution, and history-row
    navigation.'
  depends_on:
  - definition-marks
- id: snoozed-row-paint
  title: Snoozed rows and heading summaries
  size: medium
  description: 'snoozed-row-paint: paint active, due, blocked, ignored, and struck
    Snoozed rows plus heading summaries in Live Preview and Reading view.'
  depends_on:
  - woken-pills
- id: plan-chip
  title: Separate snoozed meter and wake forecast in bob-plan
  size: medium
  description: 'plan-chip: add the independent snoozed chip beside TODAY, its section
    jump, and the effective-clock wake forecast without changing ledger budget status.'
  depends_on:
  - snoozed-row-paint
- id: visual-proof
  title: Obsidian display acceptance and screenshot evidence
  size: medium
  description: 'visual-proof: verify the complete display in real Obsidian, capture
    light and dark states and accessibility evidence, fix display integration defects,
    and attach results for the land agent.'
  depends_on:
  - plan-chip
proposed_by: bbugyi200.athena.bob-cli-q5.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.5
create_time: 2026-11-26 15:37:05
status: done
bead_id: bob-cli-q5.5.1
---

- **PROMPT:** [prompts/202611/snooze_display_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snooze_display_marks.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.5.1.md)

# Snooze display marks for bob-cli-q5.5

## Outcome and completion target

Complete the display-only `plugin-marks` scope of `bob-cli-q5.5`, from
`plan:202611/snoozed_tasks.md`: an alarm-clock mark on a snoozed task definition, a
woke-time pill on its woken ledger link, paint for Snoozed rows and their heading, and a
separate snoozed chip beside TODAY. Both Live Preview and Reading view must stay current
without modifying note bytes.

This is a child epic of the assigned phase **bob-cli-q5.5**, not a replacement for the
outer epic. Its land agent verifies the whole result and closes bob-cli-q5.5 with the
verification note. Child phase workers close only their own assigned beads; they never
close bob-cli-q5.5, bob-cli-q5, or another ancestor. The land agent must not close
bob-cli-q5. The host handles the child plan's lifecycle.

## Evidence and sizing

The assigned bead is `in_progress`, has no earlier work notes, and depends on the closed
plugin-plumbing phase. No reopening or manual status change is needed. The initial
`sase bead epic-symbols bob-cli-q5.5` check found no entries.

Checked landed work before planning:

- bob-plugins `d7875aac` lands snooze plumbing; `0bae8384` fixes Done recovery. The
  cache, shared parser/classifier, planned-today exclusions, and Obsidian waker already
  exist. Do not reimplement those phases.
- `plugins/bob-ledger-tools/src/290a-snoozed-today-cache.js` supplies `rows`, `byKey`,
  `planned`, `blocked`, `nextWakeAt`, readiness, and minute refresh. Active snapshots
  carry resolved keys, source lines, target times, and state; history snapshots carry
  target spelling, block id, and `wokeAt`, but no resolved key. History identity needs
  resolution before joining to a ledger link.
- The Today mark implementations still only render tomatoes. The existing `275`, `276`,
  `290a`, and `340` fragments are respectively 957, 994, 936, and 958 lines. New sibling
  fragments and small integration hooks are needed to respect the repository's
  1,000-line limit.
- `planBlockModel` already accepts `isSnoozed` for exclusions, but emits no snoozed
  meter. Its rendered block is in `290-plugin-today-and-location.js`, not just
  `340-dashboard-views-and-plan-block.js`.
- `docs/snooze-marks.md` is absent and has no landed history. The startup smoke runner
  exercises editor mounts; it does not already provide snooze state screenshots or the
  required Reading-view acceptance matrix.

The four rendering surfaces, source-to-DOM identity checks, lifecycle integration, and
real-Obsidian visual proof exceed one medium implementation turn. Each phase below is
bounded to one coding agent. Dependencies are intentionally sequential: all plugin
phases touch the same bundle manifest, lifecycle hooks, styles, test registry, and
deployment. This avoids concurrent deployment and integration races.

## Governing contracts and fixed choices

Read the parent plan through `sase artifact read` and read the assigned bead to obtain
its final decisions. These remain final: `uncommitted_rows=ignore`, `wake_notice=hud`,
`hold_in_meetings=yes`, `glossary_snoozed_task=yes`, and `decision_record=no`. This
child epic needs no memory edits; the authorized glossary work belongs to the outer
close-out phase. Do not create or edit a decision record.

The governing decisions are:

- `decisions:today-is-read-from-the-ledger`: Today remains dedicated links under today's
  open Pomodoros. Snooze is not another Today membership source.
- `decisions:task-lanes-are-sticky`: display and navigation never change lanes.
- `decisions:ready-is-freshness-gated`: use the landed shared classification and
  exclusion rules; do not add a second freshness or review evaluator.
- `decisions:off-clock-pomodoros-count-apart`,
  `decisions:off-clock-follows-tag-then-note-default`, and
  `decisions:off-clock-note-defaults-filter-by-task-type`: the wake forecast uses the
  existing effective-clock resolver, including explicit tags and inherited type-filtered
  defaults; recorded time remains unchanged.

The current `docs/snooze.md`, `docs/plan.md#snoozed-forecast`, `docs/today-marks.md`,
and `docs/event-link-times.md` supply the detailed contracts. The snoozed meter counts
unique Asleep/Due/Blocked identities; only on-clock Asleep/Due identities join the
forecast. Blocked is visible but not forecast as waking. The next time comes from live
Asleep/Due rows, never Blocked or ignored.

No CLI, wake writer, notification policy, snooze card, Mac app, lane semantics, live
dashboard queries, release versions, or changelog changes belong here. Do not add Rust
behavior just to support plugin display.

## Common implementation rules

Open bob-plugins with `/sase_repo`; use only the printed checkout and read its
`AGENTS.md`. Paths below are repository-relative. `LT` means
`bob-plugins/plugins/bob-ledger-tools/src/`.

Edit source fragments and `fragments.json`, then regenerate `main.js` with
`npm run build`; never edit the generated bundle. Keep every hand-edited fragment at or
below 1,000 lines. Add tests to `package.json`'s `test:unit` list. New helpers may be
private or exported to the test harness; do not invent a public namespace.

Reuse the shared Snoozed parser and classifier. A display model takes explicit clock,
document, resolved identity, and task facts. It does not execute a writer. The current
editor document is authoritative in Live Preview, including unsaved edits. Reading view
uses proven source mapping and the established source caches; no synchronous file read
or subprocess is allowed during render. Missing facts, stale caches, ambiguous identity,
or an uncertain DOM mapping yield neutral paint.

Use fresh sibling fragments for each surface, with tiny hooks in existing near-limit
files. Reuse the Today source-line mapping and Event-link placement patterns instead of
heuristically matching rendered titles or aliased link text. Scan once per document
version, resolve each distinct path once per build, and decorate only visible editor
ranges. Do not rebuild on selection alone.

Use one coalesced refresh path from the landed snooze cache and minute lifecycle. It
must cover countdown changes, Asleep-to-Due, retiming with unchanged membership, Blocked
transitions, wake-history changes, 60-minute pill expiry, midnight, source-cache fills,
metadata updates, rename/delete, toggle changes, and popouts. The existing cache's
membership-only plan rerender is insufficient for all of these changes. Do not spawn
another waker or install redundant polling timers. Dispose listeners and timers on
unload.

Every tooltip uses accessible text and never contains `::`. Alarm shape and visible
words distinguish states independently of color. No mark changes line height, note
bytes, copy/yank output, or document positions. No unnecessary tab stops or live
announcements. Reduced motion removes animations; forced colors retain a visible glyph,
outline, and readable state text.

## Phase display-contract

Work only in bob-cli so the display contract lands before plugin code.

1. Write `docs/snooze-marks.md`, modeled on Today marks and Event link times. Cover
   principles, eligibility, placement, CSS anatomy, exact visible strings, tooltips,
   navigation, liveness, toggles, historical notes, and performance. Link it from
   `docs/snooze.md` and the appropriate existing display-contract nav lists in
   `mkdocs.yml`; keep the existing published paths.
2. Add canonical `tests/fixtures/snooze/marks.json` with stable `SM-*` identifiers and
   explicit clock/date, source, task/resolver facts, and expected display models. The
   contract documents each vector and its expected result; the JSON is its executable
   data, not a second independently authored contract.
3. Partition vectors into `definition`, `woken`, `rows`, and `plan` groups. Later plugin
   phases copy the file byte-for-byte under the ledger-tools fixture directory and
   execute their group's inputs and expected values verbatim. Follow the existing
   canonical-fixture parity pattern using `BOB_CLI_DIR`; no test should silently
   substitute simplified local data.
4. Specify the details below so later phases share one answer:
   - Today's definition precedence is Today tomato, then planned Snoozed clock, then
     absent. Blocked clocks are muted. Ignored rows never create a clock.
   - Definition clocks honor both the Today-marks toggle and the new session-only
     `toggle-snooze-marks` toggle, initially enabled. The latter also gates the
     decorative daily row and woken-link paint. The plan meter remains available because
     it reports counts; source text is readable with all paint off.
   - Due definitions visibly say `due`; due rows say `due · waking`. Blocked rows say
     `waits on prerequisites`. History stays dim. Ready rows say `Ready — won't wake`;
     other ignored states have short, accurate reasons.
   - Fresh wake styling is today's `0 <= age < 60 minutes`; exactly 60 minutes settles
     to outline. Negative age is neutral. Earlier daily notes retain a quiet history
     pill under eligible open entries, never a fresh accent or live countdown; no old
     note contributes to today's marks or counters.
   - Match wake history by resolved path/block identity. Prefer the latest valid wake
     time, then later source order on ties, matching the Rust agenda's
     `attach_snooze_history` selection. Unresolved/ambiguous targets are neutral.
   - Summary counts distinguish asleep, due, and blocked. Omit zero subcounts; omit
     `next` when no live row exists. An empty planned set has no plan chip. A section
     containing only ignored/history rows gets no live heading summary.
5. Include vectors for aliases/local links, duplicate IDs/rows, inline and fenced
   examples, missing source, already-linked precedence, stale date/cache, closed
   targets/entries, re-snooze removing history, all toggle combinations, boundary times,
   blocked-only and off-clock-only queues, and forecast at/over cap.

Read `docs.md` through `/sase_memory_read` before editing docs. Run `just docs-check`
and check JSON validity and unique vector IDs. Do not add tests that merely mirror the
doc edit. Close only this child phase after its contract is ready to land.

## Phase definition-marks

Implement shared presentation helpers and the definition surface in bob-plugins.

- Add small snooze display/model and runtime fragments. Copy the canonical `marks.json`,
  add its byte-parity test, and run the `definition` vectors.
- Extend the existing Today candidate/source mapping at `LT/143`, `275`, `276`, and
  `277` through shared helpers or thin sibling adapters. Retain all guards: non-daily
  hosts only, exact task token and trailing unique ID, no code/fences, no Tasks-query
  markup or unproven embed mapping, open status, current cache. A snoozed-only task must
  work when the Today cache has no IDs for that note.
- Build `span.bob-snooze-mark`: Lucide `alarm-clock` through CSS mask variable
  `--bob-snooze-mark-glyph`, `currentColor`, and tabular time through
  `::after`/`attr(data-bob-snooze-time)`. Asleep is indigo from blue/purple; Due is
  amber with one gentle transition pulse; Blocked is muted. Do not replay the pulse on
  scrolling/rerender/startup. The clock disappears when Today wins; the existing tomato
  entrance behavior remains the 600 ms entry window.
- Share the existing mark position (`side: -1`) and identity-slot placement. No
  simultaneous clock and tomato, including during cache transitions.
- Add the snooze toggle command and mirror its gate to all windows. Preserve existing
  Today toggle and tomato behavior. Plain clicks keep ordinary editor placement;
  Mod+primary click jumps to the current Snoozed row. Revalidate the target against the
  open daily note, rescan after line moves, and show the established kind of teaching
  Notice if the row vanished; never jump to a stale unrelated line. Reuse an existing
  daily leaf and guard repeated jumps.
- Install the common refresh hooks described above, including updates when only row
  details change. Provide a reusable row/section jump helper for later phases.
- Extend the existing Today Live Preview and Reading tests to prove no regressions in
  daily-note exclusions, popouts, toggle restoration, and copy-safe DOM. Add tests for
  clock-to-tomato transition and minute/date changes.

Document the toggle in the plugin reference and regenerate the inventory as required.
Run the common plugin gates and deploy below.

## Phase woken-pills

Implement the ledger surface beside the Event-link-time machinery (`LT/291d`, `291e`,
`291f`), in separate small fragments.

- Parse that daily note's real `## Snoozed` range with the existing section finder and
  shared row parser. Resolve history targets relative to that daily note; do not assume
  history snapshots already have keys or compare raw link spelling.
- Join valid struck history to dedicated, unstruck Task Links under **open** Pomodoro
  entries, with open resolved non-Event targets. Use the established ledger parser:
  mixed prose, nested note links, fenced rows, closed entries, missing/ambiguous
  targets, and Event Tasks are ineligible.
- Insert a zero-width widget at `side: -3`, ahead of type/progress marks; do the same
  order in Reading view regardless of postprocessor order. Add its class to
  `ledgerMarkClass` in `LT/269` so ledger mark rebuilds preserve it.
- Paint `⏰ woke 15:32` using the shared alarm glyph: warm amber-gold for the first 60
  minutes, quiet outline thereafter. The tooltip gives target and actual time and exact
  early/on-time/late meaning. Mod+click jumps to the matching struck row; plain click
  preserves editor behavior.
- Use the common refresh/toggle lifecycle. Re-snoozing, disappearing history, closing a
  link/entry, or completing its task removes an ineligible pill.
- Run the `woken` vectors and interaction/source-mapping tests, including 59/60 minutes,
  same-block IDs in different notes, aliases, history ties, older days, and coexistence
  with Event, off-clock, type, and progress marks.

Run the common plugin gates and deploy.

## Phase snoozed-row-paint

Implement daily Snoozed row and heading paint in separate model/editor/reading
fragments, reusing the shared classification and display helpers.

- Paint the structural H2's own rows, with exact row source mapping. Retain notes
  beneath rows and the underlying Markdown. Do not paint a similarly named heading in a
  fence, an indented child example, or another section.
- Asleep gets an indigo time pill and faint live countdown. Due gets amber
  `due · waking`. Blocked is muted with `waits on prerequisites`. Struck rows are dim;
  ignored rows show the state's muted reason. Malformed/duplicate-header uncertainty
  must not invent an actionable time or membership.
- Add the heading summary, e.g. `2 asleep · next 19:00`, with distinct due and blocked
  counts when applicable. Use unique planned identities, and do not count a duplicate
  loser or history. Earlier daily notes get neutral historical paint, no live countdown
  or today-state heading summary.
- Update countdowns on the minute even when no row changes state. Both editor documents
  and Reading sections refresh when unsaved source, resolver facts, task status, time,
  toggles, or date changes. Remove stale hosts when rows or the heading disappear. Do
  not mutate Markdown to decorate it.
- Run the `rows` vectors and DOM/editor tests for subsection boundaries, child notes,
  every ignored reason, duplicates, midnight, no section, history-only sections, and
  Reading-view idempotence.

Run the common plugin gates and deploy.

## Phase plan-chip

Extend `planBlockModel` and the actual block renderer in
`LT/290-plugin-today-and-location.js`, extracting a small helper/mixin as necessary.

- Add a separate snoozed model from the current classified queue and render it
  immediately beside TODAY: `⏰ 2 · 19:00`. Omit the time when only Blocked rows remain.
  Use indigo, or amber when at least one row is Due. It never inherits `workToneClass`,
  over-cap styling, or TODAY's status color.
- The accessible tooltip reads, for example,
  `2 snoozed · next wakes 19:00 · not counted in TODAY · 11/13 when all wake`. Add
  due/blocked words when needed; never claim a next wake for blocked-only queues. Click
  jumps to today's `## Snoozed` using the common section jump.
- Compute forecast from the current on-clock ledger count plus unique eligible
  Asleep/Due on-clock targets. Reuse the same effective-clock helper as
  `181-plugin-plan-budget-off-clock.js`; Blocked, ignored, history, duplicates,
  already-linked, and off-clock rows are excluded from the addition. Do not alter
  `budget.status`, `links`, TODAY, lane counts, or wake permission.
- Queue-only notes without a Pomodoros section still show a snoozed chip. Missing
  task/cache data stays neutral. An empty queue removes the chip immediately. Retiming,
  minute transitions, and inherited clock/type changes rerender it even when queue
  membership is unchanged. Existing day-strip plan hosts retain their lifecycle and
  layout.
- Run the `plan` vectors plus existing plan-budget, off-clock, dashboard-parity,
  planned-today, and day-strip plan-host tests. Include exactly-at-cap, over-cap,
  missing-budget, blocked-only, off-clock-only, and duplicate-identity cases.

Run the common plugin gates and deploy.

## Phase visual-proof

Use real Obsidian in an isolated scratch vault/profile on the available desktop host.
Reuse the repository's owned-process/CDP and fixture-runner patterns; the generic
`npm run smoke:obsidian` is a startup check, not snooze visual evidence. If needed, add
a focused `scripts/obsidian-snooze-marks-e2e.cjs` runner rather than a new
general-purpose test framework. Use synthetic notes, an injected fixed clock, and a stub
wake command so the fixture cannot mutate the real vault.

Capture and inspect light and dark screenshots for both Live Preview and Reading view:
Asleep, Due, Blocked, ignored Ready, struck history, fresh and settled woken pills,
heading summaries, and the plan chip. A contact sheet is acceptable only if each state
and surface remains clearly readable. Include normal Today tomato and an Event-link-time
row as regression controls.

Verify with runtime assertions and screenshots:

- mark and unmarked baselines have the same line height; narrow layouts and long titles
  stay readable; editor and Reading placement agree;
- reduced motion has no transition/pulse and forced colors preserve readable shape and
  text; inspect actual tooltips and ensure no `::`;
- both toggles work in main and popout windows; source mode and copy/yank retain the
  original Markdown; snapshot note hashes before/after passive rendering;
- Mod+click reaches the correct active/history row after source lines move; plain clicks
  do not navigate; the plan chip reaches the heading;
- retiming without membership change, Asleep-to-Due, wake to tomato, fresh-to- settled,
  deletion, and midnight update without reopening the note;
- no renderer exceptions, duplicated hosts, Event pill regressions, or stray
  writer/subprocess calls from display code.

Fix defects in these display integrations and rerun the affected checks. Do not replace
unavailable visual evidence with mocked-DOM or offline smoke claims. Record exact
runtime/build identity and limitations. Attach the screenshots and report using
`sase bead attach` to this phase, and add their durable references to bob-cli-q5.5. Read
`sase_artifacts.md` before managing evidence references. Run all final gates below, then
deploy the verified plugin bundle.

## Gates, deployment, and landing

For each plugin implementation phase, regenerate the build/inventory as needed and run
`npm run validate`, `npm test`, and `npm run docs:check`, with `BOB_CLI_DIR` pointing to
the current bob-cli workspace. Run `just docs-check` in bob-cli for the new display
contract and at final integration. Existing canonical Snoozed behavior vectors must
remain unchanged and pass. Use `/sase_monitor` for long commands, wait for the staging
command to exit, and finish the turn through `/sase_final`; never leave a live
subprocess behind.

After changes to bob-plugins, run `bob plugins sync` as its AGENTS.md requires. Follow
`docs/develop/deployment.md` if another checkout's deployment is protected: fetch and
inspect the named landed work; never use `-F` to bypass it. The contract phase's bob-cli
commit must land before the first plugin implementation. Do not manually commit; use the
required SASE finalizer declarations for every changed repository. Do not bump bob-cli
release versions or edit CHANGELOG.md.

The child epic's land agent reviews all SM groups, visual attachments, gate results, and
deployment evidence against the complete parent plugin-marks scope. Before closing
bob-cli-q5.5, run `sase bead epic-symbols bob-cli-q5.5`; resolve each entry or re-key it
to a still-open owning bead. Then close only bob-cli-q5.5 with
`sase bead close bob-cli-q5.5 --note "<verified scope, checks, evidence, deployment>"`.
Do not close bob-cli-q5 or any outer plan bead.

Workers do not create follow-up beads. Record discoveries as
`PROPOSED FOLLOW-UP: <summary — detail>` notes on their own assigned phase; the land
agent collects relevant references on bob-cli-q5.5 for the outer land agent. A failure
that reproduces identically on a clean base is recorded with its base SHA, command, and
output (and an existing tracking bead if known); it does not leave a completed phase
open. New failures and missing required display behavior must be fixed before claiming
completion.
