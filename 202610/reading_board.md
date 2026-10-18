---
tier: epic
title: Reading Board — a read-only kanban of the reading queue, drawn by bob
goal: 'dash_references opens on a calm, read-only Reading Board (📖 Reading · ⭐ Next
  · 📥 Ready · 💤 Snoozed) whose lanes, wake dates, and waiting-on reasons come from
  `bob ref list` JSON, so the board, the terminal, and Bob Mac Capture always agree
  on what can be read now.

  '
decisions:
  pdf_action:
    ask: Should board cards open the reference PDF in Highlights (a PDF button and
      ⌘↵)?
    default: true
    why: Reading starts in Highlights, and ⌘↵ already opens the PDF from a reading
      task
    answer: true
  human_reason:
    ask: Should human `bob ref list` rows also say why a blocked reference is snoozed?
    default: true
    why: The terminal and the board then agree on why each reference sleeps
    answer: true
  memory_board_record:
    ask: Record decisions:reading-board-is-bob-rendered (bob-drawn board; Bases kanban
      rejected)?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: ref-json
  title: Reading-task facts and blocked reasons in bob ref JSON
  depends_on: []
  size: medium
  description: 'ref-json: add task priority and scheduled date plus a typed `blocked_by`
    reason list (schedule, open prerequisites, blocked project) to every `bob ref`
    row, an optional human reason suffix, fixtures, tests, and docs/ref.md.

    '
- id: board-data
  title: Board data path and pure lane models
  depends_on:
  - ref-json
  size: medium
  description: 'board-data: in bob-ledger-tools, share the stats spawn core, fetch
    and cache `bob ref list` JSON with stale-while-revalidate, and build the pure,
    tested lane, card, snoozed-group, ordering, and fence models.

    '
- id: board-ui
  title: The bob-reading-board block, interaction, and look
  depends_on:
  - board-data
  size: medium
  description: 'board-ui: render the board (lanes, cards, Snoozed rows, honest states),
    keyboard and mouse routing, the Open Reading Board command, the optional Highlights
    hand-off, CSS, and a screenshot preview harness reviewed in both themes and three
    widths.

    '
- id: rollout
  title: Put it on dash_references, document, verify live
  depends_on:
  - ref-json
  - board-ui
  size: small
  description: 'rollout: add the board above the existing table on dash_references,
    document it in docs/ref.md and docs/dashboard.md, verify against the live vault,
    record follow-ups, and handle the memory decision.'
proposed_by: bbugyi200.apollo.8z
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 13:02:19
status: wip
bead_id: bob-cli-99
---

- **PROMPT:** [prompts/202610/reading_board.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/reading_board.md)
- **BEAD:** [bob-cli-99](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-99/README.md)

# Plan: Reading Board — a read-only kanban of the reading queue, drawn by bob

## Context

Bryan asked for the read-only reading-queue kanban recommended by
`research:202610/obsidian_1_14_upgrade_for_bob_setup/obsidian_1_14_upgrade_for_bob_setup.md`
(§"Bases kanban and groups"), designed to be intuitive, reliable, and beautiful. Every
phase reads that report with `sase artifact read` before starting.

