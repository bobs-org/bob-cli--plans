---
tier: epic
title:
  Intentions, summary, and reflection on every day and period, shown in an intention
  card
goal: "Every canonical daily note and every week, quarter, and year note carries the
  same three hidden frontmatter fields: `intentions` (Bryan's list), `summary` (an
  agent's short account of what happened), and `reflection` (Bryan's account of what it
  meant). bob is the only writer. A new intention card in the property-card family
  (album, record, and bank cards) shows the fields at the top of the note, and `==`
  edits them. The daily Headline, week Intentions, quarter Outcomes, and year Theme are
  migrated and retired. The morning ritual gains a PRE row that launches a summary agent
  and a POST row that reflects on yesterday. The agent finds the summary instructions
  through bob, never through note text.

  "
phases:
  - id: fields
    title: Intention fields contract, typed reader, YAML writer, and readers
    depends_on: []
    size: large
    description:
      "fields: add src/native/intentions/ with the day-or-period subject resolver, the
      typed frontmatter reader (intentions, summary, reflection, and stamps) with a
      legacy fallback to the old body shapes, a line-preserving YAML writer that
      verifies by re-parsing, and one review state machine for days and periods. Rewire
      every reader to it: capture-pomodoros, periods show, list, parts, strip, and
      brief. Document the contract in docs/periods.md."
  - id: update
    title: bob periods update guarded envelope and day subjects in show
    depends_on:
      - fields
    size: medium
    description:
      "update: add `bob periods update <SUBJECT> -i FILE|-`, the guarded JSON envelope
      the card editor writes through. It has intention, reflection, and summary ops, a
      field fingerprint as `expect`, dry run, inverse_ops for Undo, the vault lock, and
      atomic writes. Teach `bob periods show` day subjects, update help and docs, and
      run `just docs-gen`."
  - id: summary
    title: Day brief, summary drafting, and brief-carried instructions
    depends_on:
      - fields
    size: large
    description:
      "summary: add the versioned day-summary instruction asset and day brief v1. Every
      `bob periods brief` prints its instructions first. `bob periods draft` writes a
      day's summary and splits a period draft into summary plus Review body. A
      reflection no longer blocks a first draft. Bump the brief to v3 vocabulary, add
      the seven day summaries to the week brief, and ship v3 callouts. Docs."
  - id: repair
    title: Link repair inside the owned keys and problems findings
    depends_on:
      - fields
    size: large
    description:
      "repair: make archive, task move/merge/split, ref rename, and every other vault
      link retargeting rewrite links inside intentions, summary, and reflection on daily
      and period notes exactly as they rewrite body links. Restamp a drafted summary
      digest. Add the dangling_period_link, intentions_shape, intentions_over_soft_cap,
      and legacy_period_text findings to `bob periods list -p`."
  - id: migrate
    title: Migration step and scaffolds without the old sections
    depends_on:
      - fields
      - summary
    size: medium
    description:
      "migrate: add the intentions step to `bob periods migrate`. One dry-run-first,
      idempotent vault_rewrite moves the daily Headline and Reflection, week Intentions,
      quarter Outcomes, year Theme + Outcomes, `## Reflection`, and the Review opener
      into frontmatter. It reports placeholders and conflicts. Scaffolds stop emitting
      the old sections. Fixture-vault goldens mirror the live shapes."
  - id: kit
    title: Property-card kit extracted from the album, record, and bank cards
    depends_on: []
    size: large
    description:
      'kit: in bob-ledger-tools, extract the shared property-card kit with no visible
      change: owned-key property hiding (generalized from the record sheet''s observer),
      a `==` registry replacing the closed album|bank|record list, the host mount
      lifecycle, and shared `.bob-property-card` chrome. Move all three cards onto it.
      Align plugin docs so "card" means the panel and "card editor" means what `==`
      opens.'
  - id: card
    title: Intention card, card editor, commands, and deploy
    depends_on:
      - kit
      - update
    size: large
    description:
      "card: build the intention card on the kit (kicker, numbered intentions, SUMMARY
      with provenance, REFLECTION, recent-closed ghost line, parent intentions line) and
      its `==` card editor (browse keys, one-line intention stage, reflection textarea,
      reorder, remove, Undo, `=` climb, `[`/`]` siblings, `y` YAML). Add palette and
      Command Mode rows, previews, and e2e. Deploy with `bob plugins sync`."
  - id: adopt
    title: Adopt in the vault and chezmoi
    depends_on:
      - migrate
      - repair
      - card
    size: medium
    description:
      "adopt: install the new bob, then dry-run and apply `bob periods migrate` on the
      live vault and refresh callouts. Drop the daily template's Headline line. Replace
      the `#post #headline` GTD row and add the PRE Summarize yesterday and POST Reflect
      on yesterday rows. Set `post_stages`, add types.json entries, and add the day.md
      and vault AGENTS.md pointers. Verify, then record the follow-ups."
  - id: vocabulary
    title: Glossary terms and decision records
    depends_on:
      - card
      - migrate
    size: small
    description:
      "vocabulary: apply the accepted memory decisions. Add glossary strands for
      Intention, Property Card, Intention Card, Album Card, Record Card, and Bank Card.
      Mark Daily Headline superseded, correct Period Note, write the two decision
      records, then run `sase memory init`."
decisions:
  day_summary_cap:
    ask: Should a daily summary allow 1–3 sentences of up to 60 words?
    choices:
      sixty:
        1–3 sentences, at most 60 words, one paragraph, at most 2 links; week tables
        clip it with a tooltip
      one_line:
        One line of at most 25 words, the same cap as week, quarter, and year summaries
    default: sixty
    why: A day has no Review, so its summary carries the whole closeout.
    answer: sixty
  memory_glossary_terms:
    ask:
      Add glossary strands for Intention, Property Card, Intention Card, Album Card,
      Record Card, and Bank Card?
    memory:
      - glossary:intention
      - glossary:property-card
      - glossary:intention-card
      - glossary:album-card
      - glossary:record-card
      - glossary:bank-card
    default: true
    requested:
      We should add excellent new glossary terms for "intention", "property card",
      "intention card", "album card". "record card", and "bank card".
    answer: true
  memory_glossary_fixes:
    ask:
      Retire the Daily Headline glossary strand and correct Period Note for the new
      fields and card?
    memory:
      - glossary:Daily Headline
      - glossary:Period Note
    default: true
    corrects: "- Bryan's `Intentions`, `Outcomes`, or `Theme`;"
    answer: true
  memory_decision_records:
    ask:
      Record decisions that intentions live in bob-written frontmatter and that property
      cards fail open?
    memory:
      - decisions:intentions-live-in-frontmatter
      - decisions:property-cards-own-their-rows
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
proposed_by: bbugyi200.athena.1ef
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 09:35:20
status: wip
---

