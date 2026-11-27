---
tier: epic
title: Tree-note link walks
goal: "On area, project, notebook, and prose notes, `[n` / `]n` step through the note's
  same-type siblings under its parent (wrapping), and `[N` / `]N` tour that type's
  parents in outline order. Every Property Card ends with the link-walk rail, below all
  cabinets. Keys, repeats, the links footer, and the rail always agree. Nothing is
  written to the vault, and none of it races the bob-cli-qq cabinets go-live.

  "
phases:
  - id: engine
    title: Sibling-ring engine and the tree route in api.navGroups
    depends_on: []
    size: medium
    description:
      "engine: add the cross-epic gate edges, then build the lazy, generation-stamped
      tree index and the derived `tree` route in Bob Ledger Tools' api.navGroups
      (membership, catalog-order comparators, project lanes, the outline-order parent
      tour, exists-only stops, and explicit none reasons), with parity vectors and a
      perf test."
  - id: nav
    title: Matcher hardening, key notices, and the links footer copy
    depends_on:
      - engine
    size: medium
    description:
      "nav: stop matching step links inside inline code, count authored step links only
      on nav lines on tree routes, pass the tree fields through resolveNavLinks and
      walkMode.links, give `[n`/`]n`/`[N`/`]N` useful notices, and teach the `n`
      walker-footer panel parents, lanes, wraps, and singletons."
  - id: rail
    title: Rail kit polish, the card-floor helper, and the tree rail adapter
    depends_on:
      - engine
    size: medium
    description:
      "rail: add the kit's card-floor helpers, step-item wrap and tooltip support, and
      the pure tree rail model plus its activation, re-validation, and live-refresh
      mixin, all unit-tested and previewed. No card mounts it yet."
  - id: cards
    title: The rail on every tree card and the card-floor invariant
    depends_on:
      - nav
      - rail
    size: medium
    description:
      "cards: after bob-cli-qq.10.5 closes, mount the rail on area, project, notebook,
      and prose cards in both views, make the foot and cabinet stack insert above the
      floor, move the day/period and record rails to the floor, and prove that any rail
      is the card's last child on every species."
  - id: acceptance
    title: Real-Obsidian link-walk acceptance with screenshot review
    depends_on:
      - cards
    size: medium
    description:
      "acceptance: add a scratch-vault runner for all eight keys on all four types,
      check that rail, footer, and keys agree, run the floor, passivity, console-error,
      and perf checks plus re-validation, review every screenshot on athena, and reach
      GO."
  - id: docs
    title: Plugin docs, the link-walk glossary term, and approved memory
    depends_on:
      - acceptance
    size: medium
    description:
      "docs: after bob-cli-qq.10.6 closes, document the tree route, the walk rules, and
      the rail-as-floor contract in the bob-plugins docs. Then write the approved memory
      (the link-walk strand, the walker-footer correction, and any approved extras),
      regenerate, and pass both docs gates."
decisions:
  memory_glossary:
    ask:
      Add the drafted glossary:link-walk strand and point glossary:walker-footer's links
      footer at it?
    memory:
      - glossary:link-walk
      - glossary:walker-footer
    requested:
      As a part of this change we should add an excellent new glossary term for "link
      walk".
    default: true
    answer: true
  memory_decision_record:
    ask:
      Add decisions:tree-notes-walk-sibling-rings recording the derived ring and
      parent-tour design?
    memory:
      - decisions:tree-notes-walk-sibling-rings
    default: false
    answer: true
proposed_by: bbugyi200.apollo.research.6o.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 15:00:00
status: wip
---

- **PROMPT:**
  [prompts/202611/tree_note_link_walks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/tree_note_link_walks.md)

# Tree-note link walks

## Why

Days, periods, and records already walk with `[n` / `]n` and group with `[N` / `]N`.
Area, project, notebook, and prose notes do not. Their `parent:` tree is Bryan's main
way of organizing them, but `]n` there says `No next link found`. This epic makes the
`n` family work on those four note types. It also makes the link-walk rail the floor of
every Property Card, below the whole cabinet stack.

The research report `research:202611/tree_note_link_walks/tree_note_link_walks.md` is
the evidence base (read it with `sase artifact read`). Bryan accepted all of its
recommended requirements. Its sase memory edits are the exception: only the reviewer
decisions above authorize those. This plan is the binding contract. Where it is more
specific than the research, this plan wins.

**The one rule to teach:** `]n` walks your siblings under one parent, wrapping; `]N`
walks the outline's families, landing on each one's first child.

## Governing decisions

The decisions roster was read at planning time
(`decisions:decision-rules-are-read-at-planning-time`).

- `decisions:bob-cli-releases-ride-a-daily-train`: there is no bob-cli code. bob-cli
  changes only memory and the generated glossary/decisions pages, never `Cargo.toml` or
  `CHANGELOG.md`.
- `decisions:mac-capture-is-a-thin-client`: Bob Mac Capture is untouched.
- `decisions:prose-notes-are-owned-prose`: the walk is derived at read time. No bytes
  are ever written into prose bodies, and none into any other note.
- `decisions:bank-hub-is-path-canonical-never-area`: `bank.md` is excluded by path. It
  never walks and is never a group.
- `decisions:notebook-routes-write-notes`: bob owns the `## Notebooks` / `## Areas` /
  `## Projects` catalog bytes. The walk reads frontmatter only and never touches
  catalogs.

## The design (binding)

### Membership

- **Who joins.** A note joins the walk when
  `propertyCardNoteSpecies({ path, frontmatter })` (`240t`) returns `area`, `project`,
  `notebook`, or `prose`. Legacy `[[article]]` counts as prose.
