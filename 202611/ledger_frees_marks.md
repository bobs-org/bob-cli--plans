---
tier: epic
title: Ledger frees marks
goal: 'In today''s daily note, each Task Link under an open Pomodoro whose task is
  the last blocker of other open tasks wears a quiet "🔑 frees N" mark. N always matches
  the task''s own BLOCKS row, the hover names the freed tasks, and the mark is verified
  in real Obsidian.

  '
phases:
- id: contract
  title: Authoritative display contract (bob-cli)
  depends_on: []
  size: small
  description: 'contract: write and register bob-cli docs/frees-marks.md (rule, look,
    placement, tooltip, motion, liveness, FL/FC/FT/FP vectors) and cross-link BLOCKS
    §14.9.'
- id: ledger_mark
  title: The mark in bob-ledger-tools (bob-plugins)
  depends_on:
  - contract
  size: medium
  description: 'ledger_mark: implement the pure core, Live Preview and Reading mixin,
    CSS, wiring, vector tests with BLOCKS parity, docs inventory, and manifest bump;
    build, test, sync.'
- id: live_verify
  title: Real-Obsidian e2e and visual review (bob-plugins, bob-cli)
  depends_on:
  - ledger_mark
  size: medium
  description: 'live_verify: add an isolated real-Obsidian e2e with light/dark screenshots,
    tune CSS from the review, and record live verification in the contract.'
decisions:
  pill_text:
    ask: Should the mark spell out the word "frees" next to the key?
    choices:
      words: 'Reads "🔑 frees 2": readable without hovering and word-for-word the BLOCKS
        row summary'
      compact: 'Reads "🔑 2": about half as wide, so you learn what it means from the
        tooltip'
    default: words
    why: The point is to catch your eye while you plan, and "frees 2" needs no legend.
    answer: compact
proposed_by: bbugyi200.athena.176
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 12:45:28
status: done
bead_id: bob-cli-g1
---

- **PROMPT:** [prompts/202611/ledger_frees_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/ledger_frees_marks.md)
- **BEAD:** [bob-cli-g1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-g1/README.md)

# Ledger frees marks: show what closing a planned task unlocks

## Goal

In today's daily note, each Task Link under an open Pomodoro whose task is the **last
blocker** of other open tasks gets a quiet trailing mark that says how many tasks
closing it would free:

```text
- [ ] () — READ
	- sase > ^ref-the-ai-native-sdlc-playbook
	- sase > ^ref-task-backlog-cleanup-chain-review 🔑 frees 1
```

Hovering the mark shows:

```text
Closing this frees 1 task
• Launch another epic chain for sase task beads! · sase
```

This is the forward-looking partner of the existing ledger marks. ◐ before a link means
"in progress now". 🔓 after a link means "something done today freed this task". The new
🔑 after a link means "finishing this frees N tasks". It helps Bryan put unlocking work
first when he plans the day.

## Design (decided)

### Vocabulary already in the vault, reused verbatim

- `🔑` is the BLOCKS-chip **last-blocker** badge. `frees N` / `frees N of M` is the
  BLOCKS row summary (`docs/task-dependencies.md` §14.9; bob-plugins
  `plugins/bob-ledger-tools/src/141-blocks-model.js`, `blocksLastBlocker`,
  `blocksDependentsIndex`, `blocksChipModel`).
- `🔓` is the unblocked hand-off glyph after a live ledger link
  (`139-unblocked-glyph.js`, `271-plugin-unblocked-glyph.js`). The new mark copies its
  link set, placement family, refresh triggers, and toggle pattern.
- The live-link set is the one ◐ and 🔓 use: `progressMarkLinks`
  (`137-progress-marks.js`).

So the new mark adds no new glyph, word, or hue. It brings the 🔑 and `frees N` that
Bryan already sees on a task's own BLOCKS row into the ledger, where he plans.

### What N counts (the one rule)

