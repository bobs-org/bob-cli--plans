---
tier: epic
title: Task link marks — status rings on links that point to tasks
goal: 'Every rendered link whose target is a `#task` line wears a small status ring
  (Ready, Next, In Progress, Blocked, Done, Cancelled) in Live Preview and rendered
  views, so Bryan can tell at a glance which links point to tasks and what state those
  tasks are in. Links to notes, headings, and non-task blocks stay exactly as they
  are, nothing is ever written to the vault, and a session toggle restores plain links.

  '
decisions:
  ledger_rows:
    ask: Should dedicated Pomodoro Task Links in daily-note ledgers also wear the
      ring?
    choices:
      skip: Ledger rows keep their 🍅 / ◐ / type-drift slot; only prose links in daily
        notes get rings
      mark: Every ledger row also wears a ring (◐ rows keep only ◐); status at a glance,
        one glyph per row
    default: skip
    why: Every ledger row is a Task Link already, so a ring there adds noise but no
      new information.
    answer: skip
  dim_closed_links:
    ask: Should links to Done or Cancelled tasks also dim their link text, not just
      show the closed ring?
    default: false
    why: The ring already says "closed"; leaving the link text alone keeps Obsidian's
      link styling intact.
    answer: false
phases:
- id: core
  title: Contract, pure core, ring glyphs, and visual preview
  depends_on: []
  size: medium
  description: 'core: write the bob-cli display contract with conformance vectors,
    add the pure bob-ledger-tools core (link scanner, line context, target-line parser,
    state and tooltip models), define the ring glyph family in CSS, and tune it with
    a light/dark preview harness.'
- id: live_preview
  title: Live Preview rings, resolver, refresh, and toggle
  depends_on:
  - core
  size: medium
  description: 'live_preview: add the target resolver (live doc, then Tasks memo,
    then metadata cache), the CM6 widget extension, click handling, the swap animation,
    refresh plumbing, and the session toggle, with vector-driven and performance tests.'
- id: reading
  title: Reading view, embeds, hover popovers, and query rows
  depends_on:
  - live_preview
  size: medium
  description: 'reading: add the post-processor for Reading view, embeds, hover popovers,
    and Tasks/Dataview rows, apply the DOM exclusion rules, and add a stamp-and-sweep
    refresh that updates rings in place across every window.'
- id: landing
  title: Real-Obsidian e2e, docs, version, and sync
  depends_on:
  - reading
  size: medium
  description: 'landing: prove the feature in a real Obsidian scratch vault (LP, Reading,
    cross-note status changes, toggle, line height, light/dark screenshots), fix what
    it finds, finish the plugin docs and manifest, bump the version, and sync to the
    vault.'
proposed_by: bbugyi200.athena.16q
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 09:24:19
status: done
bead_id: bob-cli-f5
---

- **PROMPT:** [prompts/202611/task_link_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_link_marks.md)
- **BEAD:** [bob-cli-f5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-f5/README.md)

# Plan: Task link marks

## Context

A **Task Link** (glossary) is a block link to an Obsidian task: `[[note#^id]]`. Obsidian
renders it as `bob > ^mac-gcal`. That looks the same as a link to a non-task block
(`[[dev_tools#^z-241023-0r|tmux]]`, one of tens of thousands of legacy zorg block
links), a heading, or a note. Outside the two places that already handle task links,
nothing tells them apart. The first place is the managed rows (Depends-On, BLOCKS,
LINKS, and LINKED FROM render as chips). The second is today's Pomodoro ledger (◐ In
Progress marks). So a line like
`This should be implemented with [[bob#^move-task-to-bullet]]!` in `bob.md` gives no
sign that the link points to a task, or whether that task is still open.

This epic adds **task link marks**: a quiet status ring in front of every rendered link
whose target is a `#task` line. The ring's silhouette means "this is a task", and its
fill and ink give the task's status. This is the same idea as GitHub's issue-state icons
and Linear's status circles. Bob already uses that language in the ledger's ◐ half-ring
In Progress mark, and that mark becomes the In Progress member of the new family.

Ownership: **bob-ledger-tools plus bob-cli docs only.** There are no Rust, CLI, capture
grammar, or Mac changes, and `cli_rules.md` does not change.

- **bob-cli** (this repo): add a new `docs/task-link-marks.md` contract and update
  `mkdocs.yml`, `docs/README.md`, `docs/reference/vault-format.md`, and `docs/plan.md`.
- **bob-plugins** (linked repo; open it with `/sase_repo`:
  `sase repo open bob-plugins`): change only `plugins/bob-ledger-tools/`, plus its
  `docs/bob-ledger-tools.md`, `package.json` test list, and new `scripts/` files. Read
  its `AGENTS.md` first:
  - Edit `src/` fragments and never `main.js`.
  - Register new fragments in `src/fragments.json`.
  - Keep each hand-edited fragment at or below 1000 lines.
  - Run `npm run build`, then `npm test`, then `bob plugins sync` at the end of every
    bob-plugins phase.
