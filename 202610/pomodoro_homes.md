---
tier: epic
title: Pomodoro homes — derived area and project labels and stats
goal: 'Every Pomodoro is labeled, on every surface, by the areas and projects of the
  tasks it links. The labels are derived at read time from Task Links and never written
  to the vault. `bob stats` and `dash_stats.md` aggregate time and task throughput
  by area and project. `rollup: subprojects` on `sase.md` reports every sase sub-project
  as `sase`. Authored names keep working exactly as they do today, but naming a session
  becomes optional.

  '
decisions:
  review_bridge:
    ask: Count unnamed Jun–Aug sessions that link the daily ^gtd row as GTD review
      time?
    default: true
    why: In the named era ^gtd and GTD agree for 92 of 94 sessions; this keeps 114
      h of summer review
    answer: true
  review_time_predicate:
    ask: Redefine the built-in review_time metric as exactly the GTD review row's
      minutes?
    default: true
    why: The page then has one definition of review time, so the tile and the row
      always agree
    answer: true
  memory_mac_pom:
    ask: Update the Mac Menu Bar Pomodoro Indicator glossary term for derived home
      titles?
    memory:
    - glossary:mac-menu-bar-pomodoro-indicator
    default: true
    corrects: where THEME is the Pomodoro's ` — NAME` (`UNTITLED` when
    answer: true
  memory_decision_record:
    ask: Record "Pomodoro homes are derived, never written; rollup is reporting-only"
      as a decision?
    memory:
    - decisions:pomodoro-homes-are-derived
    default: false
    answer: false
  memory_home_term:
    ask: Add a glossary term Home (the derived area/project a task or Pomodoro belongs
      to)?
    memory:
    - glossary:home
    default: false
    answer: false
phases:
- id: homes-core
  title: Shared homes resolver, rollup switch, contract, and vault config
  depends_on: []
  size: medium
  description: 'homes-core: build the one Rust resolver for task homes, session links,
    review sessions, splitting, labels, chips, and hues. Parse and lint `rollup` and
    area `color`. Write the normative docs/pomodoro-homes.md and the shared conformance
    vectors, show the rollup flag in `bob projects list`, and configure sase.md and
    the six top-level area notes.'
- id: stats-parts
  title: Area and project parts in bob stats
  depends_on:
  - homes-core
  size: medium
  description: 'stats-parts: index closed-session links in the stats pass and add
    area and project parts to the pomodoro and task metrics, with buckets, a homes
    catalog, apportioned minutes, and health. Print every parts dimension in the CLI.
    Add an H-vector fixture vault and update docs/stats.md.'
- id: stats-drilldown
  title: Pomodoro in/review filters and session drill-down data
  depends_on:
  - stats-parts
  size: medium
  description: 'stats-drilldown: add the pomodoro `in:`, `exclude_in`, and `review:`
    filters, put review_time on the review predicate (per decision), and add the top-level
    `sessions` list that the panel drill-down reads. Add vectors and docs.'
- id: live-surfaces
  title: Homes on capture-pomodoros, bob plan, and bob pomodoro
  depends_on:
  - homes-core
  size: medium
  description: 'live-surfaces: add an additive `homes` object to capture-pomodoros
    --tasks entries and to bob plan JSON entries, and dim chips to capture-pomodoros
    human output. Make `bob pomodoro` print the derived label for unnamed sessions.
    Update the chezmoi menu-bar spec and tests, the docs, and the mac pom glossary.'
- id: ledger-chips
  title: Obsidian ledger home chips
  depends_on:
  - homes-core
  size: medium
  description: 'ledger-chips: add a bob-ledger-tools JavaScript mirror of the resolver
    that runs the shared vectors verbatim. Draw quiet lowercase home chips on Pomodoro
    entry lines in Live Preview and Reading view, with the area-hue CSS tokens, the
    suppression rules, and a toggle. Then sync the plugins.'
- id: stats-panel
  title: Where the time went panel on dash_stats.md
  depends_on:
  - stats-drilldown
  - ledger-chips
  size: medium
  description: 'stats-panel: replace the "by theme" pomodoro context panel with an
    Areas & projects view (area bar, projects grouped by area, apportioned tint, buckets,
    drill-down, pinned-day filter) and keep Themes one click away. Show the same table
    for task metrics, add preview fixtures and tests, then sync.'
- id: mac-agenda
  title: Bob Mac Capture agenda shows homes
  depends_on:
  - live-surfaces
  size: small
  description: 'mac-agenda: decode the additive `homes` object. Unnamed agenda and
    Refs Today entries show the derived label instead of "Untitled Pomodoro", and
    named entries gain a muted chip with an area-hue dot. Add fixtures and tests,
    and gate on portable-check plus macOS CI.'
proposed_by: bbugyi200.athena.research.4t.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-18 16:35:48
status: wip
bead_id: bob-cli-9k
---

- **PROMPT:** [prompts/202610/pomodoro_homes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/pomodoro_homes.md)
- **BEAD:** [bob-cli-9k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9k/README.md)

# Plan: Pomodoro homes — derived area and project labels and stats

## Why, and the one design move

