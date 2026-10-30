---
tier: tale
title: Review footer tier colors
goal:
  Each review footer group and the current tier wear a distinct, legible, theme-native
  tier color, with a commitments/upkeep divider, progressive narrow-bar fitting, and
  hover feedback.
size: medium
decisions:
  glossary_footer_strand:
    ask:
      Update the Morning GTD Review Footer glossary strand for tier colors and the new
      fit order?
    memory:
      - glossary:morning-gtd-review-footer
    default: true
    corrects:
      tooltip, and it drops groups, then detail, hint, and meter as space runs out.
    answer: true
proposed_by: bbugyi200.apollo.ba
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 13:29:09
status: wip
---

# Plan: Review footer — a distinct color for each review group

## Goal

Give every review group in the bob-ledger-tools review footer (the Obsidian status-bar
item for the morning `]s` walk) its own text color, and fix the smaller UX problems that
surfaced while designing it. The footer stays a read-only projection. The counts,
labels, tooltip text, aria-label, click target, and hide rules all stay as they are;
only how the groups and the current tier are painted and fitted changes.

The design mock (light and dark, five footer states, built from the CSS below) is the
image artifact `file:explicit:7544763e8a2d022c6c9371fb`.

## Today's problems

1. All groups render as one muted text run (`NEW 1 · WIP 2 · TICKS 1 · ROTTEN 3`). They
   use the same `·` separator as the context, detail, and hint parts around them, so the
   eye cannot tell where the group cluster starts or which tier is which.
2. The current-tier context (`NEW 1/2`) is plain bold text, or the interactive accent
   whenever any NEW task exists. That second rule colors a ROTTEN context with the
   accent too, because the rule keys off "NEW exists" rather than the current tier.
3. When space runs out, every group disappears at once. A slightly narrow window loses
   the whole cluster even though most of it would fit.
4. The main button gets no hover feedback, unlike native clickable status-bar items.
5. The footer splits TICKS from ROTTEN so the boundary between commitments and upkeep is
   visible. In practice nothing marks that boundary except the order.

## Design

### One color (ink) per tier, by walk order

| Tier      | Footer label | Hue source                                                    | Why this hue                                                                                              |
| --------- | ------------ | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| PRE       | PRE          | `--color-cyan`                                                | Matches the teal `#gtd #pre` sunrise identity mark                                                        |
| NEW       | NEW          | `--color-red`                                                 | The dashboard NEW chip turns red while unconfirmed tasks exist                                            |
| PROJECTS  | PROJECTS     | `--color-purple`                                              | The dashboard PROJECTS chip uses the (default purple) accent                                              |
| PENDING   | WIP          | `--task-status-in-progress`, falling back to `--color-yellow` | The vault's In Progress status color                                                                      |
| NEXT      | NEXT         | `--task-status-next`, falling back to `--color-green`         | The vault's Next status color                                                                             |
| RECURRING | RECUR        | `--color-pink`                                                | The remaining hue; it sits between green NEXT and blue TICKS                                              |
| TICKLER   | TICKS        | `--color-blue`                                                | Blue against orange is the strongest contrast at the commitments/upkeep boundary                          |
| ROTTEN    | ROTTEN       | `--color-orange`                                              | The dashboard ROTTEN warn chip and the orange due freshness mark                                          |
| POST      | POST         | `--text-normal` (no hue)                                      | The closeout "finish flag". Obsidian has only eight named hues, and the other eight tiers use all of them |

How these were chosen and checked:

- **Theme-native hues.** Each hue comes from the theme's named colors or the vault's
  `--task-status-*` lane colors, so community themes and the `task-statuses` snippet
  stay coherent with the footer.
- **Equal lightness.** Each ink keeps the hue's own hue angle and chroma (capped at
  0.16), but lightness is fixed per theme in OKLCH: 0.52 in light, 0.80 in dark. No tier
  shouts louder than another, and every ink clears WCAG AA on the status bar.
  - Measured against Obsidian's default theme, with the status bar on
    `--background-secondary`: light is 4.54–5.6:1 and dark is 6.9–8.6:1.
  - POST, which uses `--text-normal`, measures 14.7:1 in light and 10.8:1 in dark.
