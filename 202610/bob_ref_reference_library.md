---
tier: epic
title: 'bob ref: a reference library for agents and Bryan'
goal: '`bob ref` is the canonical command for Bob''s reference library. `find`, `list`,
  and `show` answer "is this already in my library, what am I reading or planning
  to read, what have I finished, and what did I note?" correctly on the real vault.
  Each answer comes in beautiful human output, Markdown, or versioned JSON, and is
  honest about coverage. The six Highlights pipeline verbs work unchanged under `bob
  ref` and under the permanent `bob highlights` and `bob highlights-ref` aliases.
  Annotations no longer carry the leaked marker mirror, and a URL that only a legacy
  note records can be captured again. A deployed `bob_ref` skill makes "check the
  library first" the default step for agents that recommend reading.

  '
phases:
- id: rename
  title: Promote bob ref to the canonical command
  depends_on: []
  size: medium
  description: 'rename: make `ref` the canonical Vault command with permanent `highlights`
    and `highlights-ref` aliases, grouped help, canonical diagnostics, completion
    paths, fixtures, docs, and alias-equivalence tests.'
- id: region
  title: Managed-region and note-anatomy parser
  depends_on: []
  size: small
  description: 'region: a read-only parser for rendered ref-note bodies (annotation
    blocks with quote and comment kept apart, tombstones, mirror and preamble detection,
    the user''s own notes, and the Tasks section), round-tripped against the renderer.'
- id: index
  title: Read-only ref index, reading state, and identity
  depends_on:
  - region
  size: medium
  description: 'index: a new `ref_library` module that builds one read-only row per
    ref note (status precedence, derived reading state, identity keys, dates, origin,
    supersession, diagnostics, coverage) plus query resolution and title scoring,
    over a mixed-corpus fixture vault.'
- id: find
  title: bob ref find and the library CLI plumbing
  depends_on:
  - rename
  - index
  size: medium
  description: 'find: add the Library help group, shared output plumbing and JSON
    envelope, `docs/ref.md`, and batch identity lookup with verdicts, optional intake
    checks, and human, Markdown, and JSON output.'
- id: list
  title: bob ref list
  depends_on:
  - find
  size: medium
  description: 'list: the reading queue by default and filtered library views (reading
    state, status, type, origin, parent, since), with a consistent cap, opt-in Git
    dates, and grouped human output.'
- id: show
  title: bob ref show
  depends_on:
  - list
  size: medium
  description: 'show: exact resolution of one or more references and their metadata,
    annotations (quote and comment kept apart), the user''s own notes, and tasks,
    as human, Markdown digest, or JSON.'
- id: doctor
  title: Library health and coverage rows in doctor
  depends_on:
  - rename
  - index
  size: small
  description: 'doctor: add warning-level library rows to `bob ref doctor`: index
    totals, diagnostics, duplicate identities, leaked mirrors, and the count of unindexed
    zorg-era reading records outside the ref directory.'
- id: sync-fixes
  title: Remove leaked marker mirrors and stamp completion dates
  depends_on:
  - rename
  - region
  size: medium
  description: 'sync-fixes: fix bob-cli-4r so neither sidecar preambles nor marker
    mirrors render as annotations, drop previously leaked blocks without tombstones
    while keeping every genuine block ID stable, and stamp completion or cancellation
    dates when sync closes a `^ref` task.'
- id: legacy-capture
  title: Capture URLs that only a legacy note records
  depends_on:
  - rename
  size: small
  description: 'legacy-capture: shared dedupe refuses only PDF-backed note hits, so
    a URL recorded only by a legacy note captures with a warning instead of hitting
    the dead end introduced by the bob-cli-4s legacy `url:` dedupe.'
- id: skill
  title: The bob_ref agent skill
  depends_on:
  - show
  - legacy-capture
  size: small
  description: 'skill: author the `bob_ref` skill source in the linked chezmoi repo
    so reading-recommendation agents check the library first, label what they find
    honestly, and never write to the vault.'
- id: verify
  title: Live verification, install, and skill deployment on athena
  depends_on:
  - doctor
  - sync-fixes
  - skill
  size: small
  description: 'verify: run the acceptance exercise and the counts, performance, alias,
    and dry-run scan checks against the real vault; install the new bob on athena;
    deploy the skill; and record bead hygiene and follow-ups.'
proposed_by: bbugyi200.athena.research.3s.linker.w0
create_time: 2026-10-06 20:15:51
status: done
bead_id: bob-cli-4w
---

- **PROMPT:** [prompts/202610/bob_ref_reference_library.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_ref_reference_library.md)
- **BEAD:** [bob-cli-4w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4w/README.md)

# Plan: `bob ref`, a reference library for agents and Bryan

## Background

### Evidence used

- **Research.** The research is
  `research:202610/bob_ref_reference_library_migration/bob_ref_reference_library_migration.md`.
  Bryan agrees with every recommendation in it, and this plan adopts them. The
  research's recommended answers to its own open questions are adopted too:
  - The noun is `ref`.
  - The legacy-status mapping is the evidence-based one.
  - `ref/chat` agent reports are an interest signal, not reading history.
  - A URL that matches only a legacy note is captured with a warning instead of being
    refused.
  - Migrating the zorg-era records is a separate epic.
  - Agents propose captures and never write.
- **The bob-cli-4s epic** (closed; plan `plan:202610/highlights_create_listen.md`)
  landed after the research. It changes what this plan must do:
  - `create` is now the front door for every target: Markdown, a local PDF, a PDF URL,
    an arXiv URL, or a web article URL. Its `-L/--listen` option narrates the target and
    binds the audio. When the target is already captured, attach mode adds the audio to
    the existing capture.
  - Notes now carry `audio`, `author`, `published`, and `captured`. arXiv captures stamp
    `source_url: https://arxiv.org/abs/<id>[vN]`.
  - The shared dedupe module `src/native/highlights_ref/sources.rs` now reads the legacy
    `url:` key as well as `source_url`. `clip_url::dedupe_key_for` maps every arXiv URL
    spelling to `https://arxiv.org/abs/<id>` through `arxiv::ArxivPaper`. That is most
    of the shared identity the research asked for (A8), and the index reuses it.
  - **The dead end the research warned about is now live.** `check_dedupe` refuses every
    ref note that matches, so `clip` or `create` on any of the 275 legacy notes with a
    `url:` fails with "already captured as ref/ai/…". `--listen` then fails with "ref
    note has no resolvable source_pdf", because legacy notes have no Highlights PDF.
    Phase `legacy-capture` fixes this.
  - 4s added 38 hard-coded `bob highlights …` strings across `create.rs`, `clip.rs`,
    `stamp.rs`, `pdf_target.rs`, `companion.rs`, `listen.rs`, `attach.rs`, `doctor.rs`,
    `clip_adapter.rs`, and `cli.rs`. It also added `docs/highlights-create.md`. All of
    them take part in the rename.
- **Beads.**
  - `bob-cli-4r`: the leaked marker mirror, found in 113 of 115 annotated notes. Phase
    `sync-fixes` fixes it.
  - `bob-cli-39`: clip recapture and versioning. Only its legacy-only subset lands here.
  - `bob-cli-4a`: the JSON flag convention. This plan uses the house `-f/--format`.
  - `bob-cli-4b`: the noun policy. This plan uses `ref`.

### Facts the design rests on

These were checked in the code and in the vault (`~/bob`, read-only).

**The ref directory**

- `ref/` holds 595 notes:
  - `ai/` 282: legacy zorg migrations. They have no `^ref` line and no `ref_type`. They
    carry `status: "legacy"` plus `legacy_status`, `url:`, and
    `source_block: "^z-YYMMDD-…"`.
  - `chat/` 291
  - `docs/` 11
  - `blogs/` 6
  - `papers/` 5
- Modern notes look like this:
  - A frontmatter `status`.
  - One `- [c] #task #ref [[lib/….pdf]] #hide ^ref` tracker line.
  - A `## Highlights` heading over the
    `<!-- highlights:begin -->`…`<!-- highlights:end -->` region.
  - Optionally, a `## Tasks` section and an audio embed.
- Date coverage is thin:
  - `created` is on only 29 notes, `captured` on 2, and `research:` on 1 chat note.
  - 225 closed trackers have no `[completion::]` or `[cancelled::]` date.
- Some frontmatter values are YAML lists (`url:` on 10 notes) or block lists (`tags:`).
  The line-preserving Highlights parser (`frontmatter.rs`) ignores both. `serde_yaml`
  0.9 is already a dependency and is what Dataview uses.

**How the managed region renders** (`sidecar_render.rs`)

