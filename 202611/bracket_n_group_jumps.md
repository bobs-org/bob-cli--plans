---
tier: epic
title: "[N / ]N group jumps for days, periods, records, and any note"
goal: "`[N` / `]N` (and `[{` / `]}` after any `n` walk) open the first note of the
  previous / next group on every canonical daily, week, and quarter note, on records of
  albums that set `walk_by`, and on any note whose nav line carries `«` / `»` links.
  `[n` / `]n` walk a grouped album one group at a time. The property card's docked rail
  and the links walker footer show the same group stops, and a stop whose note does not
  exist yet looks clearly, calmly different from one that will never be created.

  "
decisions:
  daily_create:
    ask:
      Should `]N` / `[N` create a missing daily note from the daily template, the way
      `]n` does?
    default: true
    answer: true
  walk_more_albums:
    ask:
      Should movies and podcasts also walk by status, matching their status-grouped
      album index?
    default: false
    why: Both are short two-group walks; adopt after v1 proves out on three albums
    answer: false
  memory_glossary:
    ask:
      Update the Record Note and Walker Footer glossary strands for grouped walks and
      group stops?
    memory:
      - glossary:record-note
      - glossary:walker-footer
    corrects: linking it to the previous and next records in the album's title order
    default: true
    answer: true
  memory_decision_group_stops:
    ask:
      Add a decisions-web record that group stops are derived on Bob notes and authored
      only on nav lines?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: nav-grammar
    title: Group-link grammar, one resolver, and the [N / ]N commands
    depends_on: []
    size: medium
    description:
      "nav-grammar: in bob-navigation-hotkeys, tighten the `[n`/`]n` matcher so doubled
      arrows never step, recognize `«`/`»` (typed `<<`/`>>`) group links on link-only
      nav lines, build the one resolver (authored, then the Ledger Tools provider), add
      the group commands with the creation guard and repeat guard, route `[{`/`]}` on
      family `n`, publish `walkMode.links` v7, and map `[N`/`]N` in the vault vimrc."
  - id: calendar-groups
    title: Calendar group provider, legacy step fill-in, and the pre-adoption guard
    depends_on: []
    size: medium
    description:
      "calendar-groups: in bob-ledger-tools, add the `api.navGroups` v1 provider for
      daily, week, and quarter group stops plus derived step stops for notes without a
      written nav link, classify every target as existing, creatable, or ghost, and make
      `periodNotes.ensureNote` refuse pre-adoption past years."
  - id: walk-by-cli
    title: Album `walk_by` and group-major record walk lines in bob
    depends_on: []
    size: large
    description:
      "walk-by-cli: in bob-cli, add the album `walk_by` key (non-list choices field),
      order `ring_slots` group-major, rewrite at most five walk lines when a record's
      `walk_by` value changes, report groups and group heads in albums and records JSON
      with a `record_walk_groups_v1` capability, add the `bob albums update` set/unset
      op, strip `«`/`»` from walk labels, extend the shared golden vectors, and document
      it."
  - id: record-groups
    title: Grouped record walk model and the record group provider
    depends_on:
      - walk-by-cli
      - calendar-groups
    size: medium
    description:
      "record-groups: in bob-ledger-tools, mirror bob's group-major order in the record
      walk model from the shared vectors, gate sync requests on bob's capability, and
      extend `api.navGroups` with record group stops (heads of the neighboring occupied
      groups, wrapping)."
  - id: group-rail
    title: The group row at the bottom of property cards
    depends_on:
      - nav-grammar
      - calendar-groups
      - record-groups
    size: large
    description:
      'group-rail: add the kit group-row builder and the shared missing-target look
      (dashed `+` creatable, dotted ghost), adopt it on day, week, and quarter cards and
      the record walk rail, give legacy dailies a derived step row, add the album
      card''s "walks by" pill, and prove the look in the preview matrix.'
  - id: links-footer
    title: Group arms and hints in the links walker footer
    depends_on:
      - nav-grammar
      - calendar-groups
      - record-groups
    size: medium
    description:
      "links-footer: add outer `«`/`»` group arms, the current-group context, the `]}
      next week` and boundary hints, ghost and creatable arm states, end-first shedding,
      and re-validated arm activation through the group commands."
  - id: adopt-and-prove
    title: Album adoption, real-Obsidian proof, and final deploy
    depends_on:
      - group-rail
      - links-footer
    size: medium
    description:
      "adopt-and-prove: verify every host runs plugins and a bob that support grouped
      walks, set `walk_by` on the zettel, subscriptions, and eat albums (plus movies and
      podcasts if accepted), sync them, add a real-Obsidian group-jump e2e, run the
      existing walk, footer, card, and title-block e2e suites, and deploy the final
      tree."
  - id: glossary
    title: Accepted memory updates
    depends_on:
      - adopt-and-prove
    size: small
    description:
      "glossary: apply only the memory decisions answered yes, then run `sase memory
      init`."
proposed_by: bbugyi200.athena.1es
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-21 07:41:32
status: wip
---

- **PROMPT:**
  [prompts/202611/bracket_n_group_jumps.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bracket_n_group_jumps.md)

# `[N` / `]N` group jumps for days, periods, records, and any note

## Why this epic exists

Bryan asked for `[N` / `]N` keymaps once the property-card title block (`bob-cli-o6`) is
done: dailies jump to the first day of the previous / next week, period notes jump to
the first same-level note of the previous / next higher period (never on year notes),
grouped albums walk one group at a time, `[{` / `]}` repeat the jump, the stops render
at the bottom of property cards and in the walker footer, and a stop whose note does not
exist yet is visibly, beautifully different. Bryan also asked for it to be configurable
in any note and asked the planner to lead the design.

Bryan accepted every requirement the consolidated research recommends, except memory
edits, which only the `memory_*` decisions here can authorize. This plan's design
contract makes the calls the research left open. Every phase implements against it.

**Precondition.** This epic builds on `bob-cli-o6` (title block, kit rail, title-slot
mount, record card adoption). Every phase starts with
`sase bead read bob-cli-o6 -r "<why>"`. `nav-grammar`, `calendar-groups`, and
`walk-by-cli` touch nothing o6 is changing and may run while o6 finishes.
`record-groups`, `group-rail`, and `links-footer` must not edit a fragment that an open
o6 phase is changing; if o6 is still open and the work cannot proceed without such an
edit, stop with a bead note `BLOCKED: waiting for bob-cli-o6.<n>` and change nothing.

Read first, through audited interfaces:

- `sase artifact read research:202611/bracket_n_group_jumps/bracket_n_group_jumps.md "<why>"`:
  the consolidated research. Its requirement adjustments 1–9 are accepted requirements;
  this contract refines a few, and wins where they differ.
- `sase artifact read plan:202611/property_card_title_block.md "<why>"`: the title-block
  contract this epic extends (skeleton, rail, missing-note language, fail-open,
  coordination rules).
- Before editing docs or CLI help, `sase memory read docs.md` and
  `sase memory read cli_rules.md`. Before reaching other hosts,
  `sase memory read tailnet.md`.
- bob-plugins phases follow that repo's `AGENTS.md`.

## Design contract

### The one rule

**Lowercase steps one note. Uppercase opens the first note of the previous / next
group.** It completes the grammar `]S`, `]W`, and `]B` already teach.

| Current note                | Its group                       | `[N` opens                                           | `]N` opens                      | Wraps       |
| --------------------------- | ------------------------------- | ---------------------------------------------------- | ------------------------------- | ----------- |
| Daily `YYYY/YYYYMMDD`       | its ISO week                    | Monday of the previous week                          | Monday of the next week         | no          |
| Week `YYYY/YYYY-Www`        | its Bob quarter                 | first week of the previous quarter (W01/W14/W27/W40) | first week of the next quarter  | no          |
| Quarter `YYYY/YYYY-Qn`      | its ISO year                    | Q1 of the previous year                              | Q1 of the next year             | no          |
| Year `YYYY`, `all_time`     | none                            | notice only                                          | notice only                     | —           |
| Record in a `walk_by` album | records sharing the field value | head of the previous occupied group                  | head of the next occupied group | yes, `↺`    |
| Any other note              | as authored                     | the `«` link on its nav line                         | the `»` link on its nav line    | as authored |

Worked examples (today is 2026-11-21; 2025 has no week notes; 2026-W49–W53 do not exist
yet; 2027-W01, 2027-W02, and 2027-Q1 exist):

| From                               | `[N`                       | `]N`                           |
| ---------------------------------- | -------------------------- | ------------------------------ |
| `2026/2026-W30.md`                 | `2026/2026-W14.md`         | `2026/2026-W40.md`             |
| `2026/2026-W47.md`                 | `2026/2026-W27.md`         | `2027/2027-W01.md`             |
| `2026/2026-W01.md`                 | `2025/2025-W40.md` — ghost | `2026/2026-W14.md`             |
| `2026/2026-Q4.md`                  | `2025/2025-Q1.md` — ghost  | `2027/2027-Q1.md`              |
| `2026/20261121.md` (Sat, W47)      | `2026/20261109.md`         | `2026/20261123.md` — creatable |
| `2026/20261231.md` (Thu, 2026-W53) | `2026/20261221.md`         | `2027/20270104.md`             |
| `2025/20251229.md` (Mon, 2026-W01) | `2025/20251222.md` — ghost | `2026/20260105.md`             |

Group jumps are deliberately not mutual inverses (from W30, `[N` lands on W14, and `]N`
from W14 lands on W27). `<C-o>` jump history is the way back. Quarter jumps are
year-scale, and the hints say so (`]} next year`). Daily, week, and quarter paths and
weeks always come from the calendar model (Bob's 13-week quarters, Q4 takes W53, daily
paths use the Gregorian year, week paths the ISO week-year), never string arithmetic.

### Grammar: configuring any note

| Role                  | Canonical label            | Typing aliases       | Recognized                                   |
| --------------------- | -------------------------- | -------------------- | -------------------------------------------- |
| previous step (`[n`)  | `‹ X`, exact `prev`        | single `< X`         | first match anywhere in the body (unchanged) |
| next step (`]n`)      | `X ›`, `› X`, exact `next` | single `X >` / `> X` | first match anywhere in the body (unchanged) |
| previous group (`[N`) | `« X`                      | `<< X`               | only on a nav line                           |
| next group (`]N`)     | `X »`, `» X`               | `X >>`, `>> X`       | only on a nav line                           |

- **Doubles are checked first and never step.** The current prev matcher (`/^[‹<]/`)
  would let `<< W14` hijack `[n`. Tighten it: prev is exact `prev`, `^‹`, or `^<(?!<)`.
  Next is exact `next`, `^›`, `›$`, `^>(?!>)`, or `(?<!>)>$`. A label that starts with
  `«` / `<<`, or starts or ends with `»` / `>>`, is a group role only.
- **A nav line** is a body line outside frontmatter and code fences that holds at least
  one link and nothing else but whitespace and the separators `·` `•` `|` `/` `,` `–`
  `—` `-`. The period nav line, the daily `prev · W47 · next` line, the record walk
  line, and a hand-written `[[part_2|« Part 2]] · [[part_4|Part 4 »]]` all qualify.
  Prose such as `Read more »` in a clipped article never does. Both wikilinks and
  Markdown links count.
- **Conflicts refuse.** Two distinct targets in one group role across the note's nav
  lines refuse with `Two « links on nav lines — keep one` (or `»`). Identical duplicates
  deduplicate.
- **`«` / `»` pair with `‹` / `›`.** A single arrow steps; a double arrow leaps. No
  other spellings.

### One resolver, four surfaces

The keys, `[{` / `]}`, the card rail, and the footer all show and execute one resolved
answer. Bob Navigation Hotkeys owns the grammar, the commands, opening and creating, and
jump history. Bob Ledger Tools owns the derived providers, because it owns the calendar
model (`296a`) and the record walk model (`242t`). Providers read the metadata cache,
vault file existence, and the cached album report only. They are synchronous, never
spawn bob, and never write.

**Per-role precedence:** the authored nav-line link for that role, then the provider's
derived stop for that role, then nothing. Written `‹` / `›` step links always win over
derived step stops.

**Ledger `api.navGroups` v1** (an additive namespace; the top-level api stays v3; never
throws):

```text
resolve({ path, content }) → { ok: true, route, kind, step, group, prevGroup, nextGroup, none }
                           | { ok: false, reason: "invalid-input" | "unavailable" }
  route      "calendar" | "record" | null
  kind       "day" | "week" | "quarter" | "year" | "all" | "record" | null
  step       { prev, next, parent }  — calendar notes only, each a Stop or null
  group      { noun, label, shortLabel, position, size, index, count,
               members: [{ label, path, state: "current"|"exists"|"missing" }] | null,
               atStart, atEnd, enteringNext, enteringPrev }  — or null
  prevGroup  Stop | null
  nextGroup  Stop | null
  none       { reason, message } | null  — why there is no group route
Stop = { path, target, groupLabel, landingLabel, shortLabel, state, ghostReason, wraps, source }
  state      "exists" | "creatable" | "ghost"
  target     link path without `.md`
  source     "calendar" | "record"
```

`members` lists the current group (7 days, 13 or 14 weeks, 4 quarters, or the record
group). It is null for a record group longer than 21. `enteringNext` / `enteringPrev`
name the group a single `]n` / `[n` enters when the note sits at a group boundary.

**Nav `api.walkMode` v7** extends `links({ path, content })` additively. `prev` and
`next` gain `source` (`authored` | `calendar`), `state`, and `ghostReason`. New members:
`prevGroup`, `nextGroup` (Stops with `source` `authored` | `calendar` | `record` and a
rendered `text`), `group`, `groupNone`, and `groupConflict` (`prev` | `next` | null).
Consumers feature-detect each member. Without Ledger Tools (or an older one), authored
links still work and calendar notes report `groupNone` with
`Day and period groups need Bob Ledger Tools`.

### Missing targets

One visual rule everywhere: **dashed means "not created yet"; dotted and faint means
"won't be created from here."** Never error red and never Obsidian's unresolved-link
styling, because a missing future week is expected, not broken.

| State       | When                                           | Look                                        | Activation                                                                                                                                                                                  |
| ----------- | ---------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exists`    | the file is in the vault                       | normal arm                                  | opens, with jump history                                                                                                                                                                    |
| `creatable` | missing, and the existing writer may create it | dashed outline, muted ink, a small `+`      | an explicit `[N` / `]N`, rail click, or footer click creates it through the existing writer (Templater daily template for dailies, `api.periodNotes.ensureNote` for periods), then opens it |
| `ghost`     | missing, and navigation must not create it     | dotted outline, faint ink, `· not in vault` | a notice; nothing is written                                                                                                                                                                |

- **Pre-adoption ghosts.** A target is pre-adoption when its ISO week-year has no week
  note in the vault and the target ends before today. Pre-adoption targets are always
  ghosts. This guards a verified hazard: `bob periods ensure` scaffolds the whole week →
  quarter → year chain, and `bob periods` problems treats any year with a week note as
  adopted, so one `[N` from 2026-W01 would otherwise create 2025-W40, 2025-Q4, and
  `2025.md` content and flood problems with every Zorg-era 2025 daily. As a backstop,
  `periodNotes.ensureNote` itself refuses pre-adoption past targets
  (`code: "pre-adoption"`), which also stops `[n` from 2026-W01 scaffolding 2025.
- **Album records** never get created by navigation. A missing record target keeps the
  existing album-folder ghost guard and `⟳ syncing`.
- **Repeat keys never create.** `[{` / `]}` open existing targets only. On a creatable
  target they show `Mon Nov 23 isn't created yet — press ]N to create it` and write
  nothing, so holding `]}` can never chain-create notes. This honors the documented
  contract that `]]` / `[[` / `]}` / `[{` never write notes.
- Existence is resolved live and repaints on vault create, delete, and rename, plus the
  midnight rollover. Nothing about existence is stored.

> [!decision] daily_create Missing daily targets in adopted years and the future are
> `creatable`: `]N` from Saturday creates next Monday from the daily template, exactly
> as `]n` creates a next-day link target today.

> [!decision] daily_create = no Missing daily targets are `ghost` with the reason
> `Group jumps don't create daily notes`. Their tooltip and notice point at the
> daily-notes flow (`-` for today). Missing period targets keep the `creatable` rule.

### Keys and repeat

- New commands `open-prev-group-link` ("Open previous group link ([N)") and
  `open-next-group-link` ("Open next group link (]N)"). The vault vimrc maps
  `nnoremap [N` / `]N` beside `[n` / `]n` through `exmap`.
- Both commands record family `n`. `BRACKET_REPEAT_UPPERCASE_PLANS.n` routes `[{` / `]}`
  to the same method with `{ viaRepeat: true }`. After `]N`, `]]` still steps one note
  and `]}` still leaps one group.
- Counts are ignored (`N]N` and `N]}` on family `n` jump once), as `]n` ignores them
  today. Document this beside the other families' count parity.
- No-route notices are specific and truthful:
  `Years have no group jump — ]n steps years`, `All time has no group jump`,
  `SASE Blog Zettel has one section group — ]n walks it`,
  `Movies walks by title — set walk_by on the album to group it`,
  `No « link on a nav line here`. The dead `No uppercase link walk` notice is removed.
  The uppercase-repeat command titles become `(]S, ]W, ]N, or ]B)`, and the `547`
  indicator tooltip names `]N` for `n`.
- **Legacy step fill-in.** On canonical daily and period notes with no written prev /
  next link, `[n` / `]n` use the calendar provider's derived step stops (same states and
  guards). Of about 1,196 dailies, only 176 carry a nav line, so this revives `[n` /
  `]n` on the rest. Written links always win.

