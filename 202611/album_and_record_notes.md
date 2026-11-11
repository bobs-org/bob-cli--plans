---
tier: epic
title: Album notes — every record collection hangs under the area it serves
goal: 'Every record note reaches an area of responsibility. Each record collection (eat,
  podcasts, tv, movies) becomes an album note: `type: "[[album]]"` under its owner area,
  holding its record schema in frontmatter instead of `config.yml`, de-zorged, and
  showing a generated, grouped `[!records]` index. Every owner area or project shows a
  beautiful generated `## Albums` catalog. Records become `type: "[[record]]"` children
  of their album. Schema and data version together in the vault, a broken album fails
  alone, and glossary terms for album note and record note are published.

  '
decisions:
  ledger_mining:
    ask:
      Should the epic link the 2024–2025 spending ledger to restaurant records and
      backfill `visited`?
    default: true
    why: Turns 2 of 104 visited dates into about 80, from real dated spending lines
    answer: true
  memory_glossary_terms:
    ask: Should the plan add the Album Note and Record Note glossary strands?
    memory:
      - glossary:album-note
      - glossary:record-note
    default: true
    requested:
      As a part of this change, we should add excellent new glossary terms for "album
      note" and "record note".
    answer: true
  memory_glossary_edits:
    ask:
      Should the plan update the Capture, Notebook, and Area Note strands for albums and
      records?
    memory:
      - glossary:capture
      - glossary:notebook
      - glossary:area-note
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
  memory_albums_decision:
    ask: Should the plan add the albums-own-record-schemas decision record?
    memory:
      - decisions:albums-own-record-schemas
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
phases:
  - id: vault-cleanup
    title: Vault-only cleanup — owners, the eat.md de-zorg, and record health
    depends_on: []
    size: medium
    description: "vault-cleanup: with today's bob and no code, point eat at body,
      podcasts at dev, and tv at fun; de-zorg eat.md into a legacy_eat notebook under
      body after carrying footnote 17; rewrite the mapped block links; strip
      `companions`; fill the six empty categories; fold the location spellings; and gate
      on zero `bob records check` findings and no new broken links.

      "
  - id: album-model
    title: Album recognition, the vault schema registry, and bob albums list and check
    depends_on: []
    size: large
    description: 'album-model: recognize `type: "[[album]]"` and `type: "[[record]]"`,
      parse album frontmatter (top-level facts plus one nested `record:` schema) with
      key-path and line diagnostics, build the vault-aware record-kind registry with
      per-album isolation and the album-wins transition rule over config, and ship the
      read-only `bob albums list` and `check` with beautiful human output and JSON.

      '
  - id: catalog-engine
    title: Shared host-catalog region engine extracted from notebooks
    depends_on: []
    size: medium
    description: "catalog-engine: extract the notebook catalog's owned-callout region,
      host planning, and guarded sync transaction into a shared engine parameterized by
      callout name, heading, head title, and placement, keeping every notebook byte and
      test identical, so the albums catalog and records index reuse it.

      "
  - id: album-records
    title: Records, capture, and completion read schemas from albums
    depends_on:
      - album-model
    size: large
    description: 'album-records: route all eight config call sites through the registry
      so record kinds follow `--bob-dir`, write album records as `type: "[[record]]"`
      under their album, teach `bob records check` the album membership rules, enrich
      capture, capture-targets, and JSON additively with album identity, and make a bare
      album route (`Blaze @eat`) a teaching refusal.

      '
  - id: album-index
    title:
      The generated records index inside each album, rebuilt with every record write
    depends_on:
      - album-records
      - catalog-engine
    size: large
    description: "album-index: render the bob-owned `[!records]` index (tally title,
      groups in declared status order, sub-groups by `index_by`, faceted rows), rebuild
      it in the same transaction as every capture or `bob records` write from the album
      slice alone, and add `bob albums sync` plus its nightly step.

      "
  - id: album-catalog
    title: The Albums catalog in owner notes, and bob albums create
    depends_on:
      - album-index
    size: large
    description: "album-catalog: write one stable `[!albums]` row per album into each
      owner's `## Albums` section on the shared engine (placed after `## Notebooks`),
      extend sync, list, and check to catalogs, protect the section from capture, and
      add `bob albums create`, which also replaces `records init` for album kinds.

      "
  - id: plugins-albums
    title: Obsidian album cards, live chips, and the album header
    depends_on:
      - album-catalog
    size: large
    description: "plugins-albums: in bob-ledger-tools, style the `[!albums]` and
      `[!records]` callouts as hue-accented cards, draw live count, lead-group, peek,
      and freshness chips from `bob albums list -f json`, fold long index groups, render
      the album header with breadcrumb, add-record affordance, and schema card, and run
      a debounced `bob albums sync`.

      "
  - id: mac-albums
    title: Bob Mac Capture presents albums
    depends_on:
      - album-records
    size: medium
    description: "mac-albums: show each record route's album title, owner, and summary
      from the additive capture-targets fields, render the bare-album refusal with its
      one-key `@eat/` rewrite, and name the album in record previews and success lines,
      gated on the exact-SHA macOS CI run.

      "
  - id: vault-cutover
    title: Album cutover in the vault, then delete records from chezmoi
    depends_on:
      - vault-cleanup
      - album-catalog
    size: medium
    description: "vault-cutover: once every host runs album-aware bob, move each schema
      into its album's frontmatter under an equivalence gate, add the album and record
      type notes, merge and delete the four kind stubs, retype the 152 records, update
      the Base filters, sync the indexes and catalogs, and delete `records:` from
      chezmoi.

      "
  - id: ledger-mining
    title: Link the spending ledger to restaurant records and backfill visited
    depends_on:
      - vault-cutover
    size: medium
    description: "ledger-mining: when accepted, rewrite the 653 mappable `[[eat#<lid>]]`
      spending links in historical `_habit.md` notes to their records and backfill each
      restaurant's `visited` from its latest ledger date in one reviewed commit;
      otherwise close as a no-op.

      "
  - id: albums-wrapup
    title: Retire config-declared records, end-to-end proof, docs audit, and memory
    depends_on:
      - plugins-albums
      - mac-albums
      - vault-cutover
      - ledger-mining
    size: medium
    description:
      "albums-wrapup: remove the config `records:` reader and its transition fallback,
      turn `records init` into a hidden teaching refusal, walk every album gesture end
      to end in a temporary vault, audit every album and record doc, and publish the
      accepted glossary strands and decision record."
proposed_by: bbugyi200.apollo.research.43.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 04:56:41
status: wip
---

- **PROMPT:**
  [prompts/202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_and_record_notes.md)

# Plan: Album notes — every record collection hangs under the area it serves

## Why

Bryan wants everything he tracks to reach an area of responsibility. Records (one note
per restaurant, podcast, show, or movie) don't today:

- `eat.md`, the parent of 104 restaurant records, is a zorg export whose parent is the
  retired `[[h2_role]]`. No restaurant reaches an area.
- Record schemas live in chezmoi `~/.config/bob/config.yml` (`records:`) while records
  live in the vault. The two sync on different paths, and they have already drifted:
  chezmoi `b6eecd85` dropped `companions` from three schemas, and `bob records check`
  (bob 0.5.0) now reports **143 errors and 3 warnings** (137 `companions`, 6 empty
  categories).
- One malformed `records:` block disables every record feature. The notebook snapshot
  swallows that error, so a schema typo silently turns every record into an ordinary
  note.
- Each kind is spread over four artifacts: a config block, a prose type stub
  (`restaurant.md`), a collection note (`eat.md`), and a `.base`.