- **What is excluded.** The same exclusions apply as that function's: `done/`,
  `_templates/`, `_conflicts/`, `area.md`, `project.md`, and `bank.md`.
- **The parent.** It is the note's single `parent:` link. Normalize it with the home
  card's `homeCardParentTarget` (`249b`) and resolve it with
  `metadataCache.getFirstLinkpathDest`. Identity is the resolved vault path, never a
  label, alias, or basename.
- **Bad parents.** A missing, list-valued, unresolved, or self parent is reported
  explicitly as a `none` reason (below). So is a parent chain that returns to the note
  itself within 16 hops. None of these ever produce a phantom group or a guessed match.
  A cycle farther up the chain, one that does not contain the note, never suppresses the
  note's own ring.
- **Sentinels.** `parent: "[[area]]"` and `parent: "[[project]]"` resolve to the untyped
  sentinel notes. A sentinel is a valid group parent labelled **Top level**. It is never
  a walk member and never a navigable `↑` target.
- **Rings by type.** A ring holds a single note type; mixed types under one parent never
  share it. A notebook's parent may be an area, a project, or a notebook. Use the
  immediate parent and never flatten to the owner. `root: true` affects reporting only,
  never the walk.
- **Project lanes.** Projects walk in two lanes: open walks with open and closed with
  closed. The lane comes from the metadata cache. Take the `cache.blocks.prj` line, find
  the `listItems` entry on that line, and read its `task` char: `x`, `X`, or `-` means
  closed. Anything else is open, including a missing or unreadable `^prj`. Areas,
  notebooks, and prose have no lane (`lane: null`).

### Order

| Type                  | Order inside a ring                                                                                      | Label |
| --------------------- | -------------------------------------------------------------------------------------------------------- | ----- |
| Area                  | stem, case-folded, then path                                                                             | stem  |
| Project (either lane) | stem, case-folded, then path                                                                             | stem  |
| Notebook              | display title (first H1 in `cache.headings` → first alias → humanized stem), case-folded, then path      | title |
| Prose                 | display title (`proseNoteCardTitleParts`, `249l`: `title:` → first alias → stem), case-folded, then path | title |

These order rules mirror bob-cli's catalog comparators: `src/native/areas/graph.rs`
`sort_paths`, `src/native/projects/catalog.rs` `sort_rows` (open lane), and
`src/native/notebooks/catalog.rs` `sort_level`. Within the same members, the walk must
never disagree with the cabinet that lists them. The one accepted divergence: the closed
lane walks alphabetically, while the Projects cabinet's Closed fold sorts newest-closed
first. Close dates live in line text the metadata cache does not carry.

### `[n` / `]n`: the sibling ring

The ring holds every member with the same type, the same resolved parent path, and (for
projects) the same lane, in the order above. `]n` opens the next member and `[n` the
previous one. Both wrap, and a wrapping stop carries `wraps: true`. `]n` never crosses
into another parent.

### `[N` / `]N`: the parent tour (why this grouping works)

The groups are the parents. `]N` opens the **first member** of the next group, and `[N`
the first member of the previous one, wrapping. The group universe for a type (and lane)
is every **occupied** parent: a parent with at least one member. Groups are ordered in
**outline order**, a pre-order traversal of the note tree:

- Each parent's key is the list of `(kindRank, foldedLabel, path)` from its topmost
  reachable ancestor down to itself. `kindRank` is area 0, project 1, notebook 2, prose
  3, other 4, which is the cabinet order on every card. `foldedLabel` uses that node's
  own row in the Order table.
- A sentinel's key is empty. The ancestor walk stops at a sentinel, an unresolved or
  list parent, a cycle, or 16 hops.
- Keys compare element by element, and a prefix sorts first.

Read top to bottom, this is the outline the catalogs print, cut into families, so `]N`
steps down that outline one family at a time. Repeated `]N` cycles a fixed, wrapping
ring of families. Every note gets a working `]N` as soon as a second occupied parent of
its type exists anywhere. That rescues the many singleton families: about a third of
notebooks and half of project parents have only one member, and for them `]N` is the
only useful key. This matches every other family, where the uppercase key tours all of
the walk's groups (`]S` review groups, `]W` lanes, `]B` bank days).

Rejected alternatives:

- **Grandparent-bounded "cousins".** `]N` would be dead on 10 of 18 areas.
- **Flat alphabetical parents.** This scatters subtrees across the tour.
- **Up/down `]N`.** This breaks the previous/next meaning every family shares.

### Worked examples

```text
pig (area)
├── foo (project)
│   ├── bar (project)   ← start here
│   └── baz (project)
├── goo (project)
│   ├── alpha (project)
│   └── beta (project)
└── zoo (project; no project children)
```

Project groups in outline order: `pig` → `foo` → `goo` (`zoo` has no members and is
skipped).

