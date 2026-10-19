---
tier: epic
title: "Today marks: a 🍅 on task lines that are on today's plan"
goal: "Every open #task line whose task is on today's plan (it has a dedicated Task Link
  under one of today's open Pomodoros) shows a display-only 🍅 Today mark right after
  its identity glyph. The mark appears in Live Preview, and in rendered views wherever
  the line's identity is exact. Its tooltip names the Pomodoro and whether it is running
  or queued. Mod+click and a keyboard command jump to the Task Link, and a session
  toggle hides every mark. The mark is truthful or neutral: it never guesses, never goes
  stale, and is never written.

  "
decisions:
  keyboard_jump:
    ask:
      Ship a "Go to today's Task Link" command, the keyboard twin of Mod+click, in v1?
    default: true
    why: Bryan is vim-first; without it the mark's only action needs the mouse
    answer: true
  memory_today_mark_record:
    ask:
      Record the Today mark design as a new decisions-web record
      (decisions:today-mark-shows-ledger-today)?
    memory:
      - decisions:today-mark-shows-ledger-today
    default: false
    answer: false
phases:
  - id: contract
    title: Today marks display contract in bob-cli docs
    depends_on: []
    size: small
    description:
      "contract: write the authoritative docs/today-marks.md display contract with
      conformance vectors TM1–TM23, link it from docs/README.md and docs/plan.md §Today,
      and (only if the memory decision is yes) add the decisions-web record."
  - id: cache
    title: Today cache details, generation guard, and revalidation
    depends_on: []
    size: medium
    description:
      "cache: extend the bob-ledger-tools Today cache with per-key Pomodoro details, a
      per-note id index, entered-key timestamps, and a details signature that refreshes
      only the marks. Add an async generation guard and focus/visibility revalidation,
      all with clock-injected tests."
  - id: mark_core
    title: Today mark pure core, widget, and styles
    depends_on:
      - cache
    size: medium
    description:
      "mark_core: add src/143-today-marks.js (line candidate, insert offset, duplicate
      ids, session text, tooltip, listener-free element, TodayMarkWidget) plus the
      styles.css block and a new test suite for the pure vectors."
  - id: live_preview
    title: Today marks in Live Preview with toggle and jump
    depends_on:
      - contract
      - mark_core
    size: medium
    description:
      "live_preview: add the src/275-plugin-today-marks.js mixin (ViewPlugin
      decorations, refresh fan-out, session toggle, the Mod+click and keyboard jump to
      the Task Link) and lifecycle wiring. Run the TM vectors against Live Preview, then
      bump, build, test, and sync."
  - id: reading
    title: Today marks in rendered views
    depends_on:
      - live_preview
    size: medium
    description:
      "reading: add src/276-plugin-today-marks-reading.js. Rendered rows get marks only
      on exact section mapping and stay neutral otherwise. Add row stamping with a live
      refresh sweep, the daily-note view class, and delegated Mod+click, then test,
      bump, and sync."
proposed_by: bbugyi200.athena.11r
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-19 09:47:16
status: wip
---

- **PROMPT:**
  [prompts/202610/today_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/today_marks.md)

# Plan: Today marks

## Context

Since sticky lanes (`decisions:task-lanes-are-sticky`), a `[*]` or `[/]` checkbox means
"committed", not "on today's plan". Nothing on a task's own line says whether it is on
today's plan, so in an area or project note the Next & In Progress rows look the same
either way. The research report
`research:202610/today_task_line_marks/today_task_line_marks.md` measured the gap: 29
open Next/Pending tasks against 13 on today's plan, and 5 of 8 rows in `sase.md` are on
the plan. The report proposed a design, and Bryan accepted every requirement it
recommends. This epic builds that design. The few places where this plan refines the
report are listed under "Design calls this plan makes".

Ownership is **bob-ledger-tools only** (plus bob-cli docs). There are no Rust, capture
grammar, CLI, or Mac changes, and `cli_rules.md` is untouched.

- **bob-cli** (this repo): `docs/today-marks.md`, `docs/README.md`, `docs/plan.md`, and
  optionally one memory strand.
- **bob-plugins** (linked repo, open it with `/sase_repo`:
  `sase repo open bob-plugins`): `plugins/bob-ledger-tools/` only. Read its `AGENTS.md`
  first. Edit `src/` fragments, never `main.js`. Each hand-edited fragment must stay ≤
  1000 lines. Run `npm run build`, then `npm test`, then `bob plugins sync` at the end
  of every bob-plugins phase.

Baseline: `scripts/test-ledger-tools-unblocked-glyph.cjs` already fails 4 tests because
it mixes a wall-clock daily path with fixed October 9 dates (tracked as `bob-cli-6v`).
Those 4 failures are pre-existing. Do not "fix" them here, and do not copy the pattern:
**every new test injects its clock.**

## The design (canonical spec; the contract doc transcribes it)

### Which tasks get the mark

The mark means exactly **Today** as `docs/plan.md` §Today defines it: open `#task` lines
with a dedicated Task Link under one of today's **open** Pomodoros. It is the same set
as dash's TODAY section, `bob plan`, and the `x` lineup
(`decisions:today-is-read-from-the-ledger`). It is read from the existing Today cache
(`todayCache.rank`) and is never stored. It does not mean residence in the daily note
(that would mark only `^gtd`), and it does not mean history (tasks linked only under
closed 🍅 Pomodoros get no mark in v1).

A task line shows the mark only when **all** of these hold:

1. **The cache is ready:** `isTodayCacheReady(now)` holds for today's date and daily
   path.
