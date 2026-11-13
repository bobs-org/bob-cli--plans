---
tier: epic
title: Sub-projects move into a
goal: 'Every area and project note lists its direct child projects in one bob-owned
  `> [!projects]` callout under a `## Projects` heading Bryan owns, rendered by Bob
  Ledger Tools as a card in the notebook/album family. The `🧩 **Sub-projects:**` bullet
  under `^prj` is retired, and no reader ever mistakes a parent for finished while
  it still has open children.

  '
decisions:
  sync_lifecycle:
    ask: How should a real `bob projects sync` run install its writes?
    choices:
      locked: Lock, pre-sync, guarded write, one commit, post-sync; fails when vault-sync
        can't run
      guarded: Lock, guarded write, one commit, no vault-sync cycles; offline cron
        runs still write
    default: locked
    why: Same lifecycle as notebooks and albums sync, so the card's ↻ behaves the
      same everywhere
    answer: locked
  memory_decision_record:
    ask: File a new decisions strand recording the accepted sub-project catalog design?
    memory:
    - decisions
    default: true
    requested: Review the subprojects_catalog_section.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file
    answer: false
  memory_capture_glossary:
    ask: Correct the Project Note Capture glossary claim that sync adds the Sub-projects
      line?
    memory:
    - glossary:project-note-capture
    default: true
    corrects: refuses the capture, and `bob projects sync` later adds the note to
      the parent's
    answer: true
phases:
- id: catalog-guards
  title: PROJECTS region spec and generated-heading guards
  depends_on: []
  size: small
  description: 'catalog-guards: add the PROJECTS RegionSpec and make every generated
    catalog heading (Notebooks, Albums, Projects) a never-drawer, never-section-target
    heading, fixing the live #notebooks drawer bug.'
- id: subopen-reader-rust
  title: Rust catalog row grammar and dual-format SUBOPEN reader
  depends_on:
  - catalog-guards
  size: medium
  description: 'subopen-reader-rust: add the shared row grammar/parser, read SUBOPEN
    from the [!projects] callout with legacy-marker fallback in surface.rs, and publish
    shared parity vectors.'
- id: subopen-reader-plugins
  title: Plugin dual-format SUBOPEN reader and conversion guard
  depends_on:
  - subopen-reader-rust
  size: medium
  description: 'subopen-reader-plugins: mirror the reader rule in task-status-cycler
    against the byte-identical vectors, keep nav conversion from copying the callout,
    and deploy before any writer ships.'
- id: catalog-planner
  title: Pure sub-project catalog planner and renderer
  depends_on:
  - subopen-reader-rust
  size: medium
  description: 'catalog-planner: compute hosts, membership, row states, cleaned criteria,
    ordering, and per-host postimages (callout plus legacy-line removal) as a pure,
    tested planner.'
- id: highlights-snapshot
  title: Highlights snapshot renders the projects catalog
  depends_on:
  - subopen-reader-rust
  size: small
  description: 'highlights-snapshot: render [!projects] in Highlights PDFs with state
    chips, omitting the Closed group, while legacy notes keep their masthead row.'
- id: projects-list-json
  title: bob projects list -f json report
  depends_on:
  - catalog-planner
  size: medium
  description: 'projects-list-json: add the versioned JSON report (projects, catalogs,
    findings) that the card consumes, plus a human catalog footer.'
- id: catalog-sync-writer
  title: Guarded projects sync writes the catalog and retires the line
  depends_on:
  - catalog-planner
  - subopen-reader-plugins
  size: large
  description: 'catalog-sync-writer: compose lifecycle and catalog edits per file,
    install them through the host-catalog transaction, delete the legacy-line writer,
    add -f json, dry-run diffs, and a nightly step.'
- id: catalog-at-birth
  title: Capture writes the parent catalog row at birth
  depends_on:
  - catalog-sync-writer
  size: medium
  description: 'catalog-at-birth: project-note captures stage the parent''s catalog
    row in the same all-or-nothing batch, refusing a damaged parent catalog.'
- id: projects-catalog-card
  title: Bob Ledger Tools projects catalog card
  depends_on:
  - projects-list-json
  - catalog-sync-writer
  size: large
  description: 'projects-catalog-card: paint [!projects] as a hued card with live
    chips, a folded Closed group, sync and capture affordances, auto-sync, and e2e
    screenshots.'
- id: rollout-wrapup
  title: Docs sweep, decision record, and vault preview
  depends_on:
  - highlights-snapshot
  - catalog-at-birth
  - projects-catalog-card
  size: small
  description: 'rollout-wrapup: sweep remaining Sub-projects docs, record the decision
    and glossary memory, and dry-run the real vault for Bryan''s migration review.'
proposed_by: bbugyi200.athena.1c6
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 10:58:41
status: wip
bead_id: bob-cli-ld
---

- **PROMPT:** [prompts/202611/subprojects_catalog_section.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/subprojects_catalog_section.md)
- **BEAD:** [bob-cli-ld](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ld/README.md)

# Plan: Sub-projects move into a `## Projects` host catalog

Source research:
`research:202611/subprojects_catalog_section/subprojects_catalog_section.md` (read it
with `sase artifact read`). Bryan accepted every requirement it recommends, including
both open items: area notes get catalogs in v1, and the closed ledger is derived from
the vault rather than preserved and marked. This plan turns that design into phases.
Where the research left a detail open, the contract below settles it.

## Why this shape

Today `bob projects sync` keeps one machine-owned bullet under each open `^prj`:

