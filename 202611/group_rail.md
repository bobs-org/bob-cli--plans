---
tier: epic
title: The group row at the bottom of property cards
goal: Day, week, and quarter cards and grouped record walk rails show a quiet group
  row (`« Q3 · W27 … week 8 of 14 ▮▮■▯ … 2027-Q1 · W01 »`) from the same resolver
  behind `[N` / `]N`; legacy dailies get a derived step row; every missing stop reads
  dashed `+` (creatable) or dotted and faint (ghost) across rail items, chips, and
  arms; the album card says `· walks by Section`; and a preview matrix plus the existing
  e2e suites prove it on athena.
parent_bead: bob-cli-oj.5
phases:
- id: row-kit
  title: Kit group-row builder, rail missing-target states, and their CSS
  size: medium
  depends_on: []
  description: 'row-kit: add propertyCardBuildGroupRow (new 240l fragment, exported
    through a new 374d fragment), let propertyCardBuildRail take per-item `state`
    and an optional `group` row between the step row and the children strip, add the
    CSS for both rows (light, dark, narrow below 28rem, forced-colors, reduced motion),
    unit-test the builder, then build, test, bump, commit, and sync.'
- id: calendar-rail
  title: Day, week, and quarter cards adopt the group row and the derived step row
  size: medium
  depends_on:
  - row-kit
  description: 'calendar-rail: compute one nav answer per card from walkMode.links
    v7 (fallback api.navGroups), render the group row and the legacy derived step
    row, classify every rail item and chip as exists/creatable/ghost, repaint on vault
    create/delete/rename and midnight, run the group commands with re-validation,
    test it, update docs/intention-card.md, then build, test, bump, commit, and sync.'
- id: record-rail
  title: Grouped record walk rail and the album card's "walks by" pill
  size: medium
  depends_on:
  - row-kit
  description: 'record-rail: give grouped record walk rails the kit group row in place
    of the whole-walk thumb track, reword the count tooltip, activate arms through
    the group commands with re-validation, add the album card''s `· walks by <Field>`
    pill, test it, update docs/record-card.md and docs/albums.md, then build, test,
    bump, commit, and sync.'
- id: rail-proof
  title: Preview matrix, real-Obsidian e2e runs, and final deploy for the group rail
  size: medium
  depends_on:
  - calendar-rail
  - record-rail
  description: 'rail-proof: add scripts/preview-group-rail.cjs over the real card
    builders, render the contract''s fixtures in light and dark at 375px and wide
    plus grayscale and forced-colors, inspect every image and fix what looks wrong,
    run the intention-card, record-card, record-walk, and title-block e2e suites on
    athena, attach every report, pass the gates, and deploy the final tree.'
proposed_by: bbugyi200.athena.bob-cli-oj.5
proposed_with: claude/opus
create_time: 2026-11-21 12:21:08
status: done
bead_id: bob-cli-oj.5.1
---

- **PROMPT:** [prompts/202611/group_rail.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/group_rail.md)
- **PARENT:** [202611/bracket_n_group_jumps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bracket_n_group_jumps.md)
- **BEAD:** [bob-cli-oj.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oj/bob-cli-oj.5.1.md)

# The group row at the bottom of property cards

## Scope and ownership

This child epic delivers **bob-cli-oj.5** (`group-rail`), a phase of epic **bob-cli-oj**
(`[N` / `]N` group jumps). The phase had no prior notes or partial work. It is split
because it combines a shared kit builder, two independent adopters, and a visual and
real-Obsidian proof that each need a full agent. `calendar-rail` and `record-rail` may
run in parallel after `row-kit`.

**Workers close only their own phase bead.** This child epic's land agent closes
**bob-cli-oj.5** through the host's delegated closeout once all four phases are closed;
nobody closes bob-cli-oj, bob-cli-oj.6, bob-cli-oj.7, or any other bead. Workers never
create beads: record discoveries as `PROPOSED FOLLOW-UP:` notes on their own phase bead
(`sase bead note <own-bead> 'PROPOSED FOLLOW-UP: …'`), citing duplicate bead IDs when
known. The land agent forwards every follow-up to bob-cli-oj.5.

