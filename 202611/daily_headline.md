---
tier: epic
title: 'Daily Headline: one line a day, shown wherever the week looks at its days'
goal: 'Every daily note can carry one Bryan-written `**Headline:**` line naming the
  result that would make the day count. bob reads it (and never writes it) and serves
  it to the week note''s Days table and strip, the weekly brief, the terminal, and
  the top of the Bob Mac Capture idle agenda. A concise POST checklist row in gtd_daily.md
  prompts it each morning.

  '
decisions:
  morning_clause:
    ask: Drop the Morning review row's stale "highlight chosen (≤3 themes, highlight
      first)" clause?
    choices:
      drop: Delete the clause; the new Headline row is the one place the day's pick
        is made
      keep: Leave the Morning review row byte-for-byte unchanged, retired theme cap
        included
    default: drop
    why: The ≤3-theme cap it cites is retired, and two rows asking for the day's pick
      would compete
    answer: drop
  stage_headline_row:
    ask: Give the Headline row its own `headline` POST stage in the chezmoi-managed
      bob config?
    choices:
      stage: Add `headline` to post_stages; the row walks after Monday's weekly review,
        before Morning review
      file_order: Leave config alone and rely only on the row sitting above Morning
        review in gtd_daily.md
    default: stage
    why: A stage keeps the walk order wherever recurrence or regrouping moves the
      row
    answer: stage
  memory_glossary_headline:
    ask: Add a "Daily Headline" glossary strand (it also renders into the docs site
      glossary)?
    memory:
    - glossary:daily-headline
    default: false
    answer: true
  memory_decision_record:
    ask: Record a decision strand "The Daily Headline is one Bryan-owned line, read
      but never written by bob"?
    memory:
    - decisions:daily-headline-is-bryans-line
    default: false
    answer: false
phases:
- id: cli
  title: bob-cli reads the Daily Headline and serves it
  depends_on: []
  size: medium
  description: 'cli: add the shared daily-headline parser, put `headline` on week-report
    day columns and children, day brief parts, and `capture-pomodoros --tasks`, update
    terminal output, docs, and tests.'
- id: plugins
  title: Week note Days table and strip show each day's headline
  depends_on:
  - cli
  size: small
  description: 'plugins: bob-ledger-tools reads the day-row `headline`, renders it
    in the Days table Summary (future days too), adds it to week-strip day tooltips,
    then builds, tests, and syncs.'
- id: mac
  title: Headline row at the top of the Mac idle agenda
  depends_on:
  - cli
  size: medium
  description: 'mac: decode the snapshot `headline`, add a fixed headline row under
    the Today title, measure, plan, and draw it, cover it with tests and a design
    render, update docs, and gate on macOS CI.'
- id: vault
  title: Template slot, morning checklist row, and POST stage
  depends_on:
  - cli
  size: small
  description: 'vault: add the empty `**Headline:**` slot to the daily template, add
    the concise staged POST row to gtd_daily.md, apply the morning-clause and staging
    decisions, and verify end to end.'
proposed_by: bbugyi200.athena.1dq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 17:21:16
status: done
bead_id: bob-cli-mw
---

- **PROMPT:** [prompts/202611/daily_headline.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/daily_headline.md)
- **BEAD:** [bob-cli-mw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mw/README.md)

# Plan: Daily Headline

## Design

### The concept

A **Daily Headline** is one sentence Bryan writes on a daily note each morning: the one
result that would make today count. It is the day's leaf in the period tree, the place
where the week's Intentions turn into one result for today. It is a plan, not a score.

The name is the existing period-note word. A week, quarter, or year Review already opens
with a `**Headline:**` line, and a parent note's parts table shows each child's
Headline. The day gets the same thing: **a note's Headline is the one line its parent
note shows for it.** Weeks get theirs from the agent-drafted Review at the end of the
week. Days get theirs from Bryan at the start of the day. The weekly note's `## Days`
table then reads like the quarter's `## Weeks` table: one line per child. A filled week
tells its story in seven sentences, right below that week's Intentions.

