---
tier: epic
title: '#off / #on clock tags and inherited offclock: area defaults'
goal: 'Off-clock life needs no tag: an area or project note''s `offclock: true`, inherited
  down the task''s root walk and overridable by a child''s `offclock: false` or a
  task''s `#on`, decides the clock of every untagged task and Event Task beneath it;
  `#off` replaces `#offclock` as the one spelling bob writes in Obsidian and reads
  from Google Calendar; `#offclock`, `#off-clock`, and `#untracked` keep working while
  `bob gcal sync` and `bob gcal list` warn about them (with a once-a-day Obsidian
  Notice); past Event Task rows are rewritten to `#off`; and `love.md` turns on the
  first area default.

  '
decisions:
  memory_decision_records:
    ask: Should new clock-rule and capture clock-tag decision records partly supersede
      the two records they amend?
    memory:
    - decisions:off-clock-follows-tag-then-note-default
    - decisions:off-clock-pomodoros-count-apart
    - decisions:capture-reserves-clock-tags
    - decisions:capture-sigils-say-where-what-how
    default: true
    corrects: it is off-clock only when at least one remains and every link resolves
      to a task whose own line has the standalone `#offclock` tag or an alias.
    answer: true
  memory_glossary:
    ask: 'Should the glossary describe #off, #on, and offclock note defaults (off-clock-pomodoro,
      event-task, area/project notes)?'
    memory:
    - glossary:off-clock-pomodoro
    - glossary:event-task
    - glossary:area-note
    - glossary:project-note
    default: true
    corrects: A session goes off the clock exactly when it keeps at least one [[task-link]]
      and every one resolves to a task tagged `#offclock` (aliases `#untracked`, `#off-clock`)
    answer: true
phases:
- id: clock-contract
  title: 'bob-cli: one clock spelling table, #on, offclock: note facts, and the effective-clock
    resolver'
  size: medium
  depends_on: []
  description: 'clock-contract: add the shared spelling table and tri-state ClockTag,
    make #off canonical and #on win on vault lines, parse offclock: into NoteFacts
    with lints, and add effective_clock plus Event Task enrichment over the PT13 root
    walk.'
- id: clock-writers
  title: 'bob-cli: capture start, close, restamp, and MEET event runs stamp the effective
    clock'
  size: medium
  depends_on:
  - clock-contract
  description: 'clock-writers: thread a home resolver into every capture clock decision,
    name inherited sources in summaries and JSON, extend the shared pomodoro_clock
    vectors with frontmatter cases, and write the clock decision record and glossary.'
- id: clock-readers
  title: 'bob-cli: agenda JSON, reconcile settle, and stats drift use the effective
    clock'
  size: medium
  depends_on:
  - clock-contract
  description: 'clock-readers: enrich Event Tasks in the capture-pomodoros agenda,
    settle inherited off-clock Event Tasks in reconcile, and recompute clock_drift
    by the same rule, with docs.'
- id: gcal-clock-tags
  title: 'bob-cli: gcal reads #off/#on, renders only explicit tags, plans on the effective
    clock, and warns on deprecated spellings'
  size: large
  depends_on:
  - clock-contract
  description: 'gcal-clock-tags: tri-state Google clock from titles and directive
    lines, #on as a directive, explicit-only badge render with in-place migration,
    effective clock for clusters and created MEETs, tri-state flip reports, and deprecation
    warnings plus deprecations[] in sync and list.'
- id: capture-clock-tags
  title: 'bob-cli: capture keeps #off/#on at item edges as tags, never type markers'
  size: small
  depends_on:
  - clock-contract
  description: 'capture-clock-tags: reserve the clock spellings in the capture parser
    and editor scanner, refuse them as type names, teach the existing-task error,
    and record the narrow sigil exception.'
- id: plugins-clock
  title: 'bob-plugins: spelling tables, offclock: inheritance in Obsidian starts and
    closes, copied vectors'
  size: medium
  depends_on:
  - clock-writers
  description: 'plugins-clock: one spelling table per plugin, #on wins, the ledger-tools
    chain walker reads offclock: and exports taskEffectiveOffClock, se<X> starts and
    the cycler close use it with a tag-only fallback, and vectors are copied byte-for-byte.'
- id: plugins-marks-notice
  title: 'bob-plugins: Event Task clock marks for #off, #on, and inherited rows, and
    the daily deprecation Notice'
  size: medium
  depends_on:
  - plugins-clock
  - gcal-clock-tags
  description: 'plugins-marks-notice: paint inherited off-clock Event Tasks dashed
    with a from-note tooltip, keep #off tag marks, give #on a neutral override tooltip
    in both views, and raise one Notice a day from deprecations[].'