- `src/170-plugin-lifecycle.js` is already **exactly 1000 lines**. Wire the new setup
  without growing it. For example, move the run of `this.setup…Marks()` calls into one
  `setupMarkFamilies()` method in another fragment and call that instead.
- Every new test injects its clock and fakes `app`. Do not copy the wall-clock pattern
  in `test-ledger-tools-unblocked-glyph.cjs`.

## The design (canonical spec; the contract doc transcribes it)

### What you see

```text
before: This should be implemented with bob > ^move-task-to-bullet!
after:  This should be implemented with ◯ bob > ^move-task-to-bullet!      (Ready task)

before: Blocked on bob > ^gcal-projects, see also dev_tools > ^z-241023-0r
after:  Blocked on ⊖ bob > ^gcal-projects, see also dev_tools > ^z-241023-0r
                                                     (non-task block: unchanged)
```

The ring is the only change. Link text, link color, underline, hover preview, and click
behavior stay Obsidian's own (see `dim_closed_links` below for the one optional
exception). Nothing is written to the vault. Without the plugin, the vault looks exactly
as it does today.

### Principles

- **Identity by shape, state by fill.** Every state is drawn on the same circle, so
  "task link" reads at a glance even in grayscale. The fill tells the state, and the ink
  repeats it. Color is never the only signal.
- **Truthful or neutral.** A ring appears only when the target provably resolves to
  exactly one `#task` line. Unresolved notes, missing ids, duplicate ids, non-task
  blocks, plain checklists, and unknown data render **no ring**. No warning glyph
  appears in prose, because a link to a non-task block is legitimate. Broken links stay
  chip territory.
- **Rings, not checkboxes.** A square glyph next to a link would invite clicking to
  complete the task. A ring reads as a status indicator, and it matches the existing ◐
  mark.
- **Quiet.** The ring is small and has no pill or background at rest. It never changes
  line height, never takes document positions, and never rebuilds on selection-only
  updates.

### Which links get a ring

A link gets a ring only when **all** of these hold:

1. **It is a block link.** It is a wikilink `[[path#^id]]` or `[[path#^id|alias]]`, a
   same-note `[[#^id]]`, or an internal Markdown link `[text](path.md#^id)`. The
   Markdown destination may use the `<…>` form or `%20`-style percent encoding, and `^`
   may be written as `%5E`. Block ids match `[A-Za-z0-9-]+`, the same rule as
   `RELATION_BLOCK_ID_RE`.
2. **It is not an embed.** `![[…]]` and `![…](…)` get no ring, because the transclusion
   already shows the task's own checkbox.
3. **It is not inside** inline code, a fenced code block, a `%% … %%` comment, or
   frontmatter or properties.
4. **It is not on a managed edge row.** Depends-On, BLOCKS, LINKS, LINKED FROM, and
   Sub-projects rows (any `mentionIsEdgeLine` verdict except not-a-line) get no ring.
   Chips own those rows. When chips are toggled off, or the cursor reveals the raw text,
   the row is meant to read raw.
5. **It is not a dedicated Pomodoro Task Link row** in a canonical daily note's
   `## Pomodoros` section. Such a row is a direct child bullet of a column-0 Pomodoro
   entry (open or closed). Its body is exactly one `[[target#^id]]` (alias allowed),
   optionally after a `🍅 ` prefix, optionally struck `~~…~~`, and optionally followed
   by a trailing ` #` move-only marker. Deeper bullets, mixed prose, the Events section,
   and every other section **do** get rings.

   > [!decision] ledger_rows = mark Rule 5 changes. Dedicated ledger rows also get a
   > ring, except a row where the ◐ In Progress progress mark renders, because ◐ is the
   > ring family's In Progress glyph and a link never shows two. On a row with a dashed
   > type-drift `#TYPE` prefix, the ring sits after that prefix, next to the link.
   > Vector TC8 flips to "ring" (or "◐ only" for a progress-marked row). The reading
   > phase generalizes the same DOM rule.

6. **The target is a task.** The link resolves to a file
   (`getFirstLinkpathDest(linkpath, sourcePath)`; an empty linkpath means the source
   note). The block id names exactly one line in that file. That line is a list item
   with a checkbox **and** an exact `#task` tag token outside inline code. A plain
   checklist row without `#task` is not a Bob task.
7. **The session toggle is on.**

### Target data: one resolver, three sources, in this order