```markdown
- [ ] #task #prj Ship the parent outcome! #hide ^prj
  - 🧩 **Sub-projects:** 🗓️ [[alpha]] • [[beta]] • ~~[[gamma]]~~ ✅
```

On `sase.md` that line is 1.8 KB with 61 links. It also serves as the SUBOPEN cache that
`projects/surface.rs` and task-status-cycler read to decide whether a parent still has
open children. That decision drives `#hide` surfacing, the close-gesture pull-in, and
the successor offer.

The new home is the third member of the existing host-catalog family (`## Notebooks`,
`## Albums`). Bryan owns the heading; bob owns one callout maintained by the
`host_catalog` engine.

We reject three alternatives:

- **One heading per child.** It collides with drawers, `@route#section`, and the
  outline.
- **Bob-owned H3 groups.** They break the "bob owns only the callout" rule, and the
  outline would churn as groups empty and fill.
- **Mirroring child tasks into the parent.** Every task has exactly one residence.

The "H3 sub-sections" instinct is honored in two ways:

- Grouping lives inside the callout as a bold **Closed** group that the card folds, the
  same way an album's `[!records]` index groups rows.
- Bryan may add his own H3s under `## Projects`, outside the callout, and bob never
  touches them.

## The contract (shared by every phase)

### Markdown

```markdown
## Projects

> [!projects]
>
> - **[[sase_agent_recovery]]** — Failed, killed, or interrupted agents can be recovered
>   in one action!
> - **[[sase_agents_repo]]** — Use `sase—agents` repos to sync sase agents across
>   machines!
> - 🗓️ **[[sase_decks]]** — Ship the deck pipeline!
> - ⛓️ **[[sase_post_blog]]** — Plan features to launch after sase_blog post!
> - ⚠️ **[[sase_scratch]]** — no valid `^prj` task
> - **Closed**
>   - ✅ **[[sase_agent_history]]** — Migrate agent artifacts tab to "Agents" tab!
>   - ❌ **[[sase_back_policy]]** — Develop architecture policy that auto-deprecates
>     backward compatibility code!
```

Afterward the `^prj` task is a clean lifecycle task: criterion, Depends-On line, and
logs. It has no Sub-projects child.

### Hosts and membership

- **Hosts.** A host is any area note or project note, of any status, that has at least
  one direct child project. When a host's last child leaves, its callout is deleted. The
  `## Projects` heading is deleted only if its section is then blank; this is existing
  `host_catalog` behavior.
- **Children.** A child is a project note whose scalar `parent:` resolves to the host
  through `project_dependencies::resolve_project_parent` over a vault
  `vault_links::NoteIndex`.
  - Use the same note set `bob projects sync` walks, so excluded directories and `done/`
    archives never contribute.
  - Resolution rules: an exact path wins; otherwise only a unique case-insensitive
    basename resolves. List-form, ambiguous, unresolved, and self parents produce no
    row.
  - These are the rules the blocking cascade already uses, so catalog membership and
    inherited blocking can never disagree.
  - The host must be an area note or a project note.
- **Depth.** Direct children only, with no rollup. A `root: true` child is still listed
  in its parent's catalog.

### Row grammar

Each row is `> - <glyph> **<link>** — <criterion>`. Closed rows sit two spaces deeper
under a `> - **Closed**` group row. The leading glyph is the row's state. Readers key on
the glyph, never on group membership.

