---
tier: tale
title: Split bob-ledger-tools onto the plugin source build
goal:
  Complete bob-cli-47.2 by moving bob-ledger-tools into ordered source fragments under
  the existing build contract, with load-time behavior and export parity preserved and
  the generated plugin deployed.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-47.2
bead: bob-cli-47.2
create_time: 2026-10-04 08:00:15
status: wip
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:** bob-cli-47.2

# Split bob-ledger-tools onto the plugin source build

Implement the reserved phase **bob-cli-47.2**, `split-ledger-tools`, from
`plan:202610/split_largest_bob_plugins_js_files.md`. Apply the source-build contract
that bob-cli-47.1 already established. Do not invent a second build system. This is
bounded, scripted work for one implementation agent, so use a medium tale rather than
further phase beads.

## Scope and evidence

The primary project is bob-cli. Implementation belongs in the **bob-plugins linked
repository**. Run
`sase repo open bob-plugins -r "Implement bob-cli-47.2 bob-ledger-tools source split"`
and edit only the checkout path it prints. Read that repo's `AGENTS.md` before editing.
It already states the fragment contract: edit `src/`, run `npm run build`, never
hand-edit a generated `main.js`, then deploy with `bob plugins sync`.

Inspected linked HEAD was `6f8aca0`. `plugins/bob-ledger-tools/main.js` is unchanged
since the epic's measurement commit `f4b3562`
(`git diff --stat f4b3562 HEAD -- plugins/bob-ledger-tools` was empty). Re-measure
before extraction if the checkout has moved, and re-cut from the new file instead of
hand-merging.

Measured on that file with Acorn (`ecmaVersion: latest`):

- 20,404 lines (`wc -l`), 212 `module.exports.helpers` keys, 191 own plugin methods, all
  `MethodDefinition` of kind `method`, all unique.
- The plugin is `module.exports = class BobLedgerToolsPlugin extends Plugin` at line
  9373, closed at line 18371. There is no separate class declaration.
- The plugin class has no fields, getters, setters, statics, private names, or `super`.
  The only `super(` calls are inside `WidgetType` class expressions:
  `DependencyChipWidget` (8839), `FreshnessMarkWidget` (9022), and the nested
  `HeadingWidget` inside `buildNoteReadyHeadingDecorations` (13216).
- Top-level `let`s that the load path depends on are the CodeMirror slots in lines 1–103
  (`ViewPlugin`, `Decoration`, `WidgetType`, `StateEffect`, `RangeSetBuilder`,
  `editorInfoField`, `editorLivePreviewField`, `syntaxTree`, `freshnessMarksRefresh`,
  `noteReadyRefresh`, `dependencyChipsRefresh`) plus `DependencyChipWidget` (8835),
  `FreshnessMarkWidget` (9018), and `planCapsStatCache` (18591, reassigned in
  `resetPlanCapsCache` and `loadPlanCaps`).
- Exactly one eager `StateEffect.define()` runs, inside the `try` at lines 58–64.
  `ensureNoteReadyRefresh` and `ensureDependencyChipsRefresh` define their effects
  lazily.

`scripts/build-plugins.mjs`, `src/fragments.json` ordering, the 1000-line fragment
limit, `npm run build`, and `npm run build:check` already exist and already opt in every
plugin that has `src/fragments.json`. task-status-cycler is the only opted-in plugin
today. Do not rewrite the builder.

The current parity tool cannot load this plugin. On identical source,
`node scripts/check-split-parity.mjs --plugin bob-ledger-tools --base 6f8aca0` fails
because `@codemirror/state` is not stubbed and the two load errors embed different temp
paths. Hardening that stub is part of this phase. Do not change fragment concatenation,
banner format, or the cycler fragment layout.

Out of scope: behavior changes, test renames or deletions, edits to
`plugins/bob-navigation-hotkeys/main.js` (bob-cli-47.3 is already in progress on that
file), hand-edits to task-status-cycler's generated `main.js`, splitting
block-id-prompt, bob-cli runtime code, and memory edits. Record any bug you notice as a
`PROPOSED FOLLOW-UP:` note instead of fixing it.