```text
X       = the task a qualifying ledger link resolves to (Tasks memo; see "Which links")
x       = X's [id::] value (Tasks `id`), trimmed. No [id::] → no mark: closing X can
          never unblock anyone (task-dependencies.md §12.2 GATE)
OPEN(t) = t.status.type ∈ {TODO, IN_PROGRESS, ON_HOLD}
LIVE(x) = blocksDependentsIndex(memo).get(x) minus X itself
          (open tasks whose dependsOn names x; exactly the BLOCKS row's live set)
FREED   = [D ∈ LIVE(x) : blocksLastBlocker(D, X, memo, memo.dateText).lastBlocker]
          (no other dependsOn id resolves to an open task, no scheduled date after
           today, no #prj_is_blocked: exactly the BLOCKS 🔑 set)
N       = |FREED|;  WAITING = |LIVE(x)| − N
mark   ⇔ OPEN(X) ∧ N ≥ 1
```

**Parity invariant.** For every task X, the pill's N equals the `lastCount` in X's own
BLOCKS row summary (`frees N`, or the N in `frees N of M`). Both read the same memo,
`this.freshnessEnsureMemo()`, through the same two helpers. A test asserts this on every
count fixture. The ledger and the task's own note can never disagree.

**What the number promises.** In the same Tasks snapshot, every counted task leaves
Blocked the moment X closes. The Ctrl+Enter successor picker offers each one as a
hand-off, with three exceptions, and the tooltip marks the first two:

- a `#hide` task is recovered but never offered;
- a `^prj` task counts as one, as it does on BLOCKS, and its open residents are offered
  as the `Project ‹note›` group;
- a task already linked today is not offered again.

**Deliberately not counted.** These never use 🔓/"unblocked" wording in the successor
copy rules (§12.6), so they don't belong in "frees" either:

- keep-moving offers (⏭, §12.15). These are already-Ready tasks of a leaf project,
  never dependents.
- relation successors (🪢 `starts after`, §12.18). These are never blocked, only
  sequenced. They are deferred as a possible separate 🪢 segment.
- project-finished pull-ins (§12.10). Closing X finishes the project; it frees nothing.
- dependents still waiting on other work. They show in the tooltip as `Also blocks …`,
  never in the pill.

### Which links get the mark

A link gets the mark only when **all** of these hold. They are the same gates the 🔓
glyph uses.

1. The note is **today's** canonical daily note (`currentTodayDailyPath`). Other daily
   notes never get it.
