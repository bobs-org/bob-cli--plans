---
tier: epic
title: Walker footers follow the active bracket walk
goal: "Obsidian's status bar shows exactly one walker footer that matches the persisted
  bracket walk: the review footer only while `s` is active (with a calm clear state
  instead of hiding), a new lane footer for `w`, a new links compass for `n`, a quiet
  review-due dot on the selector's `s` letter, byte-compatible legacy fallbacks, updated
  docs, and a published Walker Footer glossary term.

  "
phases:
  - id: navapi
    title: nav api.walkMode v1 and the walk-mode doorbell
    depends_on: []
    size: small
    description:
      "navapi: expose the persisted bracket-repeat mode through an additive nav api
      namespace (current/select) plus a workspace doorbell event fired on mode change
      and once at setup; no visible change."
  - id: host
    title: Ledger walker-footer host and the gated s panel
    depends_on:
      - navapi
    size: medium
    description:
      "host: route the existing ledger status item by walk mode, show the review footer
      only while s is selected with a Review-clear state and ]] next hints, hide on
      w/n/unset for now, and keep a byte-compatible legacy fallback when the nav api is
      absent."
  - id: lanes
    title: Lane footer for the w walk
    depends_on:
      - host
    size: medium
    description:
      "lanes: add the w panel as a pure view over api.laneWalk v2 queue counts
      (WIP/NEXT/READY with n/cap only when over, TODAY meter), with validated-row
      positions, lane answer hints, toast-matching clear states, and a dash-parity test."
  - id: links
    title: nav links() and the links footer for the n walk
    depends_on:
      - lanes
    size: medium
    description:
      "links: add a read-only walkMode.links() preview built on nav's own prev/next
      matcher and an n panel destination compass with directional arms, dashed create
      targets, generation-checked async, and a parity test against openLabeledBodyLink."
  - id: dot
    title: Review attention dot on the selector's s
    depends_on:
      - links
    size: small
    description:
      "dot: ledger pushes a display-only attention value when review commitments are due
      off-s; nav paints a tiny tier-tinted dot on the selector's s letter with
      tooltip/aria counts, forced-colors support, and no motion."
  - id: visual
    title: Visual previews, e2e, and placement
    depends_on:
      - dot
    size: medium
    description:
      "visual: generalize the review-footer preview script into a walker preview
      covering every panel state, add motion/accessibility polish and e2e checks
      (selector rect stable across s/w/n), and verify the adjacent-to-selector placement
      in real Obsidian with current placement as fallback."
  - id: ship
    title: Docs, memory, and deploy
    depends_on:
      - visual
    size: medium
    description:
      "ship: update bob-plugins and bob-cli docs that say the review footer is
      persistent, publish the Walker Footer glossary strand and the two strand
      amendments per the accepted memory decisions, optionally record the decision
      record, and deploy with bob plugins sync."