Bryan asked to label every Pomodoro automatically by the areas and projects of its Task
Links, to aggregate `dash_stats.md` by area and project, and to let `sase.md` report its
sub-projects as itself. He accepted every requirement in the research report
`research:202610/pomodoro_area_project_attribution/pomodoro_area_project_attribution.md`.
Phase agents should read it with `sase artifact read` for evidence and figures. This
plan turns that report into buildable phases.

**The design move: a label is derived and displayed, never written.** Each Pomodoro
already lists its tasks, and each task already lives in an area or project note, so Bob
computes where a session's time went at read time. This is the same principle Bob
already applies to Today (`decisions:today-is-read-from-the-ledger`) and to Stats ("the
engine never writes"). It gives correct labels for every session back to June. It needs
no migration and causes no git churn on three hosts, and it never drifts when a task
moves, reconcile relocates a bullet, or a project is re-parented.

Typography makes the result readable at a glance:

- **ALL CAPS** is what Bryan meant: the authored ` — NAME`.
- **lowercase** is where the work landed: the derived home, a note stem exactly as typed
  in links.
- **One fixed hue per top-level area** is used everywhere: the ledger dot, the menu bar,
  the Mac agenda, and the Stats bar.

Today's ledger then reads:

```text
- [x] (0710-0800 50m) — GTD
- [x] (1310-1325 15m) — BOB
- [x] (1350-1405 15m) — BLOG     ● sase
- [ ] ()              — MEMORY   ○ sase
- [ ] ()              — HACK     ○ bob · dev +1
```

A filled dot means attributed (closed). A hollow dot means forecast (open; it updates as
links change). There is no chip when the name already says it (`— BOB`), on named review
sessions (`— GTD`), or when a session is unlinked.

## Vocabulary

- **Home:** the derived area or project a task belongs to (rules R1–R2), or a reserved
  bucket.
- **Session homes:** the homes of a Pomodoro's distinct linked tasks, with each one's
  task count and its minute share (R3–R6).
- **Review session:** a session whose minutes all go to the `GTD review` bucket (R4).
- **Buckets:** reserved homes. Every bucket is neutral gray except review, which takes
  its area's hue.
  - `@review:<NAME>`: labeled `GTD review`.
  - `@unlinked`: labeled `unlinked`.
  - `@unfiled`: labeled `unfiled`.
- **Forecast vs attributed:** an open session's homes are a forecast. A closed session's
  homes are the attribution that Stats counts.

## Normative rules

homes-core turns these rules into `docs/pomodoro-homes.md`. That file is the contract
for Rust, the JavaScript mirror, and the Mac decoder. Each rule gets at least one
conformance vector.

### R1 — A task's home

Input: a resolved link target `(note path, block id)`. Only the **note** decides the
home; the block is used only for de-duplication. Homes therefore never need a task scan,
and archived targets work naturally.

1. **Resolve** the link with the Today rules. An empty target is the session's own daily
   note. Otherwise use the exact vault path with or without `.md`, then a unique
   case-insensitive basename (`VaultLinkResolver` / `NoteIndex`). An ambiguous or
   missing target resolves to `@unfiled` and is counted in health (`ambiguous_links` /
   `unresolved_links`).
2. **Walk `parent:`** from that note. The parent must be a scalar wikilink; aliases,
   `.md`, and headings are stripped (`[[love.md|love]]` → `love.md`). A list-form,
   missing, or self parent ends the walk. The walk is cycle-safe: it uses a visited set
   and a depth cap of 16 (the `MAX_PROJECT_DEPTH` precedent). Archive notes
   (`done/*_done.md`, `type: [[done]]`) and ref notes (`type: [[ref]]`) have no type of
   their own here and are walked through.
3. **The nearest home** is the first note on the chain, starting with the note itself,
   whose frontmatter type is `area` or `project` (the `projects::scan` type helpers). A
   closed or canceled project is still a valid home. If no area or project is found, the
   home is `@unfiled`.
4. **Daily `^gtd` rows:** a link whose resolved note is a canonical daily note and whose
   block id is `gtd` (case-insensitive) has the home `@review:<first exempt name>`.
5. **Area:** the outermost `type: area` note on the chain is the **top-level area**. The
   nearest area at or above the home that is not the top-level one is its **subarea**,
   kept for drill-down only. A home with no area above it has the area `@unfiled`.

### R2 — The rollup switch

- `rollup: subprojects` in a **project** note's frontmatter makes every descendant
  project report as that project. This applies at any depth and whatever the
  descendant's status, and it covers labels, Stats parts, task-metric parts, and
  sessions.
- Absent and `off` both mean off.
- Any other value, or `rollup` on a non-project note, raises the lint `rollup_invalid`
  once per note, and the switch is treated as off. This follows the `ready_cap` /
  `note_ready_cap_invalid` precedent.
- **Outermost wins.** If several projects on the chain set it, the home is lifted to the
  outermost one.
- There is no child-side opt-out in v1. If one is ever needed, it will be a parent-side
  `rollup_except` later.
- **Reporting only.** Rollup never changes blocking or `#prj_is_blocked` inheritance,
  Ready caps, Sub-projects lines, review tiers, capture routing, or the plan budget.
  `docs/projects.md` says so explicitly.
- **Rollup groups; it does not hide.** The pre-rollup chain is kept, so the pomodoro
  `in:` filter (stats-drilldown) can still measure `sase_blog` alone.

### R3 — Which links count

- A session's links are its **dedicated Task Links**. These are direct child bullets at
  the entry's first child indentation whose body is exactly one block link, after
  stripping 🍅 markers and one `~~…~~` strike. The link can be `[[t#^id]]`,
  `![[t#^id]]`, or `[[t#^id|alias]]`.
- This is the Today and `=x` rule (`list_queued_links` in `capture_pomodoro_start.rs`)
  with three differences: struck links count, embeds count, and 🍅 is decoration only.
- Add a sibling of `list_queued_links` that reuses its helpers and includes struck
  links. `list_queued_links` itself stays unchanged.
- Excluded:
  - fenced code;
  - mixed-prose bullets;
  - deferred `[[t]]#` bodies;
  - deeper descendants and work notes;
  - Depends-On, Work Log, and Schedule Log lines.
- **De-duplication:** within a session, by resolved `(path, block id)`, or by the raw
  `(target, block id)` when the link is unresolved. Five links to one task count as one
  task.
- Open and closed sessions use the same rule. Open homes are a forecast; they are never
  counted as worked time.

### R4 — Review sessions

A session is a **review session** when either of these holds:

- it is named and every name component is in `plan.exempt` (default `[GTD]`); or
- it is **unnamed** and one of its dedicated links is that day's daily `^gtd` row.

> [!decision] review_bridge The second condition is the history bridge. If the answer is
> no, drop it: an unnamed `^gtd` session is attributed by its links. Its `^gtd` link
> still counts as one review-home task (R1.4).

All of a review session's minutes go to `@review:<NAME>`. `<NAME>` is the session's
exempt component; if several are exempt, the minutes split across them like composite
themes. The bridge uses the first configured exempt name, and if `plan.exempt` is empty,
the bridge is off. A review session's links never attribute.

The review bucket's area is the top-level area of the note that `[[<lowercased NAME>]]`
resolves to (`GTD` → `gtd.md` → `gtd`). If that note does not resolve to an area or
project, the area dimension uses the bucket key itself. Writing `GTD + SASE` makes the
session a work session again, so its links decide.

### R5 — Splitting and the lead home

- **Minutes.** Each distinct task gets an equal share of the session's counted minutes:
  the floor of the even split, with the remainder going to the first task in ledger
  order. Shares are summed per home, and an area's minutes are the sum of its homes'.
- **Count mode** (`sessions`): the whole session goes to its **lead home**. That is the
  home with the most tasks; ties go to the home whose first task comes first in ledger
  order.
- **Apportioned** (per dimension): minutes in a part that came from sessions with two or
  more distinct keys _in that dimension_.
  - Example: a `sase` + `dev_plex_hack_patch` session is apportioned in `project` but
    not in `area`.
- No composite home names (`sase + bob`) are ever generated.
- Every dimension's parts sum to the metric total, always.

### R6 — Unattributed buckets

- `@unlinked`: a closed session with no dedicated link that is not a review session.
- `@unfiled`: a task whose note has no area or project above it, or a link that is
  ambiguous or unresolved.
- Both buckets are always shown, never folded into `other`, and never guessed from the
  name. Their fix-it copy is "link the task to attribute it" and "file the task in an
  area or project".

### R7 — Labels and chips

- **Label:** the session's home labels ordered by share (descending; ties go to ledger
  order). At most two are shown, joined by `·`, then ` +N` for the rest: `bob · dev +1`.
  - A project or area home's label is its note stem, lowercased.
  - Review buckets use `GTD review`.
  - An unlinked session has an empty label.
