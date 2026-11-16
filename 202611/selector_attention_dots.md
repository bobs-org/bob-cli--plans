---
tier: tale
title: Selector attention dots for w (lanes over limit) and b (bank rows waiting)
goal:
  The [[ s w n b ]] selector dots w red while any PENDING/NEXT/READY badge is red and
  dots b reliably while bank rows wait.
size: medium
decisions:
  b_dot_hue:
    ask:
      Should the b dot stay amber, the bank badge's waiting ink, instead of turning red
      like w?
    choices:
      amber:
        Keep amber; waiting rows read as an inbox, and red stays reserved for over-limit
        trouble
      red:
        Make it red like w; one alarm color, but it stops matching the bank badge's
        waiting ink
    default: amber
    why: Amber already means "waiting" on the bank badge and the b panel's WAITING word
    answer: amber
  s_dot_hues:
    ask:
      Should the s dot switch to the review footer's tier hues so it always matches the
      group it summons?
    choices:
      keep:
        Keep today's palette; the morning PRE dot stays red but disagrees with the
        footer's cyan PRE group
      match:
        Use the footer's hues; the dot always matches its first group, but the morning
        PRE dot turns cyan
    default: keep
    why: You know the s dot as red, and recoloring it was not requested
    answer: match
  mem_walker_footer:
    ask: Amend glossary:walker-footer so its dot sentence names the new w dot?
    memory:
      - glossary:walker-footer
    default: true
    corrects:
      A quiet dot marks the selector's `s` while review commitments are due off-`s`, and
      an amber dot marks `b` while bank rows wait off-`b`.
    answer: true
proposed_by: bbugyi200.apollo.eh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 12:02:02
status: wip
---

# Attention dots for `w` (lanes over limit) and `b` (bank rows waiting)

## Goal

The status bar's `[[ s w n b ]]` bracket selector (bob-navigation-hotkeys) already puts
a small dot on the `s` letter while review commitments are due and another walk is
selected. Extend the same language to two more letters:

- **`w`**: a red dot whenever any PENDING, NEXT, or READY badge would be red. That is, a
  lane's count is over its limit, exactly as the dash and daily `bob-plan` badges judge
  it.
- **`b`**: a dot whenever any bank row is unticked (waiting). This dot already exists
  (amber, nav `api.walkMode` v4, ledger `walkerFooterPushBankAttention`), but a
  load-order bug makes it unreliable (see Context). Fix it so it lights and clears
  promptly.

The result must be intuitive (each dot means one plain thing and its tooltip says what),
reliable (the dot and its source surface can never disagree), and beautiful (same
geometry and voice as the existing `s` dot, and the active `w` panel keeps the signal
visible).

## Context (verified while planning)

All code lives in the linked repo **bob-plugins**. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first. Its rules
apply: edit `src/` fragments, run `npm run build`, never hand-edit generated `main.js`,
keep each fragment at or under 1000 lines, and finish with `bob plugins sync` (never
`-F` on your own).

- **The selector and its dots (nav).**
  `plugins/bob-navigation-hotkeys/src/547-bracket-mode-indicator.js` paints the
  selector. `src/547a-bracket-mode-view.js` (`buildBracketModeView`) shows a segment's
  dot only while `attention[type]` is set **and** that family is not the active one.
  `src/482-walk-mode-nav-api.js` holds `api.walkMode` v4. Its
  `setAttention(type, {tier, label} | null)` currently accepts only `"s"` and `"b"`. The
  CSS in `plugins/bob-navigation-hotkeys/styles.css` draws a 5px `::after` dot
  (top-right, 1px ring in the status-bar background) colored by `--bob-review-attention`
  per `data-attention-tier`. The tier rules are near line 4301 (`pre` red, `new` orange,
  …), the `waiting` rule near line 3996 (`--text-warning`), and forced colors use
  `Highlight`.
- **Who pushes attention (ledger).**
  `plugins/bob-ledger-tools/src/240-plugin-freshness-status.js`
  `updateFreshnessStatusBar()` runs on every footer paint: cursor/doc changes
  (`contextOnly`), Tasks cache updates via `scheduleLiveWidgetRefresh`, Today cache
  changes, and the nav walk-mode doorbell. It calls `walkerFooterPushReviewAttention`
  (`src/122-walker-footer-host.js`) and `walkerFooterPushBankAttention`
  (`src/125-bank-walker-footer.js`). It fetches the lane-walk queue only inside the `w`
  branch. The unload hook in `setupFreshnessStatusBar` clears `s` (v3+) and `b` (v4+).
