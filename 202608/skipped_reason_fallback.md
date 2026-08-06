---
tier: tale
title: Log an unexplained entry when a task already keeps a schedule log
goal:
  Skipping the `Ctrl+Shift+P` reason prompt on a task that already has a `🗓️ **SCHEDULE LOG**` records a `🤷 no reason
  given` entry so the history has no gaps, while a task with no log is still left untouched and a skipped prompt on an
  unchanged date still writes nothing.
proposed_by: bbugyi200.athena.tu.f0.f0.f1
create_time: 2026-08-06 09:38:23
status: done
---

- **PROMPT:**
  [prompts/202608/skipped_reason_fallback.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/skipped_reason_fallback.md)
- **AGENTS:**
  - [bbugyi200.athena.tu.f0.f0.f1](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.tu.f0.f0.f1.md)
- **COMMITS:**
  - [1830491](https://github.com/bobs-org/bob-plugins/commit/1830491d242fb488afa58a8b0307fd314be901a2) —
    feat(bob-navigation-hotkeys): log an unexplained entry when skipping the reason prompt on a task with a log

# Plan: Log an unexplained entry when a task already keeps a schedule log

## The ask

Today, pressing `↵` on an empty reason prompt writes the date and nothing else — no entry, no marker. That escape hatch
was designed for a task with no log yet, and for that task it is still right. But when the task **already** carries a
`🗓️ **SCHEDULE LOG**`, skipping the prompt silently punches a hole in a history the task is otherwise keeping: the date
moved, the log says nothing, and six weeks later the top entry is a lie about why the task sits where it does.

So: when the marker already exists, an empty reason writes a deterministic entry instead of writing nothing.

## Design

### The rule

> **Once a task keeps a schedule log, every scheduled change the picker makes is recorded in it — including the ones the
> user declined to explain. A task with no log is never given one by an empty reason.**

That is the whole feature, and it composes cleanly with what already ships:

| Gesture on a `scheduled` date          | Task has no log                | Task already has a log                  |
| -------------------------------------- | ------------------------------ | --------------------------------------- |
| Reason typed                           | creates the marker + the entry | prepends the entry                      |
| Reason skipped (`↵` on an empty input) | **nothing written** (as today) | **prepends an unexplained entry** (new) |
| `Esc`                                  | nothing written, date included | nothing written, date included          |
| Priority level / pinned roll row       | creates the marker + 🎲 entry  | prepends the 🎲 entry                   |

The marker is the opt-in switch. You opt in the first time you type a reason (or the first time a roll writes one), and
from then on the log is complete. Nothing new to learn, nothing to configure, and the existing escape hatch is untouched
for anyone who never wanted a log on that task.

### The deterministic text

```markdown
- [?] #task Ship the thing [scheduled:: 2026-08-27] ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-20 → 2026-08-27_ — 🤷 no reason given
    - _2026-08-13 → 2026-08-20_ — waiting on the API review to land
```

`🤷 no reason given`. Three decisions behind it:

- **A leading emoji already means "the plugin wrote this text, nobody typed it."** `🎲` marks a date the software
  rolled; `🤷` marks a date the user chose and declined to explain. One glance down the log separates human sentences
  from machine ones, and the two machine kinds from each other.
- **It states exactly what happened and claims nothing more.** The software does not know why the date moved, so it does
  not guess, does not restate the dates that are already in the same line, and does not editorialize. "No reason given"
  is the honest, complete fact.
- **It is short.** The entry's value is the date span; the reason column just has to not be empty.

Rejected: `· deferred 14 days`-style tails. The `🎲` entry earns its `· random in 8–30 days` tail because the roll
window is config state you cannot recover from the note. A day-count tail is arithmetic on two dates printed two words
to its left, so it is noise.

`🤷` is `U+1F937`, a single code point — no variation selector to preserve (unlike `🗓️`) and no gendered ZWJ sequence.

### An unexplained entry never claims a change that did not happen

`planScheduleLogEntry` already has the rule for machine-written text, in `shouldWriteAutomaticScheduleLog`: a rolled
date that landed on the date the task already had writes nothing, because `*2026-08-20 → 2026-08-20* — 🎲 …` asserts a
move that never occurred. The skipped-prompt text is machine-written, so it obeys the same rule: **picking the current
date and skipping the prompt writes no entry, even on a task that keeps a log.**

A _typed_ reason on an unchanged date is still written, exactly as `docs/projects.md` already promises. A human sentence
is a decision worth keeping; a generated one is not.

### Where the decision is made

The check is "does this task have a marker?", which only `planScheduleLogEntry` is in a position to answer — it is the
one place that already calls `findScheduleLogParent`, and in a counted session it is called once per target so each task
decides for itself. So the payload gains one field and the pure planner grows the branch; the three writers change only
their pre-flight guard and their notice text.

`options.scheduleLog.fallbackReason` — reason text used **only** when `reason` is empty **and** the task already has a
marker **and** the date actually moved. The modal always supplies it; the roll paths never do (their `reason` is never
empty), so the automatic path is completely unaffected.

### What the prompt shows

The reason stage's contract is that the single preview row is a literal picture of what `↵` will write. That row now has
three states instead of two, and the footer hint follows it exactly:

| Input                            | Row                                                                  | Footer                   |
| -------------------------------- | -------------------------------------------------------------------- | ------------------------ |
| text typed                       | `*2026-08-20 → 2026-08-27* — waiting on the review` + where it lands | `↵ Log reason`           |
| empty, task has no log           | `No reason` · `scheduled → 2026-08-27 only; no schedule log entry`   | `↵ Skip reason`          |
| empty, task has a log            | `*2026-08-20 → 2026-08-27* — 🤷 no reason given` + where it lands    | `↵ Log without a reason` |
| empty, task has a log, same date | `No reason` · `scheduled → 2026-08-20 only; no schedule log entry`   | `↵ Skip reason`          |

The third row is styled as a new `is-fallback` state: muted icon and title like the inert `No reason` row, but carrying
the real preview box. Subdued because you gave nothing; visible because something will still be written. The subtitle's
trailing `· nothing written yet` clause is unchanged and still true.

In a counted session the cursor task is only a sample of the batch, so the empty row previews the fallback form and its
preview line speaks about the batch (`Appends to every counted task that already has a 🗓️ **SCHEDULE LOG**`); each
target is still decided independently at write time.

## Repos touched

- **`bob-plugins`** (linked repo — open with `/sase_repo` first and use the printed path as the only path): all plugin
  code, styles, tests, README, and manifest changes. Everything under `plugins/bob-navigation-hotkeys/` and
  `scripts/test-navigation-hotkeys.cjs`.
- **`bob-cli`** (your own workspace checkout): `docs/projects.md` only.

Do not edit plugin files under `~/bob/.obsidian/plugins/`; they are overwritten by `bob plugins sync`. No vault content
migration: this feature only changes what future writes produce.

## Implementation

Line numbers are as of `bob-navigation-hotkeys` 1.19.0 and drift as you edit — find the symbols by name.

### 1. Constants (`main.js`, after `IMPLICIT_PRIORITY_LEVEL_LABEL`, ~line 267)

```js
// A skipped reason prompt still records the change on a task that already keeps
// a log: a gap in a history the task maintains is worse than an unexplained
// entry. The shrug marks the text as plugin-written, matching the die that
// marks a machine-rolled date. `🤷` is U+1F937 — one code point, no variation
// selector and no gendered ZWJ sequence.
const SCHEDULE_LOG_SKIPPED_REASON_EMOJI = "🤷";
const SCHEDULE_LOG_SKIPPED_REASON_TEXT = `${SCHEDULE_LOG_SKIPPED_REASON_EMOJI} no reason given`;
// Reason codes planScheduleLogEntry guards out with that are ordinary outcomes
// rather than failures: nothing was asked for, the task keeps no log, or the
// date did not move. They produce no notice text; anything else does.
const SCHEDULE_LOG_SILENT_GUARD_REASONS = Object.freeze(new Set(["empty-reason", "no-schedule-log", "unchanged-date"]));
```

### 2. `planScheduleLogEntry` grows the fallback branch (`main.js`, ~line 1581)

Replace the empty-reason guard:

```js
const normalized = normalizeScheduleReasonText(details.reason);
const fallback = normalizeScheduleReasonText(details.fallbackReason);
// An empty reason falls back only on a task that already keeps a log; a task
// with no marker is still left completely untouched, which is the documented
// escape hatch for "I do not want a log on this one".
const usedFallback = normalized.empty && !fallback.empty;
const reasonText = usedFallback ? fallback.reason : normalized.reason;
if (!reasonText) {
  return guard("empty-reason");
}
```

Set `reason: reasonText` in `entryFields`, then gate the fallback right after `existingParent` is resolved and before
the existing-parent branch runs:

```js
const existingParent = findScheduleLogParent(lines, taskIndex);
if (usedFallback) {
  if (!existingParent) {
    return guard("no-schedule-log");
  }
  // Generated text never claims a change that did not happen — the same rule
  // shouldWriteAutomaticScheduleLog applies to a rolled date's reason. A typed
  // reason on an unchanged date is a human decision and is still written.
  if (!shouldWriteAutomaticScheduleLog(entryFields.from, entryFields.to)) {
    return guard("unchanged-date");
  }
}
```

Add `usedFallback` to all three returned shapes: `false` inside `guard()` (a guarded plan wrote nothing, so the flag is
meaningless there) and inside the create branch (a fallback can never create a parent), `usedFallback` in the
existing-parent branch. Nothing else in the function changes — `insertLine`, `createdParent`, `lineTexts`, and
`lineText` are all as they are.

### 3. Two small shared helpers (`main.js`, immediately after `applyScheduleLogEntryToLines`, ~line 1670)

```js
// True when a scheduleLog payload carries anything a writer could log: a typed
// reason, or a fallback that a task with an existing log would use. The writers
// call this before planning so an absent payload costs nothing.
function hasScheduleLogReasonInput(scheduleLog) {
  if (!scheduleLog) {
    return false;
  }

  return (
    !normalizeScheduleReasonText(scheduleLog.reason).empty ||
    !normalizeScheduleReasonText(scheduleLog.fallbackReason).empty
  );
}

// Map a planned (and attempted) schedule-log write to the outcome the writers
// report. Null means "say nothing": the task keeps no log, or the date did not
// move, and neither is a failure worth a notice.
function getScheduleLogWriteOutcome(plan, applied) {
  if (!plan) {
    return null;
  }

  if (plan.valid && applied) {
    return plan.createdParent ? "created" : plan.usedFallback ? "added-fallback" : "added";
  }

  return !plan.valid && SCHEDULE_LOG_SILENT_GUARD_REASONS.has(plan.reason) ? null : "guard-failed";
}
```

Export `hasScheduleLogReasonInput` and `getScheduleLogWriteOutcome` from `module.exports` alongside the other
schedule-log helpers (~line 20538).

### 4. The three writers

**a. `setInlineBulletPropertyValues`** (~line 15824) — replace the schedule-log block with:

```js
let scheduleLogOutcome = null;
if (hasScheduleLogReasonInput(options.scheduleLog)) {
  const scheduleLogPlan = planScheduleLogEntry(String(cm.getValue() || ""), cursor.line, options.scheduleLog);
  scheduleLogOutcome = getScheduleLogWriteOutcome(
    scheduleLogPlan,
    scheduleLogPlan.valid && insertEditorLine(cm, scheduleLogPlan.insertLine, scheduleLogPlan.lineText),
  );
}
```

The `plan.valid &&` short-circuit is load-bearing: `insertEditorLine` must not run for a guarded plan. Then extend the
plain-notice suffix chain with the new outcome, before the `guard-failed` arm:

```js
        scheduleLogOutcome === "added-fallback"
          ? "; logged without a reason"
          : ...
```

**b. `setProjectNoteScheduledValue`** (~line 15528) — the same rewrite, with
`applyScheduleLogEntryToLines(plannedSource.lines, scheduleLogPlan) > 0` as the `applied` argument, and a matching arm
in the `parts` chain (~line 15599):

```js
    } else if (scheduleLogOutcome === "added-fallback") {
      parts.push("logged without a reason");
    }
```

**c. `planCountedBulletPropertyBatch`** (~line 10908) — relax the early-out and thread the fallback through:

```js
const normalizedReason = normalizeScheduleReasonText(rawReason);
// planScheduleLogEntry decides per target whether an empty reason falls
// back, since only it knows whether that task already keeps a log.
if (normalizedReason.empty && normalizeScheduleReasonText(scheduleLogOptions.fallbackReason).empty) {
  return null;
}
if (scheduleLogOptions.automatic && !shouldWriteAutomaticScheduleLog(from, to)) {
  return null;
}
return planScheduleLogEntry(source.lines.join(source.lineEnding), detail.line, {
  from,
  to,
  reason: normalizedReason.reason,
  fallbackReason: scheduleLogOptions.fallbackReason,
});
```

Count fallback entries alongside the others: declare `let scheduleLogFallbackTaskCount = 0;` beside
`scheduleLogCreatedParentCount`, increment it inside the apply loop when `scheduleLogPlan.usedFallback`, and return it
on the plan next to `scheduleLoggedTaskCount`.

Then in `setCountedBulletPropertyValue`'s notice (~line 15040), word the suffix honestly. A counted session shares one
typed reason across every target, so either all logged entries are fallbacks or none are:

```js
const scheduleLogSuffix =
  plan.scheduleLoggedTaskCount > 0
    ? `; ${
        plan.scheduleLogFallbackTaskCount === plan.scheduleLoggedTaskCount
          ? "logged without a reason on"
          : "logged reason on"
      } ${formatCountLabel(plan.scheduleLoggedTaskCount, "task")}`
    : "";
```

The priority notice card needs no change: a priority write's reason is always the generated `🎲` text, never empty, so
it can never take the fallback branch and `getPriorityNoticeOutcomeParts` keeps its existing `logged reason` part.

### 5. The modal (`BulletPropertyPickerModal`)

**a.** New method next to `getPendingScheduleFrom` — the single source of truth for "would `↵` on an empty input still
write something?", shared by the footer hint, the preview row, and the initial stage chrome:

```js
  // Whether pressing ↵ on an empty input still logs an entry: only when the
  // date actually moves and the task already keeps a log. A counted session
  // answers yes on behalf of the batch — each target is decided at write time.
  willLogWithoutReason() {
    const pending = this.pendingScheduleReason;
    if (!pending || !pending.to || pending.from === pending.to) {
      return false;
    }

    return (
      this.isCountedSession() ||
      Boolean(findScheduleLogParent(this.getEditorContent(), this.cursor.line))
    );
  }
```

**b.** `showScheduleReasonStage` — the initial footer hint must already be right for a task that keeps a log, since
`renderAll` is skipped when there is no `resultsEl`. Compute it after `this.pendingScheduleReason` is assigned:

```js
      footerHints: getBulletPropertyScheduleReasonHints({
        empty: true,
        fallback: this.willLogWithoutReason(),
      }),
```

**c.** `getFilteredItems()`'s `reason` branch — add the two flags the renderer needs:

```js
if (this.stage === "reason") {
  const normalized = normalizeScheduleReasonText(this.getRawQuery());
  const parentExists = Boolean(findScheduleLogParent(this.getEditorContent(), this.cursor.line));
  return [
    Object.freeze({
      kind: "schedule-reason-preview",
      ...normalized,
      parentExists,
      counted: this.isCountedSession(),
      fallback: normalized.empty && this.willLogWithoutReason(),
      searchText: normalized.reason,
    }),
  ];
}
```

**d.** `renderScheduleReasonPreviewItem` — three states. Only the inert row keeps the old `No reason` shape:

```js
const state = item.empty ? (item.fallback ? "fallback" : "empty") : item.hasInlineField ? "warning" : "valid";
```

Guard the early return with `if (item.empty && !item.fallback)` so a fallback row falls through to the shared preview
body, and render its title with the generated text:

```js
      formatScheduleLogEntryText({
        from: pending ? pending.from : "",
        to: pending ? pending.to : "",
        reason: item.empty ? SCHEDULE_LOG_SKIPPED_REASON_TEXT : item.reason,
      }),
```

The icon selector stays exactly as it is — `minus-circle` for both empty states is correct, because nothing was typed
either way. Finally, make the preview line counted-aware:

```js
      text: item.counted
        ? item.empty
          ? `Appends to every counted task that already has a ${SCHEDULE_LOG_MARKER_TEXT}`
          : `Appends to every counted task, adding a ${SCHEDULE_LOG_MARKER_TEXT} where missing`
        : item.parentExists
          ? `Appends to the existing ${SCHEDULE_LOG_MARKER_TEXT} on this task`
          : `Adds a ${SCHEDULE_LOG_MARKER_TEXT} child bullet to this task`,
```

(The counted arm for a _typed_ reason corrects a pre-existing inaccuracy — the old text described only the cursor task —
and is included so the two counted rows do not contradict each other.)

**e.** `renderResults()` — read the flags off the item the base class just computed instead of re-normalizing the query,
so the footer and the row can never disagree:

```js
  renderResults() {
    super.renderResults();
    if (this.stage === "reason") {
      const item = (this.visibleItems || [])[0];
      this.footerHints = getBulletPropertyScheduleReasonHints({
        empty: Boolean(item && item.empty),
        fallback: Boolean(item && item.fallback),
      });
      this.renderFooter();
    }
  }
```

`FilteredPickerModal.renderResults` assigns `this.visibleItems = this.getFilteredItems()` as its first statement (~line
7855), so the item is always current here.

**f.** `confirmScheduleReason` — always hand the writers a payload; the planner decides what it means:

```js
  confirmScheduleReason(item) {
    const pending = this.pendingScheduleReason;
    if (!pending || !item) {
      return false;
    }

    // The payload is supplied even for an empty input: a task that already keeps
    // a log records the change anyway, and planScheduleLogEntry is what decides
    // that per task (per target, in a counted session).
    return this.applySelectedValue(pending.dateItem, {
      scheduleLog: {
        from: pending.from,
        to: pending.to,
        reason: item.empty ? "" : item.reason,
        fallbackReason: SCHEDULE_LOG_SKIPPED_REASON_TEXT,
      },
    });
  }
```

**g.** `getBulletPropertyScheduleReasonHints` (~line 7249) — a three-way label:

```js
function getBulletPropertyScheduleReasonHints(options = {}) {
  return [
    {
      keys: ["↵"],
      label: options.empty ? (options.fallback ? "Log without a reason" : "Skip reason") : "Log reason",
    },
    { keys: ["esc"], label: "Cancel" },
  ];
}
```

### 6. Styles (`plugins/bob-navigation-hotkeys/styles.css`, after the `.is-empty` rules ~line 567)

```css
.bob-cnp-schedule-reason-row.is-fallback .bob-cnp-row-icon,
.bob-cnp-schedule-reason-row.is-fallback .bob-cnp-row-title {
  color: var(--text-muted);
}

.bob-cnp-schedule-reason-row.is-fallback.is-selected {
  border-left-color: var(--text-muted);
  background-color: color-mix(in srgb, var(--text-muted) 10%, transparent);
}

.bob-cnp-schedule-reason-row.is-fallback .bob-cnp-schedule-reason-preview {
  border-color: color-mix(in srgb, var(--text-muted) 35%, transparent);
}
```

Muted like the inert row, but it keeps the bordered preview box the `is-valid` row has, so the row reads as "subdued,
and still a real write". No narrow-viewport override is needed: `.bob-cnp-schedule-reason-preview` already has one and
it is state-independent.

## Tests (`scripts/test-navigation-hotkeys.cjs`)

The suite is 310 tests today and must stay green.

### Fixtures to update

- `choosing a priority roll writes immediately with a deterministic reason instead of prompting` (~line 4824) — its
  second half confirms a preset with an empty reason on a task with **no** log, so its expectation is unchanged and it
  now doubles as the regression guard for the escape hatch. Add a one-line comment saying so.
- `empty reason through the counted planner writes no schedule log entries` (~line 6848) — neither fixture task has a
  log, so the assertions hold as written. Add a second half that passes
  `{ reason: "   ", fallbackReason: "🤷 no reason given" }` and still asserts content identical to the no-payload plan,
  proving a fallback alone never creates a marker.

Run the full suite as the authority on which other fixtures move; the change is designed to be inert for every task
without a marker, so few should.

### Tests to add

1. **`planScheduleLogEntry` uses the fallback only when a marker exists** — one fixture with a marker and one without,
   both with `reason: ""` and `fallbackReason: "🤷 no reason given"`. Assert the marker case returns
   `{ valid: true, usedFallback: true, createdParent: false }` with
   `lineTexts === ["\t\t- *2026-08-13 → 2026-08-20* — 🤷 no reason given"]`, and the marker-less case returns
   `{ valid: false, reason: "no-schedule-log", usedFallback: false }`.

2. **A typed reason wins over the fallback** — both supplied, marker present: the entry carries the typed text and
   `usedFallback` is `false`.

3. **The fallback is suppressed on an unchanged date** — marker present, `from === to`, empty reason: `valid: false`,
   `reason: "unchanged-date"`. Pair it with the existing `a typed reason on an unchanged date is still written` test by
   name in a comment, since together they are the whole automatic-vs-human rule.

4. **No reason and no fallback still guards `empty-reason`** — the unchanged pure-function contract.

5. **`getScheduleLogWriteOutcome`** — `created` / `added` / `added-fallback` for applied plans, `null` for each reason
   in `SCHEDULE_LOG_SILENT_GUARD_REASONS`, `guard-failed` for `not-list-item` and for a valid plan that failed to apply.

6. **`hasScheduleLogReasonInput`** — `false` for `null`, `{}`, and `{ reason: "  " }`; `true` for a typed reason and for
   a fallback-only payload.

7. **End-to-end through the modal, existing log** — `createBulletPropertyPickerHarness` on a task that already has a
   marker and one entry; enter the reason stage on a preset, confirm empty via `confirmScheduleReasonStage`, and assert
   the exact resulting content has the new `🤷 no reason given` entry prepended above the old one, plus a notice
   matching `/; logged without a reason/`.

8. **End-to-end through the modal, no log** — the same flow on a task with no marker: content has the date only and no
   `SCHEDULE LOG` anywhere, and the notice does **not** match `/schedule log/`. This is the guard against the silent
   guard-reason set regressing into noisy `; schedule log not written` notices.

9. **The footer hint reads `Log without a reason` on a task that keeps a log** — extend the existing
   `the reason-stage footer hint flips between Skip reason and Log reason as the user types` test with a third picker
   over a task that has a marker, asserting the empty-input label is `Log without a reason` while the typed label is
   still `Log reason`.

10. **The preview row previews the entry it will write** — in the reason stage over a task with a marker, empty input:
    `getFilteredItems()[0]` has `{ empty: true, fallback: true, parentExists: true }`, and over a task without one,
    `fallback` is `false`. Assert `fallback` is also `false` when the picked date equals the current one.

11. **A counted `scheduled` session logs only the tasks that already keep a log** — three targets, one with a marker,
    one without, one unchanged; empty reason plus the fallback. Assert the exact content,
    `scheduleLoggedTaskCount === 1`, `scheduleLogFallbackTaskCount === 1`, `scheduleLogCreatedParentCount === 0`, and
    unchanged `cursorLine`.

12. **The counted notice wording** — the same batch through `setCountedBulletPropertyValue`, asserting the notice ends
    `; logged without a reason on 1 task`, and a typed-reason batch still ending `; logged reason on N tasks`.

13. **A counted priority batch is unaffected** — its generated `🎲` reasons are never empty, so no target takes the
    fallback branch: assert `scheduleLogFallbackTaskCount === 0` on the existing counted-priority fixture.

### Running the suite

From the `bob-plugins` repo root:

```bash
npm test
node scripts/validate-manifests.mjs
```

## Docs and release

1. **`plugins/bob-navigation-hotkeys/manifest.json`** — bump `version` from `1.19.0` to `1.20.0` (new behaviour).
2. **`bob-plugins/README.md`** line 16 — update the version column to `1.20.0` and extend the schedule-log clause: an
   empty reason on a task that already keeps a log records `🤷 no reason given` rather than nothing.
3. **`bob-cli/docs/projects.md`**, the _Schedule-log reason prompt_ section (~lines 298-349):
   - Rewrite the sentence "pressing `↵` on an empty input writes the date only, with no entry and no marker created"
     into the two-case rule: on a task with no log it still writes the date only and creates nothing; on a task that
     already has a `🗓️ **SCHEDULE LOG**` it records `🤷 no reason given` so the history has no gaps. Say plainly that
     the marker is the opt-in — the log is complete once it exists, and a task without one is never given one by a
     skipped prompt.
   - Add `🤷 no reason given` as a row in the existing deterministic-reason table, headed "Reason prompt skipped on a
     task that already has a log", and extend the 🎲 sentence to cover both machine glyphs.
   - Extend the "An automatic entry is skipped when the roll lands on the date the task already has" paragraph to say
     the same suppression applies to a skipped prompt, and keep the existing contrast that a typed reason on an
     unchanged date is still written.
   - Note that in a counted session the rule is applied per task: only counted tasks that already keep a log get an
     unexplained entry.
   - Leave the `Esc` and `Ctrl+D` sentences exactly as they are; neither changes.
4. **Deploy** — from the `bob-plugins` repo root, after committing:

   ```bash
   bob plugins sync -r "$PWD"
   ```

   The explicit `-r "$PWD"` is required; without it `bob plugins sync` resolves the default repo path instead of the
   linked-repo checkout.

Commit `bob-plugins` and `bob-cli` separately, each with the `/sase_git_commit` skill.

## Edge cases the implementation must handle

1. **Task has no marker** — unchanged: the date is written, nothing else, and no notice mentions the log.
2. **Task has a marker, date moves** — the unexplained entry is prepended above the existing entries, reusing their
   indentation and the marker's own list character. Unchanged `planScheduleLogEntry` placement behaviour.
3. **Task has a marker, date does not move** — nothing logged, no notice text, footer already said `Skip reason`.
4. **Task has a marker but no `scheduled` value yet** (log left over from a `Ctrl+D`) — `from` is empty, so the entry
   renders `*2026-08-20* — 🤷 no reason given`, the existing single-date form.
5. **Legacy `Schedule log:` marker on disk** — recognized by `SCHEDULE_LOG_PARENT_RE` exactly as today, so it also opts
   the task into unexplained entries. Never rewritten.
6. **Marker exists but has no entries yet** — `getScheduleLogEntryIndent` falls back to marker + one Tab, unchanged.
7. **Whitespace-only reason** — `normalizeScheduleReasonText` reports it empty, so it takes the fallback path, not the
   typed one.
8. **A reason of `"🤷 no reason given"` typed by hand** — indistinguishable in the note, and that is fine: it says the
   same thing. `usedFallback` is `false`, so the notice says `logged reason`.
9. **`Esc` during the prompt** — unchanged: nothing is written, the date included.
10. **Priority levels and the pinned roll row** — unchanged; they never enter the reason stage and their `reason` is
    never empty, so `fallbackReason` is never consulted.
11. **`^prj` lifecycle task** — the date goes to frontmatter, the unexplained entry goes under the `^prj` task bullet,
    same as a typed reason.
12. **Counted session, mixed markers** — decided per target; the notice counts only the tasks that were logged.
13. **The note changed under the modal** — every writer's existing guard runs first and aborts the whole write. Do not
    add a second, weaker guard.
14. **A genuine plan failure** (`not-list-item`, `task-out-of-range`, or a failed insert) — still reported as
    `; schedule log not written`. Only the three silent guard reasons are swallowed.
15. **`bob projects sync`** — still unaffected: `parse_prj_sub_block` marks only `🧩 **Sub-projects:**` lines, the entry
    carries no inline Dataview field, and the "no preceding blank line" constraint is untouched.

## Out of scope

- Back-filling unexplained entries for scheduled changes made outside the picker (`bob capture`, hand edits, or
  `bob projects sync` propagation). The log records what the picker does.
- Prompting or logging on `Ctrl+D`. Removal is not a reschedule.
- Making the `🤷` glyph, the text, or the opt-in rule configurable.
- Applying the "skip when the date did not move" rule to typed reasons.
- Auto-creating the marker on an empty reason. That is the escape hatch this plan deliberately preserves.
- Reading or querying the log from `bob`.