| Glyph    | Meaning                    | Derived from (the effective post-sync child state)                                                                                                                                | SUBOPEN |
| -------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| _(none)_ | open and ready             | open child, `^prj` not `[?]`, no future date                                                                                                                                      | open    |
| 🗓️       | open, scheduled later      | the effective `^prj` schedule is after today (inline date, or the frontmatter value being hoisted; the same rule as today's `child_future_scheduled`)                             | open    |
| ⛓️       | open, blocked              | the `^prj` checkbox is `[?]` with no future date (its own Depends-On, or inherited and tagged `#prj_is_blocked`); the catalog reads reconcile's mark and never re-derives Blocked | open    |
| ⚠️       | open, unreadable lifecycle | a non-terminal child with a missing, malformed, or duplicate `^prj`; its text is the literal ``no valid `^prj` task``                                                             | open    |
| ✅       | done                       | `SubprojectState::Done`                                                                                                                                                           | closed  |
| ❌       | canceled                   | `SubprojectState::Canceled`                                                                                                                                                       | closed  |

**Glyph bytes.**

- Writers emit `🗓️` (U+1F5D3 U+FE0F), `⛓️` (U+26D3 U+FE0F), `⚠️` (U+26A0 U+FE0F), `✅`
  (U+2705), and `❌` (U+274C).
- Readers accept each glyph with or without U+FE0F.
- The separator is a space, an em dash (U+2014), and a space.

**Closed rows use a leading glyph, not strikethrough.** Every state then sits in the
same leading slot, raw Markdown stays quiet, and the parse rule is glyph-only.

**Links.** A row links `[[stem]]`, using the child's stem in its original case, when
that resolves uniquely to the child. Otherwise it links
`[[<vault path without .md>|<stem>]]`, mirroring the notebook catalog's `group_link`. No
title alias: stems are what Bryan types in capture routes.

**Criterion.** The criterion is the child's `^prj` description, cleaned in this order:

1. Drop `#task`, `#prj`, and every other `#tag`.
2. Drop inline fields (`[k:: v]` and `(k:: v)`), Tasks emoji date fields, the trailing
   block ID, and the Keep `[💡](…)` link.
3. Reduce `[[x|y]]` to `y`, `[[dir/x]]` to `x`, and `[t](url)` to `t`.
4. Collapse whitespace.
5. Neutralize any remaining Markdown openers with the notebook catalog's
   `notebooks::catalog::inert_text`.
6. Omit the ` — …` tail when the result is empty, is the template placeholder
   `<short_project_completion_criteria_goes_here>`, or equals the stem
   (case-insensitive).

There is no length cap; the card clamps.

**Order.**

- Open rows come first, glyph or not, sorted A–Z by stem case-insensitively with the
  path as tie-break. A state change therefore never moves a row.
- The `**Closed**` group comes last, and only when it holds a row. Its rows are newest
  first by the child's closure date: `^prj` `[completion::]` or `✅ date`, and
  `[cancelled::]` or `❌ date`, read with the shared task-field parsers. Undated rows go
  last, A–Z.

**Closed ledger.** The closed ledger is derived from the vault: every closed direct
child appears. This retires today's "a hand-deleted closed entry stays pruned" rule; the
card's default fold is the new way to hide closed rows.

**Never in Markdown.** Counts, dates, hues, blocked reasons, and task numbers are never
written into the file. The catalog's bytes change only on structural change: a child is
created, closed, reopened, reparented, blocked or unblocked, crosses the future-schedule
boundary, or has its criterion edited.

### Ownership, placement, damage

- **Ownership.** bob owns exactly the `> [!projects]` callout lines. The `## Projects`
  heading, its position, prose, and authored H3s belong to Bryan. Recognition ignores
  frontmatter and fenced examples. BOM, CRLF/LF, and EOF-newline state survive (all
  inherited from `host_catalog`).
- **First placement.** The spec uses `Placement::BeforeFirstH2`. On `sase.md` the note
  then reads: `^prj` → H1 → Projects → Notebooks → Tasks. That is goal → parts →
  knowledge → work. After the first write, the position is Bryan's.
- **Duplicates.** Duplicate callouts self-heal by keeping the first.
- **Damage.** The spec sets `strict_bare: true`, so a malformed `[!projects` opener is
  damage.
  - A damaged region refuses that host with `projects_catalog_damaged`.
  - It is never rewritten by guessing.
  - It is never read as an empty child set.

### The reader rule (SUBOPEN), identical in Rust and JS

`SUBOPEN(N)` is computed in three steps:

1. **Catalog present.** If N's body holds any real `[!projects]` callout, the catalog
   decides, and the legacy marker is ignored. SUBOPEN is true when any region has an
   open row whose key differs from `closed_child`'s stem.
   - A row is a quote line whose content is a list item containing a wikilink.
   - Its key is the first wikilink's target with alias and heading dropped, reduced to
     the last path segment and lowercased.
   - It is closed if and only if its leading glyph is ✅ or ❌. Every other row is open,
     including ⚠️ rows.
   - The `**Closed**` group row has no wikilink and is ignored.
2. **No catalog.** Read today's legacy marker exactly as now.
3. **Damaged region.** Rust `project_surface*` returns `None`: N is not plannable, the
   same as a file with a sync issue, so there is no pull-in. JS treats it as open. Both
   are fail-safe.

Sync's `#hide` surfacing uses the same membership and counts ⚠️ children as open, so the
writer and the readers agree.

Byte-identical parity vectors live at bob-cli
`tests/fixtures/project_catalog/vectors.json` and bob-plugins
`plugins/task-status-cycler/scripts/fixtures/project_catalog/vectors.json`. This follows
the `task_mentions/vectors.json` precedent.

## Rollout rules (every phase)

1. **Readers before writers.** `catalog-sync-writer` depends on both reader phases, and
   the plugin reader phase deploys with `bob plugins sync`. Rust readers ship in the
   same binary as the writer.
2. **No real writes to Bryan's vault.** No phase runs a real `bob projects sync`,
   `bob nightly`, or capture against Bryan's vault (`~/bob`). Dry runs (`-d`,
   `--dry-run`) are fine. The first real write happens when the Mac cron
   (`bob projects sync` at :05/:20/:35/:50) or athena's nightly runs an upgraded binary.
3. **Daily release train.** bob-cli changes ride the daily release train
   (`decisions:bob-cli-releases-ride-a-daily-train`). Never edit the `Cargo.toml`
   version or `CHANGELOG.md`, and mark CLI/JSON contract breaks with `!`.
4. **Legacy recognizers stay forever.** Keep the Rust ones: `SUBPROJECTS_MARKER_PREFIX`,
   scan's `is_marker`, `subproject_displays_in_marker_line`, `display_for_wikilink`,
   `task_relations::parse::is_subprojects_line`, and `task_children`'s `SubProjects`.
   Keep the JS ones in cycler, ledger-tools mentions and marks, and nav. Old backups,
   `done/` copies, and lagging hosts must keep parsing.
5. **Gates.**
   - bob-cli: `just check`, plus `just docs-check` when docs change.
   - bob-plugins: open it with `/sase_repo` and read its `AGENTS.md`. Run
     `npm run build` and `npm test`, then `bob plugins sync`.
   - bob-plugins docs live in its `docs/`; bob-cli's `docs/reference/plugins/*.md` are
     mounted copies.
6. **Unused code.** The `-D unused` gate rejects unused items. Code that lands before
   its caller uses `#[cfg_attr(not(test), allow(dead_code))]`, as notebooks did, and the
   wiring phase removes the attribute.

## Phase catalog-guards — PROJECTS spec and generated-heading guards

This phase is bob-cli only and ships on its own merits.