2. The link is a dedicated Task Link row that `progressMarkLinks` accepts. The row is a
   direct child, at the first child indent, of an **open** Pomodoro entry (running, or a
   planned `()` placeholder, named or unnamed). Its body is exactly one
   `[[target#^id]]`, with an optional alias, an optional leading `🍅`, and an optional
   trailing ` #` move-only marker. Excluded: struck `~~[[…]]~~`, embedded `![[…]]` (the
   transclusion already shows the task's own BLOCKS row), mixed prose, deeper bullets,
   fenced lines, closed entries, Events and every other section.
3. The target resolves through the Tasks memo to exactly one `#task`, using the same
   path/block-id resolution as `unblockedGlyphLinkedTask`. Same-note `[[#^id]]` targets
   resolve against the daily path. The target is OPEN and carries an `[id::]`.
4. N ≥ 1.
5. The Tasks memo is warm. A cold memo draws nothing.
6. The session toggle is on.

**Truthful or neutral.** Unresolved, ambiguous, closed, or id-less targets, and any
thrown error, show no mark. Nothing ever writes to the vault. Without the plugin, the
vault looks exactly as it does today.

### Anatomy and look

- Host element:
  `span.bob-frees-mark[role=img][aria-label][data-tooltip-position=top][data-count][data-block-id]`.
  It holds `span.bob-frees-mark-key` (the text `🔑`) followed by the text `frees N`
  (decision `pill_text`; `compact` renders just `N`). It has no `title` attribute and is
  listener-free, so it survives Dataview's innerHTML round-trip. A DOM builder takes a
  fake `doc`, as `buildProgressMarkElement` does.
- Quiet, muted, borderless at rest. The emoji key carries the only color, so no hue from
  the spent budget (`docs/task-tag-marks.md` → Ink) is used. Starting CSS, in
  `plugins/bob-ledger-tools/styles.css` after the unblocked-glyph block:

```css
.bob-frees-mark {
  display: inline-flex;
  align-items: baseline;
  gap: 0.22em;
  margin-inline-start: 0.4em;
  padding: 0 0.42em;
  border-radius: 999px;
  font-family: var(--font-interface);
  font-size: 0.8em;
  font-variant-numeric: tabular-nums;
  line-height: 1;
  white-space: nowrap;
  vertical-align: baseline;
  color: var(--bob-frees-mark-color, var(--text-muted));
  user-select: none;
  cursor: default;
  print-color-adjust: exact;
  transition:
    background-color 120ms ease,
    color 120ms ease;
}
.bob-frees-mark-key {
  font-size: 0.95em;
}
.bob-frees-mark:hover {
  color: var(--text-normal);
  background-color: color-mix(in srgb, currentColor 12%, transparent);
}
.bob-frees-mark.bob-frees-mark-tick {
  animation: bob-frees-mark-tick 180ms cubic-bezier(0.2, 0.9, 0.3, 1.25);
}
@keyframes bob-frees-mark-tick {
  from {
    opacity: 0.3;
    scale: 0.6;
  }
  to {
    opacity: 1;
    scale: 1;
  }
}
@media (prefers-reduced-motion: reduce) {
  .bob-frees-mark,
  .bob-frees-mark.bob-frees-mark-tick {
    animation: none;
    transition: none;
  }
}
@media (forced-colors: active) {
  .bob-frees-mark {
    color: CanvasText;
  }
}
body:not(.bob-frees-marks) .bob-frees-mark {
  display: none;
}
```

The mark never changes line height, never wraps inside itself, and never takes document
positions. `--bob-frees-mark-color` is a theme hook that is unset by default.

### Placement

- **Live Preview:** a zero-width `Decoration.widget` at the link token's **end** (just
  after `]]`, before any trailing ` #`), with `side: 2`. 🔓 sits at the same offset with
  `side: 1`, so a link that has both reads `link 🔓 🔑 frees N`: the past (freed by),
  then the future (frees). The widget covers no text, so it stays put while the cursor
  reveals raw `[[…]]`, and it never rebuilds on selection alone.
- **Reading view:** insert the host right after the `span.bob-unblocked-glyph` that
  follows the link if one is present. Otherwise insert it right after the
  `a.internal-link`, or after its `del`/`s` wrapper when the anchor is that wrapper's
  only child.

### Tooltip (`aria-label`, Obsidian tooltip, never contains `::`)

```text
Closing this frees N task|tasks
• ‹title› · ‹note›                    (up to 3 rows, BLOCKS canonical order)
• 🧩 ‹project note› project            (a ^prj dependent)
• ‹title› · ‹note› · hidden            (a #hide dependent: freed, not offered)
+K more                                 (when N > 3)
Also blocks W waiting on other work     (when WAITING = W ≥ 1)
```

- Titles are cleaned with `dependencyCleanTaskText`, then cut to 48 characters (47 plus
  `…`), following the §12.6 copy rule.
- `‹note›` is the dependent's file basename without `.md`, or `this note` when it lives
  in today's daily note.
- Rows use the same order as `blocksSortLinks`, so the tooltip lists tasks in the order
  X's BLOCKS row shows them.
- Any stray `::` becomes `:`.

### Interaction

- **Plain click (mousedown):** place the caret at the widget position (the link end) and
  focus the editor. Never navigate, never write. This matches ◐/🔓.
- **Mod+click (primary button):** `workspace.openLinkText(linktext, sourcePath, "tab")`,
  the same as Mod-clicking the link. It opens X's block, where the BLOCKS row lists the
  🔑 dependents.
- No tab stop and no live-region announcement.

### Motion

- No animation on first paint, startup, scroll, or rerender.
- Each editor view keeps a bounded LRU map (about 2,000 keys) of
  `path\0blockId → last N`, for every eligible link it has evaluated, including N = 0.
  When a key already in the map changes to a new N ≥ 1, the host gets
  `bob-frees-mark-tick` for about 400 ms. So a pill that appears or changes count
  because of an edit elsewhere pops once. A pill whose count drops to 0 simply goes
  away.
