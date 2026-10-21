---
tier: epic
title: PRE/POST checklist glyphs and compact recurrence marks
goal: "In Obsidian, a canonical `#task #gtd #pre` / `#task #gtd #post` run renders as
  one teal sunrise / flag identity glyph, and every readable `[repeat:: …]` rule renders
  as a quiet loop glyph plus a spoken cadence (`daily`, `Mondays`, `monthly · 1st`).
  This works in Live Preview, reading view, and Tasks query results. Nothing changes in
  storage, and anything uncertain keeps its honest pill.

  "
decisions:
  when_done_label:
    ask:
      Should recurrence marks also print `when done` on the line (for example `daily
      when done`)?
    default: false
    why:
      It is identical on 9 of 10 checklist rows; the tooltip and the repair pill already
      carry it
    answer: false
phases:
  - id: checklist-glyphs
    title: PRE/POST checklist identity glyphs
    depends_on: []
    size: medium
    description:
      "checklist-glyphs: render a canonical, conflict-free `#task #gtd #pre|#post` run
      as one teal sunrise or flag in the task identity slot. Covers Live Preview,
      rendered views, CSS-only Tasks rows, and the Today 🍅 reading-view anchor. Also
      adds the shared checklist-membership helper the recurrence guard reuses, and the
      task-tag-marks / today-marks contract sections."
  - id: recurrence-marks
    title: Recurrence marks in Live Preview and rendered views
    depends_on:
      - checklist-glyphs
    size: medium
    description:
      'recurrence-marks: render each canonical, Tasks-readable `[repeat:: …]` as a quiet
      date-family mark (loop glyph plus spoken cadence label, exact rule in the
      tooltip). Adds the checklist `when done` guard, the repeat repair flag, the
      "Recurrence (not a date)" date-marks contract section, and the vault-format
      `repeat` row.'
  - id: recurrence-tasks
    title: Recurrence marks in Tasks query results
    depends_on:
      - recurrence-marks
    size: small
    description:
      "recurrence-tasks: extend the full-mode Tasks frame pass to `span.task-recurring`,
      draw the loop glyph in short mode with CSS, add `hide recurrence rule` to the
      vault's blocked.md RECURRING query, and add the RT conformance vectors."
proposed_by: bbugyi200.apollo.research.28.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-21 08:49:57
status: wip
---

- **PROMPT:**
  [prompts/202610/gtd_checklist_glyphs_and_recurrence_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/gtd_checklist_glyphs_and_recurrence_marks.md)

# Plan: PRE/POST checklist glyphs and compact recurrence marks

## Context

This plan implements the recommended solution in the research report
`research:202610/gtd_checklist_glyphs_and_recurrence_marks/gtd_checklist_glyphs_and_recurrence_marks.md`.
Bryan accepted every requirement and adjustment in that report (A1–A7). Each phase agent
should read it first with `sase artifact read <ref> "<reason>"`. Its Figure 1 mockup
(`…__final_assets/mockup.png`) shows the target look. This plan settles the report's
open details and splits the work into phases. Where this plan and the report differ,
**this plan wins**: it changes tooltip wording, places the new fragments, and states
guard scope precisely.

Baselines: bob-cli `c1301621`, bob-plugins `ee16031` (bob-ledger-tools 1.60.3), Tasks
8.4.0 (`taskFormat: dataview`, `globalFilter: #task`). Today there are 10 open checklist
rows, all in `gtd_daily.md` and all canonical. The vault has 973 `[repeat:: …]` fields,
954 of them `every day when done`.

### Design principles (apply to every phase)

1. **Display-only.** Stored text never changes. No storage, tag, Tasks-setting, Rust,
   CLI, `bob query`, or Mac Capture change. Rendering writes zero bytes.
2. **Truthful or neutral.** A glyph appears only when the display is certain. Anything
   conflicting, non-canonical, or unreadable keeps its native pill. In Live Preview that
   leftover pill is the visible "fix me" signal.
3. **One family each.** The checklist glyphs join the task-tag mark family (teal
   identity ink, one identity slot, as the 📖 book does). Recurrence joins the date-mark
   family (muted ink, glyph plus label). No new toggles, commands, settings, or api
   namespaces. `api.dateMarks` v1 stays dates-only.
4. **Reversible.** The cursor or a click reveals the raw text. The two existing session
   toggles (`toggle-task-tag-marks`, `toggle-date-marks`) restore the pills. Without the
   plugin, the vault looks as it does today.

### Shared rules for every phase

- **Repos.** Open bob-plugins with `/sase_repo`
  (`sase repo open bob-plugins -r "<reason>"`) and read its `AGENTS.md`. Code lives in
  `plugins/bob-ledger-tools/src/` (fragments listed in `src/fragments.json`) and
  `plugins/bob-ledger-tools/styles.css`. Contracts live in this bob-cli repo under
  `docs/`.
