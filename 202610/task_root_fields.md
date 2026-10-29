---
tier: epic
title: Task root fields replace the GTD review buckets
goal: 'A task can name its root parent with `[root:: NOTE]`, rendered in Obsidian
  as a calm hue-dot place mark. Every daily `^gtd` task carries `[root:: gtd_daily]`,
  so morning review time is attributed through Task Links exactly like all other work.
  The `@daily` "GTD" bucket, the `@review:<NAME>` "PLAN review" buckets, review sessions,
  and the "daily review" theme step are gone from Rust, the Obsidian mirror, docs,
  and memory.

  '
decisions:
  review_time:
    ask: What should the built-in review_time Stats metric measure once review sessions
      are gone?
    choices:
      gtd_daily: 'Minutes attributed to gtd_daily tasks (in: [gtd_daily]); the dash
        keeps its line'
      plan_theme: 'Minutes in PLAN-themed sessions (themes: [PLAN]); repeats the theme
        breakdown'
      retire: Drop the built-in; the dash's Review time line says it is not in the
        report
    default: gtd_daily
    why: It measures the morning review itself, through the same Task Links as all
      work
    answer: gtd_daily
  backfill:
    ask: 'Backfill [root:: gtd_daily] into every existing daily note''s ^gtd task,
      not just the template?'
    default: true
    why: Without it, past review time turns unfiled once the GTD bucket is retired
    answer: true
  theme_default:
    ask: Replace the "daily review" theme suggestion with a theme PLAN default on
      gtd_daily.md?
    default: true
    why: The root field already routes ^gtd to gtd_daily, so a note default removes
      one more special case
    answer: true
  root_parent_glossary:
    ask: Rewrite the Root Parent glossary strand for root fields and the retired GTD
      bucket?
    memory:
    - glossary:root-parent
    default: true
    corrects: Review buckets and tasks defined in canonical daily notes group under
      the `GTD` bucket (key `@daily`)
    answer: true
  decision_record:
    ask: Add a decision record saying session names never choose a home and root fields
      re-home tasks?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: root-field
  title: Rust root field contract and attribution
  depends_on: []
  size: medium
  description: 'root-field: add the [root:: NOTE] parser, per-block root facts, as-if-residence
    home resolution, task-metric and theme-default support, health lints, shared vectors,
    and the R11 contract, all additive.'
- id: root-mark
  title: Obsidian root mark
  depends_on:
  - root-field
  size: medium
  description: 'root-mark: render each root field as a hue-dot place mark in Live
    Preview, Reading view, and Tasks/Dataview rows in bob-ledger-tools, with compact,
    rolls-up, and repair states, a toggle, tests, docs, and a plugins sync.'
- id: vault-backfill
  title: Vault template and ^gtd backfill
  depends_on:
  - root-field
  size: small
  description: 'vault-backfill: add [root:: gtd_daily] to the daily template and,
    per the backfill decision, every existing ^gtd line; optionally add theme PLAN
    to gtd_daily.md; commit under the sync lock and verify.'
- id: retire-review
  title: Retire review buckets in Rust, docs, and memory
  depends_on:
  - root-field
  - vault-backfill
  size: medium
  description: 'retire-review: delete @daily, @review buckets, review sessions, the
    review metric filter, and the daily-review theme step; redefine review_time; update
    vectors, fixtures, docs, and the memory decisions; ship as a breaking JSON change.'
- id: plugin-mirror
  title: Obsidian homes mirror and Stats cleanup
  depends_on:
  - root-mark
  - retire-review
  size: medium
  description: 'plugin-mirror: bring the JS homes mirror and theme suggester to the
    new contract, drop @daily, @review, and short_label handling from Stats and the
    Day strip, sync shared vectors, add root-mark parity, and deploy.'
proposed_by: bbugyi200.athena.15o
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 22:26:40
status: wip
bead_id: bob-cli-e2
---

- **PROMPT:** [prompts/202610/task_root_fields.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_root_fields.md)
- **BEAD:** [bob-cli-e2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e2/README.md)

# Plan: Task root fields replace the GTD review buckets

## Why

Pomodoro time is attributed through Task Links to the note a task lives in
(`docs/pomodoro-homes.md`, R1/R2). The daily note's `^gtd` task has no real home, so Bob
grew a pile of special cases to give morning-review time somewhere to go:

- `@daily`, a synthetic bucket labeled `GTD`, for any task in a canonical daily note;
- `@review:<NAME>` buckets labeled `<NAME> review` (with a `short_label`), reached by a
  `^gtd` link;
- review sessions (`kind: "review"`): named sessions whose every component is in
  `plan.exempt` (PLAN) swallow their links, plus an unnamed "review bridge";
- review buckets rolling up to root `@daily`;
- theme suggestion step 3, "daily review" (`^gtd` suggests the first exempt name);
- a Stats `review:` metric filter and the `review_time` built-in that depend on them.

Today's `bob stats show worked -r 7d` reads `root parent GTD 8h 15m …` and
`project … PLAN review 8h 15m`. After this epic, the same time reads `gtd_daily 8h 15m`,
in gtd's orange, because the `^gtd` task says `[root:: gtd_daily]` and is attributed
like any other task.

Vault facts as of 2026-10-28: 1,173 canonical daily notes, 133 with a `^gtd` line (all
shaped `- [.] #task [[gtd_daily]] … ^gtd`, none with a root field), and 86 same-day
`[[#^gtd]]` Pomodoro links. 102 PLAN sessions: 59 link only `^gtd`, 25 link `^gtd` plus
other tasks, and 18 link only other tasks. No PLAN session lacks links, and no unnamed
session uses the review bridge. `gtd_daily.md` is an area under the `gtd` area
(`color: orange`). Nothing in the vault sets a `theme:` note default yet.

## Design

### The root field (contract owned by bob-cli, `docs/pomodoro-homes.md` R11)

**Syntax.** `[root:: NOTE]` or `(root:: NOTE)` on a task line. Recognition is the shared
`task_fields::inline_fields(line, "root")`: the key is case-sensitive and any spacing
around `::` is allowed. The canonical written form is `[root:: gtd_daily]`: one space
after `::` and a bare note name.

**Value.** A note name or a vault-relative path, with or without `.md`. It resolves
relative to the task's own note using Bob's link rules (`HomeSource::resolve`: an exact
vault path, then a unique case-insensitive basename).

**States**, mirroring the PT13 theme field:

- _none_: no field.
- _set_: exactly one field whose value resolves to one note X.
- _repair_: the field is ignored and the task attributes from its own note exactly as if
  it had no field. Bob reports the repair. Reasons:
  - `duplicate`: two or more root fields.
  - `invalid`: blank, or contains any of `[ ] | # ^`. A wikilink value is invalid, and
    the hint says to write the bare name.
  - `missing`: no note matches.
  - `ambiguous`: two or more basename matches.
  - `no_home`: X's walk reaches no area or project.
- Closed tasks keep their field, and it keeps applying, so history stays attributed.
- Root fields never chain. Only the linked task's own field counts; fields on tasks
  inside X are irrelevant. A field naming the task's own note is a silent no-op.

**Meaning: as-if residence.** With a set field naming X, Bob resolves the task's homes
exactly as if the checkbox lived in X:

- The R1 chain starts at X: X, then X's scalar `parent:` walk, with the same 16-note and
  cycle limits and the same archive/ref walk-through.
- R2 picks the root from that chain.
- The task's own note contributes nothing.

So an area at any depth, a `root: true` project, or a top-level project names itself as
the root parent. When the chain from X yields a root R ≠ X (for example X is a
sub-project without `root: true`, or a ref note), the time still follows X's chain and
groups under R. Bob reports `root_field_not_root` naming R, with the fix: "sase_goals
groups under sase; write [root:: sase] or set root: true on sase_goals". This keeps the
set of root parents a function of note frontmatter: `root: true` stays the one way to
create a root, and a per-task field can never invent a stray root.

**Scope: reporting only, like `root: true`.** The field applies to:

- Pomodoro homes on every homes surface: `bob plan`, capture outputs,
  `capture-pomodoros --tasks`, the Obsidian ledger chips, and `bob stats` sessions,
  parts, and catalog.
- Stats task metrics (done/created/dropped project, area, subarea, and `root_parent`
  parts).
- Stats `in:` / `exclude_in:` matching (against X's chain, not the task's own note).
- The PT13 note-default theme walk, which starts at X.

