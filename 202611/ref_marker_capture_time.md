---
tier: tale
title: Minute-precision capture time in every ref create marker
goal:
  Every PDF marker that bob ref create stamps records when the reference was captured,
  to the minute.
size: medium
decisions:
  stamp_format:
    ask: Which minute-precision shape should the marker's `captured` stamp use?
    choices:
      offset:
        "2026-11-04T06:44-0500: `created` minus seconds; an exact instant that can fill
        added_at"
      local: "2026-11-04T06:44: Obsidian's date-time shape, but no time zone"
      spaced:
        "2026-11-04 06:44: easiest to read in the PDF sidebar; no zone, not ISO 8601"
    default: offset
    why:
      Matches the existing `created` convention and stays unambiguous across hosts and
      DST.
    answer: offset
proposed_by: bbugyi200.athena.181
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-04 12:30:17
status: wip
---

# Stamp a minute-precision capture time in every `bob ref create` marker

## Goal

From now on, every page-1 Highlights marker composed by `bob ref create` should include
`captured`: the local capture time, accurate to the minute. For example:

```text
- status: ready
- parent: sase
- title: Silent Agent Failures on Athena: Live Code Swaps Under Running Agents
- id: athena_silent_agent_failures_live_update_skew
- captured: 2026-11-03T17:36-0500
```

Today, only some routes stamp `captured`: the URL routes (PDF URL, arXiv, web article,
collection) and `-f overwrite`. They stamp it as a date only (`YYYY-MM-DD`, from
`note::current_local_date()`). The Markdown route, the vault-note snapshot route
(including `-M`), and the local-PDF route never stamp it. As a result, a chat report
such as `lib/chat/athena_silent_agent_failures_live_update_skew.pdf` shows only
status/parent/title/id in its marker. Its ref note does have
`created: 2026-11-03T17:36:49-0500`, but that value lives only in the note.

## Why widen `captured` instead of adding `created` to the marker

- `created` is note-local provenance. It is written once, when the ref note is first
  written. `parse_marker_with_normalization` (`src/native/highlights_ref/marker.rs`)
  rejects a marker `created` key on purpose (see `docs/highlights-ref-sync.md`). Keep
  that rule unchanged.
- `captured` already means "when bob captured this reference":
  - it is a standard synced user field (`COMMON_USER_FIELDS` in
    `src/native/highlights_ref/mod.rs`) and needs no `highlights_marker_fields` opt-in;
  - create already prints it;
  - it comes first in the `bob ref list` `added` precedence;
  - the already-captured refusal shows it.

  Making it minute-precise and stamping it on every route gives one capture-time field.
  Adding a second one would duplicate it.

- The two times can differ. A PDF queued with `-k`, or one that could not be ingested
  inline, gets its note (and its `created`) later from `bob ref scan`. The marker still
  records the real capture moment.

## Design

### 1. One clock read per capture

In `src/native/highlights_ref/note.rs`, add a helper next to `current_local_date()`. For
example, a `CaptureClock { stamp, date }` returned by `capture_clock()`:

- Build both fields from one `bob_env::current_datetime()` read, so `BOB_NOW` still pins
  the value.
- `stamp` uses the chosen format (`stamp_format`). `date` is `YYYY-MM-DD` from the same
  read, so the two never disagree around midnight.

Resolve the local offset the same way `capture_project_note::format_created_timestamp`
does. Generalize that function to take a format string, or add a minute-precision
sibling. Keep its ambiguous-time and DST-gap handling so `created` and `captured` always
agree on offsets.

> [!decision] stamp_format = offset The format is `%Y-%m-%dT%H:%M%z`
> (`2026-11-04T06:44-0500`). In `ref_library`, when `captured` supplies `added` and the
> `created` instant does not match that day (or is missing), parse the stamp
> (`NaiveDateTime::parse_from_str` / `DateTime::parse_from_str` with `%Y-%m-%dT%H:%M%z`)
> and use it for `added_at`.