**Outer DECISIONS are final** (`sase bead read bob-cli-oj.5 -r "<why>"` shows them):
`daily_create = yes` (missing dailies in adopted years and the future are `creatable`),
`walk_more_albums = no`, `memory_glossary = yes` belongs only to the outer `glossary`
phase, and `memory_decision_group_stops = no`. **This child epic edits no memory file.**
The skipped decisions-web follow-up is already recorded on bob-cli-oj.5.

## Read first

- `sase artifact read plan:202611/bracket_n_group_jumps.md "<why>"`: the binding design
  contract. Read "The group row at the bottom of property cards", "Missing targets",
  "One resolver, four surfaces", "Records and `walk_by`", and the `group-rail` phase.
  This plan refines it; the contract wins where they differ.
- `sase artifact read plan:202611/property_card_title_block.md "<why>"`: the kit
  skeleton, rail, fail-open, and coordination rules this extends.
- Open bob-plugins only through `/sase_repo` (`sase repo open bob-plugins -r "<why>"`)
  and read its `AGENTS.md`, `docs/develop/architecture.md`, and
  `docs/develop/deployment.md`. Before docs edits, read `docs.md` with
  `/sase_memory_read`.

## Verified starting point (recheck; the base moves)

- bob-plugins `origin/master` was `bd8f2d4`; Bob Ledger Tools `1.149.2`, Bob Navigation
  Hotkeys `2.96.1`.
- Nav `api.walkMode` v7 is live: `links({ path, content })` returns
  `{ ok, prev, next, prevGroup, nextGroup, group, groupNone, groupConflict, groupConflicts }`
  from `resolveNavLinks` (`plugins/bob-navigation-hotkeys/src/483-nav-group-links.js`).
  Step stops carry `source` `authored` | `calendar`; group stops carry `source`
  `authored` | `calendar` | `record`, plus `state`, `ghostReason`, `groupLabel`,
  `landingLabel`, `shortLabel`, `wraps`, `target`, `path`, `text`. It is synchronous,
  never throws, and requires the source file to exist. Ledger code reaches it as
  `app.plugins.plugins["bob-navigation-hotkeys"].api.walkMode` and must feature-detect
  `Number(version) >= 7` (the links footer, `124-links-walker-footer.js`, shows the
  lookup).
- Commands `bob-navigation-hotkeys:open-prev-group-link` / `open-next-group-link` exist,
  refuse conflicts, open `exists`, create `creatable` on an explicit press, and show a
  notice for `ghost`. `open-prev-link` / `open-next-link` fall back to derived step
  stops.
- Ledger `api.navGroups` v1 (`296e`/`296f`/`296g`) resolves calendar and record routes.
  Inside Ledger Tools call `this.navGroupsResolve({ path, content })`,
  `this.navGroupsPathIsPreAdoption(path)`, and `this.navGroupsPreAdoptionMessage(path)`
  directly. Group `noun` is the member noun (`day`, `week`, `quarter`, or the record
  field key). Record `members` is not yet null above 21; the kit handles size > 21
  itself.
- Day and period cards: `296ab` `intentionCardRailModel` (rail only from a recognized
  nav line in the title range), `296b` `intentionCardDomRail` and
  `intentionCardBuildDom`, `296c` `intentionCardModelFor`,
  `intentionCardModelSignature`, `intentionCardOnSetup` (metadata listener and midnight
  timer only; no vault create/delete/rename listener), `296d`
  `intentionCardRailNavigate`, `intentionCardLeafForEvent`, `intentionCardOpenLink`.
- Record walk rail: `242t` `recordWalkModel` (has `walkBy`, `group`, `groupMembers`,
  `prevGroup`, `nextGroup`), `242u` `recordWalkRailBuildEl` (own DOM: arms, centre
  `album · 8 of 12` with tooltip `Alphabetical, wraps around`, and a
  `bob-record-walk-track` thumb), `242v` `recordWalkFor`, `recordWalkOptions`,
  `recordWalkOpen`, `buildRecordSheetDecorations`, `242ia` `recordSheetWalkSignature`.
