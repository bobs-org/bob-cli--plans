---
tier: epic
title: Meeting notes become event-linked references
goal: 'One deliberate gesture, from the CLI, Obsidian, or Bob Mac Capture, files AI
  meeting notes (Wispr Flow first, any producer) as a `meetings` reference. The reference
  is filed under the meeting''s area or project, gets a searchable recap and a polished
  dossier PDF, and has a Next reading task related `notes from` its Event Task. The
  `~/tmp/bryan_derrick_meet_notes.txt` notes are filed onto `20261123#^bryan-bugyi-derrick`
  this way as the first use case.

  '
decisions:
  scaffolding:
    ask: After filing the first notes, how should first-use retire the hand-built
      review chain?
    choices:
      rehang: The review chore is superseded; the email it gates now waits on the
        new reading task
      release: The review chore is superseded; the email it gates becomes Ready at
        once
      leave: Only the notes are filed; the receipt prints the commands to retire the
        chain
    default: rehang
    why: The reading task is the review now; rehanging keeps your read-before-email
      intent
    answer: rehang
  recap_scope:
    ask: How much of the normalized notes should the reference note's recap hold?
    choices:
      full: TL;DR, next steps, and every section; all of it is searchable in Obsidian
      brief: TL;DR and next steps only; section detail lives only in the PDF
    default: full
    why: Obsidian search and hover cannot see inside PDFs, and summaries are only
      ~3 KB
    answer: full
  memory_decision_record:
    ask: Add a decision record that meeting notes are references linked `notes from`
      their Event Task?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_term:
    ask: Add a glossary term for the meetings reference (meeting notes ref, its recap,
      and `bob ref meet`)?
    memory:
    - glossary
    default: false
    answer: false
phases:
- id: notes-normalizer
  title: Structural notes normalizer, dossier source, and recap renderer
  depends_on: []
  size: medium
  description: 'notes-normalizer: pure Rust module that reads notes text, detects
    its shape, builds a MeetingNotes model without changing a word, and renders the
    round-trippable dossier Markdown plus the inert, digest-stamped recap region,
    with synthetic golden fixtures.'
- id: meeting-resolver
  title: Event Task resolution, matching, recent-meetings listing, and agenda notes
    state
  depends_on: []
  size: medium
  description: 'meeting-resolver: read-only meetings module that resolves EVENT tokens,
    scores auto-matches with evidence, lists recent Event Tasks with their filed-notes
    state, looks up Google identity, resolves the parent or inbox, and adds the additive
    notes state to the capture agenda''s event rows.'
- id: dossier-print
  title: Meeting dossier print profile
  depends_on:
  - notes-normalizer
  size: medium
  description: 'dossier-print: a meeting print profile beside the note-snapshot profile
    (shared fonts, glyph coverage, warnings) that renders the dossier Markdown into
    a one-to-two-page PDF with a title card, TL;DR box, next-steps table, sections,
    provenance, and an optional transcript appendix.'
- id: meeting-birth
  title: Marker-carried meeting identity births the LINKS relation and recap
  depends_on:
  - notes-normalizer
  size: medium
  description: 'meeting-birth: intake and sync births for a meetings PDF whose marker
    carries `event` give the reading task a LINKS child and seed the note''s recap
    from the birth store, byte-identical whether ingested inline or later by `bob
    ref scan`.'
- id: ref-meet-cli
  title: The `bob ref meet` command
  depends_on:
  - meeting-resolver
  - dossier-print
  - meeting-birth
  size: large
  description: 'ref-meet-cli: the CLI leaf wiring resolver, normalizer, print profile,
    and birth into the create/intake pipeline, with list mode, dry run, editor review,
    clipboard/stdin intake, dedupe and overwrite, inbox re-root, the JSON contract,
    the receipt, `meetings:` config, catalog rows, and docs.'
- id: first-use
  title: File the first meeting's notes
  depends_on:
  - ref-meet-cli
  size: small
  description: 'first-use: run the new command on the real notes file against the
    live vault, verify every artifact (including the PDF visually), retire the hand-built
    review chain per the scaffolding decision, and record deferred follow-ups.'
- id: obsidian-surfaces
  title: Obsidian surfaces for meeting notes
  depends_on:
  - ref-meet-cli
  size: medium
  description: 'obsidian-surfaces: bob-plugins adds a Command Mode FOR THIS NOTE row
    for the Event Task under the cursor, a `ref-meet` outcome adapter, and a Reading
    Board meetings kind, then deploys with `bob plugins sync`.'
- id: mac-surfaces
  title: Bob Mac Capture meeting-notes mode and Meetings kind
  depends_on:
  - ref-meet-cli
  size: large
  description: 'mac-surfaces: bob-mac-capture adds a Meeting Notes mode (hotkey, menu
    item, and a Notes action on ended agenda rows) that previews and files clipboard
    notes through `bob ref meet` JSON, plus a first-class Meetings kind in Bob Refs,
    gated green on macOS CI.'
proposed_by: bbugyi200.apollo.research.65.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-24 10:38:42
status: wip
bead_id: bob-cli-pw
---

- **PROMPT:** [prompts/202611/meeting_notes_event_refs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/meeting_notes_event_refs.md)
- **BEAD:** [bob-cli-pw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/README.md)

# Plan: Meeting notes become event-linked references

## Why and what

Bryan records meetings with Wispr Flow Notetaker and wants the notes filed against the
meeting as an excellent reference. The reference should sit under the right area or
project and be properly linked to the Event Task. Today this takes hand work. For the
first meeting, Bryan hand-built a review chore (`job#^whisperflow-meeting`). It is
blocked on the bob build task and in turn gates a follow-up task.

The design follows the consolidated research report `meeting_notes_event_refs.md`
(research sidecar; read it with `sase artifact read` if you need the rationale). Bryan
accepted all of its requirements:

1. **Notes, with transcripts opt-in.** The vault note never holds a transcript.
2. **The parent is the Event Task's `[root::]`.** `-P` overrides it. An inbox root asks
   once, and that answer also re-roots the event.