### Records and `walk_by`

```yaml
# sase_blog_zettel.md: top level, beside index_by / index_facets
walk_by: section
```

- A new key, separate from `index_by`. Unset means today's flat title ring and zero
  churn. The album index and the walk stay independent: the zettel index still groups by
  `status` while its walk follows `section`.
- `walk_by` must name a declared, non-list choices field
  (`walk_by names undeclared field "x"`,
  `walk_by must name a single-value choices field`). An invalid value falls back to the
  flat title ring and reports an issue; it never damages a line.
- **Order:** groups in the field's declared choice order (from bob's report
  `fields[].choices[]`, never JS object key order), title order within a group, empty
  groups skipped, and records with a missing or invalid value in one trailing
  `⚠️ Needs <field>` group (reuse the index's needs-group label rule). The ring wraps
  from the last record of the last group to the first record of the first.
- bob writes the existing three-link walk lines in that order, so `[n` / `]n` walk the
  group, then the next group, through the line machinery that already exists. The 3-link
  cap stays. Group stops are derived, never written.
- **Churn:** setting or unsetting `walk_by`, or changing that field's choice order,
  rewrites the album's lines once, like a retitle. A record's `walk_by` value edit
  through `bob records update` (and so the Record Card) rewrites at most five lines (old
  neighbors, new neighbors, self) in the same guarded transaction. That edit rewrote
  zero before.
- **Adoption** (`adopt-and-prove`): `sase_blog_zettel: walk_by: section` (requested; 8
  occupied sections across 12 records, in blog-outline order),
  `subscriptions: walk_by: status` (to_decide 9, active 13, cancelled 6: `]n` walks the
  whole triage queue in one run), and `eat: walk_by: status` (to_try 12, liked 86,
  not_liked 7: the 12 places to try become one contiguous walk; `location` is a text
  field and is not eligible in v1). `tv` stays unset: all 23 records are `to_watch`, one
  group.

> [!decision] walk_more_albums `movies` and `podcasts` also get `walk_by: status`
> (`to_watch` 10 / `watched` 2; `subscribed` 9 / `unsubscribed` 6), so their walks
> follow the status groups their album index already shows.

### The group row at the bottom of property cards

The kit rail gains a quieter second row, shown only when a group route exists. Year
notes, `all_time`, ungrouped albums, and single-group albums keep exactly today's rail.
Rows, top to bottom: the step row (unchanged), the group row, then the period children
strip.

```text
‹ W46                       ↑ 2026-Q4                        W48 + ›
« Q3 · W27          ▮▮▮▮▮▮▮■▯▯▯▯▯▯  week 8 of 14      2027-Q1 · W01 »
[Mon 16] [Tue 17] … [Sun 22]
```

```text
‹ Bead Lifecycle            ↑ 🗂 SASE Blog Zettel · 8 of 12        SASE is Too… ›
« 🛑 Why SASE is still NOT ready? · AI Slop   🔮 Future Work · 2 of 3 ▮■▮   🏁 Conclusion · … » ↺
```

- **Arms.** `«` / `»` glyphs are `aria-hidden`. Each arm reads group label first and
  landing second (`« Q3 · W27`), in UI-small type one step below the step row: the group
  label in medium weight with `--text-muted`, the landing in `--text-faint`. Labels
  truncate with an ellipsis, never wrap, and show in full in the tooltip.
- **Center.** Calendar cards show the position (`day 6 of 7`, `week 8 of 14`,
  `quarter 4 of 4`) beside a tick track, one 3px rounded bar per member: accent fill for
  this note, muted fill for an existing member, and a hollow outline for a missing one.
  Record cards show the current group's choice label, `· 2 of 3`, and the track. A group
  larger than 21 shows a continuous bar with a thumb instead of ticks. The track is
  `aria-hidden`; the position text carries the meaning. While grouped, the record rail's
  whole-walk thumb track is replaced by this row; the overall `8 of 12` stays in the
  step row.
- **States** use the missing-target rule above, and the **step row adopts the same
  look** for its missing items (today's next-day arm on a daily becomes the dashed `+`
  form), so one language covers both rows, the children chips, and the footer. A
  wrapping record arm shows `↺`. Tooltips name the key and the destination
  (`]N · next quarter: 2027-Q1 → 2027-W01`,
  `2027-W01 isn't created yet — ]N creates it with bob periods ensure`,
  `2025-Q1 is before Bob's period notes began — navigation never creates it`).
- **Activation.** A plain click on a group arm runs `open-prev-group-link` /
  `open-next-group-link` in the card's own pane after re-validating that the resolver
  still returns the same stop, exactly as rail prev / next run `open-prev-link` /
  `open-next-link` today. A modified click opens an existing target like a Markdown
  link. A ghost arm is `aria-disabled="true"`, keeps focusability for its tooltip, and
  only shows the notice. Without Navigation Hotkeys, the card calls `api.navGroups`
  directly (derived stops only) and clicks open existing targets.
- **Legacy dailies get a step row.** When a canonical daily or period note has no
  recognized nav line and `walkMode.links` reports `source: "calendar"` for prev / next,
  the card renders a derived step row from that answer, so the rail and `[n` / `]n`
  always agree and nothing duplicates a visible nav line.
- **Narrow** (container below 28rem): the group row collapses to `« Q3` · `8/14` ·
  `2027-Q1 »` with no landing labels and no track, and targets stay at least 32px wide.
- **Accessibility.** Accessible names spell the role and state:
  `Previous quarter group: Q3, opens 2026-W27`,
  `Next week group: W48, not created yet — activating creates it`, and
  `Previous year group: 2025, not in vault`. Contrast is at least 4.5:1 for text and 3:1
  for borders, in light and dark. Under `forced-colors: active`, dashed and dotted
  borders use `CanvasText`, ghost text uses `GrayText`, and the current tick uses
  `Highlight`, so no state depends on color. `prefers-reduced-motion` disables
  transitions.
- **Album card.** When `walk_by` is set and valid, the folded schema line gains
  `· walks by Section` (the field label), with the tooltip
  `Record walk: grouped by Section, then title`. It is display-only. Setting `walk_by`
  is the bob op or a YAML edit.

### The links walker footer

The `n` panel finally uses its empty slots, from the same `walkMode.links` answer:

| Note           | Due          | Context              | Arms                                               | Hint                                     |
| -------------- | ------------ | -------------------- | -------------------------------------------------- | ---------------------------------------- |
| Daily Sat, W47 | `Links`      | `W47 6/7`            | `« W46` `‹ Fri` · `Sun+ ›` `W48+ »`                | `]} next week`                           |
| Week W30       | `Links`      | `Q3 4/13`            | `« Q2` `‹ W29` · `W31 ›` `Q4 »`                    | `]} next quarter`                        |
| Quarter Q4     | `Links`      | `2026 4/4`           | `« 2025` (ghost) `‹ Q3` · `2027-Q1 ›` `2027 »`     | `]} next year`                           |
| Year           | `Links`      | —                    | `‹ 2025` · `2027 ›`                                | none; tooltip `Years have no group jump` |
| Grouped record | `Links 8/12` | `🔮 Future Work 2/3` | `« 🛑 Why SASE…` `‹ …` · `… ›` `🏁 Conclusion » ↺` | `]} next section`                        |
| Authored note  | `Links`      | —                    | step arms plus authored group arms                 | `]} next group`                          |

- Group arms sit outside the step arms, `« ‹ · › »`, so the farther leap is visually
  farther out. They reuse `bob-freshness-link-create` for creatable and gain
  `bob-freshness-link-ghost` for ghosts.
- **Boundary hint.** On the last member of a group the hint turns predictive:
  `]n enters W48`, `]n enters 🏁 Conclusion`, or `]n wraps to <first title>`. On the
  first member, `[n` gets the mirrored hint in the tooltip.
- **Shedding** (end-first as the bar narrows): meter, then hint, then context, then the
  group arms' text shortening to the glyph plus the group short label, then the group
  arms, and only then the step arms.
- **Activation** re-validates the group stop against a fresh `walkMode.links` answer
  (the `linksWalkerFooterSameLink` pattern) before running the group command. A note
  with only `«` / `»` links still paints a useful panel. The record tooltip's
  `alphabetical, wraps around` becomes `by <field>, then title · wraps around` for
  grouped albums.

## Phases

### `nav-grammar`: group-link grammar, one resolver, and the commands

bob-plugins, `plugins/bob-navigation-hotkeys`.

- **Matcher.** Rewrite `renderedLinkMatchesNavLabel` (`680-plugin-link-parse.js`) to the
  four disjoint roles in the grammar table, with doubles checked first. Before changing
  it, scan the live vault read-only for link labels that begin or end with `<<`, `>>`,
  `«`, or `»` and record the count in the phase bead note; nothing should regress.
- **Nav lines and authored stops.** Add a pure fragment (for example
  `680a-plugin-nav-line-roles.js`) with `navLineIsLinkOnly(line)`,
  `navLineGroupLinks(content)` (frontmatter and fences skipped, wikilinks and Markdown
  links), and conflict detection with deduplication.
- **Resolver.** One internal `resolveNavStops({ path, content })` composes authored
  stops with `ledger.api.navGroups.resolve` (feature-detected by `version >= 1`) under
  per-role precedence. It fills derived step stops only when no written link of that
  role exists. `walkMode.links` v7 returns its result; existing members keep their
  shapes, and `createEligible` / `exists` stay for v2 consumers.
- **Commands.** `openGroupLink(direction, { viaRepeat })` resolves live, refuses
  conflicts and no-route cases with the contract's notices, and opens `exists` stops
  through the existing open path, with jump history and leaf reuse. On `creatable`
  stops, an explicit press creates through `toLineLinkCandidate` and
  `openOrCreateLinkCandidate` (Templater for dailies, `openPeriodNoteThroughBob` for
  periods); a repeat shows the press-`]N` notice. On `ghost` stops, it shows the ghost
  notice. `openLabeledBodyLink` falls back to derived step stops the same way.
- `openPeriodNoteThroughBob` surfaces `code: "pre-adoption"` as its own notice.
- Register the two commands, set `BRACKET_REPEAT_UPPERCASE_PLANS.n`, update the repeat
  command titles, remove the no-uppercase notice, and update the `547` tooltip.
- **Vault vimrc** (`obsidian_vimrc.md` in the vault, git-synced): add
  `exmap bob_prev_group_link …` / `bob_next_group_link …` and `nnoremap [N` / `]N`
  beside `[n` / `]n`, and extend the `]}` / `[{` comment to name `]N` / `[N`. Commit it
  through the vault's normal flow.
- **Tests.** Cover matcher vectors (`<< X`, `X >>`, `» X`, `‹ X`, `X ›`, `< X`, `X >`,
  exact `prev` / `next`, and prose `Read more »`), nav-line acceptance and rejection,
  conflict and dedup, precedence (authored beats derived per role; written step beats
  derived step), a fake provider with all three states, the repeat guard, family
  recording, counts ignored, and `walkMode` v7 shape plus v2 compatibility. Update the
  tests that pin the walkMode version.
- **Docs.** In `docs/bob-navigation-hotkeys.md`: the grammar, the commands table, the
  uppercase-repeat table row for `n`, count behavior, the never-create repeat rule, and
  the pre-adoption refusal. Run `npm run build`, `npm test`, and `npm run docs:check`,
  bump the manifest, commit, and run `bob plugins sync`.

### `calendar-groups`: calendar provider, step fill-in, and the pre-adoption guard

bob-plugins, `plugins/bob-ledger-tools`. Parallel with `nav-grammar`; the API contract
above is the seam.

- Add a provider fragment (for example `296e-plugin-nav-groups-calendar.js`) built on
  `intentionCardSubjectForPath` and its day, week, quarter, and year infos: the group,
  its members, both group stops, the derived step stops (previous, next, parent), and
  `none` for year and `all_time`. Labels follow the contract (`W46` / `Mon Nov 9`, `Q3`
  / `W27`, `2027-Q1` / `W01` across years, `2025` / `Q1`).
- **States.** `exists` from `vault.getAbstractFileByPath`. Pre-adoption uses a cached
  set of ISO years that have at least one `YYYY/YYYY-Www.md`, invalidated on vault
  create, delete, and rename of matching paths, and today from the same local-day helper
  the day card uses. Apply the `daily_create` answer to missing dailies.
- Add `api.navGroups` v1 (`296f-plugin-nav-groups-api.js` or similar) with `resolve`
  dispatching on route; records return `route: null` until `record-groups` lands.
  Register it on `this.api` beside `periodNotes`. It never throws.
- Make `periodNotesEnsureForPath` refuse pre-adoption past targets with
  `{ ok: false, code: "pre-adoption", message }` unless
  `options.allowPreAdoption === true`, and bump `api.periodNotes` to v2. The daily
  `bob periods ensure` run and "Open this week's note" are unaffected (current periods
  are never pre-adoption).
- **Tests.** Cover every worked example in the contract, W53 years, W01 Mondays filed in
  the previous Gregorian year, 14-week Q4 members, 1970 and 2999 bounds (null, never a
  throw), all three states, cache invalidation, the ensure refusal, and the provider on
  non-calendar paths.
- Document the namespace in `docs/develop/` where the other api namespaces live (respect
  word budgets). Run `npm run build`, `npm test`, and `npm run docs:check`, bump,
  commit, and run `bob plugins sync`.

### `walk-by-cli`: album `walk_by` and group-major walk lines

bob-cli. Read `cli_rules.md` and `docs.md` first.

- **Declaration.** Add `walk_by: Option<String>` to `AlbumDeclaration`
  (`src/native/albums/mod.rs`), parsed and validated beside `index_by` with the
  contract's messages and its own issue codes. Invalid means the effective value is
  `None`.
- **Order.** Make `ring_slots` (`src/native/albums/walk.rs`) take the effective grouping
  and order group-major: declared choice order, `title_order` within a group, and a
  trailing needs group. Reuse `grouping.rs`'s `group_value` / `group_records` bucketing
  instead of a second implementation. `plan_album_walk` and `WalkScope::Radius` then
  stage exactly the lines whose canonical text changed.
- **Value edits.** In `plan_album_write_for_record` (`src/native/records/cli.rs`), plan
  `WalkScope::Radius { created: {} }` when the update changes the record's `walk_by`
  field value, for both `execute_update` and `execute_update_envelope`, and report
  `walk_files` as the create path does.
- **Labels.** `sanitize_walk_label` drops `«` and `»` everywhere, so a title can never
  pose as a group link. The existing `‹` / `<` / `›` / `>` trimming stays.
- **Reports.** The albums entry gains top-level `walk_by` (effective key or null) and
  `walk.groups[]` (`{ value, label, count, head: { stem, path, title } }` in walk order,
  with the needs group's `value` null). `capabilities` gains `record_walk_groups_v1`.
  `bob records show` `walk` gains `group`
  (`{ field, value, label, index, count, position, size }`) plus `prev_group` /
  `next_group` heads and their wrap flags. Additive JSON: no `!`.
- **Update op.** Add a `bob albums update` op that sets or unsets top-level `walk_by`,
  validated like the declaration, with restore-path whitelisting. Any op that changes
  `walk_by` or the walk field's choices replans the walk in the same transaction.
  `field.remove` refuses a field named by `walk_by`, mirroring the `index_by` refusal; a
  field rename keeps `walk_by` consistent the same way `index_by` is handled.
- **Golden vectors.** Extend `tests/fixtures/record_walk/vectors.json` with grouped
  cases (choices order, needs group, empty groups skipped, single-group, two-group,
  wrap, `«` / `»` sanitizer cases) consumed by the unit and proptest suites.
  `record-groups` copies the file byte-for-byte into bob-plugins.
- **Tests.** Cover CLI fixtures for validation, grouped sync output, set and unset
  churn, the at-most-five-lines value edit through both update paths, no churn for
  albums without `walk_by`, capture and `records new` radius in grouped albums, and
  report and show JSON.
- **Docs.** In `docs/records.md`: walk order, the `walk_by` key, and a churn-table row
  for value edits. Also `docs/reference/json-outputs.md` (within its word budget) and
  the generated CLI docs (`just docs-gen`). Gates: `just fmt`, `just check`, and
  `just docs-check`, then `just install` on athena. Never edit the Cargo version or
  `CHANGELOG.md`. Do not set `walk_by` on a live album in this phase.

### `record-groups`: grouped record walk model and provider

bob-plugins, `plugins/bob-ledger-tools`.

- Copy the updated `vectors.json` byte-for-byte into `scripts/fixtures/record_walk/` and
  extend `scripts/test-ledger-tools-record-walk.cjs` to run the grouped cases.
- `recordWalkOrder` (`242t`) takes the album's grouping from the cached report
  (`walk_by` and `fields[].choices[]`) and mirrors bob's order exactly, including the
  `«` / `»` sanitizer. `recordWalkModel` gains `group` (field, label, value, index,
  count, position, size), `prevGroup` / `nextGroup` heads, and wrap flags.
- **Cross-check safety.** When the album's frontmatter sets `walk_by` but the cached
  report lacks `record_walk_groups_v1`, the model reports `syncing` with reason
  `grouping_unavailable` and `recordWalkRequestSync` does **not** request a sync, so an
  old bob can never flatten grouped lines. The rail keeps following written links, as
  today.
- Extend `api.navGroups.resolve` with `route: "record"` from the model: group stops are
  the heads of the neighboring occupied groups (wrapping, `wraps` set), `members` lists
  the current group, and `none` covers ungrouped, single-group, and syncing albums with
  the contract's notices. Records have no derived step stops; `[n` / `]n` keep following
  the written line.
- **Tests.** Cover vectors parity, the capability gate, provider output for first,
  middle, last, and wrapping members, and the needs group. Run `npm run build`,
  `npm test`, and `npm run docs:check`, bump, commit, and run `bob plugins sync`.

### `group-rail`: the group row at the bottom of property cards

bob-plugins, `plugins/bob-ledger-tools`. Confirm `bob-cli-o6` is closed first (see the
precondition).

- **Kit.** Add `propertyCardBuildGroupRow(doc, spec)` (for example
  `240l-plugin-property-card-group-row.js`, exported through
  `374-exports-property-card.js`): arms with state, wrap, glyphs, and labels, the center
  position text, and the tick or thumb track, per the contract. Teach
  `propertyCardBuildRail` the shared missing-target states for prev, next, parent, and
  children items (`state` beside the old `unresolved`, which maps to `creatable`). Add
  the CSS for both rows, light and dark, narrow, `forced-colors`, and reduced motion.
  Existing cards' pixels change only where the contract says (missing items and the new
  row).
- **Day and period cards** (`296b`–`296d`): read `walkMode.links` v7 (fall back to
  `api.navGroups`) for the group row and the legacy derived step row; repaint on vault
  create, delete, and rename and at midnight; and run the group commands per the
  activation rule.
- **Record walk rail** (`242u` / `242v`): add the group row for grouped albums, replace
  the whole-walk thumb track while grouped, and update the count tooltip wording.
- **Album card:** the `walks by` pill on the folded schema line.
- **Preview and proof.** Extend `scripts/preview-property-cards.cjs` (or the title-block
  preview) with fixtures for a day mid-week, today with a creatable next Monday, a week
  mid-quarter, W47 crossing into 2027, 2026-W01 with a ghost, Q4 with a ghost, a year
  (no row), a legacy no-nav-line daily, grouped zettel records (first, middle,
  wrapping), and a single-group album. Render light and dark × 375px and wide, plus
  grayscale and `forced-colors`, inspect every image, and fix what looks wrong. Unit
  tests cover the builder, states, narrow form, and activation re-validation.
- Update `docs/intention-card.md` (Day and period cards), `docs/record-card.md`, and
  `docs/albums.md` (behavior only, within word budgets). Run `npm run build`,
  `npm test`, and `npm run docs:check`, bump, commit, run `bob plugins sync`, then run
  the intention-card, record-card, record-walk, and title-block e2e scripts on athena
  and attach their reports with `sase artifact create --bead`.

### `links-footer`: group arms and hints in the links walker footer

bob-plugins, `plugins/bob-ledger-tools` (`121`, `124`, or a new `124a` fragment).

- Add `linkGroupPrev` / `linkGroupNext` parts outside the step arms in
  `freshnessFooterEnsureDom` and paint them in `freshnessFooterPaintLinks`. Add the
  `bob-freshness-link-ghost` class, the context text, the hints, the boundary hints, and
  the shedding order from the contract in `freshnessFooterFit`.
- Strip `«` / `»` / `<<` / `>>` in `linksWalkerFooterLabel`. Add the new members to the
  paint key, the re-validation, and activation through `open-prev-group-link` /
  `open-next-group-link`.
- **Tests.** Cover views for every row of the footer table, ghost and creatable arms,
  conflicts, the only-group-links note, shedding at narrow widths, and stale-snapshot
  refusal. Extend `scripts/obsidian-walker-footer-e2e.cjs` for a week and a daily and
  run it on athena.
- Update the links-footer section of `docs/bob-ledger-tools.md`. Run `npm run build`,
  `npm test`, and `npm run docs:check`, bump, commit, and run `bob plugins sync`.

### `adopt-and-prove`: album adoption, real-Obsidian proof, final deploy

- **Host gate.** On athena, apollo, and the Mac (per `tailnet.md`), confirm that the
  vault's deployed `bob-navigation-hotkeys` and `bob-ledger-tools` manifests match
  `origin/master`, and that `bob albums list -f json` reports `record_walk_groups_v1`.
  If any host lags, adopt nothing. Attach the evidence, leave
  `PROPOSED FOLLOW-UP: adopt walk_by once <host> runs bob <version>`, and continue with
  the proof on fixtures.
- **Adopt.** When every host passes: set `walk_by` through the new `bob albums update`
  op on `sase_blog_zettel` (`section`), `subscriptions` (`status`), and `eat`
  (`status`), then run `bob albums sync` (dry-run, review, apply) per album. Confirm the
  zettel walk order follows the outline and that only walk lines changed.

> [!decision] walk_more_albums Also set `walk_by: status` on `movies` and `podcasts` in
> the same pass.

- **Real-Obsidian e2e.** Add `scripts/obsidian-group-jump-e2e.cjs` (isolated CDP profile
  and vault, offline wrapper in `npm test`) and run it on athena under xvfb. It must
  prove:
  - `[N` / `]N` on W30, W47, Q4, a Saturday daily, a legacy daily (including derived
    `[n` / `]n`), and a year (notice);
  - authored `«` / `»` and `<<` / `>>` on a plain note, and that `<< X` does not hijack
    `[n` and prose `Read more »` is ignored;
  - a creatable future week and daily created through their writers, a pre-adoption
    ghost that writes nothing (hash the vault before and after), and `]}` never
    creating;
  - a grouped album: `]n` crosses a section boundary, `]N` lands on the next head, `↺`
    wraps, and a Record Card section edit rewrites at most five walk lines;
  - rail and footer clicks match the keys, and `<C-o>` returns.
- Run the record-walk, walker-footer, intention-card, record-card, and title-block e2e
  suites, and render the final preview matrix. Run every gate: bob-plugins
  `npm run build:check`, `npm test`, and `npm run docs:check`; bob-cli `just check` and
  `just docs-check`. Deploy the final clean tree with `bob plugins sync`. Attach the e2e
  reports, preview index, adoption diffs, and gate logs as artifacts.

### `glossary`: accepted memory updates

Use `/sase_memory_write`, apply only the memory decisions answered yes, then run
`sase memory init`. If all are no, add a bead note saying so and change nothing.

> [!decision] memory_glossary Update two strands in the `glossary` web:
>
> - `glossary:record-note`: replace "linking it to the previous and next records in the
>   album's title order" with: the walk follows the album's title order, or with
>   `walk_by` set, the field's choice groups in declared order and then title order, so
>   `[n` / `]n` finish a group before entering the next and `[N` / `]N` jump between
>   group heads.
> - `glossary:walker-footer`: replace "the links footer for `n` (a destination compass
>   over the note's prev/next links)" with: the links footer for `n` (a destination
>   compass over the note's step and group stops — written or authored links first, then
>   the stops Bob derives for days, periods, and grouped records).

