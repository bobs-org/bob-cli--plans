---
tier: epic
title: Google Calendar descriptions become Event Task bullets
goal: "`bob gcal sync` writes the human part of each Google Calendar event description
  as sanitized, capped, flat bullets directly under the Event Task it creates. Directive
  lines and video-call boilerplate are left out. The bullets follow Google until Bryan
  edits them, they can never trigger Bob, Dataview, or Tasks behavior, and they read
  cleanly in Source mode, Live Preview, and Reading view.

  "
phases:
  - id: note-escaper
    title: Inert note-text escaper
    depends_on: []
    size: small
    description:
      "note-escaper: lift gkeep's escape_child_text into task_text_escape.rs unchanged
      and add escape_note_text / escape_note_inline, which make imported prose inert."
  - id: cancel-log-indent
    title: CANCEL LOG indentation matches the plugin writer
    depends_on: []
    size: small
    description:
      "cancel-log-indent: make the gcal CANCEL LOG writer reuse the task's existing
      child indentation (else TAB) so a later CANCEL LOG nests correctly above
      tab-indented description bullets."
  - id: description-parser
    title: Structured description parser
    depends_on: []
    size: medium
    description:
      "description-parser: one linear HTML/plain-text parse into structured lines that
      feeds both directive verdicts (shared vectors stay byte-identical) and the future
      bullets. GcalEvent carries the result behind a redacted Debug."
  - id: boilerplate-filter
    title: Separator, badge-duplicate, and vendor boilerplate filter
    depends_on:
      - description-parser
    size: medium
    description:
      "boilerplate-filter: drop separators, lines that only repeat the 📹 or 📍 target,
      and fenced or header-anchored Meet, Zoom, Teams, and Webex blocks. Keep a join
      link only when the row has no 📹 badge."
  - id: bullet-renderer
    title: Bullet renderer, guards, and scanner-isolation gate
    depends_on:
      - note-escaper
      - description-parser
    size: medium
    description:
      "bullet-renderer: turn parsed lines into sanitized, capped bullet bodies with tidy
      links, drawer and managed-log guards, an overflow line, an oversize fallback, and
      a stable hash. A differential test against Bob's real scanners proves the output
      is inert."
  - id: birth-bullets
    title: Bullets on created and upgraded Event Tasks
    depends_on:
      - boilerplate-filter
      - bullet-renderer
    size: medium
    description:
      "birth-bullets: add the config keys, write the bullets with every created Event
      Task and Maybe→Yes upgrade, add the tri-state journal description record carried
      by every row record, match rows only on top-level lines, add ≡ N and
      description_lines, and document birth-time behavior."
  - id: live-bullets
    title: Description bullets follow Google until edited
    depends_on:
      - birth-bullets
      - cancel-log-indent
    size: medium
    description:
      "live-bullets: locate the journaled block, rewrite or remove it while untouched,
      keep Bryan's edits with one kept_description notice, backfill rows synced before
      the feature, match crash adoptions, and update the ownership contract docs."
  - id: decision-and-review
    title: Decision record and end-to-end review
    depends_on:
      - live-bullets
    size: small
    description:
      "decision-and-review: author the new decision record and the supersession mark,
      regenerate docs, run the full gates, and do a read-only end-to-end review against
      the A1–A10 requirements."
decisions:
  decision_record:
    ask: Add the "Gcal Description Bullets Follow Google Until Edited" decision record?
    memory:
      - decisions:gcal-description-bullets-follow-google-until-edited
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  supersede_propose_once:
    ask:
      Mark bob-gcal-is-read-only-and-proposes-once as superseded in part by that new
      record?
    memory:
      - decisions:bob-gcal-is-read-only-and-proposes-once
    default: true
    corrects: the vault owns the checkbox, `[root::]`, children, and the row itself
    answer: true
proposed_by: bbugyi200.athena.research.5o.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 09:28:49
status: wip
---

- **PROMPT:**
  [prompts/202611/gcal_description_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_description_bullets.md)

# Plan: Google Calendar descriptions become Event Task bullets

## Context

