---
tier: epic
title: Spotify episodes become first-class podcast references
goal: "Every Spotify episode link given to `bob ref create`, capture, or Keep becomes
  one deduplicated `podcasts` reference. Its PDF is an episode dossier: cover, show,
  date, duration, clickable chapters, full show notes, and a highlightable transcript
  when the publisher's own feed proves the match. Every other Spotify link is refused,
  or is kept as an ordinary item, with a hint that says what to do instead. No Spotify
  link reaches the article engine again.

  "
decisions:
  legacy_refs:
    ask:
      Upgrade the two existing Spotify refs in place (title stems + new dossier PDF)
      once the epic ships?
    default: true
    why:
      Both are cited, unannotated episodes; rename and overwrite keep their tasks and
      links
    answer: true
  decision_record:
    ask: Add a decisions memory record for the podcast-episode reference policy?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: clean_capture
    title: General capture fixes the audit exposed
    depends_on: []
    size: medium
    description:
      "clean_capture: reject base62 ids as URL stems, persist the write-once source
      store on the ingest path, and print the recorded URL (not the post-redirect URL)
      in web PDF mastheads."
  - id: spotify_identity
    title: Spotify link identity, canonical dedupe, and routing guards
    depends_on: []
    size: medium
    description:
      "spotify_identity: add the spotify.rs link grammar with canonical case-preserving
      identity and ?t= capture, resolve short links, gate capture/Keep admission, and
      refuse non-episodes, -L, -H, and collections with teaching hints."
  - id: episode_metadata
    title: Episode metadata ladder and show-notes segmentation
    depends_on:
      - spotify_identity
    size: medium
    description:
      "episode_metadata: fetch and parse browserless episode metadata (page state,
      JSON-LD/meta, embed, oEmbed), segment show notes into notes/chapters/sponsor
      blocks, and download cover art, all fixture-tested."
  - id: dossier_render
    title: Adapter render op and the episode dossier template
    depends_on:
      - clean_capture
    size: medium
    description:
      "dossier_render: add the offline adapter `render` op and the podcast-episode
      dossier layout, CSS, and source view, plus the Rust client method and fake-adapter
      support."
  - id: episode_route
    title: The podcast route in bob ref create
    depends_on:
      - episode_metadata
      - dossier_render
    size: medium
    description:
      "episode_route: wire Spotify episodes into create (new, dry run, reread,
      overwrite) as podcasts refs with title stems, honest reports, the source store,
      help, and docs, then smoke-test live."
  - id: episode_ingest
    title: Capture jobs and Keep file episodes through the same route
    depends_on:
      - episode_route
    size: medium
    description:
      "episode_ingest: route Spotify episodes in ingest_url through the shared dossier
      builder, carry ?t= from the original URL, turn %l into a hint, and type every
      failure."
  - id: feed_match
    title: Publisher feed discovery and proven item match
    depends_on:
      - episode_metadata
    size: medium
    description:
      "feed_match: find the show's public RSS feed through iTunes Search, parse it
      safely with a capped fetch, and accept only a unique item agreeing on two of three
      signals."
  - id: transcript_text
    title: Publisher transcript parsing and paragraphing
    depends_on:
      - episode_metadata
    size: medium
    description:
      "transcript_text: parse JSON/VTT/SRT/HTML/text transcripts into sanity-checked,
      speaker-labelled paragraphs with honest role inference."
  - id: transcript_dossier
    title: Feed enrichment in the dossier and both routes
    depends_on:
      - episode_ingest
      - feed_match
      - transcript_text
    size: medium
    description:
      "transcript_dossier: attach proven feed notes, chapters, and transcripts to create
      and ingest captures under a time budget, render the transcript pages, add the
      enrich knob, and report each step-down once."
  - id: rollout
    title: Podcast surfaces across repos and the legacy refs
    depends_on:
      - transcript_dossier
    size: small
    description:
      "rollout: give podcasts refs their own kind in Bob Mac Capture, the reading board,
      and refs.base, upgrade the two legacy refs per decision, and author the optional
      decision record."
proposed_by: bbugyi200.apollo.research.5r.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 14:59:38
status: wip
---

- **PROMPT:**
  [prompts/202611/spotify_episode_refs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/spotify_episode_refs.md)

# Plan: Spotify episodes become first-class podcast references

## Context

The current output is wrong in several ways. `~/bob/ref/blogs/4ugvfwty2i8hgagsvt7pjc.md`
(_Software factories and code reviews with Harjot Gill_, The Pragmatic Engineer) came in
through share-to-Keep and is filed like this:

- it is filed under `blogs/` with an opaque base62 stem and id;
- its `source_url` still carries `si`/`sci` share tokens;
- it has no `author` or `published`;
- its PDF is one page of Spotify player chrome: the duplicated title, a half-page cover,
  a relative date ("Wednesday"), the masthead URL with `nd=1&dlsi=…` tracking, "56
  words", the show notes cut off mid-word ("Built for th"), and "See all episodes".

Its twin, `ref/blogs/5ygcxo9nxh38ns7ghtbddm.md` (the Satya Nadella episode), has the
same problems. Bryan cites both episodes, but neither has a highlight, because the PDFs
hold nothing worth marking.

The design follows the consolidated research report. Read it before starting any phase:

```bash
sase artifact read "research:202611/spotify_episode_capture_route/spotify_episode_capture_route.md" "Requirements for the Spotify episode epic"
```

Bryan accepted every requirement in that report, but memory changes need his separate
approval (see the `decision_record` decision). Facts re-verified at HEAD `22391208`
while this plan was written:

- **User agents get different pages.** A plain `bob-cli/1.0` user agent (what `fetch.rs`
  sends) gets a 228 KB server-rendered episode page. Its base64
  `<script id="initialState">` entity `spotify:episode:4UgVFWty2I8hGAgSvT7pjC` carries:
  - `name`;
  - a 4,210-char `htmlDescription` that holds 16 timestamp lines after a "Timestamps"
    label, a "Brought to You By:" sponsor run ending at a `—` line, and 16 U+2060
    characters;
  - `duration.totalMilliseconds`, `releaseDate.{isoString,precision}`, and
    `language.code`;
  - cover art at 64, 300, and 640 px;
  - `showOrAudiobook.data.{name,publisher.name,uri}`.

  The page also has a JSON-LD `PodcastEpisode` block and `music:duration` /
  `music:release_date` meta tags. The embed page (`/embed/episode/<id>`) has
  `__NEXT_DATA__` with the title, show, date, and duration. oEmbed returns only the
  title and a thumbnail. Chrome's user agent gets an empty app shell, which is why
  today's capture fails.