## 1. Record the base and script the cut

From the opened checkout, record `git rev-parse HEAD` as the parity base while the tree
is still clean. Keep a byte copy of `plugins/bob-ledger-tools/main.js` from
`git show <base>:plugins/bob-ledger-tools/main.js`.

Write a throwaway extractor under `/tmp`, not in the repo. It must slice the recorded
source by exact line ranges, preserve bytes and line endings, and assert the anchor
declaration at each boundary before writing. Account for every original line exactly
once, except line 9373 and line 18371, which the glue below replaces. If upstream moves
the file, re-run the script on the new bytes and refresh the parity base. Do not
hand-copy method bodies.

Run the existing `npm test` and `npm run validate` once on the clean tree before
editing, and keep the result. Use `/sase_monitor` for a run that may outlast the turn. A
failure that reproduces identically on this clean base does not block the phase: record
it with
`sase bead note bob-cli-47.2 'PROPOSED FOLLOW-UP: <summary — identical base evidence>'`
and continue. Fix failures the split introduces.

## 2. Make parity able to load ledger-tools

Extend `scripts/check-split-parity.mjs` only enough to load this plugin under the same
stubs on both sides. Keep the public CLI (`--plugin`, `--base`, repeatable
`--split-helper`) and the cycler comparison rules.

In available mode, stub at least:

- `obsidian`: the existing `MarkdownView`, `Notice`, `Plugin`, and `setIcon`, plus
  harmless `normalizePath` and `parseYaml` values. `Plugin` must stay a constructor
  because the plugin class `extends` it.
- `@codemirror/state`: `Prec` and `StateEffect.define`. `define` must record one stub
  call per invocation and return a distinct effect type, so a second eager
  `StateEffect.define()` fails parity. Include `RangeSetBuilder`.
- `@codemirror/view`: `EditorView`, `keymap`, `ViewPlugin`, `Decoration`, and a
  constructable `WidgetType`. The unconditional
  `const { EditorView, keymap } = require("@codemirror/view")` and the later `try` must
  both succeed, and the `WidgetType` class expressions must be evaluated.
- `@codemirror/language`: a `syntaxTree` value so that `try` succeeds.

In missing-CodeMirror mode, keep throwing `MODULE_NOT_FOUND` with
`error.request === "@codemirror/view"` for `@codemirror/view` only. Still stub
`@codemirror/state` in that mode. Ledger requires state before view, so an unstubbed
state require never reaches the view special case. Do not compare absolute paths inside
load errors. The existing "both sides missing `@codemirror/view`" branch, which compares
load calls rather than error message text, is the right behavior once the throw happens
on that request.

Leave task-status-cycler's require sequence unchanged: it requires `obsidian` and
`@codemirror/view` only. Adding unused stub fields must not change its load-call list.

Checkpoint, before extraction, from the opened repo:

```bash
node scripts/check-split-parity.mjs --plugin bob-ledger-tools --base <recorded-base>
node scripts/check-split-parity.mjs --plugin task-status-cycler --base 6f8aca0^
```

The first must pass on the still-unsplit ledger file (212 helpers, 191 own prototype
methods). The second must still pass for the cycler split. If `6f8aca0^` is not the
cycler pre-split commit because history moved, use the parent of the cycler split commit
that still contains the hand-edited cycler `main.js`.

`scripts/check-split-parity.mjs` stays under 1000 lines. Do not add a shared directory
under `plugins/`.

## 3. Extract thirty-five ordered fragments

Create `plugins/bob-ledger-tools/src/fragments.json` and the fragments below. Numbers
are manifest order. Ranges are inclusive original lines on `6f8aca0` and include the
comments and blank lines that belong to that group. Snap a boundary to the blank line
between statements if a re-measure shifts it, but do not reorder groups or split a
statement, a widget `if` block, or a method.

Top-level slices are copied byte-for-byte. Each stays a valid standalone script under
`node --check` and at or below 1000 lines.

