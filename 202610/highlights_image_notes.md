---
tier: tale
title: Highlights image notes in bob ref
goal:
  A screenshot path pasted into a Highlights note is copied into the vault and rendered
  inline below its highlight.
size: medium
decisions:
  refresh:
    ask: Should later edits to the file at a pasted path re-copy over its vault image?
    choices:
      snapshot:
        Copy once, never overwrite; delete the vault copy to re-import. No cross-host
        flapping
      mirror:
        Re-copy when source bytes change so edits propagate; adds guarded overwrites,
        can flap
    default: snapshot
    why:
      macscrot names each capture uniquely, so frozen copies fit and vault files are
      never overwritten
    answer: snapshot
proposed_by: bbugyi200.athena.11k
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-18 16:16:51
status: wip
---

# Highlights image notes: paste a screenshot path into a highlight's note, see the image in the ref note

## Goal

Bryan annotates PDFs in the Highlights app on the MacBook. He wants to attach a
screenshot to a highlight by pasting its path into that highlight's note, for example
`~/tmp/screenshots/20261018_104022.png`. `bob ref scan` / `bob ref sync` must copy the
image into the vault and render it inline, directly below the highlight it belongs to,
in the generated reference note. This feature is called **image notes**. It needs no new
CLI flags: it runs as part of every sync.

The user's priorities, in order: **intuitive**, **reliable**, **beautiful**.

## Background (what exists today)

- **Sidecar parsing:** `src/native/highlights_ref/sidecar.rs`. `parse_sidecar_chunk`
  produces
  `SidecarAnnotation { kind, text, comment, task_source, image, order, ordinal_on_page }`
  (model in `model.rs`). Highlight comments are plain paragraphs, or `- ` list items in
  the linked-page export. A leading `Comment:`/`Note:` label is stripped. Real vault
  sidecars do not backslash-escape `_`.
- **Rendering:** `sidecar_render.rs`. `render_sidecar_highlights` →
  `render_annotation_block` → `push_callout_block` / `push_annotation_comment_callout`.
  Highlight block IDs hash PDF path, kind, page, ordinal, and quote text, never the
  comment, so adding a path to a comment cannot churn `^h-…` IDs. Beautification
  (`text.rs` `beautify_annotation_text`) is render-only.
- **Existing image copies:** TextBundle image selections already copy into
  `ref/<type>/<stem>.assets/h-<id>.<ext>` through `ImageAssetWrite` / `ImageAssetAction`
  (`model.rs`), resolved in `sidecar.rs::resolve_sidecar_image_assets`, and executed by
  `sync.rs::execute_image_asset_write` in both the v1 and v2 write paths.
  - Copies are atomic: temp file plus rename via `atomic_copy`.
  - An existing destination with different bytes is an error.
  - `plan.image_assets` dest paths already feed the dirty-target guard and the
    create-time intake commit path set (`guard.rs` `scoped_write_path_set`), cross-PDF
    collision checks (`validate_planned_asset_collisions`), v2 preconditions
    (`validate_v2_asset_preconditions`), `write_summary`, and the `image assets` scan
    counts.
- **Read side:** `region.rs::parse_managed_region` parses the rendered region back into
  `RegionBlock { page_label, kind, quote, comment, asset, block_id, mirror, in_preamble }`.
  Its consumers are:
  - `ref_library/mod.rs::annotation_counts`: index annotation/comment counts plus row
    diagnostics such as `unparsed_region`, which `bob ref doctor` rolls up by code under
    `library diagnostics`.
  - `ref_library/show.rs`: `bob ref show` in JSON, human, and markdown forms.

  A block the parser cannot classify becomes `unparsed`. The renderer and the parser
  **must change together**, or every note with an image note will raise
  `unparsed_region` warnings.

- **Bryan's screenshot workflow** (`macscrot` in the chezmoi repo): saves
  `~/tmp/screenshots/YYYYMMDD_HHMMSS.png` on the Mac, uploads the same relative path to
  athena and apollo, and copies the literal `~/tmp/screenshots/<name>.png` to the
  clipboard.
  - The Mac's 15-minute `bob highlights scan` cron job therefore resolves it locally.
  - A real capture is a Retina PNG with a `pHYs` chunk of 5669 px/m (144 dpi), e.g.
    2880×1664 px for a full screen.
- **Vault `.gitignore`** allowlists `png jpg jpeg gif webp svg` but not
  `bmp avif heic heif tif tiff`. A copied image outside that list would never sync to
  other hosts.
- No bob-plugins code parses the managed region. `note_snapshot` already understands
  `![[img|<width>]]` hints via `width_hint_to_percent`.

