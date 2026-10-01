---
tier: epic
title: bob highlights clip — web URL to Highlights reference PDF
goal: '`bob highlights clip <URL>` turns a web article into a beautiful, readable,
  provenance-stamped PDF in the Highlights intake (`~/bob/xlib/blogs/` by default),
  which the existing `bob highlights scan` turns into a `~/bob/ref/` note. The Cloudflare-protected
  OpenAI Symphony post is captured on a host that can run headed Chrome (athena),
  and every unsupported case fails loudly with a next step, never with a silently
  wrong PDF.

  '
phases:
- id: stamp-core
  title: Shared target, marker, and install helpers
  depends_on: []
  size: small
  description: 'stamp-core: factor create''s target planning, collision guards, marker
    composition, and stamp-plus-atomic-install into a shared module; add optional
    provenance marker keys; make `captured` a standard synced field.'
- id: adapter-capture
  title: Web clip adapter capture and extraction
  depends_on: []
  size: medium
  description: 'adapter-capture: pinned uv/Playwright adapter (protocol v1) that launches
    Chrome, falls back to headed Xvfb on bot challenges, snapshots a stable DOM, runs
    vendored Defuddle in isolation, layers metadata, checks fidelity, sanitizes, and
    localizes images; placeholder renderer; offline self-test.'
- id: reader-template
  title: Reader print template and renderer
  depends_on:
  - adapter-capture
  size: medium
  description: 'reader-template: Bob-owned Chromium print template with bundled OFL
    fonts, masthead, Highlights-safe typography, Pillow image normalization, offline
    headless print with outline and n / N footers.'
- id: clip-command
  title: bob highlights clip Rust command
  depends_on:
  - stamp-core
  size: medium
  description: 'clip-command: clap subcommand, URL validation and slug rules, source_url
    dedupe, adapter client with BOB_WEB_CLIP_ADAPTER seam, dry-run and success reports,
    doctor rows, fake-adapter CLI tests, docs.'
- id: live-verify
  title: Live OpenAI capture verification and docs finish
  depends_on:
  - reader-template
  - clip-command
  size: medium
  description: 'live-verify: build bob, capture the OpenAI Symphony URL on athena
    into a scratch vault and then the real intake queue, inspect pages and text layer,
    prove scan writes the ref note, confirm apollo fails closed, fix issues, record
    follow-ups.'
proposed_by: bbugyi200.apollo.3s
create_time: 2026-10-01 02:07:04
status: done
bead_id: bob-cli-35
---

- **PROMPT:** [prompts/202610/web_url_highlights_clip.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/web_url_highlights_clip.md)
- **BEAD:** [bob-cli-35](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-35/README.md)

# Plan: `bob highlights clip` — web URL to Highlights reference PDF

## Background and evidence

Bryan wants a reliable way to turn a web URL such as
`https://openai.com/index/open-source-codex-orchestration-symphony/` into a beautiful,
readable reference PDF for the Bob vault, in the spirit of `bob highlights create -i`
(Markdown → pandoc/XeLaTeX PDF → page-1 marker → `xlib/<ref_type>/<stem>.pdf`). This
epic implements the recommendation of the research report
`research:202610/web_url_reference_pdf_capture/web_url_reference_pdf_capture.md` (read
it with `sase artifact read` if you need more depth). The design adds a capture front
end to the existing Highlights intake.

How `create` and `scan` work today (`src/native/highlights_ref/create.rs`, `mod.rs`,
`docs/highlights-ref-sync.md`, `docs/vault-git-sync.md`):

- `create` plans `xlib/<ref_type>/<stem>.pdf` and refuses an existing same-stem `.md` or
  `.textbundle` sidecar. It also refuses an occupied library destination even with
  `--force`, and refuses an existing target unless `--force` is passed. It then renders,
  embeds a page-1 `/Text` marker with `lopdf`, and installs with `atomic_save_pdf`.
- `scan` (run on the Mac by a 15-minute cron, after `bob_xlib_pull` drains the athena
  and apollo `~/bob/xlib/` queues) moves `xlib/<rel>` to `lib/<rel>` and writes
  `ref/<rel>.md`. Unknown marker keys round-trip into note frontmatter. `source_url`,
  `author`, and `published` are already standard synced fields (`COMMON_USER_FIELDS`).
- **The clip command never writes `ref/` notes and never scans.** Scan stays the only
  note writer. Never run a writing `scan` against the real vault on athena or apollo.
- `bob gkeep` is the in-repo precedent for a pinned Python helper. It embeds
  `scripts/gkeep_adapter.py` (registered in `src/scripts.rs` `SUPPORT_ASSETS`),
  materializes it with `crate::runner::materialize_scripts()`, and runs
  `uv run --quiet --script` with one JSON request on stdin and one JSON response on
  stdout. `BOB_GKEEP_ADAPTER` is the test seam (`src/native/gkeep/adapter.rs`).

Facts measured on 2026-10-01 while planning. They drive the design:

| Probe                                                                                             | Result                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `curl` of the OpenAI URL from apollo (DigitalOcean) and athena (home server)                      | 403 `cf-mitigated: challenge` on both                                                                                                                                                                                                                                                                  |
| Playwright + installed Google Chrome 154, **headless**, on athena                                 | 403 "Just a moment..."                                                                                                                                                                                                                                                                                 |
| Same, **headed under Xvfb** (`xvfb-run`), on athena                                               | **200**, full article (~10.5k visible words), challenge cleared without any interaction                                                                                                                                                                                                                |
| Headed with `navigator.webdriver === true` (stock Playwright)                                     | page loads, then about 2 s later OpenAI's app replaces the article with "This page couldn't load" (13 words)                                                                                                                                                                                           |
| Headed with Chrome switch `--disable-blink-features=AutomationControlled`                         | DOM stays stable (10,858 words after 8 s); `chromium_sandbox=True` works; color scheme irrelevant                                                                                                                                                                                                      |
| Defuddle 0.19.4 `dist/index.full.js` (UMD, 766 KB, sets `window.Defuddle`) injected into the page | title, author "Alex Kotliarskyi, Victor Zhu, and Zach Brock", site "OpenAI", 8,921 words, 2 images, 1 `<pre>`                                                                                                                                                                                          |
| Defuddle `published`                                                                              | `2026-03-08T03:45:54Z`, **wrong**: it comes from an embedded tweet's `<time>`. The visible date above the H1 is **"April 27, 2026"** (it matches the OpenAI RSS `pubDate`)                                                                                                                             |
| Defuddle images                                                                                   | it collapsed each `<picture>` to its **first `<source>`**, which is the `(prefers-color-scheme: dark)` variant (`*-Desktop-Dark-*.svg`). On white paper those diagrams print nearly invisible. The `<img>` fallback and the plain `(min-width: 768px)` source are the `*-Desktop-Light-*.svg` variants |
| Prototype print (Defuddle HTML in a bare template, headless Chrome `page.pdf`)                    | 27 Letter pages, 2.9 MB, `1 / 27` footer via CSS margin boxes, outline and tagged output OK                                                                                                                                                                                                            |
| apollo tooling                                                                                    | no Chrome, Chromium, or Xvfb; `uv`, `node`, `pandoc`, `mutool` present; glibc 2.39                                                                                                                                                                                                                     |
| athena tooling                                                                                    | `/usr/bin/google-chrome` (154), `Xvfb`, `xvfb-run`, `uv`, `pandoc`, `~/.cache/ms-playwright`; glibc 2.41, x86_64; an apollo-built `bob` binary runs there                                                                                                                                              |
| Dependency pins resolvable with `exclude-newer = 2026-09-01`                                      | `playwright==1.62.0`, `pillow==12.3.0`, `nh3==0.3.7`                                                                                                                                                                                                                                                   |
| Fonts                                                                                             | `@fontsource-variable/source-serif-4`, `@fontsource-variable/inter`, `@fontsource-variable/jetbrains-mono` 5.3.0 (OFL-1.1). Latin `wght` woff2 files are about 50 KB each                                                                                                                              |

