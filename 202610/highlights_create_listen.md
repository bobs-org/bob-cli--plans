---
tier: epic
title: bob highlights create --listen and every sase-listen target
goal: '`bob highlights create <TARGET>` accepts every document target that `sase-listen
  render -e full` accepts (Markdown files, local PDFs, PDF URLs, arXiv paper URLs,
  and web article URLs) and installs one marker-stamped PDF into the Highlights intake.
  A PDF is stamped as-is, never re-rendered. `--listen` runs the configured `highlights.listen_command`
  (Bryan''s chezmoi config sets `sase-listen render {target} -e full -o {audio}`)
  with its output streamed unchanged, and binds the published episode as the PDF''s
  companion audio. `bob highlights scan` then writes a ref note with an audio player.
  When the target is already captured, `--listen` attaches the new episode to the
  existing ref note instead. Every article or paper Bryan listens to ends up tracked
  in his Obsidian ref system. Nothing is ever written to the vault unless every step
  succeeded.

  '
phases:
- id: listen-command
  title: Configurable listen command contract and runner
  depends_on: []
  size: small
  description: 'listen-command: add highlights.listen_command plus its env override,
    a template parser with shell-quoted {target}/{pdf}/{audio}/{title} placeholders,
    a runner that streams output unchanged and verifies the MP3 written to {audio},
    and a doctor row.'
- id: fetch-arxiv
  title: URL fetcher, arXiv identity and metadata, and shared dedupe
  depends_on: []
  size: small
  description: 'fetch-arxiv: add a curl-based fetcher that validates every redirect
    hop, arXiv URL parsing that mirrors sase-listen, arXiv API metadata, the short-title
    stem rule, and a shared dedupe module with arXiv and legacy url keys.'
- id: pdf-targets
  title: create accepts local PDFs, PDF URLs, and arXiv papers
  depends_on:
  - fetch-arxiv
  size: medium
  description: 'pdf-targets: add TARGET classification to create, a stamp-as-is PDF
    route with title, stem, and validation rules, private scratch staging, the new
    -N/-T flags and per-kind ref-type defaults, explicit --audio on every route, CLI
    tests with a fake curl, and docs/highlights-create.md.'
- id: article-targets
  title: create routes web article URLs through the clip engine
  depends_on:
  - pdf-targets
  size: small
  description: 'article-targets: refactor clip into a callable engine and route create''s
    HTML and bot-walled URL targets through it with create''s options, explicit companion
    audio support, and tests.'
- id: create-listen
  title: Wire --listen into create and clip, with attach mode
  depends_on:
  - listen-command
  - article-targets
  size: medium
  description: 'create-listen: add -L/--listen to create and clip on every route with
    all-or-nothing ordering, attach mode for already-captured targets, dry-run and
    reports, fake-listen CLI tests, docs, and the chezmoi listen_command line.'
- id: live-verify
  title: Live end-to-end verification on athena
  depends_on:
  - create-listen
  size: small
  description: 'live-verify: build bob and exercise every target kind into a scratch
    vault, run one real unpublished sase-listen render under a TTY, scan the scratch
    vault to prove the ref note gets the player, fix what breaks, and record follow-ups.'
proposed_by: bbugyi200.athena.research.3r.linker.w0
create_time: 2026-10-06 15:45:25
status: wip
bead_id: bob-cli-4s
---

- **PROMPT:** [prompts/202610/highlights_create_listen.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/highlights_create_listen.md)
- **BEAD:** [bob-cli-4s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4s/README.md)

# Plan: `bob highlights create --listen` and every sase-listen target

## Background and evidence

### What bob already does

- **`bob highlights create <MD_FILE>`** (`src/native/highlights_ref/create.rs`) renders
  Markdown with pandoc/XeLaTeX. It plans `xlib/<ref-type>/<stem>.pdf` (default ref type
  `chat`), refuses sidecar, library, and existing-target collisions
  (`stamp.rs::refuse_target_collisions`), stamps a page-1 `/Text` marker, and installs
  it with `stamp_and_install`.
- **Companion audio already exists end to end** (`audio.rs`, commits `f7c268d` and
  `2152202`):
  - `create` binds audio from `--audio PATH`, from frontmatter `audio.episode_id`, or by
    hashing a `<stem>_narration.md` script against the sase-listen library manifests.
  - It copies that audio beside the target as `<stem>.<ext>` before the PDF is
    installed, and deletes the copy if the install fails.
  - `scan` moves same-stem `mp3/m4a/ogg/opus` files from `xlib/` to `lib/` together with
    their PDF. It writes `audio: "[[lib/…mp3]]"` and puts the Obsidian player under the
    `^ref` task.
  - Scan also **late-pairs** orphan `xlib/<rel>.<ext>` audio onto an existing
    `lib/<rel>.pdf`. `docs/highlights-ref-sync.md` already calls this the way to
    backfill audio onto a PDF that has been scanned.
- **`bob highlights clip <URL>`** (`clip.rs`, `clip_url.rs`, `clip_adapter.rs`, plan
  `202610/web_url_highlights_clip.md`) captures web articles:
  - It runs the pinned uv/Playwright adapter, which retries headed on a bot challenge
    and downloads PDFs directly, but only on hosts that have a browser.
  - It stamps `source_url`, `author`, `published`, `captured`, and `id`.
  - It dedupes against `source_url` in ref notes and in queued intake markers. Its
    default ref type is `blogs`.
- **The xlib bridge:**
  - On the Mac, `scan` runs from cron.
  - Its pre-scan hook `bob_xlib_pull` (chezmoi `home/bin/executable_bob_xlib_pull`)
    rsyncs athena's and apollo's `~/bob/xlib/` with `--remove-source-files` and **no
    filters**.
  - `doctor.rs::collect_pdf_paths_from_dir` does **not** skip hidden files either.
  - So any temporary file left in `xlib/` can be scanned or pulled away while bob is
    still using it. Today `create` renders `.<stem>.<pid>.render.pdf` beside the target,
    but only for seconds. A listen run lasts minutes.
- **The vault:**
  - `ref/` has `ai/ blogs/ chat/ docs/ papers/`.
  - arXiv papers live in `lib/papers/` under short, hand-picked stems (`ea_graph`,
    `memory_os`, `human_mem_arch`).
  - Their notes carry an ad hoc `url: "https://arxiv.org/pdf/2608.04278"` key instead of
    `source_url` (for example `ref/papers/ea_graph.md`, titled "EA-Graph:
    Artifact-Anchored Verification Memory for Coding Agents under Upstream Drift").
- **Config and dependencies:**
  - `bob/config.yml` lives in the linked `chezmoi` repo at
    `home/dot_config/bob/config.yml`. Its `highlights:` block has only `pre_scan_hook`.
  - `src/native/config/mod.rs` parses
    `highlights.{pre_scan_hook, audio_link_template, audio_library}`.
  - Unknown keys are ignored, because serde's default is used and `deny_unknown_fields`
    is not set.
  - bob has no HTTP client crate. It already shells out to `pandoc`, `uv`, and `sh`.

### What `sase-listen render` does

Source: an audited read of `gh:sase-org/sase-listen` at `f8154ad`.

- **Targets with `-e full`:** only http(s) URLs and existing local PDFs are accepted.
  Markdown files, narration scripts, `kind:path` refs, `arxiv:ID`, and bare IDs exit 2
  with "Generated brief and full editions are available for article URLs and PDF files."
- **arXiv** (`web/arxiv.py`): hosts `arxiv.org`, `www.arxiv.org`, and
  `export.arxiv.org`.
  - Paths match `^/(abs|html|pdf)/(<id>)(\.pdf)?/?$`, where `.pdf` is allowed only on
    `/pdf/`.
  - New-style ids match `\d{4}\.\d{4,5}(v\d+)?`. Old-style ids match
    `[a-z]+(-[a-z]+)*(\.[A-Za-z]{2})?/\d{7}(v\d+)?`.
  - Query and fragment are ignored, and the version is kept.
  - The fetch always goes to `https://arxiv.org/pdf/<id>`. It never falls back to the
    abstract page.
