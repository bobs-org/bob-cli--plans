---
tier: tale
title: Title-free gcal badge links
goal:
  Calendar, join, and maps badges written by bob gcal sync open correctly from every
  Obsidian link path, including Jump to link (Ctrl+L), and existing rows migrate in
  place.
size: medium
decisions:
  maps_title:
    ask:
      Should the 📍 maps badge also lose its location title, with the agenda reading the
      location from the Maps URL instead?
    choices:
      drop: All three badges go title-free; Ctrl+L opens a clean Maps search too
      keep:
        Only 📅/📹 go title-free; 📍 keeps its tooltip but Ctrl+L sends a messy Maps
        query
    default: drop
    why:
      One title-free rule for every badge; the place stays recoverable from the Maps
      query
    answer: drop
proposed_by: bbugyi200.athena.175
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 12:50:14
status: wip
---

# Title-free gcal badge links (fix the 400 from `📅` links)

## Problem and verified root cause

Event Task rows written by `bob gcal sync` carry Markdown link titles:

```markdown
[📅](https://www.google.com/calendar/event?eid=EID "Open in Google Calendar")
[📹](URL "Join call") [📍](MAPS_URL "Place")
```

The stored URL is clean, and plain Obsidian handles the title correctly. Verified
against real Obsidian 1.14.5 and the Mac's 1.15.0 with the vault's full plugin set:
mouse clicks in Reading view and Live Preview, Mod-clicks in Source mode, and
`editor:follow-link` all open the bare URL.

The breakage comes from the **Jump to link** community plugin (`mrj-jump-to-link`
0.6.2). Bryan binds `mrj-jump-to-link:activate-jump-to-link` to Ctrl+L. In editing mode
it finds external links with `/\[[^[\]]+?\]\((.+?:\/\/.+?)\)/g` and calls
`window.open(captured)`. The capture runs up to the first `)`, so it includes
` "Open in Google Calendar"`. The browser encodes that as
`eid=EID%20%22Open%20in%20Google%20Calendar%22`. Google validates `eid` syntax before
checking sign-in and returns `400. That's an error.` for any trailing byte (a lone `%20`
is enough), while a well-formed `eid` passes to sign-in.

Evidence:

- The Mac's Chrome history for the screenshot shows exactly this URL at 12:24:51.
- A scratch-vault probe reproduced it with Ctrl+L. Titled `📅`, `📍`, and `💡` links
  opened `URL "title"`, while an untitled `📅` opened cleanly.

`📹` (`meet.google.com/...%20%22Join%20call%22`) breaks the same way. Keep `💡` badges
also break, but they are out of scope here (see "Follow-up").

**Fix:** Bob-written gcal badges carry no Markdown link title: `[📅](URL)`, `[📹](URL)`,
`[📍](MAPS_URL)`. Every parser opens these correctly, including naive regexes like Jump
to link's. Existing untouched rows migrate in place through the existing three-way span
update. The journal still holds the hash of the titled badge span Bob last wrote, the
current text matches it, and so the fresh title-free render replaces it with no
`kept_edit` warning.

> [!decision] maps_title = keep Leave the `📍` format string and the agenda `location`
> reader unchanged; drop only the `📅` and `📹` titles. Skip step 2, and limit the
> regression assertion in step 4 to the `📅`/`📹` badges.

## Changes

### 1. Render title-free badges (`src/native/gcal/render.rs`)

- `calendar_badge`: emit ` [📅](<escaped url>)`.
- `badges`: emit ` [📹](<escaped url>)` and ` [📍](<maps_url>)`.
- `escape_link_title` becomes unused. Delete it if nothing else calls it, so
  `-D dead_code` passes.
- Above `badges`, add a short comment. Badge links are deliberately title-free, because
  Obsidian's Jump to link plugin (Ctrl+L) and other naive Markdown parsers fold a
  `"title"` into the opened URL. Google then 400s on the corrupted `eid`.