Conclusions:

- A headless-only tool cannot capture the motivating URL from any Bryan host.
- Headed real Chrome on a residential IP can capture it. That means athena under a
  private Xvfb display, or the Mac.
- On apollo, the command must fail closed with an actionable hint.
- The automation switch and the picture fix are both required for a correct OpenAI
  capture.

## Design decisions

1. **Command:** `bob highlights clip <URL>`, a sibling of `create` that shares its
   target, collision, marker, and install code. It writes one marker-stamped PDF into
   `xlib/<ref_type>/<stem>.pdf` (default `ref_type` **`blogs`**, matching the existing
   `lib/blogs/` and `ref/blogs/`). `scan` creates `ref/blogs/<stem>.md` later.
2. **Reader mode only in v1.** The output re-typesets the article with a Bob-owned
   template; it is not a facsimile of the site. Printing the live page is out of scope.
3. **"Reliable" means "never silently wrong".** The command fails closed, with no PDF
   and a hint, on bot challenges it cannot clear, login walls, thin extractions, missing
   browsers, and private-network URLs. It warns on dropped images or code blocks.
4. **Renderer:** headless Chrome prints a Bob-owned HTML/CSS template (bundled fonts, no
   auto-hyphenation, no ligatures, left-aligned). This handles SVG, WebP, MathML, CJK,
   and emoji natively, where pandoc/XeLaTeX hard-fails on SVG/WebP. Highlights quote
   quality on the Mac can only be checked by Bryan by hand. That check is a recorded
   follow-up, and if it fails, only the render stage changes.
5. **Capture strategy:**
   - Start with a headless launch.
   - If a challenge is detected, retry automatically in a headed browser on a display
     nobody sees: a private Xvfb display on Linux, or an off-screen window on macOS.
   - If no headed display is available, fail with a hint.
   - Use a fresh temporary profile, keep the sandbox on, and pass the single switch
     `--disable-blink-features=AutomationControlled`. Measured: without it, openai.com
     destroys its own article.
   - No user-agent spoofing, stealth plugins, fingerprint patches, cookie reuse from
     Bryan's profile, or automated challenge solving.
6. **Extraction:** vendored, pinned Defuddle runs in an isolated offline page against a
   cleaned static snapshot of the live DOM. On top of that sit layered metadata, a
   fidelity check, and an `nh3` allowlist sanitizer.
7. **Provenance is mandatory.** Every marker carries `status`, `parent`, `title`, `id`
   (always stamped and equal to the file stem), and `source_url`. It also carries
   `author`, `published` (`YYYY-MM-DD`), and `captured` (`YYYY-MM-DD`, local date) when
   known. A field is omitted rather than written wrong. The PDF masthead also prints the
   source URL and capture date.
8. **Deduplication:** a URL that is already a ref note (`source_url` in `ref/**.md`
   frontmatter) is reported and refused, even with `--force`. A URL already queued in
   `xlib/**.pdf` with a different target is refused too. Library PDFs are never
   overwritten.
9. **Defaults:** `parent` is `obsidian_ref` and `status` is `ready`, the same as
   `create`. Pages are US Letter.

## Contracts shared by several phases

### CLI surface (owned by `clip-command`)

Options are sorted by short flag (uppercase before lowercase), the same way the `create`
help test orders them. Every long option has a short alias (`sase/memory/cli_rules.md`).

```text
bob highlights clip [OPTIONS] <URL>

Arguments:
  <URL>  http(s) URL of the article to capture; recorded as the marker source_url

Options:
  -A, --author <NAME>      Override the extracted author
  -b, --bob-dir <PATH>     Bob vault root; defaults to BOB_DIR or ~/bob
  -d, --dry-run            Capture and extract, then print the plan, marker, metadata sources, and fidelity report; write nothing
  -f, --force              Overwrite an existing intake PDF for this capture (never a library PDF)
  -H, --html <FILE>        Use a page already saved from a browser (Save Page As, SingleFile); `-` reads stdin
  -l, --lib-dir <PATH>     Highlights PDF library; defaults to BOB_HIGHLIGHTS_LIB_DIR or lib
  -N, --name <STEM>        Output filename stem and marker id [default: derived from the URL]
  -o, --output <PDF>       Complete path for the generated PDF, including the .pdf filename
  -P, --parent <NOTE>      Bare Obsidian note target for the marker parent [default: obsidian_ref]
  -p, --published <DATE>   Override the extracted publish date (YYYY-MM-DD)
  -r, --ref-dir <PATH>     Reference note output directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -s, --status <STATUS>    Lifecycle status embedded in the marker [default: ready] [ready|next|wip|read|abandoned|legacy]
  -T, --title <TITLE>      Override the extracted title
  -t, --ref-type <DIR>     Single library subdirectory for the generated PDF [default: blogs]
  -x, --xlib-dir <PATH>    Highlights PDF intake directory; defaults to BOB_HIGHLIGHTS_XLIB_DIR or xlib
```

- `--output` conflicts with `--ref-type` and `--name`.
- The parent flag `bob highlights --no-hooks clip …` is accepted and ignored, as for
  `create`.
- The after-help explains: the reader-mode pipeline; that `scan` creates the ref note;
  the headed-browser fallback and which hosts can clear bot challenges; the `--html`
  escape hatch; and the environment variables below.

Environment variables:

| Variable                    | Meaning                                                                                                                |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `BOB_WEB_CLIP_ADAPTER`      | Replaces the `uv run --quiet --script …/web_clip/web_clip_adapter.py` invocation (test seam, like `BOB_GKEEP_ADAPTER`) |
| `BOB_CHROME`                | Chrome/Chromium executable the adapter launches instead of auto-discovery                                              |
| `BOB_WEB_CLIP_TIMEOUT_SECS` | Overall adapter timeout (default 300; the first run may download pinned Python deps)                                   |
| `BOB_WEB_CLIP_KEEP_WORKDIR` | `1` keeps the scratch directory for debugging and prints its path                                                      |

Success output (plain `key: value` lines like `create`, with a colored `ok` prefix and
yellow `warning:` lines):