## UX contract

### 1. Writing an image note (grammar)

An image note is a **line of an annotation's note text whose entire content is one local
image path**. The rule a user should remember: _"Put the path on its own line."_ It
applies to:

- highlight comments (the main case),
- comments on TextBundle image selections,
- standalone notes.

`parse_image_note_line(line) -> Option<ImageNoteRef>` accepts a line after:

1. trimming whitespace;
2. stripping one unordered list marker (`- `, `* `, `+ `);
3. stripping a leading `Comment:` / `Note:` label (same case variants as
   `strip_comment_label_only`);
4. stripping one matching wrapper pair: `` `…` ``, `<…>`, `"…"`, `'…'`, `“…”`, `‘…’`
   (macOS smart quotes);
5. stripping one trailing `.`, `,`, or `;` when what remains ends in an image extension.

What remains must be one of:

- `~/…` (home-relative),
- `/…` (absolute),
- `file://…`: decode with the existing `url` crate (`Url::parse` + `to_file_path`). The
  canonical text is the decoded absolute path.

After that, backslash escapes are unescaped (`\ ` → space, `_` → `_`). Spaces inside the
path are allowed when the whole line is the path, as in macOS names like
`Screenshot 2026-10-18 at 10.40.22 AM.png`.

- A line of several whitespace-separated tokens, each of which is itself a path ending
  in an image extension, yields one image per token.
- Prose containing a path mid-sentence is **not** an image note. Neither is a relative
  path. Both stay ordinary comment text.

The extension is lowercased for the checks below:

- **Supported:** `png jpg jpeg gif webp svg`. These are the formats Obsidian renders
  _and_ the vault gitignore tracks.
- **Recognized but unsupported:** `heic heif tif tiff bmp avif`. These render an "Image
  type not supported" notice (§4) instead of silently staying text.

`ImageNoteRef` carries:

- `source_text`: the canonical text as written, e.g.
  `~/tmp/screenshots/20261018_104022.png`. It is the identity key and is never
  host-expanded.
- `extension`
- `supported: bool`

Resolution expands `~/` with `env::home_dir()` only when it reads files. The key
therefore stays the same on every host.

### 2. Where the copy lives

- **Directory:** beside TextBundle assets in the note's existing assets directory,
  `ref_note_assets_dir(note)` = `ref/<type>/<stem>.assets/`.
- **File name:** `<safe-stem>-<hash8>.<ext>`.
  - `hash8` is the first 8 hex digits of SHA-256 over `source_text`.
  - `safe-stem` is the source file stem with every run of characters outside
    `[A-Za-z0-9._-]` collapsed to `_`, trimmed of leading/trailing `._-`, truncated to
    64 chars on a char boundary, and set to `image` if empty.
  - Example: `~/tmp/screenshots/20261018_104022.png` →
    `ref/papers/foo.assets/20261018_104022-07154fc6.png`.
- **Why this naming:**
  - The timestamp Bryan pasted stays recognizable.
  - The name is a pure function of the note text, so it is identical on every host and
    survives the source file being deleted.
  - Two different paths sharing a basename can never collide.
  - It can never collide with TextBundle `h-<id>.<ext>` names.
- **Shared images:** the same `source_text` used on several annotations maps to one
  copy, and each annotation embeds it.

### 3. How it renders

Images go **directly below the highlighted text, inside the highlight's `[!quote]`
callout**, in source order, each preceded by a bare `>` spacer. Any remaining comment
text follows as the usual nested Comment callout. The trailing `^h-…` block ID still
covers the quote, its images, and the comment.

Image only:

```md
> [!quote] Latency is not throughput.
>
> ![[ref/books/example.assets/20261018_104022-07154fc6.png|420]]

^h-2b91f0a4c7de
```

Image plus text comment, written in the Highlights note as two lines (path first or
last, either works):

```md
> [!quote] Latency is not throughput.
>
> ![[ref/books/example.assets/20261018_104022-07154fc6.png|420]]
>
> > [!note] Comment Compare with the p99 chart.

^h-2b91f0a4c7de
```

The other annotation kinds follow the same layout:

- **TextBundle image selections:** `> [!quote] Image ![[…h-<id>.png]]`, then `>` and the
  image-note embed lines, then the Comment callout.
- **Standalone notes:** `> [!note] <text>`, then `>` and the embed lines. An image-only
  note renders the bare `> [!note]` header followed by `>` and the embed.

Splitting rule:

- When separating image lines from comment text, replace each image line with a blank
  line _before_ beautifying. Prose on either side then stays in separate paragraphs
  instead of being reflowed together.
