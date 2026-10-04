---
tier: epic
title: Split the five largest bob-plugins JavaScript files into files of at most 1000
  lines
goal: 'The five largest JavaScript files in the bob-plugins linked repo are each split
  into multiple hand-edited files of at most 1000 lines. Plugin runtime behavior,
  the `helpers` test surface, `bob plugins sync`, and the full `npm test` / `npm run
  validate` suite stay unchanged.

  '
phases:
- id: split-task-status-cycler
  title: Split task-status-cycler main.js and establish the plugin source build
  depends_on: []
  size: large
  description: 'split-task-status-cycler: pilot the src/ fragment build, staleness
    check, and parity check on the smallest target plugin. Document the new contract
    and split plugins/task-status-cycler/main.js into fragments plus plugin-class
    mixins.'
- id: split-ledger-tools
  title: Split bob-ledger-tools main.js
  depends_on:
  - split-task-status-cycler
  size: large
  description: 'split-ledger-tools: apply the established build contract to plugins/bob-ledger-tools/main.js.
    Preserve its load-time CodeMirror let/try setup and widget classes, and repoint
    keep-in-sync comments in other plugins.'
- id: split-navigation-hotkeys
  title: Split bob-navigation-hotkeys main.js
  depends_on:
  - split-ledger-tools
  size: large
  description: 'split-navigation-hotkeys: apply the build contract to the 52k-line
    plugins/bob-navigation-hotkeys/main.js. This includes mixin splits of both the
    plugin class and the 6.4k-line BulletPropertyPickerModal, with its super calls
    and duplicate onClose.'
- id: split-navigation-hotkeys-tests
  title: Split scripts/test-navigation-hotkeys.cjs
  depends_on:
  - split-navigation-hotkeys
  size: large
  description: 'split-navigation-hotkeys-tests: move the shared preamble and file-wide
    helpers into a harness module, and split the 500 tests into per-area test files
    listed in package.json. The same 500 names must still pass.'
- id: split-task-status-cycler-tests
  title: Split scripts/test-task-status-cycler.cjs
  depends_on:
  - split-navigation-hotkeys-tests
  size: large
  description: 'split-task-status-cycler-tests: reuse the harness convention to split
    the 179 cycler tests into per-area files. Keep the stub-assigned MarkdownView/TestModal
    classes shared through the harness.'
proposed_by: bbugyi200.apollo.4z
create_time: 2026-10-04 07:13:44
status: wip
bead_id: bob-cli-47
---