- **The ingest path skips the source store.** `web_source::persist_article_store` has
  one caller (`clip.rs`), and `ingest.rs` passes `expected_source_pointer: None`. So
  `bob ref links` and `bob ref code` are broken for every Keep- and capture-ingested
  article.
- **The masthead prints the post-redirect URL.** The adapter writes `final_url` into
  `article.json`'s `source_url` (`web_clip_adapter.py` around line 2107), and the
  masthead prints it (`web_clip_render.py::_masthead_fields`). The PDF marker records
  the cleaned request URL, so the PDF and its own marker disagree.
- **`is_opaque_segment`** (`clip_url.rs`) rejects only hex ids and UUIDs. **`si`/`sci`**
  are not tracking parameters, so they stay in the dedupe key. Ref types are free-form
  directories, and nothing in bob-cli enumerates them.
- **Three places map ref types to labels:**
  - Bob Mac Capture `RefKind` (`Sources/RefsCore/RefsModel.swift`; unknown types get the
    grey `doc` tile);
  - bob-plugins `READING_BOARD_KIND_MAP`
    (`plugins/bob-ledger-tools/src/242-plugin-reading-board-models.js`; unknown types
    get 🔖);
  - the vault's `~/bob/refs.base` `category` formula (folder-keyed; unknown folders get
    "🔖 Ref").

### What the finished epic does

```text
Spotify link ── spotify.rs parse (no network) ─┬─ not Spotify ─────────────► today's routes
  open.spotify.com, play.spotify.com,          ├─ show/track/album/… ───────► refuse exit 2 + hint
  /intl-xx/, /embed/, spotify:episode:…,       │                              (capture/Keep: ordinary item)
  spotify.link (resolved, then re-parsed)      └─ episode ── canonical id ──► dedupe (legacy si/sci match)
                                                    │
     Spotify page → JSON-LD/meta → embed → oEmbed ──┤  fail closed without a real title
     iTunes → public RSS feed → unique 2-of-3 match ┤  fail open, one warning per step-down
                                                    ▼
     Bob-built article.json ─► adapter `render` op (Chromium, reader template) ─► dossier PDF
                                                    ▼
     stamp marker ─► install ─► write-once source store ─► intake files ref/podcasts/<title_stem>.md
```

### Never in this epic

- downloading episode audio (the 30 s `og:audio` preview is never companion audio);
- the Spotify Web API, OAuth, or Spotify's in-app transcripts;
- excluding `spotify.com` hosts;
- `-L/--listen` on an episode;
- growing the global `TRACKING_PARAMS` list.

These later ideas are out of scope: an Apple Podcasts front door, Whisper transcription,
show-record linking, and cross-type `bob ref rename -t`.

### Rules for every phase

- **Before changing help or docs, read the memory notes.** Run
  `sase memory read cli_rules.md docs.md -r "<why>"`. This epic adds no CLI flags.
  Behavior changes go into `docs/highlights-create.md` and the other contract pages
  named below. Then run `just docs-gen` (help changes) and `just docs-check`.
- **Run the gates.** Every phase runs `just check`. Phases that touch
  `scripts/web_clip/` also run `just check-web-clip-adapter`.
- **Commits and releases.** Use Conventional Commits, and mark CLI or JSON contract
  breaks with `!`. `spotify_identity` is one: `-H` and non-episode Spotify links change
  behavior. Never edit the `Cargo.toml` version or `CHANGELOG.md`; the daily release
  train owns them.
- **Honor these accepted decisions:**
  - `references-are-ingested-at-birth`
  - `reference-ids-follow-file-stems` (id = final stem)
  - `reference-sources-are-preserved-write-once`
  - `reference-renames-keep-identity`
  - `mac-capture-is-a-thin-client` (bob-cli owns parsing; the Mac only renders bob's
    output)
  - `code-copy-reads-render-time-catalog`
- **Shared module layout, to avoid cross-phase collisions:**
  - `src/native/highlights_ref/spotify.rs` (identity; `spotify_identity`)
  - `src/native/highlights_ref/podcast/mod.rs` with `metadata.rs`, `notes.rs`, and
    `cover.rs` (`episode_metadata`)
  - `podcast/dossier.rs`, the shared capture builder (`episode_route`)
  - `podcast/feed.rs` (`feed_match`)
  - `podcast/transcript.rs` (`transcript_text`)
- **Put test fixtures under `tests/fixtures/spotify/` and `tests/fixtures/podcast/`.**
  When a phase captures live public responses as fixtures, trim them to the elements the
  parser reads.

## clean_capture — General capture fixes the audit exposed

Each of these fixes helps every web reference, not only Spotify ones.

1. **Opaque base62 URL segments never become stems.** Extend
   `clip_url::is_opaque_segment` with a base62-id test. A segment counts as an id when
   all of these hold:
   - it is at least 16 characters, ASCII alphanumeric only;
   - it has at least one digit, one uppercase letter, and one lowercase letter;
   - split into tokens at every letter/digit boundary and at CamelCase boundaries
     (before an uppercase letter that follows a lowercase one, or that starts an
     uppercase-then-lowercase word), its mean token length is under 2.5.

   | Segment                     | Expected   |
   | --------------------------- | ---------- |
   | `4UgVFWty2I8hGAgSvT7pjC`    | opaque     |
   | `iPhone15ProMaxReview`      | not opaque |
   | `DeepSeekV3TechnicalReport` | not opaque |
   | `GPT4oSystemCard`           | not opaque |

   With an opaque segment, the existing waterfall falls back to the title stem.
   Unit-test all four segments plus random 22-character ids.