The Stats `note` part stays the file that holds the checkbox. The field never changes
residence, blocking (`#prj_is_blocked`), Ready caps, `task_refresh` freshness intervals,
review tiers and the `]s` walk, capture routing, task move, archive, `## Tasks`
grouping, `bob projects`, or the plan budget.

**Placement and preservation.** The canonical position is the left edge of the trailing
metadata run, the same position `task_theme::set_task_theme` uses. A field authored
anywhere on the line is still recognized. Root is a description-region field: it is
never part of the Obsidian Tasks suffix, freshness stamps land after it, and it must not
be added to the fresh/refresh/keeps/unblocked run-extending keys in
`freshness/placement.rs`. The following keep it byte-identical:

- task move and archive;
- freshness stamping, keeps, and unblock;
- theme set/clear;
- reconcile, complete, and cancel;
- ref rename's link rewrite.

No Bob writer authors the field in this epic; only the vault template and backfill do.

**Lints.** `bob stats` reports two additive health lists, once per (path, line), for
root fields it reads while attributing in-window sessions and task rows. The human
output has matching `warning <message>  <code>` lines.

- `health.homes.root_field_invalid: [{path, line, raw, reason}]`, reason from the repair
  list above.
- `health.homes.root_field_not_root: [{path, line, raw, root}]`.

The unfiled repair hint becomes "file the task in an area or project, or give it a root
field".

### What gets retired (phase retire-review)

- **Session names never choose a home.** A PLAN session linking `[[#^gtd]]` attributes
  to gtd_daily. A PLAN session linking `[[sase_blog#^s0-spike]]` attributes to
  sase_blog. An unlinked closed PLAN session is `@unlinked`. "How much PLAN" remains the
  Stats theme dimension, which already reads `PLAN 8h 15m`. R4 becomes "Session names
  never choose a home", keeping the R-numbers stable.
- **Only two buckets remain:** `@unlinked` and `@unfiled`. A canonical daily note is
  walked like any other note. Its `parent:` chain reaches no area or project, so a
  daily-note task without a root field is `@unfiled` and counts as an unfiled link,
  which surfaces strays instead of hiding them.
- **JSON changes (breaking).** These disappear: `homes.kind: "review"`; parts and
  catalog keys `@daily` and `@review:*`; labels `GTD` and `<NAME> review`;
  `short_label`; the `review:` metric key; theme suggestion `source: "review"` /
  `label: "daily review"`. `plan.exempt` keeps every budget role: exempt entries skip
  `max_themes` and `max_links`, exempt destinations write no `[theme::]`, and exempt
  hues stay muted.

### Obsidian look

The theme pill rule in `styles.css` reads "names wear a tint, places wear a dot". A root
is a place, so the root mark is the same visual as a ledger home chip:

- a filled `●` in the root's R8 area hue;
- the lowercase note stem in muted text, sized like the other task-line field marks.

On a Pomodoro line, the chip for a PLAN session that links `^gtd` reads `● gtd_daily`.
On the `^gtd` task, the root mark shows that same dot. The two ends of the attribution
rhyme visually.

The `^gtd` description is already `[[gtd_daily]]`. Writing the stem again would stutter,
so the mark collapses to its dot when the task text already links the same note. The
result is `gtd_daily ●  ✓ today …`. A task elsewhere reads
`Fix flaky sync ● sase  FIX  ✓ today`.

## Phase root-field: Rust root field contract and attribution

All in bob-cli. This phase is additive: review buckets keep working until retire-review,
but a set root field wins.

1. **Parser.** Add `src/native/task_root.rs`, modeled on `task_theme.rs`:
   - `parse_task_root(line) -> TaskRootState { None, Set { raw }, Repair { kind } }`
     with `Duplicate` / `Invalid`. This is syntax only; resolution happens in the
     resolver.
   - `set_task_root` / `clear_task_root` helpers using the theme placement rule, with
     unit tests (needed for preservation tests and future writers).
   - Register the module in `src/native.rs`.
2. **Per-block facts.** `NoteFacts::parse` (`pomodoro_homes/facts.rs`) already receives
   full note text. Collect `block id → TaskRootState` for task lines that carry a
   trailing `^id` and contain `root::`. Use the cheap substring prefilter, skip
   frontmatter and fenced code, and match block ids the way `note_tasks`' `by_block_id`
   does. `HomeSource` does not change: the indexed, disk, and capture sources all flow
   through `NoteFacts`.