- If nothing but whitespace remains, render no Comment callout.
- The split is **render-only**, like beautification: `SidecarAnnotation.text`,
  `comment`, and `task_source` stay raw. Block IDs, the `[h:: …]` identities, `#task`
  extraction, and marker-mirror detection are therefore unchanged.

### 4. States (all visible in the note, none fail the PDF)

| State       | Condition                                                                            | Rendered at the image's position                                                                       |
| ----------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Embedded    | vault copy exists, or source readable (copy planned)                                 | `> ![[<vault path>]]` or `> ![[<vault path>\|<w>]]`                                                    |
| Not found   | no vault copy and source missing on this host                                        | `> > [!warning] Image not found` + `` > > `<source_text>` ``                                           |
| Unreadable  | no vault copy and source read failed for another reason (permission, is a directory) | `> > [!warning] Image unreadable` + `` > > `<source_text>` ``                                          |
| Unsupported | recognized but unsupported extension                                                 | `> > [!warning] Image type not supported` + ``> > `<source_text>` — use PNG, JPEG, GIF, WebP, or SVG`` |

Each warning callout is preceded by a bare `>` spacer, like an image. These states are
self-healing: the next scan that can read the file copies it and replaces the warning
with the image. They never fail the PDF, never change the exit status, and never block
other annotations.

### 5. Sizing

Retina screenshots should appear at the size they had on screen, not at 2×:

- Read the PNG `IHDR` width and the `pHYs` chunk by hand. Scan chunks up to the first
  `IDAT`; no new dependency.
- If the unit is meters and the density exceeds 1.25 × 2835 px/m (more than about 90
  dpi): logical width = round(px_width × 2835 / px_per_meter).