- Album card: `241u` `albumHeaderModel` / `albumHeaderSignature`, `241w`
  `albumHeaderFoldText`, `241x` `albumSchemaPaintSummary`. bob's albums report carries
  top-level `walk_by` (the effective key, or null when unset or invalid) and the
  `record_walk_groups_v1` capability.
- Kit: `240g` `propertyCardBuildRail` (items take `unresolved`, which adds the
  Obsidian-style `is-unresolved` dotted underline). `styles.css` is one hand-edited
  file; the kit block sits between `Property-card identity kit … start` and `… end`
  markers. Tests pin `is-unresolved` on missing rail items
  (`test-ledger-tools-property-card- identity.cjs`,
  `test-ledger-tools-intention-card-title-block.cjs`) and
  `obsidian-intention-card-e2e.cjs` reads it.
- Known base failures (not this epic's): bob-cli `docs/reference/vault-format.md`
  6016/6000 docs lint (`bob-cli-oc`), `plugin-apis.md` develop-tier word warning
  (`bob-cli-ob`), and two load-sensitive Ledger timing assertions recorded on
  bob-cli-oj.1. Confirm the actual failure matches before citing one.

## Coordination rules (every phase)

- **bob-cli-o6 precondition.** Before editing, run
  `sase bead read bob-cli-o6.. -r "<why>"` and
  `git log --format='%h %(trailers:key=SASE_BEAD,valueonly)' origin/master -- <file>`
  for each fragment you will touch. While an o6 phase is open, do not edit a fragment it
  is changing. At planning time that meant `240e`, `240i`, `240k`, `241v`, `242kb`,
  `296d`, and the `obsidian-title-block-e2e*` scripts. This plan routes around them with
  new fragments. If your work truly cannot proceed without such an edit, add the bead
  note `BLOCKED: waiting for bob-cli-o6.<n>` and change nothing.
- Edit `src/` fragments only, run `npm run build`, never hand-edit `main.js`, and keep
  every hand-edited fragment at or below 1000 lines. New fragments go into
  `plugins/bob-ledger-tools/src/fragments.json` at the stated position; new mixins are
  registered in `310-install-methods.js` and exported for tests in a `37x` exports
  fragment.
- `calendar-rail`, `record-rail`, and the outer `links-footer` phase (bob-cli-oj.6) run
  concurrently. Before every sync: `git fetch`, merge `origin/master`, keep both sides'
  `fragments.json`, `310-install-methods.js`, `package.json` `test:unit` entries, and
  `styles.css` blocks, regenerate `main.js` with `npm run build` instead of merging it,
  and take the next free Ledger Tools manifest minor version.
- New `styles.css` rules go in their own delimited block
  (`/* Property-card group row (bob-cli-oj group-rail): start/end. */`) right after the
  kit block, so parallel phases never edit the same lines.
- Gates per phase: `npm run build:check`, the phase's focused tests, the full
  `npm test`, and `npm run docs:check` (with `BOB_CLI_DIR` pointing at an opened bob-cli
  checkout). Run long commands in the foreground with explicit large timeouts. Commit
  through the normal host finalizer flow, then deploy with `bob plugins sync`. If sync
  refuses another checkout's deploy, follow bob-plugins `AGENTS.md` exactly; never pass
  `-F` on your own.
- No changes to Navigation Hotkeys, daily templates, period scaffolds, walk-line bytes,
  the links footer (`121`/`124`, owned by bob-cli-oj.6), or the vault vimrc.

## Shared design: the group row and missing-target states

### Missing-target language

| State       | Item / chip look                                                    | Arm look                                                             | Activation                                                     |
| ----------- | ------------------------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------- |
| `exists`    | today's look                                                        | normal                                                               | opens (commands for prev/next/group, Markdown link otherwise)  |
| `creatable` | dashed 1px outline, muted ink, small trailing `+` (`aria-hidden`)   | same, `+` after the landing                                          | explicit click creates through the existing writer, then opens |
| `ghost`     | dotted 1px outline, faint ink; chips put the reason only in tooltip | same, plus visible `· not in vault`; `aria-disabled="true"`, no href | a Notice with `ghostReason`; nothing is written or opened      |

- Never error red and never Obsidian's unresolved-link styling. Keep the `is-unresolved`
  class on non-existing rail items and chips as a compatibility marker (tests and the
  intention e2e read it), but rail items and chips with `data-state` no longer get its
  dotted underline or opacity.
- Text contrast is at least 4.5:1 and borders at least 3:1 in light and dark. Start from
  `--text-muted` / `--text-faint` as the contract names them; where measurement falls
  short, mix toward `--text-normal` until it passes. Under `forced-colors: active`,
  dashed and dotted borders use `CanvasText`, ghost text uses `GrayText`, and the
  current tick uses `Highlight`. `prefers-reduced-motion` disables transitions.

### Kit API (`row-kit` owns it; adopters only consume it)

`propertyCardBuildGroupRow(doc, spec)` returns
`{ element, arms: { prev, next }, center, track }` or null when there is no arm and no
center. It never throws and writes text only through text nodes.

```text
spec = {
  label?: string,                      // aria-label of the row, e.g. "Week groups"
  prev?: Arm | null, next?: Arm | null,
  center?: {
    name?: string,                     // record: group choice label ("🔮 Future Work")
    text: string, shortText: string,   // "week 8 of 14" / "8/14"; record "2 of 3" / "2/3"
    ariaLabel?: string,
    position: number, size: number,
    members?: [{ label, state: "current" | "exists" | "missing" }] | null,
  } | null,
  onActivate?(direction: "prev" | "next", arm, event),
}
Arm = { groupLabel, landingLabel, shortLabel, target, state, ghostReason?, wraps?,
        tooltip, ariaLabel }
```

DOM (classes are the contract tests and CSS rely on):

- `div.bob-property-card-group-row[aria-label]` with three grid cells like the step row.
- Arm: `a.bob-property-card-group-arm[data-group="prev|next"][data-state]`, `title` =
  `tooltip`, `aria-label` = `ariaLabel`, `href`/`data-href` = target except on ghosts
  (ghosts get `role="link"`, `tabindex="0"`, `aria-disabled="true"`, and Enter/Space
  activation). Children in reading order: `span.bob-property-card-group-glyph` (`«` /
  `»`, `aria-hidden`, outside the labels), `span.bob-property-card-group-label` (group
  label, medium weight), `span.bob-property-card-group-sep` (`·`, aria-hidden),
  `span.bob-property-card-group-landing`, `span.bob-property-card-group-plus` (`+`,
  creatable), `span.bob-property-card-group-note` (`· not in vault`, ghost), and
  `span.bob-property-card-group-wrap` (`↺`, aria-hidden, when `wraps`). The prev arm
  reads `« Q3 · W27`; the next arm reads `2027-Q1 · W01 »`. Labels truncate with an
  ellipsis and never wrap.
- Center: `div.bob-property-card-group-center` with optional
  `span.bob-property-card-group-name`, `span.bob-property-card-group-position` (wide and
  narrow spans, the pattern of `bob-record-walk-count`), and
  `span.bob-property-card-group-track[aria-hidden]`: one
  `span.bob-property-card-group-tick[data-state]` per member (3px rounded bar; accent
  fill for current, muted fill for exists, hollow outline for missing). When `size > 21`
  or members are absent, the track is a continuous bar with one
  `span.bob-property-card-group-thumb` placed at the position.
- Narrow (the enclosing `bob-property-card` or `record-walk` container below 28rem): the
  row collapses to `« Q3` · `8/14` · `2027-Q1 »`, hides landings, `+`/note text, and the
  track, and keeps every arm at least 32px wide.
- Group row type is UI-small, one step below the step row; arms use `--text-muted` for
  the group label and the faint token for the landing (subject to the contrast rule).

`propertyCardBuildRail(doc, spec)` changes additively:

- `prev`, `parent`, `next`, and each child accept `state` (`exists` | `creatable` |
  `ghost`) and `ghostReason`. A missing `state` with `unresolved: true` means
  `creatable`; neither means `exists`. Non-existing items get `data-state`,
  `is-<state>`, the compatibility `is-unresolved`, the `+` or `· not in vault` span
  (chips: `+` only), and ghosts drop `href` and become `aria-disabled`. The rail never
  decides activation; it passes the item (with its state) to `onNavigate` as today.
- New `spec.group` (a group-row spec): the rail builds it with
  `propertyCardBuildGroupRow` and inserts it after the step row and before the children
  strip. A rail with only a group row renders without an empty step row. The return
  gains `group` (the built row or null). Existing callers that pass no `state` and no
  `group` render byte-identical DOM, except that `is-unresolved` items now also carry
  `data-state="creatable"`.

### Labels, tooltips, and accessible names (adopters build these)

- Group unit: day card → `week`, week card → `quarter`, quarter card → `year`; record →
  the `walk_by` field label, lowercased (`section`, `status`).
- Arm tooltip: `]N · next quarter: 2027-Q1 → 2027-W01` (key, direction, unit, group
  label, landing note id). Creatable:
  `2027-W01 isn't created yet — ]N creates it with bob periods ensure`, or for a daily
  `Mon Nov 23 isn't created yet — ]N creates it from the daily template`. Ghost: the
  stop's `ghostReason`. Wrapping record arm: append `· wraps around`.
- Accessible names: `Previous quarter group: Q3, opens 2026-W27`,
  `Next week group: W48, not created yet — activating creates it`,
  `Previous year group: 2025, not in vault`, `Next section group: 🏁 Conclusion, opens
  <title>, wraps around`.
- Center text: `day 6 of 7`, `week 8 of 14`, `quarter 4 of 4` (calendar, from group
  noun, position, size); record `2 of 3` beside the choice label.

### Activation (both adopters)

- A plain click on a group arm re-validates first: read the source pane's current text
  (the leaf's editor or view data, else `vault.cachedRead`), ask `walkMode.links` (or
  `navGroupsResolve` without Navigation Hotkeys) again, and compare `target`, `path`,
  and `state` of that role's stop with the rendered arm. On a mismatch, schedule a
  repaint, show `The group stop changed — try again`, and do nothing else. On a match,
  make the card's own leaf active (focus) and run `open-prev-group-link` /
  `open-next-group-link`, exactly as rail prev/next run `open-prev-link` /
  `open-next-link` today.
- A modified click (Mod, Alt, Shift, middle) on an `exists` arm opens it like a Markdown
  link (tab or split); on a `creatable` arm it behaves like a plain click.
- A ghost arm only shows its `ghostReason` Notice.
- Without Navigation Hotkeys (or walkMode < 7), arms come from `navGroupsResolve`;
  `exists` arms open through the card's link opener, and `creatable` arms show
  `<landing> isn't created yet — Bob Navigation Hotkeys creates it with ]N`.

## Phase row-kit

Files: new `src/240l-plugin-property-card-group-row.js` (after `240k` in
`fragments.json`), `src/240g-plugin-property-card-title-block-dom.js`, new
`src/374d-exports-property-card-group-row.js` (after `374c`), `styles.css`, new
`scripts/test-ledger-tools-property-card-group-row.cjs` (register in `test:unit`), and
updates to `scripts/test-ledger-tools-property-card-identity.cjs` only where the
additive `data-state` changes an assertion.

1. Implement `propertyCardBuildGroupRow` and the `propertyCardBuildRail` additions
   above. Keep `240g` within 1000 lines; put helpers in `240l`.
2. CSS in its own delimited block: both rows' state looks, the group row grid and
   typography, ticks and the continuous bar, the narrow container rule for both the
   `bob-property-card` and `record-walk` containers, dark-theme adjustments where the
   tokens need them, `forced-colors`, and reduced motion. Existing card pixels change
   only for missing items and the new row.
3. Tests: builder null cases; arm order, glyphs `aria-hidden`, tooltips, accessible
   names; each state (creatable `+`, ghost `· not in vault`, `aria-disabled`, no href,
   Enter activation); wrap glyph; ticks per member state; `size > 21` continuous bar;
   narrow spans present; rail `state` mapping including `unresolved` back-compat; group
   row placement between the step row and children; group-only rail; unchanged DOM for
   legacy specs.
4. Gates, bump to the next free minor, commit, and `bob plugins sync`. Note the version
   and commit on the phase bead.

## Phase calendar-rail

Files: new `src/296ca-plugin-intention-card-group-rail.js` (after `296c`; mixin
`BobLedgerToolsIntentionCardGroupRailMixin` registered right after
`BobLedgerToolsIntentionCardMixin`, exported through `375-exports-intention-card.js` or
a new `375a` fragment), small edits to `296b` and `296c`, new
`scripts/test-ledger-tools-intention-card-group-rail.cjs`, `docs/intention-card.md`. Do
not edit `296d`.

1. **One nav answer per card.** In `intentionCardModelFor`, after `model.rail`, set
   `model.navRail = this.intentionCardNavRail(path, current.source, model)`. It asks
   `walkMode.links` v7 when available (else `navGroupsResolve`) and returns plain data:
   the group-row spec inputs (`group`, `prevGroup`, `nextGroup`, unit, kind), the
   derived step stops, and the derived parent (`navGroupsResolve(...).step.parent`). It
   never throws; on any failure it returns null and the card paints exactly today's
   rail.
2. **Group row.** Shown only when the answer has a group or a group stop (so years,
   `all_time`, and conflict-only answers keep today's rail). A conflicted role simply
   has no arm. Build the kit spec with the labels, tooltips, accessible names, and
   center text from the shared design; members map to ticks.
3. **Derived step row.** When `model.rail` is null (no recognized nav line in the title
   range) and every present step stop has `source: "calendar"` (or the answer came from
   `navGroupsResolve`), render prev / parent / next from the derived stops with their
   states. Labels follow today's rail (`Fri 20`, `W47`, `Sun 22`; `W46`, `2026-Q4`,
   `W48`). Never render a derived row when any step stop is authored, so nothing
   duplicates a visible nav line. No derived children.
4. **States on the written rail.** Classify each written rail item and child chip with
   one helper: resolve the target to a path (calendar subject path, else
   `metadataCache.getFirstLinkpathDest`); existing → `exists`; missing calendar path
   that `navGroupsPathIsPreAdoption` flags → `ghost` with `navGroupsPreAdoptionMessage`;
   other missing → `creatable` (`daily_create = yes`).
5. **Rendering.** `intentionCardDomRail` passes `state` on items, passes `group`, and
   renders when either the written rail, the derived step row, or a group row exists.
6. **Item activation.** In `296b`'s `onNavigate`, before delegating: a ghost item or
   chip shows its reason and stops. Unmodified prev/next keep running the step commands
   (they handle creatable and ghost). A creatable parent or chip whose path is a period
   note creates through `this.periodNotesEnsureForPath(path)` (surface its refusal
   message, including `pre-adoption`) and then opens it; a creatable daily keeps today's
   open path. Group arms follow the shared activation rule (put the logic in `296ca`;
   reuse `intentionCardLeafForEvent` and `intentionCardOpenLink`).
7. **Repaint.** In `intentionCardOnSetup`, register vault `create`, `delete`, and
   `rename` listeners that schedule a refresh when the old or new path is a calendar
   path (`intentionCardSubjectForPath`). Add `navRail` (stop targets and states, group
   position and member states, derived-row presence) to `intentionCardModelSignature` so
   existence changes and the midnight rollover repaint.
8. **Tests** (fake vault, fake walkMode, and the no-Navigation-Hotkeys fallback): the
   contract's worked examples (`2026-W30`, `2026-W47` into 2027, `2026-W01` with a ghost
   `2025-W40`, `2026-Q4` with a ghost `2025-Q1`, Saturday `20261121` with a creatable
   `20261123`, a Thursday in 2026-W53, `20251229` in 2026-W01); a year and `all_time`
   (no group row); a legacy daily with no nav line (derived row, parent, group row); an
   authored `prev`/`next` elsewhere in the body (no derived row); a conflicted role;
   ghost items never open or write; creatable period chip goes through
   `periodNotesEnsureForPath`; re-validation mismatch refuses; the plain-click command
   path targets the card's leaf; repaint on vault create and on the midnight rollover.
   Update existing intention tests only where the additive DOM changes them.