> [!decision] stamp_format = local The format is `%Y-%m-%dT%H:%M`. No offset is
> computed, and `added_at` never comes from `captured`.

> [!decision] stamp_format = spaced The format is `%Y-%m-%d %H:%M`. No offset is
> computed, and `added_at` never comes from `captured`.

### 2. Every marker producer stamps `captured` with the minute stamp

For each site below, the marker gets `stamp`. Anything that needs a date keeps using
`date` from the same clock read.

- **Markdown route.** In `src/native/highlights_ref/create.rs`, thread the stamp through
  `plan_markdown`, `compose_markdown_marker` (add a `captured` parameter), and
  `stamp_marker_for_outcome`, and set `CreatePlan.captured`. The recompose after the
  return-link render must reuse the plan's stamp, not read the clock again. The planned
  marker and the stamped marker must differ only by `return_links`.
- **Vault-note snapshot and `-M`.** `create_vault_note_route` builds the marker with
  `compose_markdown_marker`, so it gets the same parameter; set `CreatePlan.captured`
  there too.
- **Local PDF.** `plan_local_pdf` in `src/native/highlights_ref/pdf_target.rs` sets
  `captured: Some(stamp)`. `compose_pdf_marker` already emits it.
- **PDF URL and arXiv.**
  - Routes: `create_pdf_url_route`, `create_arxiv_route`, `overwrite_pdf_url`, and
    `overwrite_arxiv` in `create.rs`.
  - Planners: `plan_pdf_url` and the arXiv planner in `pdf_target.rs`.
  - Pass the stamp for the marker. The `<host>_<YYYYMMDD>` fallback stem keeps using
    `date`.
- **Web article, collection, and background ingest.** This covers the three marker sites
  in `clip.rs`, `collection_marker` in `collection.rs`, and `ingest_url` in `ingest.rs`
  (used by the capture worker and `bob gkeep pull`). The marker gets `stamp`. The
  clip-adapter `CaptureRequest` and compose `captured` values stay `date`, so the
  adapter protocol and its printed header do not change. The fallback stems also stay on
  `date`.
- **Overwrite.** These use `stamp`:
  - `stamp_overwrite_source`;
  - the Markdown overwrite plan (`captured: Some(super::current_local_date())` in
    `overwrite_markdown`);
  - the vault-note overwrite plan (same expression, in `overwrite_vault_note`).

  The note-local `recaptured: YYYY-MM-DD` stays date-only. The staged-marker and
  index-identity checks in `overwrite/execute.rs` compare exact strings that all come
  from the one composed marker, so they keep working.

Before finishing, grep `src/native/highlights_ref` for `current_local_date()`,
`"captured"`, and `FIELD_CAPTURED`. Confirm that every value written into a marker is
the minute stamp, and every remaining date-only use is intentional (adapter protocol,
stems, `recaptured`).

### 3. Create output shows the stamp on every route

Real and dry-run human output print `captured: <stamp>` after author/published and
before status, as the URL routes already do. Add it to:

- the snapshot success output (`created note snapshot`);
- `print_snapshot_plan`;
- the Markdown success output;
- the local-PDF success output;
- `print_plan`.

`print_pdf_dry_run` already prints it when present. The dry-run `marker:` preview shows
the field automatically.

### 4. Readers take the date part where they need a date

- **`ref_library` `added_date`** (`src/native/ref_library/mod.rs`): return
  `date_part(&captured)` as `added`. Today it returns the raw value, so a minute stamp
  would break `--since`, row ordering, and JSON `added`. Keep the row's `captured` field
  raw: `overwrite/execute.rs` compares `row.captured` with the staged marker's exact
  value.
- **`added_at`**: keep using the `created` instant when its date matches `added`. The
  fallback is described in the `stamp_format` decision callouts above.
- **`member_label`** in `birth.rs`: the `earlier:` labels stay `vN · YYYY-MM-DD`, using
  the date part.
- **`duplicate.rs`**: the refusal line keeps `captured YYYY-MM-DD` (date part).
  `compare_captured` compares the date part first, then the full value, so mixed old and
  new values on one day still order newest first.