| Fragment                     |     Lines | Anchor                                                                                                                                                         |
| ---------------------------- | --------: | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `010-load-and-constants.js`  |     1–134 | Requires, CodeMirror `let`/`try`, both `ensure*` functions, constants through `LEGACY_STOPWATCH_DURATION_GLOBAL_RE`. This slice stays first and in this order. |
| `020-time-and-pomodoro.js`   |   135–923 | `parseEmDashTrigger` through `resolveEditPomodoroTarget`                                                                                                       |
| `030-snippets.js`            |  924–1101 | `computeRange` through `expandLineAtCursor`                                                                                                                    |
| `040-editor-and-daily.js`    | 1102–1889 | `sameEditorPosition` through `replaceEditorLine`                                                                                                               |
| `050-plan-budget.js`         | 1890–2672 | Plan-budget comment through `computePlanBudget`                                                                                                                |
| `060-today.js`               | 2673–3075 | Today comment through `laneBudgetFromTasks`                                                                                                                    |
| `070-ready-and-review.js`    | 3076–3836 | READY comment through `paintReviewElement`                                                                                                                     |
| `080-freshness-placement.js` | 3837–4529 | Placement comment through `readFreshness`                                                                                                                      |
| `090-freshness-config.js`    | 4530–4867 | Config comment through `loadFreshnessConfig`                                                                                                                   |
| `100-freshness-evaluate.js`  | 4868–5527 | Evaluation comment through `freshnessComparePathLine`                                                                                                          |
| `110-freshness-queue.js`     | 5528–5929 | Queue comment through `freshnessStatusView`                                                                                                                    |
| `120-freshness-footer.js`    | 5930–6817 | `FRESHNESS_FOOTER_TIERS` through `freshnessFooterPaint`                                                                                                        |
| `130-freshness-marks.js`     | 6818–7769 | `freshnessReviewUnavailable` through `buildFreshnessMarkElement`                                                                                               |
| `140-dependency-model.js`    | 7770–8619 | `parseDependencyLine` through `dependencyChipModel`                                                                                                            |
| `150-dependency-element.js`  | 8620–8830 | `buildDependencyChipElement`                                                                                                                                   |
| `160-widgets-and-row.js`     | 8831–9372 | Both widget `let`/`if` blocks, `freshnessRowFromTask`, `freshnessMemoReviewChanged`, and the `__FRESHNESS_*_END__` markers                                     |

`160-widgets-and-row.js` keeps `DependencyChipWidget` and `FreshnessMarkWidget` as the
original `let` plus `if (WidgetType)` class expressions. Do not convert them to mixins.
Their `super()` calls stay inside those expressions. Do not add, remove, or reorder a
load-time `require` or `StateEffect.define()`.

Plugin methods move byte-for-byte, indentation included, into a core class and thirteen
plain mixin classes. The core fragment replaces line 9373 with
`class BobLedgerToolsPlugin extends Plugin {` and contains only the original `onload`
and `onunload` bodies. Do not put `module.exports =` on that class. Each mixin fragment
is `class <Name> {` plus the original method lines plus `}`. No mixin `extends`
anything. The original class-closing `};` on line 18371 is not copied.

| Fragment                              |       Lines | Mixin class                                        |
| ------------------------------------- | ----------: | -------------------------------------------------- |
| `170-plugin-lifecycle.js`             |  9374–10030 | core `BobLedgerToolsPlugin` (`onload`, `onunload`) |
| `180-plugin-plan-and-ready.js`        | 10031–10841 | `BobLedgerToolsPlanAndReadyMixin`                  |
| `190-plugin-note-ready.js`            | 10842–11820 | `BobLedgerToolsNoteReadyMixin`                     |
| `200-plugin-dashboard.js`             | 11821–12409 | `BobLedgerToolsDashboardMixin`                     |
| `210-plugin-ready-notes.js`           | 12410–12842 | `BobLedgerToolsReadyNotesMixin`                    |
| `220-plugin-heading-and-review.js`    | 12843–13668 | `BobLedgerToolsHeadingReviewMixin`                 |
| `230-plugin-freshness-api.js`         | 13669–14582 | `BobLedgerToolsFreshnessApiMixin`                  |
| `240-plugin-freshness-status.js`      | 14583–14926 | `BobLedgerToolsFreshnessStatusMixin`               |
| `250-plugin-freshness-mark-model.js`  | 14927–15411 | `BobLedgerToolsFreshnessMarkModelMixin`            |
| `260-plugin-freshness-mark-render.js` | 15412–16062 | `BobLedgerToolsFreshnessMarkRenderMixin`           |
| `270-plugin-dependency-model.js`      | 16063–16522 | `BobLedgerToolsDependencyModelMixin`               |
| `280-plugin-dependency-render.js`     | 16523–17350 | `BobLedgerToolsDependencyRenderMixin`              |
| `290-plugin-today-and-location.js`    | 17351–18073 | `BobLedgerToolsTodayLocationMixin`                 |
| `300-plugin-vim-and-snippets.js`      | 18074–18370 | `BobLedgerToolsVimSnippetsMixin`                   |

