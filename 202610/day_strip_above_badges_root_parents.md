---
tier: epic
title: Day strip above the badges, with working time grouped by root parent
goal: "On every canonical daily note the Day strip sits above the bob-plan badges, and
  its full-width working-time bar, grouped by root parent (top-level project, or the
  area a task lives in), separates it from them. The stats dash defaults to the same
  root-parent grouping, with sub-project detail one click away. The rollup frontmatter
  key and the `## Pomodoros` heading total are retired. Four new glossary strands name
  the concepts.

  "
decisions:
  heading_cleanup:
    ask: Which daily notes should lose the `## Pomodoros` heading total?
    choices:
      template: Only the daily template; every existing note keeps its total
      strip_notes:
        Template plus the ~27 notes with a bob-plan block, where the Day strip repeats
        it
      all_notes:
        Template plus all ~146 notes with it; notes before 2026-09-29 then show no total
    default: strip_notes
    why: Removes the total exactly where the Day strip already shows it
    answer: strip_notes
  glossary_top_project:
    ask: Add the glossary strand for "top-level project" (alias "top project")?
    memory:
      - glossary:top-level-project
    requested:
      'We should add new glossary memory web strands for: "top-level project" (aka "top
      project"), "root parent", "day strip", "stats dash" (an easy way to refer to the
      ~/bob/dash_stats.md file)'
    default: true
    answer: true
  glossary_root_parent:
    ask: Add the glossary strand for "root parent"?
    memory:
      - glossary:root-parent
    requested:
      'We should add new glossary memory web strands for: "top-level project" (aka "top
      project"), "root parent", "day strip", "stats dash" (an easy way to refer to the
      ~/bob/dash_stats.md file)'
    default: true
    answer: true
  glossary_day_strip:
    ask: Add the glossary strand for "day strip"?
    memory:
      - glossary:day-strip
    requested:
      'We should add new glossary memory web strands for: "top-level project" (aka "top
      project"), "root parent", "day strip", "stats dash" (an easy way to refer to the
      ~/bob/dash_stats.md file)'
    default: true
    answer: true
  glossary_stats_dash:
    ask: Add the glossary strand for "stats dash"?
    memory:
      - glossary:stats-dash
    requested:
      'We should add new glossary memory web strands for: "top-level project" (aka "top
      project"), "root parent", "day strip", "stats dash" (an easy way to refer to the
      ~/bob/dash_stats.md file)'
    default: true
    answer: true
phases:
  - id: retire-rollup
    title: Retire rollup frontmatter in bob-cli
    depends_on: []
    size: medium
    description:
      "retire-rollup: delete rollup parsing, selection, lints, catalog counts, and the
      projects-list marker from bob-cli; rename the pre-rollup chain to the residence
      chain; update vectors, H-vectors, and docs."
  - id: root-dimension
    title: The root_parent dimension in bob stats
    depends_on:
      - retire-rollup
    size: medium
    description:
      "root-dimension: derive each home's root parent in the resolver and report an
      additive root_parent parts dimension and catalog field for Pomodoro and task
      metrics, with human output, tests, docs, and a performance check."
  - id: strip-above
    title: Day strip above the badges with a full-width root-parent bar
    depends_on: []
    size: medium
    description:
      "strip-above: host the strip as the bob-plan block's first child, add the
      full-width bar row under a legend and meta row, prefer root_parent parts with an
      area fallback, and extend the preview harness and tests."
  - id: stats-roots
    title: Stats dash Root parents view and plugin rollup removal
    depends_on:
      - root-dimension
      - strip-above
    size: medium
    description:
      "stats-roots: replace Areas & projects with a Root parents view grouped by area
      with sub-project disclosure, route the strip's hand-off to it, drop rollup from
      the JS home mirror, and refresh fixtures and previews."
  - id: closeout
    title: Contract docs, heading total cleanup, glossary strands
    depends_on:
      - stats-roots
    size: small
    description:
      "closeout: rewrite the Day strip and Stats block contract in docs/stats.md, remove
      the heading total per the decision, add the four glossary strands, and settle
      memory beads bob-cli-bt and bob-cli-bs."
proposed_by: bbugyi200.athena.145
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 10:27:21
status: wip
---

- **PROMPT:**
  [prompts/202610/day_strip_above_badges_root_parents.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/day_strip_above_badges_root_parents.md)

# Plan: The Day strip moves above the badges, and working time groups by root parent

## Context

Bryan wants the Day strip (epic `bob-cli-bz`,
`plan:202610/day_strip_under_the_badges.md`) moved from under the `bob-plan` badges to
above them, so that its working-time bar becomes the natural separator between the day's
facts and the plan badges. He also wants that bar to group by **root parent**, the stats
dash to offer a good root-parent view, four new glossary strands, and two retirements:
the `rollup:` frontmatter key and the working-time total in the `## Pomodoros` heading.
He asked me to lead the design.

An earlier research prompt for this change also asked to restore the old tile look. This
request dropped that line. **Keep the shipped label-value grammar.** This change moves
and regroups the strip; it does not bring back tiles, history marks, the chevron, or
`<button>` elements.

### Verified facts this plan relies on

