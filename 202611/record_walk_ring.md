---
tier: epic
title: "Record walk: ‹ › links on every record, N of M painted live"
goal:
  Every record note carries one bob-maintained `‹ prev · ↑ album · next ›` line that
  forms a cycling alphabetical ring per album, `[n` / `]n` follow it with no key
  changes, and Obsidian paints `N of M` beside it on a record-sheet rail and in the `n`
  links footer, without ever writing a count into Markdown.
phases:
  - id: walk-core
    title: Pure ring, line grammar, and edit planner
    size: medium
    depends_on: []
    description:
      "walk-core: add the pure src/native/albums/walk.rs module (shared title order,
      ring math, label sanitizer, walk-line render/recognize/classify, in-place
      insert/replace planning, album-level and radius planners) plus shared
      tests/fixtures/record_walk/vectors.json; no CLI-visible change."
  - id: walk-sync
    title: Albums sync repairs walk lines; list/check report them
    size: medium
    depends_on:
      - walk-core
    description:
      "walk-sync: compose walk-line edits into bob albums sync (standalone and nightly),
      add the walk object, record_walk_v1 capability, and album_walk_* findings to
      albums list/check, and write the Record walk contract in docs/records.md."
  - id: walk-records
    title: Records show/check/tidy learn the walk line
    size: small
    depends_on:
      - walk-core
    description:
      "walk-records: add record.walk to records show (JSON + human card), record_walk_*
      findings to records check, teach tidy's probes to see past the walk line, and pin
      Notes-insertion goldens with a walk line present."
  - id: walk-writers
    title: Record creation stages its neighbours in the same transaction
    size: medium
    depends_on:
      - walk-core
    description:
      "walk-writers: capture creates and bob records new plan the ring from the final
      overlay, stage the new record's line plus only the neighbour lines the batch
      changes under preimage guards, and report them as walk_files[]."
  - id: nav-guard
    title: Album-folder ghost guard and footer label fallback
    size: small
    depends_on: []
    description:
      "nav-guard (bob-plugins): [n / ]n and the links footer never create a note inside
      an album folder (notice + best-effort sync request instead), and footer arms whose
      label strips to empty fall back to the target stem."
  - id: walk-model
    title: Plugin walk model, cross-check, and footer count
    size: medium
    depends_on:
      - walk-core
      - nav-guard
    description:
      "walk-model (bob-plugins): pure JS walk-line parser and metadataCache ring (shared
      vectors), neighbour cross-check with a syncing state, live refresh on sibling
      changes, and the N/M position in the n links footer."
  - id: walk-rail
    title: Record-sheet walk rail
    size: medium
    depends_on:
      - walk-model
    description:
      "walk-rail (bob-plugins): render the walk line as a docked rail under the record
      sheet in Live Preview and Reading view (prev, album + N of M, next, hue track,
      wrap cue, narrow collapse, a11y), with an Obsidian e2e and docs."
  - id: walk-ship
    title: End-to-end verification, docs pass, and memory
    size: small
    depends_on:
      - walk-sync
      - walk-records
      - walk-writers
      - walk-rail
    description:
      "walk-ship: run the end-to-end CLI walkthrough in a temp vault, review a read-only
      albums sync dry run of the live vault, finish cross-repo doc links, and apply the
      accepted memory decisions."
decisions:
  footer_copy:
    ask: How should the n links footer show a record's position?
    choices:
      bare: "`Links 3/104`; the album name lives in the tooltip and on the rail"
      album:
        "`Links · Eat 3/104`; names the album, but is wider and sheds sooner as the
        status bar narrows"
    default: bare
    why: Rhymes with the lane footer's `Lanes 3/12`; the rail already names the album
    answer: bare
  memory_decision_record:
    ask:
      Record a decision that record walks store links and never counts, with albums sync
      as the only repair owner?
    memory:
      - decisions:record-walks-store-links-never-counts
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary_record_note:
    ask:
      Add one Record Note glossary sentence naming the bob-owned walk line on each
      record's first body line?
    memory:
      - glossary:record-note
    default: false
    answer: true
proposed_by: bbugyi200.apollo.research.5b.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 11:07:16
status: wip
---

- **PROMPT:**
  [prompts/202611/record_walk_ring.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_walk_ring.md)

# Record walk: ‹ › links on every record, N of M painted live

## Context

Bryan wants every record note to carry `<` / `>` links to the previous and next record
in its album. The walk is alphabetical, cycles at both ends, and works with the `[n` /
`]n` keymaps. A two-record album gives each record links that both point at the other.
`N of M` (position and album size) renders near the links. The design brief is
intuitive, reliable, and beautiful.

