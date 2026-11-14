---
tier: epic
title: Generated "## Areas" catalogs for area notes and the area root
goal: 'Every area note with sub-areas, and the root note `[[area]]` resolves to (today
  `area.md`), carries a bob-owned, nested `## Areas` catalog that stays correct through
  `bob areas sync` and nightly, always sits above `## Projects`, is protected from
  capture, turns the "areas parent only areas" convention into checked findings, and
  renders as a calm, colored map card in Obsidian while staying readable as plain
  Markdown everywhere else.

  '
decisions:
  card_titles:
    ask: How should the Areas card show an area's explicit H1 or alias title?
    choices:
      tooltip: Rows keep the stored stem links; explicit titles appear on hover only
      inline: Titled rows show the title as link text; the stem moves to the tooltip
    default: tooltip
    why: Stems match @route names and the stored bytes, so all 19 rows read alike
    answer: tooltip
  card_auto_sync:
    ask: Should the Areas card run `bob areas sync` by itself when its catalog goes
      stale?
    default: false
    why: Areas change rarely; a stale pill plus nightly avoids surprise background
      commits
    answer: false
  memory_decision_record:
    ask: Add a decision record that area catalogs nest the descendant forest?
    memory:
    - decisions:area-catalogs-nest-the-descendant-forest
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
  memory_glossary_area:
    ask: 'Update the glossary Area Note strand to mention the generated ## Areas catalog?'
    memory:
    - glossary:area-note
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: catalog-engine
  title: Shared catalog engine - AREAS spec, placement order, heading adoption, capture
    guard
  depends_on: []
  size: medium
  description: 'catalog-engine: add the AREAS region spec and the AfterSectionThenFirstH2
    placement, move PROJECTS below an existing Areas section, teach every catalog
    to adopt an authored heading instead of duplicating it, and refuse capture into
    `## Areas`.'
- id: areas-graph
  title: Area graph and pure catalog planner
  depends_on:
  - catalog-engine
  size: medium
  description: 'areas-graph: retain the root note''s bytes in the vault snapshot,
    build the validated area graph (root rule, parent findings, cycles), and plan
    nested callout rows per host with golden tests on the real vault shape.'
- id: areas-report
  title: bob areas list and check
  depends_on:
  - areas-graph
  size: medium
  description: 'areas-report: ship the read-only `bob areas` noun with list/check,
    the versioned JSON report (hues, project and Today rollups, catalogs, findings),
    a colored tree view, registration everywhere commands are listed, and the docs/areas.md
    contract.'
- id: areas-sync
  title: bob areas sync, nightly step, and Highlights coverage
  depends_on:
  - areas-report
  size: medium
  description: 'areas-sync: add the guarded `bob areas sync` writer and dry run, the
    nightly step before projects, the Highlights regression test, integration tests,
    sync docs, and a read-only preview against the real vault.'
- id: areas-card
  title: bob-ledger-tools Areas card shell
  depends_on:
  - areas-report
  size: medium
  description: 'areas-card: in bob-plugins, add the display-only `[!areas]` card -
    report service, Reading and Live Preview painting, family chrome, hue dots, health
    line with sync and retry, and the root card''s unplaced-areas notice - then deploy.'
- id: areas-card-chips
  title: Areas card folds, chips, tooltips, and add-summary
  depends_on:
  - areas-card
  - areas-sync
  size: medium
  description: 'areas-card-chips: add branch folds with per-pane memory, fold-aware
    project and Today chips, row tooltips, the hover-only add-summary affordance,
    and narrow-width polish, then deploy.'
- id: areas-memory
  title: Decision record and glossary update
  depends_on:
  - areas-sync
  size: small
  description: 'areas-memory: record the nested-forest decision and teach the Area
    Note glossary strand about the generated catalog, then regenerate memory.'
proposed_by: bbugyi200.apollo.e2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 16:32:58
status: wip
bead_id: bob-cli-ln
---

- **PROMPT:** [prompts/202611/areas_catalog.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/areas_catalog.md)
- **BEAD:** [bob-cli-ln](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ln/README.md)

# Plan: Generated `## Areas` catalogs

## Context

Bob already keeps three bob-owned "host catalogs" in vault notes, all built on the
shared engine in `src/native/host_catalog/` (`RegionSpec`, byte-span owned regions,
damage handling, guarded transactions): `## Notebooks` (`[!notebooks]`), `## Albums`
(`[!albums]`), and `## Projects` (`[!projects]`, direct child projects with lifecycle
glyphs). Nothing points _down_ the area tree: children name their parent through
`parent:`, but `area.md` and parent areas such as `love.md` (five sub-areas) list none
of them, and hand-written "Related:" lines already drift.