| From   | Key  | Lands on | Why                                                                                 |
| ------ | ---- | -------- | ----------------------------------------------------------------------------------- |
| `bar`  | `]n` | `baz`    | the next sibling under `foo` (Bryan's example)                                      |
| `baz`  | `]n` | `bar` ↺  | wraps within `foo` and never crosses                                                |
| `bar`  | `]N` | `alpha`  | the next family after `foo` is `goo`'s; first member                                |
| `bar`  | `[N` | `foo`    | the previous family is `pig`'s; its first project is `foo`                          |
| `foo`  | `]n` | `goo`    | `foo` lives in `pig`'s ring                                                         |
| `foo`  | `]N` | `bar`    | the next family down the outline is `foo`'s own children                            |
| `beta` | `]N` | `foo` ↺  | `goo`'s family is last in this tree, so the tour wraps to `pig`'s ring              |
| `zoo`  | `]n` | `foo` ↺  | `zoo` is last in `pig`'s ring; it has no children, so it heads no family of its own |

### Authored links and the matcher

Authored links still win per role, so a curated reading order or prose series can
override the alphabetical walk. Two hardening rules apply:

1. **Inline code never matches a step link, anywhere.** A link inside backticks is not a
   rendered link. Today `` `[[<person>]]` `` in `bob.md` answers both `[n` and `]n`.
2. **On a `tree` route only**, an authored _step_ link counts only on a nav line: a body
   line that holds only links and separators, the rule group links already follow. Then
   a `Read more ›` in prose or a `> X` label on a task line cannot hijack `]n`. Calendar
   and record routes, and notes with no derived route, keep today's anywhere-in-body
   step matching. Record walk lines carry non-link text, so they must not regress.

The rail and footer always paint the **effective** answer: authored if one wins,
otherwise derived. When an effective stop is authored, its arm says so in its tooltip
and accessible name ("Authored nav link") and carries `data-source="authored"`. The tick
track keeps showing the derived ring.

### The `tree` route contract (api.navGroups v1, additive)

`api.navGroups.resolve({ path, content, lazy? })` dispatches in this order: record, then
**tree**, then calendar. For a tree note it answers:

```text
{ ok: true, route: "tree", kind: "area"|"project"|"notebook"|"prose",
  lane: "open"|"closed"|null,
  step: { prev: Stop|null, next: Stop|null, parent: Stop|null },
  stepNone: { reason, message }|null,
  group: { noun: "parent", memberNoun: <kind>, label, shortLabel, parentPath,
           position, size, index, count, members: [{ label, path, state }],
           atStart, atEnd, enteringNext: null, enteringPrev: null } | null,
  prevGroup: Stop|null, nextGroup: Stop|null,
  none: { reason, message }|null }
```

- **`Stop`** is
  `{ path, target, groupLabel, landingLabel, shortLabel, state: "exists", ghostReason: null, wraps, source: "tree" }`.
  - On step stops, `groupLabel` is the ring's parent label. On group stops it is that
    group's parent label.
  - `landingLabel` is the landing note's label.
  - `step.parent` is the parent note, or null for a sentinel.
- **`group`** describes the current ring.
  - `position` and `size` index within the ring; `index` and `count` index the tour.
  - `members[].state` is `current` or `exists`.
- **`navGroupsOk`** (`296e`) gains `lane` and `stepNone`, both null by default. Existing
  routes are otherwise byte-identical, and the namespace stays `version: 1`.
- **`lazy: true`** is for paint callers. If the index is cold, the answer is
  `none: { reason: "indexing" }`, with nothing built on the paint path. Without `lazy`
  (keys, footer), a cold index is built synchronously once.

Reasons and copy. Use these strings verbatim unless acceptance finds them clipped:

| Reason                  | Where               | Message                                                                                                                             |
| ----------------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `no-parent`             | `stepNone` + `none` | `bar has no parent — Ctrl+Shift+- sets one`                                                                                         |
| `list-parent`           | both                | `bar's parent lists several notes — keep one`                                                                                       |
| `unresolved-parent`     | both                | `bar's parent qux is not in the vault`                                                                                              |
| `self-parent` / `cycle` | both                | `bar's parent chain loops back to bar`                                                                                              |
| `only-child`            | `stepNone`          | `bar is foo's only open project — ]N visits goo's` (lane word only for projects; drop the `]N` clause when there is no other group) |
| `single-group`          | `none`              | `foo holds every open project — ]n walks them` (`Top level holds every area — ]n walks them`)                                       |
| `indexing`              | `none` (lazy only)  | `Link walk is indexing`                                                                                                             |
| `unavailable`           | answer `ok: false`  | (keys fall back to authored links and today's notices)                                                                              |

### The rail (reuse the kit and add no new widget species)

```text
  ▸ Projects     2 open · 1 closed                              _2
  ▸ Properties   6                             created Jun 2     -
  ────────────────────────────────────────────────────────────────
  ‹ bob_git ↺               ↑ bob                     bob_plugins ›
  ⏮ pig                     ▮ ■ ▮                              goo ⏭
```

- **Step row** (`propertyCardBuildRail`, `240g`): `‹ prev`, `↑ parent`, `next ›`.
  - It appears only when the ring has at least 2 members.
  - The `↑` item is absent for a sentinel parent.
  - A wrapping arm adds a quiet aria-hidden `↺`.
  - Tooltips spell out the key and the landing: `]n · next: bob_plugins`,
    `]n · wraps to bob_git, first of bob's 3 open projects`.
- **Group row** (`propertyCardBuildGroupRow`, `240l`).
  - The arms name the neighbouring **parents**: `⏮ pig`, `goo ⏭`, and `↺` when the
    jump wraps. Each tooltip and accessible name spells out the landing:
    `]N · next: goo's projects → alpha`.
  - The centre is the current ring's tick track with the current note lit (bar and thumb
    above 21 members). Its title and hidden label read `PROJECT · bob · 2 of 3 open`.
  - With only one occupied group, the row is a ticks-only centre and has no arms.
- **Progressive disclosure.**
  - Ring ≥ 2 and groups ≥ 2: step row plus a full group row.
  - Ring ≥ 2 and groups = 1: step row plus a ticks-only centre.
  - Ring = 1 and groups ≥ 2: group row arms only.
  - Ring = 1 and groups = 1: **no rail**. The kicker crumb already shows the parent.
  - A malformed parent also gets no rail; the keys explain why.
- **Look.**
  - The rail inherits the card's root hue (`data-root-hue` → `--bob-walk-ink`). It adds
    no palette values and reuses the kit's spacing and type scale.
  - There is exactly one hairline between the cabinet stack and the rail, never two.
  - Narrow panes compact to the kit's narrow labels, and arms never clip.
  - It keeps the kit's ≥ 24 px targets, focus rings, forced-colors, and reduced-motion
    behavior.
  - It never pulses, animates, or reflows on a metadata refresh.
- **Activation.**
  1. A click re-resolves the walk.
  2. If the painted stop's path changed, show `The stop changed — try again` and repaint
     the rail.
  3. Otherwise run nav's own command (`open-prev/next-link`,
     `open-prev/next-group-link`) in the card's pane, the way `intentionCardRunInLeaf`
     does. Jump history, `]]`, and `<C-o>` then agree with the keys.
  4. A modified click opens a tab through `openLinkText`. Without nav, open the stop
     directly.