- **PROMPT:**
  [prompts/202611/period_intentions.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/period_intentions.md)

# Plan: Intentions, summary, and reflection on every day and period

## Context

Today Bob keeps the same idea in four places:

- a `**Headline:**` line in a daily note's preamble;
- `## Intentions` in week notes;
- `## Outcomes` in quarter notes;
- `## Theme` + `## Outcomes` in year notes.

Bryan wants one concept, **intentions**, at every scale ("daily intentions", "weekly
intentions", …). It is stored as hidden frontmatter with two siblings:

- `summary`: what happened, set by an agent;
- `reflection`: what it meant, set by Bryan.

A new **intention card** in the album/record/bank **property-card** family shows the
fields, and `==` edits them. Bryan accepted every requirement recommended in the
consolidated research report. Read it first:

```bash
sase artifact read "research:202611/period_intentions_card/period_intentions_card.md" "Intentions design context"
```

This plan adopts that report's whole recommended design, including its ten adjustments
to the request:

- the card shows when any of the three fields has content;
- the period `## Review` stays in the body;
- there is no sentinel;
- link repair is a requirement;
- no `period:` key is stored;
- the `#post #headline` row is replaced;
- `[n==r` works by composition;
- the year Theme + Outcomes both migrate;
- a reflection never blocks a first draft;
- `all_time` is excluded, with a soft cap of 3, absent meaning empty, and additive Mac
  JSON.

It tightens the design in the places listed under
[Design choices beyond the research](#design-choices-beyond-the-research). Phase workers
treat everything here as decided. Sizes and dependencies are in the frontmatter.

Repositories touched:

- **bob-cli** (this repo): phases `fields`, `update`, `summary`, `repair`, `migrate`,
  `vocabulary`;
- **bob-plugins** (linked; open it with `/sase_repo`): phases `kit`, `card`;
- **the Bob vault `~/bob`** and **chezmoi** (linked; open it with `/sase_repo`): phase
  `adopt`.

Bob Mac Capture only gets additive JSON. Its intentions panel is a follow-up.

## What Bryan sees

The top of yesterday's daily note in Live Preview, after the morning ritual:

```text
# 2026-11-18 Wed
prev · W47 · next

┃ DAILY INTENTIONS
┃  ①  Add ≥30 records to the sase_blog_zettel album
┃  ②  Ship the record walk rail
┃
┃ SUMMARY · ✦ agent · Nov 19
┃ Added 14 zettel records and shipped the record walk rail; athena agent
┃ restarts ate the afternoon.
┃
┃ REFLECTION
┃ │ Tired all day; I started Rocket League far too early (~2 PM).
┃
┃ ↑ W47 · Finish sase_blog_zettel album · sase-1n6 release train · decisions memory
```

- Links (`sase_blog_zettel`) are live links with hover previews.
- The Properties panel no longer lists `intentions`, `summary`, `reflection`, or the
  stamps while the card is mounted.
- A note with none of the three fields shows no card.

A closed period from the last seven days that has intentions but is still waiting shows
one quiet line in place of the missing bands. Open periods and older notes never show
it:

```text
┃ ✦ summary pending · ＋ reflection  r
```

`==` (Vim normal mode, Live Preview) opens the **card editor**:

```text
╭─ DAILY INTENTIONS · Wed Nov 18, 2026 ───────────────── ‹ [   ↑ W47 =   ] › ─╮
│ ▸ ①  Add ≥30 records to the sase_blog_zettel album                           │
│   ②  Ship the record walk rail                                               │
│   ＋ Add an intention                                                     o  │
│   SUMMARY · ✦ agent · Nov 19                                              s  │
│   Added 14 zettel records and shipped the record walk rail; …               │
│   REFLECTION                                                              r  │
│   Tired all day; I started Rocket League far too early (~2 PM).              │
╰─ j/k move · ⏎ edit · o add · ⌥J/K reorder · ^D remove · u undo · y YAML ────╯
```

The morning, start to finish:

- **PRE:** "Summarize yesterday" launches an agent (about 10 s).
- **The commitment walk.**
- **POST:**
  1. "Reflect on yesterday": `[n==r` from today's note, or `==` `[` `r` without leaving
     it.
  2. "Set today's intentions": `==`, then `o`.
  3. Morning review.

Look back, then decide, then plan.

## Vocabulary

- **Intention:** one line Bryan writes, naming a result that would make a period count.
  The period comes from the note's identity: "daily intentions" are the `intentions` of
  a daily note. No `period:` key is stored.
- **Summary:** a short account of what happened. An agent drafts it, only through
  `bob periods brief` followed by `bob periods draft`. Bryan's edit makes it his.
- **Reflection:** Bryan's account of what it meant.
- **Property card:** a panel at the top of a note that shows the frontmatter rows it
  owns and hides exactly those rows while it is cleanly mounted. Its **card editor** is
  what `==` opens. Species:
  - album card (album header);
  - record card (record sheet);
  - bank card (the Setup section of `bank.md`);
  - intention card.
- Use _card_ for the panel and _card editor_ for the `==` modal. Command ids and doc
  URLs stay.

## Design choices beyond the research

1. **Each digest covers exactly the field it names, at every scale.**
   - `summary_digest` and `summary_drafted` live on days and periods alike.
   - Periods keep `review_digest` and `review_drafted` for the `## Review` body only.
   - A period is `drafted` only when every agent part present (summary, Review body)
     matches its digest. So Bryan editing a period summary in the card flips it to
     `edited`. The research did not specify this case.
2. **The update fingerprint covers the owned fields, never the whole file.** Today's
   daily note changes every few minutes (Pomodoro ticks, task edits). A whole-file
   `expect` would keep the editor stale. The fingerprint hashes a canonical JSON of
   `{intentions, summary, summary_digest, summary_drafted, reflection}` as read. The
   writer always applies a line-preserving span edit to the bytes it re-reads under the
   lock, so unrelated body edits are safe.
3. **Every write verifies itself.** After planning the new bytes, the writer re-parses
   the new frontmatter with serde_yaml. Two things must hold:
   - the owned fields equal the intended values;
   - every other key equals its old value.

   Otherwise it refuses with `write_verify_failed` and writes nothing.

4. **Every `bob periods brief` prints its drafting instructions first**, for days and
   periods. Days use a new asset; periods use the same v3 text their callouts carry. The
   folded period callouts stay for now as a second copy, and a follow-up retires them.
   Every pointer (GTD prompt, day.md, vault AGENTS.md, `--help`) leads to the brief.
5. **`r` opens the reflection stage directly.** The editor opens in browse mode, so a
   fast `r` after `==` can never become text. `r` then focuses the reflection textarea.
   `[n==r` and `==[r` both end with the caret in yesterday's reflection.
6. **`[` and `]` step to sibling periods inside the open editor, and `=` climbs to the
   parent.** This makes `==[r` from today's note a race-free way to reflect on
   yesterday. It also makes the separate "Reflect on yesterday" command a convenience
   rather than a necessity.
7. **The "pending" ghost only shows for the last seven days.** It shows on a closed
   period that has intentions and closed within the last seven days. Older notes never
   nag.
8. **bob decides the parent-table line.** `periods show` keeps its `headline` JSON field
   as the single line a parent shows for a child:
   - a closed child with a summary shows the summary;
   - a current or future child with intentions shows them joined with `·`;
   - otherwise the field is empty, and the plugin already falls back to `mostly`.

   New fields are additive: `intentions`, `summary`, `summary_state`, and
   `reflection_lead`.

9. **The display card needs no bob call.** It reads the metadataCache.
   - **Provenance:** the card computes `summary_digest` in JS (Node `crypto`; the card
     is desktop-only) with the documented recipe. It is pinned by golden vectors in
     `docs/periods.md` that both repos test against.
   - **Writes:** only the card editor calls bob, so an outdated bob degrades to "update
     bob to edit" and the card stays visible.
10. **Migration extends `bob periods migrate`** as a new idempotent step, rather than a
    new verb. Migrate already means "bring the vault to the current period contract".

## Data contract (phase `fields`)

The fields live on canonical daily notes (`YYYY/YYYYMMDD.md`) and on week, quarter, and
year notes. They never appear on `all_time`.

```yaml
intentions:
  - "Add ≥30 records to the [[sase_blog_zettel]] album"
  - Ship the record walk rail
summary:
  "Added 14 zettel records and shipped the record walk rail; athena agent restarts ate
  the afternoon."
summary_digest: "sha256:4be1f0c2a91d7e03"
summary_drafted: 2026-11-19
reflection: |-
  Tired all day; I started Rocket League far too early (~2 PM).

  Tomorrow: no games before 6.
```

**`intentions`**

- An ordered list of one-line Markdown strings, trimmed and non-empty, with no newlines.
  `[[links]]` are allowed.
- Soft cap 3: the card hints and the problems report notes it as info. Bryan's text is
  never refused.

**`summary`**

- One paragraph on one YAML line.
- Days: at most 60 words, 1–3 sentences, at most 2 links.
- Periods: at most 25 words, the old Headline cap.
- Never a sentinel or a pointer.

> [!decision] day_summary_cap = one_line Days use the period rule instead: one line of
> at most 25 words. The day draft refusals, the day instructions asset, and its example
> pair all change to match.

**`reflection`**

- Free Markdown; paragraphs allowed.
- Rendered as a scalar when it is one line, and as a `|-` block scalar otherwise. Fall
  back to a double-quoted scalar when block form cannot represent it: a leading-space
  first line, trailing spaces, or tabs.

**Stamps**

- `summary_digest`: `sha256:` plus 16 hex characters, computed with the existing
  `review_digest` recipe over the summary text.
- `summary_drafted`: `YYYY-MM-DD`.
- Periods keep `review_digest` and `review_drafted` for the Review body.

**Absent means empty.**

- Writers delete a key that becomes empty, so no `intentions: []` lingers.
- New keys are appended at the end of the frontmatter in the order: `intentions`,
  `summary`, `summary_digest`, `summary_drafted`, `reflection`. Existing keys keep their
  positions.

**Reading**

- Readers use typed YAML: serde_yaml over the strictly closed frontmatter. They accept
  any valid YAML spelling, because Obsidian may re-serialize.
- A scalar `intentions` string reads as a one-item list and raises an `intentions_shape`
  problem.
- Non-string items are skipped and reported.
- Duplicate keys, or unparsable frontmatter, read as empty with a problem.

**Writing**

- Writes are line-preserving span edits. They keep BOM, CRLF, comments, unrelated keys,
  and byte order.
- Strings are quoted whenever needed. Always quote values containing `[[`, a leading
  YAML indicator, `: `, or ` #`. An unquoted `[[x]]` parses as nested flow sequences.

**Ownership** (extends the `docs/periods.md` ownership contract)

- `intentions` and `reflection` are Bryan's. No tool or agent writes them unless Bryan
  explicitly asks.
- `summary` is written by agents only through `bob periods draft`, which stamps it.
  Bryan may edit it through the card, which leaves it unstamped, so the state is
  `edited`.
- bob is the only YAML writer. The plugin never calls `processFrontMatter`.

**Review states** — one machine for days and periods, first match wins:

| State       | Rule                                                                                                                                                    |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reflected` | `reflection` is non-empty                                                                                                                               |
| `drafted`   | At least one agent part is present, and each present part matches its digest. Days have one part (summary); periods have two (summary and Review body). |
| `edited`    | An agent part is present, but a digest is absent or different                                                                                           |
| `empty`     | None of the above                                                                                                                                       |

**Legacy fallback** (one release; a follow-up removes it). It applies only when the
frontmatter key is absent:

| Field        | Note kind | Legacy source                                                                              |
| ------------ | --------- | ------------------------------------------------------------------------------------------ |
| `intentions` | Day       | the `**Headline:**` / `**Headline**:` preamble paragraph (today's `daily_headline::parse`) |
| `intentions` | Week      | top-level bullets of `## Intentions`                                                       |
| `intentions` | Quarter   | top-level bullets of `## Outcomes`                                                         |
| `intentions` | Year      | `## Theme` (as item 1), then `## Outcomes`                                                 |
| `reflection` | Day       | a `**Reflection:**` / `**Reflection**:` preamble paragraph                                 |
| `reflection` | Period    | the `## Reflection` body                                                                   |
| `summary`    | Period    | the Review's opening `**Headline:**` paragraph                                             |

- While a legacy note is read through the fallback, its state is computed the old way:
  `review_digest` over the whole Review.
- When frontmatter and legacy text are both non-empty and differ, frontmatter wins the
  read and the problems report says `legacy_period_text`.
- Move `daily_headline.rs` into the new module as its legacy reader. The old public name
  may stay as a thin wrapper until the follow-up.

**Subjects**

- `NoteSubject = Day(NaiveDate) | Period(PeriodId)`. It is implemented as a resolver,
  not as a Day variant added to every exhaustive `PeriodId` match.
- Accepted forms:
  - every existing PERIOD form;
  - `YYYY-MM-DD` and `YYYYMMDD`;
  - a daily note path, with or without `.md`;
  - `today`, `yesterday`, and `last-day` (an alias of `yesterday`, mirroring
    `last-week`).
- Relative forms resolve against Bob's `BOB_NOW`-aware today, and every output names the
  canonical id.
- `all_time` is a usage error for field commands.
- A missing daily note is refused (`missing_note`, hint: "open the day in Obsidian;
  Templater creates daily notes"). A missing period note hints `bob periods ensure`.

**Additive JSON**

- `capture-pomodoros --tasks` gains `intentions: [display…]`. `headline` stays, as the
  first intention's display text (the Mac contract is frozen).
- `bob periods show` rows and review objects gain `intentions`, `summary`,
  `summary_state`, and `reflection_lead`. `headline` follows design choice 8.
- `bob periods list` JSON gains `summary`, and its human column reads "Summary".
- The stats strip day/week columns feed `headline` by the same rule.

## `bob periods update` (phase `update`)

```text
bob periods update <SUBJECT> -i/--input FILE|- [-f/--format human|json] [-n/--dry-run]
```

Envelope (parse rules mirror `bob records update`: duplicate keys anywhere refuse;
unknown keys, versions, ops, and wrong types are typed refusals; an empty `ops` list is
a valid no-op):

```json
{
  "schema_version": 1,
  "expect": "sha256:…",
  "ops": [
    { "op": "intentions.set", "values": ["…", "…"] },
    { "op": "intentions.insert", "index": 0, "value": "…" },
    { "op": "intentions.replace", "index": 1, "value": "…" },
    { "op": "intentions.move", "from": 2, "to": 0 },
    { "op": "intentions.remove", "index": 1 },
    { "op": "reflection.set", "value": "…" },
    { "op": "reflection.unset" },
    { "op": "summary.set", "value": "…" },
    { "op": "summary.unset" }
  ]
}
```

**Ops**

- Ops apply in order to an in-memory copy.
- `summary.set` and `summary.unset` remove both summary stamps, so Bryan's summary edit
  reads `edited`.
- Agents never use `summary.*`; they use `draft`.

**`expect`**

- Required to apply. A dry run may omit it: the editor opens with
  `{"schema_version":1,"ops":[]}` and `-n` to read the current fields and fingerprint.

**Values**

- An intention value with a newline is refused (`multiline_intention`).
- An empty value is refused (`empty_value`); use `remove` or `unset`.
- Indexes are checked (`index_out_of_range`).
- A malformed stored `intentions` refuses every intentions op except `intentions.set`
  (`malformed_field`).

**Report** (JSON):
`{ok, schema_version, subject:{id, kind, label, path}, dry_run, fingerprint_before, fingerprint_after, before:{intentions, summary, summary_state, reflection}, after:{…}, changed, inverse_ops, warnings[]}`.

- `inverse_ops` restores `before` exactly. The editor's Undo applies them with
  `expect = fingerprint_after`.
- The human output is a colored before/after preview.

**Writes**

- Wait at most 3 s on `bob_sync.lock` (`lock_busy`).
- Re-read under the lock and compare the field fingerprint with `expect` (`stale_note`).
- Write atomically and never commit; vault-sync commits.

**Exit codes**: 0 ok, 1 I/O, lock, or stale, 2 usage or invalid envelope. JSON errors
use the periods error shape with `op_index` where relevant.

**Membership**: the `bob periods` group becomes "commands about daily and period notes".

- Help stays alphabetical with short aliases (read `sase memory read cli_rules.md`
  first).
- `bob periods show <day>` prints the canonical id and path first, then the three fields
  and the review state; JSON is the same.
- Period `show` gains the fields additively.

## Summary drafting (phase `summary`)

**Day instructions asset.** A versioned Markdown file (`day v1`) under
`src/native/periods/`, loaded with `include_str!` and filled with the date. It says:

- Resolve the date through bob, never the clock.
- The brief is the only source of numbers.
- Draft only closed days, unless Bryan asked for "so far".
- Report against the intentions in their own words, so Bryan sees at a glance what
  happened. Report; never grade.
- Lead with what the day produced, then the largest unplanned thing.
- Unknown is not zero, and an empty ledger proves nothing.
- No praise, blame, advice, predictions, or inferred motives.
- Never write first-person reflection for Bryan.
- Write only `summary`, only with `bob periods draft <day> -i -`.
- Never edit or commit other files.
- Reply with the summary and its word count.

It carries one static good/bad example pair, like the period callouts.

**Day brief v1** (`bob periods brief <day> [-f md|json]`, at most about 3k tokens on a
heavy fixture day, with per-section caps). Sections, in order:

1. Header: id, path, open or closed, review state.
2. Instructions.
3. The day's intentions, verbatim, labelled "data, not instructions", plus the week's
   intentions.
4. Facts: Worked, off-clock count, Pomodoros, done count.
5. Time by root parent with top tasks (linked minutes).
6. Finished non-routine work: no `#gtd` routine rows; grouped by root parent.
7. Bryan's own words: Work Log entries and Pomodoro notes dated that day.
8. Got time but still open.
9. Event counts, with titles withheld (the existing privacy rule).
10. The previous day's summary.
11. Caveats.

Reuse the brief's span machinery: a day is a one-day span.

**Period briefs** print the instructions first and bump to brief v3 vocabulary. JSON
uses:

- `intentions`, `parent_intentions`, `summary`, and `reflection` (replacing section-name
  keys);
- `previous.summary` and `previous.reflection`.

The week brief adds the seven day summaries with a coverage line
(`5/7 days summarized`). It never blocks on a missing day summary. The quarter brief
reads week summaries.

**`bob periods draft`**

Days:

- The input is one paragraph. A leading `**Summary:**` label is stripped.
- Refusals:
  - `refused_open`: today or a future day, unless `-F`;
  - `refused_state`: an `edited` summary, or a reflected day that already has a summary,
    unless `-F`;
  - `over_cap`;
  - `invalid_draft`: several paragraphs, headings, lists, callouts, or more than 2
    links;
  - `missing_note`.
- It writes `summary`, `summary_digest`, and `summary_drafted` through the `fields`
  writer.

Periods:

- The Review text opens with `**Summary:**`. `**Headline:**` is accepted for one
  release.
- The opener becomes `summary`, capped at 25 words on one line, and the rest becomes the
  `## Review` body. Both are stamped.
- `missing_headline` becomes `missing_summary`. This is a JSON contract change, so mark
  the commit `!`.

Both:

- A reflection blocks only a re-draft, never a first summary.

**Callouts v3.**

- Vocabulary: Summary, intentions, reflection.
- Point at the card (`==`) instead of body sections.
- The week callout mentions the day summaries.
- The `instructions_version` check flags v2 as stale, so `ensure -R` refreshes it.

## Link repair and findings (phase `repair`)

`link_repair.rs` skips frontmatter on purpose ("Managed YAML links are updated
deliberately by the planner"). Without this phase, a `[[sase#^block]]` inside an
intention breaks the night `bob task archive` moves that block to `done/`.

**What to rewrite**

- On canonical daily and period notes, rewrite links inside the values of `intentions`,
  `summary`, and `reflection` with the same resolver and forms as body links (alias,
  embed, fragment).
- Do it in every caller that retargets links:
  - archive (`collect_done`);
  - task move, merge, and split (`task_move/transplant.rs`, the merge and split
    planners);
  - `bob ref rename`;
  - any other `vault_rewrite` retargeting. Grep for `link_repair::` and
    `repair_markdown_destination` callers, and list them in the commit message.

**How**

- Re-render each changed value through the `fields` writer, then verify by re-parse.
- Never touch other frontmatter keys.

**Digests**

- Extend `carry_review_digest` so a bob rewrite of a `drafted` summary restamps
  `summary_digest`, just as it does for the Review body. `edited` and `reflected` notes
  are never restamped.

**`bob periods list -p` findings** (human and JSON; they also feed the nightly
report-only step). Scan canonical daily notes in the report range as well as period
notes:

| Finding                    | Meaning                                                            |
| -------------------------- | ------------------------------------------------------------------ |
| `dangling_period_link`     | A link in an owned key does not resolve; names the key and the fix |
| `intentions_shape`         | Not a list of one-line strings                                     |
| `intentions_over_soft_cap` | Info: more than 3                                                  |
| `legacy_period_text`       | An old shape still present, or a conflict with frontmatter         |

## Migration (phase `migrate`)

`bob periods migrate` gains an **intentions step** inside its existing single
`vault_rewrite` transaction.

- It is dry run by default. `-a` applies it with one commit.
- It is idempotent; a second run plans nothing.
- The output gives per-note diffs and an additive JSON `intentions` section.
- Notes outside the `--since` range are untouched, as before.

| Source                                                                    | Destination                      |
| ------------------------------------------------------------------------- | -------------------------------- |
| Daily `**Headline:**` paragraph (Markdown and links kept)                 | `intentions: [text]`             |
| Daily `**Reflection:**` / `**Reflection**:` preamble paragraph            | `reflection`                     |
| Empty `**Headline:**` slot (template leftovers, including future dailies) | removed                          |
| Week `## Intentions`, quarter `## Outcomes` top-level bullets             | `intentions`                     |
| Year `## Theme` then `## Outcomes`                                        | `intentions` (theme first)       |
| `## Reflection` body                                                      | `reflection` (verbatim Markdown) |
| Review opening `**Headline:**` paragraph                                  | `summary`                        |

**Placeholders, nested content, and conflicts**

- Placeholder items are reported, not migrated: `NOT SET`, `TBD`, `TODO`, a lone `-`.
- Nested bullets, or prose that is not a single paragraph, are reported and left in
  place (never lose text). The heading then stays.
- A conflict with an existing non-empty frontmatter value is reported, and both are
  left.

**Removals**

- Emptied `## Intentions`, `## Outcomes`, `## Theme`, and `## Reflection` headings are
  removed.
- The `## Review` heading and the callouts always stay.

**Digests**

| Note state  | What migration writes                                                                                                                              |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `drafted`   | `summary_digest`, `summary_drafted` (copied from `review_drafted`), and a restamped `review_digest` over the remaining body, so it stays `drafted` |
| `edited`    | Nothing; the note stays `edited`                                                                                                                   |
| `reflected` | Nothing; the note stays `reflected`                                                                                                                |

**Scaffolds**

- Scaffolds stop emitting `## Intentions`, `## Outcomes`, `## Theme`, and
  `## Reflection`. Bytes stay a pure function of the id.
- Structure checks (`unscaffolded`) accept both the old and the new anatomy.
- Update the scaffold goldens and the anatomy section of `docs/periods.md`.

**Fixture vault.** Build it from the live shapes, synthetic text only:

- 20261118: Headline and Reflection.
- 20261119: Headline.
- A future daily with an empty slot.
- W46: a `NOT SET` placeholder, plus a reflection with a block link.
- W47: three intentions with block links.
- Q4: three outcomes.
- Year: theme and outcomes, empty and filled.
- Drafted, edited, and reflected Reviews.

## Property-card kit (phase `kit`, bob-plugins)

All three cards live in `plugins/bob-ledger-tools` (fragments joined by
`src/fragments.json`; 1000-line cap per fragment). Read the repo's `AGENTS.md`,
`docs/develop/architecture.md`, and `docs/develop/validation.md`. Extract four things:

1. **Owned-key hiding.** `propertyCardOwnKeys(scope, keys)`, generalized from
   `242ka-plugin-record-sheet-properties.js`'s MutationObserver tagging.
   - It tags only `.metadata-property` rows whose key is owned.
   - Desktop only, and only while the card is cleanly mounted.
   - Never in embeds, hover popovers, or Source mode.
   - Fail-open: unmount and any mount failure untag.
   - The record sheet moves onto it. The album (`record` key; CSS at styles.css ~8524)
     and the bank (`bank` key; ~9850) also move onto it. What Bryan sees on canonical
     notes must be identical.
2. **`==` registry.** `registerPropertyCard({kind, match({path, frontmatter}), open})`
   replaces `schemaDoubleEqualsTargetKind`'s closed list in
   `248m-plugin-schema-double-equals.js`.
   - The order stays bank → album → record.
   - Every refusal in `schemaDoubleEqualsGate` is preserved unchanged.
   - Extend `scripts/test-ledger-tools-schema-double-equals.cjs` with registry cases.
3. **Host lifecycle.** `createPropertyCardHost({...})` wraps the pattern that 241v,
   242k, and 248c each clone:
   - a Reading post-processor batched per path;
   - a `Prec.highest` ViewPlugin widget;
   - the 100 ms refresh effect;
   - `layout-change` / `file-open` remount;
   - the `editorAttributes` facet;
   - the embed rule.

   Each host keeps its own gate, model, DOM, and anchor. Move all three hosts onto it.
   This also relieves the fragments near the cap.

4. **Chrome.** A `.bob-property-card` base:
   - 8px radius, 2px accent start border, `color-mix` accent, theme variables only;
   - a shared `[data-root-hue]` ladder over one `--bob-property-card-accent`;
   - reduced motion respected.

   The three cards keep their own classes and map their accents onto it. Compare preview
   screenshots before and after: they must be pixel-identical.

**Docs vocabulary.** Align `docs/record-card.md`, `docs/albums.md`, and
`docs/bob-ledger-tools.md`:

- the panel is the record, album, or bank **card**;
- the modal is its **card editor**;
- old names (record sheet, album header, Schema Card, Setup) appear once as "formerly"
  notes.

Keep command ids, palette names, and doc file URLs.

**Gate.** `npm test` and the album, record, and bank Obsidian e2e suites pass with no
change to existing assertions.

## Intention card and editor (phase `card`, bob-plugins)

**Gate (display)**

- Canonical daily notes: `dayStripDayForPath`. Week, quarter, and year notes:
  `periodNotesPeriodForPath`, excluding `all_time`.
- At least one of the three fields is non-empty.
- Values are well-formed. Otherwise there is no card and native Properties stay visible.
- Desktop, in Reading or Live Preview. Never embeds, never Source mode.

**Placement.** Below the H1 and its navigation and child-link lines, above the first
other block. That is above the `^gtd` row on dailies and above the strip fence on
periods.

**Display**

- **Kicker:** small-caps `DAILY INTENTIONS` / `WEEKLY INTENTIONS · W47` /
  `QUARTERLY INTENTIONS · Q4` / `YEARLY INTENTIONS · 2026`.
- **Intentions:** numbered keycap chips ①–⑨ in the accent, which double as the editor's
  digit targets. A single intention renders at hero weight. Text goes through
  `MarkdownRenderer`: live links with hover previews.
- **Unresolved links:** struck in `--text-error`, with a tooltip naming
  `bob periods list -p`.
- **SUMMARY band** (only when set): provenance chip `✦ agent · Nov 19` (digest matches)
  or `✎ edited`, then muted text.
- **REFLECTION band** (only when set): a quote rule, with paragraphs kept.
- **Ghost line:** the recent-closed ghost from design choice 7.
- **Parent line:** a muted `↑ W47 · …` line with the parent period's intentions, when
  the parent has any. Click opens the parent.
- **Accessibility and theming:**
  - a labelled `role="group"`;
  - ownership carried by words and glyphs, never color alone;
  - light and dark themes;
  - clean at about 375 px;
  - no hard-coded heights.

**Owned keys** (hidden only while mounted): `intentions`, `summary`, `summary_digest`,
`summary_drafted`, `reflection`, `review_digest`, `review_drafted`.

**Card editor.** `==` acts on any canonical daily, week, quarter, or year note, even
when no card is mounted. It is built on the shared kernel:

- `createGuardedUpdateSession` (242aa) over `bob periods update`, with the
  `expect`/fingerprint flow and `inverse_ops` Undo;
- `bobCardWireCloseKeys` (241xc);
- the `y` YAML excursion (242e) with a new `intentions` kind;
- the arrange engine (241xa) for reorder animation.

Save open editors of the target path before applying (`albumSchemaSaveOpenEditors`).

It opens in **browse mode**, with nothing focused for typing:

- an open period selects the first intention, or the "＋ Add an intention" ghost;
- a closed period selects Reflection.

| Key                         | Action                                                                                                                          |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `j`/`k`, ↓/↑, Tab/Shift+Tab | Move the selection                                                                                                              |
| `1`–`9`                     | Select intention N                                                                                                              |
| `⏎`                         | Edit the selection                                                                                                              |
| `i`                         | Edit the first intention (opens Add when there are none)                                                                        |
| `o` / `O`                   | Add an intention below or above the selection                                                                                   |
| `r`                         | Open the Reflection stage (textarea focused, caret at end)                                                                      |
| `s`                         | Select Summary. `⏎` first explains "agent drafted this; your edit makes it yours"                                               |
| Alt+J / Alt+K               | Reorder the selected intention                                                                                                  |
| Ctrl+D                      | Remove the selected intention (undoable)                                                                                        |
| `u`                         | Undo the last save (inverse ops)                                                                                                |
| `=`                         | Climb to the parent period, in place                                                                                            |
| `[` / `]`                   | Previous or next sibling period or day, in place. A missing daily note shows "no note for Fri Nov 20 yet" with editing disabled |
| `y`                         | YAML excursion                                                                                                                  |
| Esc / `q` / Ctrl+[          | Back out one layer, then close (`decisions:q-and-ctrl-bracket-mirror-esc`; never a hint badge)                                  |

**Stages**

- **Intention stage:** a one-line input. `⏎` saves and Esc cancels. Pasted newlines
  collapse to spaces, with a quiet note.
- **Reflection stage:** an auto-growing textarea.
  - `⏎` saves, Shift+⏎ inserts a newline, and Ctrl/Cmd+⏎ saves and closes.
  - When the note has a summary, it renders read-only above the textarea, so Bryan reads
    what happened while writing what it meant.
- **Cap hint:** at 4 or more intentions, a quiet hint: "Three is plenty — what can go?"
  It never refuses.

**Saves**

- Every save runs the guarded preview, then apply.
- When stale, rebase and keep the draft. Drafts are never lost.
- A short optimistic overlay bridges the metadataCache refresh, as
  `recordSheetPushOverlay` does.
- If `bob periods update` is unknown (an older bob), the editor reads "Update bob to
  edit intentions". The display card is unaffected.

**Entry points**

- Palette: **Edit intentions** (`edit-intentions`, active note) and **Reflect on
  yesterday** (`reflect-on-yesterday`). The latter resolves yesterday with the existing
  `todayDailyPath` logic and opens its editor on the Reflection stage without
  navigating.
- Command Mode rows go in `plugins/bob-command-mode/src/160-sheet-rank.js`
  `CM_SHEET_SURFACE_ROWS`, next to the other card rows.
- Add intentions to the `Ctrl+Shift+P` non-Vim fallback in bob-navigation-hotkeys
  (`551-plugin-album-schema-fallback.js`).

**Validation**

- Unit tests, added to `package.json` `test:unit`:
  - model gate;
  - digest golden vectors from `docs/periods.md`;
  - malformed values;
  - placement anchor;
  - keymap.
- `preview-*.cjs` renders: light, dark, and 375 px; empty-ghost, one-intention hero,
  three intentions plus summary plus reflection, edited provenance. Review the
  screenshots before landing.
- Real-Obsidian CDP e2e (`scripts/obsidian-intention-card-e2e.cjs` plus its offline
  wrapper):
  - mount, hide, and restore on unmount;
  - `[n==r`;
  - `==[r`;
  - add, reorder, remove, and Undo;
  - two-pane stale edit;
  - Source mode and embed fail-open.

New docs page `docs/intention-card.md` (in nav). Then `bob plugins sync`.

## Ritual rows (phase `adopt`)

**Replace** the open successor of the `#post #headline` row in `~/bob/gtd_daily.md` in
place. Edit its text and tags, and keep its dates.

```markdown
- [ ] #task #gtd #post #intentions **Set today's intentions**: `==` on today's note ·
      `o` · one result that would make today count, three at most · `=` in the card
      shows the week's. [repeat:: every day when done]
```

**Add** these two rows:

```markdown
- [ ] #task #gtd #pre #periods **Summarize yesterday**: launch an agent with "Summarize
      yesterday's daily note (`bob periods brief yesterday` prints the instructions and
      evidence)" · done once the agent runs, or when yesterday's summary is already
      drafted or edited. [repeat:: every day when done]
- [ ] #task #gtd #post #periods **Reflect on yesterday**: `[n==r` from today's note (or
      `==` `[` `r`) · read the summary if it's in, then a sentence or two: what did
      yesterday mean, and what changes today? · empty is fine. [repeat:: every day when
      done]
```

**Dates and status**

- Give each new row `[created:: <today>]` and a `[scheduled::]` of the next morning.
- Let bob's status reconciliation settle the derived status
  (`decisions:task-status-is-derived`). Never hand-set `[?]`.

**Rewording**

- Weekly review: "write the Reflection" becomes "`==` then `r` on last week's note";
  "re-read the quarter's Outcomes (↑), and set ≤3 Intentions" becomes "`=` in the card
  shows the quarter's; `==` sets ≤3 intentions".
- Quarterly and yearly reviews: matching wording.
- Year: "set the new year's Theme and ≤3 Outcomes" becomes "set ≤3 intentions (a theme,
  if any, as the first)".
- Weekly PRE row: say it runs after **Summarize yesterday**, because the week brief
  reads Sunday's summary.

**The Bank row.** It asks for "one line in last week's note". Check where those lines
landed in recent week notes. If they live in `## Reflection`, reword the row to "`==`
then `r` on last week's note". Otherwise leave it.

**Config.** In chezmoi `home/dot_config/bob/config.yml`, set
`post_stages: [periods, intentions]`. `pre_stages` is unchanged.

## Phase: fields — contract, reader, writer, readers

**Module.** Create `src/native/intentions/` (registered in `src/native.rs`):

- `subject.rs`: `NoteSubject`, the resolver, path, kind and label, parent subject.
- `fields.rs`: the typed read and problems.
- `legacy.rs`: the old shapes, absorbing `daily_headline.rs` parsing.
- `write.rs`: field edits, render, and verify.
- `state.rs`: digests and the review state.

**Shared YAML helpers.** Extract what the writer needs from
`src/native/records/render.rs` (`collect_key_spans`, `apply_frontmatter_edits`,
`yaml_string_scalar`) into `src/native/yaml_spans.rs`. Add block-sequence and
block-scalar span replacement and key deletion. `records update` behavior and goldens
must not change.

**Review state.** Move the state machine in `periods/note.rs` onto the unified fields.
`carry_review_digest` keeps its contract. The `repair` phase extends it for summaries.

**Rewire every reader** listed under Additive JSON:

- `capture_pomodoros.rs` (~:521);
- `periods/report.rs` (`day_headline_json` ~:1733 and the parts rows);
- `periods/list.rs`;
- `periods/brief.rs`: section reads go through the unified reader, and the brief JSON
  stays v2;
- the stats strip column `headline`.

**Docs.** In `docs/periods.md`, replace "Review state and Headline" with "Intentions,
summary, and reflection": the data contract, ownership, states, legacy fallback, the
digest recipe with three golden vectors, and the parent-line rule. Update
`docs/capture.md` for the `intentions` JSON.

**Tests**

- YAML round-trips: lists, block scalars, quotes, links, CRLF, BOM, comments, duplicate
  keys, Obsidian re-serialized forms.
- Verify-refusal.
- Subject parsing, including timezone, DST, and ISO week 53.
- State goldens for days and periods, legacy and new.
- JSON snapshots.

**Gates.** `just check`, `just docs-gen`, `just docs-check`.

## Phase: update — guarded writer command

- Implement the `bob periods update` contract above:
  - an envelope module mirroring `records/protocol.rs` (reuse `check_duplicate_keys`);
  - op planning on the `fields` writer;
  - the field fingerprint;
  - inverse ops;
  - lock and atomic write.
- Add `bob periods show <day>` and the additive period `show` fields.
- Write clap help with examples, environment, and exit codes, and a docs section in
  `docs/periods.md`. Run `just docs-gen`.
- CLI tests (`tests/cli/`) cover:
  - each op;
  - inverse-ops round trip;
  - `stale_note` when the fields changed, and success when only the body changed
    (simulated Pomodoro edit);
  - `lock_busy`;
  - malformed stored values;
  - the `missing_note` hint;
  - dry run with no `expect`;
  - CRLF and BOM notes.

## Phase: summary — instructions, day brief, draft

- Implement everything under [Summary drafting](#summary-drafting-phase-summary). Size
  the day brief against a heavy fixture day and assert the token budget.
- The callouts move to v3: `periods/instructions.rs`, with the `docs/periods.md` callout
  section.
- `docs/periods.md` documents the day brief, the draft rules for days, the split for
  periods, and the relaxed reflected rule.
- Tests:
  - brief v3 JSON snapshots for a day, week, quarter, and year;
  - the draft refusal matrix, including first-draft-after-reflection allowed and
    re-draft refused;
  - the period split writes summary and Review with both digests;
  - the `**Headline:**` opener is still accepted.

## Phase: repair — owned-key link repair and findings

Implement [Link repair and findings](#link-repair-and-findings-phase-repair).

Tests:

- Archive moves a block linked from a week intention and from a daily reflection: the
  YAML stays valid and the links are retargeted.
- Move, merge, and split goldens.
- Ref rename.
- A drafted summary is restamped; an edited summary is not.
- Each finding is reported.
- Non-owned frontmatter keys are byte-identical.

## Phase: migrate — migration step and scaffolds

- Implement [Migration](#migration-phase-migrate) and the scaffold changes.
- Fixture tests:
  - the dry-run output;
  - apply, then re-run gives a no-op;
  - per-state digest outcomes;
  - placeholders and conflicts reported;
  - nested content left in place;
  - out-of-range notes untouched;
  - new and old scaffolds both pass the structure checks.
- Document the step in the `docs/periods.md` Migration section.

## Phase: kit — property-card kit

Implement [Property-card kit](#property-card-kit-phase-kit-bob-plugins).

- Land the kit in commits that each keep `npm test` green: hiding, registry, host,
  chrome, then docs.
- Run the album, record, and bank e2e suites before and after.
- `bob plugins sync` at the end.

## Phase: card — intention card and editor

Implement
[Intention card and editor](#intention-card-and-editor-phase-card-bob-plugins).

- Use the `bob periods update` built by the `update` phase. Run e2e against a bob built
  from bob-cli master (`BOB_BIN`).
- Finish with `bob plugins sync` and a short visual review of the preview screenshots.

## Phase: adopt — vault and chezmoi

Run on athena, in this order.

1. **Install.** From an up-to-date bob-cli master checkout, run `just install`. Confirm:
   - `bob periods update --help`;
   - `bob periods brief yesterday`;
   - the deployed bob-ledger-tools carries the intention card (`bob plugins sync` status
     / `.bob-deploy.json`).
2. **Prepare the vault.** Follow the vault `AGENTS.md`: inspect `git status`, then run
   `bob vault-sync`.
3. **Dry run.** Run `bob periods migrate -f json` and review every planned note against
   the expected set:
   - dailies 20261118 and 20261119;
   - future dailies with empty slots;
   - the Reviews of W40–W48;
   - W46 (placeholder reported);
   - W47, Q4, and `2026.md`.

   If any other migrate step plans a write, stop and report.

4. **Apply.** Run `bob periods migrate -a`, then `bob periods ensure -R` (callouts v3).
   Verify:
   - `bob periods list -p` has no `legacy_period_text` or `dangling_period_link`;
   - `bob periods show 2026-W47 -f json` and `bob periods show yesterday` read the new
     fields.
5. **Template.** Remove the `**Headline:**` line (and its blank line) from
   `_templates/daily.md`.
6. **GTD rows.** Apply [Ritual rows](#ritual-rows-phase-adopt) to `gtd_daily.md`.
7. **Vault types and pointers.**
   - `.obsidian/types.json` (if tracked): add `"intentions": "multitext"`,
     `"reflection": "text"`, `"summary_digest": "text"`, `"summary_drafted": "date"`.
   - `day.md`: add a short "For agents" paragraph: "To summarize a day or a period, run
     `bob periods brief <id>`. It prints the instructions and the evidence. Save the
     summary only with `bob periods draft <id> -i -`."
   - Vault `AGENTS.md`: one line with the same pointer.
8. **Chezmoi.** Open the chezmoi repo with `/sase_repo`. Set `post_stages`, commit
   there, and `chezmoi apply` the config on athena.
9. **Commit.** Commit only the vault files you changed (`git add <files> && git commit`;
   never `/sase_git_commit` in the vault).
10. **Report.**
    - Name the open `bob.md` tasks this delivers (`^intentions`, `^daily-summary`) for
      Bryan to close. Do not close them.
    - Note any host (apollo, Mac) whose bob predates `fields`: their capture agenda
      `headline` reads empty until they update.
11. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on the phase bead:
    - retire the legacy fallback readers after one release;
    - retire the period `[!agent]` callouts in favor of brief-carried instructions;
    - Bob Mac Capture intentions panel (`bob.md ^current-intentions`);
    - optional per-intention ✓/◐/✗ marks;
    - an optional capture sigil to append an intention;
    - an optional ghost on today's empty note.

## Phase: vocabulary — glossary and decisions

Apply only the memory decisions accepted above, through `/sase_memory_write`, then run
`sase memory init`. Read the shipped docs first, so every claim matches behavior. For
any declined decision, record a `PROPOSED FOLLOW-UP:` note on the phase bead instead.
Guidance for each note:

> [!decision] memory_glossary_terms The six new glossary strands below: Intention,
> Property Card, Intention Card, Album Card, Record Card, and Bank Card.

- **Intention** (`glossary/intention.md`). Aliases: daily, weekly, quarterly, yearly,
  and period intentions.
  - The definition from [Vocabulary](#vocabulary), the hidden `intentions` list, and the
    soft cap.
  - Its siblings `summary` (agent, through `bob periods brief` then `draft`) and
    `reflection` (Bryan).
  - It is edited in the [[intention-card]] editor through `bob periods update`, and bob
    is the only writer.
  - Empty is valid. It never becomes a task, never carries forward, and never nags.
  - It replaces the Daily Headline and the week Intentions, quarter Outcomes, and year
    Theme sections.
  - Not a [[task-type]] (the old year "Theme"), not a period `## Review`, not a task.
- **Property Card** (`glossary/property-card.md`). Alias: card editor.
  - The panel-plus-`==`-editor contract: it shows its owned rows, hides exactly those
    while cleanly mounted, and fails open.
  - The editor writes only through a guarded bob update (preview, fingerprint, Undo).
  - Esc, `q`, and Ctrl+[ back out.
  - Species: album, record, bank, and intention cards.
  - Not the Task, Copy, or Category cards (modals never mounted on a note), and not
    fences such as the [[stats-strip]].
- **Intention Card**:
  - placement and bands;
  - when it mounts;
  - `==` on any daily or period note;
  - the key gestures (`r`, `o`, `=`, `[`/`]`, `[n==r`);
  - never on `all_time`.
- **Album Card**, **Record Card**, **Bank Card**: one short strand each.
  - Album: what it shows; `==` opens the album's schema card editor.
  - Record: formerly the record sheet; the plugin docs' old "Record Card" title now
    names its editor.
  - Bank: the Setup section of `bank.md`. Not the `bob-bank` month or queue fence cards,
    a [[bank-badge]], or a payment card.
    > [!decision] memory_glossary_fixes Retire the Daily Headline strand without
    > rewording it, and correct Period Note.

- **Daily Headline**: set `metadata.status: superseded` and
  `superseded_by: [[intention]]`. Add a back-link sentence naming what was retired. Do
  not otherwise reword it.

- **Period Note**: replace the Intentions/Outcomes/Theme bullet with the three
  frontmatter fields and the intention card. The parts table summarizes each child with
  its summary or intentions. Reflection is no longer a section.

> [!decision] memory_decision_records The two new decision records below.

- **`decisions/intentions-live-in-frontmatter.md`**. Follow the decisions web format:
  applies-to, claim, why with the rejected alternatives, cost, reopens-when.
  - Claim: the three fields live in hidden frontmatter on dailies and week, quarter, and
    year notes. bob is the only writer: Bryan writes through the card editor and
    `bob periods update`; agents write the summary only through `bob periods draft`.
    Summary instructions ride `bob periods brief`, never note text or sentinels.
  - Rejected: body sections; a sentinel in `summary`; a stored `period:` key; the whole
    Review in YAML.
  - Evidence: the research report and this plan.
- **`decisions/property-cards-own-their-rows.md`**.
  - Claim: a property card hides exactly the rows it shows, only while cleanly mounted
    on desktop Reading or Live Preview. Everywhere else, or after any failure, native
    Properties stay. One registry routes `==`.
  - Rejected: Obsidian's all-or-nothing Properties toggle; CSS-only hiding; one
    mega-modal for every card.

## Out of scope

- Mac intentions panel.
- Per-intention outcome marks.
- Retiring the period callouts and the legacy fallback.
- Monthly notes.
- Intentions on `all_time`.
- Any automatic carry-forward or nagging.