The research proposed a Bases kanban view in `refs.base`, grouped by
`formula.status_badge`. Grouping by a formula makes a Bases board read-only. Planning
found the flaw: **Bases can only group by a ref note's frontmatter `status`, and it
cannot see the `^ref` reading task.** Bryan chose the alternative (Q1, "bob-rendered
board"): a read-only kanban drawn by bob-ledger-tools from `bob ref list -f json`,
following the `bob-stats` block pattern, placed on `dash_references` above the existing
table. This plan designs it.

### Facts the design rests on (measured 2026-10-18 on the live vault)

- `bob ref list -s ready,next,wip -A -f json` returns the 45 modern open references in
  about 0.2 s (50 KB, 0 legacy rows). The default `bob ref list` also returns 355
  zorg-era legacy rows; `-s` drops them.
- Lanes today:
  - Not blocked: 1 wip, 6 next, 15 ready.
  - Blocked `[?]`: 23 rows, all with frontmatter `ready` or `next`. A Bases board would
    show all 23 in Ready or Next as if they could be read now.
- Why those 23 are blocked, read from their task lines:
  - 12 only by a future `[scheduled::]` date. Most come from priority-ladder rolls; the
    latest is Nov 1.
  - 6 by open prerequisites. These are mostly reading chains: Minions Part 2 waits on
    Part 1, and the Gas City report waits on two other reports.
  - 4 by `#prj_is_blocked` (they live in the blocked `sase_post_blog` project).
  - 1 by both a project block and a schedule.
- JSON `blocked` is a row overlay (`docs/ref.md` "Reading state") that is true when the
  selected tracker is `[?]`. JSON carries no reason, date, or priority. Human
  `bob ref list` prints a blocked row as plain `READY`/`NEXT`.
- bob already splits Blocked this way elsewhere, without running reconcile.
  `highlights_ref/note_snapshot/bob.rs` (`apply_scheduled_lanes`) renders **Scheduled**
  (date-only ticklers, grouped This week / This month / Later) and **Waiting on** (open
  `dependsOn`, `#prj_is_blocked`, nested tasks). It uses `#prj_is_blocked` as the
  sanctioned project proxy (`decisions:subprojects-inherit-project-blocking`: "the tag
  is the stored proxy read by engines that do not walk the tree").
- Reconcile's blocking rule (`bob task reconcile --help`; `task_status_hooks/sync.rs`
  `task_dependency_states`) has three parts:
  - A `[dependsOn::]` id blocks while some task carrying that `[id::]` is open.
  - Unresolved ids never block.
  - A `[scheduled::]` date blocks while it is strictly after today. Reason kinds are
    `local_dependency`, `local_schedule`, `project_dependency`, and `project_schedule`
    (`docs/task-dependencies.md` §13.9).
- The plugin side already has everything the board should reuse:
  - The stats data path in `src/236-plugin-stats-data.js`: `bob` resolution through
    `stats.bob_path` and well-known paths, the `execFile` spawn with `BOB_DIR` and
    `NO_COLOR`, a 20 s timeout, the cache, and the error model.
  - The block lifecycle in `src/238-plugin-stats-block.js`.
  - Pure mark builders with an injected `doc`: `dateMarkModel`/`buildDateMarkElement`
    (`136`) and `priorityMarkModel`/`buildPriorityMarkElement` (`135`).
  - The synchronous Today cache (`todayCacheKey` = `path#blockId`, `isTodayCacheReady`,
    `src/180-plugin-plan-and-ready.js`).
  - The `hover-link`/`openLinkText` patterns (`src/210-plugin-ready-notes.js`).
  - The vault's task-status colors (`task-statuses.css`): `--task-status-in-progress`,
    `--task-status-next`, `--task-status-blocked`.
  - The headless-Chromium preview harness (`scripts/preview-stats-block.cjs`).
- bob-navigation-hotkeys already opens a reference PDF in Highlights
  (`openReferencePdfPath(path)`, `src/677-plugin-ref-pdf-open.js`; it is macOS-only and
  validates the path). Its api has no member for that today.
- The dash REFERENCES badge counts the `🔖 Reading Queue` Base view
  (`api.dashboardCollections`). Keeping that embed keeps the badge correct.

### The idea in one sentence

> **bob decides which lane every reference is in and why it sleeps; the board only draws
> it, never writes, and reads as one calm page you can take in at a glance.**

## Design principles

1. **One authority.** Lane membership, blocked state, wake dates, and prerequisites come
   from `bob ref list` JSON. The board parses no vault text and keeps no second copy of
   the blocking rules (`decisions:mac-capture-is-a-thin-client`, which the stats page
   also follows).
2. **Read-only by construction.** No drag, no "+", no writes, no reconcile. Lifecycle
   changes go through the `^ref` reading task, as they do today. The board makes that
   one keystroke away.
3. **Answer the reader's question.** "What can I read now?" fills the first three lanes.
   "Why can't I read this yet, and when can I?" is the Snoozed lane: every card there
   shows a wake date or what it waits on.
4. **Calm and native.** Theme tokens and the vault's own task-status colors. Snoozed is
   muted, never alarm-red, because Blocked is informational (`docs/dashboard.md`). Dates
   and priorities wear the same marks they wear on task lines.
5. **Honest states.** A skeleton on first paint, the last good frame on failure, an
   actionable error card, and `–`-style unavailability. An empty board never looks
   broken.

## Departures from the research

| Research said                                                       | This plan does                                                                        | Why                                                                                                                       |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| A `🗂️ Reading Board` Bases kanban grouped by `formula.status_badge` | A `bob-reading-board` code block drawn from `bob ref list` JSON                       | Bases can't see the `^ref` task, so the 23 blocked refs would show as readable (Q1)                                       |
| Lanes ⭐ Next / 🛠️ In Progress / 📥 Ready                           | 📖 Reading · ⭐ Next · 📥 Ready · 💤 Snoozed, in pull order                           | Reading leads the pull order (`decisions:task-lanes-are-sticky`). "Reading" matches `bob ref show` and the Mac Refs panel |
| "Keep `bob ref list` as the authority"                              | The board _is_ `bob ref list`. Human `ref list` also prints the reason (human_reason) | No drift to explain                                                                                                       |
| Duplicate the Reading Queue table                                   | Keep the `🔖 Reading Queue` embed under the board                                     | Bryan's choice. It also keeps the REFERENCES badge and the full sortable table                                            |

## Data contract (bob-cli, phase `ref-json`)

All additions are **additive**. `schema_version` stays 1 for `ref list`, `ref show`, and
`ref find`. Bob Mac Capture decodes with `decodeIfPresent` and ignores unknown fields,
so it is unaffected.

### Task facts (`task` object, after `archived`)

```json
"task": {
  "path": "sase.md",
  "block_id": "ref-astra-why",
  "link": "[[sase#^ref-astra-why]]",
  "mark": "?",
  "archived": false,
  "priority": "high",
  "scheduled": "2026-10-23"
}
```

- `priority` is one of `highest | high | medium | low | lowest`, from the selected task
  line's `[priority::]` inline field (`task_fields::inline_fields`, ignoring inline
  code). An absent or unrecognized value is `null`.