- **Fragment budget.** Every hand-edited fragment must stay at or below 1,000 lines.
  Current counts: `138` 811, `264` 729, `136` 640, `266` 985, `267` 485, `277` 155,
  `276` 994. Never touch `276`. New pure logic goes in the new core fragments named
  below. A new fragment shares an existing numeric prefix and is listed right after its
  sibling in `fragments.json` (`239-*` is the precedent). Never hand-edit the generated
  `main.js`.
- **Code style.** Match the surrounding mark code: synchronous, never throws
  (`try`/`catch` with `null` / empty-list fallbacks), frozen return values,
  listener-free DOM builders that survive Dataview's `innerHTML` round trip, and
  tooltips that never contain `::`. Export every new pure helper and constant from
  `src/350-exports.js` so tests can reach it.
- **Tests.** Add a new `scripts/test-ledger-tools-*.cjs` suite per phase (stubbed the
  way `scripts/test-ledger-tools-task-tag-marks.cjs` / `-date-marks.cjs` are) and
  register it in the root `package.json` `test` script. Run the conformance vectors from
  the contract verbatim. Also pin the `styles.css` contract with a CSS-text test, as the
  existing suites do.
- **Gates (bob-plugins).** `npm run build`, `npm run build:check`, `npm test`,
  `npm run docs:check`.
- **Version and description.** Bump the bob-ledger-tools `manifest.json` minor version
  from whatever is current (a feature), and extend its `description` with one clause for
  the phase's feature. Run `npm run docs:gen` to refresh the generated facts in
  `docs/bob-ledger-tools.md`, and update that page's hand-written mark bullets (around
  the "task tag mark" / "date mark" bullets).
- **Deploy.** Run `bob plugins sync` from the opened bob-plugins checkout, passing
  `--repo` with that checkout's path if it refuses. Confirm that the vault's
  `.obsidian/plugins/bob-ledger-tools/manifest.json` shows the new version.
- **Gates (bob-cli docs).** `just docs-gen`, then `just docs-check`. Commit the
  generated output. bob-cli commits are `docs(...)` with no `!`, because no CLI or JSON
  contract changes. Never edit `Cargo.toml`'s version or `CHANGELOG.md` (the release
  train owns them).
- **No decision record or memory change** is needed. These are display-only extensions
  governed by the existing display contracts.

---

## Phase checklist-glyphs: PRE/POST checklist identity glyphs

### What you see

```text
before: - [?] ⌗ #gtd #pre Brush teeth  REPEAT│every day when done  + today  ⧗ tomorrow
after:  - [?] ☀ Brush teeth  REPEAT│every day when done  + today  ⧗ tomorrow     (☀ teal sunrise)
        - [?] ⚑ Morning review: PRE chores done … then ]S and complete this …   (⚑ teal flag)
        - [x] ☀ Check weather …                                                 (closed: rests faint)
Today:  - [ ] ☀ 🍅 Brush teeth ^teeth                                            (🍅 follows the slot)
pills:  - [ ] ⌗ #gtd #pre #post Conflict stays raw  /  - [ ] ⌗ #pre Missing gtd
```

The whole identity run `#task #gtd #pre` (or `#post`) becomes **one** glyph in the slot
where the hash sits, exactly like `#task #ref` → 📖. The shape says the kind (sunrise =
the morning chores that open the review; flag = the finish line that closes it). The
teal ink still says "tracked task".

### Glyphs

Use the hash/book stroke language: `viewBox 0 0 16 16`, `fill none`, `stroke-width 1.8`,
round caps and joins, used as a CSS mask, tunable by ±0.3 units. Define both on `body`
beside `--bob-ref-task-glyph`:

```text
--bob-pre-task-glyph  (sunrise): M1.4 12.2h13.2M3.8 12.2a4.2 4.2 0 0 1 8.4 0M8 3.9v2.4M2.13 6.33l1.7 1.7M13.87 6.33l-1.7 1.7
--bob-post-task-glyph (flag):    M3.6 14.2V2.3M3.6 2.9h8.6l-2.1 3.2 2.1 3.2H3.6
```

### Labels and tooltips

These are constants in the new core fragment. Lines are joined with `\n`, never contain
`::`, and follow the ref precedent: `aria-label` = label, `title` = tooltip,
`data-tooltip-position="top"`.

| Kind   | Variant class        | `aria-label`                  | Tooltip                                                                                        |
| ------ | -------------------- | ----------------------------- | ---------------------------------------------------------------------------------------------- |
| `pre`  | `bob-pre-task-mark`  | `PRE opening checklist task`  | `#task #gtd #pre · PRE opening checklist` ⏎ `Opens the ]s walk · Ctrl+Enter when done`         |
| `post` | `bob-post-task-mark` | `POST closing checklist task` | `#task #gtd #post · POST closing checklist` ⏎ `Closes the ]s walk (]S) · Ctrl+Enter when done` |

The glyph never encodes status. It does not change when a row is Blocked,
future-scheduled, or done. Closed rows (`x X -`) rest at the family's existing 40% mix
toward `--text-faint`, opacity 0.75, through the shared `.bob-task-tag-mark` resting
rule.