2. **The live line is an open task:** `freshnessTaskStatus(line)` is one of ` `, `*`,
   `/`, `?`. The cache is a link-key set and does not check rule 5, so the mark applies
   it itself. Completing a task removes its mark on the same frame.
3. **The line is a real `#task`:** it has an exact `#task` tag (the task-tag-mark token
   rule, `taskTagMarkRanges`) that is not inside inline code or a fence, and it ends in
   a standalone `^id`.
4. **The key is in Today:** the key `"<host path>.md#<id>"` is in `todayCache.rank`.
5. **The id is unique in the host note.** If several lines in the note end in the same
   `^id`, none of them gets a mark. Rust `today_tasks` treats a duplicate the same way:
   it reports `today_link_unresolved` and skips the link.
6. **The host is not a canonical daily note** (`PLAN_DAILY_PATH_RE`, any date) and is
   not rendered inside one.
7. **The session toggle is on.**

### Placement

The mark goes at the first non-space character after the identity slot, before the
title. The identity slot is the leading exact `#task` token, or the `#task #ref` pair
(📖). On `#task #prj` lines the mark sits between the hash and the `#prj` pill. When
`#task` is not the body's first token, the mark goes at the body start, right after the
checkbox and its space.

In Live Preview the mark is a zero-width `Decoration.widget({ side: -1 })` at that
offset:

- A caret at the title start draws after the mark.
- The widget occupies no document positions, so vim motions (`0`, `^`, `I`, `w`) are
  unchanged.
- It covers no text, so it stays visible with the cursor on the line, including when the
  cursor reveals the raw `#task`.
- It never rebuilds on `selectionSet`.

Unmarked rows reserve no space.

### Glyph anatomy (beautiful, quiet, rare)

The mark is 🍅, Bryan's own Pomodoro symbol. On a Bob task row it already means "linked
under today's Pomodoro" (the Blocks-stage picker badge). It appears on only about 5–15
lines a day, so it is meant to be noticed: one warm accent in a clean column on
otherwise quiet lines.

- **Glyph source:** the host is an **empty** `span.bob-today-mark`. CSS `::before` draws
  the glyph from `content: var(--bob-today-mark-glyph, "🍅")`. Copying a line never
  picks up 🍅, and a theme snippet can swap in a mask with no code change.
- **Color emoji:** `::before` uses the font stack
  `"Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji", sans-serif` and
  `font-variant-emoji: emoji`. This guarantees the color tomato, never a monochrome text
  glyph.
- **Box metrics:** `inline-flex`, centered, `1.2em × 1.2em`, `font-size: 0.85em`,
  `line-height: 1`, `vertical-align: -0.14em`, `margin-inline: 0.06em 0.3em`,
  `border-radius: 999px`, `user-select: none`, `cursor: default`. The box never changes
  line height.
- **Hover:** a capsule tinted with the line's own status accent,
  `color-mix(in srgb, var(--task-status-accent, var(--text-muted)) 14%, transparent)`.
  The vault's `task-statuses.css` sets `--task-status-accent` per status line.
  Transition is `background-color 120ms ease`.
- **Entrance:** plays only when the key joined Today within the last 600 ms, for example
  right after Ctrl+Shift+Enter. It never plays on startup, on scroll, on rerender, or at
  midnight. The animation is
  `bob-today-mark-in 180ms cubic-bezier(0.2, 0.9, 0.3, 1.25)`, from
  `opacity: 0; scale: 0.5` to `opacity: 1; scale: 1`: a gentle pop, like setting a
  tomato down.
- **Accessibility and print:**
  - `prefers-reduced-motion: reduce` turns off the animation and the transition.
  - `forced-colors: active` makes the hover capsule `Canvas` with a
    `1px solid CanvasText` outline.
  - `print-color-adjust: exact`.
- **Toggle gate:** opt-out, as
  `body.bob-today-marks-off .bob-today-mark { display: none; }`. It is not
  `body:not(.bob-today-marks)`, because plugin body classes exist only on the main
  window and an opt-in gate would hide marks in popout windows (see `bob-cli-9q`). JS
  gating (the decoration builder and the post-processor check the flag) stays the source
  of truth.

### Tooltip and accessibility

The tooltip is the element's `aria-label` (Obsidian's tooltip, with
`data-tooltip-position="top"`). There is no `title`, so no tooltip shows twice. It has
two lines joined with `\n` and never contains `::`, because Dataview re-scans
`innerHTML`.

- **Line 1:** `Today · running in <session>` for the unique open timed entry (the
  Blocks-stage running rule), otherwise `Today · queued in <session>`. When more than
  one open entry links the task, it gets a ` +N more` suffix. The primary entry is the
  running one, else the first in ledger order.
- **`<session>`:**
  - A named entry gives its name plus its range when timed: `GTD 0725–0815`, `BLOG`.
  - An unnamed entry gives its derived Pomodoro home label (`pomodoroHomeModels`, for
    example `bob · dev +1`) plus its range when timed.
  - An unnamed entry with no derived label gives its range alone (`0940–1010`), else
    `unnamed Pomodoro`.
  - Ranges use an en dash.
  - The word "in" ends the state phrase, so interpuncts inside a home label never read
    as separators.
- **Line 2:** `⌘-click to open in today's note` on macOS (`Platform.isMacOS`),
  `Ctrl-click to open in today's note` elsewhere, and omitted on mobile
  (`Platform.isMobile`).
- The line itself never gets the word "today", and the mark adds no tab stop and no
  live-region announcement.

### Interaction

- **Plain click** places the cursor at the title start (`view.posAtDOM(dom)`) and
  focuses the editor. It never writes and never navigates; this is the mark-family rule.
