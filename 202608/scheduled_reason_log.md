---
tier: tale
title: Prompt for a reason after picking a scheduled date
goal:
  After choosing a `scheduled` date in the `Ctrl+Shift+P` bullet-property picker, the user is prompted for an optional
  reason that is recorded as a dated entry under a managed `🗓️ **Schedule log:**` child bullet on the task, with an
  empty reason writing nothing but the date.
proposed_by: bbugyi200.athena.tu
create_time: 2026-08-06 07:41:25
status: done
---

- **PROMPT:**
  [prompts/202608/scheduled_reason_log.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202608/scheduled_reason_log.md)
- **AGENTS:**
  - [bbugyi200.athena.tu](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.athena.tu.md)
- **COMMITS:**
  - [780cf45](https://github.com/bobs-org/bob-cli/commit/780cf456346cff63dc97cb104c7d9070dbb528cb) — docs(projects):
    document schedule-log reason prompt

# Plan: Schedule-log reason prompt for the `Ctrl+Shift+P` scheduled picker

## Repos touched

- **`bob-plugins`** (linked repo — open with `/sase_repo` first): all plugin code, tests, README, and manifest changes.
  Everything under `plugins/bob-navigation-hotkeys/` and `scripts/test-navigation-hotkeys.cjs`.
- **`bob-cli`** (your own workspace checkout): user-facing docs only — `docs/projects.md`.

Do not edit plugin files under `~/bob/`; they are overwritten by `bob plugins sync`.

## Background: what exists today

`plugins/bob-navigation-hotkeys/main.js` implements the `Ctrl+Shift+P` **Set bullet property** picker as
`BulletPropertyPickerModal`, a subclass of `FilteredPickerModal`. It is a staged modal driven by `this.stage`:

| Stage        | Entered by                     | Items                                                 |
| ------------ | ------------------------------ | ----------------------------------------------------- |
| `properties` | opening the picker             | configured properties from `~/.config/bob/config.yml` |
| `value`      | choosing a property            | date presets / priority levels / open tasks           |
| `blockid`    | linking a dependency w/o an ID | one synthetic preview row driven by the raw query     |

The `blockid` stage is the key precedent for this feature: it is a **free-text stage** that renders a single synthetic
preview item from `this.getRawQuery()` (see `getFilteredItems()` and `renderBlockIdPreviewItem()`), and it **defers all
writes** until the prompt is confirmed. Its `onClose()` comment states the contract explicitly: _"Dismissing the modal
mid-prompt is a clean cancel: no writes happen until the final block ID is confirmed."_

Choosing a date today calls `applySelectedValue(item)`, which fans out to one of three writers depending on the target:

| Target                                     | Writer                                                             | Application style                                   |
| ------------------------------------------ | ------------------------------------------------------------------ | --------------------------------------------------- |
| ordinary task, inline `[scheduled:: ...]`  | `setBulletPropertyValue` → `setInlineBulletPropertyValues`         | single `replaceEditorLine`                          |
| `^prj` lifecycle task, project frontmatter | `setProjectNoteScheduledValue`                                     | `applyEditorContentTransaction` over `plan.content` |
| counted session (`N<Ctrl+Shift+P>`)        | `setCountedBulletPropertyValue` → `planCountedBulletPropertyBatch` | `applyEditorContentTransaction` over `plan.content` |

There is also an existing precedent for **managed child bullets** on a task: `planDependencyNavigationBulletSync` /
`applyDependencyNavigationBulletSyncPlan` insert and rewrite dependency bullets in the task's child block, using
`findCurrentBulletChildBlock` and `getDependencyChildIndent`. Dependency bullets are inserted at `collection.startLine`
— that is, as the **first** child.

## Design

### The markdown shape

```markdown
- [?] #task Ship the thing [priority:: medium] [scheduled:: 2026-08-20] ^ship
  - ![[#^blocked-by-this]]
  - Some freeform note I wrote by hand
  - 🗓️ **Schedule log:**
    - **2026-08-13 → 2026-08-20** — waiting on the API review to land
    - **2026-08-06 → 2026-08-13** — was out sick
```

- **Parent marker bullet:** `🗓️ **Schedule log:**` with nothing after the colon. This matches the newest house
  convention for a managed child bullet, `🧩 **Sub-projects:**` (see `bob-cli`'s `src/native/projects.rs`,
  `SUBPROJECTS_MARKER_PREFIX`) — emoji, bold title-case label, trailing colon — rather than the plugin's older
  `🔗 **DEPENDS ON:**` all-caps form, which is explicitly documented in `main.js` as legacy and being migrated away.
- **Entry bullet:** `**<from> → <to>** — <reason>` when the property had a previous value, and `**<to>** — <reason>`
  when it did not. The bolded date span is what satisfies "clearly show the scheduled date that it is/was associated
  with": the right-hand date is always the date the reason justifies. Keeping the old date on the left makes each entry
  self-describing, so the deferral chain survives even when intervening changes were made without a reason.
- **Ordering: newest entry first**, immediately under the parent marker. The top entry then always answers "why is this
  task scheduled where it is _now_", which is the question you have when you look at the task.
- **Placement: the parent marker is appended as the _last_ direct child** of the task when it does not already exist.
  Dependency bullets deliberately claim the first child slot, and hand-written notes sit near the top where they are
  most actionable; a growing log belongs at the bottom. When the marker already exists it is reused wherever it is,
  never moved.
- **Plain markdown, no inline fields.** The entry must not contain a `[key:: value]` Dataview field: child list items
  are indexed by Dataview, and a stray `scheduled::` on a grandchild would pollute vault queries. Bold text keeps the
  entry greppable without that risk. Wikilinks inside a reason are allowed and are a feature.

### Interaction with `bob projects sync` (verified, no conflict)

`parse_prj_sub_block` in `bob-cli`'s `src/native/projects.rs` collects every descendant line under a `^prj` task, but
only lines whose content starts with `🧩 **Sub-projects:**` are marked `is_marker`, and only `marker_line.links` feed
the sub-projects ledger. Desired sub-projects come from real child project notes, not from arbitrary sub-block links,
and `subprojects_need_normalization` only inspects marker lines. A `🗓️ **Schedule log:**` bullet and its entries —
including entries containing wikilinks — are therefore invisible to `bob projects sync`.

One caveat to preserve: `parse_prj_sub_block` stops at the first blank line, so the schedule log must be inserted
**without** a preceding blank line.

### The flow

```
Ctrl+Shift+P  →  [properties]  →  scheduled  →  [value: date]  →  [reason]  →  write
```

The reason stage is a new `stage === "reason"` in `BulletPropertyPickerModal`, built on exactly the `blockid` stage's
pattern.

| Element        | Content                                                            |
| -------------- | ------------------------------------------------------------------ |
| title          | `Reason`                                                           |
| header icon    | `message-square-quote`                                             |
| input label    | `Schedule reason`                                                  |
| placeholder    | `Why this date? (↵ to skip)`                                       |
| subtitle       | `2026-08-13 → 2026-08-20 · Thu · in 14 days · nothing written yet` |
| results        | exactly one synthetic preview row                                  |
| footer (empty) | `↵ Skip reason` · `esc Cancel`                                     |
| footer (typed) | `↵ Log reason` · `esc Cancel`                                      |

Build the subtitle's weekday and relative-distance clauses with the existing `getBulletPropertyDateWeekday`,
`getLocalDayOffset`, and `formatRelativeDayOffset` helpers so the wording matches the priority notices.

The single preview row renders the literal markdown that will be written, mirroring `renderBlockIdPreviewItem`:

- **Empty input** — muted row, icon `minus-circle`, title `No reason`, meta
  `scheduled → 2026-08-20 only; no schedule log entry`.
- **Typed input** — icon `check-circle-2`, title = the rendered entry bullet text
  (`**2026-08-13 → 2026-08-20** — waiting on the API review to land`), and a preview line that says either
  `Appends to the existing 🗓️ **Schedule log:** on this task` or
  `Adds a 🗓️ **Schedule log:** child bullet to this task`.
- **Typed input containing `::`** — still valid, but the row carries a warning state (icon `alert-triangle`) and meta
  `"::" creates a Dataview inline field on this bullet`. This is advisory only; Enter still writes.

### Write and cancel semantics (the important decision)

**Nothing is written until the reason stage resolves.** The date write is deferred out of `applySelectedValue` until the
reason prompt is confirmed or skipped, so the date field and the log bullets land in one guarded pass. This is the same
contract the `blockid` stage already establishes in this modal, and it keeps the two edits from ever diverging.

Consequences, which this plan deliberately accepts:

- `↵` with text → write the date **and** append the log entry (creating the parent marker if needed).
- `↵` on an empty input → write the date only. No entry, **and no parent marker is created**. This is the requested
  escape hatch.
- `esc` → cancel everything; the date is **not** written. This preserves the universal "esc closes the modal without
  doing anything" expectation and matches the block-ID stage. The subtitle's trailing `· nothing written yet` clause
  exists specifically so this is never a surprise.

### Scope of the prompt

| Situation                                                                                               | Prompts?      | Why                                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `scheduled` date chosen in the value stage (typed date, preset, or the pinned priority-roll suggestion) | **yes**       | this is "the user selects a new scheduled date"                                                                                                                                                                                  |
| `scheduled` on a `^prj` lifecycle task (project frontmatter)                                            | **yes**       | same picker, same gesture; the log still goes under the `^prj` task bullet                                                                                                                                                       |
| counted session `N<Ctrl+Shift+P>` → `scheduled` → date                                                  | **yes**, once | one prompt, one reason, applied to every counted task — a counted session exists to apply one decision in bulk                                                                                                                   |
| choosing a `priority` level (P1–P4)                                                                     | **no**        | the user picked a priority, not a date; the roll is incidental and the flow is deliberately fast                                                                                                                                 |
| `Ctrl+D` on `scheduled`                                                                                 | **no**        | a removal, not a new date                                                                                                                                                                                                        |
| any date property not named `scheduled`                                                                 | **no**        | `scheduled` is already special-cased throughout `main.js` (`PROJECT_NOTE_PROPERTY_TARGETS`, the `normalizeBulletPropertyName(name) === "scheduled"` branches); keeping the prompt scoped to it avoids inventing a config surface |

The prompt fires on **every** confirmed date in the `scheduled` value stage, even when the chosen date equals the
current one. "Choosing a date always asks why" is a simpler, more predictable rule than a change-detection branch, and
pressing Enter on an empty input costs one keystroke.

## Implementation

### 1. Constants (`main.js`, near `DEPENDENCY_NAVIGATION_*` around line 220)

```js
const SCHEDULE_LOG_EMOJI = "🗓️";
const SCHEDULE_LOG_LABEL = "Schedule log";
const SCHEDULE_LOG_SEPARATOR = " — ";
const SCHEDULE_LOG_TRANSITION = " → ";
```

Plus two regexes built the same way `DEPENDENCY_NAVIGATION_BULLET_RE` is:

- `SCHEDULE_LOG_PARENT_RE` — named groups `indent`, `marker`; matches an optionally emoji-less `**Schedule log:**`
  bullet with nothing after the colon, so a marker written by hand without the emoji is still recognized.
- `SCHEDULE_LOG_ENTRY_RE` — named groups `indent`, `marker`, `from` (optional), `to`, `reason`. Used by tests and by
  grandchild-indent detection.

Note that `🗓️` is `U+1F5D3 U+FE0F`; keep the variation selector in both the constant and the regex source.

### 2. Pure helpers (`main.js`, module scope, all added to `module.exports`)

Add these next to the dependency-navigation helpers so the two managed-child-bullet families sit together:

- `formatScheduleLogParentBullet(indent, marker)` →
  `` `${indent}${marker} ${SCHEDULE_LOG_EMOJI} **${SCHEDULE_LOG_LABEL}:**` ``
- `formatScheduleLogEntryBullet(indent, marker, { from, to, reason })` →
  `` `${indent}${marker} **${from ? from + SCHEDULE_LOG_TRANSITION : ""}${to}**${SCHEDULE_LOG_SEPARATOR}${reason}` ``
- `parseScheduleLogParentBullet(line)` → `{ indent, marker, hasEmoji }` or `null`
- `parseScheduleLogEntryBullet(line)` → `{ indent, marker, from, to, reason }` or `null`
- `normalizeScheduleReasonText(raw)` → frozen `{ reason, empty, hasInlineField }`. Trims, collapses all internal
  whitespace runs (including any `\r`/`\n` a paste could introduce) to single spaces, and sets `hasInlineField` when
  `/::/` matches. Never mutates the text otherwise — wikilinks, backticks, and markdown in the reason are preserved
  verbatim.
- `findScheduleLogParent(lines, taskLine)` → `{ line, indent, marker }` or `null`. Scans
  `findCurrentBulletChildBlock(lines, taskLine)` for the first line matching `SCHEDULE_LOG_PARENT_RE` whose
  `findNearestParentListItem(lines, index)` is `taskLine`, so a marker belonging to a nested bullet is never hijacked.
- `getScheduleLogEntryIndent(lines, parentLine)` → reuse an existing entry's indentation when the marker already has
  entries; otherwise `getBulletIndent(markerLine) + "\t"`, matching `getDependencyChildIndent`'s Obsidian-default-Tab
  fallback and `task-status-cycler`'s `CHILD_BULLET_INDENT_UNIT`.
- `planScheduleLogEntry(content, taskLine, { from, to, reason })` → frozen plan:

  ```js
  { valid, reason: guardReason|null, changed, createdParent,
    insertLine, lineTexts, lineText }
  ```

  Guards (return `changed: false` with a `reason` string, never throw) when `taskLine` is out of range, is not a list
  item, or `reason` is empty. When the marker is missing it emits **two** lines — parent then entry — inserted at
  `findCurrentBulletChildBlock(...).endLineExclusive` (last direct child slot). When the marker exists it emits **one**
  line inserted at `parent.line + 1` (newest-first) using the marker's own list-marker character and the resolved entry
  indent.

- `applyScheduleLogEntryToLines(lines, plan)` → splices `plan.lineTexts` into a mutable `lines` array and returns the
  number of lines inserted. This is the shared primitive the two content-level writers use.

`planScheduleLogEntry` must use `getBulletIndent`/`getBulletIndentWidth`, which already treat `>` quote levels as indent
stops, so a task inside a blockquote gets its log inside the same quote context.

### 3. Threading the reason through the three writers

Add an `options.scheduleLog` payload — `{ from, to, reason }` — that each writer converts into a plan and applies.
`from` is the property's previous value (empty string when unset); `to` is the new value.

**a. `setInlineBulletPropertyValues`** (single task, line-level). After the existing `replaceEditorLine` succeeds and
before the notice, and only when `options.scheduleLog` is present with a non-empty reason:

1. Re-read `cm.getValue()`.
2. `planScheduleLogEntry(content, cursor.line, options.scheduleLog)`.
3. Apply with `insertEditorLine(cm, plan.insertLine, plan.lineText)` — the existing helper already joins multi-line text
   and appends correctly past the last line, and `applyDependencyNavigationBulletSyncPlan` uses it the same way.
4. Track the outcome (`"added"`, `"created"`, `"guard-failed"`) for the notice.

Insertions are strictly below `cursor.line`, so the existing `setEditorCursorSafely(cm, cursor.line, ...)` call keeps
the cursor on the task. A guard failure must **not** roll back the date write; report it in the notice instead (see step
5).

**b. `setProjectNoteScheduledValue`** (`^prj`, content-level). `plan.cursorLine` already points at the `^prj` task
inside `plannedSource.lines` after the frontmatter edit. Insert the log lines into `plannedSource.lines` right after
`applyBulletPropertyEdits` writes `finalLine` back and **before** `plannedSource.lines.join(...)`. Because insertions
are below `plan.cursorLine`, the cursor position passed to `applyEditorContentTransaction` is unchanged.

**c. `planCountedBulletPropertyBatch`** (counted session, content-level). Accept `options.scheduleLog` as
`{ reason, fromByLine }` — one shared reason, per-task previous values captured from `getCountedPropertyTargetState`.
After the per-target loop has written every `source.lines[mappedLine] = nextLine`, and before `source.lines.join(...)`:

1. Build one plan per **changed** target against the current `source.lines`. Skip unchanged targets — a task whose date
   did not move should not get a log entry.
2. Apply them in **descending `insertLine` order** so earlier insert positions stay valid.
3. `session.targets[0]` is the cursor line and every insertion is at or below `targets[0] + 1`, so `cursorLine` needs no
   adjustment. Add a comment saying so.
4. Return `scheduleLoggedTaskCount` and `scheduleLogCreatedParentCount` on the plan for the notice.

### 4. The reason stage in `BulletPropertyPickerModal`

- **Constructor:** add `this.pendingScheduleReason = null` and clear it in `clearPendingBatch()` so `onClose()` already
  drops it on dismissal.
- **`showValueStage`:** when the property is the `scheduled` date property, change `openItem` to
  `(item) => { this.showScheduleReasonStage(item); return false; }`. Every other property keeps
  `openItem: (item) => this.applySelectedValue(item)`. Because `getFilteredItems()` also injects typed-date items and
  the pinned priority-roll item into this same stage, all three date sources route through the reason stage
  automatically.
- **`showScheduleReasonStage(dateItem)`:** sets `this.stage = "reason"`, stores
  `this.pendingScheduleReason = { dateItem, from, to: dateItem.value }`, and calls `applyOptions({...})` with the chrome
  from the design table plus `filterItem: () => true`,
  `renderItem: (i, el, q) => this.renderScheduleReasonPreviewItem(i, el, q)`,
  `openItem: (i) => this.confirmScheduleReason(i)`. Finish with `this.renderAll({ clearQuery: true })` so the input
  starts empty (unlike the block-ID stage, which prefills). `from` comes from
  `this.getCurrentPropertyValue("scheduled")` for the inline case, from `propertyItem.currentValue` for the
  project-frontmatter case, and is omitted for a counted session (per-task values are resolved in the planner).
- **`getFilteredItems()`:** add a `this.stage === "reason"` branch, before the existing `blockid` branch, returning a
  single frozen synthetic item built from `normalizeScheduleReasonText(this.getRawQuery())` plus the pending from/to and
  whether the marker already exists. `openItemAtIndex` requires `visibleItems[0]` to exist, so this branch must always
  return exactly one item — including for empty input.
- **`renderResults()` override:** call `super.renderResults()`, then, when `this.stage === "reason"`, recompute
  `this.footerHints` from the current emptiness and call `renderFooter()`. The base class's `input` listener only calls
  `renderResults()`, so this is what makes the `↵ Skip reason` ⇄ `↵ Log reason` flip live. Mirror the existing
  `refreshLocalTaskFooter` shape.
- **`getBulletPropertyScheduleReasonHints({ empty })`** — new module-scope helper next to
  `getBulletPropertyBlockIdHints`.
- **`confirmScheduleReason(item)`:** calls `applySelectedValue(this.pendingScheduleReason.dateItem, { scheduleLog })`,
  where `scheduleLog` is `null` for an empty reason. Returns the writer's boolean so `openItemAtIndex` closes the modal
  on success.
- **`applySelectedValue(item, options = {})`:** thread `options.scheduleLog` into the counted, project-frontmatter, and
  inline branches. The `priority` branches ignore it (they are never reached from the reason stage).
- **`handleKeydown`:** no new bindings. `Enter` and `Escape` already do the right thing through the base class and
  Obsidian's modal.

### 5. Notices

Extend the existing notices so the write is legible without opening the note:

- inline single task: append `; logged reason` (or `; created schedule log` when the marker was created) to the existing
  `` `${name} → ${value}` `` notice built in `setInlineBulletPropertyValues`.
- `^prj`: push `logged reason` into the existing `parts` array in `setProjectNoteScheduledValue`.
- counted: append `` `; logged reason on ${formatCountLabel(plan.scheduleLoggedTaskCount, "task")}` `` to the counted
  notice.
- If a log plan guards out while the date write succeeded, say so explicitly — `; schedule log not written` — rather
  than failing silently.

### 6. Styles (`plugins/bob-navigation-hotkeys/styles.css`)

Add `.bob-cnp-schedule-reason-row` rules modeled on the existing `.bob-cnp-blockid-preview-row` block (around line 512)
and its `.bob-cnp-blockid-preview` body (line 532), including the narrow-viewport override near line 786:

- `.is-empty` → muted icon and body text, matching `.bob-cnp-pill.is-muted`.
- `.is-valid` → the same accent treatment as `.bob-cnp-blockid-preview-row.is-valid`.
- `.is-warning` → the warning treatment used by `.bob-cnp-blockid-preview-row.is-invalid`, reused for the `::` advisory
  (the row is still confirmable, so do not introduce a new "error" color).
- `.bob-cnp-schedule-reason-preview` → preview body styled like `.bob-cnp-blockid-preview`, so the rendered bullet reads
  as markdown source.

## Edge cases the implementation must handle

1. **Empty reason** — no entry, and no parent marker. Verified by a test that asserts the note content is byte-identical
   to the date-only write.
2. **Whitespace-only reason** — treated as empty by `normalizeScheduleReasonText`.
3. **Marker already exists** — reused in place, never moved, never duplicated; its own list-marker character (`-`, `*`,
   `+`, `1.`) and indentation are preserved, matching how `formatDependencyNavigationBulletFromDetails` respects
   existing markers.
4. **Marker exists but has no entries yet** — entry indent falls back to marker indent + one tab.
5. **Marker written by hand without the emoji** — recognized by `SCHEDULE_LOG_PARENT_RE` and reused; the emoji is not
   back-filled, so the user's line is never silently rewritten.
6. **Two markers under one task** — the first is used; the second is left alone.
7. **Task is the last line of the file / no trailing newline** — `insertEditorLine` already appends after the last line;
   cover it with a test.
8. **Task inside a blockquote (`> - [ ] #task`)** — `getBulletIndent` preserves the `>` prefix, so the log stays inside
   the quote.
9. **No previous scheduled value** — entry renders as `**<to>** — reason`.
10. **`^prj` task where the previous value lives in frontmatter** — `from` is the frontmatter `scheduled` value, not an
    inline field.
11. **Counted session with mixed previous values** — each task's entry uses its own `from`, from `fromByLine`.
12. **Counted session where some tasks are unchanged** — unchanged tasks get no entry, and the notice count reflects
    only the tasks that were logged.
13. **Blank line directly under the task** — `findCurrentBulletChildBlock` skips blank lines when scanning but only
    extends `endLineExclusive` past non-blank deeper lines, so a task with no children inserts at `taskLine + 1`, above
    any blank line. Assert this with a test; it is what keeps `bob projects sync`'s "stop at the first blank line" scan
    working.
14. **The note changed under the modal** — every writer's existing guard (`getInlinePropertyWriteContext`,
    `getProjectScheduledWriteContext`, `getCountedTaskWriteContext`, `validateCountedTaskSession`) still runs first and
    still aborts the whole write. Do not add a second, weaker guard.