```text
ok created Highlights-ready web PDF
pdf: /home/bryan/bob/xlib/blogs/open_source_codex_orchestration_symphony.pdf
title: An open-source spec for Codex orchestration: Symphony
author: Alex Kotliarskyi, Victor Zhu, and Zach Brock
published: 2026-04-27
captured: 2026-10-01
source_url: https://openai.com/index/open-source-codex-orchestration-symphony/
status: ready
parent: obsidian_ref
id: open_source_codex_orchestration_symphony
capture: chrome 154 · headed (Xvfb) after a headless bot challenge
pages: 27 · images: 2/2 · size: 2.4 MB · fidelity: ok
next: bob highlights scan
```

On failure, stderr shows `bob highlights: error: <message>` plus a `hint: <next step>`
line, the exit code is 1, and nothing is written.

### Adapter protocol v1 (owned by `adapter-capture`; `clip-command` codes against it with a fake)

The Rust side writes one compact JSON object to stdin and closes it. The adapter writes
exactly one JSON object to stdout and logs to stderr. It exits 0 whenever it wrote a
response, whether `ok` is true or false. Non-JSON stdout or a nonzero exit is a protocol
error, and Rust shows the stderr tail.

`ping` request: `{"protocol": 1, "op": "ping"}`. Response:

```json
{
  "protocol": 1,
  "ok": true,
  "op": "ping",
  "python": "3.12.3",
  "playwright": "1.62.0",
  "pillow": "12.3.0",
  "nh3": "0.3.7",
  "defuddle": "0.19.4",
  "browser": {
    "kind": "chrome",
    "path": "/usr/bin/google-chrome",
    "version": "154.0.8037.92"
  },
  "headed": "xvfb",
  "fonts": ["Source Serif 4", "Inter", "JetBrains Mono"]
}
```

`browser` is `null` when none is found. `headed` is one of `xvfb`, `display`, `macos`,
or `null`.

`capture` request:

```json
{
  "protocol": 1,
  "op": "capture",
  "url": "https://openai.com/index/open-source-codex-orchestration-symphony/",
  "html_path": null,
  "workdir": "/tmp/bob-clip-1234-5678",
  "out_pdf": "/tmp/bob-clip-1234-5678/render.pdf",
  "dry_run": false,
  "captured": "2026-10-01",
  "overrides": { "title": null, "author": null, "published": null }
}
```

`capture` success response:

```json
{
  "protocol": 1,
  "ok": true,
  "op": "capture",
  "kind": "article",
  "final_url": "https://openai.com/index/open-source-codex-orchestration-symphony/",
  "title": "An open-source spec for Codex orchestration: Symphony",
  "author": "Alex Kotliarskyi, Victor Zhu, and Zach Brock",
  "published": "2026-04-27",
  "site": "OpenAI",
  "description": "Learn how Symphony, …",
  "metadata_sources": {
    "title": "h1",
    "author": "byline",
    "published": "visible-date",
    "site": "og:site_name"
  },
  "word_count": 8921,
  "capture": {
    "browser": "chrome",
    "browser_version": "154.0.8037.92",
    "mode": "headed-xvfb",
    "retried_after_challenge": true
  },
  "fidelity": {
    "status": "ok",
    "page_large_media": 2,
    "kept_large_media": 2,
    "page_code_blocks": 1,
    "kept_code_blocks": 1,
    "page_words": 10858,
    "kept_words": 8921
  },
  "images": { "total": 2, "kept": 2, "skipped_small": 0, "failed": 0 },
  "pdf_bytes": 2400000,
  "warnings": []
}
```

- `kind` is `"pdf"` when the URL serves `application/pdf`. In that case `out_pdf` holds
  the original bytes; title, author, and published are `null` unless overridden; and
  Rust derives the title.
- `capture.mode` is one of `headless`, `headed-xvfb`, `headed-display`, `headed-macos`,
  `html-file`, or `direct-pdf`.
- With `dry_run: true`, the adapter skips image downloads and rendering, `pdf_bytes` is
  `null`, and `out_pdf` is not written.
- Rust counts pages with `lopdf`, so the protocol carries no page count.

Failure response:
`{"protocol": 1, "ok": false, "op": "capture", "error": {"kind": "…", "message": "…", "hint": "…"}}`.
Error kinds:

| Kind                  | Meaning                                                      |
| --------------------- | ------------------------------------------------------------ |
| `blocked`             | Challenge or login wall not cleared                          |
| `thin`                | Extraction too small                                         |
| `browser`             | No usable browser or launch failure                          |
| `network`             | DNS, TLS, or HTTP errors, including private-network refusals |
| `timeout`             | Capture took too long                                        |
| `unsupported_content` | Image, video, or other non-HTML, non-PDF content             |
| `render`              | Printing failed                                              |
| `invalid_request`     | The request was malformed                                    |

### Marker and note fields (owned by `stamp-core`, used by `clip-command`)

- **Marker keys:** `status`, `parent` (`[[…]]`), `title`, `id`, `source_url`, `author`,
  `published`, `captured`. The last three are optional.
- **`captured` becomes a standard synced field** (`COMMON_USER_FIELDS`), so marker and
  frontmatter edits sync both ways.
- **Round trip:** after `scan`, the note frontmatter carries every key. A second `scan`
  makes no changes and reports no conflicts. Dates render consistently in the marker and
  the frontmatter.

### Slug and URL rules (owned by `clip-command`)

- **Validation:**
  - Only `http`/`https`, with no userinfo.
  - Reject `localhost`, `*.localhost`, `*.local`, and `*.internal`.
  - Reject IP literals that are loopback, private, link-local, unspecified, CGNAT
    (`100.64.0.0/10`, the Tailscale range), or ULA.
  - The adapter separately refuses requests whose hosts resolve to such addresses, which
    also covers redirects.
- **Stored `source_url`:** the user's URL with the fragment and tracking parameters
  (`utm_*`, `fbclid`, `gclid`, `mc_cid`, `mc_eid`, `ref_src`) removed. Otherwise kept as
  given.
- **Dedupe key:**
  - lowercase scheme and host, with a leading `www.` removed and the default port
    dropped;
  - path without a trailing slash;
  - remaining query parameters sorted.
- **Stem:**
  - `--name` if given; it must match `[A-Za-z0-9][A-Za-z0-9_.-]*`, and a trailing `.pdf`
    is stripped.
  - Otherwise, walk the percent-decoded path segments from last to first.
  - Skip segments that are empty, numeric, two characters or fewer, or date parts.
  - Also skip `index`, `default`, `home`, `post`, `posts`, `p`, `article`, `articles`,
    `blog`, `amp`, `en`, and `en-us`, with or without the extensions below.
  - Strip `.html`, `.htm`, `.php`, `.asp`, `.aspx`, and `.shtml` extensions, and a
    leading `YYYY-MM-DD-` prefix.
  - snake*case the first remaining segment: lowercase, every run of characters outside
    `[a-z0-9]` becomes `*`, trimmed, and capped at 80 characters at a `_` boundary.
  - Fall back to the snake*cased title after capture, then to `<host>*<YYYYMMDD>`.
  - Example: `/index/open-source-codex-orchestration-symphony/` →
    `open_source_codex_orchestration_symphony`.

## Phase stamp-core: Shared target, marker, and install helpers

