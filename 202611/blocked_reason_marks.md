---
tier: epic
title: Blocked-reason marks — every [?] checkbox says why
goal: 'In Obsidian, a Blocked task''s checkbox shows why it is blocked. It shows &
  for its own open prerequisite, ^ for a blocked project, 1–4 for a P1–P4 priority
  deferral, ⟲ for a recurring rule''s next run, ? for a date picked by hand, and a
  dashed ? when no reason is left. Solid boxes mean held by other work, hollow boxes
  mean waiting for a date. Stored Markdown, reconcile, and the Tasks registry stay
  exactly as they are.

  '
decisions:
  precedence:
    ask: When a task is both held by its own prerequisite and in a blocked project,
      which mark wins?
    choices:
      own_first: Own prerequisite shows & even inside a blocked project; residents
        stand out
      project_first: Project-blocked rows always show ^, matching blocked.md's grouping
        exactly
    default: own_first
    why: In a project note every resident would show ^; the own blocker's chips sit
      right beneath it
    answer: own_first
  prj_glyph:
    ask: Which glyph should the project-blocked (held by its project) checkbox draw?
    choices:
      chevron: A ^ chevron mask; ties to ^prj and reads as "the reason is above you"
      folder_lock: A Lucide folder-lock mask; more literal for strangers, busier at
        16 px
    default: chevron
    why: Matches Bob's typographic checkbox alphabet; a one-mask swap later if it
      doesn't land
    answer: chevron
  hollow_tint:
    ask: How strongly should rows waiting for a date (hollow marks) tint the line?
    choices:
      quiet: About 5.5% red tint with a 45% rail; held-by-work rows stand out (109
        vs 478)
      full: Today's 13% red tint and full rail; only the checkbox changes
    default: quiet
    why: The research's point is that real blockers should be the only loud red rows
    answer: quiet
  memory_decision_record:
    ask: Record an accepted decision that Blocked checkboxes show their reason while
      storage stays [?]?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: contract
  title: Blocked-marks display contract and decision record (bob-cli)
  depends_on: []
  size: medium
  description: 'contract: write bob-cli docs/blocked-marks.md (marks, precedence,
    families and tones, tooltip copy, surfaces, reliability rules, api.blockMarks
    v1, BM vectors), amend the priority-marks eligibility and vectors in docs/projects.md,
    cross-link the sibling contracts, register the page, and add the accepted decision
    record.'
- id: model
  title: Pure block-mark model and api.blockMarks.model (bob-ledger-tools)
  depends_on:
  - contract
  size: medium
  description: 'model: add the pure classifier fragment, the memo-local open-prerequisite
    index, the plugin method that feeds it live line text, the ladder, and today,
    and api.blockMarks v1 model(); node tests run every BM vector verbatim. No visible
    change.'
- id: editor_marks
  title: Checkbox rendering in Live Preview and reading view, CSS, toggle, tooltips
  depends_on:
  - model
  size: large
  description: 'editor_marks: render the marks on Blocked checkboxes in Live Preview,
    reading view, embeds, hover previews, and Dataview TASK views. Covers the gated
    CSS families and tokens, glyph masks and the corner pip, the session toggle, hover
    tooltips, invalidation, api.blockMarks.render, a contrast test, and a real-Obsidian
    e2e harness with light and dark screenshots.'
- id: redundancy_folds
  title: Fold the priority bars on digit rows and the project tag on ^ rows
  depends_on:
  - editor_marks
  size: small
  description: 'redundancy_folds: where the checkbox digit already says the priority,
    fold the signal bars. Where the ^ mark already says the project is blocked, fold
    the #prj_is_blocked pill. Applies in Live Preview and reading view, keeps cursor
    reveal, and is gated on both toggles.'
- id: tasks_results
  title: Block marks in Tasks query results (CSS-only)
  depends_on:
  - redundancy_folds
  size: medium
  description: 'tasks_results: draw the marks on Tasks query result rows from Tasks''
    own row attributes, with no JS on Tasks DOM. Also hide the priority glyph on digit
    rows and the project tag on ^ rows. Verify the Tasks 8.4.0 attributes in the e2e
    harness.'
- id: chips
  title: Block marks inside DEPENDS ON and BLOCKS chips
  depends_on:
  - tasks_results
  size: small
  description: 'chips: a Blocked prerequisite or dependent chip draws its own miniature
    block mark and reason, exposing whether the critical path is waiting on work or
    on a date.'
- id: task_card
  title: Task Card reason header and teaching digit (bob-navigation-hotkeys)
  depends_on:
  - editor_marks
  size: medium
  description: 'task_card: the Task Card''s BLOCKED lane chip becomes the block mark
    plus every reason in After / Until / Via words (the touch and mobile path). The
    P1–P4 strip shows the hollow digit box beside each level''s bars. Both go through
    api.blockMarks v1 and fall back to today''s look when the api is missing.'
- id: visual_review
  title: Cross-surface visual review, polish, and live checklist
  depends_on:
  - chips
  - task_card
  size: small
  description: 'visual_review: screenshot every surface in light and dark through
    the e2e harness, fix polish defects against the reference mockup, run the full
    suites, deploy, finalize the contract''s live-verification checklist, and record
    follow-ups.'
proposed_by: bbugyi200.athena.16w
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 06:28:09
status: wip
bead_id: bob-cli-ft
---

- **PROMPT:** [prompts/202611/blocked_reason_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/blocked_reason_marks.md)
- **BEAD:** [bob-cli-ft](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ft/README.md)

# Plan: Blocked-reason marks — every `[?]` checkbox says why

## Context

Of the vault's 552 open `#task` lines, 478 (87%) are Blocked `[?]`. Today every one of
them looks the same: a solid red box with a white `?` on a red-tinted row (vault snippet
`.obsidian/snippets/task-statuses.css`). Behind that one look are different situations
that need different responses: finish a prerequisite, fix the project, or wait or
reschedule.