The design research for this feature is
`research:202611/areas_catalog_design/areas_catalog_design.md` (read it with
`sase artifact read research:202611/areas_catalog_design/areas_catalog_design.md "<why>"`).
Bryan approved all of its requirements. This plan adopts them, closes two gaps the
research missed, and pins the contract each phase builds on:

- **Gap 1:** the research says an existing authored `## Areas` heading is reused "by
  engine behavior". It is not: `insert_at` in `host_catalog/region.rs` always writes
  heading plus callout, so an authored heading would be duplicated. Phase
  `catalog-engine` adds heading adoption for every catalog family.
- **Gap 2:** the card's `● today` chip and project counts need data. The card may not
  re-resolve parents in JS, so bob reports both rollups in `bob areas list -f json`.
  That reuses the same Today pipeline and home walk that `bob plan` and `today_roots`
  already use.

Today's vault (checked 2026-11-14) has 19 area notes, all at the vault root, max depth
3, and every parent is valid. `area.md` is the untyped type note: `parent: "[[type]]"`,
**no `type:` field**, no H2. Five notes will gain a catalog: `area.md`, `cash.md`,
`gtd.md`, `inbox.md`, and `love.md`. The other fourteen areas are leaves and stay
untouched. No area has `summary:` yet.

Areas carry no lifecycle and feed no task state. Nothing in surface, reconcile,
freshness, Ready caps, Pomodoro homes, capture targets, or root-parent walks changes.
The root note stays untyped and invisible to those pipelines. Only `src/native/areas/`
and the snapshot retention rule learn what the root is.

## The contract (shared by every phase)

### Hosts and the root

- **Root.** The root is the note that the bare `[[area]]` link resolves to through the
  snapshot's `NoteIndex` (exact `area.md`, then a unique case-insensitive basename),
  outside `done/` and excluded directories. It has no frontmatter requirement. It is a
  terminal sentinel: its own `parent:` is never followed and it is never a row, even
  when a test fixture types it `type: "[[area]]"`.
  - If nothing resolves, report one `area_root_missing` warning. If several notes match,
    report one `area_root_ambiguous` warning.
  - In either case no root catalog is planned, and top-level areas whose parent targets
    `area` get no per-area parent finding: the single root finding covers them.
  - Never create the root file. `type.md` and other type notes never host.
- **Hosts.** The root when it has at least one top-level area, plus every area note with
  at least one valid child edge.
- **Cleanup.** An area note, project note, or the root that holds an owned `[!areas]`
  region but is not a host has that region removed. Examples are a former host whose
  last child left, or an area retyped as a project. The heading goes only when its
  section is then blank. Notes of any other kind are never touched: an `[!areas]`
  callout there is authored content.

### Membership and findings

The children are notes classified `LiteKind::Area` (outside `done/`, excluding the root
itself) whose scalar `parent:` resolves to the root or to another area. Each area
produces at most one parent finding. All parent and cycle findings are errors; none of
them creates a row:

| Code                     | When                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------- |
| `area_parent_missing`    | no `parent:` value                                                                        |
| `area_parent_list`       | list-form or duplicate `parent:` keys (more than one parent)                              |
| `area_parent_unresolved` | target resolves to no note (suppressed for `area` when the root finding fired)            |
| `area_parent_ambiguous`  | target matches several notes                                                              |
| `area_parent_self`       | target is the area itself                                                                 |
| `area_parent_kind`       | target is neither the root nor an area note (a project, notebook, album, or untyped note) |
| `area_cycle`             | the resolved area-to-area walk revisits a note or exceeds 16 hops                         |
| `area_parent_unquoted`   | warning only, when the parent otherwise resolves; mirrors `notebook_parent_unquoted`      |

Aliased (`[[love.md|love]]`), path, and heading link forms resolve the way the Projects
catalog resolves them.

For a cycle, the parent edges _of the cycle members_ are dropped. Edges from non-members
into a member remain, so a bad ancestor never suppresses valid edges beneath it: that
member still hosts its valid children.

Summaries are an optional one-line `summary:` scalar. A missing summary is never a
finding. The two soft warnings `area_summary_long` (more than 140 characters) and
`area_summary_multiline` mirror the notebooks rules. Body prose is never scraped.

Catalog-state findings mirror the notebook and album codes:

- `area_catalog_stale` is a warning that covers both missing and stale catalogs, with
  distinct messages.
- `area_catalog_damaged` is an error that carries the opener's line number.

### Rows

Each host lists its whole descendant area forest, nested. Every row is:

```text
> {two spaces per depth}- **[[link]]**[ — summary]
```

The callout is `> [!areas]`, then a lone `>`, then the rows. That is the same shape the
Notebooks and Projects callouts use.

- **Link.** The note stem, or `path-without-md|stem` when the stem alone is ambiguous.
  Reuse `notebooks::scan::link_destination`, or lift the Projects `link_inner` into a
  shared helper.