- `### <page label>` headings.
- `> [!quote] <text>` for a highlight. A comment becomes the separator line `>` followed
  by `> > [!note] Comment <text>`.
- `> [!note] <text>` for a standalone note.
- `> [!quote] Image ![[…assets/h-<id>.<ext>]]` for an image.
- Every block ends with `^h-<12 hex>`.
- Tombstones go under `### Removed highlights` as
  `> [!warning] Removed highlight This annotation is no longer present in the Highlights sidecar.`

**House CLI conventions**

- `-f/--format human|json` and compact one-line JSON carrying `"ok"` and an integer
  `"schema_version"`.
- Color through `style::Styler` (TTY plus `NO_COLOR`), widths through
  `style::terminal_width()` and `truncate`.
- The clock is `env::current_datetime()`, which honors `BOB_NOW`.
- Options are sorted by short flag, case-insensitively, with the uppercase letter first
  (`-L` before `-l`). Every long option has a short alias (`sase/memory/cli_rules.md`).
- Root aliases are argv-prefix rewrites (`runner.rs` `ALIASES`, `rewrite_alias_args`),
  so an alias is byte-identical by construction.

## Design decisions

1. **One noun, two help groups.** `bob ref` is canonical, in the root **Vault** section.
   - `bob highlights` and `bob highlights-ref` are permanent, silent, hidden aliases.
     The old `highlights-ref` rename was never aliased, and `tests/cli/vault_sync.rs`
     still asserts that it errors; that assertion is removed.
   - `bob ref -h` shows two groups: **Library** (`find`, `list`, `show`) and
     **Highlights pipeline** (`clip`, `create`, `doctor`, `marker`, `scan`, `sync`).
   - Bare `bob ref` keeps printing help.
   - Everything else that names the integration or stored merge state keeps its name:
     env vars (`BOB_HIGHLIGHTS_*`), the `highlights:` config, `<!-- highlights:* -->`
     markers, `highlights_*` frontmatter, `pipeline_version`, the module name, and the
     doc filenames.
2. **Library verbs are strictly read-only.**
   - `find`, `list`, and `show` never write, run hooks, or open PDFs. The one exception
     is `find -i`, which reads intake PDF markers.
   - They need none of Obsidian, pandoc, a browser, or Highlights.
   - The `^ref` checkbox plus `scan`/`sync` stay the only writers of reading state.
   - Queueing a reading is `bob ref create <URL|PDF|MD>`, with `clip` as the web-article
     specialist.
3. **Reading state is derived and shown with its evidence.** Every row carries:
   - the effective `status`;
   - the raw `legacy_status`;
   - a derived `reading_state`: `queued`, `started`, `finished`, `dropped`, or
     `unknown`;
   - a `reading_state_source` that records the evidence.

   Nothing new is stored. No field ever asserts "never read".

4. **Identity reuses bob-cli-4s.**
   - URL keys come from `clip_url::validate_and_clean` and `dedupe_key_for`, which are
     already arXiv-aware.
   - The index adds `arxiv:<id>` and `doi:<doi>` keys, plus opaque `raw:` keys for
     non-public values such as `go/` links.
   - `find` returns every match, each with its `match_kind`. A title match is only ever
     a candidate.
5. **Supersession is inferred, not stored (adjusts the research).** Suppose a PDF-backed
   note (one with `source_pdf`) and a note without a PDF share an identity key. The
   index then marks the PDF-less note `superseded_by` the PDF-backed one. This resolves
   today's _Harness engineering_ duplicate and every future legacy recapture with no
   vault writes. That is why `legacy-capture` adds no `supersedes:` marker field.
6. **Coverage is explicit.**
   - Every JSON response carries a `coverage` object stating:
     - only `<ref-dir>/**/*.md` is indexed;
     - absence is never proof of not having read something;
     - annotations are the last scan's snapshot.
   - `bob ref doctor` counts the zorg-era reading records outside `ref/`. The research
     counted about 425.
7. **Defaults are the same in every format (adjusts the research).**
   - Bare `bob ref list` is the reading queue (`queued` and `started`) whether the
     format is human, JSON, or Markdown.
   - Any filter option searches every reading state unless `-R` narrows it.
   - `--limit 50` is the default cap in every format, and `-A` lifts it.
   - The envelope always states `matched`, `returned`, `truncated`, and the effective
     filters.
8. **Git dates are opt-in, and only for modern notes (adjusts the research).**
   - `list -g` fills a missing `added` or `finished` date from one `git log` pass over
     `^ref` tracker lines.
   - Legacy notes have no tracker, so their dates stay with their zorg block date. Their
     Git history only shows the 2026-06 migration, which would be wrong evidence.
9. **`show` stays lean (adjusts the research).** `-c/--comments-only` and
   `-N/--no-annotations` are the only content flags.
   - `-a` is dropped: JSON already separates every field.
   - `-D` is dropped: tombstones carry no text, so they are counted instead of shown.
10. **External callers stay on `bob highlights`.** The cron script
    `maybe_bob_highlights_sync`, the `bob_xlib_pull` hook, the SASE research file hook
    (`bob highlights create --include-id`), and the Mac's scheduled scan all keep
    running through the alias. Rewriting them would break any machine whose installed
    bob predates `ref`. Only comments in the chezmoi config change.
11. **Beautiful means legible without color.**
    - Every state is a glyph plus a word (`✓ FINISHED`), and color only reinforces it.
    - Output is aligned, truncated to the terminal width, and drops low-value columns
      first.
    - Dim text marks secondary detail.
    - Stdout in JSON mode is pure JSON.

## Shared contracts

### Command tree and help

Root `bob -h`, Vault section, sorted:

```text
Vault:
  nightly     Run nightly maintenance: vault-sync, task archive, vault-sync
  query       Run Dataview or Tasks queries against the vault
  ref         Find, list, and read references; sync Highlights PDFs into them
  vault-sync  Reconcile the vault through Git (default: run) or show status
```

`bob ref -h`, as it reads once phase `show` lands:

```text
Find, list, and read Bob reference notes, and sync Highlights PDFs into them

Usage: bob ref [OPTIONS] <COMMAND>

Library:
  find    Look up URLs, arXiv IDs, DOIs, paths, or titles in the reference library
  list    List reference notes by reading state, status, type, origin, or date
  show    Show reference notes with metadata, annotations, and your own notes

Highlights pipeline:
  clip    Capture a web article URL into a Highlights intake PDF
  create  Create a Highlights-ready PDF from Markdown, a PDF, or a URL
  doctor  Check library health and Highlights sync prerequisites
  marker  Inspect the marker note for one PDF
  scan    Scan the configured Highlights library
  sync    Sync one PDF marker note into its Bob reference note

Options:
  -h, --help      Print help
  -n, --no-hooks  Ignore the configured highlights.pre_scan_hook

Examples:
  bob ref find https://arxiv.org/abs/1706.03762   Is this paper already in the library?
  bob ref list                                    Show the reading queue
  bob ref list -R finished -S 30d                 Show what was finished in the last 30 days
  bob ref show ea_graph -c                        Read your comments on one reference
  bob ref create <URL|PDF|MD> -L                  Capture a reference and narrate it
  bob ref scan                                    Sync Highlights PDFs into reference notes
```

**How help is built**

- A static `HELP_GROUPS` table in `src/native/highlights_ref/cli.rs` is the single
  source of the groups. It holds `(group title, &[subcommand names])`.
- The groups are rendered into a custom `help_template`, replacing clap's flat command
  list. Rows use the same style as root help (`runner.rs` `append_help_row`): a cyan
  literal name when color is on, wrapped at 80 columns.
- Subcommands stay **unhidden**, because completion walks them.
- A unit test requires that:
  - every registered subcommand appears in exactly one group;
  - names are alphabetical within each group;
  - each group's `about` equals the subcommand's clap `about`.

**Phasing of the help text**

- Phase `rename` ships the pipeline group and its examples.
- Phase `find` adds the Library group with its `find` row and example.
- Phases `list` and `show` add their rows and examples.
- Phase `doctor` changes the doctor row's `about`.

### Reading state

**Effective `status`, decided per note:**

1. **Exactly one `^ref` tracker with a known mark.**
   - The mark maps through the existing `PdfTaskLine` mapping:
     - `[ ]` → `ready`
     - `[*]` → `next`
     - `[/]` → `wip`
     - `[x]` or `[X]` → `read`
     - `[-]` → `abandoned`
   - A tracker is a list task line whose whitespace tokens include `^ref`. It is read
     outside fenced code and outside the managed region. The PDF wikilink is optional
     for reading.
   - Compare with the frontmatter `status`, normalized by `normalize_deprecated_status`
     (`unread` → `ready`, `done` → `read`):
     - **Equal** → `status_sync: ok`.
     - **Different, and frontmatter still equals the stored base** (or there is no base)
       → `status_sync: pending`. The tracker wins, matching what the next sync does. The
       base is the `status` inside the `highlights_marker_base` JSON.
     - **Different, and frontmatter also moved away from the base** → `status: conflict`
       and `status_sync: conflict`. Both values are reported, and `reading_state` is
       `unknown`.