This epic implements the recommended solution of the consolidated research report
`research:202611/blocked_reason_marks/blocked_reason_marks.md`. Read it with
`sase artifact read research:202611/blocked_reason_marks/blocked_reason_marks.md "<why>"`.
Its reference mockup is `blocked_reason_checkboxes__cld_mockup.png` beside it, readable
the same way. Bryan accepted every requirement in that report. This plan is the
authoritative design. Where it deliberately departs from the report, the departure is
listed under [Design-lead adjustments](#design-lead-adjustments).

The core rule is that marks are **display-only**. Markdown keeps `[?]`, reconcile keeps
deriving it (`decisions:task-status-is-derived`), and the Tasks registry keeps its
single `?` = Blocked (`ON_HOLD`) status. Only the rendered checkbox changes. With
bob-ledger-tools disabled or the toggle off, the vault looks exactly as it does today.

## The design

### The marks

| Key       | Box                       | Family    | Meaning                                         | Rows today |
| --------- | ------------------------- | --------- | ----------------------------------------------- | ---------- |
| `dep`     | solid, white `&`          | `held`    | waiting on its own open prerequisite            | 53         |
| `prj`     | solid, white chevron mask | `held`    | its project (or an ancestor project) is blocked | 56         |
| `p1`–`p4` | hollow digit `1`–`4`      | `waiting` | deferred by a P1–P4 priority roll               | 346        |
| `repeat`  | hollow loop mask          | `waiting` | recurring; the next run is later                | 15         |
| `date`    | hollow `?`                | `waiting` | a date picked by hand                           | 8          |
| `stale`   | dashed `?`, muted         | `stale`   | Blocked, but no live reason is left             | 0          |
| (none)    | today's plain `?`         | —         | the reasons can't be proven yet                 | —          |

> [!decision] prj_glyph = chevron The `prj` box draws a chevron-up mask (viewBox
> `0 0 16 16`, a single stroked polyline near `M4 10.25 8 6.25l4 4`, round caps and
> joins, stroke width about 2.1; the contract records the final geometry). It is a mask,
> not the `^` character, because `^` sits too high in the interface font.

> [!decision] prj_glyph = folder_lock The `prj` box draws Lucide `folder-lock` as a
> mask, simplified to read at 16 px. It is still white on solid red.

Only rows whose stored checkbox symbol is exactly `?` and whose line is a `#task` line
can get a mark. The `#task` tag is the vault's Tasks global filter. The line must sit
outside code and fences, and the check is quote-aware. Ready ` `, Next `*`, Pending `/`,
closed, and cancelled boxes are never touched, even when the row has a priority or a
future date. A known lane box never gets a block mark.

### Classification and precedence

Inputs, all line-local except the prerequisite lookup:

1. **Open prerequisite.** Parse the `[dependsOn:: a,b]` / `(dependsOn:: …)` ids from the
   line text. In Live Preview this is the editor's current text, so unsaved edits count.
   Look each id up in a memo-local `id → tasks` index over the Tasks cache. A target is
   open when its `status.type` is `TODO`, `IN_PROGRESS`, or `ON_HOLD`; reuse
   `blocksTaskIsOpen` and `blocksTaskIds` from `141-blocks-model.js`. This mirrors
   reconcile (`docs/task-status-hooks.md`: open targets block; a duplicated id blocks if
   any instance is open).
   - The dependency input is **open** when any id resolves to an open task.
   - It is **closed** when every id resolves and none is open, or when the line has no
     ids.
   - It is **unknown** when the Tasks cache isn't ready, or when some id resolves to no
     cached task and no other id is open. The cache is ready when the Tasks plugin
     returns a non-empty task array.
2. **Project blocked.** An exact `#prj_is_blocked` tag token (not
   `#prj_is_blocked_old`). This is the sanctioned proxy
   (`decisions:subprojects-inherit-project-blocking`).
3. **Own future date.** A canonical `[scheduled:: YYYY-MM-DD]` strictly after the local
   date today. Reuse the date-marks field parser and `dateMarksToday`. A date that is
   today does not block.
4. **Date source** (only when 3 holds): a readable `[repeat:: …]` rule (reuse the
   recurrence-marks reader), else a canonical ladder priority, else hand. A canonical
   ladder priority means `priorityMarkCanonicalField` with value `high`, `medium`,
   `low`, or `lowest`.

> [!decision] precedence = own_first Winner order: **own open prerequisite > project
> blocked > own future date**. Within date holds the order is recurring > ladder
> priority > hand. If the dependency input is _unknown_, the row shows plain `?`,
> because the top-precedence input can't be ruled out. With no live reason and every
> input known, the row shows `stale`.

> [!decision] precedence = project_first Winner order: **project blocked > own open
> prerequisite > own future date**. Within date holds the order is recurring > ladder
> priority > hand. A project-tagged row shows `prj` even when its dependency input is
> _unknown_. Otherwise an unknown dependency input yields plain `?`, and no live reason
> with every input known yields `stale`.

**Digit.** Use the configured ladder label when it matches `^P([1-4])$` (`P2` → `2`).
Otherwise, or when the ladder is unknown (for example on mobile), use the default map
`high` → 1, `medium` → 2, `low` → 3, `lowest` → 4. Off-ladder `highest` and
non-canonical values (`High`, `P2`, duplicates) are date-held rows with no digit. They
show hollow `?` and keep their urgent `!` mark or repair pill. Ladder membership, not
Schedule Log provenance, picks the digit. That matches 346/346 rows today.

**More reasons.** Every live reason that loses becomes a corner pip and an `Also`
tooltip line. The pip takes the ink of the highest losing reason.

### Families and tones

- **`held` (solid):** today's filled Blocked box (`--task-status-blocked`, falling back
  to `--color-red`) with a white glyph. The line keeps today's 13% red tint and 2 px
  rail.