- `scheduled` is the strict `[scheduled::]` date (`parse_strict_calendar_date`), past or
  future, else `null`.
- v1 in-note trackers read the same fields from their tracker line.

### Blocked reasons (`blocked_by`, on every row, right after `blocked`)

```json
"blocked": true,
"blocked_by": [
  {
    "kind": "local_dependency",
    "prerequisites": [
      {
        "id": "sase__ref-github-native-merge-queue-primer",
        "title": "How GitHub's native merge queue works",
        "path": "sase.md",
        "block_id": "ref-github-native-merge-queue-primer",
        "link": "[[sase#^ref-github-native-merge-queue-primer]]",
        "mark": "*",
        "ref_path": "ref/chat/github_native_merge_queue_primer.md"
      }
    ]
  },
  { "kind": "local_schedule", "date": "2026-10-23" },
  { "kind": "project", "project": "sase_post_blog.md", "link": "[[sase_post_blog]]" }
]
```

Rules. They read the selected reading-task line only and never run reconcile:

- `blocked_by` is always present. It is `[]` whenever `blocked` is false.
- **`local_dependency`** lists, in `[dependsOn::]` order, every id (outside inline code)
  that names an **open** task. "Open" follows reconcile's rule: some task outside
  `done/` carries `[id:: <id>]` and has an open Tasks status type. Closed,
  archived-only, and unresolved ids are omitted. The `local_dependency` entry is omitted
  when no id qualifies.
  - `title` is the prerequisite reference's title when the prerequisite is itself an
    indexed reading task; `ref_path` names that reference. Otherwise `title` is the
    task's display text, the same text the `&` prerequisite picker shows
    (`capture_dependency_tasks`), and `ref_path` is `null`.
- **`local_schedule`** appears when `scheduled` is strictly after today. Today comes
  from `env::current_datetime()`, so `BOB_NOW` pins it, using reconcile's `date > today`
  rule.
- **`project`** appears when the line carries the exact `#prj_is_blocked` tag (outside
  inline code).
  - `project` is the reading task's residence note path, and `link` is that note's
    wikilink.
  - This entry is the tag proxy for reconcile's `project_dependency`/`project_schedule`.
    The owning reason may sit on an ancestor project.
- Order is `local_dependency`, `local_schedule`, `project`, which mirrors
  `effective_block_reasons`.
- `blocked: true` with `blocked_by: []` means no reason remains on the line, for example
  a schedule that arrived today. The next reconcile returns the task to its lane.
- Cost: resolve dependency ids **only when some blocked row carries `[dependsOn::]`**.
  - Either run one extra prefiltered parallel pass for the wanted `[id::` carriers, or
    collect them additively in the existing walk.
  - Measure both and keep the faster. Keep `RefTaskIndex` callers' behavior unchanged
    and reuse the shared Tasks status reading; never fork a parser.
  - Target: `bob ref list -s ready,next,wip -A -f json` stays ≤ 0.35 s on the real
    vault. Record the measured number in `docs/ref.md`. CI has no timing assertions.

> [!decision] human_reason
>
> Human `bob ref list` appends one dim reason to each blocked row, after the kind/`♫`
> columns and before the dim path:
>
> - `💤 until YYYY-MM-DD` (schedule)
> - `⛓ after <first prerequisite title> +N` (dependencies)
> - `🧱 <project stem> blocked` (project)
> - `⏸ blocked` (no reason left)
>
> Several reasons join with `·`. On a narrow terminal the path drops first, then the
> reason, then titles truncate. Markdown output is unchanged (its six-cell table is a
> tested contract).

## The Obsidian board (bob-ledger-tools, phases `board-data` and `board-ui`)

### Where it lives