- **Hosting.** The strip is a stable host `div` inside the rendered `bob-plan` block,
  painted after the `.bob-plan` badges container. The relevant bob-plugins code is:
  - `plugins/bob-ledger-tools/src/239-plugin-day-strip-plan-host.js`, through
    `dayStripPlanHostPlacePlanFirst` and `dayStripPlanHostEnsurePlaceholder`, which
    returns `el.children[0]`;
  - the `renderPlanBlock` / `paintPlanBlock` hooks in
    `290-plugin-today-and-location.js`.
- **Mix.** The strip's mix reads `worked.points[D].parts.area` (`dayStripMix` in
  `239-plugin-day-strip-model.js`). It draws a 7.5em bar plus a legend of the top 3 and
  `+N`, ordered by `statsPanelHomeAreaRows`, which sorts by area hue slot, then label,
  with buckets last.
- **Stats panel.** "Where the time went" has two toggles, `Areas & projects` and
  `Themes` (`statsPanelHomeToggle` in `239-plugin-stats-homes-render.js`).
  - The areas view draws an area bar, then area groups. Each group holds up to 3 project
    rows plus `N more`, and shows `includes N sub-projects` from the catalog's
    `rolled_up`.
  - `homesView` is in-memory view state (`statsViewStateFor`), not persisted.
  - `openStatsForDay` (`239-plugin-day-strip-fence.js`) sets `revealPanel = "homes"` but
    never sets `homesView`.
- **Home resolution.** It lives in `src/native/pomodoro_homes/resolve.rs`
  (`HomeResolver::resolve_note`):
  - It walks scalar `parent:` links, at most 16 notes, stopping on a cycle.
  - The home is the first area or project note on the chain. Archive and reference notes
    are walked through.
  - `rollup: subprojects` reselects the outermost opted-in project.
  - `pre_rollup_chain` keeps the original chain for `in:` / `exclude_in:` filters
    (`home_chain_matches` in `src/native/stats/metrics.rs`).
- **The vault (2026-10-25).**
  - No note has a `rollup:` key, so retiring it changes no live result.
  - 96 project notes collapse to **9 root parents**: `sase` has 82 sub-projects, `bob`
    6, and `job_2026` 2. The maximum project depth is 4. There are 13 area notes,
    including the sub-areas `gtd_daily`, `mac_inbox`, and `gkeep_inbox`.
- **The heading total.** It is a Dataview inline query, not bob output. The exact line
  is:

  ```text
  ## Pomodoros (`= durationformat(default(sum(nonnull(map(filter(this.file.tasks, (task) => task.completed AND startswith(meta(task.section).subpath, "Pomodoros")), (task) => task.t))), dur("0m")), "h'h' m'm'")`)
  ```

  - It appears in `~/bob/_templates/daily.md` and in **146** daily notes, from
    2026-06-02 on.
  - Only **27** of those have a ` ```bob-plan ` block (2026-09-29 on), so only they show
    a Day strip.
  - Every parser already accepts a bare `## Pomodoros`: `is_pomodoros_heading` in
    `src/native/pomodoro.rs`, and the plugins' `POMODOROS_HEADING_RE`.

- **Registry text.** `src/native/stats/registry.rs` describes `worked` as "…; equals
  each daily note's heading."
- **Shared vectors.** The plugins' home vectors
  (`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/vectors.json`) are a
  byte-for-byte copy of bob-cli's `tests/fixtures/pomodoro_homes/vectors.json`.
- **Research input.** Three swarm reports informed the design:
  - `research:202610/day_strip_root_parent_design__cdx.md`
    (`explicit:3165ae72832e74772fb74b39`);
  - `research:202610/day_strip_above_badges_root_parent__mus.md`
    (`explicit:cbc99be7a2f49d5eb3e0ac96`);
  - `research:202610/day_strip_root_parent_and_layout_redesign__gem.md`
    (`explicit:e15bbca96e1c8cf010a2770f`).

  I adopted these points from them:
  - the whole strip moves above the badges, never split around them;
  - bob computes the root dimension, never the plugin;
  - allocate per task first, then group by root;
  - the lead root comes from task counts grouped by root;
  - the strip's hand-off must set the view explicitly.

  I rejected these:
  - restoring the tiles;
  - a deprecate-then-remove rollup cycle (no vault note uses the key);
  - a three-button dash (see the dash section below).

## Design contract (every phase builds to this)

### Mockups

These mockups use the production `styles.css` plus the proposed rules. They are the
visual target; phases tune the details in the real preview harnesses.

- `file:explicit:0142f95e874aea8dd9c54b03` shows the daily note in these states:
  - today: light and dark at 700 px, and light at a 410 px split;
  - a past day;
  - no Pomodoros, in dark;
  - the skeleton.
- `file:explicit:586e90f4b27e75bb67c147b2` shows the stats dash Root parents view, with
  one sub-project disclosure open.
- `file:explicit:2fb991e812b955b23e2dc601` is the generator, holding the proposed CSS
  and DOM.

### Vocabulary

- **Top-level project.** A project note whose `parent` walk reaches an area note before
  any other project note. Normally its `parent` is an area, and a sub-area counts. A
  project whose `parent` is another project is a **sub-project**, at any depth. "Top"
  describes the project/area boundary, never folder depth, size, or `^prj` status.
