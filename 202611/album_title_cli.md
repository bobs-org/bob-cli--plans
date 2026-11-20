---
tier: tale
title: Album `title:` key, H1-less album create, and `bob albums tidy`
goal:
  bob reads album frontmatter `title:` first everywhere, `bob albums create` writes
  `title:` and no H1, and a dry-run-first `bob albums tidy` moves existing album H1
  titles into `title:` with byte-identical walk lines, records indexes, and owner
  catalogs before and after.
size: medium
proposed_by: bbugyi200.athena.bob-cli-o6.3
proposed_with: claude/opus
bead: bob-cli-o6.3
create_time: 2026-11-20 12:00:37
status: wip
---

- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)
- **BEAD:**
  [bob-cli-o6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-o6/bob-cli-o6.3.md)

# Album `title:` key, H1-less album create, and `bob albums tidy`

Phase `album-title-cli` (bead `bob-cli-o6.3`) of epic `bob-cli-o6`
(`plan:202611/property_card_title_block.md`). bob-cli only. This phase does not apply
tidy to the live vault. Release policy: never edit the `Cargo.toml` version or
`CHANGELOG.md` (`decisions:bob-cli-releases-ride-a-daily-train`). Read
`sase memory read cli_rules.md docs.md -r "<why>"` before touching CLI help or docs.
Make no memory edits: the epic authorizes memory edits only in its `glossary` phase, and
no album glossary strand states a title rule.

## Why

Albums show their title three times today (filename, H1, crumb). The epic moves each
album's title into a bob-owned frontmatter `title:` key, so the card can be the only
visible title. bob must read `title:` first everywhere. New albums must be born with
`title:` and no H1. Existing albums need a guarded, dry-run-first migration that changes
no derived output (walk lines, `[!records]` indexes, `[!albums]` catalogs).

## What exists (verified)

- **One title choke point.** Every album title reader goes through
  `AlbumDeclaration.title`, which `parse_album_declaration` in
  `src/native/albums/mod.rs` sets with `derive_title(contents, first_alias, stem)`
  (first H1 before a bullet or `##`, cleaned, then the first alias, then the humanized
  stem). The readers are:
  - the album report `title` (`albums/report.rs`) and owner `[!albums]` rows
    (`albums/catalog.rs`);
  - walk labels `↑ <Album title>` (`albums/walk.rs` `sanitize_walk_label`);
  - registry breadcrumbs (`albums/registry.rs` `album_breadcrumb`) and record context
    and identity (`records/context.rs`);
  - capture targets and parse (`capture_targets.rs`, `capture_parse.rs`,
    `capture/record_note.rs`, `capture/output.rs`, `capture_record.rs`);
  - `bob records show` and `list` (`records/show.rs`, `records/cli.rs`) and bank
    subscriptions.

  Notebook title paths (`capture_notebooks.rs`, `notebooks/scan.rs`) title only
  notebooks. `note_catalog` already uses frontmatter `title` and then the stem for every
  kind, so it needs no change. Non-album `derive_title` stays unchanged.

- **Live vault (read-only survey).** `eat.md`, `movies.md`, `podcasts.md`,
  `sase_blog_zettel.md`, `subscriptions.md`, and `tv.md` are albums. None has `title:`,
  and each has one plain H1. So the reader change alone changes nothing live until tidy
  runs.
- **Create.** `render_new_album` (`albums/create.rs`) writes `parent`, `type`,
  `summary`, optional `emoji`, `created`, and `record:`, then `---`, a blank line, and
  `# <title>`. `plan_create` appends `\n<intro>` and parses the result back through
  `parse_album_declaration`.
- **Precedent.** `bob records tidy` (`src/native/records/tidy.rs`) has the report, JSON,
  `host_catalog::run_locked`, `PlannedTxn`, full-preimage `ReadDependencies`, verify
  closure, and one scoped commit. The shared H1 rule is
  `records::notes_section::strip_redundant_h1(original, title) -> H1Strip { after, kept }`.
  It handles fences, CRLF, and BOM. Its kept reasons are "the H1 is not the first body
  content", "the H1 differs from the frontmatter title", and "a later H1 is kept".
- **Byte-safe frontmatter spans.** `src/native/yaml_spans.rs` has `split_lines`,
  `join_lines`, `collect_key_spans` (it refuses duplicate keys), and
  `yaml_string_scalar` (plain when safe, otherwise JSON-quoted).
