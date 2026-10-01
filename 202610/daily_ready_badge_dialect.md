---
tier: tale
title: Daily READY badge matches its neighbors and raises a READY over-cap lint
goal:
  In daily notes, READY looks exactly like the PENDING and NEXT chips and shows a
  ready_cap_exceeded lint line when over its cap. The dash READY chip keeps its two-tone
  style, and the shared badge logic stays in one place.
size: small
proposed_by: bbugyi200.apollo.3x.f0
create_time: 2026-10-01 14:29:36
status: wip
---

# Daily READY badge: match the daily chips and add a READY over-cap lint

## Problem

Two complaints from Bryan, with screenshots (`~/tmp/screenshots/20261001_135152.png` for
the daily note and `~/tmp/screenshots/20261001_104952.png` for the earlier dashboard):

1. In daily notes, the READY badge in the ` ```bob-plan ` block looks different from its
   neighbors. TODAY, PENDING, and NEXT render as one dark, bold, small-caps text run
   (`PENDING 50/10`). READY renders a small muted `READY` label next to a larger red
   `211/100` value.
2. When READY is over its cap (`211/100`), the block shows lint lines for NEXT
   (`next_cap_exceeded`) and PENDING (`pending_cap_exceeded`) but none for READY.

Bryan suspects the daily notes and `~/bob/dash.md` need different badges.

## Diagnosis

### Is the suspicion right? Partly

**Yes:** the two pages need different _presentations_. No single style can match both,
because each page styles its chips differently:

- **`dash.md`** uses a two-tone style. Its inline `<style>` (`.task-count-*`) gives each
  chip a muted `0.78em` small-caps label and a separate `0.9em`, weight-750,
  tabular-figure value in the chip's accent color.
- **The daily `bob-plan` block** uses a single-tone style. In bob-ledger-tools
  `styles.css`, each `.bob-plan-chip` (TODAY, PENDING, NEXT) is one text node such as
  `PENDING 50/10`, rendered in `--text-normal`, `all-small-caps`, weight 650, with
  `0.045em` letter-spacing. The accent color only tints the chip's border and
  background.

**No:** they do not need two badge implementations. The count, cap, over state, tooltip,
aria label, `dash#READY Tasks` navigation, keyboard and hover handling, the label/value
span structure, and live refresh can all stay shared. Only the CSS should depend on
which page hosts the badge.

### Root cause 1: one style was forced onto both pages

READY is the only chip that both pages draw with the same shared renderer
(`paintReadyElement`, which `renderReadyBadge` also calls). The badge has swung between
the two styles:

- **Before 1.9.2**, READY was flat text. It matched the daily chips but looked wrong on
  the dashboard (screenshot `20261001_104952.png`: large dark `READY 210/100`).
- **The `ready_badge_style` plan (commit `58b6200`)** added `.bob-plan-ready-label` and
  `.bob-plan-ready-value` spans. It styled them with unscoped rules
  (`.bob-plan-ready .bob-plan-ready-label`, `.bob-plan-ready .bob-plan-ready-value`),
  and the `styles.css` comment states the goal outright: "so both the daily `bob-plan`
  wrapper and the dashboard host render identically".

That goal is the bug. It applies the dashboard's two-tone style inside the daily block.
The same unscoped rules also give the daily READY a `translateY(-1px)` hover lift and a
120ms transition, which the daily PENDING and NEXT chips don't have.

Commit `854bdbe` (the fix for the Obsidian `.text` setter) is unrelated. Neither the
markup nor the counting logic is wrong; only the CSS scope is.

### Root cause 2: the READY lint never existed

This is a gap in the spec, not a rendering regression:

- `planBlockModel` in `plugins/bob-ledger-tools/main.js` adds lane lints only for NEXT
  and PENDING (`PLAN_LINT_NEXT_CAP`, `PLAN_LINT_PENDING_CAP`). A READY overflow only
  turns the badge red.
- bob-cli `docs/plan.md` defined READY without a lint: "its feedback is the shared
  badge; this feature adds no native READY count, lint, capture enforcement, or tmux
  meter".