### Core: new fragment `src/138-checklist-task-marks.js` (listed after `138-task-tag-marks.js`)

Pure helpers, constants, and the widget:

- `checklistTagTokens(text)`: the whole-token tags in `text`. A token is `#` preceded by
  the text start or whitespace, followed by one or more tag characters
  (`/[\p{L}\p{N}_/-]/u`, the same class `TASK_TAG_MARK_TAG_CHAR_RE` uses), maximal
  munch. It returns lowercased token strings, so `#pre,` yields `#pre`, while
  `#gtd/pre`, `#prep`, and `#pressed_juice` are their own tokens.
- `checklistKindForText(text)`: membership exactly as the evaluator defines it. It
  delegates to the existing `freshnessChecklistForTags(checklistTagTokens(text))`, so
  `pre` | `post` | `null`, and PRE wins a conflict. **The `recurrence-marks` phase
  reuses this helper.**
- `checklistTagConflict(text)`: true when the whole-token tags include both `#pre` and
  `#post`.
- `checklistTaskRunEnd(lineText, taskTo)`: `{ to, kind }` or `null`. The match after the
  `#task` token ending at `taskTo` is one whitespace run, tag token A, one whitespace
  run, tag token B. Lowercased, {A, B} must equal {`#gtd`, `#pre`} → `pre` or {`#gtd`,
  `#post`} → `post`, in **either order**. It returns `null` when
  `checklistTagConflict(lineText)` is true.
- `checklistTaskRunAnchors(taskAnchor)`: the reading-view twin,
  `{ kind, anchors: [a, b] }` or `null`. It takes the next two element siblings of the
  `#task` anchor, skipping whitespace-only text nodes (any other node in between means
  `null`, as in `refTaskPairAnchor`). Both must be `a.tag` anchors whose lowercased text
  (and `href`, if present) form the same pair as above. The conflict gate scans the
  nearest `li` ancestor's **own** `a.tag` anchors, skipping nested
  `ul`/`ol`/`code`/`pre`, for both `#pre` and `#post`. It reads text only, never
  annotation classes, so the Today code can call it whether or not the marks are on.
- `CHECKLIST_TASK_HIDDEN_CLASS = "bob-checklist-task-hidden"`, plus
  `annotateChecklistTaskMark(taskAnchor, run)` and `stripChecklistTaskHidden(el)`.
  Annotation calls `annotateTaskTagMark`, adds the variant class, and sets `aria-label`
  / `title` from the table. It then hides both partner anchors in place with the hidden
  class, never removing them, so copy, Dataview's round trip, and the toggle keep
  working.
- `ChecklistTaskMarkWidget(kind)`: `eq` compares the constant key `pre-task-mark` or
  `post-task-mark` (no flicker). `toDOM` builds through
  `buildTaskTagMarkElement(doc, kind)` and adds `addTaskTagMarkReveal`. It is defined
  only when `WidgetType` exists.

Edits to `src/138-task-tag-marks.js`. The fragment has room, but keep the edits small:

- `taskTagMarkRanges`: for each eligible `#task` token, check the ref pair first
  (unchanged). Otherwise, if `checklistTaskRunEnd` matches, emit one
  `{ from, to: run.to, kind }` range over the whole run. Otherwise emit `task`. `kind`
  is now `task | ref | pre | post`. Update the doc comment.
- `buildTaskTagMarkElement(doc, kind)`: delegate `pre`/`post` to the table above
  (classes `bob-task-tag-mark bob-pre-task-mark` and the matching post class).
- `stripTaskTagMark`: also remove `bob-pre-task-mark` and `bob-post-task-mark`.

Code exclusion stays in the decoration builder
(`freshnessMarkPosInCode(tree, absFrom + 1)`). A run whose partner sits inside code
fails the whitespace-run match because of the backtick.

### Plugin side: `src/264-plugin-task-tag-marks.js`

- `taskTagMarksAvailable` also requires `ChecklistTaskMarkWidget`.
- `buildTaskTagMarkDecorations`: use `new ChecklistTaskMarkWidget(kind)` for
  `pre`/`post` ranges. One `Decoration.replace` covers the whole run, so any inclusive
  selection touch reveals all three tags. `]s` landings at column 0 never reveal.
- `annotateTaskTagAnchor`: ref partner first, then `checklistTaskRunAnchors` →
  `annotateChecklistTaskMark`, else the hash.
- `stripTaskTagMarksInDocument`: also collect `a.tag.bob-checklist-task-hidden` (in both
  the `querySelectorAll` branch and the manual-walk fallback) and reveal it with
  `stripChecklistTaskHidden`. The existing loop already strips the `#task` anchor.

### Today 🍅 placement

- **Live Preview needs no code change.** `todayMarkInsertOffset` (`143`) already follows
  the first range when it starts at the body start, and that range now spans the run. So
  🍅 lands after the sunrise, at offset 22 for
  `- [ ] #task #gtd #pre Brush teeth ^teeth`.
