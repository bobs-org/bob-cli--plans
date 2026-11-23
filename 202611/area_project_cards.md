---
tier: epic
title: Area and project cards with a bob-validated settings editor
goal: 'Every area and project note opens with a calm property card (an Area Card or
  a Project Card) that says what the home is, where it sits, its lifecycle, and the
  policies it declares. `=+` (and `==`) edit every area/project setting through typed,
  inheritance-aware stages that bob validates and writes byte-preservingly. Once no
  second frontmatter writer remains, the cards take over native Properties on these
  notes, bank.md included, and every failure path brings native Properties back.

  '
decisions:
  card_structure_counts:
    ask: Should the card's live readouts also count sub-areas and sub-projects?
    default: false
    why: The catalogs right below already count them, so the card stays calm
    answer: false
  memory_area_card:
    ask: Add an Area Card glossary strand?
    memory:
    - glossary:area-card
    default: true
    requested: As a part of this change, we should add new glossary terms for "area
      card" and "project card".
    answer: true
  memory_project_card:
    ask: Add a Project Card glossary strand?
    memory:
    - glossary:project-card
    default: true
    requested: As a part of this change, we should add new glossary terms for "area
      card" and "project card".
    answer: true
  memory_property_card:
    ask: Update the Property Card glossary strand for the area and project species
      and bank.md takeover?
    memory:
    - glossary:property-card
    default: true
    corrects: The shared family of note-top panels — album, record, day, period, and
      bank
    answer: true
phases:
- id: one-writer
  title: Retire the second frontmatter writers on project notes (bob-plugins + vault)
  depends_on: []
  size: medium
  description: 'one-writer: remove the bob-project-tasks count writer and the project-creation
    processFrontMatter call, sweep the retired count keys out of the vault through
    bob props, and update projects.base and project.md, retiring status waiting.

    '
- id: contract-writes
  title: Area/project roles, validation, and setting ops in bob props (bob-cli)
  depends_on: []
  size: medium
  description: 'contract-writes: complete the role table for area/project notes and
    bank.md, validate contract keys on every write path, and add bob-expanded setting.set
    / setting.clear ops with golden byte tests.

    '
- id: contract-reads
  title: The settings contract in bob props show, dry-run effects, and list issues
    (bob-cli)
  depends_on:
  - contract-writes
  size: medium
  description: 'contract-reads: expose each setting''s stage, choices, declared and
    effective value with its source, the ^prj line facts, dry-run consequence effects,
    per-row contract_issues in areas/projects list, the props_contract_v1 capability,
    and docs.

    '
- id: home-cards
  title: Area Card and Project Card painted from local data (bob-plugins)
  depends_on: []
  size: large
  description: 'home-cards: add the area and project species and one shared home adapter
    that paints kicker, crumb, title, subtitle or lifecycle line, authored chips,
    the Ready readout, the pf foot with where-marks, collapse, and material, with
    mount rules, fail-open, e2e, screenshots, and docs. Native Properties stays visible.

    '
- id: card-live
  title: Report-backed lifecycle reasons, readouts, lint, and the ^prj goal line (bob-plugins)
  depends_on:
  - home-cards
  - contract-reads
  size: medium
  description: 'card-live: patch the cards in place from the already-cached areas/projects
    reports (blocked reasons, close dates, open/today readouts, contract issues in
    the foot lint) and decorate the recognized ^prj line as the note''s goal.

    '
- id: editor-settings
  title: The settings group in =+, ghost rows, simple stages, and == registration
    (bob-plugins)
  depends_on:
  - home-cards
  - contract-reads
  size: large
  description: 'editor-settings: add the AREA/PROJECT SETTINGS group with ghost rows
    and clear-to-inherit, the ^prj handoff rows, locked and managed groups, the summary,
    ready cap, review interval, capture names, and Stats root stages, == on area/project
    notes, and the degraded mode.

    '
- id: editor-policy-stages
  title: Hue, default type, and clock-policy stages (bob-plugins)
  depends_on:
  - editor-settings
  size: medium
  description: 'editor-policy-stages: add the swatch stage with a live tint preview,
    the type picker with legacy theme migration, and the four-mode clock-policy stage
    with shadowed-pair cleanup.

    '
- id: editor-parent
  title: The Lives under stage with consequence preview and catalog refresh (bob-plugins)
  depends_on:
  - editor-settings
  size: medium
  description: 'editor-parent: add the restricted home picker (reusing the nav-hotkeys
    parent picker model), the bob dry-run consequence preview, and post-apply catalog
    syncs inside the guarded session, with byte-exact undo.

    '
- id: takeover
  title: Hide native Properties on area/project notes and arbitrate bank.md (bob-plugins
    + memory)
  depends_on:
  - one-writer
  - card-live
  - editor-policy-stages
  - editor-parent
  size: large
  description: 'takeover: own the frontmatter on eligible area/project notes behind
    props_contract_v1, run the bank.md spike and mount the Area Card there with bank_foot
    suppression, reroute native entry points, run the fail-open matrix, finish the
    docs, and apply the approved glossary decisions.'
proposed_by: bbugyi200.apollo.research.5y.linker.w1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 20:56:37
status: wip
bead_id: bob-cli-pl
---