- Rust `bob plan` has no READY count, so it cannot raise this lint.
- `planBlockModel`'s aggregate `over` flag, which adds ", over plan" to the block's aria
  label, also ignores READY. NEXT and PENDING overflows already set it.

## Decision

Keep one shared READY badge, and choose its CSS by the page that hosts it:

- **Inside the daily `.bob-plan` block**, the READY label and value spans inherit the
  chip's single-tone typography. The badge then looks exactly like `PENDING 50/10`.
- **Everywhere else** (`dash.md` and any other `renderReadyBadge` host), the existing
  two-tone rules stay unchanged.

Add a bob-ledger-tools-only `ready_cap_exceeded` lint to the daily block, and let READY
contribute to the block's aggregate `over` flag. The PLAN `status` stays unchanged.

Rejected alternatives:

- **A separate flat-text READY renderer for daily notes.** It would duplicate the
  anchor, tooltip, aria, navigation, and listener code. It would also reopen the
  Obsidian `.text` setter problem fixed in `854bdbe`, because `setReadyAnchorContent`
  deliberately removes direct text nodes.
- **Restyling the daily TODAY, PENDING, and NEXT chips in the two-tone style.** That
  would also make the chips consistent, but it changes how the whole daily block looks,
  which Bryan didn't ask for. It is a contained follow-up if he later wants one style
  everywhere.
- **A JS `variant` option on `paintReadyElement`.** It would have to survive
  `setReadyAnchorContent`, which resets the `class` attribute on every refresh. CSS
  scoped to the `.bob-plan` ancestor gets the same result with no API change.
- **A native READY lint in `bob plan`.** There is no native READY count, and freshness
  gating exists only in the plugin (`docs/plan.md`, decision
  `ready-is-freshness-gated`).

## Coordination constraints (read first)

- **Concurrent work.** Epic `bob-cli-3b` (freshness-gated READY) is still active. Phase
  `bob-cli-3b.3` (rotten vocabulary rename, freshness namespace v3) is `IN_PROGRESS` and
  edits the same plugin `main.js`, `README.md`, and `manifest.json`. To limit conflicts:
  - Start from the latest bob-plugins `origin/master`.
  - Keep edits limited to the places named below. Rename nothing.
  - Fetch again before finishing and rebase cleanly if master moved.
  - Bump the version from whatever `manifest.json` says at implementation time. It was
    `1.12.0` at planning time.
- **Deployment gate.** The dash-gating plugin releases (1.11.0 and later, including
  gated READY and the status-bar fallback to `rotten`) are on bob-plugins master but
  have not been deployed together with their vault changes. Bead `bob-cli-3b.2` note #2
  records the pending push of vault commit `f7d7a2c9`, the sync, and the live
  verification. At planning time:
  - `~/bob` on apollo had bob-ledger-tools 1.10.0.
  - `~/bob` had no `rotten.md`.

  Deploying from master would ship the epic's changes ahead of its vault rollout. Step 8
  makes deployment conditional for that reason.

## Implementation

### 1. Open the repos

- Use `/sase_repo` to open `bob-plugins` with a reason, and read its `AGENTS.md`.
- Edit bob-cli `docs/plan.md` in your own workspace checkout.
- Don't edit anything under `~/bob/.obsidian/plugins` by hand.

### 2. `plugins/bob-ledger-tools/styles.css`: CSS scoped to the daily block

Keep the existing unscoped two-tone rules as they are. Those are
`.bob-plan-ready .bob-plan-ready-label`, `.bob-plan-ready .bob-plan-ready-value`,
`a.bob-plan-ready:hover`, and the `.bob-plan-ready` transition. They remain the default
for the dashboard and other hosts.

1. **Rewrite the comment above the READY span rules.** It should state the actual rule:
   the two-tone label/value style is the default, matching the dashboard's
   `.task-count-*` chips; inside the daily `.bob-plan` block, the spans take on the
   block's single-tone style so READY reads like `PENDING n/10`. Delete the "render
   identically" sentence.