- **Dry-run-by-default precedent.** `bob periods migrate` takes `-a/--apply` ("Write the
  plan and make one verified commit").
- **Command Mode precedent.** One-time migrations are `Excluded` in
  `src/native/catalog/table.rs` with reason "One-time migration; run it from a
  terminal".

## Design

### 1. Readers: frontmatter `title:` wins for albums

In `src/native/albums/mod.rs`:

- Add `fn frontmatter_title(frontmatter: &Frontmatter<'_>) -> FrontmatterTitle`. It
  decodes only the `title` entry's own span: the first column-0 line `title:`, plus any
  indented or blank continuation lines up to the next column-0 non-blank line, parsed
  with `serde_yaml` as a one-key mapping. A malformed `record:` section therefore cannot
  hide the title. Variants:
  - `Absent`: no top-level `title` key.
  - `Valid(String)`: a YAML string, trimmed, nonempty, with no `\n` or `\r`. Escapes
    decode, so `"Ma \"Pa\" Deli"` gives `Ma "Pa" Deli`.
  - `Invalid`: the key is present but is a non-string, blank, multi-line, or unparseable
    value.
- In `parse_album_declaration`, set `title = Valid(t) => t, otherwise derive_title(...)`
  (unchanged fallback). Store the variant on `AlbumDeclaration` as a new field (for
  example `title_key: FrontmatterTitle`) so tidy can tell absent from invalid. Update
  the struct's construction sites.
- On `Invalid`, push one warning finding: code `album_title_invalid` (the code create
  already uses for a bad `-t`), key path `title`, the line of the key, and the message
  "frontmatter title must be one nonblank line; using the H1, alias, or stem title".
  This makes `bob albums check` explain the fallback instead of hiding it.
- Change nothing else. Every reader listed above picks up the rule through the
  declaration.

### 2. Create: write `title:` and no H1

- `render_new_album` writes keys in canonical order: `parent`, `type`,
  `title: <yaml_string_scalar(title)>`, `summary`, optional `emoji`, `created`, then
  `record:`. Then `---\n`, and no H1. `plan_create` still appends `\n<intro>`, so a new
  album reads `---\n\nTrack <plural> with ...`, then the empty records index, which is
  placed before the first H2 as today.
- Update the doc comments ("an H1" becomes "no H1; the title lives in `title`") and the
  `render_intro` comment ("under the H1" becomes "under the frontmatter").
- `-t` keeps its validation. The default stays the humanized stem. `plan_create`'s
  parse-back must yield `declaration.title == title`. Add a debug assertion or refusal
  so a quoting bug cannot ship silently.

### 3. `bob albums tidy`: new module `src/native/albums/tidy.rs`

**CLI.** `bob albums tidy [STEM] [-a] [-b DIR] [-f FORMAT] [-h]`. Options are
alphabetical, and every long option has a short alias.

- `STEM` (optional): an exact root album stem from `registry.albums`. There is no noun
  fallback and no path. An unknown stem is a usage error: exit 2 before any lock, code
  `unknown_album`, hint "use an album stem from `bob albums list`".
- `-a/--apply`: "Write the plan and make one verified commit". Without it, the run is a
  dry run that takes no lock, writes nothing, runs no Git, and does no sync.
- `-b/--bob-dir`, `-f/--format human|json`, and `-h/--help` are the same as in the
  sibling commands.
- About: "Move album H1 titles into frontmatter `title` (dry run by default)".
- The long about states the rule below, the lifecycle, and the mixed-version caution in
  "Tidy rule". The after-help examples are `bob albums tidy`, `bob albums tidy eat`,
  `bob albums tidy -f json`, and `bob albums tidy -a`, ending with "See docs/records.md
  for the album title rule."

**Scope.** Every album in `AlbumRegistry::from_snapshot(...).albums`, including albums
with schema findings, because the edit touches neither the schema nor the index. If
`DiscoveryHealth::Partial` holds, fail with the `io` failure `records tidy` uses ("the
vault scan was incomplete…"). Unreadable notes go to `io_errors[]`.

**Tidy rule.** Each album gets one pass:

1. **Title.**
   - `title_key == Absent`: insert `title: <yaml_string_scalar(declaration.title)>`.
     Here `declaration.title` is the cleaned first H1, or the alias or stem fallback,
     which is exactly what every reader shows today. So the change is invisible to walk
     lines, indexes, and catalogs.
     - **Insertion point:** directly after the full span of the top-level `type` key
       (canonical order), else after the `parent` span, else right after the opening
       `---`. Spans come from `yaml_spans::collect_key_spans`, so a multi-line `type:`
       list stays intact.
     - **Line ending:** the ending of the line just before the insertion point, so CRLF
       stays CRLF. BOM bytes stay on line 0.
     - **Damaged frontmatter:** duplicate keys (a `collect_key_spans` error) or a
       missing closing `---` leave the album unchanged and add a `warnings[]` entry
       naming the path and reason.
   - `Valid`: write no title.
   - `Invalid`: leave the album unchanged and add a `warnings[]` entry: "<path>:
     frontmatter title is not one nonblank line; fix it by hand". Never overwrite an
     author's key.
2. **H1.** Run `strip_redundant_h1(text_after_step_1, effective_title)` with no
   walk-line skipping. It removes the H1 only when that H1 is the first body content
   outside fences and its text equals the title exactly. It then removes the blank lines
   after the H1, which yields `---\n\n<intro>` for created-shape albums. Report its
   `kept` reason in `kept_h1[]`. Add one album-only reason: when no H1 exists outside
   fences but a level-1 ATX line sits inside a fenced block, record "the H1 is inside
   fenced code" (use `crate::native::markdown` `fence_marker`, `closes_fence`, and
   `atx_heading`). An H1 with wikilinks or emphasis differs from its cleaned title, so
   it is kept and reported as differing.
3. **Defensive re-check.** A planned `after` must itself plan to no change, or the note
   is skipped, as in records tidy. Tidy is idempotent: a second run plans nothing. Kept
   H1s are reported again, but nothing changes.

Caution for help and docs: an older bob derives album titles from the H1. If an older
bob runs `bob albums sync` after tidy removes an H1, it rewrites walk lines and catalog
rows to the humanized stem. Apply tidy only once every host that runs album sync has
this bob. The live apply belongs to the epic's `obsidian-proof` phase.

**Lifecycle.** This mirrors records tidy:

- `-a` takes `host_catalog::run_locked(vault, &child_env, json_mode, "album", ...)` and
  replans inside the lock.
- `PlannedTxn` is built from `host_catalog::staged_rewrites`, with a full-preimage
  `ReadDependencies` per album and the subject `albums: tidy (N notes)`.
- The verify closure re-reads every written album and requires three things: it parses
  as an album, `declaration.title` equals the planned title, and a fresh pass plans no
  change.
- One commit. Post-sync runs only when the run committed. A no-op writes nothing and
  skips post-sync.
- Do not add album tidy to `bob nightly`.

**Exit codes.** 0 for a clean run, or a dry run with pending changes. 1 for runtime,
guard, verify, or I/O failure. 2 for a usage error.

**JSON** (`schema_version: 1`, `command: "tidy"`):

```json
{
  "schema_version": 1,
  "ok": true,
  "command": "tidy",
  "dry_run": true,
  "counts": {
    "scanned": 6,
    "changed": 6,
    "title_added": 6,
    "h1_removed": 6,
    "h1_kept": 0
  },
  "changes": [
    {
      "path": "eat.md",
      "title": "Restaurants",
      "title_added": true,
      "h1_removed": true,
      "h1_kept_reason": null,
      "diff": "..."
    }
  ],
  "kept_h1": [{ "path": "...", "reason": "..." }],
  "commit": null,
  "sync": null,
  "warnings": [],
  "io_errors": []
}
```

`diff` appears only on dry runs. `commit` is `{sha, subject}` and `sync` is
`{pre, post}` on applied runs. `error {code, message, hint}` appears on failure.

**Human output.** It follows records tidy:

- **Dry run with changes:** colored unified diffs, then "! dry run: N albums would
  change (X titles added; Y redundant H1s); nothing was written. Rerun with -a to
  apply".
- **Clean:** "✓ albums are tidy: N scanned, 0 tidied".
- **Applied:** "✓ tidied N albums (<sha9>): …", with one line per path.
- **Kept H1s:** a dim block listing each kept H1 with its reason.
- **Warnings:** printed with a yellow `!`.

**Reuse, don't copy.** Make `records::tidy::color_diff` `pub(crate)` (or move it next to
`Styler`) and call it. Reuse `strip_redundant_h1`, `yaml_spans`, and the `host_catalog`
transaction pieces. Keep the per-album planner a pure function, for example
`plan_album(text, declaration) -> AlbumPass`, so unit tests need no vault.

