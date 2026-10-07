---
tier: epic
title: In Progress marks and the Alt+[ / Alt+] lane toggle for Pomodoro Task Links
goal: 'In today''s daily note, every Task Link under an open Pomodoro whose task is
  In Progress `[/]` shows a rendered amber half-ring mark (Next shows none), and Alt+[
  / Alt+] on any Pomodoro Task Link toggles its task between Next and In Progress,
  offering an optional Work Log entry when it goes back to Next. Nothing new is ever
  written into the daily note.

  '
phases:
- id: progress-marks
  title: bob-ledger-tools In Progress marks
  depends_on: []
  size: medium
  description: 'progress-marks: add a display-only half-ring mark before In Progress
    Task Links under today''s open Pomodoros (Live Preview widget, Reading view post-processor,
    CSS, toggle command, refresh wiring) plus the `api.progressMarks` v1 hint namespace,
    with conformance tests P1-P14.'
- id: link-lane-toggle
  title: bob-navigation-hotkeys Task Link lane toggle and api.taskLinkLane
  depends_on: []
  size: medium
  description: 'link-lane-toggle: build the two-state Next <-> In Progress toggle
    for Pomodoro Task Links on the Alt+N link pipeline (matcher, start-wins batch
    planner, a styled optional Work Log prompt, preimage-checked commit, notices,
    progressMarks hint) and expose it as nav `api.taskLinkLane` v1, with tests L1-L15.'
- id: cycler-keys
  title: task-status-cycler delegates Alt+[ / Alt+] on Task Links
  depends_on:
  - link-lane-toggle
  size: small
  description: 'cycler-keys: route single and counted Alt+[ / Alt+] presses on Pomodoro
    Task Links to nav `api.taskLinkLane`, skip those lines in counted ranges that
    start elsewhere, and keep every other line''s behavior unchanged.'
- id: docs-verify
  title: Docs, end-to-end verification, and memory follow-up
  depends_on:
  - progress-marks
  - cycler-keys
  size: small
  description: 'docs-verify: document the mark and toggle contract in bob-cli `docs/plan.md`
    (vectors, live checklist, Surfaces/Notices rows), cross-check the plugin README
    rows, run the full plugin suite and sync, and record a PROPOSED FOLLOW-UP for
    memory updates.'
proposed_by: bbugyi200.apollo.5j
create_time: 2026-10-07 10:18:27
status: done
bead_id: bob-cli-56
---

- **PROMPT:** [prompts/202610/in_progress_task_link_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/in_progress_task_link_marks.md)
- **BEAD:** [bob-cli-56](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-56/README.md)

# Plan: In Progress marks and the Alt+[ / Alt+] lane toggle for Pomodoro Task Links

## 1. What Bryan sees

Today's daily note in Live Preview:

```
## Pomodoros
- [x] (**0900-0930** [t:: 30m])
	- 🍅 Fix the parser crash          ← worked in a past session (written text, unchanged)
	- 🍅 Review Ana's PR
- [ ] (**0940-1010** [t:: 30m])        ← running
	- ◐ Fix the parser crash           ← target is In Progress [/] (rendered mark)
	- Write release notes              ← target is Next [*] (no mark)
- [ ] ()                               ← planned
	- ◐ Benchmark the cache            ← In Progress
	- Answer support email             ← Next
```

- `◐` is drawn as a small amber half-filled ring, the same amber as the vault's `[/]`
  checkbox (`--task-status-in-progress`). Hovering it shows a two-line tooltip:
  `In Progress` / `Alt+[ or Alt+] → Next`.
- With the cursor on any Task Link inside a Pomodoro, **Alt+[ or Alt+]** toggles the
  linked task Next ↔ In Progress. The mark appears or disappears instantly, and a short
  Notice confirms the change, for example
  `◐ In Progress · Fix the parser crash · PENDING 8/10 · NEXT 11/15`.
- Going from In Progress back to Next first opens a small **Move to Next** prompt. It
  shows the task, has an optional "What did you get done?" field, and previews the Work
  Log entry live. Enter commits. A blank summary moves the task without logging. Esc
  cancels and writes nothing.

## 2. Design decisions

### D1. The mark is rendered and never written

The 🍅 marker is history that is written into the file. The new mark shows the _current_
status of a task that lives in another note. It is rendered at display time by
bob-ledger-tools and is never written into the daily note. Reasons:

- **Reliability.** Every dedicated-Task-Link parser accepts only `🍅 ` and `!`. These
  include block-id-prompt `DEDICATED_TASK_LINK_PREFIX_RE` and `isDedicatedLinkBullet`,
  nav `LINK_PICKER_DEDICATED_BODY_RE`, `140-scheduled-recovery.js` and
  `120-project-notes.js`, ledger-tools `todayLinkFromBody`, task-status-cycler's bare
  and move-only parsers, and Rust `strip_pomodoro_markers`, `bare_plain_link`,
  `list_queued_links` (Today), `classify_sub_bullets` (`=x`), `list_open_entry_links`
  (the `^` picker), `find_movable_task_links`, and reconcile's 🍅 repair. A written `◐ `
  prefix would drop links out of Today, out of `=x` numbering, and out of Work Log
  routing. It would also make reconcile write `◐ 🍅 [[…]]`, and every one of those
  parsers would need changing.
- **No second copy of state.** Status lives on the task line. Writing it into the ledger
  repeats the `#today`-tag mistake that `decisions:today-is-read-from-the-ledger`
  rejected: churn, races, and staleness.
- **House style.** Date, priority, freshness, and dependency marks are all display-only.
  Without the plugin the vault looks exactly as before.
- **Scope.** No Rust, capture, or Mac Capture changes are needed.

### D2. Glyph, color, and placement

- **Shape.** A ring with its right half filled (a Linear-style "in progress" circle):
  started, not finished. The shape carries the meaning, so the mark never relies on
  color alone.
- **Color.**
  `var(--bob-progress-mark-color, var(--task-status-in-progress, var(--color-yellow, #c69026)))`.
  This uses the status hue the vault already gives `[/]` checkboxes, row tints, and
  dependency chips. The theme hook is unset by default.
- **Placement.** Immediately before the link token, the same slot 🍅 takes under closed
  Pomodoros. This gives one marker slot with two tenses: 🍅 means worked in a past
  session, ◐ means in progress now.
- **Next has no mark.** That matches the request and the "quiet by default" principle.

### D3. Where the mark appears ("truthful or neutral")

A link gets the mark only when all of these hold:

1. The editor's file is **today's** daily note (`currentTodayDailyPath`). At midnight
   rollover the minute tick refreshes the marks.
2. The link sits under an **open** Pomodoro entry (running or planned placeholder, named
   or unnamed; the same open test as `planParseEntry(...).state === "open"`).
3. It is a **dedicated plain** Task Link at the entry's first child indent. The body is
   exactly one `[[target#^id]]` (optional alias), after stripping leading 🍅. An
   optional trailing `#` move-only marker (with or without a space) is allowed.
   Excluded: struck `~~[[…]]~~` links, embedded `![[…]]` links (the transclusion already
   shows its own amber checkbox), fenced lines, deeper nested bullets, and links mixed
   with other text.
4. The target resolves to a list item whose checkbox is exactly `/`.

Everything else shows nothing: Next, Ready, Blocked, done, unresolved, a non-task block,
any other daily note, and links outside `## Pomodoros`.

### D4. Where Alt+[ / Alt+] act as a lane toggle

Alt+[ and Alt+] become a lane toggle on a **Pomodoro Task Link line**. That is a line
nav's `parseLinkPickerTaskLink` accepts (a dedicated bullet: plain, 🍅, `#` move-only,
struck, or embedded) that sits inside a Pomodoro entry's sub-bullet block within the
`## Pomodoros` section of a daily-note-shaped path. The entry can be open or closed.
Toggling from a 🍅 history line is allowed: it changes the task, not the line.

- Alt+[ and Alt+] do the **same** thing there, because a two-state ring has no
  direction. The bullet is **never reformatted**, just as Depends-On lines never are
  (`docs/task-dependencies.md` §8).
- Embedded Pomodoro links switch from the full-ring transcluded cycle to this toggle,
  because they are Task Links too and the rule is "Task Links only toggle Next ↔ In
  Progress". Ctrl+Enter still closes and reopens them.
- **Unchanged:**
  - task/checkbox lines, including the Pomodoro entry line itself and `[x] [[…]]`
    checklist rows;
  - Depends-On lines;
  - embedded links outside Pomodoros;
  - the formatting toggle on every other plain bullet, including plain link bullets
    outside Pomodoros.

### D5. Only two statuses

| Target status                                       | Single press                                                                  | In a counted batch             |
| --------------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------ |
| Next `[*]`                                          | → In Progress `[/]`                                                           | "start"                        |
| In Progress `[/]`                                   | prompt, then → Next `[*]`                                                     | "pause"                        |
| Ready `[ ]`                                         | refused: `Task is Ready · Alt+N commits it to Next`                           | skipped, counted               |
| Blocked `[?]`                                       | refused: `Blocked is derived · clear its dependency or future schedule first` | skipped, counted               |
| Done `[x]` / Cancelled `[-]` (incl. struck links)   | refused: `Task is done · Ctrl+Enter reopens it` / `Task is cancelled`         | skipped, counted               |
| Missing, duplicated, non-task, or unreadable target | the existing resolver error Notice                                            | whole batch refused (as Alt+N) |