- **`waiting` (hollow):** a 1.5 px border in the level's ink, a 12% wash of that ink,
  and the glyph in the same ink.
- **`stale`:** a dashed 1.5 px border in `--text-muted`, a muted `?`, no wash, no line
  tint, and no rail.

> [!decision] hollow_tint = quiet `waiting` lines use `--bob-block-defer-tint: 5.5%` red
> tint with the rail at 45% opacity.

> [!decision] hollow_tint = full `waiting` lines keep today's 13% tint and full rail
> (`--bob-block-defer-tint: 13%`); only the box goes hollow.

The ink ramp is one ordinal red-to-grey ramp, not four hues. The hue budget is spent
(`docs/task-tag-marks.md` → _Ink_), and the priority-marks contract already rejected a
per-level hue ramp:

```css
body {
  --bob-block-ink-anchor: color-mix(in srgb, var(--color-red) 72%, var(--text-normal));
  --bob-block-ink-1: var(--bob-block-ink-anchor); /* P1, hand ? */
  --bob-block-ink-2: color-mix(
    in srgb,
    var(--bob-block-ink-anchor) 68%,
    var(--text-muted)
  );
  --bob-block-ink-3: color-mix(
    in srgb,
    var(--bob-block-ink-anchor) 38%,
    var(--text-muted)
  );
  --bob-block-ink-4: color-mix(
    in srgb,
    var(--bob-block-ink-anchor) 12%,
    var(--text-muted)
  ); /* P4, ⟲ */
}
```

A node test computes each glyph-on-wash-on-tint contrast with Obsidian default-theme
tokens, light and dark, and enforces at least 4.5:1. If a value fails, adjust the mix
percentages and record the final values in the contract.

**Glyphs.**

- `&`, the digits, and `?` are text through `--task-status-symbol`, in the interface
  font with `font-variant-numeric: tabular-nums`.
- `prj` is the mask chosen above.
- `repeat` reuses the existing `--bob-date-glyph-repeat` mask, so the checkbox loop
  matches the `⟲ daily` recurrence mark on the same line.

**Pip.** A 6 px dot at the box's top-right corner, ringed 1.5 px in
`--background-primary` and filled with the losing reason's ink (solid red for
`dep`/`prj`). It never changes line height or the box's hit area.

**Motion.** Color, border, and background changes may ease over 120 ms or less. Nothing
animates under `prefers-reduced-motion`.

### Tooltip and accessible name

Tooltip lines are joined with `\n` and never contain `::`. Dates use the date-marks
tooltip grammar `{short date} · {relative}` (for example `Fri, Nov 20 · in 19 days`).

| Mark      | Line 1                                                                                                    | Line 2                                                                                                                                                                    |
| --------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dep`     | `Blocked · after <first open prerequisite's clean text>` (+ ` +N` for more open ones)                     | `Ctrl+Shift+B to edit prerequisites`                                                                                                                                      |
| `prj`     | `Blocked · via project <task's note stem>`                                                                | best effort, one hop: the note's `^prj` own reason as `after <text>` or `until <date>`, or `via a parent project` when that `^prj` is itself tagged; omitted when unknown |
| `p1`–`p4` | `Blocked · P2 deferral · until Fri, Nov 20 · in 19 days`                                                  | `Ctrl+Shift+P to reschedule or change priority`                                                                                                                           |
| `date`    | `Blocked · until Wed, Nov 4 · in 3 days`                                                                  | `Ctrl+Shift+P to reschedule`                                                                                                                                              |
| `repeat`  | `Recurring · next run Mon, Nov 2 · tomorrow`                                                              | —                                                                                                                                                                         |
| `stale`   | `Blocked, but no reason is left`                                                                          | `Clears on the next reconcile`                                                                                                                                            |
| + losing  | one line each: `Also after <text>`, `Also via project <stem>`, `Also scheduled <short date> · <relative>` |                                                                                                                                                                           |

A prerequisite plus a date gives an _earliest_ return, not a promise. Never word it as
an unblock date; that is why a losing date reads `Also scheduled …`, not `until`.

### Surfaces and phasing

| Surface                                                     | Mechanism                                                                              | Phase              |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------ |
| Live Preview                                                | `Decoration.line` attributes on `.HyperMD-task-line[data-task="?"]`                    | `editor_marks`     |
| Reading view, embeds, hover previews, Dataview `TASK` views | markdown post-processor sets the same attributes on `li.task-list-item[data-task="?"]` | `editor_marks`     |
| Priority bars / `#prj_is_blocked` pill                      | folded where the box already says it                                                   | `redundancy_folds` |
| Tasks query results                                         | CSS on Tasks' own row attributes                                                       | `tasks_results`    |
| `DEPENDS ON` / `BLOCKS` chips                               | miniature box through the shared element builder                                       | `chips`            |
| Task Card                                                   | `api.blockMarks` v1 in bob-navigation-hotkeys                                          | `task_card`        |

Row attributes (identical on every JS surface):

- `data-bob-block="<key>"`;
- `data-bob-block-family="held|waiting|stale"`;
- `data-bob-block-more="<losing key>"`, only when a pip shows.

CSS is gated by `body.bob-block-marks` and wins by specificity over the vault snippet,
never by `!important`. It styles only the row's **own** checkbox: use a child combinator
in reading view, and in Live Preview the line holds exactly one box. Nested child items
must never inherit a mark. Marked rows draw their whole box (border, background, glyph),
so they look right even without the vault snippet. Plain `?` stays the snippet's job.

**Out of scope:** splitting `### Blocked` sections by reason; task-link rings; any
change to reconcile, lanes, review walks, capture, or stored Markdown;
Rust/`bob plan`/Mac Capture adoption; moving `task-statuses.css` into the plugin.
`visual_review` records the last two as follow-ups.

### Reliability rules