- Reading view never animates.
- Reduced motion turns this off. Copy the `task-link-marks` swap machinery
  (`149-task-link-mark-live-model.js`) where it fits.

### Liveness

One coalesced `scheduleFreesMarksRefresh()` (about 120 ms) runs on:

- the `scheduleLiveWidgetRefresh()` fan-out (the Tasks `cache-update`), where dependents
  in other notes change;
- metadataCache `changed` for today's daily path;
- the local-midnight rollover;
- the toggle.

Live Preview also rebuilds on `docChanged`, `viewportChanged`, and the Live Preview ↔
source switch. The Reading-view refresh sweeps stamped hosts in today's daily note,
adding, updating, or removing them in place. Copy the 🔓 mixin's structure method for
method.

### Toggle

The command `toggle-frees-marks` is named "Toggle Task Link frees marks". It is a
session toggle, default on, using `applyMarkGateClass("bob-frees-marks", on)` plus
`registerMarkGatePopoutSync`. v1 has no api namespace.

### Performance

- Build the dependents index once per memo generation (it is already cached on the memo
  by `blocksDependentsIndex`).
- Resolve each unique `(linkpath, blockId)` once per build.
- Build only for visible ranges.
- Never read from disk during render.
- A node perf test with a synthetic 200-link ledger and 2,000-task memo keeps one build
  under about 25 ms.

### Deferred (record in the contract with triggers)

- A separate `🪢 N` "starts after this" segment for relation successors. Trigger:
  `relations.successor_labels` is configured and Bryan asks.
- Echoes in Bob Mac Capture's idle agenda, the Task Card, and nav pickers. Trigger:
  Bryan asks. It needs a Rust/JSON count, because the Mac is a thin client.
- Joint unlocks: "frees 1 with ^y" when two of today's planned tasks together free a
  dependent.
- Links under closed entries, other daily notes, and prose links.
- Counting a freed project's residents instead of the project as one.

## Conformance vectors (the contract copies these; JS tests run them verbatim)

**FL — which links.** Each row is today's daily note `## Pomodoros`, and its target X is
open, has an `[id::]`, and frees 1 unless stated otherwise.

| #    | Ledger                                                        | Mark?                                    |
| ---- | ------------------------------------------------------------- | ---------------------------------------- |
| FL1  | `- [ ] () — READ` / `\t- [[sase#^a]]`                         | yes                                      |
| FL2  | `- [ ] (**1210-1235** [t:: 25m]) — BUILD` / `\t- [[sase#^a]]` | yes                                      |
| FL3  | `- [x] (**0800-0815** [t:: 15m])` / `\t- [[sase#^a]]`         | no (closed entry)                        |
| FL4  | open entry / `\t- 🍅 [[sase#^a]]`                             | yes                                      |
| FL5  | open entry / `\t- [[sase#^a]] #`                              | yes; widget at the link end, before ` #` |
| FL6  | open entry / `\t- ~~[[sase#^a]]~~`                            | no                                       |
| FL7  | open entry / `\t- ![[sase#^a]]`                               | no                                       |
| FL8  | open entry / `\t- read [[sase#^a]] first`                     | no                                       |
| FL9  | open entry / `\t\t- [[sase#^a]]` under another child          | no                                       |
| FL10 | same ledger in a daily note that is not today's               | no                                       |
| FL11 | `## Events` row linking `[[sase#^a]]`                         | no                                       |
| FL12 | open entry / `\t- [[sase#^a\|Read it]]`                       | yes                                      |
| FL13 | open entry / `\t- [[#^a]]`, X in the daily note itself        | yes                                      |
| FL14 | open entry inside a fenced block                              | no                                       |
| FL15 | Tasks memo cold (no tasks)                                    | no                                       |
| FL16 | toggle off                                                    | no (CSS gate and JS)                     |

**FC — count.** X is `- [/] #task Read it [id:: x] ^a` in `sase.md`. Today is
2026-11-02.