- id: vault-landing
  title: 'Vault landing: install everywhere, rewrite legacy rows to #off, turn on
    love.md, verify'
  size: small
  depends_on:
  - clock-writers
  - clock-readers
  - gcal-clock-tags
  - capture-clock-tags
  - plugins-marks-notice
  description: 'vault-landing: confirm every host runs the new bob and plugins, rewrite
    re-scanned legacy Event Task rows to #off, add offclock: true to love.md, list
    Google titles to retitle, verify stats and drift, and close ^off-tag.'
proposed_by: bbugyi200.apollo.research.31.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 17:38:42
status: wip
bead_id: bob-cli-gb
---

- **PROMPT:** [prompts/202611/off_on_tags_and_offclock_areas.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/off_on_tags_and_offclock_areas.md)
- **BEAD:** [bob-cli-gb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gb/README.md)

# `#off` / `#on` clock tags and inherited `offclock:` area defaults

## Goal

Epic `bob-cli-fl` made an Off-Clock Pomodoro (`[off:: Nm]`) out of every session whose
counted Task Links all target `#offclock` tasks. Two things still hurt:

- Off-clock life already has a structural home (`love.md` and its sub-areas `abby`,
  `family`, `fun`, `kelly`, `vacation`, …), yet every lunch, school pickup, or park trip
  still needs a hand-typed tag, and Google events routed with `@love`, `@abby`, or
  `@family` still need `#offclock` in their titles.
- `#offclock` is long to type in a phone Calendar title; `#off` matches the `[off::]`
  key it produces.

After this epic, the task's clock is resolved as `#on`, then `#off` (or a deprecated
alias), then the nearest area/project `offclock:` on the task's root walk, then
on-clock. `love.md` gets `offclock: true`, so every untagged love task and every
love-routed Event Task is off the clock; a child area can opt back in with
`offclock: false`, and any task or Google event can opt back in with `#on`.

Context: research report
`research:202611/off_on_tags_and_offclock_areas/off_on_tags_and_offclock_areas.md` (read
it with `sase artifact read`). Bryan accepted every requirement and adjustment it
recommends (A1–A10); this plan adopts them and splits the work. The predecessor epic is
`bob-cli-fl` (`plan:202611/off_clock_pomodoros.md`), now closed.

## The design in one page

**Vocabulary (rename only the tag).**

| Spelling                                                                                                | Status                                                                                                                     |
| ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `#off`                                                                                                  | canonical off-clock task tag; the only off spelling any writer emits                                                       |
| `#on`                                                                                                   | new on-clock override                                                                                                      |
| `#offclock`, `#off-clock`, `#untracked`                                                                 | deprecated aliases: parsed everywhere forever; warned about only at the Google boundary (`bob gcal sync`, `bob gcal list`) |
| `offclock: true` / `offclock: false`                                                                    | new area/project frontmatter default; absent or blank means inherit                                                        |
| `[off:: Nm]`, JSON `off_clock` (+ deprecated `untracked`), Stats `off-clock` lens, "Off-Clock Pomodoro" | unchanged                                                                                                                  |

**How a task's clock is resolved — first match wins:**

1. The task's own line has a standalone `#on` → **on-clock**. If an off spelling is also
   present, `#on` still wins and the diagnostic `clock_tag_conflict` is raised.