2. **Add one commented section** next to those rules, scoped to the daily block:

   ```css
   /* Daily `bob-plan` block: shared split chips speak the block's
      single-tone chip language (one dark small-caps run like
      `PENDING 50/10`); the accent only tints border and background. */
   .bob-plan .bob-plan-ready,
   .bob-plan .bob-plan-review {
     gap: 0;
     transition: none;
   }

   .bob-plan a.bob-plan-ready:hover {
     transform: none;
   }

   .bob-plan .bob-plan-ready .bob-plan-ready-label,
   .bob-plan .bob-plan-ready .bob-plan-ready-value,
   .bob-plan .bob-plan-review .bob-plan-review-label,
   .bob-plan .bob-plan-review .bob-plan-review-value {
     color: inherit;
     font-size: inherit;
     font-weight: inherit;
     font-variant-caps: inherit;
     font-variant-numeric: inherit;
     letter-spacing: inherit;
   }

   /* A real (no-break) word space instead of the flex gap, so the
      label/value spacing equals the lane chips' `PENDING 50/10`. */
   .bob-plan .bob-plan-ready .bob-plan-ready-value::before,
   .bob-plan .bob-plan-review .bob-plan-review-value::before {
     content: "\00a0";
   }
   ```

   Why these selectors win:
   - **Specificity.** `.bob-plan .bob-plan-ready …` is (0,3,0), beating the unscoped
     (0,2,0) span rules. `.bob-plan a.bob-plan-ready:hover` beats
     `a.bob-plan-ready:hover`. `.bob-plan .bob-plan-ready` beats the `.bob-plan-chip`
     gap.
   - **Over state.** `.bob-plan-over` and the blue accent still tint only the border and
     background, exactly as on PENDING and NEXT.
   - **Unchanged.** The `.bob-plan-unavailable` dimming and the shared `:focus-visible`
     outline stay as they are.

   The `.bob-plan-review` selectors anticipate the NEW and ROTTEN chips that the docs
   already promise for the daily block (see Out of scope). The same bug would otherwise
   hit them the day `renderReviewChip` is hosted there. Today they match nothing.

3. **Adjust if the rules don't apply.** If a live check shows the generated no-break
   space is not applied, adjust the selector or declaration so the rendered spacing
   still equals a word space. Don't fall back to an approximate `gap`.

### 3. `plugins/bob-ledger-tools/main.js`: READY lint and aggregate `over`

1. **Add the lint code.** Add `const PLAN_LINT_READY_CAP = "ready_cap_exceeded";` next
   to `PLAN_LINT_PENDING_CAP`. Add a one-line comment saying it is emitted only by the
   bob-ledger-tools `bob-plan` block, because `bob plan` has no native READY count.

2. **Emit the lint in `planBlockModel`,** directly after the PENDING lint push.
   - **Condition:** `hasTasks && ready && Number.isInteger(ready.count) && ready.over`.
     Keep the strict `count > cap` boundary.
   - **Code:** `ready_cap_exceeded`.
   - **Message:**
     `` `READY has ${ready.count}/${ready.cap} tasks; prune at the weekly review` ``.
     This matches the "prune at the weekly review" hint that bob-navigation-hotkeys
     already shows for over-cap lanes.

   The block then renders the line
   `READY has 211/100 tasks; prune at the weekly review ready_cap_exceeded`, in the same
   `.bob-plan-lint` style, after NEXT and PENDING. Its numbers are the gated count and
   cap that the daily badge displays.

3. **Count READY in the aggregate `over` flag.** Add `(hasTasks && ready && ready.over)`
   to `over`. Leave `budget.status` and PLAN enforcement untouched.

4. **Update the `paintReadyElement` header comment.** Say that the structure, classes,
   fraction, tooltip, and destination are shared, and that the styling depends on the
   host page through `styles.css`. Don't change any other code.

### 4. Tests (bob-plugins `scripts/test-ledger-tools-ready-badge.cjs`)

Add these tests, reusing the existing `readyTask`, `planBlockModel`, and
`defaultPlanCaps` helpers. Remember that unstamped tasks fall into NEW when a gate
predicate is supplied; the existing fixtures without a gate stay ungated.

