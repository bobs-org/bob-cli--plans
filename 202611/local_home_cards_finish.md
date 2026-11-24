---
tier: tale
title: Finish local home cards and their nested landing
goal:
  Repair local card policy and lifecycle defects, prove the required Obsidian
  interactions, and close bob-cli-pl.4.1 and parent phase bob-cli-pl.4.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-pl.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pl.4.1
status: done
---

- **PARENT:**
  [202611/local_home_cards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/local_home_cards.md)
- **BEAD:**
  [bob-cli-pl.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pl/bob-cli-pl.4.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-pl.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-pl.4.1.land.md)
- **COMMITS:**
  - [45697ae](https://github.com/bobs-org/bob-plugins/commit/45697ae018dbeb60fd687890faccb781196c3a42)
    — fix(home-cards): finish local Area and Project card landing

# Finish local home cards and land bob-cli-pl.4.1

Repair the bounded model/adapter defects and complete the missing acceptance proof for
**bob-cli-pl.4.1**, then close that epic and its parent phase **bob-cli-pl.4**. This
tale is the completion of the interrupted landing. It has no land agent after its coder;
the coder must perform the closeout below in the same work sequence. Do not wait for
this work's eventual commit SHA, push, or CI before closing.

## Scope and governing decisions

Read bob-cli-pl.4.1 and its three children with `sase bead read ... -r "..."`, including
the landing audit and follow-up disposition notes. Read
`plan:202611/local_home_cards.md` and `plan:202611/area_project_cards.md` through
`sase artifact read`. Open bob-plugins through `/sase_repo`, read its `AGENTS.md`, and
use only the printed checkout. All product changes belong in that repository.

Keep the approved decisions: `card_structure_counts=no`, native Properties fully
visible, local-only painting, existing title/host/foot kit, no memory edits. No Rust
feature work, report-backed counts or reasons, goal-line styling, settings groups, `==`
registration, bank takeover, new YAML writer, or vault migration belongs here. Those
remain later phases of bob-cli-pl. Use the landed bare `-` drawer and `+` Property
Editor bindings. Settings/chips continue to open the existing browse API.

Read `decisions:off-clock-note-defaults-filter-by-task-type` and
`decisions:theme-hue-is-config-pin-exempt-area-hash` through `/sase_memory_read` before
repairing their adapters. Read `docs.md` before documentation changes.

## Audit evidence and drift already reviewed

The lander reviewed every note on the epic and its children and these bob-plugins
commits: `5717d518` (model/species/DOM), `b5f2f6c1` (host/foot), `21bbd30c`
(acceptance/docs), and `71cf808a` (acceptance assertion changes). The 82 tests in the
following six files pass at bob-plugins `71cf808a`:

- `scripts/test-ledger-tools-home-card-model.cjs`
- `scripts/test-ledger-tools-home-card-dom.cjs`
- `scripts/test-ledger-tools-home-card-host.cjs`
- `scripts/test-obsidian-home-cards-e2e.cjs`
- `scripts/test-ledger-tools-freshness-review-model.cjs`
- `scripts/test-ledger-tools-note-ready.cjs`

These passing tests miss the defects below. The host suite mostly tests helpers; its
successful-builder assertion does not prove actual mount failure restoration.

The sole intervening non-child bob-plugins commit is `b4023b85` (outer phase
bob-cli-pl.1): it retires project count/frontmatter writers and fixes test event fan-out
for home-card listeners. Preserve both changes. The cache-update failures reported
against `21bbd30c` are resolved by those test changes; do not file or repair them again.
Retired task counts remain inspectable/deletable in the managed drawer.

In bob-cli, post-start changes through `3deee354` add the home-settings contract
(`d39680dd`, `4b23bbef`, follow-up lint/macOS repairs), retired-count deletion
(`31131fe2`), and independent meeting/Gmail features. Settings value rules in
`src/native/props/contract/validate.rs` now provide useful parity evidence, but do not
call that contract during paint. Review newer drift at implementation start, especially
any sibling card-live/editor-settings commits, and preserve their work.

## 1. Correct the local model and adapter seams

Work in the existing `249b-plugin-home-card-model.js`, `249d-plugin-home-card-host.js`,
and `249e-plugin-home-card-foot.js` fragments, splitting if necessary to keep each
handwritten fragment at most 1000 lines. Rebuild generated entrypoints; never edit
`main.js` directly.

1. **Authored policy values must be truthful.** The lander's direct calls to the
   exported model reproduce these outputs:
   - `offclock_types: []` with `offclock: true` produces an **off-clock** chip keyed to
     `offclock` while its tooltip says that same value is ignored. The list's presence
     wins even when invalid; do not fall back to the boolean.
   - `offclock_types: [MEET, 7]` silently drops the invalid element and presents a
     valid-looking MEET policy.
   - `task_type: 42` becomes `#42`; `task_refresh: {bad: true}` becomes
     `review [object Object]`. Reuse the existing local semantic readers where
     appropriate, with value-shape checks matching the landed contract. Preserve absent,
     empty, false, and zero distinctions. Valid authored values retain the approved
     labels/order. Invalid or unsupported shapes stay inspectable/recoverable in the
     drawer and must not be represented as valid policy chips or claim a chip
     where-mark. Preserve the clock shadowing explanation without falsely painting the
     ignored policy. Do not migrate or rewrite any key while painting.
2. **Only a real project task determines lifecycle.**
   `homeCardPrjTask('- note\n  - [x] #task #prj example ^prj\n')` currently returns
   `kind: one`, mark `x`. Its 0–3-space regex accepts a nested example. Recognize
   top-level task context safely, preserve legal top-level indentation, and reject
   nested/example/fenced/ambiguous candidates. Retain exact source offsets and existing
   archived terminal-status fallback. Add a two-space nested regression plus valid
   top-level, fenced, and duplicate controls.
3. **Use the actual type hue API.** `homeCardThemeHue` calls nonexistent
   `this.taskThemeOf` and `nav.api.taskThemeOf`. The existing ledger API is
   `api.themes.hueFor(key, name)` / `pomodoroThemeHueFor`; its `taskThemeOf` member
   parses task text and is not a color resolver. Read the already-loaded hue through the
   existing API, preserve null/unconfigured colors, and never fetch or compute an
   independent color. Test configured color propagation and missing API behavior through
   the production host path. Ensure the top ancestor's crumb pip uses that ancestor's
   hue when a descendant has its own color override.
4. **Respect the inbox route authority.** `homeCardIsInboxNote('nested/inbox.md')`
   returns true even when the available `api.inboxRoute.isInboxNote` explicitly returns
   false. Remove the duplicate filename fallback; feature-detect the route API and honor
   its answer. Missing API is unknown/no badge. Cover true, false, absent, throwing API,
   and the actual canonical inbox via the real route API in acceptance.
5. **Exercise source and availability boundaries in the host.** Do not combine live
   source keys with stale cached frontmatter values as confirmed policy. Test a changed
   type/value before metadata catches up, pending source, lost title slot, delayed read
   after navigation/rename/delete, and unload/remount. Reuse the kit's source/identity
   safeguards and fix failures exposed by those tests. Use `noteReady` snapshot
   availability before displaying a count; the current `forNote` implementation returns
   null for unavailable Tasks, which is already correct for that case. Preserve unknown
   rather than inventing zero. Keep the five-key managed grouping adapter-specific and
   where-marks truthful.

Add behavioral regressions to the existing model/host suites; do not replace them with
source-string checks. Assert real removal/restoration on failed mount and unchanged
`^prj` offsets. Preserve shared kit behavior for other species.

## 2. Complete the existing real-Obsidian acceptance runner

Extend `scripts/obsidian-home-cards-e2e.cjs` and its library/wrapper. The present runner
covers ten notes, two modes, 24 expanded screenshots, one cursor edit, and Ctrl+Enter.
It does not exercise collapse, drawer, Settings/chips, modifier links, denied contexts,
failed mounts, or cleanup. `FIXTURE_TODAY` is declared but never pins the browser clock.
`71cf808a` removed both the Task Card and no-subprocess assertions rather than producing
the required evidence.

Keep the isolated synthetic Git vault, owned Obsidian process/profile/CDP target,
explicit binary/plugin provenance, short temporary paths, JSON report, and exception
capture. Follow the current validation runbook and read tailnet memory if the real app
must run on athena. Never use personal vault notes as fixtures.

- Pin time deterministically for the fixture's scheduled dates. Extend fixtures with
  archived terminal projects, a late H1 after `^prj`, deep ancestry, long summary,
  shadowed/invalid clock values, cap-off/unknown Ready, and missing/invalid created
  stamps. Keep the existing lifecycle/species matrix.
- Drive actual collapse and `-` drawer gestures in Live Preview and Reading; verify
  collapsed content, visible foot/lifecycle, independent drawer state, focus, split-pane
  sharing, and unique DOM IDs. Exercise Settings and authored chip handoff to the
  existing browse editor and missing-API inert rendering.
- Verify modifier-aware crumbs (tab, split, middle-click), Source transitions,
  eligibility changes, navigation, rename/delete, unload/remount, and injected
  mount/adoption failure with native title/H1 restoration. Exercise the denied contexts
  and print/mobile gates without letting them gain home-card chrome. Assert native
  Properties remains visible throughout.
- Before testing modals, assert `window.activeWindow === window` and
  `window.activeDocument === document`. The shared `trustCommunityPlugins` helper still
  returns immediately after the trust click; see bob-cli-ox and the existing prose-field
  runner's focus repair. Use that established remedy in this runner if needed, rather
  than interpreting invisible modals as a card regression. Restore Task Card open/close
  on `^prj`, verify cancel through existing close keys, and use a separate fixture for
  the existing task completion gesture. Preserve a card-disabled or untyped control and
  verify persisted source plus eventual active-editor content. Do not delete assertions
  to obtain green. If a residual independent defect survives the correct harness and the
  control, use `/sase_new_task` with the proposing bead bob-cli-pl.4.1.3 and retained
  reproduction; record that disposition in the epic note. Epic-caused failures must be
  repaired here.
- Measure note hashes and subprocesses around settled passive navigation, painting,
  folding, drawer and resize operations. Attribute unrelated existing services by actual
  provenance/call sites. Retain the trace and assert zero home-card-triggered
  subprocesses and writes. A comment or direct model-only spy is not a substitute for
  this real-app assertion.
- Capture and inspect light/dark, wide and genuinely narrow card containers (below the
  approximately 420px breakpoint), active/terminal, long summary, collapsed and
  open-drawer states, and reduced motion. Record actual container geometry and visual
  verdicts, not just screenshot existence or viewport size. Retain the report and
  inspected screenshots through `/sase_artifact`, with canonical refs in the epic's
  close note. The prior child notes link no retained real-app proof that establishes
  these missing cases.

Add the missing runner instructions to bob-plugins `docs/develop/validation.md` and keep
`docs/area-and-project-cards.md` accurate. Do not document later parent features as
shipped.

## 3. Verify, deploy, and finish this landing

Use `/sase_monitor` before long suites or real-app runs, with an explicit continuation
that finishes this tale and both closeouts. Finish staging the monitor before ending a
provider turn; never wait for a staged successor.

Run focused home-card and shared identity/title/host/foot regressions, the two
cache-update suites above, and the runner's offline checks. Then `npm run build`,
`npm test`, `npm run validate`, and
`BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check`. Run `just check` in bob-cli
for file-change verification; **do not run `just check-full`**. Preserve known-failure
evidence and isolated controls for bob-cli-km/io rather than weakening their assertions.
Resolve newly caused failures before closeout. Run `bob plugins sync` from the opened
plugin checkout and follow its documented overwrite-refusal procedure without `-F`.
Verify the intended build is synced. Neither deployment nor closeout may be made
conditional on this work's own future commit SHA, push, or CI result.

All original follow-ups have already been triaged by the lander and recorded on
bob-cli-pl.4.1. Carry these outcomes into the final close note:

| Proposal                             | Recorded disposition                                                                                                                                                                    |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| pl.4.1.1 #2 docs mount collision     | Corroborated bob-cli-it; later phase docs gate passed. No new task.                                                                                                                     |
| pl.4.1.1 #3 protected deployment     | Resolved by mount-foot's landed-commit sync and acceptance's 34 unchanged/0 skipped sync. No task.                                                                                      |
| pl.4.1.1 #4 timing flakes            | Corroborated bob-cli-km and bob-cli-io with attributed phase evidence.                                                                                                                  |
| pl.4.1.3 #2 cache-update failures    | Resolved by b4023b85; lander's combined focused suite passed 82/82. Its alleged clean base already contained the epic. No task.                                                         |
| pl.4.1.3 #1 Task Card/editor refresh | New task declined pending valid focused-window controls; diagnostic recorded on bob-cli-ox without claiming confirmed causality. Restore acceptance here and record the actual outcome. |

The final step is the concrete epic and parent closeout, before `/sase_final`:

1. Re-read bob-cli-pl.4.1, all descendants/notes and its linked plan; confirm all
   assigned work and the repairs above are complete. If this proposal is represented by
   a separate descendant tale bead, close that completed descendant normally before
   closing its ancestor. Do not leave an open descendant for the finalizer to resolve
   after the epic close.
2. Run `sase bead epic-symbols bob-cli-pl.4.1`. The lander's scan was empty, but repeat
   it. Resolve every listed symbol (wire up, privatize, supported non-test pragma, or
   delete); re-key only if a named still-open later bead genuinely requires it. Do not
   leave stale exemptions.
3. Run
   `sase bead close bob-cli-pl.4.1 --note "<source/drift verification, test and real-app artifact evidence, deployment result, and all follow-up outcomes>"`.
   Retry only after resolving any rejected symbols/readiness issues. Never use `--force`
   to make a successful nested landing advance. Unfinished successful phases must be
   completed, not force-closed.
4. Run `just symvision` if the recipe is available (it is absent in the reviewed bob-cli
   and bob-plugins trees); record absence if still unavailable. Open the plans sidecar
   through `/sase_repo`, resolve `plan:202611/local_home_cards.md`, and set
   `status: done` in that epic file's frontmatter. Its relative PLAN path is
   `202611/local_home_cards.md` under the printed plans root.
5. Run `sase bead read bob-cli-pl.4.1 -r "Need the parent link"`. Its parent is the
   phase **bob-cli-pl.4**. Re-read that phase and verify this child completed the
   original home-cards acceptance in `plan:202611/area_project_cards.md`, including
   preserved native Properties and project-task interaction. Record actual host/foot
   entry points (`homePropertyCardHost`, `homeCardModelFor`, `homeCardOpenSetting`,
   `homeCardFootSpec`, refresh/collapse methods), current `-`/`+` key bindings, and
   artifact evidence for later phases. Run `sase bead epic-symbols bob-cli-pl.4`,
   resolve/re-key any entries, then close **only that parent phase** with
   `sase bead close bob-cli-pl.4 --note "<verified original phase acceptance and API/keymap handoff>"`.
   Run symvision if available. Leave **bob-cli-pl** and its plan status to its waiting
   lander.
6. Finish through `/sase_final`, declaring every repository changed in this tale. Do not
   defer closeout until the host commits the final implementation.
