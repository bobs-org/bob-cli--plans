---
tier: epic
title: "Several URLs, one reference: multi-URL bob ref create collections"
goal: "`bob ref create URL URL…` merges 2 to 20 web articles, in argument order, into
  one Highlights-ready PDF and one reference. The PDF has a contents page, one bookmark
  per article, and continuous page numbers. The name comes from `-N` or from the words
  every URL's slug shares. The command is all-or-nothing and fails fast, before any
  network call, whenever no trustworthy name exists.

  "
decisions:
  output_dir:
    ask:
      Also add a curl-style `-O, --output-directory DIR` option now, alongside the
      unchanged `-o, --output PDF`?
    choices:
      defer:
        No directory option; `-N` plus `-t` already set the name and the intake folder
        together
      add:
        The gate phase adds `-O DIR` (writes DIR/<name>.pdf), combinable with -N,
        conflicting with -o/-t
    default: defer
    why:
      All six recorded -o uses were intake paths that -N/-t express; repurposing -o
      breaks `-o x.pdf`
    answer: defer
  decision_record:
    ask:
      Record this design as a new `decisions` memory strand when the docs phase lands?
    memory:
      - decisions:ref-create-merges-several-urls
    default: false
    answer: false
phases:
  - id: dedupe
    title: List-aware import dedupe for source_url and url lists
    depends_on: []
    size: small
    description:
      "dedupe: record one dedupe source per item of a list-valued source_url/url in ref
      notes and intake markers, so create, ingest, capture routing, and find -i see
      every URL of a list; delete the documented gap in docs/ref.md."
  - id: naming
    title: Pure collection naming and metadata rules
    depends_on: []
    size: medium
    description:
      "naming: add a network-free module implementing the slug-word naming rule with its
      no-name reasons, the name waterfall, the part-order warning, and the collection
      title/author/published derivation, all table-tested."
  - id: adapter
    title: Adapter render:false capture, compose op, and the collection layout
    depends_on: []
    size: medium
    description:
      "adapter: let capture skip printing, add a compose op that typesets ordered
      article.json parts into one PDF (opening page, linked contents, a fresh page and
      bookmark per part, namespaced ids and assets), plus the Rust client types;
      single-capture requests stay byte-identical."
  - id: gate
    title: Variadic TARGET, fail-fast validation, and collection overlap policy
    depends_on:
      - dedupe
      - naming
    size: medium
    description:
      "gate: make TARGET variadic, route two or more targets into a new collection
      module that runs every pre-network check (kinds, -H, duplicates, cap, name
      waterfall, overlap, target plan) with polished errors, and rewrite the -N/TARGET
      help; the route itself is a stub until the next phase."
  - id: route
    title: Collection route - classify, capture, compose, stamp, install, report
    depends_on:
      - gate
      - adapter
    size: medium
    description:
      "route: replace the stub with the all-or-nothing pipeline (per-part
      classification, sequential render:false captures, metadata derivation, compose,
      list-valued marker, stamp, guarded install), dry-run and success reports, explicit
      --audio, and fake-adapter integration tests."
  - id: listen
    title: Narrate and attach collections with --listen
    depends_on:
      - route
    size: small
    description:
      "listen: support -L for collections by narrating the staged collection PDF before
      the guarded install, and attach a new episode when the exact same URL set is
      already captured or queued."
  - id: docs
    title: Docs, live verification, and follow-ups
    depends_on:
      - listen
    size: small
    description:
      "docs: document several-URL collections in the create, clip, and ref docs, run a
      live end-to-end check in a scratch vault on athena, and record follow-ups (plus
      the optional decisions strand)."
proposed_by: bbugyi200.athena.10k
decided_by: reviewer
decided_via: tui
create_time: 2026-10-15 12:30:27
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_create_multi_url_collections.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_create_multi_url_collections.md)

# Plan: Several URLs, one reference

## Context

Bryan wants `bob ref create` to accept several URLs whose articles merge into **one
reference and one PDF**. He also endorsed every requirement adjustment in the research
report
`research:202610/ref_create_multi_url_collections/ref_create_multi_url_collections.md`
(read it with `sase artifact read` before starting any phase; its section names are
cited below). Those adjustments change three of the original bullets:

- **Name flag.** Do not add `-n|--name`. `-n` is already `--no-audio`, so taking it
  would make `-n` silently swallow the next argument. **`-N, --name` already exists.**
  It already names the PDF, the marker `id`, and the `ref/<type>/<name>.md` note that
  `scan` writes. It has never been used because its help says only "Output filename
  stem". Rewrite that help; don't add a new flag.
- **Output flag.** Keep `-o, --output PDF` as it is. Turning it into a directory would
  make `-o x.pdf` create a directory named `x.pdf/`. Every recorded `-o` use was an
  intake path, which `-N` (plus `-t`) expresses more safely. A curl-style `-O` stays
  optional; see decision `output_dir`.