9. **Docs.** `docs/intention-card.md` "Docked rail": the group row, the derived step row
   for legacy notes, the three states, and click behavior, within the page's word
   budget.
10. Gates, merge, next free minor, commit, and `bob plugins sync`.

## Phase record-rail

Files: new `src/242va-plugin-record-walk-group-rail.js` (after `242v`; mixin
`BobLedgerToolsRecordWalkGroupRailMixin` registered next to
`BobLedgerToolsRecordWalkRailMixin`, with test exports), small edits to `242u`, `242v`,
`242ia`, `241u`, `241w` or `241x`, new
`scripts/test-ledger-tools-record-walk-group-rail.cjs`, additions to
`scripts/test-ledger-tools-album-header.cjs`, `docs/record-card.md`, `docs/albums.md`.
Do not edit `241v` or `242kb`.

1. **Answer.** In `recordWalkFor`, after the model is built, attach
   `walk.groupRail = this.recordWalkGroupRail(path, sourceText, walk)`: the
   `walkMode.links` v7 answer (else `navGroupsResolve`) reduced to the group-row inputs
   plus the field label from the album report (`fields[].label` for `walk.group.field`).
   It is null unless `walk.state === "ok"` and the answer has a group with at least two
   groups or a group stop. Ungrouped, single-group, syncing, and `grouping_unavailable`
   albums keep exactly today's rail.