- **Summary.** Rendered through `notebooks::catalog::inert_text`.
- **Order.** Siblings sort A–Z by stem, ignoring case, with a path tie-break.
- **Nothing else.** No glyphs, Closed group, counts, dates, titles, or hues. Everything
  visual is drawn by the card, never stored.

The first sync on today's vault writes this to `area.md`, at EOF after its prose:

```markdown
## Areas

> [!areas]
>
> - **[[body]]**
> - **[[cash]]**
>   - **[[bank]]**
>   - **[[recur]]**
> - **[[dev]]**
> - **[[gtd]]**
>   - **[[gtd_daily]]**
>   - **[[inbox]]**
>     - **[[gkeep_inbox]]**
>     - **[[gmail_inbox]]**
>     - **[[mac_inbox]]**
> - **[[job]]**
> - **[[love]]**
>   - **[[abby]]**
>   - **[[family]]**
>   - **[[fun]]**
>   - **[[kelly]]**
>   - **[[vacation]]**
> - **[[meet]]**
```

The other four hosts get the following on first sync:

- `gtd.md`: `gtd_daily`, plus `inbox` with its three trays nested, inserted above its
  existing `## Projects`.
- `love.md`: its five children, inserted above `## Notebooks`.
- `cash.md`: `bank` and `recur`.
- `inbox.md`: its three trays.

### Placement and ownership

- `AREAS` =
  `RegionSpec { callout: "areas", heading: "Areas", head_title: None, placement: BeforeFirstH2, legacy: None, strict_bare: true }`.
- `PROJECTS.placement` becomes the new `Placement::AfterSectionThenFirstH2("Areas")`. A
  first write goes at the end of an existing `## Areas` section, else before the first
  H2, else at EOF. Section order therefore never depends on which catalog came first.
- On a host that gets every catalog at once, the first-write outline is: Areas, then
  Projects, then Notebooks, then Albums, then Tasks.
- **Heading adoption** applies to all four families. Suppose a host has no owned region
  but has a real (unfenced, after frontmatter) H2 titled exactly like the spec's
  heading. Then the first write puts only the callout at the end of that section: after
  its last non-blank line, separated by one blank line, with one blank line before any
  following content. The heading is never duplicated, and the first such heading wins.
- Bob owns exactly the `> [!areas]` callout bytes. Bryan owns the heading, its position
  after the first write, and every other byte.
  - Duplicate well-formed regions self-heal by keeping the first.
  - A malformed `> [!areas` opener is damage: that host is left byte-identical and
    reported, while other hosts still apply.
  - The BOM and CRLF endings survive.
- `GENERATED_CATALOGS = &[AREAS, NOTEBOOKS, ALBUMS, PROJECTS]`. The drawer and merge
  guards follow this list automatically. The capture section refusal table gains the
  `areas_section_generated` row.

### Commands

`bob areas` sits in the Vault section of `bob --help`, alphabetically between `albums`
and `articles`. A bare `bob areas` runs `list`.

- `bob areas [list] [-b DIR] [-f human|json]` is read-only. It prints a colored tree and
  always exits 0.
- `bob areas check [-b DIR] [-f human|json]` is read-only. It prints findings and exits
  1 on any error finding.
- `bob areas sync [-b DIR] [-d] [-f human|json]` is the guarded writer:
  - It takes the shared vault lock and pre-syncs.
  - It plans from one complete snapshot and refuses a partial scan.
  - It installs preimage-guarded edits through `host_catalog::txn`.
  - It verifies by re-planning, then makes one scoped commit
    `areas: sync catalogs (N notes)` and post-syncs.
  - `-d` prints colored diffs and writes nothing. A no-op run commits nothing.
  - Damaged hosts are skipped, and the run then exits 1.

### JSON report (`schema_version: 1`)

`list` and `check` share one envelope. `check` uses `"command": "check"`. `ok` is false
when any error-severity finding exists.

```json
{
  "schema_version": 1,
  "ok": true,
  "command": "list",
  "root": { "path": "area.md", "stem": "area", "state": "found" },
  "areas": [
    {
      "path": "cash.md",
      "stem": "cash",
      "link": "cash",
      "title": "Cash",
      "titled": false,
      "summary": null,
      "parent": "area.md",
      "parent_raw": "[[area]]",
      "depth": 1,
      "top": "cash",
      "hue": "magenta",
      "hue_source": "color",
      "children": ["bank.md", "recur.md"],
      "projects": { "direct": 2, "subtree": 2 },
      "today": { "direct": 0, "subtree": 1 },
      "issues": []
    }
  ],
  "catalogs": [
    {
      "host": "cash",
      "path": "cash.md",
      "kind": "area",
      "state": "current",
      "rows": [
        { "path": "bank.md", "link": "bank", "depth": 0 },
        { "path": "recur.md", "link": "recur", "depth": 0 }
      ],
      "issues": []
    }
  ],
  "findings": [
    {
      "path": "x.md",
      "code": "area_parent_missing",
      "severity": "error",
      "line": 2,
      "message": "..."
    }
  ],
  "counts": {
    "areas": 19,
    "top_level": 7,
    "hosts": 5,
    "errors": 0,
    "warnings": 0,
    "catalogs": { "current": 5, "missing": 0, "stale": 0, "damaged": 0 }
  }
}
```