2. **Ingest persists the write-once source store.** After install,
   `ingest.rs::ingest_article_route` calls a quiet variant of
   `web_source::persist_article_store`:
   - **No output.** It writes no stdout or stderr: ingest's contract forbids both, and
     `bob gkeep pull --json` must stay clean. Warnings go through `request.progress`.
   - **Pointer.** Thread the verified pointer into `apply_intake`'s
     `expected_source_pointer` (today `None`).
   - **Queued intake.** It still adopts the store by PDF digest at scan time.
   - **Tests.** A seeded article job in `tests/cli/highlights/jobs.rs`, and a Keep pull
     in `tests/gkeep_pull.rs`, each file a note with `source_markdown`. Running
     `bob ref links` on it lists the article's links, and `gkeep pull --json` stdout is
     unchanged.
3. **Web PDF mastheads print the recorded URL.**
   - **Adapter.** In `run_capture` (both the headless and headed paths), write
     `article.json` with `source_url` set to the request `url`, which Rust already
     cleaned. Add a separate `base_url` set to the post-redirect URL.
   - **Renderer.** Use `base_url` (falling back to `source_url`) only for
     `resolve_relative_hrefs`. The masthead and the source view print `source_url`.
   - **Unchanged.** The response's `final_url` stays as it is, because Rust's redirect
     guards use it. Collections get the same treatment per part.
   - **Tests.** Add self-test cases: a redirected capture prints the request URL, and
     relative links still resolve against the final page.
4. **Docs.** In `docs/highlights-create.md`, the Ingest boundary section must say that
   ingest saves the source store. Its stale "no audio" wording is also fixed: ingest
   supports `%l`.

## spotify_identity — Spotify link identity, canonical dedupe, and routing guards

New `src/native/highlights_ref/spotify.rs`, modeled on `arxiv.rs` and re-exported the
way `ArxivPaper` is for `url_routing`. It is pure: no network except
`resolve_short_link`.

```rust
pub(crate) enum SpotifyKind { Episode, Show, Track, Album, Playlist, Artist, Audiobook, Chapter, User, Other }
pub(crate) enum SpotifyLink {
    Entity { kind: SpotifyKind, id: String, shared_at: Option<u32> }, // id case preserved
    Short { url: String },       // spotify.link, spotify.app.link, spoti.fi
    Unsupported { url: String }, // a Spotify host whose path names no entity
}
impl SpotifyLink { pub(crate) fn parse(raw: &str) -> Option<Self> } // None = not a Spotify link
pub(crate) struct SpotifyEpisode { pub id: String, pub shared_at: Option<u32> }
// canonical_url() = https://open.spotify.com/episode/<id>; dedupe_key() = same; at(secs) = canonical + ?t=<secs>
```

**Grammar**

- **Web URLs.** `http(s)` on the exact hosts `open.spotify.com` or `play.spotify.com`
  (case-insensitive). The path is an optional `/intl-<lang>[-<REGION>]`, then an
  optional `/embed` or `/embed-podcast`, then `/<kind>/<id>`, with an optional trailing
  slash.
- **Ids.** Entity ids are exactly 22 `[0-9A-Za-z]` characters, and their case is
  preserved: the lowercased id 404s. `user/<anything>` parses as `User`. Any other path
  on these hosts parses as `Unsupported`.
- **URIs.** `spotify:<kind>:<id>`; the scheme and kind are case-insensitive.
- **Short links.** Only the exact hosts `spotify.link`, `spotify.app.link`, and
  `spoti.fi`.
- **Not Spotify.** Every other host returns `None` and stays on today's routes. That
  includes other `*.spotify.com` subdomains, `atspotify.com`, and lookalikes such as
  `open.spotify.com.evil.example` or `evilspotify.com`.
- **Shared moment.** `?t=<non-negative integer seconds>` becomes `shared_at`. Any other
  `t` value is ignored, never an error. No query parameter or fragment is ever part of
  identity.

**Identity**

- **Canonical URL.** `validate_and_clean` sets `cleaned` to the canonical
  `https://open.spotify.com/<kind>/<id>` for every Spotify entity link, and
  `dedupe_key_for` returns it. Recorded sources are re-keyed through `dedupe_key_for`
  when they are read (`sources.rs`), so both legacy `si`/`sci` refs dedupe with no
  migration. Test with their exact stored `source_url`.
- **Lookups.** In `ref_library::identity`, `bob ref find` resolves every Spotify
  spelling, including `spotify:episode:<id>` URIs, to that key.

**Short links.** `resolve_short_link(url, scratch, progress)` uses `fetch::fetch_url`,
which already validates every redirect hop. The final URL must parse as an `Entity`;
otherwise it fails with:

```text
error: could not resolve Spotify short link <url> to an open.spotify.com link
hint: open it in Spotify and share the open.spotify.com link instead
```

It also returns the downloaded final body, so `episode_route` can reuse that body as the
first metadata source.

**Create.** Normalize Spotify spellings once, when the TARGET is prepared: a URI becomes
the canonical https URL, and a short link is resolved and re-parsed. This happens before
the recapture preflight and every `resolve_url_syntactic` caller (`create.rs` around
lines 1106, 1271, and 1568), so later steps see an ordinary `open.spotify.com` URL.
Refusals exit 2:

| Input                                                           | `error:`                                                                                            | `hint:`                                                                                                             |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| show                                                            | `Spotify show links are not references: <url>`                                                      | `capture one episode (⋯ → Share → Copy link); follow the show as a record in your podcasts album (docs/records.md)` |
| track / album / playlist / artist                               | `Spotify music links are not references: <url>`                                                     | `bob files Spotify podcast episodes only`                                                                           |
| audiobook / chapter                                             | `Spotify audiobooks are not references: <url>`                                                      | `bob files Spotify podcast episodes only`                                                                           |
| user / other / unsupported path                                 | `not a Spotify episode link: <url>`                                                                 | `share the episode link (open.spotify.com/episode/…)`                                                               |
| `-L` with any Spotify link                                      | `--listen does not apply to Spotify episodes: the episode is already audio`                         | `drop -L; pass --audio FILE to bind audio you already have`                                                         |
| `-H` with any Spotify link                                      | `--html does not apply to Spotify links: bob reads episodes from Spotify's public episode metadata` | `drop -H`                                                                                                           |
| a Spotify link among several URLs (`collection.rs`, like arXiv) | `Spotify episodes are captured one at a time: <url>`                                                | `run bob ref create once per episode`                                                                               |

Until `episode_route` lands, episodes stay on today's route, now with the canonical
`source_url` and dedupe key.

**URL routing**