- **Cost.** Classification is O(1) per visible line: field regexes plus one index lookup
  per `dependsOn` id. Rendering never runs a CLI subprocess, reads disk, or walks the
  vault. The `id → tasks` index lives on the freshness memo, like
  `dependencyTasksIndex`, so it shares the memo's invalidation.
- **Invalidation.** Rebuild Live Preview decorations on doc and viewport changes, Live
  Preview or file switches, and a block-marks refresh effect. Dispatch that effect on
  the Tasks `cache-update` fan-out (`scheduleLiveWidgetRefresh`), the toggle, ladder
  config changes, the date-marks midnight rollover, and app resume (`visibilitychange`).
  Never rebuild on selection alone. Reading view re-classifies in place on the same
  triggers, like `relabelRenderedDateMarks`.
- **Stability.** Line decorations carry stable attribute values, so unchanged marks
  never flicker. Marks never change line height. The overlay never intercepts pointer
  events: clicking the box does exactly what clicking `[?]` does today.
- **Truthful or neutral.** The cache is cold, an id is unresolved, or the line is
  non-canonical: show plain `?`. `stale` appears only when every input is known.
- **Reversible.** The session command **Toggle blocked-reason marks** (id
  `toggle-block-marks`, on by default) flips `body.bob-block-marks`, refreshes editors
  and rendered views, and triggers the Tasks re-render event. It shows the Notice
  `Blocked-reason marks on` / `off`.

### `api.blockMarks` v1

The new namespace leaves the top-level api version unchanged. Every function is
synchronous and never throws.

- `model(lineText, { path })` returns a frozen
  `{ key, family, digit, more, reasons[], tooltip, modelKey }`, or `null` when the row
  gets no mark (neutral, ineligible, or toggled off). Each `reasons[]` entry is
  `{ kind: "dep"|"prj"|"date", key, text }`, winner first.
- `render(host, model, { decorative })` appends a listener-free miniature box
  `span.bob-block-mark[data-bob-block][data-bob-block-family]` to `host` and returns it,
  or returns `null`. It gets `role="img"` and an `aria-label` unless `decorative`.

`model` ships in `model`; `render` ships in `editor_marks` (it shares the CSS).

### Design-lead adjustments

These depart from the report:

1. **Accessible name on the real checkbox.** The checkbox keeps its checkbox role.
   `aria-label` and `data-tooltip-position="top"` go on the input itself. The report's
   `role="img"` would strip an interactive control's semantics. Only the miniature
   `render` element uses `role="img"`.
2. **No `Ctrl+Enter` hint.** In the editor, `Ctrl+Enter` closes tasks
   (task-status-cycler). The digit tooltip points at `Ctrl+Shift+P` instead.
3. **Unresolved ids are neutral, not ignored.** Reconcile ignores missing ids. The Tasks
   cache, however, indexes only `#task` lines, so an id it can't see may still be an
   open target to reconcile. Rows with only unresolved ids therefore show plain `?`, not
   `stale`.
4. **Stale wording.** `Clears on the next reconcile`, not `Returns to Ready`. A
   recovered task returns to its derived rank and may become a TICKLER.
5. **Vectors live in the contract table, not a bob-cli JSON fixture.** Like every
   sibling mark family, plugin tests copy them verbatim. A JSON fixture becomes
   worthwhile only when Rust adopts the mark key, which is a follow-up.
6. **Live Preview tooltips attach lazily.** A capture-phase `mouseover` on the editor
   sets `aria-label` on the hovered checkbox, computed from the line's model, so
   Obsidian's tooltip handler sees it on the same hover. Obsidian's checkbox widget is
   never replaced or wrapped, and no click is intercepted.

## Conformance vectors (seed set; `contract` owns the final table)

Today is 2026-11-01 (local). The default ladder is P1 `high` 2–7, P2 `medium` 8–30, P3
`low` 31–90, P4 `lowest` 91–365. The Tasks cache is ready and holds open `[id:: x]`,
open `[id:: y]`, and done `[id:: d]`. No cached task carries `nope`. Expectations assume
the default decisions.

