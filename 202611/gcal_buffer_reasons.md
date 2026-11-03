---
tier: tale
title: Buffer reasons for Google Calendar events
goal:
  A Google Calendar description line like `-30 Drive to Kelly's` or `+15 Drive home`
  sets that side's buffer and makes bob gcal sync write one labelled reason bullet that
  clearly reads as the buffer's reason, at the top of the Event Task's Google-owned
  description bullets.
size: medium
decisions:
  memory_buffer_reasons_record:
    ask:
      Write a buffer-reasons decision record and mark event-buffers-widen-the-span
      superseded in part?
    default: true
    memory:
      - decisions:event-buffers-widen-the-span
      - decisions
    corrects:
      Buffers are read from Google description **directive lines** (combinable,
      HTML-safe `@route` / `#untracked` / `-N` / `+M` tokens) and written as one
      Google-owned `[buffer::]` field.
    answer: true
proposed_by: bbugyi200.apollo.c8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 06:20:15
status: wip
---

# Buffer reasons for Google Calendar events

## What Bryan sees

In Google Calendar, Lunch with Kelly (12:00–13:00) has this description:

```text
Bring the gift card
-30 Drive to Kelly's
+15 Drive home
@love #off
```

After `bob gcal sync`, today's note has:

```markdown
- [*] #task **1200-1300** Lunch with Kelly [📅](…) #off [buffer:: -30m +15m] [root::
  love] [type:: MEET] ^lunch-with-kelly
  - ↤ _30m before_ · Drive to Kelly's
  - ↦ _15m after_ · Drive home
  - Bring the gift card
```

Each part of the bullet shows where it came from:

- **Glyph.** `↤` (U+21A4, "←|") reaches back from the meeting's start, and `↦` (U+21A6,
  "|→") reaches forward from its end. They match the hatched lead-in and lead-out wings
  that Bob Ledger Tools paints around the time pill. `⇤`/`⇥` are ruled out because `⇥`
  is already Bob's Tab-key glyph in pickers and footers.
- **Label.** `*30m before*` / `*15m after*` is in italics, Bob's voice (the same style
  as `*↻ synced 14:05 · …*` and `*…N more lines…*`). It uses the words of the existing
  pill tooltip (`Holds 11:30–13:15 · 30m before · 15m after`), and the number matches
  both the `-30` Bryan typed and the row's `[buffer:: -30m +15m]`.
- **Separator.** `·` splits Bob's label from Bryan's own words, as in the Events stamp.
- **Position.** Reasons are always the first description bullets, in timeline order
  (before, then after), so the block reads like the day: leave, meet, come home.

A buffer without a reason (`-30` alone) writes no bullet. The `[buffer::]` field already
says it.

## Design

### Grammar: buffer reason lines

A **buffer reason line** is a description content line (a paragraph or list item, list
marker stripped exactly as for directive lines) that is not a directive line and whose
projected text matches:

```text
<sign><1–3 digits>[m|M] <separator> <reason>
sign      '-' | '−' (U+2212) | '–' (U+2013)  → before;  '+' → after
separator (':' | ',' | whitespace) whitespace* [ ('—' | '–' | '-' | ':') whitespace+ ]
reason    rest of the line, trimmed, non-empty; its first character is a letter
          (char::is_alphabetic) or a non-ASCII character other than a dash/minus
          (U+2010–U+2015, U+2212)
```

- The value comes from `event_task::parse_buffer_token` applied to the token text
  (`<sign><digits>[m]`), so it has the same 720 cap and spelling rules as directive
  tokens.
- `-30 Drive`, `-30m: Drive`, `−30 — Drive`, `+15, Walk back`, `- -30 Drive` (a list
  item), `<li>-30 Drive</li>`, and `-30 🚗 Drive` are reason lines.
- These stay ordinary prose, exactly as today:
  - `+1 555-1212`, `+1 (555) 010-0123 US`, and every Zoom/Teams/Webex dial-in line (the
    reason starts with a digit or `(`).
  - `-30 +15 Drive` (the reason starts with `+`).
  - `-1234 Drive` (four digits) and `-30Drive` (no separator).
  - `-30 "Drive"` (ASCII punctuation start).
- The token and separator must sit in the line's leading `Inline::Text` runs. Bold or
  italic is fine (`<b>-30</b> Drive`). A token inside a link label disqualifies the
  line, which then stays prose.