- **Route hint.** `RouteHint::Podcast` (serialized as `podcast`) is used for episode
  links and short links; Bob Mac Capture already renders an unknown hint as a
  capitalized chip, "Podcast". `classify_token` returns the canonical `cleaned`,
  `dedupe_key`, and `display`.
- **Admission.** `UrlRoutingPolicy::admits` returns `false` for non-episode Spotify
  entities and unsupported Spotify paths. Capture and Keep therefore keep them as
  ordinary items, exactly like excluded hosts. Short links are admitted; the worker
  resolves them.

**Ingest.** `ingest_url` resolves a short link before both dedupe probes. If it resolves
to a non-episode, `ingest_url` returns a typed, non-retryable `unsupported_content`
error with the table's message and hint. Build it directly, not through the substring
classifier.

**Docs.** Update these pages:

- `docs/capture.md` and `docs/ref-jobs.md`: the `route_hint` value list gains `podcast`.
- `docs/highlights-create.md`:
  - the Dedupe section describes Spotify canonical identity;
  - the `-H` text in the create help and docs names the Spotify exception.

**Tests**

- **Grammar table:** every kind, `intl-*`, `embed`, `embed-podcast`, the play host,
  URIs, a trailing slash, `?si=&sci=&t=`, fragments, case preservation, ids of 21 and 23
  characters, lookalike hosts, and other subdomains.
- **Dedupe:** a legacy `si`/`sci` record matches a clean link; two different ids stay
  distinct.
- **Capture:** `bob capture --dry-run` on a show URL plans an ordinary task, and on an
  episode URL plans a ref job with `route_hint: podcast` and the canonical key. Add the
  matching Keep-plan test.
- **Create:** refusal messages and exit 2 for each table row.
- **Find and short links:** `bob ref find spotify:episode:<id>`, and a short link
  through a fake-curl redirect.

## episode_metadata — Episode metadata ladder and show-notes segmentation

This phase adds a library only; no route uses it yet.

- **`fetch.rs`.** Add `fetch_url_capped(url, dest, max_time_secs, max_bytes, progress)`.
  `fetch_url` keeps its behavior.
- **`podcast/metadata.rs`.** Add
  `fetch_episode(&SpotifyEpisode, scratch, progress, prefetched_page) -> Result<EpisodeMetadata, EpisodeError>`.
  It fills fields from these tiers in order. A later tier fills only fields that are
  still empty, and every field records its source (`spotify_page`, `spotify_embed`,
  `spotify_oembed`):
  1. **The episode page** (`https://open.spotify.com/episode/<id>`, 5 MiB cap), parsed
     with `scraper`:
     - the base64 `initialState` entity under the exact key `spotify:episode:<id>`, with
       `__typename == "Episode"`;
     - the JSON-LD `PodcastEpisode`, either as an object or as an array member, whose
       `@id`/`url` names the id. Its description loses the prefix "Listen to this
       episode from <show> on Spotify. ", and that prefix also yields the show name;
     - the `og:title`, `og:url` (must name the id), `og:image`, `music:duration`, and
       `music:release_date` meta tags.

     Ignore `og:type`, which says `music.song`.

  2. **The embed page** (`/embed/episode/<id>`) `__NEXT_DATA__`, when the title or show
     notes are still missing. Validate the id.
  3. **oEmbed** (`https://open.spotify.com/oembed?url=<canonical>`), when the title is
     still missing. The thumbnail host varies.

  **Failure behavior:**
  - **Fail closed** when no tier yields a title:
    `Spotify returned no episode metadata for <id>`, with the hint
    `retry later; if it persists, Spotify changed its page and bob needs an update`.
  - **HTTP 404:** `Spotify has no public episode <id> (HTTP 404)`.
  - **HTTP 429 or 5xx:** a typed, retryable status.
  - **Unknown shapes** yield `None` and never panic.
  - **A fallback that lost content** (for example, no show notes) adds exactly one
    warning that names what is missing.

- **`EpisodeMetadata`** holds:
  - identity: `id`, `canonical_url`, `title`;
  - the show: `show`, `show_uri`, `publisher`;
  - timing: `released {instant, precision}`, `duration_ms`, `language`;
  - content: `cover_url` (episode art at most 640 px, else show art), `notes_html`,
    `notes_text`;
  - bookkeeping: per-field sources and `warnings`.

  `published_date()` converts the instant to the local date with the zone
  `note::capture_clock` uses. DAY precision uses the date as given.

- **Text cleanup.** Remove U+2060, U+200B, and U+FEFF from the title, show, and notes,
  and collapse whitespace in the title.
- **`podcast/notes.rs`** splits `notes_html` into ordered `NotesBlock`s:
  - `Html(String)` holds raw publisher fragments; the renderer sanitizes them.
  - `Chapters(Vec<Chapter { start_secs, title }>)` is an inline run of three or more
    consecutive block lines: `<p>`, `<li>`, or `<br>`-separated lines. Each line's text
    must match `^[(\[]?((\d{1,2}:)?\d{1,2}:\d{2})[)\]]?\s*[-–—:|]?\s*(.+)$`. The times
    must strictly increase, the first must be at most 10:00, and the last must be
    shorter than the duration. A directly preceding label line of at most four words
    (such as "Timestamps", "Chapters", or "In this episode:") is consumed. The block
    stays where the run was, so the publisher's order is kept.
  - `Sponsors(String)` is a run that starts with a line matching
    `^(this episode is )?(brought to you by|sponsored by|thanks to our sponsors|our sponsors|sponsors)\b`
    (case-insensitive) and ends at the next separator line (`—`, `–`, `-{3,}`, `\*{3,}`,
    `_{3,}`, `• • •`) or at the end of a bullet run. It is moved to the end only when
    that terminator is found; otherwise it stays in place. The separator is dropped.
  - Leading and trailing separators are trimmed, and interior separator-only lines
    become `<hr>`.
  - A word count of the notes is kept for reporting.
- **`podcast/cover.rs`.** Download the cover with a 5 MiB cap. Accept only JPEG, PNG, or
  WebP, checked by magic bytes. A failure returns `None` plus one warning.
- **Fixtures** for episode `4UgVFWty2I8hGAgSvT7pjC`: today's page, embed, and oEmbed
  responses, trimmed. Add synthetic drift cases: no `initialState`, a mismatched entity
  id, a JSON-LD array, oEmbed only, and a 404.
