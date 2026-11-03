---
tier: epic
title: Off-clock link chips — show the `#off` chip after ledger Task Links to off-clock
  tasks
goal: 'In a daily note''s Pomodoro ledger, every Task Link whose task resolves off
  the clock (an explicit `#off` or alias, or an inherited area/project `offclock:
  true`) wears the same dashed `timer-off` `off` chip that off-clock Event Tasks already
  wear, in Live Preview and Reading view, so Bryan can see at a glance which planned
  or logged work will keep a session off the work clock.

  '
decisions:
  history_rows:
    ask: Also chip links under closed entries and in past daily notes, not just today's
      open Pomodoros?
    default: true
    why: 'The clock belongs to the task, like the #off tag it stands in for; home
      chips mark history too.'
    answer: true
  prose_links:
    ask: Also put the chip after prose task links (the ones wearing a status ring),
      not only ledger rows?
    default: false
    why: The clock only decides Pomodoro sessions; chips across every love-note link
      would be noise.
    answer: false
phases:
- id: contract
  title: Authoritative display contract (bob-cli)
  depends_on: []
  size: small
  description: 'contract: write docs/off-clock-link-chips.md with the which-links,
    resolution, look, placement, tooltip, liveness, toggle, and vector tables below,
    register it, and add two cross-links.'
- id: chip
  title: The chip in bob-ledger-tools (bob-plugins)
  depends_on:
  - contract
  size: medium
  description: 'chip: implement the contract in bob-ledger-tools — pure core, Live
    Preview widget, Reading-view sweep, shared CSS, toggle, liveness, unit and parity
    tests, plugin docs, build, and bob plugins sync.'
- id: live_verify
  title: Real-Obsidian e2e and visual review (bob-plugins, bob-cli)
  depends_on:
  - chip
  size: medium
  description: 'live_verify: drive the chip in an isolated real-Obsidian scratch vault,
    compare its computed look with the Event Task chip, record screenshots and timings,
    and write the contract''s Live verification section.'
proposed_by: bbugyi200.apollo.c9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 09:51:46
status: done
bead_id: bob-cli-gf
---

- **PROMPT:** [prompts/202611/off_clock_link_chips.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/off_clock_link_chips.md)
- **BEAD:** [bob-cli-gf](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gf/README.md)

# Off-clock link chips

## Goal

Off-clock Event Tasks already show their clock: in the `## Events` section of a daily
note, a standalone `#off` tag (or a deprecated alias) renders as a quiet chip — a
`timer-off` icon, the tag name without `#`, muted ink, a dashed pill outline — and the
event's time pill turns dashed. Untagged Event Tasks that inherit `offclock: true` get
the dashed time pill too.