The research report (`research:202611/daily_aim_design/daily_aim_design__final.md`)
called this the "Aim". This plan keeps that report's main constraints (one line,
user-owned, empty is fine, no task-state effects, no carry-forward, no daily agent
Review). It renames the concept to Headline and drops its result marks, the `· I#`
Intention suffix, and its zero-code trial phase. Marks and the suffix add a second
ritual step and grammar that every display would have to parse. The planned follow-up,
an agent-written day **Summary** under the Headline (bob task `bob#^daily-summary`), is
the right home for whatever actually happened that day.

### The note format

The daily template gains one line between the nav line and the `^gtd` row:

````markdown
# 2026-11-18 Wed

[[2026/20261117|prev]] · [[2026/2026-W47|W47]] · [[2026/20261119|next]]

**Headline:** Converge take-home submitted and the zettel album's first draft published

- [ ] #task [[gtd_daily]] [root:: gtd] [created::2026-11-18] ^gtd

```bob-plan

```
````

The note reads like a newspaper front page: dateline, then the headline. In Live Preview
the empty slot shows as a bold `Headline:` waiting to be filled. The label is the same
token the period Reviews use, so one word means one thing in every note.

Parse rules (one shared bob-cli function, used by every surface):

1. Look only at the **preamble**: the lines after YAML frontmatter and before the first
   `## ` heading. Skip fenced code blocks. A `**Headline:**` line pasted under
   `## Pomodoros` or `## Events` never counts.
2. A line whose trimmed form starts with `**Headline:**` or `**Headline**:` opens the
   Headline. Match the label ASCII case-insensitively.
3. The Headline is that line's remainder plus any soft-wrapped continuation lines,
   joined with single spaces. The paragraph ends at a blank line, a list item, a
   heading, a fence, or another bold `**Label:**`. This is exactly the period Review
   Headline rule, so a future `**Summary:**` line directly under the Headline stays
   separate.
4. The first non-empty Headline wins. An empty slot means "no headline", never an error.
   Extra Headline lines are ignored, which enforces the soft cap of one.
5. `text` is the joined line verbatim (Markdown kept). `display` is what screens show:
   - `[[target|alias]]` becomes the alias;
   - `[[target]]` becomes its last path segment without `#…`;
   - `[label](url)` becomes the label;
   - `**`, `__`, `~~`, `==`, and backticks are removed, and whitespace is collapsed.

### Policies

- **Bryan alone writes it.** bob, plugins, the Mac app, and agents only read it. No bob
  command creates, fills, or edits it. The one exception is a vault-wide link repair
  such as `bob task archive`, which updates a link inside it like any other link.
- **No task-state effects.** Writing, changing, or deleting a Headline never changes a
  task's lane, Today membership, freshness, priority, or schedule
  (`decisions:today-is-read-from-the-ledger`). It is not a task row and creates no Task
  Mentions.
- **Empty is valid** (rest, travel, or a day eaten by obligations). An absent Headline
  is "none", never a failure. Old notes are not backfilled.
- **No carry-forward, no score.** Tomorrow's slot starts empty. No surface shows
  streaks, hit rates, or ✓/✗ marks for Headlines.
- **No caps enforced.** Guidance (docs and the checklist row): one sentence, a state
  that's true by tonight, short enough to read at a glance. Clients clamp it to two
  lines visually; bob never truncates it.

### Where it shows

| Surface                                           | What changes                                                                                                                                                                    |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Daily note                                        | The `**Headline:**` slot under the nav line (template)                                                                                                                          |
| Week note `## Days` (`bob-period-parts`)          | The Summary column shows the day's headline in normal text, replacing the muted "mostly" shares. A future day that already has one (planned the night before) shows it faintly. |
| Week note strip (`bob-stats-strip`)               | Each day column's hover tooltip carries the headline under its Worked line                                                                                                      |
| `bob periods show` (terminal + JSON)              | Day columns and children carry `headline`                                                                                                                                       |
| `bob periods brief` (weekly agent bundle)         | Day parts carry Bryan's headline, labelled as his                                                                                                                               |
| Bob Mac Capture idle agenda                       | A headline row directly under the `Today · Tue 17 Nov` title                                                                                                                    |
| `bob capture-pomodoros --tasks` (terminal + JSON) | Top-level `headline`                                                                                                                                                            |