- **Chip:** the label, or null when suppressed. A chip is suppressed when:
  - the session is unlinked;
  - it is a _named_ review session; or
  - the set of name components equals the set of home labels, compared
    case-insensitively (`— BOB` → bob, or `— SASE` with `sase_blog` rolled up into
    `sase`).
- An unnamed bridge session's chip reads `GTD review`. An unfiled-only session's chip
  reads `unfiled`, in gray.
- **Dot:** filled `●` when attributed, hollow `○` when forecast. Its color is the lead
  home's area hue.
- A chip never routes captures, reorders anything, changes a status, or feeds the plan
  budget. Derived homes are never capture `#name` targets: two `sase` sessions stay
  distinct.

### R8 — Hues

- The categorical slots come from the validated dataviz reference palette, in fixed
  order. The palette passes adjacent-pair CVD and normal-vision checks in both modes; it
  was checked with `validate_palette.js` while planning.

  | Slot | Name             | Light      | Dark       | xterm-256 |
  | ---- | ---------------- | ---------- | ---------- | --------- |
  | 1    | `blue`           | `#2a78d6`  | `#3987e5`  | 33        |
  | 2    | `orange`         | `#eb6834`  | `#d95926`  | 208       |
  | 3    | `aqua`           | `#1baf7a`  | `#199e70`  | 36        |
  | 4    | `yellow`         | `#eda100`  | `#c98500`  | 214       |
  | 5    | `magenta`        | `#e87ba4`  | `#d55181`  | 211       |
  | 6    | `green`          | `#008300`  | `#008300`  | 28        |
  | 7    | `violet`         | `#6250d6`  | `#9085e9`  | 62        |
  | 8    | `red`            | `#e34948`  | `#e66767`  | 167       |
  | —    | `gray` (buckets) | text-muted | text-muted | dim       |

  The xterm codes are nearest matches. homes-core may refine them and must record the
  final values in the doc.