- Nothing in `body.md`, `dev.md`, or `fun.md` says these collections exist.
- The vault tasks `bob.md ^record-h2-sections` (and its child "Migrate record type
  configuration to corresponding note files") and the blocked `^sub-project-h2-sections`
  ask for exactly this.

This epic implements the research report
`research:202611/album_and_record_notes/album_and_record_notes.md`. Bryan agreed with
every requirement it recommends (A1–A12). Every phase worker reads the report sections
relevant to the phase with
`sase artifact read research:202611/album_and_record_notes/album_and_record_notes.md "<why>"`.
Verified facts in the report (counts, anchors, collisions, call sites) are the starting
point; re-verify the numbers your phase acts on, because the vault moves daily.

### Settled requirements (from the research, accepted by Bryan)

- **A1.** Promote the existing `podcasts.md` to the podcasts album. `podcast.md` is the
  type stub, not the collection, so nothing is renamed; the stub is merged and deleted
  at cutover.
- **A2.** All four collections become albums: `eat → body`, `podcasts → dev`,
  `tv → fun`, `movies → fun`. Then `records:` is deleted from chezmoi.
- **A3.** Records become `type: "[[record]]"`. The four kind stubs merge into their
  albums. `record.noun` keeps `restaurant`, `podcast`, `show`, and `movie` as the CLI,
  JSON, and Mac spellings.
- **A4.** Album facts are top-level scalars; the record contract is one nested `record:`
  mapping. Stem = folder = capture route (`dir:` disappears), and records always point
  at their album (the config `parent:` key disappears).
- **A5.** The album summary is a generated, grouped `[!records]` index with the
  actionable group first. The Base stays, linked from the intro.
- **A6.** Owners get `## Albums` with one stable row per album. Counts, backlog,
  freshness, and peeks are drawn by the plugin, never written.
- **A7.** Albums and records hold no tasks. Bare `Blaze @eat` becomes a teaching
  refusal.
- **A8.** Strip `companions` from the 137 records. Bryan confirmed this by accepting the
  research's recommendations.
- **A9.** De-zorg `eat.md` with the `job.md` recipe into `legacy_eat.md`, a notebook
  under `[[body]]`. Clean up `podcasts.md` and `tv.md`.
- **A10.** podcasts goes under `[[dev]]`.
- **A11.** Glossary strands for Album Note and Record Note, edits to the Notebook,
  Capture, and Area Note strands, and the decision record `albums-own-record-schemas`.
- **A12.** Mining the spending ledger is optional; see the `ledger_mining` decision.
- The six empty categories get filled in (recommended over making `category` optional).

### Design calls this plan makes beyond the research

1. **The tally is the index's callout title.**
   `> [!records] 104 restaurants · 🔖 11 to try · …` means a folded, skimmed, or
   plain-Markdown index still answers "how big, and what shape?".
2. **`index_facets`**, an optional top-level list, picks a row's facets (movies: `year`,
   `verdict`). Otherwise a default rule picks them, capped at three, so rows stay one
   glance wide.
3. **Plural and tally grammar are deterministic.** The plural comes from
   `record.plural`, or else from the label with its first letter lowercased (unless it
   starts with an acronym) plus "s". Choice labels split into a leading icon and text:
   `🔖 To try` → `🔖 11 to try`.
4. **An empty album keeps its index** and shows a teaching line. Its purpose is to say
   how to add the first record.
5. **Record writes never scan the vault.** The index is planned from the album's head
   plus the folder index that capture already loads. Mac live preview runs on every
   keystroke pause, and the vault has about 6,750 notes.
6. **`record_checkbox` is an error.** A record holds facts and verdicts, never work.
7. **`record.fields` is optional.** `bob albums create` writes the minimal valid album
   and prints a paste-ready `fields:` example, so it imposes no schema.
8. **Owner = parent.** An album's parent must be an area or project directly: no nested
   albums, and no album under a notebook. That keeps routing and catalogs predictable.
9. **The bare-route refusal applies only to album notes.** Config-declared folders keep
   the old `record_collection_route` warning until the config reader retires.
10. **The config reader retires inside this epic.** That happens in the wrap-up, after
    the cutover proves every kind is album-claimed, not in some unscheduled "later
    release". `records init` becomes a hidden teaching refusal, per the CLI rule that
    commands are never removed.
11. **The shared catalog engine is its own refactor phase**, byte-identical for
    notebooks, so it runs in parallel with the album model. It is also what
    `^sub-project-h2-sections` will reuse.
12. **chezmoi's `records:` is deleted inside the cutover**, once its all-hosts gate has
    passed, rather than as a dangling manual step.

## Vocabulary

- **Album note (album):** a vault-root note with `type: "[[album]]"`. It defines and
  indexes one collection of records of one kind, and hangs under the area or project it
  serves. Its stem names the record folder and the capture route: `eat.md` owns `eat/`
  and `@eat/`.
- **Record note (record):** one durable thing, one file, in an album's folder, with
  `type: "[[record]]"` and `parent: "[[<album>]]"`.
- **Noun:** what one record is (`restaurant`). It is today's record type id, accepted
  wherever a record TYPE is, and emitted as JSON `type`.
- **Owner:** the album's parent, always an area or project. **Root:** the owner's Root
  Parent.
- Not to be confused with a **notebook** (free-form note bullets, no schema), a Task
  Type, an Obsidian Base, or a folder.

## The contract

### The album note

`eat.md` after cutover:

```markdown
---
parent: "[[body]]"
type: "[[album]]"
summary: Restaurants Kelly and I have tried, liked, or want to try.
emoji: 🍽️
index_by: location
created: 2026-06-11T13:07:00-04:00
record:
  noun: restaurant
  label: Restaurant
  dated_notes: true
  fields:
    status:
      required: true
      values:
        to_try: 🔖 To try
        liked: 💚 Liked
        not_liked: 🚫 Not liked
    category:
      required: true
      suggest: true
    location:
      required: true
      suggest: true
    visited:
      values: date
    url:
      values: url
  legacy:
    - lid
    - source_*
---

# Restaurants

Restaurants [[kelly|Kelly]] and I like, don't like, or want to try. Add one with
`Name @eat/`. Browse and sort in [[eat.base]] · Related: [[drink]] · [[maybe_eat]] ·
[[legacy_eat]]

## Records

> [!records] 104 restaurants · 🔖 11 to try · 💚 86 liked · 🚫 7 not liked …generated
> (see below)…
```

| Key                                                    | Rule                                                                                                                                                                                                     |
| ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `parent`                                               | Required. Exactly one parent, resolved path-aware like every other parent, to an area or a project of any status. It may not be a notebook, an album, or anything else. The owner chain must be acyclic. |
| `type`                                                 | `"[[album]]"`, in every spelling `frontmatter_type_names` accepts.                                                                                                                                       |
| `summary`                                              | One plain line, at most 140 characters, exactly as for notebooks. Required by `create`; `check` warns when it is missing, long, or multi-line.                                                           |
| `emoji`                                                | Optional. Used in catalog rows, capture-targets, previews, and the Record Card (this replaces the config `emoji`).                                                                                       |
| `index_by`                                             | Optional field name. Sub-groups large index groups.                                                                                                                                                      |
| `index_facets`                                         | Optional list of field names, at most 3. Picks a row's facets.                                                                                                                                           |
| `created`                                              | Written by `create`. Informational.                                                                                                                                                                      |
| `record.noun`                                          | Required, `[a-z][a-z0-9_]*`, unique across albums. It may not be reserved: today's reserved ids plus `album` and `record`.                                                                               |
| `record.label`                                         | Optional display name. Defaults to the noun capitalized, with `_` turned into spaces.                                                                                                                    |
| `record.plural`                                        | Optional. Defaults to the label with its first letter lowercased (kept when the first two letters are uppercase, e.g. `TV show`), plus `s`.                                                              |
| `record.dated_notes`, `record.fields`, `record.legacy` | **Exactly** today's `records.<id>` semantics, including field options, kinds, and the storage-shape agreement, now checked across albums. `fields` may be absent (no schema fields).                     |

- **Placement and identity.**
  - Albums live at the vault root.
  - The stem is the folder and the route. It must be a valid route token, may not use a
    reserved directory (dot-folders, `.bob`, `done`, `ref`), and may not equal another
    album's noun.
  - Unknown keys inside `record:` are errors, with suggestions. Unknown top-level keys
    are tolerated, as on every vault note.
- **Title.** Derived exactly like a notebook's: the first H1, then `aliases:`, then the
  humanized stem.