Bryan accepted every recommendation in the consolidated research report. Read it before
starting any phase. Its evidence appendix gives the line-level citations this plan
relies on:

```bash
sase artifact read "research:202611/record_nav_ring/record_nav_ring.md" "Record walk design context"
```

Planning inspected clean bob-cli HEAD `f0857be8` and bob-plugins `origin/master` on
2026-11-18. The live vault has 6 albums and 192 records (eat 104, subscriptions 28, tv
23, podcasts 15, movies 12, sase_blog_zettel 10). Every record is `type: "[[record]]"`.
No title contains `|[]#^<>‹›`. No record body already contains an arrow-labelled link.
Bodies start empty (47), with `## Notes` (144), or with one prose line
(`eat/brown_fox_coffee_box.md`).

### What Bryan will see

Raw Markdown, which is what `[n` / `]n`, Source mode, mobile, git, and GitHub see:

```markdown
---
parent: "[[eat]]"
type: "[[record]]"
title: "Ramen Nagi"
…
---

[[eat/pho_house|‹ Pho House]] · [[eat|↑ Eat]] · [[eat/ramen_ya|Ramen Ya ›]]

## Notes

…
```

Obsidian, desktop Live Preview and Reading view, where the raw line becomes a rail
docked under the record sheet:

```text
┌ Ramen Nagi ───────────────────────────────────────────── ⊞ Edit  ▾ ┐
│ Body › Eat › restaurant                                            │
│ Status    💚 Liked          Category  Ramen                        │
└────────────────────────────────────────────────────────────────────┘
  ‹ Pho House              🍽️ Eat · 3 of 104              Ramen Ya ›
  ─────●──────────────────────────────────────────────────────────────
```

The `n` links footer reads `⇄ Links 3/104 · ‹ Pho House · Ramen Ya ›`.

- **First record:** the prev arm reads `‹ ↺ Zuni Café` with the tooltip "Wraps around to
  Zuni Café (104 of 104)". The last record mirrors this on its next arm.
- **Two records:** both arms name the other record, as requested.
- **One record:** the line is only `[[eat|↑ Eat]]`, and the rail shows `🍽️ Eat · 1 of 1`
  with no arms.

### Adjustments to the request (accepted with the research)

1. Labels use the typographic `‹` / `›` (U+2039 / U+203A), never bare ASCII `<` / `>`.
   The `[n` / `]n` matcher accepts both. Titled labels also give the footer, hover
   previews, and raw Markdown a destination name.
2. `N of M` is painted live, never stored. Storing it would rewrite every record in the
   album on each insert or delete: 105 files for one new restaurant, versus 3 when only
   the links are stored.
3. "Alphabetical" means the album-wide `title_order` the `[!records]` index already uses
   (lowercase title, then path), flat across status groups.
4. M = 1 writes no self-links. M = 2 cycles both ways.
5. Rule change: `[n` / `]n` never create a note inside an album folder.

### Design refinements beyond the research

This plan tightens the research where implementation facts sharpened the picture:

- **Bounded write radius for creates.** A writer stages only the lines its batch
  actually changes: the new record plus the neighbours whose canonical line differs
  between the before ring and the after ring. Capturing into a not-yet-backfilled album
  therefore writes 3 files, not 104. Repairing everything else is sync's job.
- **Updates never touch walk lines.** `title` is a reserved key that
  `bob records update` (and the Record Card envelope) cannot change, so updates never
  reorder the ring. Retitles happen only by hand in Obsidian. The plugin watcher's 3 s
  debounced `bob albums sync` repairs them, and nightly backstops.
- **One repair owner.** `bob albums sync` is the only command that repairs walk lines.
  `bob records tidy` never writes or removes walk lines; it only learns to see past
  them. With that rule, tidy and sync can run in either order.
- **The rail follows the bytes, not a capability.** The rail and the count render
  whenever a recognized walk line exists, so they work with or without a cached album
  report. `record_walk_v1` gates only the plugin's automatic repair request.
- **Quiet findings.** `records check` reports stale or missing lines as one finding per
  album, not one per record. Damaged lines stay per record because each needs a hand
  fix.
- **Plain up label.** The bytes read `↑ Eat`, with no emoji, matching period notes'
  `↑ 2026-Q4`. The rail paints the album emoji.

## Shared contract

Every phase implements against this section. Where a phase and this section disagree,
this section wins. Bob-plugins phases mirror the grammar and order in JS, and the shared
vectors catch drift.

### Order and membership

- **Members** are the same structural set the `[!records]` index counts: the immediate
  Markdown children of the album folder, as `albums::write::overlay_folder_members`
  returns them, including readable members whose YAML is invalid. Nested files,
  non-Markdown files, `.base` files, and the album note are excluded. Invariant (to be
  tested): `M` equals the index tally for the same member slice.