- **Assignment.** A top-level area's hue is its own `color:` frontmatter when that names
  a slot.
  - Otherwise the fallback is slot `fnv1a32(lowercased stem) % 8`. This is local and
    deterministic, so the JavaScript and Swift sides need no global knowledge.
  - An invalid `color:` raises the lint `area_color_invalid` and uses the fallback.
  - Sub-areas, projects, and the review bucket take their top-level area's hue.
  - `color:` on any other note is ignored.
- `bob stats` warns `area_hue_collision` when two top-level areas share a hue, with the
  fix-it "set `color:` on one of them".
- **The Areas bar orders segments by slot**, so the hue sequence is always a validated
  adjacency. Slots are therefore assigned in order of how often each area appears; when
  rare areas are missing, the bar just loses its tail. While planning, a 4-week subset
  that skipped a middle slot failed the dark-mode CVD check (aqua next to magenta, ΔE
  1.6). That failure is why the assignment goes by frequency.
- **Bryan's assignment**, pinned by homes-core:
  - `dev` blue
  - `gtd` orange
  - `job` aqua
  - `love` yellow
  - `cash` magenta
  - `body` green
- Assignments are vault data, not code. stats-panel re-validates the subsets the live
  vault actually shows and may re-pin them.
- Text always uses text tokens. Color only identifies the dot or mark.

### R9 — The canonical `homes` JSON object

Every surface emits this one shape. New fields are additive: no `schema_version`
changes, and old consumers ignore it.

```json
"homes": {
  "state": "attributed",
  "kind": "work",
  "label": "bob · dev +1",
  "chip": "bob · dev +1",
  "hue": "orange",
  "parts": [
    {"key": "bob.md", "label": "bob", "kind": "project", "area": "gtd.md",
     "area_label": "gtd", "hue": "orange", "tasks": 1, "share": 34, "minutes": 17}
  ]
}
```

- `state` is `attributed` or `forecast`.
- `kind` is `work`, `review`, or `unlinked`.
- `chip` and `hue` are null when they do not apply.
- `share` is a whole percent of the session: minute-based when attributed, task-based
  when forecast.
- `minutes` appears only when attributed.
- `parts` is empty for an unlinked session.
- A part's `kind` is `project`, `area`, or `bucket`.

### R10 — History follows today's note tree

- Homes are computed from the current notes. Moving a task, re-parenting a project, or
  toggling `rollup` re-homes past minutes; totals never change.
- Every Stats surface says so: "Grouped by today's note tree · from Task Links".
- Snapshots are rejected for v1.

## Phase homes-core — shared resolver, rollup, contract, vault config

**bob-cli: new module `src/native/pomodoro_homes/`.** Split it by concern (facts,
resolve, session, label, palette, json, tests), each file under the house size norms.

- `NoteFacts { kind: Area|Project|Other, parent, rollup: Off|Subprojects|Invalid(raw), color: Option<raw>, daily: bool }`.
  Parse it with the existing `projects::scan` frontmatter helpers and
  `wikilink_path_target`; do not write a new YAML reader.
- A small source trait with two backends:
  - an **indexed** source for Stats, built from the stats pass's in-memory buffers and
    `NoteIndex`;
  - a **disk** source for live surfaces, using `VaultLinkResolver` with an exact-path
    check first and lazy frontmatter reads.
- A memoizing `HomeResolver` over that trait. It returns a `Home` with:
  - key, label, kind;
  - top-level area key and label, subarea, and hue;
  - the **pre-rollup chain**, kept for `in:`.

  Results are memoized per note: about 2.2k links hit about 107 distinct notes.

- `session_links(...)`: the R3 sibling of `list_queued_links`.
- `attribute_session(name, links, minutes, exempt, daily_path, resolver)` →
  `SessionHomes`. It implements R4–R7:
  - kind;
  - per-task homes;
  - per-home parts with tasks, minutes, and share;
  - the lead home, label, chip, and dot hue.

  It also exposes the pure split helper that stats-parts reuses.

- Serde types for the R9 object.
- Lints `rollup_invalid` and `area_color_invalid` (stable codes, once per note), plus
  counters for unfiled, ambiguous, and unresolved links.
- Read the exempt names from the existing `plan.exempt` config loader.

**Conformance vectors:** `tests/fixtures/pomodoro_homes/vectors.json`.

- The file holds one shared mini-vault (raw note text by path) and a list of ledger
  cases. Each case gives a daily path, the `## Pomodoros` text, and the exempt names. It
  expects, per entry line: kind, state, label, chip, hue, and parts (key, tasks, shares,
  minutes).
- Mark JavaScript-incompatible cases (ambiguous basenames) with `"rust_only": true`, as
  the Today vectors do.