### 4. Registrations (the three places that broke the `ms` and `nh` landings)

- `src/native.rs`: add `NativeCommand::AlbumsTidy` and dispatch it to
  `albums::tidy::run_tidy(args)`.
- `src/native/albums/mod.rs`: add `pub(crate) mod tidy;` and re-export `build_cli_tidy`.
- `src/runner.rs`:
  - add an `ALBUMS_MEMBERS` entry `tidy` between `sync` and `update` (about = clap
    about);
  - add two `ALBUMS_AFTER_HELP` example lines (`bob albums tidy`, a preview of title
    moves as diffs; `bob albums tidy -a`, which moves titles in one commit);
  - add `NativeCommand::AlbumsTidy` to the canonical-path test list.
- `src/native/completion/tree.rs`: map `AlbumsTidy` to `build_cli_tidy()`.
- `src/native/completion/kinds.rs`: map `["albums","tidy"]` arg `stem` to
  `Kind::AlbumStem`. Every other value arg already has a generic decision; run the
  `every_value_arg_has_a_decision` test to confirm.
- `src/native/catalog/table.rs`: add
  `Excluded { path: &["albums","tidy"], reason: "One-time migration; run it from a terminal" }`.
  Update any catalog test that counts exclusions.
- Keep the root group's "Inspect album notes" as it is. Regenerate help fixtures only if
  the tests demand it.