- **YAML.** bob writes canonical block-style YAML: no flow maps, no comments. The
  migration converts today's flow maps. Hand-edited flow maps still parse.
- **Diagnostics.** Every schema error names the file, the 1-based line, and the key path
  (`eat.md:24 record.fields.location.values: …`). Find the line with an
  indentation-aware walk of the frontmatter lines; fall back to the `record:` line.

### The record note

```yaml
---
parent: "[[eat]]"
type: "[[record]]"
title: "Ani Ramen House"
status: to_try
category: "Ramen"
location: "Cranford"
created: 2026-11-08T19:15:47-0500
---
```

The rest of `docs/records.md` is unchanged: the filename rule, update semantics,
span-preserving edits, `## Notes`, and dated notes.

- **Membership is structural.**
  - A record is a note in an album's folder (`eat/*.md`, the same folder scope the
    record catalog reads today).
  - It counts, appears in the index, and is listed even when it is invalid or its
    album's schema is broken. A schema error never makes records invisible.
- **Findings, never silent repairs** (`bob records check`). Errors:
  - `record_parent_mismatch`: the parent is not this album.
  - `record_outside_album`: a `[[record]]`-typed note outside every album folder.
  - `record_checkbox`: any checkbox row outside fenced code.
  - Every existing field error.
- **Warning:** `record_type_legacy` when the type is the album's legacy noun link
  (`[[restaurant]]`). It is accepted during the transition.
- **Non-record children of an album stay legal.** `drink.md`, `podcast_ref.md`, and
  `lit/podcasts.md` walk through the album to its owner and are never indexed.
  `bob albums check` gives an info-level hint (`album_child_reparent`) suggesting the
  owner.

### Walks, tasks, and routes

- Root, clock, type, and Pomodoro-home walks already pass through any note by `parent`
  (`pomodoro_homes/resolve.rs::resolve_note`). Albums and records need no walk code.
- An album is never a Root Parent, Pomodoro home, Ready lane, review item, Stats root,
  or task residence. Records hold no tasks.
- **Capture.**
  - `Name @eat/` and `@eat/stem` are unchanged.
  - Bare `Blaze @eat` on an album note refuses with `album_holds_no_tasks`:

    ```text
    eat.md is an album and holds no tasks — @eat/ adds a restaurant; tasks go to @body
    ```

    The editor diagnostic is an error with a rewrite to `@eat/`.

  - Other task-home paths (`-P eat`, task move, inbox routing) teach the owner exactly
    as notebooks do. This is a contract change, so its commit is marked `!`.

- **Broken album.** `@eat/` refuses with `album_schema_invalid` (file, line, key path)
  and never degrades into task capture. Other albums keep working.

### The registry and the transition rule

- **One vault-aware registry** replaces the eight `load_records_config*` call sites:
  capture_record, capture_targets, capture_complete (candidates and shell), completion
  providers (three), records CLI, and the notebooks snapshot. It always reads the vault
  that `-b/--bob-dir`, or else `BOB_DIR`, names.
  - Discovery rides a vault-root frontmatter scan.
  - A route-specific resolver (`@eat/`, completion keystrokes) reads only `<stem>.md`'s
    head.
  - Ordinary task drafts keep their no-record-I/O fast path.
- **It yields today's `RecordType`** (with `id = noun`, `dir = stem`,
  `parent = "[[stem]]"`) plus a `source` (`album` with the path, or `config`).
  Downstream planners, renderers, and completion stay unchanged except where this plan
  says otherwise.
- **Transition rule.** This is the only place two sources coexist, and the wrap-up
  removes it.
  - An album that claims a noun or a folder owns that kind entirely.
  - Config `records.<id>` is used only for kinds that no album claims. Every use warns
    `records_config_deprecated`. Proving that a kind is unclaimed may use full
    discovery, which is rare.
  - An album with a broken schema **never** falls back to config.
  - Fields are never merged across sources.
  - A duplicate noun, a stem/noun clash, or a storage-shape conflict across albums
    implicates every album involved and never picks a winner.
- **The snapshot.** `VaultSnapshot` stops swallowing schema errors. Record recognition
  is `type: "[[record]]"` or a known noun link. Albums get a retained `LiteKind::Album`.

### The `[!records]` index (inside each album)

**Reader questions, in order:**

1. How big is this, and what shape is it in?
2. What is actionable now?
3. Can I find one thing ("that place in Clark")?
4. What did we think of it? That answer is the record itself, one click away.

**Rendered `eat.md`** (real counts):

```markdown
## Records

> [!records] 104 restaurants · 🔖 11 to try · 💚 86 liked · 🚫 7 not liked
>
> - **🔖 To try** · 11
>   - [[eat/ambeli_greek_taverna|Ambeli Greek Taverna]] — Greek · Cranford
>   - [[eat/ani_ramen_house|Ani Ramen House]] — Ramen · Cranford
>   - …
> - **💚 Liked** · 86
>   - **Clark** · 5
>     - [[eat/bubbakoos_burritos|Bubbakoo's Burritos]] — Burritos
>     - …
>   - **Cranford** · 21
>     - …
>   - **—** · 2
>     - …
> - **🚫 Not liked** · 7
>   - [[eat/1_walnut|1 Walnut]] — Diner · Cranford
>   - …
```

**Rules.**

- **Pure function** of the album declaration, its stem, and each record's frontmatter.
  It reads each record only through the closing `---`, as `records/catalog.rs` does. No
  dates, ages, mtimes, or note bodies are written. A current index produces no edit.
- **Head line:** `> [!records] <N> <plural>`, then ` · <icon> <count> <text>` for each
  group with a nonzero count, in declared choice order.
  - The text is the choice label minus its leading icon, with its first letter
    lowercased unless the first two letters are uppercase.
  - A nonzero "needs a status" group adds ` · ⚠️ <k> need a status`.
  - An album with no group field shows only the total.
  - Then one `>` blank line.
- **Group field:** `status` when it is a choices field, otherwise the first choices
  field in schema order, otherwise none (one flat list).
  - Groups follow declared choice order, so the actionable status Bryan declares first
    leads.
  - A group line is `- **<choice label>** · <count>`.
  - Empty groups are omitted.
  - Records with a missing or unknown value go last, under
    `- **⚠️ Needs a status** · <k>`. `bob records check` explains why.
- **Sub-groups:** only when `index_by` is set, the group has more than 12 rows, and
  there are at least two distinct values.
  - Values fold by case and by link display text (`Cranford` and
    `[[nj_towns#…|cranford]]` match). The displayed spelling is the most common one,
    with ties going to the plain-text spelling.
  - Sub-groups sort by displayed value, case-insensitively.
  - Missing values go last under `**—**`.
  - Small groups stay flat and show the value as a facet.
- **Rows:** `- [[<stem>/<file stem>|<Title>]]`, then ` — <facet> · <facet>` when any
  facet has a value.
  - The title is the record's `title:`, else its humanized file stem.
  - Rows sort by title case-insensitively, with the path as tie-breaker.
  - Alias text reuses the notebook catalog's inert escaping and drops `|`, `[`, `]`,
    `#`, and `^`.
- **Facets:**
  - With `index_facets`: those fields, in that order.
  - Otherwise: the non-group choice fields, then the `suggest: true` fields, in schema
    order, minus the group and sub-group fields, capped at 3.
  - Choices show their label, links their display text, lists join with `, `, and
    numbers print plainly (`2021`). Empty values are skipped.
- **Empty album:** the head line `> [!records] No <plural> yet`, then the body line
  ``> Add one with `Name @<stem>/`.``
- **Owned region:** the notebook region contract with callout name `records`.
  - Recognition ignores case and accepts a fold sign and any title. Writes are
    canonical.
  - **First write:** `## Records`, then the callout, immediately before the first real
    `##` heading after the frontmatter, or else at EOF after one blank line.
  - Later writes touch only the callout lines, wherever Bryan moved it.
  - Duplicates self-heal to the first.
  - BOM, CRLF, and every other byte survive.
  - Only album notes are planned. A `[!records]` callout elsewhere is never touched.