**`root` and `areas[]`:**

- `root.state` is `found`, `missing`, or `ambiguous`; `root.path` is null unless the
  state is `found`.
- `areas[]` is in tree order (depth-first from the root, siblings A–Z), followed by
  unplaced areas A–Z.
- `depth` is 1 for top-level areas, and `depth` and `top` are null for an area with no
  valid chain to the root. `parent` is the resolved path of a valid parent, else null.
- `title` comes from `notebooks::scan::derive_title`. `titled` is true only when that
  title came from an H1 or an alias rather than the stem.

**Hues:**

- `hue` and `hue_source` come from `pomodoro_homes::HomeResolver::resolve_note` over the
  snapshot, the same way the notebooks scanner resolves hues. `hue_source` is `color`,
  `muted`, or `hash`.
- Clients render the hue bob reports. The root has no hue.

**`projects` rollup:**

- `direct` counts non-terminal project notes whose nearest area on their parent walk is
  this area. The walk passes through projects, notebooks, and albums, with a 16-hop cap.
- `subtree` adds the counts of all descendant areas.

**`today` rollup:**

- It counts the distinct Today tasks: the open tasks that
  `plan_budget::today::today_tasks` finds under today's daily note, using the same
  "today" and daily-path resolution as `bob plan`.
- Each task is attributed to the nearest area on the walk
  `plan_budget::roots::today_roots` uses (`HomeResolver::home_for_link`). A valid
  `[root:: X]` starts that walk at X, and archive and ref wrappers pass through.
- `direct` is this area's own count; `subtree` includes descendants. A missing daily
  note yields zeros. Today data is report-only and never written.

**`catalogs[]`:**

- There is one record per host or cleanup note, sorted by path.
- `kind` is `root`, `area`, or `project`. `rows[].depth` is relative to the host (0 is a
  direct child).

### Nightly, capture, Highlights

- **Nightly.** An `areas` step runs immediately before `projects`, giving the order:
  vault-sync, task archive, articles, period notes, areas, projects, notebooks, albums,
  vault-sync.
- **Capture.** `bob capture '@love#areas' …` refuses with `areas_section_generated`.
- **Highlights.** PDF note snapshots already project unknown callouts generically with
  their kind. Lock that in with a regression test: `[!areas]` projects as
  `kind="areas"`, with the nested rows intact.

### The card (beauty layer, bob-plugins `bob-ledger-tools`)

The card is display-only. It styles the stored rows, reads `bob areas list -f json`,
never edits catalog text, and never re-resolves parents in JS. Without the plugin
(mobile, git, agents, PDFs) the callout is ordinary readable Markdown.

- **Chrome.** It uses the same card family as `[!projects]` and `[!notebooks]`: border,
  radius, hue wash, container queries, nested-list rail. The callout icon is
  `lucide-layers`.
  - An area host washes in its own reported hue.
  - The root card stays neutral, so its rows carry the rainbow of top-level colors.
- **Marker.** Each row carries a **solid dot** in that child's reported hue. Projects
  use a ring because a project can finish; an area never does. A muted hue paints a grey
  dot.
- **Chips.** Rows get at most two calm, muted chips, drawn and never stored:
  `N projects` and `● today`.
  - An expanded row shows `direct` counts.
  - A folded row shows `subtree` counts, because a folded row speaks for its whole
    branch.
  - Zero-valued chips are hidden. Task details live in the tooltip, not the row.
- **Header and health.**
  - The header shows a muted row count.
  - A missing or stale catalog shows `stale catalog · ↻ sync`, which runs
    `bob areas sync -f json`.
  - A damaged catalog shows its line and message and never offers a write.
  - A failed report shows Retry and keeps the last good rows.
  - The root card adds `⚠ N areas unplaced` when parent or cycle findings exist; its
    tooltip lists them.
- **No phantom cards.** A note gets a card only when the report lists it as a catalog
  host. It gets no `＋ area` button, because bob has no area-creation flow.

### Out of scope

The following are not part of this epic:

- `bob areas create`, an area capture sigil, or any creation button.
- Writing `summary:` lines into Bryan's notes; that remains his call.
- Mac Capture changes, config pills, and generalizing to other type notes such as
  `project.md`.

## Phase `catalog-engine`

Work in `src/native/host_catalog/region.rs` (re-export `AREAS` from
`host_catalog/mod.rs`), `src/native/capture/sections.rs`, and their tests.