### D6. Counts: `N<A-]>` / `N<A-[>`

A count covers the cursor link plus the next N sibling Task Links, clamped at the
Pomodoro entry's end. This is exactly `discoverLinkPickerTargets`, the rule Alt+N uses.
The mode is decided **once, across all note groups**:

- **Start wins.** If any target is Next, every Next target goes to In Progress, and In
  Progress targets are left alone.
- Otherwise, if any target is In Progress, every In Progress target goes to Next. One
  prompt is shown, and its entry is logged on each task, as Alt+N release does.
- Otherwise the batch is refused with the first target's D5 message.

Duplicate targets are deduped.

### D7. Where the toggle lives

**bob-navigation-hotkeys owns the toggle; task-status-cycler owns the keys and
delegates.**

Nav already has the whole Alt+N Task Link pipeline:

- `discoverLinkPickerTargets`;
- `resolveLinkPickerTargets`, which reads open editors first;
- the preimage-verified multi-note `commitLinkPickerNoteWrites` with rollback;
- `insertLaneWorkLogEntry`;
- the freshness stamper;
- `readLaneBudgets`.

Nav exposes `api.taskLinkLane` v1 (§3). `matches()` is the **single** definition of
"Pomodoro Task Link line", so the two plugins can never disagree.

If nav or the namespace is missing, task-status-cycler keeps today's legacy behavior.
That is acceptable because nav is always synced alongside it.

### D8. The Move to Next prompt

The prompt is a new nav modal styled like block-id-prompt's polished `Unlink task`
prompt (`bid-wlp`), so the two Work Log prompts look like one family.

**Layout**

- **Header:** Lucide `pause-circle` tinted with the in-progress amber, the title
  `Move to Next`, and the subtitle
  `Optional: what did you get done? Saved to the Work Log.`
- **Context card:**
  - single task: the task's display text plus the note path;
  - batch: `N tasks`, up to three names, then `+k more`, with the line
    `The same entry is logged on each task.`
- **Field:** labelled `Work summary (optional)`, placeholder `What did you get done?`.
- **Live preview:** `🛠️ **WORK LOG**` / `*YYYY-MM-DD* — summary`, or
  `No work-log entry will be added.` when the field is blank. It is an
  `aria-live="polite"` region.
- **Warning:** the `::` Dataview warning.
- **Buttons:** `Cancel` and a call-to-action button that reads `Move to Next` when the
  field is blank and `Log & move to Next` when it has text.

**Behavior**

- The input is focused on open.
- Enter submits, but not while an IME composition is active (`event.isComposing`).
- Esc or Cancel cancels the whole gesture. Nothing is written and no Notice is shown.
- Summaries are normalized with nav's `normalizeLaneWorkSummary`.
- The entry is dated with today's local date (`laneReleaseDateText`), uses the same
  grammar, and is prepended newest-first under an existing direct-child 🛠️ WORK LOG or a
  newly created one.

### D9. Instant feedback

Target notes that are open in an editor update the metadata cache only after autosave,
which can take about 2 s. To make the mark flip on the same frame, nav calls
ledger-tools `api.progressMarks.expect([{ path, blockId, status }])` after every
successful commit.

- The hint overrides the cached status for that block until either the cache reports the
  same status or **4 s** pass, whichever comes first.
- Setting a hint refreshes the mark editors immediately, with no debounce.
- A missing or old ledger-tools api makes `expect` a no-op, and the marks fall back to
  cache-driven refresh.

### D10. Fit with the hooks and the decisions

- **Sticky lanes still hold** (`decisions:task-lanes-are-sticky`). Only release lowers a
  task to Ready; this gesture moves it between Next and Pending, never to Ready.
- **Freshness.** The gesture stamps `[fresh:: today]`, as every lane gesture does.
- **Reconcile.** `bob task reconcile` never lowers `[/]` and leaves a directly linked
  `[*]` alone, so it keeps both directions. One documented edge case: a task that is a
  prerequisite of In Progress work reachable from today's open Pomodoros is raised back
  to `[/]` by dependency inheritance.
- **Blocked.** Blocked stays derived and is never a destination (D5).
- **Review walk.** Task Links are never walk landings, so the gesture neither captures
  nor advances the walk.

## 3. Cross-plugin contracts

Both contracts are additive namespaces. The top-level api versions stay as they are
(ledger-tools v3, nav v3), and consumers feature-detect the namespace version.

**ledger-tools `api.progressMarks`** (frozen, every member synchronous, never throws):

```js
progressMarks: {
  version: 1,
  // Optimistic statuses just written by a lane gesture. Each entry:
  // { path: "<vault path>.md", blockId: "abc", status: "/" | "*" | " " | ... }.
  // Overrides the cached status until the cache agrees or 4 s pass; refreshes now.
  expect(entries) {},
  refresh() {},          // schedule a mark refresh (debounced 150 ms)
  isEnabled() {},        // session toggle state
}
```