2. The line has `#off` or a deprecated alias → **off-clock** (beats a note's `false`).
3. The nearest **area or project** note on the task's root walk whose `offclock:` is
   present → that value. `false` ends inheritance; a deeper `true` below a `false` turns
   it back on.
4. That nearest value is not `true`/`false` (case-insensitive, bare or quoted) →
   diagnostic `note_offclock_invalid`, **on-clock** (a typo never inherits the parent's
   `true`; ambiguity fails toward Worked).
5. Nothing set → **on-clock**.

**Root walk (same as the PT13 type-default walk in
`src/native/pomodoro_themes/suggest.rs`):** start at the task's valid `[root:: NOTE]`
target (the `set_root_walk_start` rule: resolves and is not `@unfiled`), otherwise at
its residence note; follow `HomeResolver::resolve_note(..).residence_chain` (16-note
cap, cycle-safe, archive and reference wrappers walked through). A canonical daily note
contributes nothing unless `[root::]` retargets the walk, so Event Tasks inherit only
through their root. An `offclock:` key on a note that is neither area nor project is
ignored (diagnostic `note_offclock_ignored`), and the walk continues through it, so a
task in untyped `dog.md` still inherits from `love.md`.

**Unchanged rules.** The counted-links rule (`clock_verdict` over the links a session
keeps) is untouched: an unlinked session, an unresolved link, or a mixed lineup (one
love task plus one job task) stays Worked in full. Tags on a parent task, heading, or
Task Link never inherit; only notes inherit.

**Compute at write time; never materialize.** Writers resolve the effective clock when
they stamp `[t::]`/`[off::]`; readers keep trusting the stamp; closed rows stay frozen.
Nothing ever writes an inherited `#off` or `#on` onto a task line, and `bob gcal sync`
renders only the tag Google sent. Paths without vault access fall back to tag-only,
which fails toward Worked.

## Decisions made here (adopted from the research, not reviewer choices)

- **A1** `#off-clock` is deprecated alongside `#offclock` and `#untracked`.
- **A2** Deprecation warnings appear in `bob gcal sync` and `bob gcal list` (human
  stderr and JSON `warnings[]`), plus an additive structured `deprecations[]` in both
  JSON outputs, and the Obsidian daily auto-sync raises one Notice a day from it.
- **A3** The warning suggests deleting the tag when the event's root already inherits
  off-clock.
- **A4** "Belonging" to an area/project is the PT13 root walk above.
- **A5** `#on` beats an off spelling on the same line or event (with a diagnostic); an
  explicit `#off` beats a note's `false`; values are strict `true`/`false`; an invalid
  nearer value resolves on-clock.
- **A6** Inherited off-clock Event Tasks auto-settle in `bob task reconcile` and are
  completed, exactly like tagged ones (the accepted cost "the calendar is not proof of
  attendance" widens to every ended, unstarted love-routed event; escape hatches: cancel
  before it ends, or `#on`).
- **A7** Capture reserves the clock spellings at item edges so they stay tags.
- **A8** Only Event Task rows are migrated, and only after every host runs the new code;
  re-scan at landing. Prose mentions (`bob.md` `^untracked-gcal-event`), code spans,
  `.bob/ref-sources/**`, and `_conflicts/**` are left alone.
- **A9** Only the tag is renamed (see the vocabulary table).
- **A10** Sync never writes an inherited `#off`/`#on`; it renders only what Google sent.
- No release-note `!`: aliases keep parsing and every JSON change is additive.
- Bob Mac Capture needs no change (`decisions:mac-capture-is-a-thin-client`): it decodes
  bob's `off_clock`, which now carries the effective value. Its fixtures that still say
  `#untracked` stay valid.

## Shared definitions (all phases)

**Spelling table (bob-cli).** One module (suggested `src/native/clock_tags.rs`) owns
`OFF_TAG = "#off"`, `ON_TAG = "#on"`, and
`DEPRECATED_OFF_TAGS = ["#offclock", "#off-clock", "#untracked"]`. Every hard-coded list
goes away: `event_task.rs` (`line_has_off_clock_tag`, `strip_trailing_off_clock`),
`gcal/model.rs` (`is_off_clock_token`, `strip_off_clock_tokens`), `gcal/lifecycle.rs`
(`strip_trailing_off_clock_token`), `gcal/render.rs` (writes ` #offclock`), and
`gcal/description.rs` (directive tokens). `clock-contract` replaces the vault-side lists
(`event_task.rs`); `gcal-clock-tags` replaces the gcal ones.

**Matching rules (unchanged, now applied to all five spellings).** Vault lines: a
standalone token, ASCII case-insensitive, preceded by start-of-line or whitespace,
followed by end, whitespace, or one of `])}:.,;!?`; never nested (`#off/x`), never a
prefix (`#off` does not match `#offclock`, `#office`, `#off-clock`; `#on` does not match
`#one`, `#online`), never inside code spans or Markdown links (the existing
`mask_code_and_markdown_links`). Google titles and description directive lines: whole
whitespace-split tokens compared case-insensitively after trimming trailing `.,;:!?`.

**`ClockTag` (tri-state parse of one line or event).** `None`, `Off { deprecated }` (the
deprecated spellings seen, possibly empty), `On`, or `Conflict { deprecated }` (`#on`
plus an off spelling; resolves exactly like `On`). Helper names are the implementer's;
`explicit_off()` (true only for `Off`) is the tag-only fallback.

**`NoteOffClock` (frontmatter `offclock:`).** Read with the existing
`frontmatter_scalar`: absent or blank → `None` (inherit); `true`/`false`
case-insensitive, bare or quoted → `True`/`False`; anything else (including
`yes`/`no`/`on`/`off`/`1`/`0`/lists) → `Invalid(raw)`. Exactly one key: `offclock`.

**Clock source vocabulary** (JSON values, additive fields):

| `clock_source` | Meaning                                       | `clock_note`                     |
| -------------- | --------------------------------------------- | -------------------------------- |
| `on_tag`       | `#on` on the line or event (incl. a conflict) | null                             |
| `off_tag`      | `#off` or a deprecated alias                  | null                             |
| `note`         | the nearest area/project `offclock:` decided  | vault-relative path of that note |
| `invalid_note` | the nearest value was invalid → on-clock      | that note's path                 |
| `default`      | nothing set → on-clock                        | null                             |

**Diagnostic codes.** `clock_tag_conflict`, `note_offclock_invalid`,
`note_offclock_ignored` (both note lints are recorded once per note in `ResolverHealth`,
next to `area_color_invalid`, and render wherever resolver lints already render, such as
`bob stats` health). Capture surfaces the first three as structured warnings only when
they decided a counted link's clock.

**Human "via" copy.** When every counted link of an off-clock session resolved through
note defaults of the same note, start and close summaries say
`(60m · off-clock via love)` (note stem); otherwise the existing `· off-clock`.

## Phase details

### clock-contract (bob-cli)

Lay the shared contract. After this phase every existing writer already honors explicit
`#off`/`#on` (tag-only); inheritance is wired by the following phases.

1. Add the spelling table and the `ClockTag` parse. Reimplement `line_has_off_clock_tag`
   on top of it (or replace its call sites with `line_clock_tag(line).explicit_off()`),
   so `#off` counts and `#on` wins everywhere a line is read today: capture start/close
   (`capture_pomodoro_start.rs`, `capture_pomodoro_close/linked_tasks.rs`), stats
   (`stats/homes.rs`), and Event Task parsing (`event_task.rs`).
   `clean_event_title`/`strip_trailing_off_clock` strip trailing `#off`, `#on`, and the
   aliases from Event Task titles.
2. `NoteFacts` (`src/native/pomodoro_homes/facts.rs`) gains
   `offclock: Option<NoteOffClock>`.
3. Add
   `effective_clock(resolver, residence_path, line) -> ClockResolution { off_clock, source, note }`
   implementing the precedence and root walk above, reusing `set_root_walk_start` logic
   and the residence chain exactly as `note_default` does (share a helper rather than
   copying it). Record `note_offclock_invalid` / `note_offclock_ignored` in
   `ResolverHealth` (new vectors in the struct, rendered where `area_color_invalid`
   renders).
4. `EventTask` and `EventNoteRow` keep the line's `ClockTag` and gain a
   `clock_source`/`clock_note`; parsing initializes `off_clock` (and the deprecated
   `untracked` mirror) tag-only. Add one enrichment helper, e.g.
   `enrich_event_clocks(daily_path, contents, &mut events, resolver)`, that re-reads
   each row's line (for `[root::]`) and sets the effective `off_clock`, `untracked`,
   `clock_source`, and `clock_note`. Later phases call it at each entry point with vault
   access.
5. Tests (unit, in the touched modules): spelling boundaries for all five spellings
   (`#off.`, `#OFF`, `#off/x`, `#offclock` vs `#off`, `#on` vs `#one`, code spans,
   Markdown links); `#on` + `#off` → on with `clock_tag_conflict`; note value parsing
   (`true`, `"TRUE"`, `false`, `maybe`, blank); the walk on a fixture tree: love `true`
   → abby inherits; fun `false` → on; surf (under fun) `true` → off; untyped `dog` with
   `offclock: false` is ignored + linted and `lenard` (under dog) inherits love's
   `true`; a nearer `offclock: maybe` → on + lint; a daily-note task without root →
   default; an Event Task `[root:: abby]` → off via love; invalid root → falls back to
   the residence (daily → default) with the existing `root_field_invalid`; a parent
   cycle terminates.
6. Docs: `docs/reference/vault-format.md` (the tag table, matching rules, the precedence
   list, and `offclock:` documented beside `ready_cap`/`task_refresh`); a "same walk,
   different field" pointer in `docs/task-types.md` and `docs/pomodoro-homes.md`. Run
   `just docs-gen` and `just docs-check`.

Done when `just check` passes and an explicit `#off` task stamps `[off::]` through
`bob capture` in a scratch vault.

### clock-writers (bob-cli)

Every capture writer stamps the effective clock.

1. Capture start: `resolve_session_clock_links` → `resolve_clock_fact` /
   `clock_fact_from_task` (`src/native/capture_pomodoro_start.rs`) resolve each counted
   link with `effective_clock` over a `HomeResolver` built on the capture's snapshot
   (reuse `CaptureHomeSource` from `capture/task_theme.rs` or adapt
   `SnapshotCloseVault`; unsaved planner state must win over disk). The same-note Event
   Task branch uses `enrich_event_clocks`. With no vault, stay tag-only.
2. Capture close: `remaining_close_clock_facts` / `resolve_close_clock_fact`
   (`capture_pomodoro_close/linked_tasks.rs`) likewise.
3. Running-lineup restamps (`restamp_session_clock` and every caller listed in
   `bob-cli-fl` note #3) and MEET event runs (`capture/meet_event_run.rs`, which reads
   `event.off_clock` from the note's events) enrich events before clustering; event
   conflict warnings at start (`capture/event_conflict.rs`) use the effective clock.
4. `CountedLinkFact`/`CloseClockFact` carry the `ClockResolution`. Start and close human
   summaries use the "via" copy; start/close JSON gains additive `clock_note` (the
   deciding note's path when the "via" rule applies, else null). The
   `pomodoro_clock_mixed` warning is unchanged except that it now also fires for a love
   task mixed with a job task. Surface `clock_tag_conflict` and `note_offclock_invalid`
   as structured capture warnings when they decide a link.
5. Shared vectors: extend `tests/fixtures/pomodoro_clock/vectors.json` (and its README:
   `#off` canonical, `#on`, aliases, and that `notes` may carry frontmatter) with cases
   whose `notes` include frontmatter: love `offclock: true` → an untagged child task
   goes off; an Event Task `[root:: abby]` (abby under love) goes off; a `false` child
   with a `true` grandchild; untyped `dog` passes inheritance through; `#on` under love
   stays worked; `#on` + `#off` conflict stays worked; `offclock: maybe` stays worked;
   invalid root stays worked; a love task plus a job task stays worked with
   `pomodoro_clock_mixed`; an explicit `#off` in a `false` child goes off. Switch the
   canonical existing case to `#off` and keep alias cases. Add an inherited-root case to
   `tests/fixtures/meet_event_run/vectors.json` when the event-run code path changes.
   Keep the README's pin rule (plugins copy the file from the commit that adds the
   cases).
6. Docs: `docs/capture.md` (which clock a session is on, the "via" copy),
   `docs/reference/json-outputs.md` (`clock_note`). `just docs-gen`, `just docs-check`.
7. Memory (author with `/sase_memory_write`, then `sase memory init` and `just docs-gen`
   so the generated `docs/develop/decisions.md` and `docs/reference/glossary.md` follow;
   `capture-clock-tags` may land a sibling record in parallel, so regenerate after
   rebasing rather than hand-merging generated files):

> [!decision] memory_decision_records Add
> `decisions:off-clock-follows-tag-then-note-default` (frontmatter like its siblings:
> keyword, aliases, one-line summary, `metadata.status: accepted`, `decided:`). It
> records the precedence list and root walk, stamp-time-only evaluation (closed rows
> frozen; toggling `offclock:` surfaces as drift), the canonical/deprecated spellings
> and that the vocabulary is now settled, warnings only at the Google boundary, and the
> widened settle scope (A6); evidence: this plan and the research report. Rejected
> alternatives: materializing inherited tags, a Stats-only filter, `clock:` or
> `task_type: off` as the switch, keeping `#offclock` canonical, splitting mixed
> sessions. Mark `decisions:off-clock-pomodoros-count-apart` with
> `metadata.status: superseded-in-part` and `superseded_by` the new record, plus one
> back-link sentence saying the "own line has the standalone `#offclock` tag" clause is
> retired; change nothing else in its body.

> [!decision] memory_glossary Update `glossary:off-clock-pomodoro` (the off-clock
> condition becomes "every one resolves off-clock: `#off` (deprecated `#offclock`,
> `#off-clock`, `#untracked`) or an inherited area/project `offclock: true`, with `#on`
> and `offclock: false` as overrides"; the settle sentence covers inherited Event Tasks)
> and `glossary:event-task` (`#off`, `#on`, and inheritance through `[root::]`). Keep
> both strands about as long as they are now.

> [!decision] memory_glossary Add one sentence each to `glossary:area-note` and
> `glossary:project-note`: an `offclock: true|false` key sets the clock default for
> tasks on the root walk beneath it (see
> [[decisions:off-clock-follows-tag-then-note-default]]).

Done when `just check` passes and the new vectors pass through
`tests/cli/capture/pomodoro_clock.rs`.

### clock-readers (bob-cli)

Read-side consumers agree with the writers.

1. Agenda: `src/native/capture_pomodoros_agenda.rs` (it already builds a `HomeResolver`
   near the `events_in_note` call) and the event rows in
   `src/native/capture_pomodoros.rs` call `enrich_event_clocks`; event-row JSON keeps
   `off_clock`/`untracked` (now effective) and adds `clock_source` and `clock_note`.
2. Settle (A6): `plan_settle` (`src/native/task_status_hooks/settle.rs`) receives a
   resolver from its reconcile caller and enriches events before the
   `if !event.off_clock` filter, so an ended, open, unstarted Event Task rooted under an
   off-clock area settles into a closed `[off:: Nm]` MEET and is completed, exactly like
   a tagged one. `#on` rows and rows under `offclock: false` never settle. Idempotence
   and trimming are unchanged.
3. Stats drift: `ClockTaskLookup::task_off_clock` and `resolve_link_clock`
   (`src/native/stats/homes.rs`) resolve tasks and Event Tasks with `effective_clock`
   over the resolver they already receive, so `clock_drift` matches the writers. Earlier
   love-rooted `[t::]` sessions will show as drift after `offclock: true`, by design.
4. Tests: agenda JSON for an untagged `@love` Event Task (`clock_source: note`), an
   `#on` row, and an untagged row with no default; reconcile settles an ended
   `[root:: abby]` event once and leaves an `#on` one alone; drift flags a `[t::]`
   session whose only link is now inherited off-clock and a closed `[off::]` row whose
   task gained `#on`.
5. Docs: `docs/task-status-hooks.md` (settle scope), `docs/stats.md` (drift after
   toggling a note default; `#off` examples), `docs/reference/json-outputs.md` (agenda
   fields). `just docs-gen`, `just docs-check`.

Done when `just check` passes.

### gcal-clock-tags (bob-cli)

Google input, rendering, planning, and warnings. Read `docs/gcal.md` and the decision
records `bob-gcal-is-read-only-and-proposes-once`,
`gcal-root-follows-google-until-edited`, and `synced-meetings-link-by-phase` first.

1. **Tri-state Google clock.** `GcalEvent` (`src/native/gcal/model.rs`) replaces the
   explicit boolean with a `ClockTag`-like value built from title tokens and description
   directive lines (`gcal/description.rs`). `#off`, `#on`, and the aliases are all
   directive tokens, so `@love #on -30` routes, overrides, and buffers. Clock tokens are
   stripped from the displayed title before route detection, so `Lunch @love #on` routes
   to love. Keep the existing `untracked` field of `DescriptionDirectives` working
   (deprecated mirror) and add cases additively to
   `tests/fixtures/event_buffer/vectors.json` (plugins copy it) rather than changing
   existing ones.
2. **Render only the explicit tag.** `render_task_row` / `badge_text` (`gcal/render.rs`)
   emit ` #off` when Google says off (any spelling), ` #on` when Google says on, and
   nothing otherwise — even when the root inherits off-clock. The badge span hash covers
   it, so untouched rows carrying `#offclock` migrate in place on that day's next sync
   (the mechanism behind `preexisting_off_clock_title_migrates_both_spans_in_place`).
   `gcal/lifecycle.rs` recognizes `#off`, `#on`, and the aliases when it peels the badge
   run off existing rows.
3. **Plan with the effective clock.** Past closed clusters (`cluster_past`), the created
   running MEET, and cluster grouping in `gcal/plan.rs` use the effective clock of the
   row as this run leaves it: a new row's rendered line with its chosen root, or an
   existing row's current vault line (its own tag and `[root::]`, so Bryan's root and
   tag edits win), with the Google tag applied when the run rewrites an untouched badge
   span. Build one `HomeResolver` per run (one already exists in `gcal/root.rs` and
   `gcal/render.rs`).
4. **Flip reports compare explicit tags.** `kept_off_clock` compares the row line's
   explicit `ClockTag` with Google's (none/off/on), never effective booleans, so an
   untagged love-routed row is not a flip. Copy names `#off` and `#on` ("add #off by
   hand to follow Google", "remove #on by hand …"). Keep the `kept_off_clock` code.
5. **Deprecation warnings (A1–A3).** Whenever a deprecated spelling is observed in an
   occurrence's title or directive lines — on created, updated, and unchanged rows, and
   in `--dry-run` — emit one warning line per occurrence per run (deduped across title
   and description) in `bob gcal sync` and `bob gcal list`:
   `Swings!: #offclock is deprecated in Google Calendar; write #off (or drop it: abby is already off the clock via love.md)`.
   The parenthetical appears only when the root walk alone (no tag) resolves off-clock.
   When `#on` also appears:
   `Planning: #offclock is deprecated in Google Calendar and #on wins over it; drop #offclock`.
   `#on` plus `#off` (no alias) warns `clock_tag_conflict`. Keep the strings in
   `warnings[]` (stderr for humans) and add additive JSON
   `deprecations: [{calendar_id, event_id, title, token, where: "title"|"description"|"both", replacement: "#off", drop_suggested, on_wins}]`
   to both sync and list output (empty array when none).
6. **List shows the clock.** `bob gcal list` uses the chosen root for the effective
   clock: the suffix reads `task · off-clock` (tag) or `task · off-clock via love`
   (note); JSON events keep `off_clock`/`untracked` (now effective) and add `clock`
   (`"off"`, `"on"`, or null: what Google sent), `clock_source`, and `clock_note`.
7. Google stays read-only: no new scope, no title edits.
8. Tests (`tests/gcal_lifecycle.rs`, plan tests, model tests): each alias renders
   ` #off` and warns once, including on an unchanged re-sync and in dry-run; `#on`
   overrides an off-clock root and renders ` #on`; an untagged `@love` event yields an
   off-clock past cluster and an off-clock created MEET with no tag on the row; a
   vault-edited `[root::]` keeps its own inheritance; no inherited tag is ever rendered;
   an untagged love row never reports `kept_off_clock`; `deprecations[]` shape in sync
   and list JSON; `Lunch @love #on` routes to love.
9. Docs: `docs/gcal.md` (off-clock events, `#off`/`#on` directives, inheritance through
   the root, deprecation warnings, JSON), `docs/reference/json-outputs.md`.
   `just docs-gen`, `just docs-check`.

Done when `just check` passes.

### capture-clock-tags (bob-cli)

`bob capture "Walk the dog #off"` fails today with `type_not_configured`.

1. In `src/native/capture_language/type_marker.rs` (and the editor scanner that shares
   it), a free `#` token at an item edge whose name is `off`, `on`, `offclock`,
   `off-clock`, or `untracked` (ASCII case-insensitive, same right-boundary rules) is
   not a type marker: it stays task text exactly as typed (no `\#` escape), so the
   created line carries the tag. Mid-line tokens are already prose and stay so.
   `capture-parse` emits no type-marker span for it; `capture-complete` never offers
   these names as types. No grammar version bump: drafts that failed now succeed.
2. On an item that selects an existing task (link, complete, relate, …) a reserved clock
   token is refused with a teaching error ("capture writes clock tags only on new tasks;
   edit the task line to add #off"), never silently dropped.
3. Type-name validation shared by the config `types:` loader and
   `bob types rename`/`split` refuses `OFF`, `ON`, `OFFCLOCK`, `OFF-CLOCK`, and
   `UNTRACKED` with a message naming the clock tags.
4. Tests: `bob capture "Walk the dog #off"`, `"#on Plan the sprint"`, and
   `"Gym #offclock"` write the tag verbatim; `"Walk \#off"` stays an escaped literal as
   today; an existing-task item with `#off` errors; `bob types rename FIX OFF` errors;
   parse spans for a draft with `#off` show no type marker.
5. Docs: `docs/capture.md` and `docs/reference/capture-syntax.md` (the reserved tags).
   `just docs-gen`, `just docs-check`.

> [!decision] memory_decision_records Add `decisions:capture-reserves-clock-tags` (the
> narrow exception: free `#off`, `#on`, and the deprecated spellings at item edges are
> task tags, not types; why: one clock vocabulary across Calendar, Obsidian, and
> capture; rejected: escaping (`\#off`) as the only path, a `%` directive). Mark
> `decisions:capture-sigils-say-where-what-how` superseded-in-part by adding the new
> record to its `superseded_by` list and one back-link sentence; change nothing else.

Done when `just check` passes.

### plugins-clock (bob-plugins, via `/sase_repo`)

Open `bob-plugins` with `/sase_repo`, read its `AGENTS.md`, edit `src/` fragments only,
and run `npm run build`.

1. **One spelling table per plugin** (`#off` canonical, `#on`, the three aliases), used
   by `bob-ledger-tools` `035-meet-event-run.js`, `036-pomodoro-clock.js`,
   `291-plugin-gcal-events.js`, and `task-status-cycler` `062-pomodoro-clock.js`. Drop
   the "permanent aliases" comments. Mirror the tri-state parse and boundary rules
   exactly (`#on` wins; `#off` never matches `#offclock`).
2. **Inheritance.** Extend the ledger-tools chain walker
   (`279-plugin-task-theme-home-facts.js` `pomodoroHomeResolveTaskThemeFacts`, or a
   shared helper beside it) to read `frontmatter.offclock` with the same root start,
   area/project-only rule, strict booleans, and invalid → on-clock. Export
   `taskEffectiveOffClock(lineText, file)` → `{ offClock, source, notePath }` on the
   ledger-tools `api` (`350-exports.js`), versioned like its other namespaces.
3. **Writers.** ledger-tools `se<X>` starts (`036`, the call site near
   `300-plugin-vim-and-snippets.js`) use it. The cycler close path (`062`,
   `060-pomodoro.js`) calls it through the ledger-tools runtime `api` (as
   `170-plugin-pomodoro.js` already reaches `api.themes`) and falls back to tag-only
   when the api is missing. Unsaved editor buffers win over disk for line text.
4. **Vectors.** Copy bob-cli's `tests/fixtures/pomodoro_clock/vectors.json` and
   `meet_event_run/vectors.json` byte-for-byte from the commit that added the
   frontmatter cases into both plugins' `scripts/fixtures/`; teach the JS harnesses to
   expose each `notes` entry's frontmatter through the mocked `metadataCache`.
5. Ship: `npm test`, `npm run build:check`, `npm run validate`, `npm run docs:check`,
   update `docs/bob-ledger-tools.md` and `docs/task-status-cycler.md` (clock rule,
   `#off`, `#on`, inheritance), commit, then `bob plugins sync`.

Done when both plugins pass the copied vectors and are deployed.

### plugins-marks-notice (bob-plugins, via `/sase_repo`)

1. **Event Task marks** (`291-plugin-gcal-events.js` painter, Live Preview and Reading
   view): rows with `#off` or an alias keep the dashed time pill and tag mark; untagged
   rows that resolve off-clock through `taskEffectiveOffClock` get the same dashed pill
   with the tooltip `Off-clock · from love` (note stem); `#on` rows get a neutral
   tooltip `On the clock (override)`. `291` is at about 944 lines: move the clock mark
   code into a new fragment (registered in `src/fragments.json`) so every hand-edited
   fragment stays at or below 1000 lines.
2. **Deprecation Notice.** `gcalEventsDecodeSync` decodes the additive `deprecations[]`
   (absent → empty). After a successful daily auto-sync with a non-empty list, show one
   Notice per day (reuse the `gcalSyncNoticeDays` pattern with its own set), for example
   `3 calendar events still use #offclock · run bob gcal list`.
3. Copy `tests/fixtures/event_buffer/vectors.json` from bob-cli if `gcal-clock-tags`
   changed it.
4. Tests for both marks, the tooltip copy, the once-a-day Notice, and decoding without
   `deprecations[]`; then `npm test`, `npm run build:check`, `npm run validate`,
   `npm run docs:check`, docs updates, commit, and `bob plugins sync`.

Done when both views paint the three row kinds and the Notice fires once per day.

### vault-landing (vault checklist; the only phase that edits `~/bob`)

Read the `obsidian.md` and `tailnet.md` reference memory (via `/sase_memory_read`)
first.

1. **Every host runs the new code.** Run `just install-all` on apollo, and over
   Tailscale SSH on athena and the Mac. Confirm on each host that
   `bob capture --dry-run "Walk the dog #off"` succeeds and that the deployed plugins
   contain `taskEffectiveOffClock`. If the Mac is unreachable, ask Bryan with
   `/sase_questions` whether to continue before step 2; do not edit the vault while an
   unconfirmed host could still write old clocks.
2. **Rewrite legacy Event Task rows.** Re-scan canonical daily notes' `## Events`
   checkbox rows for standalone `#offclock`, `#off-clock`, or `#untracked` (the
   scanner's rules; skip code spans, links, `.bob/ref-sources/**`, `_conflicts/**`, and
   every non-Event-Task line). On 2026-11-02 that was four rows: `2026/20261031.md`
   Halloween, `2026/20261101.md` Grocery Store, `2026/20261102.md` Swings! and Lincoln
   Park. Replace the token with ` #off` in the same badge position the new renderer
   uses, review `git -C ~/bob diff`, and let `bob vault-sync` commit it (or run it). No
   gcal journal edits: a row equal to the fresh render re-baselines its hash.
3. **Turn on love.** Add `offclock: true` to `~/bob/love.md` frontmatter next to
   `color: yellow`; touch nothing else.
4. **Google titles.** Run `bob gcal list` for the coming week and report, in the phase
   bead note and the final answer, each event Bryan should retitle: delete the tag on
   events routed to love/abby/family, rename the rest to `#off`, and check routes that
   do not resolve (journal roots `@dad` do not match a note, so those events cannot
   inherit). Never edit Google.
5. **Verify.** `bob stats -k off-clock` groups recent off-clock time under love's
   subtree; `bob stats` `clock_drift` shows only expected love-rooted `[t::]` sessions;
   `bob capture --dry-run` of a start on an untagged love task previews
   `off-clock via love`; `bob task reconcile --dry-run` behaves.
6. Refresh bob-cli's mounted plugin reference copies (`just docs-gen`,
   `just docs-check`) if the plugin docs changed them.
7. Complete `^off-tag` in `bob.md` with `bob capture '!bob:off-tag'`.

Done when the vault diff is committed, love is off-clock, and the verification output is
recorded on the phase bead.

## Compatibility and rollout

- Order: bob-cli phases land on master (hosts install from master via
  `just install-all`; the crates.io train follows on its own schedule), plugins deploy
  with `bob plugins sync`, then the vault landing.
- Readers keep trusting stamps; closed rows never change. Turning `offclock:` on or off
  only changes new decisions and is visible as drift.
- `#offclock`, `#off-clock`, and `#untracked` keep working forever with no removal date;
  only the Google boundary warns.
- A host on old code would read `#off` as on-clock, which is why the vault rewrite and
  `love.md` wait for step 1 of the landing.

## Verification (scratch vault, `BOB_NOW`, fake `BOB_GCAL_GOG`)

1. `love` area with `offclock: true` plus an untyped child: an ordinary child task and a
   `Lunch @love` event resolve off-clock with no tag; the row renders no tag and
   `bob gcal list` shows `off-clock via love` with `clock_source: note`.
2. A `false` child area and a `Planning @love #on` event resolve Worked; the latter
   renders ` #on`.
3. An all-love start/close writes `[off::]` and says `via love`; a love task plus a job
   task writes `[t::]` and warns `pomodoro_clock_mixed`.
4. `Gym #offclock` renders ` #off` and warns once per run, including on an unchanged
   re-sync; `deprecations[]` lists it.
5. An ended, unstarted `@abby` event settles once on `bob task reconcile`.
6. Flipping love back to `false` changes new decisions but leaves closed rows
   byte-identical; drift reports the difference.
7. The same `pomodoro_clock` vectors pass in bob-cli, bob-ledger-tools, and
   task-status-cycler.