- **Root parent.** The note a task's time is grouped under:
  - the area note the task lives in, when it lives in an area note;
  - otherwise, the top-level project reached by walking `parent` up from the project
    note it lives in.

#### Resolution algorithm (Rust `resolve_note`)

Take `chain` as built today, and `h` as the index of the nearest area or project (the
home).

- If `chain[h]` is an area, `root = chain[h]`.
- If `chain[h]` is a project, let `a` be the first area index above `h`, or the chain's
  end. Then `root` is the **last project** in `chain[h..a]`. Non-area, non-project notes
  in between are walked through, exactly as R1 walks archive and ref notes.
- If the walk never reaches an area, the outermost project reached stands in as the
  root. It is not a top-level project, and its area stays `@unfiled`.
- A bucket is its own root. That covers `@review:<NAME>`, `@unlinked`, and `@unfiled`
  (an unresolved, ambiguous, or homeless link).

#### Examples

| Task lives in                           | Walk                                      | Root parent                       |
| --------------------------------------- | ----------------------------------------- | --------------------------------- |
| `sase_blog.md` (project)                | `sase_blog → sase (project) → dev (area)` | `sase`                            |
| `sase.md` / `bob.md` (project)          | `→ dev (area)`                            | itself                            |
| `dev.md` (area)                         | —                                         | `dev`                             |
| `gtd_daily.md` (sub-area)               | —                                         | `gtd_daily`, the area it lives in |
| `p.md` (project, parent `gtd_daily`)    | `p → gtd_daily (area)`                    | `p` (a sub-area counts)           |
| `done/sase_blog_done.md` (archive)      | `→ sase_blog → sase → dev`                | `sase`                            |
| `ref/x.md` (parent `bob`)               | `→ bob → dev`                             | `bob`                             |
| `ref/y.md` (parent `dev`)               | `→ dev`                                   | `dev`                             |
| orphan project `o.md` (no area above)   | `o → …`                                   | `o`, area `@unfiled`              |
| GTD review session / `[[#^gtd]]` bridge | —                                         | `@review:GTD` ("GTD review")      |
| closed session with no Task Link        | —                                         | `@unlinked`                       |
| missing or ambiguous link               | —                                         | `@unfiled`                        |

#### Allocation and identity

- **Allocation is unchanged.** R5 splits each session's minutes per distinct task first.
  Root parts are those per-task shares summed by root, so every dimension still sums to
  the metric total.
- **Apportioned** root minutes come from sessions with two or more distinct roots.
- **Count mode.** The lead root is the root with the most distinct tasks, ties going to
  the first task in ledger order.
  - It is not the lead project's root. With A=2 and B=2 tasks under root R, and C=3
    under root S, the lead is R.
- **Labels** are the lowercase note stem (`sase`, `gtd`). Buckets keep `GTD review`,
  `unlinked`, and `unfiled`.
- **Hue** is the root's top-level area hue (R8). Colour means area; a segment means a
  root parent. Same-area roots share a hue and are told apart by order, gaps, and
  labels.
- **No snapshot (R10).** Roots are derived from today's note tree at read time. Moving a
  project regroups past minutes and never changes totals.
- **Filters.** `in:` and `exclude_in:` keep matching the full residence chain, so
  `in: [sase_blog]` still measures blog time on its own.

### `bob stats` JSON contract (schema 1, additive)

- **Parts.** Every metric that carries `area` and `project` parts (Pomodoro and task
  kinds) also carries `root_parent`, in all four places:
  - `points[].parts.root_parent`, `points[].parts_display.root_parent`,
    `points[].apportioned.root_parent`;
  - `parts_summary.root_parent`, whose `PartSummary` rows hold the bare catalog `label`
    plus minutes-mode `apportioned` / `apportioned_display`, like `project`.
  - These cover grouped and prior points too, and keep every key: no `other` collapse.
- **Catalog.** Every `homes[<key>]` entry gains `root_parent: "<root key>"`. A root maps
  to itself, and so does a bucket.
  - The catalog contains every root a report references. `sase.md` is present even when
    only `sase_blog.md` has minutes in the span.
  - `prune_home_catalog` keeps root keys.
- **Removed** (a contract break; commit subject marked `!`):
  - the `homes[].rolled_up` field;
  - the `health.homes.rollup_invalid` list;
  - the `rollup_invalid` warning code.
- **Unchanged.** The R9 session `homes` object, `sessions[]`, `area`, `project`, and
  `theme`. The `project` dimension becomes plain nearest-home detail, with no rollup.
  Because the R9 object is unchanged, Bob Mac Capture and the ledger home chips need no
  change beyond the rollup deletion.
- **Human output.** A `root parent` line prints between `area` and `project`:
  - Pomodoro dimensions in order: `area`, `root parent`, `project`, `theme`.
  - Task dimensions in order: `kind`, `area`, `root parent`, `project`, `note`.
  - Every key is printed with its catalog label and a `●` in the area hue. Buckets are
    dim.

### The Day strip