`190-plugin-note-ready.js` and `230-plugin-freshness-api.js` are the tight ones (about
980 and 916 original lines). A class wrapper of two lines still fits. If a wrapper or a
re-measure pushes a fragment over 1000, move the last whole method and its leading
comment to the next fragment. Do not split a method. `renderDependencyChipsIn` (about
402 lines) and `onload` (about 473 lines) stay whole. `buildNoteReadyHeadingDecorations`
keeps its nested `HeadingWidget = class extends WidgetType` and that `super()` call.

`310-install-methods.js` is new glue, modeled on task-status-cycler's
`installTaskStatusCyclerMixins`. Copy own descriptors from each mixin prototype with
`Object.getOwnPropertyDescriptors`, skip `constructor`, throw on a duplicate method
name, and preserve descriptor flags. Install the thirteen mixins in the table order
above. Do not instantiate mixins.

Post-class slices stay after the installer and before exports, byte-for-byte.
`let planCapsStatCache` stays a `let` in the first of these files, with both
reassignments in that same file.

| Fragment                                |                          Lines | Contents                                                                                       |
| --------------------------------------- | -----------------------------: | ---------------------------------------------------------------------------------------------- |
| `320-plan-lane-and-config.js`           |                    18372–18721 | Plan lane counter, plan config, `planCapsStatCache`                                            |
| `330-note-ready-model.js`               |                    18722–19308 | `noteFrontmatterFor` through `noteReadyCrowdedKey`                                             |
| `340-dashboard-views-and-plan-block.js` |                    19309–20190 | Dashboard collection helpers, note-ready view models, `planBlockModel`                         |
| `350-exports.js`                        | 20191–20404 plus one glue line | `module.exports = BobLedgerToolsPlugin;` then the original helpers object, key order unchanged |

`fragments.json` lists these 35 paths in the order shown. Run `npm run build`. The
committed `main.js` is the builder output only. A second build must leave those bytes
unchanged, and `npm run build:check` must pass.

## 4. Repoint the two ledger path comments

Only `plugins/block-id-prompt/main.js` names `bob-ledger-tools/main.js` as a copy
source:

- Line 41, the Pomodoro recognizers (`POMODOROS_HEADING_RE`, `LEVEL_TWO_HEADING_RE`,
  `LEDGER_LINE_RE`).
- Lines 104–107, the daily-notes format constants.

Both originals live in `010-load-and-constants.js`. Change only the path text so those
comments name `plugins/bob-ledger-tools/src/010-load-and-constants.js`. Do not change
the copied declarations. block-id-prompt stays a hand-edited plugin; this phase does not
split it. The 1000-line limit applies to fragments and other files this phase authors.
This comment retarget is the explicit exception for that already-oversized file.

Do not edit `plugins/bob-navigation-hotkeys/main.js`. Its Pomodoro comment around lines
40–48 mirrors ledger conventions but does not name `bob-ledger-tools/main.js`, and
bob-cli-47.3 is already editing that file. Record this instead:

```bash
sase bead note bob-cli-47.2 'PROPOSED FOLLOW-UP: retarget bob-navigation-hotkeys Pomodoro recognizer comment (about lines 40–48) at plugins/bob-ledger-tools/src/010-load-and-constants.js — left untouched because bob-cli-47.3 is already splitting that file'
```

task-status-cycler source comments name the ledger API, not `bob-ledger-tools/main.js`.
Leave them, and do not hand-edit the generated cycler `main.js`. Rebuild cycler only if
a stub or script change makes `build:check` report it stale, which it should not.

## 5. Docs, version, verification, deploy, and close

Update the bob-plugins README, and nothing else in bob-cli docs:

- In the plugins table, bump Bob Ledger Tools from `1.28.0` to `1.28.1`.
- In the Layout tree, show
  `bob-ledger-tools/{manifest.json,main.js,styles.css,src/fragments.json,src/*.js}`.
- In Development model, say that both task-status-cycler and bob-ledger-tools opt into
  the existing fragment build. Keep the shared-scope, mixin, check, and parity
  description. Mention ledger in the parity example. Do not rewrite the contract.

`AGENTS.md` already describes every plugin that has `src/fragments.json`. Leave it.
bob-cli `docs/plugins.md` describes deployed `manifest.json`, `main.js`, and
`styles.css`; it does not say `main.js` is hand-edited source. Do not edit it. The
README sentence that lists which plugins ship `styles.css` already omits ledger; leave
that omission and record it as a `PROPOSED FOLLOW-UP:` rather than expanding this phase.

Bump `plugins/bob-ledger-tools/manifest.json` from `1.28.0` to `1.28.1` only. Leave the
description and the other fields. This matches the cycler patch bump for a
behavior-neutral split. Do not bump other plugins.

Verify against the recorded base:

1. Self-parity of the unsplit file passed after the stub fix, and cycler parity against
   its pre-split parent still passes after that fix.
2. `npm run build`, a second build, and `npm run build:check` pass. The second build
   changes no generated bytes.
3. `node scripts/check-split-parity.mjs --plugin bob-ledger-tools --base <recorded-base>`
   passes: 212 helpers and 191 own prototype methods. No `--split-helper` is needed; the
   widget classes are not exported. Explain any intentional source difference in the
   commit message. There should be none in helper or method bodies.
4. `npm test` and `npm run validate` pass, including every existing ledger-tools test
   and `scripts/test-plugin-build.cjs`. No test name or body is removed or renamed. The
   freshness-mark surfaces test and the dependency chips test must still see one eager
   `StateEffect.define()` and must still capture `DependencyChipWidget` through their
   own stubs.
5. Every new or hand-edited fragment, the parity script, and any other file this phase
   authors is at most 1000 lines. `node --check` passes for each fragment and the
   bundle. `git diff --check` is clean. The diff is code movement, the installer and
   export glue, the parity stubs, the two block-id-prompt comment paths, the README
   lines, and the patch version.
6. A check failure that matches the clean-base run is a `PROPOSED FOLLOW-UP:` note, not
   a reason to keep the bead open. Fix failures the split causes.

Deploy from the opened checkout:

```bash
bob plugins sync --repo <opened-bob-plugins-path> --no-pull -p bob-ledger-tools
```

Confirm managed-file equality with `bob plugins list` using the same `--repo` and
`--no-pull`. If sync refuses because vault files are dirty, stop and report the
conflict. Never pass `--force`.

The runtime already owns this phase's `in_progress` status. Do not set status by hand.
Do not create beads. Do not close `bob-cli-47` or any ancestor.

Immediately before closing, run `sase bead epic-symbols bob-cli-47.2`. Planning found no
entries. If any appear, resolve each symbol or re-key that Justfile line to the
still-open parent epic or a later phase, then re-run until none remain. Close only this
bead:

```bash
sase bead close bob-cli-47.2 --note "<parity counts, test/validate, line limit, and sync evidence>"
```

Preserve the linked-repo changes through the required SASE final declaration with a
commit decision. Do not run raw `git commit`. Summarize the fragment split, the parity
stub hardening, and the 212-helper / 191-method parity result. Ancestor settlement
belongs to the land agent.