1. **Live Preview, same-note target** (the resolved path is the editor's own file): read
   the **live editor document**. Build a block-id → line map once per document version.
   Parse the target line with the core's task-line parser: checkbox symbol, exact
   `#task` outside inline code, trailing standalone `^id`. A duplicate id means no ring.
   A same-note status change shows on the same frame, before any save.
2. **The Tasks memo**, through the existing `dependencyTasksIndex()` keyed
   `path\0blockId` (the one cache the dependency and relation chips share). A memo hit
   is a `#task` line, because the Tasks global filter is `#task`. Extend that index pass
   to also record keys seen more than once, so a duplicate id is neutral (TR13).
3. **Metadata cache fallback**, used when Tasks is absent, not yet indexed, or misses
   the key: `blocks[id]` gives the line, `listItems` gives the checkbox char at that
   line, and `tags` must hold `#task` on that line. Match block ids the same way
   `relationIndexHasCachedBlock` does. This path has no title text, so the tooltip drops
   its title line (TT6). Never read files from disk during render.

**State from symbol:** ` ` gives ready, `*` next, `/` in-progress, `?` blocked, `x`/`X`
done, and `-` cancelled. Any other symbol falls back to the Tasks status type: DONE
gives done, CANCELLED gives cancelled, NON_TASK gives no ring, and anything else gives
ready.

### The ring family (glyphs and ink)

Each state has one CSS mask on a 16×16 viewBox, defined once on `body` and shared by
every host. Geometry may be tuned by ±0.3 units in `core`'s preview pass, and the final
numbers go into the contract.

| State             | `data-state`  | Glyph                                                                               | Ink (theme hook first)                                                                                          |
| ----------------- | ------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Ready `[ ]`       | `ready`       | empty ring                                                                          | `--bob-task-link-ready-color`, then the task-identity teal `var(--bob-task-tag-color, var(--bob-task-tag-hue))` |
| Next `[*]`        | `next`        | ring + center dot                                                                   | `--bob-task-link-next-color`, then `--task-status-next` (green)                                                 |
| In Progress `[/]` | `in-progress` | ring + filled right half (**the existing `--bob-progress-glyph`, reused verbatim**) | `--bob-task-link-in-progress-color`, then `--task-status-in-progress` (amber)                                   |
| Blocked `[?]`     | `blocked`     | ring + horizontal bar (⊖)                                                           | `--bob-task-link-blocked-color`, then `--task-status-blocked` (red)                                             |
| Done `[x]`        | `done`        | filled disc with the check knocked out                                              | `--bob-task-link-done-color`, then `var(--task-status-done, var(--text-muted))`                                 |
| Cancelled `[-]`   | `cancelled`   | ring + diagonal slash (⊘)                                                           | `--bob-task-link-cancelled-color`, then `--task-status-cancelled` (gray)                                        |

Each status ink gets a literal fallback hex like the existing rules
(`var(--color-green, #3f8f5f)`).

- **Geometry:**
  - Ring: `circle cx 8 cy 8 r 6.05`, `stroke-width 1.9`, matching the progress glyph.
  - Next dot: `circle r 2.35`.
  - Blocked bar: `M5.1 8h5.8`.
  - Cancelled slash: `M5.3 10.7 10.7 5.3`.
  - Bar and slash: round caps, `stroke-width 1.9`.
  - Done: a `circle r 7` disc with the check `M5 8.3 7.1 10.4 11 6.1` (stroke 1.9, round
    caps and joins) knocked out through an SVG `<mask>` inside the data URI.
- **Why teal for Ready:** Ready has no lane tint in the vault, and teal already means
  "tracked task" (the `#task` hash glyph). So the most common open state reads as "task,
  nothing special". Teal stays distinct from the violet link text it sits beside. Done
  and Cancelled recede (muted, `opacity: 0.8`). The other states use the vault's
  established status hues.
- **Host:** an empty `span.bob-task-link-mark[data-state]`. `::before` draws the mask in
  `background-color: var(--bob-task-link-ink)`. Starting metrics, tunable in `core`:
  - `display: inline-flex`, centered, a `1em × 1em` box with a `0.86em` glyph.
  - `vertical-align: -0.13em`, `margin-inline: 0.04em 0.24em`, `line-height: 1`.
  - `border-radius: 999px`, `user-select: none`, `cursor: default`,
    `text-decoration: none`.
  - Copying text never copies a glyph, and the box never changes line height.
- **Hover:** full opacity plus a capsule,
  `color-mix(in srgb, var(--bob-task-link-ink) 12%, transparent)`, with `120ms ease`
  transitions.
- **Swap animation:** the ring plays
  `bob-task-link-mark-swap 180ms cubic-bezier(0.2, 0.9, 0.3, 1.25)` (`scale 0.6 → 1`,
  `opacity 0.3 → 1`) only when a key that was already painted this session changes
  state. Examples: Ctrl+Enter elsewhere turns its links to ✓, or Alt+] turns ◉ into ◐.
  It never plays on first paint, startup, scroll, or rerender. A bounded per-key
  last-state map (about 2,000 keys, LRU) drives it.