- `badge_text` is shared by the row renderer, the context-row renderer
  (`render_context_row` via `calendar_badge`), and journal span hashes. Change only the
  format, not those call sites.

### 2. Keep the agenda `location` field (`src/native/capture_pomodoros_agenda.rs`)

`location: markdown_link_title(raw_line, "📍")` reads the place name from the `📍`
title. Keep that as the first source, because legacy rows and hand-written rows still
carry it. Add a fallback: when the `📍` link has no title, decode the `query=` value of
its target URL (`markdown_link_target(raw_line, "📍")`).

- Decode `%XX` sequences as UTF-8 bytes, and treat `+` as a space for hand-typed URLs.
  Stop at the next `&`, and return `None` for an empty result.
- `render::maps_url` writes `percent_encode_query(location.trim())`, so the round-trip
  must be exact. `Main St Dental, Suite 2 & 3` →
  `Main%20St%20Dental%2C%20Suite%202%20%26%203` → the original text.
- Put the decoder next to `markdown_link_target` as a small private helper. Don't reach
  into `articles/links.rs`, whose `percent_decode` is private and article-specific.
- The JSON contract (`location: string|null`) does not change.

### 3. Docs

- `docs/gcal.md`: update the two example rows (`[📅](… "Open in Google Calendar")`
  around lines 560 and 607) to `[📅](…)`.
- `docs/gcal.md`: add one sentence where the badge run or reconciling section first
  describes badges. Badges are plain title-free links, so link-hint plugins such as Jump
  to link open them correctly, and rows written by older Bob versions migrate on that
  day's next sync while untouched. Also extend the "Reconciling Google-side changes"
  bullet, or add a short note, saying a badge-format change rides the same in-place span
  update.
- `docs/reference/vault-format.md` (around line 387): state that the `📅`/`📹`/`📍`
  badges are plain `[emoji](url)` links with no link title.
- No CLI flags or help text change. Run `just docs-check`. Per the release-train
  decision, do **not** edit `CHANGELOG.md` or the Cargo version. Use a `fix(gcal):`
  commit subject (not breaking: the row grammar stays parseable both ways).

### 4. Tests

Update every assertion on **rendered** output. Leave fixtures that model legacy or
hand-written vault input titled, because they now double as backward-compatibility
coverage. Find them with `rg -n 'Open in Google Calendar|Join call' src tests`.

Expected rendered-output updates (non-exhaustive):

- `src/native/gcal/render.rs` unit tests: rows at about lines 358, 401, 483, and 504.
  Also the `📍` expectation in `badge_urls_and_maps_links_are_encoded`.
- `src/native/gcal/plan/tests.rs`: any assertion on a freshly _written_ row. Rows seeded
  as pre-existing note content can stay titled.
- `tests/gcal_sync.rs`, `tests/gcal_lifecycle.rs`: expected written rows. In
  `preexisting_off_clock_title_migrates_both_spans_in_place`, keep the titled
  `old_badges` input and update the expected migrated row to the title-free form.

New tests:

1. **Render regression** (`render.rs`): for a task row and a context row with all three
   badges, assert the badge span is exactly `[📅](URL) [📹](URL) [📍](MAPS)`. Also
   assert the span contains no `"` (guards against titles coming back).
2. **In-place migration** (`tests/gcal_lifecycle.rs`): a new test,
   `preexisting_titled_badges_migrate_in_place`, modeled on
   `preexisting_off_clock_title_migrates_both_spans_in_place`.
   - Seed a `[*]` Event Task with legacy titled `📅`/`📹`/`📍` badges, and seed a
     journal whose `badges` hash is `span_hash` of that legacy span.
   - After sync: the row carries the title-free badges, and the checkbox, `[root::]`,
     and `^block-id` are unchanged.
   - The output has no `kept your edit` warning, and the journal now records the
     title-free badge hash.
   - A rerun is a no-op.
   - Add a second row whose badge span was hand-edited, so it no longer matches the
     journal. It stays untouched and reports `kept_edit` once (existing behavior, pinned
     here for the format change).
   - Include a completed `[x]` legacy row as well. The checkbox is Bryan-owned and does
     not block the badge update.