```text
WORKED 5h 25m  REVIEWED 64  DONE 30  CREATED 40  REVIEW CLOSED not yet     ← line 1: facts (unchanged)
● bob 45m  ● sase 2h 10m  ● GTD review 1h 05m  +3            so far · 14:50  ← line 2: legend + meta
▬▬▬▬▬▬▬▬ ▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬ ▬▬▬▬▬▬ ▬▬▬▬▬▬▬▬▬▬ ▬▬▬▬ ▬▬            ← line 3: full-width bar
[TODAY 3/3 · 7/10] [PENDING 4/10] [NEXT 15/15] [READY 61/100]  ★ GTD · BOB   ← plan badges
## Pomodoros
```

#### Placement

- The strip host is the **first child** of the rendered `bob-plan` block, with
  `.bob-plan` directly after it. This holds on first paint, on every 5 s repaint, and in
  both Live Preview and Reading view.
- The host node is never detached. Its paint, observer, hover, and focus survive
  repaints, exactly as today.
- With no host (strip off, future day, non-daily note, dashboard), `.bob-plan` stays the
  first child as today.
- Every other Where-it-renders rule from `docs/stats.md` is unchanged:
  - nothing on mobile, in hover popovers, embeds, or canvas, or in source mode;
  - the fence keeps its behaviour and gets the same layout.

#### Line 1: facts

Exactly as shipped: five `LABEL value` span links, metric order, and every state.

#### Line 2: legend and meta

- The legend is one `span[role=link]` holding up to three items:
  - the top three roots by minutes, shown in bar order;
  - each item is a hue dot, the bare label, and the value;
  - then `+N`.
- The meta (`so far · HH:MM`, plus the stale and `refresh failed` variants) is
  right-aligned, as today.
- **Labels never ellipsize.** When the strip is narrower than about 30em, a container
  query hides the rank-3 item and swaps in a second counter, `+N` with N = segments − 2.
  The painter renders both counters, and each one's `title` lists the roots it hides.
- `no Pomodoros` stays the empty-state legend text.

#### Line 3: the bar

- A full-width `div` (`aria-hidden`), 4 px tall and fully rounded.
- One segment per root, with `flex-grow` = minutes and **2 px gaps of note background**
  between segments, not track colour.
- Each segment has a `title` of `<label> <display>`.
- The track (`--background-modifier-border`) shows only when the bar is empty or in the
  skeleton.
- The bar is a second, mouse-only target for the legend's action. It has no tabindex.
  `mousedown` calls `preventDefault`.
- The bar is the strip's last line, so it reads as the separator above the badges.

#### Ordering, dimension, and fallback

- Segments are ordered with `statsPanelHomeAreaRows`: area hue slot, then label, with
  buckets last in gray.
- The mix prefers `parts.root_parent`. A report without it, from an older bob, falls
  back to `parts.area`, and the copy then says "Time by area" honestly.
- A redefined `worked` (not minutes, or no parts) omits line 2's legend and line 3 as
  today. Line 2 then holds only the meta on today's note.

#### Copy and accessibility

- The legend link's `aria-label` and `title` read
  `Time by root parent: bob 45m, sase 2h 10m, …`, listing every segment.
- The group `aria-label` sentence uses the same phrase.

#### Spacing and size

- `.bob-day-strip` gets `margin: .5rem 0 0` and `container-type: inline-size`.
- `.bob-plan-has-day-strip > .bob-plan` gets `margin-top: ~.7rem` and keeps
  `margin-bottom: 1rem`. This replaces today's shrink-the-bottom rule.
- Line 2 sits `~.2em` below line 1, and the bar sits `~.45em` below line 2.
- There is still no card, border, background, shadow, or `<button>`.
- **Height budget at a 16 px base:**
  - ≤ 56 px when the facts fit on one line (540 and 700 px);
  - ≤ 80 px when they wrap once (410 px).
- The skeleton keeps the loaded geometry, including the empty bar track.
- Typography, colour tokens, motion, and the honest states follow the shipped contract.

### Stats dash: "Where the time went"

I chose **two views: Root parents (default) and Themes.** The root view replaces Areas &
projects without losing detail.

- Area groups keep their area totals.
- Every leaf project is one click away under its root.
- After rollup retires, a leaf-project grouping would show up to 82 `sase_*` rows under
  `dev`, which is exactly the noise rollup existed to hide. A third button would keep
  that noisy view alive for no question the root view cannot answer.

The view, top to bottom:

- **Header.** `Where the time went · <span> · <total>`, with the toggle
  `[Root parents] [Themes]`.
- **Root bar.** It reuses the area-bar component and is fed root rows in the same order
  and hues as the Day strip bar.
  - Segment labels show inside when wide enough.
  - `aria-label` reads `Root parent split: …`.
- **Groups.** One group per area, ranked by area total (from `area` parts). Each group
  has:
  - its header row: dot, area link, `value · share`, `sessions N`;
  - up to 3 root rows, plus `N more` / `Show fewer`, as today. The area's own root reads
    `<area> · area tasks` (catalog `project_label`), because it sits under a header that
    means the whole area.
- **Root rows.** Each row has a label link, `value · share`, the apportioned mini bar,
  `sessions N` (sessions with any part whose catalog `root_parent` is this root), and
  fix-it copy for buckets.