- **The `b` bug (root cause).** `onload` in `src/170-plugin-lifecycle.js` calls
  `setupStatusBarItems()` (→ `setupFreshnessStatusBar` → `subscribeBankWalkerFooter` →
  `onBankWalkChange`, which lazily creates `this.bankWalkListeners` and adds the footer
  listener) **before** `setupBankWalk()` (`src/249a-plugin-bank-walk-index.js`).
  `setupBankWalk` then does `this.bankWalkListeners = new Set()` and silently drops that
  listener. Consequences:
  - The footer never repaints when the bank index becomes ready at startup (the first
    paint sees `available: false, reason: "indexing"` and pushes `null`).
  - The footer never repaints when `bob bank import` or another device changes
    `bank/YYYY-MM.md` on disk.

  The `b` dot only catches up on the next unrelated repaint, such as a cursor move.
  Existing tests always subscribe _after_ `setupBankWalk`, so they never caught it.
  (Today the vault has 0 waiting rows, so a missing `b` dot is also correct right now.)

- **What "badge is red" means.** `workBadgeTone(count, cap)`
  (`src/075-work-badge-tone.js`) returns `"red"` exactly when `count > cap`; at cap is
  magenta. The PENDING/NEXT badges (`dashboardLaneBadgeModel`,
  `src/070-ready-and-review.js`) color the dash **section** count, which excludes Today,
  against `effectivePlanCaps(caps).maxPending` / `.maxNext`. The READY badge
  (`readyBadgeModel`) colors the freshness-gated READY count against
  `planReadyCapOrDefault(effective.maxReady, READY_FALLBACK_CAP)`. The lane-walk queue
  (`laneWalkQueueFromTasks`, `src/071-lane-walk-model.js`; plugin `apiLaneWalkQueue` /
  `laneWalkApi().queue()` in `src/180-plugin-plan-and-ready.js`) is memoized and
  documents that `counts.pending` / `counts.next` equal the dash section counts and
  `counts.ready` equals the gated READY count on the same snapshot. So the `w` dot can
  read the queue's counts with no extra scan.
- **The `w` panel today.** `laneWalkerFooterView` (`src/123-lane-walker-footer.js`)
  shows `n/cap` for a lane only when it is over its cap. It shows that text with no
  over-limit color, and it omits the current lane from the groups strip. Its cap lookup
  (`caps[lane.cap]`) does not use the READY fallback the badge uses.
  `freshnessFooterPaintGroups` (`src/121-freshness-footer-paint.js`) paints group tokens
  with `data-review-tier` lane hues.
- **History.** The walker-footers epic (`plan:202611/walker_footers.md`, adjustment A4)
  rejected a `w` pip that lit whenever lanes were non-empty ("permanent wallpaper").
  This plan triggers only on **over-limit**, the same state the dash already paints red.

## Design

### The three dots: one rule, one voice

A dot means: **"the walk under this letter needs you, and you're not on it."** Every dot
keeps the existing geometry (5px, top-right, ring, no number, no motion, `Highlight`
under forced colors). Every dot shows only while its family is **not** the active walk;
on the active walk, the panel itself carries the message. Every dot clears the moment
its condition clears. Unavailable data (Tasks indexing, Today loading, bank indexing,
old nav) never lights a dot.

| Letter | Lights while                                       | Color                                                                   | Tooltip / aria label                                                   |
| ------ | -------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `s`    | review commitments due (unchanged)                 | first due tier's hue (see `s_dot_hues`)                                 | `Review: 9 due · 5 commitments — PRE first; click to show` (unchanged) |
| `w`    | any PENDING/NEXT/READY count is over its limit     | badge red, `var(--color-red, #c65367)` (the `.bob-work-tone-red` value) | `Lanes over limit: WIP 12/10 · READY 104/100 — click to show`          |
| `b`    | at least one unticked bank row (unchanged trigger) | amber `--text-warning` (see `b_dot_hue`)                                | `Bank: 3 waiting · oldest Nov 14 — click to show` (unchanged)          |