- **Mod+click** (Cmd on macOS, Ctrl elsewhere, primary button) opens today's daily note
  at the Task Link line:
  - If an open leaf already shows the daily note (`findOpenMarkdownLeafByPath` and
    `activateMarkdownLeaf`), that leaf is reused. Otherwise the note opens exactly as
    the existing "Open today's daily note" command opens it
    (`openTodayDailyNoteWithState`).
  - The cursor goes to the start of the link token on the primary entry's line, with the
    line scrolled to center.
  - If the cached `ledgerLine` no longer holds that link, the open note is re-scanned
    with `computeTodayLinks` plus `resolveTodayLinkKeys`.
  - If the link is gone, a Notice says
    `This task's Task Link is no longer in today's note`.
  - A second press while a jump is in flight is ignored.
    > [!decision] keyboard_jump **"Go to today's Task Link"** (`go-to-today-task-link`,
    > an editor command with no default hotkey) runs the same jump for the task on the
    > cursor line. When that line is not on today's plan, it shows `Not on today's plan`
    > (or `No task on this line`).

> [!decision] keyboard_jump = no No keyboard command ships: `live_preview` drops it and
> TM23's keyboard half, and the contract lists the command under "Deferred".

- **"Toggle Today marks"** (`toggle-today-marks`) is a session toggle, default on. It
  flips `body.bob-today-marks-off`, refreshes every editor (and, after `reading`, sweeps
  rendered rows), and shows a `Today marks on` / `Today marks off` Notice.

### Surfaces

- **Live Preview** of area, project, reference, and inbox notes: **v1, exact**
  (`live_preview`).
- **Reading view, block embeds, hover popovers:** exact source mapping or neutral
  (`reading`).
- **Canonical daily notes**, and anything rendered inside one: never.
- **Tasks query results** (dash, freshness, rotten, blocked, crowded): out of scope.
  Every dash query is already Today-exclusive, so a mark there carries no information.
- **Source mode:** never; it shows raw text.
- **Later, each with a trigger** (recorded in the contract's "Deferred" section):
  - a "running now" tone;
  - a "worked earlier today" ghost tone;
  - 🍅 on Depends-On/BLOCKS chips, the Task Card, nav pickers, and Mac Capture
    candidates;
  - narrowing Ctrl+Shift+Enter's presence test.

### Design calls this plan makes (refinements of the research)

1. **Contract home:** a dedicated `docs/today-marks.md`, like `task-tag-marks.md` and
   `date-marks.md`, with a pointer from `docs/plan.md` §Today. The research said either
   home is fine, and `plan.md` is already over 1,100 lines.
2. **Duplicate ids are neutral (TM15).** The research marked the first line. Neutral
   matches Rust's `Duplicate` → unresolved rule, and with a duplicate id Obsidian's link
   target is ambiguous anyway.
3. **No vector for ambiguous basenames.** JS resolution with `getFirstLinkpathDest` is a
   documented divergence (`docs/plan.md` §Today rule 4), so the research's V16 is
   dropped.
4. **Tooltip wording** is `running in` / `queued in`. It reads naturally and stays
   unambiguous when a derived home label contains `·`.
5. **Opt-out toggle class** (`bob-today-marks-off`), so the mark survives popout
   windows.
6. **Gentle-overshoot entrance** at 180 ms. It keeps the family scale but feels like
   setting a tomato down.
7. **The keyboard twin ships in v1**, under decision `keyboard_jump`.
8. **No `api.todayMarks` namespace.** There is no consumer, and every write path that
   links a task (Ctrl+Shift+Enter writes with `vault.modify`; Mac and `bob vault-sync`
   writes arrive as vault events) reaches the cache through the metadata `changed`
   event. Add an `expect`-style hint only if live verification shows visible lag.

## Phase `contract` — Today marks display contract in bob-cli docs

Write `docs/today-marks.md` as the authoritative display contract, in the style of
`docs/task-tag-marks.md`. Its opening paragraph names the JS mirror:
`src/143-today-marks.js` (pure core), `src/275-plugin-today-marks.js` (Live Preview,
toggle, jump), and `src/276-plugin-today-marks-reading.js` (rendered views). It states
that the plugin tests run the vectors verbatim and that there is no api namespace.
Sections:

1. **What you see:** a before/after `text` block on realistic rows, for example from
   `sase.md` Next & In Progress. Include a `#task #ref` row and an unmarked committed
   row.
2. **Which tasks get the mark:** the Today set, linked to `plan.md#today`, and why not
   residence or history.
3. **Principles:** display-only; truthful or neutral; one Today; rare and warm.
4. **Placement**, **Glyph anatomy**, **Tooltip and accessibility**, **Interaction**, and
   **Toggle**: transcribe the spec above, including the CSS variable and the opt-out
   class.
5. **Surfaces**, as a table.
6. **Staying current:** the cache rebuilds on the daily note's metadata `changed` event,
   on vault create/delete/rename, on `resolved`, on the minute rollover tick, and on
   focus/visibility revalidation. The readiness gate applies. Detail-only changes
   (start, rename, move between entries) update tooltips without a Tasks reload. Late
   async reads are dropped by the generation guard. A read failure degrades to no marks.
7. **Conformance vectors** TM1–TM23 (below), verbatim.
8. **Deferred,** with triggers: the running tone (if Bryan hovers to tell now from
   later); the history ghost tone (if he misses that signal); chips, Task Card, and
   pickers (once a consumer needs `api.today`); Mac Capture (Rust `today: true` on
   candidates; `decisions:mac-capture-is-a-thin-client`); narrowing Ctrl+Shift+Enter's
   presence test (recommended to leave).