- **Accessibility and print:**
  - `prefers-reduced-motion: reduce` turns off the animation and transitions.
  - In `forced-colors: active` the ink is `CanvasText`, so the shapes still carry state.
  - `print-color-adjust: exact`.
- **Gate:** use the mark-gate mirror (`applyMarkGateClass("bob-task-link-marks", on)`
  plus `registerMarkGatePopoutSync`) with
  `body:not(.bob-task-link-marks) .bob-task-link-mark { display: none; }`. The JS checks
  in the builder and post-processor stay the source of truth.

> [!decision] dim_closed_links Links whose ring is `done` or `cancelled` also get
> `.bob-task-link-closed` on the link text. In Live Preview this is a `Decoration.mark`
> over the link token. In Reading view it is a class on the `a.internal-link`. The class
> sets `opacity: 0.7`, with no strike, so authored `~~` keeps its meaning. It shares the
> ring's toggle. Add the vector TG7 "closed link text dims".

### Placement

- **Live Preview:** a zero-width `Decoration.widget` at the link token start. That is
  `[[` or `[`, or the opening `~~` of a struck link, so the strike never runs through
  the ring.
  - The widget takes no document positions, so vim motions are unchanged.
  - It covers no text, so it stays put when the cursor reveals raw `[[…]]`.
  - When another widget shares the offset (a Today mark on a task title that starts with
    a link), the ring is always **last**, closest to the link. Choose the `side` values
    so this holds, and pin it with a test.
- **Reading view:** insert the host immediately before the `a.internal-link`, or before
  its `del`/`s` wrapper when the anchor is that wrapper's only child.

### Tooltip

The tooltip is the element's `aria-label` plus `data-tooltip-position="top"` (Obsidian's
tooltip), with `role="img"`. There is no `title` attribute. The tooltip never contains
`::`, because Dataview re-scans `innerHTML`, so any stray `::` becomes `:`.

- **Line 1:** the target's title, cleaned with `dependencyCleanTaskText` and cut to 80
  characters (79 plus `…`).
- **Line 2:** `<Status> · <note>`, where Status is one of Ready, Next, In Progress,
  Blocked, Done, or Cancelled. `<note>` is the target file's basename without `.md`, or
  `this note` for a same-note target.
- When no title is known (metadata fallback), only line 2 shows.

Example:

```text
Move a task to a bullet
Ready · bob
```

### Interaction (the mark-family rule)

- **Plain click** never navigates and never writes. In Live Preview it places the caret
  just before the link (`view.posAtDOM`) and focuses the editor. In Reading view it does
  nothing.
- **Mod+click** (Cmd on macOS, Ctrl elsewhere, primary button) behaves exactly like
  Mod+clicking the link text: `workspace.openLinkText(linktext, sourcePath, "tab")`.
- No tab stop and no live-region announcement.

### Liveness

One coalesced `scheduleTaskLinkMarksRefresh()` (about 120 ms) runs on these triggers:

- the existing `scheduleLiveWidgetRefresh()` fan-out (the Tasks `cache-update` event);
- `metadataCache` `changed` and `resolved`;
- vault `rename` and `delete`;
- the toggle.

On refresh it does two things. It dispatches a `StateEffect` that rebuilds every Live
Preview editor's rings. It also sweeps stamped Reading-view anchors in every known
document (main window and popouts), adding, updating, or removing rings in place without
re-rendering the note. A Live Preview rebuild also happens on `docChanged`,
`viewportChanged`, a Live Preview ↔ source switch, and a file switch, the same as
`progressMarkShouldRebuild`. It never rebuilds on selection alone.

### Surfaces

- **Live Preview** of every Markdown note, including canvas cards that use the Markdown
  editor.
- **Reading view, block and note embeds, and hover popovers.** The transcluded content's
  own links resolve against the embedded file, because its post-processor context
  carries that file's path.
- **Tasks and Dataview query rows** (`dash`, `rotten`, `blocked`, and others): rings
  appear when Obsidian runs the post-processor on the row. Tasks renders descriptions
  with the task's own path, so links resolve correctly. If a row's path cannot be
  proven, it gets no ring.
- **Live Preview tables and callouts:** these render through Obsidian's embedded
  renderer. They get rings when that renderer runs post-processors with a source path,
  and no ring otherwise.
- **Source mode:** never.

### Performance

- Process only visible ranges. A per-line prefilter (`#^` or a case-insensitive `#%5E`)
  runs before any scan.
