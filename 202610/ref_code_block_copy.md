---
tier: epic
title: Exact code-block copy from Bob Refs, and Command-Y in Highlights
goal: 'While reading a bob-rendered reference in Highlights, Command-Y C Return puts
  the code block on the current page on the clipboard byte for byte (tabs, indentation,
  long lines, Mermaid source). The original Markdown of every new reference is preserved
  in the vault, one key (M) away, and `bob ref code` gives agents and the CLI the
  same data.

  '
decisions:
  pointer_home:
    ask: Where should a reference record the pointer to its saved Markdown?
    choices:
      note: Note frontmatter only, derived at birth; PDFs never change and backfill
        edits notes only
      marker: Page-1 marker plus frontmatter; backfill later makes scans rewrite ~430
        library PDFs
    default: note
    why: Every intake path still finds the store, and no PDF or Highlights tab is
      ever touched
    answer: note
  backfill:
    ask: How far should this epic take saved sources for existing agent-report references?
    choices:
      run: Build bob ref attach-source, dry-run it, then attach verified sources on
        athena in batches
      build: Build and dry-run the command only; Bryan runs the write pass himself
      defer: Skip the attach phase; its worker files a follow-up task instead
    default: run
    why: About 430 existing reports gain code copy at once; every batch is one revertable
      commit
    answer: run
  record_decisions:
    ask: Add decision records for the source store and the render-time code catalog?
    memory:
    - decisions
    default: true
    requested: Review the ref_code_block_copy.md file in the research sidecar repo
      for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file (but keep in mind that you are responsible
      for planning and splitting up this work appropriately).
    answer: true
phases:
- id: walkers
  title: Keep bob's note walkers out of .bob
  depends_on: []
  size: small
  description: 'walkers: add .bob to the always-excluded note directories so task
    archive, link repair, project scans, and gkeep/articles walkers can never read
    or edit a saved source, with regression tests.'
- id: catalog
  title: Render-time code catalog on both pandoc routes
  depends_on: []
  size: medium
  description: 'catalog: add the two-pass code-catalog Lua filter, invisible per-block
    PDF anchors, --preserve-tabs with filter-side tab expansion, the frozen render
    input, and the promoted /Dests reader, so every Markdown and snapshot render returns
    an exact CodeCatalog with pages while the PDF stays unchanged.'
- id: store
  title: Write-once source store and the source_markdown pointer
  depends_on:
  - walkers
  - catalog
  size: medium
  description: 'store: persist source.md plus code.json under .bob/ref-sources at
    create time, record the source_markdown pointer through every intake path, include
    the store in the ingest commit, warn when the catalog is gitignored, allow it
    in athena''s vault .gitignore, and expose the pointer on RefRow.'
- id: code-cmd
  title: bob ref code read surface
  depends_on:
  - store
  size: medium
  description: 'code-cmd: add the read-only bob ref code command (human, json, markdown,
    raw, page filter, saved source) with its versioned JSON contract, bob-computed
    summaries and fenced forms, help, completion, and docs.'
- id: lifecycle
  title: Overwrite repointing and the doctor sources check
  depends_on:
  - store
  size: medium
  description: 'lifecycle: make -f overwrite stage, install, repoint, and roll back
    the store inside its transaction, and add a bob ref doctor ref-sources row for
    dangling pointers, digest mismatches, orphans, and an ignored catalog.'
- id: attach
  title: Verified attach of existing references' sources
  depends_on:
  - store
  - code-cmd
  size: medium
  description: 'attach: add bob ref attach-source, which finds same-stem Markdown
    candidates, verifies them against the library PDF text, and attaches only verified
    sources; then dry-run and run it on athena per the backfill decision.'
- id: mac-key
  title: Command-Y takeover in Highlights and type-ahead
  depends_on: []
  size: medium
  description: 'mac-key: register Command-Y as a Highlights takeover that opens the
    Command-T session with the Copy card on the focused PDF, buffer keys typed before
    the card exists, and queue card keys behind a pending load so fast gestures always
    do what was typed.'
- id: mac-core
  title: RefsCore code catalog, C and M rows, and the code picker model
  depends_on:
  - code-cmd
  - mac-key
  size: medium
  description: 'mac-core: decode source_markdown and the bob ref code JSON, add the
    C and M rows with their hidden reasons, the pure RefsCodePicker with page-aware
    initial selection, the preview-well model, and a shared sectioned window, all
    unit-tested in RefsCore.'
- id: mac-ui
  title: Code picker, preview well, and panel wiring
  depends_on:
  - mac-core
  size: medium
  description: 'mac-ui: fetch the catalog when the card opens, draw the C and M rows,
    the code picker, and the fixed-height preview well in light and dark, add HUD
    strings, VoiceOver, design fixtures, and docs, and verify the Command-Y C Return
    gesture on the Mac.'
- id: records
  title: Decision records
  depends_on:
  - lifecycle
  - mac-ui
  size: xsmall
  description: 'records: author the two decisions-web records for the write-once source
    store and the render-time code catalog, citing the landed commits and the research
    report.'
proposed_by: bbugyi200.athena.research.5b.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 09:00:54
status: wip
bead_id: bob-cli-cc
---

- **PROMPT:** [prompts/202610/ref_code_block_copy.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_code_block_copy.md)
- **BEAD:** [bob-cli-cc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-cc/README.md)

# Plan: Exact code-block copy from Bob Refs, and Command-Y in Highlights

## The experience we are building

Bryan is reading an agent report in Highlights. Page 7 has a YAML block he wants in a
terminal. He presses **⌘Y**: Bob Refs appears over Highlights with the Copy card open on
that PDF. The new **C · Code block…** row already says
`6 blocks · ● 2 on this page · yaml, rust, bash`. He presses **C**: the code picker
opens with page 7's first block selected and a preview well showing exactly what will
paste, true indentation and all. He presses **↵**: the pasteboard is verified, Refs
hides, a glass HUD says `Copied yaml · 14 lines`, and Highlights is still frontmost, so
⌘V works at once.

`⌘Y C ↵` does the right thing even when typed faster than Bob can read Highlights.
`⌘Y ⌘Y` keeps its meaning (copy the default row). **M** copies the whole original
Markdown. `bob ref code <ref> 2 | pbcopy` gives the CLI and agents the same bytes.

## Design principles

1. **The card stays the card.** No row is re-lettered or reordered. `C` and `M` are new
   fixed letters in a new group; rows that do not apply hide and explain themselves when
   their key is pressed. The default row and selection stay `I`/`W`.
2. **What you see is what you paste.** The picker's preview well shows the payload
   itself: no soft wrapping, no line numbers, tabs drawn visibly. Nothing is ever copied
   without being shown first, even when a reference has one block.
3. **Start where the reader is.** When Bob knows Highlights' page, the C row counts that
   page's blocks and the picker opens on them.
4. **Exact by default, Markdown on request.** `↵` copies the plain block; `⌥↵` copies it
   fenced. ⌥ swaps previews in place, as everywhere else on the card.
5. **One source of truth.** bob records the copy text while it renders the PDF. The PDF
   text layer is never a copy source, and Bob Mac Capture only presents what `bob`
   returns (`decisions:mac-capture-is-a-thin-client`).
6. **Fast keys are never misread.** Keys typed before the card exists, or while it waits
   for data, are queued and replayed in order.

## Where this plan departs from the research

The research report (`research:202610/ref_code_block_copy/ref_code_block_copy.md`) is
the basis. Its data, UX, and store design are adopted. These points change:

- **The pointer is note-local by default** (decision `pointer_home`). The research
  stamped `source_markdown` into the page-1 PDF marker and left that to Bryan. But a
  synced marker field means the backfill's frontmatter edits make the next coordinated
  scan rewrite about 430 annotated library PDFs. Instead, the pointer is derived **at
  birth** in the shared single-PDF sync plan, exactly the way `birth.rs` derives
  `earlier:` lineage: the intake that creates the note looks up the store by reference
  type, stem, and the incoming PDF's SHA-256. Inline intake and queued drains share that
  code path, so both get the pointer, and no PDF or open Highlights tab ever changes.
- **The JSON is complete.** `bob ref code -f json` carries every byte the card can copy,
  including the saved Markdown text, so the card needs one request and never fetches at
  copy time. The research's `too_large` lazy path is dropped (median source 30 KB).
- **Two key queues, not one.** Besides the pre-card buffer for ⌘Y, the card itself
  queues keys behind any pending load. Without it, `C ↵` typed before the catalog
  arrives would copy the default row instead.
- **The store is written under `bob_sync.lock`**, so a concurrent vault sync can never
  commit half a store.
- **A sixth status, `no_catalog`,** separates "source saved, catalog missing" from "no
  code", so M keeps working when only the catalog failed.
- **The backfill command is `bob ref attach-source`**, which also attaches a single
  reference to a file you name, rather than a one-off `backfill-sources`.
- **The preview well draws tabs** as a faint `→`, so a Makefile's real tabs are visible
  before you paste.

## Architecture at a glance

```text
bob ref create report.md                                   (bob-cli, any host)
  frozen copy of the input ─► pandoc ─► code_catalog.lua records each CodeBlock
       │                         │      + \BobCodeAnchor before/after each block
       │                         ▼
       │                   render.pdf ─► /Dests reader ─► pages per block
       ▼                                                     │
  .bob/ref-sources/<type>/<stem>--<hash12>/source.md   ◄─────┤ (write-once, under bob_sync.lock)
  .bob/ref-sources/<type>/<stem>--<hash12>/code.json   ◄─────┘ (catalog + PDF sha at birth)
       │
  intake (inline or queued drain): new note ─► birth lookup (type, stem, PDF sha)
       └─► note frontmatter  source_markdown: .bob/ref-sources/…/source.md

bob ref code <ref> -f json ─► verified catalog + source text   (read-only)
       │
Bob Mac Capture: Copy card C / M rows ─► code picker + preview well ─► pasteboard
       ▲
⌘Y in Highlights ─► ⌘T-style session ─► focused PDF + page ─► Copy card (keys replayed)
```

## Shared contracts

Every phase builds against these. A phase that must deviate records why in its bead and
updates this contract's docs.

### Store layout

```text
~/bob/.bob/ref-sources/<ref_type>/<stem>--<hash12>/
  source.md    exact Markdown bytes pandoc read (immutable)
  code.json    render-time catalog, version 1
```

- `hash12` is the first 12 hex digits of SHA-256(`source.md`). `<stem>` is the final
  target stem at birth (after `_2` collision suffixes and `__vN` rereads). `<ref_type>`
  is the type directory the PDF lands under in `lib/`.
- `source.md` is write-once. An existing directory whose `source.md` digest matches is
  reused. A digest mismatch refuses that store write with a warning and never
  overwrites.
- `code.json` is derived data. A later render of the same bytes for the same stem (an
  in-place overwrite) may replace it atomically.
- Writes take `bob_sync.lock` (unless the caller already holds it): `source.md` first,
  then `code.json`, each written to a temp file and renamed. A reader treats a directory
  without `code.json` as `no_catalog`.
- The store is hidden from Obsidian (it is a dot-folder) and from bob's walkers (phase
  `walkers`). It syncs through vault git: `source.md` is covered by `!*.md`, and
  `code.json` needs `!/.bob/ref-sources/**/*.json` (phase `store`).
- `rename` never moves a store. A reread makes a new store. An in-place overwrite makes
  a new store when the bytes changed; the old one stays as history.

`code.json` version 1:

```json
{
  "version": 1,
  "captured_at": "2026-10-20T14:03:11-04:00",
  "route": "markdown",
  "source": {
    "sha256": "5d3be944e47e…(64 hex)",
    "bytes": 41213,
    "origin": "/home/bryan/…/202610/ace_refresh_loop/ace_refresh_loop.md",
    "source_note": null,
    "source_note_sha256": null
  },
  "pdf": { "path": "lib/chat/ace_refresh_loop.pdf", "sha256": "…(64 hex)" },
  "renderer": { "bob": "0.x.y", "pandoc": "3.1.11.1" },
  "catalog_error": null,
  "blocks": [
    {
      "n": 2,
      "kind": "code",
      "lang": "yaml",
      "classes": ["yaml"],
      "attrs": {},
      "text": "refresh:\n  interval_ms: 250\n…",
      "section": { "number": "3", "title": "Root cause" },
      "page": 7,
      "page_end": 7,
      "page_source": "anchor"
    }
  ]
}
```

- `route` is `markdown` (a plain `.md` file), `plain_note` (`-M` on a vault note),
  `note_snapshot` (the projected snapshot), or `attached` (phase `attach`).
- `pdf.sha256` is the SHA-256 of the PDF exactly as create installed it. It is the
  birth-lookup key.
- `kind` is `mermaid` when the classes include `mermaid`, else `code`. `lang` is the
  first class or `null`. `n` is 1-based document order over every `CodeBlock`, including
  blocks in lists and quotes.
