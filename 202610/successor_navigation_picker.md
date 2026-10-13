---
tier: tale
title: Successor offer picker and review-walk integration
goal:
  Obsidian offers explicit successor selection and advances the review walk once after
  the picker settles.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7c.5
bead: bob-cli-7c.5
create_time: 2026-10-13 10:48:56
status: wip
---

- **PARENT:**
  [202610/successor_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/successor_offers.md)
- **BEAD:**
  [bob-cli-7c.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/bob-cli-7c.5.md)

# Successor offer picker and review-walk integration

Implement the `nav_picker` phase of **bob-cli-7c.5**, within epic **bob-cli-7c**. A
managed Done gesture with successor offers should open one grouped chooser; only the
chosen keys go back to the cycler. A close from a review landing must wait for that
chooser and its completion result, then advance once with one composed toast, including
when the chooser is dismissed.

This is one bounded coding-agent task: a navigation modal, its additive API,
gesture-lock integration, notice copy, and focused tests. The phase has no previous
implementation or remaining-work notes. No child epic is needed.

## Binding context and ownership

- Re-read `sase bead read bob-cli-7c.5 -r "Need current phase scope and notes"` before
  implementation. Do not hand-set its already assigned status.
- Read `plan:202610/successor_offers.md` through `sase artifact read`. Its shared design
  and `nav_picker` section are binding. The accepted design research is
  `research:202610/successor_pull_in_offers/successor_pull_in_offers.md`; the epic
  overrides the report where they differ.
- The implemented contract is bob-cli `docs/task-dependencies.md` §12.13; §12.6 governs
  notices and SO21, SO22, SO25, and SO32 are key UI vectors.
- Use `/sase_repo` to run
  `sase repo open bob-plugins -r "Implement successor navigation picker for bob-cli-7c.5"`.
  Work only in that returned checkout and read its `AGENTS.md`. Do not edit deployed
  vault plugin files.
- Read relevant reference memory through `/sase_memory_read`: the walk decisions
  `answering-advances-the-walk` and `review-walk-is-tiered`, glossary `Task Link` and
  `Pomodoro`, `obsidian.md` before deployment, and `sase_beads.md` before bead
  operations.
- The epic's two memory decisions are final. Memory work belongs to `bob-cli-7c.8`; this
  tale makes **no memory edits** and re-asks no decision.
- Cycler planning, re-planning, recovery, selection writes, receipts, and engine notice
  composition belong to `bob-cli-7c.4`. Build against the versioned picker model with
  mocks even if that phase is not merged yet. The destination override UI belongs to
  `bob-cli-7c.6`: keep the return type extensible with optional `destination`, but add
  no session chooser or Ctrl+Shift+Enter hint in this phase.

## Source map and baseline

All paths below are relative to the opened **bob-plugins** checkout:

- `plugins/bob-navigation-hotkeys/src/200-filtered-picker-modals.js`:
  `FilteredPickerModal`, its filtering/keyboard/render lifecycle, and the separate
  `pickerOpen` flag required by modern Obsidian Modal.
- `400-picker-local-task.js` and `450-picker-keydown.js` in the same directory: existing
  stable-key marks, Tab toggle-and-advance, and footer.
- `480-review-jump-and-nav-api.js`: `createDependencyNavApi`, frozen top-level api v3;
  `490-plugin-lifecycle.js`: install/unload and tracked modals.
- `535-plugin-review-checklist-walk.js`: `claimReviewWalkCtrlEnter` and
  `completeReviewChecklistRow`, including returned `successorNotice`.
- `536-plugin-review-advance.js`: capture/continue, sequence/epoch guards,
  `reviewWalkBusy`, three-second lock and 350 ms settling, shared advance.
- `285-unblocked-notice.js`: pure notice model, fallback text, count pill, rendering,
  and `createNoticeApi`.
- `src/fragments.json`, `src/690-install-plugin-mixins.js`, `src/700-exports.js`, plugin
  `styles.css`, `manifest.json`, root README, and root `package.json` complete the
  build/test/version wiring.

Planning baseline: plugin commit `76924cc`, nav version `2.32.0`, clean checkout. The
five test files listed in verification below pass (114 tests); `npm run build:check` and
`npm run validate` pass. Re-check the actual implementation checkout rather than pinning
to these incidental versions. `sase bead epic-symbols bob-cli-7c.5` currently reports no
entries.

Several existing fragments are nearly full: `200` is 935 lines, and `480` and `536` are
each 966. Put substantial new code in new ordered fragments, after the base modal for
subclasses and before the relevant API/mixin installation. Keep every hand-edited
fragment at most 1000 lines. Generate `main.js` with the build; never edit it by hand.