2. **Rail.** In `recordWalkRailBuildEl`, when `walk.groupRail` exists, append the kit
   group row after the grid in place of `bob-record-walk-track`; keep the overall
   `8 of 12` count in the centre. Center: the current group's choice label, `2 of 3`
   (`2/3` narrow), and the member track. Arms: `« <group> · <head title>` and
   `<group> · <head title> »`, `↺` on wrapping arms, ghost for a missing head (record
   targets are never created; keep `⟳ syncing` and the album-folder guard). The count
   tooltip becomes `By <Field label>, then title · wraps around` for grouped albums and
   stays `Alphabetical, wraps around` otherwise. Hue tokens follow the rail's
   `--bob-walk-ink`.
3. **Activation.** `recordWalkOptions` gains `handlers.groupOpen`, implemented in
   `242va` with the shared rule and `recordSheetMarkdownLeaf(sourcePath)` for the pane.
   A ghost arm shows its reason and, like `recordWalkOpen`, may request an album sync
   when bob reports the record-walk capability.
4. **Repaint.** Add the group rail (stop targets, states, wraps, group index, position,
   member states) to `recordSheetWalkSignature` so Live Preview and Reading rebuild when
   it changes.
5. **Album pill.** `albumHeaderModel` gains `walkBy` (the report's effective `walk_by`
   or null) and `walkByLabel` (`albumHeaderFieldLabel`), both in `albumHeaderSignature`.
   On a healthy schema summary line, append `·` and `span.bob-album-schema-walks-by`
   reading `walks by Section` with the tooltip
   `Record walk: grouped by Section, then title`. Display-only: no click handler beyond
   the summary's own. Absent when `walk_by` is null.