- **The spec.** In `src/native/host_catalog/region.rs`, add
  `PROJECTS: RegionSpec<'static>` with callout `projects`, heading `Projects`,
  `head_title: None`, `Placement::BeforeFirstH2`, `legacy: None`, and
  `strict_bare: true`. Re-export it from `host_catalog/mod.rs`. Add a
  `GENERATED_CATALOGS: &[RegionSpec]` list holding `[NOTEBOOKS, ALBUMS, PROJECTS]`.
- **Drawer guard.** `capture_drawers.rs::is_structural_h2_key` must treat every
  generated-catalog heading as structural. This fixes the verified live bug where
  `@proj+prj#notebooks` commits a drawer bullet directly under a `[!notebooks]` callout.
  Task merge (`task_merge/containers.rs`, `merge_section_reserved`) and transfer assist
  inherit the fix through that function; prove it with tests.
- **Section targets.** In `capture/sections.rs`:
  - `is_bullet_section` excludes every generated-catalog heading. This feeds section
    completion and `non_tasks_section_headings`.
  - Fold `reject_generated_notebooks_selector` and `reject_generated_albums_selector`
    into one `reject_generated_catalog_selector` driven by a per-spec `(message, code)`
    table.
  - Notebook and album messages and codes stay byte-identical.
  - Projects uses code `projects_section_generated` with this message: "`## Projects` is
    the generated sub-project catalog (written by `bob projects sync`) and is not a
    section target; pick another section, or capture into one of its projects". The
    caller remains `capture/batch.rs` (the `CaptureKind::Bullet` arm).
- **Tests.**
  - Placement tests (`capture/tests/placement.rs`) for the `#projects` refusal.
  - Drawer regression tests for `#notebooks`, `#albums`, and `#projects` on a `^prj`:
    the reserved-drawer refusal fires and nothing is written under the callout.
  - A merge-section-reserved case.
- **Docs.**
  - The generated-section paragraph in `docs/capture.md` (around the notebooks/albums
    refusal text) lists all three catalogs.
  - Update `docs/notebooks.md` §section protection.
  - Update any error-code list that names the notebook/album codes.

## Phase subopen-reader-rust — row grammar and dual-format SUBOPEN

- **Read API.** Expose a read-only region API from `host_catalog`:
  `owned_regions(spec, text) -> Result<Vec<OwnedRegion>, RegionDamage>`.
  - Each `OwnedRegion` carries its start and end lines and its line texts.
  - Build it on the existing `Scanned::owned_regions`, so frontmatter, fence, and
    `strict_bare` handling are identical to the writer's.
- **Row grammar module.** Add `src/native/projects/catalog_rows.rs`, the single home of
  the row grammar.
  - Glyph constants (accept with or without FE0F).
  - `CLOSED_GROUP_LABEL = "**Closed**"` and the separator.
  - `CatalogRowState { Open, Scheduled, Blocked, Warning, Done, Canceled }` with
    `is_open()`.
  - `parse_catalog(text) -> CatalogRead { Absent, Damaged(RegionDamage), Rows(Vec<CatalogRow>) }`.
    Each row holds its key, raw link target, state, and 1-based line.
  - `catalog-planner` adds `render_row` here later, so grammar and parser stay in one
    file.
- **SUBOPEN.** In `projects/surface.rs`, compute `subopen` and `open_subprojects` from
  one helper that applies the reader rule above: catalog first, legacy fallback, damaged
  → `None`, with the `closed_child` override for both formats.
  - Update the module docs, which today say "Sub-projects come from R's sync-maintained
    marker line".
  - Callers (`task_complete/{project_pull_in,successors,keep_moving}.rs` and the capture
    trigger (c) preloads) need no API change; confirm with their existing tests.
- **Parity vectors.** Create `tests/fixtures/project_catalog/vectors.json`. Each case is
  `{name, text, closed_child?, expect: {source: "catalog"|"legacy"|"none"|"damaged", open: N}}`.
  Cover:
  - every current legacy PP6 case: open, struck without a marker, ✅, ❌, fenced, and
    prose mentions;
  - each catalog glyph, including ⚠️ as open;
  - a mixed note where the catalog wins;
  - an empty callout;
  - a callout inside a fence or frontmatter;
  - the `[[dir/x|Alias]]` and `[[x#h]]` link forms;
  - `closed_child` matching in both formats;
  - a malformed `> [!projects` opener (damaged);
  - two regions (union);
  - CRLF;
  - glyphs without FE0F.

  A Rust test iterates the file. Keep the JSON stable: the plugin phase copies it
  byte-for-byte.

- **Docs.** Rewrite the SUBOPEN definition in `docs/task-dependencies.md` §12.10 Terms
  to the dual-read rule, naming the vectors.

## Phase subopen-reader-plugins — cycler dual-read and conversion guard

This phase lands in bob-plugins; open it with `/sase_repo`.

- **Shared helper.** In task-status-cycler, add one helper fragment, e.g.
  `src/089a-project-catalog-rows.js`, registered in `src/fragments.json` and kept at or
  below 1000 lines.
  - It mirrors `catalog_rows.rs`: region scan (≤3-space `>` opener, case-insensitive
    `[!projects]`, quote-line continuation, fences and frontmatter ignored), row parse,
    glyph state, damage.
  - It exposes `projectChildCache(lines)` and
    `projectHasOpenChild(lines, prjLine, options)`. Rows are matched against
    `options.closedChildPaths` by resolved stem, exactly as today.