- **PDF detection for URLs:** `application/pdf` or `application/x-pdf` must have `%PDF-`
  in the first 1024 bytes. For `application/octet-stream`, `binary/octet-stream`, or a
  missing content type, the body is sniffed for that signature.
- **`-o, --output PATH`:** "Copy the MP3 to PATH". This runs after the library commit
  and before publish. It creates parent directories and writes atomically through
  `.tmp-<name>`.
- **Output:**
  - A Rich live checklist goes to stderr on a TTY, and plain lines otherwise.
  - A summary goes to stdout: `♪ Ready in …`, then the audio path, then "Published to
    apollo — refresh the feed in AntennaPod to download it.", then "Copied to …".
  - `--json` disables all progress. There is no mode that shows progress and also prints
    machine output.
- **Publishing:** Bryan's chezmoi `home/dot_config/sase-listen/config.yml` sets
  `feed.auto_publish: true` and `feed.host: apollo`.
  - Full editions of URLs and PDFs are kind `article`, so they auto-publish.
  - A remote publish failure is queued with a warning and the exit code is still 0.
- **Exit codes:** 0 ok, 2 usage, 3 config, 4 synthesis, 5 quality gate, 6 lint, 130
  interrupted. There are no interactive prompts. The exception is `api_key_command`
  (`pass show …`), which inherits stdin.
- **Caches:** the source, writer, and chunk caches make a rerun after a failure cheap. A
  rerun always re-masters and re-publishes; there is no skip-if-exists.

### Research recommendations adopted

These come from
`research:202610/web_url_reference_pdf_capture/web_url_reference_pdf_capture.md`, which
Bryan agrees with:

- **A1:** the command writes the intake PDF, and `scan` writes the ref note.
- **A4:** provenance is mandatory: `source_url`, `author`, `published`, `captured`.
- **A5:** URL captures always stamp `id` and get their own defaults.
- **A6:** library PDFs are immutable, and captures are deduped by URL.
- **A7:** a URL that already points at a PDF is downloaded and stamped, never
  re-rendered.
- **A8:** no third-party renderer.
- **"Never silently wrong":** every failure exits nonzero with a `hint:`.
- **Keeping `clip`:** the research preferred a separate `clip` subcommand over
  `create <URL>`. That still holds:
  - `clip` remains the web-article specialist, with saved-page replay and
    author/published overrides.
  - `create` gains only `-L`, `-N`, and `-T`, and hands articles to the clip engine with
    defaults.
  - So `create`'s help stays compact while it accepts the same targets as
    `sase-listen render`.

## Design decisions

1. **`create` is the front door for every target.** It takes Markdown, a local PDF, a
   PDF URL, an arXiv paper URL, or a web article URL. "I want this in my ref system and
   in my ears" becomes one command: `bob highlights create <TARGET> -L`.
2. **A PDF is never re-rendered.**
   - bob copies it, stamps the page-1 marker, and sets the document Info Title/Author.
   - It checks that the marker round-trips, then installs the file into
     `xlib/<ref-type>/<stem>.pdf`.
   - Scan writes the ref note.
3. **arXiv parity with sase-listen, plus better metadata.** URL recognition and the
   PDF-only fetch mirror `web/arxiv.py` exactly. bob also reads the title, authors, and
   first-version date from the arXiv API. The ref-note title is the most visible field
   in the vault, and arXiv PDF Info titles are often empty.
4. **bob never names sase-listen in code.**
   - `--listen` runs the shell command in `highlights.listen_command`, with bob-quoted
     `{target}`, `{pdf}`, `{audio}`, and `{title}` placeholders. The contract is "write
     MP3 audio to `{audio}`".
   - Bryan's chezmoi config sets `sase-listen render {target} -e full -o {audio}`.
   - There is no built-in default. An unconfigured `--listen` is an error that shows the
     config snippet.
5. **The listen command's output is shown in full.**
   - It inherits bob's stdin, stdout, and stderr, and bob never captures or reformats
     it. sase-listen's live checklist and its "Published to apollo — refresh the feed in
     AntennaPod" summary appear unchanged.
   - Because bob cannot also parse `--json`, the `{audio}` handshake (sase-listen's
     `-o`) is how the episode audio comes back.
6. **All or nothing.** The order is:
   1. Run every preflight first: config, target, dedupe, and collisions.
   2. Produce the PDF in a private scratch directory.
   3. Run the listen command.
   4. Install the audio, then the PDF.

   If the listen command fails, the vault is untouched. Rerunning the same command is
   cheap because sase-listen resumes from its caches.

7. **Attach mode.**
   - When `--listen` is given and the target is already captured, create attaches the
     new episode: it drops the audio at `xlib/<rel>.mp3`, and scan late-pairs it onto
     the existing PDF and ref note.
   - Without attach mode, an already-captured paper could never gain a listen episode
     through bob, which defeats "always tracking what I listen to".
   - Identity is proven, never guessed. It comes from a source-URL dedupe hit or from
     the target path itself being a captured PDF.
8. **Scratch staging outside the vault.** Renders, downloads, stamped copies, and
   episode audio live in a 0700 scratch directory until the final atomic installs. This
   is because scan and `bob_xlib_pull` read every file in `xlib/`.
9. **Dedupe understands arXiv and legacy `url:`.**
   - Any spelling of an arXiv paper (abs, html, or pdf, with or without a version)
     dedupes to `https://arxiv.org/abs/<id>`.
   - Ref notes that carry the legacy `url:` key take part in dedupe. Without that, the
     existing paper refs would be duplicated.
10. **Fetching uses `curl`.** It runs as a subprocess, follows redirects manually, and
    passes every hop through `clip_url::validate_and_clean`. PDF and arXiv targets work
    on hosts with no browser (apollo). Only article pages need the clip adapter.
11. **Defaults are chosen per kind:**
    - Default ref types: `chat` for Markdown (unchanged), `papers` for any PDF, `blogs`
      for web articles (the clip default, unchanged).
    - arXiv stems use the paper's short name. "EA-Graph: …" becomes `ea_graph`, the same
      stem Bryan picked by hand.