The `w` tooltip lists only the over lanes, in dash order (WIP, NEXT, READY). It uses the
`w` panel's own lane labels (`LANE_WALKER_FOOTER_LANES`: `WIP`, `NEXT`, `READY`), so the
tooltip and the panel the click reveals never disagree. Clicking the letter keeps
today's behavior: select that walk, never walk, and the dot hides because the family is
now active.

### The active `w` panel keeps the signal

The dot hides once `w` is selected, so the panel must show the overage, or the signal
would vanish exactly when Bryan acts on it:

- A lane over its limit renders its `n/cap` group token count in the same badge red, for
  example `WIP 12/10`. The text still carries the warning, so color is never the only
  cue.
- An over-limit lane is **never omitted** from the groups strip, even when it is the
  current lane. Today the current lane drops out. Releasing tasks with `Alt+N` then
  ticks the red count down, and the token disappears when the lane is back within its
  limit. A lane that is not over keeps today's rules.
- Caps in the panel, its tooltip rows, and the dot all come from one resolver that
  matches the badges, including the READY fallback.

> [!decision] b_dot_hue = red Change only the `waiting` tier rule to
> `var(--color-red, #c65367)`, update the docs to say "red", and, if `mem_walker_footer`
> is on, call the `b` dot red in the strand. Everything else is identical.

> [!decision] s_dot_hues = match Replace nav's `s` tier dot colors with the review
> footer's `--bob-review-hue` values from `plugins/bob-ledger-tools/styles.css`: `pre`
> cyan, `new` red, `projects` purple, `pending` in-progress/yellow, `next`
> task-status-next/green, `recurring` pink, `tickler` blue (same variables and
> fallbacks). Add a test that parses both stylesheets and asserts that every commitment
> tier maps to the same hue expression. With `keep`, leave the `s` tier rules untouched.

## Implementation

### 1. Nav: `api.walkMode` v5 accepts `w` (bob-navigation-hotkeys)

- `src/482-walk-mode-nav-api.js`: bump `walkMode.version` to `5`. Make `setAttention`
  accept `"s"`, `"w"`, and `"b"` (still refuse `"n"` and anything else with
  `invalid-type`). Update the header comment (v5: `w` attention for lanes over limit).