> [!decision] memory_decision_group_stops Add the strand
> `decisions:group-stops-are-derived` in the decisions web format. Its claim: `[N` /
> `]N` group stops are derived from note identity on dailies, periods, and grouped
> records and never written, and are authored only as `«` / `»` links on link-only nav
> lines, which override the derived answer. Cite this plan and the research report.

## Coordination rules for every phase

- bob-plugins phases edit `src/` fragments only, rebuild `main.js`, keep hand-edited
  fragments at or below 1000 lines, and deploy with `bob plugins sync` (never `-F` on
  your own; follow the refusal procedure in that repo's `AGENTS.md`).
- `nav-grammar`, `calendar-groups`, and `walk-by-cli` may run in parallel, as may
  `group-rail` and `links-footer`. When rebasing, keep both sides' `fragments.json` and
  test entries, regenerate `main.js` instead of merging it, take the next free manifest
  minor version, and fetch, merge, and build before every sync.
- API changes after a phase lands are additive and feature-detected. Cross-plugin
  members keep `version` fields and never throw.
- Phase workers never create beads. They record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead.
- No memory file changes outside `glossary`, and only for memory decisions answered yes.

## Acceptance (land agent)

- On today's daily, a past daily, a legacy no-nav-line daily, `2026/2026-W30.md`,
  `2026/2026-W47.md`, `2026/2026-W01.md`, `2026/2026-Q4.md`, `2026.md`, and a zettel
  record, `[N` / `]N`, `[{` / `]}`, the card group row, and the footer all name and open
  the same destination. Year notes show no group row and a truthful notice.
- Creatable stops look dashed with `+` and create through the existing writers only on
  an explicit press or click. Ghost stops look dotted and faint and never write. `]}`
  never creates. No pre-adoption year gains a period note.
- `[n` / `]n` on the zettel walk a whole section before entering the next. `[N` / `]N`
  land on group heads and wrap with `↺`. Albums without `walk_by` are byte-unchanged.
- The adoption was applied, or its host gate is recorded with the blocking host.
- All gates are green, or red only on known, already-tracked flakes, each cited. The
  vault deploy matches `origin/master` for both touched plugins.

## Non-goals

- No `«` / `»` bytes written into dailies, periods, or records. No change to daily
  templates, period scaffolds, or the 3-link walk line format.
- Free-text grouping (`eat.location`), multi-field grouping, and an album index that
  follows `walk_by` are v2.
- No counts on `]N` / `]}` for family `n`. No `{` / `}` group keys inside the card
  editor. No Schema Card editing of `walk_by`. No mobile surfaces.
- No memory edits except accepted memory decisions, applied in `glossary`.