3. **The link is one inert labelled relation authored on the reading task.** The Event
   Task row is never authored. Its LINKED FROM chip comes from reconcile.
4. **The reading task is the review.** It is born Next by default, and `-s read` files
   the notes "on the record" with no review.
5. **Filing is producer-agnostic.** Paste, file, or stdin is the contract. The
   undocumented Wispr store adapter comes later.
6. **Structure is normalized, words are never rewritten.** AI enrichment is a later,
   separate layer.

This epic builds the research's Phases 1–2 (bob-cli core plus the Obsidian and Mac
surfaces) and files the first meeting with the real command.

**Privacy rule for every phase (hard requirement).** bob-cli, bob-plugins,
bob-mac-capture, and this plan's sidecar are **public** GitHub repositories. Never copy
real meeting content, real attendee or company names, vault task text, or the notes file
into code, fixtures, docs, tests, or commit messages. Fixtures and examples use invented
people, companies, and products. They should reproduce the _shape_ of a Wispr summary,
not its words. The real notes touch only the private vault, and only in `first-use`.

## The experience (design of record)

### The gestures

Each surface ends in the same `bob ref meet` call, so behavior is identical everywhere.

| Where                  | Gesture                                                                                                                                                                       | Keys after copying the notes in Wispr |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Bob Mac Capture        | Capture hotkey, then the Meeting Notes key. The sheet auto-matches the clipboard notes to the meeting, previews the outline and destination, and Return files.                | 3                                     |
| Bob Mac Capture agenda | An ended meeting row without notes shows a **Notes** action that opens the same sheet with that meeting preselected.                                                          | 2–3                                   |
| Obsidian               | With the cursor on the Event Task row, `:` opens Command Mode. The **FOR THIS NOTE** row _File meeting notes from clipboard_ previews on Shift+Enter and files on Ctrl+Enter. | 3                                     |
| CLI / agents           | `bob ref meet -c` (clipboard, auto-match), `bob ref meet 20261123#^bryan-bugyi-derrick -i notes.txt`, or `… -i -` from stdin.                                                 | —                                     |
| Discovery              | Bare `bob ref meet` lists recent meetings and which ones still need notes.                                                                                                    | —                                     |

### What one filing writes (one ingest commit)

Below is the first use case. `job` is the event's root, the dossier is born Next, and
the stem is the meeting day plus the event title's content words.

Reading task, in the parent note's `## Tasks` (`job.md`):

```markdown
- [*] #task #ref
  [[ref/meetings/261123_bryan_bugyi_derrick_blakely|Bryan Bugyi and Derrick Blakely — Meeting Notes (Mon 2026-11-23, 15:00–15:30)]]
  [created::YYYY-MM-DD] ^ref-261123-bryan-bugyi-derrick-blakely
  - 🪢 **LINKS:** notes from [[20261123#^bryan-bugyi-derrick]]
```

Reference note `ref/meetings/261123_bryan_bugyi_derrick_blakely.md`. Its frontmatter
gains synced user properties next to the usual pipeline fields:

```yaml
title: "Bryan Bugyi and Derrick Blakely — Meeting Notes (Mon 2026-11-23, 15:00–15:30)"
source_url: "<the row's 📅 Google Calendar URL, when present>"
event: "[[2026/20261123#^bryan-bugyi-derrick]]"
event_id: "<decoded Google event id, when known>"
meeting_start: 2026-11-23T15:00
meeting_end: 2026-11-23T15:30
notes_producer: Wispr Flow
ref_type: meetings
```

Body: the managed embed, then a **bob-managed recap region**, then the usual Highlights
region.

```markdown
# Bryan Bugyi and Derrick Blakely — Meeting Notes (Mon 2026-11-23, 15:00–15:30)

![[job#^ref-261123-bryan-bugyi-derrick-blakely]]

<!-- bob:meeting-recap:begin v1 sha256:<16 hex> -->

## Notes

_Mon 2026-11-23 · 15:00–15:30 ·
[[2026/20261123#^bryan-bugyi-derrick|Bryan Bugyi and Derrick Blakely]] · Wispr Flow_

> [!summary] TL;DR <the lede, verbatim>

### Next steps

- **<Owner>** — <action, verbatim>

### <Section heading, verbatim>

- <paragraph, verbatim>
<!-- bob:meeting-recap:end -->

## Highlights

<!-- highlights:begin -->
<!-- highlights:end -->
```

> [!decision] recap_scope = full The recap holds the TL;DR, next steps, and every
> section.

> [!decision] recap_scope = brief The recap holds only the meta line, the TL;DR, and
> next steps. Sections appear only in the PDF, and the receipt says so.

Dossier PDF `lib/meetings/<stem>.pdf`, with its exact render input in the write-once
source store. Event Task row after the next reconcile, which is after rollover when the
row is in the previous daily note: `🪢 **LINKED FROM:** [[job#^ref-…]] (notes from)`.
Nothing writes the event row except an inbox re-root.

### `bob ref meet` contract