- **Existing notes and PDFs**: never rewrite their date-only `captured` values. Readers
  accept both shapes.

### 5. Unchanged

These stay as they are:

- marker parse and sync rules, and the marker hash and base;
- the `created` rejection, and note-local `created` and `recaptured`;
- `--listen` attach, which composes no new marker;
- `bob ref sync`, `scan`, and `marker` behavior for existing PDFs.

## Non-goals

- Backfilling existing library PDFs or notes. That would rewrite annotated library PDFs
  and needs its own plan.
- Changing the clip-adapter protocol, or the format of `created` or `recaptured`.

## Tests

All tests use `BOB_NOW` and `TZ=America/New_York`, as the existing `created` tests in
`tests/cli/highlights/sync.rs` do.

- **Unit tests:**
  - the clock helper formats `stamp` and `date` from one read, including a DST-ambiguous
    time and a DST-gap time;
  - `compose_markdown_marker` and `compose_pdf_marker` place `captured` in
    `MARKER_EXTRA_ORDER` position;
  - recomposing for `return_links` keeps the same stamp.
- **CLI tests** (`tests/cli/highlights/create.rs` and siblings): for each route, the
  stamped PDF marker (read with `pdf_marker_contents`) contains `- captured: <stamp>`.
  The routes are:
  - Markdown;
  - vault-note snapshot;
  - vault-note `-M`;
  - local PDF;
  - PDF URL;
  - arXiv;
  - web article (fake clip adapter);
  - collection;
  - `-f overwrite`.

  For the Markdown route, also check that the inline-filed ref note's frontmatter
  carries the synced value, quoted, for example `captured: "2026-10-16T12:00-0400"`. For
  the web route, check that the fake adapter's recorded request still sends a date-only
  `captured`.

- **Output checks:** dry-run and real create print the `captured:` line on the Markdown,
  snapshot, and local-PDF routes.
- **`ref_library`:** a fixture row with a minute-stamp `captured` gets:
  - `added` = its date part;
  - `added_source` = `captured`;
  - the raw `captured` value unchanged;
  - `added_at` as the decision specifies.

  The existing date-only fixture (`clipped_blog.md`) must keep its current values.

- **Regressions:** a reread's `earlier:` label stays `vN · YYYY-MM-DD`, and the
  already-captured refusal line stays `captured YYYY-MM-DD`.
- Update any existing create assertions that pinned the old marker or output shapes.
  Leave hand-written fixture markers for sync and scan tests alone.

## Docs

Follow the documentation rule that a behavior change updates its contract/reference
page. Update:

- `docs/highlights-create.md`:
  - the PDF route paragraph that says URL routes stamp `captured` as a local
    `YYYY-MM-DD` and that local PDFs stamp only `id`;
  - the success example's `captured:` line.
- `docs/highlights-clip.md`: "Marker fields and dedupe". `captured` becomes the minute
  stamp and is always present.
- `docs/highlights-ref-sync.md`:
  - the standard synced user fields paragraph that defines `captured` as the web
    snapshot date;
  - the `created` paragraph that contrasts it with `captured`. State that the marker
    carries the capture time and the note keeps its own `created`.
- `docs/ref.md`: the `added` precedence paragraph. `added` takes the date part of
  `captured`; describe any `added_at` fallback.

Run `just docs-check`. Do not edit `CHANGELOG.md` or the `Cargo.toml` version.

## Verification and commit

- Run `just fmt`, `just lint`, the affected tests (`cargo test` for the highlights and
  ref_library CLI and unit suites), and then `just test`.
- Use a Conventional Commit with `!`. The JSON `captured` value in
  `bob ref list/find/show --format json` widens from `YYYY-MM-DD` to the minute stamp,
  and `added_source` becomes `captured` for new Markdown, local-PDF, and snapshot
  references. For example:
  `feat(highlights)!: stamp minute-precision captured time in every ref create marker`,
  with a `BREAKING CHANGE:` footer that names both effects.