- `page`/`page_end` are 1-based physical pages (library PDFs have no `/PageLabels`, so
  they equal Highlights' "Page N"). They are `null`, with `page_source: null`, when no
  anchor resolved.
- `catalog_error` is set, and `blocks` is `[]`, when the render's catalog report was
  missing or invalid.

### The `source_markdown` pointer

The note frontmatter field `source_markdown` holds the vault-relative path to
`source.md`, for example
`.bob/ref-sources/chat/ace_refresh_loop--5d3be944e47e/source.md`. It is a plain string,
not a wikilink, because Obsidian cannot open dot-folders.

> [!decision] pointer_home = note `source_markdown` is a note-local, bob-managed field.
> It never enters the PDF marker, projection, or hash. It is derived once, when the
> shared single-PDF sync plan creates a new note (`plan_single_pdf_sync`, alongside
> `birth::derive_birth_lineage`), by
> `find_birth_store(vault, ref_type, stem, pdf_sha256)`: list
> `.bob/ref-sources/<ref_type>/<stem>--*/code.json` and take the single store whose
> `pdf.sha256` equals the incoming PDF's digest. It is carried as birth-only plan state,
> and repeat syncs preserve it verbatim, like `created` and `earlier:`. No match means
> no pointer. Overwrite and attach set it explicitly.

> [!decision] pointer_home = marker `source_markdown` becomes a standard synced user
> field. Add it to `COMMON_USER_FIELDS` (`src/native/highlights_ref/mod.rs`) and
> `MARKER_EXTRA_ORDER` (`stamp.rs`), pass it through `compose_markdown_marker`,
> `stamp_marker_for_outcome`, and the snapshot route's marker, and let intake copy it to
> frontmatter as it does `source_note`. Because the pointer is content-addressed, create
> can compute it before stamping and write the store after. Overwrite repoints it
> through the staged marker, and its absence clears it. A note-side edit by `attach`
> propagates to the PDF marker on the next coordinated `bob ref scan -w`. Document that
> one-time PDF rewrite.

### Clipboard contract

- **Plain (default):** the block's recorded `text`. Fences and container indentation are
  removed. Tabs, spaces, blank lines, trailing spaces, and Unicode are kept. Lines end
  in LF, and no trailing newline is added, so a pasted shell command never runs itself.
- **Fenced (⌥):** a fence of backticks one longer than the longest backtick run in the
  text (minimum 3), then `lang` (if any), a newline, the text, a newline, the closing
  fence, and a final newline.
- **Markdown (M):** the exact `source.md` text. **⌥M** copies its `~`-abbreviated
  absolute path.
- **Never:** provenance comments, prompt stripping, Unicode normalization, or
  truncation.

### `bob ref code`

```text
bob ref code <REF> [N] [-f, --format human|json|markdown|raw] [-p, --page PAGE] [-s, --source]
```

- `REF` resolves exactly like `bob ref show` (`ref_library::show::resolve_show`).
- Without `N` the default is `human`: a colored list grouped by printed section showing
  the block number, a language chip, a page label, the line count, and the summary. It
  ends with a `hint: bob ref code <ref> 2 | pbcopy` line, where `<ref>` is the shortest
  spelling that resolves uniquely.
- With `N` the default is `raw`: exactly that block's `text` bytes, so
  `bob ref code x 2 | pbcopy` reproduces the card's copy byte for byte. `-f markdown`
  prints the fenced form. `-f json` keeps only that block.
- `-p PAGE` keeps blocks whose `[page, page_end]` contains `PAGE`.
- `-s` prints the saved `source.md` bytes and conflicts with `N`, `-p`, and `-f`.
- `-f markdown` (agents): `# <title>`, then for each section that has blocks a
  `## <number> <title>` heading, then each block as `Block <n> · <lang> · p. <page>`
  followed by its fenced form.
- The command is read-only, never runs pandoc, and verifies the source digest.
- Exit codes:
  - 0 for success, and for every JSON response that reports a `status`.
  - 1 for an unknown or ambiguous reference (`ref show`'s error envelope and codes), a
    missing block `N` ("no block 7 (6 blocks)"), or any non-`available` status in a
    non-JSON format. The last case prints a one-line reason and a hint, except that
    `no_code` with human output prints "no code blocks" and exits 0.
  - 2 for usage errors.

JSON (schema version 1, the envelope shape of `ref show`):

````json
{
  "ok": true,
  "schema_version": 1,
  "command": "ref code",
  "generated_at": "2026-10-26T09:12:44-04:00",
  "reference": {
    "path": "ref/chat/ace_refresh_loop.md",
    "title": "Why `sase ace` Is Slow…",
    "marker_id": "ace_refresh_loop"
  },
  "status": "available",
  "detail": null,
  "source": {
    "path": ".bob/ref-sources/chat/ace_refresh_loop--5d3be944e47e/source.md",
    "sha256": "…",
    "bytes": 41213,
    "captured": "2026-10-20",
    "route": "markdown",
    "text": "# Why `sase ace` Is Slow…"
  },
  "blocks": [
    {
      "id": "5d3be944e47e/2",
      "n": 2,
      "kind": "code",
      "lang": "yaml",
      "title": null,
      "summary": "refresh:",
      "caption": "interval_ms: 250",
      "lines": 14,
      "bytes": 312,
      "text": "refresh:\n  interval_ms: 250\n…",
      "fenced": "```yaml\nrefresh:\n…\n```\n",
      "section": { "number": "3", "title": "Root cause" },
      "page": 7,
      "page_end": 7,
      "page_source": "anchor"
    }
  ]
}
````

- `status`:

  | Status           | Meaning                                                               |
  | ---------------- | --------------------------------------------------------------------- |
  | `available`      | Verified, at least one block.                                         |
  | `no_code`        | Verified, zero blocks.                                                |
  | `no_catalog`     | Source verified, catalog missing or recorded with `catalog_error`.    |
  | `no_source`      | No pointer.                                                           |
  | `source_missing` | Pointer set, but `source.md` is absent on this host.                  |
  | `source_invalid` | Digest mismatch, unreadable file, or unsupported `code.json` version. |

  `source.text` is present exactly when `source.md` verified (`available`, `no_code`,
  `no_catalog`). `detail` is a human sentence for non-`available` statuses.

- `id` is generation-scoped (`hash12/n`), so a recapture can never substitute a
  different block.
- `title` is the `title=` attribute. `summary` is the first meaningful line: trimmed,
  skipping blank lines, rule lines (`---`, `===`, `***`), lines made only of box-drawing
  characters, and lone brackets or braces. For Mermaid, a leading `---`…`---`
  frontmatter block is also skipped, so the summary is the diagram type line.
- `caption` is `title`, else the Mermaid frontmatter `title:`, else the next meaningful
  line, else `null`. `lines` is the line count (0 for empty text). `bytes` is the UTF-8
  length.
- Growth is additive: the app decodes everything except `schema_version` loosely.

## Bob Mac Capture UX

### Copy card: C and M rows

A new group, `.contents`, sits between Files and Your Notes, separated by the existing
hairline and without a header:

```text
╭───────────────────────────────────────────────────────────────────╮
│ ▣ COPY                                            p. 7 in Highlights │
│   Why `sase ace` Is Slow: A Chronic Refresh Loop Plus an Acute…    │
│ ▌[I] Marker ID      ace_refresh_loop                               │
│  [T] Title          Why `sase ace` Is Slow: A Chronic Refresh…     │
│  [W] Wiki link      [[ref/chat/ace_refresh_loop]]                  │
│  ───────────────────────────────────────────────────────────────── │
│  [N] Note path …  [O] …  [P] …  [F] …  [R] …                      │
│  ───────────────────────────────────────────────────────────────── │
│  [C] Code block…    6 blocks · ● 2 on this page · yaml, rust, bash │
│  [M] Markdown       source.md · 41 KB · captured Oct 20            │
│  ───────────────────────────────────────────────────────────────── │
│  YOUR NOTES 3 · quote + comment + link                             │
│ ↵ copies Marker ID · ↑↓ choose · esc back                          │
╰───────────────────────────────────────────────────────────────────╯
```

- **C · Code block…** opens the code picker and copies nothing itself, like `H`. Its
  preview is `N blocks`, then `● K on this page` (the dot in accent color) when a page
  is known and K > 0, then up to three languages by frequency (`mermaid` included).
  - While the catalog loads, the preview reads `Loading code…`. Pressing C then goes
    pending ("Loading code… esc cancels") and opens the picker when the catalog arrives.
- **M · Markdown** copies the whole saved source. Its preview is
  `source.md · 41 KB · captured Oct 20`.
  - ⌥ swaps it to `Markdown path · ~`, which copies
    `~/bob/.bob/ref-sources/…/source.md`. That path also works for agents on athena.
- **Header chip** `p. 7 in Highlights` (caption2, accent tint) appears on the card and
  the picker whenever a page is known.
- **Page known** means the selected row is the focused Highlights tab in the current
  Tabs listing and that tab has a page. That covers every Tabs show, including ⌘Y. The
  page is frozen when the card opens.
- **Picker rows show what Return does.** When `C` or `H` is selected, the hint reads
  `↵ opens Code block…`.
- **Visibility.** Rows that do not apply are hidden. The reason appears in the message
  line when their key is pressed:

  | State                         | C       | M       | Message on key press                                                       |
  | ----------------------------- | ------- | ------- | -------------------------------------------------------------------------- |
  | item has no `source_markdown` | hidden  | hidden  | No saved Markdown — code copy needs a reference bob rendered from Markdown |
  | catalog loading               | loading | loading | (pending) Loading code… esc cancels                                        |
  | `available`                   | shown   | shown   | —                                                                          |
  | `no_code`                     | hidden  | shown   | No code blocks in this reference — M copies its Markdown                   |
  | `no_catalog`                  | hidden  | shown   | Code blocks weren't recorded for this reference — M copies its Markdown    |
  | `source_missing`              | hidden  | hidden  | Saved Markdown isn't on this Mac yet — sync the vault                      |
  | `source_invalid`              | hidden  | hidden  | Saved Markdown doesn't match its digest — run bob ref doctor               |
  | older `bob` (no `ref code`)   | hidden  | hidden  | Code copy needs a newer bob                                                |
  | fetch failed                  | hidden  | hidden  | Code unavailable — nothing copied                                          |

### The code picker

```text
╭───────────────────────────────────────────────────────────────────╮
│ ▣ CODE BLOCKS 6                                   p. 7 in Highlights │
│   Why `sase ace` Is Slow: A Chronic Refresh Loop Plus an Acute…    │
│   2  Reproduction                                                  │
│  [1] bash    sase ace --profile 2>&1 | tee ace.log            p. 5 │
│              3 lines                                               │
│   3  Root cause                                                    │
│ ▌[2] yaml    refresh:                                       ● p. 7 │
│              interval_ms: 250 · 14 lines                           │
│  [3] rust    fn refresh_loop(state: &mut AppState) {     ● pp. 7–8 │
│              41 lines                                              │
│ ╭ yaml · Root cause ───────────────────────── 14 lines · p. 7 ╮    │
│ │ refresh:                                                    │    │
│ │   interval_ms: 250                                          │    │
│ │   jitter_ms: 40                                             │    │
│ │ →   on_focus: true                                          │    │
│ │   …                                         +6 more lines   │    │
│ ╰─────────────────────────────────────────────────────────────╯    │
│ ↵ copies · ⌥↵ fenced · 1–9 pick · ↑↓ choose · esc back             │
╰───────────────────────────────────────────────────────────────────╯
```

- **Keys** follow `H`:
  - `↑↓` and `⌃N/P/J/K` move the selection and the preview; the window scrolls.
  - `1`–`9` copy the block at that visible position.
  - `↵`, ⌘Y, or a click copies the selection plain. Holding ⌥ shows the fenced form;
    `⌥↵`, ⌥+digit, and ⌥-click copy it.
  - `esc`/`⌃[` returns to the card. A letter with no binding shows `Nothing on ‘x’` and
    changes nothing.
- **Rows** use the 36 pt picker metrics.
  - Line 1: keycap digit, then a lowercase monospaced language chip in a quiet capsule
    (`primary` at 6% fill, a fixed minimum width so summaries align; untagged blocks
    read `text`). Then the summary in the card's 11.5 pt monospaced primary style. A
    right-aligned page label (`p. 7` or `pp. 7–8`, caption monospacedDigit, secondary)
    is led by an accent `●` when the block's range contains the known page.
  - Line 2: the caption (secondary), then `N lines` (tertiary). Mermaid rows read
    `diagram source · N lines`.
  - There are no per-language hues: the reference-kind tiles already own color.
- **Section headers** use the printed section number in tertiary monospacedDigit plus
  the title, drawn like `H` group titles. They are not selectable, and a window that
  starts mid-section repeats the current header. Blocks before the first heading have no
  header.
- **Preview well** (the one new visual element):
  - Size: a fixed-height rounded rectangle (corner radius 8, matching rows) whose height
    is set once per card. Moving the selection never resizes anything. Its height is 45%
    of the space under the header, clamped to 6–18 code lines plus an 18 pt header
    strip. The row window gets the rest, at most 9 rows and at least 2.
  - Surface: `primary` at 4% (8% with Increase Contrast) on top of the opaque card, with
    a hairline stroke at 10% (25%). It is already opaque under Reduce Transparency.
    Light and dark are both designed.
  - Header strip: `yaml · Root cause` on the left (caption2 semibold, secondary) and
    `14 lines · p. 7` on the right (caption2 monospacedDigit, tertiary). It gains
    `· fenced` while ⌥ is held.
  - Body: 12 pt monospaced, primary color, with no soft wrapping and no line numbers. A
    line wider than the well is clipped behind an 8 pt trailing fade. Each tab is drawn
    as a tertiary `→` followed by spacing to the next 4-column stop (display only). The
    payload is untouched.
  - Overflow: when the block has more lines than fit, the last visible line fades and a
    capsule `+N more lines` (caption2, secondary fill) sits at bottom right.
  - Fenced mode (⌥): fence lines draw in tertiary color.
  - Display caps (first 200 lines, 400 characters per line) bound layout cost and never
    affect the payload.
- **Initial selection.** With page P known, select the first block whose range contains
  P. Otherwise select the first block that starts after P, else the last block before
  it. Scroll so the selection is the first visible row when possible. Without a page,
  select block 1.
- **Never auto-copy**, even with one block. The preview is the product.
- **HUD:**
  - Title: `Copied yaml · 14 lines`, `Copied yaml · fenced`, `Copied Mermaid source`,
    `Copied Markdown · 41 KB`, or `Copied Markdown path`.
  - Value: the block's summary, or the path.
- **VoiceOver:**
  - A row reads "Code block 2 of 6, yaml, 14 lines, page 7, on this page, key 2".
  - The well reads "Preview, 14 lines". Code is never spoken.
  - Opening the picker announces "Code blocks. Press a number or Return."

### ⌘Y in Highlights

We do not ship Highlights, so ⌘Y is a Bob Mac Capture takeover key, like ⌘O and ⌘T.
Highlights 2026.1.2 binds no ⌘Y; the research verified this from its compiled menu.

- **Registration:** `HotKeyAction.refsHighlightsCopy = 5` with a new setting, Settings ›
  References › "Copy key in Highlights: Command-Y / Off" (default Command-Y). It is
  registered only while Highlights (or the override app) is frontmost **and Refs is
  hidden**. While Refs is visible it always steps aside, because the panel's own ⌘Y
  opens the card.
- **Session:** ⌘Y is ⌘T plus Copy. It uses a new presentation source, `.highlightsCopy`,
  which behaves exactly like `.highlightsTabs` (stash the remembered view, fresh
  defaults, Tabs on, focused PDF selected, nothing remembered). It also arms a pending
  copy.
- **While armed:** the inspector column shows the Copy card's skeleton: the `COPY`
  caption, `Finding the open PDF in Highlights…` with a small spinner, and `esc cancels`
  in the message line. Every key except Esc, `⌃[`, and Ctrl+Tab goes into a buffer (cap
  16, original modifiers kept) and never reaches the search field.
- **Resolution:** when the Tabs read completes and the selection is the focused tab
  (`isCurrent`) matched to a library row, open the Copy card on it (page frozen from the
  tab) and replay the buffer in order.
- **Failure:** drop the buffer, stay in the Tabs view, and replace the skeleton with the
  reason:
  - No Accessibility access: today's Tabs explainer.
  - "Highlights has no open document".
  - "“x.pdf” isn't in your Bob library".
  - "Two references match “x.pdf” — pick one, then ⌘Y".
  - "Couldn't read Highlights — ⌘R retries".
  - "Highlights didn't answer — try ⌘Y again" after 2 s without a target.
- **Esc** while armed cancels the pending copy and leaves the plain ⌘T view. Esc from
  the card returns to the ⌘T view, and Esc again hides Refs.
- **After a copy**, Refs hides, the HUD shows, and Highlights stays frontmost.

## Phase walkers

Prerequisite: no bob walker may treat `.bob/**/*.md` as a vault note.
`collect_done::collect_markdown_files` (used by `bob task archive` and by link repair
for task moves and inbox routing), `projects/scan.rs` (projects and the note-ready
scan), the gkeep ledger and migrations, and the `articles` walkers do not skip
dot-directories today. A preserved project note would show up as a duplicate project,
and a saved report's `- [x] … #task` line would be archived out of it.

- Add `".bob"` to `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES` in `src/native.rs`.
- Audit every caller of `is_always_excluded_note_directory_name` and every vault-wide
  `.md` walker. Make sure each one honors the constant: `collect_done/transform.rs`
  `should_skip_directory`, `projects/scan.rs` `is_excluded_directory`,
  `gkeep/ledger.rs`, `gkeep/migrate.rs`, `gkeep/migrate_tasks.rs`, `articles/vault.rs`,
  `articles/marker.rs`, and `capture_related/corpus.rs`. Walkers that already skip
  dot-directories need no change. Keep gkeep's `imports_dir` handling working.
- Regression tests: a fixture vault with
  `.bob/ref-sources/chat/x--0123456789ab/source.md` containing `type: project`
  frontmatter, `- [x] shipped #task ^done1`, an open `- [ ] … #task`,
  `[[some_note#^blk]]` links, and a `%%gkeep:` marker. Assert that:
  - `bob task archive` and `bob task move` link repair leave it byte-identical;
  - project and ready scans do not count it;
  - task-status hooks, `bob query`, and the gkeep ledger ignore it.
- Docs: `docs/reference/vault-format.md` states that `.bob/` is machine data, never
  scanned as notes.
- Done when the tests pass and `just` checks are green.

## Phase catalog

Goal: every Markdown render (the plain `.md` route, `-M` vault notes, and vault-note
snapshots) returns an exact `CodeCatalog`, and the PDF looks exactly as it does today.
Nothing is persisted in this phase.

- **Filter, two passes** (new `src/native/highlights_ref/code_catalog.lua` and
  `code_catalog_report.lua`, embedded like `return_links.lua`):
  - The **record** pass runs first, before the diagrams swap filter, so Mermaid sources
    are still visible. It walks every `CodeBlock` in document order, including blocks in
    lists, quotes, and Divs. It records `n`, `kind`, `lang`, `classes`, `attrs`, and the
    exact `text`, plus the stringified title of the nearest preceding header (by header
    order).
  - It then expands tabs to 4-column stops in that `CodeBlock` and in every inline
    `Code`. That restores today's AST under `--preserve-tabs`.
  - It inserts `RawBlock("latex", "\\BobCodeAnchor{bob-code-N}")` before each block and
    `\\BobCodeEndAnchor{bob-code-N-end}` after it.
  - It hands its records to the report pass through document metadata.
  - The **report** pass runs last, after return links. It assigns each block's section
    from the final AST: titles from the record pass, and numbers emulating LaTeX
    `--number-sections` over the final levels (after `number_from_top_level`), skipping
    `unnumbered`. The number is `null` on routes that do not number sections. The pass
    writes `{version: 1, blocks: […]}` to the path in `-M bob-code-report=<path>`.
  - Neither pass writes to stderr or can make pandoc fail; they use `pcall`, like
    `diagrams.lua`.
- **Anchors** (new `code_catalog.tex`, appended to `header-includes` on both routes):
  - The start anchor is `\par` plus
    `\special{pdf:dest (#1) [@thispage /XYZ null @ypos null]}` plus `\nopagebreak[4]`
    plus `\@nobreaktrue`. This is the heading trick that stops `\addpenalty` before the
    next list or verbatim, so the anchor cannot be stranded at the bottom of a page.
  - The end anchor is the same after `\nopagebreak[4]`.
  - Both are invisible and take no vertical space.
- **Tabs:**
  - Add `--preserve-tabs` to `render_temp_pdf` (`create.rs`), to the snapshot pandoc
    call in `note_snapshot/profile.rs`, and to the diagrams collect pass
    (`diagrams.rs`).
  - Give `diagrams_collect.lua` the identical expansion function before it hashes
    Mermaid text, so swap ids (`sha1(cb.text)`) agree between the collect pass and the
    swap.
  - Share the function's source between the filters (a common Lua prelude concatenated
    in Rust), with a test that both embed it.
- **Frozen input:** `create_markdown_plan` (and the `-M` route) renders from the bytes
  already read into `CreatePlan.source_markdown`. Write them to
  `<scratch>/frozen/<original file name>`, then pass that path to pandoc and to the
  diagrams pre-pass. Keep `--resource-path=<original parent>` so images resolve. The
  snapshot route already renders `<scratch>/projected.md`.
- **Pages:**
  - Promote the test helper `named_dest_pages` (`create.rs` tests) into a production
    module, `pdf_dests.rs` (lopdf is already a normal dependency), and keep the tests
    using it.
  - `code_catalog::build(report, rendered_pdf)` maps `bob-code-N` and `bob-code-N-end`
    to `page`/`page_end` with `page_source: "anchor"`.
  - A missing or invalid report becomes `CatalogOutcome::Missing`/`Invalid`, in the
    shape of `return_links::ReportOutcome`.
- **Plumbing:**
  - `RenderFilters` gains the two filter paths and the report path, all written by
    `write_render_filters`.
  - `render_temp_pdf` and `profile::render_note_pdf` return the catalog outcome beside
    the return-link outcome. Update every caller: listen, overwrite, reread, and the
    snapshot route.
- **Tests:** real pandoc and XeLaTeX, skipped with the existing `skip_pandoc_xelatex`
  pattern when they are missing. Fixtures:
  - a Makefile block with leading tabs and a mid-line tab;
  - a fenced block inside a nested list and inside a blockquote;
  - an untagged block containing a tab;
  - inline code with a tab;
  - a Mermaid block containing a tab, inside a blockquote, still swapped;
  - a block with trailing spaces, Unicode, and emoji;
  - a markdown block that itself contains ``` fences, with a longer outer fence;
  - an 80+-column line;
  - an indented (4-space) code block;
  - a block forced to start at the top of a page, and a block that spans a page break.

  Assertions:
  - Each block's recorded `text` equals the expected bytes exactly.
  - The pandoc native AST after the record pass equals today's AST without
    `--preserve-tabs` for top-level and list-nested blocks. Document the blockquote
    leading-tab difference (4 columns instead of 2).
  - Each anchor `page` equals the page where `pdftotext` finds the block's first line,
    and `page_end` equals the page of its last line, for every fixture.
  - Section numbers and titles match the rendered TOC.
  - The snapshot route (commonmark_x reader) yields the same `text` for the same fences.

- Docs: `docs/highlights-create.md` gains a short "Code catalog" subsection (what is
  recorded, tab handling, the blockquote note).

## Phase store

Goal: every new Markdown-born reference in the library has a saved source, and its note
points to it.

- **Module** `src/native/highlights_ref/source_store.rs`:
  - Layout helpers: `store_dir`, `pointer_for`.
  - `code.json` v1 serde types.
  - `write_store`: write-once `source.md`, atomic `code.json`, caller holds
    `bob_sync.lock`.
  - `read_store(vault, pointer)`, which returns a status from the shared contract plus
    the verified bytes and catalog.
  - `find_birth_store(vault, ref_type, stem, pdf_sha256)`.
  - Reuse `io.rs`/`reference_mutation::atomic_write_bytes` for atomic writes.
- **Create wiring:**
  - Markdown route (`create_markdown_plan`, both the listen and plain branches), `-M`,
    and the snapshot route (`create_snapshot_plan`).
  - After `stamp_and_install`, compute the installed PDF's SHA-256, take `bob_sync.lock`
    (`ob::acquire_lock_waiting`), and write the store from the frozen bytes and the
    catalog. `route` and `origin` are set per the shared contract. For snapshots, also
    record `source_note` and the note's digest.
  - Then call `finish_create_intake` as today.
  - A reread is an ordinary create under its `__vN` stem.
  - Output lines: `source_markdown: .bob/ref-sources/…/source.md` and
    `code: 6 blocks (yaml, rust, bash)` (or `code: none`, or
    `code: not recorded (<reason>)`).
  - Dry run prints the planned `source_markdown:` path (computable from the bytes) and
    still `writes: none`.
  - `-o` outside the library (`TargetWorkflow::External`) writes no store and prints
    `source_markdown: none (output outside the library)`.
  - A store write failure prints a `warning:` and never fails the create: the reference
    still files.
- **Pointer:** implement the `pointer_home` branch chosen above.
  - For `note`: add birth-only plan state next to birth lineage in the shared single-PDF
    sync plan, and render `source_markdown` into the new note's frontmatter. Repeat
    syncs and `bob ref scan` preserve it verbatim. Inline intake and a queued `xlib/`
    drain (`bob ref scan -i`) both set it.
  - For `marker`: carry the field through the marker per the callout.
- **Ingest commit:**
  - In `IntakeSession::ingest` (`intake.rs`), add the store's two files to the scoped
    commit path set after the guard. Do not add them to `scoped_write_path_set`: the
    guard refuses untracked paths.
  - When the pre-sync already committed them, adding them is a no-op. Ignored paths are
    already filtered with a warning.
- **Trackability:**
  - Move gkeep's quiet `git check-ignore` preflight (`gkeep/imports.rs`
    `preflight_trackable`) into a shared vault-git helper.
  - After writing a store, if `code.json` is ignored, warn:
    `warning: code catalogs are ignored by the vault .gitignore; code copy won't reach other hosts — add "!/.bob/ref-sources/**/*.json" (docs/vault-git-sync.md#reference-sources)`.
- **Vault (athena, live):**
  - Add `!/.bob/ref-sources/**/*.json` to `~/bob/.gitignore` beside the other `.bob`
    allowances, with a one-line comment ("Code catalogs for saved reference sources").
  - Verify with
    `git -C ~/bob check-ignore -v .bob/ref-sources/chat/probe--000000000000/code.json`
    (a negated last match).
  - Commit `chore(vault): allow reference source catalogs` and run `bob vault-sync`.
  - The phase `walkers` exclusion must already be on master, because hosts track master.
- **RefRow:**
  - `src/native/ref_library/row.rs` gains `source_markdown: Option<String>` right after
    `research_ref`, parsed from frontmatter. It appears in `ref list`, `ref find`, and
    `ref show` JSON (additive; `schema_version` stays 1).
  - Update `docs/reference/json-outputs.md` and the JSON goldens.
- **Tests** (`tests/cli/highlights/`):
  - Use a fake pandoc (the `overwrite_local.rs` `write_fake_pandoc` pattern) that also
    writes a canned catalog report to the `bob-code-report` metadata path, plus one real
    pandoc/XeLaTeX end-to-end test (skipped when missing).
  - Inline ingest: the store and pointer exist, `code.json` holds the Makefile tabs byte
    for byte, and the scoped ingest commit (or pre-sync) contains both files.
  - `-k` queue, then `bob ref scan -i` drain: the pointer is set.
  - Snapshot and `-M` routes, and a reread: each gets a new store.
  - `bob ref rename` keeps the pointer and does not move the store.
  - Dry run writes nothing. `-o` outside the library writes no store.
  - An ignored `code.json` warns. A store write failure warns and the create still
    succeeds.
  - A `source.md` digest collision refuses reuse.
- **Docs:**
  - `docs/highlights-create.md`: a "Saved Markdown" section covering layout, which bytes
    are kept, the write-once and lifecycle table, and that hidden content such as HTML
    comments and frontmatter is preserved and synced.
  - `docs/highlights-ref-sync.md`: the field and its birth derivation (or marker
    semantics).
  - `docs/vault-git-sync.md`: a "Reference sources" section with the allowlist line and
    the check-ignore command.
  - `docs/reference/vault-format.md`: the `.bob/ref-sources/` tree line.

## Phase code-cmd

Goal: the read surface from the shared contract, golden-tested.

- New `src/native/ref_library/code.rs`, with a clap builder beside `show_command` in
  `ref_library/cli.rs` and dispatch in `highlights_ref/mod.rs`.
  - Add `code` to the `Library` help group in `highlights_ref/cli.rs`. It sorts first;
    the existing tests enforce the order.
  - Add an `Entry { path: &["ref","code"], arg: "ref", kind: Kind::VaultNote }` to
    `src/native/completion/kinds.rs`.
  - Every long option gets its short alias, and options are alphabetical
    (`cli_rules.md`).
  - Write excellent `--help` with examples:
    - `bob ref code ace_refresh_loop`
    - `bob ref code ace_refresh_loop 2 | pbcopy`
    - `bob ref code ace_refresh_loop -p 7`
    - `bob ref code ace_refresh_loop -s > report.md`
    - `bob ref code ace_refresh_loop -f json`
- Reuse `resolve_show` and its error envelope, and `source_store::read_store`. Compute
  `summary`, `caption`, `lines`, `bytes`, and `fenced` here, never in the app.
- Human output uses `Styler`: the title, then the path · block count · source size ·
  captured date line, then dim section headers. Block numbers are bold, the language is
  in a colored chip, pages are dim, and a `●`-free plain layout applies when not a TTY.
  It ends with the hint line.
- `bob ref show`'s human output gains one metadata row when a pointer exists:
  `markdown  .bob/ref-sources/…/source.md · bob ref code <ref>`. The JSON already
  carries `source_markdown` from phase `store`.
- Tests (`tests/cli/ref_library/code.rs`) against hand-built fixture stores:
  - a JSON golden;
  - raw output byte for byte (tabs, trailing spaces, no trailing newline);
  - a fenced form whose text contains triple backticks;
  - `-p` filtering and `-s` bytes;
  - each of the six statuses;
  - an unknown and an ambiguous reference;
  - a missing `N`;
  - human output without color;
  - Mermaid summary and caption from its frontmatter title.
- Docs:
  - `docs/ref.md`: a "`bob ref code`" section in the shared contract's terms, linked
    from "Choose a command".
  - `docs/reference/json-outputs.md`: the `ref code` envelope.
  - Run `just docs-gen` for `docs/reference/cli/ref.md`, then `just docs-check`.

## Phase lifecycle

- **Overwrite** (`-f overwrite` on a filed Markdown file or a same-day snapshot:
  `overwrite_markdown` and `overwrite_vault_note` in `create.rs`, and
  `overwrite.rs`/`overwrite/execute.rs`):
  - Stage the new `source.md` and `code.json` in scratch before the transaction.
  - Add the store files to `touched` as created paths in `prepare`, so
    `apply_and_verify` expects them, rollback (`recovery::restore_from_base`) removes a
    newly created store directory, and the commit manifest includes them.
  - Install them during apply; the mutation session already holds `bob_sync.lock`.
  - Set the pointer: with `pointer_home = note`, set or replace `source_markdown` in
    `fresh_note_contents` (`retain_note_local_frontmatter` would otherwise keep the old
    value). With `marker`, it flows through the staged marker.
  - Unchanged bytes reuse the existing directory, and `code.json` is refreshed only if
    it differs. The old directory stays as history.
  - An overwrite whose new input is not Markdown-rendered clears the pointer.
  - If this host still ignores `code.json`, the overwrite must not fail. Commit
    `source.md`, keep `code.json` on disk, warn, and keep `prove_commit` sound. Test
    this case.
- **Doctor:**
  - In `append_library_doctor_rows` (`doctor.rs`), add `append_ref_sources_row`:
    - `ref sources: ok (412 saved · 1,903 code blocks)`, or
    - `warn (2 dangling · 1 digest mismatch · 3 orphans · e.g. a, b, c — …)`.
  - Report:
    - dangling pointers (store absent);
    - digest mismatches;
    - missing catalogs or `catalog_error`;
    - orphan store directories that no note points to;
    - an ignored `code.json` (with the allowlist line).
  - Hints name `bob ref attach-source` for references without a source and
    `docs/vault-git-sync.md#reference-sources` for the allowlist.
  - Doctor stays read-only.
- **Tests:**
  - An overwrite repoints. Unchanged bytes reuse the directory.
  - A failpoint rollback removes the new store and restores the old pointer.
  - A non-Markdown overwrite clears the pointer.
  - Each doctor case.
- **Docs:** `docs/ref.md` (doctor row) and the Recapture lifecycle table in
  `docs/highlights-create.md`.

## Phase attach

> [!decision] backfill = defer Do not build the command. File one follow-up task bead
> with `/sase_new_task` (type `feature`, size `large`) that cites this section, then
> close the phase.

> [!decision] backfill = build Build and test the command, run only the dry run on
> athena, and record its accept/skip summary in the phase bead with the exact write
> command for Bryan.

> [!decision] backfill = run Build and test the command, run the dry run on athena,
> review it against the gates below, then run the write pass.

`bob ref attach-source [REF]... -s, --search PATH... [-d, --dry-run]` (Library help
group; help and completion as in phase `code-cmd`):

- **Eligible references:**
  - PDF-backed, with no `source_markdown`, and no `source_note`. Snapshots are never
    attached from a live note, because the note has changed since.
  - `REF` arguments restrict the set; the default is every eligible reference.
- **Candidates:** a file `<stem>.md` under each `PATH`, searched recursively and
  skipping dot-directories. A `PATH` may also be a single file, which makes it the
  candidate for one named `REF`. Two candidates for one stem means a skip (`ambiguous`).
- **Verification**, with no XeLaTeX:
  1. Run pandoc with the record pass to native output, using the same reader flags as
     the Markdown route, to get the candidate's blocks and title (frontmatter, then H1).
  2. Extract the library PDF's per-page text with `pdftotext -layout`. If it is missing,
     exit 1 with an install hint.
  3. Normalize both sides: drop whitespace, fvextra wrap glyphs, return-link tag glyphs,
     and `↩ p. N` pills.
  4. Accept when the title matches the note's and at least 80% of matchable blocks (at
     least 12 normalized characters, not Mermaid) are found in order at 85% or higher
     similarity. A candidate with no matchable blocks must instead match its five
     longest paragraphs the same way.
  5. Pages come from the match position: `page_source: "text_match"`.