- **Sub-project disclosure.** A project root with at least one other leaf in the span
  gets a quiet `▸ N sub-projects` toggle, where N excludes the root itself.
  - It expands nested leaf rows from `project` parts whose catalog `root_parent` is the
    root, ranked by value.
  - Each leaf row has a label link, a value, and a mini bar scaled to the root's value.
  - The root's own leaf reads `<root> · own tasks`.
  - The toggle state is `homeSubsExpanded[root]`, with focus key
    `panel-home-subs-<key>`, collapsed by default.
- **Bucket groups.** `@unlinked` and `@unfiled` render as their header row plus fix-it
  copy only. They no longer repeat themselves as a child row.
- **Footer.**
  `▒ split from sessions with several root parents · grouped by today's note tree · from Task Links`.
- **Task metrics** (`{ title: true }` path): `<Label> by root parent`, with the same
  grouping and no bar.
- **Hand-off.** `openStatsForDay({ panel: "homes" })` sets `homesView = "roots"`, even
  after a Themes visit.
- **Old bob fallback.** With no `parts_summary.root_parent`, the first button reads
  `Areas & projects`, renders today's leaf grouping, and shows
  `Update bob to group by root parent.` A stale `homesView === "areas"` maps to `roots`.

### Retirements

- **`rollup:` frontmatter.**
  - Delete it from Rust (`facts.rs`, `resolve.rs`, `stats/homes.rs`, `report.rs`, and
    `projects/{mod,model,scan,output}.rs`) and from the JS home mirror
    (`142-pomodoro-home-chips.js`, `273-plugin-pomodoro-home-chips.js`, and the Stats
    `rolled_up` line).
  - A leftover key is silently ignored: no effect, no warning.
  - `bob projects list` drops the `rollup` marker and warning, and its help drops
    "reporting rollups". The CLI reference is regenerated.
  - Rename `pre_rollup_chain` to `residence_chain`.
- **Heading total.** Replace the exact line above with `## Pomodoros` per
  `heading_cleanup`. Change nothing else in those files.

## Phase: retire-rollup

**Repo:** bob-cli. Read the `docs.md` and `cli_rules.md` memory first.

1. **Delete rollup** everywhere listed under Retirements, in Rust:
   - In `resolve_note`, the home is the nearest area or project.
   - Remove `Rollup`, `rollup_value`, `NoteFacts.rollup*`, `rollup_linted`,
     `lint_rollup`, and `ResolverHealth.rollup_invalid`.
   - Remove the `rolled_up` counting, `rollup:` from `needs_global_inspect`, the
     `rollup_invalid` warnings, and `HomeCatalogEntry.rolled_up`.
   - Rename `pre_rollup_chain` → `residence_chain` across `pomodoro_homes` and `stats`.
2. **`bob projects list`.** Remove the marker, the warning, and the model fields. Edit
   the help text, then run `just docs-gen`.
3. **Fixtures and tests**
   - `tests/fixtures/stats/homes_vault/sase.md` drops `rollup:`. Delete `invalid.md`.
   - In `tests/fixtures/pomodoro_homes/vectors.json`:
     - remove every `rollup:` key;
     - rename `rollup-archive-review` → `subproject-archive-review` and
       `rollup-depth-and-parent-shapes` → `depth-and-parent-shapes`;
     - recompute their expected homes by hand, so `sase_blog` is now its own home;
     - drop `rollup_invalid` from the health expectations.
   - Do **not** copy the vectors to bob-plugins yet; `stats-roots` does that together
     with the JS mirror.
   - Update the H-vectors:
     - H3 becomes the leaf projects `sase 30` and `sase_blog 50`; everything else is
       unchanged and still sums to 167;
     - H8 loses `rollup_invalid`;
     - H10 reads "measures on its own".
   - Delete the rollup unit and CLI tests. Add one test that a leftover
     `rollup: subprojects` neither regroups nor warns.
4. **Docs**
   - In `docs/pomodoro-homes.md`, R2 becomes a one-paragraph "Retired: project rollup"
     note. Root parents arrive in the next phase and replace it.
   - Remove the rollup mentions from R10, the vectors table, and the limits list.
   - In `docs/projects.md`, remove the "Rollup (reporting only)" section, its TOC entry,
     and the list-output sentences.
   - In `docs/stats.md`, cover the `in:` wording, the schema fields, the warning codes,
     and H3, H8, and H10.
   - Also update `docs/reference/vault-format.md` and the summaries in `docs/README.md`,
     `docs/develop/contracts.md`, and `docs/_meta/contracts.yml`.
5. **Gates:** `just check` and `just docs-check`.
   - Commit as `feat(homes)!: retire project rollup frontmatter`.
   - Never edit `Cargo.toml`'s version or `CHANGELOG.md`.

## Phase: root-dimension

**Repo:** bob-cli. Build exactly the Vocabulary, Allocation, and JSON contract above.

1. **Resolver.** `Home` gains `root_parent: String`, computed in `resolve_note` by the
   algorithm above. `review_home`, `unfiled_home`, and `unlinked_home` set their own
   key.
2. **Engine** (`src/native/stats/homes.rs`)
   - `SessionSplit` gains `root_parent`, `root_parent_apportioned`, and
     `lead_root_parent`, filled in both `split_from_homes` and `split_from_kept`.
   - Group the per-task shares by root (use `SessionHomes.task_homes`). Never split a
     session by roots directly.
   - Every referenced root is remembered in the catalog (resolve the root path's
     `Home`), and `catalog_entry` sets `root_parent`.