3. **Resolver** (`pomodoro_homes/resolve.rs`):
   - In `home_for_link`, after `LinkResolution::Found(path)`, look up the block's root
     state. A set field resolves X relative to `path`:
     - `Found(x)` → `resolve_note(x)`. If the result is `@unfiled`, record `no_home` and
       fall back. Otherwise, if `home.root_parent != path_string(x)`, record `not_root`.
     - Missing / Ambiguous → record the lint and fall back to `resolve_note(path)`.
   - Consult the set field before the existing `^gtd` → review-home and canonical daily
     → `@daily` mappings. In `session.rs`, a same-day `^gtd` link whose task has a set
     root field no longer triggers the unnamed review bridge, because the bridge check
     runs before task homes are computed. Named all-exempt sessions stay review sessions
     until retire-review.
   - Add `home_for_task(path, line)` for callers that hold the task line itself.
   - Add `root_field_invalid` / `root_field_not_root` lists to `ResolverHealth`,
     deduplicated by (path, line).
4. **Stats.**
   - Task metrics call `home_for_task(source_path, line)` instead of
     `home_for_note(source_path)` (`stats/metrics.rs` near the task-home lookup).
   - `in:` / `exclude_in:` already follow `Home.residence_chain`, which is now X's
     chain; add tests proving it.
   - Surface both health lists in `stats/report.rs` (`HomeLintReport`-style rows) and
     `stats/homes.rs` warnings, next to `area_color_invalid`.
5. **Theme default walk.** In `pomodoro_themes/suggest.rs`, a task with a set root field
   walks note defaults from X. This overrides "daily notes have no default walk".
   `from.chain` starts at X. Step 3 stays until retire-review.
6. **Preservation regressions.** Mirror the PT13 Preservation tests for `[root:: X]` /
   `(root:: X)` lines through: freshness stamp, keep, and unblock; theme set/clear; task
   move; archive; reconcile; complete/cancel; ref rename. Freshness must land after the
   root field.
7. **Vectors.**
   - Add a `root-fields` case to `tests/fixtures/pomodoro_homes/vectors.json`. Its
     mini-vault has `gtd.md` (area, color orange), `gtd_daily.md` (area, parent gtd), a
     `sase` project with a `sase_goals` sub-project, and an inbox area.
   - The case covers:
     - a same-day `^gtd` with `[root:: gtd_daily]` in an unnamed session (home
       gtd_daily, root gtd_daily, area gtd, hue orange);
     - an inbox task `[root:: sase]`;
     - the paren form;
     - `[root:: sase_goals]` (groups under sase, with a lint);
     - the `missing`, `duplicate`, `invalid`, and `no_home` fallbacks;
     - a closed task that keeps attributing.
   - Do not mark any of it `rust_only`.
   - Add a `root-field-note-default` case to
     `tests/fixtures/pomodoro_themes/vectors.json`.
   - Use only unnamed or non-exempt named sessions in the homes case, so it means the
     same thing before and after retire-review.
8. **Docs.**
   - `docs/pomodoro-homes.md`: add `## R11 — Task root fields` (the full contract
     above), and add R1/R2 pointers and table rows for `[root:: …]` tasks.
   - `docs/pomodoro-themes.md` PT13: the walk start and the preservation line.
   - `docs/stats.md`: task-metric parts, `in:`, and health lints.
   - `docs/reference/vault-format.md`: document the task field.
   - Run `just docs-gen` and `just docs-check`.
9. **Gate:** `just check`. Commit as an additive `feat(homes): …`.

## Phase root-mark: Obsidian root mark

In the bob-plugins linked repo (open it with `/sase_repo` and read its `AGENTS.md`),
plugin `bob-ledger-tools`. Edit `src/` fragments only, register new fragments in
`src/fragments.json`, keep each fragment ≤1000 lines, and never hand-edit `main.js`.
Follow the shared mark pattern: a pure core fragment (model, `::`-free tooltip,
listener-free `build…Element`, `WidgetType`) plus a plugin mixin (toggle, `Prec.highest`
ViewPlugin, post-processor, Tasks-rows pass). The closest templates are
`279-plugin-task-theme-{model,marks,editor,reading,reading-dom,tasks}.js`. Reuse:

- `pomodoroThemeInlineFields` for parsing;
- `dateMarkContentStart`, `dateMarkFoldLength`, and `dateMarkSelectionFold` for folding
  and reveal;