## Tests

**Unit, `albums/mod.rs`.**

- `title:` beats H1, alias, and stem.
- Quoted escapes decode.
- `title: "  "`, `title: 42`, `title: [a]`, and a block scalar with two lines each fall
  back to the H1 and emit the `album_title_invalid` warning with its line.
- Absent `title` keeps the H1, alias, and stem chain byte-for-byte as today. Keep the
  existing `First H1` test.
- A malformed `record:` section does not hide a valid `title`.

**Unit, `albums/create.rs`.**

- Rename and update `renders_canonical_key_order_h1_intro_and_label`. It expects `type`
  then `title: Restaurants` then `summary`, and no `# ` line.
- Titles needing quotes round-trip through the parse-back: `Eat & Drink`, `2026`, `yes`,
  `a: b`, and `"quoted"`.
- The plan's album text starts `---\n…---\n\nTrack`.

**Unit, `albums/tidy.rs`.** Use the pure planner, plus a small fixture vault with Git,
as records tidy does. Cover:

- **The main case:** absent title with a matching H1 adds the title and removes the H1.
- **Title present:** with an equal H1, only the H1 is removed. With a differing H1, it
  is kept as "differs".
- **H1 placement:** with prose first, the title is added and the H1 is kept as "not the
  first body content". A later H1 is kept as "a later H1 is kept".
- **No H1:** a fenced-only H1 adds the title from the alias or stem and is kept as
  "inside fenced code". With no H1 at all, the title comes from the alias or stem.
- **Refusals:** an invalid title key gives a warning and no change. Duplicate
  frontmatter keys give a warning and no change.
- **Byte safety:** CRLF and BOM are preserved. The insertion lands after a multi-line
  `type:` list. A file whose `parent` follows `type`, and one with no `type`-first
  order, both insert correctly.
- **Idempotency and selection:** the second plan is empty. A STEM selects one album. An
  unknown STEM gives `unknown_album`. A partial scan fails with `io`.
- **Transaction:** the subject is `albums: tidy (1 notes)`. Verify rejects a write that
  leaves a redundant H1 or a wrong title.

**Integration, new `tests/cli/albums_tidy.rs`** (register in `tests/cli/main.rs`). Reuse
the `albums_sync.rs` harness shape: `init_vault_sync_pair` and an env with `BOB_NOW`,
`TZ`, lock, and state files.

- **Bare run is a dry run.** It prints diffs and leaves HEAD and file bytes unchanged.
  JSON counts are correct and `dry_run: true`.
- **`-a` commits once.** It makes one commit, `albums: tidy (N notes)`. A rerun is a
  no-op with no new commit, and its JSON `changes` is empty.