Mac idle agenda, top rows (sketch):

```text
Today · Wed 18 Nov                                        9 done · 3h 30m
Converge take-home submitted and the zettel
album's first draft published
TODAY 9/13
▶ BUILD                                                    15:45–16:10
```

Week note `## Days` (sketch):

```text
Child    Dates       Worked   Done  Review  Summary
Mon 16   2026-11-16  4h 10m     14  ✓       Capture merge-in arrow shipped
Tue 17   2026-11-17  3h 30m     12  ◐       Daily headline designed and planned
Wed 18   2026-11-18                         Converge take-home submitted   (faint)
Thu 19   2026-11-19
```

### The morning ritual

One new recurring row in `~/bob/gtd_daily.md`, short enough to read in one glance:

```markdown
- [?] #task #gtd #post #headline **Write today's headline**: fill in today's
  `**Headline:**` line with the one result that would make today count. [repeat:: every
  day when done] [created:: <date>] [scheduled:: <date>]
```

It is a POST row, so it walks after the lane walk, once Bryan has seen the day's ledger,
Next, and calendar. Choosing a headline is a post-walk decision, and PRE is for intake.
With its own stage it walks after Monday's weekly review (which sets the Intentions it
should serve) and before the Morning review row, which closes the walk.

> [!decision] stage_headline_row = stage Add `headline` to `freshness.post_stages`
> (`[periods, headline]`) in the chezmoi-managed config, and also place the row directly
> above the Morning review row. Stage order survives recurrence and regrouping. The file
> position covers any host whose config has not updated yet.

> [!decision] stage_headline_row = file_order Leave the config untouched. Place the row
> directly above the Morning review row so unstaged POST file order (path ↑, line ↑)
> walks it before the Morning review.

> [!decision] morning_clause = drop Remove
> ` · highlight chosen (≤3 themes, highlight first)` from the open Morning review
> occurrence, so the Headline row is the single place the day's pick is made. The
> ≤3-theme cap the clause cites was retired from the code.

> [!decision] morning_clause = keep Do not touch the Morning review row.

### Rollout and compatibility

- Every contract change is additive: JSON `schema_version`s stay put, there is no `!`
  commit marker, and the release train ships bob-cli (never edit `Cargo.toml`'s version
  or `CHANGELOG.md`).
- Older-bob behavior stays graceful. The plugin falls back to "mostly" when a day row
  has no `headline`, and the Mac app draws no headline row when the snapshot lacks one.
  Clients upgrade independently.
- The Mac app follows `decisions:mac-capture-is-a-thin-client` and
  `decisions:idle-capture-shows-ledger-agenda`: bob owns the fact (including the display
  text), and the app only decodes, measures, and draws it.

## Phase cli — bob-cli reads the Daily Headline and serves it

Work in bob-cli.

1. **Shared parser.** Add `src/native/daily_headline.rs` (register it in
   `src/native/mod.rs`) with `DailyHeadline { text, display }` and
   `parse(contents) -> Option<DailyHeadline>`, implementing the parse rules above. Share
   code, don't copy it:
   - Reuse the paragraph-ending rule from `src/native/periods/note.rs`
     (`ends_headline_paragraph`): expose it `pub(crate)` or move it here and have
     `note.rs` call it.
   - Move `wikilink_display` / `flatten_wikilinks` out of `src/native/periods/brief.rs`
     into this module, have the brief use them from here, and build `display` from them,
     adding `[label](url)` → label and the emphasis/code-marker strip.

   Unit tests cover:
   - both bold forms and label case;
   - an empty slot;
   - soft-wrap joining, and stopping at a following `**Summary:**` line;
   - the first non-empty Headline winning;
   - a `**Headline:**` inside frontmatter, a fence, or under `## Pomodoros` being
     ignored;
   - CRLF input;
   - display flattening of each link and marker kind.