## Contracts shared by several phases

### CLI surface

Options are sorted by short flag, matching `clip` and `tests/cli/help_options.rs`. Every
long option has a short alias (`sase/memory/cli_rules.md`).

```text
bob highlights create [OPTIONS] <TARGET>

Create a Highlights-ready PDF from Markdown, a PDF, or a URL

Arguments:
  <TARGET>  Markdown file, PDF file, PDF URL, arXiv paper URL, or web article URL

Options:
  -a, --audio <PATH>       Use this companion audio file instead of discovering one
  -b, --bob-dir <PATH>     Bob vault root; defaults to BOB_DIR or ~/bob
  -d, --dry-run            Preview work without modifying the vault or PDF
  -f, --force              Overwrite an existing target PDF
  -i, --include-id         Embed the output filename stem as the marker id (URL targets always embed it)
  -L, --listen             Narrate TARGET with highlights.listen_command and bind the episode as companion audio
  -l, --lib-dir <PATH>     Highlights PDF library; defaults to BOB_HIGHLIGHTS_LIB_DIR or lib
  -N, --name <STEM>        Output filename stem [default: derived from TARGET]
  -n, --no-audio           Skip companion audio discovery and copy
  -o, --output <PDF>       Complete path for the generated PDF, including the .pdf filename
  -P, --parent <NOTE>      Bare Obsidian note target for the marker parent [default: obsidian_ref]
  -r, --ref-dir <PATH>     Reference note output directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -s, --status <STATUS>    Lifecycle status embedded in the marker [default: ready] [possible values: …]
  -T, --title <TITLE>      Override the derived title
  -t, --ref-type <DIR>     Single library subdirectory [default: chat for Markdown, papers for PDFs, blogs for web articles]
  -x, --xlib-dir <PATH>    Highlights PDF intake directory; defaults to BOB_HIGHLIGHTS_XLIB_DIR or xlib
```

- Conflicts: `-L` conflicts with `-a` and `-n`. `-o` conflicts with `-t` and `-N`, as in
  `clip`.
- `--ref-type` loses its clap `default_value`. The default is computed per kind.
- `-N` is validated with `clip_url::validate_name`.
- With `-N`, `-i` embeds the name.
- The after-help is rewritten into scannable sections: **Targets:** (one line per kind
  with its route and default ref type), **Listen:**, **Audio:** (existing discovery
  prose, tightened), and **Examples:**.
- `bob highlights clip` gains `-L, --listen` with the same help line and semantics. This
  lands in `create-listen`.
- `about` becomes "Create a Highlights-ready PDF from Markdown, a PDF, or a URL". The
  positional's clap id becomes `target`, and its completion
  (`src/native/completion/kinds.rs`, currently `md-file` → `*.md`) offers `*.md` and
  `*.pdf` files.

### Target resolution

Resolution is syntactic first. It needs no network until step 3.

1. **URL detection.** If TARGET matches `^https?://` (case-insensitive), it is a URL.
   - It must pass `clip_url::validate_and_clean`: public http(s) only, no userinfo, no
     private hosts.
   - Anything else is a local path.
2. **Local paths.**
   - The path must exist and be a regular file. Otherwise the error is
     `TARGET does not exist or is not a file: <path> (URLs must start with http:// or https://)`.
   - `*.md` (case-insensitive) goes to the **Markdown route**, which keeps today's
     behavior.
   - `*.pdf` (case-insensitive), or any file whose first 1024 bytes contain `%PDF-`,
     goes to the **local PDF route**.
   - Anything else is an error:
     `TARGET must be a Markdown file (.md), a PDF file, or an http(s) URL: <path>`.