- **No churn (most important).** Build a fixture with:
  - an owner area `body.md`;
  - an album `eat.md` whose H1 is `# Restaurants` (not the stem) and whose schema has a
    `status` choice field;
  - three records in `eat/`;
  - a second album with no H1.

  Steps:
  1. Run `bob albums sync` (real) and commit.
  2. Snapshot every record file's bytes, the album's `[!records]` region, the owner's
     `## Albums` region, and `bob albums list -f json` titles.
  3. Run `bob albums tidy -a`.
  4. Run `bob albums sync -f json`.

  Assert all of the following:
  - the second sync is a no-op (no `changes`, no new commit);
  - record files are byte-identical;
  - the `[!records]` and `[!albums]` regions are byte-identical;
  - list titles are equal;
  - the album files differ only by the inserted `title:` line and the removed H1 and its
    blank lines.

- **STEM selection.** `bob albums tidy eat -a` touches only `eat.md`. An unknown stem
  exits 2 and writes nothing.

**Existing tests to update.**

- `tests/cli/albums_create.rs`: `EXPECTED_ALBUM` gains `title: Eat` after `type` and
  drops `\n# Eat\n`. Replace the `contains("# Movies\n")` and
  `contains("# Eat & Drink\n")` checks with `title:` assertions.
- Fix any other test that asserts a created album's H1. Search with
  `rg -n 'render_new_album|# Eat\\n' src tests`.
- Update the catalog, completion, help, and `docs_reference` tests the new command
  touches.

## Docs

- **`docs/records.md`:**
  - **Album example:** under `## Albums`, add `title: Eat` after `type` and drop
    `# Eat`.
  - **Title bullet:** add one: "frontmatter `title` (one nonblank line) wins, else the
    first H1 before the first bullet or `##` (wikilinks to display text), the first
    alias, then the humanized stem; an unusable `title` warns `album_title_invalid`."
  - **Creating an album:** list `title` in the canonical key order and say "no H1"
    instead of "an H1".
  - **New section:** add `### Tidying album titles` after "Syncing records indexes". It
    covers the rule, the kept reasons, the dry-run default and `-a`, the lifecycle, the
    JSON shape, that nightly does not run it, and the mixed-version caution.
- **`docs/reference/vault-format.md` "Album notes":** add one sentence on the title
  source, create writing `title` with no H1, and `bob albums tidy`.
- **`docs/reference/json-outputs.md` "Album reports":** add one terse row, for example:

  ```
  | `bob albums tidy` | `-f json` | Tidied albums, counts, kept H1s, transaction results | `schema_version: 1` | [Tidying album titles](../records.md#tidying-album-titles) |
  ```

  The reference tier caps at 6000 words and this page is near it. If `just docs-check`
  flags the budget, trim words elsewhere on the page. Never raise a ceiling.

- **Generated docs:** run `just docs-gen` to regenerate `docs/reference/cli/*`. Do not
  hand-edit generated pages.

## Gates and wrap-up

1. Run `just fmt`, `just check`, `just docs-gen`, and `just docs-check`. Each must pass.
   A failure that reproduces identically on a clean `origin/master` tree does not block.
   Record it with `sase bead note bob-cli-o6.3 'PROPOSED FOLLOW-UP: …'`, citing any
   existing task bead.
2. Run `just install` on athena.
3. Run a read-only dry run against the live vault, `bob albums tidy -f json` with no
   `-a`. It takes no lock and writes nothing. Record its counts and kept-H1 list on the
   bead as evidence for `obsidian-proof`. Never pass `-a` against the live vault in this
   phase.
4. Record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on `bob-cli-o6.3`. Create
   no beads. One known candidate is that the runner's `records tidy` member about
   ("Remove empty `## Notes` sections from Record Notes") omits the redundant-H1 removal
   its clap about states. Another is any mixed-version risk you observe, since a new
   album created before every host upgrades has no H1 for an older bob to read.
5. Run `sase bead epic-symbols bob-cli-o6.3` and resolve any leftovers. Then close only
   this bead with
   `sase bead close bob-cli-o6.3 --note "<gates run, no-churn test, live dry-run counts>"`.
   Never close epic `bob-cli-o6` or any ancestor.

## Non-goals

- Applying tidy to the live vault, which belongs to the `obsidian-proof` phase.
- Any bob-plugins change, which belongs to `collection-cards`.
- A title rename operation (`bob-cli-n1`).
- An album emoji writer (`bob-cli-ml`).
- Any change to `album.set` keys.
- Any change to non-album `derive_title`.
- Any change to `note_catalog`.
- Any memory edit.