Under `## Pomodoros`, nothing shows a task's clock. A LUNCH session that links
`[[love#^lunch]]` looks exactly like a BUILD session that links `[[bob#^parser]]`, even
though the first task is off the clock (it inherits `love.md`'s `offclock: true`) and
the session it is linked under will be stamped `[off:: Nm]` instead of `[t:: Nm]`. Bryan
asked to render the `#off` tag, the same way it is rendered on Event Tasks, next to Task
Links whose task is off-clock — **whether or not the task's own line carries `#off`**.

```text
before:
- [ ] () — LUNCH
	- love > ^lunch
- [ ] (**1210-1235** [t:: 25m]) — MEET
	- this note > ^standup-1210
	- this note > ^school-pickup

after:
- [ ] () — LUNCH
	- love > ^lunch ⊘off                 (inherited: love.md has offclock: true)
- [ ] (**1210-1235** [t:: 25m]) — MEET
	- this note > ^standup-1210
	- this note > ^school-pickup ⊘off    (Event Task tagged #off)
```

(`⊘off` stands for the existing dashed `timer-off` chip.) In the MEET example the
sibling link has no chip, which is also the visual explanation of why that mixed session
stays on the work clock.

## What exists today (read before implementing)

All paths below are in the **bob-plugins** linked repo (open it with `/sase_repo`; read
its `AGENTS.md`) unless they start with `docs/` in bob-cli.

- **The look to reuse.**
  `plugins/bob-ledger-tools/src/291b-plugin-gcal-event-buffers.js`
  (`buildCalendarEventOffClockChip`, `CalendarEventOffClockChipWidget`) and
  `src/291c-plugin-gcal-event-clock.js` (`GCAL_EVENTS_OFF_CLOCK_TITLE`,
  `gcalEventsFromNoteTitle`, `gcalEventsPaintOffClockTag`). CSS in
  `plugins/bob-ledger-tools/styles.css`: the
  `.bob-calendar-events .bob-calendar-event-off-clock` rule (inline-flex pill,
  `outline: 1px dashed color-mix(in srgb, var(--text-muted) 55%, transparent)`, muted
  ink, `0.88em`), its `::before` mask drawn from `--bob-calendar-off-clock-icon` (Lucide
  `timer-off`, defined on `.bob-calendar-events`), and the `forced-colors` rule that
  keeps the dash. All of it is scoped under the `bob-calendar-events` body class (the
  calendar-painting toggle). Documented in bob-plugins `docs/bob-ledger-tools.md` →
  "Calendar Events".
- **The clock rule, already mirrored in JS.** `src/036-pomodoro-clock.js`:
  `taskEffectiveClockResolution(path, line, metadataCache, vault)` (`#on` → `#off`/alias
  → nearest area/project `offclock:` on the root walk, starting at a valid `[root::]` →
  on-clock), `pomodoroClockLinkFacts(lines, links, resolveTask, resolveClock)` (one fact
  per link: same-note links read `lines`, Event Tasks first, then the unique checkbox
  line carrying the id; other-note links go through `resolveTask`), and `clockVerdict`.
  The spelling table and tag ranges are in `src/035-meet-event-run.js`
  (`OFF_CLOCK_TAGS`, `eventTaskClockTag`, `eventTaskOffClockTagRange`). Shared vectors:
  `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_clock/vectors.json` (a byte copy
  of bob-cli `tests/fixtures/pomodoro_clock/vectors.json`).
- **The writer's per-link resolvers.** `src/273-plugin-pomodoro-home-chips.js`:
  `pomodoroClockResolveTaskLine(link)` and `pomodoroClockResolveLinkClock(link, line)`
  (used by `se<X>` in `src/300-plugin-vim-and-snippets.js`). Both use the **active
  file** as the source path. Other-note task lines come from
  `pomodoroHomeChipTaskLineForBlock` → `taskThemeLineTextFor` → the source-text cache in
  `src/279-plugin-task-theme-api.js` (`taskThemeSourceTextFor`,
  `taskThemeRequestSourceText`, a 32-note LRU filled by `vault.cachedRead`, refreshed
  from open editors, and **never invalidated when a closed note changes**).
- **The ledger-row rule.** `src/148-task-link-marks.js`:
  `taskLinkMarkIsLedgerTaskLinkLine(lines, index, path)` and
  `taskLinkMarkLedgerBodyIsDedicated(body)` — a direct child bullet, at the first child
  indent, of a column-0 Pomodoro entry (open or closed) in a canonical daily note's
  `## Pomodoros` section, whose body is exactly one `[[target#^id]]` (alias allowed),
  optionally after `🍅 `, optionally struck `~~…~~`, optionally followed by ` #`. Task
  link status rings skip exactly these rows (bob-cli `docs/task-link-marks.md`, rule 5 /
  TC8), so rings (prose) and chips (ledger) never share a link.
- **Sibling ledger marks to copy.** 🔑 frees marks (`src/141b-frees-marks.js`,
  `src/271b-plugin-frees-marks.js`, `scripts/test-ledger-tools-frees-marks.cjs`,
  contract bob-cli `docs/frees-marks.md`) and 🔓 (`src/139-unblocked-glyph.js`,
  `src/271-plugin-unblocked-glyph.js`). Their Live Preview widgets sit at the link end
  with `side: 1` (🔓) and `side: 2` (🔑). Reading view maps a rendered `li` to its link
  through `progressMarkReadingContext(li, root, options)` in
  `src/269-plugin-progress-marks-reading.js`, whose `ledgerMarkClass` allow-list names
  every display-only host that may sit beside a ledger link.
- **Plumbing.** `setupMarkFamilies()` in `src/284-plugin-task-link-marks.js` (the one
  setup hook, because `src/170-plugin-lifecycle.js` is at the 1000-line fragment limit),
  `scheduleLiveWidgetRefresh()` in `src/220-plugin-heading-and-review.js` (the Tasks
  `cache-update` fan-out), `applyMarkGateClass` / `registerMarkGatePopoutSync` in
  `src/266-plugin-mark-gate-mirror.js`, mixin installation in
  `src/310-install-methods.js`, test exports in `src/350-exports.js`, and the fragment
  order in `src/fragments.json`.

## Design (decided)

### Principles

- **Display-only.** Nothing is written. No `#off` is ever materialized onto a task line
  or a ledger row (decision `off-clock-follows-tag-then-note-default` rejects
  materializing inherited tags). Without the plugin the vault looks exactly as it does
  today.
- **One rule, one resolver.** A chip appears iff the session-clock writer would resolve
  that link off-clock: the chip runs `pomodoroClockLinkFacts` with the same
  `resolveTask` / `resolveClock` sources `se<X>` uses, parameterized by the rendered
  note's path instead of the active file. A fact with `offClock === true` gets a chip;
  `false` or `null` (unresolved) gets none.
- **About the task, not the session.** The chip shows the linked task's current
  effective clock. It never reads, judges, or changes the session's stamped `[t::]` /
  `[off::]` key (a closed row's key is history, and drift is reported elsewhere). A
  mixed lineup shows chips only on its off-clock links.
- **Truthful or neutral.** Unresolved notes, missing or ambiguous same-note ids, lines
  that are not checkbox rows, a note whose text is not cached yet, and any thrown error
  show no chip and no warning.
- **Same look as the Event Task chip.** One shared CSS rule draws both chips; changing
  it changes both.

### Which links get a chip

A link gets a chip only when **all** of these hold:

1. The rendered note is a canonical daily note (`PLAN_DAILY_PATH_RE`).

> [!decision] history_rows Any canonical daily note, under any Pomodoro entry, open or
> closed.

> [!decision] history_rows = no Only today's canonical daily note
> (`currentTodayDailyPath`) and only rows under open entries (running or `()`
> placeholders); add the local-midnight rollover to the refresh triggers, as 🔑 does.

2. The line is a dedicated ledger Task Link row per `taskLinkMarkIsLedgerTaskLinkLine`:
   plain, `🍅`-prefixed, struck `~~[[…]]~~`, aliased, or followed by the ` #` move-only
   marker. The chip describes the task, so struck (completed) and ` #` (deferred) rows
   are included even though ` #` rows do not count toward the session clock.
3. Not embedded (`![[…]]` renders the task's own line), not mixed prose, not a deeper
   bullet, not fenced, not in frontmatter or a `%%` comment, not in `## Events` or any
   other section.
4. The link's fact resolves `offClock === true`.
5. The session toggle is on.

> [!decision] prose_links Also chip prose task links (see "Prose links" under phase
> `chip`); rings and chips then cover every rendered task link, rings for status and
> chips for the clock.

> [!decision] prose_links = no Prose links keep only their status ring. Record "prose
> links" under Deferred with the trigger "Bryan asks".

### Resolution (what "off-clock" means here)

Exactly the counted-links rule's per-link step (bob-cli `docs/reference/vault-format.md`
→ "Task clock tags and note defaults"): a standalone `#on` on the task line wins (no
chip, even beside an off spelling); else `#off`, `#offclock`, `#off-clock`, or
`#untracked` (case-insensitive, standalone, outside code spans and Markdown links) is
off; else the nearest area or project `offclock: true|false` on the root walk (a valid
`[root:: NOTE]` starts the walk; untyped notes are walked through; an invalid nearer
value is on-clock); else on-clock. Same-note links (`[[#^id]]`, including Event Tasks in
the daily note's `## Events`) read the note's own current lines — in Live Preview the
live editor document, so an unsaved `#off` edit shows on the same frame. Other-note
links read the target line through the writer's source (metadata-cache block line +
source-text cache).

### Anatomy and look

- Host:
  `span.bob-off-clock-link-chip[role=img][aria-label][data-tooltip-position=top][data-clock-source][data-block-id]`,
  text content exactly `off` (always the canonical spelling, even for an alias or an
  inherited clock; the tooltip names the source). `data-clock-source` is `off_tag` or
  `note`. No `title` attribute; listener-free DOM builder that takes a `doc`, like
  `buildFreesMarkElement`.
- Look: identical to the Event Task chip. Refactor `styles.css` so one rule serves both:
  define the `timer-off` mask variable on `body` (rename it `--bob-off-clock-icon` and
  update its one user), and add `.bob-off-clock-link-chip` to the Event Task chip's look
  rule, its `::before` icon rule, and its `forced-colors` rule — without scoping the new
  class under `.bob-calendar-events`, so turning calendar painting off does not hide
  link chips. Chip-only additions: `margin-inline-start: 0.4em`,
  `vertical-align: baseline`, `user-select: none` (copying never copies `off`),
  `cursor: default`, `print-color-adjust: exact`. The chip never changes line height and
  takes no document positions.
- Gate:
  `body:not(.bob-off-clock-link-chips) .bob-off-clock-link-chip { display: none; }`; the
  JS checks in the builder and the sweep remain the source of truth.

### Placement

- **Live Preview:** a zero-width `Decoration.widget` at the link end — just after `]]`,
  or after the closing `~~` of a struck link, and before any trailing ` #` — with
  `side: 3`, so a link that also has 🔓 (`side: 1`) and 🔑 (`side: 2`) reads
  `link 🔓 🔑 N off`. The chip stays put while the cursor reveals raw `[[…]]`.
- **Reading view:** insert the host after the run of `span.bob-unblocked-glyph` /
  `span.bob-frees-mark` siblings that follows the link's `a.internal-link` (or its
  `del`/`s` wrapper when the anchor is that wrapper's only child); with none, directly
  after the anchor or wrapper. A later 🔓 or 🔑 insertion lands before the chip on its
  own, so the order converges in any processor order.
- **Allow-list:** add `bob-off-clock-link-chip` to `ledgerMarkClass` in
  `progressMarkReadingContext`, so ◐, 🔓, 🔑, and the ring's ledger-row exclusion still
  recognize a row that already wears a chip. Without this, a chipped ledger row would
  stop being a ledger row and could grow a status ring.

### Tooltip (`aria-label`, Obsidian tooltip, one line, never contains `::`)

- `off_tag`: `Off-clock · tagged <spelling>`, where `<spelling>` is the tag exactly as
  the task line writes it (from `eventTaskOffClockTagRange`), e.g.
  `Off-clock · tagged #off`, `Off-clock · tagged #Untracked`.
- `note`: `Off-clock · from <stem>` via `gcalEventsFromNoteTitle` — the same copy the
  inherited Event Task time pill uses, e.g. `Off-clock · from love`.
- Any stray `::` becomes `:`.

### Interaction

Copy 🔑's handlers: a plain mousedown in Live Preview places the caret at the widget
position and focuses the editor; Mod+click (primary button) opens the linked task like
Mod-clicking the link (`workspace.openLinkText(linktext, sourcePath, "tab")`). Never
navigates on a plain click, never writes, no tab stop, no live-region announcement. No
animation.

### Liveness

One coalesced `scheduleOffClockLinkChipsRefresh()` (about 120 ms) runs on:

- the `scheduleLiveWidgetRefresh()` fan-out;
- `metadataCache` `changed` (any note: an `offclock:`, `type:`, or `parent:` edit
  changes inheritance, and a target line can gain or lose a tag) and `resolved`;
- vault `rename` and `delete`;
- a source-text cache fill (`taskThemeRequestSourceText` already refreshes theme marks
  and home chips there; add this refresh beside them);
- the toggle.

Live Preview also rebuilds on `docChanged`, `viewportChanged`, the Live Preview ↔
source switch, and a file switch — never on selection alone. The Reading refresh sweeps
rendered daily notes in every window (main and popouts), adding, updating, or removing
hosts in place.

**Source-text freshness (small shared fix).** On `metadataCache` `changed(file, data)`,
when `file.path` is already in the source-text cache, replace the cached text with
`data` (do not add new entries). Today a note edited outside an open editor (sync, `bob`
writes) keeps stale cached text, so both home chips and `se<X>` can read an old line;
this makes the chip, home chips, and the writer agree with the file.

### Toggle

Command `toggle-off-clock-link-chips`, named "Toggle Task Link off-clock chips";
session-only, default on, `applyMarkGateClass("bob-off-clock-link-chips", on)` plus
`registerMarkGatePopoutSync`. Independent of "Toggle calendar event painting". No api
namespace.

### Performance

Resolve each unique `(pathPart, blockId)` once per build or sweep; add Live Preview
widgets only inside visible ranges; never read from disk during render (the source-text
cache fills asynchronously outside render, as today). A node perf test with a synthetic
200-link ledger over 50 target notes keeps one Live Preview build under about 25 ms.

### Deferred (record in the contract with triggers)

- Bob Mac Capture's idle agenda rows, the Task Card, and nav pickers. Trigger: Bryan
  asks. The Mac is a thin client (`decisions:mac-capture-is-a-thin-client`), so this
  needs a per-link clock in bob's `capture-pomodoros --tasks` JSON first.
- Embedded `![[…]]` ledger rows.
- An `#on` override marker on links whose task opts back onto the clock under an
  off-clock note.
- A tooltip line naming the session effect (for example why a mixed session stays
  worked).
- Prose links (only when decision `prose_links` is `false`).

## Conformance vectors (the contract copies these; JS tests run them verbatim)

Shared fixture unless a row says otherwise: daily note `2026/20261103.md` (today,
2026-11-03); `love.md` has `type: "[[area]]"` and `offclock: true` and holds
`- [ ] #task Lunch ^lunch`; `bob.md` has `type: "[[project]]"`, no `offclock`, and holds
`- [ ] #task Fix the parser ^parser`.

**OC — which links.**

| #    | Ledger (under `## Pomodoros`)                                       | Chip?                                         |
| ---- | ------------------------------------------------------------------- | --------------------------------------------- |
| OC1  | `- [ ] () — LUNCH` / `\t- [[love#^lunch]]`                          | yes                                           |
| OC2  | `- [x] (**1200-1300** [off:: 60m]) — LUNCH` / `\t- [[love#^lunch]]` | yes (`history_rows`); no when it is `false`   |
| OC3  | open entry / `\t- 🍅 [[love#^lunch]]`                               | yes                                           |
| OC4  | open entry / `\t- [[love#^lunch]] #`                                | yes; at the link end, before ` #`             |
| OC5  | open entry / `\t- ~~[[love#^lunch]]~~`                              | yes; after the closing `~~`                   |
| OC6  | open entry / `\t- [[love#^lunch\|Lunch]]`                           | yes                                           |
| OC7  | open entry / `\t- ![[love#^lunch]]`                                 | no (embed)                                    |
| OC8  | open entry / `\t- lunch with [[love#^lunch]]`                       | no (mixed prose)                              |
| OC9  | open entry / `\t\t- [[love#^lunch]]` under another child            | no (deeper bullet)                            |
| OC10 | OC1 in the past daily note `2026/20261102.md`                       | yes (`history_rows`); no when it is `false`   |
| OC11 | `## Events` row or a `## Notes` bullet linking `[[love#^lunch]]`    | no (`prose_links`: no; yes when it is `true`) |
| OC12 | OC1 inside a fenced block                                           | no                                            |
| OC13 | OC1 in a non-daily note that has a `## Pomodoros` heading           | no                                            |
| OC14 | open entry / `\t- [[bob#^parser]]`                                  | no (on-clock)                                 |
| OC15 | toggle off                                                          | no (CSS gate and JS)                          |

**OR — resolution.** Each row is an OC1-shaped ledger link to the given task.

| #    | Linked task                                                                                | Chip / `data-clock-source`        |
| ---- | ------------------------------------------------------------------------------------------ | --------------------------------- |
| OR1  | `- [ ] #task Walk #off ^a` in `bob.md`                                                     | yes / `off_tag`                   |
| OR2  | `- [ ] #task Walk #OFFCLOCK ^a` in `bob.md`                                                | yes / `off_tag`; text still `off` |
| OR3  | untagged task in `abby.md` (area, `parent: "[[love]]"`, no `offclock`)                     | yes / `note` (love)               |
| OR4  | untagged task in `fun.md` (area under love, `offclock: false`)                             | no                                |
| OR5  | `#off` task in `fun.md`                                                                    | yes / `off_tag`                   |
| OR6  | `- [ ] #task Lunch #on ^a` in `love.md`                                                    | no                                |
| OR7  | `- [ ] #task Lunch #on #off ^a` in `love.md`                                               | no (`#on` wins)                   |
| OR8  | untagged task in `odd.md` (area under love, `offclock: maybe`)                             | no                                |
| OR9  | untagged task in `dog.md` (no `type`, `parent: "[[love]]"`)                                | yes / `note` (love)               |
| OR10 | untagged `- [ ] #task Swim [root:: abby] ^a` in `inbox.md`                                 | yes / `note` (love)               |
| OR11 | same-note `[[#^evt]]` to Event Task `- [ ] **1200-1230** School pickup #off ^evt`          | yes / `off_tag`                   |
| OR12 | same-note `[[#^evt]]` to untagged Event Task `- [ ] **1200-1230** Park [root:: abby] ^evt` | yes / `note` (love)               |
| OR13 | `[[nowhere#^a]]`, a missing id, or a same-note id carried by two lines                     | no                                |
| OR14 | target line `- Walk #off ^a` (no checkbox)                                                 | no                                |
| OR15 | other-note target whose text is not cached yet                                             | no; appears after the cache fills |
| OR16 | `- [x] #task Lunch ^lunch` in `love.md` (done; the link is struck)                         | yes / `note` (love)               |
| OR17 | Live Preview, same-note task edited to add `#off`, unsaved                                 | yes on the same frame             |
| OR18 | `#off` inside a code span or a Markdown link text on the target line                       | no                                |

**OT — tooltip.**

| #   | Case           | aria-label                     |
| --- | -------------- | ------------------------------ |
| OT1 | OR1            | `Off-clock · tagged #off`      |
| OT2 | OR2            | `Off-clock · tagged #OFFCLOCK` |
| OT3 | OR3            | `Off-clock · from love`        |
| OT4 | OR11           | `Off-clock · tagged #off`      |
| OT5 | any stray `::` | becomes `:`                    |

**OP — placement.**

| #   | Case                                           | Result                                                   |
| --- | ---------------------------------------------- | -------------------------------------------------------- |
| OP1 | Live Preview                                   | widget at the link end, `side: 3`; no line-height change |
| OP2 | the link also has 🔓 and 🔑                    | order: link, 🔓, 🔑 N, chip (both views)                 |
| OP3 | Reading view, no other mark                    | host directly after `a.internal-link` (or its `del`)     |
| OP4 | cursor on the row (raw `[[…]]` revealed)       | chip still at the link end                               |
| OP5 | a chipped row in Reading view                  | ◐ / 🔓 / 🔑 still paint; no status ring appears          |
| OP6 | Event Task chip and link chip, computed styles | same outline style, outline color, ink, font size, icon  |

**OV — parity with the writer.** For every case in the shared
`scripts/fixtures/pomodoro_clock/vectors.json` whose session clock comes from its
existing lineup (exclude MEET event-run cases, close cases, and cases whose token adds
links; list each excluded id and why in the test), `clockVerdict` over the chip facts of
that session's links equals `expected_off_clock`, and a link wears a chip iff its fact
is off-clock.

## Phase `contract` — authoritative display contract (bob-cli)

Write `docs/off-clock-link-chips.md` in bob-cli, in the style of `docs/frees-marks.md`
and `docs/task-link-marks.md`. Sections: What you see (the before/after block);
Principles; Which links; Resolution; Anatomy and look; Placement; Tooltip; Interaction;
Liveness (including the source-text freshness fix); Toggle; Performance; Deferred;
Conformance vectors (OC, OR, OT, OP, OV copied from this plan). Write each decision's
selected branch as plain contract text (no decision callouts in the published page).
Name the JavaScript mirror files phase `chip` adds (core
`src/141c-off-clock-link-chips.js`, mixins `src/271c-plugin-off-clock-link-chips.js` and
`src/271d-plugin-off-clock-link-chips-reading.js`, test
`scripts/test-ledger-tools-off-clock-link-chips.cjs`) and state that there is no api
namespace.

Register the page:

- `docs/_meta/contracts.yml`, "Tasks and projects" group, right after `frees-marks`,
  with the next free `order` and a summary like "Off-clock link chips: the Event Task
  `off` chip after ledger Task Links whose task resolves off the clock, tagged or
  inherited";
- `mkdocs.yml` nav, right after `frees-marks.md`;
- `docs/_meta/url_contract.txt` only if the docs gate requires every published URL
  there.

Cross-links (one sentence each, nothing else in those pages):

- `docs/reference/vault-format.md` → "Task clock tags and note defaults": Obsidian shows
  an `off` chip after ledger Task Links whose task resolves off-clock, tagged or
  inherited ([Off-clock link chips](off-clock-link-chips.md)).
- `docs/task-link-marks.md` → rule 5 (ledger rows): ledger rows wear no ring; when their
  task is off-clock they wear the off-clock link chip instead.

> [!decision] prose_links Also add a short "Off-clock chip" subsection to
> `docs/task-link-marks.md` (after Placement): prose links that wear a ring get the same
> chip at the link end when the ring's target resolves off-clock, gated only by the chip
> toggle; and remove "prose links" from the new page's Deferred list.

Do not edit guides, memory, `Cargo.toml`, or `CHANGELOG.md`. Run `just docs-gen` if the
registry change needs regenerated indexes (`docs/README.md`,
`docs/develop/contracts.md`), then `just docs-check` until it passes. Suggested commit
subject: `docs(off-clock-chips): add ledger off-clock link chip display contract`.

## Phase `chip` — the chip in bob-ledger-tools (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Implement bob-cli
`docs/off-clock-link-chips.md` exactly; the contract wins, and if it is wrong, fix it in
bob-cli in this phase. Edit only `src/` fragments, then `npm run build` (never hand-edit
`main.js`); keep every hand-edited fragment at or below 1000 lines.

1. **Path-parameterized writer resolvers** (`src/273-plugin-pomodoro-home-chips.js`).
   Add `pomodoroClockResolveTaskLineFrom(link, sourcePath)` and
   `pomodoroClockResolveLinkClockFrom(link, line, sourcePath)`; make the existing
   active-file methods delegate to them with the active file's path, so `se<X>` behaves
   exactly as before. In `src/036-pomodoro-clock.js`, add the resolved line text to each
   `pomodoroClockLinkFacts` fact as an additive `lineText` field (needed for the tooltip
   spelling; existing callers ignore it).
2. **Pure core** `src/141c-off-clock-link-chips.js` (after `141b-frees-marks.js` in
   `fragments.json`), synchronous and never throwing, in the style of
   `141b-frees-marks.js`:
   - `offClockLinkChipLinks(lines, path, options)` → frozen
     `[{line, ch, pathPart, blockId, token, entryOpen}]` for every row that
     `taskLinkMarkIsLedgerTaskLinkLine` accepts (respecting fences, frontmatter, and
     comments through the same document context the ring uses), where `ch` is the link
     end (after `]]`, or after the closing `~~`; before ` #`). `options.openOnly`
     filters to open entries (decision `history_rows`).
   - `offClockLinkChipModel(fact)` →
     `{text: "off", source, notePath, spelling, tooltip}` or null unless
     `fact.offClock === true`; `offClockLinkChipTooltip(model)` per the OT table.
   - `buildOffClockLinkChipElement(doc, model)` (listener-free) and
     `OffClockLinkChipWidget` (`eq` on tooltip and source; handlers copied from 🔑).
3. **Live Preview** in `src/271c-plugin-off-clock-link-chips.js`
   (`BobLedgerToolsOffClockLinkChipsMixin`, installed after the frees mixin in
   `src/310-install-methods.js`): setup from `setupMarkFamilies()`, the toggle command,
   the ViewPlugin with the rebuild rules above, `buildOffClockLinkChipDecorations(view)`
   (live document lines → links → `pomodoroClockLinkFacts` with the `…From` resolvers
   and a per-build `(pathPart, blockId)` memo → widgets at `side: 3` in visible ranges),
   `scheduleOffClockLinkChipsRefresh()`, editor refresh via a `StateEffect`, and
   teardown wired wherever the frees teardown is.
4. **Reading view** in `src/271d-plugin-off-clock-link-chips-reading.js`: a markdown
   post-processor (priority after 🔑's 60) that records the rendered root and sweeps it,
   plus a refresh sweep over every rendered daily note in every window. For each `li`,
   `progressMarkReadingContext(li, root, { includeStruck: true, includeClosedEntry: <history_rows>, allowLedgerMarks: true })`
   gives the anchor, target, and block id; facts come from the note's own lines
   (`taskThemeSourceTextFor(sourcePath)`, requesting the text when absent) for same-note
   links and from the `…From` resolvers otherwise. Add, update, or remove hosts in place
   per Placement; remove orphaned hosts. Add `bob-off-clock-link-chip` to
   `ledgerMarkClass` in `src/269-plugin-progress-marks-reading.js`.
5. **Liveness wiring.** Call `scheduleOffClockLinkChipsRefresh()` from
   `scheduleLiveWidgetRefresh()`, from the source-text fill in
   `taskThemeRequestSourceText`, and from the mixin's own `metadataCache`
   `changed`/`resolved` and vault `rename`/`delete` listeners; add the source-text
   freshness fix (update an already-cached entry from `changed`'s `data`).
6. **CSS** (`plugins/bob-ledger-tools/styles.css`): the shared-rule refactor, chip-only
   additions, and gate from "Anatomy and look". Verify the Event Task chip renders
   unchanged (existing gcal-events tests stay green; check the computed declarations in
   the test harness or by inspection).
7. **Prose links.**

> [!decision] prose_links Extend the ring pipeline: in `buildTaskLinkMarkDecorations`
> (`src/284-plugin-task-link-marks.js`) and the Reading reconcile
> (`src/285-plugin-task-link-marks-reading.js`), for each link that resolves to a ring,
> compute the clock with `taskEffectiveClockResolution(targetPath, line, …)` where
> `line` is the resolver's raw target line (the live-document and Tasks-memo sources;
> the metadata fallback has no line, so no chip), and add the same chip at the link end
> (`side: 3`) or after the anchor. Gate prose chips only by the chip toggle: when rings
> are off the scan still runs for chips. Add the corresponding vectors (OC11 flips to
> yes) to the tests.

> [!decision] prose_links = no Skip this step; prose links are untouched.

8. **Exports and tests.** Export the pure helpers through `src/350-exports.js`. Add
   `scripts/test-ledger-tools-off-clock-link-chips.cjs` (register it in `package.json`
   `test`) running OC, OR, OT, OP1–OP5 (fake DOM, as the frees tests do), OV parity over
   the shared pomodoro_clock vectors, the writer-delegation check (the active-file
   methods return what they returned before), the source-text freshness update, the
   toggle gate, and the perf bound. Keep `scripts/test-ledger-tools-pomodoro-clock.cjs`,
   `scripts/test-ledger-tools-gcal-events.cjs`, the frees, unblocked, progress, and
   task-link-marks tests green.
9. **Docs** in bob-plugins `docs/bob-ledger-tools.md`: one bullet in "Marks and rendered
   blocks" linking the contract (`https://bobs-org.github.io/off-clock-link-chips/`),
   one sentence at the end of "Calendar Events" saying ledger Task Links to off-clock
   tasks wear the same chip, and a "Session toggles" paragraph for the new command. Add
   one clause to the `plugins/bob-ledger-tools/manifest.json` description and bump its
   minor version (as every ledger-tools feature does), then `npm run docs:gen`.
10. **Gates.** `npm run build`, `npm run build:check`, `npm test` (compare any failure
    with a baseline run on the unmodified tree and report pre-existing failures instead
    of fixing unrelated code), `npm run validate`, `npm run docs:check`. Then run
    `bob plugins sync`. Suggested commit subject:
    `feat(ledger-tools): show the off-clock chip after ledger links to off-clock tasks`.

## Phase `live_verify` — real-Obsidian e2e and visual review (bob-plugins, bob-cli)

Adapt `scripts/obsidian-frees-marks-e2e.cjs` and its lib (isolated scratch vault and
profile, owned Obsidian process, CDP probes; never the live vault) into
`scripts/obsidian-off-clock-link-chips-e2e.cjs` plus
`scripts/obsidian-off-clock-link-chips-e2e-lib.cjs`, with an offline self-test
`scripts/test-obsidian-off-clock-link-chips-e2e.cjs` registered in `npm test` like the
frees one. Run it under `xvfb-run -a`.

The scratch vault holds `love.md` (area, `offclock: true`), `abby.md` (area under love),
`fun.md` (area under love, `offclock: false`), `bob.md` (project, one `#off` task, one
untagged task), and today's daily note with `## Events` (one `#off` Event Task, one
untagged Event Task with `[root:: abby]`) and `## Pomodoros` (a LUNCH with a love link,
a MEET with both events, a BUILD mixing a love link and a bob link, one closed entry,
one struck row, one ` #` row). Use Tasks settings from the frees e2e. Verify, in Live
Preview and Reading view:

- chips exactly per OC/OR on those rows, with OT tooltips;
- OP2 order on a link that also has 🔓 or 🔑 (seed a last-blocker dependency), and OP5
  (◐ / 🔓 / 🔑 still paint, no ring on chipped rows);
- OP6: the link chip's computed outline style and color, ink, font size, and mask image
  match the Event Task chip on the same page;
- liveness: setting `fun.md` to `offclock: true` adds its chip, and adding `#off` to the
  untagged `bob.md` task while that note is closed adds its chip, each within about one
  second and without reopening the daily note; an unsaved same-note `#off` edit shows on
  the same frame;
- the toggle hides and restores chips in both views and in a popout;
- the calendar-painting toggle off leaves link chips visible.

Save light and dark screenshots of the ledger in both views and review them for line
height, spacing, and alignment against the Event Task chip. If Obsidian or `xvfb-run` is
unavailable on the host, run the offline self-test, record the live run as skipped with
the reason, and say so in the final report.

Then, in bob-cli, add a "Live verification" section to `docs/off-clock-link-chips.md` in
the style of `docs/frees-marks.md`'s (what ran, where, measured refresh timings, and
screenshot notes), and run `just docs-check`. Run `bob plugins sync` after any
bob-plugins change. Suggested commit subjects:
`test(off-clock-chips): add isolated Obsidian e2e` in bob-plugins and
`docs(off-clock-chips): record live verification` in bob-cli.