| #    | Dependents (all `#task`)                              | N / WAITING                                      | Mark                                                                  |
| ---- | ----------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------- |
| FC1  | `[?] D [dependsOn:: x]`                               | 1 / 0                                            | `🔑 frees 1`                                                          |
| FC2  | `[?] D [dependsOn:: x,y]`, y open                     | 0 / 1                                            | none                                                                  |
| FC3  | `[?] D [dependsOn:: x,y]`, y done                     | 1 / 0                                            | `🔑 frees 1`                                                          |
| FC4  | `[?] D [dependsOn:: x] [scheduled:: 2026-11-03]`      | 0 / 1                                            | none                                                                  |
| FC5  | `[?] D [dependsOn:: x] [scheduled:: 2026-11-02]`      | 1 / 0                                            | `🔑 frees 1`                                                          |
| FC6  | `[?] D [dependsOn:: x] #prj_is_blocked`               | 0 / 1                                            | none                                                                  |
| FC7  | `^prj` task of `sase_blog.md` with `[dependsOn:: x]`  | 1 / 0                                            | `🔑 frees 1`; tooltip row `🧩 sase_blog project`                      |
| FC8  | `[?] D #hide [dependsOn:: x]`                         | 1 / 0                                            | `🔑 frees 1`; tooltip row ends `· hidden`                             |
| FC9  | X has no `[id::]`                                     | —                                                | none                                                                  |
| FC10 | X is `[x]`                                            | —                                                | none                                                                  |
| FC11 | `[x] D [dependsOn:: x]` and `[-] E [dependsOn:: x]`   | 0 / 0                                            | none                                                                  |
| FC12 | `[?] D [dependsOn:: x,zzz]`, zzz resolves to nothing  | 1 / 0                                            | `🔑 frees 1` (unresolved never blocks, R4)                            |
| FC13 | 3 freeing, 2 waiting                                  | 3 / 2                                            | `🔑 frees 3`; tooltip lists 3 + `Also blocks 2 waiting on other work` |
| FC14 | 5 freeing                                             | 5 / 0                                            | `🔑 frees 5`; tooltip lists 3 + `+2 more`                             |
| FC15 | stale `[ ] D [dependsOn:: x]` (reconcile not yet run) | 1 / 0                                            | `🔑 frees 1` (matches SL15: still offered)                            |
| FC16 | X names itself: `[dependsOn:: x]` on X                | 0 / 0                                            | none                                                                  |
| FC17 | every FC row                                          | pill N == `blocksChipModel(...)` lastCount for X | parity                                                                |

**FT — tooltip.**

| #   | Case                                                                   | aria-label                                                                           |
| --- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| FT1 | FC1, D = `Launch another epic chain for sase task beads!` in `sase.md` | `Closing this frees 1 task\n• Launch another epic chain for sase task beads! · sase` |
| FT2 | N = 2, one in the daily note                                           | second row ends `· this note`; header says `2 tasks`                                 |
| FT3 | a 60-character title                                                   | first 47 characters plus `…`                                                         |
| FT4 | a title with links, tags, fields, `^id`                                | cleaned per `dependencyCleanTaskText`                                                |
| FT5 | a stray `::`                                                           | becomes `:`                                                                          |
| FT6 | WAITING = 1                                                            | last line `Also blocks 1 waiting on other work`                                      |

**FP — placement.**

| #   | Case                                     | Result                                                   |
| --- | ---------------------------------------- | -------------------------------------------------------- |
| FP1 | Live Preview                             | widget at the link end, `side: 2`; no line-height change |
| FP2 | the same link also has 🔓                | order: link, 🔓, 🔑 mark                                 |
| FP3 | Reading view, no 🔓                      | host directly after `a.internal-link`                    |
| FP4 | Reading view with 🔓                     | host directly after `span.bob-unblocked-glyph`           |
| FP5 | cursor on the row (raw `[[…]]` revealed) | mark still at the link end                               |

## Phases

### Phase `contract` — authoritative display contract (bob-cli)

Write `docs/frees-marks.md` in bob-cli. It is the authoritative display contract and
copies the style of `docs/task-link-marks.md` and `docs/today-marks.md`. Include these
sections:

- What you see (both mock blocks above);
- Principles (display-only; truthful or neutral; one rule shared with BLOCKS; the
  promise and its three exceptions; what is deliberately not counted, and why);