- A reason line carries exactly one buffer. `@route`, `#off`, and `#on` inside the
  reason are reason text, not directives (`-30 Drive @love` sets no route). Lines made
  only of directives keep today's meaning: `-30 @love` is still a directive line.

### Attaching reasons (one pass, in source order)

Directive lines and reason lines are processed together in source order, so "first value
per side wins" covers both kinds of line:

- Side not set yet: the reason line sets the value and attaches its reason. The line is
  consumed, so it is not copied as a prose bullet.
- Side already set to the **same** value and no reason attached yet: the reason attaches
  and the line is consumed. So `@love #off -30` followed by `-30 Drive to Kelly's`
  works.
- Otherwise (a different value, or a second reason for the side): the line is **not**
  consumed and stays an ordinary description bullet. Nothing Bryan wrote disappears, and
  the missing `↤` label shows that the line did not set the buffer. Example: `-30 Drive`
  then `-10 Park` gives a 30-minute buffer with reason "Drive", plus a plain `-10 Park`
  bullet.

### Rendering and the block

- Reason bullets are depth 1 with marker `-`. The body is `- ↤ *{N}m before* · {reason}`
  or `- ↦ *{N}m after* · {reason}`, where N is that side's minutes.
- The reason goes through the existing inline renderer: links, bold/italic, Google
  redirect unwrapping, `one_line`, and Tasks-emoji neutralization. It uses **inline**
  escaping (`escape_note_inline`, not body-start escaping) because the label leads the
  body. Reason text stays inert: no tags, task links, fields, block IDs, drawers, or
  managed logs.
- Block order: before reason, after reason, then the copied description bullets
  (capped), then the overflow line. In the oversized (>64 KiB) case: reasons, then the
  single fallback bullet. Directives are still parsed when the description is oversized,
  so reasons are too.
- `gcal.description_lines` caps **copied description lines** only. Reasons (at most two)
  never count and are never truncated.

### Ownership, journal, kill switch, privacy

- Reasons belong to the same Google-owned description block and its one `gcal.row`
  description record (`{hash, lines}` covers them). There is no new journal field and no
  second block. The existing three-way update, adoption, `kept_description`, `HandsOff`,
  and Maybe→Yes upgrade paths apply unchanged: Google edits rewrite an untouched block
  in place, and Bryan's edit, nested note, or deletion makes the whole block his.
- `gcal.description: off` writes no bullets, reasons included. The buffer itself still
  applies, because directives are always parsed.
- Context rows get no buffer and no bullets, as today.
- Command output never prints reason text: the dry-run diff stays the only place.
  `description_lines` in JSON and the dim `≡ N` in human `list`/`sync` count reason
  bullets because they are written lines. No JSON schema change.
- `EventDescription`'s `Debug` stays text-free: it prints a reason count, never reason
  text.

### Migration

A row synced before this change whose description already had `-30 Drive …` holds that
line as a plain prose bullet. On the next sync of its day, while the row is untouched,
the row gains `[buffer::]` (the existing Google-owned span update). The untouched block
then rewrites in place to the labelled reason bullet, because the recorded hash still
matches. Existing MEET entries are never retimed, as with any buffer change today.

### Rejected alternatives (for the decision record)

- **A separate tracked reason block with its own journal record.** It means two
  ownership rules, a journal schema change, and two adjacent windows to find. One block
  keeps a single rule: "description bullets follow Google until you edit them".
- **The reason inside the field (`[buffer:: -30m "Drive"]`).** The field is parsed by
  Rust, JS, and Swift, the row gets long, and the reason would be organizer prose on the
  task line.
- **Reasons on the MEET Pomodoro entry.** The ledger records sessions, not event
  logistics, and the link moves between entries by phase.
- **Summing several lines per side.** It contradicts the accepted "first value per side
  wins" rule.
- **`⇤`/`⇥` glyphs.** `⇥` already means Tab across Bob's pickers and footers.
- **Clock-time labels (`↤ *11:30* · …`).** They go stale in a hand-edited block when the
  meeting moves, and they no longer echo the `-30` Bryan typed.

## Implementation

### 1. `src/native/gcal/description.rs` — parse