- **Reading view.** In `src/277-plugin-today-marks-reading-clicks.js`
  `todayMarkInsertAnchor`, after the ref-partner check, add: if
  `checklistTaskRunAnchors(tag)` matches, insert after its second anchor. Do **not**
  edit `276`.

### Tasks query results (CSS-only, like `#task` and `#ref`)

These rules live inside
`body.bob-task-tag-marks .plugin-tasks-list-item .task-description`. Match tag pills by
`a.tag:is([data-tag-name="#x" i], [href="#x" i])`.

- **`#pre` pill:** draws the sunrise when the description also has a `#gtd` pill. Reuse
  the ref-pill box rules: 1.08em box, transparent text, and a `::before` mask in the
  teal ink.
- **`#post` pill:** draws the flag under the same condition.
- **`#gtd` pill:** gets `display: none` when the description has a `#pre` or `#post`
  partner.
- **Conflict:** scope every rule out of descriptions that contain both `#pre` and
  `#post`, using `:has()` / `:not(:has(...))`.
- **Resting:** add one Tasks-row resting rule
  (`li.plugin-tasks-list-item[data-task="x"|"X"|"-"]`) for the book and both new glyphs,
  so closed rows rest faint like everywhere else.
- **Known limit (document it):** CSS cannot enforce the slot gate, because sibling
  combinators ignore text. This is harmless, since every vault row is canonical.

Check `data-tag-name` against the installed Tasks `main.js`. JS never touches Tasks'
DOM.

### Contract updates (bob-cli)

- **`docs/task-tag-marks.md`:**
  - Add a "GTD checklist tasks" section after "Reference reading tasks" covering: what
    you see, glyph paths, ink and tones, tooltips and labels, eligibility (run, either
    order, conflict gate, slot), surfaces, Tasks CSS with its known limit, and toggle
    behavior.
  - Change the core description to `kind` ∈ `task | ref | pre | post`.
  - Add the CK/CR vectors below to "Conformance vectors".
  - Add live-verification lines.
  - Note that pickers, notices, the footer, CLI/TUI, `bob query`, and Mac Capture are
    unchanged.
- **`docs/today-marks.md`:** in "Placement", the slot is also the PRE/POST checklist run
  (☀/⚑). Extend TM16 with the `#task #gtd #pre` case (🍅 after the sunrise run).
- **`docs/_meta/contracts.yml`:** broaden the task-tag-marks summary, for example "the
  teal identity glyphs (hash, book, PRE sunrise, POST flag) that replace task identity
  tag pills". Then run `just docs-gen`.

### Conformance vectors (`taskTagMarkRanges` unless noted)

| #    | Input                                                                                                    | Expected                                                                                      |
| ---- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| CK1  | `- [ ] #task #gtd #pre Brush teeth`                                                                      | `[{[6,21), pre}]`                                                                             |
| CK2  | `- [?] #task #gtd #post Morning review`                                                                  | `[{[6,22), post}]`                                                                            |
| CK3  | `- [x] #task #gtd #pre Check weather [completion:: 2026-10-21]`                                          | `[{[6,21), pre}]` (resting tone is CSS)                                                       |
| CK4  | `- [ ] #task #pre #gtd Swapped`                                                                          | `[{[6,21), pre}]`                                                                             |
| CK5  | `- [ ] #task #GTD #Pre Loud`                                                                             | `[{[6,21), pre}]`                                                                             |
| CK6  | `- [ ] #task  #gtd   #post Extra spaces`                                                                 | `[{[6,25), post}]`                                                                            |
| CK7  | `- [ ] #task #gtd #pre #post Conflict` / `- [ ] #task #gtd #pre Brush #post`                             | `[{[6,11), task}]`; pills stay                                                                |
| CK8  | `- [ ] #task #gtd Topic only` / `- [ ] #task #pre Missing gtd`                                           | `[{[6,11), task}]`                                                                            |
| CK9  | `- [ ] #task #gtd #prep Typo` / `- [ ] #task #gtd/pre Nested`                                            | `[{[6,11), task}]`                                                                            |
| CK10 | `- [ ] #task Brush teeth #gtd #pre`                                                                      | `[{[6,11), task}]` (not in the slot)                                                          |
| CK11 | `- [ ] #task #ref #gtd #pre Odd`                                                                         | `[{[6,16), ref}]` (the ref pair wins)                                                         |
| CK12 | `- #task #gtd #pre Not a task` / ``- [ ] Note `#task #gtd #pre` ``                                       | none / none in Live Preview (code)                                                            |
| CK13 | `- [ ] #task #gtd #pre Brush teeth ^teeth`, Today-linked                                                 | 🍅 widget at offset 22                                                                        |
| CK14 | `checklistKindForText` on `#gtd #pre` / `#GTD #Post` / `#gtd #pre #post` / `#gtd/pre` / `#pre,` + `#gtd` | `pre` / `post` / `pre` / `null` / `pre`                                                       |
| CR1  | reading view: task li with adjacent `#task #gtd #pre` anchors                                            | `#task` drawn as the sunrise (variant class, label, title); `#gtd` and `#pre` hidden in place |
| CR2  | reading view: `#task #gtd #pre #post`, or `#task` + `#gtd` + a word + `#pre`                             | hash only; all pills visible                                                                  |
| CR3  | toggle off, then on                                                                                      | variant classes, labels, and hidden classes stripped document-wide; then re-annotated         |
| CR4  | Today reading view on a CR1 row                                                                          | 🍅 inserted after the hidden `#pre` anchor (before the title)                                 |
| CR5  | CSS: Tasks row with `#gtd` + `#post` pills / with `#pre` + `#post`                                       | flag drawn, `#gtd` hidden / all pills native                                                  |