Goal: `create` behaves exactly as before, while its reusable parts move to a new
`src/native/highlights_ref/stamp.rs` (declared in `mod.rs`) that `clip` can call.

- **Move into `stamp.rs` (`pub(super)`):**
  - the workflow enum (rename `CreateWorkflow` → `TargetWorkflow`, with
    `Intake { library_destination }`, `Library`, and `External`);
  - a target planner: given a config, `stem`, `ref_type`, optional exact `output`, and
    `force`, it returns `TargetPlan { target, sidecar, workflow }` and runs the existing
    `refuse_create_collisions` rules unchanged;
  - `resolve_exact_output_path`, `validate_pdf_output_path`, `classify_create_target`,
    `relative_inside`/`path_is_inside`/`normalize_lexically`, `validate_ref_type`,
    `library_destination_sidecars`, `embed_marker`, and `print_next_step` (taking a
    `TargetPlan`).
- **Extend marker composition:** keep the validation path of
  `compose_marker(status, parent, title, id, extras: &[(&str, String)])`. Extras are
  inserted as `MarkerValue::String` in the order `source_url`, `author`, `published`,
  `captured`, and empty values are skipped. `create` passes no extras.
- **Add
  `stamp_and_install(rendered_pdf: &Path, target: &Path, marker: &str, info: PdfInfo) -> Result<usize>`:**
  - load the PDF with `lopdf` and count pages;
  - embed the marker;
  - optionally set the document Info `/Title` and `/Author` (`PdfInfo` has optional
    fields; `create` passes none, so its output is unchanged);
  - save with `atomic_save_pdf`.
  - `create`'s `render_and_install` keeps its pandoc invocation and calls this helper.
- **Make `captured` a standard field:** add `FIELD_CAPTURED = "captured"` to
  `COMMON_USER_FIELDS` in `mod.rs`, and update the standard-field list in
  `docs/highlights-ref-sync.md` (Synced Properties section) to explain `captured`
  (snapshot date of a web capture).
- **Move or keep the existing `create.rs` unit tests** so they still cover the shared
  planner.
- **Add tests:**
  - stamping a small fixture PDF generated in-test with `lopdf` (one page, no `/Annots`;
    and a second case where `/Annots` already exists) with all extra keys;
  - `read_pdf_marker` returns them verbatim;
  - Info `/Title`/`/Author` are set when requested;
  - a CLI-level test in `tests/cli/highlights/` that stamps through the `create` path
    with the fake pandoc still passes unchanged;
  - a scan round-trip test (temp vault, `--no-hooks`) proving a marker with
    `source_url`, `author`, `published`, and `captured` lands in the ref note
    frontmatter, and that a second scan is a no-op.
- **Acceptance:** `just all` passes; `bob highlights create` output and behavior are
  byte-for-byte unchanged in the existing tests.

## Phase adapter-capture: Web clip adapter capture and extraction

Create `scripts/web_clip/` with these files:

- `web_clip_adapter.py`, the PEP 723 entry point. Model its structure, logging, and
  self-test style on `scripts/gkeep_adapter.py`. Header: `requires-python = ">=3.10"`,
  `dependencies = ["playwright==1.62.0", "pillow==12.3.0", "nh3==0.3.7"]`, and
  `[tool.uv] exclude-newer = "2026-09-01T00:00:00Z"`.
- `snapshot.js`, the in-page snapshot, probe, and metadata code, injected with
  `page.evaluate`.
- `web_clip_render.py`, a **placeholder** renderer (bare HTML, like the planning
  prototype) so the pipeline works end to end; `reader-template` replaces it. The
  adapter imports it as a sibling module, because the script's directory is on
  `sys.path`.
- `vendor/defuddle.full.js` (from npm `defuddle@0.19.4`, `package/dist/index.full.js`,
  fetched with `npm pack`) and `vendor/DEFUDDLE_LICENSE` (MIT text from the package),
  plus a `vendor/README.md` recording the version, source, and SHA-256.
- Self-test fixtures in `fixtures/` (not embedded).

Register every runtime file (not the fixtures) in `src/scripts.rs` `SUPPORT_ASSETS` with
`install_path` `web_clip/<same relative path>`. Add a justfile recipe
`check-web-clip-adapter`: `python3 -m py_compile` on the Python files, then
`uv run --quiet --script scripts/web_clip/web_clip_adapter.py --self-test`.

Capture pipeline, with all stages in the adapter:

1. **Request validation and scratch setup.**
   - Validate the protocol version, op, and an absolute `workdir`.
   - Set `TMPDIR` to `workdir` before starting Playwright, so Chrome's profile and
     socket paths stay short. SASE's deep per-agent `TMPDIR` breaks Chromium sockets.
2. **Browser discovery.** The first match wins:
   1. `BOB_CHROME` (as `executable_path`);
   2. installed Google Chrome via Playwright `channel="chrome"`. Probe
      `/opt/google/chrome/chrome`, `/usr/bin/google-chrome`,
      `/usr/bin/google-chrome-stable`, and
      `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome`;
   3. Playwright's bundled Chromium, if its executable exists.
   - If none is found, return error `browser` with the hint "install Google Chrome or
     set BOB_CHROME=/path/to/chrome".
   - Report the version through `browser.version`.
3. **Direct-PDF probe.** Send a `context.request.head` (fall back to a ranged GET when
   HEAD is refused). If the content type is `application/pdf`, download it to `out_pdf`
   with a 95 MiB cap and return `kind: "pdf"`, `capture.mode: "direct-pdf"`. Also handle
   a navigation that turns into a download. Other non-HTML types return
   `unsupported_content`.
4. **Launch.**
   - Call
     `chromium.launch(headless=…, chromium_sandbox=True, args=["--disable-blink-features=AutomationControlled"])`.
     If the launch fails specifically because the sandbox is unavailable, relaunch with
     `chromium_sandbox=False` and add a warning.
   - Context options: viewport 1280×1600, `color_scheme="light"`,
     `reduced_motion="reduce"`, `service_workers="block"`, `accept_downloads=False`,
     locale `en-US`, and no user-agent override. Close popups as they open.
5. **Private-network guard.** A `context.route("**/*")` handler resolves each request
   host (cached per host) and aborts requests to non-global addresses. A refused
   top-level document becomes error `network`.
6. **`--html` replay.** When `html_path` is set, fulfill only the first top-level
   document request for `url` with the file's bytes (`text/html`). Subresources load
   live as best effort. The mode is `html-file`, and the challenge retry is skipped.