- Which links;
- The count rule (the code block above);
- Anatomy and look (host element, metrics, ink, gate);
- Placement, Tooltip, Interaction, Motion, Liveness, Toggle, Performance;
- Deferred;
- Conformance vectors (FL, FC, FT, FP, copied exactly from this plan).

Apply decision `pill_text` everywhere the visible text appears. Name the JavaScript
mirror files the `ledger_mark` phase will add (core `src/141b-frees-marks.js`, mixin
`src/271b-plugin-frees-marks.js`, test `scripts/test-ledger-tools-frees-marks.cjs`).
State that there is no api namespace.

Register the page:

- `docs/_meta/contracts.yml`, in the "Tasks and projects" group next to
  `task-link-marks`/`today-marks`, with a summary like "Frees marks: a quiet
  `🔑 frees N` after Task Links in today's ledger whose task is the last blocker of N
  open tasks";
- both `mkdocs.yml` spots where `task-link-marks.md` appears;
- `docs/_meta/url_contract.txt` if the gate requires every published URL there.

Add a one-bullet cross-link at the end of `docs/task-dependencies.md` §14.9 (BLOCKS
chips): "Ledger echo: the same 🔑 set drives the frees mark after Task Links in today's
ledger ([Frees marks](frees-marks.md))". Do not edit guides or memory. Run
`just docs-gen` if registry changes need regenerated indexes, then `just docs-check`
until it passes. Commit in bob-cli as
`docs(frees-marks): add ledger frees-mark display contract`. Do not touch `Cargo.toml`
or `CHANGELOG.md`.

### Phase `ledger_mark` — the mark in bob-ledger-tools (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Implement
`docs/frees-marks.md` from bob-cli exactly. The contract wins; if it is wrong, fix the
contract in bob-cli in the same phase.

- **Core fragment** `plugins/bob-ledger-tools/src/141b-frees-marks.js`. It is pure,
  synchronous, and never throws, in the style of `139-unblocked-glyph.js`:
  - `freesMarkLinks(lines, memo, today, {dailyPath})` returns
    `[{line, ch (link end), target, blockId, count, waiting, freed:[…]}]`. It builds on
    `progressMarkLinks` and on resolution through memo tasks with the
    `unblockedGlyphLinkedTask` rules.
  - `freesMarkCount(owner, memo, today)` uses `blocksDependentsIndex` and
    `blocksLastBlocker`, excludes the owner itself, and orders rows by
    `blocksSortLinks`.
  - `freesMarkTooltip(model)`.
  - `buildFreesMarkElement(doc, model)`, listener-free.
  - `FreesMarkWidget` (`eq` on blockId + count + tooltip). Its `toDOM` adds only the
    mousedown caret-place and Mod+click open.

  Keep each fragment at or below 1000 lines. Add new fragments to `src/fragments.json`
  in source order: 141b after `141-blocks-model.js`, 271b after
  `271-plugin-unblocked-glyph.js`.

- **Mixin** `src/271b-plugin-frees-marks.js` (`BobLedgerToolsFreesMarksMixin`). Copy
  `271-plugin-unblocked-glyph.js` method for method:
  - setup, available, extension, shouldRebuild, decorations built with `side: 2` at the
    link end, toggle, editor refresh, schedule (about 120 ms);
  - changed-file and rollover hooks;
  - the Reading post-processor and sweep (placement FP3/FP4);
  - the per-editor count LRU that drives `bob-frees-mark-tick`.

  Read data from `this.freshnessEnsureMemo()` (the memo BLOCKS rows use), never
  `planBlockTasks`, so the parity invariant holds. If the mixin passes 1000 lines, split
  Reading view into `271c-plugin-frees-marks-reading.js`.

- **Wiring:**
  - install the mixin in `src/310-install-methods.js`;
  - call `setupFreesMarks()` beside `setupUnblockedGlyph()` in
    `src/170-plugin-lifecycle.js`;
  - add the changed-file and rollover calls beside the unblocked-glyph ones there;
  - call `scheduleFreesMarksRefresh()` beside `scheduleUnblockedGlyphRefresh()` in
    `scheduleLiveWidgetRefresh()` (`src/220-plugin-heading-and-review.js`);
  - clear the timer on unload.