## Testing

All tests go in `scripts/test-navigation-hotkeys.cjs`, matching its existing style: pure functions imported from the
module's `module.exports`, `node:test` + `node:assert/strict`, string fixtures for note content.

Pure-function coverage:

- `formatScheduleLogParentBullet` / `formatScheduleLogEntryBullet` — with and without `from`, with tab and two-space
  indents, with `*` and `+` markers.
- `parseScheduleLogParentBullet` / `parseScheduleLogEntryBullet` — round-trip the formatters; reject a marker line with
  trailing content; accept the emoji-less form.
- `normalizeScheduleReasonText` — trims; collapses newlines and runs of spaces; reports `empty` for `""` and `"   "`;
  sets `hasInlineField` for `"blocked:: x"`; leaves `"blocked by [[sase_gate]]"` untouched.
- `findScheduleLogParent` — finds a marker among mixed children; ignores a marker that belongs to a nested grandchild;
  returns `null` when absent.
- `getScheduleLogEntryIndent` — reuses an existing entry's indent; falls back to marker indent + tab.
- `planScheduleLogEntry` — creates parent + entry as the last direct child; prepends a second entry above an existing
  one; guards on a non-list-item line, an out-of-range line, and an empty reason; handles the last-line-of-file case;
  preserves a blockquote prefix.