- **Who writes it:**
  - Every record create or update (capture, `bob records new|update`) in the same
    transaction.
  - `bob albums sync` and the nightly step, which repair hand edits.
  - `bob albums create` writes the empty index.
  - The plugin's debounced sync.

### The `## Albums` catalog (inside each owner)

**What an owner note should say about its albums**, ranked by value per byte:

| Content                                                          | Why                                                                   | Form                                                       |
| ---------------------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------- |
| Which collections serve this area: link and one-line scope       | Discovery, which is the point                                         | **Written**, stable                                        |
| Emoji identity                                                   | Scans next to `## Notebooks`                                          | **Written**                                                |
| Size and shape (`104`)                                           | At-a-glance health                                                    | Drawn chip                                                 |
| Lead group (`🔖 11 to try`), with a hover peek of up to 3 titles | Planning: "Friday dinner" from `body.md`, "movie night" from `fun.md` | Drawn chip and peek; a click opens the album at that group |
| Freshness ("last added 2d")                                      | Quietly flags a lapsed collection                                     | Drawn age chip                                             |
| Full record lists                                                | That is the album's job                                               | Neither                                                    |

**Rendered** `fun.md` (summaries are authored album text):

```markdown
## Albums

> [!albums]
>
> - 🎬 **[[movies|Movies]]** — Movies Kelly and I want to watch, and what we thought.
> - 📺 **[[tv|TV]]** — Shows we want to watch, are watching, or finished.
```

- **Rows:** `- <emoji> **[[stem|Title]]**`, then ` — <summary>` when a summary exists.
  Title and summary are inert text (the notebook escaping rule). Emoji is omitted when
  absent. No counts, dates, or colors are ever written.
- **Scope:** the notebook rule, through one engine.
  - A host lists albums whose owner lies in its subtree, stopping at another root.
  - Albums owned by a non-root sub-project appear in that project and in its root, under
    a `🧩 [[project]]` group line.
  - Rows sort by title, case-insensitively, with the path as tie-breaker.
  - An album with no valid owner appears in no catalog, and `check` reports it.
- **First write:** `## Albums`, then the callout.
  - It goes at the end of an existing `## Notebooks` section (immediately before the
    next heading of level 2 or less after it).
  - Otherwise immediately before the first real `## Tasks`.
  - Otherwise at EOF.
  - Later writes patch only the callout, wherever it was moved.
- **Empty subtree:** the callout is deleted. The heading goes too only when its section
  is then blank.
- **Protection:** `## Albums` is never a section-bullet target or completion row.
  `@body#albums` refuses with `albums_section_generated`.
- **Stability:** rows change only on album create, reparent, rename, or a title,
  summary, or emoji edit. **A record write never touches an owner note.**

### Commands

These follow `sase/memory/cli_rules.md`: Vault section, alphabetical, every long option
with a short alias, and the bare group runs the read-only `list`.

```text
bob albums [list] [-b DIR] [-f human|json] [-r ROOT]
bob albums check  [-b DIR] [-f human|json]
bob albums create PARENT STEM -n NOUN -s SUMMARY [-b DIR] [-d] [-e EMOJI] [-f human|json] [-l LABEL] [-o] [-t TITLE]
bob albums sync   [-b DIR] [-d] [-f human|json]
```

- **Human `list`** groups albums by root with root hues: emoji, title, `stem/`, count,
  per-group counts, and a freshness age. It shows issue indicators and the `sync` hint
  when an index or catalog is not current.

  ```text
  body
    🍽️ Restaurants   eat/        104 · 🔖 11 to try · 💚 86 · 🚫 7    2d
  dev
    🎙️ Podcasts      podcasts/    15 · 🎧 9 subscribed · 🚫 6         3w
  fun
    🎬 Movies        movies/      10 · 🔖 9 to watch · ✅ 1            1d
    📺 TV            tv/          23 · 🔖 23 to watch                  1d
  ```

  Output respects `NO_COLOR` and pipes, and never depends on terminal width. With no
  albums, `list` prints a teaching empty state that names `bob albums create`.

- **Human `check`** groups findings by album, then catalog hosts, and ends with an
  album, record, error, and warning summary.
- **Exit codes:**
  - `list` exits 0.
  - `check` exits 1 on any error.
  - `sync` exits 1 when anything is damaged or the run fails.
  - `create` exits 2 on validation or refusal and 1 on a runtime failure.
  - `list` and `check` never write.
- **`bob records new|update|show|check|list` keep their shape.**
  - TYPE accepts a noun or an album stem.
  - Human and JSON output gain the album.
  - `records list` shows each kind's `source`.

### JSON (additive; `schema_version: 1`)

`bob albums list` and `check` share one report:

```json
{
  "ok": true,
  "schema_version": 1,
  "albums": [
    {
      "stem": "eat",
      "path": "eat.md",
      "title": "Restaurants",
      "emoji": "🍽️",
      "summary": "Restaurants Kelly and I …",
      "noun": "restaurant",
      "label": "Restaurant",
      "plural": "restaurants",
      "parent": "body",
      "parent_kind": "area",
      "owner": "body",
      "root": "body",
      "breadcrumb": ["body", "Restaurants"],
      "fields": [
        {
          "key": "status",
          "label": "Status",
          "kind": "choices",
          "required": true,
          "list": false,
          "choices": [{ "value": "to_try", "label": "🔖 To try" }]
        }
      ],
      "group_field": "status",
      "index_by": "location",
      "index_facets": ["category", "location"],
      "records": 104,
      "groups": [
        {
          "value": "to_try",
          "label": "🔖 To try",
          "count": 11,
          "peek": ["Ambeli Greek Taverna", "Ani Ramen House", "Bubble Tea"]
        }
      ],
      "needs_status": 0,
      "last_record": "2026-11-08",
      "index": "current",
      "issues": []
    }
  ],
  "catalogs": [
    {
      "host": "body",
      "path": "body.md",
      "root": "body",
      "state": "current",
      "albums": 1,
      "issues": []
    }
  ],
  "counts": { "albums": 4, "records": 152, "roots": 3, "errors": 0, "warnings": 0 }
}
```

- `last_record` is the latest record `created` date.
- `peek` holds the first three titles of each group, in index order.
- `index` arrives with `album-index`, and `catalogs` with `album-catalog`. Earlier
  phases omit them.
- **Record payloads** (records CLI, capture items, `note_files`) gain
  `"album": {"stem", "title", "owner", "breadcrumb"}`. `"type": "restaurant"` keeps its
  meaning.
- **capture-targets record rows** keep `label`, `emoji`, and `count`, and fill the
  existing `title`, `summary`, `owner`, and `breadcrumb` fields.
- **`sync` and `create`** print the notebook-style envelope (`ok`, `dry_run`, `changed`,
  `changes[]`, `findings`, `commit`, `sync`, `warnings`), and `create` adds `album`.

### Findings

| Code                                                                | Severity | Trigger                                                          |
| ------------------------------------------------------------------- | -------- | ---------------------------------------------------------------- |
| `album_parent_missing`                                              | error    | Missing, empty, or unresolvable single parent                    |
| `album_parent_ambiguous`                                            | error    | Several parent declarations or targets                           |
| `album_parent_kind`                                                 | error    | Parent is not an area or project                                 |
| `album_cycle`                                                       | error    | Self-parent or cyclic chain                                      |
| `album_not_at_root`                                                 | error    | Album below a subdirectory                                       |
| `album_stem_invalid`                                                | error    | Stem is not a route token, or uses a reserved directory          |
| `album_schema_invalid`                                              | error    | Any `record:` problem, with file, line, and key path             |
| `album_noun_duplicate`                                              | error    | Two albums claim one noun, or a stem equals another album's noun |
| `album_property_conflict`                                           | error    | Storage shape disagrees across albums (names every album)        |
| `album_index_field_unknown`                                         | error    | `index_by` or `index_facets` names a field the schema lacks      |
| `album_checkbox`                                                    | error    | A checkbox row in an album                                       |
| `album_summary_missing` / `_long` / `_multiline`                    | warning  | As for notebooks                                                 |
| `album_parent_unquoted`                                             | warning  | `parent: [[x]]` without quotes                                   |
| `album_index_stale`                                                 | warning  | The index is missing or stale (hint: `bob albums sync`)          |
| `album_catalog_stale`                                               | warning  | A host's catalog is missing or stale                             |
| `record_type_legacy`                                                | warning  | Record typed with the legacy noun link                           |
| `records_config_deprecated`                                         | warning  | A kind still comes from config `records:`                        |
| `album_child_reparent`                                              | info     | A non-record child could name the owner instead                  |
| `record_parent_mismatch`, `record_outside_album`, `record_checkbox` | error    | See the record note                                              |

