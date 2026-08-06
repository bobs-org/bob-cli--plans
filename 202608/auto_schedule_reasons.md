---
tier: tale
title: Deterministic schedule-log reasons for machine-rolled dates
goal:
  Choosing a priority level, or the pinned priority-roll date in the `scheduled` stage, writes its own `🗓️ **SCHEDULE
  LOG**` entry naming the level transition and roll window instead of prompting for a reason, and skips the entry when
  the roll did not move the date.
proposed_by: bbugyi200.athena.tu.f0.f0
create_time: 2026-08-06 09:04:51
status: done
---

- **PROMPT:**
  [prompts/202608/auto_schedule_reasons.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/auto_schedule_reasons.md)
- **AGENTS:**
  - [bbugyi200.athena.tu.f0.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.tu.f0.f0.md)
- **COMMITS:**
  - [d97762d](https://github.com/bobs-org/bob-plugins/commit/d97762d47bcab960d6bb5ad1c2a3796c8855425f) —
    feat(bob-navigation-hotkeys): log deterministic reasons for machine-rolled scheduled dates

# Plan: Deterministic schedule-log reasons for machine-rolled dates

## The ask

Two `Ctrl+Shift+P` gestures produce a `scheduled` date the user did not type:

1. **Choosing a priority level** (`priority` → `P1`…`P4`) writes `[priority:: …]` and rolls a random date into
   `scheduled`. Today it does not prompt and writes no log entry.
2. **Choosing the pinned priority-roll row** in the `scheduled` value stage (`P2 roll`, the `dices` row pinned above the
   presets). Today it _does_ prompt, because the reason stage was wired to every item in the `scheduled` value stage.

Neither should prompt. Both should write a deterministic reason entry, creating the `🗓️ **SCHEDULE LOG**` marker when it
does not exist. Everything else about the reason prompt stays exactly as it is.

## Design

### Why a prompt is wrong for these two rows and an entry is still right

The reason prompt exists to capture a judgement the software cannot infer. For a rolled date there is no judgement — the
software chose the date, and it knows exactly why. Asking is friction on the fastest gesture in the picker; staying
silent loses the one fact that makes the log trustworthy. Writing the reason itself is the only option that keeps the
log complete without slowing anything down. So: **the log records every scheduled change made through the picker; the
prompt appears only when the reason is not already known.**

### The deterministic reason grammar

An entry keeps its existing shape — `*<from> → <to>* — <reason>` — and only the reason text is generated:

| Gesture                                             | Reason text                                 |
| --------------------------------------------------- | ------------------------------------------- |
| `priority` → `P2`, task was `P1`                    | `🎲 priority P1 → P2 · random in 8–30 days` |
| `priority` → `P2`, task had no priority field       | `🎲 priority P0 → P2 · random in 8–30 days` |
| `priority` → `P2`, task was already `P2`            | `🎲 priority P2 · random in 8–30 days`      |
| pinned roll row in the `scheduled` stage (now `P2`) | `🎲 P2 roll · random in 8–30 days`          |

In the note:

```markdown
- [?] #task Ship the thing [priority:: medium] [scheduled:: 2026-09-02] ^ship
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-13 → 2026-09-02_ — 🎲 priority P1 → P2 · random in 8–30 days
    - _2026-08-06 → 2026-08-13_ — waiting on the API review to land
```

Four decisions behind that grammar:

- **🎲 marks a machine-written entry.** The picker already renders the roll row with Lucide's `dices` icon
  (`renderValueItem`, ~line 12805), so the die is this feature's established symbol. One glyph tells you months later
  that nobody typed this line, without a heavy `[auto]`-style label, and it keeps automatic and human entries visually
  sortable in a log that mixes both.
- **The head names the gesture the user made,** so the entry is recognizable as the row they clicked: `priority P1 → P2`
  for a level row, `P2 roll` for the row literally labelled `P2 roll`.
- **The priority transition is recorded, not just the new level.** `*<from> → <to>*` already shows how the date moved;
  `P1 → P2` is the part that says _why_ it moved, and it is the single most useful thing the log can capture. A task
  with no priority field is written as `P0`, the documented implicit level (`docs/projects.md` line 250: "A task with no
  priority field is implicit P0"). When the level did not change, the transition is dropped rather than written as
  `P2 → P2`.
- **The window is recorded verbatim from the picker row.** `random in 8–30 days` is the exact string the roll row shows
  in its detail line, and §1 below makes both call the one formatter so they can never drift. Windows are configurable,
  so recording the window in force at write time is what keeps an old entry honest after `config.yml` changes.

### Automatic entries are written only when the date actually moves

A roll can land on the date the task already has (a ~1-in-23 chance on P2). Writing
`*2026-09-02 → 2026-09-02* — 🎲 priority P2 · …` would be noise claiming a change that did not happen, so an automatic
entry is skipped when its `from` equals its `to`.

This rule applies **only** to automatic entries. A typed reason is a human decision and is honored even when the date is
unchanged, which is the behaviour `docs/projects.md` already documents ("The prompt fires on every confirmed date … even
when the chosen date equals the current one"). The payload therefore carries an explicit `automatic: true` flag rather
than the writers guessing.

### Per-task reasons in a counted priority session

In `N<Ctrl+Shift+P>` → `priority` → `P2`, each counted task may have had a _different_ previous priority — the picker
even shows `current values mixed` for exactly this case. One shared reason would put `P1 → P2` on a task that was `P3`.
The counted priority path therefore passes a `reasonByLine` map, one reason per target, resolved from that target's own
`[priority:: …]` field.

The counted `scheduled` path needs no such map: `getPriorityRollLevel` pins the roll row only when _every_ counted task
shares one configured priority, so `🎲 P2 roll · random in 8–30 days` is true for all of them.

### Feedback

There is no prompt to preview these writes, so the notice carries the receipt. Priority writes use the rich
`buildPriorityNoticeModel` card, which gets a new chip: `logged` for a single task, `3 logged` in a counted session. The
pinned-roll row in the `scheduled` stage goes through the ordinary `scheduled` writers, whose plain notices already
append `; logged reason` / `; created schedule log`.

No `styles.css` change and no footer-hint change. The stage-two `↵ Set` hint is accurate for both row kinds, the roll
row already advertises itself with the `dices` icon and its `random in 2–7 days` detail, and adding selection-aware
footer machinery for one row would cost more than it explains.

## Repos touched

- **`bob-plugins`** (linked repo — open with `/sase_repo` first and use the printed path as the only path): all plugin
  code, tests, README, and manifest changes. Everything under `plugins/bob-navigation-hotkeys/` and
  `scripts/test-navigation-hotkeys.cjs`.
- **`bob-cli`** (your own workspace checkout): `docs/projects.md` only.

Do not edit plugin files under `~/bob/.obsidian/plugins/`; they are overwritten by `bob plugins sync`. No vault content
migration is needed — this feature only changes what future writes produce.

## Implementation

All line numbers are as of `bob-navigation-hotkeys` 1.18.1 and will drift as you edit; find the symbols by name.

### 1. Constants and pure helpers (`main.js`)

Add next to the existing schedule-log constants (~line 250, after `SCHEDULE_LOG_TRANSITION`):

```js
// A machine-rolled date logs its own reason instead of prompting for one. The
// die matches the `dices` icon the picker already uses for the pinned
// priority-roll row, and marks the entry as written by the plugin rather than
// typed by a human.
const SCHEDULE_LOG_AUTO_REASON_EMOJI = "🎲";
const SCHEDULE_LOG_AUTO_REASON_SEPARATOR = " · ";
// A task with no priority field is the implicit highest level, P0. The picker
// has no P0 row (Ctrl+D clears the field instead), so this label exists only to
// render the previous side of a priority transition in a log entry.
const IMPLICIT_PRIORITY_LEVEL_LABEL = "P0";
```

Add the helper block immediately after `applyScheduleLogEntryToLines` (~line 1660), keeping the schedule-log family
together:

```js
// The roll window as the picker states it, e.g. `random in 8–30 days`. Shared
// by the pinned roll row's detail line and by the deterministic log reason so
// the note can never disagree with the row the user clicked. Note the en dash.
function formatPriorityRollWindowText(level) {
  return `random in ${level.minDays}–${level.maxDays} days`;
}

// The previous priority as a picker label for the left side of a transition.
// An absent field is the implicit P0; a value outside the configured levels
// falls through as itself rather than being dropped.
function getPriorityRollFromLevelLabel(property, value) {
  return getBulletPropertyCurrentLabel(property, value) || IMPLICIT_PRIORITY_LEVEL_LABEL;
}

// Deterministic reason text for a machine-rolled scheduled date. `source` is
// "priority" when the user picked a level (the level, and any change to it, is
// the reason) or "scheduled" when they picked the pinned roll row in the date
// stage (the roll itself is the reason).
function formatPriorityRollScheduleReason(details = {}) {
  const level = details.level;
  if (!level || !level.label) {
    return "";
  }

  const head =
    details.source === "priority"
      ? `priority ${
          details.fromLevelLabel && details.fromLevelLabel !== level.label
            ? `${details.fromLevelLabel}${SCHEDULE_LOG_TRANSITION}${level.label}`
            : level.label
        }`
      : `${level.label} roll`;
  return `${SCHEDULE_LOG_AUTO_REASON_EMOJI} ${head}${SCHEDULE_LOG_AUTO_REASON_SEPARATOR}${formatPriorityRollWindowText(
    level,
  )}`;
}

// An automatic entry records a scheduling change, so a roll that landed on the
// date the task already had writes nothing. A typed reason is a human decision
// and is never suppressed this way.
function shouldWriteAutomaticScheduleLog(from, to) {
  const fromValue = normalizeBulletPropertyValue(from);
  const toValue = normalizeBulletPropertyValue(to);
  return Boolean(toValue) && fromValue !== toValue;
}

// Build the `options.scheduleLog` payload for a machine-rolled date, or null
// when nothing should be logged. Returning null (rather than a flag the callers
// must check) lets every writer keep its existing "falsy scheduleLog means no
// log" guard unchanged.
function buildPriorityRollScheduleLog(details = {}) {
  if (!shouldWriteAutomaticScheduleLog(details.from, details.to)) {
    return null;
  }

  const reason = formatPriorityRollScheduleReason(details);
  if (!reason) {
    return null;
  }

  return Object.freeze({
    from: normalizeBulletPropertyValue(details.from),
    to: normalizeBulletPropertyValue(details.to),
    reason,
    automatic: true,
  });
}
```

Then make `createPriorityRollDateItem` (~line 11267) use the shared window text:

```js
detail: `${value} · ${weekday} · ${formatPriorityRollWindowText(level)}`,
```

Export `formatPriorityRollWindowText`, `getPriorityRollFromLevelLabel`, `formatPriorityRollScheduleReason`,
`shouldWriteAutomaticScheduleLog`, and `buildPriorityRollScheduleLog` from `module.exports` alongside the other
schedule-log helpers (~line 20336).

### 2. The pinned roll row skips the prompt (`BulletPropertyPickerModal`)

**a.** Extract the previous-scheduled-value lookup currently inlined at the top of `showScheduleReasonStage`
(~line 12220) into a method, so the prompt path and the new roll path cannot drift:

```js
// The scheduled value a picked date replaces: frontmatter for a ^prj task,
// the inline field otherwise. Empty in a counted session, where each target's
// own previous value is resolved by the counted planner instead.
getPendingScheduleFrom() {
  const propertyItem = this.selectedPropertyItem;
  if (!propertyItem || this.isCountedSession()) {
    return "";
  }

  return normalizeBulletPropertyValue(
    propertyItem.target && propertyItem.target.kind === "project-frontmatter"
      ? propertyItem.currentValue
      : this.getCurrentPropertyValue(propertyItem.property.name),
  );
}
```

`showScheduleReasonStage` then sets `from: this.getPendingScheduleFrom()`. Behaviour is identical; the old code already
normalized the value at the same point.

**b.** Add the payload builder next to it:

```js
// The pinned priority-roll row is a date the plugin chose, so it never prompts:
// it writes straight through with a deterministic reason. Null when the roll
// landed on the date the task already has.
buildPriorityRollScheduleLogForItem(item) {
  if (!item || !item.priorityRoll || !item.level) {
    return null;
  }

  return buildPriorityRollScheduleLog({
    source: "scheduled",
    level: item.level,
    from: this.getPendingScheduleFrom(),
    to: item.value,
  });
}
```

**c.** In `showValueStage` (~line 12199), route the roll row past the reason stage:

```js
openItem:
  normalizeBulletPropertyName(property.name) === "scheduled"
    ? (item) => {
        if (item.priorityRoll) {
          return this.applySelectedValue(item, {
            scheduleLog: this.buildPriorityRollScheduleLogForItem(item),
          });
        }
        this.showScheduleReasonStage(item);
        return false;
      }
    : (item) => this.applySelectedValue(item),
```

Returning the `applySelectedValue` promise makes `openItemAtIndex` close the modal on a successful write, exactly like a
priority-level row. Only the pinned row carries `priorityRoll`; typed dates and presets still prompt.

In a counted session `getPendingScheduleFrom()` returns `""`, so the payload is always built and the planner resolves
each target's real `from` — see §4.

### 3. Single-task priority writes (`setBulletPriorityValue`, ~line 15743)

Both branches already accept `options.scheduleLog`; they just need it supplied. After `rolledValue` is computed and
before the `scheduledTarget.kind` branch:

```js
// Read the live line rather than the captured one so the transition is correct
// even if `lineText` was not supplied; the writers' own expectedLine guard is
// what aborts the whole write if the note moved underneath us.
const currentLine = getEditorLine(cm, cursor.line) ?? lineText ?? "";
const priorityField = findBulletPropertyField(currentLine, property.name);
const fromLevelLabel = getPriorityRollFromLevelLabel(property, priorityField ? priorityField.value : "");
```

**Project-frontmatter branch:** add to the `setProjectNoteScheduledValue` options object

```js
scheduleLog: buildPriorityRollScheduleLog({
  source: "priority",
  level,
  fromLevelLabel,
  from: expectedScheduledValue,
  to: rolledValue,
}),
```

**Inline branch:** add to the `setInlineBulletPropertyValues` options object

```js
scheduleLog: buildPriorityRollScheduleLog({
  source: "priority",
  level,
  fromLevelLabel,
  from: (findBulletPropertyField(currentLine, property.schedules) || {}).value || "",
  to: rolledValue,
}),
```

Nothing else in either writer changes: the log is planned against the post-write content and inserted below the task
line, and a guard failure still leaves the date write standing.

### 4. Counted writes (`planCountedBulletPropertyBatch`, ~line 10800)

Generalize the existing schedule-log block so it serves the priority operation too. Today it is gated on
`operation === "set" && propertyName === "scheduled"` and reads `from` from the priority-or-scheduled `state` and `to`
from the single `normalizedValue`. For a priority batch each target has its own rolled date and its own previous
scheduled value.

First hoist the project-frontmatter value so the block can see it — declare `let projectScheduledValue = "";` beside
`let projectPropertyChanged = false;` (~line 10600) and assign it inside the existing `if (projectTargets.length > 0)`
branch instead of re-declaring it there.

Then replace the block with:

```js
// One entry per changed target, using that target's own previous value
// (captured above via getCountedPropertyTargetState) and its own rolled date.
// A priority batch supplies reasonByLine because each task may have had a
// different previous level; a scheduled batch supplies one shared reason.
// Insertions apply in descending insertLine order so an earlier (smaller-index)
// insert position is never invalidated by a later one.
let scheduleLoggedTaskCount = 0;
let scheduleLogCreatedParentCount = 0;
const scheduleLogOptions =
  options.scheduleLog && (isPriorityOperation || (operation === "set" && propertyName === "scheduled"))
    ? options.scheduleLog
    : null;
if (scheduleLogOptions) {
  const entryByOriginalLine = new Map(targetStates.map((entry) => [entry.target.line, entry]));
  const scheduleLogPlans = changedTargets
    .map((detail) => {
      const entry = entryByOriginalLine.get(detail.originalLine);
      const scheduledState = isPriorityOperation ? entry && entry.scheduledState : entry && entry.state;
      // Only the first ^prj target's roll reaches frontmatter, so every
      // project-frontmatter target logs the value that was actually written.
      const to = !isPriorityOperation
        ? normalizedValue
        : scheduledState && scheduledState.target.kind === "project-frontmatter"
          ? projectScheduledValue
          : normalizeBulletPropertyValue(scheduledValueByLine.get(detail.originalLine));
      const from = scheduledState ? scheduledState.value : "";
      const rawReason =
        scheduleLogOptions.reasonByLine instanceof Map && scheduleLogOptions.reasonByLine.has(detail.originalLine)
          ? scheduleLogOptions.reasonByLine.get(detail.originalLine)
          : scheduleLogOptions.reason;
      const normalizedReason = normalizeScheduleReasonText(rawReason);
      if (normalizedReason.empty) {
        return null;
      }
      if (scheduleLogOptions.automatic && !shouldWriteAutomaticScheduleLog(from, to)) {
        return null;
      }
      return planScheduleLogEntry(source.lines.join(source.lineEnding), detail.line, {
        from,
        to,
        reason: normalizedReason.reason,
      });
    })
    .filter((scheduleLogPlan) => scheduleLogPlan && scheduleLogPlan.valid)
    .sort((first, second) => second.insertLine - first.insertLine);
  for (const scheduleLogPlan of scheduleLogPlans) {
    if (applyScheduleLogEntryToLines(source.lines, scheduleLogPlan) > 0) {
      scheduleLoggedTaskCount += 1;
      if (scheduleLogPlan.createdParent) {
        scheduleLogCreatedParentCount += 1;
      }
    }
  }
}
```

Every plan is still computed against the pre-insert content and applied descending, so positions stay valid.

Then in `setCountedBulletPriorityValue` (~line 14995), build the per-target reasons and pass them. `target.rawLine` is
guaranteed to equal the live line here — `getCountedTaskWriteContext` ran `validateCountedTaskSession` above — so the
previous priority can be read straight off it:

```js
const scheduleLogReasonByLine = new Map(
  session.targets.map((target) => [
    target.line,
    formatPriorityRollScheduleReason({
      source: "priority",
      level,
      fromLevelLabel: getPriorityRollFromLevelLabel(
        property,
        (findBulletPropertyField(target.rawLine, property.name) || {}).value || "",
      ),
    }),
  ]),
);
```

and add `scheduleLog: { automatic: true, reasonByLine: scheduleLogReasonByLine }` to the
`planCountedBulletPropertyBatch` options object.

The counted _scheduled_ path (`setCountedBulletPropertyValue`, ~line 14841) needs no change: it forwards
`options.scheduleLog` verbatim, and the roll row's payload already carries `automatic: true`.

### 5. Notices

**a.** `getPriorityNoticeOutcomeParts` (~line 11443) — push a part after the `#hide` part and before the Blocked part:

```js
const scheduleLoggedTaskCount = normalizePriorityNoticeCount(outcome.scheduleLoggedTaskCount);
if (scheduleLoggedTaskCount > 0) {
  parts.push(
    scope === "counted" ? `logged reason on ${formatCountLabel(scheduleLoggedTaskCount, "task")}` : "logged reason",
  );
}
```

**b.** `getPriorityNoticeChipTone` (~line 11488) — `if (/^logged reason/.test(text)) { return "info"; }` before the
muted fallback.

**c.** `getPriorityNoticeChipText` (~line 11501) — compress for the chip, mirroring the existing Blocked compression:

```js
const loggedMatch = /^logged reason(?: on (\d+) tasks?)?$/.exec(text);
if (loggedMatch) {
  return loggedMatch[1] ? `${loggedMatch[1]} logged` : "logged";
}
```

**d.** Feed the outcome in. The two single-task writers currently swallow their schedule-log outcome whenever
`options.buildNotice` is supplied, which is exactly the priority path:

- `setInlineBulletPropertyValues` (~line 15711) — pass `scheduleLogOutcome` into the `buildNotice` payload:
  `options.buildNotice({ blocked, recoveryOutcome, recoveryCounts, scheduleLogOutcome })`.
- `setProjectNoteScheduledValue` (~line 15459) — add `scheduleLogOutcome` to the `buildNotice` payload object.
- `setBulletPriorityValue` — in both `buildNotice` callbacks, add
  `scheduleLoggedTaskCount: outcome.scheduleLogOutcome === "added" || outcome.scheduleLogOutcome === "created" ? 1 : 0`
  to the `outcome` object passed to `buildPriorityNoticeModel`. (The inline callback destructures today; take the whole
  argument object instead so it can read `scheduleLogOutcome`.)
- `setCountedBulletPriorityValue` (~line 15047) — add `scheduleLoggedTaskCount: plan.scheduleLoggedTaskCount` to the
  `outcome` object. The plan already returns it.

A `guard-failed` outcome yields no chip; the date still landed and the plain-notice path already words that case for
non-priority writes.

## Tests (`scripts/test-navigation-hotkeys.cjs`)

### Fixtures that change because priority writes now append a log

Every priority-write test that asserts full editor content or a `^…$`-anchored match gains the two (or one) new lines.
Work through these and update each expectation to the exact new content:

- line 4323 `priority picker writes priority then rolled schedule in one guarded edit` — also update the notice string,
  which now ends `…; marked task Blocked; logged reason`.
- line 4346 `priority picker writes lowest priority with a wide rolled schedule` — the `assert.equal` on full content
  becomes three lines:

  ```js
  assert.equal(
    harness.editor.content,
    [
      "- [?] #task One [priority:: lowest] [scheduled:: 2026-11-02] ^one",
      "\t- 🗓️ **SCHEDULE LOG**",
      "\t\t- *2026-11-02* — 🎲 priority P0 → P4 · random in 91–365 days",
    ].join("\n"),
  );
  ```

  plus the `; logged reason` notice suffix. This is also the canonical no-previous-priority, no-previous-date case.

- line 4366 `priority picker replaces both existing values without duplicating fields` — the `[priority:: medium]` →
  `P1` case, so the entry reads `*2026-09-01 → 2026-08-08* — 🎲 priority P2 → P1 · random in 2–7 days`. The two
  `match(/\[priority::/g).length === 1` counts still hold because the entry contains no inline field.
- line 4390 `counted priority writes keep the rolled date right of the priority`.
- line 4414 `priority picker writes project priority inline and schedule to frontmatter atomically` — the log lands
  under the `^prj` task; `assert.doesNotMatch(harness.editor.content, /#task Ship.*\[scheduled::/)` still passes because
  the entry is on its own line.
- line 4448 `counted priority picker rolls an independent schedule for every task` — this one maps over
  `content.split("\n")` calling `findBulletPropertyField(line, "priority").value`, which will now throw on the inserted
  log lines. Filter to task lines first (e.g. `.filter((line) => line.includes("#task"))`) and add a separate assertion
  on the three logs. Notice gains `; logged reason on 3 tasks`.
- line 4489 / 4525 `counted priority planning …` — planner-level tests; update `plan.content` expectations and assert
  `plan.scheduleLoggedTaskCount`.

Search for any other `#task .*priority::` content assertion the suite makes and re-check it; treat a full-suite run as
the authority, not this list.

### Tests to rewrite

- line 4691 `choosing a priority roll uses the ordinary scheduled write path` — its premise is now false by design. The
  roll writes immediately without a reason stage, while the preset still prompts. Rewrite as
  **`choosing a priority roll writes immediately with a deterministic reason instead of prompting`**:

  ```js
  const rolled = makeHarness();
  const picker = await openBulletPropertyValueStage(rolled, "scheduled");
  assert.equal(picker.visibleItems[0].priorityRoll, true);
  await picker.openItemAtIndex(0);
  assert.notEqual(picker.stage, "reason");
  assert.equal(
    rolled.editor.content,
    [
      "- [?] #task One [priority:: high] [scheduled:: 2026-08-05] ^one",
      "\t- 🗓️ **SCHEDULE LOG**",
      "\t\t- *2026-08-05* — 🎲 P1 roll · random in 2–7 days",
    ].join("\n"),
  );
  ```

  Keep the preset half of the old test as the contrast: a non-roll row still enters the reason stage, and confirming it
  empty writes the date with no log.

- line 6708 `choosing a priority level does not enter the reason stage` — keep the stage assertion and extend it to
  assert the deterministic entry actually landed. It currently stubs `plugin.setBulletPriorityValue`, so either assert
  the `scheduleLog` payload handed to a capturing stub, or switch it to `createBulletPropertyPickerHarness` with
  `createPriorityPickerConfig()` and assert real content. Prefer the real harness.

### Tests to add

1. **`formatPriorityRollScheduleReason` covers all four shapes** — the table in the Design section, verbatim, against a
   `{ label: "P2", minDays: 8, maxDays: 30 }` level: changed transition, `P0` transition for an empty `fromLevelLabel`,
   no transition when `fromLevelLabel === level.label`, and the `source: "scheduled"` roll form. Assert it returns `""`
   for a missing level.

2. **The picker row and the log agree on the window text** — assert
   `createPriorityRollDateItem(level, baseDate, "", () => 0).detail` ends with the same
   `formatPriorityRollWindowText(level)` the reason contains. This is the regression guard for the drift §1 exists to
   prevent.

3. **`buildPriorityRollScheduleLog` returns null when the roll does not move the date** — same `from` and `to` → `null`;
   different → a frozen `{ from, to, reason, automatic: true }`. Also `null` for an empty `to`.

4. **`getPriorityRollFromLevelLabel`** — configured value → label, empty → `"P0"`, unconfigured value (`"highest"`) →
   itself.

5. **A counted priority batch writes one entry per task using each task's own previous level** — three tasks at `P1`,
   `P3`, and no priority, all set to `P2`; assert the exact resulting content, that the three reasons differ (`P1 → P2`,
   `P3 → P2`, `P0 → P2`), that `cursorLine` is unchanged, and that `scheduleLoggedTaskCount === 3`.

6. **A counted priority batch skips a task whose rolled date equals its current date** — assert no entry for that task
   and that `scheduleLoggedTaskCount` counts only the others. Drive the rolls with a scripted `random`.

7. **A priority roll onto the current date writes the date but no log** — single task whose `scheduled` already equals
   the rolled value; assert content has no `SCHEDULE LOG` marker.

8. **The priority notice chip** — extend the existing
   `priority notice model summarizes single counted and project writes` test (line 3862) with
   `scheduleLoggedTaskCount: 1` → a `logged` chip with tone `info`, and `scheduleLoggedTaskCount: 3` in `counted` scope
   → `3 logged` and the text form `logged reason on 3 tasks`.

9. **A typed reason on an unchanged date is still written** — the negative control for the `automatic` flag: confirm the
   reason stage with text on a date equal to the current one and assert the entry lands. This is what stops a future
   refactor from applying the automatic rule everywhere.

### Running the suite

From the `bob-plugins` repo root:

```bash
npm test
node scripts/validate-manifests.mjs
```

The suite is 302 tests today and must stay green.

## Docs and release

1. **`plugins/bob-navigation-hotkeys/manifest.json`** — bump `version` from `1.18.1` to `1.19.0` (new behaviour, not a
   fix).
2. **`bob-plugins/README.md`** line 16 — update the version column to `1.19.0` and extend the schedule-log clause to
   note that priority levels and the pinned roll suggestion log a deterministic reason instead of prompting.
3. **`bob-cli/docs/projects.md`**:
   - _Priority property and scheduled rolls_ (lines 230-282) — after the "Choosing P1, P2, P3, or P4 writes the priority
     and rolls a `scheduled` date…" paragraph, state that the write also records a `🗓️ **SCHEDULE LOG**` entry naming
     the level transition and the roll window, with a worked example line; and in the pinned-suggestion paragraph
     (line 278) state that choosing the suggestion writes immediately with its own deterministic reason rather than
     prompting.
   - _Schedule-log reason prompt_ (lines 284-317) — replace the sentence beginning "Choosing a priority level (which
     rolls a `scheduled` date as a side effect) and pressing `Ctrl+D`…" (line 312) with the new rule: the log records
     every scheduled change the picker makes, the prompt appears only when the reason is not already known, and `Ctrl+D`
     still writes nothing. Add the four deterministic reason shapes, the 🎲 convention, that an automatic entry is
     skipped when the roll does not move the date (while a typed reason on an unchanged date is still written), and that
     a counted priority session gives each task its own transition.
4. **Deploy** — from the `bob-plugins` repo root, after committing:

   ```bash
   bob plugins sync -r "$PWD"
   ```

   The explicit `-r "$PWD"` is required; without it `bob plugins sync` resolves the default repo path instead of the
   linked-repo checkout.

Commit `bob-plugins` and `bob-cli` separately, each with the `/sase_git_commit` skill.

## Edge cases the implementation must handle

1. **Roll lands on the current date** — no entry, no marker; the date write proceeds normally.
2. **Task has no priority field** — logs `P0 → P2`.
3. **Task has a priority value outside the configured levels** (e.g. `highest`) — the raw value is used as the label
   rather than dropped, so the entry stays truthful.
4. **Level re-picked unchanged** — no `P2 → P2`; the head is just `priority P2`. The date still moved (a fresh roll), so
   the entry is still written.
5. **Task has no previous scheduled date** — the entry renders `*<to>* — <reason>`, the existing single-date form.
6. **`^prj` lifecycle task** — priority stays inline, the rolled date goes to frontmatter, and the log goes under the
   `^prj` task bullet, matching the typed-reason path. `from` is the frontmatter value.
7. **Counted batch with several `^prj` targets** — only the first target's roll reaches frontmatter, so every
   project-frontmatter target logs `projectScheduledValue`, not its own discarded roll.
8. **Counted batch with mixed previous priorities** — per-task reasons via `reasonByLine`.
9. **Counted batch where a target is unchanged** — no entry, and the notice count reflects only logged tasks. Existing
   behaviour, preserved by keying off `changedTargets`.
10. **Existing marker on the task** — reused in place; the new entry is prepended above the existing entries with their
    indentation and the marker's own list character. Unchanged `planScheduleLogEntry` behaviour.
11. **Legacy `Schedule log:` marker on disk** — still recognized and reused, never rewritten. Unchanged.
12. **The note changed under the picker** — every writer's existing guard runs first and aborts the whole write. Do not
    add a second, weaker guard.
13. **Log plan guards out while the date write succeeded** — the date stands; no chip appears. Same contract as the
    typed-reason path.
14. **`bob projects sync`** — still unaffected: `parse_prj_sub_block` marks only `🧩 **Sub-projects:**` lines, and the
    entry carries no inline Dataview field. The "no preceding blank line" constraint is untouched.

## Out of scope

- Prompting or auto-logging on `Ctrl+D` (unscheduling). Removal is not a reschedule.
- A schedule-log entry from `bob capture <text> p:<N>`. Capture creates the task, so there is no previous date and no
  history to record; a one-entry log on a brand-new task is noise.
- Making the deterministic reason text, emoji, or `P0` label configurable.
- Applying the "skip when the date did not move" rule to typed reasons.
- Reading or querying the log from `bob`.