- **Unit tests:**
  - field extraction, ladder order, and id-mismatch rejection;
  - U+2060 stripping;
  - an `02:00Z` release is the previous local day in America/New_York;
  - the real notes yield 16 chapters from 0:00 to 1:24:30, with "Timestamps" consumed;
  - the "Brought to You By:" run is demoted at the `—` line;
  - a sponsor run with no terminator stays in place;
  - non-increasing times are not chapters.

## dossier_render — Adapter render op and the episode dossier template

**Adapter op.** `scripts/web_clip/web_clip_adapter.py` gains a protocol-1 `render` op,
shaped like `op_compose`. The request is
`{protocol: 1, op: "render", article_json, workdir, out_pdf, tables}`, with absolute
paths:

- **What it skips:** no network, no Defuddle, and no fidelity verdict.
- **Printing:** it uses the existing `print_pdf`, where `outline=True` and `tagged=True`
  already turn headings into PDF bookmarks.
- **Response:** `pdf_bytes`, `source_html`, `code_anchors`, `tables`, `images`, and
  `warnings`.
- **Errors:** payload errors are typed `invalid_request` failures.

**Rust side.** `clip_adapter.rs` gains `ClipAdapterClient::render`. The fake adapter in
`tests/cli/highlights/fake_clip.rs` answers `"op":"render"` the way it answers compose:
it copies the fixture PDF and returns the canned `render.json`.

**Renderer.** `web_clip_render.render()` branches on
`article["kind"] == "podcast_episode"`:

- **Sanitizing.** It cleans every `episode.blocks[].html` and `episode.sponsors_html`
  with the adapter's nh3 allowlist. Share `sanitize_html` rather than copying it. It
  drops `<img>` from the notes; the op is offline, and the cover is the only image.
- **Language.** It sets `<html lang>` from `article["language"]` (default `en`) so
  hyphenation and the font stack fit the episode.

The payload contract (Rust writes it in `episode_route`; `transcript` stays `null` until
`transcript_dossier`):

```json
{
  "kind": "podcast_episode",
  "title": "Software factories and code reviews with Harjot Gill",
  "source_url": "https://open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC",
  "captured": "2026-11-21",
  "published": "2026-11-18",
  "language": "en",
  "episode": {
    "show": "The Pragmatic Engineer",
    "publisher": "Gergely Orosz",
    "duration_ms": 5197032,
    "cover": "assets/cover.jpg",
    "listen_url": "https://open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC",
    "shared_at_secs": 1235,
    "chapter_links": true,
    "blocks": [
      { "type": "notes", "html": "<p>In this conversation…</p>" },
      { "type": "chapters", "chapters": [{ "start_secs": 0, "title": "Intro" }] },
      { "type": "notes", "html": "<p>The Pragmatic Engineer deepdives…</p>" }
    ],
    "sponsors_html": "<p>• turbopuffer – …</p>",
    "provenance": [
      {
        "label": "Episode details and show notes",
        "source": "Spotify's public episode page"
      }
    ],
    "transcript": null
  }
}
```

**Visual spec.** Use the existing reader palette and fonts: ink `#1f2328`, muted
`#57606a`, accent navy `#2b4c7e`, hairline `#d0d7de`; Source Serif 4, Inter, and
JetBrains Mono. Page 1 must stand alone as a card:

```text
┌────────────┐  THE PRAGMATIC ENGINEER                       ← kicker: show name, Inter 600 7.5pt caps, navy, +0.08em
│  cover     │  Software factories and code reviews          ← title once, Source Serif 4 semibold (existing masthead-title size)
│  1.35in sq │  with Harjot Gill
│  hairline  │  Gergely Orosz                                ← publisher; omitted when it equals the show
└────────────┘  Podcast episode · November 18, 2026 · 1 h 26 min   ← muted meta; absolute date, duration (no word count)
                [▶ Listen on Spotify]  [⏱ Shared at 20:35]     ← pills: Inter 500 8pt navy, #f3f6fa fill, 0.5pt #b6c3d6 border
────────────────────────────────────────────────────────────────
open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC · Captured November 21, 2026   ← existing provenance rule
In this conversation, we step into the world of an AI-pilled CEO…     ← notes in article-body type
CHAPTERS                                                        ← label in the collection contents-label style
 0:00  Intro                      46:05  Evals                  ← two columns past 8 rows; times right-aligned, tabular,
 3:17  How Harjot shipped…        50:25  Formal methods            navy links to …/episode/<id>?t=<secs> when chapter_links
…rest of the notes (deepdive links kept)…
┌ SPONSORS ───────────────────────────────────────┐            ← quiet card: #f6f8fa fill, 0.5pt #e1e4e8 border, 9pt muted
│ • turbopuffer – a vector and full-text search…   │              break-inside: avoid; placed after the notes
└─────────────────────────────────────────────────┘
Episode details and show notes: Spotify's public episode page · Content from Spotify · Captured November 21, 2026
```

**Details:**

- **Grid.** The masthead is a `1.35in 1fr` grid with a 0.22in gap. The cover is
  unmodified: no crop, recolor, or overlay. Use `object-fit: contain` for non-square
  art.
- **Pills.** Each pill is a real link, in Bob's palette (never Spotify green).
- **Shared-at pill.** It appears only when `shared_at_secs` is below the duration.
- **Formats.** Durations: `1 h 26 min`, `48 min`, `45 s`. Times: `m:ss` under an hour,
  otherwise `h:mm:ss`. Dates come from the existing `format_long_date`.
- **Notes.** The notes carry an `h2` "Show notes" bookmark heading, which can be
  visually quiet. Chapters render where their block sits.
- **Footer.** The provenance footer lists every `provenance` entry, plus "Content from
  Spotify" linked to `listen_url`. The running footer title is unchanged.
- **Page count.** Never force one page.

**Source view.** It is the Markdown that becomes the store and drives `bob ref links`:

- the H1, the byline, a meta line, and the provenance link;
- the notes, with the chapters as a plain list (`- 3:17 How Harjot shipped…`). Chapter
  times are not Spotify deep links, so `bob ref links` lists the publisher's links.
- `## Sponsors`.

**Tests**