decisions:
  mem_walker_footer:
    ask: Publish the new Walker Footer glossary strand?
    memory:
      - glossary:walker-footer
    requested:
      We should add an excellent new "walker footer" glossary term to describe these
      footers (note that a "review footer" should be considered a type of "walker
      footer").
    default: true
    answer: true
  mem_strand_amends:
    ask:
      Amend the review-footer and task-freshness strands to describe the gated s member?
    memory:
      - glossary:morning-gtd-review-footer
      - glossary:task-freshness
    corrects:
      When nothing is due, it hides unless the cursor is on an open task with a
      priority; then it shows only `⟳ Nothing due` plus that preview.
    default: true
    answer: true
  mem_decision_record:
    ask:
      Also record a walker-footer-follows-bracket-walk decision record in the decisions
      web?
    memory:
      - decisions:walker-footer-follows-bracket-walk
    default: false
    answer: false
proposed_by: bbugyi200.athena.1bu
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 12:30:10
status: wip
---

- **PROMPT:**
  [prompts/202611/walker_footers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/walker_footers.md)

# Plan: Walker footers follow the active bracket walk

## Why

The `[[ s w n ]]` selector in Obsidian's status bar is the truth about what `]]` / `[[`
will do, yet the big review footer beside it always narrates morning review: after `]w`,
the bar says `⟳ Review 12 due` while `]]` walks lanes. This epic makes the status bar an
honest instrument: one **walker footer** whose panel follows the persisted bracket walk
— the existing review footer for `s`, a new lane footer for `w`, a new links compass for
`n` — plus a quiet dot on the selector's `s` so due review commitments are never
invisible.

The design is settled. Research report
`research:202611/bracket_walk_footers/bracket_walk_footers.md` (read it with
`sase artifact read` before implementing any phase — it carries the full rationale,
state tables, and a rendered mockup) consolidated five independent reviews, and Bryan
accepted **all** of its requirement adjustments A1–A13. The binding ones:

- **A1** One walker-footer status-bar item with three panels — never three items.
- **A2** "Active" means the **persisted bracket family** (`bracketRepeatType`), never a
  session, timer, or cursor inference.
- **A3** The `n` panel is a destination compass: no invented rank/total.
- **A4** Quiet tier-tinted dot on the selector's `s` when review **commitments** are due
  while another family is selected (ROTTEN-only/POST-only never light it). No `w` pip.
- **A5** A selected walk always paints a panel, including calm clear states; hidden only
  when data is untrustworthy (Tasks indexing / Today loading) or on mobile.
- **A6** Unset mode (fresh device) shows no panel; the selector's unselected state and
  the existing empty-repeat notice stand; the dot still works.
- **A7** Lane counts are walk membership (`queue.counts.pending/next/ready`, Today
  excluded, NEW/ROTTEN Ready excluded); caps in the tooltip; `n/cap` shown only when a
  lane is over its cap, in the existing over-cap tone (text carries the warning).
- **A8** Alt+P preview scope: `s` keeps everything; `w` keeps the off-row
  prioritized-cursor preview but a landed lane row's hint slot prefers the lane's own
  answer (`Alt+N release` / `Alt+N next`); `n` drops it.
- **A9** Never auto-switch the mode; internal advances never steal selection.
- **A10** Placement target: panel immediately left of the selector (CSS `order`) so
  panel width changes never move the click target — verified in real Obsidian, with
  current placement as the fallback (the Keep item's CSS documents a layout dependency
  on the footer host's `margin-right: auto`).
- **A11** Legacy fallback: nav api absent/old → ledger paints today's review footer
  exactly as now; ledger absent → nav's selector and commands keep working alone. Each
  plugin ships and rolls back alone.
- **A12** Hints read `]] next` (was `]s next`) in every panel; family keys stay in
  tooltips. Legacy fallback keeps `]s next`.
- **A13** One glossary parent term (Walker Footer) with the review footer amended as its
  `s` member (see the `ship` phase and the memory decisions).

## Where the code lives

All plugin work is in the **bob-plugins** linked repo — open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and follow its `AGENTS.md`: edit `src/`
fragments only (never generated `main.js`), keep each fragment at or below 1000 lines,
run `npm run build` after fragment edits and `npm run build:check` before finishing, and
run `bob plugins sync` after any change (follow `AGENTS.md` exactly if sync refuses an
overwrite). Unit tests run with `node --test scripts/<file>.cjs`; register new test
files in `package.json`'s `test:unit` list.

Verified anchors (bob-plugins at `f714fee`; only an albums commit landed after the
research's verified snapshot, so every research citation still holds):

| Fact                                                                                                                                                                                                                                                 | Anchor                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bracket mode `bracketRepeatType` (`s`/`w`/`n`/null), `rememberBracketRepeatType` acts only on change, storage key `bob-navigation-hotkeys-bracket-repeat`, `repeatBracketWalk` dispatches `jumpToDueTask` / `jumpToLaneTask` / `openLabeledBodyLink` | nav `src/546-plugin-bracket-repeat.js` (150 lines)                                                                                                                      |
| Only family commands and selector letter clicks record the mode; `]]`/`[[` repeats never change it                                                                                                                                                   | nav `src/490-plugin-lifecycle.js`, `546`                                                                                                                                |
| `[[ s w n ]]` selector item with sliding thumb, flashes, click-selects-without-walking                                                                                                                                                               | nav `src/547-bracket-mode-indicator.js` (870 lines)                                                                                                                     |
| nav api v3 with per-member versioned namespaces (`quickRoll`, `notice`, `successors`, ...)                                                                                                                                                           | nav `src/480-review-jump-and-nav-api.js:880-996`                                                                                                                        |
| `]n`/`[n` first-match prev/next/arrow-label body-link matcher, skipping frontmatter and fences; missing period notes create through Bob                                                                                                              | nav `src/680-plugin-link-parse.js`, `670-plugin-project-files.js`                                                                                                       |
| Review footer view-model: hides when Tasks unavailable; hint-only `⟳ Nothing due · <Alt+P preview>` on a prioritized cursor row when queue is empty; `]s next` default hint                                                                          | ledger `src/120-freshness-footer.js:680-760` (925 lines — new models go in new fragments)                                                                               |
| Footer painter + fit/shed choreography                                                                                                                                                                                                               | ledger `src/121-freshness-footer-paint.js`                                                                                                                              |
| Status item setup adds `.bob-freshness` classes                                                                                                                                                                                                      | ledger `src/240-plugin-freshness-status.js`                                                                                                                             |
| `api.laneWalk` v2 `queue()` → frozen `{entries, counts}`; counts `{pending, next, ready, total, todayPending, todayNext, todayReady, reviewReady}`; memoized on Tasks identity, date, Today generation, freshness snapshot                           | ledger `src/071-lane-walk-model.js`, `src/180-plugin-plan-and-ready.js:900-962`                                                                                         |
| Footer host CSS `margin-right: auto; flex: 1 1 auto`; Keep inbox item "never flexes so the footer shrinks first"                                                                                                                                     | ledger `styles.css:324-330`, `:7293-7295`                                                                                                                               |
| Ledger loads **before** nav in the live vault                                                                                                                                                                                                        | `~/bob/.obsidian/community-plugins.json`                                                                                                                                |
| Existing tests to extend                                                                                                                                                                                                                             | `scripts/test-navigation-bracket-repeat.cjs`, `test-navigation-bracket-mode-indicator.cjs`, `test-ledger-tools-freshness-footer.cjs`, `test-ledger-tools-lane-walk.cjs` |
| Preview script to generalize                                                                                                                                                                                                                         | `scripts/preview-review-footer.cjs`                                                                                                                                     |

Docs and memory work in the `ship` phase additionally touches this bob-cli repo's
`docs/` tree and `sase/memory/` (per the plan's memory decisions only).

## Shared design contract (binding for every phase)

**One anatomy, three panels.** Every panel fills the same slots left to right — glyph +
position (a button that acts like `]]`), current group, detail, hint, remaining groups,
meter — and sheds end-first as the bar narrows (remaining groups first, marked `…`, then
detail, then hint, then meter), with **per-panel drop lists** (do not blindly reuse the
review fit's TICK answer-chip exceptions). Queue fractions are positions in a changing
queue, never completion percentages: no progress bars, no percent done. Ellipsis must
never make a missing link destination look present.

**The nav contract.** One additive namespace on nav api v3 (top-level `version`
unchanged), following the existing per-member versioning pattern:

```js
walkMode: {
  version: 1,                // navapi phase; links phase bumps to 2; dot phase to 3
  current(),                 // "s" | "w" | "n" | null — readBracketRepeatType()
  select(type),              // same as a letter click: records, never walks
  links({ path, content }),  // v2 — { ok, prev: {text,target,path,exists}|null, next: … }
                             //      uses the ]n matcher; read-only, never creates
  setAttention(type, value), // v3 — value: null | { tier, label }; display-only
}
```

Plus a **doorbell** workspace event `bob-navigation-hotkeys:walk-mode` fired by
`rememberBracketRepeatType` on change and once at setup after reading storage. Receivers
re-read `current()` so a stale payload can never win. Do not expose the raw
local-storage key as a contract; do not mirror selection into ledger settings; a failing
listener must never break a command. Ledger feature-detects `api.walkMode?.version >= N`
per member.

**Reliability invariants** (every phase's tests must respect these):

1. One snapshot per paint — counts and positions come from the same immutable queue
   result; never today's counts with yesterday's rank.
2. Generation-checked async — a link-preview response paints only if family, source
   file, and editor revision are still current; switching family invalidates an
   outstanding read; unload invalidates everything.
3. Mode paint is synchronous; data work is coalesced — the mode event bypasses the 150
   ms data debounce; cursor updates keep the 50 ms context path; no polling, no
   full-vault rescans.
4. Lifecycle explicit — subscribe at setup then read once; test both plugin load orders,
   unload/reload, missing APIs, thrown listeners, no duplicate status items; a missing
   provider is "unavailable", never "the mode is s".
5. Activation rechecks context — clicks route through existing commands with a mode
   guard; a direction click revalidates note, revision, and destination before opening
   (and never creates a note from a vanished preview).
6. Read-only means rendering — paint, hover, focus, resize, and mode selection never
   write tasks or create notes; only explicit activation may, via existing commands.
7. Selector geometry stable — the selector's bounding rect is unchanged across `s` → `w`
   → `n`.

Phases are chained (`navapi → host → lanes → links → dot → visual → ship`) because
`lanes`, `links`, and `dot` all edit the ledger router/styles and the shared walker test
file — the chain prevents merge contention, not data flow. Each phase is independently
shippable thanks to A11, and each must leave `npm run build:check`, `npm run validate`,
and the full `npm test` green before finishing, then deploy with `bob plugins sync`.

## Phase: navapi — nav api.walkMode v1 and the walk-mode doorbell

In bob-navigation-hotkeys:

- Add `walkMode` (version 1, `current()` + `select(type)`) to the api object built in
  `src/480-review-jump-and-nav-api.js`. That fragment is at 996/1000 lines, so put the
  factory (`createWalkModeApi(plugin)`) in a new fragment and register it in
  `src/fragments.json` in manifest order. `current()` delegates to
  `readBracketRepeatType()`; `select(type)` normalizes, refuses invalid types with
  `{ok: false, reason: "invalid-type"}`, and otherwise behaves exactly like a selector
  letter click — records via `rememberBracketRepeatType`, refreshes the indicator, never
  walks, never throws.
- Fire the doorbell `app.workspace.trigger("bob-navigation-hotkeys:walk-mode")` from
  `rememberBracketRepeatType` **only when the normalized value actually changed**
  (inside the existing change guard in `src/546-plugin-bracket-repeat.js`), and once at
  plugin setup after the stored value is first read, so a late-loading listener still
  gets one event. The event carries no authoritative payload.
- Document the namespace in the api comment block the way `quickRoll` et al. are
  documented.

Tests — extend `scripts/test-navigation-bracket-repeat.cjs`: the event fires exactly
once per actual change and once at setup; repeats (`]]`/`[[`) and re-selecting the same
letter fire nothing; `select` records without invoking any walk method; `current()`
normalizes invalid stored values to null. No visible behavior change in this phase.

## Phase: host — ledger walker-footer host and the gated s panel

In bob-ledger-tools:

- `setupWalkerFooter`: keep the **same** status-bar item and `.bob-freshness*` classes
  (CSS and existing tests keep applying) and add `.bob-walker-footer`. Resolve the mode
  on `onLayoutReady` (ledger loads before nav), subscribe to the doorbell, and on every
  event re-read `api.walkMode.current()`. A mode change paints synchronously (invariant
  3); the paint key must include the mode.
- Route one controller: when mode is `s`, build the existing `freshnessFooterView`
  (content unchanged — nine tiers, commitment/upkeep hairline, stage detail, tier
  colors, TICK answer chip, meter) with three deltas:
  - visible **only** while `s` is the family;
  - default hint `]] next` instead of `]s next` when nav is available (A12);
  - a clear state replacing hide-on-empty (A5): `⟳ Review clear · ✓ 29 today` — calm
    muted ink, green `⟳`, the Alt+P preview retained on a prioritized cursor row (the
    old hint-only view folds into this state), no celebration. "Clear" describes the
    current eligible queue, not the week's commitments.
- Modes `w` and `n` hide the item **in this phase only** (their panels land in `lanes`
  and `links`; note the transition in code comments is unnecessary — the router makes it
  obvious). Unset mode shows no panel (A6). Tasks indexing / Today loading hide the item
  (never a success-like zero); mobile stays hidden.
- Legacy fallback (A11): nav api absent, or `api.walkMode?.version < 1` → paint today's
  review footer exactly as now (ungated, `]s next`, hide-on-empty). Keep this path
  byte-compatible with current output.
- `src/120-freshness-footer.js` is at 925/1000 lines: the router and any new view-model
  code go in new fragments.

Tests — extend `scripts/test-ledger-tools-freshness-footer.cjs` (or add a sibling): mode
gating for all four mode values, the clear state, `]] next` hints, legacy fallback
output equals today's snapshot, both plugin load orders, unload/reload with no duplicate
items, thrown doorbell listener never breaks painting. Review answer/advance semantics,
freshness eligibility, and task writes are untouched — zero writer changes in this
phase.

## Phase: lanes — lane footer for the w walk

In bob-ledger-tools, a new fragment `laneFooterView` (plus painter glue), a **pure
view** over `api.laneWalk` v2's immutable `queue()` result. States (copy mirrors the
existing lane toast so the two surfaces never disagree):

| State                           | Reads                                                                                    |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| Off a walk row                  | `Lanes 20 · ]] next · WIP 10/8 · NEXT 7 · READY 3 · TODAY 12`                            |
| On a validated walk row         | `Lanes 13/20 · NEXT 3/7 · worked 2d ago · Alt+N release · WIP 10/8 · READY 3 · TODAY 12` |
| Every lane task on today's plan | `Lanes clear · TODAY 12`                                                                 |
| Ready waiting in review         | `Lanes clear · REVIEW 5 · ]s walks NEW and ROTTEN`                                       |
| No lane tasks at all            | `No Pending, Next, or Ready tasks`                                                       |
| Tasks indexing / Today loading  | hidden                                                                                   |

Rules:

- Counts are walk membership (A7): `queue.counts.pending/next/ready` plus `TODAY` from
  the same snapshot — exactly what `]w` can visit, matching the dash sections by
  construction. Never rescan checkboxes; one snapshot per paint (invariant 1).
- `n/cap` appears only when a lane is over its cap, in the existing over-cap tone; caps
  otherwise live in the tooltip. Example tooltip:
  `20 tasks off today's plan · WIP 10 of 8 (over) · NEXT 7 of 13 · READY 3 of 21 · TODAY 12 already planned · REVIEW 5 wait in ]s · ]] / [[ step · ]} / [{ jump lanes`.
- Cursor matching reuses the existing queue-cursor matcher (entries carry
  path/line/blockId). The position fraction appears **only** for a validated current
  member — no fabricated `0/24`, no stale fraction after the row leaves; a moved or
  released task never teleports the origin (the walk resumes from the old lane tuple per
  `resolveLaneWalkOrigin`).
- Hints (A8): a landed lane row's hint slot prefers the lane's own answer
  (`Alt+N release` on Next/WIP, `Alt+N next` on Ready; the Alt+P sentence moves to the
  tooltip); off-row keeps the prioritized-cursor Alt+P preview, else `]] next`.
- The work-log age detail chip reuses the lane toast's helper **copied into a shared
  fragment** — never a cross-plugin call per cursor move.
- Counts are informational in v1: clicking `READY 3` must not skip work or change
  filters (lane jumps already have `]}`/`[{`, taught in the tooltip). The leading
  `Lanes` button acts like `]]`.
- Shed order: remaining lane groups end-first (`…`), then detail, then hint, then meter;
  the whole-walk position and captions shed before any lane count; minimum state
  `Lanes 20 · WIP 10/8 N 7 R 3`.

Tests — new `scripts/test-ledger-tools-walker-footer.cjs` (register in `test:unit`):
every state above, over-cap rendering, no-fabricated-fraction cases
(release/move/delete/undo, midnight date change), Today/NEW/ROTTEN exclusions, plus a
**dash-parity fixture** pinning footer counts to the dash badge models
(`test-ledger-tools-dashboard-parity.cjs` shows the pattern).

## Phase: links — nav links() and the links footer for the n walk

In bob-navigation-hotkeys: add `walkMode.links({path, content})` and bump
`walkMode.version` to 2. It returns
`{ok, prev: {text, target, path, exists} | null, next: …}` using the **same**
first-match matcher behind `]n`/`[n` (`src/680-plugin-link-parse.js`: exact
`prev`/`next` labels or arrow labels `‹`/`<` prefix for prev, `›`/`>` prefix/suffix for
next, scanning live editor text, skipping frontmatter and fences). Read-only: never
calls `ensureNote`, never loads templates, never opens anything.

In bob-ledger-tools, a new fragment `linkFooterView` — a destination compass (A3):

| State                                | Reads                                                            | Activation                                  |
| ------------------------------------ | ---------------------------------------------------------------- | ------------------------------------------- |
| Both directions exist                | `⇄ Links · ‹ Wed Nov 11 · Fri Nov 13 ›`                          | left arm `[n`, right arm `]n`               |
| Arrow labels (period notes)          | `⇄ Links · ‹ W45 · W47 ›` (label minus arrow)                    | same                                        |
| Target missing but creation-eligible | `W47+ ›` faint/dashed; tooltip `No W47 note yet — ]] creates it` | existing creation route, only on activation |
| One direction only                   | `‹ W45 · no next`                                                | live arm only                               |
| No matching labels                   | `⇄ Links · no ‹ / › links in this note`                          | disabled                                    |
| Non-Markdown view                    | `⇄ Links · open a note` (faint)                                  | disabled                                    |

Rules: daily labels are literally `prev`/`next`, so the panel shows the resolved
target's readable stem/date, full path and subpath in the tooltip, path context for
duplicate basenames, names rendered as text never HTML. The preview reads the live
editor content (matching the key) and is generation-checked against family, source file,
and editor revision (invariant 2) — a stale response never paints; switching to `w`
invalidates an outstanding `n` read. Direction arms are real buttons routing through the
existing `[n`/`]n` commands with full revalidation before opening (invariant 5) — a
vanished preview never creates a note. No Tasks/Today dependency: the `n` panel works
when Tasks is unavailable. First-match rule preserved even when a later same-label link
would resolve. The period up-link (`↑ Q4`) is out of v1. No Alt+P hint in this panel
(A8).

Tests — a **parity test** asserting `links()` targets equal what `openLabeledBodyLink`
would open on daily, period, arrow-label, missing-target, fenced-code, and
unsaved-editor fixtures; ledger tests for every state row, stale response rejection,
disabled arms, and zero-write paint.

## Phase: dot — review attention dot on the selector's s

- Ledger computes "review commitments due" from the existing freshness memo (commitment
  tiers only — ROTTEN-only or POST-only queues never light it, A4) and pushes
  `api.walkMode.setAttention("s", null | {tier, label})`, display-only, nav
  `walkMode.version` 3. Re-push on every mode event and data refresh so load order can
  never lose it; push `null` when commitments clear or mode returns to `s`.
- Nav paints a ~5px dot on the selector's `s` letter only while mode ≠ `s` and attention
  is set: tier-tinted with a ring in the bar background, `Highlight` color under forced
  colors, no number, no motion. Tooltip and `aria-label` carry the counts, e.g.
  `Review: 9 due · 5 commitments — click to show`. Clicking the letter keeps its
  existing select-without-walking behavior (which clears the dot since mode becomes
  `s`).
- A missing or old nav api degrades silently (ledger skips the push); a thrown listener
  never breaks selection or painting.

Tests — extend `scripts/test-navigation-bracket-mode-indicator.cjs` (paint conditions,
clearing on mode `s`, forced-colors class, aria text, no motion class) and the walker
footer test (push conditions: commitments vs upkeep-only, mode transitions, null
pushes).

## Phase: visual — visual previews, e2e, and placement

- Generalize `scripts/preview-review-footer.cjs` into a walker-footer preview: every
  panel state from the three state tables, light and dark themes, three widths, plus an
  integrated-bar fixture that includes the selector and the Keep inbox item.
- Motion and accessibility polish across the three panels: ~120 ms content crossfade on
  mode switch while the existing selector thumb slides; count changes never animate;
  `prefers-reduced-motion` removes the fade; fixed item height; DOM stable (text updates
  only on change, no rebuild per cursor tick). Real buttons for movement/selection with
  visible focus and selected/disabled states (not every text chip); active mode and
  current lane carry text/shape cues as well as color; small text meets 4.5:1 in both
  themes; tabular numerals; test minimum widths with large counts and long Unicode
  names.
- e2e: extend the repo's Obsidian e2e harness (`scripts/obsidian-*-e2e*.cjs` pattern or
  the startup smoke) to assert the selector's bounding rect is unchanged across `s` →
  `w` → `n` (invariant 7) and that `]w` swaps the panel within a frame.
- **Placement (A10):** target the panel immediately left of the selector via CSS `order`
  so panel width changes never move the click target. The Keep item's CSS
  (`styles.css:7293-7295`) documents a dependency on the footer host's
  `margin-right: auto` (`styles.css:324-330`) — respect or deliberately replace that
  contract, and verify in real Obsidian in the live vault after `bob plugins sync` (read
  `sase/memory/tailnet.md` with `/sase_memory_read` first if this requires reaching the
  Mac). If the host layout fights, keep current positions and say so in the phase notes.

## Phase: ship — docs, memory, and deploy

Docs (read `sase/memory/docs.md` with `/sase_memory_read` before editing):

- bob-plugins `docs/bob-ledger-tools.md`: the footer section (around lines 190–199 and
  the long description row) gains the walker-footer framing, the three panels, clear
  states, `]] next` hints, and the dot; `docs/bob-navigation-hotkeys.md`: the bracket
  repeat and status-bar indicator passages (around line 499 and the description row)
  document `api.walkMode`, the doorbell, and the dot. Run `npm run docs:gen` if
  inventories change and `npm run docs:check`.
- bob-cli docs: `docs/freshness.md` — the desktop-footer paragraph at :595 (gating +
  clear state), the hint-only passage at :786-791, the footer notes at :937, and the
  ledger row at :990 that says "persistent review footer";
  `docs/guides/run-morning-review.md` §2 (line 68). Run `just docs-check`.

Memory (use `/sase_memory_write`; only the notes covered by accepted memory decisions;
regenerate with `sase memory init` after edits):

> [!decision] mem_walker_footer

- New strand `glossary:walker-footer` (aka walk footer, lane footer, links footer).
  Start from the research report's drafted definition (§"Glossary and memory") and keep
  its load-bearing claims: one contextual bob-ledger-tools status-bar panel explaining
  the active bracket walk; exactly one member at a time chosen by the persisted mode,
  never cursor/timer/remaining work; the three members (the
  [[morning-gtd-review-footer]] for `s`, the lane footer for `w`, the links footer for
  `n`); shared anatomy, hues, end-first shedding; clear states distinct from
  loading/unavailable; positions never progress; read-only projection; the `s` dot; no
  panel with no saved family; not the selector, a landing toast, the day strip, or a
  stats strip. Link `[[morning-gtd-review-footer]]` and other named memory.

> [!decision] mem_strand_amends

- Amend `glossary:morning-gtd-review-footer`: the first sentence names it the `s` member
  of the [[walker-footer]], shown while the review walk is the active bracket walk;
  `]s next` becomes `]] next`; replace the hide-when-nothing-due sentence with the
  `⟳ Review clear` state plus the retained Alt+P preview; a closing line notes the
  off-`s` dot.
- Amend `glossary:task-freshness` (same decision): point "Its status-bar view is the
  [[morning-gtd-review-footer]]" at the `s` walker footer (the review footer shown while
  `s` is the active bracket walk).

> [!decision] mem_decision_record

- Record `decisions:walker-footer-follows-bracket-walk`: claim (the status-bar footer
  follows the persisted bracket walk; one item, three panels), rejected alternatives
  (three items; session/cursor-inferred visibility; always-on review footer; morning
  auto-`s`; moving the review painter into nav), costs (no ambient review count off-`s`;
  a cross-plugin contract), reopen-if (missed morning reviews; dot too subtle), citing
  the research report and this plan. Skip entirely if the decision is off.

Finish with `bob plugins sync` so the vault runs the final build, and confirm the synced
version in the vault.

## Out of scope

No Rust/CLI change, no release-train or Mac Capture impact, no mobile surface, no change
to review answer/advance semantics, freshness eligibility, lane writers, or task writes
anywhere in this epic. The period up-link, clickable lane-count filters, and any `w`
attention pip are explicitly deferred.