TT3 (`#task #gtd #context/home`) and every existing TT/TR/TM vector stay green
unchanged.

### Done when

All gates pass. The plugin is synced. In the vault, `gtd_daily.md` rows render ☀/⚑ in
Live Preview and reading view, and a hand-typed conflict row keeps its pills. Commit
bob-plugins (`feat(ledger-tools): …`) and bob-cli (`docs(task-tag-marks): …`).

---

## Phase recurrence-marks: Recurrence marks in Live Preview and rendered views

### What you see

```text
before: - [?] ☀ Brush teeth  REPEAT│every day when done  + today  ⧗ tomorrow
after:  - [?] ☀ Brush teeth  ⟲ daily  + today  ⧗ tomorrow                       (⟲ muted loop)
        - [?] ⚑ Weekly prune: …  ⟲ Mondays  + Oct 19  ⧗ Mon
        - [ ] ⌗ Pay Kelly salary!  ⟲ monthly · 1st  + Oct 8  ⧗ Nov 1
        - [ ] ⌗ Update cash balance snapshots  ⟲ every 30 days  + Aug 31  ⧗ Oct 30
guard:  - [?] ☀ Hand-typed row  [REPEAT│every day]  ⧗ tomorrow     (dashed orange repair pill)
```

> [!decision] when_done_label When the rule ends in `when done`, append ` when done` to
> the label (`daily when done`, `Mondays when done`, `monthly · 1st when done`,
> `every 30 days when done`). Pin this in the grammar vectors (RM1/RM3/RM5 expectations
> gain the suffix). The tooltip is unchanged.

> [!decision] when_done_label = no The label never includes `when done`. The tooltip's
> second line and the checklist guard carry it.

### Anatomy

Recurrence is a fifth member of the date-mark family and reuses its element, widget,
metrics, fold, reveal, toggle, and resting tone. Only the date logic differs; it never
parses a date.

- **Element.** `span.bob-date-mark[data-field="repeat"]` with `data-rule` (the rule as
  written, whitespace-collapsed), `data-when-done` (`true`/`false`), and
  `data-fold-space` (+ `data-rendered="true"` on rendered views). It contains the usual
  `span.bob-date-mark-glyph` + `span.bob-date-mark-label`. It has **no** `data-date` and
  **no** `data-when`, and keeps `role="img"`, `aria-label` = tooltip, and
  `data-tooltip-position="top"`.
- **Glyph.** A rectangular two-arrow loop, a refined monochrome 🔁. It deliberately
  avoids the circular arrow, which already means `⟳ Review` and due freshness. Use
  stroke 1.8, round caps and joins, tunable by ±0.3:
  `M2.6 9.4V6.6A2.6 2.6 0 0 1 5.2 4h8.2M11 1.6 13.4 4 11 6.4M13.4 6.6v2.8A2.6 2.6 0 0 1 10.8 12H2.6M5 9.6 2.6 12 5 14.4`.
  Define it as `--bob-date-glyph-repeat`.
- **Ink.** Always the quiet tone: `--bob-date-color-repeat`, else `--text-muted`, at
  opacity 0.7 (like `created`). There is no emphasis state and no hue. **Closed rows
  rest faint.** The existing resting selector requires `[data-when]`, so add explicit
  `li.task-list-item[data-task=x|X|-] .bob-date-mark[data-field="repeat"]` and
  `.HyperMD-task-line[...]` rules.
- **Rollover.** `relabelRenderedDateMarks` already skips these marks, because
  `dateMarkModel` returns `null` for `repeat`. Pin this with a vector (RO2); add no code
  for it.

### Validity gate (all must hold, else the native pill stays)

1. **Occurrence gate.** Exactly one `/(^|[^A-Za-z0-9_-])repeat\s*::/gi` on the line or
   text node.
2. **Canonical form.** `[repeat:: …]` or `(repeat:: …)` with matching brackets, a
   lowercase key touching `::`, and padding spaces only.
3. **Value.** After trimming, the value matches Tasks' own charset `^[a-zA-Z0-9, !]+$`
   and starts with `every` plus a space and more text (case-insensitive).