- **Self-test** (`--self-test`) gains pure checks:
  - the title appears once;
  - the duration and date formats;
  - chapter `?t=197` links appear only when `chapter_links` is set;
  - the publisher is omitted when it equals the show;
  - the sponsor aside comes after the notes;
  - `img`, `script`, `style`, and `on*` attributes are stripped;
  - the shared-at pill follows its rule;
  - the `lang` attribute is set.
- **Visual check.** On a host with a browser, render a fixture payload built from the
  real episode. Rasterize every page with `mutool draw -r 110`, look at the images, and
  iterate until they meet the spec. Store the page PNGs as artifacts with
  `/sase_artifact`.
- **Docs.** Document the op wherever the adapter protocol ops are documented (grep docs/
  for `compose`).

## episode_route — The podcast route in bob ref create

**Shared builder.** `podcast/dossier.rs` is used by create here and by ingest in
`episode_ingest`. It runs:

1. `fetch_episode`;
2. the cover download;
3. `notes.rs` segmentation;
4. writing `article.json` per the contract, with the cover in the workdir's `assets/`;
5. `ClipAdapterClient::render`.

It returns the scratch PDF, the source view paths, and the plan fields.

**Create.** `create.rs` gets `create_podcast_route` and `overwrite_podcast`, which
mirror `create_arxiv_route` and `overwrite_arxiv`. `create_pdf` dispatches to them right
after the arXiv branch.

**Fields:**

| Field                     | Rule                                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------------------------- |
| `source_url`, dedupe keys | canonical                                                                                                |
| ref type                  | `podcasts` (new `DEFAULT_PODCAST_REF_TYPE`, a `"podcast"` arm in `default_ref_type_for_kind`); `-t` wins |
| title                     | `-T`, else metadata                                                                                      |
| author                    | `-A`, else show name                                                                                     |
| published                 | `-p`, else the local release date                                                                        |
| stem                      | `-N`, else `title_stem(title)` (80-char cap), else `spotify_episode_<lowercased id>`; never the raw id   |
| id                        | the final stem                                                                                           |
| PDF Info author           | the show                                                                                                 |

**Audio and recapture**

- **Audio.** `--audio FILE` and audio discovery behave exactly as on the arXiv route.
- **Reread and overwrite.** Bare `-f` (reread) and `-f overwrite` keep full parity with
  the arXiv route:
  - `prepare_web_overwrite`;
  - the `refuse_redirected_*` guards, keyed on the canonical key;
  - the source store staged the way web overwrites stage theirs (`prepare_web_store`).

  Overwrite keeps the reading's note, stem, id, task, and type.

- **Store.** After install, `web_source::persist_article_store` runs with the render's
  `source.html` and origin set to the canonical URL; `finish_create_intake` follows.
- **Folder check.** Confirm that album and record walkers ignore `ref/podcasts/`; albums
  are root-level, and `~/bob/podcasts/` holds the show records.

**Output.** A dry run uses the network for metadata but does not render or write:

```text
✓ would create Highlights-ready podcast PDF
source: https://open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC (Spotify episode)
pdf: …/xlib/podcasts/software_factories_and_code_reviews_with_harjot_gill.pdf
title: Software factories and code reviews with Harjot Gill (spotify page)
author: The Pragmatic Engineer (spotify page) · published: 2026-11-18 (spotify page)
duration: 1 h 26 min · chapters: 16 · cover: yes
… existing sidecar/library/status/parent/id/marker/intake lines …
writes: none
```

A real create prints the arXiv-style block with `created Highlights-ready podcast PDF`.
It adds a `duration:` line, a capture line such as
`capture: episode dossier · 16 chapters · show notes 512 words`, and
`pages: N · size: …`, followed by the store lines.

Each degradation prints exactly one `warning:` line, naming what was lost:

- the metadata fell back to another tier;
- the cover is missing;
- the show notes are missing.

The report never mentions audio or a transcript that it did not keep (`audio: none`).

**Help and docs**

- **Create help.** Add a `Targets` line to the create help:
  `Spotify episode (open.spotify.com/episode/…, spotify:episode:…, spotify.link) -> episode dossier from Spotify's public metadata (default ref type podcasts); other Spotify links refuse`.
  Also add the example
  `bob ref create https://open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC -d -P sase`.
- **Ref-type list.** Add `podcasts` to the type list in `ref_library/cli.rs`.
- **`docs/highlights-create.md`:**
  - a row in the Targets table;
  - a new `## Spotify episodes` section beside `## arXiv` covering the grammar,
    identity, dossier, refusals, and fail-closed behavior;
  - an update to the route → type table.

**Tests**

- **CLI tests** use fake curl serving the fixtures plus the `FakeClip` render op:
  - the dry-run block;
  - a real create files `ref/podcasts/<title_stem>.md` with the canonical `source_url`,
    show author, and published date;
  - `request.json` has `kind: podcast_episode` with 16 chapters and sponsors;
  - a legacy `blogs` ref with `si`/`sci` makes create refuse `already captured`;
  - reread produces `__v2`, and overwrite works in place;
  - `-N`, `-t`, `-T`, `-A`, and `-p` overrides;
  - a non-ASCII title falls back to the `spotify_episode_<id>` stem;
  - a metadata failure writes nothing;
  - a cover failure prints exactly one warning.
- **Live smoke test (required).** Use a throwaway vault, real network, and a real
  browser:
  `bob -b <tmp vault> ref create https://open.spotify.com/episode/4UgVFWty2I8hGAgSvT7pjC -P <area>`.
  `mutool draw -F txt` must show:
  - the title once, the absolute date, and the duration;
  - no "See all episodes";
  - untruncated notes.

  Rasterize the pages and compare them against the visual spec.

## episode_ingest — Capture jobs and Keep file episodes through the same route

- **Routing.** After short-link resolution, `ingest_url` (around `ingest.rs:667`) sends
  Spotify episodes to `ingest_podcast_route`, which uses the shared builder:
  - **Defaults:** `INGEST_PODCAST_REF_TYPE = "podcasts"`, status `ready`.
  - **Route name:** `IngestRoute::Podcast`, serialized as `podcast` wherever routes
    appear. This is an additive JSON change.
  - **Store:** the quiet store helper from `clean_capture` runs, and its pointer goes to
    intake.