- Emit `|<logical>` only when the logical width is below the pixel width and at most
  `600` (constant `IMAGE_NOTE_MAX_SIZED_WIDTH`, which fits inside a callout at
  Obsidian's default 700 px line width).
- Otherwise emit no hint, and Obsidian fits the image to the callout.
- Compute the width from the vault copy when it exists, else from the source bytes.
  Non-PNG and malformed files get no hint, and a malformed file is not an error.

Examples:

- A full-screen 2880 px capture (logical 1440) gets no hint and fits the callout.
- An 840 px region capture at 144 dpi renders at `|420`.

### 6. Sync semantics

These rules apply under the default `refresh = snapshot` decision.

- **Copy once, never overwrite.** If the destination exists, it is used as is. The
  source is not even read. If it does not exist and the source is readable, plan an
  `ImageAssetWrite` with action `Copy`, and execution copies it atomically.
- **To refresh an image**, delete its file under `<stem>.assets/` or paste a new
  screenshot path. The next scan re-imports it.
- Because the rendered region depends only on sidecar text plus bytes that every host
  converges on via git, scans on different hosts produce the same note and never flap.
- Planned copies join `plan.image_assets`, so they inherit every existing path
  unchanged:
  - the dirty-target guard;
  - the intake commit path set;
  - the asset collision check;
  - the v2 precondition check (a destination that appears with different bytes between
    plan and write still errors);
  - `execute_image_asset_write` (copy if absent, skip if identical, error if different);
  - `write_summary` and the `image assets` counts.
- Existing copies produce no write entry.
- Dry runs plan and report copies, render the region as if they were copied, and write
  nothing.

> [!decision] refresh = mirror Replace "copy once" with "mirror while present, keep when
> gone".
>
> - When the destination exists _and_ the source is readable with different bytes, plan
>   a new `ImageAssetAction::Update { expected_dest_sha256 }`.
> - `execute_image_asset_write` must re-hash the destination immediately before
>   overwriting and fail with "changed on disk after planning" on mismatch.
> - `validate_v2_asset_preconditions` must accept `Update` entries whose destination
>   still matches `expected_dest_sha256`.
> - The source is read on every scan.
> - Docs say the vault copy mirrors the file while it exists and keeps the last version
>   once the file is gone. They also warn that hosts holding different bytes at one path
>   will overwrite each other.

### 7. Reporting

- `RenderedHighlights` gains `image_note_count: usize` and
  `image_notes_unresolved: Vec<UnresolvedImageNote { source_text, state }>`, where
  `state` is `not_found`, `unreadable`, or `unsupported`. `SyncWriteReport` gains the
  same two values.
- **Verbose and single-PDF plan/write reports** (`report.rs`, next to the existing
  `images:` lines):
  - `image_notes: N` when N > 0;
  - one `image_note_unresolved: <source_text> (<state>)` line per unresolved image note.
- **Concise scan details** (`scan_details`): add `N image note(s)`, plus a yellow
  `N unresolved` detail when any are unresolved. A note's line only prints when it
  changed, so the first scan that discovers a missing image says so.
- **Scan JSON:** `ScanJsonNote` gains additive `image_notes` (count) and
  `image_notes_unresolved` (array of `{source, state}`), both `skip_serializing_if`
  zero/empty. Existing JSON stays byte-identical for notes without image notes.
- **Steady state** (still missing, note unchanged) is surfaced by the note's own warning
  callout and by `bob ref doctor` (§8), not by repeating the warning on every cron tick.

### 8. Read side

- **`region.rs`:** `RegionBlock` gains `images: Vec<String>` (vault paths, without any
  `|width`) and `unresolved_images: Vec<String>` (source texts from the warning
  callouts). Generalize body classification for `quote`, `quote Image`, and `note`
  blocks:
  - After the header line, split the remaining lines into depth-1 lines and depth-2
    callout groups. Each group starts after a bare `>` spacer with a `[!type] …` header.
  - A depth-1 line that is exactly `![[<target>]]` or `![[<target>|<digits>]]` with a
    supported image extension is an image, not quote or note text.
  - A depth-2 `[!warning] Image …` or `[!warning] Image type not supported` group
    contributes its backticked source text to `unresolved_images`.
  - A depth-2 `[!note] Comment …` group is the comment, and must be the last group.
  - Any other depth-2 group keeps today's `unparsed` behavior.
  - Trim blank ends of quote/note text so the spacer lines do not leak.
  - Notes now allow depth-2 warning groups.
  - Existing notes without image notes must parse exactly as before.
- **`ref_library/mod.rs::annotation_counts`:**
  - A block with images or unresolved images counts as commented. A screenshot is
    Bryan's own annotation.
  - Add a row diagnostic `unresolved_image_note`:
    `"<N> image note(s) not in the vault yet: <first source> (+<N-1> more)"`.
    `bob ref doctor`'s `library diagnostics` rollup then lists it with example paths. No
    doctor code change should be needed beyond confirming that.
- **`ref_library/show.rs`:**
  - **JSON:** each annotation gets always-present `images` and `unresolved_images`
    arrays (additive under `schema_version` 1, next to `asset`).
  - **`-c/--comments-only`:** keeps annotations that have images or unresolved images.
  - **Human:** under the quote, print `[image: <vault path>]` in the existing
    `[image: …]` style, and `[image not found: <source>]` (or `unreadable` /
    `unsupported`) in yellow.
  - **Markdown:** after the quote, print `Image: <vault path>` lines (the existing
    image-selection form) and `Image not found: <source>` lines. An agent reading the
    digest can then open the screenshot file.

## Implementation steps

1. **New module `src/native/highlights_ref/image_notes.rs`** (register in `mod.rs`;
   follow the module's `use super::*;` idiom). It holds:
   - `ImageNoteRef`;
   - `parse_image_note_line`;
   - `split_image_note_lines(text) -> (Vec<ImageNoteRef>, String)`, which keeps
     paragraph breaks;
   - `image_note_dest_path(ref_note_path, &ImageNoteRef)`, built on
     `ref_note_assets_dir`;
   - `resolve_image_note(config, ref_note_path, &ImageNoteRef) -> Result<ImageNoteResolution>`,
     with outcomes `Embedded { vault_path, width, write: Option<ImageAssetWrite> }` and
     `Unresolved { state }`;
   - `retina_display_width(bytes) -> Option<u32>`.

   Keep `~/` expansion in a small pure helper that takes `home: &Path`, so unit tests
   never touch `$HOME`. Unexpected IO on the _destination_ side is a real error.
   Source-side failures map to `not_found` / `unreadable`.

2. **Render integration** (`sidecar_render.rs`):
   - In `render_sidecar_highlights`, split each annotation's comment source
     (`task_source.unwrap_or(comment)`), or a standalone note's text, then resolve each
     ref.
   - Merge planned writes into `image_asset_writes_by_dest`, which already dedupes by
     destination.
   - Fill the new `RenderedHighlights` fields.
   - Extend `render_annotation_block` / `push_annotation_comment_callout` to emit images
     and warnings in the §3/§4 layout, with the spacers.
   - Keep `annotation_block_id` and every identity input untouched.
3. **Model and plumbing** (`model.rs`, `sync.rs`, `report.rs`, `scan_json.rs`):
   - Thread the new counts into `SyncWriteReport` at both report construction sites (v1
     ~`sync.rs:1369`, v2 ~`sync.rs:1700`).
   - Add the §7 report lines, details, and JSON fields.
   - Reuse `ImageAssetWrite` for copies: set `block_id` to the owning annotation's block
     ID and `source_sha256` from the bytes read at plan time.
   - `validate_v2_asset_preconditions` and `execute_image_asset_write` need no change
     for snapshot.
4. **Region parser** (`region.rs`) per §8, plus every `RegionBlock` construction site.
5. **ref_library** (`mod.rs` counts and diagnostic, `show.rs` fields and the three
   renderers) per §8.
6. **Docs:**
   - **`docs/highlights-ref-sync.md`:**
     - Add an **Image notes** subsection under _Generated Body Contract_, with a
       _Contents_ entry. Cover the grammar with examples, the storage path and naming,
       the rendering examples, the state table, Retina sizing, snapshot semantics with
       the refresh recipe, the host note (the Mac scan resolves macscrot paths;
       athena/apollo hold uploads too), and a troubleshooting checklist
       (`bob ref sync <pdf> --dry-run` shows `image_notes:` / `image_note_unresolved:`;
       `bob ref doctor` lists `unresolved_image_note`).
     - Add one line in _What it does_ and one bullet in the MacBook validation
       checklist.
   - **`docs/ref.md`:** add the `bob ref show` `images` / `unresolved_images` fields,
     the `-c` semantics, and the `unresolved_image_note` diagnostic.
   - **`README.md`:** add one sentence on the `scan` bullet.
   - No CLI help text changes; no options are added.

## Tests

Unit tests go in `src/native/highlights_ref/tests/` (add `image_notes.rs` and extend
`sidecar.rs` / `region.rs`). Use temp vault helpers like `temp_bob_dir` /
`test_config_for_bob_dir`, absolute temp paths for sources, and a tiny hand-built PNG
with and without `pHYs`.

- **Grammar table:**
  - Accepted: tilde, absolute, `file://` with `%20`, backticks, angle brackets, straight
    and smart quotes, list markers, a `Comment:` label, a trailing period, escaped
    spaces, a name with spaces, uppercase `.PNG`, and a multi-token line.
  - Rejected: prose with an inline path, a relative path, and non-image extensions.
  - `.heic` yields an unsupported ref.
- **Naming:** deterministic; sanitized; two paths with the same basename get different
  names; never an `h-` collision.
- **Sizing:** 144 dpi narrow PNG → hint; 72 dpi → none; no `pHYs` → none; logical > 600
  → none; JPEG / garbage → none.
- **Render:**
  - image-only highlight;
  - image + comment, with paragraphs preserved;
  - two images;
  - not found / unreadable / unsupported;
  - standalone-note image;
  - image selection + image note;
  - the same path on two highlights → one write;
  - an existing copy with a missing source still embeds and plans no write;
  - **block IDs and annotation-task candidates are identical with and without an image
    line in the comment.**
- **Region round-trip:**
  - Each render case above → `parse_managed_region` gives the right `quote` / `comment`
    / `images` / `unresolved_images` and no `unparsed`.
  - Existing region fixtures parse unchanged.
- **CLI integration** (`tests/cli/highlights/`, new `image_notes.rs` or `scan.rs`). Run
  the binary with `HOME` set to a temp dir containing
  `tmp/screenshots/20261018_104022.png`, and a sidecar whose highlight comment is
  `~/tmp/screenshots/20261018_104022.png`.
  - `bob ref scan --dry-run` writes nothing and reports the planned copy.
  - `bob ref scan` creates `ref/<type>/<stem>.assets/20261018_104022-<hash8>.png`
    byte-identical to the source and embeds it under the quote.
  - A second scan reports `writes: none`.
  - After deleting the source, a further scan still reports `writes: none` and keeps the
    embed.
  - A sidecar pointing at a missing file yields the warning callout, a zero exit, and an
    `unresolved_image_note` row in `bob ref doctor`. After the file is created, the next
    scan swaps in the image.
  - `bob ref show <ref> -f json` lists `images`.

## Verification

Run `just check`: `cargo fmt --check`, clippy with the repo gate, and
`cargo test --no-fail-fast`. All three must pass.

Manually, preview a real note on athena without writing:

```bash
bob ref sync ~/bob/lib/<type>/<pdf>.pdf --dry-run
```

Use a PDF whose sidecar has a comment pointing at a file in `~/tmp/screenshots/`.

## Non-goals

- Converting HEIC/TIFF/BMP/AVIF; size caps; garbage-collecting unreferenced copies
  (TextBundle assets have the same deferral).
- Fetching images from other hosts.
- Writing anything back to the PDF or sidecar.
- Changes to bob-plugins or Bob Mac Capture.