1. Add `Placement::AfterSectionThenFirstH2(&'a str)` to `insertion`. It resolves to the
   existing section's end when another heading follows, to EOF when the section runs to
   EOF, and to the first H2 (or EOF) when the section is absent.
2. Set `PROJECTS.placement = AfterSectionThenFirstH2("Areas")` and update its doc
   comment.
3. Add the `AREAS` spec and put it first in `GENERATED_CATALOGS`.
4. Implement heading adoption in `plan_host`'s `Missing` branch for every spec, exactly
   as the contract states. Keep edits byte-span-based, so BOM, CRLF, and missing final
   newlines are handled. Update any existing test that encoded the old duplicate-heading
   behavior, and say so in the commit body.
5. Add the `areas_section_generated` row to `reject_generated_catalog_selector`. Its
   message is: "`## Areas` is the generated sub-area catalog (written by
   `bob areas sync`) and is not a section target; pick another section, or capture into
   one of its areas".

Tests:

- Placement in both orders:
  - Areas first, then a first Projects write: Projects lands after the Areas section.
  - Projects first, then a first Areas write: Areas lands before `## Projects`.
- An authored `## Overview` stays above both catalogs.
- A moved, current region gets no edit.
- An Areas section running to EOF makes Projects append after it.
- Adoption cases, for AREAS, NOTEBOOKS, and PROJECTS:
  - a heading with prose
  - an empty heading
  - a heading immediately followed by the next H2
  - EOF without a final newline
  - CRLF with a BOM
  - a fenced `## Areas` ignored
  - two authored headings, where the first is adopted
- A malformed `> [!areas` opener is damaged; the lookalike `[!areas-card]` is authored
  content.
- The capture refusal for `@host#areas`.
- The drawer and merge guard tests extended to the fourth heading.

Docs:

- In each catalog contract that states first-write placement (`docs/projects.md` §
  Projects catalogs, `docs/notebooks.md` § The catalog, and the albums and records
  catalog bullets in `docs/records.md`), add one sentence on adopting an authored
  heading.
- In `docs/projects.md`, state the new "after `## Areas`" rule.
- In `docs/capture.md`, list `areas_section_generated` beside its siblings.

Gate: `just check`.

## Phase `areas-graph`

Add the new module `src/native/areas/` (`mod.rs`, `graph.rs`, `catalog.rs`), registered
in `src/native.rs`, and change `src/native/notebooks/snapshot.rs`.

1. **Snapshot retention.** `SnapshotFile::from_contents` also retains `contents` for any
   note whose file stem equals `area` ASCII-case-insensitively. That covers every
   possible `[[area]]` target, including in the no-frontmatter early return. Update the
   module doc comment. This also puts the root's bytes into
   `ReadDependencies::from_snapshot`, which is intended.
2. **Graph** (`graph.rs`, pure over `&VaultSnapshot`):
   - Root resolution comes from `snapshot.index().resolve_detailed(None, "area")`.
   - Area nodes carry path, stem, link, title and `titled`, summary, and the parent edge
     (reuse the notebook scanner's parent-edge facts: multiple, ambiguous, missing,
     unquoted).
   - Children are kept sorted.
   - Cycle detection uses a visited set plus the 16-hop cap.
   - Compute `depth` and `top`, and emit every finding in the contract with line numbers
     from `parent_line`.
3. **Planner** (`catalog.rs`, pure):
   - Hosts and cleanup candidates follow the contract.
   - Rows are rendered depth-first, then passed through `host_catalog::plan_host_region`
     with `AREAS`.
   - It produces
     `CatalogPlan { hosts: Vec<HostPlan { path, stem, kind, state, rows: Vec<RowRecord { path, link, depth }>, issues, edit: Option<FileEdit> }> }`,
     sorted by path, with an `edits()` accessor shaped like the notebooks plan.

Tests (in-memory snapshots, golden bytes):

- **Real vault shape.** A fixture mirrors today's 19 areas and the untyped root. Exact
  post-images for the five hosts match the contract, and the 14 leaves get no edit.
- **Root.**
  - The real-shape root hosts.
  - A synthetic typed `area.md` (`type: "[[area]]"`, `parent: "[[area]]"`) hosts and is
    never a row.
  - `type.md` never hosts.
  - A root `area.md` wins over `sub/area.md`.
  - With no root, `area_root_missing` fires once with no per-area noise, and area hosts
    still plan.
  - Ambiguous candidates without an exact `area.md` produce `area_root_ambiguous`.
  - The root's body is retained.
- **Parents.**
  - The aliased, path, and heading forms resolve.
  - Each finding code fires.
  - A two-node cycle fires `area_cycle`.
  - A 17-deep chain fires `area_cycle`.
  - Valid children below a broken ancestor still render in that ancestor's own catalog.