**nav `api.taskLinkLane`** (frozen):

```js
taskLinkLane: {
  version: 1,
  // Synchronous, never throws. True when `line` (0-based) of `content` is a
  // Pomodoro Task Link line (D4) in a daily-note-shaped `path`.
  matches({ content, line, path }) {},
  // Runs the D5/D6 toggle for the cursor link (+ N siblings when countExplicit).
  // Uses only the passed count (never re-reads Vim state). Resolves
  // { ok: true, mode: "start" | "pause", changed } or { ok: false, reason }
  // (reasons: "busy" | "cancelled" | "refused" | "stale" | "error" | "unavailable").
  // Never rejects. A second call while one is in flight resolves
  // { ok: false, reason: "busy" } with no write and no Notice.
  toggle({ editor, view, countExplicit = false, additionalTaskCount = 0 }) {},
}
```

## 4. Conformance vectors

The tests encode these vectors, and `docs/plan.md` repeats them verbatim. Notation: `D`
is today's daily note, and `T` is a project note holding `- [s] #task Fix ^fix`.

**Marks (P)**

| ID  | Setup                                                                             | Mark?                                                              |
| --- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| P1  | open timed entry, `\t- [[T#^fix]]`, `s=/`                                         | yes, before `[[`                                                   |
| P2  | same, `s=*`                                                                       | no                                                                 |
| P3  | open placeholder `- [ ] ()` entry, `s=/`                                          | yes                                                                |
| P4  | named open entry `- [ ] () — GOALS`, aliased `[[T#^fix\|Fix]]`, `s=/`             | yes                                                                |
| P5  | closed entry `- [x] (…)` with `\t- 🍅 [[T#^fix]]`, `s=/`                          | no                                                                 |
| P6  | open entry, `\t- ![[T#^fix]]`, `s=/`                                              | no (embed shows its own checkbox)                                  |
| P7  | open entry, `\t- ~~[[T#^fix]]~~`, `s=/`                                           | no                                                                 |
| P8  | open entry, `\t- [[T#^fix]]#` and `\t- [[T#^fix]] #`, `s=/`                       | yes, both                                                          |
| P9  | open entry, nested `\t\t- [[T#^fix]]`, or `\t- read [[T#^fix]] today`, `s=/`      | no                                                                 |
| P10 | `[[T#^fix]]` inside a fence, outside `## Pomodoros`, or in yesterday's daily note | no                                                                 |
| P11 | unresolved target, or a non-task block id                                         | no                                                                 |
| P12 | `s` is ` `, `?`, `x`, or `-`                                                      | no                                                                 |
| P13 | target task lives in `D` itself, edited live to `[/]`                             | yes, read from the live doc without waiting for the cache          |
| P14 | `expect([{T, fix, "/"}])` while the cache still says `*`                          | yes until the cache says `/` or 4 s pass; after 4 s the cache wins |

**Toggle (L)**

| ID  | Cursor / setup                                                                         | Result                                                                                                                            |
| --- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| L1  | Alt+] on a Next link                                                                   | `[*]→[/]`, `[fresh:: today]` stamped, no prompt, Notice `◐ In Progress · Fix …`, mark appears                                     |
| L2  | Alt+[ on an In Progress link, blank summary                                            | prompt, `[/]→[*]`, no Work Log entry, Notice `→ Next · Fix …`                                                                     |
| L3  | as L2 with `Finished the API review`                                                   | entry `*YYYY-MM-DD* — Finished the API review` prepended under the existing 🛠️ WORK LOG (or a new marker), Notice ends `· logged` |
| L4  | as L2, Esc                                                                             | nothing written, no Notice                                                                                                        |
| L5  | Alt+[ vs Alt+]                                                                         | identical results                                                                                                                 |
| L6  | Ready / Blocked / done / cancelled / struck-done target                                | refusal Notices from D5, nothing written                                                                                          |
| L7  | `2<A-]>` over `[*] [/] [*]` siblings                                                   | both Next become In Progress, the In Progress one is untouched, no prompt                                                         |
| L8  | `2<A-]>` over `[/] [/] [?]`                                                            | one prompt, both In Progress become Next with the same entry, Notice includes `1 Blocked skipped — Blocked is derived`            |
| L9  | `5<A-]>` with 2 siblings left                                                          | clamped at the entry end, no error                                                                                                |
| L10 | Alt+] on a closed-entry `🍅 [[T#^fix]]` line                                           | toggles the task; the line is unchanged                                                                                           |
| L11 | target edited while the prompt is open                                                 | `A linked note changed; no tasks were updated`, nothing written                                                                   |
| L12 | embedded `![[T#^fix]]` under a Pomodoro                                                | two-state toggle (no longer the full ring)                                                                                        |
| L13 | Pomodoro entry line, a task line, a Depends-On line, or a plain link outside Pomodoros | behavior exactly as before                                                                                                        |
| L14 | the same task linked twice in a counted batch                                          | written once                                                                                                                      |
| L15 | second Alt+] while a toggle or prompt is in flight                                     | swallowed (`busy`), no write                                                                                                      |