- `freshnessMarkPosInCode` for code skipping;
- `applyMarkGateClass` / `registerMarkGatePopoutSync` for the toggle;
- the `142-pomodoro-home-chips.js` resolver (`pomodoroHomeResolve` and its hue helper)
  for X's root and hue, so the mark agrees with the homes mirror.

**Anatomy.** `span.bob-root-mark[data-root-state][data-root-hue][data-fold-space]`
containing `span.bob-root-mark-dot` (●) and `span.bob-root-mark-label`. CSS sits next to
`.bob-pomodoro-home-chip` and uses the same `--bob-hue-*` variables, dot size, and muted
label color. Size follows the other task-line marks (0.8em interface font). On hover the
label takes normal text color, and focus uses the theme mark's ring. Provide both light
and dark rules.

**States.**

| State      | When                                                      | Looks like                                             | Tooltip (no `::`)                                                                                  |
| ---------- | --------------------------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `set`      | one field, X resolves, X is its own root                  | `● gtd_daily`, dot in X's area hue                     | `Root parent gtd_daily · gtd area · time on this task counts here · Mod-click to open`             |
| `compact`  | `set`, and the task text before the field already links X | dot only, `aria-label` keeps the full sentence         | same as `set`                                                                                      |
| `rolls-up` | X resolves but groups under root R ≠ X                    | `● sase_goals` with a dotted underline, dot in R's hue | `Root field names sase_goals, which groups under sase · set root: true on sase_goals or name sase` |
| `repair`   | duplicate, invalid, missing, ambiguous, or no home        | hollow `○` plus the raw value, dashed repair outline   | e.g. `Root field ignored · no note named gtd_dialy · time counts under this task's own note`       |

Notes on the states:

- The Obsidian resolver (`metadataCache.getFirstLinkpathDest`) decides missing.
  Ambiguity is Rust-only; Obsidian picks a match, as the homes vectors already allow.
- Gray is used when X has no area.

**Behavior.**

- **Live Preview:** `Decoration.replace` over the field, with per-field reveal when a
  selection touches it.
- **Reading view:** replace Dataview's native
  `.dataview.inline-field[data-dv-norm-key="root"]` node or the raw text without
  rewriting `innerHTML`, so the toggle can restore it.
- **Tasks/Dataview query rows:** a bounded pass like `279-plugin-task-theme-tasks.js`.
- **Left raw:** source mode, fenced code, inline code, and non-task prose.
- Closed tasks keep their mark.
- **Click** puts the caret in the field, like the theme pill. **Mod-click** opens X via
  `app.workspace.openLinkText(x, sourcePath, newLeaf)`.
- **Session command** **Toggle root marks** (default on), gated on
  `body.bob-root-marks`.
- Re-render when the root note's frontmatter changes (`metadataCache` `changed`), so a
  hue or root change shows up without a reload.

**Tests and docs.**

- Add `scripts/test-ledger-tools-root-marks.cjs` to `npm test`. It covers:
  - the state model and the compact rule;
  - tooltip text with no `::`;
  - DOM shape;
  - Live Preview ranges and reveal;
  - reading-view replacement and restore;
  - Tasks rows;
  - the toggle.
- Do not copy the bob-cli vectors yet (the JS homes mirror would fail the new case);
  plugin-mirror adds parity.
- Docs: add a root-mark bullet to `docs/bob-ledger-tools.md` "Marks and rendered
  blocks", and to `docs/develop/plugin-apis.md` if an API is exposed.
- Run `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`, then
  `bob plugins sync`. Commit in bob-plugins.

## Phase vault-backfill: Vault template and ^gtd backfill

Edits `~/bob` directly (not a repo checkout), using a bob built from a checkout that
contains root-field.

1. **Template.** In `~/bob/_templates/daily.md`, the `^gtd` line becomes
   `- [*] #task [[gtd_daily]] [root:: gtd_daily] [created::<% tp.file.creation_date("YYYY-MM-DD") %>] ^gtd`.
   Nothing else in the template changes.

   > [!decision] backfill Also edit every existing canonical daily note
   > `YYYY/YYYYMMDD.md`, today's and any future-dated ones included.