- **Distinct neighbors.** The mapping was chosen by search: the fixed anchors were WIP,
  NEXT, ROTTEN, and PRE, and the search maximised the perceptual distance between groups
  that sit next to each other or one apart in walk order, including when the current
  tier's group is removed. The closest such pair, NEXT and WIP in dark, is OKLab ΔE
  ≈ 11. Red NEW and orange ROTTEN are the closest pair overall (ΔE ≈ 7), but six
  positions separate them.
- **Pink is safe to use.** The magenta "count exactly at its limit" tone is reserved
  only among `.bob-plan-chip` badges. Footer groups are not chips and never express
  limits.
- **Text still carries the meaning.** Color never carries information alone: labels and
  counts are unchanged text, and the tooltip and aria-label keep the full summary.

### Typography and layout

- **Group tokens.** Each group is one token: the label in tier ink at
  `var(--font-medium, 500)` and the count in tier ink at `var(--font-bold, 700)`,
  separated by a CSS no-break space. This copies the bob-plan chips'
  `::before { content: "\00a0" }` precedent. The result is calmer than all-bold, and the
  counts read first.
- **No separators inside the cluster.** Groups no longer have `·` between them; a 0.7em
  gap separates them, so the cluster reads as one legend. The parts outside the cluster
  keep their existing `·` separators.
- **Commitments/upkeep divider.** A hairline divider (`border-inline-start` in
  `--background-modifier-border-hover`) goes before the first ROTTEN or POST group, but
  only when a commitment group precedes it in the cluster. It makes the boundary that
  the footer already orders by visible.
- **Current-tier context.** `NEW 1/3`, `ROTTEN 1/9` and the like wear their own tier's
  ink at weight 600. Seeing a tier's color in the context and in the cluster ties
  position to group.
- **No accent on NEW.** The interactive-accent overrides for NEW mode are removed: one
  on the due text and one on the context. In the default theme the accent is purple and
  would collide with PROJECTS, and NEW's own red ink now carries that signal. The `mode`
  classes stay as hooks with no styling.
- **Hover feedback.** Hovering the main button shows `--background-modifier-hover` with
  `--radius-s`, matching native clickable status-bar items.

### Fitting a narrow status bar

When the footer overflows, groups are dropped one at a time from the end of the walk
order, which sheds upkeep and closeout before commitments. A faint `…` marks the cut (it
is `aria-hidden`; the tooltip and aria-label still list every group). Once no group is
left, the whole cluster and its marker go. Detail, then hint, then meter are dropped
after that, exactly as today. The main due text, the rank, and the current tier never
drop.

### Non-goals

- Clicking a single group to jump to that tier needs a new navigation API.
- Coloring the `]s` landing notices (bob-navigation-hotkeys) or the dashboard chips.
- Any change to counts, labels, `groupsText`, tooltip lines, aria-label text, the click
  action, the hide rules, or the paint key.

## Implementation

Most of the work is in the **bob-plugins** linked repo. Open it with your `/sase_repo`
skill (`sase repo open bob-plugins`) and read its `AGENTS.md` first. All paths below are
relative to that repo until the bob-cli section. The bob-ledger-tools plugin is a
fragment build: edit `plugins/bob-ledger-tools/src/` and `styles.css`, run
`npm run build`, and never hand-edit `main.js`. Each fragment must stay at or below 1000
lines; `src/120-freshness-footer.js` is already at 970.

### 1. Split the footer fragment (pure move)

- Create `plugins/bob-ledger-tools/src/121-freshness-footer-paint.js` and list it in
  `src/fragments.json` immediately after `120-freshness-footer.js`.