- **Shared moment.** `IngestRequest` carries the original URL: the job's `url`, and
  Keep's `intent.original`. That way `?t=` becomes the dossier's Shared-at pill while
  identity stays canonical.
- **`%l` on an episode:**
  - **Capture** plans the job without listen, and the item's existing `listen_hint`
    becomes
    `Spotify episodes are already audio · %l ignored · to bind your own audio run: bob ref create <url> --audio FILE`.
  - **Keep** files the episode without narration and reports the same line through its
    per-item summary channel.
  - **Ingest** ignores `listen` for Spotify, so a job queued before this upgrade still
    files.
- **Typed errors**, built directly rather than through substring matching:

  | Failure                   | Kind                         |
  | ------------------------- | ---------------------------- |
  | 404                       | `http_status`, not retryable |
  | 429 or 5xx                | `http_status`, retryable     |
  | network or timeout        | retryable                    |
  | no title                  | `thin`                       |
  | render or browser failure | `render` or `browser`        |

- **Docs.** Update the `docs/highlights-create.md` Ingest boundary (route defaults now
  include `podcasts`), `docs/ref-jobs.md`, `docs/capture.md` (the `%l` hint), and any
  Keep doc that lists routes.
- **Tests:**
  - `jobs.rs`: a seeded Spotify job files `ref/podcasts/<stem>.md` with the canonical
    `source_url` and `source_markdown`, and the original `?t=1235` reaches
    `shared_at_secs`;
  - `gkeep_pull.rs`: a Keep note holding a share URL;
  - capture dry run with `%l`: the hint;
  - capturing an episode already filed as legacy `si`/`sci` yields `in_library`.

## feed_match — Publisher feed discovery and proven item match

`podcast/feed.rs` provides `find_feed_item(&EpisodeMetadata, scratch, deadline)`. It
returns `Matched(FeedItem)`, `NoFeed(reason)`, or `NoMatch(reason)`, and it fails open.

- **Finding the feed.** Query iTunes Search:
  `https://itunes.apple.com/search?media=podcast&entity=podcast&limit=10&term=<show>`.
  - Candidates are results whose normalized `collectionName` equals the normalized show
    name (case-folded, punctuation stripped, leading "the " stripped).
  - Rank them by how well `artistName` matches the publisher.
  - Try up to three `feedUrl`s; a 404 moves on to the next candidate.
- **Parsing the feed.** Use `fetch_url_capped` (25 MiB, 30 s). Parse with `roxmltree`;
  add it as a dependency and keep its default refusal of DTDs, so no entity expansion
  happens. Read the `itunes`, `podcast` (`https://podcastindex.org/namespace/1.0`), and
  `content` namespaces.
- **Matching an item.** Each item scores up to three signals:
  - its normalized title equals the episode's (U+2060 stripped, punctuation stripped,
    and a leading episode number such as `#123:` or `Ep. 12 -` stripped);
  - its `pubDate` is within 36 h of the Spotify release instant;
  - its `itunes:duration` is within max(5 s, 2 %) of the Spotify duration.

  An item missing a field just doesn't score that signal. The result is `Matched` only
  when exactly one item has at least two signals; a tie gives `NoMatch("ambiguous")`.

- **`FeedItem`** holds the feed URL and host, `guid`, title, `pub_date`, and duration;
  the notes HTML (`content:encoded`, else `description`); the transcripts
  `[{url, mime, language, rel}]`; and the chapters URL and type.
- **Chapters.** Add a Podcasting 2.0 chapters JSON fetch and parse
  (`chapters[].startTime/title`). Times must increase; invalid JSON is ignored with a
  reason.
- **Config.** Parse `highlights.podcasts.enrich: auto | off` (default `auto`) in
  `config/mod.rs`, next to the other `highlights` keys. `off` means no iTunes or feed
  requests at all. That also covers privacy and restricted networks.
- **Tests** use trimmed fixtures from the real Pragmatic Engineer iTunes result and feed
  (2–3 items), plus these cases:
  - 0, 1, and 2 candidates;
  - a 404 feed, then a working second candidate;
  - duplicate-title items (a tie);
  - a dynamic-ad-insertion duration mismatch that title and date still match;
  - a feed over the cap;
  - malformed XML;
  - a feed with a DTD (refused);
  - `enrich: off` makes no requests (check the fake curl log).

## transcript_text — Publisher transcript parsing and paragraphing

`podcast/transcript.rs` is pure: it takes bytes, a MIME type, the URL, and the episode
duration.

- **Choosing a format.** When several are offered, prefer Podcasting 2.0 JSON
  (`segments[].startTime/endTime/speaker/body`), then VTT, SRT, HTML (`<cite>` speaker,
  `<time>`, `<p>`), and text (blank-line paragraphs, no times).
- **VTT.** Handle the header, cue ids, settings, NOTE and STYLE blocks, `<v Name>` voice
  spans, inline tags, and entities.
- **Sanity checks.** Drop the transcript, with a reason, when:
  - it has fewer than 200 words;
  - it has more than 120,000 words;
  - it is timed, the duration is known, and the last cue starts before 50 % or after 110
    % of the duration.
- **Speakers.** Voice tags and JSON speakers count as real names unless they are
  diarization tokens (`SPEAKER_00`, `Speaker 1`, `spk_0`, `S1`). Tokens become
  `Speaker 1/2/…` in order of first appearance. With exactly two diarized speakers, if
  one's question rate is at least 3× the other's and it asks at least 10 questions, it
  is `Host` and the other is `Guest` (`roles_inferred = true`). Never guess names.
- **Paragraphs.** Merge consecutive same-speaker cues into paragraphs: close at the
  first sentence end after 95 words, and force a close at 160. A speaker change always
  closes a paragraph. Each paragraph keeps its first cue's start time.
- **Output.**
  `Transcript { format, paragraphs: [{start_secs, speaker, text}], words, roles_inferred, names_from_file }`.
- **Tests** use a trimmed real Pragmatic Engineer VTT (about the first 3 minutes) plus
  SRT, JSON, HTML, and text fixtures:
  - garbage is dropped with a reason;
  - diarized and voice-tagged files;
  - paragraph bounds;
  - the question-rate rule, both ways.

## transcript_dossier — Feed enrichment in the dossier and both routes