- The rail is compass chrome, not a cabinet. It has no `_` number, never sits inside a
  collapsible body, never floats sticky, and never touches cabinet pick state or the
  walker-footer overlay (`122a`). It renders whichever bracket family is selected.

### The card floor

The invariant is: card identity → body → cabinet stack (content cabinets, config,
Properties) → **link-walk rail, last**. Enforce it structurally, not by call order:

- `propertyCardFloorDock(card, rail)` marks the rail `data-card-floor="link-walk"`,
  replaces any existing direct-child floor, and appends it as the card's last child.
- `propertyCardFloorAppend(card, node)` inserts `node` before a direct-child floor if
  one exists, else appends it.
- `propertyCardFloorRelease(card)` removes the floor.
- The foot append (`240r` `propertyCardFootAdapterAppend`) and the stack mount (`240wl`
  `propertyCardCabinetsAssemble`) use `propertyCardFloorAppend`. Lazy cabinet bodies
  mount inside panels, so the order survives lazy mounts, Reading finalization, and
  repaints.
- Day/period rails (`296b`) and record rails (`242k` Live Preview, `242v` Reading) move
  to the floor. That reverses the documented "foot follows the rail" contract
  family-wide and fixes the verified record mismatch (rail above the cabinets in Live
  Preview, below them in Reading).

### The links footer (`n` panel)

The changes are copy only, in `124-links-walker-footer.js`:

- **Context** reads `bob 2/3` (with `open` / `closed` added for projects). A one-member
  ring shows only its parent label.
- **Group arms** name the parents. The hint noun is `parent` (`]} goo`).
- **Wrap and singleton notices** reuse the `stepNone` / `none` messages.
- **`createEligible`** follows `state === "creatable"` only, so derived stops never get
  the dashed `+` look.

### Reliability and performance

- **One index.** A single lazily built, generation-stamped index lives in memory:
  `path → { kind, lane, parentPath | problem, label, sortKey }`, plus a ring map and a
  tour map per `kind|lane`.
  - It is built in one pass over `getMarkdownFiles()` and `getFileCache()`, reading the
    settled cache (`propertyCardSettledFileCache`) when present.
- **Warm-up and invalidation.**
  - It warms off the paint path, at `onLayoutReady` and then idle time.
  - A vault `create`, `delete`, or `rename` bumps the generation.
  - A metadata `changed` event recomputes that file's entry and bumps the generation
    only when kind, lane, parent, or label changed.
  - Registered listeners hear about each bump, and mounted rails repaint in place when
    their signature changes.
- **Safety.** The provider is synchronous and never throws, writes, or spawns bob. On
  any failure it answers `unavailable`, and the keys fall back to authored links.
- **Budgets.**
  - The index build must take ≤ 30 ms best-of-5 for 8,000 synthetic files in Node, on
    the `test:perf` lane.
  - A warm `resolve` must do no vault scan.
  - The rail must add no long task over 50 ms and must not regress the Reading cabinet
    paint that bob-cli-qq.10 tuned (acceptance measures this).

## Running beside bob-cli-qq (coordination)

When this plan was written, bob-cli-qq.10.1–.3 were closed. qq.10.4 (acceptance GO),
qq.10.5 (gate deletion: `240o`, `240r`, `240s`, `240w*`, `241q`, `241tb`, `241v`,
`242kc`, `248m`, `374k`, the cabinets runner, and `docs/develop/validation.md`
Cabinets), and qq.10.6 (docs pages and the property/area/project/cabinet strands) were
in progress.

| Phase        | Touches                                                                                     | Overlap with qq                                                              | Gate                       |
| ------------ | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- | -------------------------- |
| `engine`     | `296e`, `296f`, new `296h`/`296i`/`378a`, new tests                                         | none                                                                         | starts at approval         |
| `nav`        | nav `483`, `670`, `680`, `482`, the open-link notices; ledger `124`                         | none                                                                         | after `engine`             |
| `rail`       | `240g`, `240l`, new `240u`/`296j`/`296k`/`378b`, a new CSS region, `preview-group-rail.cjs` | none                                                                         | after `engine`             |
| `cards`      | `240r`, `240wl`, `242k`, `242v`, `296b`, home/notebook/prose card fragments                 | `240r`/`240wl` are gate-deletion sites; qq.10.4 owns the Reading perf budget | **bob-cli-qq.10.5 CLOSED** |
| `acceptance` | new runner scripts; athena                                                                  | athena runs                                                                  | after `cards`              |
| `docs`       | bob-plugins docs pages; bob-cli memory and generated pages                                  | qq.10.6 rewrites the same pages and strands                                  | **bob-cli-qq.10.6 CLOSED** |