- **"Name required iff no long, useful common prefix."** A character prefix doesn't work
  (see [Why words, not characters](#why-words-not-characters)). The rule below is the
  precise, testable form of that requirement. `-N` is always allowed and always wins.

Why the feature is worth building: on 2026-10-11 Bryan captured Stripe's two-part
"Minions" series as two references a minute apart, then linked part 2 to part 1 by hand.
That is two PDFs, two `^ref` tasks, and a dependency edge for one piece of reading. 23
of 635 external ref notes already bundle several URLs by hand. `-L` appears in 10 of 28
recorded `create` runs, so a merged reference must narrate as one episode.

Reading the code against bob-cli `ba71710` confirmed the research's seams:

- `create.rs::run`/`create_pdf` is the single-target dispatch.
- `clip.rs::capture_article` is the single-article pipeline to mirror.
- `clip_url.rs` already has `validate_and_clean`, `stem_from_url`/`stem_from_segment`,
  `snake_case`, `validate_name`, and `humanize_stem`.
- `sources.rs` reads only scalar `source_url`/`url`.
- `stamp.rs::DefaultTargetIdentity::Url { keys: Vec<_> }` already takes several keys.
- `compose_marker` already accepts `MarkerValue::List`.
- `arxiv.rs::author_display` is the existing "A and B / A, B, and C / A et al."
  formatter.
- In the adapter, `run_capture` writes `article.json` and then calls
  `web_clip_render.render()`; that seam is where printing gets split off.

## The contract every phase builds toward

### Usage and help

```text
Usage: bob ref create [OPTIONS] <TARGET>...

  <TARGET>...         Markdown file, PDF file, PDF URL, arXiv paper URL, or web article URL;
                      two or more web article URLs become one reference
  -N, --name <NAME>   Reference name: the PDF filename and the ref note `bob ref scan` writes
                      [default: derived from TARGET; for several URLs, the words their slugs share]
  -o, --output <PDF>  (unchanged) with several URLs, its filename is the name
```

- **Parsing.** `TARGET` takes `num_args(1..)` with normal option parsing, so options can
  come before or after the URLs. Targets stay `OsString`. There is no trailing-var-arg
  mode and no comma splitting.
- **Help.** No option is added or renamed (unless `output_dir = add`), so the
  alphabetical help test keeps its order. `after_help` gains a **Several URLs**
  paragraph: the naming sentence below, all-or-nothing, the 20-URL cap, web articles
  only, and "merge material you'll read as one unit; capture an ongoing series part by
  part under the same `-P`". It also gains two examples:
  - `bob ref create https://example.com/2024/01/tokio-internals-part-1 https://example.com/2024/02/tokio-internals-part-2 -L -P sase`
  - `bob ref create URL1 URL2 URL3 -N spotify_bg_agents -P sase`
- **One target is untouched.** It behaves byte-for-byte as today: same output, same
  errors, and the same adapter request JSON.

### Naming a collection: the precise "iff"

**The name waterfall** runs before any network access:

1. `-N`.
2. The `-o` filename stem.
3. The slug-word rule below.
4. `snake_case(-T)`, if non-empty.
5. Otherwise, exit 2 with the error under [Messages](#messages).

This mirrors the single-URL order (`-N` → URL slug → title). Whatever wins then goes
through `validate_name` and the usual `_2`/`_3` collision walk.

**Help sentence (verbatim intent):** With several URLs and no `-N`, bob names the
reference after the words every URL's slug starts with, or failing that the folder they
all share, ignoring dates, ids, part numbers, and filler. It needs two meaningful words,
or one when the pages differ only by a part or page number. Otherwise pass `-N` (or
`-T`).

#### Why words, not characters

A character prefix of real URLs is dominated by the scheme, host, and date folders, and
it cuts words in half. Spotify's three parts give
`https_engineering_atspotify_com_2025_1`; `cats`/`cars` give `ca`. Python has
`os.path.commonpath` for exactly this reason. Shared _folders_ are mostly site structure
(`docs`, `core`, `configuration`, `pdf`).

The research tested five candidate rules on 51 labeled sets and on 2,987 same-host pairs
of unrelated refs from the live library. This rule was the only one with zero bad names.
It names real series (Stripe, Fowler's "Exploring Gen AI") and stays silent almost
everywhere else. A false "pass `-N`" costs ten keystrokes. A bad name sticks in `lib/`,
`ref/`, and every wikilink, and there is no `bob ref rename`. **So the gate is strict,
and implementers must not loosen it without re-running a corpus.**

#### The rule

The rule is a pure function of the cleaned URLs, in order:

1. **Same site.** Every URL's `WebUrl::host` (lowercased, without `www.`) must be equal.
   Otherwise: no name, reason `different_sites`.
2. **One slug per URL.** Use the segment `stem_from_url` picks today: the last segment
   that isn't noise (`SKIPPED_SEGMENTS`, 2 characters or fewer, all digits, a date),
   with the page extension and any `YYYY-MM-DD-` prefix removed. Additionally:
   - skip `@handle` and `~user` segments;
   - when every URL has the same path and they differ only in the query, the slug counts
     only if every differing parameter is `page`, `part`, or `p` (so `essay?page=2` has
     a slug and `item?id=…` does not).

   If any URL has no slug: no name, reason `no_slug(index)`. Don't change
   `stem_from_url` itself; single-URL stems must not move.

3. **Shared slug start.** Split each slug into lowercase `[a-z0-9]+` words and take the
   longest word prefix they all share. Trim words from both ends that are:
   - filler: `a an and at by for from in of on or the to vs with`;
   - series words:
     `part pt chapter ch section episode ep lesson vol volume no series page`;
   - number words: all digits;
   - hex-id words: 6 or more characters of `[0-9a-f]`, at least one of them a digit.

   Interior words stay (`end_to_end`, `building_a_raft`).

4. **Gate.** Words that are not filler, series, number, or hex-id words are
   _meaningful_. Accept when there are **at least two meaningful words and at least one
   is not generic**. The generic list is:

   `about api basics book config configuration core doc docs documentation faq getting guide guides handbook help home index install installation intro introduction latest main manual notes overview quickstart readme reference setup started tutorial tutorials usage user wiki`

   Also accept **exactly one remaining word of 4 or more letters that is not generic**,
   but only when the URLs differ only by a number. That means their dedupe keys become
   identical once every maximal digit run is replaced by `#`
   (`raft-part-1`/`raft-part-2`, `essay?page=1`/`?page=2`).

5. **Folder fallback.** If step 4 fails and every URL has the same folder (the path
   minus its last segment), apply the step 2 segment cleaning and steps 3–4 to that
   folder's last segment, without the one-word branch. Multi-word folders are series
   titles (`/exploring-gen-ai/`). One-word folders are site structure and never qualify.
6. **Finish.** Join the words with `_` through `snake_case` (80-character cap), then
   apply `validate_name`. Return the name and its source (`slug_words` or `folder`), or
   a no-name reason. When the rule failed but something was shared, the reason is
   `too_thin(shared)` and carries the shared text; when nothing was shared, it is
   `nothing_shared`.

**Part-order warning.** If every URL carries a series number (the first number word
after the shared start, or a `page`/`part`/`p` query value), and those numbers aren't
non-decreasing in argument order, warn:
`warning: the URLs are in part order 2, 1; the PDF follows argument order`. Never
re-sort.

**Required naming table** (unit tests; host `example.com` unless shown):

| URLs                                                                                                                         | Result                                              |
| ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `/2024/01/tokio-internals-part-1`, `/2024/02/tokio-internals-part-2`                                                         | `tokio_internals` (slug words)                      |
| `stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents`, same `-part-2`                                          | `minions_stripes_one_shot_end_to_end_coding_agents` |
| `medium.com/@alice/building-a-raft-kv-store-part-1-abc123def456`, `…-part-2-0fe1d2c3b4a5`                                    | `building_a_raft_kv_store`                          |
| `/articles/exploring-gen-ai/01-intro.html`, `/articles/exploring-gen-ai/02-prompting.html`                                   | `exploring_gen_ai` (folder)                         |
| `/raft-part-1`, `/raft-part-2`                                                                                               | `raft` (one-word series)                            |
| `/essay?page=1`, `/essay?page=2`                                                                                             | `essay` (one-word series)                           |
| `news.ycombinator.com/item?id=1`, `…/item?id=2`                                                                              | none: `no_slug(1)`                                  |
| Spotify's three parts (`spotifys-background-coding-agent-part-1`, `context-engineering-…-part-2`, `feedback-loops-…-part-3`) | none: `nothing_shared`                              |
| `/core/configuration`, `/core/usage`                                                                                         | none: `too_thin("core")`                            |
| `/getting-started-1`, `/getting-started-2`                                                                                   | none: `too_thin("getting started")`                 |
| `/introduction-to-x`, `/introduction-to-y`                                                                                   | none: `too_thin("introduction")`                    |
| `a.example.com/x-y-z`, `b.example.org/x-y-z`                                                                                 | none: `different_sites`                             |
| `/tokio-internals-part-2`, `/tokio-internals-part-1`                                                                         | `tokio_internals` plus the part-order warning       |

Also add permutation tests: the name must not depend on URL order, and the
`www.`/case/trailing-slash spellings of one URL must give the same result.

#### Messages

When no name is found, exit **2** with the same `error:` + `hint:` shape as the
missing-`-P` error. Nothing has touched the network at this point. Show URLs without
their scheme and keep the index column dim:

```text
bob ref create: error: these 3 URLs need a reference name
  1  engineering.atspotify.com/2025/11/spotifys-background-coding-agent-part-1
  2  engineering.atspotify.com/2025/11/context-engineering-background-coding-agents-part-2
  3  engineering.atspotify.com/2025/12/feedback-loops-background-coding-agents-part-3
hint: their slugs share no common start; pass -N <name> (or -T <title>)
```

The hint depends on the reason:

| Reason            | Hint                                            |
| ----------------- | ----------------------------------------------- |
| `different_sites` | `they come from different sites (a.com, b.org)` |
| `no_slug(i)`      | `URL i has no slug to name from`                |
| `too_thin(s)`     | `they only share "s"`                           |
| `nothing_shared`  | `their slugs share no common start`             |

Each hint ends with `; pass -N <name> (or -T <title>)`.

A resolved name is always printed with its source, in normal and dry-run output, and
never asks for confirmation:

| Source     | Line                                                                 |
| ---------- | -------------------------------------------------------------------- |
| `-N`       | `name: spotify_bg_agents (from -N)`                                  |
| `-o`       | `name: tokio (from the -o filename)`                                 |
| slug words | `name: tokio_internals (words every URL starts with; -N overrides)`  |
| folder     | `name: exploring_gen_ai (the folder every URL shares; -N overrides)` |
| `-T`       | `name: tokio_deep_dive (from -T; -N overrides)`                      |

### Fail-fast order

Nothing touches the network before step 8. Steps 2–7 are all local. Today's `run()`
already validates `-N`/`-t`/`-p` and resolves `-P` before dispatching, and it may keep
doing so. The order below only decides which error wins when several apply, so tests
should pin one error per invocation.

1. Two or more targets select the collection route. One target takes today's path,
   unchanged.
2. Every target looks like an http(s) URL and none is syntactically arXiv. `-H` is
   absent. Errors (exit 2), each with a `hint:` that suggests capturing that target on
   its own:
   - `TARGET 2 is a local file (notes.md); several targets must all be web article URLs`
   - `URL 2 is an arXiv paper; merging supports web articles only`
   - `--html replays one saved page; several URLs are captured live`
3. Clean every URL. Refuse duplicate dedupe keys: `URL 1 and URL 3 are the same page`
   (exit 2). Refuse more than **20** URLs:
   `too many URLs (23); one reference takes at most 20` (exit 2). The 20 cap reflects
   the 95 MiB vault limit and adapter timeouts, and it catches accidental `$(cat …)`
   lists. Known gap: `%40` and `@` spellings of the same path are not yet the same key;
   that bug is tracked as `bob-cli-7u` and is out of scope.
4. Resolve `-P` and validate `-N`, `-t`, and `-p`, as today.
5. Run the name waterfall, or fail as in [Messages](#messages).
6. Run the overlap check below on every part. Then plan the default target with
   `DefaultTargetIdentity::Url { keys: all part keys }` (or the guarded exact `-o`
   target), then re-check overlap against the planned target (the same two-pass shape as
   `clip.rs`).
7. Run the listen preflight when `-L` is set (it is local).
8. Classify each URL with the existing `fetch_and_route`. A part that is a PDF fails
   with `part 2/3 is a PDF (<url>); merging supports web articles only`. A non-2xx
   response keeps today's message, prefixed with `part 2/3`.
9. Capture parts sequentially, compose, stamp once, and install.

### Overlap policy (one URL, one reference)

Group the recorded sources by path, so each recorded note or intake PDF has its full key
set. Compare that set with the collection's key set:

| Situation                                                          | Behaviour                                                                                                                              |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| A part is in a PDF-backed ref note whose key set differs from ours | Refuse (exit 1); list every overlapping part, see below                                                                                |
| A part is in a queued intake PDF whose key set differs from ours   | Refuse the same way; `--force` doesn't override partial overlap                                                                        |
| A PDF-backed note has exactly our key set (any order)              | Today's `already captured as …` refusal with the `--listen` attach hint; with `-L`, attach (listen phase)                              |
| An intake PDF has exactly our key set                              | Today's `already queued in …` refusal; `--force` overwrites only when the planned target is that PDF; with `-L`, attach (listen phase) |
| A part appears only in a legacy note without a PDF                 | Warn once per note and continue; dry run lists it as `legacy: … (superseded by this capture)`                                          |

Partial overlap error:

```text
bob ref create: error: 2 of 3 URLs are already in the library
  1  stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents   captured as ref/blogs/minions_stripes_one_shot_end_to_end_coding_agents.md
  2  stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2   captured as ref/blogs/minions_stripes_one_shot_end_to_end_coding_agents_part_2.md
hint: a URL belongs to one reference; remove those references first, or capture the other URLs on their own
```

Queued parts read `queued in xlib/…pdf`. Refusing is deliberate. Creating the collection
anyway would give both notes `duplicate_identity` in the library and break the
one-URL-one-reference invariant that `find`, routing, and capture rely on. Absorbing
existing refs (`--absorb`) is v2 (see `bob-cli-39` for the recapture/versioning design
space).

### Metadata, derived after capture (never blocks)

- **Title.** Use `-T` if given. Otherwise use the longest prefix shared by every part
  title:
  - cut it at a word boundary, treating whitespace, `—`, `–`, `:`, `|`, `(`, and `,` as
    boundaries;
  - then repeatedly trim trailing separators and trailing ordinals (`Part`, `Pt`,
    `Chapter`, `Ch`, `Episode`, `Ep`, `Lesson`, `Vol`, `Volume`, `No`, numbers, `#N`,
    and the Roman numerals I–XX);
  - require at least two meaningful words.

  Failing that, use `humanize_stem(name)`. Never use part 1's title for the whole
  collection. Examples:
  - The Stripe titles "Minions: Stripe’s one-shot, end-to-end coding agents" and the
    same title plus "—Part 2" (an em dash with no spaces) give the first title.
  - "Tokio Internals, Part 1: The Scheduler" and "Tokio Internals, Part 2: The Reactor"
    give "Tokio Internals".

- **Author.** Use `-A` if given. Otherwise use the single value when every part that has
  an author agrees (compared trimmed and case-insensitively). Otherwise pass the
  distinct authors, in part order, to `author_display` (move it from `arxiv.rs` to a
  shared spot). If no part has an author, there is none.
- **Published.** Use `-p` if given, otherwise the earliest part date. Report it as
  `published: 2024-01-08 (first part)`. Each part keeps its own date on its own page.
- **Captured.** Today.
- **Marker.** It gains no new keys. `source_url` becomes an inline list of the cleaned
  URLs in argument order (it stays a scalar for one URL). `id` is the final name. URLs
  containing `,`, `[`, or `]` are already forced into JSON quoting. Add a round-trip
  test anyway, plus one proving that `scan` writes the list into the note's frontmatter.

### The document

**Merge at the HTML level and print once.** Concatenating separately printed PDFs would
restart page counters and repeat mastheads. Build on `reader.css` (Source Serif 4,
Inter, JetBrains Mono); don't fork the look.

- **Opening page.**
  - It contains, in order:
    - the site kicker, when every part shares a site;
    - the collection title, styled like `.masthead-title`;
    - the byline;
    - a muted meta line:
      `2 articles · 6,420 words · ~28 min read · captured October 15, 2026`;
    - a **Contents** label;
    - a numbered list where each entry links to `#part-N`, shows the part title, and has
      a muted `site · author · date · ~N min` line.
  - It uses **no heading elements**, so the PDF outline is exactly one top-level
    bookmark per part.
  - Reading time is 230 words per minute, rounded up, with a minimum of 1.
  - Chromium has no `target-counter()`, so the contents have links but no page numbers
    in v1.
  - The Highlights marker stays on page 1, as today.
- **Each part.** A `<section class="part" id="part-N">` with `break-before: page`, so
  each part starts on a fresh page with no blank leaves. Inside it:
  - a small-caps kicker, `Part 2 of 3`;
  - the existing masthead (site, `h1` title, dek, byline, and provenance with the source
    link and word count);
  - the body through the existing `shift_headings`.

  Part titles become top-level bookmarks with their sections nested underneath.

- **Footer.** The left footer shows the collection's short title, and the opening page
  suppresses it the same way the single-article template does. The right footer keeps
  `counter(page) / counter(pages)` across the whole volume. Add per-part named-page
  footers (`<short title> · Part N`) only if a prototype print under the pinned
  Playwright proves the template's margin-box cascade quirk is handled. Otherwise ship
  the collection-title footer.
- **Isolation.**
  - Prefix every `id` and same-document `#fragment` in part N with `pN-`. This covers
    `href="#…"`, `id`, `xlink:href`, and SVG `url(#…)` references. Two parts can both
    have `#introduction` and `#fn1`.
  - Resolve each part's relative links against that part's own final URL.
  - Rename each part's assets with a `pN-` prefix (`p2-img-001.jpg`). The offline route
    handler forbids `/` in asset names, so use a prefix, not a subdirectory.
- **Budgets.** Each part keeps the per-capture image budget. The combined printed PDF
  keeps the 50 MiB warning and the 95 MiB refusal.
- **PDF Info.** The collection title and author, set by Rust's stamp as today.

### Output

Successful runs print to stdout through `Styler`. Progress goes to stderr, on a TTY
only: `capturing 2/3 engineering.atspotify.com…`, then `composing 3 articles…`.

```text
ok created Highlights-ready PDF from 2 web articles
name: tokio_internals (words every URL starts with; -N overrides)
title: Tokio Internals (shared by every part title; -T overrides)
parts:
  1/2  Tokio Internals, Part 1: The Scheduler   example.com · 2,140 words · fidelity ok
  2/2  Tokio Internals, Part 2: The Reactor     example.com · 1,980 words · fidelity ok
pdf: ~/bob/xlib/blogs/tokio_internals.pdf
author: Jane Doe
published: 2024-01-08 (first part)
captured: 2026-10-15
status: ready
parent: sase
pages: 14 · images: 9/9 · size: 2.1 MB
next: bob ref scan
```

- The part rows dim their metadata line.
- Any `renamed:` line from the collision walk follows the `ok` line.
- Adapter warnings print as `warning: part 2/2: …`.
- Failures name the part and promise no writes:
  `bob ref create: error: part 2/3 (engineering.atspotify.com/…): <adapter message>`,
  then any adapter hint, then `nothing was written`.
- `--dry-run` keeps today's meaning: the pages load in the browser, but localization,
  compose, and install are skipped. It prints `would create …`, the name with its
  source, the parts, the destination and `library_destination`, any `legacy:` lines, the
  planned marker, and `writes: none`.

### Out of scope for v1

These are recorded as follow-ups in the docs phase, not built:

- PDF, arXiv, or local-file parts (they would need PDF stitching);
- `-H` with several URLs;
- `--absorb` for existing refs;
- contents page numbers (two-pass render);
- a Bob Mac Capture "Sources ▸" submenu (today its Open/Copy actions use `urls.first`,
  which is acceptable);
- capture-side or ref-jobs bundling gestures;
- the `%40` dedupe bug (`bob-cli-7u`).

## Phase dedupe: list-aware import dedupe

Fix the import-dedupe gap on its own first, because a merged reference is otherwise
invisible to `create`, the ingest queue, and `bob capture` routing.

- `sources.rs::collect_ref_note_sources` reads `source_url`/`url` with
  `value.as_string()`, which drops lists. Push one `RecordedSource` per item instead,
  covering every frontmatter list form `bob ref find` indexes:
  - inline `[a, b]` lists;
  - block `- a` lists. If the entry parser here doesn't see block lists, match what
    `ref_library`'s `front.get_all` reads.
- Do the same in `collect_intake_sources` (marker `MarkerValue::List`) and in
  `collect_intake_records_inner` (`find -i`: one record per distinct URL, keeping the
  existing dedupe of identical values).
- Update the module doc comment. Delete the "Import deduplication currently differs…"
  paragraph from `docs/ref.md` § Identity, and state there that lists are honored
  everywhere.
- **Tests.** Unit tests in `sources.rs`:
  - a list note (inline and block) yields N sources;
  - a list intake marker yields N.

  CLI tests:
  - `bob ref create <URL from a PDF-backed list note>` refuses with `already captured`;
  - a URL from a legacy list note warns and proceeds (fake adapter);
  - `bob capture --dry-run` / url_routing reports `in_library` for a PDF-backed list
    note;
  - `bob ref find -i` sees a list intake marker.

- Nothing else changes, and single-URL behavior is otherwise identical.

## Phase naming: pure collection naming and metadata

Add `src/native/highlights_ref/collection_name.rs` (and, if it reads better,
`collection_meta.rs`). Both are pure, with no I/O and no network access.

- `derive_collection_name(urls: &[WebUrl]) -> NameOutcome`: implements
  [the rule](#the-rule) and returns either `Derived { name, source }` or
  `NoName(reason)`, with the four reasons above.
- `resolve_collection_name(name_flag, output_stem, title_flag, urls) -> Result<ResolvedName>`:
  the waterfall. `ResolvedName { name, source }` carries the five sources and their
  exact report phrases.
- `name_error_message(urls, reason) -> String`: renders the [Messages](#messages) block
  (the error line, the numbered scheme-less URL lines, and the `hint:`) so the gate
  phase only prints it. Keep any ANSI styling out of the pure output, or behind a
  `Styler` parameter, so tests compare plain text.
- `part_order_warning(urls) -> Option<String>`.
- `collection_title(part_titles, title_flag, name)`,
  `collection_author(part_authors, author_flag)`, and
  `collection_published(part_dates, published_flag)`, implementing
  [Metadata](#metadata-derived-after-capture-never-blocks). Move `author_display` to a
  shared location and keep `arxiv.rs` calling it.
- Reuse `stem_from_segment`, `snake_case`, `validate_name`, `percent_decode`, and
  `humanize_stem` from `clip_url.rs`. Widen visibility as needed, without changing their
  behavior.
- **Tests:**
  - every row of the naming table;
  - the permutation and spelling invariants;
  - every hint string;
  - every report phrase;
  - the title examples (the Stripe em dash, Tokio, the no-shared-title fallback, and
    Roman-numeral trimming);
  - the author cases (one shared author, two, three, four or more, none);
  - the earliest-date selection.

  Keep the lists (filler, series, generic) as named `const` slices with a short comment
  saying why they're strict.

- The module isn't wired into the CLI yet. Use `#[allow(dead_code)]` only where the gate
  phase will remove it.

## Phase adapter: render:false, compose, and the collection layout

**Python (`scripts/web_clip/`).**

- **`capture` gains an optional `"render": false`.** It writes `article.json` and
  `assets/` into the request's `workdir`, then skips printing. The response is the same,
  with `pdf_bytes: null` and an additive `"article_json": "<abs path>"`.
  - `out_pdf` may be omitted when `render` is false.
  - Headed retries, fidelity checks, bot-wall handling, and the direct-PDF path stay
    unchanged. A direct-PDF response stays `kind: "pdf"`; Rust refuses it.
  - Requests without `render` behave exactly as today.
- **New `compose` op.** The request is:

  ```json
  {"protocol":1,"op":"compose",
   "parts":[{"article_json":"/abs/part-1/article.json"}, …],
   "collection":{"title":…,"author":…|null,"published":…|null,"captured":"YYYY-MM-DD"},
   "workdir":"/abs/compose","out_pdf":"/abs/collection.pdf"}
  ```

  The response is
  `{"protocol":1,"ok":true,"op":"compose","pdf_bytes":N,"images":{…}, "warnings":[…]}`.
  Failures use the existing `error_response` kinds (`invalid_request`, `render`,
  `browser`). The op launches its own headless browser through the existing discovery
  and `_RendererLauncher`. Add `"compose"` to `OPS` and to the module docstring.

- **`web_clip_render.render_collection(parts, collection, workdir, out_pdf, launcher)`.**
  - It builds the [document](#the-document) by reusing `normalize_images`,
    `shift_headings`, `show_dek`, `format_long_date`, and the masthead code (factor the
    masthead out of `assemble()`; don't duplicate it).
  - Rename assets per part before normalization, so no two parts can collide.
  - Add collection rules to `reader.css`, or to the inline `<style>` where the
    template's cascade note requires it.
  - `render()` for one article must produce the same HTML as before.
- **Self-test.** Add a two-article fixture whose parts both use `#introduction`, `#fn1`,
  and an `img-001` asset. Assert on the assembled HTML (this needs no browser):
  - ids and fragments are prefixed and every contents link resolves to a `part-N`
    section;
  - the opening page has no `h1`–`h6`;
  - there is exactly one `h1` per part;
  - asset names are unique.

  When a browser is available, print the fixture. Then verify that the outline has
  exactly two top-level entries, there are no blank pages, and the page counter runs
  continuously. Use any tool available in scratch (for example `uv run --with pypdf …`,
  or `qpdf --json`) and record the result in the phase notes. Don't add a pinned
  dependency for this.

**Rust (`clip_adapter.rs`).**

- **`CaptureRequest`** gains a `render: Option<bool>` with
  `skip_serializing_if = "Option::is_none"`, and `out_pdf` becomes optional the same
  way. Add a unit test proving the single-capture request JSON is byte-identical to
  today's.
- **`CaptureSuccess`** gains `article_json: Option<String>`.
- **New types and method.** Add `ComposeRequest`/`ComposeSuccess` and
  `ClipAdapterClient::compose()`. Compose uses the same timeout env, with its own call
  deadline. Add serialization tests for both.
- The protocol stays `1` (the change is additive). The adapter is embedded and
  materialized with the binary, so both sides ship together.

**Gate:**

- `just check-web-clip-adapter`. Run it with a short `TMPDIR` such as `TMPDIR=/tmp`;
  `bob-cli-5b` documents Chrome aborting under SASE's long TMPDIR.
- The Rust unit tests in `just check`.

## Phase gate: variadic TARGET, fail-fast validation, and overlap

- **`command()`.**
  - Make `target` take `num_args(1..)` with `ArgAction::Append`.
  - Rewrite the `TARGET` and `-N` help as in [Usage and help](#usage-and-help); the
    value name becomes `NAME`.
  - Mention the several-URL `-o` meaning in `-o`'s help, keeping the existing contract
    strings the help test asserts.
  - Add the **Several URLs** after-help paragraph and the two examples.
- **`run()`.** It reads `get_many::<OsString>("target")`. One target goes to
  `create_pdf` unchanged. The `-H` "requires an http(s) URL TARGET" check stays for one
  target. Two or more targets go to a new `collection.rs` entry point with
  `CreateOptions` and the resolved config.
- **`collection.rs`.** It runs fail-fast steps 1–7 in order, using the naming phase's
  pure functions:
  - Exit codes: 2 for target-list usage errors (step 2, duplicates, the cap, no name); 1
    for overlap and runtime failures.
  - It prints the part-order warning to stderr when one applies.
  - It warns about legacy hits once.
- **`check_collection_overlap(recorded, keys, planned_target, force, listen)`.** Put it
  in `sources.rs` or `collection.rs`. It implements the
  [overlap policy](#overlap-policy-one-url-one-reference) and returns either
  `Clear { legacy }` or `Attach(hit)` (the attach is used by the listen phase), or a
  polished error that lists every overlapping part.
- **The stub.** After every gate passes, return
  `error: merging several URLs is not wired up yet` (exit 1). The route phase replaces
  it. Until then, an `-L` or `--audio` collection reaches the same stub.
- **Tests (`tests/cli/highlights/create.rs`, with a `FakeClip` that must not be
  called):**
  - every step 2–6 error, with its exit code and exact text;
  - a missing name exits 2 _without_ any adapter or `curl` call (use the fake's
    `called()` and a sentinel `PATH` curl);
  - `-N`, `-o`, and `-T` each satisfy naming;
  - options placed after the URLs parse;
  - `-n` still means `--no-audio`;
  - `-o` plus `-N` is still a clap conflict;
  - a single target is unchanged (existing tests stay green);
  - overlap with a PDF-backed note, a queued intake, a same-set collection, and a
    legacy-only note;
  - the alphabetical help test and the clip-alias byte-identical help test still pass.

> [!decision] output_dir = add This phase also adds `-O, --output-directory <DIR>`
> (`ValueHint::DirPath`, `~` expanded, cwd-relative). It conflicts with `-o` and `-t`,
> combines with `-N`, and applies to every route. The target becomes `DIR/<name>.pdf`
> through `plan_exact_output` (the clip route keeps its library-path refusal), with no
> `_2` walk. Place it in the help order immediately before `-o, --output`, then add
> help, after-help, and tests for each route kind.

## Phase route: classify, capture, compose, stamp, install, report

Replace the stub with the all-or-nothing pipeline, mirroring `clip.rs::capture_article`.

1. **Classify.** Run `target::fetch_and_route` on each URL, in order, inside one
   `ScratchDir`. Failures follow fail-fast step 8.
2. **Capture.** For each part, sequentially, call `ClipAdapterClient::capture` with
   `render: Some(false)` and `workdir = <scratch>/part-N`, with no metadata overrides
   (`-T`/`-A`/`-p` are collection-level). Print the TTY progress line. A failure maps to
   `part N/M (<host/path>): <message>` plus its hint plus `nothing was written`. A
   response with `kind: "pdf"` fails the same way as a classified PDF.
3. **Dry run.** Captures run with `dry_run: true`. Skip compose and install, and print
   the dry-run report.
4. **Compose.** Derive the title, author, and published date with the naming phase's
   functions, then call `compose` into `<scratch>/collection.pdf`, printing the progress
   line.
5. **Install.**
   - Re-collect the recorded sources and re-run the overlap check and
     `refuse_target_collisions`, because the vault may have changed during minutes of
     capture.
   - Compose the marker with the list `source_url` and stamp it with `PdfInfo`.
   - Install through the shared `stamp_and_install`. An explicit `--audio PATH` follows
     clip's `plan_explicit_audio` → `copy_audio_for_install` → cleanup-on-failure
     sequence.
   - `-i` is a no-op (URL routes always embed `id`) and `--no-audio` is accepted.
6. **Report.** Print the [success report](#output) and the per-part warnings.
   `print_next_step`.

`-L` still reaches a clear `--listen with several URLs arrives in the next phase` error
until the listen phase lands.

**Fake adapter.** Extend `tests/cli/highlights/fake_clip.rs`:

- save each request as `request-<n>.json`;
- answer captures from `response-<n>.json` when present, else `response.json`;
- dispatch `"op":"compose"` to copy the fixture PDF to `out_pdf` and print a compose
  response;
- let a test make capture N fail with an `ok:false` body.

**Tests:**

- the marker carries the ordered list and `id`;
- `bob ref scan` writes one note whose frontmatter has the list `source_url` and one
  `^ref` task;
- a later single-part `bob ref create` refuses, and `bob ref find <part URL>` returns
  the collection;
- a failing part 2 leaves `xlib/`, `lib/`, and `ref/` byte-identical and prints
  `part 2/2` and `nothing was written`;
- a part classified or captured as PDF is refused;
- the `_2` collision walk;
- the dry-run report, with `writes: none` and no compose request;
- `-o` installs at the exact path;
- explicit `--audio`;
- the 2-part and 3-part success reports (title and author derivation visible).

## Phase listen: `--listen` for collections

- **Normal run.** Reuse create.rs's `stamp_listen_and_install_pdf` (widen it to
  `pub(super)`):
  - stamp the composed PDF into scratch first;
  - run the listen command with `{target}` = `{pdf}` = the staged, unstamped collection
    PDF, `{audio}` = the scratch episode, and `{title}` = the collection title;
  - re-check collisions and the collection overlap;
  - install the audio, then the PDF.

  Post-listen failures keep the scratch audio with the existing `kept:` line, and the
  bind hint lists the original URLs.

- **Dry run.** Prints the `would run:` line through `listen::would_run_line`, plus the
  `audio: … (from --listen)` line.
- **Attach.** When the overlap check returns `Attach(hit)` (exactly our key set, on a
  PDF-backed note or a queued intake), call `attach::run_attach` with `{target}` = the
  existing capture's PDF and the source line `sources: N URLs (same set as <path>)`.
  Partial overlap still refuses, even with `-L`.
- **Tests (the `listen.rs` harness with the fake listen command):**
  - narration receives the staged PDF and the collection title;
  - a listen failure writes nothing;
  - the same-set attach for a PDF-backed note and for an intake PDF;
  - partial overlap with `-L` refuses;
  - the dry-run `would run:` line.

## Phase docs: docs, live verification, and follow-ups

- **`docs/highlights-create.md`.** Add a **Several URLs** section covering:
  - usage and examples;
  - the naming sentence, the full rule, and the naming table's highlights;
  - the waterfall and the messages;
  - the fail-fast order, all-or-nothing, the cap, and web articles only;
  - the overlap policy;
  - `-L`;
  - the "read as one unit" guidance.

  Update § Targets, § Dedupe, and § Listen where they say "TARGET" in the singular.

- **`docs/highlights-clip.md`.** Add `render: false` and the `compose` op to the adapter
  protocol summary, and describe the collection layout.
- **`docs/ref.md`.** Collections store `source_url` as an ordered list.
- **Live check on athena, in a scratch vault (never Bryan's real vault).**
  1. Set up a `-b` bob dir with a minimal area note for `-P`.
  2. Pick a real 2- or 3-part public series, not the Stripe pair, and run `--dry-run`,
     then `create`, then `bob ref scan -b …`.
  3. Inspect the PDF: the outline (one bookmark per part), the contents links,
     fresh-page part starts, continuous footers, images, and the note's list
     `source_url` and `^ref` task.
  4. Also confirm that the missing-name error and the part-failure message read well in
     a real terminal.

  Don't run a paid `-L` narration; `-L` stays covered by the fake.

- **Follow-ups.** Record each [out-of-scope](#out-of-scope-for-v1) item worth doing as a
  `PROPOSED FOLLOW-UP:` note on this phase bead.

> [!decision] decision_record Add the new strand
> `decisions:ref-create-merges-several-urls` through `/sase_memory_write`, in the
> existing strand format (Applies to, Claim, Why, Cost, Reopens when, Evidence). Claim:
> several `bob ref create` URLs merge into one reference; the name comes from `-N`, the
> `-o` filename, the shared slug words, or `-T`, else creation stops before any network
> call; any partial overlap refuses. Evidence: the research report, this plan, and the
> epic's commits. Then run `sase memory init`.

## Verification

- **Every phase.** `just check`. Phases that touch the adapter also run
  `TMPDIR=/tmp just check-web-clip-adapter`.
- **Known baseline failures.** These are unrelated to this epic: `bob-cli-6m`
  (percent-listen capture tests) and `bob-cli-5t` (Pandoc `\hyperlink` return-link
  tests). Compare against `master` before attributing a failure to this work.
- **End-to-end invariants the route and listen phases must prove with tests:**
  - no network or adapter call precedes a naming or validation error;
  - a failed part leaves the vault byte-identical;
  - one target's behavior and adapter request JSON are unchanged;
  - a merged reference is one note, one `^ref` task, and one PDF whose every part URL
    dedupes, routes, and finds back to it.

## Risks

- **Template cascade for footers.** The template already documents a margin-box ordering
  quirk. Per-part footers are conditional on a prototype print, and the collection-title
  footer is the fallback.
- **Long runs.** Twenty headed captures can take a long time. Each part keeps its own
  timeout and the cap bounds the total; progress lines keep it legible.
- **Snapshot semantics.** Adding a part later would shift pages under existing
  annotations. `create` never updates a collection; the docs say so.