Integration-style coverage over content strings:

- `planCountedBulletPropertyBatch` with `options.scheduleLog` — three counted tasks, one of them already carrying a
  schedule log, one unchanged; assert the exact resulting content, `scheduleLoggedTaskCount`, and that `cursorLine` is
  unchanged.
- A counted batch where insertions must not corrupt later targets — assert that descending-order application produces
  the expected content.
- Empty reason through the counted planner — content identical to the existing no-reason expectation.

Modal-level coverage (the harness already stubs `obsidian`, so `BulletPropertyPickerModal` is instantiable):

- Choosing a `scheduled` date leaves the modal open with `stage === "reason"` and writes nothing.
- Choosing a `priority` level does **not** enter the reason stage.
- `getFilteredItems()` in the reason stage returns exactly one item for empty input and for typed input.
- The footer hint label flips between `Skip reason` and `Log reason`.
- `onClose()` during the reason stage clears `pendingScheduleReason` and performs no write.

Run the full suite from the `bob-plugins` repo root:

```bash
npm test
node scripts/validate-manifests.mjs
```

## Docs and release

1. **`plugins/bob-navigation-hotkeys/manifest.json`** — bump `version` to `1.18.0` (feature-level bump, matching commit
   `5442f90`).
2. **`bob-plugins/README.md`** — extend the Bob Navigation Hotkeys description row (line 16) to mention that
   `Ctrl+Shift+P` prompts for an optional reason after a `scheduled` date and records it under a `🗓️ **Schedule log:**`
   child bullet. Keep the version column in sync with the manifest.