9. **Live verification** checklist (the list at the end of this plan, as `- [ ]` items).

Also:

- `docs/README.md`: add a `today-marks.md` table row and mention Today marks beside task
  tag and date marks in the Obsidian-interface sentence.
- `docs/plan.md` §Today: append one short paragraph. Today marks show this set on task
  lines as a display-only 🍅, with a link to `today-marks.md`. Do not duplicate the
  contract.

Notation: `T` is an area note holding `- [s] #task Fix ^fix`, `D` is today's daily note,
and every link sits at an open entry's first child indent unless stated.

| ID   | Setup                                                                                                 | Mark on T?                                                                                    |
| ---- | ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| TM1  | `\t- [[T#^fix]]` under the unique open timed entry `- [ ] (**0725-0815** [t:: 50m]) — GTD`, `s=*`     | yes; tooltip `Today · running in GTD 0725–0815`                                               |
| TM2  | same link under placeholder `- [ ] () — BLOG`, `s=/`                                                  | yes; `Today · queued in BLOG`                                                                 |
| TM3  | `s=*` or `/`, no link in `D` (sticky lane)                                                            | no                                                                                            |
| TM4  | only `\t- 🍅 [[T#^fix]]` under a closed `- [x]` entry                                                 | no                                                                                            |
| TM5  | TM4 plus a placeholder link                                                                           | yes                                                                                           |
| TM6  | `🍅 [[T#^fix]]` under an open entry                                                                   | yes                                                                                           |
| TM7  | embedded `\t- ![[T#^fix]]` under an open entry                                                        | yes, on T's line                                                                              |
| TM8  | struck, nested, mixed-text, trailing `#` move-only, fenced, or outside `## Pomodoros`                 | no                                                                                            |
| TM9  | `s` is `x`, `X`, or `-`, link still present                                                           | no; it flips on the same frame                                                                |
| TM10 | `s=?` or `s=" "`, link still present (reconcile lag)                                                  | yes                                                                                           |
| TM11 | no exact `#task` tag (`#tasks`, `#Task`), or no standalone trailing `^id`                             | no                                                                                            |
| TM12 | linked under two open entries; then one link removed                                                  | one mark throughout; tooltip `… +1 more`, then without it                                     |
| TM13 | the task lives in a canonical daily note (`^gtd`), or T is rendered inside a daily note               | no                                                                                            |
| TM14 | cache not ready (startup), or the date rolled over                                                    | no                                                                                            |
| TM15 | `^fix` ends two lines of T                                                                            | no mark on either                                                                             |
| TM16 | `#task #ref` / `#task #prj` lines                                                                     | after the 📖 pair / between the hash and `#prj`                                               |
| TM17 | cursor at the title start; cursor inside `#task`                                                      | mark shown, caret after it; hash revealed, mark stays                                         |
| TM18 | Ctrl+Shift+Enter link, then unlink, from T                                                            | appears with the entrance pop, then disappears                                                |
| TM19 | rendered view where the source line cannot be proven                                                  | no (neutral)                                                                                  |
| TM20 | toggle off, then on                                                                                   | hidden everywhere at once, then restored; source unchanged                                    |
| TM21 | `#task` only inside inline code, or the line is inside a fence                                        | no                                                                                            |
| TM22 | the linking placeholder starts (`() — BLOG` → `(**0900-0930** [t:: 30m]) — BLOG`), with no key change | tooltip flips `queued` → `running`; no Tasks reload                                           |
| TM23 | Mod+click the mark; plain click; the keyboard command on T's line                                     | opens D at the Task Link (running entry first); cursor at title start; same jump as Mod+click |