| #    | Input                                                                                                        | Expected                                                                                                                            |
| ---- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| BM1  | `- [?] #task A [priority:: medium] [scheduled:: 2026-11-20]`                                                 | `p2` waiting · `Blocked · P2 deferral · until Fri, Nov 20 · in 19 days⏎Ctrl+Shift+P to reschedule or change priority` · bars folded |
| BM2  | `- [?] #task B [scheduled:: 2026-11-04]`                                                                     | `date` waiting · `Blocked · until Wed, Nov 4 · in 3 days⏎Ctrl+Shift+P to reschedule`                                                |
| BM3  | `- [?] #task C [repeat:: every day when done] [scheduled:: 2026-11-02]`                                      | `repeat` waiting · `Recurring · next run Mon, Nov 2 · tomorrow`                                                                     |
| BM4  | `- [?] #task D [dependsOn:: x]`                                                                              | `dep` held · `Blocked · after <x text>⏎Ctrl+Shift+B to edit prerequisites`                                                          |
| BM5  | `- [?] #task D [dependsOn:: d]`                                                                              | `stale` · `Blocked, but no reason is left⏎Clears on the next reconcile`                                                             |
| BM6  | BM4 with the Tasks cache not ready                                                                           | no mark (plain `?`)                                                                                                                 |
| BM7  | `- [?] #task E #prj_is_blocked`                                                                              | `prj` held · `Blocked · via project <stem>`                                                                                         |
| BM8  | `- [?] #task F [dependsOn:: x] [priority:: low] [scheduled:: 2026-12-20]`                                    | `dep` held, more `p3` · adds `Also scheduled Sun, Dec 20 · in 49 days` · bars shown                                                 |
| BM9  | `- [?] #task G #prj_is_blocked [priority:: medium] [scheduled:: 2026-11-20]`                                 | `prj` held, more `p2` · bars shown                                                                                                  |
| BM10 | `- [?] #task H [priority:: highest] [scheduled:: 2026-11-20]`                                                | `date` waiting · urgent `!` kept                                                                                                    |
| BM11 | `- [?] #task I [priority:: medium] [scheduled:: 2026-11-01]`                                                 | `stale` (today doesn't block) · bars shown                                                                                          |
| BM12 | `- [*] #task J [priority:: high] [scheduled:: 2026-11-20]`                                                   | no mark; `*` unchanged; 4 bars                                                                                                      |
| BM13 | `- [?] #task K [priority:: high] [scheduled:: 2026-11-05]` whose newest Schedule Log entry is a typed reason | `p1` waiting (ladder membership wins)                                                                                               |
| BM14 | `- [?] #task L #prj_is_blocked_old`                                                                          | not a project token → `stale`                                                                                                       |
| BM15 | `- [?] #task M [priority:: High] [scheduled:: 2026-12-01]`                                                   | `date` waiting · repair pill still shows                                                                                            |
| BM16 | `- [?] plain bullet [scheduled:: 2026-11-20]`                                                                | no mark (not a `#task` line)                                                                                                        |
| BM17 | `- [?] #task N [dependsOn:: nope]`                                                                           | no mark (unresolved id → unknown)                                                                                                   |
| BM18 | `- [?] #task O [dependsOn:: nope, x]`                                                                        | `dep` held (an open id wins over an unresolved one)                                                                                 |
| BM19 | `- [?] #task P #prj_is_blocked [dependsOn:: y]`                                                              | `dep` held, more `prj`                                                                                                              |
| BM20 | `> - [?] #task Q [priority:: lowest] [scheduled:: 2027-03-03]`                                               | `p4` waiting (quote-aware)                                                                                                          |
| BM21 | ``- [?] #task R `[scheduled:: 2026-11-20]` ``                                                                | `stale` (field in code is not a field)                                                                                              |
| BM22 | BM1 with ladder `{label: "Soon", value: medium, 8–30}`                                                       | `p2` (non-`P<n>` label falls back to the default map)                                                                               |
| BM23 | BM3 evaluated after midnight on 2026-11-02                                                                   | `stale`                                                                                                                             |
| BM24 | Tasks row `data-task="?"`, `data-task-priority="medium"`, future `data-task-scheduled`, no project tag       | CSS `2` (Tasks results)                                                                                                             |

## Phase: `contract` — display contract and decision record (bob-cli)

Work in this bob-cli checkout.

1. **Write `docs/blocked-marks.md`.** It is a root behavior contract in the style of
   `docs/date-marks.md` and the _Priority marks_ section of `docs/projects.md`.
   Sections:
   - What you see: a before/after text sketch and the legend;
   - Principles;
   - The marks;
   - Classification and precedence, including the unknown/neutral rules and the digit
     rule;
   - Families, tones, and tokens, with the ink CSS block and the contrast floor;
   - Glyphs, with the `prj` mask geometry per the `prj_glyph` decision and the reused
     repeat mask;
   - Pip;
   - Tooltip and accessible name;
   - Surfaces and interaction;
   - Tasks query results and their known limits: no pip, no tooltip, `&` by elimination,
     `⟲` only if Tasks' recurring marker proves usable;
   - Reliability and invalidation;
   - Session toggle;
   - `api.blockMarks` v1;
   - Rejected alternatives: stored statuses / a `[>]` Deferred status, icons for every
     mark, always-visible reason text, a per-level hue ramp, priority out of the box
     entirely, `role="img"` on the checkbox, field-presence `&`;
   - Conformance vectors: the table above, finalized, with any additional edge vectors
     you find;
   - Live verification (Bryan, in Obsidian): a pending checklist.

   State that the JavaScript mirror is `api.blockMarks` (namespace v1) and that its
   tests run the vectors verbatim. Write to the default decisions, or to whatever
   decisions the reviewer approved, which `sase bead read` shows as final.

2. **Amend `docs/projects.md` → _Priority marks_.**
   - Eligibility gains one rule: while blocked-reason marks are on, a row whose block
     mark is a digit (`p1`–`p4`) folds its priority mark to zero width (cursor and click
     still reveal the raw field), and Tasks results hide its glyph.
   - Annotate PM2: its date is past relative to the vector today, so it is not a digit
     row and the bars stay.
   - Add vectors: PM18, a BM1-style row folded with block marks on; PM19, the same row
     with 3 bars when block marks are off.
   - Note the folded `#prj_is_blocked` pill on `prj` rows wherever tag rendering is
     described, if anywhere.
3. **Cross-link** (one or two sentences each, no duplication):
   - `docs/task-tag-marks.md` → _Ink_: red now also carries the Blocked ink ramp;
     `#prj_is_blocked` folds on `prj` rows;
   - `docs/date-marks.md`: the checkbox loop shares the repeat glyph;
   - `docs/task-status-hooks.md` Blocked rules: the rendered checkbox shows the reason,
     while storage stays `?`;
   - `docs/task-dependencies.md` chips section: Blocked chips show the target's mark.
4. **Register the page** everywhere `date-marks.md` is registered: both `mkdocs.yml`
   lists, `docs/develop/contracts.md`, `docs/README.md`, and
   `docs/reference/vault-format.md` if it lists mark contracts. Read the `docs.md`
   reference memory first (`sase memory read docs.md -r "<why>"`).
5. Run `just docs-check` until it passes.

> [!decision] memory_decision_record Use your `/sase_memory_write` skill. Add
> `sase/memory/decisions/blocked-checkboxes-show-their-reason.md` in the strand format
> of its siblings:
>
> - frontmatter: `keyword`, `aliases`, a one-line `summary`, and
>   `metadata: {status: accepted, decided: <today>}`;
> - body sections: **Applies to.** bob-plugins, bob-cli docs, vault. **Claim.** **Why.**
>   **Cost.** **Reopens when.**
> - Claim: Blocked checkboxes render their reason (&, ^, 1–4, ⟲, ?, dashed ?) as
>   display-only marks, and storage stays `[?]`.
> - Why: a new stored status would be a second source of truth, touching 179 `'?'` Rust
>   references and ~336 plugin JS ones, and would reopen
>   `decisions:task-status-is-derived`.
> - Cost: Tasks results get only CSS marks; reconcile lag shows as `stale`.
> - Reopens when: hollow marks prove insufficient and a stored Deferred status is
>   wanted.
> - Cite the research ref and the contract. Link `[[decisions:task-status-is-derived]]`
>   and `[[decisions:subprojects-inherit-project-blocking]]`.
> - Then run `sase memory init`.

> [!decision] memory_decision_record = no Skip the decision record and record nothing.

**Done when:**

- the contract reads as a complete spec a plugin agent can implement without the
  research file;
- `just docs-check` passes;
- the decision strand appears on the `decisions` roster (if accepted).

## Phase: `model` — pure classifier and `api.blockMarks.model` (bob-ledger-tools)

Open bob-plugins with your `/sase_repo` skill (`sase repo open bob-plugins -r "<why>"`).
Read its `AGENTS.md` and work only in the printed path. Read `docs/blocked-marks.md` in
bob-cli; it is the spec.

1. **New pure fragment**, for example `src/142c-block-marks-model.js`, placed in
   `src/fragments.json` after `141-blocks-model.js`. It holds synchronous,
   never-throwing helpers in the style of `135-priority-marks.js`:
   - `blockMarkEligible(lineText)`: `[?]` + `#task`, quote-aware, not in code. Reuse
     `freshnessTaskStatus` and the code-exclusion helpers.
   - `blockMarkDependsOnIds(lineText)`: square and parenthesized fields, comma lists.
   - `blockMarkDigit(value, ladder)`.
   - `blockMarkClassify(inputs)`: the precedence, returning key, family, digit, more,
     and reasons.
   - `blockMarkTooltip(model, facts)`: the copy table, reusing the date-marks short-date
     and relative-phrase helpers.
   - `blockMarkModel(lineText, ctx)`: a frozen model with a stable `modelKey`.

   `ctx` supplies `today`, `ladder`, `lookupId(id) → task[] | null` (null when the cache
   isn't ready), `notePath`, and an optional `projectReason(notePath)`. Reuse
   `priorityMarkCanonicalField`, the date-mark canonical field parser, the recurrence
   reader, `blocksTaskIsOpen`, `blocksTaskIds`, and `dependencyCleanTaskText`. Do not
   fork them.

2. **Plugin glue** in a small mixin (installed in `310-install-methods.js`):
   - `blockMarksIdIndex()`: an `id → tasks[]` map stored on the freshness memo
     (`memo.blockMarksIdIndex`), built once per memo generation like
     `dependencyTasksIndex`. It reports "not ready" when `planBlockTasks` is null or
     empty.
   - `blockMarkProjectReason(notePath)`: one hop to the note's `^prj` cached task.
   - `blockMarkModelForLine(path, lineText)`: assembles `ctx` from the memo, the cached
     priority ladder snapshot used by priority marks, and `dateMarksToday()`.
   - `this.blockMarksEnabled`: session state, default `true`. The toggle UI comes in
     `editor_marks`.
3. **Expose `api.blockMarks` v1 `model`** in `350-exports.js`, and document the
   namespace in bob-plugins `docs/develop/plugin-apis.md`.
4. **Tests.** Add `scripts/test-ledger-tools-block-marks.cjs` and register it in the
   root `package.json` test list. It runs every BM vector except BM24 verbatim, with
   fake Tasks tasks and a fixed today. It also covers duplicate ids (one open instance
   suffices), a `^prj` line with its own Depends-On (`dep`), and `modelKey` stability.
5. `npm run build`, `npm test`, bump `plugins/bob-ledger-tools/manifest.json` (minor),
   `npm run docs:gen`, `bob plugins sync`.

**Done when:**

- the vectors pass;
- `npm test` is green;
- nothing renders differently in Obsidian yet.

## Phase: `editor_marks` — checkbox rendering, CSS, toggle, tooltips (bob-ledger-tools)

This phase is `large`: plan before implementing. The spec is `docs/blocked-marks.md`.

1. **Live Preview.** A `Prec.highest` ViewPlugin, mirroring
   `265-plugin-priority-marks.js` setup, emits `Decoration.line({ attributes })` on
   visible `[?]` task lines. It uses the row attributes from the design. Rebuild per the
   invalidation rules, with a dedicated refresh `StateEffect` dispatched from the
   triggers listed there. Hook the date-marks midnight rollover and a `visibilitychange`
   resume handler (registered through `registerDomEvent`).
2. **Reading view.** A markdown post-processor finds `li.task-list-item[data-task="?"]`
   that are not `.plugin-tasks-list-item`. It resolves each row's source line through
   `ctx.getSectionInfo` + the `li`'s `data-line` (see
   `276-plugin-today-marks-reading.js` and `279-plugin-task-theme-reading-dom.js`). When
   that fails, it falls back to the row's own text (`dependencyReadingOwnText`), and if
   that is still not canonical it leaves the row alone. It sets the row attributes on
   the `li` and `aria-label` + `data-tooltip-position="top"` on the `li`'s own checkbox.
   Rendered rows re-classify in place on the refresh triggers.
3. **CSS** in `plugins/bob-ledger-tools/styles.css`, all under `body.bob-block-marks`:
   - the ink tokens and `--bob-block-defer-tint`;
   - the three families for Live Preview
     (`.markdown-source-view.mod-cm6 .HyperMD-task-line[data-task="?"][data-bob-block]`)
     and reading view
     (`li.task-list-item[data-task="?"][data-bob-block] > .task-list-item-checkbox`);
   - the per-key `--task-status-symbol` overrides;
   - the `prj` and `repeat` masks on `::after`;
   - the pip on `::before`, with the box's `overflow` kept visible;
   - the line tints and rails per the `hollow_tint` decision;
   - the reduced-motion guard.

   Use specificity over the vault snippet, never `!important`, and style only the row's
   own box.

4. **Tooltips in Live Preview.** Use a capture-phase `mouseover` handler on the editor.
   When the target is a `.task-list-item-checkbox` inside a `[data-bob-block]` line, it
   sets that input's `aria-label` + `data-tooltip-position="top"` from the line's model
   before Obsidian's tooltip handler runs. Prove in the e2e that the tooltip shows and
   that clicking the box still cycles exactly as before. If the capture-phase approach
   can't show a tooltip reliably, fall back to setting the attributes on visible
   checkboxes in a `requestMeasure` write pass after each rebuild. Never wrap or replace
   Obsidian's checkbox widget.
5. **Toggle.** Command `toggle-block-marks` ("Toggle blocked-reason marks"). Use
   `applyMarkGateClass` / `registerMarkGatePopoutSync` like priority marks. It refreshes
   editors and rendered rows, triggers the Tasks re-render event, and shows the Notice.
   When off, JS creates no attributes and the vault looks exactly as today.
6. **`api.blockMarks.render`**: the listener-free miniature box (shares the family CSS).
7. **Contrast test**: `scripts/test-ledger-tools-block-marks-contrast.cjs`. It mixes the
   tokens in sRGB from Obsidian default-theme light and dark values and asserts at least
   4.5:1 for each `waiting` ink on its wash over the line tint, and for white on the
   solid box. Record the measured values in the contract table (bob-cli).
8. **Real-Obsidian e2e.** Add `scripts/obsidian-block-marks-e2e.cjs` + an offline
   `--self-test` wrapper in `npm test`, cloned from `obsidian-task-link-marks-e2e.cjs`.
   - Seed a scratch vault with one note of BM rows. Use dates relative to the real
     today, so the vectors stay valid.
   - Copy the vault's `task-statuses.css` snippet, enabled through `appearance.json`
     `enabledCssSnippets`, plus the Tasks and Dataview plugins from `~/bob` when
     present.
   - Assert `data-bob-block*` on each row in Live Preview and reading view. Assert that
     the toggle removes them and that nested children have no mark.
   - Capture light and dark screenshots of both views.
   - Run it with `xvfb-run -a node scripts/obsidian-block-marks-e2e.cjs`. **Look at the
     PNGs** and compare them against the research mockup (alignment, baseline, glyph
     weight, pip clipping, tint balance). Iterate until they look right.
9. Update bob-plugins `docs/bob-ledger-tools.md` (_Marks and rendered blocks_, _Session
   toggles_). Then bump the manifest (minor), `npm run build`, `npm test`,
   `npm run docs:gen`, `npm run docs:check`, `bob plugins sync`. Update bob-cli
   `docs/blocked-marks.md` with any measured value or mechanism detail that changed.

**Done when:**

- every BM row except BM24 renders its mark in both views, in both themes;
- clicking a marked box behaves exactly like today;
- the toggle restores today's look;
- the e2e and `npm test` pass.

## Phase: `redundancy_folds` — fold what the box already says (bob-ledger-tools)

1. **Priority bars on digit rows.** In `265-plugin-priority-marks.js`, when block marks
   are on and `blockMarkModelForLine` gives `p1`–`p4`, Live Preview emits a zero-width
   folded mark instead of the bars. That is a `PriorityMarkWidget` variant rendering an
   empty `span.bob-priority-mark[data-folded="true"]`. Keep fold-space handling,
   selection reveal, and click-to-reveal intact. The reading-view post-processor
   replaces the field text with the same folded element, so Dataview's pill never
   appears. Every other row keeps its bars: `dep`, `prj`, `date`, `repeat`, `stale`,
   lanes, and closed rows.
2. **`#prj_is_blocked` pill on `prj` rows.**
   - Live Preview: a replace decoration over the exact tag token and its leading space.
     It hides while the selection overlaps, and a click reveals it, like the priority
     fold.
   - Reading view: mark the exact `a.tag` element (`href="#prj_is_blocked"`) with
     `data-bob-block-folded="true"`, hidden by gated CSS.
   - Rows whose mark isn't `prj` keep the pill, for example BM19's `dep` row.
3. Both folds require both gates: block marks on and, for bars, priority marks on.
   Either toggle off restores the raw look.
4. Tests:
   - extend `test-ledger-tools-priority-marks.cjs` with PM18/PM19;
   - add fold vectors to the block-marks test (BM1 folded, BM8/BM9 shown, BM7 pill
     folded, BM19 pill kept);
   - extend the e2e assertions and screenshots.
5. Bump the manifest, build, test, update bob-plugins docs, run docs:gen, run
   `bob plugins sync`.

**Done when:**

- digit rows show no bars and `prj` rows no pill;
- the cursor reveals both raw fields;
- everything else is unchanged.

## Phase: `tasks_results` — Tasks query results, CSS only (bob-ledger-tools)

1. **Prototype first** in the e2e harness. Add a note with Tasks queries over the BM
   rows, including `short mode` and a `hide priority` variant. Confirm what Tasks 8.4.0
   puts on `li.plugin-tasks-list-item`:
   - `data-task`;
   - `data-task-priority`;
   - `data-task-scheduled` values (`today` / `future-…` / `past-…`);
   - `.tag[data-tag-name]`;
   - any recurring marker.

   Record the findings in the contract's _Tasks query results_ section. Do **not**
   resolve rows by `data-line` (Tasks writes the result index there), and don't infer
   status from `input.checked`.

2. **CSS** under `body.bob-block-marks` on `li.plugin-tasks-list-item[data-task="?"]`:
   - `:has(.tag[data-tag-name="#prj_is_blocked"])` → `prj`. This comes first under
     either precedence decision, because Tasks rows can't see open prerequisites;
     document it as a known limit.
   - Else a future `data-task-scheduled` plus `data-task-priority` in
     `high|medium|low|lowest` → the digit.
   - Else a future date → hollow `?`, or `⟲` if a recurring marker proved usable.
   - Else → `&` by elimination.

   Also:
   - hide `.task-priority` on digit rows (the clip pattern priority marks already use,
     never `display:none`);
   - hide the `#prj_is_blocked` tag on `prj` rows;
   - no pip, no tooltip.

3. Extend the e2e with assertions on computed styles and screenshots. Update the
   contract's known-limits list, bump the manifest, build, test, docs:gen,
   `bob plugins sync`.

**Done when:** `blocked.md`-style Tasks queries show the marks in both themes. BM24
passes in the e2e. The toggle restores Tasks results.

## Phase: `chips` — miniature marks in DEPENDS ON / BLOCKS chips (bob-ledger-tools)

1. **The state glyph.** Where `dependencyChipModel` and the BLOCKS chip builder mark a
   target as `blocked` (state symbol `?`), compute the target's block mark from its
   cached `originalMarkdown` and path through `blockMarkModelForLine`. Render the
   miniature box via the shared element builder in place of the plain `?` glyph in
   `150-dependency-element.js` / `152-blocks-widget.js`, in both Live Preview and
   reading paths. Neutral models keep today's `?`.
2. **Chip tooltip.** It gains the target's reason line (for example
   `Blocked · P2 deferral · until Fri, Nov 20 · in 19 days`).