- Resolve each unique `(linkpath, sourcePath)` once per build.
- No disk reads in render, and no line-height change.
- The Reading sweep queries only stamped anchors.
- A node performance test builds a synthetic 1,000-line note with 2,000 task links. The
  viewport-limited build must stay under a generous bound (about 25 ms), in the style of
  the existing `*-performance` tests.

### Deferred (the contract lists these with their triggers)

- Readable titles: show a bare task link's text as the task title instead of
  `bob > ^id`.
- Kind glyphs inside the ring for `^prj`, `#ref`, and Event Tasks.
- A broken-block flag for links whose id has vanished.
- Rings in nav pickers, the Task Card, and Mac Capture candidates.
- Mod+click opening the Task Card.
- Ledger rows, if `ledger_rows = skip`.

### Conformance vectors (transcribed verbatim into the contract; tests run them verbatim)

**TS — scan.** `taskLinkMarkSpans(lineText)` returns
`[{at, linkpath, blockId, kind, alias}]`, where `at` is the UTF-16 offset where the ring
goes.

| #    | line                                             | result                                     |
| ---- | ------------------------------------------------ | ------------------------------------------ |
| TS1  | `See [[bob#^fix]] now`                           | one span: at 4, `bob`, `fix`, wiki         |
| TS2  | `See [[bob#^fix\|the fix]]`                      | one span: at 4, alias `the fix`            |
| TS3  | `[[#^fix]]`                                      | one span: at 0, linkpath empty (same note) |
| TS4  | `![[bob#^fix]]`                                  | none (embed)                               |
| TS5  | `~~[[bob#^fix]]~~`                               | one span: at 0 (before the strike)         |
| TS6  | `` `[[bob#^fix]]` ``                             | none (inline code)                         |
| TS7  | `[the fix](bob.md#^fix)`                         | one span: at 0, `bob.md`, `fix`, md        |
| TS8  | `[x](<my note.md#^fix>)`                         | one span: linkpath `my note.md`            |
| TS9  | `[x](my%20note.md#^fix)`                         | one span: linkpath `my note.md`            |
| TS10 | `[x](bob.md#%5Efix)`                             | one span: blockId `fix`                    |
| TS11 | `[[bob]] [[bob#Plan]] [x](https://e.com/a#^fix)` | none                                       |
| TS12 | `![x](bob.md#^fix)`                              | none (embed)                               |
| TS13 | `[[bob#^fix]] and [[#^ship]]`                    | two spans, in order                        |
| TS14 | `[[bob#^not an id]]`                             | none                                       |
| TS15 | `%%[[bob#^fix]]%%`                               | none (inline comment)                      |

**TC — context.** These are document-level line eligibility rules.

| #    | where                                                                                                                                   | ring?                     |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| TC1  | prose line in an area note                                                                                                              | yes                       |
| TC2  | inside a fenced code block                                                                                                              | no                        |
| TC3  | inside a multi-line `%% … %%` comment                                                                                                   | no                        |
| TC4  | frontmatter                                                                                                                             | no                        |
| TC5  | `⛓️ **DEPENDS ON:** [[#^a]]`, and the same for BLOCKS, LINKS, LINKED FROM, and Sub-projects rows (malformed included)                   | no                        |
| TC6  | a `#task` line whose title mentions `[[bob#^fix]]`                                                                                      | yes                       |
| TC7  | a Cancel, Schedule, Work, or Rotten Log entry                                                                                           | yes                       |
| TC8  | a daily-note ledger row whose body is `[[bob#^fix]]`, `🍅 [[bob#^fix]]`, `~~[[bob#^fix]]~~`, `🍅 ~~[[bob#^fix]]~~`, or `[[bob#^fix]] #` | no (`ledger_rows = skip`) |
| TC9  | a bullet nested under a ledger Task Link: `see [[bob#^fix]]`                                                                            | yes                       |
| TC10 | a ledger entry's direct child with mixed prose: `pair with [[bob#^fix]]`                                                                | yes                       |
| TC11 | `- [[bob#^fix]]` in an area note                                                                                                        | yes                       |
| TC12 | a daily note outside `## Pomodoros` (Notes, Events rows)                                                                                | yes                       |
| TC13 | a heading, blockquote, or callout line                                                                                                  | yes                       |

**TR — resolve and state.** Each row gives a target line, or a situation, and its
result.