So coordination is needed only as **ordering**. `engine`, `nav`, and `rail` have zero
file overlap with qq. None of them changes card DOM or the footer overlay, so qq's
acceptance captures and perf numbers stay valid while they land. `cards` waits for the
gate deletion and `docs` waits for qq's docs. There are no meetings and no shared
branches:

- **Gate edges.** `engine` step 0 adds the cross-epic bead edges.
- **Phase gates.** `cards` and `docs` re-check their gate before editing anything.
- **Mechanical conflicts.** These are resolved with the qq rule (below), never by
  hand-merging.
- **Deploys.** Deploys are serialized by the `bob plugins sync` refusal protocol.

## Rules every phase follows

- **Repos.**
  - All plugin code lives in the linked **bob-plugins** repo. Open it with
    `sase repo open bob-plugins -r "<why>"`, use only the printed path, and read its
    `AGENTS.md` first.
  - bob-cli changes only in `docs`.
  - Read the `obsidian.md` reference memory before any vault read, and `tailnet.md`
    before using athena. Real-Obsidian runs use scratch vaults only; never open the live
    vault in an automated run.
- **Fragments.**
  - Edit `src/` fragments only, and keep each at or below 1000 lines.
  - Run `npm run build`, and never hand-edit `main.js`. `styles.css` is hand-edited; add
    rules in a named region.
  - Prefixes: `navGroupsTree*` (engine), `propertyCardFloor*` (kit), `treeWalkRail*`
    (rail) in JS; `.bob-link-walk-*` and `[data-card-floor]` in CSS.
  - Add new test scripts to `package.json` `test:unit`, and perf tests to `test:perf`.
- **Hands off.** Before `cards`, do not edit any qq-owned file: `240o`, `240r`, `240s`,
  `240w*`, `241al`, `241q`, `241tb`, `241v`, `242kc`, `248m`, `374k*`, `122a`,
  `scripts/obsidian-cabinets-e2e*.cjs`, `scripts/test-obsidian-cabinets-e2e.cjs`, and
  the Cabinets section of `docs/develop/validation.md`.
- **Rebase discipline.** This epic's parallel phases and qq all bump manifests. On a
  conflict in `main.js`, `styles.css`, `fragments.json`, `package.json`, a manifest
  version, or the docs inventory:
  1. Take upstream.
  2. Re-apply your additive lines.
  3. Rebuild, regenerate the inventory, and re-bump.

  Never hand-merge a bundle.

- **For each plugin change.**
  1. Bump that plugin's manifest.
  2. Run `npm run build`, `npm run build:check`, the focused suites, and `npm test`.
  3. Run `bob plugins sync` without `-F`, following the refusal protocol in `AGENTS.md`.

  A pre-existing `npm test` failure that also fails on clean master is recorded, not
  fixed.

- **Invariants.**
  - **Passive.** Paint, keys, footer, and rail never write the vault or spawn bob.
  - **Never throw.** If a part is missing (no nav plugin, an old provider, a cold
    index), show less rather than error: no rail, or today's notices.
  - **Unchanged elsewhere.** Calendar and record answers stay byte-identical except for
    the two null fields `navGroupsOk` gains.
- **Beads.**
  - Phase workers never create beads. Record `PROPOSED FOLLOW-UP:` notes on your own
    phase bead.
  - Retain real-app evidence with `/sase_artifact` against your phase bead.
  - Run `sase bead epic-symbols <phase>` before closing.
- **Long work.** Builds, `npm test`, waits, and athena runs go through `/sase_monitor`.
  Finish every turn through `/sase_final`.

## Phase `engine`

0. **Gate edges.** Read this epic's bead and find the `cards` and `docs` phase beads.
   Then add the edges:
   - `sase bead dep add <cards-bead> bob-cli-qq.10.5`
   - `sase bead dep add <docs-bead> bob-cli-qq.10.6`

   Confirm both with `sase bead read`. If a command refuses, note it on your bead; the
   in-phase gates still hold.

1. **Pure model** in new `296h-plugin-nav-groups-tree-model.js`, with no DOM and no app
   access:
   - membership and exclusions;
   - parent normalization and problems;
   - labels and comparators;
   - the lane from cache blocks and list items;
   - ring building;
   - outline keys and the tour;
   - stop and group shaping;
   - the reason/message table above.
2. **Index mixin** in new `296i-plugin-nav-groups-tree-index.js`:
   - the lazy build, the generation counter, and per-file recompute;
   - vault and metadata listeners registered through `registerEvent`;
   - warm-up at layout ready, then idle;
   - `navGroupsTreeOnChange(listener) → unsubscribe`;
   - teardown on unload.

   Install it beside `BobLedgerToolsNavGroupsMixin`.

3. **Dispatch.** In `296f`, `navGroupsResolve` tries record, then tree, then calendar,
   and honours `lazy`. `teardownNavGroups` tears the index down. In `296e`,
   `navGroupsOk` gains `lane` and `stepNone`.