- A Rust test runs every case through the indexed source and, where cheap, the disk
  source. Expected values are **computed by hand** from the fixture text, never
  snapshotted.
- Cover at least:
  - **Rollup:** a rolled-up child and grandchild; outermost-wins with nested flags;
    `rollup: yes please` lints and is off; the pre-rollup chain is kept.
  - **Resolution:**
    - a `done/` archive target;
    - a ref note parented to a sub-project;
    - an area-resident task;
    - a closed project;
    - a parent cycle;
    - a `[[love.md|love]]` parent;
    - an ambiguous basename (rust_only), giving `@unfiled` plus a counter.
  - **Review:**
    - a GTD session with foreign struck links;
    - an unnamed `^gtd` bridge session;
    - `GTD + BOB`;
    - a `^gtd` link under a work session.
  - **Splitting:** a 2:1 split with an odd minute count; the same task linked twice; the
    count-mode lead home with a tie.
  - **Chips:**
    - the name equals the home, so the chip is suppressed;
    - a composite name equals the set of homes;
    - an unfiled-only session;
    - an unlinked session;
    - a forecast with no minutes;
    - two `sase` sessions stay distinct.
  - **Hue:** an explicit `color:`; an invalid `color:`, which lints and falls back; an
    FNV fallback vector with its value pinned by hand.

**Docs.**

- Write the normative `docs/pomodoro-homes.md`: R1–R10, the hue table, the JSON shape,
  the vector index, limits, and rejected alternatives (writing names, freezing at close,
  a Mixed bucket, guessing from names, child-side opt-outs).
- Add a "Rollup (reporting only)" section to `docs/projects.md`.
- Add index entries in `docs/README.md` and the root `README.md`.

**`bob projects list`.** A project with `rollup: subprojects` shows a dim `rollup` tag
after its name. An invalid value prints one stderr warning line and leaves the exit code
unchanged. Update the help and docs if they describe the columns.

**Vault (live `~/bob`, git-synced; edit in place following the vault conventions):**

- Add `rollup: subprojects` to `sase.md`'s frontmatter, after `status:`.
- Add `color:` to the six top-level area notes according to R8: `dev.md`, `gtd.md`,
  `job.md`, `love.md`, `cash.md`, `body.md`.
- Then confirm that `bob projects sync`, `bob task-status-hooks`, and the frontmatter
  writers preserve both keys. Run them, or their dry runs, and diff the result.

**Memory.**

> [!decision] memory_decision_record Through `/sase_memory_write`, add a decisions
> strand, "Pomodoro Homes Are Derived, Never Written". It covers labels from Task Links,
> review sessions as their own home, rollup as reporting-only, and history following the
> current tree. Cite the research artifact and this epic. Then run `sase memory init`.

> [!decision] memory_home_term Through `/sase_memory_write`, add a glossary strand,
> `Home`, with the R1/R2 definition and the forecast/attributed distinction. Then run
> `sase memory init`.

**Done when:**

- `just check` passes.
- Every vector passes.
- `bob projects list` shows `sase … rollup`.
- A disk-source smoke test against the live vault resolves today's ledger. The labels
  should match the research example: `BLOG` gets `sase`, and `HACK` gets `bob · dev +1`.
- `just install`.

## Phase stats-parts — area and project parts in `bob stats`

**Index.**

- `StatsIndex` keeps each closed session's dedicated links (struck included, R3) next to
  `PomodoroSession`. It builds the indexed homes source from the buffers it already
  reads; no second vault pass.
- Archive paths keep mapping to their source residence, as they do today.

**Pomodoro metrics** (`worked`, `sessions`, `review_time`, and custom metrics):

- Add parts dimensions `area` and `project` next to `theme`.
- A session's **counted minutes** C are the existing theme-selected shares. The home
  split (R5) distributes C, so every dimension sums to the metric total even under a
  `themes:` filter.
- Review and unlinked sessions map to their buckets.
- Count mode uses the lead home.

**Task metrics** (`done`, `created`, `dropped`, and custom metrics): add `area` and
`project` by residence through the same resolver. There are no review or unlinked
buckets here, but `@unfiled` applies.

**Report** (schema 1, additive):

- A top-level `homes` catalog, keyed by every key used anywhere in the report:
  - fields: `label`, `kind` (project|area|bucket), `area`, `subarea`, `path`, `hue`;
  - `project_label` on area-kind entries (`job · area tasks`);
  - `rolled_up` (the descendant-project count) on rollup homes.
- `PartSummary` gains, for these two dimensions only:
  - an optional dimension-specific `label`;
  - `apportioned` and `apportioned_display`, in minutes mode.
- Points gain an optional `apportioned` map, `{dimension: {key: minutes}}`, omitted when
  empty, so a pinned day can draw its tint.
- `area` and `project` are **not** collapsed in JSON. Both are bounded by the note tree,
  and the panel needs per-area rosters. `note` and `theme` keep their top-five rule.
- Add `health.homes`:
  - in-window counts: `unlinked_minutes`, `unlinked_sessions`, `unfiled_links`,
    `ambiguous_links`, `unresolved_links`;
  - global lists for `rollup_invalid`, `area_color_invalid`, and hue collisions.