- Add `BufferReason { minutes: u16, inlines: Vec<Inline> }` and
  `BufferReasons { before: Option<BufferReason>, after: Option<BufferReason> }` with
  `is_empty()`, `count()`, and an accessor that yields `(BufferPart, &BufferReason)` in
  before→after order. Do not derive a text-printing `Debug`: implement it by hand to
  print presence or count only.
- Add `pub buffer_reasons: BufferReasons` to `EventDescription`. Its manual `Debug` adds
  `buffer_reasons: <count>`.
- Add `fn reason_line(line: &SourceLine) -> Option<(BufferPart, u16, Vec<Inline>)>` that
  implements the grammar above. Compute the prefix length on the projected text, check
  that it lies within leading `Inline::Text` runs, drop that many characters from the
  inline list (removing consumed runs, trimming the partial one), and pass the rest
  through `merge_and_trim`. Return `None` when the reason is empty or fails the
  first-character rule.
- Change `directives_from_source` into one source-order pass that returns the
  directives, the `BufferReasons`, and a per-line `consumed` mask. Directive lines keep
  their exact current token handling. Reason lines follow the attach rules above and set
  `before_set`/`after_set` just as a directive token does. `untracked` is computed as
  today.
- `filter_source_lines` drops directive lines **and** consumed reason lines (by index)
  before `boilerplate::filter`, so blank and separator lines are still present for the
  vendor passes, as today.
- Keep the `#[cfg(test)] description_directives` helper working.

### 2. `src/native/gcal/description/bullets.rs` — render

- Add `const BEFORE_GLYPH: char = '↤'` and `const AFTER_GLYPH: char = '↦'`, and
  `fn render_reason(part: BufferPart, reason: &BufferReason) -> Bullet`, which returns
  depth 1 and `format!("- {glyph} *{}m {side}* · {}", reason.minutes, rendered)`.
- Give `render_inlines` a `first_starts_body: bool` parameter (`true` from
  `render_line`, `false` from `render_reason`) so reason text uses inline escaping only.
- `render_bullets`: build the reason bullets first, then today's capped copied lines
  (the cap math is unchanged and applies only to copied lines), then concatenate. In the
  oversized branch, emit the reasons, then the fallback bullet.
- Update the module and function docs. `bullets_hash` and `materialize` stay as they
  are.

### 3. Sync, list, journal — verify only

`sync.rs` (`render_bullets` at the `PlannedOccurrence` build), `list.rs`
(`description_lines`), `plan.rs` (insert/three-way/adopt/upgrade), `bullet_block.rs`,
and `journal.rs` already work on the rendered `Bullet` list. They should need no code
change. Confirm by reading them, and fix any spot that assumes "bullet count == copied
description lines". `model.rs` already takes `buffer` from `directives.buffer`, so
reason-line buffers reach phases, MEET clusters, the TIME label, `[buffer::]`, and JSON
`buffer` automatically.

### 4. Fixtures and unit tests

- `tests/fixtures/gcal_description/outline.json`: add an optional `reasons` key
  (`{"before": {"minutes": 30, "inlines": [...]}, "after": null}`). If the key is
  omitted, there are no reasons, and the test asserts that for **every** case, so
  existing cases prove they create no reasons. Add cases: `reason-plain-both-sides` (the
  "What Bryan sees" input), `reason-separators` (`-30: Drive` / `+15 — Walk back`),
  `reason-en-dash-m-comma` (`–20m, Pack up`), `reason-html-bold-token-and-link`
  (`<b>-30</b> Drive to <a href="https://maps.example/k">Kelly's</a><br>+15 Drive home`),
  `reason-list-item` (`- -30 Drive\n- Bring cake`),
  `reason-after-bare-directive-same-value` (`@love -30\n-30 Drive`),
  `reason-disagreeing-value-stays-prose` (`-30 Drive\n-10 Park`),
  `reason-second-for-side-stays-prose` (`+15 Drive home\n+15 Stretch`),
  `reason-before-bare-directive-wins` (`-10 Park\n@love -30` → before 10, reason Park),
  `reason-emoji-start` (`-30 🚗 Drive`), `reason-routes-and-tags-are-text`
  (`-30 Drive @love #off` → no route, clock none), `reason-cap-720`
  (`-999 Overnight prep` → 720), `not-reason-phones-and-punctuation` (`+1 555-1212`,
  `+1 (555) 010-0123 US`, `-30 +15 Drive`, `-1234 Drive`, `-30Drive`, `-30 "Drive"` → no
  buffer, no reasons, all prose), and `not-reason-token-in-link`
  (`<a href="https://x.example">-30 Drive</a>`).