4. **Exports.** Add test exports in new `378a-exports-nav-groups-tree.js`.
5. **Tests** in new `scripts/test-ledger-tools-nav-groups-tree.cjs`:
   - **Behavior.**
     - every row of the worked-examples table, both directions, including the wraps;
     - the top-level area sentinel and the `project.md` sentinel;
     - nested notebooks under notebooks, areas, and projects;
     - subprojects, and the lane split and its flip when `^prj` toggles;
     - legacy `[[article]]`;
     - `[[x.md|y]]` and heading forms of `parent`;
     - duplicate basenames in different folders;
     - missing, list, unresolved, self, and cyclic parents, plus an ancestor-only cycle
       that keeps the ring;
     - excluded paths, `bank.md`, and an album note (not a member).
   - **Invariants.**
     - every step target shares the source's type, parent, and lane;
     - every group target is a first member of an occupied group of the same type and
       lane;
     - repeated `]N` cycles a stable ring;
     - calendar and record answers are unchanged.
   - **Comparator parity vectors** copied from the Rust catalog tests (areas, projects
     open lane, notebooks).
   - **Index behavior.**
     - create, delete, and rename each bump the generation;
     - an irrelevant `changed` event does not;
     - `lazy` cold answers `indexing` and builds nothing;
     - the provider never throws on a broken `app`, and a passivity spy records zero
       writes and zero spawns.
6. **Perf.** New `scripts/test-ledger-tools-nav-groups-tree-performance.cjs` on
   `test:perf` enforces the 30 ms / 8,000-file build budget and the no-scan warm
   resolve.
7. **Ship.** Bump bob-ledger-tools, build, test, and sync. After this phase, `[n`, `]n`,
   `[N`, `]N`, `[[`, `]]`, `[{`, and `]}` already work on all four types through nav's
   existing provider fallback.

## Phase `nav`

1. **Inline code.** `findFirstRenderedLinkInLine` (`680`) skips links inside inline code
   spans. Count backtick runs the way CommonMark does: a run of N backticks closes only
   on another run of exactly N.
2. **Nav-line-only steps on tree routes.** In `resolveNavLinks` (`483`), when the
   provider answers `route: "tree"`, take authored step links only from nav lines. Reuse
   the line consumer `authoredNavGroups` already applies. Other routes keep
   `findFirstRenderedLink` exactly as today.