2. **Backfill** (skip the vault-wide part when backfill = no; still edit today's and any
   future-dated notes so the new behavior starts now).
   - Write a one-off script outside the vault and the repo. Run it with `python3 -I`.
   - Target: each note with exactly one task line ending in `^gtd`.
   - Shape check: every target must match
     `^- \[.\] #task \[\[gtd_daily\]\]\s.*\^gtd\s*$` and lack `root::`. Any other shape,
     or a note with several `^gtd` lines, stops the run with a report and no edits.
   - Edit: insert `[root:: gtd_daily] ` immediately after `[[gtd_daily]] `, which is the
     left edge of the trailing metadata run (before `[fresh::` / `[created::`).
   - Verify each edited file differs only by that insertion.
   - Expect about 133 notes. Run a dry run listing the paths first.

   > [!decision] theme_default Add the scalar `theme: PLAN` to `~/bob/gtd_daily.md`
   > frontmatter, so `^gtd` (and the gtd_daily chores) keep suggesting PLAN after
   > retire-review drops the "daily review" step. Skip this when theme_default = no.

3. **Commit safely** (`docs/vault-git-sync.md` recipe).
   - Check `git -C ~/bob status --short` first.
   - Apply and commit while holding the maintenance lock:

     ```bash
     lock="${BOB_VAULT_SYNC_LOCK_FILE:-${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock}"
     flock -w 120 "$lock" <apply + git -C ~/bob add -- <paths> + git -C ~/bob commit -m "vault: add [root:: gtd_daily] to daily ^gtd tasks" -- <paths>>
     ```

   - Then run `bob vault-sync`. `bob vault-sync status --json` must show no new
     conflicts or clobbers.

4. **Verify.**
   - Rescan: every `^gtd` line has exactly one root field.
   - The root-field `bob stats show worked -r 4w -f json` reports empty
     `root_field_invalid` / `root_field_not_root`.
   - When theme_default = yes, `bob themes suggest` for today's `^gtd` shows no
     `theme_default_invalid` warning for gtd_daily.md. Step 3 still answers first until
     retire-review removes it.
   - Report the counts in the phase bead.

## Phase retire-review: Retire review buckets in Rust, docs, and memory

All in bob-cli. Commit subject marked breaking, for example
`feat(homes)!: retire GTD review buckets for task root fields`. Do not edit
`Cargo.toml`'s version or `CHANGELOG.md`.

1. **Homes core.**
   - Delete `DAILY_KEY`, `DAILY_LABEL`, `daily_home`, and `review_home`, plus the `^gtd`
     branch and the `bridge_name` parameter of `home_for_link`.
   - Delete the canonical-daily short-circuit in `resolve_note`; daily notes walk like
     any note.
   - In `session.rs`, delete `review_session`, `named_review`, the bridge,
     `raw_name_components`, and the `exempt` parameter of `attribute_session` /
     `attribute_ledger_entry`.
   - Delete `SessionKind::Review`. `kind` is `work` or `unlinked`.
   - Update the root-parent doc comment on `Home`.
2. **Callers.** Drop the attribution-only `exempt` argument at:
   - `capture_pomodoros.rs`
   - `plan_budget/mod.rs`
   - `capture/existing_task_link.rs`
   - `capture/pomodoro_start.rs`
   - `capture_complete/pomodoros.rs`
   - `pomodoro.rs`
   - `stats/homes.rs` (`HomesEngine`)
   - `stats/metrics.rs` (`default_exempt`)
   - `stats/compute.rs`

   Leave every budget, theme-write, and hue use of `plan.exempt` untouched
   (`plan_budget` theme/link exemption, `capture/task_theme.rs`,
   `capture_complete/pomodoros.rs` preselect, `ref_library/today.rs`,
   `config/themes.rs`, `pomodoro_themes/rename.rs`).

3. **Labels and chips (R7).**
   - Only `unlinked` and `unfiled` are reserved labels.
   - Remove named-review chip suppression and the bridge chip.
   - Remove `short_label` from `HomeCatalogEntry` and `catalog_entry`.
   - Remove the `@daily` catalog insertion, the `bucket_home` `@daily` arm, and the
     `SessionKind::Review` health arm.