- `tests/fixtures/gcal_description/bullets.json`: add
  `reason-bullets-lead-in-time-order` (input orders `+15` before `-30`; output is ↤ then
  ↦ then prose), `reason-only` (`-30 Drive\n@love` → one bullet),
  `reason-escapes-inline` (`-30 Pick up #cake at [[Bakery]] a:: b` →
  `- ↤ *30m before* · Pick up \#cake at \[\[Bakery]] a\:\: b`; pin the renderer's actual
  inline escaping), `reason-link`, `reason-cap-exempt` (cap 2 with two reasons and three
  prose lines → both reasons, one prose line, `*…2 more lines in …*`), and
  `reason-disagreeing-line-renders-as-prose` (pin the existing escaped body of
  `-10 Park`).
- `boilerplate_vectors`: assert `buffer_reasons.is_empty()` for every case. Add one fake
  Zoom dial-in wall (`+1 646 555 0100 US (New York)`, `+1 (555) 010-0123,,123#`) whose
  lines must produce no reasons.
- `description.rs` unit tests: `debug_prints_counts_only` gains a reason line with a
  secret phrase that must not appear in `Debug`. `oversized_flag_does_not_stop_parse`
  checks that a reason still attaches.
- `bullets.rs`: an `oversize_input_keeps_reasons_above_the_fallback` test. Extend
  `hostile_bullets_are_isolated_from_bob_scanners` with a loop over hostile reason texts
  (`x #task`, `x [[#^x]]`, `x a:: b`, `x ^abc-1`, `Bring 🍅 [[#^swings]]`,
  `Due 📅 2026-11-03`, `TBD`, `WORK LOG`, `x (type:: MEET)`). Each builds
  `-30 {text}\n+15 {text}`, so both are real reason bullets (first wins per side), and
  runs the same scanner-isolation assertions on the materialized block.
- Leave the shared `tests/fixtures/event_buffer/vectors.json` **unchanged**. It is
  mirrored into bob-plugins, and no existing case starts with a buffer token followed by
  a word.

### 5. Integration tests (`tests/gcal_lifecycle.rs`, `tests/gcal_list.rs`)

Use the existing `GcalEnv`/`FakeGog`/`dentist_with_description` helpers.

- `buffer_reasons_lead_the_description_block`: description
  `Bring insurance card\n-30 Drive to the dentist\n+15 Drive home` → the row has
  `[buffer:: -30m +15m]`, and its first children are exactly
  `\t- ↤ *30m before* · Drive to the dentist`, `\t- ↦ *15m after* · Drive home`,
  `\t- Bring insurance card`. JSON row `description_lines == 3` and
  `buffer == {30, 15}`, and stdout never contains `dentist`. Human output ends the row
  with `≡ 3`. A rerun is `unchanged`. `-d` prints both reason bullets in the diff.
- `google_reason_edit_rewrites_untouched_block_and_keeps_edits`: Google changes the
  reason → in-place rewrite. Bryan edits the `↤` bullet in the vault, then Google
  changes it again → the edit is kept, `kept_description` is reported once, and the
  `[buffer::]` field still follows Google while the row's spans are untouched.
- `plain_bullet_migrates_to_reason_bullet`: sync with `Drive to the dentist`, then
  Google changes it to `-30 Drive to the dentist` → the row gains `[buffer:: -30m]` and
  the block becomes the single reason bullet. No duplicate.
- `description_off_keeps_buffer_without_reason_bullets`: with `gcal.description: off`,
  `-30 Drive` gives `[buffer:: -30m]` and no child bullets.
- `tests/gcal_list.rs`: the same event lists TIME `… −30m +15m` and EVENT `… ≡ 3`, and
  `-f json` carries `description_lines: 3` and no reason text.

### 6. Docs (read `docs.md` memory first; run `just docs-check`)

