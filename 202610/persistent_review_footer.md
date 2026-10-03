---
tier: tale
title: A quiet, persistent review footer for Obsidian
goal:
  Show nonempty review groups and persistent task context in a compact native footer
  that disappears when no review tasks remain.
size: medium
proposed_by: bbugyi200.apollo.4w
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4w](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4w.md)
- **COMMITS:**
  - [0b7693b](https://github.com/bobs-org/bob-cli/commit/0b7693b3fb12774dee7491d277dfb12f596cc650)
    — docs(freshness): document the persistent review footer

# A quiet, persistent review footer for Obsidian

## Outcome

Replace Bob's long desktop review counter with one compact, live review component in
Obsidian's native bottom status bar. It appears whenever there is a trustworthy,
nonempty review queue, displays only nonempty review groups, and keeps a condensed
version of the `]s` notice visible while the cursor is on a review task. When no tasks
remain, remove the entire component from layout, including its progress meter and
shortcut hint. Keep the existing detailed navigation notices.

This is a `tale`, sized `medium`: one implementation agent can deliver the bounded
presentation, refresh, test, documentation, and deployment work. Planning this tale is
`large` work under the canonical SASE size guidance. There are no independent epic
phases.

## Context and source of truth

The user's screenshot is `~/tmp/screenshots/20261003_172431.png`. Its footer currently
shows
`3 new · 0 projects · 11 pending · 33 next · 8 references · 119 rotten · ✓ 0 today`.
Empty groups add noise, and useful navigation context disappears with the toast.

Implementation belongs primarily in the configured linked **bob-plugins** repo. From the
implementing agent's own workspace, use
`sase repo open bob-plugins -r "Implement persistent Obsidian review footer"`, use only
the returned path, and read its AGENTS.md. Edit plugin sources there, never deployed
plugin sources in the vault. That repo requires `bob plugins sync` after changes. The
primary **bob-cli** repo owns the authoritative freshness documentation.

Inspected implementation anchors in bob-plugins:

- `plugins/bob-ledger-tools/main.js`: `FRESHNESS_TIER_ORDER`, `freshnessQueue`,
  `freshnessCounts`, `freshnessStatusView`, `freshnessBuildMemo`, `freshnessEnsureMemo`,
  `setupFreshnessStatusBar`, `updateFreshnessStatusBar`, `freshnessStatusClicked`,
  `scheduleLiveWidgetRefresh`, and the existing CodeMirror update listener and unload
  cleanup.
- `plugins/bob-ledger-tools/styles.css`: `.bob-freshness` and its state classes.
- `plugins/bob-navigation-hotkeys/main.js`: `buildReviewJumpNotice`,
  `buildReviewBoundaryNotice`, `jumpToDueTask`, `landOnReviewQueueEntry`,
  `planReviewJump`, and `reviewAnchor`.
- `scripts/test-ledger-tools-freshness.cjs` and `scripts/test-navigation-freshness.cjs`:
  pure helper, plugin lifecycle, and navigation method harnesses. `package.json` exposes
  `npm test` and `npm run validate`.
- Each changed plugin's `manifest.json` and the linked repo's `README.md`.
- In bob-cli: `docs/freshness.md` §§4/6/7 and `docs/plugins.md`'s sync contract.

Before implementation, read applicable reference memory through `/sase_memory_read`,
including `obsidian.md`, `decisions:review-walk-is-tiered`, and
`decisions:ready-is-freshness-gated`. Read the implementation anchors afresh rather than
relying on the planning checkout's exact versions or line numbers.

## Product design

### Placement and visual hierarchy

Use the existing ledger-tools status-bar item as the single owner. Give that item the
available space on the leading side of the status bar; leave other plugins' document
metadata in their normal trailing area. Scope layout rules to Bob's own item, without
reparenting native elements or globally restyling the status bar. Keep one normal-height
row: this should feel native and avoid covering the note or creating a second toolbar.

The item contains a main review button, a short context phrase when applicable, a
passive ordered group breakdown, and the upkeep meter. Use the interface font, tabular
numerals, theme-native text/background/border variables, restrained spacing, and subtle
separators. Labels are muted, numbers are legible, and the current tier has one
understated accent treatment. Do not color the entire sentence or introduce a rainbow of
badges, animations, or a progress bar. The footer represents a changing queue, not a
fixed completion percentage.

Illustrative wide-layout states (counts are examples, not a vault census):

```text
Before landing on a review task:
⟳ Review 7 due · 4 commitments · ]s next   NEW 1 · PENDING 2 · RETURNED 1 · ROTTEN 3   ✓ 0/15 today

Cursor on the second queue entry:
⟳ Review 2/7 · PENDING 1/2 · confirmed yesterday   NEW 1 · PENDING 2 · RETURNED 1 · ROTTEN 3   ✓ 0/15 today

Only upkeep remains:
⟳ Review 3 due · Commitments done · ]s next   ROTTEN 3   ✓ 12/15 today

Upkeep budget met, with tasks still due:
⟳ Review 3 due · Upkeep budget met · ]s next   ROTTEN 3   ✓ 15/15 today
```

Use NEW → PROJECTS → PENDING → NEXT → RETURNED → REFERENCES → ROTTEN order. Display a
group only when its actual tier count is positive. Split RETURNED from ROTTEN in this
footer so its labels match the walk and make the commitment/upkeep boundary visible.
Existing dashboard ROTTEN chips retain their established folded RETURNED-plus-ROTTEN
meaning. Provide this distinction in the footer tooltip and docs.

### Persistent navigation context

When the active editor's cursor resolves unambiguously to a due queue task, show its
current overall rank/queue length, tier rank/tier total, and a condensed factual detail
from the same presentation model as the notice:

- NEW: the tier and ranks are sufficient.
- PROJECTS: `Empty project`, with never-confirmed/due context when room permits.
- PENDING and NEXT: never confirmed or confirmation age.
- RETURNED: the returned-since date.
- REFERENCES: `Reference`, with never-confirmed/due context when room permits.
- ROTTEN: due today or overdue age and interval.

The full detail and relevant existing keep/release/today shortcuts are available in the
accessible tooltip; they need not all fit on the row. Keep notices' wrap messages and
boundary preambles transient. Do not make a past `wrapped around` notice sticky.

Without a resolved current task, show the total due and commitment summary plus the
`]s next` hint. This includes startup, reading view, a non-Markdown leaf, and the cursor
leaving the review task. Do not preserve a misleading last-task fraction after the user
navigates elsewhere, and do not select or jump to a task just to populate the footer.

`Review r/N` and `TIER i/M` are positions in the current queue. They may change after an
edit or stamp; they never mean how many tasks were completed in a review session. Never
exclude a merely visited or anchored task from due totals.

### Responsive and accessible behavior

Fit to the width actually available to Bob's item, not only the screen width. On
decreasing width, omit the passive group strip first, then the optional detail, shortcut
hint, and upkeep meter as needed. Preserve the main due total or current rank and tier
as long as possible. Use a bounded flex layout and component-local width handling; if
using ResizeObserver, feature-detect it and clean it up. Avoid horizontal page
scrolling, wrapped status rows, split numerators/denominators, and overlap with native
metadata. Even in compact mode all nonempty group counts and details remain in the
focus/hover tooltip and accessible description.

Use a native button for the main summary. Pointer activation, Enter, and Space call the
existing `freshnessStatusClicked` behavior: next due task, with the `rotten` review-page
fallback if navigation is unavailable. Feature-detect the navigation command: only
promise `]s next` when it is available; otherwise label the action as opening the review
page. Group labels are passive; avoid nested controls or surprising group-specific
navigation. Provide a visible theme-native focus ring and accessible label with the full
summary. Refresh children without dropping button focus. Update politely only when the
summary changes; do not announce every cursor movement through an assertive live region.
Keep the existing desktop-only scope.

## Implementation

1. **Create shared, pure presentation helpers in ledger-tools.** Build an ordered
   positive-only group list, total, commitment count, optional current-entry context,
   upkeep meter, and full accessible tooltip from one memo's `counts` and `queue`.
   Prefer `byTier` over Ready-state totals; validate numbers and use the existing legacy
   fallbacks only where needed. Do not use the dashboard's visible-pool `reviewModel()`
   for this footer: hidden review-only references belong in the full walk. Derive
   visibility from a usable, warm snapshot with queue length > 0, never from `due`, a
   budget, refreshed-today totals, or an old notice string. Initialize the item hidden
   to avoid startup flashes. Hide it on unavailable data or an evaluation error, rather
   than showing stale numbers or an apparent all-clear.

2. **Share task wording with navigation through an additive API.** Extract the tier
   labels, detailed context, compact context, and lane action hint into a pure
   `freshnessReviewEntryView(entry, { todayText })` helper. Expose it as optional
   `api.freshness.reviewEntryView`; keep top-level api v3 and freshness namespace v5
   because this is additive. Navigation feature-detects this method and uses its
   presentation in `buildReviewJumpNotice` through its options, preserving notice ranks
   from the jump plan, wrap/boundary logic, and the current fallback for older
   ledger-tools versions. Never import another plugin's main.js at runtime. The helper
   formats already-evaluated entry fields; it does not reimplement the evaluator or
   mutate rows. Guard invalid input and exceptions. Existing notice text should remain
   equivalent; shortening applies to the footer's compact field.

3. **Resolve the current cursor against the live memo.** Match the active Markdown
   editor's path and zero-based cursor line to queue entries using the correct line
   convention and raw source verification. On moved lines, accept only a unique verified
   identity; duplicate text or block IDs must not borrow another task's rank. Reuse
   suitable existing exact-match helpers and memo indices. Show the generic summary when
   unresolved, just stamped, or edited ahead of the Tasks cache. Do not display a
   hypothetical next task before a successful landing. Use CodeMirror selection/document
   updates, active-leaf changes, and file-open events to schedule a lightweight context
   refresh. Cross-note jumps must update when the deferred cursor landing occurs, not
   merely when the file opens. No new navigation state machine or stored review session
   is needed.

4. **Render and refresh the component.** Replace the plain `setText` status renderer
   with stable owned child elements and visibility/width states. Enforce positive groups
   consistently in visible text, tooltip, and accessible description. The upkeep meter
   may correctly show zero while tasks remain; it is not a review group. An empty queue
   hides everything, including a met budget or nonzero today count, and consumes no
   margin/padding or keyboard focus. With tasks remaining, NEW and every commitment tier
   take precedence over budget-met styling; the optional ROTTEN backlog remains visible
   after the budget is met.

   Reuse the existing debounced live-refresh fan-out, Tasks cache generation,
   metadata/frontmatter refreshes, Today membership invalidation, config cache, and
   local-day rollover tick. Audit deletion/rename and Tasks availability recovery so a
   footer cannot remain stranded. Cursor-only updates must reuse a warm snapshot and
   avoid filesystem reads, config parsing, or O(all tasks) rebuilding on each keystroke.
   Add only the refresh hooks this feature needs. Register listeners and
   remove/disconnect timers, observers, and owned DOM on unload; reloading produces
   exactly one item.

5. **Document and release.** Update bob-plugins README's status-bar and optional API
   description and bob-cli `docs/freshness.md` with visibility, split footer tiers,
   queue-position meaning, and compact behavior. Bump each changed plugin's own manifest
   version according to the current repo convention and match README's version table. Do
   not change task writers, freshness cadence, tier order, dashboard chip contracts, CLI
   options, or memory files for this UI feature.

## Verification and acceptance

Add focused behavioral coverage to the existing freshness/navigation harnesses, plus a
small footer lifecycle/DOM harness if useful. Test actual display and interaction
contracts rather than snapshots that merely repeat implementation structure:

- Every individual tier at zero is absent, including `projects 0`; every positive tier
  appears exactly once in queue order. A tracker with NEW Ready state appears only in
  its actual PROJECTS/REFERENCES tier. RETURNED and ROTTEN are separate.
- An empty queue removes the entire item from layout and focus, even with nonzero upkeep
  or a met budget; 0 → 1 → 0 transitions need no reload. Unavailable, cold, or failed
  data never render false counts and recover when a valid snapshot returns.
- Ranks and compact details agree with current queue entries and the shared notice
  descriptor for all seven tiers. Keep legacy v3/v4 navigation fallback tests, as well
  as boundary, endpoint, wrap, and anchor tests.
- Same-note and cross-note jumps, deferred landing, cursor departure, duplicate/moved
  source lines, a stamp while cache lags, release, completion, deferral, Today links,
  delete/rename, and local-day rollover refresh or clear the appropriate context. No
  arbitrary cursor or footer action mutates a task or changes review membership.
- NEW/commitments cannot read as complete merely because a budget is met. Only ROTTEN
  remaining shows commitments done; a met budget with ROTTEN remaining keeps the footer
  visible.
- Pointer, Enter, and Space invoke next-due once, preserve the existing missing-nav
  fallback, and retain focus through count updates. Unload/reload leaves no duplicate
  item, listener, observer, or timer. Repeated cursor movement reuses the memo.

Run focused Node tests during implementation, then `npm test` and `npm run validate` in
bob-plugins once after the final code change. Use `git diff --check` in both affected
repos. No Rust behavior changes are planned; the primary repo's change should be
documentation only.

Inspect the actual footer in Obsidian if that runtime is available. Otherwise use a
small disposable browser fixture driven by the real footer renderer/styles and clearly
report that it is a fixture, not an Obsidian runtime check. Verify light and dark
themes, the screenshot's wide window, narrow available space alongside native metadata,
large counts, keyboard focus, empty state, and compact tooltip access. Tune
spacing/contrast based on the rendered result before deploying. Avoid adding browser
tooling dependencies solely for a mockup. Keep fixtures/screenshots outside the shipped
source unless they are necessary reusable tests.

Deploy only the changed plugins from the implementing agent's opened source checkout
using `bob plugins sync --no-pull --repo <opened-bob-plugins-path> --plugin <id>` for
each changed plugin. Run the matching `--dry-run` first and inspect it. The explicit
repo path is essential: the default plugin repo must not replace this implementation. Do
not automatically force past dirty vault-file refusals. Check that both changed plugins
actually report copied/up-to-date; sync can exit zero while skipping a dirty file.
Report any deployment limitation and whether a plugin reload is needed.

The coding agent's completion report should describe the final UI, tests run, visual
check performed, and deployment result. Do not claim runtime verification if only the
fixture was checked. Before ending normally, use `/sase_final` as required by SASE.