3. **Pass-through.**
   - `resolveNavLinks`, and so `walkMode.links` (still v7, additive), also returns
     `route`, `kind`, `lane`, `parent` (the provider's `step.parent`), and `stepNone`.
     Each is null when absent.
   - Update the walkMode and navGroups sections of `docs/develop/plugin-apis.md`
     **only** in `docs`. Record the shape now in a bead note.
4. **Notices.**
   - `openLabeledBodyLink` shows `stepNone.message` when there is no step stop, and
     otherwise keeps `No ${label} link found`.
   - `openGroupLink` already shows `groupNone.message`; check it shows the tree reasons.
5. **Footer copy.** Update `124-links-walker-footer.js` as described in _The links
   footer_.
6. **Tests.**
   - Extend `scripts/test-navigation-group-links.cjs` and
     `scripts/test-navigation-period-links.cjs`:
     - inline-code vectors on every route;
     - nav-line-only on `tree` and anywhere-in-body on calendar and records (a record
       walk line with a position count still steps);
     - authored-override vectors (an authored nav line beats the derived step).
   - The `bob.md` hazard: `` `[[<person>]]` `` no longer steps.
   - Extend `scripts/test-navigation-bracket-repeat.cjs` (`]]` and `]}` repeat tree
     stops) and `scripts/test-ledger-tools-walker-footer.cjs` (context, lanes, parent
     arms, wrap and singleton copy, `createEligible`).
7. **Ship.** Bump both plugins, build, test, and sync.

## Phase `rail`

1. **Kit floor.** Add new `240u-plugin-property-card-floor.js` with
   `propertyCardFloorDock`, `propertyCardFloorAppend`, and `propertyCardFloorRelease`.
   They are pure DOM, return null instead of throwing, and act only on direct children.
2. **Kit polish.**
   - In `240g`, rail step items accept `wraps` (aria-hidden `↺`), `tooltip`, and
     `source` (`data-source`).
   - In `240l`, confirm a ticks-only centre needs no arms.
   - Existing callers stay byte-identical; an existing-caller snapshot test proves it.
3. **Pure tree rail model** in new `296j-plugin-tree-walk-rail-model.js`.
   `treeWalkRailModel({ answer, provider })` returns null (no rail) or
   `{ step, group, label, signature }`:
   - the progressive-disclosure table;
   - labels, tooltips, accessible names, and the `PROJECT · bob · 2 of 3 open` centre
     copy;
   - the authored cue;
   - a stable signature over every painted target and label.
4. **Rail mixin** in new `296k-plugin-tree-walk-rail.js`:
   - `treeWalkRailAnswer(path, source)` follows `intentionCardNavAnswer`'s
     `{ from, answer, provider }` shape:
     1. Call the provider first with `lazy: true`. An `indexing` answer means no rail
        now and a repaint on warm.
     2. Then call nav's `walkMode.links` when it is v7 or later.
   - `treeWalkRailBuild(...)` returns the built kit rail.
   - Activation and re-validation follow _The rail_.
   - A mounted-rail registry fed by `navGroupsTreeOnChange` swaps a rail in place
     through `propertyCardFloorDock` only when its signature changes.
5. **CSS.** Add a new `Link walk rail` region in `styles.css`:
   - the floor hairline and spacing under `.bob-property-card > [data-card-floor]`;
   - hue ink;
   - narrow, forced-colors, and reduced-motion rules.

   Reuse kit variables only.

6. **Tests and previews.**
   - New `scripts/test-ledger-tools-property-card-floor.cjs` covers dock, replace,
     append-above, and release, with non-card and foreign-child guards.
   - New `scripts/test-ledger-tools-tree-walk-rail.cjs` covers:
     - every disclosure state, lanes, wraps, the sentinel, and the authored cue;
     - signature stability;
     - re-validation on a changed stop;
     - modified click;
     - no nav plugin, a cold index, and zero writes and spawns.
   - Add tree fixtures (full, ticks-only, arms-only, bar above 21, narrow) to
     `scripts/preview-group-rail.cjs`. Retain the rendered previews with
     `/sase_artifact`.
7. **Ship.** Bump bob-ledger-tools, build, test, and sync. No card mounts the rail yet,
   so cards stay byte-identical.

## Phase `cards`

0. **Gate.** `sase bead read bob-cli-qq.10.5` must show CLOSED.
   - If it is not closed, edit nothing. Wait through `/sase_monitor`, polling at most
     every 10 minutes, and resume when it closes.
   - If it stops (a NO-GO or `PROPOSED FOLLOW-UP` note) or is still open after 24 hours,
     note `BLOCKED ON bob-cli-qq.10.5` on your bead and stop.
   - Rebase onto master after it closes.
1. **Floor-aware appends.** Make the two appends floor-aware:
   - In `240r`, `card.appendChild(built.element)` becomes `propertyCardFloorAppend`.
   - In `240wl`, `card.appendChild(stack)` becomes `propertyCardFloorAppend`.
2. **Migrate the existing rails.** Dock them through `propertyCardFloorDock`:
   - `296b`: the day/period rail;
   - `242k`: the record rail in Live Preview;
   - `242v` `recordWalkPaintReading`: the record rail in Reading. Its paragraph-fallback
     path keeps its in-body placement.

   Update the "foot follows the (nav) rail" comments in `240o`, `242k`, and `styles.css`
   to "the rail is the card's floor".

3. **Mount the tree rail.** Wire `treeWalkRailBuild` + `propertyCardFloorDock` into the
   area/project home card (`249c`/`249d`), the notebook card (`249h`/`249i`), and the
   prose card (`249m`/`249n`), in both Live Preview and Reading.
   - Register each mount with the rail registry, and release it on teardown, demotion,
     and unload.
   - A rail failure leaves the card exactly as it was without a rail.
4. **Tests.**
   - New `scripts/test-ledger-tools-card-floor-invariant.cjs`. Whenever a rail exists it
     is the card's last direct child. Cover:
     - every species fixture (area, project, notebook, prose, day, week, quarter, year,
       album, record, bank, ref), in both Live Preview and Reading DOM;
     - cabinet open and close, a lazy body mount, a repaint, and an in-place rail swap.
   - Update the existing tests that pin rail-before-foot order (intention-card group
     rail, record walk, foot adapters, home/notebook/prose card DOM) to the new
     contract.
   - Extend `scripts/test-ledger-tools-property-card-passive-rendering.cjs` with tree
     rails.
5. **Ship.** Bump bob-ledger-tools, build, test, and sync.

## Phase `acceptance`

1. **The runner.** Add new `scripts/obsidian-tree-walk-e2e.cjs` plus the wrapper
   `scripts/test-obsidian-tree-walk-e2e.cjs` (on `test:unit`), modeled on
   `obsidian-group-jump-e2e.cjs` and `obsidian-record-walk-e2e.cjs`. Its scratch-vault
   fixture holds:
   - the `pig/foo/bar/baz/goo/alpha/beta/zoo` tree;
   - top-level areas and nested areas;
   - open and closed sibling projects;
   - notebooks nested three deep;
   - two prose notes under one parent;
   - malformed parents;
   - the inline-code hazard note and an authored nav-line note;
   - a 64-child project parent.
2. **Mandatory checks** (both views where they apply):
   - **Keys.** All eight keys on all four types land where the table says. The rail's
     painted targets equal each key's landing, and the footer arms name the same notes.
   - **Floor.** The rail is the card's last child through cabinet toggles and repaints,
     on tree, day, and record cards.
   - **Live updates.** A reparent, rename, or `^prj` toggle updates the rail without a
     card rebuild.
   - **Re-validation.** A stop deleted between paint and click shows `The stop changed`.
   - **Passivity.** Spies record zero vault writes and zero bob spawns.
   - **No console errors.** Subscribe to `Runtime.consoleAPICalled` as the cabinets
     runner does. Any `console.error` fails the run.
   - **Perf.** The Reading card-paint median on the 64-child host with the rail must be
     ≤ 1.15× the median with the provider stubbed unavailable through a test hook, with
     no long task over 50 ms. Record the index build time.
3. **Captures.** Run on athena under `xvfb-run`, twice. Capture every type in Live
   Preview and Reading, in light and dark, at 420 px, and in forced colors.
4. **Review.** Copy the screenshots back and inspect **every** PNG. Write
   `acceptance-review.md` with a verdict per image and a beauty checklist:
   - the rail aligns with the stack edges;
   - there is one hairline;
   - nothing clips at 420 px;
   - the hue is correct;
   - focus rings are visible;
   - nothing is visually noisy.
5. **Verdict.**
   - **GO** requires every mandatory check to pass in both runs and every capture to be
     reviewed. Retain the reports, the review, and the screenshots with
     `/sase_artifact`, and note `VERDICT: GO` with the source commit and the perf
     numbers.
   - **On NO-GO:** fix in place within this epic's files and rerun. Otherwise record the
     evidence and a `PROPOSED FOLLOW-UP:`, then stop.

## Phase `docs`

0. **Gate.** `sase bead read bob-cli-qq.10.6` must show CLOSED. Wait and stop exactly as
   in the `cards` gate. Then read the `docs.md` reference memory and re-read the pages
   below as they stand after qq.
1. **bob-plugins docs.**
   - `docs/bob-navigation-hotkeys.md`, `n` family: tree routes, rings, lanes, the parent
     tour, the matcher rules, and the notices.
   - `docs/area-and-project-cards.md`, `docs/notebook-card.md`, `docs/prose-card.md`:
     the rail at the floor.
   - `docs/intention-card.md` and `docs/record-card.md`: replace "The foot follows the
     rail" with the floor contract. Check `docs/albums.md` for the same sentence.
   - `docs/develop/plugin-apis.md`: the `tree` route, `lane`/`stepNone`, `lazy`, and the
     walkMode pass-through fields.
   - `docs/develop/validation.md`: the new runner.
   - `docs/bob-ledger-tools.md`: keep it at or under the 6000-word reference cap.
   - Regenerate the inventory, then run `npm run docs:check` (which also runs bob-cli
     `docs-check`). Then sync.
2. **Memory** (bob-cli). Use `/sase_memory_write` and only the approved decisions.

   > [!decision] memory_glossary Add `sase/memory/glossary/link-walk.md`:
   >
   > ```markdown
   > ---
   > keyword: Link Walk
   > aliases:
   >   - "n walk"
   >   - "links walk"
   >   - "sibling walk"
   > ---
   >
   > The `n` bracket walk between neighbouring notes: `[n` / `]n` step to the previous /
   > next note, `[N` / `]N` jump to the first note of the previous / next group, and
   > `[[` / `]]` and `[{` / `]}` repeat them while `n` is the selected family. One
   > resolver answers the keys, the repeats, the links panel of the [[walker-footer]],
   > and the card rail, so they always agree. An authored link on a nav line (a body
   > line holding only links) wins its role: `‹`/`›` or `prev`/`next` for steps, `«`/`»`
   > for groups. Otherwise Bob Ledger Tools derives the stop from the note itself and
   > writes nothing. Daily notes and [[period-note]]s walk the calendar, grouped by the
   > enclosing period; [[record-note]]s walk their album's ring, grouped by `walk_by`.
   > [[area-note]]s, [[project-note]]s, [[notebook]]s, and [[prose-note]]s walk their
   > siblings: notes of the same type with the same resolved `parent`, in the parent
   > catalog's order, wrapping, with open and closed projects in separate lanes. Their
   > groups are those parents, toured in outline order (each parent before its children,
   > children in cabinet order), so `]N` steps down the outline one family at a time.
   > The walk paints as the rail on the floor of the note's [[property-card]], below
   > every cabinet. Positions are places in a ring, never progress or review state;
   > nothing about a derived stop is stored, and derived stops open only existing notes
   > (missing days and periods keep their own create rules). Not the `]s` review walk,
   > the `]w` lane walk, the `]b` bank walk, a [[task-link]], or a [[task-relation]].
   > See `docs/reference/plugins/bob-navigation-hotkeys.md`.
   > ```

   > [!decision] memory_glossary In the same approved change, in
   > `glossary:walker-footer`, replace "then the stops Bob derives for days, periods,
   > and grouped records" with "then the stops Bob derives for days, periods, grouped
   > records, and the sibling rings of area, project, notebook, and prose notes (see
   > [[link-walk]])". Change nothing else.

   > [!decision] memory_decision_record Add `decisions/tree-notes-walk-sibling-rings.md`
   > (keyword "Tree Notes Walk Sibling Rings And Tour Their Parents",
   > `metadata.status: accepted`, decided on the landing date). It needs these parts:
   >
   > - **Applies to:** bob-plugins nav and ledger tools.
   > - **Claim:** derived same-type sibling rings that wrap; project lanes; groups are
   >   occupied parents in outline order; nothing is written; authored nav lines win;
   >   the rail is the card's floor.
   > - **Rejected:** written walk lines, cousin-bounded `]N`, flat alphabetical parents,
   >   up/down `]N`, and `]n` crossing parents.
   > - **Evidence:** the research ref, this plan, and the landed plugin versions.
   > - **Cost:** `]N` can tour the whole type.
   > - **Reopens when:** Bryan needs walk order visible in Source mode or git, or
   >   prefers bounded `]N`.

   Run `sase memory init`, then `just docs-gen` (this regenerates
   `docs/reference/glossary.md` and, when a record was added, the decisions page) and
   `just docs-check` in bob-cli. Never touch `Cargo.toml` or `CHANGELOG.md`.

## Epic acceptance (land agent)

- **Phases.** All six phases are closed, and `acceptance` recorded `VERDICT: GO` with
  retained evidence.
- **Build and deploy.** bob-plugins master builds (`npm run build:check`), `npm test` is
  green apart from recorded pre-existing failures, and `bob plugins sync` reports the
  landed versions deployed, with no `-F`.
- **Docs gates.** Both `npm run docs:check` (bob-plugins) and `just docs-check`
  (bob-cli) pass.
- **Memory.** Only approved memory changed, and the generated pages are current.
- **Behavior on master.** Spot-check the acceptance evidence against master: the
  worked-examples table, the floor invariant on every species, and the `bob.md`
  inline-code hazard gone.
- **Out of scope.** Albums and ref notes do not walk as tree notes, and nothing writes
  walk bytes. Record any wish for either as a `PROPOSED FOLLOW-UP:`.