- Move these functions there verbatim from `120-freshness-footer.js`:
  - `freshnessFooterSetText`, `freshnessFooterSetAttr`, `freshnessFooterToggleClass`
  - `freshnessFooterCreateChild`, `freshnessFooterEnsureDom`,
    `freshnessFooterSetOmitted`
  - `freshnessFooterFit`, `freshnessFooterPaint`
- Fragments share one script scope, so the exports in `src/350-exports.js` do not
  change.
- Run `npm run build` and `node --test scripts/test-ledger-tools-freshness-footer.cjs`.
  Both must pass before any behavior change, so the move is reviewable on its own.

### 2. View model: mark the boundary (`120-freshness-footer.js`)

- In `freshnessFooterView`, after `groups` is filtered against the current tier, map
  each group to a new object `{ key, label, count, boundary }`. Exactly one group gets
  `boundary: true`: the first one whose key is not in
  `FRESHNESS_FOOTER_COMMITMENT_TIERS` (so ROTTEN or POST), and only if some earlier
  group in the same list is a commitment tier. Every other group gets `boundary: false`.
- Do not mutate or extend what `freshnessFooterGroups` returns; keep that contract.
- `groupsText`, the tooltip, `ariaLabel`, `mode`, and the hidden view stay
  byte-identical. The paint key in `240-plugin-freshness-status.js` needs no change:
  membership, order, boundary, and the current tier are all already implied by
  `groupsText` and `contextText`.

### 3. Painter (`121-freshness-footer-paint.js`)

- Add `freshnessFooterPaintGroups(container, groups)`. It never throws (cosmetic
  try/catch, like its siblings). Steps:
  1. Clear the container's children. Use `empty()` when available, else
     `textContent = ""`, mirroring `freshnessFooterEnsureDom`.
  2. For each group, create a `span.bob-freshness-group` with
     `data-review-tier="<key>"`, plus class `bob-freshness-group-boundary` when
     `group.boundary` is true.
  3. Inside each group span, create `span.bob-freshness-group-label` (text = label) and
     `span.bob-freshness-group-count` (text = `String(count)`).
  4. After the groups, append a `span.bob-freshness-groups-more` with text `…` and
     `aria-hidden="true"`, initially omitted.
  5. Remember the nodes on the container as `_bobFreshnessGroupNodes` (array) and
     `_bobFreshnessGroupsMore`.
- In `freshnessFooterPaint`:
  - Replace `freshnessFooterSetText(parts.groups, view.groupsText)` with
    `freshnessFooterPaintGroups(parts.groups, view.groups)`. In the hidden branch, paint
    `[]` instead of setting empty text. Never call `setText` on the groups container
    again, because it would wipe the children.
  - Set `data-review-tier="<tier>"` on `parts.context` when `view.current` exists and
    its `tier` is one of `FRESHNESS_FOOTER_TIERS`. Otherwise remove the attribute; this
    covers off-row, hidden, and legacy `references` contexts, which keep today's plain
    `.bob-freshness-tier` look. Keep the existing `bob-freshness-tier` class toggle.

### 4. Progressive fit (`121-freshness-footer-paint.js`)

Rewrite `freshnessFooterFit(el, parts, view)`:

- **State.** `nodes = parts.groups._bobFreshnessGroupNodes || []` and `shown` starts at
  `nodes.length` when `view.groupsText` is non-empty, else 0. `detail`, `hint`, and
  `meter` start as today.
- **`apply()`.**
  - The groups container is omitted when `shown === 0`.
  - Node `i` is omitted when `i >= shown`.
  - The `…` marker shows only when `0 < shown < nodes.length`.
  - Detail, hint, and meter are applied as today, through `freshnessFooterSetOmitted`.
- **Unknown width.** When the width is unknown (`clientWidth <= 0`), return with
  everything shown, as today.
- **Shedding order.**
  1. While the footer overflows and `shown > 0`, decrement `shown` and `apply()`.
  2. Then drop detail, hint, and meter in order, as today.
- **Fallback.** If `nodes` is empty but `groupsText` is set (a host that couldn't build
  the children), treat the container as one unit, exactly as today.