2. **No usable tracker** (none, several, or an unknown mark): use the normalized
   frontmatter `status`. Several trackers or an unknown mark also add a diagnostic.
3. **Neither:** `status: null`, and `reading_state` is `unknown`.

`era` is `legacy` when there is no tracker and the frontmatter status is `legacy`.
Otherwise it is `modern`.

**Reading state mapping (derived, never stored):**

| `reading_state` | From modern status                                 | From `legacy_status` (case-insensitive)             |
| --------------- | -------------------------------------------------- | --------------------------------------------------- |
| `queued`        | `ready`, `next`                                    | `unread`                                            |
| `started`       | `wip` (a sticky lane, not proof of active reading) | `collect_fleeting_notes`                            |
| `finished`      | `read`                                             | `read`, `review_fleeting_notes`, `review_lit_notes` |
| `dropped`       | `abandoned`                                        | `abandoned`                                         |
| `unknown`       | none, or `conflict`                                | `book`, missing, or unrecognized                    |

**`reading_state_source`** names the evidence, for example:

- `ref_task:[x]`
- `frontmatter:read`
- `legacy_status:review_lit_notes`
- `conflict:ref_task=[x],frontmatter=ready`
- `none`

**Most-advanced order**, used to pick a primary match: `finished`, `started`, `queued`,
`dropped`, `unknown`.

### Identity

- **Stored values** come from `source_url` and `url`. Each may be a scalar or a list,
  and lists are flattened.
- **For an http(s) value**, use `clip_url::validate_and_clean` to get the `url_key`
  (`dedupe_key`, already arXiv-aware). Then add:
  - `arxiv:<id>` (versionless) when `ArxivPaper::parse` succeeds;
  - `doi:<lowercased doi>` for `doi.org` and `dx.doi.org` paths, percent-decoded;
  - `arxiv:<id>` as well for arXiv DOIs of the form `10.48550/arXiv.<id>`.
- **For a value that fails validation** (a private or `go/` host, or not a URL at all),
  the key is `raw:<lowercased, trimmed, scheme-less, trailing-slash-trimmed value>`.
  Such a value also adds an `opaque_url` diagnostic.
- **Query classification** (one shared function, in this order):
  1. An `http(s)://` URL → `url`.
  2. An arXiv ID, bare or with an `arxiv:` prefix (new style `\d{4}\.\d{4,5}(v\d+)?`,
     old style `archive/\d{7}`) → `arxiv`.
  3. A DOI (`10.\d{4,9}/\S+`, optionally `doi:`-prefixed) → `doi`.
  4. A path: it contains `/`, ends with `.md` or `.pdf`, or is absolute → `path`. It
     matches the note path (vault-relative, `.md` optional, or absolute inside the
     vault) and `source_pdf`.
  5. A single token → `name`. It matches the note stem or the frontmatter `id`,
     case-insensitively.
  6. Anything else → `title`.
- **Title scoring** applies to the `name` and `title` kinds, and to URL misses through
  their slug.
  - Normalize: casefold, turn every non-alphanumeric run into a space, then drop the
    stopwords `a an and for in is of on the to with`.
  - Score: `round(100 × 2|Q∩T| / (|Q|+|T|))` over the two token sets.
  - The score is raised to at least 90 when one normalized string contains the other and
    the shorter has 3 or more tokens.
  - Normalized equality scores 100 and has the match kind `title_exact`.
  - At most 5 candidates are kept, ordered by score.
- **Slug fallback.** A URL query with no identity match takes the words of its last
  non-empty path segment, without extension and split on `-` and `_`, when there are at
  least 2 words. Those words are title-scored as `slug_title` candidates.
- **Match kinds:**
  - exact: `identity`, `path`, `source_pdf`, `id`, `stem`;
  - candidates: `title_exact`, `title`, `slug_title`;
  - intake only: `intake`.

### Ref index rows

**Membership**

- `<ref-dir>/**/*.md`.
- Skipped: hidden directories, `*.assets/` directories, and conflict copies (file names
  containing ` (conflict`, ` (Conflicted copy`, or `.sync-conflict-`). Skips are counted
  in `coverage.skipped`.
- A note that cannot be read or parsed still yields a row, with a diagnostic. It is
  never silently dropped.

**Frontmatter**

- Parse the block that `split_frontmatter` delimits with `serde_yaml`.
- Normalize unquoted wikilinks (`parent: [[x]]`, which YAML reads as nested sequences)
  back to `[[x]]`.
- On invalid YAML, fall back to `parse_frontmatter_entry` line by line and add an
  `invalid_yaml` diagnostic.

**Row fields** (serde field order as listed; JSON uses these names):

| Field                                         | Rule                                                                                                                                                                                                                                                           |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `path`, `link`                                | Vault-relative path (`ref/papers/ea_graph.md`) and `[[ref/papers/ea_graph]]`                                                                                                                                                                                   |
| `title`                                       | Frontmatter `title`, else the first H1, else `clip_url::humanize_stem(stem)`                                                                                                                                                                                   |
| `origin`                                      | `agent-report` when under `<ref-dir>/chat/` or `ref_type: chat`, else `external`                                                                                                                                                                               |
| `ref_type`                                    | Frontmatter `ref_type`, else the first directory under the ref dir, else `null`                                                                                                                                                                                |
| `era`                                         | `modern` or `legacy` (see Reading state)                                                                                                                                                                                                                       |
| `status`, `status_sync`, `frontmatter_status` | See Reading state. `frontmatter_status` is present only when `status_sync` is not `ok`                                                                                                                                                                         |
| `legacy_status`                               | Raw value as written, or `null`                                                                                                                                                                                                                                |
| `reading_state`, `reading_state_source`       | See Reading state                                                                                                                                                                                                                                              |
| `parent`                                      | Bare note name from `parent` (`[[sase_ref]]` → `sase_ref`). The first entry if it is a list                                                                                                                                                                    |
| `urls`                                        | Every `source_url` value, then every `url` value, as written                                                                                                                                                                                                   |
| `identity`                                    | `{ "keys": [...], "arxiv": id or null, "doi": doi or null }`                                                                                                                                                                                                   |
| `author`, `published`, `captured`             | Frontmatter strings, or `null`                                                                                                                                                                                                                                 |
| `added`, `added_source`                       | `captured`, else the date part of `created`, else the zorg `source_block` date (`^z-YYMMDD-…` → `20YY-MM-DD`), else Git (`list -g`, modern notes only), else `null`. The source is `captured`, `created`, `zorg_block`, or `git`                               |
| `finished`, `finished_source`                 | Only for `finished` and `dropped`. The tracker's `[completion:: D]` or `[cancelled:: D]` (`✅ D` and `❌ D` also accepted), else Git (`list -g`), else `null`. The source is `ref_task` or `git`                                                               |
| `source_pdf`, `audio`                         | Frontmatter values with the wikilink brackets stripped, or `null`                                                                                                                                                                                              |
| `annotation_count`, `comment_count`           | Live highlight, image, and note blocks, excluding mirror-shaped, preamble, and tombstone blocks. `comment_count` counts highlights with a comment plus standalone notes                                                                                        |
| `snapshot`                                    | `{ "synced_at": highlights_synced_at, "highlights_count": raw }`, or `null` if the note was never synced                                                                                                                                                       |
| `research_ref`                                | `research:<value>` when frontmatter `research` is set, else `null`. Nothing is guessed                                                                                                                                                                         |
| `superseded_by`                               | Path of the PDF-backed note that shares an identity key, else `null`. If several PDF-backed notes share the key, the first by path is used and a `duplicate_identity` diagnostic is added                                                                      |
| `diagnostics`                                 | `[{ "code": "...", "detail": "..." }]` with stable codes: `invalid_yaml`, `multiple_ref_trackers`, `unknown_ref_mark`, `status_conflict`, `marker_mirror_excluded`, `preamble_excluded`, `unparsed_region`, `opaque_url`, `duplicate_identity`, `missing_type` |

### JSON envelope

There is one `REF_SCHEMA_VERSION = 1`, shared by `find`, `list`, and `show`. Bump it
only for a breaking change; new optional fields keep the version. Output is compact
one-line JSON on stdout, with no ANSI and no prose. Warnings go to stderr.