3. **Metrics and serialization**
   - `metrics.rs`: add day accumulators and the count-mode lead root. Task metrics add
     `root_parent` beside `area` and `project` via `add_dimension_part`.
   - `compute.rs`: `prune_home_catalog` keeps root keys.
   - `label_parts` labels `root_parent` with the bare `entry.label`.
   - Confirm that week and month grouping sums the new dimension like the others.
4. **Human output** (`render.rs`). Add the `root parent` line in the order the contract
   gives. If CSV enumerates part dimensions, include it there too.
5. **Registry.** The `worked` description becomes "Minutes in closed Pomodoros under ##
   Pomodoros." The heading total is retiring.
6. **Tests**
   - Resolver cases, one per row of the Examples table:
     - depth-4 sub-projects;
     - a project → plain note → project → area chain;
     - the 16-note cap;
     - cycles.
   - H16 `worked root_parent` sums to 167: `sase 80` (sase_blog 50 + sase 30),
     `@review:GTD 35`, `bob 10`, `dev 10`, `job 4`, `job_apply 3`, `@unlinked 10`,
     `@unfiled 15`.
   - Root apportioned values, derived by hand.
   - The lead-root A/B/C unit test.
   - `done` root parts.
   - The catalog includes `sase.md` when only `sase_blog` has minutes.
   - `in: [sase_blog]` stays 50.
   - A grouped-week report conserves totals.
   - Repeated `compute` is byte-identical.
   - Golden JSON asserts `root_parent` beside unchanged `area` and `project`.
7. **Performance.** Measure release `bob stats -f json -r 4w` on the live vault (9 runs
   after a warmup) before and after.
   - The median must stay within 10% of the before median.
   - Record both medians in `docs/stats.md` § Performance.
8. **Docs**
   - In `docs/pomodoro-homes.md`, write R2 "Root parent": the definition, the algorithm,
     the examples table, buckets, hue, split and lead rules, and the residence-chain
     filter note. Mention it in R10.
   - In `docs/stats.md`, cover the data model, the schema-1 fields, the human output
     order, and the H16 row.
   - Update `docs/reference/json-outputs.md` if it enumerates dimensions.
   - Gate: `just docs-check`.
9. **Gates:** `just check`. Commit as
   `feat(stats): group Pomodoro and task parts by root parent`.

## Phase: strip-above

**Repo:** bob-plugins, plugin `bob-ledger-tools`. Open it with
`sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`:

- edit `src/` fragments, run `npm run build`, and never hand-edit `main.js`;
- keep every fragment at or below 1000 lines;
- run `bob plugins sync` after committing.

This phase runs in parallel with the bob-cli phases. Until bob ships `root_parent`, the
deployed strip honestly falls back to areas.

1. **Plan host** (`239-plugin-day-strip-plan-host.js`)
   - **Attach:** insert the host before the block's first child, falling back for
     fake-DOM test harnesses.
   - `dayStripPlanHostEnsurePlaceholder` finds `.bob-plan` by class, never by
     `children[0]`.
   - Rename `dayStripPlanHostPlacePlanFirst` to `dayStripPlanHostPlacePlanAfter`, which
     puts `.bob-plan` directly after the host.
   - **Repaint** keeps the host first.
   - `290-plugin-today-and-location.js` (about 935 lines) changes only the hook name and
     its comments.
2. **Model** (`239-plugin-day-strip-model.js`)
   - `dayStripMix` picks `root_parent`, else `area`, and returns `dimension` and a
     matching `title`.
   - Segments use bare labels. In `239-plugin-stats-panel-models.js`,
     `statsPanelHomeLabel` returns `entry.label` for `root_parent`; `stats-roots` builds
     on this.
   - Legend entries carry `rank`. The mix exposes `overflowWide` and `overflowNarrow`
     (counts plus titles).
   - Update the aria and title sentences.
3. **Painter** (`239-plugin-day-strip-render.js`)
   - DOM order: facts, then `div.bob-day-strip-mix-row` (legend link + meta), then
     `div.bob-day-strip-bar` with
     `span.bob-day-strip-bar-segment.bob-day-strip-hue-<hue>`.
   - Delete the in-row `bob-day-strip-mix-bar`.
   - Keep fixed slots so the skeleton and ready trees match: 3 legend items, both
     counters, and the bar element.
   - The bar uses `dayStripActivate`-style mouse handlers with no keyboard stop. Emit no
     `<button>`.
4. **Styles.** Rewrite the `/* Day strip */` section to the contract above:
   - placement margins, the full-width bar, and transparent gaps;
   - the container query for narrow legends, with labels never ellipsized;
   - the new `.bob-plan-has-day-strip > .bob-plan` rule.
5. **Fixtures** (`scripts/fixtures/stats/day-strip-fixtures.cjs` and its `README.md`)
   - By default, emit `root_parent` parts, displays, and catalog `root_parent`.
   - Add `areaOnly` (old bob), `sameHueRoots` (bob/sase/dev blue, GTD review/gtd orange,
     unlinked), `manyRoots` (7 roots), and a long `job_2026_apply_29` label.