- **Write:**
  - Store with `route: "attached"` and `pdf.sha256` = the current library PDF digest.
    Add `verified: {matched, total}` to `code.json`.
  - Set the pointer: note frontmatter only with `pointer_home = note`; with `marker`,
    frontmatter, and the next coordinated scan rewrites the PDF marker. Say so in the
    output.
  - Run under a reference-mutation session (`bob_sync.lock` then `scan.lock`), in
    batches of up to 100 references. Each batch is one scoped commit
    (`bob ref attach-source: N references`), revertable with `git revert`.
- **Output:**
  - One line per reference: `attached`, or `skipped (<reason>)` where the reason is
    `no candidate`, `ambiguous`, `title mismatch`, `low match 3/8 blocks`,
    `already attached`, or `snapshot`.
  - Then a summary, then `writes: none` on a dry run.
- **Tests:**
  - A fixture library PDF rendered from a fixture report (real pandoc, skipped when
    missing) attaches.
  - An edited candidate below the threshold is skipped.
  - Ambiguous and title-mismatch candidates are skipped.
  - A dry run writes nothing.
  - A batch commit's contents are correct.
- **Run on athena (`run`/`build`):**
  - Find every SASE project's research sidecar (`sase repo list --all`) and open each
    one with `sase repo open` before reading it.
  - Dry-run with every sidecar as `-s`.
  - Gate: the research expects about 429 of 473 agent reports to have a candidate. Stop
    and report instead of writing if fewer than 60% of candidates verify, or if any skip
    reason looks systematic.
  - Write. Then spot-check 10 attached references with `bob ref code <ref> N` against
    the PDF.
  - Record counts and commit SHAs in the phase bead.