> [!decision] memory_today_mark_record Only when yes: in this phase, use
> `/sase_memory_write` to add the strand
> `sase/memory/decisions/today-mark-shows-ledger-today.md` (keyword "Today Marks Show
> The Ledger Today As A Display-Only 🍅"). Copy the shape of the existing decisions
> strands. Claim: task lines on today's plan show a display-only 🍅 after the identity
> glyph, computed from the ledger Today cache, truthful or neutral, never written.
> Rejected alternatives: residence, history, a `TODAY` pill, a sun mask, a trailing
> chip, a line tint, and an opt-in body-class gate. Evidence: the research ref above,
> this plan, and `docs/today-marks.md`. Cost: a second read-time consumer of the JS
> Today cache. Reopens when Bryan finds the mark noisy or wants the history tone. Link
> `[[decisions/today-is-read-from-the-ledger]]`, then run `sase memory init`. With no,
> write no memory.

Acceptance: the doc renders cleanly, links resolve, and the vectors match this plan
exactly. If a later phase legitimately changes a vector, that phase updates this doc in
the same commit.

## Phase `cache` — Today cache details, generation guard, and revalidation

All changes are in bob-plugins `plugins/bob-ledger-tools/src/`. They are additive: the
existing Today semantics, `isToday`, `todayRank`, `todayKeys`, and every current
consumer stay byte-for-byte equivalent.

**`060-today.js` (pure):**

- `computeTodayLinks` records gain `entryRange` (`planLeadingRange(body).timeRange`, for
  example `"0725-0815"`, else `null`) and `entryPlaceholder` (boolean). Update existing
  tests only where they deep-compare whole records.
- `todayOpenEntries(content)` returns a frozen list of every open column-0 entry in
  `## Pomodoros`: `{ entryLine (1-based), name, range, placeholder }`. Fences,
  frontmatter, and the section bounds are handled exactly as in `computeTodayLinks`.
- `todayRunningEntryLine(entries)` returns the `entryLine` of the **unique** open timed
  entry (range present, not a placeholder), else `null`. This mirrors nav's Blocks-stage
  rule in `bob-navigation-hotkeys/src/302-blocks-stage-view.js`
  (`collectBlocksStageTodayLinks`).
- `resolveTodayLinkKeys(links, dailyPath, resolve)` returns `[{ link, key }]` for every
  resolved link in ledger order, without dedupe. Re-express `resolveTodayKeys` as the
  first-occurrence dedupe of it, with identical output.
- `todayLinkDetails(resolved, { runningEntryLine, homeLabels })` returns
  `Map<key, frozen detail[]>`.
  - Each detail is
    `{ entryLine, ledgerLine, name, range, placeholder, running, label, embedded }`,
    where `label` is the name, else `homeLabels.get(entryLine)`, else `null`.
  - Keep at most one detail per (key, entry).
  - Order each list with the running entry first, then ledger order.
- `todayDetailsSignature(details)` returns a stable string over the ordered keys and
  details.
- `todayKeyParts(key)` returns `{ path, blockId }`, split at the last `#`.

**Cache shape:** in `170` (both resets), `180`, and `290`, `todayCache` becomes
`{ date, dailyPath, keys, rank, details, detailsSignature, idsByPath }`.

- `idsByPath` is a `Map<path-with-.md, Set<blockId>>`.
- Every reset site produces the full shape.
- Add these accessors to the plan-and-ready mixin, all synchronous and never throwing:
  - `todayDetailsForKey(key)` returns the details array, or `[]`.
  - `todayIdsForPath(path)` returns a Set; it is empty when none.
  - `todayMarkEnteredAt(key)` returns epoch ms, or `null`.

**`rebuildTodayCache(content, dailyPath, now)` in `290`:**

- Compute the links, the resolved links, the keys (unchanged), the open entries, and the
  running line.
- Compute `homeLabels` only when some resolved link sits under an unnamed entry. Use
  `pomodoroHomeModels(content, dailyPath, metadataCache, vault, exempt)`, taking
  `exempt` from plan caps the way `273-plugin-pomodoro-home-chips.js` does (default
  `["GTD"]`). Wrap it in try/catch, and treat an empty label as none.
- Then compute the details, signature, and `idsByPath`.
- Track `enteredAt` (key → ms). When the cache **was ready for the same date and path**
  before this rebuild, stamp every newly present key with `now.getTime()`. Startup and
  rollover builds stamp nothing, so marks never pop in at load or at midnight. Prune
  stamps older than 2 s on each rebuild.
- Keep the existing fan-out exactly as it is (`TODAY_RELOAD_EVENT`, plan blocks, status
  bar, freshness marks, Reading Board) when `changed || becameReady`.
- **Additionally** call `this.scheduleTodayMarksRefresh()`, guarded with
  `typeof … === "function"` because the method lands in `live_preview`, when any of
  these hold: `changed || becameReady || detailsChanged || dateOrPathChanged`. A
  details-only change triggers **only** this call, never a Tasks reload.
- The return value is unchanged (whether the keys changed).

**Generation guard:**

- Add an integer `this.todayCacheGen`.
- `refreshTodayCacheFromDaily(now)` takes `gen = ++this.todayCacheGen` before its read.
  On completion it commits only if `gen === this.todayCacheGen`; otherwise it resolves
  `false` without rebuilding.
- The synchronous path in `refreshTodayCacheForChangedFile` (string `data`) increments
  the generation before it rebuilds, so an older in-flight read can never overwrite it.

**Revalidation:**

- `revalidateTodayCache(now = new Date())`, throttled to once per second, first calls
  `refreshTodayCacheForRollover(now)`. If that did not refresh and
  `!isTodayCacheReady(now)`, it calls `refreshTodayCacheFromDaily(now)`.
- Wire it in `170` with `registerDomEvent(window, "focus", …)` and
  `registerDomEvent(document, "visibilitychange", …)` (visible only). Guard both for
  test hosts without `registerDomEvent` or `window`. `170` is at 905 lines, so keep the
  wiring compact.

**Tests:** extend `scripts/test-ledger-tools-today.cjs`, with an injected clock only.

- Detail records for a named running entry, a placeholder, an unnamed entry with a home
  label, and an unnamed entry without one.
- Running-first ordering; one detail per (key, entry).
- `idsByPath`.
- The signature changes only on detail changes.
- A details-only change (TM22) calls `scheduleTodayMarksRefresh` but fires neither
  `TODAY_RELOAD_EVENT` nor the plan rerender.
- Entered stamps are set only when the cache was previously ready, and are pruned.
- Out-of-order async completion: two deferred reads resolved in reverse order, and the
  newer wins. An async read followed by a sync rebuild: the sync rebuild wins.
- Revalidation is throttled, and it refreshes when not ready.
- `resolveTodayKeys` output is unchanged on the existing T1–T9 fixtures.

Finish: `npm run build`, `npm test` (only the 4 known `bob-cli-6v` failures may remain),
and a patch bump of bob-ledger-tools in `manifest.json` and the `README.md` table, as
recent commits do. Then `bob plugins sync`.

## Phase `mark_core` — Today mark pure core, widget, and styles

This phase depends on `cache` only because every bob-plugins phase regenerates
`main.js`, the `package.json` test list, and the version. Serializing them avoids merge
conflicts.

**New `src/143-today-marks.js`**, registered in `fragments.json` after `142`. It holds
pure helpers in the family style: synchronous, never throwing, frozen outputs, and a
header comment naming the contract `docs/today-marks.md` in bob-cli.

- `TODAY_MARK_OPEN_STATUSES` is ` `, `*`, `/`, `?`.
- `todayMarkTrailingBlockId(lineText)` returns the id of a standalone trailing `^id`: it
  is preceded by whitespace, matches `PLAN_BLOCK_ID_RE`, and may be followed only by
  trailing whitespace. Otherwise it returns `null`.
- `todayMarkInsertOffset(lineText)` returns a UTF-16 offset in **original-line
  coordinates**.
  - Compute the body start after the indent, the blockquote prefix (`> `), the list
    marker (`-`, `*`, `+`, `1.`, `1)`), the `[s]` checkbox, and its whitespace. Reuse
    the `freshnessTaskStatus` parsing helpers and map back across the stripped prefix.
  - If the first `taskTagMarkRanges` range starts at the body start, the offset is that
    range's `to` (it already spans a `#task #ref` pair) plus the following whitespace
    run. Otherwise the offset is the body start.
- `todayMarkLineCandidate(lineText)` returns `{ blockId, status, insertCh, tagFrom }` or
  `null`. It requires an open status, at least one exact `#task` range, and a trailing
  id. `tagFrom` is the first `#task` range start, so callers can check for code.
- `todayMarkDuplicateIds(lines, ids)` returns the subset of `ids` that end more than one
  line. `lines` may be an array or an iterable of strings.
- `todayMarkSessionText(detail)` and
  `todayMarkTooltip(details, { platform: "mac" | "other" | "mobile" })` follow the
  tooltip spec exactly. Empty or invalid input yields `Today`, and the output never
  contains `::`.
- `todayMarkEntering(enteredAt, nowMs)` returns `0 <= nowMs - enteredAt < 600`.
- `todayMarkModifierPressed(event, isMac)` checks the primary button with `metaKey` on
  macOS and `ctrlKey` elsewhere.
- `buildTodayMarkElement(doc, { key, tooltip, entering })` returns an **empty**
  `span.bob-today-mark` (plus `is-entering` when entering) with `role="img"`,
  `aria-label=tooltip`, `data-tooltip-position="top"`, and `data-bob-today-key=key`. It
  has no text, no title, and no listeners. It returns `null` on bad input.
- `TodayMarkWidget` is defined only when `WidgetType` exists.
  - `constructor(key, tooltip, entering, onJump)`.
  - `eq` compares the class, `key`, and `tooltip` only. Entering and the callback are
    deliberately excluded, so a later rebuild never swaps the DOM and never replays the
    pop. A tooltip change (queued → running) swaps it quietly.
  - `toDOM(view)` builds the element and adds one `mousedown` listener. It always calls
    `preventDefault`. With the modifier pressed it calls `onJump(key)`. Otherwise it
    dispatches `selection: { anchor: view.posAtDOM(dom) }` and focuses.
- Add `ensureTodayMarksRefresh()` to `010-load-and-constants.js`, a lazy `StateEffect`
  exactly like `ensureUnblockedGlyphRefresh`.
- Export the helpers through `350-exports.js` for tests.

**`styles.css`:** add one commented block after the unblocked-glyph block, implementing
"Glyph anatomy" exactly:

- the `--bob-today-mark-glyph` variable;
- the host box and `::before` with the emoji font stack;
- the status-accent hover capsule and the `is-entering` pop keyframes;
- the opt-out `body.bob-today-marks-off` gate;
- the reduced-motion and forced-colors rules.

The daily-view suppression rule belongs to `reading`.

**Tests:** new `scripts/test-ledger-tools-today-marks.cjs`, added to the `package.json`
`test` list. It uses self-contained stubs in the style of the unblocked-glyph suite,
with an injected clock.

- **Candidate and offset:**
  - pure halves of TM9, TM10, TM11, TM16, and TM21;
  - `#task` first, `#task #ref`, `#task #prj`, `#task` mid-body, a blockquote prefix,
    extra spaces, an empty title, a numbered list, and an emoji before the title (UTF-16
    offsets).
- **Trailing-id rules:** `^id` without a leading space, a `^id` followed by text, and an
  id with an invalid character.
- **Duplicates.**
- **Tooltip:** running, queued, a named timed entry, an unnamed entry with a label, an
  unnamed timed entry without a label, a bare placeholder, `+N more`, and the mac,
  other, and mobile variants. Also assert there is no `::`.
- **Element:** attributes, an empty body, `is-entering`.
- **Widget:** the `eq` truth table; plain mousedown dispatches the selection; Mod
  mousedown calls `onJump` and does not dispatch.
- **`styles.css` contract assertions:** the variable, the opt-out gate, reduced motion,
  and forced colors.

Finish: build, test, a patch version bump, then sync. Nothing is visible yet, because
nothing registers the widget.

## Phase `live_preview` — Today marks in Live Preview with toggle and jump

**New `src/275-plugin-today-marks.js`** defines `BobLedgerToolsTodayMarksMixin`,
installed in `310-install-methods.js` after the unblocked-glyph mixin and registered in
`fragments.json`. It mirrors `271-plugin-unblocked-glyph.js` in shape.

- `setupTodayMarks()` does the following:
  - sets `todayMarksEnabled` to true when unset;
  - removes `bob-today-marks-off`;
  - adds the `toggle-today-marks` command ("Toggle Today marks");
  - adds the `go-to-today-task-link` command ("Go to today's Task Link") as an
    `editorCallback`, only when `keyboard_jump` is yes;
  - registers `createTodayMarkExtension()`.
- `todayMarkShouldRebuild(u)` rebuilds on `docChanged`, `viewportChanged`, the
  `ensureTodayMarksRefresh` effect, a Live Preview toggle, or a file switch. It never
  rebuilds on `selectionSet`. Copy the unblocked-glyph checks.
- `buildTodayMarkDecorations(view)` takes its clock from `this.todayMarksNow()`, which
  defaults to `() => new Date()` and is overridable in tests.
  - **Gates, in order:** enabled; Live Preview on; file path present; not
    `PLAN_DAILY_PATH_RE`; `isTodayCacheReady(now)`; `todayIdsForPath(path)` non-empty.
    Otherwise return `Decoration.none`.
  - **Line walk:** walk the visible ranges line by line, like the task-tag builder.
  - **Prefilter:** `text.includes("#task")`, and the trailing id is in the note's id
    set.
  - **Then:** `todayMarkLineCandidate`; skip code via
    `freshnessMarkPosInCode(tree, line.from + tagFrom + 1)`; look up the key's details
    (non-empty).
  - **Duplicates:** computed lazily, only when the first candidate passes, via
    `todayMarkDuplicateIds` over `doc.iterLines()`. Memoize in a `WeakMap` keyed on
    `view.state.doc`; CodeMirror docs are immutable.
  - **Decoration:** add
    `Decoration.widget({ widget: new TodayMarkWidget(key, todayMarkTooltip(details, platform), entering, (k) => this.goToTodayTaskLink(k)), side: -1 })`
    at `line.from + insertCh`.
- `toggleTodayMarks()` flips the flag, toggles the body class, refreshes the editors,
  calls the reading sweep when present (guarded `typeof`), and shows the Notice.
- `refreshTodayMarkEditors()` dispatches the refresh effect to every markdown leaf whose
  file is not a canonical daily note.
- `scheduleTodayMarksRefresh()` coalesces into a single ~16 ms timer, which then
  refreshes the editors and the guarded reading sweep. Marks should appear or vanish
  well within a quarter second of the metadata event.
- `goToTodayTaskLink(key)` is async, never throws, and never writes. It implements the
  Mod+click behavior from "Interaction", including the in-flight guard, the stale
  `ledgerLine` re-scan, and the Notice.
  - Place the cursor with `setEditorCursor(…, { scroll: true })`, then center the line
    with `editor.scrollIntoView(…, true)`.
- `goToTodayTaskLinkFromEditor(editor, view)` applies the same gates as the builder to
  the cursor line, then jumps or shows the Notice.

**`170-plugin-lifecycle.js`:**

- initialize `todayMarksEnabled = true`, `todayMarksTimer = null`, and
  `todayMarksJumping = false` beside the unblocked-glyph fields;
- call `this.setupTodayMarks()` right after `this.setupUnblockedGlyph()`;
- in `onunload`, clear the timer and remove `bob-today-marks-off`.

Keep the file ≤ 1000 lines. If it would overflow, move the new field initialization into
`setupTodayMarks()`.

**Tests**, appended to `scripts/test-ledger-tools-today-marks.cjs`. Use a fake
`EditorView`: `state.field(editorInfoField | editorLivePreviewField)`, a `doc` with
`line`, `lineAt`, `iterLines`, and `length`, `visibleRanges`, and a syntax-tree stub.
Seed the cache through `rebuildTodayCache` with a fixed clock.

- Run TM1–TM18 and TM20–TM23 against the decorations, the widget positions, and the
  tooltips.
- The TM14 rollover: rebuild for day N, evaluate at day N+1.
- TM18: the entering flag is true right after a link joins and false after 600 ms.
- Rebuild triggers.
- Toggle: the body class, the dispatch to non-daily leaves only, the Notice.
- Jump:
  - reuse an open leaf vs. open via the daily-note path;
  - a correct `ledgerLine`;
  - a stale `ledgerLine` that needs the re-scan;
  - a link that is gone (the Notice);
  - the in-flight guard.
- The keyboard command on marked, unmarked, and non-task lines.

**Docs and release:**

- In bob-plugins: a minor version bump in `manifest.json` (and append a sentence to its
  `description`), plus a README row phrase. It names the Today mark, the toggle, and the
  jump.
- In bob-cli: if anything in `docs/today-marks.md` turned out wrong, fix it in this
  phase.
- Then build, `npm test`, and `bob plugins sync`.

## Phase `reading` — Today marks in rendered views

**New `src/276-plugin-today-marks-reading.js`** defines
`BobLedgerToolsTodayMarksReadingMixin`, installed after the Live Preview mixin.

**Post-processor:**

- In `setupTodayMarks` (guarded `typeof`), register
  `registerMarkdownPostProcessor((el, ctx) => this.renderTodayMarksIn(el, ctx), 60)`. It
  runs after the task-tag annotation at order 50; insertion must still be robust to
  order.
- `renderTodayMarksIn(el, ctx)` starts with these gates:
  - `ctx.sourcePath` is present and not a canonical daily note.
  - The candidate rows are `li.task-list-item[data-task]` under `el`. Skip rows under
    `.plugin-tasks-list-item`, `.tasks-list-text`, `.task-description`, `code`, or
    `pre`.
- **Identity must be exact or the row stays neutral:**
  - `info = ctx.getSectionInfo(el)`. If it is `null` (common for block embeds and hover
    popovers), stamp nothing and mark nothing. That is TM19.
  - Source text is `info.text` when it is a string, else `vault.cachedRead`, as in
    `280-plugin-dependency-render.js` `dependencyReadingLineFor`.
  - **Primary mapping:** `info.lineStart + Number(li.getAttribute("data-line"))`. Accept
    it only when that source line parses as a task whose status equals the row's
    `data-task` (treat `""` and `" "` as equal).
  - **Fallback mapping:** the row's index among the section's rendered list items maps
    to the same index among the section's list-item source lines, but only when the two
    counts agree (the proven `280`/`281` pattern).
  - **Otherwise:** neutral.
  - **No title or text matching, ever.**
  - Then apply `todayMarkLineCandidate(sourceLine)`, the open status of both the source
    and the rendered `data-task`, and `todayMarkDuplicateIds` over the whole source.
- **Stamping and insertion:**
  - Stamp **every** identity-proven candidate row with
    `li.dataset.bobTodayKey = "<sourcePath>#<id>"`, whether or not it is in Today, so a
    later sweep can add a mark without a rerender.
  - Insert a mark only when the toggle is on, the cache is ready, and the key has
    details.
- **Insertion point**, in the row's own content (never inside nested lists):
  - After the first own `a.tag` whose text is exactly `#task`. If the next element
    sibling, skipping whitespace-only text, is the paired `#ref` anchor (any case, or
    `.bob-ref-task-hidden`), insert after that anchor instead.
  - With no such anchor, insert after `input.task-list-item-checkbox`.
  - To match Live Preview spacing, when the following text node starts with whitespace,
    insert after that whitespace (split the node) so the mark sits right before the
    title.

**Refresh sweep:**

- `sweepTodayMarksReading()` finds every document: the main document plus each leaf's
  `containerEl.ownerDocument`, deduplicated (popouts included).
- It queries `li[data-bob-today-key]` and sets each row's desired state: the toggle is
  on, the cache is ready, the key has details, the row's `data-task` is open, and the
  row has no `.bob-daily-note-view` ancestor.
- It then adds, removes, or updates (`aria-label`) the row's own `.bob-today-mark`,
  where `mark.closest("li") === li`.
- `scheduleTodayMarksRefresh` and the toggle call it through the guarded hooks added in
  `live_preview`.

**Daily-view suppression:**

- `refreshDailyNoteViewClasses()` toggles `bob-daily-note-view` on each markdown leaf's
  `containerEl` when its file matches `PLAN_DAILY_PATH_RE`. The rule covers every daily
  note, so it is date-independent.
- It runs on `file-open`, `layout-change`, and `active-leaf-change`.
- CSS: `.bob-daily-note-view .bob-today-mark { display: none; }`. This covers task
  embeds rendered inside daily notes (TM13), including Live Preview embeds.

**Mod+click in rendered views:**

- One delegated capture-phase `click` listener per document:
  `registerDomEvent(document, …)`, plus each popout via `workspace.on("window-open")`.
- When the target is inside `.bob-today-mark` and the modifier is pressed, call
  `preventDefault` and `stopPropagation`, then
  `goToTodayTaskLink(mark.dataset.bobTodayKey)`.
- A plain click does nothing. The span stays listener-free, so it survives Dataview's
  `innerHTML` round trip.

**Tests**, in a new `scripts/test-ledger-tools-today-marks-reading.cjs` added to
`package.json`. Use a fake DOM with an injected clock.

- Section info present vs. `null`.
- The `data-line` mapping, including rejection on a status mismatch.
- The row-index fallback when the counts agree vs. disagree.
- Tasks result rows and code are skipped.
- A daily-note `sourcePath` is skipped.
- Duplicate ids stay neutral.
- Insertion points: the `#task` anchor, the `#ref` pair, no anchor, and the whitespace
  split.
- Stamping of non-Today rows.
- The sweep: add on link, remove on unlink or completion, update the tooltip, respect
  the toggle, and skip daily ancestors.
- The daily-view class on open and switch.
- Delegated Mod+click vs. plain click.

**Release:**

- Minor version bump, manifest description, and README phrase.
- Update `docs/today-marks.md` Surfaces and Live verification in bob-cli if anything
  changed.
- Build, `npm test`, `bob plugins sync`.

## Verification

**Every bob-plugins phase:**

- `npm run build` and `npm run build:check`.
- `npm test`: only the 4 pre-existing `bob-cli-6v` unblocked-glyph failures may remain.
- `node scripts/validate-manifests.mjs` (via `npm run validate`).
- `bob plugins sync`.
- Fragments stay ≤ 1000 lines.

**The bob-cli `contract` phase:**

- The doc's links and anchors resolve, and the TM table matches this plan.

**Live verification (Bryan, in Obsidian, after `live_preview` and again after
`reading`):**

- In `sase.md`, exactly the Next rows linked under today's open Pomodoros show 🍅. The
  merge-queue ref row, the project-stats row, and the E2E row do not.
- Ctrl+Shift+Enter link and unlink make the mark pop in and vanish within about a
  quarter second, both with the daily note open in another pane and with it closed.
- Completing a marked task removes its mark on the same frame.
- Starting the placeholder Pomodoro flips the tooltip from `queued in` to `running in`.
- Light and dark themes look right:
  - on green (Next) and amber (Pending) tints;
  - beside a violet `#prj` pill;
  - after the 📖 pair;
  - with Apple emoji on the Mac.
- vim `j`/`k` does not jitter, and the caret sits after the mark at the title start.
- Mod+click and "Go to today's Task Link" land on the Task Link, reusing the open daily
  pane.
- Midnight (or waking the laptop the next day) clears the marks without a reload.
- The toggle hides and restores marks everywhere, including in a popout window.
- Daily notes and `dash.md` never show a mark.
- Reading view shows the same marks as Live Preview. If full-note Reading view shows
  none, `ctx.getSectionInfo` is unavailable there: file a follow-up rather than adding
  title matching.