3. **Agenda location** (`capture_pomodoros_agenda.rs` unit tests, or
   `tests/cli/capture/pomodoros_agenda.rs`): `location` comes from a legacy `📍` title,
   from an untitled `📍` URL (including `%2C`, `%26`, a UTF-8 place such as `Café Zoë`,
   and `+`), and is `null` when there is no `📍`. The existing `agenda-events.json`
   fixture stays green.
4. **Adoption and matching still work for both forms:** `find_row_line`, and in
   `list.rs` `row_present` and `adopted_block_id`, use `line.contains(url)`. Add or
   extend one `gcal list`/sync test showing a title-free row is recognized
   (`task · MEET` / no duplicate insert).

Run `just check` (or the repo's fmt + clippy `-D warnings` + `cargo test` gate) and
`just docs-check`.

## Rollout and the existing vault rows (no vault writes from athena)

`gcal.writer` is the Mac (`kellys-macbook-pro`). Other hosts may only run `sync -d`, so
the implementing agent must **not** write daily notes with `bob gcal sync` from athena.
It may run `bob gcal sync -D 2026-11-02 -d` against the real vault, read-only, to
sanity-check the diff. With the new binary, that diff should show only the badge-span
rewrites and the stamp time.

After the next release reaches the Mac:

- Each day's next sync on the Mac migrates its untouched rows automatically. That covers
  today and any future day: on demand, or Bob Ledger Tools' sync when a daily note is
  created.
- Six legacy rows exist today, across `2026/20261030.md`, `20261031.md`, `20261101.md`,
  and `20261102.md`. Bryan can migrate them on the Mac with
  `for d in 2026-10-30 2026-10-31 2026-11-01 2026-11-02; do bob gcal sync -D "$d"; done`.
  Each run is idempotent and preview-able with `-d`. Mention this in the final summary,
  and don't run it.

## Follow-up (file, do not implement)

Keep `💡` badges (`[💡](https://keep.google.com/u/0/#NOTE/ID "Open in Google Keep")`)
break the same way under Ctrl+L. Verified in the probe: the opened URL gains
`%20%22Open%20in%20Google%20Keep%22`.

They need a separate change because the title is gkeep's provenance marker:

- `gkeep/render.rs::parse_source_url` accepts only the exact titled shape, so ordinary
  links never create import history.
- `migrate_tasks.rs` writes and parses the titled form.
- `bob gkeep migrate-tasks --help` documents it.

Use `/sase_new_task` to file one task bead (type `bug`, size `large`) titled along the
lines of "Keep 💡 source badges break under Jump to link (Ctrl+L): make them title-free
without losing provenance". Include this root cause and the regex. `bob-cli-e1` (link
titles leaking into minted block IDs) is related but distinct, so note it as related,
not a duplicate.

Optionally note in the summary that the upstream plugin bug could be reported to
`mrjackphil/obsidian-jump-to-link`. That is outward-facing, so do not file it without
Bryan's go-ahead.

## Non-goals

- Patching or forking the third-party Jump to link plugin.
- Changing the URL itself (`www.google.com/calendar/event?eid=…` is Google's `htmlLink`;
  adding `authuser` is unrelated to this 400).
- Changing the description-bullet overflow link `[Google Calendar](htmlLink)` or the
  sync stamp link. Both are already title-free.
- bob-plugins and Bob Mac Capture. Their badge parsers already accept untitled links:
  `gcalEventsScanMarkdown` keys on the emoji, and `TaskDisplayText` and
  `CaptureAgendaInlineText` treat the title as optional.