## Phase mac-key

Repository: `bob-mac-capture`. Read its `AGENTS.md` first. Never land Swift you have not
compiled: run `just portable-check` on athena, compile a scratch copy on the Mac (rsync
without `.build`/`.git`, `bundle.sh`, `just typecheck-tests`, delete the copy), and if
the Mac is offline, say the macOS targets are unverified.

- **Key:**
  - `HotKeyAction.refsHighlightsCopy = 5` (`HotKeyRegistry.swift`) and
    `HotKeyConfiguration.highlightsCommandY` (key code 16, cmdKey).
  - A new `RefsHighlightsCopyKey` (`cmdY` default, `off`) modelled on
    `RefsHighlightsTabsKey`, persisted in `AppSettings`, with a Settings › References
    picker.
  - `HighlightsTakeover` syncs it: registered only while Highlights or the override app
    is frontmost and `isRefsVisible()` is false, and re-synced on visibility change,
    like ⌘T's `yieldsToVisiblePanel`.
  - `AppDelegate`'s hotkey switch routes it to the new
    `BobPanelCoordinator.openRefsCopyFromHighlights()`.
- **Session:** `RefsPresentationSource.highlightsCopy` (`RefsPanelModel`) shares
  `.highlightsTabs`' stash and reset path in `prepareForPresentation` and arms
  `pendingHighlightsCopy` (token, deadline, buffer).
  - When the Tabs refresh completes, resolve per the UX section:
    - Success calls `openCopyCard()` on the focused matched row, then replays the buffer
      through `handleCopyCardKey`.
    - Failure states set the skeleton message and drop the buffer.
  - The 2 s deadline is a cancellable task.