- **Builder.** With `enrich: auto`, the shared builder runs feed enrichment after the
  Spotify metadata, so it serves both create and ingest. A total 45 s deadline covers
  the feed lookup and the transcript download (10 MiB cap). On `Matched`:
  - **Notes.** The feed item's notes replace Spotify's when they are non-empty, and they
    are segmented by `notes.rs`.
  - **Chapters.** They come from valid Podcasting 2.0 chapters JSON, else from the
    inline run.
  - **Transcript.** It is fetched and parsed.
  - **Provenance.** Entries record each source: "Show notes: the publisher's RSS feed
    (<host>)", and "Transcript: publisher VTT from the show's RSS feed".
- **Drift guard.** Chapter and transcript times link to `?t=` only when |Spotify
  duration − feed duration| ≤ 10 s. Otherwise they are plain, and a muted note reads
  "Timestamps follow the publisher's file and may differ from Spotify's playback".
  Without a proven match, chapters taken from Spotify's notes keep their links.
- **Payload.** `episode.transcript` is
  `{format, words, roles_inferred, names_from_file, link_times, paragraphs, chapter_marks}`.
- **Renderer.**
  - The `## Transcript` heading starts a new page and becomes a bookmark.
  - Chapter `h3` headings, also bookmarks, sit at the paragraph nearest each chapter
    start within ±30 s; a chapter with no paragraph in that window is skipped.
  - Margin timestamps sit in a left gutter: Inter 7pt tabular, muted, or navy when they
    link.
  - An Inter 600 8pt small-caps speaker label appears at each speaker change.
  - Paragraphs use the serif body type.
  - Under the heading, one honest line reads: "Publisher transcript (VTT) from the
    show's RSS feed". It adds "· roles inferred, names not guessed" when roles were
    inferred, or "· speaker names from the file" when names came from the file.
- **Source view.** A `## Transcript` section with `**Speaker 1** (3:17)` paragraphs,
  using plain times.
- **Reports.** The capture line names everything that was attached, for example
  `capture: episode dossier · 16 chapters · publisher show notes · publisher transcript 18,736 words`.
  Each step-down prints exactly one warning:
  - `no public feed matched "<show>" (show notes from Spotify; no transcript)`
  - `the publisher feed has no transcript for this episode`
  - `transcript dropped: <reason>`
  - `feed lookup stopped after 45 s`

  With `enrich: off`, the line reads
  `capture: episode dossier (publisher feed lookup off)` and no warning prints.

- **Dry run.** A dry run looks up the feed but does not download the transcript, and
  prints `feed: <host> (matched on title, date)` and
  `transcript: publisher VTT (would attach)`.
- **Docs.** Add a publisher-feed section to `## Spotify episodes`, plus
  `highlights.podcasts.enrich` under Configuration.
- **Tests:**
  - iTunes → feed → VTT puts a transcript in the render payload;
  - a missed match warns once;
  - drift sets `link_times: false`;
  - `off` makes no iTunes request;
  - an exceeded deadline fails open with one warning;
  - the ingest path attaches a transcript;
  - the adapter self-test covers transcript assembly.
- **Live smoke test.** The Pragmatic Engineer episode yields a multi-page PDF, with the
  transcript on a new page. Rasterize that page and check it against the spec.

## rollout — Podcast surfaces across repos and the legacy refs

1. **Bob Mac Capture.** Open it with `/sase_repo` and read its `AGENTS.md`.
   - **Model.** `ref_type: podcasts` maps to a new `RefKind.podcast` in
     `Sources/RefsCore/RefsModel.swift`, with label "Podcast" and SF Symbol `mic`.
   - **Tint.** Purple, the platform's podcast color, unless another kind or status
     already uses it; then cyan. Edit `RefsVisualTokens.swift`.
   - **Ranking.** `["podcast", "episode"]` in `RefsRanking.swift`.
   - **Thumbnail.** Page 1 is the dossier card, so `RefsInspectorView.swift` shows it as
     the thumbnail.
   - **Tests.** Add `RefsModelTests` cases. The `Podcast` route chip already works.
   - **Gate.** Watch the exact-SHA macOS CI run until it is green with `/sase_monitor`,
     or gate first with `scripts/dev/macos-ci-check`.
2. **bob-plugins.** Open it with `/sase_repo`.
   - **Map.** `READING_BOARD_KIND_MAP` gains
     `podcasts: Object.freeze({ icon: "🎙️", label: "Podcast" })` in
     `plugins/bob-ledger-tools/src/242-plugin-reading-board-models.js`.
   - **Tests and fixtures.** Add a test in
     `scripts/test-ledger-tools-reading-board-models.cjs` and `"podcasts"` in the
     fixture `KINDS`.
   - **Build and deploy.** `npm run build`, `npm run build:check`, then
     `bob plugins sync`.
3. **Vault.** In `~/bob/refs.base`, insert a `/podcasts` → `"🎙️ Podcast"` branch into
   the `category` formula, before the `"🔖 Ref"` fallback, and let vault-sync commit it.
4. **Legacy refs.**

   > [!decision] legacy_refs Run the freshly built bob on each of
   > `ref/blogs/4ugvfwty2i8hgagsvt7pjc.md` and `ref/blogs/5ygcxo9nxh38ns7ghtbddm.md`:
   >
   > 1. `bob ref rename` onto its title stem. This keeps the `^ref-…` block ids, the
   >    closed tasks, and inbound links, including the zettel's wikilink.
   > 2. `bob ref create <canonical url> -f overwrite`, which swaps in the dossier and
   >    attaches a transcript if the feed matches.
   >
   > Then verify that the tasks are still closed, the links resolve, and each PDF is the
   > new dossier. The refs stay under `blogs/`, because no cross-type move exists.

   > [!decision] legacy_refs = no Skip this step and leave both refs untouched. They
   > already dedupe canonically.

5. **Decision record.**

   > [!decision] decision_record Use `/sase_memory_write` to add a `decisions` strand
   > titled "Podcast episodes are references keyed by platform ID". It covers:
   >
   > - the case-preserved canonical id and dedupe;
   > - episodes only, with other Spotify links refused or kept as ordinary items;
   > - enrichment only from a proven public publisher feed;
   > - no audio download, Web API, or in-app transcripts;
   > - `-L` and `-H` refused.
   >
   > Cite this plan and the research artifact, then run `sase memory init`.

   > [!decision] decision_record = no Write no memory.