7. **Navigate and wait.**
   - `goto(url, wait_until="domcontentloaded", timeout=45s)`; record the status and the
     `cf-mitigated`/`server` headers.
   - Then poll every 250 ms, for at most 20 s, with a probe from `snapshot.js`. The
     probe returns:
     - the title;
     - challenge markers: title starting with "Just a moment", "Attention Required",
       "Checking your browser", or "Access denied"; `#challenge-error-text`;
       `#cf-challenge-running`; `form#challenge-form`; or the `cf-mitigated: challenge`
       header;
     - login-wall phrases ("sign in to continue", "log in to continue", "subscribe to
       continue reading", "create a free account") when there are fewer than 300 visible
       words;
     - whether an article candidate exists (`h1`, `article`, or `main` with text);
     - the visible word count.
   - A non-empty title is required before a page counts as "not challenged". Planning
     hit an empty-title false positive.
8. **Richest-snapshot rule.**
   - On every non-challenged tick whose visible word count beats the best so far, take a
     snapshot. Stop once the count has been stable for 1 s and an article candidate
     exists.
   - Then run a lazy-load pass, bounded to 12 s:
     - scroll in 800 px steps;
     - set `img.loading = "eager"`;
     - open every `<details>`;
     - wait at most 5 s for `document.fonts.ready` and image completion.
   - Take a final snapshot only if the page is still healthy: at least 80% of the best
     visible words and the article candidate still present. Otherwise keep the best
     earlier snapshot and add a warning.
9. **Challenge fallback.**
   - If the headless page is still challenged when the 20 s poll ends, close it and
     relaunch **headed**.
   - Display choice, in order:
     - Linux: start a private `Xvfb` with `-displayfd`, `-nolisten tcp`, and
       `-screen 0 1440x1000x24`, pass `DISPLAY` only to the browser process, and kill it
       in a `finally` (`headed-xvfb`);
     - otherwise Linux with an existing `DISPLAY` (`headed-display`);
     - macOS: headed with `--window-position=-32000,-32000` (`headed-macos`; best
       effort, not verified in this epic).
   - Wait up to 30 s for the challenge to clear without interacting.
   - Error messages:
     - no display available: `blocked`, "this site blocks headless browsers; rerun on a
       host that can run headed Chrome (Linux with Xvfb such as athena, or the Mac), or
       save the page from your browser and pass --html FILE";
     - still challenged: `blocked`, "the bot challenge did not clear; save the page from
       your browser and pass --html FILE";
     - login wall: `blocked` with a login hint.
10. **Snapshot cleaning** (in `snapshot.js`, on a `cloneNode(true)` of the live
    `documentElement`, walking live and clone elements in parallel so they map
    one-to-one):
    - **Remove elements hidden in the live page:**
      - computed `display:none` or `visibility:hidden`;
      - screen-reader-only boxes: 1 px or smaller, with overflow hidden or clipped;
      - `aria-hidden="true"` outside figures.
    - **Remove** `script`, `style`, `link`, `template`, `noscript`, `iframe` (record
      embeds first), `form`, `input`, `button`, and `select`.
    - **Resolve every `<picture>`** to a single `<img>`:
      - pick the first `<source>` whose `media` matches `window.matchMedia` in the
        light, 1280 px-wide context and whose `type` is supported; otherwise use the
        `<img>`'s own attributes;
      - set `src` to the best candidate from that `srcset` (the largest `w` up to 2560,
        or the largest density), and copy `alt`/`width`/`height`.
      - This is the fix for the OpenAI dark-diagram bug: the result must be the
        `*-Light-*` variants.
    - **For plain images:** promote `data-src`/`data-srcset`, prefer the live
      `currentSrc`, then the largest `srcset` candidate; drop
      `srcset`/`sizes`/`loading`.
    - **Absolutize** every `href`/`src` against the page URL.
    - **Serialize inline `<svg>`** to standalone assets later; tag them here with
      `data-bob-svg`.
    - **Record the probe facts** at the same time:
      - the H1 text;
      - JSON-LD `Article`/`BlogPosting` author and `datePublished` (ignoring any inside
        embeds);
      - `meta[name=author]`, `article:author`, `article:published_time`, `og:site_name`,
        `og:title`, and `description`;
      - date candidates near the H1: `<time datetime>` elements and date-like text nodes
        within the H1's ancestors up to three levels, excluding anything inside embeds
        (`blockquote.twitter-tweet`, `[class*=tweet]`, `[data-testid*=tweet]`, iframes,
        `[class*=embed]`);
      - a `By …` byline near the H1;
      - fidelity counts: large media (rendered width ≥ 300 px, below the H1, not in
        embeds, counting `picture`/`img` once), `<pre>` count, and visible words.
11. **Extraction in isolation.**
    - Open a separate page in the same context, routed to abort every request.
    - Load an empty document, replace its `documentElement` with the imported cleaned
      snapshot, inject `vendor/defuddle.full.js` via `add_script_tag(content=…)`, and
      run `new Defuddle(document, {url}).parse()`.
    - If Defuddle needs layout that the isolated page lacks (verify this on fixtures),
      fall back to running it in the live page with `bypass_csp=True` set on the
      context, right after the snapshot. Note which path was used in a debug log.
12. **Metadata layering.** Overrides always win. Record each field's source in
    `metadata_sources`.
    - **title:**
      1. the H1, when it fuzzily matches the og/Defuddle title (case-folded, punctuation
         stripped, containment either way);
      2. the og/Defuddle title with a trailing ` | Site` or ` - Site` suffix removed;
      3. the URL slug.
    - **author:** JSON-LD → meta → visible byline (with "By " stripped) → Defuddle.
      Discard any candidate equal to an embed's author or handle.
    - **published:**
      1. JSON-LD `datePublished`;
      2. `article:published_time`;
      3. the visible date near the H1, parsed with formats such as `%B %d, %Y`,
         `%b %d, %Y`, `%d %B %Y`, and ISO;
      4. Defuddle, only if it does not equal a date found inside an embed.
      - Normalize to `YYYY-MM-DD`, or `null`.
    - **site:** `og:site_name` → Defuddle `site` → host.
13. **Content cleanup and sanitizing** (in Python, after Defuddle):
    - **Text cleanup:**
      - drop a leading heading that fuzzily repeats the title;
      - strip U+200B–U+200D, U+2060, U+FEFF, and U+00AD;
      - strip the suffixes "(opens in a new window)" and "(opens in a new tab)".
    - **`nh3` allowlist:**
      - text, list, table, code, quote, figure, `details`/`summary`, `time`, `abbr`,
        `mark`, `sub`/`sup`, `br`, `hr`, and MathML elements;
      - `img` with `src`, `alt`, `width`, and `height`;
      - `a` with an `href` that is `http(s)`, `mailto`, or `#…`;
      - `id` for footnote anchors.
      - No scripts, event handlers, iframes, forms, styles, or `javascript:` URLs.
14. **Fidelity verdict.**
    - **Fail `thin`** when kept words < 300 **and** page visible words ≥ 400. The hint
      is to retry with `--html`, or that the page is not an article.
    - **Warn** when kept large media < page large media: "page showed N large images
      below the title; extraction kept M".
    - **Warn** when kept code blocks < page code blocks, or kept words < 40% of page
      words.
    - `status` is `ok` or `warn`.
15. **Image localization** (skipped on dry runs):
    - Download each kept `img` through `context.request.get` with the page URL as
      `Referer`, so cookies and CDN rules match. Limits: 15 s, 15 MiB per image, 60 MiB
      total.
    - Decode `data:` URIs.
    - Write to `workdir/assets/img-NNN.<ext>` by sniffed content type, and serialize
      tagged inline SVG to `assets/svg-NNN.svg`.
    - Rewrite `src` to the relative asset path. A failure drops the image, counts it in
      `images.failed`, and adds a warning.
    - Write `workdir/article.json`, holding the sanitized HTML, metadata, fidelity,
      image manifest, and source URL, then call
      `web_clip_render.render(article_json_path, out_pdf, launcher)`.
    - The renderer always launches a **separate headless** browser, because `page.pdf`
      needs headless.
16. **Timeouts.** The whole capture is bounded at 150 s. Clean up the browsers, Xvfb,
    and temporary profile in `finally` blocks.

`--self-test`:

- **Pure-Python checks** that always run:
  - date parsing;
  - challenge and login classification on recorded probe dicts;
  - metadata layering, including the OpenAI case: embed tweet date 2026-03-08 vs.
    visible "April 27, 2026" must give 2026-04-27;
  - text cleanup;
  - the `nh3` allowlist on `fixtures/hostile.html`;
  - the fidelity verdicts.
- **Browser-backed fixture checks**, which run only when a browser is discovered
  (otherwise print "skipped: no browser" and still pass). Serve `fixtures/*.html` at
  `https://fixture.bob-clip.invalid/…` via routing, fully offline:
  - `article_basic.html`: H1, byline, header `<time>`, more than 400 words, a `<pre>`, a
    `<picture>` with a dark-scheme first `<source>` and a light fallback (assert that
    the light variant wins), an embedded tweet blockquote with its own `<time>` and
    author (assert it is ignored), soft hyphens and U+2060 (assert stripped), and a
    `<details>`;
  - `challenge_cloudflare.html`: expect `blocked` in headless with headed disabled;
  - `login_wall.html`: expect `blocked`;
  - `hostile.html`: scripts, `onerror`, `javascript:` links, an iframe, a form, and an
    SVG with a script; assert all are removed.
- The self-test also runs the placeholder render to a temp PDF and checks that it starts
  with `%PDF`.

Acceptance:

- `just check-web-clip-adapter` passes on apollo (browser checks skipped) and on athena
  (browser checks run). On athena, copy `scripts/web_clip/` to a temp directory and run
  the self-test there over `ssh athena`.
- A manual adapter run on athena, with the OpenAI URL and a temp workdir, returns
  `ok: true` with `capture.mode` `headed-xvfb`, `published` 2026-04-27, both images
  resolving to `*-Light-*` URLs, and a placeholder PDF. The JSON request goes through
  `uv run --quiet --script web_clip_adapter.py`, wrapped in nothing; the adapter starts
  Xvfb itself.
- `just all` passes.

## Phase reader-template: Reader print template and renderer

Replace the placeholder `scripts/web_clip/web_clip_render.py` with the real renderer.
Add `scripts/web_clip/template/reader.css` and `scripts/web_clip/fonts/`:

- Fonts: latin and latin-ext `wght` woff2 files, normal and italic, for Source Serif 4
  and Inter, plus JetBrains Mono normal. Get them from the
  `@fontsource-variable/*@5.3.0` npm packages via `npm pack`.
- Licenses: `fonts/OFL.txt` and `fonts/README.md` with the versions and SHA-256 sums.

Register the new runtime files in `src/scripts.rs` `SUPPORT_ASSETS`. Keep the total font
payload under 1 MB.

Renderer behavior:

1. **Image normalization** (Pillow), into `workdir/render/assets/`:
   - Rasters: downscale to at most 1600 px on the long edge, flatten alpha onto white,
     and encode JPEG quality 82, progressive. Use the first frame of animated images.
   - Skip images under 64 px in both dimensions and count them as `skipped_small`.
   - SVGs pass through after removing `<script>`, `<foreignObject>`, `on*` attributes,
     and external `href`s.
   - Rewrite the article HTML `src`s to match.
2. **HTML assembly** in Python, escaping all metadata with `html.escape`:
   - a CSP meta tag
     `default-src 'none'; img-src https://bob-clip.invalid data:; font-src https://bob-clip.invalid; style-src https://bob-clip.invalid 'unsafe-inline'`;
   - `<title>` set to the article title, which becomes the PDF `/Title`;
   - the masthead, then the article body.
   - Shift heading levels so the article's highest heading renders as an `h2`.
3. **Offline print.**
   - Launch a headless browser and open a page routed so that
     `https://bob-clip.invalid/…` serves `workdir/render/` and the materialized `fonts/`
     and `template/`. Abort everything else.
   - Steps: `goto("https://bob-clip.invalid/article.html", wait_until="load")`, then
     `emulate_media(media="print")`, then await `document.fonts.ready` (capped at 10 s).
   - `page.pdf(path=out_pdf, format="Letter", print_background=True, outline=True, tagged=True, prefer_css_page_size=True, display_header_footer=False)`.
4. **Report** `images` counts and `pdf_bytes`. Warn above 50 MiB. Fail `render` above 95
   MiB, because vault sync refuses files of 95 MiB or more.

Template spec:

- **Page:** `@page { size: Letter; margin: 0.9in 1in 0.85in }`.
  - Footer margin boxes in Inter 7.5pt, gray `#6e7781`: `@bottom-left` shows the short
    title, injected as a CSS string and truncated to about 60 characters;
    `@bottom-right` shows `counter(page) " / " counter(pages)`.
  - `@page :first` drops the bottom-left title.
- **Body text:**
  - Source Serif 4 at 10.75pt, line-height 1.5, color `#1f2328`.
  - A text column of about 34em (about 70 characters per line), centered. Figures,
    tables, and `pre` may use the full content width.
  - `text-align: left`, `hyphens: manual`, `font-variant-ligatures: none`, `orphans: 3`,
    `widows: 3`, `print-color-adjust: exact`.
  - Fallback stacks reach system Noto CJK and emoji fonts.
- **Masthead** (page 1):
  - the site name in Inter 8.5pt, uppercase, letter-spaced, accent `#2b4c7e`;
  - the title in Source Serif 4 at 25pt, semibold, line-height 1.15;
  - a dek from the description, shown only if it is not a prefix of the first paragraph:
    12.5pt italic `#57606a`;
  - a byline row in Inter 9.5pt: `author · Month D, YYYY`, with missing parts omitted;
  - a provenance strip between hairlines, in Inter 8pt `#57606a`:
    `Source <url> · Captured <Month D, YYYY> · <N> words`. The URL is a link and wraps
    with `overflow-wrap: anywhere`.
- **Headings:** Inter semibold. `h2` at 14.5pt with 1.6em space above, `h3` at 12pt,
  `h4` at 10.5pt. All use `break-after: avoid`. No numbering.
- **Links:** `#2b4c7e`, no underline, with a 0.5pt bottom border tint.
- **Code:**
  - Inline: JetBrains Mono at 0.86em on `#f3f4f6`.
  - `pre`: JetBrains Mono 8.25pt/1.45 on `#f6f8fa`, with a 0.5pt `#e1e4e8` border, 4pt
    radius, `white-space: pre-wrap`, and `overflow-wrap: anywhere`.
- **Blockquotes:** 2pt `#d0d7de` left rule, color `#3b434b`.
- **Figures:** centered, `max-width: 100%`, `max-height: 7in`, `break-inside: avoid`.
  Captions in Inter 8.5pt gray.
- **Tables:** Inter 8.5pt, collapsed borders, zebra rows, repeating `thead`.
- **Math:** native MathML.
- **`hr`:** a short centered hairline.

Self-test additions:

- Render `fixtures/article_basic.html` through the full pipeline to a temp PDF.
- When `mutool` is on `PATH`, extract the text and assert:
  - the title is on page 1;
  - no U+00AD, U+2060, or U+200B;
  - no ligature code points (U+FB00–U+FB06);
  - no line ending in a hyphen that is absent from the source text, i.e. no automatic
    hyphenation.

Acceptance:

- On athena, run the adapter directly against the OpenAI URL and render pages 1–3 plus
  the two diagram pages with `mutool draw -r 80` to PNG. View them, and iterate until:
  - the masthead is clean;
  - both diagrams are the light variants and clearly visible;
  - the code block wraps;
  - footers read `n / N`;
  - there is no site navigation, cookie, or newsletter junk.
- Do the same check on one static blog, such as
  `https://lilianweng.github.io/posts/2023-06-23-agent/`.
- `just check-web-clip-adapter` and `just all` pass.

## Phase clip-command: bob highlights clip Rust command

Files:

- `src/native/highlights_ref/clip.rs`: the command, options, flow, and output.
- `src/native/highlights_ref/clip_url.rs`: validation, cleaning, the dedupe key, and the
  stem. Add the `url = "2"` crate to `Cargo.toml` for parsing.
- `src/native/highlights_ref/clip_adapter.rs`: the client and serde types for protocol
  v1. Model it on `src/native/gkeep/adapter.rs` (spawn, stdin write, piped stdout/stderr
  drained on threads, timeout and kill, JSON parse, protocol check), but keep it a
  focused client. Do not refactor gkeep.
- Wire `clip` into `cli.rs` (`build_cli`, alphabetical: `clip` comes before `create`)
  and the `mod.rs` dispatch.

Flow:

1. **Validate before doing anything else:**
   - parse the options;
   - validate and clean the URL;
   - validate `--published` (`YYYY-MM-DD`), `--name`, and `--ref-type`;
   - for `--html -`, read stdin into the workdir later; otherwise the file must exist.
2. **Dedupe:**
   - Walk `ref_dir/**/*.md` and read only the frontmatter (reuse `split_frontmatter`)
     for `source_url`.
   - Walk `xlib_dir/**/*.pdf` markers (reuse `read_pdf_marker`).
   - Compare dedupe keys:
     - a hit in a ref note → error "already captured as <ref path> (source_url …)" with
       a hint to open it or delete it to recapture;
     - a hit in xlib on the same target with `--force` → allowed;
     - any other xlib hit → refused.
3. **Fail-fast target check:** when the stem is known before capture (`--name`, a URL
   slug, or `--output`), run the shared planner and collision checks before launching
   anything.
4. **Workdir:**
   - Create a 0700 directory named `bob-clip-<pid>-<nanos>` under `TMPDIR` if that path
     is 40 bytes or shorter, else under `/tmp`.
   - Remove it on exit unless `BOB_WEB_CLIP_KEEP_WORKDIR=1`, in which case print its
     path.
5. **Call the adapter:**
   - Resolve it from `BOB_WEB_CLIP_ADAPTER`, or run
     `uv run --quiet --script <materialize_scripts()>/web_clip/web_clip_adapter.py`. If
     `uv` is missing, the error says to install uv.
   - Send `capture` with `captured` set to today's local date.
   - On a TTY, show a stderr status line ("capturing <host>…") while the adapter runs.
   - Map `ok: false` to `bob highlights: error: …` plus `hint: …`, with exit code 1 and
     no writes.
6. **Finalize the metadata:**
   - title: override, else the adapter title; for `kind: "pdf"`, the PDF Info `/Title`,
     then a humanized slug;
   - stem: `--name`, else the URL slug, else the snake*cased title, else
     `<host>*<YYYYMMDD>`;
   - `id` = stem.
   - Plan the target again with the shared planner, rechecking collisions just before
     install.
   - Compose the marker with extras `source_url`, `author`, `published`, and `captured`.
7. **Dry run:** print `would create Highlights-ready web PDF`, then:
   - `source_url`, `pdf`, `sidecar_guard`, `library_destination`;
   - each metadata field with its source (for example
     `published: 2026-04-27 (visible-date)`);
   - the capture mode, the fidelity line, and the warnings;
   - the full marker, and `writes: none`.
8. **Write run:**
   - Call `stamp_and_install(render.pdf → target)` with Info `/Title` and `/Author`.
   - Print the success block from the contract.
   - Print warnings after the block.
   - Print the next step via the shared `print_next_step`.

Doctor (`doctor.rs`) adds non-fatal rows after the pandoc row:

- `web clip uv`: available (path), or warn.
- `web clip adapter`: a `ping` with a 120 s timeout; on success show
  `ok (playwright …, defuddle …)`, otherwise warn with the error.
- `web clip browser`: `chrome <version> (<path>)`, or a warn with the install hint.
- `web clip headed fallback`: `xvfb (<path>)`, `display`, or `macos`, or "unavailable
  (bot-protected sites will fail closed)".

Tests:

- **Unit tests in `clip_url.rs`:**
  - validation accept and reject tables (`ftp://`, userinfo, `localhost`, `10.0.0.1`,
    `100.101.1.2`, `[::1]`, `foo.local`);
  - URL cleaning, the dedupe key, and stem derivation (the OpenAI URL, a date-prefixed
    Lil'Log path, `/blog/index.html`, a numeric-only path, `--name` validation).
- **CLI tests in `tests/cli/highlights/clip.rs`.** Use a fake adapter: a small script
  written by the test that reads the request, saves it to a file for assertions, copies
  a tiny `lopdf`-generated fixture PDF to `out_pdf`, and prints a canned response.
  Cases:
  - success: target path, marker keys via `bob highlights marker`, output lines, Info
    `/Title`;
  - default `ref_type` `blogs`, plus `-t`, `-o`, and `-N`;
  - overrides forwarded to the adapter;
  - `--html` path forwarding and `-` stdin;
  - `dry_run` forwarded, with no writes and `writes: none`;
  - `blocked` and `thin` errors: exit 1, hint printed, nothing written;
  - a library destination collision refused before the adapter runs, asserted by the
    absence of the request file;
  - dedupe against a ref note refused before the adapter runs;
  - `--force` overwriting the same intake target;
  - `--output` conflicting with `--ref-type`/`--name`;
  - the `kind: "pdf"` title fallback;
  - a protocol error from non-JSON stdout;
  - a scan round trip: clip into a temp vault, then
    `bob highlights --no-hooks scan -b <tmp>` creates `ref/blogs/<stem>.md` with `type`,
    `ref_type: blogs`, `source_url`, `author`, `published`, `captured`, `id`, `title`,
    and the `^ref` task line, and a second scan is a no-op.
- **Help tests in `tests/cli/help_options.rs`:**
  - `highlights_ref_help_lists_subcommands_alphabetically` now starts with `clip`;
  - a new `highlights_clip_help_lists_options_alphabetically` covers the contract's
    order, the URL positional, and the after-help notes.
- **justfile `install-smoke`:** add
  `"${root}/bin/bob" highlights clip --help >/dev/null`.

Docs:

- New `docs/highlights-clip.md`:
  - usage and examples;
  - the pipeline;
  - hosts and browsers: athena can clear bot challenges headed via Xvfb; apollo has no
    browser; the Mac uses Chrome;
  - the `--html` workflow (Save Page As, or SingleFile);
  - the environment variables, the marker fields, dedupe, failure kinds with fixes, and
    the protocol summary.
- `README.md` Highlights section: add a usage line, a bullet, and the dependencies
  (`uv`, Google Chrome or `BOB_CHROME`, and Xvfb for bot-protected sites on Linux).
- `docs/highlights-ref-sync.md`: cross-link the new doc next to `create`.

Acceptance: `just all` passes, and `cargo package --list` includes the new files.

## Phase live-verify: Live OpenAI capture verification and docs finish

This is the acceptance gate Bryan asked for: a real reference PDF from
`https://openai.com/index/open-source-codex-orchestration-symphony/`. Fix whatever it
uncovers, in any file, and keep `just all` and `just check-web-clip-adapter` green.

1. **Build and copy.**
   - Run `cargo build --release`.
   - Copy `target/release/bob` to athena with `scp` into a fresh temp directory (for
     example `/tmp/bob-clip-verify/bob`). Do not install it over athena's
     `~/.cargo/bin/bob`.
   - The materialized script cache is keyed by an asset hash, so this does not disturb
     the installed bob.
2. **Doctor on athena:** `/tmp/bob-clip-verify/bob highlights doctor` shows the uv,
   adapter, browser (chrome 154), and headed fallback (`xvfb`) rows as OK.
3. **Dry run on athena:**
   `… highlights clip https://openai.com/index/open-source-codex-orchestration-symphony/ -d`
   must report:
   - target `~/bob/xlib/blogs/open_source_codex_orchestration_symphony.pdf`;
   - title `An open-source spec for Codex orchestration: Symphony`;
   - author `Alex Kotliarskyi, Victor Zhu, and Zach Brock`;
   - published `2026-04-27` from the visible date;
   - capture `headed-xvfb` after a headless challenge;
   - fidelity `ok` with images 2/2 and code blocks 1/1.
4. **Scratch-vault capture on athena:** `-b /tmp/bob-clip-verify/vault` (empty). Then
   check:
   - `bob highlights marker <pdf>` shows every provenance key;
   - `mutool draw -r 80` on pages 1–3 and both diagram pages, copied back and
     **viewed**, shows the masthead, the light diagrams clearly visible, a wrapped code
     block, `n / N` footers, and no navigation or junk;
   - a `mutool draw -F txt` text-layer check finds no U+00AD, U+2060, or ligature code
     points;
   - the size is at most 6 MB and the page count is plausible (about 20–35).
   - Then run `… highlights --no-hooks scan -b /tmp/bob-clip-verify/vault` and confirm
     `ref/blogs/open_source_codex_orchestration_symphony.md` has `type: "[[ref]]"`,
     `ref_type: blogs`, `title`, `id`, `source_url`, `author`, `published`, `captured`,
     and the `^ref` lifecycle task line. A second scan must change nothing. This is the
     only scan you run, and only against the scratch vault.
5. **Real capture on athena:** run the same command without `-b`. The PDF lands in
   athena's `~/bob/xlib/blogs/`, the intake queue the Mac's `bob_xlib_pull` drains.
   Re-running must refuse because the target exists (use `--force` only to deliberately
   redo the intake copy).
6. **Static-site check:** on athena, into the scratch vault, a static blog such as
   `https://lilianweng.github.io/posts/2023-06-23-agent/` succeeds headless with no
   headed fallback, and its images and math look right.
7. **apollo fails closed:** run the release binary in the workspace with the OpenAI URL.
   It must exit 1 with the `browser` (no Chrome) or `blocked` error and its hint, and
   must write nothing to `~/bob/xlib/`.
8. **Ref note in the vault:**
   - If `ssh -o ConnectTimeout=10 mac true` succeeds, run
     `ssh mac 'PATH="$HOME/bin:$HOME/.cargo/bin:/opt/homebrew/bin:/usr/bin:/bin" bob_xlib_pull'`.
     It is the same drain-and-scan the Mac's 15-minute cron runs.
   - Then wait, with a foreground poll of at most about 10 minutes, for vault git sync
     to bring `~/bob/ref/blogs/open_source_codex_orchestration_symphony.md` to athena or
     apollo, and check its frontmatter.
   - If the Mac is offline, report that the next Mac scan will create the note from
     athena's queue.
9. **Docs:** add a "Verified" note to `docs/highlights-clip.md` with the date, the
   hosts, the capture modes, and the OpenAI result, and correct anything the live run
   disproved.
10. **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on the phase bead. Do not create
    beads.
    - Bryan's manual Mac Highlights acceptance: highlight about 10 passages across line
      breaks, links, inline code, and captions, then compare the sidecar and ref-note
      quotes. Only if they break, swap the render stage for Defuddle Markdown → image
      normalization → the existing pandoc path.
    - Install the updated bob on athena, apollo, and the Mac after landing.
    - A `--from-chrome` Mac capture and a Bob Mac Capture or Shortcuts entry point that
      spawns `bob highlights clip`.
    - `--mode page`, image rescue for dropped figures, and `--keep-source DIR`.
    - Recapture and versioning.
    - Anything else the live run surfaced.

Report the absolute PDF path on athena, the dry-run and success output, the viewed page
findings, and whether the ref note has appeared.

## Out of scope

- Writing `ref/` notes from clip, or running `scan` against the real vault on athena or
  apollo.
- Printing the live page.
- Stealth plugins, user-agent spoofing, or reusing Bryan's Chrome profile or cookies.
- Hosted readers (Jina) and Cloudflare Browser Run.
- `wkhtmltopdf` and WeasyPrint.
- Installing Chrome or Xvfb on apollo.
- Storing extracted Markdown in the vault.
- `--from-chrome`, `--mode page`, `--keep-source`, image rescue, and `--scan`.
- A pandoc fallback renderer, unless the Mac quote check fails.
- Any change to `create`'s output.

## Risks

- **Bot walls drift.** Headed Xvfb capture works today on athena's residential IP. If it
  stops, `--html` from a real browser is the durable path. Treat live sites as smoke
  tests only; CI uses fixtures.
- **Highlights text quality on Chrome-printed PDFs is unproven.** Mitigations: bundled
  fonts, no hyphenation, and no ligatures. Bryan's manual check is the gate, and only
  the render stage would change.
- **Running a browser on untrusted pages.** Mitigations: sandbox on, a temporary
  profile, the private-network guard, the `nh3` allowlist, an offline render page with
  CSP, and caps on time, bytes, and size.
- **`--disable-blink-features=AutomationControlled` is the single automation-related
  switch.** It exists only because openai.com tears down its own DOM when
  `navigator.webdriver` is true. The richest-snapshot rule keeps capture correct even if
  the switch stops helping.