```json
{"ok":true,"schema_version":1,"command":"ref list","generated_at":"2026-10-06T19:30:00",
 "coverage":{"ref_dir":"ref","notes":595,"skipped":0,"intake":"not_checked",
   "scope":"Only notes under ref/ are indexed. Reading history elsewhere in the vault (for example zorg-era status:: records) is not; run `bob ref doctor` for a count. Absence from this index is never proof that something was not read.",
   "annotations":"Annotations are the snapshot written by the last Highlights scan, not live PDF state."},
 "filters":{...},"matched":6,"returned":6,"truncated":false,"hidden":{"superseded":1},
 "library":{"notes":594,"finished":285,"started":92,"queued":150,"dropped":21,"unknown":46},
 "refs":[ ...rows... ]}
```

- **`generated_at`** comes from `env::current_datetime()`, as `YYYY-MM-DDTHH:MM:SS`
  local time. Tests pin it with `BOB_NOW`.
- **`coverage.intake`** is one of:
  - `not_checked`;
  - `checked`;
  - `unavailable` (no intake directory).
- **`library`** counts exclude superseded notes.
- **`find`** replaces `filters` / `matched` / `refs` with `summary` and `results` (see
  Phase find).
- **`show`** rows extend the base row (see Phase show).
- **Errors in JSON mode** print
  `{"ok":false,"schema_version":1,"command":"…","error":{"code":"…","message":"…","hint":"…"}}`
  to stdout and exit 1. An ambiguous `show` reference adds
  `"candidates":[{"path":…,"title":…}]`.

### Errors and exit codes

- **Exit codes:**
  - 0 whenever the lookup or listing ran, including no matches and `not_found`.
  - 1 for I/O, vault, or resolution failures (a missing ref dir, an ambiguous or unknown
    `show` reference).
  - 2 for usage errors (clap).
- **Human errors** go to stderr as `bob ref: error: <message>`, optionally followed by
  `hint: <next step>`, the same shape the bob-cli-4s commands use.
- **A missing ref dir** gives `reference directory not found: <path>` with the hint
  `pass -b/--bob-dir or -r/--ref-dir, or set BOB_DIR`.
- **Config:** build the library config only from `bob-dir`, `ref-dir`, and (for `find`)
  `xlib-dir`. Do not call `Config::from_matches` on matches that lack `lib-dir`, because
  clap panics on undefined ids.

## Phase rename: Promote bob ref to the canonical command

**Runner and module**