| #    | target                                                             | result                                                                                                            |
| ---- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| TR1  | `- [ ] #task Fix ^fix`                                             | ready                                                                                                             |
| TR2  | `- [*] #task Fix ^fix`                                             | next                                                                                                              |
| TR3  | `- [/] #task Fix ^fix`                                             | in-progress                                                                                                       |
| TR4  | `- [?] #task Fix ^fix`                                             | blocked                                                                                                           |
| TR5  | `- [x] #task Fix ^fix` and `- [X] …`                               | done                                                                                                              |
| TR6  | `- [-] #task Fix ^fix`                                             | cancelled                                                                                                         |
| TR7  | `- [>] #task Fix ^fix`                                             | by Tasks status type (DONE gives done, CANCELLED gives cancelled, NON_TASK gives none, anything else gives ready) |
| TR8  | `- [ ] Pack charger ^fix` (no `#task`)                             | none                                                                                                              |
| TR9  | `- Plain bullet ^fix`                                              | none                                                                                                              |
| TR10 | paragraph `Some text ^fix`                                         | none                                                                                                              |
| TR11 | the id is not in the target note                                   | none                                                                                                              |
| TR12 | the note is unresolved                                             | none                                                                                                              |
| TR13 | the id ends two lines in the target note                           | none                                                                                                              |
| TR14 | ``- [ ] `#task` in code ^fix``                                     | none                                                                                                              |
| TR15 | Live Preview, same-note target edited from `[ ]` to `[x]`, unsaved | done on the same frame                                                                                            |
| TR16 | Tasks absent or not ready; metadata shows `[/]` and `#task`        | in-progress, tooltip without a title                                                                              |
| TR17 | a `^prj` task, a `#task #ref` reading task, or an Event Task       | ring by status (no kind glyph in v1)                                                                              |

**TT — tooltip.**

| #   | case                                                                          | aria-label                                |
| --- | ----------------------------------------------------------------------------- | ----------------------------------------- |
| TT1 | `- [/] #task Fix the parser crash ^fix` in `bob.md`, linked from another note | `Fix the parser crash\nIn Progress · bob` |
| TT2 | same-note Ready target                                                        | `<title>\nReady · this note`              |
| TT3 | target in `prj/rap/now_rap_target.md`                                         | `<title>\n<Status> · now_rap_target`      |
| TT4 | title with links, tags, fields, and `^id`                                     | cleaned per `dependencyCleanTaskText`     |
| TT5 | 95-character title                                                            | first 79 characters plus `…`              |
| TT6 | title unknown (TR16)                                                          | `In Progress · bob`                       |
| TT7 | a stray `::` left in the cleaned title                                        | becomes `:`                               |

**TG — glyph.**

| #   | state       | `data-state`  | glyph variable                    | ink variable                              |
| --- | ----------- | ------------- | --------------------------------- | ----------------------------------------- |
| TG1 | ready       | `ready`       | `--bob-task-link-glyph-ready`     | teal identity                             |
| TG2 | next        | `next`        | `--bob-task-link-glyph-next`      | `--task-status-next`                      |
| TG3 | in-progress | `in-progress` | `--bob-progress-glyph`            | `--task-status-in-progress`               |
| TG4 | blocked     | `blocked`     | `--bob-task-link-glyph-blocked`   | `--task-status-blocked`                   |
| TG5 | done        | `done`        | `--bob-task-link-glyph-done`      | `--task-status-done`, else `--text-muted` |
| TG6 | cancelled   | `cancelled`   | `--bob-task-link-glyph-cancelled` | `--task-status-cancelled`                 |

## Phase `core` — Contract, pure core, ring glyphs, and visual preview

**bob-cli docs:**

- Write `docs/task-link-marks.md` in the style of `docs/today-marks.md` and
  `docs/task-tag-marks.md`. It is the authoritative display contract and transcribes
  every section of the design above: what you see, principles, rules, data sources, the
  ring family, placement, tooltip, interaction, liveness, surfaces, performance,
  Deferred, and the TS/TC/TR/TT/TG vectors. Name the JS mirror fragments.
- Add the page to `mkdocs.yml` in both places where `task-tag-marks.md` and
  `today-marks.md` appear (the page list and the "Task link marks" nav entry).
- Add a table row and the inline mention near line 20 of `docs/README.md`.
- Add contract links and the Related list entry in `docs/reference/vault-format.md`.
- Add one sentence to `docs/plan.md` §In Progress marks saying that ◐ is the ring
  family's In Progress glyph and linking the new page.
- Run `just docs-check`.

**bob-plugins `src/148-task-link-marks.js`** is a pure core: no `app`, synchronous,
never throws. It adds:

- `taskLinkMarkSpans(lineText)`, the TS vectors. Reuse `relationInlineCodeRanges` and
  the `relationWikilinkSpans` approach. Add Markdown-link parsing: scheme rejection,
  `<…>`, percent-decoding, `%5E`, and image/embed rejection.
- Line-context helpers for TC:
  - a fence, comment, and frontmatter state walker;
  - managed rows through `mentionIsEdgeLine`;
  - `taskLinkMarkIsLedgerTaskLinkLine(lines, index, path)` for rule 5 (canonical daily
    path via `PLAN_DAILY_PATH_RE`, inside `## Pomodoros`, a direct child of a column-0
    entry, open or closed). Reuse the shape logic behind `progressMarkLinks` and
    `computeTodayLinks` where it fits, but rule 5 covers closed entries too.