6. **Preview harness** (`scripts/preview-day-strip.cjs` + template)
   - In the Live Preview and Reading cells, put the strip first, the mock badges after
     it, then a bare `## Pomodoros`.
   - Keep checks (a)–(h), update the budget values, and add:
     - (i) the strip precedes `.bob-plan` in the DOM, and its bottom sits 6–16 px above
       the badges' top;
     - (j) the bar spans the strip's content width (±1 px) and is its last line box;
     - (k) no legend label is truncated (`scrollWidth ≤ clientWidth + .5`) at 410, 540,
       or 700 px;
     - (l) narrow cells show at most 2 legend items, and the visible counter equals
       segments − visible items.
   - Run it, look at every screenshot against the mockup, and iterate until it is calm
     and aligned in both themes.
   - Attach the final PNGs with `sase artifact create -p <png> -l "<label>" --bead`.
7. **Tests**
   - Model: root preference and area fallback, labels, rank and counters, aria copy.
   - Painter:
     - order facts → mix row → bar;
     - a bar click calls `openStats({ metric: "worked", panel: "homes" })` with
       `newLeaf` modifiers;
     - no `button`;
     - skeleton/ready tag-sequence parity, including the bar.
   - Plan host:
     - the host is the first child in Live Preview and Reading ctx;
     - `.bob-plan` is second after paint and after repeated `rerenderPlanBlocks`;
     - the same host node is kept;
     - the placeholder is found by class;
     - toggle-off restores `.bob-plan` first;
     - rollover attach puts the host first.
   - Update other suites only where they locate `.bob-plan` by index on a daily path
     that now has a host.
8. **Docs and manifest**
   - In `docs/bob-ledger-tools.md`, update the Day strip paragraph and the session
     toggle line: above the badges, three lines, the root-parent bar.
   - The `manifest.json` sentence becomes "Render the read-only Day strip above the
     bob-plan badges of canonical daily notes, …".
   - Bump the minor version, then run `npm run docs:gen` and `npm run docs:check`.
9. **Gates:** `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`, and
   `node scripts/preview-day-strip.cjs` with zero failures. Then `bob plugins sync`.

## Phase: stats-roots

**Repo:** bob-plugins / `bob-ledger-tools`, under the same rules.

1. **Models** (`239-plugin-stats-panel-models.js`)
   - Add `statsPanelHasRootParents`.
   - `statsPanelHomeSessions` matches `root_parent` through the catalog.
   - Add `statsPanelRootSubRows(metric, report, rootKey, point)`, including the
     `· own tasks` row.
   - Root rows carry `catalog[root].area`.
2. **Render** (`239-plugin-stats-homes-render.js`, `239-plugin-stats-panel-render.js`)
   - Build the dash view exactly as specified above:
     - the toggle, root bar, groups, and disclosure;
     - bucket groups, the footer, and the task-metric title;
     - the old-bob fallback.
   - Delete the `rolled_up` line.
   - Keep each fragment ≤ 1000 lines. If needed, move the disclosure into a new
     `239-plugin-stats-roots-render.js` registered in `src/fragments.json`.
3. **Hand-off.** In `openStatsForDay`, set `state.homesView = "roots"` when
   `panel === "homes"`.
4. **JS home mirror**
   - Delete rollup parsing, `rollup_invalid` health, and warnings in
     `142-pomodoro-home-chips.js` and `273-plugin-pomodoro-home-chips.js`.
   - Copy bob-cli's current `tests/fixtures/pomodoro_homes/vectors.json` byte-for-byte
     into `scripts/fixtures/pomodoro_homes/`, and update the README's source revision.
5. **Fixtures**
   - Regenerate `scripts/fixtures/stats/canonical.json` and `canonical.meta.json` from
     bob-cli's `homes_vault`, using the README procedure against bob-cli master, which
     carries `root-dimension`.
   - Give `synthetic-homes*.json` `root_parent` parts and catalog fields, including a
     root with several sub-projects.
6. **Preview** (`scripts/preview-stats-block.cjs`)
   - Add cases for:
     - Root parents as the default;
     - a disclosure open;
     - bucket groups;
     - Themes;
     - the old-bob fallback.
   - Run them light and dark at narrow and wide widths, with no overflow.
   - Inspect the shots against `file:explicit:586e90f4b27e75bb67c147b2`, and attach the
     PNGs to the phase bead.
7. **Tests**
   - Root rows: grouping, ordering, labels, `· area tasks` and `· own tasks`.
   - Sessions filtering, disclosure toggling and focus keys, bucket groups.
   - The default view and the `areas` → `roots` mapping, plus the fallback copy.
   - The `openStatsForDay` view selection.
   - The JS mirror passes the refreshed vectors.
8. **Docs and manifest.** Update the Stats sentence in `docs/bob-ledger-tools.md`, bump
   the minor version, and run `npm run docs:gen` and `docs:check`.
9. **Gates:** as in `strip-above`, plus the stats preview. Then `bob plugins sync`.

## Phase: closeout

