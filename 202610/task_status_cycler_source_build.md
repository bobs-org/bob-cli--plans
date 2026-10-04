---
tier: tale
title: Split task-status-cycler and establish the plugin source build
goal:
  Complete bob-cli-47.1 with bounded source fragments, a deterministic build and
  staleness check, verified code parity, and a safely deployed generated plugin.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-47.1
bead: bob-cli-47.1
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:**
  [bob-cli-47.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-47/bob-cli-47.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-47.1](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-47.1.md)
- **COMMITS:**
  - [6f8aca0](https://github.com/bobs-org/bob-plugins/commit/6f8aca0beae21e66922ae61a6d865d642d056803)
    — feat(plugins): add deterministic fragment build and split task status cycler

# Split task-status-cycler and establish the plugin source build

Implement the reserved phase **bob-cli-47.1**, `split-task-status-cycler`, from
`plan:202610/split_largest_bob_plugins_js_files.md`. Establish the reusable build
contract for the subsequent ledger-tools and navigation-hotkeys phases, then split only
`plugins/task-status-cycler/main.js`. This is substantial but bounded work for one
implementation agent, so use a medium tale rather than further phase beads.

## Scope and evidence

The primary project is bob-cli; implementation belongs in its **bob-plugins linked
repository**. Run
`sase repo open bob-plugins -r "Implement bob-cli-47.1 plugin source build and task-status-cycler split"`,
and use only the path it prints. Read that repo's `AGENTS.md`, which requires deploying
changes with `bob plugins sync`.

The inspected linked HEAD was `c3349aae7fa34c34aaecd2a9c85097fef5b5d762`. The target
remains 12,051 lines, with 183 exported helpers and 187 own plugin prototype methods
excluding `constructor`. Its existing suite contains 179 tests. The plugin class has no
fields, accessors, statics, private members, or `super` calls. The one `super(app)` is
in `DemotionSectionPickerModal`, which stays intact. There is no top-level `let`.
Re-measure and check these facts before extraction if the checkout has advanced.

`package.json` has no dependencies and invokes an explicit list of Node test files.
`scripts/validate-manifests.mjs` syntax-checks the deployed `main.js` of every direct
child of `plugins/`. About 40 suites and migration consumers load those entrypoints
directly. The README currently states that there is no build step and `main.js` is
hand-edited source. Change that policy for the split plugin. The other five plugins keep
their current layout in this phase; later phases opt into the same contract.

Keep all existing function, method, modal, and test bodies byte-for-byte. Preserve
top-level declaration order and existing exports. Changes to task behavior, styles, test
names, other plugins' source, bob-cli runtime code, and memory are out of scope. Every
newly created or hand-edited file must be at most 1000 lines. The generated `main.js` is
the sole exemption; existing oversized files that are not edited stay untouched. Do not
edit the original 6987-line cycler test script in this phase.

## 1. Capture the baseline and script the extraction

After opening the repo, record its clean Git base SHA and obtain the original target
from `git show <base>:plugins/task-status-cycler/main.js`. Run baseline `npm test` and
`npm run validate`, retaining output and failure names for later comparison. Use the
SASE monitor skill for commands that may outlast this single turn, and finish any
handoff command before ending the turn. Do not run a pull over uncommitted edits.

Write a throwaway extraction script using exact line slices with preserved line endings
and indentation. Assert the expected boundary declarations, class opening, and closing
before writing. Account for every source line exactly once, except the old plugin class
wrapper and export assignment that the necessary glue replaces. Keep comments with their
declarations; in particular start the Blocked/schedule fragment at line 370 and the Work
Log mixin body at line 9486. Retain the script outside tracked deliverables until
verification completes so extraction can be repeated if the input changes. Never
hand-merge changed bodies after an upstream update: re-extract the new source and
refresh the parity base.

## 2. Establish a reusable, deterministic build contract

Add zero-dependency `scripts/build-plugins.mjs` using Node built-ins. Discover plugin
directories with `src/fragments.json`; ignore plugins without that file. The manifest is
an explicit ordered JSON array of paths relative to that plugin's `src/`, and is the
only source of bundle order. Do not add a shared directory below `plugins/`, which
manifest validation would mistake for a plugin.

For each opted-in plugin:

- Validate a nonempty list of unique existing JavaScript fragment paths confined to
  `src/`; reject absolute paths, traversal/escaping symlinks, and duplicate entries.
  Ensure no JavaScript source fragment under `src/` is omitted from the manifest.
- Check every fragment as a standalone script with `node --check`, enforce at most 1000
  physical lines, and syntax-check the assembled bundle too. These are shared
  lexical-scope fragments, not independently required CommonJS modules.
- Concatenate in manifest order without altering fragment bytes, adding safe newline
  separators, a generated-file warning naming the builder and source manifest, and a
  `// ---- src/<relative-path> ----` banner before each fragment. Include no clocks,
  absolute paths, or other nondeterministic content.
- The default mode writes committed `main.js` artifacts, preferably only when bytes
  change. `--check` computes identical output and fails for stale/missing bundles,
  oversized fragments, malformed manifests, or syntax errors, without writing files.
  Validate all inputs before replacing outputs; print actionable plugin/path errors.

Expose a small callable builder accepting a repository root so fixture tests can
exercise it without modifying the actual plugin tree; direct CLI execution defaults to
the repository containing the script. Avoid introducing extra public CLI options solely
for tests.

Add `npm run build` and `npm run build:check`. Make both `npm test` and
`npm run validate` run the staleness check first, without rebuilding automatically.
Preserve the existing explicit test list and add only the focused tooling test below.
Tests, migration scripts, Obsidian, and bob-cli sync keep consuming committed `main.js`;
no runtime relative imports of fragments are introduced.

## 3. Extract the plugin into 21 cohesive fragments

Create `plugins/task-status-cycler/src/fragments.json` and the following ordered
fragments. Ranges refer to the recorded pre-split file and include blank lines and
comments. Re-check boundaries if the source has changed. Names are the recommended
contract; an implementation adjustment must preserve cohesion, order, and limits.

| Fragment in src/               | Original lines               | Contents                                                         |
| ------------------------------ | ---------------------------- | ---------------------------------------------------------------- |
| `010-core.js`                  | 1–369                        | Requires, constants, indent/date/stamp/repeat helpers            |
| `020-schedule-and-logs.js`     | 370–953                      | Blocked cycle constants, schedule/list/log/retirement helpers    |
| `030-task-toggles.js`          | 954–1300                     | Status checks and task promote/demote/toggle                     |
| `040-sections.js`              | 1301–2221                    | Markdown section parsing and section-move plans                  |
| `050-picker-and-links.js`      | 2222–3048                    | Picker/document plans, block links, Depends-On detection         |
| `060-pomodoro.js`              | 3049–3952                    | Pomodoro helpers and plans                                       |
| `070-block-links.js`           | 3953–4350                    | Link targets, fences, strikethrough                              |
| `080-dependencies.js`          | 4351–4860                    | Dependency parsing, recovery, retirement/restoration             |
| `090-source-and-formatting.js` | 4861–5500                    | Source edits, dependency IDs/cache, checkbox/bullet format       |
| `100-editor-and-modal.js`      | 5501–6162                    | Editor/scroll/UI helpers and unchanged picker modal              |
| `110-plugin-lifecycle.js`      | 6164–6315                    | Core class with unchanged onload/onunload                        |
| `120-plugin-dependency-ids.js` | 6316–6872                    | Dependency normalization, rename reconciliation, file processing |
| `130-plugin-references.js`     | 6873–7504                    | Reference queue, recovery and strike/unstrike                    |
| `140-plugin-vim.js`            | 7505–8307                    | Vim mappings, navigation, scrolling and open-line handlers       |
| `150-plugin-commands.js`       | 8308–8977                    | Commands, physical-key listeners, range cycling                  |
| `160-plugin-completion.js`     | 8978–9485                    | Open/done and transcluded close/reopen/cycle                     |
| `170-plugin-pomodoro.js`       | 9486–10184                   | Work Log writes and Pomodoro complete/reopen                     |
| `180-plugin-editor-edits.js`   | 10185–11049                  | Toggles, demotion, status, formatting and line editing           |
| `190-plugin-targets.js`        | 11050–11864                  | Target resolve/replace, freshness and utilities                  |
| `200-install-methods.js`       | new glue only                | Duplicate-safe mixin installation                                |
| `210-exports.js`               | 11867–12051 plus export glue | Default plugin export and unchanged helpers object               |

In `110-plugin-lifecycle.js`, declare `class TaskStatusCyclerPlugin extends Plugin` with
just the original `onload` and `onunload` bodies. Wrap the next eight original method
slices in descriptively named plain mixin classes without changing any method text or
indentation. Each fragment is a complete valid script and comfortably below 1000 lines
including its new class wrapper. No methods in these slices use `super`.

The installer copies `Object.getOwnPropertyDescriptors(Mixin.prototype)` onto the plugin
prototype, excluding `constructor`. Reject duplicate own method names against the
lifecycle class and earlier mixins rather than silently overwriting. Preserve descriptor
flags and install the eight groups in original order, after their classes are declared
and before export. Do not instantiate mixins or copy inherited methods. This preserves
`new TaskStatusCyclerPlugin()` and `Object.create(TaskStatusCyclerPlugin.prototype)`
consumers. The final fragment sets `module.exports = TaskStatusCyclerPlugin` and retains
the exact original helpers object in its original key order. Build and commit the
resulting generated `main.js`.

## 4. Commit reusable parity verification and focused tooling coverage

Add `scripts/check-split-parity.mjs` with documented plugin/base arguments, for example
`node scripts/check-split-parity.mjs --plugin task-status-cycler --base <sha>`. Read the
original source with `git show`, then load it and the generated entrypoint in isolated
CommonJS module instances with identical test-derived Obsidian and CodeMirror stubs.
Restore any temporary `Module._load` interception in `finally`; do not import whole test
scripts or register tests as part of parity checking. Give both versions equivalent
filenames/require contexts and clean up temporary files if used. Use a bounded but
sufficient `git show` output limit for future 52k line files. Report exact helper or
method names on failure, and exit nonzero.

Assert identical default export names, exact ordered `Object.keys(helpers)`, helper
types, and `String(value)` for every helper function and unsplit helper class. Compare
nonfunction helper values structurally. For the exported plugin class, compare own
prototype keys, descriptors, and the exact source string of each method except
`constructor`. The core class's whole source string necessarily differs; verify its
methods rather than allowing arbitrary function mismatches. The modal remains an unsplit
helper class and must pass whole-class source equality.

Keep this tool reusable for ledger-tools/navigation without designing their splits now:
support an explicit list of additional split helper classes to compare by methods (the
later navigation modal), and test-derived dependency stubs sufficient to load current
plugins. For ledger-tools, preserve the optional CodeMirror load path and record
load-time calls so old/new side effects can be compared. Do not run lifecycle hooks or
mutate the vault from the parity tool.

Add a compact `scripts/test-plugin-build.cjs` using Node's test runner and temporary
repository fixtures. Verify declared ordering/shared scope, deterministic rebuilds,
successful read-only checks, stale-bundle rejection after fragment or output edits,
line-limit and syntax rejection, and malformed/duplicate/escaping/missing/omitted
fragment rejection. Confirm read-only failures do not mutate fixture output. Give the
parity comparator a small valid/mismatched fixture check if needed to prove it rejects a
changed helper or method rather than merely loading both sources. This coverage
validates the new build boundary; do not rewrite existing plugin tests.

## 5. Document, version, verify, deploy, and close this phase

Update bob-plugins README Layout, Development model, Validation and deployment guidance
to distinguish plugins with `src/fragments.json` from hand-edited plugins. Explain the
shared scope/order contract, generated committed entrypoint, standalone syntax checks,
mixin installer, line limit, build/check commands, and parity usage. Add to bob-plugins
`AGENTS.md`: edit `src/` for built plugins, run `npm run build`, never hand-edit their
generated `main.js`, and retain the existing sync requirement.

Follow the observed manifest-version convention: bump only task-status-cycler from
`1.23.0` to `1.23.1` for this behavior-neutral refactor (adapt if a newer upstream
version is the actual base). Keep its README table version consistent. Leave plugin
metadata other than that version unchanged. bob-cli `docs/plugins.md` was inspected and
accurately describes managed deployed files; it does not assert that `main.js` is
hand-edited source, so no host documentation edit is needed.

Verify the following against the recorded extraction base:

1. `npm run build`, a second deterministic build, and `npm run build:check` pass; the
   second build leaves all generated bytes unchanged.
2. The parity command passes for all 183 helpers and 187 own prototype methods (or
   re-measured counts for a refreshed base), with the unchanged modal included.
3. `npm test` and `npm run validate` pass, including the new focused tool tests and all
   179 cycler tests. No original test name or body was removed or renamed.
4. All touched hand-edited files pass `wc -l <= 1000`, fragment/assembled syntax checks
   pass, and `git diff --check` is clean. Review the diff for only exact code movement,
   required glue/tooling/docs, and the patch version bump.
5. Compare any check failure on an unchanged clean base before attributing it to this
   phase. A failure reproducing identically on clean base does not block closing: add
   `PROPOSED FOLLOW-UP: <summary — identical base/worktree evidence and any known tracking bead>`
   with `sase bead note bob-cli-47.1`. Fix phase-caused failures.
6. Deploy from the opened linked checkout explicitly with
   `bob plugins sync --repo <opened-repo-path> --no-pull -p task-status-cycler` and
   verify managed-file equality using `bob plugins list` with the same repo/no-pull
   arguments. If sync refuses dirty installed files, stop deployment and report the
   concrete conflict; never pass `--force` or overwrite those changes. The deploy
   requirement remains outstanding until safely satisfied.

The runtime already owns the phase's `in_progress` status; do not set status manually.
Create no beads. Append any discovered follow-up only with
`sase bead note bob-cli-47.1 'PROPOSED FOLLOW-UP: <summary — detail>'`.

Immediately before closing, run `sase bead epic-symbols bob-cli-47.1` again (planning
found no entries). Resolve any listed symbol or re-key its Justfile line to the
still-open parent epic/later phase, then re-check for no leftovers. Once this phase's
work is verified, close **only** `bob-cli-47.1` with
`sase bead close bob-cli-47.1 --note "<build, parity, test, line-limit and sync evidence>"`.
Do not close `bob-cli-47`, any ancestor plan, or unrelated phases. Preserve the linked
repo changes through the required SASE final declaration with a commit decision and an
informative summary of the build contract and parity evidence; do not use raw
`git commit`. Ancestor plan settlement belongs to its land agent/runtime.