- **Links and summaries.**
  - An ambiguous stem renders as `path-without-md|stem`.
  - Summaries are inert-escaped (`[[x]]`, `#tag`, `**`, newlines).
  - The long and multiline warnings fire.
- **Reparenting.** Moving `kelly` under `family` changes `love.md` and `area.md` and
  creates `family.md`'s catalog.
- **Cleanup.**
  - A host that loses its last child loses the callout. The heading is dropped only when
    its section is blank.
  - A retyped area-to-project note sheds its region.
  - A notebook note with `[!areas]` is untouched.
- **Damage.** A malformed opener damages only that host.
- **Idempotence.** Re-planning the post-image yields no edits.

Gate: `just check`.

## Phase `areas-report`

1. **`src/native/areas/report.rs`.** Build the JSON envelope from the contract over one
   snapshot.
   - It covers graph, plan, hues, project rollups, and Today rollups.
   - Unreadable files follow the `bob notebooks list` handling.
2. **`src/native/areas/output.rs`.** Renders the human views:
   - **`list`.** Header `Areas · 19 areas · 5 catalogs`, then the tree rooted at the
     root's stem, drawn with `├─ └─ │` connectors.
     - Each row shows a `●` in the area's hue (reuse the existing hue-to-ANSI styling
       used for homes and Today roots; muted is dim) and the bold stem.
     - After the stem come the dim explicit title (when `titled`), the dim ` — summary`,
       and dim chips `N projects` and `● today` (direct counts, hidden at zero).
     - Areas with no valid chain to the root are listed after the tree under an
       `Unplaced` heading, each with its finding code.
     - The footer is one of: `✓ 5 catalogs current`; a yellow
       `! N catalogs stale · run bob areas sync`; or a red damaged line with
       `path:line [code] message`. A findings summary follows, pointing at
       `bob areas check`.
     - `NO_COLOR` and non-TTY output are plain.
   - **`check`.** Findings grouped by path: errors as `✗` in red, warnings as `!` in
     yellow, each `path:line [code] message`. Then a summary
     `19 areas · 0 errors · 0 warnings`, or `✓ areas match the contract` when clean.
3. **`src/native/areas/cli.rs`.** Clap commands following `notebooks/cli.rs`. Options
   are sorted alphabetically, and every long option has a short alias (`-b`, `-f`,
   `-h`). Write excellent `long_about` and examples text.
4. **Registration.** Every place `notebooks` appears:
   - `NativeCommand` variants and dispatch in `src/native.rs`
   - the `Subcommand` plus `AREAS_MEMBERS` and `AREAS_AFTER_HELP` in `src/runner.rs`,
     Vault section, default `list`
   - `src/native/completion/{kinds.rs,tree.rs}`
   - command-mode rows `areas-list` ("Area tree") and `areas-check` ("Check areas") in
     `src/native/catalog/table.rs`, with tests
   - any command-inventory test the build enforces
5. **Docs.** Follow the docs memory routes.
   - Create the new contract page `docs/areas.md` covering: the root; hosts and
     membership; findings (the code table); the catalog (rows, order, placement,
     adoption, ownership, damage, cleanup); Commands; JSON.
   - Add it to `mkdocs.yml` next to each `notebooks.md` entry, including the
     CLI-reference nav.
   - In `docs/reference/vault-format.md` § Area notes, document the optional `summary:`,
     the generated catalog, and the root-host rule.
   - Add `docs/reference/json-outputs.md` rows for list and check.
   - Run `just docs-gen` for `docs/reference/cli/areas.md`, then `just docs-check`.

Tests:

- Report unit tests for:
  - tree order
  - unplaced areas
  - hue sources (color, muted, hash, and a null root)
  - the project rollup (sub-projects count toward their nearest area; terminal projects
    excluded)
  - the Today rollup (an area-resident task, a project-resident task counted at the
    project's area, a `[root:: meet]` daily-note Event Task counted at `meet`, and no
    daily note)
- CLI tests in `tests/cli/areas.rs` for:
  - help text
  - bare `bob areas` running list
  - golden JSON on a small fixture
  - `check` exit codes (0 clean, 1 on an error finding or a damaged catalog, 0 with only
    warnings)

Gate: `just check`.

## Phase `areas-sync`

1. **`src/native/areas/sync.rs`.** Mirror `notebooks/sync.rs` with these pieces:
   - `dry_run`, `run_standalone` (via `host_catalog::run_locked(..., "area", ...)`),
     `sync_with_existing_lock`, and `nightly_step`
   - `PlannedTxn`, with:
     - files from `staged_rewrites`
     - deps from `ReadDependencies::from_snapshot`
     - verify by re-loading, re-planning, and requiring every written host to be
       current. This also catches a same-stem note appearing mid-run and flipping a
       parent's resolution, and it rolls back with `verify_failed`.
   - The subject `areas: sync catalogs (N notes)`.
   - The sync JSON envelope, shaped like notebooks sync: `schema_version`, `ok`,
     `command`, `dry_run`, `changed`, `changes` (with `diff` only on a dry run),
     `catalogs`, `findings`, `commit`, `sync`, `warnings`, and `error` on failure. The
     human output is the notebooks wording with "area".