- **Order** uses today's `grouping::title_order`, extracted into one shared `pub(crate)`
  helper that both the index and the walk call. The key is the `GroupingRecord.title`
  (the frontmatter title when not blank, else `humanize_stem(stem)`), lowercased with
  Unicode default lowercase (`str::to_lowercase` in Rust, `String.prototype.toLowerCase`
  in JS). Ties break on the vault-relative slash path. Both comparisons use Unicode
  scalar order; JS must compare code points, not UTF-16 units. Collation upgrades are
  out of scope; they must move the index and the walk together.
- **Ring:** for ordered members `R[0..M)`, `position = i + 1`,
  `prev = R[(i + M − 1) % M]`, `next = R[(i + 1) % M]`. M = 1 has no arms.
- **Held albums:** when the folder is incomplete (`write::folder_complete` is false) or
  any member's text is unavailable, bob computes no ring for that album and writes no
  walk lines in it. It never publishes a smaller, healthy-looking ring. Other albums
  proceed independently.

### Bytes

- **Canonical line**, for M ≥ 2:
  `[[<album>/<prev-stem>|‹ <Prev>]] · [[<album>|↑ <Album>]] · [[<album>/<next-stem>|<Next> ›]]`.
  For M = 1 it is `[[<album>|↑ <Album>]]`. Separators are `·` (U+00B7 with one space on
  each side). Nothing is written for M = 0.