- In `src/runner.rs`:
  - Change the `SUBCOMMANDS` row to `name: "ref"`, `section: Section::Vault`, and the
    root `about` shown in [Command tree and help](#command-tree-and-help). Keep the
    table sorted: `nightly`, `query`, `ref`, `vault-sync`.
  - Add `Alias { from: "highlights", to: &["ref"] }` and
    `Alias { from: "highlights-ref", to: &["ref"] }`.
  - Keep `NativeCommand::Highlights` internally. Do not mount the leaf twice.
- In `src/native/highlights_ref/mod.rs`, set `COMMAND_NAME = "bob ref"`. The completion
  test `descriptor_names_match_canonical_paths` requires this.

**Hard-coded strings**

- Route every hard-coded `bob highlights` string in `src/` through canonical wording.
  That covers the 38 strings listed under Background:
  - `next: bob highlights scan`;
  - the five `"bob highlights: {}"` error prefixes in `create.rs` and `clip.rs`;
  - hints in `stamp.rs`, `pdf_target.rs`, `companion.rs`, `listen.rs`, `attach.rs`,
    `doctor.rs`, and `clip_adapter.rs`;
  - help text in `clip.rs`, `create.rs`, and `cli.rs`.
- Prefer `format!("{COMMAND_NAME} …")` over new literals.
- Doc comments may say `bob ref`.
- Leave env vars, config keys, markers, frontmatter fields, `PIPELINE_VERSION`, and
  `highlights://` alone.

**Help**

- Set `build_cli()`'s `about` to "Find, list, and read Bob reference notes, and sync
  Highlights PDFs into them".
- Implement the `HELP_GROUPS` template with the single **Highlights pipeline** group and
  its examples.
- Give `clip`'s `about` the wording in the target help ("Capture a web article URL into
  a Highlights intake PDF") if it differs today.
- Reword the shared `ref_dir_arg` help to "Reference note directory; defaults to
  BOB_HIGHLIGHTS_REF_DIR or ref". The library verbs reuse it, and "output" is wrong for
  a reader.

**Completion**

- In `src/native/completion/kinds.rs`, change the paths `["highlights","clip"]` and
  `["highlights","create"]` to `["ref",…]`, and update their comments.
- `every_table_entry_matches_the_tree` and `every_value_arg_has_a_decision` must pass.
  `bob-cli-4j`'s known `create:audio` failure is pre-existing; leave it alone.

**Tests**

- `tests/cli/aliases.rs`:
  - Add `(["highlights"],["ref"])` and `(["highlights-ref"],["ref"])` to the
    byte-for-byte `--help` and `--not-a-real-flag` pairs.
  - Add both names to the root-completion exclusion list.
- Add an equivalence test in a new `tests/cli/ref_library/` directory. (`ref` is a Rust
  keyword, so it cannot be a module name.) For all six verbs, under a fixed `BOB_NOW`,
  `bob highlights <verb> …` and `bob ref <verb> …` must produce identical stdout,
  stderr, exit status, and resulting files. Cover:
  - `--help` for each verb;
  - one error path per verb;
  - `doctor` and `scan --dry-run` on a temp vault;
  - the parent `--no-hooks` placement.
- Assert that no deprecation text appears anywhere.
- Remove `highlights-ref` from `renamed_old_top_level_commands_are_unknown`
  (`tests/cli/vault_sync.rs`).
- Update output-string assertions that print the canonical path:
  - `bob highlights: error:` in `tests/cli/highlights/{clip,create,listen}.rs`;
  - `Usage: bob highlights` in `tests/cli/help.rs`;
  - `bob highlights scan` in `tests/cli/help_options.rs`.
- Update the root help fixtures `tests/fixtures/help/root-short.txt` and
  `root-long.txt`: move the row from Integrations to Vault.
- `tests/cli/help.rs` root lists: `ref` becomes a root and `highlights` becomes an alias
  route.
- Leave the ~290 `.arg("highlights")` invocations in `tests/cli/highlights/` as they
  are. They become a free alias regression suite.

**Docs and justfile**

- `README.md`:
  - TOC.
  - The command tables: move the row to Vault.
  - Rename the `## Highlights` section to `## Reference library (bob ref)`.
  - The env and deps prose.
  - The docs index.
  - Add the two rows to the alias table, and reword the paragraph that says
    `highlights-ref` is no longer registered.
- `docs/README.md`.
- The command spellings in `docs/highlights-ref-sync.md`, `docs/highlights-clip.md`,
  `docs/highlights-create.md`, `docs/vault-git-sync.md`, `docs/task-status-hooks.md`,
  `docs/projects.md`, `docs/getting-started.md`, `docs/completion.md`, and
  `docs/freshness.md`. Where a doc describes the external cron or hook callers, note
  that they intentionally keep `bob highlights` (design decision 10).
- `justfile` `install-smoke`: add `ref --help`, `ref clip --help`, and
  `ref create --help` lines, and keep the `highlights` lines.

**Done when:** `just all` passes, apart from the known pre-existing `bob-cli-4j` /
`bob-cli-4u` / `bob-cli-40` failures, and `bob highlights-ref -h` prints `bob ref` help.

## Phase region: Managed-region and note-anatomy parser

Add `src/native/highlights_ref/region.rs` as a `pub(crate)` module. It sits beside the
renderer that owns the format, and it is read-only.

**Region blocks**

- `parse_managed_region(region: &str) -> RegionParse` with
  `RegionParse { blocks: Vec<RegionBlock>, removed: Vec<String>, unparsed: Vec<String> }`.
- `RegionBlock` has these fields:
  - `page_label: Option<String>`
  - `kind`: `Highlight`, `Note`, or `Image`
  - `quote: Option<String>`: the author's words. Multi-line, joined with `\n`, with the
    `> ` and `>` continuation markers stripped.
  - `comment: Option<String>`: the user's words, from the nested `> > [!note] Comment …`
    lines.
  - `asset: Option<String>`: the image embed target.
  - `block_id: String` (`h-…`)
  - `mirror: bool`
  - `in_preamble: bool`
- **Standalone notes.** For a `[!note]` standalone note the text goes in `comment`, and
  `quote` is `None`.
- **Tombstones.** Blocks under `### Removed highlights` are tombstones. Only their IDs
  go in `removed`.
- **Mirror detection.** `mirror` is true when a `Note` block's text, or a `Highlight`'s
  comment, parses with `marker::parse_marker` and carries both `status` and `parent`. It
  shares one predicate with phase `sync-fixes`: `is_marker_mirror_text(&str) -> bool`.
- **Preamble.** `in_preamble` marks blocks that appear before the first `### ` page
  heading in a region that has page headings. `### Removed highlights` is not a page
  heading.
- **Unparsed content.** A block it cannot classify goes into `unparsed` (raw text). It
  is never dropped silently.

**Note anatomy**

- `split_note_body(body: &str) -> NoteParts` with these fields:
  - `h1`
  - `tracker_line`
  - `region: Option<String>`, reusing the begin/end constants and the duplicate-marker
    rules of `note.rs` `managed_region`
  - `own_notes: String`
  - `tasks: Vec<RegionTask>`
- `own_notes` is the body outside the region minus:
  - the first H1;
  - the `^ref` tracker line;
  - audio embeds (`![[….mp3|m4a|ogg|opus]]`);
  - the `## Highlights` heading directly above the begin marker;
  - the `## Tasks` section.

  Runs of blank lines collapse and the result is trimmed. Legacy migrated bodies pass
  through verbatim.

- `RegionTask` is `{ checked, mark, text, block_id }`. It is parsed from the `## Tasks`
  section's `- [c] … [[#^h-…|🔖]] [h:: …] [created::…]` lines. `text` has the link and
  the inline fields stripped.

**Tests**

- Round-trip: render with `render_sidecar_highlights` on fixtures for every block kind:
  - a highlight with a multi-line comment;
  - a standalone note;
  - an image;
  - several pages;
  - tombstones;
  - a leaked mirror shaped exactly like the `ref/papers/ea_graph.md` example in
    bob-cli-4r.

  Then parse the output back, and assert every field and every ID.

- Unknown callout types land in `unparsed`.

**Done when:** `just all` passes. No CLI behavior changes.

## Phase index: Read-only ref index, reading state, and identity

Create `src/native/ref_library/` and register it in `src/native.rs`. It is read-only,
needs no lopdf, and spawns nothing; the only exception is phase `list`'s Git pass.

**Files**

- `mod.rs`: `RefIndex`, `build_index(&LibraryConfig) -> Result<RefIndex, LibraryError>`.
  - `LibraryConfig` is `{ bob_dir, ref_dir, xlib_dir }`, resolved like the Highlights
    config: `BOB_DIR`, `BOB_HIGHLIGHTS_REF_DIR`, `BOB_HIGHLIGHTS_XLIB_DIR`, then the
    defaults.
- `frontmatter.rs`: the serde_yaml parse, wikilink normalization, and line-parser
  fallback.
- `status.rs`: tracker detection, status precedence, base comparison, and reading-state
  mapping. Reuse the `PdfTaskLine` mark mapping and `normalize_deprecated_status`
  through narrow `pub(crate)` seams.
- `identity.rs`: stored keys and query classification.
- `resolve.rs`: exact resolution, title scoring, the slug fallback, and primary-match
  ordering. The ordering is:
  1. non-superseded first;
  2. then by the most-advanced order;
  3. then by path.
- `row.rs`: the `RefRow` serde model, dates, origin, diagnostics, and `Coverage`.

**What the index computes**

- `annotation_count` and `comment_count` come from phase `region`'s parser.
- Supersession is computed after all rows exist.
- Library counts and `coverage` are computed once per build.

**Performance**

- Read each note once.
- Target under 100 ms for 600 notes in a release build. The research measured about 10
  ms of raw reads.

**Fixture vault** Add `tests/fixtures/ref_library/vault/` (`ref/…` plus a root
`*_ref.md` hub, which must not be indexed). It contains these notes:

|   # | Note                                                                                                                                                       |
| --: | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
|   1 | A synced paper with a list-valued `url` and a leaked mirror block                                                                                          |
|   2 | A clipped blog with `source_url`, `audio`, `author`, `published`, `captured`, and real highlights, comments, a standalone note, an image, and tombstones   |
|   3 | A legacy unread note whose `url:` differs from note 2's `source_url` only by `www.` and a trailing slash, so it becomes superseded                         |
|   4 | Legacy notes for each of `collect_fleeting_notes`, `review_lit_notes`, `review_fleeting_notes`, `read`, `abandoned`, `book`, and a missing `legacy_status` |
|   5 | Chat notes in `next`, `ready`, `read`, and `abandoned`, one of them with `research:`                                                                       |
|   6 | A note whose checkbox and frontmatter disagree with frontmatter equal to the base (pending)                                                                |
|   7 | A note where both sides moved (conflict)                                                                                                                   |
|   8 | A note with two `^ref` trackers                                                                                                                            |
|   9 | A note with an unknown mark `[?]`                                                                                                                          |
|  10 | Duplicate stems in two directories                                                                                                                         |
|  11 | Malformed YAML                                                                                                                                             |
|  12 | A note with an unquoted `parent: [[x]]`                                                                                                                    |
|  13 | A `go/` link `url`                                                                                                                                         |
|  14 | An arXiv `/pdf/` URL, and an arXiv DOI on another note                                                                                                     |
|  15 | `[completion::]` and `[cancelled::]` dates, plus a `✅` date                                                                                               |
|  16 | A `created` timestamp                                                                                                                                      |
|  17 | A zorg `source_block`                                                                                                                                      |
|  18 | A conflict-copy file, a hidden directory, and a `*.assets/` directory                                                                                      |

**Tests**

- One assertion per row field and per diagnostic code.
- Query classification and scoring tables, including `abs`/`pdf`/`html`/`vN` arXiv
  variants, DOIs, `www.`/slash variants, and `go/` links.
- Supersession of fixture 3.
- Determinism: the same index twice gives identical serialized JSON.
- A side-effect check: hash every fixture file before and after building the index.

**Done when:** `just all` passes.

## Phase find: bob ref find and the library CLI plumbing

**Plumbing**

- `src/native/ref_library/cli.rs` holds the builders for the three library subcommands.
  This phase registers only `find`.
- Shared args reuse `highlights_ref::cli::{bob_dir_arg, ref_dir_arg, xlib_dir_arg}`,
  made `pub(crate)`, so help strings stay identical.
- `-f/--format human|json|markdown`, default `human`.
- Add a `find` dispatch arm in `highlights_ref::run`.
- `output.rs` holds:
  - the envelope writer;
  - the error writer (human to stderr, JSON to stdout);
  - shared human styling: state chips through `Styler` (see Human chips below), dim
    paths, and fit-to-width helpers modeled on `note_ready/render.rs`;
  - Markdown escaping for table cells.
- Add the **Library** group, with the `find` row and example, to `HELP_GROUPS`.

**CLI**

```text
bob ref find [OPTIONS] <QUERY>...

Look up URLs, arXiv IDs, DOIs, paths, or titles in the reference library

Arguments:
  <QUERY>...  URL, arXiv ID, DOI, vault path, note stem or id, or title words; `-` reads one query per line from stdin

Options:
  -b, --bob-dir <PATH>     Bob vault root; defaults to BOB_DIR or ~/bob
  -f, --format <FORMAT>    Output format [default: human] [possible values: human, json, markdown]
  -i, --include-intake     Also check queued intake PDFs (reads their markers)
  -m, --min-score <SCORE>  Minimum title-candidate score, 1-100 [default: 60]
  -r, --ref-dir <PATH>     Reference note directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -x, --xlib-dir <PATH>    Highlights PDF intake directory; defaults to BOB_HIGHLIGHTS_XLIB_DIR or xlib
  -h, --help               Print help
```

The after-help explains three things:

- the verdicts;
- that a title match is a candidate, never proof;
- that `not found` means only "not under ref/".

It also gives three examples:

- a single URL;
- a batch from stdin, `printf '%s\n' URL1 URL2 | bob ref find - -f json`;
- `-i`.

**Behavior**

- Queries are processed in order, and duplicate queries are kept.
- **Verdict per query:**
  - `in_library`: at least one exact match.
  - `in_intake`: only an intake hit, with `-i`. Intake reuses `sources.rs`'s intake
    collection (marker `source_url` and `url`), made `pub(crate)`. A missing intake dir
    gives `coverage.intake: unavailable`.
  - `possible`: only title, `title_exact`, or slug candidates scoring at least
    `--min-score`.
  - `not_found`: nothing.
- **Per-query `reading_state`** is the primary exact match's state, or `null`.

**JSON**

```json
{"ok":true,"schema_version":1,"command":"ref find","generated_at":"…","coverage":{…},
 "summary":{"queries":4,"in_library":2,"finished":1,"in_intake":0,"possible":1,"not_found":1},
 "results":[{"query":"https://arxiv.org/html/2608.04278v1","query_kind":"url",
   "keys":["https://arxiv.org/abs/2608.04278","arxiv:2608.04278"],
   "verdict":"in_library","reading_state":"finished",
   "matches":[{"match_kind":"identity","matched_key":"arxiv:2608.04278","ref":{…row…}}],
   "candidates":[{"match_kind":"title","score":72,"ref":{…row…}}],
   "intake":[{"path":"xlib/blogs/x.pdf","source_url":"…","title":"…","status":"ready"}]}]}
```

**Human output** (one block per query; the glyph and word always appear, and color only
reinforces them):

```text
bob ref find · 4 queries · ref/ (595 notes)

  ✓ FINISHED   https://arxiv.org/html/2608.04278v1
               EA-Graph: Artifact-Anchored Verification Memory for Coding Agents…
               ref/papers/ea_graph.md · arXiv 2608.04278
  ○ QUEUED     https://stevekinney.com/writing/agent-memory-systems
               Memory Systems for AI Agents · ref/blogs/steve_kinney_agent_memory.md · READY
  ≈ POSSIBLE   attention is all you need
               2 candidates · best 72 “Attention Is Not All You Need” · ref/papers/…
  · NOT FOUND  https://example.com/post

  2 of 4 in library (1 finished · 1 queued) · 1 possible · 1 not found · intake not checked
```

- A superseded secondary match prints a dim line under its primary:
  `also ref/ai/agent_ref/openai_harness_engineer.md (legacy, superseded)`.
- The dated suffix is `read 2026-10-03` or `queued since 2026-10-05` when a date is
  known.

**Markdown output**

- A table with the columns Query | Verdict | Reading state | Reference. The Reference
  cell is the wikilink plus the title.
- Followed by the line
  `Library check: N of M in library (K finished) · coverage: ref/ only`.

**Human chips** (`output.rs`, shared by all three verbs):

| Chip          | Color  |
| ------------- | ------ |
| `✓ FINISHED`  | green  |
| `▸ STARTED`   | yellow |
| `○ QUEUED`    | cyan   |
| `✗ DROPPED`   | dim    |
| `? UNKNOWN`   | dim    |
| `≈ POSSIBLE`  | yellow |
| `· NOT FOUND` | dim    |
| `⇣ IN INTAKE` | cyan   |

- Status chips are `NEXT`, `READY`, `WIP`, `READ`, `ABANDONED`, `LEGACY`, and
  `CONFLICT`.
- Glyphs must be single-column.
- Pending-sync rows add a dim `*` and a one-line footnote.

**Completion entries**

- `["ref","find"] query` → `FreeText`
- `min-score` → `FreeText`
- `format` is already `Choices`. Add any new format value description at `kinds.rs`
  `format` descriptions.

**Docs**

- New `docs/ref.md`: purpose, coverage, the reading-state and status-precedence tables,
  identity keys, the JSON envelope and row schema, exit codes, and a find section.
- Add it to the `docs/README.md` and `README.md` doc indexes.
- Add a library subsection to the README section.

**Tests** (`tests/cli/ref_library/find.rs`)

- Every verdict.
- Every match kind.
- arXiv `abs`/`pdf`/`html`/`vN` variants.
- `www.` and trailing-slash variants.
- A DOI and an arXiv DOI.
- `go/` keys.
- Batch from stdin.
- `-i` with a marker-stamped intake PDF generated at runtime (see the existing
  highlights tests' lopdf helpers), and `-i` with no intake dir.
- `--min-score`.
- Exit 0 on not found, exit 1 on a missing ref dir.
- JSON validity with diagnostics.
- No ANSI on non-TTY output.
- Byte-stable JSON under `BOB_NOW`.
- The side-effect check against the fixture vault.
- A colored-render unit test using `Styler::colored()`.

## Phase list: bob ref list

**CLI**

```text
bob ref list [OPTIONS]

List reference notes by reading state, status, type, origin, or date

Options:
  -A, --all                       Show every matching note instead of the first --limit
  -b, --bob-dir <PATH>            Bob vault root; defaults to BOB_DIR or ~/bob
  -f, --format <FORMAT>           Output format [default: human] [possible values: human, json, markdown]
  -g, --git-dates                 Fill missing added/finished dates from vault Git history (slower)
  -n, --limit <N>                 Show at most N notes [default: 50]
  -o, --origin <ORIGIN>           Only external references or agent reports [possible values: external, agent-report]
  -P, --parent <NOTE>             Only notes whose parent is this bare note name
  -R, --reading-state <STATE>...  queued, started, finished, dropped, unknown, or all (comma-separated)
  -r, --ref-dir <PATH>            Reference note directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -S, --since <DATE>              Only notes whose row date is on or after DATE (YYYY-MM-DD, 7d, 4w, 6m, 1y)
  -s, --status <STATUS>...        ready, next, wip, read, abandoned, legacy, conflict, unknown (comma-separated)
  -t, --ref-type <TYPE>...        Library subdirectory, such as papers, blogs, docs, chat, or ai (comma-separated)
  -h, --help                      Print help
```

- `-A` conflicts with `-n`.
- The after-help states the default view rule and gives examples:
  - `bob ref list`
  - `bob ref list -R finished -S 30d -g`
  - `bob ref list -o external -R finished -f json`
  - `bob ref list -s legacy -R queued`

**Semantics**

- **Combining filters.** Values within one option are ORed. Different options are ANDed.
- **The default view.** With **no filter option at all**, the filter is
  `-R queued,started` (the reading queue), in every format. Any filter option searches
  all states unless `-R` is given. The envelope's `filters` records
  `reading_state_defaulted`.
- **Superseded notes** are always excluded and counted in `hidden.superseded`. `find`
  and `show` still reach them.
- **Row date:** `finished` for the finished and dropped states, `added` otherwise.
- **`--since`:**
  - It accepts `YYYY-MM-DD` or `<N>d|w|m|y`, relative to `env::current_datetime()`.
    Calendar months use chrono `checked_sub_months`.
  - Rows without a date are excluded and counted in `filters.undated_excluded`.
- **Order** (applied to JSON, Markdown, and human output alike):
  1. reading state: started, queued, finished, dropped, unknown;
  2. modern before legacy;
  3. within queued, `next` before `ready`;
  4. row date, newest first, with undated rows last;
  5. title.
- **`--limit`** applies after filtering and ordering.

**`-g/--git-dates`**

- Make one
  `git -C <bob_dir> log --format=%x1e%cs -p -U0 --no-color --no-ext-diff -G'\^ref' -- <ref-dir>`
  pass and parse it per file:
  - `added` is the date of the earliest commit that added a `^ref` line;
  - `finished` is the date of the newest commit whose added `^ref` line carries the mark
    of the current finished or dropped state.
- It fills only missing values, only on modern notes, with source `git`.
- When `git` is missing, the vault is not a repo, or the pass fails, set
  `coverage.git_dates: "unavailable"` and print one stderr warning. This is never fatal.

**Human output**

```text
bob ref · reading queue · 6 open · 594 in library

  QUEUED  6
    NEXT   2026-10-05  The `sase ace` Startup Critical Path                  chat    ref/chat/ace_startup_critical_path.md
    NEXT   2026-10-04  Memory and Instruction File Inspiration Reading List  chat ♫  ref/chat/memory_and_instruction_file…
    READY  —           Sase Paper Follow-up Reading List                     chat    ref/chat/sase_paper_followup_reading_list.md

  + 236 zorg-era legacy notes hidden (92 started · 144 queued) · show them with -s legacy

  library  285 finished · 92 started · 150 queued · 21 dropped · 46 unknown · coverage ref/ only
```

- **Header.** The view name is `reading queue` for the default and `N matching notes`
  otherwise.
- **Groups.** One heading per reading state with its count, in a chip color. Empty
  groups are omitted.
- **Rows:**
  - columns: status chip, row date (or `—`), title, `♫` when there is audio, `ref_type`,
    and a dim path;
  - drop the path column first when the terminal is narrow, then truncate titles;
  - a pending-sync `*` gets a footnote.
- **Legacy collapse.** When `-s` is not given, legacy-era rows are summarized in one dim
  line instead of listed. The cap applies to the rows that are listed.
- **Truncation** adds `… N more · -n N or -A to show more`.
- **Empty queue:** `Nothing queued ✓`.

**Markdown output**

- A table with the columns State | Status | Date | Title | Type | Note.
- Followed by the matched/returned line and the coverage line.

**Completion**

- Add path-specific entries `["ref","list"] origin` → `Choices`. This overrides the
  global `origin` → `VaultNote` entry.
- `reading-state` → `Choices`
- `since` and `limit` → `FreeText`

**Docs:** a list section in `docs/ref.md`.

**Tests** (`tests/cli/ref_library/list.rs`)

- The default view in all three formats.
- Every filter, and OR-within / AND-across.
- `all`.
- `unknown` and `conflict`.
- Superseded hiding.
- Ordering.
- `--since` with absolute and relative dates under `BOB_NOW`.
- `--limit`, `-A`, and the truncation envelope.
- `-A` with `-n` is a usage error (2).
- Legacy collapse versus `-s legacy`.
- `-g` against a temp Git vault whose commits are dated with `GIT_AUTHOR_DATE` and
  `GIT_COMMITTER_DATE`, including a legacy note that must keep its zorg date.
- `-g` without Git.
- A colored-render unit test.
- The side-effect check.

## Phase show: bob ref show

**CLI**

```text
bob ref show [OPTIONS] <REF>...

Show reference notes with metadata, annotations, and your own notes

Arguments:
  <REF>...  Vault path, note stem or id, URL, arXiv ID, DOI, or exact title

Options:
  -b, --bob-dir <PATH>    Bob vault root; defaults to BOB_DIR or ~/bob
  -c, --comments-only     Only annotations you commented on and standalone notes, with their quotes
  -f, --format <FORMAT>   Output format [default: human] [possible values: human, json, markdown]
  -N, --no-annotations    Metadata and your own notes only
  -r, --ref-dir <PATH>    Reference note directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -h, --help              Print help
```

`-c` conflicts with `-N`.

**Resolution**

- A REF resolves through the exact match kinds plus a unique `title_exact`.
- If several notes match and exactly one is not superseded, show that one, and print a
  dim `also: <path> (superseded legacy note)` line (`also` in JSON).
- Any other multi-match fails with `ambiguous reference` and lists the candidates. It
  never picks the first stem.
- A REF with no exact match fails as `no reference note matches <REF>`. Up to 3 title
  candidates appear as `hint:` lines.
- All REFs are resolved before any output. One failure exits 1 and prints nothing else.

**JSON row extensions**

| Field                | Contents                                                                                                  |
| -------------------- | --------------------------------------------------------------------------------------------------------- |
| `annotations_status` | `parsed`, `absent` (no region), or `unparsed` (with `raw_region`)                                         |
| `annotations`        | `[{ page_label, kind: highlight\|note\|image, quote, comment, asset, block_id, link: "[[ref/x#^h-…]]" }]` |
| `excluded`           | `{ marker_mirrors, preamble, removed }`                                                                   |
| `own_notes`          | The user's own text (see phase `region`)                                                                  |
| `tasks`              | Parsed `## Tasks` lines                                                                                   |
| `also`               | Superseded companion paths                                                                                |

**Human output**

```text
EA-Graph: Artifact-Anchored Verification Memory for Coding Agents under Upstream Drift
✓ FINISHED · read 2026-10-03 · papers · external · ref/papers/ea_graph.md

  source    https://arxiv.org/pdf/2608.04278 (arXiv 2608.04278)
  pdf       lib/papers/ea_graph.pdf
  audio     lib/papers/ea_graph.mp3
  parent    sase_ref
  snapshot  annotations as of the last scan, 2026-10-03 18:00

ANNOTATIONS · 7 highlights · 1 comment
  Page 2
    “A determinism contract and replay mechanism that makes any run
     byte-reproducible from its log, including a content-addressed cache…”
      ↳ Support sase tool call replay?
  (1 leaked marker block and 2 removed highlights not shown)

NOTES
  …own notes, wrapped…

TASKS
  ☐ Compare this claim with the appendix.   Page 12
```

- Quotes and comments wrap to `terminal_width()` with hanging indents. The `↳` comment
  lines are cyan.
- Metadata rows appear only when they have a value.
- Several REFs are separated by a dim rule.
- Empty sections are omitted. Under `-c` with nothing to show, print
  `No comments or standalone notes.`

**Markdown output.** A clean, quotable digest for an agent's context:

- `## <title>`.
- Bullets for the note, its wikilink, the reading state with evidence and date, the
  source URL and arXiv/DOI, origin and type, and the annotation counts and snapshot
  date. A `Research report: research:…` bullet appears when `research_ref` is set.
- `### Annotations` with a `**Page N**` label per page:
  - each quote as a `>` blockquote;
  - each comment as a following `Comment: …` line;
  - standalone notes as `Note: …`.
- `### Notes`, then `### Tasks`.

**Completion:** `["ref","show"] ref` → `VaultNote`.

**Docs:** a show section in `docs/ref.md`, and the agent-usage guidance mirrored from
the skill.

**Tests** (`tests/cli/ref_library/show.rs`)

- Every resolution path, including superseded selection and the ambiguous-stem failure
  with candidates in both human and JSON output.
- Quote and comment separation.
- Multi-line comments.
- Images.
- Mirror and preamble exclusion with counts.
- Tombstone counting.
- An unparsed region.
- A legacy note: `annotations_status: absent`, and its migrated body as `own_notes`.
- `-c` and `-N`, and their conflict.
- Several REFs.
- Markdown output, including `research_ref`.
- A colored-render unit test.
- The side-effect check.

## Phase doctor: Library health and coverage rows in doctor

- Change the `doctor` `about` to "Check library health and Highlights sync
  prerequisites", and update its `HELP_GROUPS` row.
- Append library rows after the existing rows. They use the existing
  `name: ok|warn (detail)` style, and they are **warnings, never failures**:
  - `library: ok (594 notes · 285 finished · 92 started · 150 queued · 21 dropped · 46 unknown)`
  - `library diagnostics: warn (3 notes: 1 invalid_yaml, 1 multiple_ref_trackers, 1 status_conflict)`
    with up to 3 example paths, or `ok`.
  - `identity: ok (1 legacy note superseded by a newer capture)`, or
    `warn (N identity keys shared by more than one PDF-backed note: …)`.
  - `annotations: warn (113 notes still render a leaked marker mirror; the next bob ref scan removes them)`,
    or `ok`.
  - `coverage: warn (~425 zorg-era reading records outside ref/ are not indexed: work_ref.md 75, nvim_ref.md 69, …)`
    with the top 5 files, or `ok`.
- **Counting zorg records** (in `src/native/ref_library/coverage.rs`):
  - Walk `<bob_dir>/**/*.md`, excluding the ref dir, hidden directories, `_generated/`,
    and `*.assets/`.
  - Count lines that match
    `^\s*[*-] status::\s*(UNREAD|COLLECT_FLEETING_NOTES|REVIEW_FLEETING_NOTES|REVIEW_LIT_NOTES|READ|ABANDONED|BOOK)\b`,
    case-insensitively.
  - Subtract records already mirrored into the ref dir. A record's owner is the nearest
    preceding less-indented list line that ends in a `^z-…` block id. It is mirrored
    when some ref note has the same `source_block` and the same `source_path`.
- Document the rows in the doctor section of `docs/highlights-ref-sync.md`.
- **Tests:** temp-vault CLI tests for each row in its ok and warn forms, plus a
  mirrored/unmirrored zorg fixture. The existing doctor assertions must keep passing.

## Phase sync-fixes: Remove leaked marker mirrors and stamp completion dates

### bob-cli-4r root fix

1. **Sidecar parsing** (`sidecar.rs` `parse_sidecar_markdown`). In a sidecar that has at
   least one page heading, text before the first page heading is document preamble,
   never an annotation. That covers a setext title (`===`), an author line with a `---`
   underline, and blank lines.
2. **Mirror detection** (`sidecar_render.rs` `render_sidecar_highlights` and
   `annotation_tasks.rs`).
   - Replace the one-shot `skipped_marker_note` with content-based detection: skip an
     annotation exactly when `region::is_marker_mirror_text` holds for a standalone
     note's text, or for a linked-page highlight's comment.
   - Real notes never look like a `status`/`parent` marker list.
3. **ID stability.** No genuine annotation's `^h-` ID may change.
   - The preamble had no page label, so page ordinals are unaffected. Prove it with a
     regression test that renders the same sidecar before and after the fix and compares
     the IDs of the genuine blocks.
   - Image assets keyed by order must still resolve.
4. **Silent drop.** When an existing block ID disappears from the new render and the
   existing region shows that block as mirror-shaped or in the preamble
   (`region::parse_managed_region`), drop it without a tombstone. Every other
   disappearance still tombstones under `### Removed highlights`.
5. **Count.** `highlights_count` then excludes the mirror.
6. **Fixtures.**
   - Add a sidecar with the setext preamble shape quoted in bob-cli-4r. Reproduce the
     shape; do not copy vault files into the repo.
   - Add a ref note that already contains the leaked block.
   - Test that a dry run reports the change, that a writing sync removes the block and
     decrements `highlights_count`, that no tombstone appears, and that a second sync is
     a no-op.
   - The existing fixtures without a preamble keep rendering byte-identically.

### Completion-date stamping (research A12)

- When sync itself changes a `^ref` mark to `x` and the line has no `[completion::`
  field, insert ` [completion:: YYYY-MM-DD]` immediately before the ` ^ref` token. Do
  the same with `[cancelled::` for `-`.
- The date comes from `env::current_datetime()`.
- Existing fields are never touched. A reopen never removes a date.
- Unit and CLI tests:
  - a marker-driven close stamps exactly once;
  - an existing date is preserved;
  - a checkbox already closed by the user is untouched;
  - dry-run output shows the change.
- Document it in the status section of `docs/highlights-ref-sync.md`.

### Bead hygiene

Close `bob-cli-4r` as resolved by this phase (see `sase bead close -h`). Record on this
phase's bead that the vault cleanup of about 113 notes happens on the next real
`bob ref scan` (the Mac cron).

## Phase legacy-capture: Capture URLs that only a legacy note records

**Dedupe rule** (`src/native/highlights_ref/sources.rs`)

- A ref-note hit refuses only when it is **PDF-backed** (`source_pdf.is_some()`).
- `check_dedupe` and `find_refusing_hit` skip ref-note hits that have no `source_pdf`. A
  new helper, `legacy_hits(recorded, key) -> Vec<RecordedSource>`, returns them so
  callers can warn.
- Intake hits are unchanged.

**`clip` and `create`** (every URL route)

- On legacy hits, print to stderr, before any fetch:
  - `warning: already in the library as <path>, a note without a Highlights PDF; capturing a fresh copy`
  - `hint: bob ref find and bob ref list treat the older note as superseded once bob ref scan writes the new one`
- Then proceed.
- A dry run adds `legacy: <path> (superseded by this capture)` to its report.
- With `--listen`, a legacy-only hit is a normal capture plus listen, never attach mode.
- No new marker or frontmatter field is written (design decision 5).

**Docs**

- The dedupe sections of `docs/highlights-clip.md` and `docs/highlights-create.md`, plus
  the clip help after-help if it describes dedupe.

**Tests**

- `tests/cli/highlights/clip.rs`: a legacy `url:`-only note now captures with the
  warning, while a PDF-backed note still refuses. Update the bob-cli-4s regression test
  that asserted a legacy `url:` refuses: a `url:` on a PDF-backed note still refuses.
- `create` URL routes: an arXiv URL whose only match is a legacy `url:` captures.
- `--listen`: a legacy-only hit produces a fresh capture, not an attach.
- Unit tests in `sources.rs`.

**Bead note:** add a note on `bob-cli-39` that the legacy-only subset landed, and that
PDF-backed recapture and versioning remain open.

## Phase skill: The bob_ref agent skill

Open the linked `chezmoi` repo with `/sase_repo` and read its `AGENTS.md`. Then add
`home/sase/skills/bob_ref.md`, modeled on `home/sase/skills/bob_query.md` (frontmatter
`name`, `description`, `skill: true`).

**Description.** "Check Bryan's Bob reference library (what he has read, is reading, or
plans to read) and read his annotations through `bob ref`. Use before recommending
articles, papers, or other reading to Bryan, and whenever you need his notes on a
reference."

**Body rules**

- **Batch the lookup.** Before recommending reading, put every candidate (URL, arXiv ID,
  DOI, or title) into **one** call: `printf '%s\n' … | bob ref find - -f json`.
- **Interpret each result:**
  - `finished`: drop it, or cite it as already read.
  - `dropped`: drop it unless there is a new reason.
  - `queued` / `started`: keep it only if relevant, labeled "already in your library
    (queued since …)". `era: legacy` means the 2025 zorg backlog.
  - `possible`: confirm with `bob ref show <path>` before calling it a duplicate.
  - `in_intake`: captured, not yet scanned.
  - `not_found`: absent from `ref/` only. Never claim Bryan has not read it.
- **For taste:** use `bob ref list -o external -R finished -g -f json`. `agent-report`
  notes (`ref/chat`) are SASE research reports: a topic-interest signal, not reading
  history.
- **For Bryan's own thoughts:** use `bob ref show <REF> -c -f markdown`.
- **Respect the cap.** Honor `truncated`, and never paste `list -A` output into a
  prompt.
- **Read-only.** Never edit `~/bob`. Never run `bob ref clip/create/scan/sync` unless
  Bryan asks. Propose `bob ref create <URL>` lines (add `-L` to also narrate) for him
  instead.
- **End reading-list reports with:** "Library check: N of M candidates already in your
  library (K finished)."
- Include one short worked example.

**Comments.** Update the comments in `home/dot_config/bob/config.yml` that say
`bob highlights create --listen` to `bob ref create --listen`. Leave every command
invocation on `bob highlights` (design decision 10).

**Deployment.** Do not deploy here. `sase skill init` refuses uncommitted or unlanded
sources, so phase `verify` deploys after this commit lands.

## Phase verify: Live verification, install, and skill deployment on athena

Work from an up-to-date master checkout. The real vault (`~/bob`) is **read-only** for
this phase: run no writing `scan`, `sync`, `clip`, or `create` against it.

1. **Build and smoke test.** Run `cargo build --release`, then the `install-smoke`
   lines.
2. **Acceptance exercise** (from the research). Classify each case correctly with
   `target/release/bob ref find -f json`, and explain it in human output:
   - a finished paper (`https://arxiv.org/abs/2608.04278`, stored as `/pdf/`);
   - a URL variant (`https://arxiv.org/html/2608.04278v1`);
   - a legacy item with `review_lit_notes` (pick one with
     `rg -l 'legacy_status: "review_lit_notes"' ~/bob/ref`);
   - a queued item (a `next` chat note);
   - a title near-match;
   - an intake-only capture, using a temporary vault and a marker-stamped PDF made with
     `bob ref create <tmp.md> -b <tmp vault>` plus `find -i`;
   - a source that is not in the library;
   - the _Harness engineering_ pair, which must give `in_library`, `finished`, and the
     legacy note as `also`, superseded.
3. **Counts.**
   - `bob ref list -R all -A -f json | jq .library` totals must equal the number of
     `~/bob/ref/**/*.md` files minus skips.
   - Running `show` on every note must yield no `unparsed_region` diagnostics.
   - `marker_mirror_excluded` must appear on about 113 notes.
   - Record the numbers in a phase note.
4. **Performance.** Time `bob ref list -R all -A -f json` and a 30-query batch `find`.
   Record both. Investigate anything over 250 ms that is not caused by `-g`.
5. **Aliases.** `bob highlights doctor --no-hooks` and `bob ref doctor --no-hooks` must
   be byte-identical on the real vault, and the doctor library rows must be plausible.
   Compare the coverage row's count with the research's ~425 and explain any difference.
6. **Mirror cleanup preview.** Run `bob ref scan --dry-run --no-hooks -v`. Confirm that
   the leaked mirror blocks would be removed without tombstones, and that nothing else
   churns. Record the count of notes that would change.
7. **Install.** Run `cargo install --path . --locked` so that agents on athena can run
   `bob ref`. The note for Bryan says to run `just install-all` on the Mac and apollo.
8. **Deploy the skill.** From an up-to-date chezmoi checkout (`/sase_repo`), run
   `sase skill init -y`. It commits, pushes, and applies the generated provider files.
   Then run `chezmoi update -a --force`, as chezmoi's `AGENTS.md` requires. Verify that
   `~/.claude/skills/bob_ref/SKILL.md` exists, and that its description is the one
   written in phase `skill`.
9. **Bead hygiene.**
   - Add a note on `bob-cli-4b`: the singular `ref` was chosen for this group, per the
     research.
   - Record `PROPOSED FOLLOW-UP:` notes on this phase's bead for:
     - the zorg reading-record migration, as a separate Bryan-reviewed epic (research
       Phase 5);
     - a `memory` task proposing a `decisions` record, "Reference reading state is
       derived and the library verbs are read-only";
     - a library-wide annotation `search`;
     - bare `bob ref` defaulting to `list`;
     - a durable "ever finished" history.

## Out of scope

- Write verbs (`mark`, `track`, `adopt`, `set-status`).
- Library-wide annotation search.
- A `stats` verb.
- Caches, databases, embeddings, or an MCP server.
- Backfilling `source_url` from `url`.
- Renaming env vars, config keys, or fields.
- Migrating zorg records.
- PDF-backed recapture and versioning (`bob-cli-39`).
- Changing external callers to `bob ref`.