4. **Code.** Positions in code are excluded, exactly as for dates.
5. **Checklist guard.** No mark when the line is a checklist member
   (`checklistKindForText` from the previous phase is non-null), the task is open
   (status not `x`, `X`, or `-`), and the rule lacks `when done`. This mirrors the
   `checklist_repeat_not_when_done` lint scope (open members), so the display never
   hides what the lint flags.
   - Live Preview reads the whole line plus `freshnessTaskStatus(lineText)`.
   - Rendered views read the nearest `li.task-list-item` ancestor's own `a.tag` anchors,
     skipping nested lists and code, and its `data-task`. Resolve this lazily, only for
     text that contains a repeat source.

**Known limit (document it):** the charset check is necessary but not sufficient. A rule
like `every blursday` gets a verbatim label, which is still truthful text. The
evaluator's lints remain the authority.

### Label grammar: "say it the way a person would"

Normalize first: trim, collapse whitespace runs, and strip a trailing `when done`
(case-insensitive) into `whenDone`. Match the rest case-insensitively against
`every <body>`. Labels are lowercase except weekday and month names, use `·` (U+00B7
with spaces) between cadence and detail, and are **never truncated**.

| Body (after `every `)                                                        | Label                                                                           |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `day` / `weekday`                                                            | `daily` / `weekdays`                                                            |
| `week` / `month` / `year`, or `1 week` etc.                                  | `weekly` / `monthly` / `yearly`                                                 |
| `N days` / `weeks` / `months` / `years`, N ≥ 2                               | `every N days` (and so on)                                                      |
| `<Weekday>` or `week on <Weekday>` (one day)                                 | `Mondays` (full name, plural)                                                   |
| `week on <Weekday list>` (two or more; `,` and/or `and`)                     | `weekly · Tue, Fri`                                                             |
| `N weeks on <Weekday list>`, N ≥ 2                                           | `every N weeks · Fri` / `every N weeks · Tue, Fri`                              |
| `month on the <ordinal>` / `last` / `last <Weekday>` / `<ordinal> <Weekday>` | `monthly · 1st` / `monthly · last` / `monthly · last Fri` / `monthly · 2nd Wed` |
| `N months on the <ordinal>[ <Weekday>]`, N ≥ 2                               | `every N months · 1st`                                                          |
| `<Month> on the <ordinal>`                                                   | `yearly · Jan 15`                                                               |
| anything else                                                                | the rule as written (collapsed), minus `when done`                              |

Ordinals keep their written token, lowercased (`1st`, `24th`). Weekday and month names
are full English in input and three-letter abbreviations in detail lists. The grammar
covers every rule in the vault today.

### Tooltip

The tooltip is two lines and never contains `::`.

- **Line 1:** `Repeats <rule as written, collapsed>`, for example
  `Repeats every day when done`.
- **Line 2:** `Next date counts from completion` when `whenDone`, otherwise
  `Next date follows the schedule`.

The accessible name is the same text, via `aria-label`, as for every date mark.

### Core: new fragment `src/136-recurrence-marks.js` (listed after `136-date-marks.js`)

Pure helpers:

- `recurrenceMarkCanonicalField(text)` → `{ rule, whenDone, fieldStart, fieldEnd }` or
  `null`.
- `recurrenceRuleParts(rule)` and `recurrenceMarkLabel(rule)`.
- `recurrenceMarkModel(rule)` → frozen
  `{ field: "repeat", rule, whenDone, label, tooltip, key }` or `null`.
- `recurrenceMarkGuarded(lineText)` and `recurrenceMarkRenderedGuard(textNode, root)`.
- `buildRecurrenceMarkElement(doc, model, options)`, honoring `foldSpace`, `rendered`,
  `decorative`, and `inheritColor` like `buildDateMarkElement`.
- Dispatchers that keep `266` nearly untouched:
  - `dateFamilyMarkSources(lineText)`: date sources plus at most one repeat source,
    guard applied, sorted by `fieldStart`, the same fold rule via `dateMarkFoldLength` /
    `dateMarkContentStart`. **RangeSetBuilder needs sorted adds.**
  - `dateFamilyMarkSourcesInText(text, guardResolver)`.
  - `dateFamilyMarkModel(source, today)`: `repeat` → `recurrenceMarkModel`, else
    `dateMarkModel`.

Edits:

- **`136-date-marks.js`:** `buildDateMarkElement` delegates `model.field === "repeat"`
  to `buildRecurrenceMarkElement`, so `DateMarkWidget` works unchanged (its key comes
  from `model.key`). Leave `DATE_MARK_FIELDS`, `dateMarkModel`, and `api.dateMarks`
  unchanged: `api.dateMarks.model("repeat", …)` stays `null`.
- **`266-plugin-date-marks.js` (985 lines):** swap `dateMarkSources` →
  `dateFamilyMarkSources` and `dateMarkSourcesInText` → `dateFamilyMarkSourcesInText`
  (pass a lazy guard resolver for the text node). Swap both `dateMarkModel(...)` calls →
  `dateFamilyMarkModel(source, today)`. Keep the net growth to a few lines. If it would
  still exceed 1,000, move `dateMarksApi()` verbatim into the `267` mixin, which
  installs onto the same prototype.