3. **Rest of the chip.** Chip layout, hit areas, ＋/× controls, and navigation are
   unchanged.
4. **Tests** in `test-ledger-tools-dependency-chips.cjs` /
   `test-ledger-tools-blocks-chips.cjs` for each family. Extend the e2e with a
   `DEPENDS ON` line whose targets are `p2`, `dep`, and `prj` rows.
5. **Ship.** Manifest bump, build, test, docs, `bob plugins sync`.

## Phase: `task_card` — reason header and teaching digit (bob-navigation-hotkeys)

1. **Header.** For a single Blocked target, the header's BLOCKED lane chip
   (`320-task-card-header.js`, `getTaskCardLaneChipLabel`) renders the miniature block
   mark through `api.blockMarks.render`. Get the api like `getLedgerPriorityMarksApi` in
   `280-notices-and-dates.js`. Below it, a compact reason list shows every reason,
   winner first, in the tooltip words:
   - `after <text>`, one per open prerequisite, up to 3 then `+N`;
   - `via project <stem>`;
   - `until <short date> · <relative>`.

   Counted batches with mixed marks show today's plain BLOCKED chip.

2. **Teaching digit.** In the P1–P4 strip, a decorative (`aria-hidden`) hollow digit box
   sits beside each level's bars, using `render` with a synthetic `p<n>` model. Do this
   only when `api.blockMarks` is present.