- **Return value.** Return
  `{ groups: shown > 0, groupsShown: shown, detail, hint, meter }`.
- The ResizeObserver refit in `240-plugin-freshness-status.js` already passes the
  painted parts and needs no change.

### 5. Stylesheet (`plugins/bob-ledger-tools/styles.css`, review footer section)

1. Rewrite the section comment so it states the ink rule. Each walk tier has its own
   theme-native hue, normalized to one OKLCH lightness per theme so every ink clears
   4.5:1 on the status bar. POST is the neutral text ink. Text, never color alone,
   carries meaning.
2. Add these rules. Keep this order: the POST override must come after the `@supports`
   block, because their specificity is equal.

   ```css
   .bob-freshness {
     --bob-review-ink-l: 0.52;
   }
   .theme-dark .bob-freshness {
     --bob-review-ink-l: 0.8;
   }

   .bob-freshness [data-review-tier="pre"] {
     --bob-review-hue: var(--color-cyan, #00bfbc);
   }
   .bob-freshness [data-review-tier="new"] {
     --bob-review-hue: var(--color-red, #e93147);
   }
   .bob-freshness [data-review-tier="projects"] {
     --bob-review-hue: var(--color-purple, #7852ee);
   }
   .bob-freshness [data-review-tier="pending"] {
     --bob-review-hue: var(--task-status-in-progress, var(--color-yellow, #e0ac00));
   }
   .bob-freshness [data-review-tier="next"] {
     --bob-review-hue: var(--task-status-next, var(--color-green, #08b94e));
   }
   .bob-freshness [data-review-tier="recurring"] {
     --bob-review-hue: var(--color-pink, #d53984);
   }
   .bob-freshness [data-review-tier="tickler"] {
     --bob-review-hue: var(--color-blue, #086ddd);
   }
   .bob-freshness [data-review-tier="rotten"] {
     --bob-review-hue: var(--color-orange, #ec7500);
   }

   .bob-freshness [data-review-tier] {
     --bob-review-ink: color-mix(
       in srgb,
       var(--bob-review-hue) 60%,
       var(--text-normal)
     );
     color: var(--bob-review-ink, var(--text-normal));
   }

   /* Older Electron installers lack relative color syntax; they keep the color-mix ink. */
   @supports (color: oklch(from red l c h)) {
     .bob-freshness [data-review-tier] {
       --bob-review-ink: oklch(
         from var(--bob-review-hue) var(--bob-review-ink-l) min(c, 0.16) h
       );
     }
   }

   .bob-freshness [data-review-tier="post"] {
     --bob-review-ink: var(--text-normal);
   }

   .bob-freshness-group-label {
     font-weight: var(--font-medium, 500);
   }
   .bob-freshness-group-count {
     font-weight: var(--font-bold, 700);
   }
   .bob-freshness-group-count::before {
     content: "\00a0";
   }
   .bob-freshness-group + .bob-freshness-group {
     margin-inline-start: 0.7em;
   }
   .bob-freshness-group.bob-freshness-group-boundary {
     margin-inline-start: 0.6em;
     padding-inline-start: 0.6em;
     border-inline-start: 1px solid
       var(--background-modifier-border-hover, var(--background-modifier-border));
   }
   .bob-freshness-groups-more {
     margin-inline-start: 0.5em;
     color: var(--text-faint, var(--text-muted));
   }
   .bob-freshness-main:hover {
     background-color: var(--background-modifier-hover);
     border-radius: var(--radius-s, 4px);
   }
   ```

3. Delete the `.bob-freshness-new .bob-freshness-main .bob-freshness-due` and
   `.bob-freshness-new .bob-freshness-tier` accent rules.
4. Keep `.bob-freshness-groups` as a plain inline span. Do not make it flex: a flex
   `gap` would also widen the space after the part's own `·` separator.

### 6. Tests (`scripts/test-ledger-tools-freshness-footer.cjs`)