3. **URL routing.**
   - **Every URL** goes through **dedupe** first (see
     [Dedupe and attach mode](#dedupe-and-attach-mode)). Its key is syntactic, so no
     network is needed.
   - **arXiv paper URLs** (see [arXiv](#arxiv)) then go to the **arXiv route** without
     probing.
   - **Every other URL** is fetched into scratch:
     - PDF (by the sase-listen PDF detection rule above) → **PDF-URL route**.
     - A content type that claims PDF but bytes that lack `%PDF-` → error
       `server said PDF but sent something else`.
     - `text/html` or `application/xhtml+xml` with 2xx, or HTTP 403/429/503, whose body
       is usually a bot wall → **article route**, which is the clip engine. Its adapter
       does its own navigation, headed fallback, and direct-PDF capture.
     - Any other non-2xx → error `server returned HTTP <n> for <url>`.
     - Any other content type → error
       `unsupported content type <type>: create accepts Markdown files, PDFs, PDF URLs, arXiv paper URLs, and web article URLs`.

| Route     | Default `-t` | Title (first hit wins)                                       | Stem (first hit wins)                                                      | Marker extras                                                                            |
| --------- | ------------ | ------------------------------------------------------------ | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Markdown  | `chat`       | `-T`, frontmatter `title`, first H1, file stem (unchanged)   | `-N`, file stem                                                            | `id` only with `-i`                                                                      |
| local PDF | `papers`     | `-T`, plausible Info `/Title`, humanized stem                | `-N`, file stem if it passes `validate_name`, else `snake_case(file stem)` | `id` only with `-i`; plausible Info `/Author` as `author`                                |
| PDF URL   | `papers`     | `-T`, plausible Info `/Title`, humanized stem                | `-N`, `stem_from_url`, short-title stem, `<host>_<YYYYMMDD>`               | `id`, `source_url` (cleaned URL), `author` (plausible Info), `captured`                  |
| arXiv     | `papers`     | `-T`, arXiv API title, plausible Info `/Title`, `arXiv <id>` | `-N`, short-title stem, `arxiv_<id>` (`.` and `/` become `_`)              | `id`, `source_url` (`https://arxiv.org/abs/<id>[vN]`), `author`, `published`, `captured` |
| article   | `blogs`      | clip rules (`-T` maps to the clip title override)            | clip rules (`-N`)                                                          | clip's                                                                                   |

- **Short-title stem:** if the title has a `:` and the text before the first colon is
  1–4 whitespace-separated words, use that prefix. Otherwise use the first 6 words. Then
  apply `clip_url::snake_case`, which caps at 80 characters. Examples:
  - "EA-Graph: Artifact-Anchored …" → `ea_graph`.
  - "Attention Is All You Need" → `attention_is_all_you_need`.
- **Plausible Info title:** trim it and collapse whitespace. Reject it when any of these
  hold:
  - it is empty, shorter than 3 characters, or has no letters;
  - it equals the file or URL stem (case-insensitive);
  - it ends with `.pdf`, `.doc`, `.docx`, `.tex`, `.dvi`, `.ps`, or `.pages`;
  - it starts with `Microsoft Word - ` or `Microsoft PowerPoint - `;
  - it is `untitled` or `title`.

  The same normalization applies to `/Author`.

- **Humanized stem:** `clip_url::humanize_stem`.
- **arXiv authors:** 1 → `A`; 2 → `A and B`; 3 → `A, B, and C`; 4 or more → `A et al.`.
  `published` is the date part of the API `<published>` (the first version).

### PDF route (local, PDF URL, arXiv)

Before any write:

- The PDF must load with `lopdf`, have at least one page, not be encrypted, and be
  smaller than 95 MiB (the vault-sync refusal limit, `PDF_MAX_BYTES` in the clip
  adapter). Files of 50 MiB or more print a warning.
  - Encrypted PDFs get this message:
    `PDF is encrypted; Highlights markers cannot be embedded — save an unencrypted copy and pass that file`.
- If page 1 already has a standalone `/Text` note:
  - If it parses as a Highlights marker (`read_pdf_marker` + `parse_marker`), the PDF is
    "already captured". See attach mode.
  - Any other note is kept, and bob's marker is inserted ahead of it.
- `stamp.rs::embed_marker` **prepends** to page 1 `/Annots` for every caller, so bob's
  marker is always the first standalone note. Pandoc and Chromium PDFs carry only link
  annotations, so their behavior does not change.

Stamping:

1. Load the PDF, embed the marker, and set Info Title and Author (`PdfInfo`).
2. Save to `<scratch>/stamped.pdf`.
3. Verify: reload it, check that `read_pdf_marker` returns the composed marker, and
   check that the page count is unchanged.
4. Only then `atomic_copy` into the target. For a local target, the source file is never
   modified.

Factor the stamp-into-scratch and verify step into a `stamp.rs` helper next to
`stamp_and_install`.

**Local PDF identity:** a local PDF inside `lib_dir` or `xlib_dir` is an existing
capture if it carries a marker. If it carries none, the error is
`is inside the Highlights library/intake but has no marker; move it out and rerun create on it`.

### arXiv

- **`ArxivPaper { id, version: Option<String> }`** comes from URL parsing that mirrors
  `web/arxiv.py`. Mirror its positive and negative test tables verbatim. It provides:
  - `pdf_url()` → `https://arxiv.org/pdf/<id><version>`
  - `abs_url()` → `https://arxiv.org/abs/<id><version>`, which is the stored
    `source_url` and the human landing page
  - `dedupe_key()` → `https://arxiv.org/abs/<id>` (no version)
- **Metadata.**
  - Fetch `https://export.arxiv.org/api/query?id_list=<id><version>` with a 20 s cap,
    and parse the first `<entry>` with `regex` and an XML entity decoder.
  - Read `<title>` with whitespace collapsed, every `<author><name>`, and `<published>`.
  - An `<id>` containing `/api/errors`, a network failure, or a parse failure prints a
    `warning:` and falls back. It is never fatal.
- **PDF fetch failure is fatal and never falls back to the abstract page.** This matches
  sase-listen.
- **Out of scope:** `arxiv:ID` and bare ids. sase-listen rejects them with `-e full`.

### Fetching

The fetcher lives in `src/native/highlights_ref/fetch.rs`.

- **Program:** `curl`, or `BOB_HIGHLIGHTS_CURL` when that is set. The override is the
  test seam, like `BOB_PANDOC_COMMAND`.
- **Invocation:** `-sS --proto =http,https --connect-timeout 15 --max-time <cap>`, plus
  `--max-filesize 95M -o <dest> -w '<http_code>\n<content_type>\n<redirect_url>\n'`,
  with **no `-L`**. The user agent is honest:
  `bob-cli/<CARGO_PKG_VERSION> (+https://github.com/bobs-org/bob-cli)`.
- **Redirects:** bob follows up to 10 redirects itself. It resolves a relative
  `Location` against the current URL and passes every hop through `validate_and_clean`,
  so a redirect to a private host is refused.
- **Result:** `{ status, content_type, final_url, path, bytes }`. The size check is
  repeated after the download.
- **curl exit codes map to messages:**
  - 6 → cannot resolve host
  - 7 → connection failed
  - 28 → timed out
  - 35/60 → TLS failure
  - 63 → larger than 95 MiB
  - anything else → `curl failed (exit N): <stderr>`
- **Missing curl:** the error has the hint `install curl or set BOB_HIGHLIGHTS_CURL`.
- **Progress:** on a TTY, a single `fetching <host>…` line goes to stderr, like clip's
  `capturing <host>…`.

### Dedupe and attach mode

- **Shared module.** `collect_recorded_source_urls`, `check_dedupe`, and
  `RecordedSource` move from `clip.rs` to a shared
  `src/native/highlights_ref/sources.rs`.
- **Recorded sources** are:
  - ref-note frontmatter `source_url` **and legacy `url`**;
  - intake PDF marker `source_url` **and `url`**.

  Each record keeps its path, and for ref notes, also the vault-relative `source_pdf`
  and the presence of an `audio` field.

- **Dedupe key.** `clip_url::dedupe_key_for` maps any arXiv paper URL to
  `ArxivPaper::dedupe_key()`. Everything else keeps today's key. `clip` gains both
  improvements automatically.
- **Check order.** Dedupe runs before any fetch. For arXiv and generic URLs alike, the
  key is syntactic and uses the user's URL, not redirect targets.
- **Without `--listen`,** behavior is unchanged:
  - A ref-note hit refuses.
  - An intake hit refuses unless `--force` targets the same intake path. That is a
    recapture.
  - The hint line gains
    `add --listen to narrate it and attach the episode to the existing capture`.
  - A local PDF that is already captured refuses with
    `already captured; bob highlights sync <PDF> re-syncs it` plus the same `--listen`
    hint.
- **Attach mode** applies when `--listen` is set and the target would otherwise be
  refused as an existing capture. It covers three cases:
  1. A ref-note hit. The PDF is the note's `source_pdf`, which must resolve inside
     `lib_dir` and exist. Otherwise the error is
     `ref note has no resolvable source_pdf`.
  2. An intake hit that is not a `--force` recapture. The PDF is the queued intake PDF.
  3. A local PDF target inside `lib_dir` or `xlib_dir` that carries a marker.
- **Attach steps:**
  1. Let `<rel>` be the PDF path relative to `lib_dir` or `xlib_dir`.
  2. Refuse if the capture already has audio: any allowed companion beside
     `lib_dir/<rel>.pdf` or at `xlib_dir/<rel>.<ext>`, or an `audio` field on the ref
     note. The error is `already has companion audio: <path>`.
  3. Run the listen command with these placeholders:
     - `{target}` = the cleaned URL (cases 1 and 2) or the PDF path (case 3)
     - `{pdf}` = the existing PDF
     - `{title}` = the note title or marker title
     - `{audio}` = `<scratch>/<stem>.mp3`
  4. `atomic_copy` the audio to `xlib_dir/<rel>.mp3`. The PDF and the ref note are never
     touched.
- **What happens next:** scan moves an intake PDF together with its audio, or late-pairs
  the audio onto a library PDF. Either way the note gains the player.
- **Markdown, and local PDFs outside the vault,** whose planned library destination
  already exists keep refusing. Identity is not proven by stem alone. Their hint gains
  `to add audio to that capture, run bob highlights create <library PDF> --listen`.

### Listen command contract

- **Configuration.** The command comes from `BOB_HIGHLIGHTS_LISTEN_COMMAND`, and
  otherwise from `highlights.listen_command`.
  - The value is trimmed; blank means unset.
  - It is validated only when `--listen` runs and in `doctor`, never at config load, so
    a bad value cannot break `scan`.
- **Placeholders** are `{target}`, `{pdf}`, `{audio}`, and `{title}`. A placeholder
  matches `\{([a-z_]+)\}` and is not preceded by `$`, so `${HOME}` and `{a,b}` stay
  shell syntax. Validation fails when any of these hold:
  - `{audio}` is missing;
  - neither `{target}` nor `{pdf}` is present;
  - an unknown placeholder appears, such as `{targt}`;
  - a placeholder is wrapped in quotes (`'{x}'` or `"{x}"`). The hint is
    `bob shell-quotes placeholder values itself; remove the quotes`.
- **Unconfigured `--listen`** fails before any fetch or render. The error is
  `--listen needs highlights.listen_command in <config path> (or BOB_HIGHLIGHTS_LISTEN_COMMAND)`,
  with this hint:

  ```text
  hint: add `listen_command: sase-listen render {target} -e full -o {audio}` under `highlights:`
  ```

- **Values.**
  - `{target}` is the narration source. For URL routes it is the cleaned URL, so
    sase-listen does its own arXiv rewrite and records the URL. For a local PDF it is
    the absolute source path. For Markdown it is the staged scratch PDF, because
    sase-listen `-e full` rejects Markdown.
  - `{pdf}` is the unstamped PDF about to be installed, in scratch, or the existing
    capture in attach mode.
  - `{audio}` is `<scratch>/<stem>.mp3`.
  - `{title}` is the resolved title.
- **Quoting.** Every value is POSIX-shell-quoted: it is bare when every character is in
  `[A-Za-z0-9_./:@%+=,-]`, and `'…'` with `'\''` otherwise. Remote titles and URLs are
  never interpreted by the shell.
- **Execution.**
  - bob prints `listen: run <expanded command>` and flushes stdout. A dry run prints
    `listen: would-run <expanded command>` instead and never runs anything; this mirrors
    `pre_scan_hook`.
  - It runs `sh -c <expanded>` from the current directory with stdin, stdout, and stderr
    **inherited**.
  - While the child runs, bob ignores SIGINT and SIGQUIT, as `system(3)` does: the child
    is reset to default dispositions in `pre_exec`, and bob restores its own handlers
    after `wait`. Ctrl-C therefore reaches sase-listen's graceful stop, and bob can then
    clean up.
  - There is no timeout.
- **Result.**
  - A nonzero exit fails with `listen command failed with exit N` (or with the signal
    name).
  - Exit 130 or SIGINT means interrupted, and bob exits 130.
  - On exit 0, `{audio}` must be a regular, non-empty file that starts with `ID3` or an
    MPEG frame sync (`0xFF`, `b1 & 0xE0 == 0xE0`). Otherwise the error is
    `listen command exited 0 but wrote no MP3 audio to <path>`.

### Ordering and failure semantics (every route, with `--listen`)

1. Resolve and validate the listen command.
2. Resolve TARGET, run dedupe, decide on attach mode, and plan the target and
   collisions.
3. Plan the audio destination `<target>.mp3`:
   - Refuse if the mirrored library audio exists.
   - Refuse if the existing intake audio has different bytes, unless `--force` is given.
4. Produce the PDF in scratch: render, download, or adapter capture.
5. If this is a dry run, print the plan, including `listen: would-run …`, and stop.
6. Run the listen command.
7. Install:
   1. Re-run `refuse_target_collisions` and the audio planning, because the vault may
      have changed during a long listen.
   2. `atomic_copy` the audio.
   3. Install the stamped PDF.
   4. If the PDF install fails, delete only an audio file this run created.

If anything fails **after** the listen command produced audio:

- Keep the scratch directory and print `kept: <audio path>`.
- Print a hint:
  - normal mode: `bind it with bob highlights create <TARGET> --audio <path>`
  - attach mode: `copy it to <xlib dest> for scan to pair`

If the listen command itself fails:

- Write nothing.
- Hint:
  `nothing was written to the vault; rerun the same command once the listen error above is fixed`.

Scratch directory:

- Generalize `clip.rs`'s `ClipWorkdir` into a shared `workdir.rs::ScratchDir`. It is
  0700, under `TMPDIR` only when that path is at most 40 bytes and under `/tmp`
  otherwise, and it is removed on drop.
- It is kept when `BOB_HIGHLIGHTS_KEEP_WORKDIR=1` or the legacy
  `BOB_WEB_CLIP_KEEP_WORKDIR=1` is set, or after a post-listen failure.

### Output

Every success and dry-run report keeps the existing `key: value` lines and colored
prefixes (`Styler::success_prefix` and `warning_prefix`). New lines are only added, and
the Markdown route's existing lines and their order are kept.

Success (arXiv example):

```text
fetching arxiv.org…
listen: run sase-listen render https://arxiv.org/abs/1706.03762 -e full -o /tmp/bob-create-…/attention_is_all_you_need.mp3
  … sase-listen's live checklist and summary, unchanged …
ok created Highlights-ready PDF
source: https://arxiv.org/abs/1706.03762 (arXiv 1706.03762)
pdf: /home/bryan/bob/xlib/papers/attention_is_all_you_need.pdf
audio: /home/bryan/bob/xlib/papers/attention_is_all_you_need.mp3 (from --listen)
title: Attention Is All You Need
author: Ashish Vaswani et al.
published: 2017-06-12
captured: 2026-10-06
status: ready
parent: obsidian_ref
id: attention_is_all_you_need
pages: 15 · size: 2.2 MB
next: bob highlights scan
```

- **`source:` line by kind:**
  - Markdown: `source: <md path>`
  - local PDF: `source: <pdf path> (PDF)`
  - PDF URL: `source: <url> (PDF)`
  - arXiv: as in the example above
  - articles: clip's existing report
- **PDF routes** print `pages: N · size: X`, using clip's `format_bytes`. Markdown keeps
  `pages: N`.
- **Dry runs** add a provenance tag to the title, author, and published lines, as clip
  does: `(override)`, `(arxiv api)`, `(pdf info)`, or `(filename)`. They end with
  `writes: none`.
- **Attach mode:**

  ```text
  ok attached listen episode to existing capture
  source: https://arxiv.org/abs/2608.04278 (arXiv 2608.04278)
  pdf: /home/bryan/bob/lib/papers/ea_graph.pdf (unchanged)
  ref: /home/bryan/bob/ref/papers/ea_graph.md
  audio: /home/bryan/bob/xlib/papers/ea_graph.mp3 (from --listen)
  title: EA-Graph: Artifact-Anchored Verification Memory for Coding Agents under Upstream Drift
  next: bob highlights scan
  ```

  An attach dry run says `would attach listen episode to existing capture` and adds
  `listen: would-run …`.

- **Errors** use `bob highlights: error: <message>`, an optional `hint: <next step>`,
  and exit 1. An interrupted listen exits 130.

### Configuration, environment, doctor

- **`src/native/config/mod.rs`:** add `RawHighlights.listen_command` and
  `HighlightsConfig::listen_command()`, trimmed, with blank meaning `None`.
- **New environment variables:**
  - `BOB_HIGHLIGHTS_LISTEN_COMMAND` overrides the configured command.
  - `BOB_HIGHLIGHTS_CURL` replaces `curl`.
  - `BOB_HIGHLIGHTS_KEEP_WORKDIR=1` keeps the scratch directory.
- **`bob highlights doctor` rows.** All of them are warnings, never failures, because
  `--listen` and URL targets are optional:
  - `curl: available (<path>)`, or `curl: warn (command not found)`
  - `listen_command: none`
  - `listen_command: ok (<command>; executable: <program>)`, reusing
    `hooks::pre_scan_program` and `shell_command_available`
  - `listen_command: warn (<validation problem or executable not found>)`
- **Chezmoi** `home/dot_config/bob/config.yml`, which lands in `create-listen`:

  ```yaml
  highlights:
    pre_scan_hook: PATH="$HOME/bin:$PATH" bob_xlib_pull
    # `bob highlights create --listen` (and `clip --listen`) narrates the target with
    # this command, which must write MP3 audio to {audio}; bob binds it as the PDF's
    # companion audio. bob shell-quotes {target} {pdf} {audio} {title} itself — do not
    # quote them. sase-listen's feed.auto_publish publishes the episode. See
    # docs/highlights-create.md in bob-cli.
    listen_command: sase-listen render {target} -e full -o {audio}
  ```

## Phase listen-command: Configurable listen command contract and runner

This phase builds the contract in [Listen command contract](#listen-command-contract).
Nothing calls it from the CLI yet.

- **`src/native/config/mod.rs`:** add `listen_command` parsing, the accessor, and unit
  tests:
  - the value is trimmed;
  - blank and absent mean `None`;
  - an unrelated `highlights` key still parses.
- **New `src/native/highlights_ref/listen.rs`.**
  - It provides:
    - `ListenCommand::resolve()`, which applies env, then config, and returns
      `Option<ListenCommand>`;
    - `ListenCommand::validate()`;
    - `ListenCommand::expand(&ListenValues) -> String`;
    - `run_listen(&ListenCommand, &ListenValues) -> Result<PathBuf, ListenError>`, where
      `ListenError` distinguishes `NotConfigured`, `Invalid`, `Failed { status }`,
      `Interrupted`, and `NoAudio`, each with `message` and `hint`.
  - Register the module in `mod.rs`.
  - `ListenValues` holds `target: String`, `pdf: PathBuf`, `audio: PathBuf`, and
    `title: String`.
  - Signal handling uses `libc`, which is already a dependency: it ignores signals in
    the parent and resets them to `SIG_DFL` in the child's `pre_exec`.
- **Doctor:** add the `listen_command` row (see
  [Configuration, environment, doctor](#configuration-environment-doctor)) in
  `doctor.rs`, after the pandoc and web-clip rows.
- **Unit tests in `listen.rs`:**
  - every validation rule, including `${HOME}` and `{a,b}` passing through;
  - quoting round-trips through `sh -c 'printf "%s\n" …'` for values with spaces, `&`,
    `'`, `$`, backticks, and newlines;
  - `run_listen` with temp shell scripts: it writes an `ID3` header (Ok), exits 4
    (Failed), exits 130 (Interrupted), writes nothing (NoAudio), or writes non-MP3 bytes
    (NoAudio);
  - env-over-config precedence.
- **Done when:** `just all` passes and no CLI behavior changed.

## Phase fetch-arxiv: URL fetcher, arXiv identity and metadata, and shared dedupe

This phase adds library-level building blocks with unit tests. It makes no CLI change
beyond `clip`'s improved dedupe.

- **New `fetch.rs`:** the fetcher in [Fetching](#fetching).
  - Unit-test it with a fake curl script set through `BOB_HIGHLIGHTS_CURL`. The fake
    serves canned files by URL and prints the `-w` lines.
  - Cover: a 200 PDF; a two-hop redirect; a relative `Location`; a redirect to
    `http://10.0.0.1/` that is refused; a 404 status that is returned to the caller;
    curl exits 63 and 28 with their messages; a missing curl with its hint.
- **New `arxiv.rs`:**
  - `ArxivPaper` parsing and URLs. Port sase-listen's `tests/test_arxiv.py` positive and
    negative tables verbatim.
  - Atom metadata parsing, tested on fixtures: a multi-line title with `&amp;`, 1, 2, 3,
    and 5 authors, and an `api/errors` entry.
  - Author display.
  - `fetch_metadata`, which uses `fetch.rs` and degrades to `None` with a warning.
- **Short-title stem** (`short_title_stem` in `clip_url.rs` next to `snake_case`) and
  the **plausible Info title/author** helper (`pdf_info_metadata` in a new
  `pdf_meta.rs`, moving `clip.rs::pdf_info_title` there).
  - Stem examples: "EA-Graph: Artifact-Anchored …" → `ea_graph`; a colon prefix of five
    or more words falls back to the first 6 words; "Attention Is All You Need" →
    `attention_is_all_you_need`.
  - Info title examples: `Microsoft Word - draft.docx` and a title equal to the stem are
    rejected.
- **Dedupe:**
  - Move the source-recording and dedupe code from `clip.rs` to `sources.rs`.
  - Read legacy `url` in addition to `source_url`, from notes and from intake markers.
  - Record `source_pdf` and the `audio`-field presence for ref notes.
  - Extend `dedupe_key_for` with the arXiv key.
  - Unit tests:
    - `https://arxiv.org/pdf/2608.04278`, `/abs/2608.04278v2`, and `/html/2608.04278v1/`
      share one key;
    - a note with only `url:` is recorded;
    - non-arXiv keys are unchanged, and the existing `clip_url` tests pass.
- **Clip regressions:**
  - Existing `tests/cli/highlights/clip.rs` pass unchanged.
  - Add one clip CLI test showing a ref note with a legacy `url:` now refuses the same
    URL.
- **Done when:** `just all` passes.

## Phase pdf-targets: create accepts local PDFs, PDF URLs, and arXiv papers

This phase implements [CLI surface](#cli-surface) (except `-L`),
[Target resolution](#target-resolution) for every route except the article route,
[PDF route](#pdf-route-local-pdf-url-arxiv), and the success and dry-run lines in
[Output](#output).

- **`create.rs` orchestration.** Introduce `target.rs` with
  `enum CreateSource { Markdown, LocalPdf, PdfUrl, Arxiv, WebArticle }` and a resolver.
  - In this phase, `WebArticle` fails with this message:
    `web article URLs are captured by bob highlights clip <URL> (create support lands next)`.
- **Scratch staging.** Move `ClipWorkdir` to `workdir.rs::ScratchDir` (see
  [Ordering and failure semantics](#ordering-and-failure-semantics-every-route-with---listen)),
  and use it in clip too.
  - The **Markdown route now renders into scratch**, not `.<stem>.<pid>.render.pdf`
    beside the target, and writes its Lua filter there too.
  - Everything else in the Markdown route is unchanged: audio discovery, the Play URI,
    and the output lines plus the new `source:` line.
- **Shared companion helpers.** Move `plan_audio_copy_for_source`,
  `files_have_identical_bytes`, and the "copy audio, install PDF, delete created audio
  on failure" sequence into `companion.rs`, so every route shares them.
  - `--audio PATH` binds explicit audio on every route.
  - Discovery stays Markdown-only: frontmatter episode id and narration hash.
  - "Existing companion beside the target is reused" applies to every route.
- **PDF route** in `pdf_target.rs`: validation, title, author, and stem resolution, the
  prepend change in `embed_marker`, the stamp-into-scratch, verify, and install helper,
  existing-capture refusal (no attach yet), and the `-N`/`-T`/per-kind `-t` defaults.
  - PDF URL and arXiv targets stamp `source_url`, `captured`, and `id`.
  - arXiv targets also stamp `author` and `published` when known.
- **Doctor:** add the `curl` row.
- **Completion:** the `target` positional offers `*.md` and `*.pdf`.
- **Help:**
  - Add the new options in short-flag order.
  - Add the Targets section to the after-help.
  - Update
    `tests/cli/help_options.rs::highlights_create_help_lists_options_alphabetically`.
- **CLI tests** in `tests/cli/highlights/create.rs`, with a fake curl and no network:
  - **Local PDF:**
    - It installs `xlib/papers/<stem>.pdf`, and `bob highlights marker` reads the
      marker.
    - The Info title is used, and `-T` overrides it.
    - `-N`, `-t docs`, and `-o` work.
    - The source file's bytes are unchanged.
    - A dry run writes nothing.
    - Encrypted and non-PDF files are refused.
    - A PDF with a page-1 sticky note gets bob's marker first.
    - A marked PDF inside `lib/` is refused with the `--listen` hint.
    - A stem with spaces is snake-cased.
  - **PDF URL:**
    - The marker carries `source_url`, `captured`, and `id`.
    - The stem comes from the URL.
    - A second run is refused by dedupe.
    - A content type claiming PDF with non-PDF bytes is an error.
    - A 404 is an error.
    - An HTML response gets the temporary clip hint.
  - **arXiv:**
    - An abs URL fetches `/pdf/<id>`; assert this from the fake curl's request log.
    - Title, author, and published come from the API fixture.
    - The stem uses the short-title rule.
    - An API failure prints a warning and falls back.
    - A note with a legacy `url: https://arxiv.org/pdf/<id>` refuses the abs URL.
  - **Markdown regression:**
    - All existing create tests pass.
    - The fake pandoc's `-o` argument is outside `xlib/`.
- **Docs:**
  - Create `docs/highlights-create.md`: purpose, the Targets table, PDF and arXiv
    details, dedupe, environment variables, and examples.
  - Link it from `docs/highlights-ref-sync.md` and `docs/highlights-clip.md`.
  - Update the README usage line (it currently omits `-a` and `-n`), the environment
    variable list, and the requirements (curl).
- **Done when:** `just all` passes.

## Phase article-targets: create routes web article URLs through the clip engine

- **Refactor `clip.rs`.** Split `clip_pdf(config, matches)` into `clip_options(matches)`
  plus a callable engine:
  `capture_article(config, raw_url, &ClipOptions, companion: Companion) -> Result<(), ClipError>`.
  - `ClipOptions` gets a `pub(super)` constructor.
  - `Companion` is `None` or an explicit audio path for now; `create-listen` adds the
    listen variant.
  - `clip`'s own behavior and output are unchanged.
- **Route create's `WebArticle` through the engine.** Do this for HTML 2xx and for
  403/429/503. Map create's options:
  - `-T` → title override, and `-N` → name.
  - `-o`, `-P`, `-s`, and `-f` → the same clip options.
  - `-t` → ref type, defaulting to `blogs`.
  - `-d` → dry run.
  - `-a` → explicit companion.
  - author, published, and html stay unset.
  - `-i` is accepted and is a no-op, because clip always stamps `id`.
- **Companion audio in the clip engine.** It plans the companion against the final
  target (after capture) and installs it with the shared `companion.rs` sequence.
- **Tests** use a fake curl serving `text/html` and the existing fake adapter from
  `tests/cli/highlights/clip.rs`. Move its helpers to `tests/cli/highlights/mod.rs` or
  to support code if needed.
  - `create <article URL>` gives the same PDF, marker, and report as `clip`.
  - The option mapping is visible in the adapter's saved `request.json`.
  - A 403 is routed to the adapter.
  - Dedupe still refuses.
  - `-a` binds audio on the article route.
  - Every clip test passes unchanged.
- **Docs:** update the Targets section in `docs/highlights-create.md` and the help text.
  Remove the temporary hint.
- **Done when:** `just all` passes.

## Phase create-listen: Wire --listen into create and clip, with attach mode

- **Add `-L, --listen`** to `create` (conflicting with `-a` and `-n`) and to `clip`.
  Implement
  [Ordering and failure semantics](#ordering-and-failure-semantics-every-route-with---listen)
  on every route:
  - **Markdown:**
    - Discovery is skipped.
    - The listen card's Play URI is computed for `<stem>.mp3`.
    - `{target}` and `{pdf}` are the staged render, named `<scratch>/<stem>.pdf`. Pandoc
      sets its Info Title, which sase-listen reads.
  - **Local PDF, PDF URL, and arXiv:**
    - `{target}` is the source path or the cleaned URL.
    - `{pdf}` is the unstamped scratch copy.
  - **Article:** the engine gains `Companion::Listen`. `{target}` is the cleaned URL and
    `{pdf}` is the adapter's render.
- **Attach mode.** Implement it in a small `attach.rs`, following
  [Dedupe and attach mode](#dedupe-and-attach-mode). It covers all three identity cases
  for `create`, and the dedupe-hit cases for `clip -L`.
- **Reports, dry run, and hints.** Implement them per [Output](#output). The
  failure-after-listen path keeps the scratch audio, as specified.
- **Help:**
  - The **Listen:** after-help section explains the config key, placeholders,
    all-or-nothing behavior, attach mode, and that output streams unchanged.
  - Add the `-L` line to `clip`.
  - Update both help-order tests.
- **CLI tests** use a fake listen command set through `BOB_HIGHLIGHTS_LISTEN_COMMAND`.
  The fake is a script that logs its argv and environment to a file, prints
  `FAKE-LISTEN-STDOUT` and `FAKE-LISTEN-STDERR`, and writes an `ID3` stub to its
  `{audio}` argument. Use a fake curl, a fake adapter, and a fake pandoc as each route
  needs. Cover:
  1. **Markdown + `-L`:**
     - `{target}` exists during the run and is outside `xlib/`.
     - The audio lands beside the target and the PDF is installed.
     - The report shows `audio: … (from --listen)`.
     - A listen card gets the `lib/…mp3` Play URI.
  2. **PDF URL + `-L`:**
     - The `{target}` value is the cleaned URL, and `{title}` is quoted correctly for a
       title containing `'`.
     - `listen: run` precedes the fake's output.
     - Both fake lines appear unchanged on bob's stdout and stderr.
  3. **Listen failure:**
     - Exit 4 → bob exits 1 and `xlib/` is empty, with the error and hint.
     - Exit 130 → bob exits 130.
     - Exit 0 with no audio → error, and nothing is written.
  4. **Dry run** prints `listen: would-run` and the fake is never invoked.
  5. **Unconfigured `-L`** fails before the fake curl is called, with the snippet hint.
     An invalid template is an error.
  6. **Flag conflicts:** `-L` with `-a` or `-n` is a clap usage error.
  7. **Attach:**
     - A ref note with `source_url` gets the audio at `xlib/<rel>.mp3`, and the library
       PDF's bytes are unchanged.
     - A note that has an `audio` field, or a library companion that exists, is refused.
     - An intake hit puts the audio beside the queued PDF.
     - A library PDF given as the target attaches.
     - A legacy `url:` arXiv note attaches for an abs URL.
     - Without `-L`, the refusal hint mentions `--listen`.
  8. **Post-listen collision:** the fake creates the target PDF before exiting. bob
     prints `kept:`, keeps the scratch directory, and leaves the target as the fake
     wrote it.
  9. **Article routes:** `clip -L` and `create <article URL> -L` work with the fake
     adapter.
  10. **Doctor rows** for unset, ok, and invalid listen commands.
- **Chezmoi:**
  - Open the linked `chezmoi` repo with `/sase_repo`, read its `AGENTS.md`, and add the
    `listen_command` block from
    [Configuration, environment, doctor](#configuration-environment-doctor) to
    `home/dot_config/bob/config.yml`.
  - Commit it, and apply it with `chezmoi update -a --force`, following chezmoi's
    `AGENTS.md`.
- **Docs:**
  - **`docs/highlights-create.md`:** Listen, Attach, and Configuration sections, the
    sample output above, and the troubleshooting cases. These include: a sase-listen
    bot-wall 403 on an article that bob captured headed, a credential prompt from
    `pass`, a queued publish, and a rerun after failure that reuses caches.
  - **`docs/highlights-clip.md`:** `-L`.
  - **`docs/highlights-ref-sync.md`:** the listen config key next to `audio_library`,
    and `--listen` in the companion-audio discovery order.
  - **README:** the usage lines, environment variables, and config section.
- **Done when:** `just all` passes and the chezmoi change is committed and applied.

## Phase live-verify: Live end-to-end verification on athena

Run this on athena, which has Chrome, Xvfb, `uv`, `curl`, `pandoc`, and `sase-listen`.

**Rules for this phase:**

- Build with `cargo build --release` and run `target/release/bob`.
- Write only into a scratch vault (`-b <tmp>/vault` with `lib/`, `ref/`, and `xlib/`).
- Never write to the real `~/bob`, and never run a writing `scan` against it.
- Do not `just install`.

**Checks:**

1. `bob highlights create --help`, `clip --help`, and `doctor -b <scratch>` read well
   and show the new rows.
2. Real targets, each with `-d` and then for real:
   - an arXiv abs URL (for example `https://arxiv.org/abs/1706.03762`): check the title,
     authors, published date, `attention_is_all_you_need` stem, and a marker that
     round-trips;
   - a non-arXiv PDF URL;
   - a local PDF;
   - a friendly article URL (for example a Lil'Log post) through the clip engine;
   - a Markdown regression.
3. Run `bob highlights scan -b <scratch>` and confirm:
   - the ref notes carry `source_url`, `author`, `published`, and `captured`;
   - the second `create` of the same arXiv paper (any URL spelling) is refused with the
     `--listen` hint.
4. **One real listen run, without publishing.** Pick a short arXiv paper of 4 pages or
   fewer. The budget is ≤ $0.50.
   - Run it under a pseudo-TTY: `script -qec '…' /dev/null`.
   - Set
     `BOB_HIGHLIGHTS_LISTEN_COMMAND='sase-listen render {target} -e full -o {audio} --no-publish'`.
   - Confirm that:
     - sase-listen's live checklist renders unchanged;
     - the MP3 lands beside the intake PDF;
     - `scan -b <scratch>` moves both, writes `audio: "[[lib/papers/<stem>.mp3]]"`, and
       puts the player under the `^ref` task.
   - Then run attach mode against that scratch capture with a second paper's dedupe hit,
     or with the library PDF as the target, and rescan.
   - Skip this step and record a follow-up if sase-listen credentials would prompt
     interactively.
   - Never publish.
5. Fix any defects you find, with tests.
6. Use `/sase_new_task` to file follow-ups for issues out of this epic's scope. Likely
   ones:
   - **Bot-walled articles:** sase-listen 403s on an article that bob captured headed. A
     possible fix is a `{html}` placeholder feeding sase-listen `-H`.
   - **Legacy `url:` keys:** migrating them to `source_url` in existing paper refs.
7. Add a short "Verified on athena" note to `docs/highlights-create.md` with the date
   and the commands.
8. In the final report, tell Bryan to run `just install` (or `just install-all`) to put
   the feature on his `PATH`. Do not install it yourself.

## Out of scope

- `arxiv:ID` and bare arXiv ids, `kind:path` SASE refs, and narration scripts as
  targets. `sase-listen -e full` rejects all of them.
- An edition flag on `--listen`. The edition lives in the configured command.
- A standalone `bob highlights listen` subcommand. Attach mode covers that use.
- Recapture or versioned snapshots of an already-captured URL.
- Migrating legacy `url:` keys. They are only read for dedupe.
- Running capture and listen in parallel.
- Changes to the Bob Mac Capture app or the sase-listen repo.

## Risks

- **Bot-walled articles.** sase-listen fetches the URL itself with `curl_cffi`. A
  Cloudflare-walled page that bob captured headed can still make the listen run fail.
  The result is a clean failure with nothing written, plus a documented workaround,
  rather than a half-tracked episode. The follow-up idea is above.
- **Rewriting arbitrary PDFs.** `lopdf` rewrites the whole file (no incremental update),
  so unusual PDFs may grow or fail to load. This is mitigated by the encrypted-PDF
  refusal, the stamp-verify step before install, and the 95 MiB cap.
- **Metadata heuristics.** Info titles can be junk. The plausibility filter, `-T`, and
  the dry-run provenance tags make that visible and fixable.
- **Cost.** `-e full` renders cost money (roughly $0.10–0.25 each). bob never runs the
  listen command in a dry run, and it never retries automatically.