- Raise the matching `warning <message> <code>` entries.
- Add the four limits lines to the pomodoro metric definitions:
  - "Attributed from Task Links."
  - "Grouped by today's note tree."
  - "Sessions with several homes are split evenly by task."
  - "Links show what a session was for, not a stopwatch."

**CLI (`render.rs`).**

- Fix `parts_line`. Today it prints only the first `BTreeMap` dimension, which already
  hides `done`'s `note` line.
- Print every dimension, in a stable order:
  - pomodoro: `area`, `project`, `theme`;
  - tasks: `kind`, `area`, `project`, `note`.
- Use catalog labels. Under color, prefix area and project cells with a `●` in the
  area's xterm hue; buckets are dim.
- The project line shows the top five homes plus `other`, and buckets are always shown
  outside those five.
- When apportioned minutes are non-zero, add a dim line:
  ` split    N% of project minutes came from sessions with several homes`.
- Plain output (`NO_COLOR` or a non-TTY) drops the glyphs.

**Fixtures and docs.**

- Add a separate hand-computed homes fixture vault (for example
  `tests/fixtures/stats/homes_vault/`) with its own H-vector table in `docs/stats.md`.
  It covers worked, sessions, and done parts; buckets; apportioned; a theme-filtered
  custom metric; and the sum-to-total check in every dimension.
- Leave M1–M17 and the existing golden vault untouched.
- Update the `docs/stats.md` sections on Data model, Parts, Schema-1, and Human output.
- Reference `docs/pomodoro-homes.md` instead of restating it.

**Done when:**

- `just check` passes.
- The H vectors pass.
- Repeated computes are byte-identical.
- `bob stats show worked` prints area, project, and theme lines.
- `bob stats show done` prints kind, area, project, and note lines.
- **Real-vault check:** with `BOB_NOW=2026-10-18T12:00:00` and
  `-r 2026-09-20..2026-10-17`, compare against the research report's area and project
  lines. Major rows should land within about one percentage point; record any larger gap
  and its cause, since vault edits since Oct 18 can move them.
- **Performance:** release `bob stats -f json -r 4w` on athena stays within about 15% of
  the 174 ms baseline. Record the before and after numbers in the docs' Performance
  section.
- `just install`.

## Phase stats-drilldown — filters and session drill-down data

**Pomodoro filters**, validated like the other spec keys, with local warnings naming
`stats.metrics.<id>.<key>`:

- `in:` and `exclude_in:` match a task's **pre-rollup** chain with the existing
  `note_target_matches` semantics.
  - In minutes mode a session contributes only the shares of its kept tasks. Its theme
    parts are re-split over that contribution so every dimension still sums.
  - In count mode a session counts once if any task is kept, and its lead home is chosen
    among the kept tasks.
  - Review and unlinked sessions never match `in:` and are never removed by
    `exclude_in`.
  - Both filters AND with `themes` and `exclude_themes`.
- `review: include | only | exclude` (default `include`) selects by the R4 predicate.

> [!decision] review_time_predicate The built-in `review_time` becomes
> `{kind: pomodoro, measure: minutes, review: only}`. Recompute vector M10 by hand: an
> Oct 9 `GTD + SASE` composite no longer counts. Update its row and the built-in catalog
> table.

> [!decision] review_time_predicate = no Keep `review_time` on `themes: [GTD]`, and do
> not add the `review:` key.

**Session drill-down.**

- When the report includes at least one pomodoro metric, add a top-level `sessions`
  array, omitted when empty.
- It lists every closed session in the display window, today included, in chronological
  order.
- Each element is
  `{date, path, line, time_range, minutes, minutes_display, quality, name, homes}`,
  where `homes` is the R9 object and its keys are in the catalog.
- These are facts, not metric values: themes and filters do not change them. Measure the
  JSON size for `-r all` and record it.

**Docs and fixtures.**

- Change the `docs/stats.md` config example from `sase_time: {themes: [SASE]}` to
  `in: [sase]`, and add an `in: [sase_blog]` example that still measures blog time under
  rollup.
- Document the `sessions` field in the Schema-1 contract.
- Extend the H vectors for `in:`, `exclude_in`, `review:`, and `sessions`.

**Done when:** `just check` passes, the vectors pass, the performance budget above still
holds, and `just install`.

## Phase live-surfaces — capture-pomodoros, bob plan, bob pomodoro

- **`bob capture-pomodoros --tasks`:** every entry gains the R9 `homes` object through
  the disk source. Open entries get a forecast; completed entries get an attribution.
  The schema stays version 1.
  - The human output appends the chip, dim with its dot, after each entry's name.
  - Update the goldens under `tests/fixtures/capture_pomodoros/`.
  - The plain (non-`--tasks`) JSON lane is unchanged.
- **`bob plan -f json`:** each `entries[]` element gains `homes`. The schema stays
  version 2.
- **`bob pomodoro` (status and tmux):** an **unnamed** running session with a non-null
  chip prints it after the range: `[<13m] 0950-1015 sase`.
  - A named session prints exactly as today and does no home resolution at all.
  - An unnamed session without links prints exactly as today.
  - Add CLI tests next to `tests/cli/pomodoro.rs`.