- `src/547a-bracket-mode-view.js`: replace the inline default-label ternary with a
  per-family default map (`s`: "Review commitments are due", `w`: "Lanes are over
  limit", `b`: "Bank rows are waiting"). Visibility logic stays as is.
- `styles.css`: add
  `button.bob-bracket-mode-seg.has-attention[data-attention-tier="over"] { --bob-review-attention: var(--color-red, #c65367); }`
  beside the other tier rules. Apply the `b_dot_hue` / `s_dot_hues` branches here.
- Keep `547-bracket-mode-indicator.js` at or under 1000 lines. It should need no change,
  because painting is already per segment.

### 2. Ledger: one cap resolver, the over-lane helper, and the `w` push (bob-ledger-tools)

In `src/123-lane-walker-footer.js` (or a new small fragment registered in
`src/fragments.json` in manifest order if size demands):

- `laneWalkerFooterCapFor(laneKey, caps)`: resolves caps exactly as the badges do.
  PENDING/NEXT use `effectivePlanCaps(caps).maxPending` / `.maxNext`; READY uses
  `planReadyCapOrDefault(effective.maxReady, READY_FALLBACK_CAP)`. Use it everywhere the
  panel reads a cap (`laneWalkerFooterGroups`, the tooltip `laneRows`).
- `laneWalkerFooterOverLanes(counts, caps)`: a pure function that returns a frozen
  ordered list of `{ key, label, count, cap }` for each lane where
  `workBadgeTone(count, cap) === "red"`. It never throws and returns `[]` on bad input.
- `walkerFooterPushLaneAttention(plugin, route, queue, caps)`, mirroring the bank push:
  - Feature-detect `walkMode.version >= 5` and `setAttention`. Old nav means no call.
  - Push `null` when `route.legacy`, when `route.mode === "w"`, when the queue is not
    `available === true`, or when no lane is over.
  - Otherwise push `{ tier: "over", label }` with the tooltip format above. Never throw.
- Groups: mark over lanes with `over: true`. Always include an over lane, even when it
  is the current lane, and keep today's rules for lanes that are not over.

In `src/121-freshness-footer-paint.js` `freshnessFooterPaintGroups`, toggle an `is-over`
class on a group token when `group.over === true`.

In `plugins/bob-ledger-tools/styles.css`, add an over-limit count ink for
`.bob-freshness .bob-freshness-group.is-over .bob-freshness-group-count`. Use the badge
red hue and the **same ink recipe** as `.bob-freshness [data-review-tier]`: the
`color-mix` fallback plus the `@supports (color: oklch(from red l c h))` oklch ink with
`--bob-review-ink-l`. Redeclare both `--bob-review-hue` and `--bob-review-ink` on the
count span, since an inherited `--bob-review-ink` would keep the lane hue. Light and
dark text must pass the walker-footer preview's 4.5:1 contrast gate.

In `src/240-plugin-freshness-status.js` `updateFreshnessStatusBar`:

- Hoist the lane-queue fetch (`laneWalkApi().queue()`, v2 guard as today) and the caps
  load (keep the existing `this.laneWalkerFooterCaps` cache that reloads on
  non-`contextOnly` paints) **above** the attention pushes.
- Call `walkerFooterPushLaneAttention(this, route, queue, caps)` after the bank push.
- Pass that **same** `queue` and `caps` into the `w` branch instead of re-fetching. One
  immutable snapshot per paint feeds both the dot and the panel.
- In the unload hook, also call `setAttention("w", null)` when `walkMode.version >= 5`.

Export the new helpers through `src/354-exports-walker-footer.js` for tests.

### 3. Reliability fixes

- `src/249a-plugin-bank-walk-index.js` `setupBankWalk`: preserve listeners registered
  before setup. Replace `this.bankWalkListeners = new Set()` with "create only if not
  already a Set", and add a comment explaining the onload ordering. `teardownBankWalk`
  still clears them.
- `src/180-plugin-plan-and-ready.js` `checkReadyCapsAndDay`: when caps or the day
  change, also `scheduleFreshnessStatusBar()`. A `plan:` cap edit in
  `~/.config/bob/config.yml` then recolors the `w` dot and panel within the existing
  60-second tick, in step with the badges.

### 4. Tests (extend the existing suites; all run in `npm test`)

- `scripts/test-navigation-bracket-mode-indicator.cjs`:
  - v5 `setAttention` accepts `w`, still refuses `n`, and reports the version.
  - The `w` dot paints only while mode ≠ `w`, with `data-attention-tier="over"`, title,
    and aria text.
  - `s`, `w`, and `b` dots coexist on mode `n` and on unset mode.
  - Per-family default labels.
  - The `over` tier rule exists, carries no motion, and is covered by the forced-colors
    rule.
  - For `s_dot_hues = match`, add the cross-stylesheet hue parity test.
- `scripts/test-ledger-tools-walker-footer.cjs`:
  - `laneWalkerFooterOverLanes` handles boundaries: at cap is not over, cap+1 is over,
    and the READY fallback cap applies.
  - Push conditions mirror the existing bank tests: over off-`w` lights; `null` on `w`,
    on no overage, on an unavailable queue, and on legacy routes; old nav (v4) makes no
    `w` call; a throwing nav never escapes; unload on v5 clears `w` without disturbing
    v3/v4 behavior.
  - The tooltip lists only over lanes, in dash order.
  - The panel and the dot read the same queue object within one paint.
  - Over groups carry `over: true` and `is-over`, an over current lane stays in the
    strip, and a lane that is not over keeps today's omission.
- **Badge parity test** (new, e.g. in `scripts/test-ledger-tools-dashboard-parity.cjs`
  or the walker-footer suite): build Tasks fixtures that are under, at, and over each
  cap, including Today-linked and NEW/ROTTEN-gated Ready rows. Assert that the set of
  lanes `laneWalkerFooterOverLanes(laneWalkQueueFromTasks(...).counts, caps)` reports
  equals the set whose real badge models report `tone === "red"`:
  `dashboardLaneBadgeModel(dashboardLaneBudgetFromTasks(...))` for pending/next and
  `readyBadgeModel(readyBudgetFromTasks(...))` for ready, on the same tasks, caps, Today
  predicate, and review gate. This test pins "the `w` dot means a badge is red".
- `scripts/test-ledger-tools-bank-walk-index.cjs`: a regression test in onload order.
  Subscribe with `onBankWalkChange` **before** `setupBankWalk()`, then finish the
  initial scan and a vault `modify` of a month note. The listener must fire both times.
  Add a host-level check that the `b` attention is pushed once the index is ready, with
  no cursor event.
- `checkReadyCapsAndDay` schedules a footer repaint on a caps change.

### 5. Visual verification (make it beautiful, then prove it)

- `scripts/preview-bracket-mode.cjs`: add fixtures with all three dots lit (`s` PRE, `w`
  over, `b` waiting) on mode `n`, the `w` dot alone on mode `s`, and mode `w` with the
  `s` and `b` dots. Render light and dark, look at the screenshots, and confirm the dots
  are crisp, evenly placed, and distinguishable, and that the selector rect is
  unchanged.
- `scripts/preview-walker-footer.cjs` (and its template): add `w` panel fixtures with
  one and two over lanes, including an over current lane, in light and dark. The
  existing 4.5:1 contrast gate must pass for the new `is-over` counts.
- `scripts/obsidian-walker-footer-e2e.cjs`: extend the scenario.
  - Point the spawned Obsidian at a scratch `XDG_CONFIG_HOME` whose `plan:` caps are low
    enough that one lane is over. Assert the `w` dot is lit off-`w` with tier `over`,
    and that selecting `w` clears it and shows a red `is-over` token.
  - Seed a bank month note with one waiting row. Assert the `b` dot appears after
    startup **without** a cursor move; this is the listener regression.
  - Keep `--self-test` and the offline wrapper passing.
  - Run the live e2e only where an Obsidian binary exists; otherwise report it as
    skipped in your final notes.

### 6. Docs, versions, deploy

- `docs/bob-navigation-hotkeys.md`: in the `api.walkMode` passage, add "v5 adds `w`
  attention, so lanes over their limit show a red dot on `w`". Note that dots coexist
  for `s`, `w`, and `b`, and update the manifest description's "per-family `b` attention
  through nav `api.walkMode` v4" accordingly.
- `docs/bob-ledger-tools.md`: in the walker-footer section, cover the `w` panel's red
  `n/cap` over tokens (an over lane stays visible even when it is the current lane) and
  the red `w` dot pushed through v5 when any PENDING/NEXT/READY badge would be red.
- `docs/develop/plugin-apis.md`: `setAttention` accepts `"w"` at v5, and ledger
  feature-detects `walkMode.version >= 5` before pushing it.
- Apply the `b_dot_hue` / `s_dot_hues` wording if those branches are chosen.
- Bump the minor manifest version of both changed plugins relative to master at
  implementation time (today nav `2.83.2` → `2.84.0`, ledger `1.119.0` → `1.120.0`).
- Run `npm run build`, `npm run docs:gen`, `npm run docs:check`, `npm run build:check`,
  `npm run validate`, and the full `npm test`.
- Finish with `bob plugins sync`. If it refuses an overwrite, follow AGENTS.md exactly.
  Confirm the synced versions in the vault.

### 7. Memory (bob-cli workspace)

> [!decision] mem_walker_footer

Use `/sase_memory_write`. In `sase/memory/glossary/walker-footer.md`, replace the quoted
dot sentence with one sentence covering all three off-family dots. For example:
"Off-family dots mark the selector: `s` while review commitments are due, `w` in the
badge red while any PENDING/NEXT/READY lane is over its limit, and `b` in amber while
bank rows wait." Say red for `b` under `b_dot_hue = red`. Then run `sase memory init`,
`just docs-gen` (regenerates `docs/reference/glossary.md`), and `just docs-check`.

## Acceptance

- With mode `s`, `n`, `b`, or unset, a lane over its limit shows a red dot on `w` whose
  tooltip names exactly the over lanes. It clears as soon as every lane is back within
  its limit, and it clears when `w` is selected.
- The `w` dot is lit **if and only if** at least one PENDING/NEXT/READY badge would be
  red. The parity test proves it.
- With mode `w`, every over lane shows a red `n/cap` token, including the current lane.
- The `b` dot appears at startup and after an on-disk bank import without any cursor
  movement, and it clears when the last waiting row is ticked.
- No dot ever lights from unavailable data; old nav or old ledger degrade silently.
- All gates pass, previews were inspected in light and dark, and `bob plugins sync`
  deployed both plugins.

## Out of scope

The TODAY link-cap badge, per-note Ready caps, and any `n` dot. No change to walk
membership, answer keys, badge math, bank semantics, or any vault write. No bob-cli
Rust/CLI change, no release-train or Bob Mac Capture impact.