- **Key routing:** in `RefsPanel.processKeyEvent`, before the copy-card branch, while
  `pendingHighlightsCopy` is armed:
  - Esc and `⌃[` cancel it.
  - Ctrl+Tab keeps working.
  - Every other keyDown is appended to the buffer and consumed. `flagsChanged` updates
    the alternates state as it does for the card.
- **Card queue** (`RefsCore/RefsCopy.swift`, `RefsCopyCard`, pure and unit-tested):
  - While `pending != nil`, every command except `.close`/`.showActions`/`.consume` is
    appended to `queuedCommands` (cap 16) instead of executing.
  - When the pending key resolves (`notesLoaded`, `notesFailed`, `pendingTimedOut`, and
    later the code equivalents from phase `mac-core`), queued commands replay in order.
    Replay continues through effects that keep the card open (opening a picker, moving)
    and stops at the first effect that writes or closes; the rest are dropped.
  - `.close` clears pending and the queue.
  - This fixes `H ↵` typed before notes load, which today copies the default row.
- **Skeleton view:** a minimal state of `RefsCopyCardView`, or a sibling view in the
  inspector column, with the `COPY` caption, a spinner line, and a message line, using
  the card's tokens so it morphs into the card without a jump.
- **Tests:**
  - Takeover registration and step-aside (`HotKeyRegistryTests`-style fakes).
  - Presentation stash and restore.
  - Buffer capture and replay order, cancel, timeout, and each failure state
    (`RefsPanelModelTests`).
  - Card queue semantics (`RefsCopyTests`): `H ↵` while notes load copies the first
    highlight; `1 ↵` copies note 1; a write stops replay.