- **Callers.** `089-successor-project.js` (`projectMarkerHasOpenChild` →
  `projectSurfaceState`) and `085b-successor-offers.js`
  (`computeProjectFoldedAndHeaders`, the copied inline parse) both call the helper. The
  duplicate parse disappears. Legacy parsing stays inside the helper as the fallback.
  Damage counts as open.
- **Vector parity.** Copy the vectors byte-identically to
  `plugins/task-status-cycler/scripts/fixtures/project_catalog/vectors.json`. Add a
  `scripts/test-task-status-cycler-project-catalog.cjs` that runs every vector and
  asserts the file's bytes equal the bob-cli copy's recorded SHA-256, stated in the
  test. Extend `test-task-status-cycler-project-pull-in.cjs` with catalog-only and mixed
  notes.
- **Nav conversion.** Audit bob-navigation-hotkeys project conversion and reversal
  (`212-project-conversion.js`, `665-plugin-project-conversion.js`).
  - The `[!projects]` callout, and its heading when otherwise blank, is generated
    content, like the Sub-projects child: it must never be copied into a task, a
    destination note, or `## Tasks`.
  - Add a reversal test with a catalog-bearing project.
  - `086-relation-line-plan.js` (LINKS anchor) and `256-task-split-plan.js` keep their
    legacy-only recognizers unchanged.
- **Release.**
  - Bump each touched plugin's manifest version per repo convention.
  - Update the plugin docs (`docs/task-status-cycler.md` SUBOPEN wording).
  - Run `npm run build` and `npm test`, then `bob plugins sync`.
  - Refresh bob-cli's mounted docs copy if its docs tooling requires it.

## Phase catalog-planner — pure planner and renderer

This phase is bob-cli. Add `src/native/projects/catalog.rs`. It reads nothing from disk,
takes no locks, and prints nothing.

- **Input.** The scanned project and area notes with their contents, the vault
  `NoteIndex`, and `today`. Extend the sync walk's collection so area notes are captured
  too.
- **Model.** For each child, record:
  - path, stem, link, and `CatalogRowState`;
  - criterion and `closed_on`;
  - data the JSON report needs: the `^prj` line, mark, hidden flag, effective schedule,
    `waits_on` targets parsed from the `^prj` Depends-On line with
    `task_dependencies::parse`, and whether `^prj` carries `#prj_is_blocked`
    (inherited).

  State comes from `SubprojectState::from_project` (effective post-sync status), plus
  the glyph rules. A `None` state on a non-terminal child is ⚠️.

- **Render.** Add `catalog_rows::render_row`. The callout is
  `canonical_callout(&PROJECTS, [">", rows…])`, with the `**Closed**` group appended per
  the contract.
- **Legacy removal.** `remove_legacy_subprojects_lines(text)` deletes every direct-child
  Sub-projects marker line under the single `^prj`, with its subtree, as byte spans that
  preserve EOLs. With zero or several `^prj` lines it removes nothing.
- **Per-host plan.** `plan_host_catalog(text, rows) -> HostCatalogPlan` applies legacy
  removal, then `host_catalog::plan_host(&PROJECTS, …)`.
  - Its record holds the host, path, kind (`area`/`project`), state
    (`current|missing|stale|damaged`), and row counts (open, blocked, warning, closed).
  - A damaged host returns its text untouched, with its legacy line kept as the reader
    fallback.
  - A host with no children but a stale callout or legacy line plans the removal.
  - Also expose `children_by_host` for sync's `#hide` rule.
- **Tests.**
  - Every rendered row parses back through `parse_catalog` to the same key and state
    (round trip).
  - A criterion-cleaning table using real shapes: sase's `[sase](https://sase.sh)`
    criterion, a Keep `💡` link, `[fresh::]`/`[keeps::]`/`[dependsOn::]` fields, the
    placeholder, and a duplicate stem.
  - Ordering, including stable rows across glyph flips and closed rows newest-first with
    undated last.
  - Membership: exact path vs basename, ambiguous, self, list-form, and `done/`
    excluded.
  - Host set: area, terminal project, and zero children removing the callout while
    keeping an authored-prose heading.
  - Legacy removal composing with an insert in one postimage.
  - BOM, CRLF, and no-final-newline preservation.
  - A golden sase-like host with 40+ children, including a Closed group.

## Phase highlights-snapshot — the catalog in Highlights PDFs

This phase is bob-cli, in `src/native/highlights_ref/note_snapshot/`.

- **Render.** Render a `[!projects]` callout as a linked block:
  - Keep open rows with their criterion.
  - Convert the leading state glyph into the existing `Chip` kinds rather than raw
    emoji: 🗓️ → scheduled, ⛓️ → blocked, ⚠️ → a warning chip such as `needs ^prj`. The
    legacy masthead already does this for its markers.
  - Omit the `**Closed**` group and its rows, matching how snapshots omit closed tasks.
  - When no open rows remain, omit the callout, and let the existing empty-section
    handling decide the heading.
- **Legacy masthead.** Notes that still carry only the legacy line keep today's masthead
  "Sub-projects" relationship row (`bob.rs` recognizer kept).
- **Tests.** Add snapshot tests beside the existing `bob.rs` and `transclusion.rs`
  fixtures: catalog-only, legacy-only, and an all-closed host.

## Phase projects-list-json — the versioned report

This phase is bob-cli. Read `sase/memory/cli_rules.md` first.

- **Flag.** Add `-f|--format human|json` to `bob projects list`, defaulting to `human`,
  with options sorted.