```text
bob ref meet [OPTIONS] [EVENT]...

File meeting notes as a reference linked to their Event Task. Without notes (-c, -i, or
-e), list recent meetings and whether each has notes.

Arguments:
  [EVENT]...   The meeting: an Event Task ref (20261123#^id, 2026/20261123#^id,
               [[20261123#^id|…]], 20261123:id, ~20261123:id), a Google Calendar URL
               with eid=, or title words (every word must appear in the title)

Options (sorted by long name; every public long option has a short alias):
  -b, --bob-dir <DIR>       Bob vault root; defaults to BOB_DIR or ~/bob
  -c, --clipboard           Read the notes from the clipboard
  -D, --day <DAY>           Match meetings on one local day (same parser as gcal note -D)
  -d, --dry-run             Show the match, destination, and outline; write nothing
  -e, --edit                Review the normalized notes in $VISUAL, $EDITOR, or vi before
                            filing (with no -c/-i, start from an empty buffer)
  -f, --format <FORMAT>     human (default) or json
  -i, --input <FILE>        Read the notes from FILE (any UTF-8 text); - reads stdin
  -l, --list                List recent meetings and their notes instead of filing
  -O, --overwrite           Replace the notes already filed for this meeting
  -P, --parent <NOTE>       File under NOTE instead of the event's root; required when
                            the root is an inbox, where it also re-roots the event
  -p, --producer <NAME>     Who wrote the notes (recap, PDF, notes_producer)
                            [default: meetings.producer]
  -s, --status <STATUS>     Reading task: next, ready, wip, or read [default: meetings.status or next]
  -T, --title <TITLE>       Override the derived title
  -t, --transcript          Keep a Transcript section as a PDF appendix (never in the note)
  -w, --window <DAYS>       Look back DAYS days for meetings [default: meetings.window_days or 7]
  (hidden) --expect-sha256 <HEX>   Client guard: refuse if the notes' SHA-256 differs
```

- **Modes.** With a notes source (`-c`, `-i`, or `-e`), the command files. Otherwise it
  is a read-only list (`-l` is the explicit spelling clients use). `-l` conflicts with
  every filing option, and `-c` conflicts with `-i`.
- **Exit codes.** `0` on success. `1` on failure. `2` when bob needs your choice: an
  ambiguous or missing meeting off a TTY, an inbox or unusable root without `-P` off a
  TTY, or notes already filed without `-O`.
- **TTY behavior.** On a TTY, an ambiguous match opens a numbered picker in the
  `bob gcal note` style (`Which? [1] ›`, Enter takes 1, `q` cancels). An inbox root asks
  `File under which note? ›`, resolved by `parent_notes::resolve_parent` with its
  near-miss hints.
- **Dry run.** A dry run takes no locks, renders no PDF, writes nothing, and makes no
  network or model call. Target: under 300 ms on the vault.

**Human output.** Colored and aligned, in the style of `bob gcal note` and the
`bob ref create` receipts. Filing receipt:

```text
✓ Filed meeting notes · Bryan Bugyi and Derrick Blakely
  Mon 2026-11-23 · 15:00–15:30 · job
  note    ref/meetings/261123_bryan_bugyi_derrick_blakely.md
  task    ✱ Next in job · ^ref-261123-bryan-bugyi-derrick-blakely
  link    notes from → 20261123#^bryan-bugyi-derrick
  pdf     lib/meetings/261123_bryan_bugyi_derrick_blakely.pdf · 2 pages
  commit  1a2b3c4  bob ref meet: ingest meetings/261123_bryan_bugyi_derrick_blakely
  ↻ The event shows this link after the next reconcile (after rollover for yesterday's note).
```

- A dry run prints the same header, then the match and its evidence, the destination,
  the title, an outline, and any warnings. The outline is the TL;DR, the next steps, and
  each section heading with its item count.
- List mode prints one line per meeting, newest first. Columns: day, span, title, root,
  and `○ needs notes` or `✓ <note stem>`. The last column is the token to pass as EVENT.
  A header counts the meetings that still need notes.
- Queued ingest is reported honestly: it never says "filed". It names the queued PDF and
  `bob ref scan`.

**JSON (`-f json`).** Reuse the `bob ref` envelope (`ref_library/output.rs`:
`ok, schema_version (REF_SCHEMA_VERSION), command: "ref meet", generated_at`, and the
error envelope with `code, message, hint`). Payloads:

- **List.** `meetings[]`, each with `ref` (the EVENT token), `path`, `block_id`, `date`,
  `start`, `end`, `title`, `root`, `root_is_inbox`, `status`, `event_id?`, and `notes`.
  `notes` is `null` or `{note, task, status}`.
- **Dry run or file.** These fields:
  - `outcome`: `would_file`, `filed`, or `queued`
  - `meeting`: as in list
  - `match`: `{method: explicit|query|auto|picked, evidence[]}`
  - `filing`:
    `{parent, parent_kind, reroot?: {from, to}, title, stem, note, pdf, task: {destination, block_id, status}, relation: {label, target}}`
  - `notes`:
    `{sha256, shape, producer, outline: {tldr, next_steps[{owner, action}], sections[{heading, items}]}, transcript: kept|omitted|none}`
  - `warnings[]`, plus `commit?` and `queued?`
- **Error codes.** `notes_unreadable` (binary, invalid UTF-8, over 1 MiB, or no
  clipboard), `no_notes`, `no_meeting`, `ambiguous_meeting` (with `candidates[]` and
  their `evidence[]`), `needs_parent` (with the root and `candidates[]`),
  `already_filed` (with `existing: {note, task}`), and `notes_changed` (an
  `--expect-sha256` mismatch).

### Matching

Precedence:

1. **Explicit.** An Event Task ref or a Google Calendar URL always wins. A URL decodes
   `eid=` to the event id, then finds the row through the gcal journal or the row's 📅
   URL.
2. **Query words.** Use the `bob gcal note` rule: every word appears in the title, case
   folded, inside `-w` days or `-D`.
3. **Auto**, when no EVENT is given and notes are present.
   - Candidates are Event Tasks in the window that have started and are not cancelled.
   - A candidate is _strong_ when at least 75% of its title's content words, and at
     least two, appear in the notes. Content words are lowercased alphanumeric runs
     minus a small stopword list (`and`, `w`, `with`, `the`, `a`, `of`, `for`, `call`,
     `sync`, …).
   - Auto-select only when exactly one candidate is strong and the runner-up scores
     under 50%.
   - Otherwise, show the ranked candidates with their evidence, for example
     `title words in notes: Derrick, Blakely`. Rank by score, then by recency.

Explicit refs to an unstarted or cancelled event are allowed with a warning. A wrong
link is worse than asking.

### Filing rules

- **Parent.** Use `-P`. Otherwise use the event's `[root::]`, resolved by
  `parent_notes::resolve_parent`. `-P` is required when the root is missing,
  unresolvable, an inbox (`task_complete::successors::is_inbox_note`; this includes
  `meet_inbox`), or not an eligible parent.