2. **CLI and registration.**
   - Add the `sync` subcommand and member.
   - Add the completion entry.
   - Add the command-mode row `areas-sync` ("Sync area catalogs"), `PreviewFirst` with
     `DRY_RUN`.
3. **Nightly.** In `src/native/nightly.rs`:
   - Add the `areas` step immediately before `projects`.
   - Update its help text, the `nightly` about string in `src/runner.rs`, and the
     command-mode `nightly` confirm text. That text is already stale; give it the full
     step order.
4. **Highlights.** Add a regression test under
   `src/native/highlights_ref/note_snapshot/` asserting that a note with a nested
   `[!areas]` callout serializes `kind="areas"` with its nested list intact. Fix only if
   it fails.
5. **Integration tests** in `tests/cli/areas.rs`, with a real-vault-shaped fixture
   vault:
   - First sync writes the five hosts in one commit with the exact subject.
   - A second sync is byte-identical with no commit.
   - `-d` writes nothing and its JSON carries diffs.
   - A damaged host is skipped while the others apply, and the run exits 1.
   - The reparent case lands in one commit.
   - Placement end to end in both orders:
     - `areas sync`, then a first child project plus `projects sync`: `## Projects` ends
       below `## Areas`.
     - The reverse order gives the same result.
   - A host modified between planning and install is refused, using the existing txn
     test seams.
   - `bob nightly` output includes the areas step, if nightly is testable in the
     existing harness.
6. **Docs.**
   - Add the Sync and Nightly sections to `docs/areas.md`.
   - In `docs/vault-git-sync.md`, add `bob areas sync` to the lock-sharing list and the
     step to the nightly step list.
   - Add the sync row to `docs/reference/json-outputs.md`.
   - Run `just docs-gen` and `just docs-check`.
7. **Real-vault preview (read-only).**
   - Run `cargo run -q -- areas sync -d -b ~/bob` and
     `cargo run -q -- areas list -b ~/bob`.
   - Confirm that exactly the five hosts change and that `area.md` matches the
     contract's preview.
   - Record any discrepancy on the phase bead.
   - Never run a real sync against `~/bob`: athena's nightly applies it once the release
     train ships this.

Gate: `just check`.

## Phase `areas-card`

Open bob-plugins with `/sase_repo` and read its `AGENTS.md` first. All work is in
`plugins/bob-ledger-tools/`. Model every piece on the Projects card:
`src/241aa`–`241ae`, the `scripts/test-ledger-tools-projects-*.cjs` tests, and the
Projects block in `styles.css`.

1. **Fragments**, listed in `src/fragments.json` with each fragment at 1000 lines or
   fewer:
   - **Areas data**: `areasListArgs` = `["areas","list","-f","json"]`, decoding that
     rejects any `schema_version` other than 1, and indexes by host path and area path.
   - **Areas data service**: a stale-while-revalidate report through `bobSpawnJson`,
     with a generation counter and subscribers. Revalidation is debounced at 300 ms,
     with a 60 s rollover check.
     - Invalidation sources are metadataCache `changed`, vault create, modify, delete,
       and rename, focus and visibility, and file-open and active-leaf changes.
     - A relevance predicate limits invalidation to area notes, the root, project notes,
       and today's daily note.
     - `apiAreasSync` runs `bob areas sync -f json`, then forces a fresh list.
   - **Catalog model**: pure functions for row models, hue slots, the health model, and
     the status model.
   - **DOM**: painting reuses the notebook and album helpers (`notebookCreateEl`,
     `albumReplaceChrome`, and so on).
   - **Mixin**: a Reading view post-processor (priority 52), a CM6 Live Preview
     extension with a status widget like `createProjectsCatalogExtension`, and setup and
     teardown in `170-plugin-lifecycle*.js` and `310-install-methods.js`.
   - **A test exports fragment.**
2. **Rendering** follows the contract's card section:
   - family shell and hue wash from the host's reported hue (the root is neutral)
   - the solid `.bob-areas-dot` per row from the row's reported hue, via
     `var(--bob-area-<slot>)` tokens in light and dark themes, plus the `lucide-layers`
     icon
   - a header with the row count
   - the health and status line: stale pill with `↻ sync` and `Syncing…`, damaged text,
     Retry with last-good rows, and `⚠ N areas unplaced` on the root
   - no card on non-host notes, and no failure banner when bob is unavailable

   > [!decision] card_auto_sync = no The card never syncs by itself; only the `↻ sync`
   > click runs `bob areas sync`.

   > [!decision] card_auto_sync Auto-sync like the Projects card: when the catalog is
   > stale for a new fingerprint, `apiAreasSync` runs after a 3 s debounce
   > (`projectsScheduleSync` pattern), at most once per fingerprint, and never for a
   > damaged catalog.