### Transactions and performance

- **Record writes.**
  - A record create or update plans the record bytes and the album's index together. The
    input is one overlay: the album's head, the staged record, and the folder index
    capture already loaded.
  - Both install atomically through the existing guarded writer, with the album bytes as
    a guarded preimage.
  - A multi-item draft into one album produces one final index.
  - Dry-run shows the album diff and the same bytes commit writes.
  - Never a whole-vault scan.
- **`bob albums sync`** reuses the shared guarded transaction:
  - It verifies vault identity, takes the bounded maintenance lock, and pre-syncs.
  - It plans from one snapshot, revalidates every preimage and read dependency, and
    installs.
  - It makes one scoped commit, `albums: sync (N notes)`, then post-syncs.
  - A no-op commits nothing and skips the post-sync.
  - Damaged hosts are skipped and reported.
  - The nightly step `albums` runs right after `notebooks`, lock-free, under nightly's
    lock.
- **Read-only commands never repair.**

## Phases

```text
vault-cleanup ───────────────────────────────┐
album-model ─► album-records ─┬─► album-index ─► album-catalog ─┬─► vault-cutover ─► ledger-mining ─┐
catalog-engine ───────────────┘                                 └─► plugins-albums ─────────────────┤
                              └─► mac-albums ───────────────────────────────────────────────────────┤
                                                                                     albums-wrapup ◄┘
```

Every code phase follows the repo's habits:

- `just` gates (`just check` / tests and lints the repo defines), `just docs-gen` after
  CLI changes, and `just docs-check` after docs changes. Read the `docs` and `cli_rules`
  reference memory first.
- Conventional commits, with `!` on any CLI or JSON contract break. Never edit the
  `Cargo.toml` version or `CHANGELOG.md`.
- Vault phases read the `obsidian` reference memory first. They edit `~/bob` directly
  and let `bob vault-sync` carry the commits.

### vault-cleanup

**Goal:** every record reaches an area today, every record validates, and `eat.md` stops
being a zorg export. This uses today's installed bob and no code. The root walk already
passes through untyped notes, so this phase alone delivers "every record has a root
parent".

- **Read first:** the research report's "What exists today", "Migration and de-zorg →
  Phase 0", and the `job.md` precedent (vault commits `85fce7e7c` and `fbab51701`;
  compare `legacy_job.md` and `legacy_body.md`).
- **Baselines first.** Record each of these before any edit and keep the scripts for the
  after-run:
  - A vault-wide unresolved-link count (wikilinks, embeds, heading and block refs;
    excluding `.bob/`, `_conflicts/`, and gitignored trees).
  - `bob records check -f json` counts.
  - The block-ID sets of `eat.md`, `podcasts.md`, and `tv.md`.
  - The vault-wide `#task` count.
- **`eat.md`.**
  1. Carry footnote 17 (Oh' Brian's Picante Burger, `^z-240619-0a`) into
     `eat/oh_brians.md` `## Notes` as a dated note in the album's `dated_notes` style.
     Today it appears only inside that record's `## Source` fence.
  2. Create `legacy_eat.md` as a notebook under `[[body]]`. Use the key order of
     `legacy_body.md` (`parent`, `template`, `type`, `summary`, `created`), the H1
     `# Legacy restaurant notes`, and the summary "Zorg-era restaurant tables, moved
     here from eat.md on <date>.". Move the tables, footnotes, `LID::breakfast` /
     `LID::coffee` lines, and the old intro into Title Case `##` sections. Keep every
     block ID verbatim.
  3. Rewrite each live `[[eat#^z-…]]` link (about 15, all inside restaurant records'
     carried notes) to the record that owns the anchor, using the records'
     `source_anchors`/`source_*` legacy fields. For the case-colliding IDs `7y`/`7Y` and
     `7z`/`7Z`, never guess: point those links at `[[legacy_eat#^…]]` and list them in
     the phase report.
  4. Rewrite `eat.md` to clean, non-zorg Markdown:
     - drop every `generated_from_zorg`/`zorg_*` key, `#eat | Notes`, and the
       `ALIGN TABLES` line;
     - `parent: "[[body]]"`;
     - `created:` taken from `restaurant.md`;
     - `# Restaurants`, then the intro from the contract (Kelly as a link, the
       `Name @eat/` hint, the `[[eat.base]]` link, and Related `[[drink]]`,
       `[[maybe_eat]]`, `[[legacy_eat]]`).

     Leave `type` unset until the cutover. The 694 `[[eat#<lid>]]` heading links keep
     resolving to `eat.md`.

- **`podcasts.md`.** Set `parent: "[[dev]]"`. Rewrite the about 33 `[[podcasts#^z-…]]`
  links to their records through `source_anchors`, and keep `## Notes` with
  `^z-241007-09`. The hand lists stay until the cutover; afterwards no inbound link may
  point at their block IDs.
- **`tv.md`.** Set the canonical `parent: "[[fun]]"`. Rewrite `funny.md`'s
  `[[tv#^studio]]` to the matching record (verify `tv/the_studio.md` exists) and drop
  the stray bullet. Add `# TV`, a one-line intro with the `Name @tv/` hint, and a
  `[[tv.base]]` link.
- **`movies.md`.** It is already under `[[fun]]`. Only polish the intro to the same
  shape.
- **Records.**
  - Strip `companions` from the 137 records. Change only those frontmatter lines, keep
    every other byte, and list the files touched.
  - Fill the six empty categories (`1_walnut`, `bagel_supreme`, `clark_bagels`,
    `pardon_my_cheesesteak`, `rustic_mill_diner`, `urban_burger`). Take each value from
    the legacy table's category column or the record's own notes, reusing an existing
    category spelling. Put each value and its source in the report.
  - Fold the three duplicate location spellings to each value's most-used spelling.