- A fenced block, ` ```bob-reading-board `, rendered by bob-ledger-tools. Its body is
  optional YAML:
  - `parent: <note>` passes `-P <note>`: that area or project's reading queue, resolved
    through bob's routes and `project_name_aliases`.
  - `type: <type or list>` passes `-t <a,b>`.
  - Unknown keys and bad values become one muted lint line each. They are never fatal.
- The command is always `bob ref list -s ready,next,wip -A -f json`, plus `-P`/`-t`.
- `dash_references.md` gets an empty fence above `![[refs.base#🔖 Reading Queue]]`
  (phase `rollout`). The same block works on any note, for example a project note with
  `parent: sase`.
- A new command, **Open Reading Board** (`open-reading-board`), opens `dash_references`,
  as **Open Stats** opens `dash_stats`.

### Layout

```text
🗂️ READING BOARD   22 awake · 23 snoozed                               16:42  ↻
╭ 📖 READING  1 ──────╮╭ ⭐ NEXT  6 ──────────╮╭ 📥 READY  15 ────────╮╭ 💤 SNOOZED  23 ── ▾ ╮
│┃ CyberPower UPS:    ││┃ TODAY              ││┃ Google's release   ││ THIS MONTH           │
│┃ replace the batt…  ││┃ How GitHub's native││┃ engineering vs …   ││ tomorrow  Job Search │
│┃ 💬 Chat · dev ·    ││┃ merge queue works  ││┃ 💬 Chat · sase ·   ││ Tue  Minions: Stri…▂▄│
│┃ Oct 17             ││┃ 💬 Chat · sase ·   ││┃ today              ││ Thu  Understanding…▂▄│
│                     ││┃ Oct 17      PDF Task││ …                  ││ LATER                │
│                     ││ …                   ││                    ││ Nov 1  Bob-Grade Ta… │
│                     ││                     ││                    ││ WAITING ON           │
│                     ││                     ││                    ││ ⛓ Minions … Part 2   │
│                     ││                     ││                    ││   after Minions: S…  │
│                     ││                     ││                    ││ 🧱 Evals through go… │
│                     ││                     ││                    ││   in blocked project │
│                     ││                     ││                    ││   sase_post_blog → ⭐ │
╰─────────────────────╯╰─────────────────────╯╰────────────────────╯╰──────────────────────╯
```

- **Header** (one line):
  - The `🗂️ READING BOARD` label in the bob-plan small caps.
  - A summary, `N awake · M snoozed`. Awake = Reading + Next + Ready.
  - The data time (`HH:MM` from `generated_at`).
  - A ↻ refresh button (`aria-label="Refresh reading board"`).
- **Responsive grid** using a container query on the board root
  (`container-type: inline-size`). Lanes align to the top and never stretch.
  - ≥ 56 rem: four columns.
  - 34–56 rem: 2 × 2 (Reading | Next over Ready | Snoozed). This is Obsidian's readable
    line width.
  - Below 34 rem: one column.

### Lanes

Lane membership comes from bob's `status` and `blocked`. The board keeps only
`era: "modern"` rows whose `status` is `wip`, `next`, or `ready` (defensive; the `-s`
filter already guarantees it).

| Lane       | Rows                        | Accent (with fallback)                                | Empty text          |
| ---------- | --------------------------- | ----------------------------------------------------- | ------------------- |
| 📖 Reading | `wip`, not blocked          | `var(--task-status-in-progress, var(--color-yellow))` | Nothing in progress |
| ⭐ Next    | `next`, not blocked         | `var(--task-status-next, var(--color-green))`         | Nothing next        |
| 📥 Ready   | `ready`, not blocked        | `var(--color-blue, var(--interactive-accent))`        | Nothing ready       |
| 💤 Snoozed | `blocked: true`, any status | `var(--text-faint)`, dashed lane border, muted text   | Nothing snoozed     |

The accents match the bob-plan PENDING / NEXT / READY chips and the vault's task lines.
When the whole board is empty it shows bob's own phrase, **Nothing queued ✓**, in place
of the lanes.

### Awake card (Reading, Next, Ready)

```text
╭──────────────────────────────────────────╮
┃ TODAY                                    │  pill only when linked under today's Pomodoros
┃ How GitHub's native merge queue works *  │  title: weight 600, ≤ 3 lines, `code` spans rendered
┃ 💬 Chat · sase · Oct 17 · ♫ · ✎ 4   ▂▄▆ │  meta line (muted, smaller) · priority mark at the end
╰──────────────────────────────────────────╯   PDF · Task actions appear on hover/focus
```

- **Kind**: the `refs.base` category vocabulary.
  - `ai` → 🤖 AI, `papers` → 📄 Paper, `blogs` → 📰 Blog, `docs` → 📚 Docs, `chat` → 💬
    Chat.
  - Other types → 🔖 plus the title-cased type; `null` → 🔖 Ref.
- **Topic**: `parent`, the note where the reading task lives.
- **Added**: `dateMarkModel("created", added, today)` with `buildDateMarkElement`. ♫
  when `audio` is set, and `✎ N` when `annotation_count > 0`.
- **Priority**: `priorityMarkModel` with `buildPriorityMarkElement`, only when
  `task.priority` is set.
- **Pending sync**: when `status_sync != "ok"`, a dim `*` after the title, with the
  tooltip "Frontmatter status catches up on the next Highlights sync".
- **TODAY pill**: shown when the reading task's `path#block_id` is in the Today cache
  and `isTodayCacheReady()` is true. It never guesses: no ready cache, no pill. It uses
  the `bob-plan-plan` TODAY chip accent.
- Reused marks drop their editor-only hint lines (for example "Ctrl+Shift+P to …") and
  get board tooltips. All text uses Obsidian text tokens and is set with `textContent`.

### The Snoozed lane

Compact rows (one title line with an ellipsis, full title in the tooltip) sit in
small-caps groups, in this order:

1. **Waking**: `blocked_by` is `[]`. The chip is `⏰ now`, with the tooltip "No reason
   left — the next reconcile returns it to <lane>".
2. **This week**, **This month**, **Later**: rows whose only reason is `local_schedule`,
   sorted by wake date. The buckets match the note snapshot's Scheduled lane exactly:
   - This week: through Sunday of the current Monday-start week.
   - This month: the same calendar month.
   - Later: everything else.

   The left chip is `dateMarkLabel(date, today)` (`tomorrow`, `Tue`, `Oct 27`, `Nov 1`),
   with the tooltip "Snoozed until Thu, Oct 23 · in 5 days".

3. **Waiting on**: every other blocked row, in bob order. A second muted line says why:
   - `⛓ after <first prerequisite title>` with `+N` when there are more.
   - `🧱 in blocked project <stem>`.
   - Any schedule date as a trailing chip.

Rows show the priority mark. When the underlying status is not `ready`, a muted hint
shows where the row returns: `→ ⭐ Next` or `→ 📖 Reading`. Hovering or focusing a
Waiting-on row outlines any prerequisite card that is on the board (matched by
`ref_path`). The lane header has a ▾/▸ collapse toggle. It defaults to expanded, and the
toggle is session-only per source path.

### Ordering

- Awake lanes: cards linked today first, by Today rank, then bob's order (row date
  newest first, undated last, then title). The board never re-sorts beyond that, so its
  order matches `bob ref list`.
- Snoozed: by group as above. Wake date ascending within the date groups, ties in bob
  order.

### Interaction (read-only)

| Gesture                        | Effect                                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Click / Enter                  | Open the reference note (`openLinkText(path, sourcePath)`)                                                   |
| ⌘/Ctrl-click, middle-click     | Open the note in a new tab                                                                                   |
| ⌘/Ctrl+Enter, **PDF** button   | Open `source_pdf` in Highlights (pdf_action). Without a PDF or off macOS, open the note in a new tab instead |
| ⌥/Alt+Enter, **Task** button   | Go to the reading task (`openLinkText(<task.link target>)`), where Alt+N and Ctrl+Enter change it            |
| Hover over a title             | Obsidian page preview (`hover-link`, linktext = note path)                                                   |
| Right-click                    | Obsidian's standard file menu for the note (`file-menu`; best-effort)                                        |
| ←/→ (h/l), ↑/↓ (k/j), Home/End | Roving focus across lanes and within a lane. The board is one Tab stop                                       |
| ↻ / `r` on the focused board   | Refresh now                                                                                                  |

- Accessibility:
  - Each lane is `role="list"` with an `aria-label` such as "Next, 6 references".
  - Cards are list items.
  - One visually hidden hint is linked by `aria-describedby` and names the keys.
  - An `aria-live="polite"` region announces only manual-refresh results.
- Action buttons are `tabindex="-1"` and stay visible under `@media (hover: none)`.
  Cards are `draggable="false"`. Nothing is `contenteditable`.

> [!decision] pdf_action = no
>
> Cards have no PDF button. ⌘/Ctrl+Enter opens the note in a new tab, and
> bob-navigation-hotkeys is not touched.

### Honest states

- **First paint**: four skeleton lanes, each with two or three ghost cards at muted
  opacity. No spinner and no layout jump.
- **Background revalidation is silent**:
  - The frame stays as it is, with no dimming, because vault automation rewrites
    frontmatter often.
  - Repaint is skipped when a signature of the decoded rows plus Today keys is
    unchanged.
  - A repaint keeps focus (by note path) and scroll.
- **Manual refresh**: the ↻ button shows a busy state until the result lands.
- **Failure with a previous frame**: keep the frame and add one muted lint line:
  "Showing the board from 16:42 · bob ref list failed: <first stderr line> · Retry".
- **Failure without a frame**: an actionable card in the stats wording, with Retry.
  - Not found: name the paths tried and "set `stats.bob_path` in
    ~/.config/bob/config.yml".
  - Timeout.
  - Schema mismatch: "update bob-cli and Bob Ledger Tools together".
  - `ok: false`: show its message.
  - Off desktop: "needs the desktop app".
- **Older bob** (rows without `blocked_by`/`priority`/`scheduled`): blocked rows sit in
  Waiting on with `⏸ blocked`. One lint line says "Update bob-cli to see why references
  are snoozed".

### Visual system

- Root `.bob-reading-board` uses `--font-interface`. Lane labels and counts use the
  bob-plan chip small caps (`font-variant-caps: all-small-caps`, weight 650,
  letter-spacing 0.045em, `tabular-nums` counts).
- **Lanes**: `--background-secondary`, 10 px radius, 0.5 rem padding, and a 2 px top
  rule in the lane accent. Snoozed instead uses a transparent background, a 1 px dashed
  `--background-modifier-border` border, and `--text-muted` text.
- **Cards**:
  - `--background-primary`, a 1 px `--background-modifier-border` border plus a 2 px
    inline-start border in the lane accent, an 8 px radius, and 0.5 rem/0.6 rem padding.
  - Hover adds `--shadow-s` and a `color-mix()` accent wash at about 6%.
  - `:focus-visible` draws a 2 px accent outline with a 2 px offset.
- **Snoozed rows**: no card chrome, `--background-modifier-hover` on hover, and a
  fixed-width date column (`tabular-nums`).
- **Rules**:
  - No hex in this section; theme and vault variables only, with fallbacks.
  - `prefers-reduced-motion` turns off transitions.
  - `forced-colors: active` uses `CanvasText`/`Highlight` for borders and focus.
  - The look must hold in light and dark mode.

### Data path

- Reuse, don't fork. Extract the spawn core of `statsSpawnFetch` into one shared helper,
  for example `bobSpawnJson(args, decode, options)`. The core is:
  - `execFile`, through the optional `child_process` require.
  - An env with `BOB_DIR=<vault base>` and `NO_COLOR=1`.
  - The 20 s timeout, a generous `maxBuffer`, and the mapping of ENOENT, timeout, and
    non-zero exits that print JSON.

  Stats and the board both call it. Stats' behavior and every existing stats test stay
  unchanged. `bob` resolution stays `apiStatsBobPath()` (`stats.bob_path`, then the
  well-known paths, then PATH): one setting for every spawn.

- Decode defensively:
  - Require an object with `schema_version === 1` and an array `refs`.
  - `ok: false` becomes an error carrying its `error.message`.
  - Ignore unknown fields.
  - Missing optional fields read as `null`/`[]`.
- Cache: stale-while-revalidate keyed by argv, a small LRU (4), at most one spawn in
  flight, and one coalesced follow-up.
- Revalidate only while a board is mounted, debounced about 1.5 s, on:
  - metadataCache `changed`;
  - vault `create`/`delete`/`rename`;
  - local-date rollover, on the existing minute tick (wake dates move at midnight);
  - `active-leaf-change` into a leaf that holds a board whose data predates the last
    vault event;
  - ↻.

## Reliability and testing

1. **Rust fixtures** (`tests/fixtures/ref_library/vault`, `BOB_NOW` pinned):
   - a future schedule only, and a past or today schedule (not a reason);
   - a dependency on an open reading task (title and `ref_path` from the row), on an
     open non-ref task (display text), on a closed task, on an id found only in `done/`,
     and on an unresolved id;
   - `#prj_is_blocked`, and a dependency plus schedule plus project together;
   - `[?]` with no reason;
   - every priority value plus an unknown one;
   - a v1 tracker with `[scheduled::]`;
   - a field inside inline code (ignored).

   Assert JSON field order and presence on `list`, `show`, and `find`, plus the human
   suffix with and without color.

2. **One JSON fixture shared with the plugin.** `board-data` saves
   `bob ref list -s ready,next,wip -A -f json` from that fixture vault (with `BOB_NOW`
   pinned; synthetic titles only) as
   `plugins/bob-ledger-tools/scripts/fixtures/reading-board/ref-list.json`. The JS
   models are tested against real bob output, never hand-shaped JSON. A second, larger
   synthetic fixture of about 45 cards exists only for the preview.
3. **Read-only proof.** The board's fake-app tests fail if any vault write API is called
   (`vault.modify`/`process`/`create`/`adapter.write`/`append`).
4. **Determinism.** JS tests inject `now`, the Today cache, `execFile`, and the
   platform. Snoozed buckets are tested on a Sunday, at a month end, and at a year end.
5. **Look at the pixels.** The preview harness renders the real production code in light
   and dark at about 420, 700, and 1200 px, plus empty, error, older-bob, and collapsed
   states. Screenshots are inspected for clipping, collisions, overflow, and contrast
   before a phase is declared done.

## Phases

Ground rules for every phase:

- **bob-cli** is this project.
  - Before touching CLI help or output conventions, read `cli_rules` with
    `sase memory read`. No CLI option changes are planned.
  - Run `just all` (fmt, lint, test).
- **bob-plugins**: open it with `sase repo open bob-plugins -r "<reason>"` and read its
  `AGENTS.md`.
  - Edit `src/` fragments only. Put new code in **new fragments** listed in
    `src/fragments.json`, each ≤ 1000 lines.
  - Register new mixins in `src/310-install-methods.js`, setup and teardown in
    `src/170-plugin-lifecycle.js`, and pure helpers in `src/350-exports.js`.
  - Run `npm run build` and `npm test`. Register new test files in the `package.json`
    `test` script.
  - Bump each changed plugin's `manifest.json` minor version and extend its description.
  - Run `bob plugins sync` after changes. Never edit the deployed copies under
    `~/bob/.obsidian/plugins/`.
- **Vault edits** happen in the live vault `~/bob`, which syncs itself through
  `bob vault-sync`.
- **Interfaces**: later phases depend on the names in this plan (`blocked_by`, its kinds
  and fields, `task.priority`, `task.scheduled`, `bob-reading-board`,
  `open-reading-board`). If a constraint forces a rename, record it on the phase bead as
  `INTERFACE CHANGE:`.

### Phase `ref-json` — Reading-task facts and blocked reasons in bob ref JSON

- Implement everything in "Data contract":
  - `RefTaskView.priority`/`scheduled` and `RefRow.blocked_by`, with serde field order
    as specified, in `src/native/ref_library/row.rs`.
  - A focused reasons module, for example `src/native/ref_library/blocked_by.rs`, wired
    into row building in `src/native/ref_library/mod.rs`.
  - Update every `RefRow`/`RefTaskView` literal (`coverage.rs`, `show.rs`, `output.rs`
    tests).
  - Reuse `task_fields`, `task_dependencies::inline_code_spans`, and the exact-tag check
    `capture_dependency_tasks::has_prj_is_blocked_tag`. Reuse the Tasks status-type
    reading already used for dependency openness rather than a hard-coded mark list.

> [!decision] human_reason
>
> Add the human reason suffix in `src/native/ref_library/output.rs` (`list_row_line`
> width math) with color and `NO_COLOR` tests.

- Fixtures and tests per "Reliability and testing" item 1, in
  `tests/cli/ref_library/list.rs` (plus show and find presence checks) and the module's
  unit tests.
- `docs/ref.md`:
  - Extend "Reading state" (the `blocked` paragraph gains `blocked_by`, with its rules
    and an example).
  - Extend the `bob ref list` JSON paragraph (task `priority`/`scheduled`) and the human
    output paragraph.
  - Record the measured runtime.
  - Note that the Mac Refs panel and the coming Reading Board read these fields.
- **Acceptance**:
  - `just all` passes.
  - After `just install`, the real-vault `bob ref list -s ready,next,wip -A -f json`
    gives every blocked row a reason that a hand check of three rows confirms: one
    scheduled, one dependency, one project.
  - Unblocked rows have `[]`, and the runtime target holds.

### Phase `board-data` — Board data path and pure lane models

- Extract the shared spawn helper and move stats onto it ("Data path"). All stats tests
  pass unchanged.
- A new fragment, for example `src/241-plugin-reading-board-data.js`, holds:
  - the argv builder;
  - the decoder;
  - fetch with the cache, flight, and coalescing;
  - mounted-only revalidation with the triggers above;
  - rollover;
  - the error-card model (reusing the stats wording with "bob ref list").
- A new fragment, for example `src/242-plugin-reading-board-models.js`, holds pure,
  exported helpers:
  - `readingBoardParseFence(source)`;
  - `readingBoardModel(report, { today, todayRank, todayReady })`, which returns lanes
    with counts, card models, Snoozed groups, the summary, and lints;
  - the kind map;
  - title segmentation into text and `code` spans;
  - reason text and tooltips per kind (including `+N`, waking, and the older-bob lint);
  - the return-lane hint;
  - ordering;
  - `readingBoardSignature(model)`.

  Dates come from `dateMarkLabel`/`dateMarkModel`; never a second date formatter.

- Fixture JSON per item 2.
- Tests in `scripts/test-ledger-tools-reading-board-data.cjs`, covering:
  - argv for every fence;
  - env and timeout;
  - every decode failure;
  - cache, coalescing, debounce, mounted-only, rollover, and active-leaf behavior.

  Tests in `scripts/test-ledger-tools-reading-board-models.cjs`, covering:
  - the lane table for every status × blocked combination;
  - Today float, rank, and the not-ready cache;
  - stable bob order;
  - bucket boundaries;
  - every reason text;
  - older-bob decoding;
  - fence lints;
  - code-span titles.

- **Acceptance**: `npm test` passes and plugins are synced. A Node one-off that feeds
  live `bob ref list` JSON through `readingBoardModel` reproduces the live lane counts
  (`1 / 6 / 15 / 23` or their current values).

### Phase `board-ui` — The bob-reading-board block, interaction, and look

- A new fragment, for example `src/243-plugin-reading-board-block.js`, holds:
  - registration of the `bob-reading-board` processor, with the MarkdownRenderChild
    lifecycle, the view set, and the per-element paint-generation guard (the `bob-stats`
    precedent);
  - the skeleton;
  - the header;
  - lanes, cards, and Snoozed rows;
  - every "Honest state";
  - focus and scroll preservation and skip-repaint;
  - collapse state;
  - the interaction table and keyboard model;
  - the **Open Reading Board** command;
  - teardown.
- If the block fragment would pass 1000 lines, put DOM builders in a second fragment,
  for example `244-plugin-reading-board-render.js`. Builders use an injected `doc` (a
  `statsEl`-style helper, not Obsidian-only `createEl`) so the same code renders in
  Obsidian, fake-DOM tests, and the preview.
- CSS: a `.bob-reading-board` section in `styles.css` per "Visual system".

> [!decision] pdf_action
>
> In bob-navigation-hotkeys, add an additive api member
> `openReferencePdf(path) → Promise<{ ok, reason? }>` that wraps `openReferencePdfPath`.
> The top-level api version is unchanged. The board shows the PDF button only when that
> member exists, `Platform.isMacOS` is true, and the row has `source_pdf`. Test it next
> to `scripts/test-navigation-hotkeys-ref-pdf.cjs`.

- Before writing visual code, load the `dataviz` skill for its color and accessibility
  checks. The board has no charts, so apply only what fits.
- Preview harness: `scripts/preview-reading-board.cjs` plus a template, modeled on
  `scripts/preview-stats-block.cjs`.
  - It renders the production block from the fixture JSON and the larger synthetic
    fixture, using `styles.css`, the vault's `task-statuses.css` variables, and
    Obsidian-like light and dark tokens.
  - Widths: about 420, 700, and 1200 px.
  - States: empty, error, older bob, and Snoozed collapsed.
  - `--input FILE` renders any `ref list` JSON.
  - Screenshot with headless Chromium (Playwright's cached `chromium_headless_shell`, or
    `CHROMIUM_PATH`). Inspect every image with the Read tool, fix what you see, and keep
    it out of `npm test`.
- Tests in `scripts/test-ledger-tools-reading-board-block.cjs`, covering:
  - fake-DOM rendering of every lane and state;
  - ARIA roles and labels;
  - roving focus and every key;
  - click and modifier routing to `openLinkText`;
  - PDF button visibility rules;
  - TODAY pills;
  - prerequisite highlighting;
  - the read-only spies;
  - `draggable="false"`;
  - focus kept across repaint;
  - skip-repaint;
  - unload cleanup;
  - a CSS contract: selectors, `tabular-nums`, reduced motion, forced colors, and no hex
    in the section.
- **Acceptance**:
  - `npm test` passes and plugins are synced.
  - Screenshots in both themes and all three widths have been reviewed against this plan
    and attached with `/sase_artifact` for Bryan.

### Phase `rollout` — Put it on dash_references, document, verify live

- Vault: in `~/bob/dash_references.md`, keep the frontmatter, heading, and back link,
  then:
  - Change the description to "The reading queue: what you're reading, what's next,
    what's ready, and what's snoozed until later. The full table follows the board."
  - Insert an empty ` ```bob-reading-board ` fence directly above
    `![[refs.base#🔖 Reading Queue]]`.
- bob-cli docs:
  - `docs/dashboard.md` "Child pages": `dash_references.md` now holds the board, then
    the embed, and the REFERENCES badge still counts the Base view.
  - A new `docs/ref.md` section, "Reading Board in Obsidian", covering the fence and its
    options, lanes, card anatomy, the Snoozed groups and reasons, keys, states,
    `stats.bob_path`, and read-only behavior.
  - A pointer line from the `bob ref list` section.
  - `docs/README.md` if its ref row needs the mention.
- Live verification:
  - Run `bob plugins sync`.
  - Render the live vault's board through `preview-reading-board.cjs --input` (keep
    real-title screenshots out of git) and compare it with `bob ref list` human output.
  - Ask Bryan to run `just install-all` on the MacBook, which reinstalls bob-cli and the
    plugins and restarts Obsidian. Do not run it on the MacBook unasked. Then ask him to
    open References. If `ssh mac` answers, you may check, read-only, that the Mac's bob
    already emits the new field
    (`bob ref list -s ready,next,wip -A -f json | grep -c blocked_by`).
- Record these as `PROPOSED FOLLOW-UP:` notes on this phase bead:
  - the Mac Refs panel inspector showing the wake date and prerequisite from
    `blocked_by`;
  - the dash REFERENCES badge tooltip splitting awake and snoozed;
  - a revisit after two weeks of use.

> [!decision] memory_board_record
>
> Use `/sase_memory_write`, then add the strand
> `decisions/reading-board-is-bob-rendered`:
>
> - **Claim**: the reading-queue board is a `bob-reading-board` block drawn from
>   `bob ref list` JSON, read-only, with blocked references in a Snoozed lane showing
>   `blocked_by` reasons.
> - **Rejected**: a Bases kanban grouped by frontmatter status (it can't see the `^ref`
>   task: 23 of 45 open refs were blocked), and a JS mirror of the blocking rules.
> - **Cost**: a spawn per revalidation.
> - **Reopens when**: Bases can read task lines, or the spawn fails on the Mac.
> - **Cites**: this plan, the research report, and the shipping commits.
> - **Links**: `[[decisions/mac-capture-is-a-thin-client]]` and
>   `[[decisions/subprojects-inherit-project-blocking]]`.
>
> Run `sase memory init`.

- **Acceptance**: the board renders on `dash_references` from live data, the docs
  cross-link cleanly, and `just all` and `npm test` still pass.

## Out of scope

- Any write from the board: drag, lane moves, "+", or a "read today" button.
  `bob ref today` and the `^ref` keymaps stay the way to act.
- Bob Mac Capture changes (the JSON is additive; follow-up only).
- Changing REFERENCES badge semantics, `refs.base` views, or the Archive/All Refs
  tables.
- Legacy zorg-era rows on the board.