- **Docs:**
  - `docs/refs.md`, "Opening Bob Refs": the ⌘Y bullet, plus a Copy card line noting ⌘Y
    from Highlights.
  - `docs/keyboard.md`: the ⌘Y row.
  - `docs/settings.md`: the new picker.
  - `docs/permissions.md`: ⌘Y uses the same Accessibility read as ⌘T.
  - Then `just docs-check`.
- **Done when:**
  - On the Mac, ⌘Y in Highlights opens the card on the focused PDF.
  - `⌘Y ↵`, `⌘Y W`, and `⌘Y ⌘Y` typed within 150 ms of ⌘Y copy the right value, and
    Highlights stays frontmost.
  - ⌘Y with Refs visible opens the card for the selection.
  - Turning the setting Off frees the key.
  - Confirm that the Carbon registration beats Highlights' own key handling (the
    research infers it from ⌘O). If it does not, stop and report.

## Phase mac-core

Repository: `bob-mac-capture`, `Sources/RefsCore` plus `Tests/RefsCoreTests` (verifiable
with `just portable-check` on athena).

- **Decoding:**
  - `RefItem.sourceMarkdown` (`source_markdown`, `decodeIfPresent`, encoded for the
    snapshot cache) in `RefsDecoding.swift`.
  - `RefsCodeResponse` → `RefsCodeCatalog` (status, source, blocks) for the shared JSON
    contract. Reject any `schema_version` other than 1. Decode everything else loosely.
  - Use real output from phase `code-cmd`'s goldens as fixtures.