1. **Lint boundaries.**
   - 101 READY tasks with the default cap produce exactly one lint ending in
     `ready_cap_exceeded`. Its message is
     `READY has 101/100 tasks; prune at the weekly review`.
   - `model.over === true`, and `model.budget.status` stays `"ok"`.
   - Exactly 100 tasks produces no READY lint, and `over` is false when nothing else is
     over.
   - A custom cap (`maxReady: 5` with 6 tasks) uses the custom numbers.

2. **Lint ordering and count source.**
   - With NEXT, PENDING, and READY all over, the lints appear in the order NEXT,
     PENDING, READY.
   - When an `isReviewBucket` gate is supplied, the READY lint uses the gated count. Its
     numbers must equal `model.readyText` and `model.ready`.

3. **No lint while unavailable.**
   - `tasks: null` produces no READY lint (the badge shows `READY –`).
   - A throwing READY evaluation (`ready === null`) produces no READY lint.

4. **Stylesheet contract.** Follow the pattern of the existing `styles.css` assertions
   in `scripts/test-navigation-hotkeys.cjs`: read the file and normalize whitespace.
   Assert that:
   - The unscoped two-tone rules still exist: the label uses `color: var(--text-muted)`
     and the value uses `color: var(--bob-plan-accent)`.
   - A rule whose selector list includes
     `.bob-plan .bob-plan-ready .bob-plan-ready-label` and the matching `-value`
     selector sets `color`, `font-size`, `font-weight`, `font-variant-caps`, and
     `letter-spacing` to `inherit`.
   - `.bob-plan .bob-plan-ready` sets `gap: 0`.
   - `.bob-plan a.bob-plan-ready:hover` sets `transform: none`.
   - A `.bob-plan .bob-plan-ready .bob-plan-ready-value::before` rule sets the
     no-break-space `content`.

   Name the test so a future "unify the READY style" change fails loudly. For example:
   "READY speaks each host's chip language (daily single-tone, dash two-tone)".

Leave the existing contract tests unchanged. Those are "daily and dashboard share the
READY element contract", the span-structure tests, and "READY over is independent of the
PLAN ledger status". If an existing assertion contradicts the new `over` semantics,
update that single assertion and explain why in the test.

### 5. Version and plugin README

- **Version.** Bump the bob-ledger-tools `manifest.json` patch version over the current
  value at implementation time, and keep the matching version cell in the bob-plugins
  root `README.md` table in sync.