## 5. Phase `progress-marks`: bob-ledger-tools In Progress marks

Work in the `bob-plugins` linked repo (`sase repo open bob-plugins`, then read its
`AGENTS.md`). Edit only `plugins/bob-ledger-tools/src/` fragments, then run
`npm run build`. Never hand-edit `main.js`, and keep each fragment at or under 1000
lines. Templates to mirror: priority marks (`135-priority-marks.js`,
`265-plugin-priority-marks.js`), dependency chips (`270-plugin-dependency-model.js`,
`280-plugin-dependency-render.js`, for link-to-status lookups and Reading view), and the
Today model (`060-today.js`, `290-plugin-today-and-location.js`).

1. **Pure model in `137-progress-marks.js`** (new; add it to `src/fragments.json` after
   `136-date-marks.js`).
   - `progressMarkLinks(content)` returns `[{ line, ch, target, blockId }]` for the D3
     rules 2–3, where `ch` is the line-relative offset of `[[`.
   - Reuse `planSectionRange`, `planFencedLines`, `planParseEntry`,
     `todayIsSubBulletLine`, `todayIndentLen`, `todayBulletBody`, `todayWikilinkTokens`,
     and `planStruckInnerSpans`.
   - Implement a `progressMarkLinkFromBody(body)` that strips leading 🍅 and an optional
     trailing `[ \t]*#`, then requires a bare, non-struck, non-embedded token. Leave
     `todayLinkFromBody` untouched, because Today still excludes `#` lines.
   - Also add the `PROGRESS_MARK_TOOLTIP` lines (`In Progress`, `Alt+[ or Alt+] → Next`;
     never `::`) and `ProgressMarkWidget`:
     - it renders
       `span.bob-progress-mark[role=img][aria-label][data-tooltip-position=top][data-block-id]`;
     - `eq` compares `blockId`;
     - on mousedown it places the cursor at the link start, never opens the link, and
       never writes.
2. **Status lookup in the mixin `268-plugin-progress-marks.js`** (new; add it to
   `fragments.json` and install it in `310-install-methods.js`).
   - `progressMarkStatusFor(target, blockId, dailyPath, liveDocText)`:
     1. Resolve with `metadataCache.getFirstLinkpathDest` (an empty target means the
        daily note itself).
     2. If the target is the daily note, scan the live doc for the line carrying the
        standalone `^blockId` and read its checkbox (P13).
     3. Otherwise use `getFileCache(file)`: look up `blocks` (try the exact id, then the
        lower-cased id), find the `listItems` entry starting on that block's line, and
        read `.task`.
     4. An unexpired hint wins (P14). Any failure returns `null`.
   - Hints live in a `Map` keyed `path\0blockId` → `{ status, expiresAt }`. They are
     dropped when the cache agrees or when they expire.