3. **Tests.** Add `scripts/test-ledger-tools-areas-data.cjs` and
   `scripts/test-ledger-tools-areas-catalog.cjs` (decode guard, relevance predicate, row
   model, hue mapping, health and status models, DOM paint, sync action, no phantom
   card), then add both to `test:unit` in `package.json`.
4. **Release.**
   - Minor-bump `manifest.json` and update its description.
   - Add a `## Areas catalog card` section to `docs/bob-ledger-tools.md`.
   - Run `npm run docs:gen` and `npm test`.
   - Commit, then deploy with `bob plugins sync`, following AGENTS.md's refusal rules
     (never `-F`).
   - The installed `bob` gains `areas` only after the release train; until then the card
     shows its Retry state, which is expected.

## Phase `areas-card-chips`

This phase works in the same plugin and files as `areas-card`.

1. **Folds.**
   - Rows with children get a `▸`/`▾` toggle button before the dot, expanded by default.
   - Fold state is kept in memory per workspace leaf, note path, and row path. It
     survives repaints and is never persisted to disk or the vault.
   - Transitions respect `prefers-reduced-motion`.
2. **Chips.**
   - Rows show right-aligned muted `N projects` and `● today` chips (the dot in the
     row's hue).
   - Expanded rows use `direct` counts and folded rows use `subtree` counts. Zero values
     are hidden.
3. **Tooltips.** A tooltip on the row's dot and chips shows:
   - the explicit title when `titled`
   - the route `@stem`
   - "N projects · M tasks today", direct and subtree when they differ

   > [!decision] card_titles = tooltip Visible link text stays the stored stem on every
   > row.

   > [!decision] card_titles = inline Titled rows replace the visible link text with the
   > title (the link target stays the stem) and the tooltip shows `@stem`. Untitled rows
   > keep the stem.

4. **Add summary.** Rows without a summary show a quiet `add summary` affordance only on
   hover or focus-within, so day one is not 19 chips. It reuses the notebooks
   open-and-focus-`summary` code path (`apiNotebookOpenSummary`,
   `notebookFocusSummaryProperty`) and never writes.
5. **Narrow widths.** Under a narrow container, the summary wraps below the stem and the
   chips compact to number-plus-icon.
6. **Tests, docs, and deploy.** Extend the card tests for folds, fold-aware counts,
   tooltip text, and the hover-only affordance. Update the card docs section, minor-bump
   the manifest, run `npm test`, commit, and run `bob plugins sync`.

## Phase `areas-memory`

Use `/sase_memory_write` and edit only what the accepted memory decisions cover. Then
run `sase memory init`.

> [!decision] memory_decision_record Add a new `decisions` web strand with slug
> `area-catalogs-nest-the-descendant-forest` (selector
> `decisions:area-catalogs-nest-the-descendant-forest`), in the existing record format.
>
> - **Frontmatter.** It has a keyword, aliases, a one-line summary, and
>   `metadata.status: accepted` with `decided` set to the landing date.
> - **Applies to.** bob-cli, bob-plugins, and the vault.
> - **Claim.** Area catalogs list each host's descendant area forest, nested. Hosts are
>   area notes with sub-areas plus the note `[[area]]` resolves to. Rows hold only links
>   and optional summaries, while counts, hues, and Today are drawn by the card.
>   `## Areas` is placed above `## Projects`. Lifecycle catalogs such as Projects stay
>   flat.
> - **Why.** Cite the research ref above and the landing commits.
> - **Rejected alternatives.** Direct-only rows, Dataview or Bases, merging into
>   `## Projects`, ⚠️ orphan rows, and a hard-coded `area.md` filename.
> - **Cost.** Reparenting rewrites up to the ancestor chain.
> - **Reopen** when the tree outgrows a screen (a few dozen areas or deeper) or areas
>   gain a lifecycle.

> [!decision] memory_glossary_area In the glossary strand `glossary:area-note` (Area
> Note), add one sentence beside the Albums sentence: every area with sub-areas, and the
> root note `[[area]]` resolves to, lists its descendant areas, nested, in a
> machine-owned `## Areas` catalog maintained by `bob areas sync`. Also note that an
> area may carry an optional one-line `summary:`.

## Verification

- Every bob-cli phase passes `just check`. Phases that touch docs also pass
  `just docs-check`.
- bob-plugins phases pass `npm test` and deploy with `bob plugins sync`.
- The `areas-sync` phase records the read-only real-vault dry run, showing exactly the
  five expected hosts.
- After the release train ships, athena's nightly writes the catalogs. Bryan can preview
  them earlier with `bob areas sync -d`.