- **README prose for the `bob-plan` block** (the paragraph starting "Bob Ledger Tools
  also renders a live"). Mention that READY over its cap adds a `ready_cap_exceeded`
  lint line.
- **README `renderReadyBadge` sentence.** Keep "daily and dash use the same element,
  classes, fraction, tooltip, and `dash#READY Tasks` destination", and add that each
  host styles it in its own chip style: two-tone label/value on the dash, single-tone
  inside the daily `bob-plan` block.
- **Don't touch** the table row's NEW/ROTTEN wording (see Out of scope).

### 6. bob-cli `docs/plan.md`

- **Lints table.** After `pending_cap_exceeded`, add:
  ``| `ready_cap_exceeded` | READY is over the cap; Obsidian `bob-plan` block only (`bob plan` has no READY count); never changes `status`, nothing is refused |``.
- **"READY backlog" section.** Replace "this feature adds no native READY count, lint,
  capture enforcement, or tmux meter" with wording that keeps "no native READY count,
  `bob plan` lint, capture enforcement, or tmux meter". Add that the daily `bob-plan`
  block shows a `ready_cap_exceeded` lint line beneath its chips when READY is strictly
  over the cap. A cap exactly met raises no lint.
- **Leave the Surfaces table rows alone.** They already say "and any lints".

### 7. Bead notes

Read `sase_beads.md` with `/sase_memory_read`, then append notes to the active epic
`bob-cli-3b` (don't change its status):

- **Discovered issue.** Add
  `DISCOVERED ISSUE: bob-cli docs/plan.md (Surfaces row for the daily bob-plan block, from 286ff35) and the bob-plugins README table row (from d5c1281) say the daily bob-plan block renders NEW and ROTTEN chips, but renderPlanBlock renders only TODAY, PENDING, NEXT, and READY.`
  The memory routes issues caused by an active epic onto that epic rather than into a
  new task bead.
- **Skipped deployment.** Only if step 8 skips deployment, add a note that the
  bob-ledger-tools version from step 5 also carries this fix. The dash-gating rollout's
  live check should then confirm that the daily READY badge matches PENDING and NEXT,
  and that the `ready_cap_exceeded` line appears when READY is over the cap.

### 8. Validate, then deploy only if the gate allows

1. **Validate in bob-plugins:**
   - `node --test scripts/test-ledger-tools-ready-badge.cjs scripts/test-ledger-tools-plan-budget.cjs`
   - `npm test`
   - `npm run validate`
   - `git diff --check`

   Report baseline failures you didn't cause as baseline, with their output.

2. **Check the gate.** Deploy only if the dash-gating vault rollout has reached `~/bob`
   on this machine, meaning `~/bob/rotten.md` exists after the normal vault sync. In
   that case:
   - Run
     `bob plugins sync --no-pull --repo <opened-bob-plugins> --plugin bob-ledger-tools --dry-run`.
   - Then run the same command without `--dry-run`. Never use `--force`.
   - Confirm the deployed `manifest.json` version.

   If `rotten.md` is absent, don't deploy. Record the step 7 note and report the
   deployment as gated on the dash-gating rollout.

3. **Mac.** Bryan's screenshots come from the Mac, which needs its own
   `bob plugins sync`. Only when the gate holds and the Mac is reachable, read
   `tailnet.md` with `/sase_memory_read` first, then deploy the same way there.
   Otherwise report the Mac deployment as remaining.

4. **Visual parity.** Automated tests can't prove it. If no Obsidian GUI is available,
   say exactly what remains unverified (see the acceptance criteria).

## Acceptance criteria

- **Daily block.**
  - READY renders as one dark, bold, small-caps run, `READY 211/100`, with the same
    size, weight, color, letter-spacing, word spacing, and height as `PENDING 50/10` and
    `NEXT 32/15`.
  - Its border and background tint is blue under the cap and red over it.
  - Hover only darkens the tint, with no lift. Focus shows the shared outline.
  - This holds in Live Preview and Reading view, in light and dark themes.
- **Dashboard.** `dash.md` READY is unchanged: muted label, accent value, matching
  BLOCKED and the other dashboard chips, including the hover lift.
- **READY lint.**
  - When READY is strictly over its cap, the daily block shows
    `READY has n/cap tasks; prune at the weekly review  ready_cap_exceeded` after any
    NEXT and PENDING lint lines.
  - At or under the cap, or while READY is unavailable (`READY –`), there is no READY
    lint.
  - PLAN `status` never changes. The block's aria label reports ", over plan" when READY
    is over.
- **Shared behavior is unchanged.** READY counting, gating, tooltips, aria, navigation,
  keyboard, hover preview, live refresh, and the `.text` setter safety all behave as
  before. The public API signatures are unchanged.
- **Checks pass.** The focused suites, `npm test`, `npm run validate`, and
  `git diff --check` all pass. The manifest and README versions agree.
- **Docs and notes.** bob-cli `docs/plan.md` documents `ready_cap_exceeded` as
  plugin-only. The `bob-cli-3b` notes are recorded.
- **Deployment.** Either it happened under the gate (with the dry-run and the deployed
  version reported), or it is reported as gated, with the Mac status stated.

## Out of scope

- Adding NEW and ROTTEN chips to the daily block, or correcting the docs that claim it
  already has them. That stays with epic `bob-cli-3b` through the step 7 note. The
  scoped CSS above already covers those chips if they are added.
- Switching the daily TODAY, PENDING, and NEXT chips to the two-tone style.
- Any Rust change. Any change to READY counting or gating, the dashboard's inline
  fallback chip, `dash.md`, or other vault notes.
- Deploying the dash-gating releases ahead of their vault rollout.