The fake DOM already supports `createEl`, `empty`, `setAttribute`/`removeAttribute`
(`attrs`), and classes. Add or update these tests:

- **Update "footer paint hides the host and keeps a stable button".** Instead of
  checking the groups container's `textContent`, assert:
  - there is one `.bob-freshness-group` with `attrs["data-review-tier"] === "new"`;
  - its label text is `NEW` and its count text is `1`.
- **Groups render one tier-keyed token per nonempty group.**
  - Off-row with all nine tiers: tokens appear in walk order with labels
    `PRE NEW PROJECTS WIP NEXT RECUR TICKS ROTTEN POST`, and each carries its machine
    tier in `data-review-tier`.
  - Repainting with fewer groups leaves no stale or duplicate nodes.
  - `view.groupsText`, the tooltip, and `ariaLabel` are unchanged.
- **Boundary rules.**
  - Commitments followed by ROTTEN and POST: only ROTTEN gets the boundary.
  - Only ROTTEN and POST are present: no boundary.
  - On a ROTTEN row with commitments and POST: POST gets the boundary.
  - On a NEXT row where only ROTTEN follows: no boundary.
- **Context ink.**
  - On a queue row, the context carries the current tier's `data-review-tier`.
  - Off-row and in the hidden view, the attribute is absent.
  - A legacy `references` current entry (`legacyReferenceTier: true`) gets no attribute.
- **Progressive fit.**
  - Give a fake host a `scrollWidth` getter that sums fixed widths for every non-omitted
    part and group node.
  - Assert that groups shed tail-first, with `…` shown only while
    `0 < groupsShown < total`, then the whole cluster goes, then detail, hint, and meter
    in that order.
  - Assert the return value at each width.
  - When everything fits, nothing is omitted and the marker stays hidden.
- **The stylesheet covers every tier.**
  - Read `plugins/bob-ledger-tools/styles.css` and assert it contains a
    `[data-review-tier="<tier>"]` rule for each of the nine `FRESHNESS_FOOTER_TIERS`.
  - Assert the two removed NEW-mode accent selectors are gone.
  - This guards against drift between the JS tiers and the CSS.

### 7. Real-browser preview (`scripts/preview-review-footer.cjs`, dev-only)

- Follow the `scripts/preview-day-strip.cjs` pattern: production plugin source and
  `styles.css` loaded into headless Chromium, under an Obsidian baseline.
- The baseline must include `body.theme-light` / `body.theme-dark` variables from
  Obsidian's default theme:
  - **Light:** red `#e93147`, orange `#ec7500`, yellow `#e0ac00`, green `#08b94e`, cyan
    `#00bfbc`, blue `#086ddd`, purple `#7852ee`, pink `#d53984`.
  - **Light neutrals:** `--text-normal: #222222`, `--text-muted: #5c5c5c`,
    `--text-faint: #ababab`, `--background-secondary: #f6f6f6`.
  - **Dark:** red `#fb464c`, orange `#e9973f`, yellow `#e0de71`, green `#44cf6e`, cyan
    `#53dfdd`, blue `#027aff`, purple `#a882ff`, pink `#fa99cd`.
  - **Dark neutrals:** `--text-normal: #dadada`, `--text-muted: #b3b3b3`,
    `--text-faint: #666666`, `--background-secondary: #262626`.
  - **Status bar:** a status-bar host styled with `--background-secondary`.
- **Fixtures**, painted through `helpers.freshnessFooterView` and
  `helpers.freshnessFooterPaint`:
  - off-row with all nine groups;
  - a typical morning (NEW, WIP, NEXT, TICKS, ROTTEN, POST);
  - on a NEW row;
  - on a NEXT row with a quickRoll hint;
  - on a ROTTEN row.
- **Widths:** 1100, 700, and 460 px, each in both themes. Write one screenshot per theme
  and width, plus a summary.