3. **Fallback.** Without bob-ledger-tools or the api, the card looks exactly as today.
   This card is the touch and mobile path, where hover tooltips don't exist.
4. **Tests.** Extend `scripts/test-navigation-task-card-view.cjs` (and the header tests)
   for:
   - each family;
   - the mixed batch;
   - api absent.
5. **Ship.** Update bob-plugins `docs/task-card.md` and the nav manifest description
   (minor bump), then build, test, docs:gen, `bob plugins sync`. Add a Task Card line to
   the contract's surfaces table in bob-cli if wording changed.

## Phase: `visual_review` — cross-surface polish and live checklist

1. **Screenshots.** Run every e2e (`obsidian-block-marks-e2e.cjs` plus the existing
   ones, to catch regressions) in light and dark. Read every screenshot and compare it
   with the research mockup. Fix small polish defects:
   - glyph centering and baseline;
   - pip clipping;
   - tint balance between the `held` and `waiting` rows;
   - chip and card alignment.

   Anything bigger than polish becomes a `PROPOSED FOLLOW-UP:` note on this phase's
   bead.

2. **Suites.** Run the full `npm test`, `npm run validate`, and `npm run docs:check` in
   bob-plugins, and `just docs-check` in bob-cli. Finish with `bob plugins sync`.
