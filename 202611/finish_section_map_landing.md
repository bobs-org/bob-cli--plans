---
tier: tale
title: Finish the Section Map and land bob-cli-rc
goal:
  Repair Section Map layout, motion, failure isolation, and pop-out dismissal, prove the
  missing acceptance cases, and close bob-cli-rc.
size: medium
proposed_by: bbugyi200.athena.bob-cli-rc.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-rc
create_time: 2026-11-27 17:45:34
status: wip
---

- **PARENT:**
  [202611/ctrl_jk_section_map.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/ctrl_jk_section_map.md)
- **BEAD:**
  [bob-cli-rc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-rc/README.md)

# Finish the Section Map and land bob-cli-rc

## Scope and ownership

Repair the bounded defects found by the bob-cli-rc land audit, verify the finished
feature, and close **bob-cli-rc in this tale's implementation turn**. This tale has no
separate land agent. Do not wait for this work's commit SHA, push, or CI before the
closeout: the code and epic close belong to the same turn.

All implementation lives in the `bob-plugins` linked repository. Open it through
`/sase_repo`, read its `AGENTS.md`, edit source fragments rather than generated
`main.js`, keep each fragment at most 1000 lines, and regenerate the bundle. The epic
plan is `plan:202611/ctrl_jk_section_map.md`; read it through `sase artifact read`, and
read bob-cli-rc's latest landing note before starting. Keep the existing internal model
API where it works; renaming its fields is not part of these repairs.

The approved decisions remain **peek_lifetime = pinned** and **section_map_record =
no**. No memory edits are authorized. Governing records:

- `decisions:q-and-ctrl-bracket-mirror-esc`: this map remains passive, with no focus,
  consumed keys, or pointer handling. Activity dismissal must let Vim's Esc, q, and
  Ctrl+[ do their normal work.
- `decisions:successor-jump-parks-the-walk`: attach the map to section-jump commands,
  not literal Ctrl+J/Ctrl+K bindings; other owners retain their behavior.

The original deferred features remain deferred: hidden/folded-heading filtering, counted
jumps, jump history, clickable maps, reading-view parity, and mobile.

## Audit evidence and integration baseline

All four phases and all their notes were reviewed. The implementation is in bob-plugins
commits `44425a49`, `d4edb022`, `c3729cf1`, `592c3c9a`, and `dca84529`. At `dca84529`,
the focused model, painter, lifecycle, jump, section-nav, and e2e self-test suites pass
**46/46**. `npm run build:check` and `npm run docs:inventory:check` pass. These passing
checks miss the defects below. The ship phase reports a real-Obsidian 23-case pass; this
tale must rerun acceptance with stronger checks rather than treating that report as
proof of the new cases.

Post-start plugin commits reviewed: `3c3333b0` and `64268af9` (cabinet acceptance and
go-live), `9317ce32` (tree route), `061c41b8` (tree rail), and `5b34a69c` (navigation
matcher/footer). The last changes navigation fragments 483/670/680, not the section
picker. Preserve its link parsing and the shared build/docs/test registrations. Cabinets
intentionally hide some stored headings; do not change the Section Map scanner to filter
them. The original epic explicitly defers that. The bob-cli capture leads, prose
relocation, executable-test-stub, and successor block-identity changes through origin
`d117d288` have no Section Map consumer to adapt. Fetch and review any newer drift
before completing the work.

## 1. Keep the current section visible and the card inside its pane

Repair `131-section-map-model.js`, `474-section-map-view.js`, and the placement handoff
in `612-section-map-controller.js` as needed.

Reproduce with the production helpers from `scripts/navigation-hotkeys-harness.cjs`:

```js
const lines = Array.from({ length: 42 }, (_, i) => `# Heading ${i}`);
const rect = { left: 0, top: 0, width: 800, height: 220 };
const placement = helpers.computeSectionMapPlacement({ host: rect, scroller: rect });
const model = helpers.buildSectionMapModel(lines, {
  cursorLine: 20,
  direction: -1,
  maxRows: placement.maxRows,
});
// Currently: placement.maxRows === 5; no model.rows entry has current === true.
```

`sectionMapWindowBounds` uses five rows before an upward landing even when the entire
window has only five rows. A production-painter Chromium probe also measured a **255 px
card in a 220 px pane**, with nine rendered heading/gap rows and no current row.
Evidence: `file:explicit:be63725046c23ea0929e0b67`.

- Clamp directional context to the actual window capacity so every model with a current
  section paints that section. Preserve deterministic document order, real-picker chip
  targets, exact gap counts, and ancestor/wrap pins.
- Make placement account for the complete rendered height: heading rows, gaps, pins,
  chrome, and a two-line current label. Reduce optional context or use the existing
  compact mode when the mandatory content cannot fit. Recompute the budget after a
  Notice nudge. Keep placement stable between ordinary presses; resize/pane moves may
  recompute it.
- Let a two-line current label enlarge its row and pill. Chromium currently measures a
  30 px label inside a fixed 24 px current row. Prevent overlap with neighboring labels
  and account for that height in fitting.
- Add model checks over all directions, row budgets 1–12, note boundaries, and
  nested/wrapped lists. Add browser geometry checks at heights 168/180/200/220/280,
  narrow widths, a Notice overlap, and long current labels. Assert the current landmark
  and chip targets are actually visible and the card fits its host.

## 2. Make the existing motion rules true

The painter currently chooses `slide` after moving from section index 0 to 20 in a
42-heading note, although the old row is absent from the new painted window. It tests
the old key only against the old painted list. Choose crossfade when either endpoint is
absent from either relevant painted window; retain ordinary slides when both endpoints
are visible, and never replay entrance per press.

The reduced-motion host disables animation on the card and pill but omits the wrap
badge. A real Chromium render with `{ reducedMotion: true }` still reports
`bob-section-map-pop` on the badge (a scale animation). Cover the badge under both
host-window reduced-motion preference and explicit test options. Keep static pill
positioning correct and use no animated transforms in reduced-motion mode. Verify
computed styles and real node geometry, not just presence of CSS strings.

## 3. Preserve navigation when map preparation fails

In `613-plugin-section-jump.js`, `armSectionMapSelection()` runs before the cursor move
and outside the map's try/catch. Replacing it with a throwing stub makes
`jumpToSectionHeader()` throw and leaves the cursor at its starting line. The
map-specific heading-cache hook also runs before the guarded region.

Keep the anchor armed before the jump's own CM selection dispatch on the healthy path,
but guard optional map setup/cache access and continue navigation if it fails. A cache
failure should fall back to the same canonical scanner/picker; do not introduce a
competing target parser. Test failures in arming/setup, cache access, flash, and painter
separately: the cursor, return value, Notices, and one-scroll behavior must match a
normal section jump. Peek must remain non-moving and best effort. Do not swallow genuine
inability to navigate.

## 4. Make activity dismissal work across window clocks

`sectionMapNow()` uses the plugin realm's `performance.now()`, while the capture
listener compares it directly to each window's `KeyboardEvent.timeStamp`. Different time
origins can make unrelated keys look like the jump dispatch. A deterministic controller
probe with a pinned map, command stamp 100000, and a later Escape from a younger window
with stamp 100 leaves the map pinned after microtasks settle.

Use dispatch identity or timestamps normalized to a common origin, with explicit window
provenance. Preserve both listener orders: Obsidian may execute the command before or
after the map's capture listener. A genuine same-dispatch jump must survive, including
held repeats; a subsequent unrelated key must dismiss. Do not consume keys or bind
literal jump chords. Add tests with distinct window time origins and prove ordinary key
dismissal in a real Obsidian pop-out. Use the host window for any scheduling needed to
keep that pop-out responsive. Verify teardown still releases every listener, frame,
timer, and observer.

## 5. Complete acceptance, deploy, and review fresh drift

- Finish the promised independent scanner oracle: preserve the pre-epic scanner from
  `44425a49^` in tests, then generate frontmatter, backtick/tilde fences with unequal
  delimiter lengths, bare and indented headings, duplicate labels, and heading-shaped
  fenced text. The existing randomized test only compares consumers of the same scanner
  and generates no fences/frontmatter. Keep chip parity against the actual picker and
  the intentional single-heading no-chip case. Register any new test file in the
  explicit package test list.
- Strengthen `obsidian-section-map-e2e.cjs`: prove the Vim emulation route and ordinary
  hotkey route independently, so one cannot mask the other. For `j`/`k` dismissal, enter
  normal mode before showing the map; currently the test sends Escape via `normalMode()`
  after showing it, so Escape can satisfy the assertion. Add a passivity check for q's
  Vim macro behavior. Add short horizontal splits, long labels, reduced-motion wrap, and
  pop-out dismissal to acceptance.
- Run focused tests including the open-notes landing-flash coverage, the browser
  preview, and full real-Obsidian acceptance in an isolated scratch vault. Review every
  screenshot. Retain the existing pinned peek, both jump directions, linger,
  hold/repeat, wrap, wheel, pane-switch, unchanged-note-hash, and console checks. A
  skipped required route or an offline self-test is not an acceptance pass. Record the
  fresh report/screenshots as audited SASE artifacts on bob-cli-rc.
- Follow repository version conventions for the navigation plugin, update any affected
  validation docs after reading `docs.md` memory, and run the plugin gate:
  `npm run build`, `npm run build:check`, `npm run docs:gen`,
  `npm run docs:inventory:check`, and `npm test`. Use `/sase_monitor` before long
  commands. Run `just check` for changes in the main bob-cli repo; never run
  `just check-full`. Attribute unrelated gate failures to their existing owners.
- Deploy via `bob plugins sync` and verify the built version/bundle matches the
  deployment. Follow the repository's protected-deploy procedure without forcing.
  Recheck post-audit commits on both main branches and integrate real conflicts.

The lander has already triaged **every** proposed follow-up:

- rc.1 note #1: new large flake task **bob-cli-rh**, for the viewport CPU guard's
  post-mitigation recurrence. Its cause still needs investigation; do not blindly raise
  its budgets. pg and f8 are distinct timing nodes, not duplicates.
- rc.1 note #2 and rc.2/rc.3/rc.4 note #1: one existing snooze vector mismatch,
  corroborated on active **bob-cli-q5**. No duplicate task. On clean `dca84529`, the
  focused combined run is 57/58: all 37 task-link viewport tests pass and the sole
  failure is `snooze-add-resnooze-replaces-history`. Evidence:
  `file:explicit:98f2378ffdb7a714d3749e61`.

These failures predate Section Map and are not this tale's implementation work. Carry
all outcomes into the epic close note; new issues caused by this epic must be finished
before close.

## 6. Finish bob-cli-rc's landing in this same turn

This is the tale's final step, after local verification and deployment, with no
dependency on this turn's future commit/push/CI result.

1. Re-read
   `sase bead read bob-cli-rc -r "Need final descendant readiness and parent link"`, its
   children and notes, and the linked plan. All four original phases were closed at
   audit. Resolve any new outstanding descendant; do not leave another lander waiting
   for this tale.
2. Run `sase bead epic-symbols bob-cli-rc`. None existed at audit. For every entry now
   present, wire it up, privatize it, add an appropriate non-test pragma, or delete it
   under the Symvision policy. Re-key only when a specific still-open later bead truly
   needs the exemption.
3. Close normally with
   `sase bead close bob-cli-rc --note "<verification of phases and decisions, repairs, fresh drift review, gates/deploy/evidence, and all follow-up outcomes above>"`.
   Never force a successful landing. If symbols reject the close, clean them and retry.
   If a named phase is incomplete, finish or deliberately reopen it; use force only for
   a genuinely canceled/superseded outcome with a reason and the corresponding non-done
   resolution.
4. Run `just symvision` wherever the recipe is available and confirm a clean whitelist;
   if absent, record that fact. Open the plans repo through `/sase_repo` and set
   **status: done** in the frontmatter of **202611/ctrl_jk_section_map.md**, the plan
   shown by the epic read. Read that artifact through the audited command before editing
   it.
5. The audit found **no parent_bead** on bob-cli-rc. Confirm with
   `sase bead read bob-cli-rc -r "Need the parent link"` and finish normally when still
   absent. If a parent has since been added, follow the landing protocol: verify and
   normally close only a parent phase, leaving its epic to its lander; for a parent
   plan, recheck its previous landing note, all descendants/notes, linked plan, and
   post-child drift before normal closure, symbol cleanup, symvision, and marking its
   plan done. Repeat through complete direct plan ancestors only. Record and report the
   first incomplete or ambiguous parent.
6. Finish through `/sase_final`, declaring all changed repositories. The host commits
   this tale's code and closeout after the turn ends.