2. **`bob periods show`** (`src/native/periods/report.rs`): for week reports, every day
   entry in `columns[]` and `children[]` gains `headline`: the display string, or
   `null`. This covers past, current, and future days; read a future day's note only
   when it exists. Week/quarter/year rows keep their Headline under `review.headline`,
   unchanged. Quarter, year, and all-time reports read no daily notes for this. Terminal
   output:
   - a past or current day row's summary prefers the day headline over `mostly_display`;
   - a future day row appends its headline when present.
3. **`bob periods brief`** (`src/native/periods/brief.rs`):
   - JSON day parts keep a `headline` string (`""` when none). They still drop
     `review_state` and `reflection_lead`.
   - Markdown day rows (including "not started" future rows) append
     `Bryan's headline: “<display>”` when present, so the weekly agent reads it as
     Bryan's own plan, not as evidence of a result.
   - No `BRIEF_VERSION` or callout `INSTRUCTIONS_VERSION` bump: the field is additive
     data, and refreshing callouts would rewrite every unreflected period note.
4. **`bob capture-pomodoros --tasks`** (`src/native/capture_pomodoros.rs`):
   - The top-level `headline` is the display string, or `null`. It is `null` when
     today's note lacks one or does not exist; it is omitted entirely without `--tasks`.
     Follow the existing `Option<Option<_>>` + `skip_serializing_if` pattern.
     `schema_version` stays `1`.
   - Human `--tasks` output prints the headline as one bold line under the header when
     present.
   - Update the command's `long_about`.
5. **Docs** (read the `docs.md` reference memory first; run `just docs-gen` for the
   changed help, then `just docs-check`):
   - `docs/reference/vault-format.md` "Day notes and the Pomodoro ledger": add the slot
     to the example and a "Daily Headline" paragraph covering ownership, parse rules,
     the never-written/no-task-state policy, and where it is shown.
   - `docs/periods.md`:
     - generalize the Headline definition ("the one line a parent shows for its child";
       days get theirs from the daily note);
     - list the day `headline` in the Period reports `columns`/`children` paragraph;
     - fix brief section 4 ("JSON day parts omit … `headline` …" is no longer true).
       Days still have no Review.
   - `docs/capture.md` (`capture-pomodoros --tasks`) and
     `docs/reference/json-outputs.md` (the `capture-pomodoros` row, and the
     `periods show` row if it lists child fields): name the additive `headline` field.
6. **Tests**: CLI integration tests (`tests/cli/periods.rs` and the capture-pomodoros
   tests/goldens):
   - a week fixture with headlines on a past day, today, and a future day's note;
   - the brief day part, in JSON and Markdown;
   - `capture-pomodoros --tasks` with a headline, with an empty slot (`null`), with a
     missing note (`null`), and without `--tasks` (key absent).

   Update existing goldens only where the new key legitimately appears. Run `just all`
   (fmt, lint, test).

> [!decision] memory_glossary_headline If accepted, use `/sase_memory_write` to add the
> glossary strand **Daily Headline** (alias "day headline"). Define the line, its parse
> location, and its ownership; list where it shows; and say what it is not: a period
> Review's agent-drafted Headline, a Pomodoro entry line, or the ★ highlight. Link
> `[[period-note]]` and `[[day-strip]]`. Then run `sase memory init` and
> `just docs-gen`, since the docs glossary is generated from this web.

> [!decision] memory_decision_record If accepted, use `/sase_memory_write` to add a
> decisions strand. It records:
>
> - the claim: one Bryan-owned line, read but never written by bob, with no task-state
>   effects, empty valid, and no marks or carry-forward;
> - why it beat the research's Aim, marks, and `· I#` suffix, and the trial-first order;
> - its evidence: this plan and the research report.
>
> Then run `sase memory init`.