- **Fail the run unless:**
  - every group and context ink has contrast ≥ 4.5:1 against the status-bar background,
    using `getComputedStyle` colors;
  - the nine group inks are pairwise distinct in each theme;
  - the footer never overflows its host;
  - the visible groups are always a walk-order prefix, with `…` present exactly when
    groups were cut.
- Keep it out of `npm test`, like the other preview scripts. Usage:
  `node scripts/preview-review-footer.cjs [--out DIRECTORY]`.
- If the full Playwright Chromium build crashes headless on this host, set
  `CHROMIUM_PATH` to the Playwright `chrome-headless-shell` binary in
  `~/.cache/ms-playwright`.
- Run it, look at the screenshots, and register the light and dark 1100 px shots as SASE
  artifacts with your `/sase_artifact` skill so the result can be compared with the
  design mock.

### 8. bob-plugins docs and inventory

- In `docs/bob-ledger-tools.md`, extend the review-footer paragraph (the one starting
  "The plugin shows a desktop review footer"). Add that:
  - each group and the current tier wear their own tier color;
  - a hairline divider separates the commitment groups from ROTTEN/POST;
  - narrow bars drop groups from the end of the walk, marked `…`, before detail, hint,
    and meter.
- In `plugins/bob-ledger-tools/manifest.json`, extend the footer sentence in
  `description` with one short clause about per-tier group colors. Then run
  `npm run docs:gen`.
- Run `npm run build`, `npm test`, `npm run docs:check`, then `bob plugins sync` to
  deploy to the vault.

### 9. bob-cli docs and memory (this repo)

- In `docs/freshness.md`, add one sentence to the paragraph that begins "Dashboard
  freshness.reviewModel NEW/ROTTEN chips". It should say that the desktop footer paints
  each group and the current tier in that tier's own color, marks the commitments/upkeep
  boundary with a hairline divider, and drops groups from the end of the walk (marked
  `…`) when space runs out. No guide step becomes incorrect, so leave
  `docs/guides/run-morning-review.md` alone.

> [!decision] glossary_footer_strand Use your `/sase_memory_write` skill and edit
> `sase/memory/glossary/morning-gtd-review-footer.md`. Replace the sentence that starts
> "To fit the status bar it uses short tier labels" with:
>
> "To fit the status bar it uses short tier labels — `WIP` for PENDING (WIP is an
> official alias of the Pending status), `RECUR` for RECURRING, `TICKS` for TICKLER —
> explained in its tooltip. Each group, and the current tier's context, wears that
> tier's own color; a hairline divider separates the commitment groups from ROTTEN and
> POST; and as space runs out it drops groups from the end of the walk (marking the cut
> with `…`), then detail, hint, and meter."
>
> Then run `sase memory init` and `just docs-gen`, which regenerates
> `docs/reference/glossary.md` from the glossary web.

> [!decision] glossary_footer_strand = no Leave the strand untouched. File a `memory`
> task bead through your `/sase_new_task` skill only when the decisions block says no
> human reviewed this plan.

- Run `just docs-check`. Do not edit `Cargo.toml`'s version or `CHANGELOG.md`; the
  release train owns them.

## Acceptance

- In Obsidian, both light and dark:
  - each nonempty group shows its tier's ink, with a lighter label and a bold count;
  - POST shows the normal text color;
  - the current tier context matches its group's ink;
  - the divider appears only between commitment groups and ROTTEN/POST;
  - hovering the main button shows the hover background;
  - `Review N due` no longer turns accent-colored while NEW tasks exist.
- Narrowing the window drops groups one at a time from the end of the walk with `…`
  before detail, hint, and meter drop; widening restores them.
- Tooltip, aria-label, `groupsText`, counts, click behavior, and hide rules are
  unchanged. The existing footer, namespace, and quickRoll tests pass unchanged except
  the updated paint assertion.
- `npm test`, `npm run docs:check`, the preview script's contrast and distinctness
  checks, and bob-cli's `just docs-check` all pass, and `bob plugins sync` has deployed
  the plugin.