- **chezmoi** (open it with `/sase_repo`): the Hammerspoon parser already passes
  lowercase text through.
  - Add a README "Pomodoro menu bar" state-table row: "Unnamed session with linked tasks
    | `sase (50m) · 🍅 12:34`".
  - Add `pomodoro_countdown_spec` and `init_spec` cases for that input, and a tmux
    status sample.
  - Run that repo's test suite.
- **Docs:** add `homes` to the capture-pomodoros JSON section of `docs/capture.md`. In
  `docs/plan.md`, document `homes` and add "Names are optional; an unnamed entry still
  shows its derived homes" to Names. Update the `bob pomodoro` section of the README.

> [!decision] memory_mac_pom Through `/sase_memory_write`, update
> `glossary:mac-menu-bar-pomodoro-indicator`: an unnamed session with linked tasks shows
> its derived home label (`sase (50m) · 🍅 12:34`), and `UNTITLED` only appears when
> there is nothing to derive. Then run `sase memory init`.

**Done when:**

- `just check` passes.
- `bob capture-pomodoros --tasks -f json` stays within 15 ms on athena (it is about 8 ms
  today).
- The chezmoi tests pass.
- `just install`.

## Phase ledger-chips — Obsidian ledger home chips (bob-plugins)

Open bob-plugins with `/sase_repo` and follow its AGENTS.md (edit `src/` fragments,
build, never hand-edit `main.js`, keep fragments at or under 1000 lines).

- **JavaScript mirror** in bob-ledger-tools: a pure model fragment and a plugin mixin,
  following the `137-progress-marks.js` / `268-plugin-progress-marks.js` split.
  - It resolves links with `metadataCache.getFirstLinkpathDest`, walks `parent:`
    frontmatter, detects area and project types, and applies rollup and its lint, R4
    (with the bridge per decision), R7, and the R8 hue (including FNV-1a with
    `Math.imul`).
  - It has no minute math beyond R5 shares. Reuse the plan-budget section and entry
    parsers (`planSectionRange`, `planParseEntry`, `splitPlanComponents`) and
    `todayLinkFromBody`-style helpers rather than adding new parsers.
- **Vectors.** Copy `tests/fixtures/pomodoro_homes/vectors.json` verbatim into the
  plugin's test fixtures with a "Copied from bob-cli <rev>" note. Add a `node:test` file
  that builds a fake `metadataCache` from the vector notes, runs every non-`rust_only`
  case, and is registered in `package.json`.
- **The chip.**
  - It is a widget appended at the end of each Pomodoro entry line in **any** canonical
    daily note: a 7px dot in the area hue (filled when attributed, a ring when forecast)
    plus the label in `--text-muted` at the ledger's font size, with a small left gap.
  - It hides no text, so the cursor needs no reveal logic.
  - The tooltip lists the homes with tasks and minutes: "sase · 2 tasks · 15m".
  - It rebuilds on doc and viewport changes and on a debounced metadata-cache change, so
    a frontmatter edit to `rollup` or `parent` refreshes the chips.
  - Add a Reading-view counterpart following the progress marks' `269-…-reading.js`.
- **Tokens.** Define `--bob-area-<slot>` light and dark custom properties from the R8
  table once, at a scope both the chip and the Stats block can use. The chip and the
  Stats panel share these.
- **Toggle.** Add a "Toggle Pomodoro home chips" command and a body class, following the
  progress-marks toggle.
- **Release.** Bump the bob-ledger-tools minor version and update its manifest
  description and the README table (that table is already behind). Run `npm test` and
  `npm run validate`, then `bob plugins sync`.

**Done when:** the vectors pass in JavaScript, and a live check on today's daily note
shows the R7 chips. Look at a screenshot or the rendered DOM in light and dark mode.

## Phase stats-panel — "Where the time went" (bob-plugins)

Replace the `"<label> by theme"` pomodoro context panel (`statsPanelPomodoro`) with a
two-view panel. The header reads "Where the time went · <range> · <total>" with a
segmented `[Areas & projects] Themes` control. The view choice is view state and
persists like the other view toggles. The **Themes** view is exactly today's panel.

**Areas & projects view:**

- **Area bar:** one horizontal stacked bar.
  - Segments run in slot order (R8) in their area hue.
  - There is a 2px surface gap between segments, and rounded ends.
  - Direct labels appear on segments wide enough to hold them.
  - Buckets come last, in gray.
  - It has an `aria-label` that summarizes the split.
- **Grouped table**, ranked by minutes:
  - an area row (dot, label, total, share);
  - beneath it, the top three project rows. Each has a mini-bar scaled to the largest
    project: solid hue for direct minutes, a lighter tint of the same hue for
    `apportioned`.
  - an expandable "N more" row;
  - `sase` notes "includes 75 sub-projects" from `rolled_up`;
  - area-home rows use `project_label`;
  - bucket rows are gray and carry their fix-it copy.
- **Rows:**
  - A project or area label opens its note; Ctrl/Cmd opens it in a new leaf.
  - A separate "sessions" control expands the contributing sessions from `sessions`,
    newest first, capped at 20 with "show all". Each shows date, range, duration, name,
    and shares, and jumps to the daily-note line.