- **Labels:** start from the order title. Drop `|[]#^` (the index's `ALIAS_DROP`), then
  trim whitespace and any leading `‹`/`<` or trailing `›`/`>` from both ends. Then apply
  the index's `inert_text` escaping, so trimming happens before escaping. If the result
  is empty, use the same pipeline on `humanize_stem(stem)`, and finally the bare stem.
  `<Album>` is the same sanitizer applied to `AlbumDeclaration.title`.
- **Matcher safety** (must be tested against a Rust test mirror of bob-plugins'
  `renderedLinkMatchesNavLabel`): the prev arm answers only `prev`, the next arm answers
  only `next`, and the `↑` arm answers neither.
- **Placement.** The line is inserted as the first body line: closing `---`, one blank
  line, the walk line, then (only if the body has non-blank content) one blank line and
  the original body with its leading blank lines removed. A blank-only body becomes
  exactly `---␤␤<line>␤`. The newline is the closing delimiter's line ending, LF when
  the file ends at `---` with no newline. BOM and frontmatter bytes never change. After
  insertion the position belongs to Bryan: a recognized line he moved is rewritten where
  it sits.

### Line recognition and states

Scan only body lines (after the frontmatter block) outside fenced code, using
`crate::native::markdown::{fence_marker, closes_fence}`.

- **Walk line (for album `A`):** the trimmed line consists only of 1 to 3 wikilinks
  (`[[target]]` or `[[target|label]]`) joined by `·` with optional spaces. Every target,
  after stripping an optional `.md`, is `A` or `A/<name>`, with no subpath and no deeper
  `/`. Exactly one target is `A`, and its label starts with `↑`.
- **Nav-like line:** any other scanned line containing a `[[A|↑…]]` link.
- **Per-record states:**
  - `current`: exactly one walk line, no nav-like line, and the content (without its
    line ending) equals the canonical line.
  - `stale`: exactly one walk line and no nav-like line, but the content differs. It is
    rewritten in place, keeping its line ending.
  - `missing`: no walk line and no nav-like line. The line is inserted per Placement.
  - `damaged`: reported, never written. Reasons: `extra_text` (a nav-like line exists),
    `duplicate_lines` (two or more walk lines), or `no_frontmatter` (no complete
    frontmatter block to anchor the line; the record keeps its ring slot). Deleting the
    offending line heals it on the next sync.
- **Album state:** `held`, then `damaged`, then `stale` (any missing or stale), then
  `current`, with `empty` when M = 0.

### Write owners and radius

| Path                                                          | Walk behaviour                                                                                                                                                                                                            |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------------------ |
| `bob albums sync` (standalone, nightly, plugin watcher)       | Repairs every `missing`/`stale` line in every non-held album, in its one guarded `albums: sync (N notes)` commit. A second run is a byte-identical no-op.                                                                 |
| Capture create, `bob records new`                             | Radius only: created records plus pre-existing members whose canonical line differs between the before ring (disk) and the after ring (overlay). Damaged neighbours are skipped with a warning; the write still succeeds. |
| Capture update, `bob records update` (flags or `-i` envelope) | No walk edits.                                                                                                                                                                                                            |
| `bob records tidy`                                            | Never writes walk lines; its probes skip them.                                                                                                                                                                            |
| `bob albums list                                              | check`, `records show                                                                                                                                                                                                     | check` | Read-only reports. |

Churn per event, in record files rewritten: create 3, delete 2, a retitle that keeps its
position 2, a retitle that reorders ≤ 5, a status or field edit 0, an album retitle M
(rare). The first sync after release backfills all 192 records in one scoped commit.
After that, the steady state is quiet.

### Reports and JSON (all additive, schema version 1)

- **`bob albums list|check -f json`.** Each album gains
  `walk: {state, total, counts: {current, missing, stale, damaged}, order: [stem…]}`;
  `order` is `[]` when held. Top-level `capabilities` gains `"record_walk_v1"`. Two new
  warning findings, never errors: `album_walk_stale` (one per album with missing or
  stale lines, hint `bob albums sync`) and `album_walk_damaged` (one per album with
  damaged lines; the message names the count and the first path).
- **`bob albums sync -f json`.** `changes[]` items gain
  `kind: "index" | "catalog" | "walk"`, and `counts` gains `walk_lines`. Human output
  summarizes walk repairs as one line per album (`eat — 104 walk lines`) rather than
  listing every path. Walk-line damage never changes the exit code.
- **`bob records show TYPE STEM -f json`.** `record.walk` is
  `{state, reason, position, total, prev: {stem, path, title} | null, next: … | null}`,
  or `null` when the album is held. The human card adds
  `3 of 104 · ‹ Pho House · Ramen Ya ›`, plus a muted
  `walk line missing — bob albums sync` when the record is not current.
- **`bob records check`.** `record_walk_damaged` is one warning per damaged record
  (reason, with a hint to delete the line and let `bob albums sync` rewrite it).
  `record_walk_stale` is one warning per album with missing or stale lines (count and
  hint).
- **Capture and `bob records new` JSON.** These gain `walk_files[]` (`path`, `before`,
  `after`) for pre-existing records only, sorted by path, beside `album_files`. A
  created record's `markdown` (and `note_file.markdown` / `lines`) is its exact final
  bytes, including the walk line. `body` keeps its meaning (the authored body, without
  the walk line), so Bob Mac Capture's preview is unchanged. The Mac app needs no
  change: it shows `body`, and exact `markdown` stays in a collapsed disclosure.

### Rendering (bob-plugins)

- **Rail anatomy:** a three-slot grid (`1fr auto 1fr`). Prev is left-aligned and muted.
  The centre holds the album emoji, the album title as an internal link to the album
  note, `·`, and `3 of 104` in `tabular-nums`. Next is right-aligned. Arm titles
  ellipsize first, then the album title; the count never truncates.
- **Track:** a 1 px hairline along the rail's bottom in the record's root hue (the
  sheet's `data-root-hue`), with a 6 px thumb at x = (N − 0.5) / M. It is decorative
  (`aria-hidden`) and hidden when M = 1. It is a location in a title-ordered ring, never
  a progress or completion meter.
- **Wrap cue:** on N = 1 the prev arm shows a faint `↺` before its title, and on N = M
  the next arm shows one after. The tooltip names the destination and its position.
- **Syncing state:** when the computed neighbours disagree with the written targets, the
  count becomes a muted `⟳` with the tooltip "Record walk is syncing". The arms keep
  following the written links. A wrong number is never shown.
- **Narrow:** when the rail is narrower than 28rem (container query), it collapses to
  `‹  3/104  ›`. The arms become glyph buttons with full accessible names, and the album
  is hidden.
- **Placement:** the rail docks flush under the record sheet, with the same inset and a
  matching corner treatment. A collapsed sheet keeps the rail.
  - Live Preview: the rail replaces the walk line while the selection is outside it, and
    raw Markdown returns when the cursor enters the line.
  - Reading view: the rail replaces that line's rendered paragraph.
  - Source mode shows raw Markdown.
- **Interaction:** clicking an arm runs bob-navigation-hotkeys' `[n` / `]n` commands, so
  jump history, `<C-o>`, and bracket repeat behave exactly like the keys. Mod-click and
  middle-click keep Obsidian's open-in-new-tab behaviour. Hovering an arm shows the page
  preview, and the centre link opens the album.
- **Accessibility:** real link or button elements with targets of at least 32 px,
  visible focus rings, and names like "Previous record: Pho House, 2 of 104 in Eat".
  Respect `prefers-reduced-motion` (hover colour only, no motion). Theme variables keep
  light and dark correct; the root hue colours only the track.
- **Footer:** on a record with a healthy walk, `dueText` becomes `Links 3/104` (see
  DECISIONS `footer_copy`). The tooltip's first line reads
  `⇄ Links · Ramen Nagi is 3 of 104 in 🍽️ Eat · alphabetical, wraps around`. A syncing
  walk shows `Links ⟳`. All other notes are unchanged.

### Shared constraints

- Reuse
  `host_catalog::{FileEdit, ReadDependencies, PlannedTxn, apply_and_commit, staged_rewrites}`,
  the capture batch planner's staging and guards, and `write_staged_files_guarded`. Do
  not build a second transaction engine.
- Never change frontmatter bytes. The record fingerprint hashes frontmatter only, so a
  walk edit can never stale a Record Card `expect`. Keep it that way.
- bob-cli tests use temporary vaults with explicit `BOB_DIR`, config isolation, and
  `BOB_NOW`. No phase writes the live vault or runs a real `bob albums sync` against it.
  walk-ship's read-only `-d` dry run is the only live-vault access.
- Bob-plugins phases open the repo with `/sase_repo` (`sase repo open bob-plugins`),
  read its `AGENTS.md`, edit only `src/` fragments (each hand-edited fragment ≤ 1000
  lines), and run `npm run build`, `npm test`, and `npm run docs:check`. Deploy with
  `bob plugins sync`, following the repo's refusal procedure; never pass `-F` on your
  own.
- Do not edit the Cargo.toml version or CHANGELOG.md; the release train owns them.
  Nothing here is a breaking change. Read `cli_rules.md` and `docs.md` with
  `sase memory read` before touching help text or docs. Run `just docs-gen` when help
  changes and `just docs-check` before landing doc edits.
- Pre-existing tests whose fixtures lack walk lines will now see walk edits from sync
  and creates. Update those expectations deliberately, and add the walk assertions
  rather than weakening the originals.

## Phase walk-core

bob-cli, no user-visible change. Build the pure engine every other phase consumes.

1. Extract `title_order` from `src/native/albums/grouping.rs` into a shared `pub(crate)`
   order helper. The index calls it unchanged, so all index goldens stay byte-identical.
2. Expose the index alias sanitizer pieces (`ALIAS_DROP`, `inert_text`) as needed, and
   implement the walk label sanitizer per [Bytes](#bytes).
3. Add `src/native/albums/walk.rs`, registered in `albums/mod.rs`, containing:
   - ring types (slot with position, total, and optional prev/next member);
   - `render_walk_line`;
   - walk-line and nav-like recognition;
   - `classify` (the state plus a reason);
   - `plan_line_edit`, which returns the after-bytes or none. It must handle BOM, CRLF,
     no trailing newline, a blank-only body, a moved line, fenced examples, and a prose
     first line.
4. Add the album-level planner, roughly
   `plan_album_walk(declaration, folder, overlays, text_of, scope) -> AlbumWalkPlan`:
   - `text_of` returns current text for a vault-relative path (a snapshot, the batch
     planner, or disk), so callers decide where bytes come from.
   - `scope` is `All` (sync and reports) or `Radius { created }` (writers). `Radius`
     computes the before ring from the folder without overlays and returns edits only
     for created paths plus members whose canonical line changed.
   - The plan carries the album state, total, order, per-record walks (path, stem,
     state, reason, position, prev, next), `FileEdit`s with exact preimages, and
     warnings for damaged records inside the radius.
5. Add `tests/fixtures/record_walk/vectors.json`, following the existing
   `tests/fixtures/<topic>/vectors.json` convention (bob-cli owns it; bob-plugins keeps
   a copy). Cover order (case folding; duplicate titles tied on path; blank-title
   fallback; accented, astral, and final-sigma titles; a "The …" cluster), labels
   (arrows at either end, `|[]#^`, sanitized-empty), canonical lines for M = 1/2/3 and
   both wrap ends, and recognition cases (canonical, stale label, moved, extra text,
   duplicate, fenced). A Rust test consumes it.

**Acceptance:**

- Property tests: `next^M(x) = x`, `prev(next(x)) = x`, positions form a permutation,
  and `M` equals the index tally on the same member slice.
- Goldens for M = 1/2/3.
- The full classify and placement matrix, including CRLF, BOM, and no-trailing-newline
  cases.
- Idempotence: applying a plan and re-planning yields no edits.
- The matcher-mirror safety test.
- A radius test: inserting one record in a 104-member, un-backfilled fixture plans
  exactly 3 edits.
- `cargo test` for albums and records, plus `just check`.

## Phase walk-sync

bob-cli. Make sync the repair owner and expose walk health.

1. In `src/native/albums/report.rs`, project each album's walk from the same
   snapshot-backed member slice the index uses. Read record text only from the snapshot,
   with no extra file reads. Add `walk` to entries, the capability, and the two
   `album_walk_*` findings (warnings). Human `list` folds walk issues into the existing
   `~N` marker. Human `check` groups the findings under the album path. `list` and
   `check` stay read-only, and their exit codes are unchanged.
2. In `src/native/albums/sync.rs`, add `scope: All` walk edits to `plan_sync`. Record
   paths never collide with index or catalog hosts; `debug_assert` that in
   `merge_edits`. Tag `changes[].kind` and add `counts.walk_lines`.
   - Extend `verify_current_hosts`: every written record must re-plan as `current`.
   - In `report_of`, mark repaired albums' walk `current`, and drop `album_walk_stale`
     only when every stale or missing line was installed.
   - Held albums are skipped while others commit. Standalone and nightly
     (`sync_with_existing_lock`) share the planner. Dry-run diffs include walk files.
3. Human sync output prints one summary line per album with walk repairs.
4. Docs:
   - Add a `## Record walk` section to `docs/records.md` covering bytes, order and
     membership, states, owners and radius, the churn table, "no count is ever written
     into Markdown", and backfill. Backfill happens on the first real sync after the
     release ships, from nightly or the plugin watcher, as one scoped commit.
   - Amend "Syncing records indexes" to say sync also maintains walk lines.
   - Add the `walk`, `record_walk_v1`, `kind`, and `walk_lines` fields to
     `docs/reference/json-outputs.md` (album reports).
   - Update sync help if its wording claims to touch only indexes and catalogs, then run
     `just docs-gen`.

**Acceptance (temp git vault):**

- The first sync backfills every member in one commit.
- A second sync is a byte-identical no-op that commits nothing and skips post-sync.
- A hand retitle that reorders repairs ≤ 5 files; a hand delete repairs exactly 2.
- M = 2 and M = 1 goldens hold.
- A damaged line is untouched and warned about, with exit 0.
- An album held by an unreadable member is skipped while another album is repaired.
- A member edited between plan and install refuses the transaction with nothing written.
- Nightly parity holds, and `M` equals the index tally across the fixture vault.
- Run focused album and nightly tests, then `just check`.

## Phase walk-records

bob-cli. The record-level surfaces.

1. **`records show`** (`src/native/records/show.rs`): compute `record.walk` from the
   album's `scope: All` plan and add the human card line. A missing stem and the other
   refusals are unchanged.
2. **`records check`** (`src/native/records/check.rs`): add `record_walk_damaged` per
   record and `record_walk_stale` per album, as warnings that never change the exit
   code.
3. **`records tidy`** (`src/native/records/notes_section.rs`, `tidy.rs`): the
   redundant-H1 probe, the "first body content" rule, and post-write verification skip a
   walk line or nav-like line. An H1 directly below the walk line is still redundant and
   is stripped, leaving `---␤␤<walk>␤␤## Notes…`. Tidy never inserts, rewrites, or
   removes walk lines, and the blank-body collapse never fires while a walk line exists.
4. **Pin `records update`** Notes-insertion behaviour with goldens; the current
   `render.rs` already appends after the walk line. Cover three bodies: only a walk line
   (→ walk, blank, `## Notes`, bullets); a walk line plus `## Notes`; a walk line plus a
   legacy H1 plus sections. Confirm updates stage no walk edits.
5. **Docs:** update `docs/records.md` ("Reading one record", the check findings, the
   tidy section) and the record-reports section of `docs/reference/json-outputs.md`.

**Acceptance:** show JSON and human output for M = 1/2/3, held, missing, and damaged
records; check findings and exit codes; tidy goldens with and without a walk line,
including CRLF; update goldens. Run focused records tests and `just check`.

## Phase walk-writers

bob-cli. Creates keep their neighbours honest in the same transaction.

1. **Shared helper** in `src/native/albums/write.rs`: plan `scope: Radius` walk edits
   from the same overlay the index uses. Bind it to the album preimage already guarded
   there.
2. **Capture** (`src/native/capture/record_note.rs`, `plan.rs`): add a batch-final step
   after all items are planned and before `album_files` is derived. It must:
   - for each album with a created record, plan once from the final overlay, then stage
     each changed file through the planner (preimage-guarded, one write per file);
   - refresh created items' `markdown`, `note_file.markdown`, and `lines` from the final
     staged bytes, following `notebook_create::refresh_create_markdown`;
   - derive `walk_files[]` from the planner's original and current texts for
     pre-existing paths;
   - emit a `record_walk_damaged` capture warning for each skipped neighbour.

   Capture updates stage nothing. Bulk capture of adjacent records ends consistent.
   Dry-run bytes equal commit bytes.

3. **`bob records new`** (`src/native/records/cli.rs`): reuse the helper. Push neighbour
   `StagedTextFile`s with their preimages through `write_staged_files_guarded`, add
   `walk_files` to JSON, and list each neighbour's new walk line in the human dry run.
4. **Docs:**
   - In `docs/records.md` ("The note Bob writes", Capture, `bob records`), show the walk
     line in the examples. Replace "ends right after the closing `---`" with the
     walk-line shape, and describe `walk_files`.
   - Update the record-notes paragraph in `docs/capture.md` and the record-notes section
     of `docs/reference/json-outputs.md`.

**Acceptance:**

- Capture into an empty album writes `↑` only. A second record makes both point at each
  other in both directions (the request's example). A third forms the ring.
- Bulk capture of two adjacent records in one draft writes one file each and leaves a
  consistent ring.
- In an un-backfilled 104-member album, a capture writes exactly 3 files.
- A neighbour's preimage race refuses the whole capture with nothing written.
- A damaged neighbour is warned about and left untouched while the capture succeeds.
- Capture updates and `records update` write no walk edits.
- `records new` matches capture byte for byte.
- `body` is unchanged in capture JSON.
- Run capture album tests and `just check`.

## Phase nav-guard

bob-plugins, independent of bob-cli; deploy immediately.

1. **bob-navigation-hotkeys:** in the labeled-link open path (`openLabeledBodyLink` →
   `openOrCreateLinkTarget`, `670-plugin-project-files.js`, plus the bracket-repeat
   route), when the resolved target does not exist, check where it would be created. If
   it lies directly inside an album folder (a vault-root `<dir>.md` whose frontmatter
   type is album; reuse existing album helpers where present), refuse creation. Instead:
   - show a Notice such as "Record walk is catching up — eat/foo.md isn't there yet";
   - request a sync through bob-ledger-tools' album data API when exposed
     (`albumScheduleSync` / `apiAlbumSync`), otherwise name `bob albums sync`.

   Other missing targets keep today's create behaviour, and period notes keep theirs.

2. **bob-ledger-tools links footer** (`124-links-walker-footer.js`):
   - An arm whose target is a missing note in an album folder is not create-eligible: no
     `+`, no "creates it", and the tooltip reads "Not there yet — the Eat walk is
     syncing".
   - `linksWalkerFooterLabel` falls back to the target stem when stripping arrows leaves
     an empty label, which fixes the blank `‹  ·  ›` arms.
3. **Tests:** guard refusal and Notice, non-album creation unchanged, footer arm states,
   empty-label fallback.
4. **Docs and deploy:** in `docs/bob-navigation-hotkeys.md`, document the guard and add
   one sentence to the `[n` / `]n` entry noting that record notes carry a bob-maintained
   walk line. Build, test, run `docs:check`, and `bob plugins sync`.

## Phase walk-model

bob-plugins (`bob-ledger-tools`). The pure model and the footer count.

1. Add a pure fragment (for example `242t-plugin-record-walk-model.js`; keep fragments ≤
   1000 lines) that mirrors the [Order and membership](#order-and-membership) and
   [Line recognition](#line-recognition-and-states) contracts:
   - parse the first walk line in the active record's body (frontmatter and fences
     skipped);
   - compute members, titles, and order from `metadataCache`, comparing code points;
   - cross-check: for M ≥ 2, the computed prev and next stems must equal the written arm
     targets, and for M = 1 the written line must have no arms.

   Output something like
   `{state: "ok" | "syncing" | "absent", position, total, prev, next, wrapsPrev, wrapsNext, album: {stem, title, emoji, hue}}`.
   `absent` means no walk line, so neither the rail nor the count renders.

2. Copy `tests/fixtures/record_walk/vectors.json` from bob-cli into
   `plugins/bob-ledger-tools/scripts/fixtures/record_walk/` (header comment naming the
   owner, following `test-task-status-cycler-pomodoro-clock.cjs`). Add a test asserting
   identical order, labels, and recognition results.
3. **Refresh:** recompute and repaint the footer and sheet in the same tick on vault
   create, delete, and rename inside the album folder and on metadata changes of any
   member. Include the walk model in the sheet's widget signature so a stale rail never
   survives. Async results are generation-checked against the source path.
4. **Repair request:** on a `syncing` episode, ask the album data service to
   `albumScheduleSync` once per episode, only when the cached report carries
   `record_walk_v1`.
5. **Footer:** add the N/M position and tooltip per [Rendering](#rendering-bob-plugins)
   and DECISIONS `footer_copy`. The `⟳` state applies. Non-record notes are unchanged.
6. **Tests and deploy:** tests for the model, cross-check, refresh triggers, and footer
   views (ok, syncing, absent, M = 1, M = 2). Document the footer position in the plugin
   docs, then build, test, run `docs:check`, and `bob plugins sync`.

## Phase walk-rail

bob-plugins (`bob-ledger-tools`). The beautiful part.

1. Build the rail DOM and CSS per [Rendering](#rendering-bob-plugins), next to the
   record sheet fragments (`242i`–`242k`) and `styles.css`. Cover all of these: arms,
   centre, track, wrap cue, syncing, M = 1, M = 2, narrow collapse, and focus and hover
   states.
2. **Live Preview:** reuse the sheet's decoration source and add a replacement over the
   walk line's range that renders the rail. Drop it when the selection touches the line
   so raw Markdown appears. Use a StateField if block replacement requires one. Keep the
   sheet anchor (`recordSheetAnchor`) and its cache signature correct now that a walk
   line follows the frontmatter.
3. **Reading view:** in the sheet's reading painter, replace the walk line's paragraph
   with the rail docked under the sheet.
4. **Interaction:** wire arm clicks through nav-hotkeys' `[n` / `]n` commands (fall back
   to the plain link open when nav-hotkeys is absent). Keep mod-click and middle-click,
   add hover previews, and add the centre album link.
5. **E2e:** add `scripts/test-obsidian-record-walk-e2e.cjs`, modelled on
   `test-obsidian-walker-footer-e2e.cjs` and `test-obsidian-record-writer-e2e.cjs`, with
   a fixture vault holding canonical lines. Cover:
   - a two-record album (both arms point at the other);
   - a wrap on `[n` from record 1;
   - cursor reveal of the raw line in Live Preview;
   - the Reading-view rail;
   - the footer count;
   - a missing destination (the ghost guard refuses);
   - a frontmatter-only record (no rail).
6. **Docs and deploy:** add a "Record walk rail" section to the bob-plugins docs page
   that documents the record sheet (`docs/albums.md` or `docs/record-card.md`, wherever
   the sheet is described), with the anatomy and states. Then build, test, run
   `docs:check`, and `bob plugins sync`.

## Phase walk-ship

bob-cli. Verify the whole and close the loop.

1. **End-to-end CLI walkthrough** in a temp git vault with the built binary:
   - grow an album 0 → 1 → 2 → 3 through capture and through `records new`;
   - hand-retitle a record and hand-delete one, then run `bob albums sync`;
   - confirm a second sync is a no-op;
   - confirm a damaged line is reported and left alone;
   - check tidy with a walk line plus an H1;
   - check `records show` positions and `albums check` health.

   Record the transcript in the phase notes. If the walkthrough is valuable as a
   regression test, keep it as an integration test.

2. **Live-vault dry run (read-only).** Run
   `BOB_DIR=$HOME/bob <built bob> albums sync -d -f json` and confirm:
   - every album has walk state `missing`/`stale` with 0 damaged and 0 held;
   - `walk.total` equals the index tally for each album;
   - the planned walk edits total 192;
   - spot-checked diffs for an M ≥ 3 album look right, as does
     `eat/brown_fox_coffee_box.md` (whose prose stays below the line).

   Never run a real sync on the live vault; the backfill lands automatically after the
   release ships. Report the findings.

3. **Docs pass:** check cross-links between `docs/records.md` "Record walk" and the
   mounted plugin docs (the rail section and bob-navigation-hotkeys' `[n` / `]n` entry).
   Fix any gaps on the bob-cli side. Leave `docs/walks.md` alone: it covers walk
   attention, which `n` never carries. Run `just docs-check`.
4. **Memory**, per DECISIONS, using `/sase_memory_write`:
   - When `memory_decision_record` is accepted, add
     `sase/memory/decisions/record-walks-store-links-never-counts.md` in the house
     format (Applies to, Claim, rejected alternatives, Evidence, Cost, Reopens when).
     - The claim: the walk is a `title_order` ring; the first-body-line line is
       bob-owned in content and Bryan-owned in position; `N of M` is painted;
       `bob albums sync` is the only repairer; creates touch only their radius; managed
       record jumps never create notes.
     - Rejected alternatives: stored counts, a virtual walk, bottom placement, bare
       glyphs, and tidy as the repairer.
     - Evidence: the research ref, this plan, and the landed commits.
   - When `memory_glossary_record_note` is accepted, add one sentence to
     `sase/memory/glossary/record-note.md`.

   Run `sase memory init`. If a decision is declined and nobody reviewed the plan,
   record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

## Out of scope

- Counted `N]n` jumps, `]}` / `[{` status-group jumps, and a custom sort UI.
- A collation upgrade (natural numbers, accent folding, ignoring "The"). It needs its
  own decision and must reorder the index and the walk together.
- A regenerable vault-sync conflict class for walk-line-only conflicts. A two-host
  adjacent create can leave one `_conflicts/` copy that the next sync regenerates, the
  same class the album index already survives.
- Bob Mac Capture changes, `bob walks` schema changes, and new CLI commands or flags.