- **JSON v1.** Follow the notebook report pattern (`notebooks/model.rs::to_json`):

  ```json
  {
    "schema_version": 1,
    "ok": true,
    "command": "list",
    "today": "2026-11-13",
    "projects": [
      {
        "path": "sase_post_blog.md",
        "stem": "sase_post_blog",
        "status": "wip",
        "state": "blocked",
        "criterion": "Plan features…",
        "parent": "sase.md",
        "children": ["…"],
        "closed_on": null,
        "prj": { "line": 12, "mark": "?", "hidden": true, "scheduled": null },
        "blocked": { "kind": "dependency", "waits_on": ["sase_blog.md"] },
        "tasks": { "open": 20, "shown": 0, "today": 1 }
      }
    ],
    "catalogs": [
      {
        "host": "sase",
        "path": "sase.md",
        "kind": "project",
        "state": "current",
        "rows": { "open": 22, "blocked": 1, "warning": 0, "closed": 38 },
        "issues": []
      }
    ],
    "findings": [],
    "counts": { "projects": 110, "catalogs": 21, "errors": 0, "warnings": 0 }
  }
  ```

  - `state` is one of `open|scheduled|blocked|warning|done|canceled`.
  - `blocked` is null, `{kind: "scheduled", date}`, `{kind: "dependency", waits_on}`, or
    `{kind: "inherited"}`.
  - `children` lists direct child paths in catalog order.
  - `tasks.open` is bob's open `#task` count, not the `open_task_count` frontmatter,
    which undercounts Blocked tasks.
  - `tasks.today` counts open residents with a dedicated live Task Link under today's
    open Pomodoros, via `plan_budget::today::today_tasks`. It is `null` when today's
    daily note is missing or unreadable.
  - Unknown values are `null`, never `0`.

- **Human output.** The human list stays as is, plus one dim footer line when catalogs
  need repair, e.g. `2 Projects catalogs need bob projects sync`. A damaged catalog is
  an issue (exit 1).
- **Docs and catalog.**
  - Add "Project reports {#project-reports}" to `docs/reference/json-outputs.md`, and
    delete its "`bob projects` … human-only" sentence.
  - Update the `projects-list` row in `src/native/catalog/table.rs` (JSON format).
  - Regenerate the CLI reference with `just docs-gen`.
- **Tests.** In `tests/cli/projects/list.rs`: the envelope, each state, `blocked` kinds,
  `today` with and without a daily note, and catalog states.

## Phase catalog-sync-writer — guarded sync writes the catalog

This phase is bob-cli. Commit it as `feat(projects)!:` because sync now writes a new
vault shape and has a new install lifecycle.

- **Membership and `#hide`.** Planner membership replaces
  `subproject_children_by_parent_link_name`. `has_open_subprojects` means any open row,
  ⚠️ included.
- **One postimage per file.**
  1. Apply the existing lifecycle changes (status, `#hide`, schedule hoist and
     propagation) via `apply_project_changes`.
  2. Apply legacy removal and the catalog region, through `plan_host_catalog` on the
     post-lifecycle text.
  - Areas get only step 2.
  - The catalog and legacy step runs for every host, including terminal or unplannable
    project hosts. Lifecycle edits keep today's `can_plan` gating.
  - A damaged host skips step 2, keeps its lifecycle edits, reports
    `projects_catalog_damaged` with its line, and exits 1.
- **Delete the legacy writer.**
  - The six `ProjectChange::*Subproject*` kinds and their events.
  - `edits.rs` `sync_subprojects_line_edits`, `subprojects_insert_anchor`,
    `render_subprojects_line*`, and `sorted_subproject_entries`.
  - `sync.rs` `desired_subproject_entries`, `subprojects_need_normalization`, and the
    anchor helpers that only served the writer.
  - Keep every recognizer (rollout rule 4).
- **Install.** A file changed on disk after planning aborts the whole install with
  nothing written, and the next run retries. Nothing ever uses raw `fs::write`. The
  lifecycle around that guarded install depends on the reviewer's answer:

> [!decision] sync_lifecycle = locked Real runs go through `host_catalog::run_locked`:
> identity check, bounded lock, worktree check, pre-sync, plan, guarded preimage-checked
> install of every changed file, one scoped commit (`projects: sync …`, the notebooks
> subject style), then post-sync. Outside a Git worktree the run fails with
> `project sync writes require a Git worktree`.

> [!decision] sync_lifecycle = guarded Real runs take the maintenance lock and the
> worktree check, then call `host_catalog::apply_and_commit` with no vault-sync cycles.
> Add a small sibling of `run_locked` for this. The regular vault-sync cron publishes
> the commit.

- **Dry run.** `-d` takes no lock and needs no Git. It prints the per-project event
  lines as today, then a colored unified diff for each file that would change, then
  `dry run: N notes would change; nothing was written`. Hoist notebooks'
  `unified_diff`/`color_diff` into `host_catalog` for reuse.
  - Add catalog events: written, updated, or removed per host, plus a legacy line
    retired.
  - A repeat run with unchanged inputs produces no diff and no commit.
- **JSON.** Add `-f|--format human|json` on sync. The envelope mirrors
  `bob notebooks sync`: `schema_version`, `ok`, `command: "sync"`, `dry_run`, `changed`,
  `changes[{path, action, diff?}]`, `events`, `catalogs`, `findings`, `commit`, `sync`,
  `warnings`, and `error`. The card's ↻ uses it.