- **Pinned day or bucket:** the panel shows that span from the points' `parts` and
  `apportioned`, and the title names the span.
- **Footer:** "▒ split from sessions with several homes · grouped by today's note tree ·
  from Task Links".
- **Count metrics** (`sessions`) use the same view without tint.

**Task metrics:** below the existing flow panel, the same grouped component titled
"<label> by area & project". It shows counts with no tint.

**Labels everywhere.** Wherever the block prints an area or project key, it uses the
catalog label. That includes chart tooltips, `statsChartParts`, and the table view's Top
parts column.

**Compatibility.**

- A report without `area` or `project` parts (an older bob) shows the Themes view and a
  muted "Update bob for areas & projects".
- Missing `sessions` hides the drill-down control.
- An old plugin with a new bob keeps working, because every addition is additive.

**Fixtures and preview.**

- Regenerate a homes canonical fixture from the bob-cli homes fixture vault with a
  pinned `BOB_NOW`, documented in `scripts/fixtures/stats/README.md`.
- Add a synthetic long-stem case (`job_2026_apply_29`, many areas, all-bucket).
- Add preview cases.
- Run `node scripts/preview-stats-block.cjs`. **Look at** the light/dark × narrow/wide
  matrix for collisions and overflow.
- Run `validate_palette.js` (dataviz skill) on the area subsets the live vault shows for
  4w, 3m, and 1y in both modes. If a present adjacency fails, re-pin `color:` in the
  vault rather than changing code.
- Extend `test-ledger-tools-stats-panels.cjs`.

**Release.** Bump the minor version, update the README table, run `npm test` and
`npm run validate`, then `bob plugins sync`. Update the Obsidian Stats block section of
bob-cli `docs/stats.md` to describe the new panel.

**Done when:** the tests pass, the preview matrix has been reviewed, and `dash_stats.md`
renders the panel from the installed bob on the live vault.

## Phase mac-agenda — Bob Mac Capture shows homes

Open bob-mac-capture with `/sase_repo` and follow its AGENTS.md: never land Swift you
have not compiled.

- **Model.** Add `CaptureAgendaHomes` (state, kind, label, chip, hue, parts), decoded
  with `decodeIfPresent` and safe defaults. Unknown enums map to `.other`. Add it as an
  optional `homes` on `CaptureAgendaPomodoro`. The `schema_version` check is unchanged.
- **Presentation** (`CaptureAgendaPresentation`):
  - `displayName` uses the derived label for unnamed entries, in a secondary text style
    so derived text reads as derived. It falls back to "Untitled Pomodoro" only with no
    label.
  - Named entries add the chip as a muted header accessory with a small hue dot.
  - The fold one-row summary and the Later strip use the same rule.
  - Accessibility labels add "for sase".
  - Apply the same fallback in `RefsTodayLinkPresentation` where entries come from
    `bob plan -f json`.
- **Hues.** Add a Swift palette that mirrors the R8 table for light and dark appearance.
- **Fixtures and tests.** Add fixtures with homes: named and suppressed, named with a
  chip, unnamed with a label, unlinked, and a legacy payload without homes.
- **Verify.**
  - Run `just portable-check` on athena.
  - Compile `Sources/BobMacCapture*` changes on the mac with a scratch copy and
    `just typecheck-tests`.
  - Push and confirm that macOS CI is green.
  - If the mac is offline, say in the final report that the macOS targets are
    unverified.

**Done when:** portable-check passes, CI is green, and the README's Idle agenda section
mentions homes.

## Cross-cutting rules

- **One implementation of meaning.** Rust is the authority. The JavaScript mirror only
  re-derives what the ledger chip needs, under shared vectors. Swift decodes and draws
  and never derives. This follows `decisions:mac-capture-is-a-thin-client` and the
  `docs/plan.md` house rule "Rust implements it; bob-ledger-tools mirrors it".
- **Additive contracts only.** No schema version bumps. Every new field is optional for
  consumers.
- **Nothing writes labels.** The only vault writes in this epic are the R2 and R8
  frontmatter config in homes-core, plus any re-pinning in stats-panel.
- **Unchanged:** the plan budget and its theme cap, capture `#name` routing, blocking,
  Ready caps, review tiers, and the Themes view. Authored names keep every job they have
  today.
- Each bob-cli phase finishes with `just check` and `just install`. Each bob-plugins
  phase finishes with `npm test`, `npm run validate`, and `bob plugins sync`.

## Later (out of scope)

- A `[work::]` per-session override.
- A home-aware plan budget.
- Journal-keyed area and project parts for `reviewed`.
- A "stack by area" toggle on the Worked focus chart.
- An area-colored day ribbon.
- `rollup_except`.
- `rollup` on `bob`.

These all wait for a few weeks of real use. One question is left to Bryan and is not
part of this epic: whether `bob` should stay under the `gtd` area. The new Areas view
will show gtd at about 41%, roughly half morning review and half building Bob.
Re-parenting `bob.md` to `dev` is a one-line vault edit, and its history follows.