- `taskLinkMarkParseTaskLine(lineText)` returns symbol, exact `#task` outside inline
  code, and trailing `^id`.
- `taskLinkMarkStateFromSymbol(symbol, statusType)` (TR1–TR7).
- `taskLinkMarkTooltip({ title, state, targetPath, sourcePath })` (TT).

**CSS:** add a "Task link marks" section to `plugins/bob-ledger-tools/styles.css` with
the glyph variables, host, per-state ink, hover, swap keyframes, reduced motion, forced
colors, print, and the gate. Also add the `dim_closed_links` class if chosen.

**Visual preview:** add `scripts/preview-task-link-marks.cjs` and a `.template.html`,
modeled on `preview-day-strip.cjs` and `preview-review-footer.cjs`. It renders every
state beside real link text in a paragraph, a heading, a task line (after the teal
hash), a blockquote, a struck link, and a dense sentence with three task links and two
non-task links, in light and dark. Capture screenshots the way the existing preview
scripts do. Tune geometry and metrics within the stated bounds until the rings sit on
the text's optical center line and read as one family with ◐. Then write the final
numbers into the contract.

**Tests:** add `scripts/test-ledger-tools-task-link-marks.cjs`, which runs the TS, TC,
pure TR (TR1–TR14, TR17 by parsing), TT, and TG vectors verbatim (TG checks the CSS text
for each variable). Add it to the `package.json` test list. Then run `npm run build`,
`npm test`, and `bob plugins sync`.

## Phase `live_preview` — Live Preview rings, resolver, refresh, and toggle

Add `src/284-plugin-task-link-marks.js`, `BobLedgerToolsTaskLinkMarksMixin`. Register it
in `310-install-methods.js` and `fragments.json`, and wire setup without growing
`170-plugin-lifecycle.js`. If the fragment nears 1000 lines, split the resolver into
`284a-plugin-task-link-marks-resolve.js`.

- **`setupTaskLinkMarks()`:**
  - `taskLinkMarksEnabled = true`;
  - the gate class, plus popout sync;
  - the `toggle-task-link-marks` command, "Toggle task link marks", which flips the flag
    and gate, refreshes editors and the sweep, and shows the Notice `Task link marks on`
    or `Task link marks off`;
  - the editor extension;
  - the `metadataCache` and vault listeners;
  - a `scheduleTaskLinkMarksRefresh()` call added to `scheduleLiveWidgetRefresh()` in
    `220-plugin-heading-and-review.js`.
- **`taskLinkMarkResolve({ linkpath, blockId, sourcePath, liveDoc })`** returns
  `{ key, state, title, targetPath } | null`, using the precedence above. Add the
  duplicate-key record to the `dependencyTasksIndex()` pass in
  `270-plugin-dependency-model.js` without changing its existing return shape.
- **The CM6 `ViewPlugin`** (`Prec.highest`, like progress marks):
  - Build over visible ranges with the context walker.
  - Use one `TaskLinkMarkWidget` per span. `eq` compares key, state, tooltip, and the
    swap flag.
  - `toDOM` builds the host.
  - Mouse handling follows the interaction rule.
  - Pin the ring-is-last ordering against a same-offset Today-mark widget.
- **Swap animation:** track painted state per key, play the swap only on a change, and
  never on first paint.
- **`dim_closed_links`:** if chosen, add the closed-link mark decoration here.

**Tests:** add `scripts/test-ledger-tools-task-link-marks-live.cjs` with a fake `app`
(metadataCache, a Tasks `getTasks()`, workspace) and a fake CM6 view, following the
progress-mark and today-mark Live Preview tests. It covers:

- TC vectors through the decoration builder;
- TR vectors through each data source, including TR13, TR15 (live doc), and TR16
  (fallback);
- TT through the widget DOM;
- the toggle;
- the refresh effect;
- no rebuild on selection-only updates;
- the ordering pin;
- the swap rule;
- the performance bound.

Add the file to `package.json`. Then run `npm run build`, `npm test`, and
`bob plugins sync`.

## Phase `reading` — Reading view, embeds, hover popovers, and query rows

Add `src/285-plugin-task-link-marks-reading.js`,
`BobLedgerToolsTaskLinkMarksReadingMixin`. Register it the same way, and call its setup
from `setupTaskLinkMarks()`.

- **Post-processor** `renderTaskLinkMarksIn(el, ctx)` (priority 50–55, like its
  siblings):
  - Select candidate `a.internal-link` elements whose `data-href` holds `#^`. Both
    wikilinks and Markdown links render this way.
  - Resolve each with `ctx.sourcePath` through `taskLinkMarkResolve` (no live doc).
  - Stamp the anchor (`data-bob-task-link-src` = source path) so the sweep can
    re-evaluate it.
  - Insert or update the host per Placement.