3. **Checklist.** Finalize the contract's _Live verification (Bryan, in Obsidian)_
   checklist. Mark what the headless runs verified, and leave pending only what needs a
   human or device: iOS, Style Settings checkbox packs, a 100-task project note
   scrolling smoothly, and a real reconcile turning a row `stale` and then clear.
4. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes for:
   - Rust `BlockReason` emitting the same mark key for `bob plan` and the Mac Capture
     agenda (with a shared JSON vector fixture then);
   - moving `task-statuses.css` from a vault snippet into the plugin.

## Cross-cutting rules for every bob-plugins phase

- **Why the bob-ledger-tools phases are chained.** `redundancy_folds` → `tasks_results`
  → `chips` share the generated `main.js`, `styles.css`, the manifest version, and the
  docs inventory. Chaining them avoids regenerated-file conflicts. `task_card` touches
  only bob-navigation-hotkeys, so it can run beside them once `editor_marks` lands.

- Edit only `src/` fragments. Keep each hand-edited fragment at or below 1000 lines,
  never hand-edit generated `main.js`, and run `npm run build` then `npm test` (which
  starts with `build:check`).
- Bump the touched plugin's `manifest.json` minor version once per phase. Regenerate the
  docs inventory with `npm run docs:gen`.
- After any change, run `bob plugins sync` to deploy to the vault (bob-plugins
  `AGENTS.md`).
- Changes in both repos are part of the phase's commit obligation. Commit bob-plugins
  and bob-cli together through the normal finalizer.
- Display surfaces never act: no mark writes the vault, intercepts a click, or opens a
  card.