## Phase plugins — Week note Days table and strip show each day's headline

Work in bob-plugins (`plugins/bob-ledger-tools/src/` fragments, never the generated
`main.js`).

1. `239-plugin-period-strip-model.js` `periodStripTableRows`: set `headline` to
   `row.review.headline` when present, else the row's own `headline` string, else `""`.
   Week, quarter, and year rows behave exactly as before.
2. `239-plugin-period-strip-parts.js` `paintPeriodPartsTable`:
   - A future day row with a headline renders empty Worked/Done/Review cells plus the
     Summary cell, so the grid stays aligned. It inherits the future row's faint color.
   - Future rows without a headline still stop after Child and Dates.
   - Every headline Summary cell carries a `title` with the full text.
3. `styles.css`: `.bob-period-parts-summary-headline` wraps to at most two lines
   (`white-space: normal`, two-line clamp) instead of the single-line ellipsis, and
   keeps normal text color. The muted "mostly" style is unchanged.
4. Week strip: `periodStripColumnTooltip` adds the column's `headline` as the second
   tooltip line, under the Worked line. The fence's change signature
   (`239-plugin-period-strip-fence.js`) includes day headlines, so an edited headline
   repaints both the table and the strip.
5. Tests:
   - `scripts/test-ledger-tools-period-parts.cjs` and its fixture builder: a day row
     with `headline` renders `bob-period-parts-summary-headline`. The current assertion
     that week tables never do changes to "only days with a headline".
   - A day without one still shows "mostly".
   - A future day with one renders a faint Summary.
   - The strip tooltip includes the headline.
6. Update `docs/bob-ledger-tools.md` (parts table and strip tooltip). Run
   `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync` as the
   repo's `CLAUDE.md` requires. Its refusal rules apply; never pass `-F` on your own.

## Phase mac — Headline row at the top of the Mac idle agenda

Work in bob-mac-capture (read its `AGENTS.md`; never land Swift you have not compiled).

1. **Model** (`Sources/CaptureCore/CaptureAgendaModels.swift`):
   `CaptureAgendaSnapshot.headline: String?`, decoded with `decodeIfPresent` from
   `headline`. Trimmed; blank becomes nil. Absent (older bob) and `null` both mean "no
   headline".
2. **Presentation** (`CaptureAgendaPresentation.swift`): new
   `CaptureAgendaRowKind.headline` and `headlineRow: CaptureAgendaRow?`.
   - Build it only for a same-day snapshot with a headline. It stays nil while the
     agenda reads "Loading today…", so yesterday's headline never shows.
   - It stays on a stale-but-same-day snapshot, like the rest of the cached agenda.
   - Show it whether or not Pomodoros exist (`.noOpen` included).
   - The text is bob's display string verbatim. The app never parses Markdown here.
3. **Fit planner** (`CaptureAgendaFitPlanner.swift`): the headline is a fixed row,
   ordered title → headline → plan budget → warning → state. It never folds and is
   counted in every total. Add it wherever the fixed-row kinds are listed.
4. **Measure and draw** (`CaptureAgendaRowMeasurer.swift`, `CaptureAgendaView.swift`).
   The look is newspaper hierarchy, with no chrome, icon, or card:
   - the small secondary `Today · Wed 18 Nov` dateline, then the headline one type step
     above the task-row text, semibold, in primary color;
   - leading-aligned with the title, at most 2 lines, tail truncation;
   - the same 4 pt bottom gap the title row takes.

   Constants live in `CaptureAgendaLayoutMetrics`, so the measurer and the view agree.
   Add `.help(<full text>)` and the VoiceOver label "Today's headline, <text>". Return
   in an empty editor never acts on it, like every agenda row.