- **Re-root.** Only when the root was an inbox or missing and the answer came from `-P`
  or the prompt. Replace `[root::]` in place, exactly as gcal's
  `render::set_row_metadata` does.
  - **Never** use `task_root::set_task_root`. It would move `[buffer::]` out of the
    Google-hashed badge span.
  - The edit lands in the same ingest commit. Report it, and note that a recurring
    series learns the root and that `gcal sync -F` resets it.
  - If ingest queues, skip the re-root with a warning.
- **Type, stem, and title.**
  - The type is always `meetings`, so there is no `-t`.
  - The stem is
    `<YYMMDD of the meeting day>_<up to 6 content words of the event title>`. It is
    validated by the existing stem rules, and existing collisions get the `_2` walk.
  - The title is `<event title> — Meeting Notes (<Ddd YYYY-MM-DD>, <HH:MM>–<HH:MM>)`, or
    `-T`.
  - The reading-task id follows the existing `ref-<slug>` rule.
- **Status.** `-s`, then `meetings.status`, then `next`. `read` is the "on the record"
  archive path.
- **Identity and dedupe.** One notes reference per meeting occurrence, keyed on the
  Google event id when known (journal lookup by `(path, block_id)`, else `eid` decode).
  Otherwise the key is the `event` link.
  - A second filing refuses with `already_filed` and the hints `-O` or open the existing
    note.
  - `source_url` is the row's 📅 URL, so `bob ref find <📅 URL>` finds the notes.
  - Use the existing recapture preflight. Add an identity variant only if the URL
    identity cannot express "same Google event".
- **Overwrite (`-O`).** Reuse create's overwrite (keep the note, id, and reading task).
  - Re-seed the recap **only if its digest still matches**. Otherwise keep the user's
    recap and warn.
  - Never move the recap to `ref_history/`, and leave the LINKS line alone.
- **Commit.** The producer is `bob ref meet`, so the subject is
  `bob ref meet: ingest meetings/<stem>`. The path set includes the daily note only when
  re-rooted. Undo is reverting that one commit.

### Normalization: structure only, never a word

- **Intake.** UTF-8 only. Strip a BOM, convert CRLF to LF, and trim trailing spaces.
  Refuse NUL or binary bytes and anything over 1 MiB. Empty after trimming is
  `no_notes`.