3. **Live Preview.**
   - `setupProgressMarks()` is called from onload next to the priority and date mark
     setup. It handles the `progressMarksEnabled` default `true`, the
     `body.bob-progress-marks` class, the `toggle-progress-marks` command ("Toggle Task
     Link In Progress marks"), the editor extension, and the post-processor at sort
     order 50.
   - `createProgressMarkExtension()` is a `Prec.highest(ViewPlugin)` that builds only
     when `editorLivePreviewField` is true and the `editorInfoField` file path equals
     today's daily path.
   - It places `Decoration.widget({ widget, side: -1 })` at each link's `[[`. It parses
     the whole Pomodoros section once per doc version, because daily notes are small and
     the open/closed state needs the entry lines, but emits widgets only inside
     `view.visibleRanges`.
   - It rebuilds on `docChanged`, `viewportChanged`, a live-preview toggle, or the
     lazily defined refresh effect. It does **not** rebuild on `selectionSet`: the mark
     covers no source text, so it stays visible while the cursor is on the line.
4. **Reading view.**
   - `renderProgressMarksIn(el, ctx)` runs only for today's daily path. It maps each
     `li` back to its source line (`ctx.getSectionInfo(el)` plus the list item's line
     data, as the dependency-chip renderer does).
   - It inserts the same span before the first `a.internal-link` of each marked line,
     deduped with `data-bob-progress-processed`.
5. **Refresh wiring.**
   - Add a lazily defined `ensureProgressMarksRefresh()` StateEffect in
     `010-load-and-constants.js`. It must **not** be defined eagerly: the freshness-mark
     surfaces suite asserts a single eager define.
   - Add `scheduleProgressMarksRefresh()` (150 ms debounce) and
     `refreshProgressMarkEditors()` (dispatch to every `leaf.view.editor.cm` whose file
     is today's daily note).
   - Hook the refresh into `scheduleLiveWidgetRefresh()` in
     `220-plugin-heading-and-review.js`.
   - In the metadataCache `changed` handler in `170-plugin-lifecycle.js`, schedule a
     refresh when the changed path is today's daily note or one of the last build's
     target paths. Track these in a `Set` refreshed on each build.
   - Refresh from the midnight minute tick when the daily path rolls over.
6. **API.** Add `progressMarks: this.progressMarksApi()` to the frozen api in
   `170-plugin-lifecycle.js`, following §3. `expect` validates its entries, stores the
   hints, and calls `refreshProgressMarkEditors()` immediately.
7. **CSS** in `plugins/bob-ledger-tools/styles.css`, as a new commented block in the
   style of the priority marks comment.
   - The glyph mask is defined once on `body`:

     ```css
     --bob-progress-glyph: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 16 16'%3E%3Ccircle cx='8' cy='8' r='6.05' fill='none' stroke='black' stroke-width='1.9'/%3E%3Cpath d='M8 4.1A3.9 3.9 0 0 1 8 11.9Z'/%3E%3C/svg%3E");
     ```

   - `.bob-progress-mark`:
     - box: `inline-flex`, centered, a `1.15em` box, `margin-inline-end: 0.3em`,
       `vertical-align: -0.18em`, `border-radius: 999px`;
     - color: `--bob-progress-ink` from the D2 color stack;
     - state: `opacity: 0.92`, `user-select: none`, `cursor: default`;
     - motion: 120 ms opacity/background transitions and a 160 ms `bob-progress-mark-in`
       entrance (opacity 0→0.92, scale 0.6→1).
   - `::before` draws a `1em` masked square filled with the ink.
   - Hover sets opacity 1 and a 12% `color-mix` capsule.
   - `body:not(.bob-progress-marks) .bob-progress-mark { display: none; }`.
   - `prefers-reduced-motion: reduce` turns off the animation and transitions.
   - The glyph must not change line height.

8. **Tests.** Add `scripts/test-ledger-tools-progress-marks.cjs` (using
   `scripts/ledger-tools-harness.cjs`) to the `test` script in `package.json`.
   - Encode P1–P14.
   - Cover the widget DOM: classes, aria-label, the tooltip having no `::`, and `eq`.
   - Check the extension builds nothing outside Live Preview or outside today's daily
     note.
   - Check the Reading-view post-processor.
   - Check `api.progressMarks` shape, validation, and the never-throws behavior.
   - Check rebuild triggers, and that the existing eager-StateEffect assertion still
     passes.
9. **Ship.**
   - Bump `manifest.json` from 1.32.1 to 1.33.0.
   - Add one sentence about In Progress marks to the ledger-tools row of the bob-plugins
     `README.md`, and update the version cell.
   - Run `npm run build`, `npm test`, and `npm run validate`, then `bob plugins sync`.

## 6. Phase `link-lane-toggle`: nav Task Link lane toggle and `api.taskLinkLane`

Work in the `bob-plugins` linked repo, in `plugins/bob-navigation-hotkeys/src/`, under
the same fragment rules. Templates to mirror: `toggleTaskLaneOnLinks`
(`520-plugin-lane-links-and-review.js`), `planTaskLaneBatch` and `buildLaneToggleNotice`
(`230-schedule-review.js`), and `commitLinkPickerNoteWrites`
(`510-plugin-link-commit-and-lane.js`).

1. **Pure core in `245-task-link-lane.js`** (new).
   - `isPomodoroTaskLinkLine(content, line, path)` implements D4.
     - It checks that the path has daily-note shape, using the same recognizer nav
       already uses for Pomodoro work.
     - It checks that the line is not fenced, not a task line, and accepted by
       `parseLinkPickerTaskLink`.
     - It checks that the line is indented under a column-0 Pomodoro entry inside
       `## Pomodoros`, using nav's `140-scheduled-recovery.js` section and entry
       helpers.
   - `decideTaskLinkLaneMode(statuses)` returns `"start" | "pause" | null` (D6).
   - `planTaskLinkLaneBatch(content, session, { mode, summary, dateText, stampLine, freshDateText })`.
     It is shaped like `planTaskLaneBatch` and returns the same frozen fields plus
     `started`, `paused`, `readySkipped`, `blockedSkipped`, `closedSkipped`, and
     `workLogWrittenCount`.
     - `mode` is **passed in** and decided once across groups. Do not copy
       `toggleTaskLaneOnLinks`'s per-group mode.
     - It refuses the whole batch on any stale preimage.
     - It writes bottom-up.
     - The freshness stamp is the last line transformation.
     - In pause mode it writes `insertLaneWorkLogEntry` for each paused task when the
       summary is non-blank.
   - `buildTaskLinkLaneNotice(details)` produces the D5 refusals and the L1–L8 success
     texts.
     - Successes: `◐ In Progress · <task|N tasks>` and `→ Next · <task|N tasks>`.
     - Optional suffixes: `· logged` (or `· logged N`), skipped counts, and lane budget
       suffixes from `readLaneBudgets` adjusted by ±moved for **both** PENDING and NEXT,
       with Alt+N's 🔴 marking for an over-cap lane.
     - Task text is truncated to about 60 characters with `…`.
   - `taskLinkLanePromptState(summary, { date })` returns
     `{ isBlank, primaryButtonText, formattedEntry, hasDataviewWarning, summary, date }`.
2. **Resolver option.** Give `findUniqueLinkPickerTargetLine` (in `220`) and
   `resolveLinkPickerTargets` (in `500`) an `{ allowClosed }` option.
   - The default `false` keeps Alt+N and the picker byte-identical.
   - With `true`, closed targets resolve with their status instead of failing, so the
     planner can skip them (L6, L8).
   - Missing, duplicated, and non-task targets still fail the whole operation.
3. **Prompt in `196-task-link-lane-modal.js`** (new).
   - `TaskLinkLanePromptModal` implements D8 and calls back with `null` on cancel or
     `{ summary }` on submit.
   - Add the styles to nav `styles.css` as `.bob-tll-*` classes. Port the
     block-id-prompt `bid-wlp` rules (`plugins/block-id-prompt/styles.css`, "Work Log
     pause prompt") with the header icon tinted `--task-status-in-progress`.
   - The modal must work under `scripts/modal-harness.cjs`.
4. **Orchestration in `525-plugin-task-link-lane.js`** (new mixin; install it in
   `690-install-plugin-mixins.js`).
   `toggleTaskLinkLane({ editor, view, countExplicit, additionalTaskCount })` runs:
   1. Check the in-flight flag. If set, return `busy` with no Notice (L15).
   2. Discover targets with `discoverLinkPickerTargets`.
   3. Resolve them with `allowClosed: true`.
   4. Re-check the editor content.
   5. Group targets by note and decide the mode once.
   6. Show the refusal Notice when there is no mode.
   7. Prompt when the mode is pause and an In Progress target exists. Cancel ends the
      gesture silently.
   8. Plan every group, passing the freshness stamper and date.
   9. Commit with `commitLinkPickerNoteWrites(planned, {})` (no Pomodoro prune).
   10. On success, call ledger-tools `api.progressMarks.expect` (feature-detected
       `version >= 1`) for every changed target, then show the Notice.
   - Stale preimages produce `A linked note changed; no tasks were updated`.
   - Clear the flag in `finally`.
5. **API.** Add `taskLinkLane: createTaskLinkLaneApi(plugin)` to
   `createDependencyNavApi` in `480-review-jump-and-nav-api.js`, with the factory kept
   in `525`.
   - Follow §3 exactly: frozen, `matches` synchronous and never throwing, and `toggle`
     settling to a result object and never rejecting.
   - Update the api comment block.
6. **Tests.** Add `scripts/test-navigation-hotkeys-task-link-lane.cjs` (using
   `scripts/navigation-hotkeys-harness.cjs`) to `package.json`.
   - Encode L1–L11 and L14–L15 at the nav level: the matcher, all planner branches, mode
     decision, prompt state, modal cancel/submit, notices and budgets, the resolver
     option, the `expect` call, and the api shape.
   - Confirm the existing lane-toggle and link-picker suites still pass unchanged
     (`allowClosed` defaults off).
7. **Ship.**
   - Bump `manifest.json` from 2.10.1 to 2.11.0.
   - Add a sentence to the nav row of `README.md` (the toggle, its prompt, and
     `api.taskLinkLane`).
   - Build, run `npm test` and `npm run validate`, then `bob plugins sync`.

## 7. Phase `cycler-keys`: task-status-cycler delegation

Work in the `bob-plugins` linked repo, in `plugins/task-status-cycler/src/`.

1. **`getTaskLinkLaneApi()`** goes next to `getReviewWalkApi()` in
   `160-plugin-completion.js`. It returns nav `api.taskLinkLane` when `version >= 1` and
   both functions exist, and `null` otherwise. It never throws.
2. **`handleCycleCommand`** (in `150-plugin-commands.js`). Directly after the
   task-status branch:
   - If the api exists and
     `matches({ content: editor.getValue(), line: cursor.line, path: activePath })` is
     true, the checking phase returns `true`.
   - The exec phase calls `void api.toggle({ editor, view })` and returns `true`.
     `direction` is ignored (D4).
   - This runs before the Depends-On, transcluded, and formatting branches, so embedded
     Pomodoro links are routed too (L12).
3. **Counted listener.** In `dispatchCountedTaskCycleEvent`, when there is no task
   status and `matches(...)` is true:
   1. Consume the event exactly as the existing branch does.
   2. Call `resetPendingVimInputState(cm, "counted-task-link-lane")`.
   3. Call
      `void api.toggle({ editor, view, countExplicit: true, additionalTaskCount: pendingRepeat.repeat })`.
4. **Counted ranges that start on a task line.** `cycleTaskStatusRange` skips lines
   where `matches(...)` is true. Embedded Pomodoro links in such a range are no longer
   full-ring cycled.
5. **No nav api.** Every branch above is skipped and today's behavior remains (D7).
6. **Tests.** Add `scripts/test-task-status-cycler-task-link-lane.cjs` (using
   `scripts/task-status-cycler-harness.cjs` with a stub nav api) to `package.json`.
   - Single and counted delegation, with the exact arguments passed.
   - L5 (both keys).
   - L12.
   - L13 (Pomodoro entry line, task line, Depends-On line, plain link outside Pomodoros,
     and embedded link outside Pomodoros all unchanged).
   - The range skip.
   - Legacy fallback with no api.
   - Existing suites must stay green.
7. **Ship.**
   - Bump `manifest.json` from 1.26.0 to 1.27.0.
   - In the TSC row of `README.md`, add the Task Link toggle to the Alt+[ / Alt+]
     sentences, and note that it never reformats a Task Link bullet.
   - Build, test, validate, `bob plugins sync`.

## 8. Phase `docs-verify`: docs, verification, and memory follow-up

1. **bob-cli `docs/plan.md`.** Add `## In Progress marks and the Task Link lane toggle`
   right after `## Lanes (NEXT and PENDING)`, covering:
   - what you see (the §1 mock);
   - principles (D1–D3, including the rejected text-marker alternative and why);
   - glyph anatomy (geometry, color stack, theme hook, motion, reduced motion);
   - toggle scope and the D5 table;
   - counts (D6);
   - the prompt (D8);
   - Notices;
   - instant feedback (D9);
   - the hooks interplay and dependency re-promotion edge (D10);
   - both api namespaces (§3);
   - the P and L vectors verbatim;
   - a `Live verification (Bryan, in Obsidian)` checklist covering light and dark,
     cursor on line, toggling with the target open in a split, a counted batch, Esc, and
     Reading view.
2. **Other bob-cli docs.**
   - In `docs/plan.md` § Surfaces, update the "Daily note…" row to mention the marks,
     and add the new Notice examples to the "Obsidian Notices" row.
   - Add a one-line cross-reference in `docs/task-status-hooks.md` next to the
     sticky-lane summary: Alt+[ / Alt+] move a linked task between Next and In Progress,
     and reconcile keeps both.
   - Add a one-line keymap mention in `docs/getting-started.md` near the other Task Link
     gestures.
3. **Cross-check.** Confirm the three bob-plugins `README.md` rows and versions match
   the shipped behavior, and fix any drift in the linked repo.
4. **Verify.**
   - In bob-plugins, run `npm run build:check`, `npm test`, and `npm run validate`, then
     `bob plugins sync`.
   - In bob-cli, run its documentation lint and format checks (`just lint` / `just fmt`
     when they cover Markdown).
   - Run one combined harness check of the end-to-end path: TSC Alt+] on a Task Link →
     the nav toggle → the target written → `progressMarks.expect` → the mark built. Put
     it in the nav or TSC test file, using the real fragments where the harnesses allow.
5. **Memory follow-up.** Do not edit memory. Record a `PROPOSED FOLLOW-UP:` note on this
   phase bead proposing:
   - the `glossary:work-log` strand gains the Alt+[ / Alt+] In Progress → Next prompt as
     a trigger;
   - a new `decisions` record "In Progress marks are rendered, never written; Pomodoro
     Task Links toggle only Next ↔ In Progress", citing this epic.

## 9. Non-goals

- No text is written to daily notes. No Rust, `bob capture`, reconcile, or Bob Mac
  Capture changes.
- No mark for Next, Ready, or Blocked, and no mark in Tasks query results, `bob plan`,
  or the dash.
- Ctrl+Shift+Enter, Alt+N, Ctrl+Enter, and Alt+N's release modal look are unchanged.
  Restyling that modal to match is a possible later follow-up.