- **`styles.css`:** add the glyph var, the `data-field="repeat"` ink, opacity, and
  resting rules, and `[data-dv-norm-key="repeat"]` in the date-mark Live Preview
  repair-flag selector. With marks on, every leftover `repeat` pill (unreadable,
  duplicate, uppercase key, or checklist guard) is flagged. Template placeholders are
  flagged too, like dates.

### Contract updates (bob-cli)

- **`docs/date-marks.md`:**
  - Add a `repeat` row to "Fields and glyphs", flagged "not a date; see below".
  - Add a "Recurrence (not a date)" section covering anatomy, glyph, ink, validity gate,
    checklist guard, label grammar, tooltip, known limit, and the statement that
    `api.dateMarks` v1 stays dates-only.
  - Extend "Repair flag" with `repeat`.
  - Add the vectors below and the live-verification lines.
  - Update the intro and "What you see" so they no longer say "four" fields only.
- **`docs/reference/vault-format.md`:** add a `repeat` row to "Authored and derived
  fields": `[repeat:: every day when done]`, an authored Tasks recurrence rule that must
  start with `every`; PRE/POST checklist rows need `when done`; rendered as a recurrence
  mark. Link the new section.
- **`docs/_meta/contracts.yml`:** broaden the date-marks summary to "…for canonical task
  dates and recurrence rules". Then run `just docs-gen`.

### Conformance vectors

Today is `2026-10-21` unless noted; ⏎ separates tooltip lines.

| #    | Input                                                                                                                                                  | Expected                                                                                                                     |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| RM1  | `- [?] #task #gtd #pre Brush teeth  [repeat:: every day when done]  [created:: 2026-10-21]`                                                            | `daily`, fold 2, quiet; `Repeats every day when done⏎Next date counts from completion`; created mark unaffected and disjoint |
| RM2  | `- [ ] #task Water plants [repeat:: every day]`                                                                                                        | `daily`; `…⏎Next date follows the schedule` (not a checklist row, so no guard)                                               |
| RM3  | `every week on Monday when done` / `every week on Sunday` / `every Friday`                                                                             | `Mondays` / `Sundays` / `Fridays`                                                                                            |
| RM4  | `every month on the 1st` / `every month on the last Friday` / `every month on the 2nd Wednesday`                                                       | `monthly · 1st` / `monthly · last Fri` / `monthly · 2nd Wed`                                                                 |
| RM5  | `every 30 days when done` / `every 28 days`                                                                                                            | `every 30 days` / `every 28 days`                                                                                            |
| RM6  | `every 1 week` / `every 3 weeks on Friday` / `every January on the 15th`                                                                               | `weekly` / `every 3 weeks · Fri` / `yearly · Jan 15`                                                                         |
| RM7  | `every week on Tuesday, Friday` / `every week on Tuesday and Friday` / `every weekday`                                                                 | `weekly · Tue, Fri` (both) / `weekdays`                                                                                      |
| RM8  | `every April and December on the 1st and 24th`                                                                                                         | verbatim `every April and December on the 1st and 24th`                                                                      |
| RM9  | `(repeat:: every week)` / `[ repeat:: every  day ]`                                                                                                    | `weekly` / `daily`                                                                                                           |
| RM10 | `Every Day When Done`                                                                                                                                  | `daily`; tooltip `Repeats Every Day When Done⏎Next date counts from completion`                                              |
| RM11 | cursor inside the field / in its folded run                                                                                                            | reveals that field only / unfolds to fold 0                                                                                  |
| RM12 | `- [x] #task #gtd #pre Old [repeat:: every day]` (closed member)                                                                                       | mark shown (guard covers open rows only), resting tone                                                                       |
| RN1  | `[repeat:: weekly]`, `[Repeat:: every day]`, `[repeat :: every day]`, `[repeat:: every day; when done]`, two `repeat::` fields, `[repeat:: every day)` | no mark (pill; repair-flag selector covers `repeat`)                                                                         |
| RN2  | `- [?] #task #gtd #pre Brush [repeat:: every day]` / `- [ ] #task Brush #gtd #post [repeat:: every day]`                                               | no mark on either (the guard follows membership, not the glyph slot); the pill keeps the dashed orange flag                  |
| RN3  | ``- [ ] `[repeat:: every day]` ``                                                                                                                      | untouched (code)                                                                                                             |
| RR1  | reading view: CR1-style PRE li whose text holds `[repeat:: every day]` / `… when done`                                                                 | no mark / `daily` mark (`data-rendered="true"`)                                                                              |
| RO2  | rollover over a rendered repeat mark                                                                                                                   | untouched; no error                                                                                                          |
| RA1  | `api.dateMarks.fields` / `api.dateMarks.model("repeat", "every day")`                                                                                  | unchanged four fields / `null`                                                                                               |

### Done when

All gates pass and the plugin is synced. In the vault, `gtd_daily.md`, `recur.md`, and
`cash.md` rows show the labels above in Live Preview and reading view, and a hand-typed
checklist row without `when done` shows the flagged pill. Commit bob-plugins
(`feat(ledger-tools): …`) and bob-cli (`docs(date-marks): …`).