4. **Stats metrics.**
   - Remove `ReviewFilter` and the `review:` key. A leftover `review:` is ignored like
     any other unknown key and is listed in migration notes.
   - Remove the `review_kept_session` / `kept_task_indices` review branches; the `in:`
     rule becomes "unlinked sessions never match `in:`…".
   - Remove the `@review:` hue-dot branch in `stats/render.rs`.

   > [!decision] review_time = gtd_daily The built-in becomes
   > `kind: pomodoro, measure: minutes, in: [gtd_daily]`, described as "Minutes
   > attributed to tasks in, or rooted at, gtd_daily, the morning review's area;
   > redefine stats.metrics.review_time if your review lives elsewhere."

   > [!decision] review_time = plan_theme The built-in becomes
   > `kind: pomodoro, measure: minutes, themes: [PLAN]`, described as "Minutes in
   > PLAN-themed sessions".

   > [!decision] review_time = retire Remove the `review_time` built-in and its registry
   > tests. Note the removal in migration notes; custom configs can define it
   > themselves.

5. **Theme suggestion.**

   > [!decision] theme_default Delete PT13 step 3: `SuggestionSource::Review`,
   > `is_daily_review`, the "daily review suggests X" phrase in `capture/output.rs`, and
   > the `suggest_cli.rs` help text. Replace the `daily-review*` theme vectors with
   > root-field note-default cases. When theme_default = no, keep step 3 and its vectors
   > unchanged.

6. **Hint.** The R6 unfiled hint becomes "file the task in an area or project, or give
   it a root field".
7. **Tests and fixtures.**
   - In `tests/fixtures/pomodoro_homes/vectors.json`:
     - Drop every `exempt` key.
     - Rewrite `review-decisions` as `session-names-never-home`, covering: named PLAN
       linking a rooted `^gtd`; PLAN linking a project; an unlinked closed PLAN;
       `PLAN + BOB`.
     - Rewrite `daily-review-link` as `gtd-root-across-days`: another day's rooted
       `^gtd` attributes to gtd_daily; an unrooted one is `@unfiled`.
     - Rewrite `daily-note-task-home`: an unrooted daily task is `@unfiled`.
   - Update `tests/fixtures/stats/homes_vault` and `tests/fixtures/stats/vault`:
     - Add `gtd_daily.md` (area, parent gtd).
     - Root the fixture `^gtd` lines, keeping one unrooted line for the unfiled path.
     - Remove the `plan.md` review anchor.
   - Recalculate H3, H5, H11, H12, H15, H16, and M10 by hand. Update the `docs/stats.md`
     vector tables to match.
   - Update the `pomodoro_homes/tests.rs`,
     `stats/{homes,render,metrics,compute,registry}.rs` tests, `tests/cli/plan.rs` (no
     `kind: review` / `PLAN review`), and any other test that `cargo test` names.
8. **Docs.**
   - `docs/pomodoro-homes.md`:
     - R1: delete the `^gtd` / `@daily` paragraph.
     - R2: replace the table rows with `^gtd` `[root:: gtd_daily]` → `gtd_daily` and an
       unrooted daily-note task → `@unfiled`; buckets become only `unlinked`/`unfiled`.
     - R4: replace with "Session names never choose a home".
     - Also update R6, R7, R9 (`kind`), and the vectors table.
   - `docs/stats.md`: every review mention.
   - `docs/plan.md`: the JSON example.
   - `docs/pomodoro-themes.md`: PT13 per the theme_default decision.
   - `docs/develop/migration-notes.md`: a row for buckets, kinds, `short_label`,
     `review:`, `review_time`, and theme `source: review`.
   - `docs/reference/vault-format.md`: the `^gtd` template carries the root field.
   - `docs/reference/mac-capture/idle-agenda.md`: drop "PLAN review".
9. **Memory.** Use `/sase_memory_write`.

   > [!decision] root_parent_glossary Rewrite the `glossary:root-parent` strand. The
   > root parent is the area note a task lives in, or the note its `[root:: X]` field
   > names (resolved as if the task lived there), else the nearest `root: true` project,
   > else the top-level project. `unlinked` and `unfiled` are the only buckets. It is
   > derived at read time from today's note tree plus task root fields, and Bob never
   > writes it. Keep the existing links.

   > [!decision] decision_record Add a decisions strand, for example
   > `session-names-never-choose-a-home`. Claim: a task's home comes from its residence
   > or its `[root:: X]` field, never from a Pomodoro name; the GTD/PLAN review buckets
   > are retired. Cite this epic's commits and `docs/pomodoro-homes.md` R4/R11. Rejected
   > alternatives: keep named review sessions; literal per-task roots for sub-projects.
   > Reopen if theme-based attribution is wanted again.

   Run `sase memory init`, then `just docs-gen` (the glossary and decision pages are
   generated from memory) and `just docs-check`.