- **PROMPT:** [prompts/202611/area_project_cards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/area_project_cards.md)
- **BEAD:** [bob-cli-pl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pl/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                        | Why                                                                                |
| ------------ | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| derives-from | [research:202611/area_project_property_cards/area_project_property_cards.md][1] | Consolidated research whose recommended requirements Bryan accepted for this epic  |
| related      | [plan:202611/card_owned_frontmatter.md][2]                                      | Builds on bob-cli-pf's foot row, drawer, =+ editor, bob props writer, and takeover |

[1]:
  https://github.com/bobs-org/bob-cli--research/blob/main/202611/area_project_property_cards/area_project_property_cards.md
[2]:
  https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md

<!-- sase:links:end -->

# Plan: Area and project cards

This epic turns
`research:202611/area_project_property_cards/area_project_property_cards.md` into nine
phases. Bryan accepted every requirement that research recommends, including the default
of each of its "Decisions for Bryan" and each of its "Smaller calls". Memory edits are
the only exception: they happen only through the memory decisions above. This plan owns
the design. Where it refines the research, it says so in
[Refinements](#refinements-to-the-research). When a phase needs the reasoning behind a
rule, read the report with `sase artifact read <ref> "<why>"`. Its member report
`area_project_property_cards__cld.md` (§4–§5) has the richest card and editor detail,
and `__cdx.md` covers the clock-policy nuances.

## Launch precondition and as-landed APIs

This epic builds on `bob-cli-pf` (`plan:202611/card_owned_frontmatter.md`). That epic
must be **closed** before any phase starts. Every phase begins with
`sase bead read bob-cli-pf -r "<why>"`. If it is not closed, stop: record a note on your
own phase bead and change no files.

`bob-cli-pf` was still in progress when this plan was written, so names below that come
from it (fragment numbers, API names, JSON fields) are its _approved contract_. Before
coding, read the as-landed versions:

- bob-cli `docs/props.md`;
- bob-plugins `docs/property-editor.md` and `docs/develop/architecture.md` (kit foot
  API, ownership bit, `=` arm table);
- the `glossary:property-card` strand.

Where landed code differs from this plan's assumptions, follow the code, keep this
plan's intent, and record the difference in your land notes.

## Refinements to the research

These are the lead design calls this plan makes on top of the research:

1. **A second re-serializing writer the research missed.** Project creation
   (`bob-navigation-hotkeys/src/660-plugin-project-notes.js:58`, calling
   `applyProjectCreationFrontmatter` in `110-project-seed.js:759`) runs
   `processFrontMatter` right after Templater creates the note. `one-writer` removes it.
2. **Retired count keys stay deletable.** Once nothing writes `task_count` /
   `open_task_count`, a locked "derived" row would be a dead end. They become loose keys
   marked `legacy`, folded under Bob-managed, and Ctrl+D removes them.
3. **The cap is part of the Ready readout, not a separate chip.** The card reads
   `2/8 ready`; the tooltip names where the cap comes from. That leaves one fewer chip.
4. **Project state paints at once and never flickers.** `task-status-hooks` already
   writes Blocked as `[?]` on the `^prj` line, so the state word comes from the line's
   own checkbox mark and inline `[scheduled::]`. The report only adds the _reason_ and
   the close date.
5. **Takeover here requires `props_contract_v1`, not just `props_v1`.** Native
   Properties never disappears from these notes before the validated settings editor
   exists on that machine.
6. **No sub-area/sub-project counts on the card** unless `card_structure_counts` is
   turned on (see the decision).
7. **Re-homing is validated by bob, and the plugin runs the catalog syncs that bob's
   dry-run names.** This is the research's default ("run the existing syncs inside the
   same guarded session"), made concrete.

## The experience (design, owned by this plan)

### 1. Two species, one grammar

The cards are title-block cards on the existing kit: a species tile and kicker word, one
owned heading, at most one subtitle line, a content-gated chip row, and `bob-cli-pf`'s
foot row and `=-` drawer. Hue says _where_ (the note's effective home color under the
existing resolver). Form and word say _what_. The kit's rule is unchanged: "Hue says
where, form says what, and words say both."

| Species   | Word    | Glyph (Lucide) | Tile form (new)                                                                                                                                | Why                                              |
| --------- | ------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `area`    | AREA    | `layers`       | `field`: a soft rounded tile filled with the card accent at wash weight. An area is where hue comes from, so its tile is literally the swatch. | `layers` is already the `## Areas` catalog icon. |
| `project` | PROJECT | `target`       | `pennant`: the tile's trailing edge is cut into a shallow swallowtail notch, like a finish-line flag. Terminal projects show only the outline. | `target` matches the `#prj` mark.                |

Glyphs are monochrome and `aria-hidden`; the word is the accessible prefix ("Area:
love", "Project: sase_blog"). Both silhouettes must be distinct from the seven existing
tiles. The `home-cards` screenshot pass confirms this. If `layers`/`target` collide
visually in practice, the fallback pair is `compass`/`flag`; record the swap.

### 2. The Area Card

```text
╭─────────────────────────────────────────────────────────────────────╮
│ ▤  AREA                                                 ⊞ Settings  │  kicker row
│    love                                                             │  title
│    Family and relationships: life with Kelly and Abby, our dogs,    │  summary (≤2 lines)
│    relatives, fun, and trips.                                       │
│    #CHORE   off-clock · MEET ⚠                         0/5 ready    │  chips · readouts
│ ─────────────────────────────────────────────────────────────────── │
│ ▸ Properties · 8   ⚠ 1                                              │  pf foot row
╰─────────────────────────────────────────────────────────────────────╯

╭─────────────────────────────────────────────────────────────────────╮
│ ▤  AREA · gtd › inbox                          INBOX    ⊞ Settings  │
│    Mac Inbox                                                        │  adopted "# Mac Inbox"
│    Default target for bob capture and Bob Mac Capture; triage …     │
│                                                 14 ready · no cap   │
│ ─────────────────────────────────────────────────────────────────── │
│ ▸ Properties · 5                                                    │
╰─────────────────────────────────────────────────────────────────────╯
```

- **Kicker:** `AREA`, then the parent crumb built from the local `parent:` chain (real
  links, Mod-click opens a new tab, at most 4 crumbs with a middle ellipsis). The root
  sentinel `area` never appears, so a top-level area reads just `AREA`, never
  `AREA · area`. The top-level ancestor crumb carries a 6 px hue pip.
- **Pills (right):** `INBOX` on inbox files (the existing `api.inboxRoute.isInboxNote`
  route, `inbox.md` included). Then `⊞ Settings`, which opens `=+` on the settings group
  (the same as `==`).
- **Title:** a recognized first-body H1 (bob's rule: before any bullet or H2), owned and
  hidden at render time with Live Preview reveal-on-touch. Otherwise the first `aliases`
  entry. Otherwise the **stem verbatim** (`sase_pre_blog`, not "sase pre blog"). Bytes
  never change.
- **Subtitle:** `summary:`, clamped to two lines, with the full text in a tooltip. When
  it is absent the row collapses; there is no nag.
- **Missing `created`:** silent on areas (16/19 lack it).
- **Material:** an ~8% hue wash. With no hue (grey, or no resolvable home) the card uses
  neutral paper.

### 3. The Project Card

```text
╭─────────────────────────────────────────────────────────────────────╮
│ ◎  PROJECT · dev › sase                          ROOT   ⊞ Settings  │
│    sase_blog                                                        │
│    ● WIP                                                            │  lifecycle line
│    #WRITE                                        0/5 ready · 5 open │
│ ─────────────────────────────────────────────────────────────────── │
│ ▸ Properties · 7                                     created Jun 11 │
╰─────────────────────────────────────────────────────────────────────╯
- [ ] #task #prj My SASE post, in my own voice, is live on … #hide ^prj   ← goal line, decorated

│ ◎  PROJECT · dev › sase                                 ⊞ Settings  │
│    sase_post_blog                                                   │
│    ⛓ BLOCKED · waits on sase_blog                                   │
│                                                 0/5 ready · 20 open │

│ ◎  PROJECT · dev › bob                                  ⊞ Settings  │  muted, ~2% wash
│    bob_command_mode_improvements                                    │
│    ✓ DONE · Oct 31                                                  │
│ ▸ Properties · 7                                      created Oct 2 │
```

- **Kicker:** `PROJECT` plus the crumb. The `ROOT` pill appears for `root: true`
  (tooltip: "Pomodoro time here reports under this project").
- **Title:** the same rule as areas. Because 100/102 projects begin with the `^prj`
  line, nearly every project title is the stem.
- **Lifecycle line** (the subtitle slot). It is never the criterion: the `^prj` line
  sits ~12 px below and can never hide, so a criterion subtitle would duplicate it.

  | State                                                               | Painted from local data                                                | The report adds (`card-live`)                                                                                                                                                                   |
  | ------------------------------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `● WIP`                                                             | `^prj` mark `[ ]`, `[*]`, or `[/]`                                     | —                                                                                                                                                                                               |
  | `◷ SCHEDULED · starts Dec 2 · in 10 days`                           | mark `[?]` plus a future inline `[scheduled::]`                        | —                                                                                                                                                                                               |
  | `⛓ BLOCKED`                                                        | mark `[?]` with no future schedule                                     | `· waits on [[a]], [[b]] +N` (at most 2 links) or `· parent project [[sase]] is blocked` (`blocked.kind: inherited`; the card finds the blocked ancestor by walking the report's `parent` rows) |
  | `✓ DONE · Oct 31` / `✕ CANCELED · Oct 9`                            | mark `[x]` / `[-]`, plus the completion date when the line carries one | `closed_on` when the line has no date                                                                                                                                                           |
  | terminal, `^prj` archived                                           | no `^prj` line and `status: done`/`canceled`                           | `closed_on`. It keeps its terminal identity; there is no manufactured error.                                                                                                                    |
  | `⚠ NO ^prj TASK · add "- [ ] #task #prj … ^prj" as the first line` | an active project with no recognized `^prj`                            | —                                                                                                                                                                                               |

  State glyphs are Lucide icons followed by a word, never emoji alone. Blocked links
  navigate like crumbs.

- **Missing `created`:** a faint `no created date`, like album and record.
- **Material:** an ~5% hue wash. Terminal projects get ~2% with muted ink, the pennant
  outline, and no chip row.
- **The `^prj` line stays the project's control**
  (`decisions:projects-block-from-prj-line`). The card never hides, moves, docks, or
  re-renders it. `card-live` adds one decoration: the recognized `^prj` line (the first
  body task line before any H2) renders slightly larger and medium-weight, using a
  non-layout effect only, so nothing shifts.

### 4. The chip row

The left side holds **authored policy chips**: keys the note itself declares, in this
fixed order, each shown only when present. They paint from confirmed source bytes.

| Chip            | From                            | Example                                                                                                                                                                                                                           |
| --------------- | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Default type    | `task_type` (or legacy `theme`) | `#BUILD` in the type's hue ink; `task_type: off` reads `type off`                                                                                                                                                                 |
| Clock           | `offclock`, `offclock_types`    | `off-clock`, `on-clock` (an explicit `false`), `off-clock · MEET, CALL`. A shadowed pair (both keys) adds an amber `⚠` whose tooltip reads "offclock: true is ignored because offclock_types decides — ⊞ Settings cleans it up". |
| Review interval | `task_refresh`                  | `review 3d`                                                                                                                                                                                                                       |
| Capture names   | `project_name_aliases`          | `aka tailnet` (the first one, then `+N`)                                                                                                                                                                                          |

The right side holds **readouts**, in muted tabular numerals:

- `2/5 ready` (always): from the synchronous, local `api.noteReady`. It uses the plan
  block's existing room/full/crowded ink, and `ready_cap: off` reads
  `14 ready · no cap`. The tooltip says what it counts and where the cap comes from:
  "Ready tasks living in this note: 2 of 5 · cap from plan.max_ready_per_note"
  (`decisions:note-ready-cap-counts-the-lane`).
- `N open` (projects) and `N today` (both): added by `card-live` from the cached reports
  (`tasks.open`, `tasks.today`, areas `today.direct`). A missing snapshot leaves them
  absent, never zero.

> [!decision] card_structure_counts The readouts also show `N sub-areas` / `N projects`
> (areas, from `children`/`projects.direct`) or `N sub-projects` (projects, from
> `children`), placed after the open/today readouts.

> [!decision] card_structure_counts = no The card never counts sub-areas or
> sub-projects; the catalogs below own structure.

The card **never** shows catalog rows, per-status lane counts (the `## Tasks` badge row
owns them), progress bars or percentages, Today task lists, Stats charts, inherited
values (the editor's ghost rows teach those), or defaults.

Interaction: when the `=+` API is present (feature-detected), each policy chip is a
button that opens `=+` on its setting. Otherwise chips are plain text. Readouts are not
interactive; they carry tooltips only. When the row has no chips and no readouts, it
collapses.

### 5. Foot, drawer, collapse, and mount rules

- **Foot row:** `bob-cli-pf`'s, unchanged. Only the area `created` rule (§2) softens it.
- **Where-marks** (the adapter declarations that the drawer renders):
  - **Area:** `type`→tile, `parent`→kicker, `summary`→subtitle, `color`→tint,
    `task_type`/`theme`/`offclock`/`offclock_types`/`task_refresh`/`project_name_aliases`→chip,
    `ready_cap`→readout, `created`→foot. On bank.md (from `takeover` on), `bank`→card
    (the nested `==` row).
  - **Project:** the same, plus `root`→kicker and `status`→state.
  - **Managed fold:** `done_tasks` (rendered as a working link to the archive),
    `template`, a leftover frontmatter `scheduled`, and any lingering `task_count` /
    `open_task_count`.
- **Collapse:** the kit's per-note session fold, the same affordance as the record card.
  Collapsed, the card keeps the kicker, the title, the project lifecycle line, and the
  foot, and drops the area summary and the chip row.
- **Mount rules:** typed area and project notes only, through the existing classifiers.
  Skip:
  - the untyped `area.md` and `project.md` type notes;
  - `_templates/`, `done/`, and `_conflicts/`;
  - embeds, popovers, canvas, print, mobile, and Source mode;
  - bank.md until `takeover`.

  The cards mount in desktop Live Preview and Reading. A failed mount restores the
  native title, H1, and Properties in the same frame.

- **Painting** reads only the metadata cache, confirmed source bytes, the local JS home
  resolver (for hue), and `api.noteReady`. It makes zero bob spawns and zero writes.
  Report-backed parts arrive later as in-place patches from the _already shared_ areas
  and projects data services (`241aa`/`241ab`, `241af`/`241ag`). The card subscribes to
  them and never starts a per-card or per-paint fetch.

### 6. The settings editor: `=+` on area and project notes

There is no second modal. `=+` _is_ the schema editor on these notes. `==` on an area or
project note opens the same `=+` modal with focus on the first settings row. On bank.md,
`==` keeps opening the bank editor (registry order: bank first). The pf chassis,
grammar, write discipline, undo, YAML excursion, and close ladder
(`decisions:q-and-ctrl-bracket-mirror-esc`) carry over unchanged.

```text
╭─ ▤  AREA · PROPERTIES ──────────────────────────────────── / filter ─╮
│ love                                          love.md · 8 properties │
│                                                                      │
│ AREA SETTINGS                                                        │
│ ▸ Summary        summary        Family and relationships: life …  set│
│   Lives under    parent         [[area]] · top level              set│
│   Hue            color          ● yellow                          set│
│   Default type   task_type      #CHORE                            set│
│   Clock          offclock …     off-clock for MEET   ⚠ shadowed   set│
│   Ready cap      ready_cap      5 · Bob setting (max_ready_per_note) │  ← ghost row
│   Review         task_refresh   7 days · Bob setting                 │  ← ghost row
│   Capture names  project_na…    none — capture uses the stem         │  ← ghost row
│                                                                      │
│ PROPERTIES                                                           │
│   created …                                                          │
│ IDENTITY                                                             │
│   type           "[[area]]"                        locked · y YAML   │
│ ▸ BOB-MANAGED · 1                                                    │
│──────────────────────────────────────────────────────────────────────│
│ ⚠ offclock: true is ignored — Enter on Clock cleans it up            │
│ ↵ edit · ^D inherit · o add · y YAML · u undo · ⋯ All actions        │
╰──────────────────────────────────────────────────────────────────────╯
```

**Groups, in order:**

1. **`AREA SETTINGS` / `PROJECT SETTINGS`**, in contract order (stable across notes;
   reorder is disabled here). Each row shows a human label, the YAML key(s) in faint
   mono, the value, and a route mark (`set`, or nothing for ghosts).
   - Area order: Summary, Lives under, Hue, Default type, Clock, Ready cap, Review,
     Capture names.
   - Project order: Lives under, Hue, Default type, Clock, Stats root, Ready cap,
     Review, Capture names.
2. **`ON THE ^prj LINE`** (projects only). Three read-only virtual rows: Goal (the
   cleaned criterion), Starts (inline `[scheduled::]`, or "not scheduled"), and Depends
   on (the Depends-On targets). Enter opens the Task Card on `^prj`; when that API is
   missing, it places the cursor on the line. The editor never writes a frontmatter
   `scheduled`, `due`, `start`, or `deadline`
   (`decisions:task-deps-are-depends-on-links`).
3. **Loose properties**, in file order: everything off-contract (`created`, `aliases`,
   `tags`, `cssclasses`, `priority`, unknown keys), with full pf generic access. Nothing
   is hidden for being off-contract.
4. **Identity · lifecycle**, locked:
   - `type`: "Defines the note's kind; change it in YAML (y)."
   - project `status`: "Mirrors the ^prj checkbox — close, cancel, or reopen ^prj."
     Enter jumps to `^prj` (navigation only, no write).
5. **▸ Bob-managed · N**, a closed fold:
   - `done_tasks` (derived; "Pairs this note with its done/ archive — managed by bob
     task archive");
   - `template` (loose provenance);
   - a leftover `scheduled` (derived; "Belongs on the ^prj line; bob projects sync moves
     it for open projects");
   - retired counts (loose, legacy; "Retired count — nothing maintains it; Ctrl+D
     removes it").
   - On bank.md, the `bank` summary row sits with the card keys and hands off to `==`.

**Ghost rows.** An undeclared setting shows a faint, italic ghost: the _effective_ value
and its source, computed by bob and never stored. Enter on a ghost sets an override.
Ctrl+D on a declared setting **clears it to inherit**: one undoable `setting.clear`,
whose dry-run teaching line names what will then apply ("Clears color — this note will
use ● blue from dev"). Ctrl+D on a ghost does nothing except say "Already using the
inherited value." Precedence copy must be exact:

- `ready_cap` and `task_refresh` apply **by residence** and never inherit. Their inherit
  choice reads "Use Bob setting (5)", never "Inherit from parent".
- `task_type: off` is "Stop parent defaults", not "untyped". A task's own field, a
  reading rule, or the global default can still decide.
- `grey` is a deliberate mute, distinct from Inherit.

**The field contract.** bob owns it and its validation; the editor renders what
`bob props show` reports.

| Setting       | Key(s)                        | Kinds   | Stage                                     | Validation (bob)                                                                                                                                                                                                          | Ghost when undeclared                                                       |
| ------------- | ----------------------------- | ------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Summary       | `summary`                     | area    | one-line text with a live `N/140` counter | multiline refused (`area_summary_multiline`); over 140 warns, never refuses                                                                                                                                               | `no summary — catalogs show the link only`                                  |
| Lives under   | `parent`                      | both    | restricted home picker (`editor-parent`)  | areas: the root `area` or an area; projects: an area or a non-terminal project. Self, cycles, descendants, `done/`, missing, and ambiguous stems are refused; clearing is refused. Writes one quoted scalar `"[[stem]]"`. | never a ghost (required); a missing parent shows `⚠ no parent`             |
| Hue           | `color`                       | both    | swatches (`editor-policy-stages`)         | the 8 palette hues, `grey`/`gray`; anything else refused with the allowed list                                                                                                                                            | `● blue · from dev` / `● aqua · from the name (nothing sets a color)`       |
| Default type  | `task_type`, legacy `theme`   | both    | type picker (`editor-policy-stages`)      | a scalar type name or `off`; valid-but-unconfigured warns; saving on a `theme:` note writes `task_type` and unsets `theme` in one batch                                                                                   | `#BUILD · from dev` / `none · nothing above sets one`                       |
| Clock         | `offclock` + `offclock_types` | both    | clock policy (`editor-policy-stages`)     | four modes with exact postimages (below); an empty list is refused                                                                                                                                                        | `off-clock for MEET · from love` / `on-clock · nothing above sets a policy` |
| Stats root    | `root`                        | project | toggle (no stage)                         | `true` or absent, never `root: false`; disabled on top-level projects                                                                                                                                                     | `reports under sase` / disabled: `already a root (top-level project)`       |
| Ready cap     | `ready_cap`                   | both    | count-or-off                              | 1–999 or `off`; legacy YAML `false` reads as off                                                                                                                                                                          | `5 · Bob setting (plan.max_ready_per_note)`                                 |
| Review        | `task_refresh`                | both    | days                                      | 1–365                                                                                                                                                                                                                     | `N days · Bob setting` (naming bob's freshness config key)                  |
| Capture names | `project_name_aliases`        | both    | list                                      | blank and in-list duplicate entries refused; cross-note collisions and stem shadows warn (capture-targets rules; the exact stem always wins)                                                                              | `none — capture uses the stem`                                              |

**Clock-policy postimages** (one atomic, jointly undoable batch):

- Inherit removes both keys.
- On-clock writes `offclock: false` and removes the list.
- Off-clock writes `offclock: true` and removes the list.
- Off for types… writes a nonempty canonical block list and removes the boolean.

Opening the stage never silently cleans a dual-key note. A shadowed pair shows the
warning and a one-Enter cleanup preview.

**Stage behaviors worth naming:**

- **Hue** swatches: Inherit, 8 hues, grey, with letter accelerators. Each swatch is
  labelled with the top-level areas already using it ("used by dev"). While you browse,
  the modal header's paper tints live; Esc reverts it with no write.
- **Default type** picker: configured types in their hue ink, plus `off` ("Stop parent
  defaults") and Inherit.
- **Ready cap:** a live effect line, "6 ready · cap 8 → room for 2".
- **Review:** the scope line "Applies to tasks living in this note only".
- **Stats root:** flips on Enter (a discrete, undoable op). The consequence line says
  that time here (past and future) reports under this project, and that it stops being a
  leaf project, so keep-moving successor offers change.
- **Lives under:** shows bob's dry-run consequences before Enter can apply
  ([§7](#7-bob-owns-the-contract)). After apply, the session runs exactly the catalog
  syncs bob named: "Refreshing catalogs…", then "✓ Moved under dev · 3 catalogs
  refreshed · u undo". Undo restores the parent and re-runs the same syncs.

**Degraded modes:** with a bob that lacks `props_contract_v1`, settings render as plain
pf loose rows and the status line teaches "Update bob for typed settings". With a bob
that lacks `props_v1`, pf's read-only mode applies.

### 7. bob owns the contract

There is one writer (`bob props update`), one undo protocol, and one validator. The
typed stages, `=+` generic ops, paste, YAML fragments, and terminal use all pass through
it, so an invalid value cannot be written through any entry point.

- **Roles** (`src/native/props/roles.rs`):

  | Key                             | Role                                                                                                                |
  | ------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
  | project `status`                | derived (owner `projects`)                                                                                          |
  | `done_tasks` (both kinds)       | derived (owner `task archive`)                                                                                      |
  | leftover project `scheduled`    | derived (owner `projects`, legacy)                                                                                  |
  | `template`                      | loose, folded provenance                                                                                            |
  | `task_count`, `open_task_count` | loose, `legacy: true`                                                                                               |
  | `type`                          | identity (unchanged)                                                                                                |
  | contract keys                   | loose, with a `setting` annotation and domain validation, never `card` (that would make `=+` refuse its own writes) |

  **bank.md is a hybrid.** `note_kind` stays `bank`, `bank` stays card-owned, and its
  other keys get the area contract. Today the Bank arm returns before the area check, so
  area validation keyed only on `NoteKind::Area` would silently miss it.

- **Validation** applies to `prop.set`, `prop.set_yaml`, and `prop.rename` (onto a
  contract key) using bob's existing parsers: `NoteColor`/`Hue`, the note-theme and
  type-config parsing, `parse_offclock_types`, the note_ready cap parse, freshness
  ranges, the areas/projects parent rules, and the capture-targets alias checks.
  - `prop.unset` of a contract key is "clear to inherit" and allowed, except `parent`
    (`prop_required`).
  - `prop.restore` stays byte-exact and unvalidated, so undo always works.
  - Refusals reuse the typed shape:
    - `prop_bad_value` (its hint lists the allowed values);
    - `prop_parent_kind`, `prop_parent_cycle`, and `prop_parent_missing`;
    - `prop_required` and `prop_setting_disabled`.

    Warnings ride a `warnings[]` array on dry-runs and applies.

- **Setting ops:** `setting.set {setting, value}` and `setting.clear {setting}`. bob
  expands them into ordinary span ops, so the clock precedence, the `theme`→`task_type`
  migration, and shadowed cleanup live in exactly one place. Each batch is one
  fingerprinted, byte-preserving write, and its `inverse_ops` are `prop.restore` ops.
- **Show JSON:** a new `contract` block in `bob props show -f json`, present on area,
  project, and bank.md:
  - `kind`;
  - `settings[]` in contract order, each with `id`, `label`, `keys`, `stage`,
    `declared`, `value`,
    `effective {value, source: self|inherited|config|hash|none, from, label}`, `choices`
    (hues with `used_by` top-level areas; configured types with hue), `issues[]` (bob's
    existing finding codes), `help`, and `disabled`;
  - `prj` (projects): `line`, `criterion`, `scheduled`, `depends_on[]`, and `blocks[]`.

  A new capability `props_contract_v1` joins `capabilities`.

- **Dry-run effects:** `effects[]` lines on setting writes:
  - **Parent:** hue from→to, root parent from→to ("root parents follow the current tree,
    so past time regroups too"), inherited type and clock from→to, blocking gained or
    lost, and `syncs[]` (which of `areas`, `projects`, `albums`, `notebooks` catalogs
    change).
  - **Inherited settings** (`color`, `task_type`, clock): "Applies to N notes below that
    don't set their own".
  - **Residence-only settings:** their scope line.
- **List issues:** a per-row `contract_issues[]` in `bob areas list -f json` and
  `bob projects list -f json`, so the foot lint can include bob's findings
  (`note_offclock_shadowed`, `area_color_invalid`, …) without a spawn.

### 8. Takeover and bank.md

`takeover` wires `bob-cli-pf`'s `propertyCardOwnFrontmatter` into both cards under pf's
eligibility rules, plus `props_contract_v1`.

- **bank.md:** after a spike, the Area Card mounts in its title slot, adopts
  `# 🏦 Bank`, and owns the single foot row and the takeover. The bank card stays in the
  body with its banking content, `⊞ Edit`, and species badge, and suppresses its
  temporary `bank_foot` only while the Area owner is active. If the Area mount fails,
  `bank_foot` comes back. `==` keeps opening the bank editor, and `=+` gains the area
  settings rows.
- **Native entry points never dead-end:**
  - Mod+; is wrapped (as in pf);
  - the areas catalog's "add summary" affordance (`241ai:239`, `data-focus-key=summary`)
    and every other native-focus helper open `=+` on that setting;
  - the File Properties sidebar is never touched.

### 9. Visual language

`bob-cli-pf` §7 applies unchanged:

- no new colors, and amber only for lint;
- tabular numerals;
- `prefers-reduced-motion` honored (collapse animates `grid-template-rows` over 160 ms,
  with no motion under reduced motion);
- `forced-colors` support and 4.5:1 contrast;
- a ~420 px container query.

Card specifics:

- Chips are hairline pills in the meta ink, at 0.82 em.
- The state word uses tracked small caps in its state ink; the detail clause is muted
  text.
- Wash weights are area 8%, project 5%, and terminal 2%.
- Below ~420 px:
  - readouts drop under the chips;
  - crumbs ellipsize before the title ever wraps;
  - `⊞ Settings` collapses to its glyph with an aria-label.

Every painting phase verifies light and dark themes and narrow and wide panes with
real-Obsidian screenshots.

## Phase `one-writer`: retire the second frontmatter writers on project notes (bob-plugins + vault)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Then read the `obsidian.md`
reference memory with `/sase_memory_read` before touching the vault.

1. **bob-project-tasks** (hand-edited `main.js`, no fragments): delete the count writer
   completely: `refreshFile`, both `processFrontMatter` calls (`main.js:561,574`), the
   metadata-changed listener and debounce that drive it, `countProjectTasks`, and the
   **Recount all project tasks** command. Keep the two Highlights snapshot commands
   byte-for-byte. Update `manifest.json`'s description ("Snapshot notes to Highlights")
   and bump its version per repo convention.
2. **Project creation** (`bob-navigation-hotkeys`): replace the `processFrontMatter`
   call in `660-plugin-project-notes.js` with the existing byte-preserving
   `planParentFrontmatterEdit` (`211-parent-frontmatter-plan.js`) applied through the
   same write path that `682-plugin-parent-change.js` uses. The template owns `type` and
   `status`. If the created note's cache lacks `type: "[[project]]"`, show a Notice; do
   not write `type`/`status` from JS. Retire `applyProjectCreationFrontmatter`, or keep
   it test-only if its tests still describe the template contract. Add a test that the
   created note's bytes equal the template's output except for the `parent` line.
3. **Audit:** grep every plugin for frontmatter writes that can touch area or project
   notes (`processFrontMatter`, `metadataEditor`, `vault.process`/`modify` on
   frontmatter spans). Fix or list each one. After this phase, `processFrontMatter` must
   not run on area or project notes anywhere.
4. **Retired-key sweep:** add `scripts/retire-project-count-keys.cjs`. It is dry-run by
   default, with `--apply`. For every Markdown note that has `task_count` or
   `open_task_count` (including `done/`), it runs one
   `bob props update NOTE -f json -i -` with two `prop.unset` ops and the note's current
   fingerprint as `expect`, then prints a per-note table and a total. It skips and
   reports stale or refused notes, so re-runs are idempotent. Use a bob built from
   current bob-cli master (`BOB_BIN`) if the installed one lacks `props_v1`. Run it with
   `--dry-run`, then `--apply`. Verify with `git -C ~/bob diff --stat` that only those
   two lines changed per note. vault-sync owns the commit.
5. **Vault Bases and type note** (byte-careful hand edits):
   - `projects.base`: drop the `tasks` formula and its column from every view. Rename
     `🚀 Active & Waiting` to `🚀 Active` and filter it on `status == "wip"`. Delete the
     `⏳ Waiting` view and the `waiting` branch of `status_badge`.
   - `project.md`: the status line becomes "`status` mirrors the `^prj` checkbox
     (`bob projects sync`); edit settings with `=+`", and the machine-maintained counts
     line goes. Mention that `status: waiting` is retired.
   - bob's display-only handling of `waiting` (`capture_targets.rs:841`) stays as is.
6. **Docs:**
   - bob-plugins `docs/bob-project-tasks.md`: retire the counts section and say why (one
     writer; bob's live counts appear on the Project Card).
   - bob-plugins `docs/bob-navigation-hotkeys.md` if it describes creation.
   - bob-cli `docs/projects.md` line ~220: drop `waiting` from the status story.
   - Run `npm run docs:check` and `just docs-check`.
7. **Finish:** `npm test`, then `bob plugins sync`. In the land report, remind Bryan
   that each other host (athena, the Mac) must run `bob plugins sync`. Until it does, a
   stale host can re-add the counts. They are harmless, they show as legacy rows, and
   the script is safe to re-run.

Acceptance: no `processFrontMatter` call remains on area/project notes; no note in the
vault has `task_count`/`open_task_count`; `projects.base` renders; project creation
still sets the parent with all other template bytes intact.

## Phase `contract-writes`: area/project roles, validation, and setting ops in bob props (bob-cli)

Read the `cli_rules.md` and `docs.md` reference memory first. Releases ride the daily
train (`decisions:bob-cli-releases-ride-a-daily-train`): never edit the Cargo version or
CHANGELOG, and use `feat(props):` commits. This is additive, so there is no `!`.

- **Roles** per [§7](#7-bob-owns-the-contract), in `src/native/props/roles.rs`. Teaching
  reasons use the exact copy from
  [§6](#6-the-settings-editor--on-area-and-project-notes). Add the `legacy` flag to the
  property JSON (additive). Make bank.md a hybrid: keep `note_kind: bank`, and evaluate
  the area contract for its non-`bank` keys.
- **A contract module** (e.g.
  `src/native/props/contract/{mod,settings,validate,expand}.rs`, each focused): a static
  table of settings per kind (id, label, keys, stage, kinds, order). Validators wrap
  bob's existing parsers, which must not be re-implemented: `pomodoro_homes::facts`
  (`NoteColor`, `NoteTheme`, `NoteOffClock*`), `projects::scan` (`parse_offclock_types`,
  the `ready_cap` parse), `freshness::scan` (`task_refresh`), `note_ready::scan` (cap
  source), the areas and projects parent rules (`areas/graph.rs` lints), and
  `capture_targets` (alias collisions). Parent validation resolves the target note
  through bob's existing link resolution and refuses:
  - self;
  - cycles and descendants;
  - targets in `done/` or `_conflicts/`;
  - missing or ambiguous stems;
  - the wrong kind (areas: the root `area` or an area; projects: an area or a
    non-terminal project).
- **Write paths:** validate contract keys in `prop.set`, `prop.set_yaml`, and
  `prop.rename` (onto a contract key). `prop.unset` follows the parent rule.
  `prop.restore` stays unvalidated.
- **Setting ops:** `setting.set` and `setting.clear`, strictly parsed like the other ops
  (unknown setting ids and wrong value shapes are typed refusals). Expansion covers:
  - the clock modes, with exact postimages;
  - the `theme`→`task_type` migration (set `task_type`, unset `theme`);
  - shadowed-pair cleanup;
  - canonical value spelling (lowercase hue, `grey`, block-list `offclock_types` with
    canonical type names, `off` for a disabled cap);
  - `root: true` set, or unset on clear;
  - `setting.set root` on a top-level project refuses with `prop_setting_disabled`.
- **Tests:**
  - Golden byte fixtures for every expansion: clock postimages from each of the four
    starting states, including the dual-key shadowed notes shaped like `love.md` and
    `body.md`; theme migration; quoted and unquoted parent links; CRLF; comments beside
    contract keys.
  - Inverse-op round trips that are byte-identical.
  - The bank.md hybrid on a fixture shaped like the real `bank.md`.
  - Each refusal code.
  - Role rows on fixtures shaped like `sase_blog.md` (legacy `theme`, `root`, counts)
    and `mac_inbox.md`.
- **Docs:** extend `docs/props.md` with the roles, the setting ops, the refusal codes,
  and the bank hybrid. Run `just docs-gen` and `just docs-check`.

Acceptance: `just` gates pass. Every contract key refuses invalid values through every
op. `setting.*` ops write byte-preservingly and undo byte-exactly. `bob props show` on
real area, project, and bank notes reports the new roles.

## Phase `contract-reads`: the settings contract in bob props show, dry-run effects, and list issues (bob-cli)

The same rules as `contract-writes` apply.

- **The `contract` block** in `show -f json` per [§7](#7-bob-owns-the-contract). Compute
  each effective value and its source with bob's own resolvers, so ghost rows can never
  drift from Rust's precedence rules:
  - `HomeResolver` for hue and `hue_source`;
  - the `TypeRules` effective-type walk;
  - `effective_clock` and `ClockSource`;
  - the note_ready cap source;
  - the freshness config fallback.

  `choices`: hues with the top-level areas using each (`used_by`), and configured types
  with their hue. `prj`: the `^prj` line facts from the projects scan. Text output of
  `show` gains a compact colored "Settings" section.

- **`effects[]`** on dry-run and apply responses for setting writes, per §7. Name the
  catalog syncs precisely: compute which catalog owners gain or lose a row, rather than
  always listing all four.
- **`contract_issues[]`** per row in `bob areas list -f json` and
  `bob projects list -f json` (additive). Keep the existing `findings`/`issues`
  unchanged.
- **Capability:** add `props_contract_v1` once the block and the effects are complete.
- **Tests:**
  - snapshot JSON for an area with inherited hue, an area with a shadowed clock, an
    inbox with `ready_cap: off`, a root project, a blocked sub-project, a terminal
    project, and bank.md;
  - effects for re-homing a project between areas;
  - list `contract_issues`.
- **Docs:** `docs/props.md` (the contract JSON schema and the effects) and cross-links
  from `docs/areas.md` and `docs/projects.md`. Update `docs/reference/json-outputs.md`
  if it lists these schemas. Run `just docs-gen` and `just docs-check`.

Acceptance: `bob props show -f json` on every live area and project note reports a
contract whose effective values match the existing CLI reports (`bob areas list` hue,
`bob ready` caps, clock resolution). Parent dry-runs name their consequences.

## Phase `home-cards`: Area Card and Project Card painted from local data (bob-plugins)

Open bob-plugins with `/sase_repo` and read `AGENTS.md`. All work is in
`plugins/bob-ledger-tools`. Native Properties stays visible in this phase; the temporary
duplication is the safe half-state.

- **Species:** add `area` and `project` to `PROPERTY_CARD_KINDS` /
  `PROPERTY_CARD_SPECIES` (`240f`) with their words, glyphs, and the new
  `field`/`pennant` tile forms in `styles.css`.
- **One shared home adapter** (new fragments, each ≤ 1000 lines, modeled on the album
  header `241u`/`241v`/`241va`, e.g. `249a-…-home-card-model.js`,
  `249b-…-home-card-dom.js`, `249c-…-home-card-host.js`, `249d-…-home-card-foot.js`). It
  uses `createPropertyCardHost`, kit title adoption (`240k`), the title-block DOM
  (`240g`), and the shared foot adapter (`240r`) with the where-marks of §5.
  - **The model is pure and testable:**
    - kind and inbox detection, using the existing classifiers and
      `api.inboxRoute.isInboxNote`;
    - the crumb chain (reuse the notebook header's ancestry model `241e` if it fits);
    - the title rule;
    - the lifecycle state from the `^prj` line's source bytes (§3);
    - the authored chips;
    - the material weight.
  - **Paint per §2–§5**, with hue from the existing JS home resolver
    (`pomodoroHomeResolve`) and the Ready readout from `api.noteReady`.
  - **The species `created` rule:** silent on areas, `no created date` on projects.
- **Interaction:** crumbs and blocked links navigate with modifier-aware panes (the
  album header's `albumHeaderLinkPane` idiom). `⊞ Settings` and the chips call the `=+`
  API when it is present and are inert text otherwise. `editor-settings` makes them
  live.
- **Fail-open:** unit-test that a mount failure restores the native title and H1 in the
  same frame. Keep bank.md, the type notes, templates, `done/`, and `_conflicts/`
  excluded, along with embeds, popovers, canvas, print, mobile, and Source mode.
- **Tests:**
  - **Unit:** title rule (H1 after `^prj` is not a title; alias; stem verbatim), crumb
    (top-level, 4+ ancestors, inbox), each lifecycle row of §3, chip order and presence,
    readout text and ink, where-mark declarations, collapse persistence, and reduced
    motion. List every new test file in `package.json` `test:unit`.
  - **Real-Obsidian e2e:** `scripts/obsidian-home-cards-e2e.cjs`, with an offline
    `--self-test` in `npm test`. Cover a top-level area, a sub-area, an inbox, and open,
    blocked, scheduled, done, and canceled projects, a `root: true` project, a note with
    legacy `theme`, and a project with no `^prj`. Verify `^prj` cursor integrity: the
    cursor on body line 1 still edits `^prj`, and the Task Card and close gestures still
    work.
  - **Screenshots:** light and dark × narrow and wide.
- **Docs:** a new bob-plugins `docs/area-and-project-cards.md` (the anatomy, variants,
  chips, data sources, and mount rules), linked from `docs/nav.yml`, `docs/albums.md`
  §cards, and `docs/develop/architecture.md` (the species list). Run
  `npm run docs:check`.
- **Finish:** `npm test`, then `bob plugins sync`.

Acceptance: every eligible area and project note opens with a card of at most five lines
that matches §2–§5, with zero bob spawns and zero writes while painting; the `^prj` line
behaves exactly as before.

## Phase `card-live`: report-backed lifecycle reasons, readouts, lint, and the ^prj goal line (bob-plugins)

- **Frame-2 patch:** subscribe the home adapter to the shared projects and areas data
  services and patch in place (`propertyCardAdoptPatch` or the as-landed equivalent):
  - blocked reasons and inherited-blocking ancestors;
  - `closed_on` when the line lacks a date;
  - the `N open` / `N today` readouts;
  - the `card_structure_counts` readouts when that decision is yes.

  A missing or old snapshot hides only those parts.

- **Foot lint:** add the row's `contract_issues` to the kit lint chip's count and
  tooltip, feature-detected. When the `=+` API exists, clicking the chip opens `=+` on
  the first affected setting. It never opens the drawer by itself.
- **The `^prj` goal line:** a lazily registered CodeMirror line decoration plus a
  Reading-view class on the recognized `^prj` line (the first body task line before any
  H2, and only while a project card is mounted). It is slightly larger and medium
  weight, with no layout shift, and it is removed on unmount or Source mode.
- **Tests:**
  - unit tests for patch merging, absent-not-zero, and the ancestor walk;
  - e2e extensions for a blocked sub-project under a blocked parent, a
    dependency-blocked project, a done project without a line date, the lint chip with a
    shadowed clock, and the decorated `^prj` line with cursor and IME integrity.
- **Docs:** update `docs/area-and-project-cards.md`. Finish with `npm test` and
  `bob plugins sync`.

## Phase `editor-settings`: the settings group in =+, ghost rows, simple stages, and == registration (bob-plugins)

Extend the as-landed `=+` editor through a small per-kind extension seam (for example,
row providers that contribute virtual rows and groups). Do not fork its modal, session,
or grammar.

- **Groups and order** per §6, from the `contract` block. Disable reorder, rename, and
  kind change on settings and virtual rows, with teaching lines. Paste and copy work
  through bob's validation.
- **Ghost rows, Enter-to-override, and Ctrl+D clear-to-inherit** through `setting.set` /
  `setting.clear`, with the dry-run teaching line and inverse-op undo. Render the
  `effects` lines in the preview area.
- **Stages:**
  - Summary (one line with the `N/140` counter);
  - Ready cap (stepper, `o` toggles off, the live effect line);
  - Review (days with the scope line);
  - Capture names (the pf List stage with bob warnings inline);
  - Stats root (a discrete toggle with the consequence line, disabled on top-level
    projects).

  Hue, Default type, and Clock open the generic pf stage until `editor-policy-stages`
  lands. Lives under opens pf's Link stage until `editor-parent` lands, and bob's
  validation guards both.

- **`^prj` handoff rows** (Enter opens the Task Card on `^prj`), the locked identity and
  lifecycle rows, and the Bob-managed fold, per §6.
- **`==` registration:** register `area` and `project` in the `==` species registry
  after bank, album, and record, opening `=+` focused on the first settings row. Make
  `⊞ Settings` and the policy chips (from `home-cards`) call the same API with the
  setting id.
- **Degraded mode** without `props_contract_v1` (§6).
- **Tests:**
  - **Unit:** the group model from fixture contract JSON, ghost and declared rendering,
    the key-action table on settings rows, op building for every simple stage, and the
    degraded mode.
  - **e2e:** extend the property-editor e2e with a `BOB_BIN` built from current bob-cli
    master. Cover:
    - `==` and `⊞ Settings` landing on the settings group;
    - setting and clearing a ready cap with undo;
    - a refused invalid value showing bob's message;
    - a root toggle;
    - a locked `status` row jumping to `^prj`;
    - byte-identical untouched keys on disk;
    - light and dark screenshots.
- **Docs:** add a "Settings on area and project notes" section to
  `docs/property-editor.md`, and update the hotkey references. Finish with `npm test`
  and `bob plugins sync`.

## Phase `editor-policy-stages`: hue, default type, and clock-policy stages (bob-plugins)

- **Hue:** a swatch row (Inherit, 8 hues, grey) with letter accelerators and `used_by`
  labels. The live header tint previews the hue while browsing, Esc reverts with no
  write, and Enter applies `setting.set color`.
- **Default type:** a filterable picker over `choices` in their hue ink, plus `off`
  ("Stop parent defaults") and Inherit. An unconfigured name shows bob's warning. On a
  `theme:` note, the preview names the migration.
- **Clock policy:** a segmented control for the four modes. "Off for types…" opens a
  type checklist (Space toggles), and a live sentence says what will apply. A shadowed
  pair shows the warning and the one-Enter cleanup preview. bob expands every mode.
- **Tests:** unit tests for stage models and accelerators; e2e on `love`-shaped and
  `body`-shaped fixtures (shadowed cleanup), a legacy `theme` project, and a sub-area
  hue override with undo, plus screenshots of each stage.
- **Docs:** update `docs/property-editor.md`. Finish with `npm test` and
  `bob plugins sync`.

## Phase `editor-parent`: the Lives under stage with consequence preview and catalog refresh (bob-plugins)

- **Picker:** reuse the nav-hotkeys parent picker model
  (`208-parent-note-picker-model.js`: ranking, kind weights, `parentChangeKindRefusal`,
  `parentChangeWouldCycle`, ancestry). Expose a choose-only API from
  bob-navigation-hotkeys (e.g. `api.parentPicker.choose`, versioned in
  `docs/develop/plugin-apis.md`) that returns the pick without writing. Do not copy the
  model. Candidates show kind and hue pips, and ineligible homes show the refusal text.
- **Preview and apply:** a dry-run `setting.set parent` renders bob's `effects`, and
  Enter applies on a current preview. Then, inside the same guarded session, the stage
  runs exactly the `syncs[]` bob named (`bob areas sync`, `bob projects sync`,
  `bob albums sync`, `bob notebooks sync` through the existing data services' sync
  paths) and reports the outcome (§6). A sync failure keeps the write, shows the error,
  and leaves the catalogs' own stale-sync affordance as the backstop. Undo restores the
  parent and re-runs the same syncs.
- **Unchanged:** the existing parent-change gesture (`682`) keeps its byte-preserving
  writer.
- **Tests:** unit tests for the choose API and the preview model; e2e re-homing a
  project between two areas (hue change on the card, both catalogs refreshed, undo),
  with refused cycle and wrong-kind picks.
- **Docs:** update `docs/property-editor.md` and `docs/bob-navigation-hotkeys.md`.
  Finish with `npm test` and `bob plugins sync`.

## Phase `takeover`: hide native Properties on area/project notes and arbitrate bank.md (bob-plugins + memory)

- **Ownership:** wire `propertyCardOwnFrontmatter` into the home adapter after a
  successful mount, under pf's eligibility rules plus a cached, off-paint-path
  `props_contract_v1` probe. Until that is confirmed, per-row hiding stays as the
  fallback.
- **Re-run the retired-key sweep** in dry-run first. If counts came back, a host is
  stale: name it in the land report and apply the sweep again.
- **bank.md spike, then mount:**
  1. Probe whether the bank card's anchor under `# 🏦 Bank` survives Area-card title
     adoption.
  2. If it does, mount per §8: the Area Card owns the title, foot, and takeover, and the
     bank card suppresses its `bank_foot` while the Area owner is active and restores it
     otherwise.
  3. If it does not, mount the Area Card without title ownership (the kit fallback,
     keeping the H1 visible).
  4. If even that conflicts, keep bank.md excluded, add the area crumb and summary to
     the bank card header instead, and record why.

  Update `docs/bank.md`.

- **Entry points:** route the areas catalog's "add summary" and any remaining
  native-focus helper on area/project notes to `=+` on the matching setting. Verify that
  Mod+; opens `=+` in add mode.
- **Fail-open matrix:** extend `scripts/obsidian-frontmatter-takeover-e2e.cjs`.
  - **Hidden** in Live Preview and Reading on an area, an inbox, a project, a terminal
    project, and bank.md.
  - **Restored** by Source toggle, an embed, a popover, mobile emulation, bad YAML, an
    unclosed fence, a path change, injected mount failure, plugin unload, and an absent
    `props_contract_v1`.
  - **No focus leak**, and the File Properties sidebar still works.
  - **Invariants:** `area.md` and templates are untouched; `^prj` cursor integrity
    holds.
  - **Screenshots:** a full light/dark × narrow/wide walkthrough.
- **Docs:**
  - `docs/area-and-project-cards.md` (the takeover and bank.md);
  - `docs/property-editor.md`;
  - `docs/develop/architecture.md` (the species list and the bank arbitration);
  - `docs/develop/validation.md`.

  Retire any "native Properties stays visible" wording for these notes.

- **Memory** (only as the decisions allow): in bob-cli, use `/sase_memory_write`, then
  run `sase memory init`.

> [!decision] memory_area_card Add a `glossary` strand (keyword `Area Card`). It
> describes, concisely and as-landed:
>
> - a [[property-card]] mounted as the title block of an [[area-note]];
> - the `layers` tile, the `AREA` kicker with the parent crumb, the title rule (H1 →
>   alias → stem verbatim), the `summary` subtitle, authored policy chips, and the live
>   Ready readout;
> - the `INBOX` pill on an [[inbox-file]];
> - on bank.md, owning the title and foot while the [[bank-card]] stays in the body;
> - that `==`, `⊞ Settings`, or a chip opens `=+` on AREA SETTINGS, where bob validates
>   values and ghost rows show inherited values and their sources;
> - that painting reads local data only and a failed mount restores native chrome.
>
> Link `docs/reference/plugins/area-and-project-cards.md`.

> [!decision] memory_project_card Add a `glossary` strand (keyword `Project Card`). It
> describes:
>
> - a [[property-card]] mounted as the title block of a [[project-note]];
> - the `target` tile, the `PROJECT` kicker with the parent crumb, and the `ROOT` pill;
> - the title, and the lifecycle line derived from the [[project-task]] (WIP, SCHEDULED,
>   BLOCKED with its reason, DONE/CANCELED with a date, or an amber missing-`^prj`
>   warning);
> - authored chips, plus Ready/open/today readouts;
> - that it never shows or edits the criterion, because the `^prj` line stays the
>   project's control directly below;
> - that terminal projects render muted;
> - that `==` opens `=+` on PROJECT SETTINGS, where `status`, `done_tasks`, and the
>   `^prj` dates are read-only.
>
> Link the same docs page.

> [!decision] memory_property_card Rewrite the as-landed `glossary:property-card` strand
> so its species list includes area and project cards (AREA, PROJECT kicker words; the
> `==` registry now ends with area and project before extensions), and so any "bank
> excluded" takeover wording reflects that bank.md's frontmatter is now owned through
> its Area Card. Link [[area-card]] and [[project-card]]. Keep it concise.

Finish with `npm test` and `bob plugins sync`.

Acceptance for the epic:

- Every eligible area and project note opens with a card of at most five lines.
- `=+` and `==` edit every contract setting with a fitting stage, show the effective
  value and its source for undeclared settings, and refuse invalid values with bob's
  message.
- Every write is one byte-preserving `bob props update` with byte-exact undo.
- The `^prj` line behaves exactly as before.
- Painting, the drawer, collapse, and resize issue zero writes and zero spawns.
- Native Properties is gone on eligible notes, bank.md included, and every failure path
  restores it.
- No `processFrontMatter` call touches these notes.

## Cross-cutting rules for every phase

- Start with the [launch precondition](#launch-precondition-and-as-landed-apis).
- Open bob-plugins with `/sase_repo` (it is a linked repo) and read its `AGENTS.md`.
  Finish plugin changes with `npm test` and `bob plugins sync`, and never hand-edit a
  generated `main.js`. Follow `AGENTS.md` if sync refuses an overwrite.
- Phases that can run in parallel (`one-writer`, `contract-writes`, `home-cards`, and
  later `editor-policy-stages` with `editor-parent`) keep shared touch points additive:
  `src/fragments.json`, `package.json` `test:unit`, `styles.css`, the `=+` stage table,
  and lifecycle teardown. Rebase cleanly.
- Every modal and stage follows `decisions:q-and-ctrl-bracket-mirror-esc` through
  `bobCardWireCloseKeys`.
- Every e2e that needs the contract uses a bob built from current bob-cli master via
  `BOB_BIN`. Plugins must behave honestly against a released bob that lacks
  `props_contract_v1` or `props_v1`.
- `processFrontMatter` stays banned, and no phase adds a JS YAML writer.

## Out of scope (file task beads with `/sase_new_task` if still wanted)

- A `@route` capture-name copy affordance on the card (a research nice-to-have).
- Notebook and reference cards that reuse this adapter (a later epic).
- `title:` / `emoji:` keys, a project `summary`, and UI for `status: waiting` or
  `priority`.
- Rerouting the existing parent-change gesture (`682`) through `bob props`.
- Backfilling `created` on areas.