---

## Phase recurrence-tasks: Recurrence marks in Tasks query results

### Full mode (`dash.md` TODAY, `blocked.md` RECURRING)

Extend `src/267-plugin-date-marks-tasks.js`. `decorateTasksResultDates` gains a
`task-recurring` host. Confirm the DOM shape against the installed Tasks `main.js`; the
research saw `span.task-recurring` wrapping a single inner span with text
`" 🔁 <rule>"`.

1. Parse the rule by stripping the leading whitespace, `🔁` (U+1F501, optional U+FE0F),
   and surrounding spaces.
2. Validate it with the same value check (charset plus `every`).
3. Apply the checklist guard. Membership comes from the row's `.task-description a.tag`
   pills (`data-tag-name`, `href`, or text) through `freshnessChecklistForTags`;
   openness comes from the li's `data-task`.
4. Append `buildRecurrenceMarkElement(…, { foldSpace: true, rendered: true })` with
   `data-host="tasks"`, then flag the span `data-bob-date-mark="true"`. The existing
   clip rule then hides Tasks' inner span, and Tasks' listeners keep working through
   bubbling.

Unparseable, guarded, or unknown shapes stay native. The pass stays idempotent and
bounded by the existing frame/retry machinery, and `hide recurrence rule` is respected
automatically because nothing renders. Tasks may print its own normalized `toText()`
form of the rule; the grammar labels whatever text it shows.

### Short mode (`rotten.md`, `crowded.md`)

CSS only. Add `.task-recurring` to the short-mode glyph-box selector lists and a
`::before` rule drawing `--bob-date-glyph-repeat` at the default ink, resting from the
row's `data-task`. The existing toggle-off cover already hides stale Tasks-host marks.

### Vault (one line)

In `~/bob/blocked.md`, add `hide recurrence rule` to the "RECURRING Tasks" `tasks` query
(after `group by recurrence`). Its group heading already states the rule, so rows don't
repeat it. Edit only that query. Vault sync commits it.

### Contract updates (bob-cli `docs/date-marks.md`)

- **"Tasks query results":** document the `task-recurring` host, the guard, and short
  mode.
- **Vectors:** add the RT vectors below.
- **Live verification:** add a line for `dash.md` / `blocked.md` recurrence rows.

### Conformance vectors

| #   | Input                                                                          | Expected                                                                        |
| --- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| RT1 | complete li with `span.task-recurring > span` text `" 🔁 every day when done"` | a `daily` mark appended inside the span; span flagged; second pass adds nothing |
| RT2 | `" 🔁 every week on Monday when done"` / `" 🔁 every month on the 1st"`        | `Mondays` / `monthly · 1st`                                                     |
| RT3 | short-mode `span.task-recurring > span` text `" 🔁"`                           | JS leaves it untouched; CSS draws the loop                                      |
| RT4 | open li whose description has `#gtd` + `#pre` pills, rule `" 🔁 every day"`    | untouched (guard)                                                               |
| RT5 | `" 🔁 every day; x"` / an li without `data-task`                               | untouched / retried, then decorated or dropped as for dates                     |
| RT6 | marks off                                                                      | nothing scheduled; a stale Tasks-host repeat mark is hidden by the cover rule   |

### Done when

All gates pass and the plugin is synced. `dash.md` TODAY shows `⟲ daily` on recurring
rows, short-mode notes show the loop glyph, and `blocked.md` RECURRING rows carry no
rule text. Commit bob-plugins (`feat(ledger-tools): …`) and bob-cli
(`docs(date-marks): …`). The vault line rides vault sync.

---

## Out of scope (v1)

The following stay unchanged: the review footer, `]s` notices, pickers, the Task Card,
inbox-route subtitles, CLI/TUI, `bob query`, Bob Mac Capture, Highlights chips, and any
storage or setting change (no `#gtd/pre`, no `🔁` migration, no `removeGlobalFilter`).

If picker prefixes are wanted later, do not use 🏁: Tasks already uses it as its
`onCompletion` symbol.

## Live verification (Bryan, in Obsidian, after the last phase)

- [ ] Sunrise and flag are distinguishable at a glance on red Blocked `[?]` rows, in
      light and dark themes, and on iOS. The sunrise rays are the finest detail.
- [ ] `⟲ daily` reads as secondary beside `+` and `⧗` without going faint, and
      `⟲ Mondays` does not pair visually with `⧗ Mon`.
- [ ] Walking `]s` through PRE keeps the glyph on landing, and 🍅 sits after it on
      Today-linked rows.
- [ ] The cursor or a click reveals all three tags, and typing stays raw.
- [ ] Reading view, embeds, `dash.md` TODAY, `blocked.md` RECURRING, and PDF export look
      right.
- [ ] Both toggles restore the native pills, and completing recurring rows writes
      identical Markdown with marks on and off.
- [ ] A hand-typed `#task #gtd #pre … [repeat:: every day]` shows the orange repair
      pill.