- **`eat.base`.** Rename or re-sort the "🕐 Recently Tried" view (for example "🕐
  Recently added" by `created`) so it stops presenting zorg `source_dates` as visits.
- **`bob notebooks sync`** adds the `legacy_eat` row to `body.md`'s `## Notebooks`.
- **Gates.** Paste all of these into the phase report:
  - `bob records check` reports 0 errors and 0 warnings.
  - `bob notebooks check` has no new findings.
  - The unresolved-link count did not increase.
  - Every moved block ID is accounted for.
  - The `#task` count is unchanged.
  - A parent-walk script shows every one of the 152 records reaching an area.

### album-model

**Goal:** bob understands albums and can tell Bryan everything about them, read-only.

- **Recognition.**
  - Add `frontmatter_is_album` and `frontmatter_is_record` next to
    `frontmatter_is_notebook` (`src/native/projects/scan.rs`).
  - Add `LiteKind::Album` (retained bytes) to the notebooks snapshot. Recognize
    `RecordInstance` by `type: "[[record]]"` or a known noun link.
  - Stop swallowing schema errors in `VaultSnapshot::load`. Membership must not depend
    on them.
  - Notebook rules treat an album like any non-host parent (already
    `notebook_parent_kind`).
- **The parser** (new `src/native/albums/`).
  - Parse top-level facts with the existing frontmatter helpers, and `record:` with
    `serde_yaml` (order-preserving).
  - Refactor `records/schema.rs` so field, choice, default, legacy, and storage-shape
    parsing take a diagnostic context, either a config path (today) or an album file
    plus key path and line, instead of a hard-wired config path. Today's config
    behaviour must stay identical.
  - Implement every album rule and finding in the contract that does not need the index
    or catalog.
- **The registry.**
  - `discover` (root scan) and `resolve_route(stem)` (one head) both return `RecordType`
    plus `source`.
  - Implement the transition rule, per-album isolation, and the cross-album checks
    (duplicate noun, stem/noun clash, storage shape).
  - Unit-test every branch, including "broken album never falls back" and `--bob-dir`
    pointing at a second vault.
- **Owner and root.** Owner = the resolved parent (area or project, path-aware). Compute
  root and breadcrumb through the existing Root Parent walk.
- **`bob albums list|check`.**
  - Wire into the Vault section of `bob --help` and the completion groups (`bob <TAB>`).
  - Implement the human output and the JSON in the contract, minus `index`/`catalogs`.
    Record counts, groups, peeks, `needs_status`, and `last_record` come from the
    existing per-folder record catalog.
  - `-r ROOT` filters by computed root and recomputes counts.
- **Docs.**
  - Retitle `docs/records.md` to "Albums and record notes" and keep its URL. It becomes
    the hub: the album file, its keys and rules, the transition, and `bob albums list` /
    `check`.
  - Update `docs/reference/vault-format.md` (an "Album notes" section; record notes
    point at it) and `docs/reference/json-outputs.md`.
  - Run `just docs-gen`.
- **Tests:** a fixture vault with four valid albums, a broken schema, a duplicate noun,
  a storage conflict, an album under a notebook, a nested album, an unquoted parent, and
  a record in a broken album that still counts.

### catalog-engine

**Goal:** one engine for every bob-owned callout region in host notes, with no behaviour
change.

- Extract `src/native/notebooks/region.rs`, and the host-planning and
  `apply_and_commit`/`run_locked` scaffolding from `catalog.rs`, `sync.rs`, and
  `txn.rs`, into `src/native/host_catalog/`, parameterized by a `RegionSpec`:
  - callout name;
  - heading text;
  - optional computed head title;
  - placement: `BeforeFirstTasks` for notebooks; `AfterSectionThenTasks("Notebooks")`
    for albums; `BeforeFirstH2` for the records index;
  - whether legacy v1 markers exist (notebooks only).
- `VaultSnapshot` stays where it is (`album-model` edits it in parallel). Keep this
  phase's edits there minimal, to avoid conflicts.
- **Notebooks stay byte-identical.** Every existing notebook test passes unchanged; do
  not edit their expectations.
- **Add spec-parametrized region tests:**
  - a second callout name;
  - a head title;
  - both new placements, with and without a `## Notebooks` section;
  - fences and frontmatter staying inert;
  - CRLF/BOM survival;
  - duplicate self-heal;
  - empty-subtree deletion.
- No user-visible change and no user docs. Note the engine in any developer doc that
  describes the notebooks module layout.

### album-records

**Goal:** every record consumer reads album schemas, album records are born in their new
shape, and albums refuse tasks.

- **Call sites.** Replace every `load_records_config*` use with the registry: capture
  route policy, capture-targets, capture-complete candidates and shell, the three
  completion providers, the records CLI, and the notebooks snapshot.
  - `RecordRoutePolicy` carries per-route state, either a valid kind or an album
    diagnostic, instead of one global `config_error`.
  - The old `records_config_invalid` refusal remains for a malformed config block used
    by unclaimed kinds.
- **Writer.** For album kinds the renderer writes `parent: "[[<stem>]]"` and
  `type: "[[record]]"`. Config kinds keep `[[<id>]]` and today's parent rule.
- **`bob records check`.** For album kinds:
  - accept `[[record]]`, and warn `record_type_legacy` on the noun link;
  - error on `record_parent_mismatch`, `record_checkbox`, and `record_outside_album`
    (vault-wide, for `[[record]]`-typed notes);
  - keep every field check;
  - show the album title and owner in human output.
- **`bob records` surface.**
  - TYPE accepts a noun or an album stem.
  - JSON gains `album` and `source`.
  - `records list` shows source and album.
  - `records init` is unchanged in this phase.
- **Capture and completion.**
  - Record items and `note_files` gain `album`.
  - Human success lines name the album and owner, for example
    `🍽️ New restaurant → Restaurants · body`.
  - capture-targets record rows fill `title`, `summary`, `owner`, and `breadcrumb`.
- **Teaching refusals.**
  - Bare-album capture refuses with `album_holds_no_tasks`, plus the editor error
    diagnostic with its `@<stem>/` rewrite.
  - `-P`, task move, and inbox routing name the owner, sharing the notebook refusal
    helpers' shape (`parent_notes.rs`, `task_move/plan.rs`, `capture_language/line.rs`).
  - Config-declared folders keep `record_collection_route`.
  - Commit as `feat(capture)!:`.
- **Docs:**
  - `docs/records.md`: schemas live in albums; the new check codes.
  - `docs/capture.md` and `docs/reference/capture-syntax.md`: the refusal.
  - `json-outputs.md`.
  - `docs/reference/configuration.md`: `records:` is deprecated and serves only
    unclaimed kinds.
  - `just docs-gen`.
- **Tests:**
  - capture create and update into an album;
  - the refusal;
  - a broken album refusing only its own route;
  - completion against `--bob-dir`;
  - the transition with a config kind and an album claiming the same noun;
  - byte-exact rendered records.

### album-index

**Goal:** every album opens on its tally and the shelf of things to do next, and the
index is never stale after a bob write.

- **Renderer.** A pure function from the album declaration plus record frontmatters to
  callout lines, exactly per the contract (tally title, groups, sub-groups, facets,
  needs-a-status, empty state, escaping). Golden-test it against eat-, podcasts-, tv-,
  and movies-shaped fixtures, including the 12-row threshold, value folding, and
  `index_facets`.
- **Placement and region** through the `catalog-engine` spec (`records`, `## Records`,
  `BeforeFirstH2`). Only album notes are planned.
- **Same-transaction rebuild.**
  - Capture record create/update (`src/native/capture/record_note.rs`, staged through
    the batch overlay) and `bob records new|update` plan the album's index from the
    album head plus the folder index with the staged record overlaid, then stage the
    album edit next to the record.
  - No change means no album edit.
  - Dry-run and commit show identical bytes.
  - Verify that a multi-item draft into one album produces one index edit.
  - Verify that an ordinary task draft performs no album I/O.
- **`bob albums sync`** repairs every index through the shared transaction, plus the
  `albums` nightly step after `notebooks` (`src/native/nightly.rs`).
- **Reporting.** `list` and `check` report `index` and `album_index_stale`.
- **Docs:** records.md (the index contract), nightly and `vault-git-sync.md` (the
  `albums: sync` commit label), json-outputs, and `just docs-gen`.

### album-catalog

**Goal:** every owner area or project shows the collections that serve it, and new
albums take one command.

- **Catalog** through the `catalog-engine` spec (`albums`, `## Albums`,
  `AfterSectionThenTasks("Notebooks")`). Scope, rows, ordering, the empty subtree, and
  stability are exactly per the contract.
  - Reparenting an album updates both the old and the new host.
  - `bob projects sync` never lists an album on a Sub-projects line.
- **`sync`** now plans indexes and catalogs in one commit. The nightly step does the
  same. `list` and `check` report `catalogs` and `album_catalog_stale`.
- **Section protection.** `@body#albums` refuses with `albums_section_generated`, and
  the heading is never a section-bullet target or completion row (mirror
  `notebooks_section_generated`).
- **`bob albums create PARENT STEM -n NOUN -s SUMMARY`.** One guarded write and one
  commit, `albums: create <stem>.md`.
  - `<stem>.md`:
    - canonical block YAML keys `parent`, `type`, `summary`, `emoji`, `created`, then
      `record:` with `noun` and `label`;
    - an H1 (`-t`, else the humanized stem);
    - an intro line teaching `Name @<stem>/` and linking `[[<stem>.base]]`;
    - the empty-state index.
  - A starter `<stem>.base` filtered on `file.inFolder("<stem>")` and
    `note.type == link("record")`.
  - The owner's catalog row.
  - It never creates the folder; the first record does.
  - Refusals: invalid stem or noun, reserved, duplicate noun, existing path
    (case-insensitive), parent missing, ambiguous, or unsupported, multi-line summary,
    and a damaged affected catalog.
  - `-d` shows every file and diff. `-o` opens the album after a real create; failing to
    open is a warning.
  - Human output ends with a paste-ready `fields:` example and the `bob albums check`
    hint.
- **`bob records init TYPE`** for an album-claimed kind refuses with exit 2, teaching
  `bob albums create`. Config kinds keep today's scaffold.
- **Docs:** records.md (the catalog, `create`), `docs/notebooks.md` (`## Albums` follows
  `## Notebooks`; the shared engine), vault-format (the owner sections), json-outputs,
  the CLI reference via `just docs-gen`, and `just docs-check`.

### plugins-albums

**Goal:** the Markdown already reads well, and Obsidian makes it delightful. Work in
`bob-plugins` (open it with `/sase_repo`; read its `AGENTS.md`). Edit `src/` fragments
in `bob-ledger-tools`, run `npm run build`, `npm test`, and `npm run docs:check`, then
`bob plugins sync`.

- **Data.**
  - Build an album data fragment modelled on `241a-plugin-notebook-data.js`: one
    stale-while-revalidate `bob albums list -f json` report with `schema_version`
    validation, and one shared sync operation.
  - Run a debounced (3 s) `bob albums sync` when an album note, or a record note in an
    album folder, changes its frontmatter.
- **`[!albums]` card** in owner notes, in Reading view and Live Preview:
  - an area-hue accent, reusing the notebook catalog card's look;
  - each row gets right-aligned chips: total count, the lead group (`🔖 11 to try`, the
    first nonempty group) with a hover peek of up to three titles (clicking opens the
    album scrolled to that group), a freshness age chip from `last_record`, and a stale
    or invalid chip with the sync hint;
  - chips stack below the summary on narrow panes.
- **`[!records]` card** in albums:
  - a styled tally header;
  - groups after the first start folded when they exceed 12 rows (view state only, never
    written);
  - row age chips from each record's date fields (`visited`, `watched_on`), reusing the
    notebook age-chip helper.
- **Album header** (modelled on `241e`–`241g`):
  - a breadcrumb (`Body › Restaurants`);
  - a `＋ restaurant` affordance that copies `Name @eat/`;
  - a read-only schema card (fields, kinds, required marks, choice chips);
  - hide the raw `record` property row
    (`.metadata-property[data-property-key="record"]`) on album notes only.
- **Navigation and pickers:**
  - album and record icons;
  - the parent picker offers only areas and projects for albums, and only albums for
    records;
  - task-move and other task-home pickers never offer albums or records.
- **Docs:** the plugin reference pages for the cards and header. Never hand-edit
  generated `main.js`.

### mac-albums

**Goal:** the main capture surface shows where a record will live. Work in
`bob-mac-capture` (open it with `/sase_repo`; read its `AGENTS.md`). The app stays a
thin client (`decisions:mac-capture-is-a-thin-client`): it parses nothing and computes
nothing.

- Record-route rows (`kind == "record"`) keep emoji, label, and count, and add the album
  title, the owner breadcrumb, and a dimmed summary (as notebook rows do).
- The `album_holds_no_tasks` error renders with its one-key `@<stem>/` rewrite.
- Record previews and success lines name the album and owner from the item's `album`
  object.
- The extra album-index change in dry-run JSON adds no card and disturbs nothing.
- Build fixtures from real bob output in a temporary vault (`BOB_DIR`, `BOB_NOW`).
- Update `docs/capture-panel.md` where it lists record rows.
- Gate on the exact-SHA macOS run with `/sase_monitor` (or `scripts/dev/macos-ci-check`
  first).

### vault-cutover

**Goal:** the vault becomes self-describing. Albums own their schemas, records are
`[[record]]`, and owners show their albums.

- **Precondition.** Every host runs album-aware bob (read the `tailnet` reference memory
  for access):
  - On athena, apollo, and the Mac, `bob albums check -h` succeeds, and
    `bob capture --dry-run 'Test @x/'` against a temporary `BOB_DIR` album plans a
    record plus an index edit.
  - Update a lagging Linux host with `just install` from current master.
  - For the Mac, ask Bryan through `/sase_questions` to run `just install-all`.
  - Change nothing until all three pass.
- **Equivalence baseline.** For each noun, save `bob records show <noun> -f json` and
  `bob records list -f json`.
- **One reviewed vault change set.** Commit through `bob vault-sync`, in batches if
  needed. Each batch must leave `bob records check` at 0 errors.
  1. **Album frontmatter** on `eat.md`, `podcasts.md`, `tv.md`, and `movies.md`.
     - Keys: `type: "[[album]]"`, a crafted `summary`, `emoji` from config, `created`,
       and each schema converted from config `records.<id>` into canonical block-style
       `record:` (minus `dir`, `emoji`, `parent`).
     - `eat` gets `index_by: location`.
     - Choose `index_facets` where the default rule reads poorly (for example movies:
       `year`, `verdict`, `recommended_by`).
     - **Equivalence gate:** every saved `records show` payload is identical except for
       the added `source` and `album`.
  2. **Type notes.** Add `album.md` and `record.md` under `[[type]]` in the house style
     of `notebook.md`: `parent`, `created`, two sentences, and a `bob albums list` or
     `bob records check` pointer.
  3. **Stubs.** Merge the useful prose of `restaurant.md`, `podcast.md`, `show.md`, and
     `movie.md` into their album intros, then delete the four stubs. First verify that
     no inbound links remain besides the records' `type:` and `.bob/` archives.
  4. **Retype** the 152 records to `type: "[[record]]"`. Change only that line, with a
     span-preserving script. Expect zero `record_type_legacy` warnings afterwards.
  5. **Bases.** Change the four `.base` filters to `note.type == link("record")` and
     keep every custom formula and view.
  6. **`podcasts.md`.** Remove the `## Subscribed`/`## Unsubscribed` hand lists, after
     proving that no inbound link targets their block IDs, and keep `## Notes`. Replace
     the Base embeds in `podcasts.md` and `movies.md` with intro links.
  7. **Sync.** Run `bob albums sync`. It writes four `[!records]` indexes and three
     `## Albums` catalogs (`body`, `dev`, `fun`).
- **chezmoi.** Open the `chezmoi` linked repo with `/sase_repo`. Delete `records:` from
  `home/dot_config/bob/config.yml`, commit, and apply locally. Then confirm that
  `bob records list` shows `source: album` for all four nouns, and that no
  `records_config_deprecated` warning remains.
- **Gates.** Paste these into the report:
  - `bob records check` and `bob albums check` report 0 errors (list any warnings).
  - `bob notebooks check` is unchanged.
  - The unresolved-link count did not increase (reuse the cleanup script).
  - `bob capture --dry-run 'Test Place @eat/'` shows the record and an `eat.md` index
    diff, and no `body.md` diff.
  - `git grep` finds no `type: "[[restaurant]]"`, `[[podcast]]`, `[[show]]`, or
    `[[movie]]` frontmatter outside `.bob/`.
  - A second `bob albums sync` is a no-op.

### ledger-mining

> [!decision] ledger_mining
>
> Runs only when `ledger_mining` is accepted. Otherwise close the phase with a no-op
> report.

**Goal:** the restaurants' real visit history becomes visible.

- **Map.** Scan `_habit.md` notes (2024-02 to 2025-12) for `[[eat#<lid>]]` links (about
  694; about 657 on dated `## @BANK` lines). Map each `<lid>` through the records'
  legacy `lid` field. Expect about 653 mapped links across 80 restaurants. Leave
  unmapped LIDs untouched, and list them in the report.
- **Rewrite** each mapped link to `[[eat/<file stem>|<record title>]]`. Change nothing
  else on the line.
- **Backfill `visited`** with each restaurant's latest dated ledger line, only when the
  field is absent or older. Use a span-preserving edit in schema order, through
  `bob records update` or an equivalent byte-careful script.
- **Read the data honestly.** A spending line means "bought there" (dine-in or takeout).
  Say so in the commit message.
- **One reviewed change set.**
- **Gates:**
  - the rewritten count equals the mapped count;
  - `bob records check` reports 0 errors;
  - the unresolved-link count does not increase;
  - `bob albums sync` reports no index change, since `visited` is not a facet;
  - the report lists the before and after `visited` coverage.

### albums-wrapup

**Goal:** one schema home, proven end to end, documented, and remembered.

- **Retire config-declared records** (commit `feat(records)!:`):
  - Remove the `records:` config reader, the transition fallback,
    `records_config_deprecated`, `records_config_invalid`, and
    `record_collection_route`.
  - Hide `bob records init`, which now always refuses with exit 2 and teaches
    `bob albums create`.
  - Drop `records` from `docs/reference/configuration.md`.
  - A leftover `records:` key in any host config is simply ignored. Unknown keys are
    already tolerated.
- **End-to-end walkthrough** in a temporary vault (`BOB_DIR`, `BOB_NOW`). Paste the
  transcript into the report.
  - `bob albums create`, then capture create and update into the album, including a
    multi-item draft.
  - A bare `@album` refusal, and a broken album refusing only its route.
  - `sync`, `check`, and `list -f json`.
  - Reparenting an album moves its catalog row.
  - The index and catalog bytes.
  - `bob records new|update` keeping the index current.
- **Docs audit:**
  - `docs/records.md` reads as the hub;
  - every album and record mention in `notebooks.md`, `capture.md`, `capture-syntax.md`,
    `json-outputs.md`, `vault-format.md`, `configuration.md`, `compatibility.md`,
    `vault-git-sync.md`, the nightly docs, and the CLI reference agrees with shipped
    behaviour;
  - `just docs-check` passes.
- **Memory**, through `/sase_memory_write`, applying only the accepted decisions.
  - Follow the house strand shape: frontmatter `keyword` and `aliases`, no H1, prose
    wrapped near 88 columns, links as `[[slug]]` / `[[decisions:slug]]`, and
    frontmatter-looking `[[…]]` kept in inline code.
  - Run `sase memory init`, then `just docs-gen` (the docs glossary and decisions pages
    are generated from the memory webs), then `just docs-check`.
  - The drafts below come from the research's "Glossary drafts and decision record".
    Tighten each against shipped behaviour.

  > [!decision] memory_glossary_terms
  >
  > - **Create `glossary:album-note`** (aliases: album, collection note).
  >   - Define it: a vault-root note with `type: "[[album]]"` that defines and indexes
  >     one collection of [[record-note]]s.
  >   - Stem = folder = route (`eat.md` owns `eat/` and `@eat/`).
  >   - Frontmatter: one `parent` naming the [[area-note]] or [[project-note]] the
  >     collection serves (how everything Bryan tracks reaches an area), a one-line
  >     `summary`, and the nested `record:` schema (`noun`, `label`, `dated_notes`,
  >     ordered `fields`, `legacy`).
  >   - Walks pass through it, and it is never a [[root-parent]] or task home.
  >   - Its bob-owned `[!records]` index, and the owner's `## Albums` catalog.
  >   - What it is not: a [[notebook]], a [[task-type]], a Base, or a folder.
  >   - Link `docs/records.md` and [[decisions:albums-own-record-schemas]].
  > - **Create `glossary:record-note`** (alias: record).
  >   - Define it: one durable thing Bryan tracks, in its album's folder, with
  >     `type: "[[record]]"` and `parent` naming its [[album-note]].
  >   - The schema types its fields.
  >   - `## Notes` bullets, dated when the album sets `dated_notes`.
  >   - It holds facts and verdicts, never work. "Book Ani Ramen House" is an area task
  >     that links it.
  >   - Created and updated through `Name @eat/`, `@eat/<stem>`, and
  >     `bob records new|update`, and checked by `bob records check`.
  >   - What it is not: a [[reference-note]], a [[notebook]], or a [[task-type]].

  > [!decision] memory_glossary_edits
  >
  > - In `glossary:capture`, replace the now-false "record note (a note of a
  >   config-declared record type, see `docs/records.md`)" with "a [[record-note]] in an
  >   [[album-note]] (see `docs/records.md`)".
  > - In `glossary:notebook`, make "Not a … record note" read "Not a … [[record-note]]
  >   or [[album-note]]". Add that a notebook never names an album as its parent, and an
  >   album never names a notebook.
  > - In `glossary:area-note`, add that an area note lists the albums beneath it in a
  >   machine-owned `## Albums` catalog, next to `## Notebooks`.

  > [!decision] memory_albums_decision
  >
  > Create `decisions:albums-own-record-schemas`:
  >
  > - **Claim.** An album note's frontmatter is the only home of a record kind's schema.
  >   Every record is `type: "[[record]]"` under its album, and every album sits under
  >   the area or project it serves. bob owns the `[!records]` index and the owner's
  >   `## Albums` content; Bryan owns their positions. Counts and freshness are drawn,
  >   never written to owner notes.
  > - **Evidence.** The `companions` drift (chezmoi `b6eecd85` against 137 records), the
  >   notebook catalog precedent, the research report, and this epic's commits.
  > - **Rejected:** config `records:`; a body schema callout; flat top-level schema
  >   keys; per-kind type stubs; `type: "[[eat]]"`; Bases-only summaries; counts written
  >   into area notes.
  > - **Cost:** the Properties panel cannot edit the schema; the cutover was gated on
  >   every host; there was a one-time retype of 152 records.
  > - **Reopens when:** a second vault or a non-bob writer needs schemas, Obsidian ships
  >   nested-property editing, or Bryan wants records to hold tasks.