6. **Tests:** grouped zettel-like fixture (first member of the first group with a
   wrapping prev arm, a middle member, the last member of the last group wrapping next);
   needs group; ghost head; single-group and ungrouped albums keep the thumb track and
   no group row; `grouping_unavailable` keeps today's rail; tooltip wording; signature
   change on a group edit; activation re-validation and the leaf-targeted command;
   modified click on an existing head; the album pill present, absent, and in the
   signature.
7. **Docs.** `docs/record-card.md` (the group row on grouped walks and its keys) and
   `docs/albums.md` (`walk_by` shows as `walks by` on the card; behavior only), within
   word budgets.
8. Gates, merge, next free minor, commit, and `bob plugins sync`.

## Phase rail-proof

Files: new `scripts/preview-group-rail.cjs` (do not edit `preview-property-cards.cjs`,
`preview-intention-card.cjs`, or `preview-title-block.cjs`, which the open
bob-cli-o6.6.1 visual-matrix phase owns; importing their exports is fine), plus fixes in
whichever fragment or CSS block owns a defect.

1. **Preview.** Build cards with the real builders from the generated bundle
   (`intentionCardBuildDom` with a `navRail`, `recordWalkRailBuildEl` with a
   `groupRail`, `albumHeaderBuildEl`) and the committed `styles.css`, using the existing
   preview page and Obsidian-executable screenshot pattern. Fixtures: a day mid-week,
   today with a creatable next Monday, a week mid-quarter, `2026-W47` crossing into
   2027, `2026-W01` with a ghost, `2026-Q4` with a ghost, a year (no row), a legacy
   no-nav-line daily (derived step row), grouped zettel records (first, middle,
   wrapping), a single-group album (no row), and the album card with the `walks by`
   pill. Render light and dark × 375px content width and wide (record actual widths),
   plus grayscale and real `forced-colors: active` emulation for every fixture. Write an
   index page and a machine-readable manifest.