- **PROMPT:** [prompts/202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:** [bob-cli-47](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-47/README.md)

# Plan: Split the five largest bob-plugins JavaScript files

## Background

The user wants the five largest JavaScript files in the `bob-plugins` linked repo, the
source of the Bob Obsidian plugins, split so that no hand-edited file exceeds 1000
lines. The epic has one `large` phase per file, and the phases run strictly in sequence.
This plan recommends an approach for each file. **Each phase agent owns the final
split** and is expected to adapt it to the file as it looks when that agent runs.

### Targets

Measured with `git ls-files '*.js' '*.cjs' '*.mjs' | xargs wc -l` at bob-plugins commit
`f4b3562`. All line numbers in this plan come from that commit and will drift, so
re-measure before planning.

| Rank | File                                     | Lines  | Phase                            |
| ---- | ---------------------------------------- | ------ | -------------------------------- |
| 1    | `plugins/bob-navigation-hotkeys/main.js` | 51,891 | `split-navigation-hotkeys`       |
| 2    | `plugins/bob-ledger-tools/main.js`       | 20,404 | `split-ledger-tools`             |
| 3    | `scripts/test-navigation-hotkeys.cjs`    | 19,813 | `split-navigation-hotkeys-tests` |
| 4    | `plugins/task-status-cycler/main.js`     | 12,051 | `split-task-status-cycler`       |
| 5    | `scripts/test-task-status-cycler.cjs`    | 6,987  | `split-task-status-cycler-tests` |

The sixth-largest file, `plugins/block-id-prompt/main.js` (6,880 lines), is out of
scope. `bob-vim-surround` and `bob-project-tasks` are out of scope too.

### Why this order

The plugin files go first, smallest to largest:

1. `task-status-cycler` pilots the build contract. It is the least risky plugin: it has
   no top-level `let` and no `super` calls in its plugin class.
2. `bob-ledger-tools` hardens the contract. It has load-time `let`/`try` state.
3. The contract is then applied to the 52k-line navigation file.

The two test files go last. They need no build step, and they share a harness
convention: the navigation test split establishes it and the cycler test split reuses
it.

## Rules for every phase

1. **Open the repo properly.** Open bob-plugins with the `/sase_repo` skill
   (`sase repo open bob-plugins -r "<reason>"`), read its `AGENTS.md`, and make every
   edit in the checkout path it prints. Commit there; it becomes a repository obligation
   in `/sase_final`.
2. **You own the final split.** These phases are `large`, so you plan first. Re-measure
   your file and re-check the hazards listed for it. Treat the recommended groupings as
   a starting point: change boundaries, names, directory layout, or file count whenever
   the code calls for it. The build contract set by `split-task-status-cycler` binds
   later phases. If you must change it, update the docs and every already-split plugin
   in the same phase.
3. **Move code, don't rewrite it.** Function, method, and test bodies move
   byte-for-byte: no renames, reformatting, logic changes, or cleanups. The only new
   code allowed is glue the split requires: mixin installers, harness exports, and
   build/check scripts. Record any bug you find as a `PROPOSED FOLLOW-UP:` note on your
   phase bead instead of fixing it.
4. **Script the extraction.** Cut files from line ranges with a throwaway script instead
   of hand-copying, so moved text is exact and the split can be re-run. If the file
   changed on origin while you worked, re-run the script on the new file instead of
   hand-merging.
5. **Respect the line limit.** Every hand-edited file you create or touch must be at
   most 1000 lines (`wc -l`). Only a generated `main.js` is exempt. Aim for cohesive
   files of roughly 300–900 lines, cut along feature boundaries rather than at arbitrary
   line counts.
6. **Preserve order.** Keep the bundle in the original top-to-bottom order unless there
   is a concrete reason not to. Class declarations, `const`s, and `let`s are not
   hoisted, and contiguous slices make parity easy to prove.
7. **Definition of done:**
   - `npm test` and `npm run validate` pass, and no existing test is removed or renamed.
   - The build staleness check passes.
   - Plugin phases also pass the parity check.
   - The README describes the new layout.
   - Follow the repo's convention for `manifest.json` version bumps (check `git log`); a
     behavior-neutral refactor is at most a patch bump.
   - Plugin phases deploy with `bob plugins sync -p <plugin-id>`, as bob-plugins
     `AGENTS.md` requires. If sync refuses because vault plugin files are dirty, stop
     and report; never pass `--force`.

## Recommended build contract

The `split-task-status-cycler` phase decides this contract.

### Constraints

- **Obsidian loads one file.** Obsidian evaluates exactly one `main.js` per plugin and
  does not resolve relative `require`s against the plugin folder (and mobile has no Node
  `require`). Split source therefore has to be assembled back into `main.js`.
- **Sync copies fixed files.** bob-cli's `bob plugins sync` copies and byte-compares
  only `manifest.json`, `main.js`, and `styles.css`.
- **Tests and scripts load `main.js` directly:**
  - About 40 test scripts call `require("../plugins/<id>/main.js")` under `Module._load`
    stubs for `obsidian` and `@codemirror/*`.
  - `scripts/migrate-dependency-lines.mjs` and a spawned subprocess in
    `scripts/test-navigation-task-card-schedule.cjs` both require the navigation
    `main.js`.
  - `scripts/test-navigation-task-card-view.cjs` regex-matches the navigation `main.js`
    source text.
- **Validation assumptions.** `scripts/validate-manifests.mjs` treats every directory
  directly under `plugins/` as a plugin and runs `node --check` on its `main.js`.
- **Documented policy.** The README currently says there is intentionally no bundler or
  build step and that `main.js` is the source. This epic deliberately changes that for
  the three split plugins. The other three plugins keep a hand-edited `main.js`, so the
  docs must describe both kinds.

### Recommendation

- **Source in `plugins/<id>/src/`; `main.js` becomes a generated, committed artifact.**
  Committing it keeps several things unchanged: `bob plugins sync`, the vault deploy
  path, every test, and the migration script. No bob-cli change is needed.
- **Use shared-scope ordered concatenation rather than CommonJS modules.** Add a
  zero-dependency `scripts/build-plugins.mjs` that uses Node built-ins only
  (`package.json` is "not a bundler").
  - It concatenates an explicit, ordered fragment list into `main.js`. The list could
    be, e.g., `plugins/<id>/src/fragments.json`, or numbered filenames.
  - The output starts with a
    `GENERATED from src/ by scripts/build-plugins.mjs — do not edit` header and has a
    `// ---- src/<file> ----` banner before each fragment.

  The reasons not to use modules:
  - Each file is a single scope with hundreds of cross-referencing top-level functions:
    768 in navigation and 343 in ledger-tools.
  - ledger-tools reassigns top-level `let` state across what would become module
    boundaries.
  - The `helpers` export names nearly every function.

  Concatenation preserves semantics exactly, whereas real modules would mean thousands
  of import/export edits plus CommonJS cycle hazards. The cost is that fragments use
  identifiers from other fragments implicitly and cannot be required on their own. If
  the pilot agent finds a compelling reason to prefer real modules plus a tiny bundler,
  it may choose that and must document why.

- **Fragments are valid standalone scripts.** They contain top-level declarations only,
  so the build or validate step can run `node --check` on each one.
- **Split plugin classes with prototype mixins.**
  - Keep `onload`/`onunload` in a core `class <Name> extends Plugin { ... }`.
  - Move other method groups into plain mixin classes, e.g.
    `class TaskStatusCyclerVimMethods { ... }`. Using class bodies lets methods move
    verbatim, with no commas to add.
  - A small installer copies each group onto the core prototype using
    `Object.getOwnPropertyDescriptors(Mixin.prototype)` minus `constructor`. It must
    throw on a duplicate method name.
  - The final fragment does
    `module.exports = <Name>; module.exports.helpers = { ... };`.

  Mixins are safe for all three plugin classes. None of them has class fields,
  getters/setters, statics, `#private` members, or `super.` calls. Tests use
  `new <Plugin>()` and `Object.create(<Plugin>.prototype)`, and both keep working.

  A method that calls `super.` can only move into a mixin class that `extends` the same
  base as the target class, so `super` resolves identically. Otherwise keep it in the
  core class body.

- **Installer placement.** Keep the installer per plugin, either duplicated (about 15
  lines) or injected by the build from `scripts/`. Do not add a shared directory under
  `plugins/`, because validate-manifests would treat it as a plugin.
- **Checks.**
  - Add `npm run build`.
  - Add a `--check` mode that fails if any committed `main.js` differs from build
    output, or if any fragment exceeds 1000 lines.
  - Wire `--check` into `npm test` or `npm run validate`, so a direct edit to a
    generated `main.js` fails fast.
- **Parity check.** Commit it, e.g. as `scripts/check-split-parity.mjs`, so later phases
  can reuse it.
  - It loads two versions under the same stubs the tests use: the pre-split `main.js`,
    from `git show <base>:plugins/<id>/main.js` written to a temp file, and the new one.
  - It asserts identical `Object.keys(helpers)`.
  - It asserts identical `String(value)` for every helper function and every unsplit
    class.
  - For each split class (the plugin class, and `BulletPropertyPickerModal` later), it
    asserts the same own prototype method names and identical `String(method)` for each
    method.

  Passing proves code moved without edits. Explain any intentional difference in the
  commit message; there should be close to none.

- **Docs.**
  - Rewrite the README "Layout" and "Development model" sections to describe the
    src/-built versus hand-edited plugins and the build/check commands.
  - Add the rule "edit `src/`, run `npm run build`, never edit a generated `main.js`" to
    bob-plugins `AGENTS.md`. Its `CLAUDE.md` is just `@AGENTS.md`.
  - Check bob-cli `docs/plugins.md`, and update it only if it claims `main.js` is
    hand-edited source.

## Phase split-task-status-cycler: task-status-cycler main.js (12,051 lines)

Build the contract above, then split the file into a recommended 18–20 contiguous
fragments.

### Top level (1–5878)

There are no banner comments, so boundaries are top-level declarations. Constants are
scattered through the region; move each one with its cluster.

- Requires, core constants, and indent/date/fresh-stamp/Vim-repeat helpers (1–372).
- Scheduled-date parsing, list items, Schedule Log and Work Log, and blocked retirement
  (373–953).
- Status checks and Obsidian task promote/demote/toggle (954–1300).
- Markdown sections and section-move plans (1301–2221, about 920 lines; split it if
  needed).
- Demotion picker model, list block range, and toggle document plan (2222–2854).
- Link/transclusion parsing and dependency-line detection (2855–3048). This is small, so
  merge it with a neighbor if that reads well.
- Pomodoro (3049–3952, about 900 lines).
- Block-link targets, fences, and strikethrough (3953–4350).
- Dependency parsing, blocked-dependent recovery, and retire/restore references
  (4351–4860).
- Source-text line ops, dependency-ID normalize/rename, block cache, and checkbox/bullet
  formatting (4861–5500).
- Editor view, scroll, defer, and UI helpers, plus `DemotionSectionPickerModal`
  (5501–6162). The modal's `super(app)` is the only `super` in the file. The modal must
  precede the helpers export.

### Plugin class `TaskStatusCyclerPlugin` (6163–11866)

Split it into a core class plus about 8 mixins:

- Core: `onload` and `onunload` (6164–6315).
- Dependency-ID normalize, ambiguity, rename reconcile, and `processVaultFileText`
  (6316–6872).
- Reference mutation queue, `recoverBlockedDependentsNow`, and strike/unstrike
  (6873–7504).
- Vim mappings and navigation, rendered-query scroll, and open-line handlers
  (7505–8307).
- Command handlers, physical-key listeners, and `cycleTaskStatusRange` (8308–8977).
- Open/done toggles and transcluded close/reopen/cycle (8978–9496).
- Pomodoro Work Log writes and complete/reopen (9497–10184).
- Checkbox toggles, demotion glue, set status, bullet format, and editor line edits
  (10185–11049, about 865 lines).
- Transcluded target resolve/replace, freshness stamp, and small utilities
  (11050–11866).

### Helpers export and checks

`module.exports.helpers` (11867–12051, 183 names) becomes the final fragment.

`scripts/test-task-status-cycler.cjs` constructs `new TaskStatusCyclerPlugin()` about
100 times with `Plugin` stubbed as an empty class, so keep that working. Record the
contract choices you made in the commit message and README, because the next two phases
follow them.

## Phase split-ledger-tools: bob-ledger-tools main.js (20,404 lines)

Follow the established contract. The recommendation is about 28–32 fragments.

### Top level (1–9372)

- **1–134 must stay first and in this order.** It holds:
  - the requires
  - the optional CodeMirror `let` and `try` blocks
  - `ensureNoteReadyRefresh` and `ensureDependencyChipsRefresh`
  - the constants
- Time, daily paths, and the Pomodoro ledger (136–923).
- Snippets (924–1101) and editor/view/daily-open/scroll helpers (1102–1889). Together
  these are about 965 lines, so consider separate files.
- Plan budget (1890–2672).
- Today (2673–3075).
- READY backlog, dashboard lane badges, and review chips (3076–3836).
- Freshness placement (3837–4529) and freshness config (4530–4867).
- Freshness evaluation (4868–5940, about 1,070 lines). Split it, e.g. evaluate versus
  queue/counts.
- Review entry views and the footer view/DOM (5941–6819).
- `freshnessReviewModel` and the freshness mark core (6820–7774).
- Pure dependency-chip code (7775–8834, about 1,060 lines; split it).
- `DependencyChipWidget` and `FreshnessMarkWidget`, which are `let`s assigned inside
  top-level `if` blocks that need `WidgetType`, plus `freshnessRowFromTask` and the memo
  code (8835–9372).

### Plugin class `BobLedgerToolsPlugin` (9373–18371, about 9,000 lines)

Split it into a core class plus about 11 mixins:

- Core lifecycle (9374–10030). `onload` builds `this.api`.
- Plan/Today accessors and READY badges (10031–10850).
- Note-ready snapshot, API, and crowded chips (10851–11834).
- Dashboard collections (11835–12409).
- Ready-notes block (12410–12842).
- Note-ready heading chips and review chips (12843–13681).
- Freshness config, memo, and `apiFreshness*` (13682–14590).
- Freshness status bar and footer (14591–14926).
- Freshness marks (14927–16067). Split near `freshnessMarkShouldRebuild`, around
  line 15412.
- Dependency chips (16068–17350). Split at `renderDependencyChipsIn`, around line 16523.
- Today cache, plan block render, and daily location memory (17351–18073).
- Vim mappings, Pomodoro jump/edit, and snippet expansion (18074–18371).

### After the class

- The top-level helpers after the class (18373–20190) become about 3 fragments:
  - plan lane counter and plan config, which includes the reassigned `planCapsStatCache`
    `let`
  - note-ready cap model
  - dashboard-collections, note-ready-view, and plan-block models
- `module.exports.helpers` (20191–20404, 212 names including 10 constants) becomes the
  final fragment.

### Hazards

- **Load-time side effects.** Exactly one eager `StateEffect.define()` must run at load.
  Two tests depend on the current load-time behavior:
  - `scripts/test-ledger-tools-freshness-mark-surfaces.cjs` keeps the last-defined
    effect.
  - `scripts/test-ledger-tools-dependency-chips.cjs` captures the widget class through
    the stubbed `WidgetType`.

  So no fragment may add or reorder load-time side effects.

- **Nested class.** `HeadingWidget = class extends WidgetType` is nested inside
  `buildNoteReadyHeadingDecorations`. Leave it there.
- **Marker comments.** The unreferenced `// __FRESHNESS_*_END__` marker comments may
  stay or go.
- **Copied logic in other plugins.** block-id-prompt, task-status-cycler, and
  navigation-hotkeys keep hand-made copies of some ledger-tools logic, marked with
  comments like "keep in sync with bob-ledger-tools/main.js". Repoint those comments to
  the new fragment paths, but do not change the copies.

## Phase split-navigation-hotkeys: bob-navigation-hotkeys main.js (51,891 lines)

This is the largest job; expect about 65–75 fragments. Script it end to end.

Strongly consider subdirectories under `src/` by domain, e.g. `core/`, `dependencies/`,
`pomodoro/`, `project-notes/`, `task-card/`, `modals/`, and `plugin/`. The ordered
fragment list stays the single source of order. If you judge the work too big for one
pass, your plan may itself be an epic, e.g. separate phases for the top level,
`BulletPropertyPickerModal`, and the plugin class.

### Top level (1–35029)

The region has no top-level `let` and no IIFEs. The only load-time `const` initializers
that call functions are three `buildManagedTaskLogParentRe` regexes (around lines
363–380). They work because function declarations hoist.

Recommended clusters:

- Requires and constants (1–398). Keep these first and in order.
- Numeric utils and bullet-property config (399–1143).
- Task identity and dependency navigation bullets (1144–1855).
- Schedule Log and Cancel Log (1856–2346).
- Priority roll planning (2347–3349, about 1,000 lines; split it).
- Decay planner and card rows (3350–4378; split it).
- `FreshnessDecayCardModal` (4379–4578).
- Recommended roll batch and hand-edit mirror (4579–5389).
- Depends-On identity, planner, and writer (5390–6625; split it). `planDependencyEdit`
  alone is about 490 lines.
- Vim repeat, jump history, vault paths, and rename (6626–7475).
- Project-note create/reverse/promote (7476–9766; about 3 files).
- Dates, templates, fences, task statuses, and scheduled recovery (9767–10986; about 2
  files).
- Pomodoro ledger (10987–13310; about 3 files).
- Open-task navigation, editor primitives, and transactions (13311–14682; about 2
  files).
- Transclusion toggling and picker hints (14683–15424).
- Project/area info and the summary modals (15425–16043).
- `FilteredPickerModal`, its six small subclasses, and `RenameCurrentFileModal`
  (16044–17010).
- Line contexts, dependency snapshots, and counted targets (17011–17990).
- Link picker (17991–18357).
- Lane/schedule Work Logs, schedule review, and batches (18358–19554; split it).
- Task move (19555–20455).
- Tags, project schedules, and recovery (20456–21263).
- Counted batch planners (21264–22085).
- Date math and notice models (22086–23127; split it).
- Typed schedule parser (23128–23586).
- Vault-wide Depends-on stage (23587–24908; split it).
- Task Card model and view (24909–26876; about 2–3 files).
- Keydown helpers and freshness review (33296–35029; about 2 files).

### `BulletPropertyPickerModal` (26877–33295)

This class is about 6,400 lines with about 190 methods, and it extends
`FilteredPickerModal`. It needs its own mixin split. Suggested groups:

- Task Card chrome (26878–27612).
- Value and schedule-review stages (27613–28834; split it).
- Cancel and lane-release reason stages (28835–29216).
- Roll and recommendation (29217–29938).
- Local-task and block-id stages (29939–30837).
- Dependency commit and vault batch (30838–32870; about 2–3 files).
- Keydown and apply (32871–33295).

Hazards:

- **`super` calls.** The class calls `super.onOpen`, `onClose`, `renderAll`,
  `getFilteredItems`, `renderResults`, and `handleKeydown`. Either keep those methods in
  the core class body, or have their mixin classes `extends FilteredPickerModal` so
  `super` resolves identically, and verify either way.
- **Duplicate `onClose`.** The class defines `onClose` twice, around lines 27527
  and 29982. Today the later definition wins, so the first is dead code; it is the one
  that removes the `bob-task-card-*` classes. Preserve current behavior: the live
  `onClose` must stay the effective one. The installer's duplicate check will flag this,
  so either drop the dead copy or explicitly allow it. Record a `PROPOSED FOLLOW-UP:`
  note about the latent bug.
- **Class order.** Classes are not hoisted. `FilteredPickerModal` must precede all of
  its subclasses, and every modal must precede the plugin class and the helpers export.

### Plugin class `BobNavigationHotkeysPlugin` (35030–51315, about 16,300 lines)

Split it into a core class plus about 20 mixins. The class has no fields, getters,
statics, or `super` calls. `onload` is about 361 lines and makes 38 `addCommand` calls.

Clusters:

- `onload` and `onunload`.
- Transclusion toggles and the deferred Pomodoro snapshot.
- Link picker.
- Lane toggle and refresh interval.
- Freshness queue, review jump, and refresh stamp.
- Decay card and walk.
- Lane/cancel from the picker or editor.
- Bullet-property writes (39837–42618; about 3 files).
- Dependency edit, stage, and hand-edit mirror (42619–44724; about 2–3 files).
- Section/task jumps, counted-key listeners, and Vim mode.
- Dash, leaves, and tab pin.
- Vim mappings and the jump-history bridge.
- Leaf/tab management and dash location.
- Parent/child/template notes, yank path, and line-link candidates.
- Task move and Pomodoro move/split/merge sessions (47923–49222; about 2 files).
- Project note create/convert and rewrites (49223–50315; about 2 files).
- Templater, link open/create, and delete/rename.
- File-position tracking and link parsing.

### Helpers export and other consumers

`module.exports.helpers` (51316–51891, about 573 names) becomes the final fragment.

Other consumers of this file:

- **Source-text test.** `scripts/test-navigation-task-card-view.cjs` regex-matches the
  generated `main.js` text, e.g. the `set-bullet-property` `addCommand` literal. It
  keeps passing against the bundle; repointing it at the owning fragment is optional.
- **Runtime requires.** `scripts/migrate-dependency-lines.mjs` and the subprocess in
  `scripts/test-navigation-task-card-schedule.cjs` require `main.js` and need no change.
- **Dynamic require.** The dynamic `require("os")` / `require("fs")` inside
  `requireOptionalNodeModule` stays as is.

## Phase split-navigation-hotkeys-tests: scripts/test-navigation-hotkeys.cjs (19,813 lines, 500 tests)

No build step is involved. The recommendation is one shared harness module plus about
22–26 `scripts/test-navigation-hotkeys-<area>.cjs` files.

Replace the old entry in `package.json`'s explicit `npm test` file list with the new
files. Keep the explicit-list style unless you have a reason to switch to a glob.

### Harness module

Name it without the `test-` prefix, like `scripts/modal-harness.cjs`, e.g.
`scripts/navigation-hotkeys-harness.cjs`. Split it into two files if it would pass 1000
lines. It should own:

- **The preamble (L1–235):**
  - the `global.window` default and `notices`
  - `parseTestYaml`
  - the `Module._load` stubs for `obsidian` and `@codemirror/view`
  - the plugin `require` and `helpers`
  - `TestEditor`, `TransactionEditor`, and `RecordingFallbackEditor`
  - `compatibleTasksSettings` and `assertLineBoundedTransaction`
- **The file-wide picker harnesses (L3957–4246),** including
  `createBulletPropertyPickerHarness`, `createPriorityPickerConfig`, and
  `runCardAction`.
- **The fragment-node builders** (about L4438–4575).
- **The Pomodoro fixtures (L10335–10405).** Some tests use them before their definition
  today.
- **Shared cluster helpers.** Any cluster-local helper that your file boundaries end up
  sharing, e.g. `buildScheduleReasonConfig`, `createLinkPickerHarness`, or
  `parseFrontmatterFromContent`.

The unused `spawnSync` and `os` imports can be dropped.

### Test clusters

The tests fall into 26 themes, each contiguous in the file. Six exceed 1000 lines and
need a further cut: 461–1920, 2473–3754, 4247–5383, 5384–6563, 8836–9955, and
17341–18544.

- Section and open-task navigation (66–460).
- Schedule validation/recovery and property targets (461–1920).
- Runtime transclusion and counted scheduling (1921–2472).
- Dependencies (2473–3754).
- Task-move planning (3755–3956).
- Priority notice, picker, and roll (4247–5383).
- Picker lifecycle and Pomodoro move pickers (5384–6563).
- Task-move insertion (6564–7299).
- Runtime task move (7300–7496).
- Open-task jump, Pomodoro reorder dispatch, and Vim counts (7497–8496).
- Tab pin and close (8497–8835).
- Priority-roll reason, Schedule Log, and schedule review (8836–9955).
- Deferred Pomodoro link cleanup (9956–10334).
- Pomodoro parse, rows, and notices (10335–11171).
- `planPomodoroBulletMove` (11172–12113).
- Bullet split, rename, and merge (12114–13072).
- `planPomodoroEntryReorder` (13073–13874).
- Runtime pruning (13875–14434).
- Project-from-task (14435–15388).
- Project-note reversal (15389–16358).
- Link picker (16359–16819).
- Lane toggle (16820–17340).
- Cancel task (17341–18544).
- Forward promotion (18545–19191).
- Scheduling Work Log (19192–19503).
- Task-move Vim jump history (19504–19813).

The task-move tests are scattered across 3755, 5463–5537, 6503–7078, 7300, and 19504.
Gathering them into one file is fine, because the tests do not depend on order.

### Hazards

- **Process isolation.** Under `node --test`, each file runs in its own process. That is
  safe here: every test resets `notices` before reading it, and every test restores the
  globals it swaps (`global.document`, `global.window`, and
  `FilteredPickerModal.prototype.renderResults`).
- **Relative path.** The styles test reads
  `__dirname + "/../plugins/bob-navigation-hotkeys/styles.css"`, so keep the split files
  in `scripts/`.
- **Clock.** The runtime-prune and cancel tests use the real clock. Do not freeze or
  mock time.

### Verification

- Test names stay unchanged and unique.
- Test bodies move verbatim.
- Compare before and after, e.g. by diffing spec-reporter or TAP name lists, and confirm
  the same 500 test names pass.

## Phase split-task-status-cycler-tests: scripts/test-task-status-cycler.cjs (6,987 lines, 179 tests)

Reuse the harness convention from the previous phase. The recommendation is
`scripts/task-status-cycler-harness.cjs` plus about 8–10
`scripts/test-task-status-cycler-<area>.cjs` files, with `package.json` updated.

### Harness module

It should own:

- **The preamble (L1–175):**
  - the `Module._load` stub
  - `createTestDomNode` and its module-level `focusedEl`
  - the `try`/`finally` plugin `require`
  - `notices`
- **The file-wide helpers (L357–478):** `createInMemoryObsidianApp`, `createTextEditor`,
  `attachActiveMarkdownView`, `registerTaskToggleVimAction`, `flushAsyncActions`, and
  `getEmbeddedTarget`.

The demotion and promotion helpers (L4455–4549) only serve L4550–6041. Keep them in a
helper file local to the demotion/promotion tests, or in the harness.

### Hazard

The `obsidian` stub assigns `MarkdownView` and `TestModal` to module-level `let`s as a
side effect. Export them from the harness after the plugin `require` completes, or
through getters. That way the split files see the same classes that the `instanceof` and
`getActiveViewOfType` checks rely on.

### Test clusters

The tests fall into 14 contiguous themes:

- Dependency normalizer, rename, and blocked recovery (177–584).
- Ctrl+Enter selection, Pomodoro bullet toggle, and daily paths (585–1129).
- Recursive completion (1130–1368).
- Pomodoro markers and completion plan (1369–2084).
- Retirement coordinator (2085–2580).
- Pomodoro Work Log (2581–2977).
- Done Pomodoro reopen (2978–3642).
- Vim Ctrl+Enter and embedded transclusion (3643–3965).
- Blocked ring and Schedule Log retirement (3966–4454).
- Demotion picker (4455–5161).
- Promotion (5162–6050).
- Plain Task Links and the recovery API (6051–6463).
- Cycler freshness (6464–6686).
- Depends-On lines (6687–6987).

Several of these fit together in one file. The demotion and promotion span (4455–6050,
about 1,600 lines) needs at least two files.

### Verification

Confirm that the same 179 test names pass before and after the split.