- **Shapes.**
  - **Wispr plain-text summary** (Bryan's file):
    - Runs of two or more blank lines separate sections.
    - The first block is the TL;DR.
    - A section whose first paragraph is one short line with no terminal punctuation has
      that line as its heading.
    - Other paragraphs become items.
    - Under a heading like `Next steps`, `Action items`, or `Follow-ups`, each
      `(Owner) Action` line becomes `{owner, action}`; a line with no owner keeps
      `owner: None`.
  - **Markdown** (an ATX heading or two or more list items):
    - A leading H1 is kept as the notes' own title, shown in the PDF title card when it
      differs from the event title.
    - Prose before the first heading is the TL;DR.
    - `##` and `###` headings are sections; list items and paragraphs are items.
    - A section titled `Transcript` (case-insensitive) is the transcript.
    - `Next steps`-style sections parse as above.
  - **Anything else** files as plain paragraphs with the warning
    `unrecognized notes shape; filed as plain paragraphs`. It is never refused.
- **Transcript.** Without `-t`, it is omitted from the dossier and the store (the
  receipt says so). With `-t`, it becomes a PDF appendix. It never enters the recap.
- **Invariant, tested as a property.** With the transcript excluded, the word sequence
  of the input equals the word sequence of the dossier body and of the un-escaped recap.
  Only markup, whitespace, and the `(Owner)` parentheses may differ. Misheard terms,
  hedged phrases like "sounded like 'X'", and numeric ranges survive exactly as written.

### The recap region

- **Markers.** `<!-- bob:meeting-recap:begin v1 sha256:<digest> -->` …
  `<!-- bob:meeting-recap:end -->`. The digest uses `intentions::digest` over the inner
  text.
  - Store it in the begin marker, **not** in frontmatter, because frontmatter user
    fields sync into the PDF marker.
  - If the user edits the region, bob never rewrites it.
- **Escaping.** Every notes-derived string is inert: no live `#task`, checkbox shapes,
  `[[`, `::` fields, trailing `^id`, `%%`, `$$`, or HTML.
  - In particular, `<!--` must not appear, so notes can never close the region.
  - Reuse `task_text_escape` (`escape_note_text` / `escape_child_text`) and the gmail
    inert rules. Make a helper reachable only if needed.
- **Trusted content.** The meta line, including the event wikilink, is bob-authored and
  trusted.
- **Preservation.** Sync, scan, rename, and `bob ref show` keep the region byte for
  byte. `own_notes` may include it.

### The dossier PDF

Use a meeting print profile next to `note_snapshot/profile.rs`. Prefer generalizing
`render_note_pdf` over a profile struct (header TeX, Lua filter, reader). Reuse its
fonts (Source Serif 4, Inter, JetBrains Mono, Noto Emoji), `GlyphCoverage`, xelatex
plumbing, and typed warnings. Do not use the HTML clip adapter: it is not present on
every host, and the source store contract is pandoc input.

- **Title card.** A small-caps `MEETING NOTES` label, the event title in the display
  face, and a meta strip: day · span (duration) · root · `notes by <producer>`.
  - The accent is the root area's color when bob already resolves one. Otherwise it is
    Bob blue (`BobNextWip`).
- **Order.** A tinted TL;DR box, then **Next steps** as an owner/action table, then the
  sections.
  - Sections are unnumbered, with **no table of contents**.
  - Headings stay in the PDF outline, so Highlights sections still work.
- **Running elements.** The event title · page X of Y. The last page ends with a
  provenance footer:
  `Notes by <producer> · filed <date> by bob ref meet · structure normalized, wording unchanged`.
- **Transcript appendix (`-t`).** It starts on a new page, in smaller type, with speaker
  labels in bold.
- **Quality bar.** A Wispr-sized summary prints on one or two pages, with no overfull or
  missing-glyph warnings, and `pdftotext` returns the words in reading order.

### `meetings:` config

This is a new optional section in `config/meetings.rs`. Register it in `RawConfig`;
older binaries ignore it.

```yaml
meetings:
  producer: Wispr Flow # optional display name; default none → "Meeting notes"
  relation_label:
    notes from # default; must pass normalize_relation_label and carry no
    # cancel/complete/merge/successor effect (config error otherwise)
  status: next # default birth status
  window_days: 7 # default lookback
```

### Reliability rules (cross-cutting)

- **Same birth either way.** The event identity and producer ride the PDF marker as
  synced properties (quote `event_id`; the marker types all-digit values as numbers). A
  PDF that intake has to queue is born with the same LINKS line and recap when
  `bob ref scan` files it later. A test asserts byte-identical births on both paths.
- **Honest states.** Never print "filed" before the commit exists. `queued` names the
  next step.
- **Clipboard races.** A client's preview reports `notes.sha256`, and filing passes
  `--expect-sha256`. A mismatch is `notes_changed`, never a silent filing of different
  text.
- **Generated mirror.** Do not write the event row's LINKED FROM; reconcile owns it.
  This follows the precedent of the Highlights annotation-relation birth.
- **Tests.** Deterministic: use `BOB_NOW`, the `BOB_PANDOC_COMMAND`/xelatex stubs for
  pipeline tests, and skip real-render tests when pandoc or xelatex is missing.

### Deferred (recorded as follow-ups, not built here)

- **Wispr local store adapter** (Mac-only, schema-probed, `doctor`-gated).
- **Dedicated Copy-as-Markdown adapter** (`:::toggle`, `<@speaker:N>`). Build it from a
  real paste fixture; the generic Markdown path covers pastes until then.
- **Layer-2 enrichment.** A context packer plus a configured command or SASE skill, with
  logged corrections.
- **Owner-aware next-step offers.**
- **Display inverse pair** `notes from ⇄ notes in`.
- **`prep for` label.**

## Verified anchors (from exploration; re-verify before editing)

- **`bob ref` CLI.**
  - Builder: `src/native/highlights_ref/cli.rs`. `HELP_GROUPS` puts `meet` in
    "Highlights pipeline" alphabetically. Test at `cli.rs:414`.
  - Dispatch: `highlights_ref/mod.rs:354-403`.
- **Create.**
  - `create.rs` (Markdown route: `plan_markdown` ~5423, `create_markdown_plan` ~4018,
    pandoc ~5986).
  - Route precedent: `create/podcast_route.rs`.
  - Create checks read clap ids directly (`preflight_before_parent` ~1118). Use
    `Config::for_vault` when those ids do not exist.
  - The producer string is hard-coded at `create.rs:455`.
- **Intake.** `intake.rs`: `IntakeSession::open/ingest`, `path_set` ~513-581, outcomes
  ~139, `intake_commit_subject` ~1098.
- **Birth.**
  - `reading_plan.rs` (`plan_birth_task` ~615, `v2_birth_body` ~444).
  - `reading_execute.rs` (`warning_child`; generalize it into children).
  - `ref_tasks/insert.rs:72` already takes `children`.
  - Relation precedent: `sync.rs:2207` (`format_links_line`, `canonical_link`).
- **Marker.** `stamp.rs` (`compose_marker` ~874, `MARKER_EXTRA_ORDER` ~72) and
  `marker.rs` (unknown keys sync to frontmatter; value typing ~533).
- **Store.** `source_store.rs` (`write_for_create` ~1130, `find_birth_store` ~550).
- **Events and gcal.**
  - `src/native/event_task.rs`: `event_task_by_id` ~631, `events_in_note` ~665,
    `clean_event_title` ~1065.
  - `gcal::journal::load_state`; `gcal::note::decode_calendar_eid` is private, so
    re-export it or move it. The eid's calendar token is shorthand, so use the journal
    for the calendar id.
  - `gcal::lifecycle::split_row_spans` and `gcal::render::set_row_metadata` (private).
- **Relations.**
  `task_relations::{label::normalize_relation_label, format::format_links_line}`,
  `config::relations::RelationsConfig::label_effect`, and
  `task_dependencies::format::canonical_link`.
- **Escaping and digests.** `task_text_escape.rs`, `gmail/body/inert.rs`, and
  `intentions::digest` (`intentions/state.rs:43`).
- **Catalog.** `catalog/table.rs` (`ref-create` row ~1073; every public leaf needs a Row
  or Excluded, `catalog/tests.rs:93`).
- **Clipboard.** `capture_clip::clipboard::read_clipboard`.
- **Tests.**
  - `tests/cli/highlights/{create,ingest}.rs`
  - `tests/cli/support.rs` (`bob_command`, `init_vault_sync_pair`,
    `write_highlights_pdf_pages`)
  - `tests/cli/docs_reference.rs` (golden CLI help)

## Phases

### notes-normalizer — Structural notes normalizer, dossier source, and recap renderer

Create `src/native/meetings/` (new module; nothing named `meetings` exists) with a pure,
I/O-free notes layer:

- **Intake.** `read_notes(source)` handles a path, stdin, or a string (the CLI does the
  clipboard read). It enforces the intake rules, returns the text plus `sha256` of the
  bytes read, and raises typed errors that map to the JSON codes.
- **Model.**
  `MeetingNotes { own_title, tldr, sections[{heading, items}], next_steps[{owner, action}], transcript, shape, warnings }`
  via `normalize(text)`, implementing the shapes above.
- **Dossier.** `render_dossier(&notes, &DossierMeta) -> String`, where `DossierMeta`
  carries the event title, date, span, root, producer, filed date, transcript flag, and
  edited flag.
  - It renders pandoc `commonmark_x` input. The meta goes in YAML front matter for the
    print profile; the body is the TL;DR block, Next steps, the sections, and the
    optional transcript appendix.
  - Choose a TL;DR syntax the profile can target, for example a fenced div. Document the
    shape in the module docs.
  - **Round-trip contract:** `normalize(render_dossier(n, m))` reproduces `n`, and a
    `parse_dossier` recovers `m`. Birth relies on this, so test it as a property.
- **Recap.** `render_recap(&notes, &RecapMeta, scope) -> String` renders the region
  including its markers and digest. Also add
  `recap_state(note_body) -> Absent | Owned | Edited` and
  `replace_recap(note_body, new)`, which refuses when the state is `Edited`.
  - > [!decision] recap_scope `scope` follows the recap_scope decision. Implement only
    > the selected scope as the shipped behavior; a parameter is fine.
- **Fixtures.** `tests/fixtures/meetings/` holds **synthetic** inputs with goldens:
  - A plain-text summary in Wispr's exact shape. Use invented people and companies, but
    include a misheard product name, a "sounded like 'XYZ'" hedge, a numeric-range
    discrepancy, and a next step owned by the _other_ party.
  - A Markdown paste with an H1, `##` sections, a Transcript section, and a checkbox
    next step.
  - An adversarial input with `#task`, `- [ ] x`, `[[Link]]`, `key:: value`, a trailing
    `^id`, `%%c%%`, `<!-- bob:meeting-recap:end -->`, and raw HTML.
  - An unrecognized blob.
  - CRLF, BOM, and empty inputs.
- **Tests.** Goldens for the model, the dossier, and the recap. The word-sequence
  invariant as a property test. Every adversarial token inert in the recap. Digest
  stability.
- **Done when.** `just` lint and test gates pass. The module has no callers yet besides
  tests. Expose only `pub(crate)` APIs.

### meeting-resolver — Event Task resolution, matching, recent-meetings listing, and agenda notes state

Add a read-only layer in `src/native/meetings/`:

- **`parse_event_token(raw)`.** Normalize the ref forms in the contract (bare
  `20261123#^id`, path forms, wikilinks with alias, `note:id`, a leading `~`) to a
  `TaskLocator`, or recognize a Google Calendar URL, or fall through to query words.
- **`resolve_event(vault, token, window, day)`.** Returns
  `MeetingRow { path, line, block_id, date, span, title, status, root, root_is_inbox, calendar_url, event_id?, series_id?, calendar_id? }`.
  - Use `event_task_by_id` and `events_in_note` over daily notes from
    `gcal::snapshot::daily_relative_path`.
  - Read root and type from the raw line (`task_root::parse_task_root`,
    `task_theme::parse_task_theme`).
  - Get the 📅 URL via `gcal::lifecycle::split_row_spans`.
  - Get Google identity from `gcal::journal::load_state`, matching `(path, block_id)`
    first, then the URL. Fall back to decoding `eid`.
  - Rows without a `^id` refuse with a clear hint to add one.
- **`recent_meetings(vault, window, now)`.** Started, non-cancelled Event Task rows,
  newest first, each with its **filed state**. The state comes from scanning
  `ref/meetings/*.md` frontmatter (`event_id`, else the `event` link) and is returned as
  `{note, task, status}`.
- **`auto_match(notes_text, candidates)`.** Scoring and evidence as specified in
  Matching. Make the stopword list a reviewed constant.
- **`meeting_parent(vault, row, explicit)`.** Wraps `parent_notes::resolve_parent` and
  the inbox check. Returns `Parent | NeedsParent { root, candidates }`.
- **Capture agenda (additive).** In the `capture-pomodoros --tasks` JSON
  (`capture_pomodoros_agenda.rs`), every started Event Task row in `events.rows[]` gains
  two fields:
  - `event_ref`, the exact EVENT token, for example `20261123#^bryan-bugyi-derrick`
  - `meeting_notes: {state: "missing"}` or `{state: "filed", note, task}`

  Keep the schema version unchanged and document the fields where the agenda JSON is
  documented.

- **Tests.** Use temp-vault fixtures with synthetic daily notes and journal records:
  - each token form
  - URL plus journal identity
  - title-word queries
  - the auto-match gate, including a recurring-title tie that must not auto-select
  - inbox detection with `meet_inbox`
  - the filed-state scan
  - agenda JSON golden updates
- **Done when.** `just` gates pass, and there are no writes anywhere in this phase.

### dossier-print — Meeting dossier print profile

- **Profile.** Add the meeting profile (TeX header plus Lua filter, embedded with
  `include_str!`) and
  `render_meeting_pdf(dossier_md, scratch, filters, meta) -> RenderOutcome`. It should
  share the note snapshot's fonts, glyph table, xelatex invocation, and warning parsing;
  generalize instead of copying.
- **Visual spec.** Implement The dossier PDF spec above. Declare any new TeX package in
  `render_tex::RENDER_TEX_PACKAGES` so `bob ref doctor` checks it.
- **PDF Info.** Title (the reference title), Subject (`meeting notes · <event ref>`),
  and Keywords (`meetings`).
- **Tests.** Render the synthetic fixtures for real, skipping when pandoc or xelatex is
  missing.
  - Assert the page count for the summary fixture (1–2), no TOC, no overfull or
    missing-glyph warnings, and `pdftotext` word order.
  - Assert the transcript appendix starts on its own page with `-t`.
- **Visual check.** Render page PNGs with `pdftoppm` and look at them. Iterate until the
  title card, TL;DR box, and next-steps table look intentional. Attach the PNGs as
  artifacts with `/sase_artifact` for Bryan. They must show synthetic content only.
- **Done when.** `just` gates pass and the visual check is recorded.

### meeting-birth — Marker-carried meeting identity births the LINKS relation and recap

- **Trigger.** At birth (`plan_birth_task` / `v2_birth_body` through `IntakeSession`,
  for both inline ingest and the `bob ref scan -i` drain), a PDF whose type is
  `meetings` and whose marker carries `event`:
  1. **Reading task.** It is born with the child
     `🪢 **LINKS:** <meetings.relation_label> <canonical_link to the event>`.
     - Resolve the event link to `(path, block_id)` at birth.
     - If the event no longer resolves, birth still succeeds. Use the existing warning
       child to report it, and write no dangling LINKS link.
     - Generalize `warning_child` into an ordered list of child lines.
  2. **Note body.** It is born with the recap region between the embed and
     `## Highlights`. The recap is rendered by `notes-normalizer` from the birth store's
     `source.md`, which `find_birth_store` locates.
     - Parse it with the round-trip parser.
     - If the store is missing, birth an empty owned region and add a warning.
- **Validation.** A configured label that is invalid or carries an effect is a config
  error at load time, not a silent skip.
- **Preservation.**
  - Adopting an orphan task (`select_birth_task`) never adds a second LINKS line.
  - Later syncs, scans, renames, and status changes preserve the recap and the LINKS
    line byte for byte.
- **Tests.**
  - A fixture PDF whose marker carries meeting properties is born identically through
    inline ingest and through the queued drain. Compare the task line, its children, and
    the note bytes.
  - Missing store and missing event.
  - Rename and re-sync preservation.
  - A rejected effect label.
- **Done when.** `just` gates pass. No CLI surface yet.

### ref-meet-cli — The `bob ref meet` command

Wire everything into the command contract above.

- **CLI.**
  - Add the `meet` leaf (`highlights_ref/meet_cli.rs` or similar) with clap help that
    meets `cli_rules`: alphabetical long options, short aliases, an after-help with
    examples, and `HELP_GROUPS`.
  - Add the route `create/meet_route.rs`, modeled on `podcast_route.rs`. It builds
    `CreateOptions` directly; it does not call `prepare_target`, because in-vault `.md`
    must not become a snapshot.
- **Pipeline.** Normalize, then render the dossier with the meeting profile. Then stamp
  the marker with `title`, `parent`, `status`, the event properties, `source_url`, and
  `notes_producer`. Then install into `lib/meetings/`, persist the source store, and
  finish intake with producer `bob ref meet`.
- **Inbox re-root.** Add the daily-note edit to the intake `path_set` (in place and
  gcal-safe, as specified).
- **Modes.**
  - List mode, plus the TTY pickers and prompts and the `-e` editor flow (scissors block
    as in `gcal note -e`; record `edited` in the dossier meta and the provenance line).
  - `-c` through `read_clipboard`, `-i FILE`, `-i -`, and the hidden `--expect-sha256`.
  - `-O` with the digest-guarded recap re-seed.
- **Outputs.** Human receipts and the JSON envelope exactly as specified, including the
  honest `queued` outcome and the error codes and exit codes.
- **Config.** Add the `meetings:` section with validation. Document it in
  `docs/reference/configuration.md`.
- **Catalog.**
  - `ref-meet`: writes, preview-first, preview `-d`, an EVENT text slot, adapter
    `ref-meet`, invalidates `references` and `tasks`.
  - `ref-meet-list`: reads, fixed `-l`, report presentation.
  - Satisfy the `catalog/tests.rs` invariants.
- **Docs** (docs memory routes). Use fictional examples only.
  - A new behavior contract `docs/meeting-notes.md`.
  - A guide `docs/guides/file-meeting-notes.md`, covering Mac, Obsidian, and CLI.
  - Mentions in `docs/ref.md` and `docs/concepts/reference-library.md`.
  - `docs/reference/json-outputs.md`.
  - The regenerated CLI reference (`just docs-gen`).
  - `mkdocs.yml` nav.
  - `just docs-check` must pass.
- **Memory.**
  - > [!decision] memory_decision_record Use `/sase_memory_write` to add a decision
    > record to the `decisions` web, for example `meeting-notes-are-event-linked-refs`.
    > Cover where it applies; the claim (meetings references, root-as-parent,
    > `notes from` relation authored on the reading task, Next birth, recap ownership,
    > structure-only normalization); why it beat the alternatives (private event notes,
    > daily-note bullets, the `meet` notebook, `chat` type, a top-level `bob meet`, AI
    > in v1); cost; reopen triggers; and evidence (this plan, the research report, and
    > the commits).
  - > [!decision] memory_glossary_term Use `/sase_memory_write` to add a glossary term
    > (for example "Meeting Ref", aka meeting notes, meetings reference) defining the
    > type, its recap region, the relation, and the gestures. Link it to the Reference
    > Note, Reference Task, Event Task, and Task Relation terms.
- **Tests.** End-to-end CLI tests in `tests/cli/` with a temp vault-sync pair and pandoc
  and xelatex stubs:
  - an explicit ref, a query, an auto match, and ambiguity (exit 2 with candidates)
  - an inbox root (exit 2, then `-P` plus re-root in the same commit, gcal-safe bytes)
  - already filed and `-O` (owned versus edited recap)
  - a dry run that writes nothing and takes no locks
  - the queued path
  - JSON goldens
  - a stdin and `--expect-sha256` mismatch
  - list mode
  - golden help

### first-use — File the first meeting's notes

Run in the private vault only.

- **Build.** Build the workspace binary (`cargo build --release`) and use
  `target/release/bob -b ~/bob` (the release train may not have shipped yet).
- **Dry run.**
  `… ref meet 20261123#^bryan-bugyi-derrick -i ~/tmp/bryan_derrick_meet_notes.txt -d`.
  Also check that auto-match without EVENT picks the same row.
- **File.** Drop `-d`, setting `-p "Wispr Flow"` unless config already sets it.
- **Verify.**
  - The reading task in the event's root note, with its LINKS line.
  - The ref note frontmatter and recap.
  - The word-sequence invariant on the real file.
  - PDF page PNGs: look at them; do **not** attach them anywhere public.
  - One commit pushed.
  - `ref show`.
  - The event's LINKED FROM after the next reconcile. 20261123 is no longer the previous
    daily note by then.
- **Scaffolding.** Inspect the current state of `job#^whisperflow-meeting` and the task
  it BLOCKS. Skip any step that is already moot, dry-run every gesture, and use only
  supported gestures (capture relations or dependency gestures). Never edit a task line
  by hand.
  - > [!decision] scaffolding = rehang First re-point the gated task's Depends-On from
    > the chore to the new reading task. Then relate the chore `superseded by` the new
    > reading task, a configured cancel label, so it is cancelled with a recorded
    > reason. Doing it in that order means the gated task never flickers Ready.
  - > [!decision] scaffolding = release Relate the chore `superseded by` the new reading
    > task. The gated task becomes Ready through the normal unblock (tickler) path.
  - > [!decision] scaffolding = leave Change nothing beyond the filing. Print the exact
    > commands for either option.
  - In every case, leave `bob#^wispr-flow-meet-notes` open for Bryan to close.
- **Follow-ups.** Record each Deferred item as a `PROPOSED FOLLOW-UP:` note on this
  phase's bead. Ask for a real Wispr "Copy as Markdown" paste fixture so the dedicated
  adapter can be built.

### obsidian-surfaces — Obsidian surfaces for meeting notes

In bob-plugins (open it with `/sase_repo`; read its AGENTS.md; build from `src/`
fragments; keep fragments at 1000 lines or fewer; never hand-edit `main.js`):

- **`bob-command-mode` row.**
  - In `cmSheetCaptureContext` (`src/170-sheet.js`), detect the Event Task that owns the
    cursor line. Use `bob-navigation-hotkeys`' `api.taskScope.at`; the row must be in a
    canonical daily note's `## Events` and have a `^id`.
  - Add a FOR THIS NOTE row in `cmSheetContextRows` (`src/160-sheet-rank.js`), titled
    _File meeting notes from clipboard_ with the event title as subtitle. Its argv is
    `ref meet <YYYYMMDD>#^<id> -c`, preview-first: Shift+Enter shows the dry-run
    outline, Ctrl+Enter files.
  - Hide the row when the catalog lacks `ref-meet` (an older bob).
- **Adapter.** Add a `ref-meet` outcome adapter (`src/130-adapters.js`) with **Open
  note** and **Open event** actions, modeled on `ref-create`.
- **Reading Board.** Add a `meetings` kind (`242-plugin-reading-board-models.js`), for
  example `🤝 Meeting`.
- **Docs.** `docs/bob-command-mode.md` (FOR THIS NOTE table), the Reading Board docs,
  then `npm run docs:gen` and `docs:check`.
- **Tests.** `scripts/*.cjs` tests registered in `package.json` `test:unit`: context-row
  detection on and off an event row, argv shape, and the older-catalog fallback.
- **Ship.** `npm run build`, the repo's gates, then `bob plugins sync`.
- **Done when.** Synthetic fixtures only.

### mac-surfaces — Bob Mac Capture meeting-notes mode and Meetings kind

In bob-mac-capture (open it with `/sase_repo`; read its AGENTS.md; GitHub Actions is the
only gate for macOS targets; watch the exact-SHA run to green with `/sase_monitor`, or
gate first with `scripts/dev/macos-ci-check`):

- **Agenda.** Decode `event_ref` and `meeting_notes` with `decodeIfPresent`. Carry them
  into `CaptureAgendaCalendarRowData`.
  - On started rows whose state is `missing`, show a focusable **Notes** trailing action
    beside the Event note button (`CaptureAgendaView.swift` ~1402-1427). It opens
    Meeting Notes mode with that meeting preselected.
  - Filed rows show nothing new.
- **Meeting Notes mode.** Model it on event note mode: a CaptureCore session phase
  machine, a panel model, and views, with the capture draft retained underneath.
  - **Entry points:**
    - a capture-panel key; choose a free one after checking `docs/keyboard.md` and
      `CaptureKeyCommandRouter` (prefer ⌘M)
    - a status-menu item **File Meeting Notes…**
    - the agenda Notes action
  - **Loading.** Run `bob ref meet -l -f json` and
    `bob ref meet [<event_ref>] -c -d -f json`. bob reads the pasteboard, so the app
    never parses notes.
  - **Layout.** A header with the meeting title, day · span, and destination
    `root · Next`.
    - Show match evidence chips when auto-matched, and a **Change meeting** list from
      list mode (filed rows marked ✓).
    - Show an outline preview: TL;DR, next steps, section headings with counts, the
      transcript note, and warnings.
    - Add a Next ↔ Read status toggle.
  - **Keys.** Return files with `--expect-sha256` from the preview. Esc, `q`, and
    `Ctrl+[` back out (decision `q-and-ctrl-bracket-mirror-esc`).
  - **Errors.**
    - `needs_parent` → a parent route card from `candidates`
    - `already_filed` → a Replace confirm (`-O`)
    - `notes_changed` → re-preview with a notice
    - an empty or unreadable clipboard → a calm empty state
  - **Success.** A receipt banner with Open note, then an agenda refresh.
  - **Older bob.** When bob lacks `ref meet` (exit 2 and "unrecognized subcommand", as
    with `gcal note`), hide the feature.
- **Bob Refs.** Add `RefKind.meeting` for `ref_type: meetings` with every exhaustive
  switch the compiler flags: label "Meeting", a symbol such as `person.2.fill`, tint,
  `kindWords` (`meeting`, `notes`), captions, and inspector. Add a ⌘6 Meetings scope
  only if the key is free.
- **Docs.** New `docs/meeting-notes.md`, plus `docs/idle-agenda.md`, `docs/keyboard.md`,
  `docs/refs.md`, and `docs/protocol.md` (optional commands and compatibility). Run
  `just docs-check`.
- **Tests.** CaptureCore and RefsCore tests (session phases, JSON decoding with and
  without the new fields, expect-sha flow, kind decoding) and app-target tests where the
  pattern exists. `just portable-check` locally, then macOS CI green on the pushed SHA.
- **Done when.** Synthetic fixtures only.