## Implementation

1. **Build the successor chooser.** Add a `FilteredPickerModal` subclass and small
   helpers in new source fragments. Consume the §12.13 model directly: `title`,
   `predecessors`, `groups`, rows with opaque `key`, `text`, `notePath`, `status`,
   `destinationLabel`, `needsBlockId`, `selectable`, `mutedReason`, folded counts, and
   optional budget. Keep identity and marks in a Set of the supplied keys; never
   regenerate keys or use line numbers. The modal only presents choices and returns
   them; it performs no vault reads, writes, planning, or block-ID minting. Safely
   resolve `null` for unusable input or zero selectable offers.

   Show Hand-offs, Project ‹note› with its supplied project-task header, and Project
   finished groups. Headers and count lines are never choices. Render muted unavailable
   rows with reasons, destinations, note names, and `needs ^id` indicators. Display
   folded hidden/already-planned counts and the epic's copy for sub-projects staying
   blocked until reconcile.

   Provide a text filter matching row text and note/group context. When the model
   contains more than 12 task rows, project groups initially show a short preview and a
   `+N more` control. Provide a keyboard and mouse expansion path; any non-empty filter
   expands matching groups. Clearing the filter restores the user's expansion state.
   Keep every offer reachable; marks survive filtering/collapse and no list is capped.

   Start with zero marks. Up/down moves the highlight; Tab toggles a selectable row's
   mark and moves forward, following the existing convention. Ctrl+A marks all
   selectable rows in the highlighted group, including collapsed rows; it excludes muted
   rows and other groups. Plain Enter returns all marked rows in model order, or the
   highlighted selectable row when no marks exist. A muted row/header never yields a
   pick. Esc and outside dismissal return `null`. The footer always shows the
   prospective `pull in N` count. Optional `plan L/max → L'/max` uses that same
   prospective selection and warns when over, without refusal.

   Consume ignored Enter events: reject modifiers (Ctrl/Meta/Alt/Shift), repeats, and
   presses within 350 ms of opening. Use a controllable clock for deterministic tests.
   Do not let a second Ctrl+Enter apply the chooser or reach an underlying completion
   handler. Add scoped styles using the existing picker/theme conventions and
   distinguish highlight from marks.

2. **Expose and settle the API.** Add frozen `successors: { version: 1, pick(model) }`
   to `createDependencyNavApi`. `pick` always returns a non-rejecting Promise resolving
   `{ picks: [{ key, destination? }] }` or `null`. Track exactly one active successor
   modal on the plugin. A concurrent second call resolves `null` without touching the
   first modal, its marks, or its walk hold. Settle a modal exactly once; close it and
   clear tracking before resolving so focus can return before the walk continues.
   Failure to construct/open/render must also settle and clean up safely.

   Initialize tracking on load. On unload, close and settle any pending request even if
   Obsidian teardown throws; replace the api with its null-plugin variant. Calls through
   both the replacement api and a retained old api reference after unload must resolve
   `null`. Keep root api version 3 and `notice`/`reviewWalk` contracts compatible.

3. **Keep the review gesture alive.** Add an explicitly owned hold for a pending review
   completion that opens the successor modal. Associate ownership with the active
   gesture/landing, including sequence and epoch where used. `reviewWalkBusy()` must
   stay true throughout the chooser even after the numeric three-second timeout. Guard
   against duplicate Ctrl+Enter and manual `]s` / `[s` staling the origin while
   choosing. Do not change the origin sequence or landing epoch just for opening or
   closing the modal. Avoid timer polling or an arbitrary larger timeout.

   Cover both normal cycler capture → `reviewWalk.continue` and nav-owned
   `claimReviewWalkCtrlEnter` → `completeReviewChecklistRow` → awaited cycler
   `completeTaskAtCursor`. The checklist path does not create the ordinary captured
   origin, so it needs explicit ownership too. Keep the owner protected through result
   application and continuation; dismissal must not create an unlocked gap before the
   committed close advances. Release only the matching owner's hold in its
   completion/error/finally path and on unload. Preserve ordinary timeout recovery
   outside this flow and the existing 350 ms post-answer settle window.

   The close remains committed when the chooser returns `null`; consume and advance its
   landing once when the completion result arrives. Keep the existing stale-origin,
   failed/no-op write, and PRE/POST boundary rules, Vim jump recording, and answer
   journaling. Append the cycler's returned successor line to the existing single walk
   toast; neither the chooser nor the advance creates a separate successor notice.