- **Nightly.** Add a `projects` step ("Sync project lifecycles and sub-project
  catalogs") before `notebooks`, as a lock-free adapter like
  `notebooks::sync::nightly_step`. Update:
  - `nightly.rs` `help_text()`;
  - the `runner.rs` about-string;
  - `docs/reference/cli/nightly.md`, `docs/reference/cli/index.md`, and
    `docs/vault-git-sync.md`;
  - the help fixtures.
- **Help and docs.**
  - Rewrite the `bob projects` and `sync` `long_about`.
  - Rewrite `docs/projects.md` §Sync Rules to the contract: hosts, membership, rows,
    order, the derived ledger (hand-pruning retired), damage, `#hide` with ⚠️, and the
    install lifecycle.
  - Update `docs/vault-format.md`, and regenerate the CLI reference.
  - Update the `projects-sync` catalog row.
- **Tests.**
  - Move `tests/cli/projects/sync.rs` onto Git-initialized vaults, reusing an existing
    `git_vault` helper.
  - Rewrite marker expectations as catalog expectations.
  - Add cases:
    - Legacy line → catalog in one write, never neither.
    - Terminal and area hosts.
    - A hand-pruned closed child returns.
    - A ⚠️ child keeps the parent hidden.
    - Idempotent repeat.
    - A concurrent edit aborts the install.
    - Lock busy.
    - Not a worktree.
    - Dry-run diff.
    - JSON.
    - The nightly step.
  - Update `tests/cli/notebooks_catalog.rs` (`projects_sync_never_lists_a_notebook_…`)
    and the projects unit tests. Legacy-fixture tests elsewhere stay green through the
    readers.

## Phase catalog-at-birth — capture writes the parent row

This phase is bob-cli.

- **Stage the parent row.** In `capture/project_note.rs::plan_project_note_item`, after
  staging the new note, plan the parent host with the planner over the batch overlay.
  - The overlay is the vault's project and area notes plus staged files, so same-item
    `- ^id` / `- :id` / `- ^^` moves out of the parent compose.
  - Stage the parent postimage, catalog plus legacy removal, with
    `planner.stage_snapshot_postimage(abs, before, after)`.
  - Only the parent preimage is guarded. Siblings are read unguarded, because a
    concurrent sibling edit can only leave a row stale, and the next sync heals it.
  - Refuse with `projects_catalog_damaged` when the parent region is damaged. Nothing is
    written, and the message names the line.
- **Other creators.** Grep for other bob creators of project notes
  (`render_project_note` callers) and stage their parents the same way. Nav promotion
  ("Create project note from task") is covered by the card's auto-sync, not here.
- **Previews.**
  - `bob capture --dry-run` and the preview list the parent edit.
  - JSON changes stay additive. Bob Mac Capture is a thin client and needs no change;
    keep its contract tests green.
- **Text.**
  - Replace the success hint "'bob projects sync' adds the parent's Sub-projects line"
    (`capture/output.rs`) with one naming the parent's Projects catalog.
  - Replace the capture `--help` line in `capture/cli.rs` the same way.
  - Update `tests/fixtures/note_pull/np-move/human.txt` and
    `tests/cli/capture/project_note.rs`.
- **Tests.**
  - Create and offer forms write the note and the parent row in one batch.
  - A legacy-line parent migrates.
  - A damaged parent refuses with nothing written.
  - Moves compose.
  - The dry run shows both files.
- **Docs.** `docs/capture.md` "Project notes" and `docs/reference/capture-syntax.md`
  "Project captures".

## Phase projects-catalog-card — the Bob Ledger Tools card

This phase lands in bob-plugins, in ledger-tools. It is a sibling of the notebook and
album cards; reuse their DOM, hue, and fold helpers instead of copying them.

```text
╭ 🧩 Projects ─────────────────────────────── 22 open · 1 blocked · 38 closed ╮
│ ○ sase_agent_recovery  Failed, killed, or interrupted agents…   3 tasks  ● today │
│ ○ sase_agents_repo     Use sase—agents repos to sync agents…    5 tasks  ▸ 2     │
│ 🗓 sase_decks           Ship the deck pipeline!                  Dec 8 · in 3w   │
│ ⛓ sase_post_blog       Plan features to launch after…           waits on sase_blog │
│ ⚠ sase_scratch         no valid ^prj task                                        │
│ ▸ Closed · 38   ✓ sase_agent_history · ✕ sase_back_policy                        │
╰─────────────────────────────────────────────────────────────────────────────────╯
```

- **Data.** A projects data service, following the `241l`/`241m` album pattern:
  - `bob projects list -f json` and `bob projects sync -f json` via `bobSpawnJson`;
  - a shared in-flight request, generation invalidation, and a last-good cache;
  - subscribe and report APIs.
- **Auto-sync.** Debounced, as for notebooks. It fires when a project-note fingerprint
  changes: a project file is created, deleted, or renamed; its frontmatter `type`,
  `parent`, or `status` changes; or a `^prj` checkbox changes.
  - This is how a nav-promoted project or a hand-edited `parent:` is listed within
    seconds.
  - The plugin never writes vault bytes.
- **Card.**
  - Reading post-processor plus a Live Preview ViewPlugin on
    `[data-callout="projects"]`, class `bob-projects-catalog`.
  - Root-ancestry hue via `notebookHostHueName` / `data-root-hue`, with the notebook
    card's tint, border, and radius tokens.
  - The redundant callout title is hidden.
  - The header shows the lucide `puzzle` icon and "Projects", with the tally
    `N open · N blocked · N closed` right-aligned. Blocked counts 🗓️ and ⛓️ rows.
- **Rows.**
  - **State mark:** a hue ring for ready, then calendar-clock, chain, alert-triangle,
    check, and x.
  - **Name and criterion:** the bold stem link (an H1 title as a tooltip when one
    exists), then the muted criterion clamped to one line (full text on hover and
    focus).
  - **Right chips:** these are live and never written.
    - `N tasks` (`tasks.open`; omitted when unknown).
    - `● today` (`tasks.today > 0`).
    - `empty` (the child's `^prj` is surfaced, `prj.hidden == false`).
    - The blocked reason: `Dec 8 · in 3w`, `waits on sase_blog`, or `parent blocked`.
    - A `▸ N` sub-projects expander: a read-only nested list from `children`, held in
      view state only.
- **Closed group.**
  - A fold header `Closed · N`, folded by default with a two-row peek of the newest
    closures.
  - Space and Enter toggle it, and it carries `aria-expanded`.
  - Fold state lives per pane view (a WeakMap, like `albumRecordsFoldByView`).
  - A background refresh never resets folds or steals focus.
  - An all-closed host shows only the header and the folded group.
- **Affordances.**
  - `↻ sync` appears when the report calls this host's catalog missing or stale. It runs
    sync, then revalidates.
  - `＋ project` copies the `@<host>^` capture starter, following the album card's
    `＋ noun`.
  - A failed report shows last-good data and Retry, never fake zeros.
  - At ~375 px, chips wrap under the criterion and the tally abbreviates.
- **Scope the 🧩 recognizers.** The existing 🧩 "project-group" recognizers in the
  notebook and album cards stay scoped to their own callouts.
- **Tests.**
  - Unit tests: `scripts/test-ledger-tools-projects-catalog.cjs` and
    `-projects-data.cjs`.
  - A real-Obsidian e2e `scripts/obsidian-projects-ui-e2e.cjs`, with an offline
    self-test wrapper in `npm test`.
  - Screenshots in light and dark, at ~375 px and desktop width.
  - Fold persistence across refresh, and byte preservation (painting never changes the
    note).
  - List it in `docs/develop/validation.md`.
- **Release.**
  - Add a "Projects catalog card" section to `docs/bob-ledger-tools.md`, and refresh the
    bob-cli mounted copy.
  - Bump versions.
  - Run `npm run build` and `npm test`, then `bob plugins sync`.

## Phase rollout-wrapup — docs, memory, and the vault preview

- **Docs sweep (bob-cli).** Replace every remaining "Sub-projects line" statement with
  the catalog, or mark it legacy-only where readers still accept it.
  - `docs/task-dependencies.md`: the managed-line table, MC7, MC8, MC16, PDS14, and BR8.
  - `docs/task-relations.md`: the LINKS anchor is now legacy-only.
  - `docs/task-link-marks.md`, `docs/keep-moving-marks.md`, and `docs/notebooks.md` (its
    "never on the Sub-projects line" sentence).
  - `docs/pomodoro-homes.md`, `docs/getting-started.md`, `docs/stats.md`, and
    `docs/guides/see-task-mentions.md`.
  - Run `just docs-gen` and `just docs-check`.
- **Memory.** Use `/sase_memory_write`, then `sase memory init` and `just docs-gen`.

> [!decision] memory_decision_record File a new `decisions` strand, "Sub-projects Live
> In A Host Projects Catalog" (slug `subprojects-live-in-a-projects-catalog`).
>
> - **Applies to:** bob-cli, bob-plugins, and the vault.
> - **Claim:** one bob-owned `[!projects]` callout under Bryan's `## Projects` heading
>   per area or project host. It holds direct children only, a leading-glyph state, and
>   a derived closed ledger. Readers prefer the catalog and never infer empty from
>   damage.
> - **Rejected alternatives:** the `^prj` line, per-child headings, bob-owned H3s, live
>   queries, task mirrors, nesting, and preserve-and-mark.
> - **Evidence:** the research ref, this plan, and the landing commits.
> - No existing strand is superseded.

> [!decision] memory_capture_glossary In `glossary:project-note-capture`, replace
> "`bob projects sync` later adds the note to the parent's Sub-projects line" with: the
> same batch lists it in the parent's `## Projects` catalog.

- **Vault preview.** Build bob and run `bob projects sync -d` against Bryan's vault.
  This is read-only. Record a summary on this phase's bead for Bryan's migration review:
  - the host count (about 21: 10 open projects, 6 done projects, 5 areas);
  - any `projects_catalog_damaged`;
  - confirmation that every diff only adds a callout and removes a legacy line.

  Never run a real sync (rollout rule 2).

## Verification (epic-level)

- **Lifecycle parity.** Open, future-scheduled, dependency-blocked, done, canceled,
  reopened, and terminal hosts behave as today. One open or ⚠️ child suppresses the
  finished offer. Closing the last open child yields the same offer, never an auto-link
  (`decisions:unblocked-tasks-are-offered`).
- **Source safety.**
  - Legacy-only, catalog-only, and mixed notes resolve identically in Rust and JS
    (vectors).
  - Damaged regions never imply empty.
  - Authored prose, fenced examples, BOM, CRLF, EOF state, and moved headings survive.
  - A repeat sync is a no-op.
- **Guards.** Catalog headings refuse capture section targets, drawers, and merges.
  Conversion neither swallows nor duplicates generated content.
- **Rendering.** Sixty-plus children stay readable at ~375 px. Zero children leaves no
  empty panel. Keyboard folds and per-pane fold state work. The refresh-failure fallback
  works. Light and dark both render.

## Out of scope

- Nesting grandchildren in Markdown. Depth shows only through the card's `▸` expander.
- Multiple parents per project.
- Bob-owned headings other than the first-write `## Projects`.
- Changing reconcile's Blocked derivation.
- Any Bob Mac Capture code change.