Bryan asked for `bob gcal sync` to turn each Google Calendar event description into
sub-bullets on the Event Task it creates, leaving out the directive syntax sync already
parses. The research report
`research:202611/gcal_event_description_notes/gcal_event_description_notes.md` (read it
with `sase artifact read`) critiques that request and recommends ten requirement
adjustments, A1–A10. **Bryan accepted all of them.** This plan implements them. Where
the research left a detail open or got one wrong, this plan makes the call. Those calls
are listed under [Design calls beyond the research](#design-calls-beyond-the-research).

The accepted requirements, in short:

| #   | Requirement                                                                                                                                                                |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | Event Tasks only. Context rows never get bullets. A Maybe→Yes upgrade adds them.                                                                                           |
| A2  | Exclude directive lines, recognized conferencing blocks, separators, blank lines, and lines that only repeat the `📹` or `📍` target.                                      |
| A3  | The bullets follow Google until Bryan edits them. An edit, a nested note, or a deletion makes them his for good. One notice appears, and only if Google changes afterward. |
| A4  | One shape: flat first children, with no drawer and no header.                                                                                                              |
| A5  | Description text is inert data: a sanitizer, structural guards, and a scanner-isolation gate.                                                                              |
| A6  | Capped at 8 rendered lines, with a linked overflow line.                                                                                                                   |
| A7  | Rows synced before the feature get their bullets on the next sync of their day. Matching existing content is adopted, never duplicated.                                    |
| A8  | JSON carries only `description_lines`, never description text.                                                                                                             |
| A9  | The directive grammar does not change. `Bring shoes #offclock` stays prose.                                                                                                |
| A10 | `gcal.description: off` is the kill switch.                                                                                                                                |

## The contract Bryan sees

**One sentence, for docs and help:**

> Whatever you write in a Google event's description, except Bob's directive lines and
> video-call boilerplate, appears as bullets at the top of that meeting's Event Task,
> and keeps following Google until you edit those bullets.

**The motivating case** is real: the 08:00 event has the description
`And retainers!\n\n@love #offclock`. The directive line still drives `#offclock` and
`[root:: love]`, and the human line becomes the bullet:

```markdown
- [*] #task **0800-0815** Swings! [📅](… "Open in Google Calendar") #offclock [root::
  love] ^swings
  - And retainers!
```

**A work agenda written in Google's web editor**
(`<b>Agenda:</b><ul><li>Intros</li><li>System design – <a href="https://www.google.com/url?q=https://docs.google.com/document/d/1AbC/edit&sa=D">prep doc</a></li></ul>Bring laptop &amp; ID<br>@job -15`):

```markdown
- [*] #task **1400-1500** Onsite interview [📅](…) [buffer:: -15m] [root:: job] [type::
  MEET] ^onsite-interview
  - **Agenda:**
    - Intros
    - System design – [prep doc](https://docs.google.com/document/d/1AbC/edit)
  - Bring laptop & ID
```

The Google redirect is unwrapped, the list nests under its lead-in, and the directive
line `@job -15` is consumed: it drives `[buffer:: -15m]` and `[root:: job]` and writes
no bullet. An `<ol>` or a plain `1.` list renders as native `1. Intros` items.

**A Teams or Zoom invite whose description is only vendor boilerplate writes nothing.**
The row looks exactly as it does today.

**A long description** is capped at `gcal.description_lines` (default 8), and the
overflow line counts toward that cap:

```markdown
    - Welcome to the loop! Here is what to expect…
    - …six more bullets…
    - *…5 more lines in [Google Calendar](https://www.google.com/calendar/event?eid=…)*
```

**Hostile organizer text is inert.** It renders as plain prose in Reading view:

| Description line                                   | Bullet written                                                                           |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| ``Run `$= dv.el("b", 1)` now``                     | ``- Run \`$= dv.el("b", 1)\` now``                                                       |
| `- [ ] RSVP #task by Friday`                       | `- RSVP \#task by Friday` (checkbox glyph stripped)                                      |
| `See [[Secret Note]] ![](https://t.example/p.png)` | `- See \[\[Secret Note]]` (an image keeps only its alt text, so the tracking pixel goes) |
| `<b>Work log</b>`                                  | `- *Work log*` (bold would forge a managed log)                                          |
| `TBD`                                              | `- **TBD**` (a bare ALL-CAPS child would be a drawer)                                    |
| `Bring 🍅 salad`                                   | `- Bring &#127813; salad` (still shows 🍅)                                               |
| `Room:: 4B`                                        | `- Room\:\: 4B`                                                                          |
| `[click](obsidian://advanced-uri?…)`               | `- click` (scheme not allowed; text kept)                                                |
| `Use <% tp.date.now() %> here`                     | `- Use \<% tp.date.now() %> here`                                                        |

**After Bryan adds a note and the meeting is declined:**

```markdown
- [-] #task **1400-1500** Onsite interview [📅](…) … [cancelled:: 2026-11-03]
  ^onsite-interview
  - ❌ **CANCEL LOG**
    - _2026-11-03_ — Declined in Google Calendar
  - **Agenda:**
    - Intros
    - System design – [prep doc](…)
  - Bring laptop & ID
  - Asked recruiter to reschedule
```

Managed first children such as the CANCEL LOG, a Depends-On line, or LINKS go above the
bullets. Bryan's own notes go below them. All of them share one indentation style.

**Command surfaces:**

- `sync -d` shows the exact bullets in its diff. This is the only place description text
  appears in command output.
- Human `sync` rows end with a dim `≡ N` when the run wrote N bullet lines for that row,
  for example `+ task    08:00–08:15  Swings!  ● love  via @love  ≡ 1`.
- The `list` table adds a dim `≡ N` after the title of each Event Task with bullets: the
  number of lines sync would write.
- JSON is additive and `schema_version` stays 1. `sync -f json` rows and `list -f json`
  events gain an integer `description_lines`, and never text.
- Config:

  ```yaml
  gcal:
    description: bullets # bullets | off
    description_lines: 8 # most bullet lines per Event Task, overflow line included
  ```

## Design calls beyond the research

I verified the research against the code. These calls refine it, and every phase must
follow them:

1. **Drawer guard on every direct child, not only on bullets with children.** Capture's
   drawer grammar (`capture_drawers::parse_drawer_title`) does not require a nested
   item. So a bare `- TBD` under a task _is_ a drawer. Any bullet whose whole plain text
   passes `is_drawer_title` is rendered bold (`**TBD**`). `*` is not a drawer-title
   character, so bold text can never be a drawer.
2. **Managed-line guard falls back to italic.** A line that any managed-line predicate
   would claim (`parse_managed_task_log_marker`,
   `task_dependencies::is_managed_link_line`, the CANCEL LOG child shape, or a reserved
   drawer key) has its body rendered in italics instead. For example, `<b>Work log</b>`
   becomes `- *Work log*`. The differential gate in `bullet-renderer` is the arbiter. If
   a predicate still claims an italic line, escape further until the gate passes.
3. **Every `[` in a text node becomes `\[`.** This one rule covers wikilinks, embeds,
   footnotes, checkboxes, and reference-link definitions such as `- [y]: url`, which
   CommonMark honors inside list items. Links that Bob builds itself are emitted
   verbatim.
4. **`<` becomes `\<`, not `&lt;`.** It is just as inert, because a backslash-escaped
   `<` cannot open a tag, and it reads better in Source mode.
5. **Ordered lists stay ordered.** `<ol>` items and plain `1.` / `1)` lines render as
   native `1. text` items (Bob emits the marker; the text is escaped). The research's
   `- 1\. Intros` showed a stray backslash in Source mode. A _text_ that begins with
   `N.` or `N)` is escaped as `N\.` / `N\)`, because a backslash before a digit is not a
   CommonMark escape.
6. **Rows match only on top-level lines.** The overflow line and the oversize fallback
   contain the event's `htmlLink`. Today `find_row_line`, `insert_slot`/`row_sort_key`,
   and `list.rs`'s `row_present`/`adopted_block_id` would treat an indented bullet that
   contains the URL, or starts with `**HHMM-HHMM**`, as a row. URL, text, and slot
   matching must consider only lines that `lifecycle::split_row_spans` parses, which are
   column-0 `- ` rows. Block-id matching is unchanged.
7. **CANCEL LOG indentation matches the plugin.**
   `task_cancel::line::edit_source_create_log`, whose only caller is gcal's lifecycle
   cancel, hard-codes two spaces. A two-space marker above tab-indented bullets
   mis-nests in Obsidian: the tab reaches column 4, so the bullets render _inside_ the
   CANCEL LOG. Meanwhile Bob's byte-based `child_block_end_line` disagrees with that
   nesting. bob-navigation-hotkeys' Task Card writer already reuses an existing child
   indent, else TAB (`getDependencyChildIndent`, `MANAGED_TASK_LOG_INDENT_UNIT = "\t"`).
   The Rust writer adopts the same rule.
8. **Bullet indentation follows the row's existing children.** Depth 1 uses the row's
   first direct child indent. If the row has no children, it uses the row indent plus
   the note's `dominant_indent_unit`, falling back to TAB. Depth 2 adds the same unit
   again, where the unit is the depth-1 indent minus the row indent.
9. **The 64 KiB bound selects output, not parsing.** The tokenizer is linear-time and
   always parses the full text, so directives keep working on huge descriptions. Input
   above 64 KiB renders the single fallback bullet instead of its content.
10. **Secondary vendor anchors need a primary anchor.** Phrases such as `Meeting ID:`,
    `Passcode:`, `PIN:`, and `One tap mobile` start a vendor block only when the
    description also has a primary anchor: a vendor header such as `Join Zoom Meeting`
    or `Microsoft Teams meeting`, or a vendor join URL. That keeps "Meeting ID: decide
    whether to publish" as prose in an ordinary agenda.
11. **The journal state has three values, and a malformed value is hands-off.** The
    record is absent, `{"hash","lines"}`, or `null`. Any malformed `description` value
    parses as hands-off (never touched) rather than as absent. Absent means the row
    predates the feature, which would trigger a backfill.
12. **No description text in Debug output.** `EventDescription` implements `Debug` by
    hand, printing line counts only, so `GcalEvent`'s derived `Debug` never leaks text
    into logs or test failure output.
13. **`description_lines` must be at least 2.** One content line plus the overflow line.
    A wrong type or a value below 2 is a setup error that names the key.
14. **Ownership notices fire only when they matter.** `kept_description` is reported
    once, only when Google's rendered text changed after Bryan's edit (the `kept_root`
    model). An edit alone is silent, because editing the bullets is the expected flow.

## Architecture

### Pipeline

```text
raw description ──► tokenize (HTML or plain) ──► source lines (+ blanks, separators)
                                                      │
                     directive verdicts ◄─────────────┤  (one parse, two consumers)
                                                      ▼
            filter: directive lines │ separators │ badge duplicates │ vendor blocks
                                                      ▼
            outline: drop blanks, lead-in nesting, depth cap 2, ordered markers
                                                      ▼
            EventDescription { lines, directives, oversized }      (model, parse time)
                                                      ▼
            render: sanitize text nodes │ links │ guards │ cap + overflow │ hash
                                                      ▼
            DescriptionBullets { bullets: [(depth, body)] }        (sync, plan time)
                                                      ▼
            materialize with the row's indentation ──► vault lines
```

### Module layout

This layout keeps parallel phases out of each other's files. Follow the crate's existing
layout convention: a `plan.rs` file with a sibling `plan/` directory.

| File                                                                                     | Owner phase        | Contents                                                                            |
| ---------------------------------------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------- |
| `src/native/task_text_escape.rs`                                                         | note-escaper       | `escape_child_text` (lifted), `escape_note_text`, `escape_note_inline`              |
| `src/native/gcal/description.rs`                                                         | description-parser | Types, `parse_description`, `description_directives`, the filter and outline stages |
| `src/native/gcal/description/html.rs`                                                    | description-parser | HTML tokenizer                                                                      |
| `src/native/gcal/description/plain.rs`                                                   | description-parser | Plain-text tokenizer                                                                |
| `src/native/gcal/description/boilerplate.rs`                                             | boilerplate-filter | Separator, badge-duplicate, and vendor-block filter                                 |
| `src/native/gcal/description/bullets.rs`                                                 | bullet-renderer    | `render_bullets`, `DescriptionBullets`, `bullets_hash`, `materialize`               |
| `src/native/gcal/bullet_block.rs`                                                        | live-bullets       | Block locator, insertion point, and in-place replace (pure)                         |
| `tests/fixtures/gcal_description/{README.md,outline.json,boilerplate.json,bullets.json}` | one file per phase | bob-cli-only vectors                                                                |

`tests/fixtures/event_buffer/vectors.json` is a cross-repo contract mirrored in
bob-plugins. **It never changes in this epic and must stay green.**

### Core types

Phases may refine these names, but downstream phases depend on their shape:

```rust
// description-parser
pub(crate) struct DescriptionContext<'a> { pub join_link: Option<&'a str>, pub location: Option<&'a str> }
pub(crate) struct EventDescription {           // Debug prints counts only
    pub lines: Vec<DescriptionLine>,           // filtered + outlined, NOT yet escaped
    pub directives: DescriptionDirectives,     // existing type, same semantics
    pub oversized: bool,                       // raw input > 64 KiB
}
pub(crate) struct DescriptionLine { pub depth: u8 /* 1|2 */, pub marker: LineMarker, pub inlines: Vec<Inline> }
pub(crate) enum LineMarker { Bullet, Ordered(u32) }
pub(crate) enum Inline { Text { text: String, bold: bool, italic: bool }, Link { text: String, url: String } }
pub(crate) fn parse_description(raw: &str, ctx: &DescriptionContext<'_>) -> EventDescription;

// bullet-renderer
pub(crate) struct Bullet { pub depth: u8, pub body: String }   // body = "- …" or "1. …", no indent
pub(crate) struct DescriptionBullets { pub bullets: Vec<Bullet> }
pub(crate) fn render_bullets(desc: &EventDescription, cap: usize, html_link: Option<&str>) -> DescriptionBullets;
pub(crate) fn bullets_hash<'a>(pairs: impl IntoIterator<Item = (u8, &'a str)>) -> String;
//   = journal::span_hash of the concatenation of "{depth}\t{body}\n"; an empty block hashes "".

// birth-bullets (journal)
pub(crate) enum DescriptionState { Absent, Tracked { hash: String, lines: usize }, HandsOff }
```

### Journal record

`gcal.row` gains a sibling of `spans`. Older binaries read fields with `object.get` and
ignore it, so a downgrade is safe:

```json
{
  "kind": "gcal.row",
  "…": "…",
  "spans": { "time": "…", "title": "…", "badges": "…" },
  "description": { "hash": "3f1c9a0d6b2e8a71", "lines": 2 }
}
```

| Stored              | Meaning                                                                                                                                 |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| key absent          | `Absent`: the row predates the feature, or was synced while `off`. `live-bullets` treats it as an empty tracked block, so it backfills. |
| `{"hash","lines"}`  | `Tracked`: what Bob last wrote. The hash covers `(depth, body)` pairs and ignores indent characters.                                    |
| `null` or malformed | `HandsOff`: never touched.                                                                                                              |

Each fold _replaces_ the previous `RowRecord`. **Every** `gcal.row` that sync appends
must therefore carry the current description state forward. `PlannedRow.description` and
`AdoptedRow.description` are required fields with no `Default`, so the compiler flags
every construction site. Each occurrence gets at most one `gcal.row` record per run,
carrying both span state and description state.

## Phase: note-escaper

**Goal.** Give every importer one strict escaper for prose that must stay inert.

1. Move `escape_child_text`, `escape_leading_marker`, `is_thematic_break`, and
   `is_code_fence` from `src/native/gkeep/render.rs` into
   `src/native/task_text_escape.rs`. Keep the behavior **byte-identical**: gkeep's
   re-import comparisons may depend on it. gkeep imports them from the new home, and its
   tests pass unchanged.
2. Add `escape_note_inline(text)`, which makes mid-line text inert, and
   `escape_note_text(text)`, which is the inline escaper plus the line-start rules for
   text that begins a list item body. Apply each rule in order:

   | Hazard                                                                                             | Rule                                                                       |
   | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
   | `\` followed by ASCII punctuation                                                                  | `\\` first, so organizer-written escapes cannot unescape what follows      |
   | `%%`, `::` runs, exact `#task`, trailing `^id`                                                     | Existing `escape_task_text` rules                                          |
   | `` ` `` anywhere                                                                                   | `` \` `` (Dataview inline queries, DataviewJS `$=`, code spans and fences) |
   | `[` anywhere                                                                                       | `\[` (wikilinks, embeds, footnotes, checkboxes, reference definitions)     |
   | `<` anywhere                                                                                       | `\<` (raw HTML, Templater `<% %>`, HTML comments)                          |
   | `#` at the node start or after whitespace, followed by a non-space, non-`#` char                   | `\#` (tags, phantom `#task`, `#offclock`)                                  |
   | Two or more `$` on one line                                                                        | every `$` → `\$` (inline math such as `$5-$10`)                            |
   | `🍅` (U+1F345)                                                                                     | `&#127813;` (`row_untouched` substring match)                              |
   | Line start (`escape_note_text` only): `#`, `>`, `\|`, `+`/`*`/`-` + space, a thematic break, `~~~` | leading `\`                                                                |
   | Line start (`escape_note_text` only): `N.` or `N)` + space                                         | `N\.` / `N\)`                                                              |

3. Unit-test both directions. Every hostile sample must be neutralized. Ordinary prose
   must stay byte-identical, including `AT&T`, `5 > 3`, `a:b`,
   `see https://x.test/#frag`, `C# and F#`, `Q4: $5`, `it's 50% done`, `*emphasis*`, and
   `~~strike~~`. `see https://x.test/#frag` matters because a `#` preceded by `/` is not
   a tag start.

**Out of scope:** URL handling (that belongs to `bullet-renderer`), and any change to
gkeep's output.

**Acceptance:** `cargo test` for `task_text_escape` and `gkeep` passes, and `just check`
is green.

## Phase: cancel-log-indent

**Goal.** A CANCEL LOG created by gcal nests correctly next to tab-indented description
bullets ([design call 7](#design-calls-beyond-the-research)).

1. In `task_cancel::line::edit_source_create_log`, set the marker indent to the
   indentation of the task's first direct child when one exists. Use
   `capture::first_child_indentation`, as `capture_schedule_log` does. Otherwise use the
   task indent plus `\t`. The entry indent is the marker indent plus `\t`. This is
   exactly what bob-navigation-hotkeys' Task Card cancel writer does when it creates a
   log (`getDependencyChildIndent` plus `MANAGED_TASK_LOG_INDENT_UNIT`). Read
   `plugins/bob-navigation-hotkeys/src/030-bullet-blocks-and-logs.js` in bob-plugins
   through `/sase_repo`, read-only, to confirm parity.
2. Confirm that `edit_source_create_log`'s only caller is gcal lifecycle
   (`grep -rn edit_source_create_log src`). The existing-log path (`edit_source`) and
   the shared `tests/fixtures/task_cancel/vectors.json` stay unchanged.
3. Update the expectations in `tests/gcal_lifecycle.rs` and in unit tests that pin the
   two-space form. Add one test where a row has a tab-indented child: the CANCEL LOG
   lands first, tab-indented, and its entry is two tabs deep. Add another test with no
   children: the CANCEL LOG uses a TAB.
4. If `docs/gcal.md` describes the created CANCEL LOG's indentation, keep it truthful.

**Acceptance:** `cargo test --test gcal_lifecycle` and `just check` are green.

## Phase: description-parser

**Goal.** Build one structured parse of the description that both directive verdicts and
the bullets use, so a line can never count as both a directive and a bullet, or as
neither.

1. **Bound and detect.** Treat the input as HTML when it contains a known tag. Match
   case-insensitively on
   `</?(a|b|strong|i|em|u|s|strike|del|br|p|div|span|font|ul|ol|li|h[1-6]|table|thead|tbody|tr|td|th|img|hr|html|head|body|style|script|title|html-blob)\b`
   or `<!--`. Google's editor wraps text in `<html-blob>`. Anything else takes the plain
   path. Set `oversized` when the raw byte length exceeds 65,536. Always parse the full
   text, and keep every stage linear-time.
2. **HTML tokenizer** (`description/html.rs`, hand-rolled, in the style of today's
   `replace_html_tags` and `decode_html_entities`):
   - `<br>`, `<p>`, `<div>`, `<li>`, `<ul>`, `<ol>`, `<h1>`–`<h6>`, `<tr>`, `<table>`,
     and `<hr>` (plus their closing tags where applicable) end a line. `<hr>` emits a
     separator line.
   - `<ul>` and `<ol>` nesting sets the list depth. Number `<ol>` items per list,
     starting at 1.
   - `<a href>` becomes `Inline::Link`. Parse `href` and `alt` attributes whether they
     are double-quoted, single-quoted, or unquoted, and decode entities in attribute
     values.
   - `<b>`/`<strong>` set bold and `<i>`/`<em>` set italic. A heading line is all-bold.
   - Within a `<tr>`, join `<td>`/`<th>` cells with `·`.
   - `<img>` keeps its alt text, or nothing when it has none.
   - The contents of `<script>`, `<style>`, `<head>`, `<title>`, and `<!-- -->` are
     dropped. Today's stripper keeps that text.
   - Strip any other tag and keep its content. Decode entities in text, as
     `decode_html_entities` does today. Collapse whitespace runs (newlines included),
     turn NBSP into a space, and remove `U+200B`–`U+200D` and `U+FEFF`.
3. **Plain tokenizer** (`description/plain.rs`):
   - Normalize CRLF to LF and NBSP to a space, and strip zero-width characters. Each
     line becomes one source line. Blank lines stay in the stream as blank markers,
     because `boilerplate-filter` needs paragraph boundaries.
   - A leading marker (`-`, `*`, `•`, `◦`, `‣`, `▪`, `N.`, `N)`) followed by a space
     starts a list item. Relative indentation sets its depth.
   - Expand tabs to four columns when measuring indentation.
   - Inline parsing recognizes Markdown links `[text](url)`, Markdown images
     `![alt](url)` (kept as alt text only, like `<img>`), autolinks `<url>`,
     Outlook-style `text<https://…>` (link text = `text`), and bare `http(s)://` URLs. A
     bare URL drops trailing `.,;:!?` and an unbalanced `)`.
   - Decode entities as today, which preserves directive parity.
4. **Classify directives.** Project each source line to plain text, joining inline texts
   and using a link's text. Apply today's `is_directive_token` test to every token,
   splitting on whitespace or commas. Reimplement `description_directives(raw)` on top
   of `parse_description`, keeping first-route-wins and
   first-value-per-buffer-side-wins.
   - **Gate:** `tests/fixtures/event_buffer/vectors.json` stays byte-identical and
     green.
   - Put new directive cases in `tests/fixtures/gcal_description/outline.json`:
     `<h3>@job</h3>` (a heading now ends a line), `text<https://x>` (no longer read as a
     tag), and a directive inside `<li>`.
5. **Filter (this phase's part).** Drop directive lines. Leave a single clearly named
   hook (`filter_source_lines(lines, ctx)`) where `boilerplate-filter` inserts its
   passes. Keep blank and separator lines in the stream up to that hook.
6. **Outline.**
   - Drop blank lines. Each remaining line becomes one `DescriptionLine`, and lines are
     never joined.
   - A _list_ is one `<ul>`/`<ol>` element, or one contiguous run of plain marker lines.
   - **Lead-in rule:** the list that immediately follows a non-list line ending with
     `:`, or an all-bold line (a heading or a bold label), nests one level under that
     line. Only that one list nests.
   - Other list items start at depth 1. Nested lists add depth. Depth is capped at 2,
     and anything deeper flattens to 2.
   - Strip leading checkbox glyphs (`[ ]`, `[x]`, `[X]`, `☐`, `☑`, `☒`) from item text.
     Description checklists never become tasks.
   - Trim whitespace, merge adjacent text inlines that share flags, and drop lines that
     end up empty.
7. **Wire it in.** `GcalEvent` gains `description: EventDescription`, parsed in
   `from_google` with `DescriptionContext { join_link, location }` from the same event.
   Its directive fields come from `description.directives`. Hand-write
   `EventDescription`'s `Debug` so it prints counts only
   ([design call 12](#design-calls-beyond-the-research)). Update the module doc comment
   in `model.rs`. `sync` and `list` do not use the lines yet.
8. **Fixture.** Write `tests/fixtures/gcal_description/README.md`. It defines a compact,
   documented notation for expected lines (depth, marker, inline markup), notes that the
   files are bob-cli-only, and notes that `event_buffer/vectors.json` is the shared
   directive contract. Write `outline.json` with cases covering plain, HTML, and mixed
   input, nested and ordered lists, the lead-in rule, headings, tables, images, dropped
   `<script>`/`<style>`/comments, redirect URLs (kept raw here), and autolinks.
9. **Docs.** `docs/gcal.md` § Off-clock events currently says that `<br>`, `<p>`,
   `<div>`, and `<li>` become line breaks. Update that sentence to the new line-ending
   set (headings, list containers, table rows, `<hr>`). Read `sase/memory/docs.md` first
   with `/sase_memory_read`.

**Out of scope:** separators, vendor blocks, badge duplicates, escaping, and rendering.

**Acceptance:** both vector files pass, `just check` is green, and `just docs-check` is
green.

## Phase: boilerplate-filter

**Goal.** Make sure a real invite's conferencing wall never becomes junk bullets (A2).

All passes go in `description/boilerplate.rs` and are called from the
`filter_source_lines` hook. They run on source lines while blank and separator lines are
still present.

1. **Separators.** A separator is a line of three or more of `-_=*~` (spaces allowed),
   an `<hr>`, Google's `-::~:~::~…::-` fence, or Webex's
   `-- Do not delete or change any of the following text. --`. Separator lines are
   always dropped, after they have served as fences.
2. **Fenced vendor regions.** When a region between two fences, or between a fence and
   the end of the description, contains a primary anchor, drop the whole region along
   with its fences. This covers Teams' underscore walls and Google's fence pair.
3. **Header-anchored vendor runs.** A primary anchor starts a run at a line that begins
   with, case-insensitively:
   - **Meet:** `Join with Google Meet`, `Join by phone`, `More phone numbers`.
   - **Zoom:** `Join Zoom Meeting`, `Join by SIP`, `Join by H.323`,
     `Join by Skype for Business`.
   - **Teams:** `Microsoft Teams meeting`, `Microsoft Teams Need help?`,
     `Join the meeting now`, `Join on your computer`.
   - **Webex:** `When it's time, join your Webex meeting`, `Join by video system`,
     `More ways to join`.

   A line containing a vendor join URL (`meet.google.com/`, `zoom.us/j/`,
   `*.zoom.us/j/`, `teams.microsoft.com/l/meetup-join`, `teams.live.com/meet`,
   `*.webex.com/`) also counts as a primary anchor. A run lasts until the next blank
   line or fence. Once the description has a primary anchor, **secondary anchors** also
   start runs: `Meeting ID:`, `Passcode:`, `PIN:`, `Phone Conference ID:`,
   `One tap mobile`, `Dial by your location`, `Find your local number`, `Or call in`,
   `Dial in by phone`, `Need help?`, `Meeting options`, and `Access code:`. A vendor run
   that reaches the end of the description without a blank line or fence is an unfenced
   footer, and it runs to the end. Never drop a single substring inside prose
   ([design call 10](#design-calls-beyond-the-research)).

4. **Keep the join link only when it is needed.** When a dropped vendor region held a
   join URL and the event has no `join_link`, emit one bullet: `Join Zoom meeting`,
   `Join Teams meeting`, `Join Webex meeting`, or `Join Google Meet`, linking to that
   URL. Teams invites often have no conference data, so the row has no `📹` badge.
5. **Badge duplicates.** Drop a line whose plain text, or whose only link URL, equals
   the event's `join_link` or `location` (trimmed).
6. **Fixtures** (`boilerplate.json`, with fake IDs, numbers, and passcodes only): one
   each for a Meet block, a Zoom invite (both with and without a 📹 badge), a Teams
   invite (classic and new layouts), and a Webex invite. Also cover a human agenda above
   a vendor wall, which keeps the agenda; "Meeting ID: decide whether to publish" in an
   ordinary agenda, which stays; separators between human paragraphs, which are dropped
   while the paragraphs stay; and a line that only repeats the location, which is
   dropped.

**Acceptance:** the fixtures pass, every pure-boilerplate invite yields zero lines,
`event_buffer/vectors.json` stays green, and `just check` is green.

## Phase: bullet-renderer

**Goal.** Turn an `EventDescription` into the exact bullet bodies sync writes. They must
be deterministic, inert, and readable (A4, A5, A6).

1. **Line bodies.** A `Bullet` line renders as `- {inlines}`. An `Ordered(n)` line
   renders as `{n}. {inlines}`. Bob emits the marker; it is never escaped.
2. **Text nodes.** Escape a node with `escape_note_text` when it starts the body, and
   with `escape_note_inline` otherwise. Render emphasis as `**…**`, `*…*`, or `***…***`,
   with leading and trailing whitespace moved outside the markers. Drop emphasis that
   would be empty.
3. **Links.**
   - Unwrap Google redirects (`https://www.google.com/url?q=<target>&…`) with the `url`
     crate.
   - Allow only `http`, `https`, `mailto`, and `tel`. A link with any other scheme
     renders as its escaped text alone.
   - Destinations are percent-encoded for Markdown: space, `(`, `)`, `<`, `>`, and
     backtick. Extend or share `render.rs`'s `escape_badge_url`.
   - Link text is escaped inline, and its `]` is also escaped.
   - **Beautify:** when the text equals the URL or is empty, an `http(s)` URL of 40
     characters or fewer that has no whitespace, ``<>[]`"``, or `::` stays bare
     (Obsidian autolinks it). A longer one renders as `[host](url)` with any leading
     `www.` removed. `mailto:`/`tel:` render as `[address](mailto:address)`.
4. **Structural guards**, applied in order:
   1. Managed-line guard: if any managed-line predicate claims the rendered line,
      italicize the whole body ([design call 2](#design-calls-beyond-the-research)).
   2. Drawer guard: if the line's plain text passes `capture_drawers::is_drawer_title`,
      render the whole body bold ([design call 1](#design-calls-beyond-the-research)).
5. **Cap and overflow.** When the line count exceeds `cap`, emit `cap - 1` lines and
   then `- *…N more lines in [Google Calendar](html_link)*`, where N counts the hidden
   lines. Without a link, emit `- *…N more lines in Google Calendar*`.
6. **Oversize fallback.** When `oversized` is set, emit exactly one line:
   `- *Description too long to copy — open it in [Google Calendar](html_link)*`.
7. **Hash and materialize.**
   - `bullets_hash` is `journal::span_hash` of the concatenation of
     `"{depth}\t{body}\n"`.
   - `DescriptionBullets::materialize(depth1_indent, unit) -> Vec<String>` applies
     [design call 8](#design-calls-beyond-the-research).
   - Never put a `\n` inside a `WorkingNote.lines` element.
8. **Vectors** (`bullets.json`): map raw input plus context and cap to the exact bodies.
   Cover every example in [The contract Bryan sees](#the-contract-bryan-sees), every
   sanitizer hazard in both directions, redirect unwrapping, long and short bare URLs,
   disallowed schemes, the cap and overflow line with and without `htmlLink`, the
   oversize fallback, and determinism (the same input yields the same bytes).
9. **Differential scanner-isolation gate.** This is the reliability backstop.
   - Build a hostile corpus: `- [ ] x`, `[1] ref`, `#task`, `x ^abc-1`, `%%c%%`,
     `a:: b`, `[[#^x]]`, `![[e]]`, `[^1]`, `[y]: https://x`, `**WORK LOG**`,
     `<b>Work log</b>`, `❌ **CANCEL LOG**`, `DEPENDS ON`, `LINKS`, `AGENDA` with
     children, `TBD`, `🍅 [[#^swings]]`, `` `$= dv.el("b",1)` ``, `<% tp.date.now() %>`,
     `**0900-1000** standup`, `*↻ synced 14:05 · x*`, `#offclock`, `(type:: MEET)`, and
     `Due 📅 2026-11-03`.
   - Render the corpus through the full pipeline, under an Event Task in a realistic
     daily note, with tab and with two-space indentation.
   - Run Bob's real consumers on the note **with and without** the bullets, and assert
     identical results (allowing for line offsets). The consumers are: the capture
     drawer scanner, the managed-log, Depends-On, BLOCKS, relation, and mention
     scanners, `capture_pomodoros::scan`, `gcal::lifecycle::row_untouched`,
     `event_task::parse_event_span` over every line, gcal's `collect_block_ids`, the
     Rust Tasks-format parser used by the tasks parity fixtures, and the task-status
     hooks reconcile. Use pure in-crate entry points where they exist. Where a consumer
     only runs against a vault path, write a temp vault and run it there.
   - If the Tasks-format parser reads `Due 📅 2026-11-03` as a date, switch the renderer
     to render.rs's emoji neutralizer and record why.

**Acceptance:** the vectors and the gate pass, and `just check` is green.

## Phase: birth-bullets

**Goal.** Ship bullets on every Event Task that sync creates (A1, A4, A6, A8, A10),
recorded so that `live-bullets` can take over seamlessly.

1. **Config.**
   - Add `description: bullets | off` (default `bullets`) and `description_lines` (an
     integer, at least 2, default 8) to `src/native/config/gcal.rs` and `GcalConfig`,
     using the `tentative` pattern.
   - A wrong type or value is a setup error (exit 2) that names the key and never breaks
     other config loaders.
   - Add unit tests for the defaults and for each rejection.
2. **Occurrences.** `build_occurrences` sets
   `PlannedOccurrence.description: Option<DescriptionBullets>`:
   - `Some` for task occurrences when the mode is `bullets`, even when the block is
     empty.
   - `None` for context rows and in `off` mode.
3. **Journal.**
   - Add `DescriptionState` to `RowRecord` and `NewRow` (see
     [Journal record](#journal-record)).
   - `append_row` omits the key for `Absent`, writes `null` for `HandsOff`, and writes
     the object for `Tracked`.
   - In `parse_record`, a malformed value parses as `HandsOff`, and the record stays
     valid for `malformed_count`.
4. **Plan, insert path.** A new task row and its materialized bullets go into the note
   in one `WorkingNote::insert_block` call, in the same write as the MEET link. The
   record is `Tracked` with the rendered hash and line count, even when the line count
   is 0. New context rows record `Absent`.
5. **Plan, upgrade path.** `upgrade_context_row` inserts the bullets directly under the
   upgraded row, above any notes Bryan already has there, using design call 8's
   indentation. The record is `Tracked`.
6. **Carry-forward.** Make `PlannedRow.description` and `AdoptedRow.description`
   required.
   - `rows_updated` carries `journaled.description` forward unchanged.
   - Adoptions record `Absent` in this phase. Adoption matching arrives in
     `live-bullets`.
   - Add a test that a title update or root reroute appends a `gcal.row` that keeps the
     description record. Without that, the next phase would backfill twice.
7. **Top-level row matching** ([design call 6](#design-calls-beyond-the-research)).
   Restrict URL and text matching in `plan.rs::find_row_line`,
   `insert_slot`/`row_sort_key`, and `list.rs::row_present`/`adopted_block_id` to lines
   that `split_row_spans` parses. Add tests showing that an overflow bullet carrying the
   event URL, and a bullet starting with `**0900-1000**`, are never taken for rows or
   slots.
8. **Output.**
   - `ReportRow` gains `description_lines`: the number of bullet lines this run wrote
     for the row, 0 otherwise.
   - Human rows append a dim `≡ N` when N > 0.
   - `sync -f json` rows add `description_lines`.
   - `list` computes the count sync would write for Task inclusions in `bullets` mode (0
     otherwise). It shows a dim `≡ N` after the EVENT title, with column widths computed
     before styling the way `buffer_label` does, and `list -f json` events add
     `description_lines`.
9. **Tests** (`tests/gcal_sync.rs`, plus `plan/tests.rs` constructors):
   - Bullets in the past, current, and future phases.
   - A second run is a byte-identical no-op.
   - `-d` shows the bullets.
   - `off` writes none.
   - Tab-indented and space-indented notes.
   - A new row lands after the previous row's bullets.
   - Context rows never get bullets.
   - An upgrade lands the bullets above existing notes.
   - The cap and overflow line.
   - A Teams-only description writes nothing.
   - The motivating `And retainers!` case.
   - Extend the "no descriptions or attendees leak" test: description words appear in
     neither JSON stdout nor `.bob/gcal/*.jsonl`.
   - Update existing fixtures whose descriptions now yield bullets (`events.json`'s
     dentist, the gym case) deliberately, not blindly.
10. **Docs.** Read `docs.md` and `cli_rules.md` with `/sase_memory_read` first.
    - `docs/gcal.md`:
      - Add a **Description bullets** section, with a Contents entry. It covers the
        one-sentence rule, what is copied and what is dropped, the shape (flat first
        children, lead-in nesting, ordered items), inert text with a short examples
        table, the cap and overflow, the oversize fallback, `off`, and the A9 note that
        `Bring shoes #offclock` stays prose.
      - Update the Configuration block, the Sync and List JSON paragraphs, and Known
        limits.
      - Add an ownership table row "Description bullets | Bob, at birth | Written with
        the created or upgraded Event Task; not rewritten afterwards". `live-bullets`
        replaces this row.
    - `docs/reference/configuration.md`: the two keys.
    - `docs/reference/json-outputs.md`: `description_lines`.
    - `docs/reference/vault-format.md`: an Event Task example with one bullet.
    - Run `just docs-gen` if goldens change, and `just docs-check`.

**Acceptance:** all tests pass, `just check` and `just docs-check` are green, and a
`BOB_GCAL_GOG`-driven `sync -d` against a scratch vault shows the motivating output.

## Phase: live-bullets

**Goal.** Make the bullets follow Google until Bryan edits them (A3, A7), using the same
three-way discipline as the spans.

1. **Locator** (`src/native/gcal/bullet_block.rs`, pure, with unit tests):
   - Search only within the row's child block (`child_block_end_line`).
   - Depth is structural: depth 1 means the nearest shallower list-item parent
     (`nearest_shallower_list_item_parent`) is the row, and depth 2 means the parent is
     a depth-1 line.
   - A body is `line.trim_start_matches([' ', '\t']).trim_end()`.
   - A window of `lines` consecutive lines is **the block** when all of these hold: it
     starts at depth 1, every line in it is at depth 1 or 2, the line after it (if
     inside the child block) is at depth 1 or shallower, and its `bullets_hash` equals
     the recorded hash.
   - The first such window wins. A note nested under a Google bullet therefore hands the
     block to Bryan, while sibling notes above or below do not.
   - **Insertion point:** directly after the leading run of managed direct-child
     subtrees: Depends-On, BLOCKS, LINKS / LINKED FROM (`is_managed_link_line`), the
     `❌` CANCEL LOG, and Schedule, Work, or Rotten logs
     (`parse_managed_task_log_marker`). A window of `lines: 0` is always found at the
     insertion point.
2. **`apply_description_update`** runs for journaled task rows whose kind still matches,
   **before** `apply_span_updates`. It edits only lines below the row, so `line_index`
   stays valid, and a later re-slot moves the updated block with the row.
   - Skip it in `off` mode, for `HandsOff`, for context rows, and for rows that went
     Maybe. Treat `Absent` as `Tracked { hash: bullets_hash([]), lines: 0 }`.
   - **Block found, and the fresh hash differs from the recorded one:** replace the
     window in place with the materialized fresh bullets, or remove it when they are
     empty. Insert them when the window is empty. Record the new `Tracked` state, and
     report the row as `updated` with `description_lines`.
   - **Block found, and the fresh hash equals the recorded one:** do nothing.
   - **Backfill guard:** for an `Absent` record with fresh lines, if a window anywhere
     in the child block already matches the fresh hash (a hand copy or a crash), adopt
     it. Record it without writing.
   - **Block not found** (edited, nested under, or deleted): leave everything as it is,
     keep the old record so the block is never re-added, and call
     `push_kept_once(…, "kept your edited description bullets; Google's description changed", "kept_description")`
     only when the fresh hash differs from the recorded one.
   - Merge with `apply_span_updates` so each occurrence produces at most one
     `PlannedRow`, carrying the final span and description state. A description-only
     change must still push one.
3. **Adoption** (crash recovery): if a window matches the fresh render, record it as
   `Tracked`. An empty fresh render always matches. Otherwise record `HandsOff`.
   **Adoption never writes bullets.**
4. **Unchanged behavior to keep and test:**
   - Done, worked, and cancelled rows follow Google while untouched.
   - A cancel prepends the CANCEL LOG as the first child (tab-indented after
     `cancel-log-indent`) and leaves the bullets as history.
   - A time change re-slots the row together with its bullets.
   - Recurring occurrences are independent: nothing is keyed by series.
   - `off` leaves existing bullets alone and carries records forward.
   - An `Absent` record with no description appends no record, so a rerun is a no-op.
5. **Tests** (`tests/gcal_lifecycle.rs` and `bullet_block` unit tests):
   - A Google edit updates the block in place, including on a done row.
   - Google clearing the description removes the block.
   - Bryan's edit is kept silently. A later Google change yields `kept_description` once
     and never again.
   - A deleted block is never re-added. A nested note is kept.
   - Sibling notes and a WORK LOG above or below don't hide the block.
   - A Depends-On line inserted above the block doesn't hide it.
   - A time change re-slots the row with its bullets.
   - An upgrade gains bullets.
   - A cancel keeps the bullets under a correctly nested CANCEL LOG.
   - Crash adoption never duplicates.
   - A pre-feature row backfills once, and its second run is a no-op.
   - A title or root update carries the record forward.
   - A 🍅 in the description doesn't count as worked.
   - Context rows never get bullets and are still removed on decline.
   - A malformed journal value is hands-off.
6. **Docs** (`docs/gcal.md`):
   - Ownership table: replace the birth row with "Description bullets | Google until you
     edit them | Rewritten in place while they still match what Bob last wrote; your
     edit, nested note, or deletion makes them yours, reported once only if Google later
     changes", and rename the other row "Other child bullets (your notes) | You | Never
     touched".
   - Add a bullet to § Reconciling Google-side changes, a backfill note to Description
     bullets, and a Known-limits line about the flat-bullets boundary.
   - Run `just docs-check`.

**Acceptance:** all tests pass, and `just check` and `just docs-check` are green.

## Phase: decision-and-review

**Goal.** Record the policy change and verify the whole feature end to end.

> [!decision] decision_record

1. Following `/sase_memory_write`, add the strand
   `decisions/gcal-description-bullets-follow-google-until-edited.md`. Model its
   frontmatter and sections on `decisions:gcal-root-follows-google-until-edited`:
   Applies to, Claim, Why, Rejected alternatives, Cost, Reopens when, and Evidence.
   Adapt the content from the research report's "Decision record draft" to this plan's
   final design. Evidence cites this plan, `docs/gcal.md` § Description bullets,
   `tests/gcal_lifecycle.rs`, and the research reference. Link
   `[[decisions:gcal-root-follows-google-until-edited]]`,
   `[[decisions:bob-gcal-is-read-only-and-proposes-once]]`, and
   `[[glossary:event-task]]`.

> [!decision] supersede_propose_once

2. Mark `decisions:bob-gcal-is-read-only-and-proposes-once` as `superseded-in-part`.
   Make `superseded_by` a list that keeps the existing root record and adds the new one.
   Add a back-link paragraph stating that the "children" clause is retired for
   description bullets. Do not otherwise edit the accepted body.
3. Run `sase memory init`, then `just docs-gen` to regenerate
   `docs/develop/decisions.md`, then `just docs-check`.
4. **End-to-end review.**
   - Run the full `just check`. Hand it to `/sase_monitor` if it is long.
   - Build a scratch vault with a realistic daily note. Drive `bob gcal sync` through
     `BOB_GCAL_GOG` with the motivating event, the HTML agenda, a Teams invite, a long
     description, and the hostile corpus. Confirm that the note matches
     [The contract Bryan sees](#the-contract-bryan-sees) byte for byte, and include that
     note in the bead notes for Bryan's visual review.
   - Optionally, only if `bob gcal doctor` on this host shows a working login and
     **only** with `-d`, preview today's real note. Summarize what you see; never paste
     real description text into bead notes.
   - Write a traceability table in the bead notes mapping each of A1–A10 to the test
     that proves it.
5. Record anything left over as `PROPOSED FOLLOW-UP:` notes on the phase bead rather
   than creating beads. Examples: revisiting `html5ever` if Outlook's
   `<html><body class=MsoNormal>` input proves common, or "deleting one occurrence's
   bullets teaches the series".

**Acceptance:** memory is republished, the generated docs are current, every gate is
green, and the traceability table is complete.

## Cross-phase rules

- **Release train:** never edit the `Cargo.toml` version or `CHANGELOG.md`. Every CLI
  and JSON change here is additive, so commits use `feat(gcal): …` without `!`.
- **Privacy:** description text never appears in JSON, the journal, logs, `Debug`
  output, or warnings. Only `sync -d` and the vault show it.
- **Determinism:** the same Google event and the same note always produce the same
  bytes, and a rerun is a no-op apart from the Events stamp's per-minute rule.
- **Shared contracts:** `tests/fixtures/event_buffer/vectors.json` and
  `tests/fixtures/task_cancel/vectors.json` never change in this epic.
- **Testing:** run targeted tests while iterating, and `just check` before finishing a
  phase. Docs-touching phases also run `just docs-check`.