2. **Inspect every image** at full size, record filename, verdict, and any repair in an
   inspection note, and measure text and border contrast for each state in light and
   dark (computed styles). Fix real failures in the owning fragment or CSS block,
   rebuild, rerun affected tests, and re-render.
3. **Real Obsidian on athena** (`/snap/bin/obsidian`, independent `xvfb-run -a`,
   isolated profiles, never the live vault): run the intention-card, record-card,
   record-walk, and title-block e2e scripts. A failure caused by this epic is fixed; a
   failure that reproduces on the pre-epic base is cited with the base revision and any
   tracking bead, never relabeled green.
4. **Gates and deploy:** `npm run build:check`, `npm test`, `npm run docs:check`;
   `bob plugins sync` of the final merged tree; confirm `bob plugins list --no-pull`
   shows Ledger Tools synced at the final version with matching managed-file hashes.
5. **Evidence:** attach the preview archive (index, manifest, images, inspection note,
   contrast table) and each e2e report with `sase artifact create` on this phase bead
   (read the `sase_artifacts.md` reference memory first), and list the artifact refs in
   the closing note so the land agent can link them on bob-cli-oj.5.

## Land-agent acceptance

- All four phases are closed. Ledger Tools on `origin/master` renders the group row on
  day, week, and quarter cards and on grouped record walk rails, the derived step row on
  legacy dailies, the three missing-target looks on items, chips, and arms, and the
  album `walks by` pill; year notes, `all_time`, ungrouped, and single-group albums keep
  today's rail.
- Unit tests cover the builder, states, narrow form, activation re-validation, and both
  adopters; the preview archive and inspection note cover every fixture in light, dark,
  375px, wide, grayscale, and forced-colors; the four e2e reports are attached.
- Gates are green or red only on cited, already-tracked base failures. The vault deploy
  matches the final `origin/master` Ledger Tools build.
- Run `sase bead epic-symbols` for this child epic and for bob-cli-oj.5 and resolve or
  re-key every entry. Then complete bob-cli-oj.5 through the delegated closeout with a
  note listing the evidence artifacts and every `PROPOSED FOLLOW-UP:`. Never close
  bob-cli-oj or any sibling phase.

## Non-goals

- No Navigation Hotkeys, links-footer, vimrc, template, scaffold, or walk-line changes.
- No `walk_by` adoption on live albums (bob-cli-oj.7), no new group-jump e2e
  (bob-cli-oj.7), and no memory edits (bob-cli-oj.8 glossary).
- No Schema Card editing of `walk_by`, no mobile surfaces, no `{` / `}` keys in card
  editors.