- **DOM exclusions**, mirroring the TC rules:
  - anchors inside `.internal-embed` or `.markdown-embed` link headers, properties
    (`.metadata-container`), or plugin chip rows (`.bob-dep-row`, `.bob-relation-row`,
    the BLOCKS row, and any other plugin widget);
  - rows whose own text (nested lists excluded) is a managed edge row;
  - ledger rows: generalize `progressMarkReadingContext` (open entries only) into a
    helper that matches dedicated Task Link rows under open **or** closed top-level
    Pomodoro entries in a `## Pomodoros` section of a daily note, using the same DOM
    ancestry approach, since full-note Reading view has no section info;
  - code and comments, which Obsidian never renders as links.
- **Sweep** `sweepTaskLinkMarksReading()`: walk every known document (reuse the
  `markGateDocuments` / `todayMarksReadingDocuments` enumeration), query stamped
  anchors, and add, update, or remove each ring in place. It runs from
  `scheduleTaskLinkMarksRefresh()` and the toggle.
- **Query rows:** verify against the Tasks and Dataview row DOM used by the date-mark
  and type-mark Tasks code (`267-plugin-date-marks-tasks.js`,
  `279-plugin-task-theme-tasks.js`) that the post-processor sees rows with the task's
  path. If a query renderer does not run post-processors, add a narrow hook alongside
  those existing Tasks-row hooks. If the row's path cannot be proven, the row gets no
  ring.

**Tests:** add `scripts/test-ledger-tools-task-link-marks-reading.cjs` in the FakeNode
style of `test-ledger-tools-today-marks-reading.cjs`. It covers:

- TC in DOM form (embeds, managed rows, ledger rows open and closed, nested and mixed
  rows, chips);
- the struck-wrapper placement;
- TR and TT through the DOM;
- sweep add, update, and remove when a target changes;
- the toggle;
- a Tasks row resolving against the task's path;
- `dim_closed_links` if chosen.

Add the file to `package.json`. Then run `npm run build`, `npm test`, and
`bob plugins sync`.

## Phase `landing` — Real-Obsidian e2e, docs, version, and sync

- **E2e:** add `scripts/obsidian-task-link-marks-e2e.cjs` and
  `scripts/test-obsidian-task-link-marks-e2e.cjs`, modeled on
  `obsidian-task-wikilink-e2e.cjs` and reusing `obsidian-startup-smoke-lib.cjs`. Use an
  isolated scratch vault and profile, and **never the live vault**. Run them with
  `xvfb-run -a`.
  - **Fixture:** an area note with tasks in all six states; a prose note linking each of
    them plus a heading link, a non-task block link, a plain-checklist link, an embed,
    and a struck link; managed Depends-On and LINKS rows; a daily note with a ledger and
    a `## Notes` section; and a Tasks query note.
  - **Assert** in Live Preview and Reading view:
    - the expected `data-state` per link, and no ring where the rules say none;
    - completing a task in its note flips its rings in the other open note within the
      refresh window, with the swap class seen once;
    - the toggle hides and restores rings;
    - a hover popover's links get rings;
    - Tasks query rows get rings;
    - **line boxes keep the same height** with rings on and off.
  - **Screenshots:** capture the prose note in light and dark for the final design
    review, and list their paths in the phase's completion notes.
  - If the host cannot run Obsidian under xvfb, say so plainly and state which
    assertions only node tests cover.
- **Fixes:** fix whatever the e2e finds, in the owning fragments.
- **Docs and version:**
  - In bob-plugins, add a "task link mark" bullet to "Marks and rendered blocks" in
    `docs/bob-ledger-tools.md`, linking `https://bobs-org.github.io/task-link-marks/`.
  - Add one sentence for task link marks to the manifest `description`.
  - Bump `manifest.json` to the next **minor** version (feature convention).
  - Run `npm run docs:gen` (facts and commands tables) and `npm run docs:check`.
  - In bob-cli, finalize the contract with any numbers the e2e tuned, then run
    `just docs-check`.
- **Ship:** run `npm run build`, `npm test`, `npm run validate`, and `bob plugins sync`.
  Confirm the vault copy carries `toggle-task-link-marks`.

## Verification (whole epic)

- `npm test` and `npm run validate` pass in bob-plugins, and `just docs-check` passes in
  bob-cli.
- The real-Obsidian e2e passes, or its skip is reported honestly.
- The light and dark screenshots show the six rings beside link text, aligned and quiet,
  with non-task links untouched.
- `bob plugins sync` has deployed the final build.