3. **`bob-cli/docs/projects.md`** — add a subsection after _"Priority property and scheduled rolls"_ (which ends around
   line 277) documenting: the prompt, the exact markdown shape with a worked example, the newest-first ordering, the
   last-direct-child placement, that an empty reason writes nothing and does not create the marker, that `esc` cancels
   the date too, that priority rolls and `Ctrl+D` do not prompt, and that one reason applies to every task in a counted
   session. Also note in the existing _"Scheduling from the `^prj` task"_ section that the log is written under the
   `^prj` task bullet and is ignored by `bob projects sync`.
4. **Deploy** — from the `bob-plugins` repo root, after committing:

   ```bash
   bob plugins sync -r "$PWD"
   ```

   The explicit `-r "$PWD"` is required; without it `bob plugins sync` resolves the default repo path instead of the
   linked-repo checkout.

## Out of scope (file as follow-up task beads if wanted)

- Prompting for a reason on `Ctrl+D` (unscheduling) or on `priority` rolls.
- A `bob capture` equivalent for writing schedule-log entries from the CLI.
- Any `~/.config/bob/config.yml` schema change. This feature deliberately adds no config surface; `scheduled` is
  hard-scoped in code, consistent with the other `scheduled` special cases already in `main.js`.
- Reading or querying the log (e.g. a `bob query` view of recent reschedule reasons).