- **Report**, but do not close, the vault task `bob.md ^record-h2-sections` for Bryan to
  close. Note that `^sub-project-h2-sections` can now reuse `host_catalog`.

## Risks

| Risk                                                  | Handling                                                                                                                  |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| A Properties-panel save re-serializes the schema YAML | Canonical block YAML makes a re-serialize a no-op. The plugin hides the row. `bob albums check` validates after any edit. |
| A schema typo synced from a phone                     | It breaks one album only. Capture refuses with a file:line diagnostic and never falls back to stale config.               |
| An old bob writes old-shape records after the cutover | The cutover waits for all hosts. The new bob accepts the legacy type with a warning, and sync repairs the index.          |
| Area-note merge conflicts across hosts                | Avoided by design: record writes never touch owner notes.                                                                 |
| Broken links from the de-zorg                         | Fully mapped rewrites, never-guessed collisions, and gated unresolved-link counts.                                        |
| Capture keystroke latency                             | Route resolution reads one file head, and the index plans from the album slice.                                           |
| JSON or Mac contract break                            | Every change is additive. The bare-album refusal and the config retirement are marked `!`.                                |
| Two catalog engines diverge                           | One `host_catalog`, extracted before either album region exists.                                                          |
| A parallel phase edits `notebooks/snapshot.rs`        | `catalog-engine` keeps out of the snapshot; `album-model` owns it.                                                        |

## Success criteria

- All 152 records reach an area: restaurants via `eat → body`, podcasts via
  `podcasts → dev`, and shows and movies via `tv`/`movies → fun`.
- `bob records check` and `bob albums check` report zero errors. `bob records list`
  shows `source: album` for all four nouns, and chezmoi has no `records:`.
- `eat.md` is clean, non-zorg Markdown, its history lives in `legacy_eat.md` under body,
  and no link broke.
- Each album opens on a tally and its actionable group. `body.md`, `dev.md`, and
  `fun.md` each show a `## Albums` catalog. Obsidian draws live chips and peeks.
- Capturing a restaurant changes exactly the record and `eat.md`, never `body.md`.
- A broken album breaks only its own route, and `Blaze @eat` teaches `@eat/` and
  `@body`.
- The Album Note and Record Note glossary terms, and the decision record, are published
  as accepted.