- `docs/gcal.md`:
  - **§ Buffers:** new `### Buffer reasons` subsection. Cover the grammar in plain
    words, the before/after picture above, the attach rules (first value wins; a reason
    attaches when its line sets or repeats that value; a disagreeing line stays an
    ordinary bullet), and that one reason line carries one buffer (keep `@route`/`#off`
    on their own line).
  - **§ Off-clock events:** at "A line with any other token … is ignored whole", add the
    one exception: a buffer reason line (link to the subsection).
  - **§ Description bullets:** reasons lead the block, never count toward
    `description_lines`, follow the same ownership, and `description: off` turns them
    off too.
  - **§ Configuration:** update the `description_lines` bullet text.
  - **§ List and § Sync:** `≡ N` / `description_lines` include reason bullets.
  - **Ownership table:** "Description bullets (buffer reasons included)".
  - **§ Known limits:** someone else's line such as `+1 to that` reads as a 1-minute
    after-buffer with a reason; the first line per side wins.
- `docs/reference/configuration.md`: `gcal.description_lines` → "Most copied description
  lines per Event Task, overflow line included; buffer reasons are extra."
- `docs/reference/vault-format.md`: in the Event Task shape paragraph, one sentence
  saying that `↤`/`↦` reason bullets lead the description bullets.
- `tests/fixtures/gcal_description/README.md`: document the `reasons` notation and the
  reason-line cases. `tests/fixtures/event_buffer/README.md`: note that reason-line
  cases live in bob-cli-only `outline.json`, so the shared file does not grow.
- Do not edit `CHANGELOG.md` or the `Cargo.toml` version (release-train rule). This is
  additive, with no `!` contract break.

### 7. Memory (decision `memory_buffer_reasons_record`)

> [!decision] memory_buffer_reasons_record Use `/sase_memory_write`. Add
> `sase/memory/decisions/buffer-reasons-ride-description-bullets.md` (keyword "Buffer
> Reasons Ride The Description Bullets", status accepted, decided 2026-11-03). **Applies
> to:** bob-cli and the vault. **Claim:** a `-N reason` / `+M reason` line sets that
> side's buffer like a bare directive and writes one `↤ *Nm before* · …` /
> `↦ *Mm after* · …` bullet at the top of the Google-owned description block; the first
> value per side wins, a reason attaches when its line sets or repeats that value, and a
> disagreeing line stays an ordinary bullet; reasons share the block's ownership,
> journal record, and kill switch, and never count toward the cap. **Why**, **Rejected
> alternatives**, **Cost** (an edited block freezes reasons too; `+1 to that` in someone
> else's invite becomes a reason), **Reopens when** (reasons must survive hand-edited
> blocks, or others' prose trips the grammar), and **Evidence**
> (`plan:202611/gcal_buffer_reasons.md`, `docs/gcal.md` § Buffer reasons,
> `tests/fixtures/gcal_description/outline.json`). Link
> `[[decisions:event-buffers-widen-the-span]]` and
> `[[decisions:bob-gcal-is-read-only-and-proposes-once]]`. Mark
> `event-buffers-widen-the-span` `metadata.status: superseded-in-part`,
> `superseded_by: decisions:buffer-reasons-ride-description-bullets`, and append one
> back-link sentence: "Superseded in part: buffers are also read from reason lines
> (`-30 Drive to Kelly's`), which write a labelled reason bullet. See
> [[decisions:buffer-reasons-ride-description-bullets]]." Then run `sase memory init`
> and `just docs-gen` (which regenerates `docs/develop/decisions.md`).

> [!decision] memory_buffer_reasons_record = no Make no memory edits. File the record
> and the supersession mark as one `memory` task bead through `/sase_new_task`, as the
> memory-write skill's declined-change rule requires.

## Verification

- `just fmt`, `just lint`, `just test` (or `just all`), and `just docs-check`.
- Manual: run `BOB_NOW=… bob gcal sync -d` against a fake-gog fixture whose description
  is the "What Bryan sees" text. Confirm that the diff shows the three bullets in order
  and that `bob gcal list` shows `12:00-13:00 −30m +15m` and `≡ 3`.

## Out of scope

- bob-plugins: showing reasons in the time-pill tooltip
  (`Holds 11:30–13:15 · 30m before: Drive to Kelly's · …`) or styling the bullets
  specially. The bullets read well as plain Markdown, so this would be a later polish.
- Bob Mac Capture and the `capture-pomodoros --tasks` agenda. They read the `[buffer::]`
  field and never the children.
- Unit words (`-30 min Drive` keeps `min Drive` as its reason). The token grammar stays
  the existing one.