1. **bob-cli `docs/stats.md`**
   - Rewrite `### Day strip` to the shipped contract:
     - placement above the badges, with the bar as the separator;
     - three lines, root-parent grouping with the area fallback;
     - the narrow-legend rule.
   - Update the "Obsidian Stats block" bullets: the Root parents default plus Themes,
     area groups, the sub-project disclosure, bucket groups, and the hand-off.
   - Check `docs/dashboard.md`, and fix it only if it is now false.
   - Run `just docs-check` and commit as `docs(stats): …`.
2. **Vault** (`~/bob`, on athena)
   - Run `bob vault-sync` first.

   > [!decision] heading_cleanup = template Edit only `_templates/daily.md`.

   > [!decision] heading_cleanup = strip_notes Edit `_templates/daily.md` and every
   > canonical daily note (`YYYY/YYYYMMDD.md`) that contains a ` ```bob-plan ` fence,
   > future notes included.

   > [!decision] heading_cleanup = all_notes Edit `_templates/daily.md` and every daily
   > note containing the exact line.
   - Replace only that exact line with `## Pomodoros`, preserving line endings and every
     other byte.
   - Verify with before/after `grep -c` counts, then run `bob vault-sync` and check
     `bob vault-sync status --json`.

3. **Memory.** Use `/sase_memory_write`, then `sase memory init`. Add the four
   `glossary` strands below, each authorized by its own decision; skip any strand whose
   decision was answered no. Keep links as `[[...]]` to existing strands.

   > [!decision] glossary_top_project Add `top-level-project.md`.

   > [!decision] glossary_root_parent Add `root-parent.md`.

   > [!decision] glossary_day_strip Add `day-strip.md`.

   > [!decision] glossary_stats_dash Add `stats-dash.md`.
   - **`top-level-project.md`** (keyword "Top-Level Project", alias "top project"):
     - a [[project-note]] whose `parent` walk reaches an [[area-note]] before any other
       project — normally its `parent` is an area, and a sub-area counts;
     - a project under another project is a sub-project at any depth and belongs to the
       top-level project above it;
     - it is the project grain of [[root-parent]] reporting;
     - "top" is never folder depth, size, or status;
     - see `docs/pomodoro-homes.md`.
   - **`root-parent.md`** (keyword "Root Parent"):
     - the area note a task lives in, else the [[top-level-project]] above its project
       note;
     - archive and reference wrappers are walked through, and an area-less walk falls
       back to the outermost project;
     - buckets `GTD review`, `unlinked`, and `unfiled` stay buckets;
     - derived at read time from today's note tree and never written; `rollup:` no
       longer exists;
     - the Day strip bar and the stats dash default view group by it; Stats JSON calls
       the dimension `root_parent`;
     - it is not the nearest home (`project`), a top-level area, or a
       [[pomodoro-theme]].
   - **`day-strip.md`** (keyword "Day Strip", alias "day stats"):
     - the read-only daily Stats summary at the top of the `bob-plan` block on canonical
       daily notes, above the badges, in Live Preview and Reading view, or wherever a
       `bob-day-stats` fence embeds it;
     - its contents: five LABEL-value facts, a root-parent legend with `so far · HH:MM`,
       and the full-width root-parent bar that separates it from the badges;
     - bob owns every value, and it never writes the note;
     - `## Pomodoros` no longer repeats its total;
     - facts and the bar open the [[stats-dash]] with the day pinned;
     - "Day stats" in UI copy (`Toggle Day stats`, `stats.day_strip`);
     - not the [[morning-gtd-review-footer]].
   - **`stats-dash.md`** (keyword "Stats Dash", alias "dash_stats"):
     - `~/bob/dash_stats.md` (Obsidian aliases Stats, Dashboard Stats), whose single
       `bob-stats` fence renders `bob stats` JSON: tiles, the focus chart, day pinning,
       and family panels;
     - "Where the time went" groups by [[root-parent]] by default, with sub-projects one
       click away, or by [[pomodoro-theme]];
     - opened from the dashboard STATS chip, Open Stats, or any [[day-strip]] fact;
     - not `dash.md` or the `bob stats` terminal output.
   - Commit as
     `docs(memory): add root parent, top-level project, day strip, and stats dash terms`.

4. **Beads**
   - `sase bead close bob-cli-bt --note "<the strand landed; commit>"`.
   - Add a `sase bead note` on `bob-cli-bs` stating that the Day strip now renders above
     the badges with a root-parent bar, and that rollup is retired. The decision record
     itself stays Bryan's call. Do not edit memory for it.

## Verification on the Mac (Bryan, after `stats-roots`)

Run `bob plugins sync`, update `bob`, and reload Obsidian. Then check:

- **Today's note**
  - Facts first, then the legend and `so far`, then a thin full-width bar right above
    the badges, with nothing overlapping.
  - `sase` time from any `sase_*` sub-project shows as one `sase` segment.
  - Clicking the bar opens the stats dash on **Root parents** with today pinned, even
    right after viewing Themes.
- **Split pane:** the legend drops to two labels plus `+N`, and no label is cut off.
- **Stats dash:**
  - `dev` holds `sase`, `bob`, and `dev · area tasks`;
  - `▸ N sub-projects` under `sase` lists the leaf projects;
  - `unlinked` shows once, with its fix-it line.
- **Heading:** the daily template, and per `heading_cleanup` the existing notes, read a
  bare `## Pomodoros`.
