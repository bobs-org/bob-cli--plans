---
tier: epic
title: Ref Cards
goal: "Every library reference note opens on a calm, information-dense Ref Card in the
  Property Card family. The card answers what the reference is, where it came from,
  where Bryan is with it, and what useful material it holds, and it omits whatever a
  note lacks. `==` opens a bob-owned Reference editor. bob protects reference sync and
  identity keys from generic edits. The work ships beside bob-cli-qm and touches it only
  at named seams.

  "
phases:
  - id: ref-props
    title: bob props reference roles and the Reference contract
    depends_on: []
    size: medium
    description:
      "ref-props: scope the reference note kind to bob's library coverage, protect sync,
      identity, and reading-task keys (resolving bob-cli-qn), and add the bob-owned
      reference settings contract. It also adds the props_ref_contract_v1 capability and
      the conditional H1 retitle, with docs and tests."
  - id: ref-kit
    title: Ref species kit, preview gate, vocabulary, and scaffolding
    depends_on: []
    size: medium
    description:
      "ref-kit: owns every shared-kit edit. Adds the ref species with the ribbon tile,
      the silent created stamp, a coordinator kind, the precedence slot after prose with
      the exhaustive test, a ref-owned preview gate, and one shared ref vocabulary that
      the Reading Board also uses. Also adds fragment stubs, fixtures, a docs stub, and
      coordination notes on qm.6 and qm.7."
  - id: ref-service
    title: Shared ref-library report service
    depends_on: []
    size: medium
    description:
      "ref-service: one lazily subscribed, stale-while-revalidate `bob ref list -A -R
      all -f json` snapshot indexed by path and shared by every mounted card. It uses
      narrow, debounced revalidation triggers and never spawns from paint."
  - id: ref-card
    title: Ref Card shell from local identity
    depends_on:
      - ref-kit
    size: medium
    description:
      "ref-card: the pure identity model and the DOM. The host mounts behind the gate
      and precedence. The card owns the title with matching-H1 adoption and gets the
      byline, access chips, foot, Managed drawer, `_` compact, and `==` focus-or-add on
      title. The docs page body comes in this phase."
  - id: ref-readouts
    title: Reading-state readouts from the shared snapshot
    depends_on:
      - ref-card
      - ref-service
    size: medium
    description:
      "ref-readouts: subscribe cards to the shared snapshot. Patch the lane chip, Today
      pill, annotations, reread lineage, identifiers, and diagnostics into existing rows
      with no layout shift. Spy tests prove there are no spawns."
  - id: ref-editor
    title: The Reference editor behind ==
    depends_on:
      - ref-card
      - ref-props
    size: medium
    description:
      "ref-editor: `==` opens the shared Property Editor on bob's REFERENCE contract. It
      has editable citation rows and owned-elsewhere rows that route reading state,
      residence, and file name to their owners. It probes the capability and falls back
      to title focus-or-add."
  - id: ref-acceptance
    title: Real-Obsidian acceptance and screenshot review
    depends_on:
      - ref-readouts
      - ref-editor
    size: medium
    description:
      "ref-acceptance: after bob-cli-qm.6 closes, extend the takeover harness as a
      sibling runner. Prove takeover, single ownership, native restoration, passivity,
      data, and editor flows. Review the light, dark, narrow, wide, forced-colors, and
      grayscale screenshot matrix beside the other family cards."
  - id: ref-go-live
    title: Switch Ref Cards on, docs, approved memory, and clean deploy
    depends_on:
      - ref-acceptance
    size: medium
    description:
      "ref-go-live: after bob-cli-qm.7 closes, delete the ref preview gate, extend the
      passive proof, and finish the docs in both repos. Apply only the accepted memory
      decisions, then deploy clean with provenance."
decisions:
  memory_ref_card:
    ask: Add the glossary:ref-card strand at go-live?
    memory:
      - glossary:ref-card
    requested:
      As a part of this change, we should add an excellent new glossary term for "ref
      card".
    default: true
    answer: true
  memory_property_card:
    ask: Update glossary:property-card so the family names the Ref Card?
    memory:
      - glossary:property-card
    corrects: The shared family of note-top album, record, day, period, and bank cards
    default: true
    answer: true
proposed_by: bbugyi200.apollo.research.6l.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 14:21:59
status: wip
---

- **PROMPT:**
  [prompts/202611/ref_cards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/ref_cards.md)

# Plan: Ref Cards

## Why

Bryan wants a Property Card for reference notes. It should hold as much useful
information about the reference as possible without clutter. It should support `==` if
that makes sense, get an excellent glossary term, and ship in parallel with the
in-progress `bob-cli-qm` epic (Notebook and Prose Cards).

**Required reading for every phase:** `research:202611/ref_cards/ref_cards.md`. Read it
with `sase artifact read`; if the research sidecar is not cloned, use
`sase artifact read file:explicit:4b16f0c00db1c7b9850e7343`. Bryan agreed with every
requirement it recommends, so this plan adopts its nine recommended defaults:

1. sibling epic;
2. `==` opens a Reference editor;
3. a Title edit retitles the matching H1;
4. optional `summary:` dek;
5. open by default with `_` compact;
6. per-kind tile icons with the grayscale escape hatch;
7. legacy refs included and muted;
8. the reading-task embed stays visible;
9. `!` on the roles commit.

Memory is the exception: it changes only under this plan's memory decisions. As lead
designer, this plan makes these changes to the report:

1. **No release wait.** The report had go-live wait for a crates.io release. Bryan's
   hosts track bob-cli master (`decisions:bob-cli-releases-ride-a-daily-train`), and the
   plugin probes the capability per host anyway. Go-live depends on `ref-props` landing,
   not on a release.
2. **A leaner editor.** The report also proposed Topics and Kind rows; both are dropped.
   No note in the vault uses `topics:`, so the editor will not invent a convention. The
   kicker already shows the kind, and the drawer explains that it follows the library
   folder.
3. **Authors is one text line.** All 33 `author` values are strings
   (`Jane Doe and John Roe`, `X et al.`), and the PDF marker stores a string. Authors
   has no list stage.
4. **PDF on every desktop.** `lib/` lives inside the vault. On macOS the PDF chip opens
   Highlights through the existing nav-hotkeys `api.openReferencePdf` (v3, already used
   by the Reading Board). The report's new seam is not needed. Elsewhere the chip opens
   Obsidian's PDF view.
5. **Report data never adds a line.** Reading-state facts patch rows that local data
   already laid out: the kicker chip, readout row, byline, title chip, and foot. So
   arrival never shifts layout, and no row is reserved and then collapsed on the 93% of
   refs without annotations.
6. **No legacy chapter counts.** bob reports none, and the JS never re-derives legacy
   status rules.
7. **Honest title edits.** A Title edit cannot change the reading task's link text in
   another note, because `bob props` is single-note. The save names that plainly and
   offers a jump to the task. A cross-note `bob ref retitle` is a follow-up.
8. **A third phase starts now.** `ref-service` needs nothing from the kit. So
   `ref-props`, `ref-kit`, and `ref-service` all start at once beside `qm`.

## Verified current state

Verified at bob-cli `c261c6a3` and bob-plugins `d190ba21`. Anchors drift: re-locate
every symbol on current master before editing.

- **Library.** `bob ref list -A -R all -f json` returns 1,378 rows in ~1.0 s as ~1.5 MB
  of JSON (`schema_version: 1`).
  - Statuses: 918 finished, 161 started, 227 queued, 70 dropped. By status field: 693
    `read`, 602 `legacy`, 50 `abandoned`, 28 `ready`, 5 `next`.
  - Kinds: chat 724, ai 280, docs 170, blogs 85, papers 67, zorg 14, books 11, slides 8,
    code 6, notes 5, forums 5, podcasts 2, meetings 1.
  - Populated row fields: `task` 776, `source_pdf` 776, `annotation_count`>0 98,
    `comment_count`>0 58, `author` 33, `audio` 29, `published` 28, `blocked` 27,
    `earlier` 3, `diagnostics` 2.
  - Rows also carry `reading_state`, `status`, `status_sync`, `era`, `origin`,
    `blocked_by[]`, `task{path, block_id, link, mark, archived, priority, scheduled}`,
    `finished`, `snapshot{highlights_count, synced_at}`, `identity{arxiv, doi}`,
    `reading` (reread number), `earlier[]`, and `coverage.ref_dir`.
  - Superseded readings are hidden from the list.
- **Local frontmatter.** `title` is on 1,321 of 1,380 files, so 59 need the H1-then-stem
  fallback. `created` is on 492, `captured` on 220, and `source_url` on 35; legacy refs
  use `url` (594). `agent` is on 27 chat reports. 608 legacy notes carry zorg provenance
  (`source_note`, `source_block(s)`, `source_id`, `source_path`, `source_line_*`).
  Nothing uses `summary:`, `aliases:`, or `topics:`.
- **The title lives in four places:** frontmatter, the H1 (bob writes it once at birth
  and never retitles it), the reading task's link alias in its residence note
  (`[[ref/…|Title]]`), and the PDF marker (`highlights_marker_base`).
- **props roles gap** (`bob-cli-qn`, READY). `src/native/props/roles.rs` assigns
  `NoteKind::Reference` from the type alone, so the 13 root `*_ref.md` hub notes qualify
  too. On a reference note it protects only `type` and `source_markdown`. Every other
  key is `loose`/`props`, including `id`, `status`, `parent`, `ref_type`, every
  `highlights_*` key, `source_pdf*`, and `pipeline_version`.
- **props contract.** It exists for area and project only (`contract/settings.rs`
  `for_kind`, `known_id`). The capabilities are `props_v1` and `props_contract_v1`
  (`show.rs`, `catalog/assemble.rs`). Effects already carry a `syncs` list.
- **Kit** (`plugins/bob-ledger-tools/src`):
  - `240f`: species registry, which now includes notebook and prose. `foldDefault: true`
    means open.
  - `240t`: holds `propertyCardNoteSpecies` (bank → album → record → area/project →
    temporal → notebook → prose) and the qm preview gate. **Trap:**
    `propertyCardKindLive` returns true for any registered kind other than notebook and
    prose, and qm.7 deletes it.
  - `240q`: hardcodes the kind list and mount-source attributes.
  - `240m`: hardcodes the silent-created set.
  - `240n`: the Managed fold.
  - `248m`: `registerPropertyCard` for `==` extensions.
  - `249i`: the notebook host pattern (qm.2 landed in `5152f493`).
  - `298l`: focus-or-add on loose keys.
  - `298v`–`298z`: settings-contract routing and stages.
- **Reading Board.**
  - `241` reads `bob ref list -s ready,next,wip -A -f json`.
  - `242` `READING_BOARD_KIND_MAP` covers only 7 kinds and humanizes the rest.
  - `244` shows PDF only with macOS and `api.openReferencePdf`.
  - Today comes from `isTodayCacheReady()` and `todayCache.rank`, with no spawn.
- **qm state.** `qm.1` is closed. `qm.2` has landed but is still in progress.
  `qm.3`–`qm.7` are in progress. `qm.6` gates on `qi.4` (closed); `qm.7` gates on `qi.6`
  and `qj.4`.

## Governing decisions

- `decisions:reference-ids-follow-file-stems` and
  `decisions:reference-renames-keep-identity`: `id` is identity. The editor's File name
  row routes to `bob ref rename` and never edits `id`.
- `decisions:reference-sources-are-preserved-write-once`: `source_markdown` stays
  derived.
- `decisions:open-pdf-writes-are-coordinated`: the card and editor never write a PDF. A
  marker-synced field (title, author, published, source URL) is saved note-side. The
  editor says it "syncs to the PDF marker at the next writing scan" and never claims the
  PDF changed.
- `decisions:task-status-is-derived` and `decisions:task-lanes-are-sticky`: the reading
  lane comes from the reading task. `status` and `parent` are owned by the task and are
  never edited on the card.
- `decisions:today-is-read-from-the-ledger`: the Today pill comes only from the
  in-memory ledger cache, never from a tag or a guess.
- `decisions:theme-hue-is-config-pin-exempt-area-hash`: hue says where. There is no hue
  per `ref_type`.
- `decisions:code-copy-reads-render-time-catalog`: the card never re-parses sources;
  there are no code or link counts.
- `decisions:references-are-ingested-at-birth`: the card never files, scans, or creates
  tasks.
- `decisions:q-and-ctrl-bracket-mirror-esc`: everything `==`/`+`/`-` opens closes with
  Esc, `q`, and Ctrl+`[` through the shared ladder.
- `decisions:relation-equals-alias-and-flips`: the capture-grammar fast `==` is a
  different surface and stays untouched.
- `decisions:mac-capture-is-a-thin-client`: no Bob Mac Capture or Bob Refs change.
- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` version or
  `CHANGELOG.md` edits. The roles commit is a contract break and takes `!`.
- `decisions:decision-rules-are-read-at-planning-time`: this plan read the roster. No
  record governs card species, and none is added.

## Design

### What the card answers

Every element earns its place by answering one of four questions, in this order:

1. **What is this?** Kind, title, dek.
2. **Where did it come from?** Author, source site, publication date, identifiers.
3. **Where am I with it?** Reading lane, wake date or prerequisite, Today, finished
   date.
4. **What useful material is here?** Annotations, PDF, audio, earlier readings.

Absence is the normal case, since most refs are finished agent reports with no author,
highlights, or audio. Every part is content-gated, and nothing ever paints a zero, a
ghost row, or a "create a task" nag.

### The card

A finished, annotated blog post:

```text
╭──────────────────────────────────────────────────────────────────────────╮
│ ▛▚ REF · Blog · ● SASE › SASE PyPI releases              ✓ Read · Oct 23 │
│ ▙▟  Agentic coding is straining CI. Here's how we scaled test impact     │
│     analysis at Anthropic                                                │
│     Sachin Malhotra · claude.com ↗ · Sep 14, 2026                        │
│     ✎ 5 annotations · 2 with notes                       ▤ PDF  ♫ Listen │
│ ──────────────────────────────────────────────────────────────────────── │
│   created Oct 22, 2026                             ▸ Properties · 14 − + │
╰──────────────────────────────────────────────────────────────────────────╯
  ![[done/sase_pypi_releases_done#^ref-agentic-coding-…]]   ← stays visible
```

An open paper that is in Today, with a dek and an identifier:

```text
│ ▛▚ REF · Paper · ● Bob › Memory                     ⭐ Next · ⏫ · TODAY │
│ ▙▟  Attention Is All You Need                                            │
│     Why transformers replaced recurrence: the paper behind every LLM.    │
│     Vaswani et al. · arxiv.org ↗ · Jun 2017 · arXiv:1706.03762 ⧉         │
│     ✎ 12 annotations · 4 with notes                               ▤ PDF │
```

A blocked agent report, and a sparse legacy note. Both are equally intentional:

```text
│ ▛▚ REF · Chat · ● SASE › SASE memory      💤 Snoozed · after 1 task      │
│ ▙▟  Shrinking SASE's Agent Instruction Files to ≤100 Lines: A Budgeted   │
│     Router, Not a Trim                                                   │
│     by research.5.final                                                  │
│     ✎ 1 annotation                                                ▤ PDF │

│ ▛▚ REF · Book · ● Mind › Outlive                     🗄 Legacy · started │
│ ▙▟  Outlive                                                              │
│     amazon.com ↗                                                         │
│ ──────────────────────────────────────────────────────────────────────── │
│                                                     ▸ Properties · 6 − + │
```

Compact (`_`) keeps only the kicker row and the title.

| Element           | Source                                                                      | Rule                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------------- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tile              | Kit + vocabulary                                                            | `ribbon` form: a bookmark with a notched lower edge. It stays distinct from `stack`, `sheet`, `leaf`, `plain`, `field`, `pennant`, `spine`, and `folio` in grayscale and `forced-colors`. It holds the kind's Lucide icon from the vocabulary (fallback `bookmark`). If acceptance's grayscale review blurs the species, switch to `bookmark` everywhere.                                 |
| Kicker            | Local `ref_type` + home-card parent resolver                                | `REF · <Kind>` (kind word from the vocabulary), then the parent's crumbs with the root-area hue pip. Show at most four crumbs, with the middle collapsed. A parent that is not a home is a plain link with no pip. Crumbs navigate; Mod opens a new tab.                                                                                                                                  |
| Lane chip (right) | Report row + today cache                                                    | See [Lane chip](#lane-chip). Muted styling for finished lanes; accent only for Reading and Next. Click opens the reading task (`task.link`). With no report row there is no chip.                                                                                                                                                                                                         |
| Title             | Local                                                                       | `title:` → first-body H1 → humanized stem (the index's precedence). Adopt and hide only a first-body H1 whose text equals the chosen title. A differing H1 stays visible as content. A Live Preview cursor reveals the raw line. `role="heading"`. A `vN` chip follows when the report's `reading` ≥ 2; its tooltip lists `earlier[]` labels, and click opens the newest earlier reading. |
| Dek               | Local `summary:`                                                            | Italic, clamped to two lines, with the full text in a tooltip. Absent means nothing (no ghost).                                                                                                                                                                                                                                                                                           |
| Byline            | Local `author`, `source_url`/`url`, `published`, `agent`; report `identity` | `Author · host ↗ · date`, each part omitted independently. Host is the URL hostname minus `www.`; ↗ opens the URL. The date uses the kit formatter: `2017`, `Jun 2017`, or `Sep 14, 2026`, and is never a file date. `agent:` renders as monospace `by <agent>` when there is no author. arXiv/DOI copy chips (⧉) append from the report. The row is absent when every part is absent.  |
| Readout row       | Local `source_pdf`/`audio`; report counts                                   | The row exists only when the note declares a PDF or audio, which are the only refs that can have annotations. Left: `✎ N annotations · M with notes` (from `annotation_count`/`comment_count`), shown only when nonzero. Click jumps to `## Highlights`; the tooltip says "From the Highlights scan on <date>; not live PDF state." Right: `▤ PDF` and `♫ Listen`.                        |
| Access            | Local paths + nav API                                                       | PDF: on macOS with `api.openReferencePdf`, open in Highlights; otherwise open the vault PDF in a new Obsidian tab ("annotate in Highlights on the Mac"). Listen opens the audio file in Obsidian. A declared PDF missing from the vault drops the chip and adds one foot lint line, "PDF not found at <path>".                                                                            |
| Foot              | Kit                                                                         | `created`, else `captured`; silent when both are missing. The lint chip holds the report `diagnostics` (each code and message), a `status_sync` conflict, and the missing PDF. Then `▸ Properties · N`, `−`, `+`.                                                                                                                                                                         |
| Drawer            | Kit inventory                                                               | Every key in file order. Bob-owned keys (the table in [ref-props](#phase-ref-props)) fold into the closed **Managed · N** group.                                                                                                                                                                                                                                                          |

<a id="lane-chip"></a>**Lane chip** (the Reading Board's words; report fields):

| Report state                                          | Chip                                                                                                                                     |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| open and `blocked`                                    | `💤 Snoozed · until Dec 3`, or `· after <prerequisite, truncated>` with a tooltip naming every reason (reuse the board's reason wording) |
| `wip`                                                 | `📖 Reading`                                                                                                                             |
| `next`                                                | `⭐ Next`                                                                                                                                |
| `ready`                                               | `📥 Ready`                                                                                                                               |
| open + `task.priority`                                | Append the priority glyph                                                                                                                |
| open + today cache ready and the task is ranked today | Append `TODAY`                                                                                                                           |
| `read`                                                | `✓ Read · <finished>` (date omitted when null)                                                                                           |
| `abandoned`                                           | `Dropped`                                                                                                                                |
| `legacy`                                              | `🗄 Legacy · started/queued/finished/dropped` from `reading_state` (bare when unknown)                                                   |
| `status_sync` ≠ `ok`                                  | `⚠ Status conflict` with bob's message                                                                                                  |

**Not on the card:** PDF thumbnails, highlight excerpts, raw URLs, hashes, page counts,
reading-time estimates, completion percentages, freshness or decay pressure, queue
membership, legacy chapter counts, or anything that writes.

**Beauty and reliability rules.**

- Finished refs are set in muted ink; tone comes from the report, falling back to local
  `status` for tone only.
- The card is open by default, and `_` compacts it per note for the session.
- Narrow panes (~420 px): the lane chip takes its own reserved line under the kicker,
  and byline parts wrap.
- Check light and dark, `forced-colors`, and grayscale beside album, notebook, prose,
  and area cards.
- A scan rewrite, embed heal, or typing while the note is open patches chips. It never
  unmounts the card or flashes native Properties. Stale report data never tears down a
  valid title shell.
- Accessible name: `Ref: <title>`.

### Kind vocabulary

One shared vocabulary (`ref-kit`) feeds both the card and the Reading Board. It holds
the word, the board emoji, and the card's Lucide icon:

| `ref_type`   | Word                  | Emoji | Icon              |
| ------------ | --------------------- | ----- | ----------------- |
| papers       | Paper                 | 📄    | `graduation-cap`  |
| blogs        | Blog                  | 📰    | `newspaper`       |
| chat         | Chat                  | 💬    | `bot`             |
| ai           | AI                    | 🤖    | `sparkles`        |
| docs         | Docs                  | 📚    | `book-open-text`  |
| books        | Book                  | 📕    | `book`            |
| slides       | Slides                | 🖼️    | `presentation`    |
| code         | Code                  | 🧩    | `code-xml`        |
| notes        | Note                  | 🗒️    | `file-pen`        |
| forums       | Forum                 | 🗣️    | `messages-square` |
| podcasts     | Podcast               | 🎙️    | `podcast`         |
| meetings     | Meeting               | 🤝    | `users`           |
| zorg         | Zorg                  | 🗄️    | `archive`         |
| other / none | humanized dir / `Ref` | 🔖    | `bookmark`        |

Every icon must exist in Obsidian's bundled Lucide set (`getIconIds()`); any missing
icon falls back to `bookmark`. The board's existing seven words and emoji stay
unchanged.

### Eligibility and precedence

- **Claim rule.** Ref claims a note only when two things hold:
  - its `type` is exactly `[[ref]]` (scalar, or a single-item list);
  - its path is inside bob's library coverage: under `ref/` (the report's
    `coverage.ref_dir` once known), excluding hidden directories, `*.assets/`,
    `_generated/`, `_templates/`, `_conflicts/`, and conflict copies.
- **What it never claims:** the 13 root `*_ref.md` hub notes, `ref_history/` captures,
  and mixed types.
- **Precedence.** `ref` slots after `prose` in `propertyCardNoteSpecies`. Extend the
  exhaustive qm.1 precedence test with ref, so ref claims exactly the library notes and
  at most one species claims any note. bob-cli scopes `NoteKind::Reference` the same way
  (`ref-props`).

### Data

- **Local identity.** Tile, kicker, crumbs, title, dek, byline, access chips, and the
  foot paint synchronously from local frontmatter and the metadata cache. A missing,
  failed, or stale report still gives a correct quiet card.
- **Shared report.** `ref-service` keeps one shared snapshot, and `ref-readouts` patches
  it in. The service:
  - subscribes lazily on the first mount and holds nothing while no card is mounted;
  - keeps a `Map<path, row>` of decoded fields;
  - serves a stale snapshot immediately, then revalidates;
  - runs one spawn in flight, debounced at 1500 ms, with one trailing run;
  - is released on unload.
- **Revalidation triggers.** Only these:
  - the first subscription when there is no snapshot or it is older than 60 s;
  - a metadata `changed` event on a mounted ref, or on the residence note of a mounted
    ref's reading task;
  - a rename or delete of a mounted ref;
  - opening a ref note when the snapshot is older than 5 minutes.

  Painting, folding, scrolling, cursor motion, drawer toggles, and unrelated vault
  events never spawn bob.

- **Failure.** A failed refresh keeps the last good snapshot and changes no chips.

### The Reference editor

`==` registers the `ref` kind with `registerPropertyCard`. With the capability
`props_ref_contract_v1` on the note, `==` opens the shared Property Editor in settings
mode on bob's `reference` contract, the way area and project `==` does. There is no new
modal. Without the capability (an older bob on that host), `==` runs qm.1's focus-or-add
on `title`.

```text
REFERENCE
  Title        Attention Is All You Need                  ↻ PDF marker
  Summary      Why transformers replaced recurrence…
  Authors      Vaswani et al.                             ↻ PDF marker
  Published    2017-06                                    ↻ PDF marker
  Source       https://arxiv.org/abs/1706.03762           ↻ PDF marker
  Aliases      Transformer paper
OWNED ELSEWHERE
  Reading      ⭐ Next · task in memory.md                ↵ open the reading task
  Lives under  [[memory]] · follows the reading task      ↵ open the reading task
  File name    attention_is_all_you_need                  ↵ rename…
```

The sketch shows content; the look follows the existing settings rows.

| Row         | Key                           | Operations and validation                                                                                                                                                                                                                                                                                                                         |
| ----------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Title       | `title`                       | Set only; one nonblank line. When the first body H1 (outside fences and comments) equals the old title, the same atomic write retitles it. Effects say "Syncs to the PDF marker at the next writing scan". When a reading task exists, they also say "The reading task keeps its link text; open it to retitle", and the editor offers that jump. |
| Summary     | `summary`                     | Set or clear. One line; warns over 140 chars. Note-local and never marker-synced.                                                                                                                                                                                                                                                                 |
| Authors     | `author`                      | Set or clear. One text line, as the byline prints it. Marker-synced.                                                                                                                                                                                                                                                                              |
| Published   | `published`                   | Set or clear. Accepts `YYYY`, `YYYY-MM`, or `YYYY-MM-DD` (only the forms bob's marker and `ref list` already parse). It never coerces the global property type. Marker-synced.                                                                                                                                                                    |
| Source      | `source_url`, or legacy `url` | Set only. Writes whichever key the note already uses (else `source_url`) and validates HTTP(S). Marker-synced when the note has a marker.                                                                                                                                                                                                         |
| Aliases     | `aliases`                     | The existing list stage.                                                                                                                                                                                                                                                                                                                          |
| Reading     | `status` (shown)              | Read-only. Enter opens the reading task (the note's managed embed target); with no task, a notice says the note is a legacy reference.                                                                                                                                                                                                            |
| Lives under | `parent` (shown)              | Read-only. Enter opens the reading task; the notice says Ctrl+Shift+M moves it.                                                                                                                                                                                                                                                                   |
| File name   | stem (shown)                  | Read-only. Enter opens Command Mode on `ref rename` with this reference prefilled when `api.commandMode` supports it; otherwise it copies `bob ref rename <stem> <NEW>` with a notice.                                                                                                                                                            |

- **Refused everywhere in this editor:** adding, removing, reordering, or retyping
  fields; any edit to `id`, `status`, `parent`, `ref_type`, pointers, hashes, or
  `pipeline_version`; and any PDF write, scan, task creation, or bundle move as a side
  effect.
- **Writes.** Every write is one previewed, undoable `bob props update`. Sync
  bookkeeping never appears as a row.
- **Keys.** `+` (the generic Property Editor) and the `y` YAML stage remain the general
  and escape paths. bob's roles teach and protect the owned keys there.

### Preview gate

- **The gate.** A ref-owned switch keeps Ref Cards off in the live vault until
  `ref-go-live`. It is `refCardPreview: true` in bob-ledger-tools plugin data, read once
  at load into a ref-owned function in a ref-owned fragment.
- **What it covers.** It gates the mount, the `==` registration, and every ref UI.
- **Isolation from qm's gate.** It never reads or writes `cardPreviewKinds`, and ref
  code never calls `propertyCardKindLive`, which qm.7 deletes. A closed gate gives
  exactly today's rendering: native Properties and the plain H1.
- **Removal.** Acceptance writes the key into its disposable vault, and Bryan can set it
  by hand to preview. `ref-go-live` deletes the gate, the key, and their tests.

## Parallel delivery with bob-cli-qm

Coordination is needed, but only at four seams. Everything else is bob-cli Rust that qm
never touches, or new ref-owned fragments.

| Seam                                                                                                                     | Failure if ignored                                                                | Handling                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Shared kit files: `240f`, `240g`, `240m`, `240q`, `240t`, `styles.css`, `fragments.json`, `package.json`, `374*` exports | Concurrent rewrites; qm.7's gate deletion in `240t`; an accidentally live card    | `ref-kit` is the only ref phase that edits kit logic. Its edits are additive hunks, and while the gate is closed they are behavior-neutral for every existing species. The unchanged existing suites prove that. No descriptor-driven refactor of the hardcoded lists happens while qm is in flight (follow-up). Later phases only add their own registry lines (`fragments.json`, `package.json`, exports). On a rebase conflict, take upstream, re-apply the additive line, rebuild, and re-bump. |
| Activation                                                                                                               | `propertyCardKindLive` would ship any registered non-notebook/prose kind live     | The ref-owned gate (above). `ref-kit` records `COORDINATION:` notes on `bob-cli-qm.6` and `bob-cli-qm.7` naming ref's precedence hunk, coordinator entry, and gate fragment. qm's gate deletion must leave them intact.                                                                                                                                                                                                                                                                             |
| Real-app acceptance                                                                                                      | Concurrent harness edits mid-qm.6, and screenshots without finished sibling cards | `ref-acceptance` waits for `bob-cli-qm.6` to close done, then extends the harness as a sibling runner that imports its lib.                                                                                                                                                                                                                                                                                                                                                                         |
| `glossary:property-card`, family docs, deploy                                                                            | Two go-lives editing one strand and the same docs                                 | `ref-go-live` waits for `bob-cli-qm.7` to close done, and edits the strand and docs as qm.7 left them.                                                                                                                                                                                                                                                                                                                                                                                              |

`qj` has no seam; refs never touch prose. `qi` has none beyond what qm.7 already waits
for.

**Gate procedure.** A gated phase runs step 0 before changing anything:

1. Poll the non-audited `sase bead list --status closed --type phase -f json` for the
   gate bead under `/sase_monitor`. Allow 48 h for `bob-cli-qm.6` and 72 h for
   `bob-cli-qm.7`.
2. When the bead shows closed, run `sase bead read <id> -r "<why>"` once. Confirm it
   closed done and not canceled.
3. If it was canceled, or the wait timed out, stop without changes. Record a note on
   your phase bead for the land agent.

Launch-time `-w bead=` is not used.

## Rules every phase follows

- **Repos.**
  - bob-cli is the primary checkout; `ref-props` and `ref-go-live` edit it.
  - Work in bob-plugins through `/sase_repo` (`sase repo open bob-plugins -r "<why>"`),
    use only the printed path, and read its `AGENTS.md`.
  - Before any vault read, read the `obsidian.md` reference memory. Never open the live
    vault in an automated Obsidian run.
- **Fragments.** Edit `src/` fragments. Keep each fragment at or below 1000 lines, run
  `npm run build`, and never hand-edit generated `main.js`.
  - Prefixes: `refCard*` and `refNoteCard*` in JS, `.bob-ref-card*` in CSS, and
    `refLibraryReport*` for the service. Bare `ref*` collides with half the codebase.
- **Rebase discipline.** On a conflict in `main.js`, `styles.css`, `fragments.json`,
  `package.json`, or a manifest version: take upstream, re-apply your additive lines,
  rebuild, and re-bump. Never hand-merge a bundle.
- **Per plugin change.**
  1. Bump the manifest per repo convention.
  2. Run `npm run build`, `npm run build:check`, the focused suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`, following the `AGENTS.md` refusal protocol.

  The closed gate makes these syncs safe. Pre-existing `npm test` failures that
  reproduce on clean master are recorded, not fixed.

- **bob-cli changes.** Run `just fmt`, `just lint`, and `just test`. Never edit the
  `Cargo.toml` version or `CHANGELOG.md`.
- **Tests and evidence.** Cover every model and DOM with unit suites against the shared
  fixtures. Painting tests spy on spawns and vault writes and expect zero. Retain
  real-app evidence with `sase artifact create --bead`.
- **Beads.** Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` on your
  phase bead instead. Run `sase bead epic-symbols <phase>` before closing.
- **Long work.** Use `/sase_monitor` for builds, real-app runs, and gate waits. Finish
  through `/sase_final`.

## Phase `ref-props`

<a id="phase-ref-props"></a>All work is in bob-cli, `src/native/props/`.

1. **Scope the kind.** `NoteKind::Reference` applies only when the type contains `ref`
   and the path is inside bob's library coverage. Use the same membership predicate as
   `bob ref list` (`ref_library`), with the configured ref dir; never a second copy of
   the rules. Hub notes and `ref_history/` files become `Note`.
2. **Roles** for covered references:

   | Keys                                                                                                               | Role                                                                                                                       | Owner and teaching reason                                                                                              |
   | ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
   | `id`                                                                                                               | identity                                                                                                                   | "Follows the file stem; rename the reference with `bob ref rename`."                                                   |
   | `ref_type`                                                                                                         | identity                                                                                                                   | "Follows the library folder its PDF lives in."                                                                         |
   | `status`, `parent`                                                                                                 | derived, only when the note carries its managed reading-task embed or a v1 `^ref` tracker (a local check, no library scan) | Reading task: "Mirrors the reading task's checkbox", and "Projected from where the reading task lives; move the task". |
   | every `highlights_*` key, `source_pdf`, `source_pdf_sha256`, `pipeline_version`                                    | derived                                                                                                                    | Highlights sync: "Sync bookkeeping; `bob ref scan` maintains it."                                                      |
   | `audio`, `return_links`, `earlier`, `recaptured` (with the existing `source_markdown`)                             | derived                                                                                                                    | Reference importer / reread lineage                                                                                    |
   | `source_note`, `source_block`, `source_blocks`, `source_id`, `source_path`, `source_line_start`, `source_line_end` | derived                                                                                                                    | `bob ref migrate-zorg` provenance                                                                                      |
   | `event`, `event_id`, `meeting_start`, `meeting_end`, `notes_producer`                                              | derived                                                                                                                    | `bob ref meet`                                                                                                         |
   - **Stay loose:** `title`, `summary`, `author`, `published`, `source_url`, `url`,
     `aliases`, `captured`, `created`, `tags`, `agent`, `legacy_status`,
     `legacy_chapter_statuses`, and unknown keys.
   - **Writer-owned keys.** Before finalizing, grep bob's reference writers (create,
     sync, scan, meet, migrate-zorg, overwrite, rename) for frontmatter keys they own
     and later read or merge. Classify any key the table misses, with evidence in the
     commit body.
   - **Every mutation path.** The policy binds set, unset, rename, move, restore/undo,
     and YAML replacement. YAML replacement follows the existing precedent for `type`
     and project `status` exactly, so the `y` YAML stage stays the deliberate escape.
   - **Internal writers.** Verify that no internal bob writer (sync, rename, overwrite)
     routes through the role checks.
   - **Commit.** This commit is `fix(props)!:`, because formerly accepted generic writes
     now refuse and role values change in the JSON. Close `bob-cli-qn` with
     `sase bead close bob-cli-qn --note "<commit + verification>"`.

3. **Contract.**
   - **Shape.** `bob props show` on a covered reference adds
     `contract: {kind: "reference", settings: [...]}` in the area-contract row shape.
     The editable rows are title, summary, author, published, source, and aliases (see
     [The Reference editor](#the-reference-editor)). The owned rows `reading`, `parent`,
     and `file_name` carry `disabled: true`, a `reason`, a `route` (`reading_task` |
     `ref_rename`), and a `target` (the managed embed's link, when present).
   - **Capability.** Announce `props_ref_contract_v1` in that note's capabilities and in
     `bob __catalog` features.
   - **Settings ops.** `setting.set` and `setting.clear` accept the reference ids, with
     the per-row validation above. Marker-synced rows emit effects with
     `syncs: ["pdf_marker"]`.
   - **Title.** Title retitles a matching first body H1 in the same atomic write,
     respecting BOM, CRLF, fences, and comments. This deliberately widens props'
     frontmatter-only statement for this one setting; document it and test it. Title
     also emits the reading-task link-text effect.
   - **Compatibility.** Verify that the current plugin ignores an unknown contract kind
     (it must route only on the new capability). This commit is additive:
     `feat(props):`.
4. **Docs.** Update `docs/props.md` (reference kind, coverage scope, role table,
   contract, the H1 exception) and any JSON reference page that lists capabilities. Run
   `just docs-gen` and `just docs-check`.
5. **Tests.**
   - Roles per key, for modern, legacy, v1-tracker, hub, `ref_history`, and `*.assets`
     notes.
   - Refusal on every mutation path, and the YAML precedent.
   - Contract rows and capability presence (covered refs only); the catalog feature.
   - Set/clear validation per row; marker-sync effects.
   - H1 retitle: match, mismatch, no H1, fenced H1, CRLF, BOM, no title.
   - Golden JSON updates.

## Phase `ref-kit`

Owner of every shared-kit edit for ref. Later phases fill the regions this phase
creates.

1. **Species.**
   - In `240f`, add `ref` (word `Ref`, tile `ribbon`, icon `bookmark`,
     `foldDefault: true`) to the registry and `PROPERTY_CARD_KINDS`.
   - Let the title-block builder (`240g`) accept a per-card icon override, the way qm.1
     added the folio initial.
   - Add the `ribbon` tile form to `styles.css` in the "Note cards" region, plus an
     empty `Ref Card` sub-region.
   - Verify light, dark, forced-colors, and grayscale distinction against every existing
     tile.
2. **T1 and T3.**
   - `240m`: missing `created` is silent for `ref`.
   - `240q`: the `ref` kind, a `data-ref-source` attribute, and the `props_v1`-only
     gate.
   - Add coordinator tests.
3. **Predicates and precedence** (`240t`).
   - Add the exact-`[[ref]]` predicate and the coverage path predicate from
     [Eligibility](#eligibility-and-precedence).
   - Slot `ref` after `prose` in `propertyCardNoteSpecies`.
   - Extend the exhaustive test with `ref`, hub-note, `ref_history`, `*.assets`, and
     mixed-type cases.
   - Touch nothing in qm's gate code.
4. **Preview gate.** In a new ref-owned fragment, add `refCardPreviewLoad(plugin)` /
   `refCardPreviewLive(plugin)` reading `refCardPreview` once at load. Test both states,
   and prove `cardPreviewKinds` is ignored.
5. **Vocabulary.** Add a new fragment, ordered in `fragments.json` before `241`, that
   holds the [Kind vocabulary](#kind-vocabulary) and the lane words.
   - Rewire `READING_BOARD_KIND_MAP` (`242`) to derive from it. Board output for its
     seven kinds stays byte-identical, and the other kinds gain their words.
   - Add an icon-existence test hook.
6. **Scaffolding.** Create fragment stubs with `fragments.json` entries after the
   prose-card fragments:
   - ref card: model, DOM, host, foot, readouts, editor routing;
   - the service: `refLibraryReport*`, placed with the other `241a*` data services;
   - `374i-exports-ref-cards.js` for test hooks.

   Also create:
   - skeleton tests registered in `package.json` `test:unit`;
   - `docs/ref-card.md` with a "Preview — not yet enabled" admonition, registered in
     `docs/nav.yml` and the docs inventory.

   Record the exact file names on this bead.

7. **Fixtures.** Add `scripts/fixtures/ref-cards/` with live-shaped notes, plus a
   trimmed `ref-list.json` whose rows match them:
   - an annotated finished blog (author, published, `source_url`, audio, created,
     matching H1, managed embed);
   - an open Next chat report with `agent:` and a priority;
   - a blocked chat report (local dependency) and a scheduled-blocked paper;
   - an arXiv paper with identity;
   - a reread pair (`x`, `x__v2`) and an `earlier` replaced capture;
   - a v1 tracker;
   - a legacy zorg book (provenance keys, `legacy_status`);
   - a legacy url-only `ai` ref;
   - an H1 ≠ title note, and a no-`title` note;
   - a declared-but-missing PDF;
   - a meeting ref;
   - the non-claims: a root hub `x_ref.md`, a `[ref, project]` mix, `ref/x.assets/`, and
     `ref_history/`.
8. **Coordination.** After landing, add `COORDINATION:` notes on `bob-cli-qm.6` and
   `bob-cli-qm.7` with `sase bead note`, naming the files and hunks above.
9. Build, test, and sync. Expect no visible change: the gate is closed, and the board's
   seven kinds are unchanged.

## Phase `ref-service`

1. **Data path** (the stub fragment if `ref-kit` has landed; otherwise create a
   `refLibraryReport*` fragment and re-apply on rebase).
   - **Argv.** `["ref", "list", "-A", "-R", "all", "-f", "json"]`, through the existing
     bob spawn helper (no shell; the standard timeout and max buffer).
   - **Decode.** Validate `schema_version: 1` and decode the fields the
     [card](#the-card) uses into a frozen `Map<path, row>` plus `coverage.ref_dir` and
     `generated_at`.
   - **Old or partial bob.** An older bob's missing fields normalize to absent, never to
     zero.
2. **Lifecycle.** Follow the [Data](#data) rules:
   - `apiRefLibrarySubscribe(fn)` returns an unsubscribe;
   - `refLibraryReportRow(path)` and the snapshot age;
   - stale-while-revalidate, one in flight, the 1500 ms debounce with a trailing run,
     the narrow triggers, release on unload, and no work while there are no subscribers;
   - a failed refresh keeps the last good snapshot.
3. **Tests** (with `scripts/fixtures/fake-bob.cjs`):
   - decode of every fixture row, bad JSON, and an old schema;
   - debounce, coalescing, and the in-flight trailing run;
   - triggers fire only for mounted refs and their residence notes;
   - unrelated vault events spawn nothing; unload releases everything;
   - stale delivery before revalidation.
4. Build, test, and sync. No visible change.

## Phase `ref-card`

1. **Model (pure).** Derive from local frontmatter, the metadata cache, and the
   home-card parent resolver:
   - the kind word and icon (vocabulary), crumbs, and hue;
   - the title and its adoption range;
   - the dek and the byline parts (host parsing, date precision, `agent`);
   - the access chips (macOS + nav API vs Obsidian, missing-PDF lint);
   - `created`/`captured`, the tone fallback, and the Managed key set (mirroring the
     `ref-props` table, with a test naming that table).

   Test it against every fixture.

2. **DOM.** Build the kit title block per the [card](#the-card) table:
   - the ribbon tile with its icon, the kicker, title, dek, and byline;
   - a readout row only when a PDF or audio is declared;
   - empty slots for the lane chip, `vN`, and identifiers;
   - narrow-pane reservation for the lane line.

   Text only; everything reachable by accessible name.

3. **Host.**
   - Mount on `createPropertyCardHost` for Reading and Live Preview, behind
     `refCardPreviewLive` and the precedence function, and never `propertyCardKindLive`.
   - Register with the coordinator (`ref`, `data-ref-source`).
   - Own the title with matching-H1 adoption.
   - Wire the foot (created, lint slot, drawer with Managed), the `_` compact (default
     open, per note for the session), and `==` through `registerPropertyCard` (gated)
     running focus-or-add on `title`. `+` and `-` come from the kit.
   - The managed reading-task embed stays in the body, untouched.
4. **Access actions.** PDF and Listen as designed. There are no frontmatter-built shell
   strings; paths open through Obsidian or `api.openReferencePdf`.
5. **Docs.** Fill `docs/ref-card.md` (behavior, keys, mount rules, fallback), keeping
   the preview admonition.
6. Build, test, and sync.

## Phase `ref-readouts`

1. **Subscribe.** Use one plugin-level `apiRefLibrarySubscribe` subscription that
   patches every mounted Ref Card in place. It holds generation-bound handles across
   split panes and is released on unload.
2. **Reading model (pure).** Map a report row plus today context to:
   - the [Lane chip](#lane-chip), reusing the board's reason wording and the existing
     today cache (`isTodayCacheReady`, `todayCache.rank`, keyed exactly as the board
     keys it);
   - annotation counts;
   - `vN` and earlier readings;
   - arXiv/DOI chips;
   - diagnostics and `status_sync`;
   - tone.

   With no row there are no chips. Test every fixture row.

3. **Patch.** Fill only existing rows: the kicker chip, readout counts, the title `vN`,
   byline identifiers, and the foot lint. Clicks:
   - the chip opens `task.link`;
   - the counts jump to `## Highlights`;
   - `vN` opens the newest earlier reading;
   - identifiers copy with a notice.
4. **Proofs.**
   - Spy tests: zero spawns and zero writes on paint, fold, `_`, resize, cursor motion,
     and drawer toggles.
   - Layout tests: report arrival never changes row count or height (wide and narrow).
   - A stale or failed report keeps the shell.
5. Update the docs page. Build, test, and sync.

## Phase `ref-editor`

1. **Routing.** Extend the settings-contract routing (`298v`–`298z`) for contract kind
   `reference`, gated on the note's `props_ref_contract_v1`. The `==` open path chooses:
   - the settings mode when the capability is present;
   - qm.1 focus-or-add on `title` when it is absent or pending.

   An unknown contract kind is ignored.

2. **Rows.**
   - Render bob's REFERENCE rows with the existing stages (summary-style line, text,
     list).
   - Render the owned rows read-only, each with bob's reason. Enter follows the row's
     `route`:
     - `reading_task` opens the `target`, or shows the legacy notice;
     - `ref_rename` opens Command Mode prefilled when `api.commandMode` supports it,
       otherwise copies the command.
   - Show bob's effects in the preview, including the PDF-marker sync line and the
     reading-task link-text line with its jump.
3. **Ladder.** Esc, `q`, and Ctrl+`[` back out of stages and close the editor. Undo
   restores through `bob props`.
4. **Tests.**
   - Capability present, absent, and pending.
   - Each row's set and clear, refusals, the Title H1 effect, and the link-text jump.
   - Routes with and without Command Mode.
   - The ladder.
   - A stubbed bob for both contract versions.
5. Update `docs/ref-card.md` (the `==` section). Build, test, and sync.

## Phase `ref-acceptance`

0. **Gate.** Wait for `bob-cli-qm.6` per the
   [gate procedure](#parallel-delivery-with-bob-cli-qm).
1. **Setup.**
   - Use a bob built from current bob-cli master (containing `ref-props`), through
     `BOB_BIN` and the runner's provenance contract.
   - Use an isolated Obsidian profile with a disposable Git vault built from the
     fixtures, with `refCardPreview: true`. On athena, read the `tailnet.md` reference
     memory first.
   - Add a sibling runner that imports
     `scripts/obsidian-frontmatter-takeover-e2e-lib.cjs`. Never fork the harness.
2. **Required evidence.**

   | Concern            | Evidence                                                                                                                                                                                            |
   | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Takeover           | In LP and Reading, every claimed fixture shows one card, one title, one foot, native Properties hidden, the matching H1 hidden, and the embed visible.                                              |
   | Single owner       | Hub, mixed, `*.assets`, and `ref_history` notes mount nothing. Every other species is unchanged. A closed gate gives exactly today's rendering.                                                     |
   | Native restoration | Source, embed, hover, canvas, print, mobile emulation, bad YAML, a source/cache mismatch, rename and scan-rewrite while open, a failed mount, unload, and a disabled ledger-tools.                  |
   | Capability         | `props_v1` pending, absent, or failed. A bob without `props_ref_contract_v1` gives the title fallback. An absent or failing report gives an identity-only card. Late and stale snapshots.           |
   | Data               | No zeros or ghost rows. Arrival keeps layout. Lane chips for every fixture state. Today from the ledger. Annotation jump, `vN`, identifier copy, PDF and Listen on Linux, and the missing-PDF lint. |
   | Editor             | Every row set and clear, the Title H1 retitle, PDF-marker and link-text effects, owned-row routes, refusals, and the Esc/`q`/Ctrl+`[` ladder.                                                       |
   | Keys and focus     | `==`, `+`, `-`, and `_` in LP Vim normal mode. Vim `k`/`gg` and Up at body start. Tab.                                                                                                              |
   | Passive            | No paint-, fold-, drawer-, or resize-triggered spawns or writes. Protected fixture hashes and PDF bytes are unchanged.                                                                              |

3. **Screenshots.**
   - Capture light/dark × narrow (~420 px)/wide × LP/Reading for the finished blog, Next
     paper, blocked report, legacy book, and compact state. Shoot them beside album,
     notebook, prose, and area cards, plus the native-recovery states.
   - Check forced-colors and grayscale species distinction; this settles per-kind icons
     versus `bookmark`.
   - Inspect every image and record a verdict, size, and digest.
4. **Repairs.** Repair defects in their owning repo (build, test, sync), then rerun the
   affected rows. The phase closes only on GO.

## Phase `ref-go-live`

0. **Gate.** Wait for `bob-cli-qm.7` per the
   [gate procedure](#parallel-delivery-with-bob-cli-qm). Re-read
   `glossary:property-card`, `docs/develop/architecture.md`, and
   `docs/property-editor.md` as qm.7 left them.
1. **Switch on.** Delete the ref preview gate, the `refCardPreview` handling, and their
   tests. Ref Cards are now live.
2. **Passive proof.** Extend the family passive-rendering suite: toggle ref folds and
   drawers many times and force snapshot refreshes, asserting zero writes and zero
   paint-triggered spawns.
3. **Docs (bob-plugins).** Read the `docs.md` reference memory first.
   - Remove the preview admonition.
   - Update `docs/property-editor.md` (`==` on refs: Reference editor or fallback),
     `docs/bob-ledger-tools.md`, `docs/develop/architecture.md` (species list,
     precedence, the ref gate history), the Reading Board doc (vocabulary), and the
     related-cards links.
   - Run `npm run docs:check`.
4. **Docs (bob-cli).**
   - `docs/ref.md`: point to the Ref Card page.
   - `docs/reference/vault-format.md`: document optional `summary:` on references.
   - `docs/highlights-ref-sync.md`: protected keys, and note-side edits syncing to the
     marker.
   - Run `just docs-gen` and `just docs-check`. A doc already over its word budget must
     not grow.
5. **Memory.** Use `/sase_memory_write`, apply only the accepted decisions, then run
   `sase memory init`, `just docs-gen`, and `just docs-check`.

   > [!decision] memory_ref_card Add `glossary:ref-card` (keyword "Ref Card", aliases
   > "reference card", "ref title card"). Adjust this draft to the shipped behavior:
   >
   > The note-top [[property-card]] for a library [[reference-note]] (exact
   > `type: "[[ref]]"` under `ref/`). Bob Ledger Tools paints it in desktop Live Preview
   > and Reading view. It answers what the reference is, where it came from, where Bryan
   > is with it, and what it holds, and it omits whatever a note lacks; nothing is
   > zeroed.
   >
   > - **Identity (local, paints first).** A ribbon tile bearing the kind's icon and a
   >   `REF · <Kind>` kicker with the parent's crumbs in the root hue lead the title
   >   (`title:`, else the matching first H1, which hides, else the stem). An optional
   >   one-line `summary:` dek follows, then a byline of author, source host ↗, and
   >   publication date (never a file date), and PDF and Listen chips.
   > - **Reading (one shared `bob ref list` snapshot).** The reading lane appears in the
   >   Reading Board's words (Reading, Next, Ready, Snoozed with its wake date or
   >   prerequisite, Read with its date, Dropped, Legacy). A Today pill comes from the
   >   ledger, and annotation counts come from the last Highlights scan (never reading
   >   progress). Reread lineage, arXiv/DOI chips, and bob's diagnostics sit in the
   >   foot. Report data patches existing rows and never adds a line.
   > - **Keys.** `==` opens the Reference editor (Title, Summary, Authors, Published,
   >   Source, Aliases). It routes reading state, residence, and file name to the
   >   [[reference-task]] and `bob ref rename`. `+` opens the Property Editor; `-` opens
   >   the drawer, where bob-owned sync bookkeeping folds under Managed; `_` compacts
   >   the card to kicker and title.
   > - **Passive.** Painting never writes, spawns bob, or touches a PDF. Native
   >   Properties return on uncertain evidence, and the reading-task embed stays visible
   >   below the card.
   >
   > Not the [[ref-panel]] (Bob Refs), the Reading Board, a reading task, a PDF preview,
   > or a schema editor. Root `*_ref` hub notes get no card. See
   > `docs/reference/plugins/ref-card.md`.

   > [!decision] memory_property_card Edit `glossary:property-card` as qm.7 left it:
   >
   > - Add the Ref Card to the family list, linking `[[ref-card]]`.
   > - State that it gates on `props_v1` alone and registers `==` as an extension: the
   >   Reference editor under `props_ref_contract_v1`, else focus-or-add on `title`.
   > - State that it hides a matching first H1 like the other title-owning cards.
   >
   > Change nothing else.

6. **Deploy and provenance.**
   - Run `npm run build:check` and `npm test`.
   - Run `bob plugins sync` from clean master, then the isolated post-sync startup smoke
     in `docs/develop/deployment.md`. `bob plugins list` must show no drift.
   - Record on the bead: the bob-cli and bob-plugins SHAs, the plugin versions, and the
     acceptance evidence refs.
7. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` entries on this bead for the
   [follow-ups](#follow-ups).

## Epic acceptance (land agent)

- **References.** Every live library reference opens on one Ref Card in desktop LP and
  Reading:
  - native Properties are hidden; the matching H1 is hidden; the embed is visible;
  - the lane chip matches `bob ref list`;
  - annotations, PDF, and Listen work;
  - legacy notes are calm and muted.
- **Non-claims.** Hub notes and mixed types are untouched.
- **Editor.** `==` opens the Reference editor on a bob carrying `ref-props`. Title
  retitles the matching H1, and owned rows route correctly. On an older bob it falls
  back to title focus-or-add.
- **bob protection.** `bob props` refuses generic edits to reference sync and identity
  keys, and `bob-cli-qn` is closed.
- **Fail-open.** Fail-open, single-owner, and passive evidence is GO.
- **Clean state.** The ref gate is gone, the docs in both repos are consistent, memory
  matches the accepted decisions, and the deploy is clean.
- **Land report.** It reminds Bryan to update bob from master and run `bob plugins sync`
  on the Mac and apollo, then to glance at a finished blog, an open paper, and a legacy
  ref in his live vault.

## Out of scope

- Any Bob Mac Capture or Bob Refs change.
- Mobile cards.
- Writing `summary:` or any other content into existing vault notes.
- Retyping or re-homing the 13 root `*_ref` hub notes.
- New `bob ref list` fields.
- Cross-note title writes.
- Refactoring kit lists while qm is in flight.

## Follow-ups

Proposed by `ref-go-live`; the land agent decides:

- **F1.** Retype or re-home the root `*_ref` hub notes (Bryan's call).
- **F2.** The Reading Board consumes the shared ref-library snapshot.
- **F3.** Dock the reading-task embed into the card, if screenshots show the duplication
  hurts and a safe cross-note lane writer exists.
- **F4.** bob-supplied reading-time/page estimates and an agent-report "Bottom line"
  excerpt as additive `ref list` fields.
- **F5.** Code-block and link counts from the saved-source catalog.
- **F6.** `bob ref retitle`: title, matching H1, and the reading task's link alias in
  one previewed cross-note write, which the editor's Title row would then use.
- **F7.** Make the kit's hardcoded species lists (`240m` silent set, `240q`
  kinds/attributes) descriptor-driven from `240f`, after both go-lives.