5. **Tests** (`CaptureAgendaModelsTests`, `CaptureAgendaPresentationTests`,
   `CaptureAgendaFitPlannerTests`, `CaptureAgendaHeightConsistencyTests`, and the design
   tests):
   - decode: present, absent, `null`, blank;
   - presentation: today vs another day vs stale; `.noOpen` with a headline;
   - fixed-row heights and planner totals;
   - a new fixture, `Tests/Fixtures/agenda-headline.json` (copy `agenda-current.json`
     and add a two-line-long headline), rendered by the design tests in light and dark.
6. **Docs**: in `docs/idle-agenda.md`, a paragraph on the headline row: what bob
   supplies, older-bob behavior, and the two-line clamp. In `docs/protocol.md`, add the
   additive `headline` on `capture-pomodoros --tasks` to the capability/degradation
   tables. If the headline render is published, point `Scripts/docs_media.py` at it and
   refresh the image from the CI artifact. Run `just docs-check`.
7. **Gate**: `just portable-check` (on a Swift host if this one has none) and the
   best-effort Mac scratch typecheck that `AGENTS.md` describes. Run
   `Scripts/macos-ci-check` before landing. After the push, watch the exact-SHA macOS
   run to green with `/sase_monitor`. If the Mac is offline, say the macOS targets are
   unverified.

## Phase vault — Template slot, morning checklist row, and POST stage

Edit Bryan's vault (`~/bob/`) by its `AGENTS.md`:

- inspect `git status` first;
- touch only these files;
- commit with `git add <files> && git commit`, never `/sase_git_commit`.

1. **Template**: in `~/bob/_templates/daily.md`, insert a blank line and `**Headline:**`
   between the nav line and the `- [*] #task [[gtd_daily]] … ^gtd` row, matching the
   format section above. Leave Templater syntax and everything else alone. Do not
   backfill existing daily notes.
2. **Checklist row**: in `~/bob/gtd_daily.md`, add the row from "The morning ritual"
   directly above the open Morning review occurrence.
   - Mirror that occurrence's status and its `[repeat:: every day when done]`,
     `[created::]`, and `[scheduled::]` shape: `[?]` while scheduled in the future, and
     the same scheduled date as the open Morning review row.
   - Keep the row text exactly as written there: concise, imperative, naming the exact
     label to fill.
3. Apply `morning_clause` (see the callouts).
4. Apply `stage_headline_row`. For `stage`:
   - open the `chezmoi` linked repo with `/sase_repo`;
   - in the source for `~/.config/bob/config.yml`, change `freshness.post_stages` from
     `[periods]` to `[periods, headline]`, keeping the trailing comment true;
   - commit and push there, then update this host's applied config and confirm the live
     file reads `post_stages: [periods, headline]`;
   - note in the phase report that apollo and the Mac pick it up on their next chezmoi
     update. Until then, the row's file position keeps the order.
5. **Verify end to end** with the `cli` phase's bob build:
   - `BOB_NOW=<scheduled date>T07:00 bob freshness list -f json` shows the new row in
     POST: `stage: "headline"` for `stage`, ordered after `#periods` rows and before the
     Morning review.
   - Copy a scratch vault containing a daily note rendered from the new template shape
     with a filled Headline. Run `bob periods show <week> -f json` and
     `BOB_DAY_FILE=<that note> bob capture-pomodoros --tasks -f json` against it, and
     confirm both carry the headline. Never write a headline into Bryan's real daily
     notes.

## Out of scope

- Writing the Headline from the Mac panel, capture grammar, or a bob write command.
  Bryan types it in Obsidian, where the morning walk already happens. Revisit only if
  the row proves clumsy.
- ✓/◐/✗ marks, `· I#` Intention suffixes, lints, caps, streaks, or any score.
- The agent-written day Summary (`bob#^daily-summary`). That follow-up builds on this
  parser: a `**Summary:**` line directly under the Headline already parses as a separate
  part.
- Weekly callout v3 and `bob periods ensure -R`. The brief's attribution is enough for
  now; revisit with the Summary follow-up.