- **CSS:** the block above in `plugins/bob-ledger-tools/styles.css`, placed after the
  unblocked-glyph block.
- **Tests:** add `scripts/test-ledger-tools-frees-marks.cjs`, modeled on
  `scripts/test-ledger-tools-unblocked-glyph.cjs`, to the `npm test` list in
  `package.json`. It must cover:
  - every FL, FC, FT, and FP vector, each cited by id in a comment;
  - the parity check (FC17) against `blocksChipModel`;
  - widget `eq`;
  - no `::` in tooltips;
  - the tick map: first paint, no tick; a count change, tick; a drop to 0, no host;
  - the reading insertion order with and without 🔓;
  - toggle gating;
  - the perf bound.
- **Docs and metadata:**
  - add one bullet for the frees mark to `docs/bob-ledger-tools.md` "Marks and rendered
    blocks", next to the unblocked-glyph and In Progress mentions, linking the bob-cli
    contract;
  - add one sentence to "Session toggles" if it fits;
  - run `npm run docs:gen` so the generated command table gains `toggle-frees-marks`;
  - bump `plugins/bob-ledger-tools/manifest.json` to the next minor version, as sibling
    feature commits do;
  - add one sentence to its description: "Render a quiet 🔑 frees N mark after Task
    Links under today's open Pomodoros whose task is the last blocker of N open tasks,
    with a tooltip naming them and a session toggle."
- **Gate:** `npm run build`, then `npm test`, `npm run validate`, and
  `npm run docs:check`, all green. Then `bob plugins sync` to deploy into the vault.
  Commit in bob-plugins as
  `feat(ledger-tools): show 🔑 frees N on today's ledger Task Links`.
- **Smoke against the real vault (read-only):** with the synced plugin, confirm from the
  model in a node one-off over the vault's current Tasks-shaped data, or simply by
  reasoning from file contents, that today's ledger would mark
  `[[sase#^ref-task-backlog-cleanup-chain-review]]` with `frees 1` (its dependent
  `^task-bead-epic-chain` waits only on it). Report the result. Never edit the vault.

### Phase `live_verify` — real-Obsidian e2e and visual review (bob-plugins, bob-cli)

Add `scripts/obsidian-frees-marks-e2e.cjs`, plus `-e2e-lib.cjs` if useful, and an
offline `scripts/test-obsidian-frees-marks-e2e.cjs` `--self-test` wrapper in the
`npm test` list. Adapt `scripts/obsidian-task-link-marks-e2e.cjs` and
`scripts/obsidian-block-marks-e2e.cjs`: an isolated scratch vault and profile, an owned
Obsidian process, CDP probes, `xvfb-run -a`, and never the live vault.

The fixture vault has:

- today's daily note with open and closed entries;
- tasks covering FC1, FC2, FC7, FC8, FC13, and FC14;
- one link that also earns 🔓 (a prerequisite done today);
- a same-note target.

Assert, in both Live Preview and Reading view and in light and dark themes:

- each expected mark and its text;
- the DOM order link → 🔓 → 🔑;
- no row-height change with the toggle on vs off;
- the hover tooltip text through a CDP hover, read from the visible `.tooltip`;
- plain click places the caret and writes nothing (file bytes unchanged);
- Mod+click opens X's note;
- an edit that adds a second freed dependent elsewhere changes the count within about a
  second, and the tick class appears once;
- the toggle removes and restores every mark;
- closed-entry, struck, embed, and Events links stay bare.

Take screenshots of a representative ledger in both themes. Review them critically
against the "Anatomy and look" goals: quiet at rest, legible, the key aligned with the
text baseline, the gap from 🔓 balanced, and hover clear. Tune the CSS within the
contract's spirit, and update the contract's metrics if values change.

Append a "Live verification" section to bob-cli `docs/frees-marks.md`, in the style of
`docs/blocked-marks.md`, recording the pass count, environment, and anything still
pending. Run `npm test`, `npm run validate`, and `bob plugins sync` in bob-plugins, and
`just docs-check` in bob-cli. Commit each repo:
`test(ledger-tools): real-Obsidian e2e for frees marks` and
`docs(frees-marks): record live verification`.