10. **Gate:** `just check`. Then sanity-check against the live vault:
    `cargo run --quiet -- stats show worked -r 7d` lists `gtd_daily` as a root parent
    and shows no `GTD` or `PLAN review`. Put the before/after lines in the phase bead.

## Phase plugin-mirror: Obsidian homes mirror and Stats cleanup

In bob-plugins (`/sase_repo`), using the same fragment rules.

1. **Homes mirror** (`src/142-pomodoro-home-chips.js`, callers 273/274):
   - Implement R11: read each linked task's line by block id, reusing the task-theme
     API's line-text lookup (`279-plugin-task-theme-api.js`). Apply the
     set/repair/rolls-up rules, and re-render chips when a pending read lands.
   - Delete named review, the `^gtd` review bucket, `@daily`, the bridge,
     `pomodoroHomeBucket`'s review use, and attribution-only `exempt` plumbing.
   - `kind` is `work` or `unlinked`.
2. **Theme suggester** (`279-plugin-task-theme-suggest.js`,
   `279-plugin-task-theme-home-facts.js`):
   - Walk note defaults from a set root field's X.
   - When theme_default = yes, delete the "daily review" step and phrases:
     `279-plugin-task-theme-marks.js` reason text, and `block-id-prompt`
     `077-pomodoro-picker-intent.js`.
3. **Stats and Day strip.**
   - Remove the `@review:` hue branch (`239-plugin-stats-panel-models.js`).
   - Remove the `@daily` / `@review` fix copy and add the root-field unfiled hint
     (`239-plugin-stats-homes-render.js`).
   - Remove the `short_label` branch of `dayStripLegendLabel`
     (`239-plugin-day-strip-model.js`) and its time-lanes reuse.
   - Follow the review_time decision in `statsPanelReviewTime`
     (`239-plugin-stats-panel-render.js`). It keeps working for `gtd_daily` and
     `plan_theme`; for `retire`, remove the companion line.
4. **Shared vectors.**
   - Copy bob-cli's `tests/fixtures/pomodoro_homes/vectors.json` and
     `tests/fixtures/pomodoro_themes/vectors.json` byte-identical into
     `scripts/fixtures/…`, and update the fixture README's pinned commit.
   - Update the synthetic Stats fixtures (`synthetic-homes*.json`,
     `synthetic-lanes.json`, `canonical.json`, `day-strip-fixtures.cjs`) to drop
     `@daily`, `@review`, and `short_label`.
   - Add a parity test so root marks agree with the `root-fields` case: state, root, and
     hue per task.
5. **Tests to update:**
   - `test-ledger-tools-pomodoro-home-chips.cjs`, including the reading-view review
     asserts;
   - `test-ledger-tools-day-strip.cjs`;
   - `test-ledger-tools-stats-lanes.cjs`;
   - `test-ledger-tools-stats-panels.cjs`;
   - `test-ledger-tools-pomodoro-themes.cjs`;
   - `test-ledger-tools-task-theme-marks.cjs`;
   - `test-block-id-prompt-pomodoro-picker-view.cjs`.
6. **Docs:** `docs/bob-ledger-tools.md` and `docs/develop/plugin-apis.md` lose review
   wording.
7. **Gates:** `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`,
   then `bob plugins sync`. In Obsidian, today's PLAN sessions now show a `● gtd_daily`
   chip, and the Day strip root lane reads `gtd_daily`.

## Out of scope

- **Bob Mac Capture.** It decodes `homes.kind` and theme `source` tolerantly (unknown
  values become `.other`), so retiring `review` needs no Swift change.
- **Untouched daily-note behavior.** The daily-note `^gtd` exclusions in done, created,
  and dropped counts, freshness, the walk, status hooks, and capture stay as they are.
- **Possible follow-ups:** a Ctrl+Shift+P root picker (fuzzy area/project notes) and
  rename-aware rewriting of bare root values. A renamed root note shows the repair mark
  and a `missing` lint until it is fixed.
- **Vault `plan.md` note.** It anchored the PLAN review bucket. It becomes inert and is
  left for Bryan to keep or delete.