4. **Update notices and release metadata.** Add `not_selected` → `not pulled in` and
   `no_destination` → `no open Pomodoro`; preserve `project_member` → `project member`.
   Update fallback/summary and count pills to communicate pulled-in counts, unselected
   offers, and mixed results while retaining destination information. Retain legacy
   breaker and `not_planned_today` copy only for older payloads. Preserve task-row click
   behavior, minted-ID footer, plan-meter warning, and release hint. Bump nav's manifest
   version from the current checkout and update its README row and manifest description
   to explain explicit choice. Leave other plugins' versions and descriptions to their
   own phases.

## Verification and completion

Add a registered test file such as `scripts/test-navigation-successor-picker.cjs`, using
`navigation-hotkeys-harness.cjs` / `modal-harness.cjs` where practical. Exercise real
modal lifecycle and API settlement, not just helper outputs:

- 1, 6, 11, and 20 selectable offers start unmarked; supplied opaque keys round-trip;
  Tab subset and Ctrl+A group picks exclude muted/header rows.
- Filter/collapse preserves marks, makes every offer reachable, handles empty matches,
  and submits marks even when currently filtered out.
- Group/folded rendering, destination/ID labels, footer count, optional budget and
  over-budget submission behave as specified.
- Plain highlighted Enter (SO25), marked Enter, all modifier variants, key repeat, and
  deterministic 349/350 ms guard boundary (SO22).
- Escape, outside close, repeat close, unload, construction/open failure, concurrent
  requests, and retained API calls settle safely once.

Extend `test-navigation-review-advance.cjs`,
`test-navigation-review-advance-gestures.cjs`, and/or
`test-navigation-checklist-ctrl-enter.cjs` with a deferred chooser and controlled clock.
For normal and checklist paths, advance time beyond 3000 ms while open, try a duplicate
completion and walk jump, verify no origin change/advance/write, then choose a subset or
dismiss. Assert exactly one final landing/jump record and one toast carrying
`successorNotice`, PRE/POST boundaries, hold release on errors/unload, and rejection of
stale or double continuations (SO21). Test against the actual nav picker API with a
stubbed cycler await so this phase does not depend on engine rollout. Extend
`test-navigation-hotkeys-unblocked-notice.cjs` for current reasons, none/subset/all
summaries, mixed destinations, and legacy payloads.

Run these focused tests after rebuilding:

```sh
npm run build
node --test scripts/test-navigation-successor-picker.cjs scripts/test-navigation-review-advance.cjs scripts/test-navigation-review-advance-gestures.cjs scripts/test-navigation-checklist-ctrl-enter.cjs scripts/test-navigation-hotkeys-unblocked-notice.cjs scripts/test-navigation-hotkeys-picker-lifecycle.cjs
npm test
npm run validate
git diff --check
```

Register every new test file in the root `package.json` test script. Run bob-cli's
required `just check` from its own checkout as appropriate for final phase verification.
For a long check use `/sase_monitor`, wait for the handoff command to exit, and carry
the explicit completion scope into its successor. Do not end a turn with a live tool
session.

Deploy the built navigation plugin from the **opened** checkout:

```sh
bob plugins sync --repo "$successor_plugins_checkout" --no-pull --plugin bob-navigation-hotkeys
bob plugins sync --repo "$successor_plugins_checkout" --no-pull --plugin bob-navigation-hotkeys --dry-run
```

Set `successor_plugins_checkout` to the path returned by `sase repo open`, not a default
clone or a path copied from another agent. Verify the real sync succeeds and the dry-run
shows no remaining drift; inspect any refusal rather than blindly adding `--force`.

Record verification and any out-of-scope discoveries on **bob-cli-7c.5**. Do not create
task beads; use `sase bead note bob-cli-7c.5 'PROPOSED FOLLOW-UP: <summary — detail>'`.
If a check fails identically on the clean base, record exact reproduction and any
existing tracking bead in that format and still finish this phase. Before closing, run
`sase bead epic-symbols bob-cli-7c.5`; resolve remaining entries or re-key them to the
still-open parent epic/later phase. The authorized phase worker closes **only
bob-cli-7c.5** with `sase bead close bob-cli-7c.5 --note "<verified results>"`. Leave
**bob-cli-7c** and every ancestor plan to their land agents; a successor assigned a
different child bead supplies evidence rather than closing its ancestor. Use
`/sase_final` to declare the changed linked repository for host commit. The generated
entrypoint, sources, styles, metadata, and tests all belong in that repository's commit.