- **Fetcher:** `BobRefsFetcher.code(path:)` runs `["ref","code",path,"-f","json"]`. Map
  clap's unknown-subcommand failure (exit 2) to `.unsupported`.
- **Card model:**
  - `RefsCopyKey.codePicker` (`C`) and `.markdownSource` (`M`), parsed from `c`/`m`.
  - `RefsCopyGroup.contents`.
  - `RefsCopyCodeState` (`loading`, `loaded`, `unavailable(reason)`, `unsupported`).
  - `RefsCopyCatalog` builds the C and M rows (previews, ⌥M alternate, HUD titles) and
    the hidden reasons from the UX table.
  - `RefsCopyCard` gains `codeState`, frozen `currentPage`, a `codePicker` layer,
    `codeLoaded(path:catalog:)`/`codeFailed(path:)`, and the pending-queue resolution
    from phase `mac-key` for C and M.
  - Code loading never blocks other rows.
- **Picker model:**
  - Extract the sectioned-window algorithm (`fitWindow`, `revealSelection`, group-header
    budgeting, and `windowRows`) from `RefsHighlightPicker` into a generic
    `RefsSectionedWindow<Entry>`. Keep every existing H test green.
  - Build `RefsCodePicker` on it:
    - entries with section groups;
    - page-aware initial selection;
    - digit, arrow, and ⌥ handling;
    - `copyRow(alternate:)` with plain and fenced payloads and the HUD strings;
    - VoiceOver labels.
  - `RefsCodePreview` (pure) is what the well draws: visible lines under the display
    caps, tab-to-`→` display runs at 4-column stops, an overflow count, and clipped-line
    flags. The C-row preview string (`N blocks · ● K on this page · langs`) is also
    pure.
- **Tests:**
  - Decoding (all statuses, an older payload without `source_markdown`).
  - Row visibility and messages for each status.
  - Initial selection cases (on page, between blocks, after the last block, no page).
  - Window plus headers.
  - Plain and fenced payloads byte for byte.
  - Preview tab runs and overflow.
  - `C ↵` queued while loading opens the picker and copies the preselected block.

## Phase mac-ui

Repository: `bob-mac-capture`, the app target. Compile per its `AGENTS.md`.

- **Panel model:**
  - `openCopyCard` starts `bob ref code` in parallel with the notes request whenever
    `item.sourceMarkdown != nil`. The task is token-guarded and cancelled in
    `closeCopyCard`.
  - Freeze `currentPage` from the Tabs listing (selected row is the current tab with a
    page).
  - Apply code effects through the existing `applyCopyCardEffect` (pasteboard check,
    hide, HUD).
- **Views:**
  - In `RefsCopyCardView`: the `.contents` group rows; the header page chip; the
    `CODE BLOCKS N` header; code picker rows; `↵ opens …` hints; and the ⌥ fenced
    preview.
  - A new `RefsCodePreviewWell` drawing `RefsCodePreview`.
  - Well and chip metrics go in `RefsVisualTokens`.
  - Light and dark, Increase Contrast, and Reduce Transparency are respected throughout.
    Follow the visual spec above exactly; it is the design.
- **HUD:** the code and Markdown titles from the UX section, through the existing
  `RefsCopyHUDView`.
- **Design fixtures:** extend `RefsCopyCardDesignTests` with light and dark versions of:
  - a card with code rows and an on-page count;
  - the code picker on page 7;
  - the picker with ⌥ fenced;
  - a Mermaid selection;
  - a long-line and tabs block;
  - a loading code row;
  - a `no_code` card.

  Review the CI render artifacts and iterate until they look right; spacing, alignment,
  and the well's proportions matter. Importing them into `docs/images` is optional.

- **Docs:**
  - `docs/refs.md` "Copy card": the C and M rows, the picker, the well, the messages,
    and ⌘Y C ↵.
  - `docs/keyboard.md`: C, M, the picker keys, and `⌥↵`.
  - Then `just docs-check`.
- **End-to-end acceptance on the Mac:**
  1. On athena, `bob ref create` a fixture report with a Makefile block (tabs), a YAML
     block on a later page, a Mermaid block, and a long line.
  2. Let the vault sync to the Mac and open the PDF in Highlights at the YAML block's
     page.
  3. Type `⌘Y C ↵` fast and paste into a file. `cmp` it against `bob ref code <ref> <N>`
     from athena.
  4. Repeat for the Makefile block (`⌘Y C`, digit) and with `⌥↵`.
  5. Check that M copies `source.md` exactly and that Highlights stays frontmost
     throughout.
  6. Report any mismatch as a failure, not a caveat.

## Phase records

> [!decision] record_decisions Use `/sase_memory_write` (this decision authorizes it) to
> add two strands to the `decisions` web, then run `sase memory init`. Each strand
> states where it applies, the claim, why it beat the alternatives, the cost, what
> reopens it, and its evidence (the research report and the landed commits of phases
> `catalog`, `store`, `lifecycle`, `code-cmd`, and `mac-ui`):
>
> 1. **Reference sources are preserved write-once in `.bob/ref-sources`.** It covers:
>    - the content-addressed layout;
>    - which bytes are kept;
>    - the pointer home actually chosen (`pointer_home`) and why;
>    - the `.bob` walker exclusion;
>    - the lifecycle.
>
>    Rejected alternatives: `lib/<stem>.md` (the Highlights sidecar slot), files under
>    `lib/`, and PDF embedded files.
>
> 2. **Code copy reads bob's render-time catalog, never the PDF text layer.** It covers:
>    - the catalog filter, anchors, and tab fidelity;
>    - the Mac presents and never extracts;
>    - the clipboard contract.
>
>    Rejected alternatives: PDF text extraction, and parsing the Markdown at read time
>    with a second parser.

## Verification and release

- **bob-cli checks:** every bob-cli phase runs the repository's full checks (`just`
  targets, including clippy and tests) and `just docs-check` when docs change. Phases
  never touch the Cargo.toml version or `CHANGELOG.md`; releases ride the daily train
  (`decisions:bob-cli-releases-ride-a-daily-train`). The changes are additive, so
  commits use `feat(ref): …`/`fix(ref): …` without `!`.
- **bob-mac-capture checks:** per its `AGENTS.md` (portable-check on athena, scratch
  compile on the Mac, CI for XCTest). Every phase states in its bead whether the macOS
  targets were verified.
- **Compatibility:** an older `bob` with the new app hides C and M ("needs a newer
  bob"). A new `bob` with an older app ignores the additive fields.

## Risks

- **A future Highlights binds ⌘Y.** The setting goes Off, and the research's NIB scan
  can be re-run after upgrades.
- **Anchor page drift.** The `\@nobreaktrue` start anchor plus fixtures with page-top
  and page-spanning blocks guard it. Ranges keep "on this page" tolerant of a one-page
  error.
- **A host without the allowlist line.** `code.json` stays local there. Create warns,
  doctor reports, and the Mac shows "Saved Markdown isn't on this Mac yet". The line
  lands on athena in phase `store` and reaches other hosts by sync.
- **Hidden source content.** HTML comments, frontmatter, and the like are kept in
  `source.md` and synced. That is acceptable in the private vault and documented.
- **Attached sources may differ slightly from the PDF.** Verification requires 85% or
  better per block; `code.json` records `route: attached` and the match counts.
