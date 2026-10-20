---
tier: tale
size: medium
title: Finish and land the generated Obsidian plugin reference
goal:
  Fix the remaining plugin-reference defects and integration gaps, then close epic
  bob-cli-a0.10.1 and its parent phase bob-cli-a0.10 with verified evidence.
proposed_by: bbugyi200.apollo.bob-cli-a0.10.1.land
proposed_with: claude/opus
bead: bob-cli-a0.10.1
create_time: 2026-10-20 01:24:20
status: wip
---

- **PARENT:**
  [202610/plugin_reference.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/plugin_reference.md)
- **BEAD:**
  [bob-cli-a0.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.10.1.md)

# Finish and land the generated Obsidian plugin reference

## Context

The phases of epic `bob-cli-a0.10.1` are closed. Together they:

- mounted bob-plugins `docs/` at `/reference/plugins/` and `docs/develop/` at
  `/develop/plugins/` in the Bob site;
- added the static inventory generator `scripts/docs-inventory.mjs` (7 plugins, 76
  commands);
- wrote seven plugin pages, seven topic pages, and a developer hub with five pages;
- shortened the bob-plugins README to a front door;
- added the `npm run docs:check` wrapper `scripts/docs-check.sh`.

The land agent's review found one blocking wrapper bug, some orphaned README details,
small doc defects, a scanner gap, and bob-cli pages written by parallel epics that still
point at the old README. This tale fixes all of them and then closes the epic and its
parent phase. Everything below was already verified; you do not need to re-triage it.
Follow-up triage is done and recorded on the epic, so create no task beads.

**Repositories.**

- bob-cli is your workspace checkout.
- Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "..."`), use only
  the path it prints, and read its `AGENTS.md`.
- The plans sidecar holds the epic plan file. Open it the same way
  (`sase repo open plans`).

Commit evidence for the epic is `f3678291` and `b1f98d99` in bob-cli, and `be09382`,
`7e74901`, `fdf834d`, and `2af765d` in bob-plugins (base `bd6ff0a`).

**Conventions.** Read `docs.md` with `/sase_memory_read` before editing docs, and follow
`docs/develop/writing-docs.md`. The epic plan (`plan:202610/plugin_reference.md`, read
with `sase artifact read`) defines the shared content contract:

- Typed pages have a title and a description of at most 160 characters.
- Hubs have at most 400 handwritten words; reference pages at most 6,000.
- Generated regions stay untouched.
- Links inside one mounted tree are relative. Links between mounts or into bob-cli are
  absolute `https://bobs-org.github.io/...` URLs.
- No bead IDs, personal paths, or private hostnames in published prose.
- Registered default hotkeys must stay distinguishable from custom
  `.obsidian/hotkeys.json` bindings and editor/Vim handlers.

bob-cli root pages link into mounted pages with relative paths, as `docs/index.md` does
with `reference/plugins/index.md`. No plugin runtime file (`plugins/**`) may change.

## Step 1 — Fix the `docs-check.sh` wrapper (bob-plugins, blocking)

`refresh_cache` runs `git ... reset --hard`, which prints `HEAD is now at ...` to
stdout. The script captures stdout as the path in `cli_root=$(resolve_cli)`, so every
run that reuses the cached clone fails
(`cd: $'HEAD is now at …\n…/bob-cli': No such file`). Fix and harden
`scripts/docs-check.sh`:

1. Send every git command's stdout in `clone_cache` and `refresh_cache` to stderr
   (`>&2`). Keep the existing failure messages and exit codes. `resolve_cli` must print
   only the chosen path on stdout.
2. Make "validate the chosen checkout" real. Today `is_bob_cli` accepts any directory
   that contains a justfile. Require bob-cli markers as well: a justfile plus
   `tools/docs/pyproject.toml` and `docs/_meta/mounts.yml`. Extend `describe_invalid` so
   the message names the missing marker.
3. Before refreshing an existing cache clone, check that its `origin` URL is the
   expected bob-cli remote; accept a trailing `.git` or slash. If it is not, fail
   clearly and never fetch or reset it. User-owned trees (`$BOB_CLI_DIR` and the
   sibling) must still never be fetched or reset.
4. Compute `script_dir`/`plugin_root` with `pwd -P` so the forwarded
   `BOB_DOCS_BOB_PLUGINS_DIR` is canonical.

Extend `scripts/test-docs-check.mjs`:

- Make the git stub's `reset` print `HEAD is now at aaaaaaa init` to stdout, as real git
  does.
- Make the stub `clone` write the new bob-cli markers.
- Make the stub answer `remote get-url origin` (or whatever command you use).
- Add tests for:
  - refreshing an existing cache clone, asserting the exact `cwd` and
    `BOB_DOCS_BOB_PLUGINS_DIR` forwarded to `just`;
  - a cache clone with a mismatched origin, which fails without fetch or reset;
  - a directory with only a justfile, rejected as `$BOB_CLI_DIR`;
  - a symlinked plugin path, forwarded canonically.

Update existing fixtures (`writeJustfile` users) to create the markers. Keep the test
file in `package.json`'s explicit test list.

## Step 2 — Make the inventory scanner fail instead of skipping (bob-plugins)

The plan requires "every actual `addCommand` registration accounted for; unknown forms
fail with file/line". `findCommandCalls` in `scripts/docs-inventory.mjs` currently does
`continue` whenever the token after `addCommand` is not `(`. That silently skips
`this.addCommand?.({...})`, `.bind(...)`, `.call(...)`, and aliasing such as
`const add = this.addCommand`. Change it as follows:

- Treat the optional call `addCommand?.(` exactly like `addCommand(`.
- Allow a non-call reference only as the operand of a `typeof` guard
  (`typeof this.addCommand` / `typeof plugin.addCommand`). The 11 guards in
  bob-ledger-tools must keep working.
- Fail with file:line:column on any other non-call reference.
- Report empty command id/name errors at the registration's real location instead of
  `file:1:1`.
- Table cells that hold a code span (`escapeCode` values) must not double backslashes:
  backslash escapes are not interpreted inside code spans. Escape only `|` as `\|` in
  those cells. If the value contains backticks, use a backtick fence longer than its
  longest backtick run.

Add `node:test` fixtures to the existing scanner test file for each case: optional call,
`.bind`/`.call`, aliasing, the `typeof` guard still accepted, and a code-span cell
containing a backslash. Then run `npm run docs:gen`. The output must be byte-identical
for the real plugins: still 7 plugins and 76 commands (2, 2, 15, 46, 3, 0, 8). Run it a
second time and confirm no bytes change.

## Step 3 — Restore orphaned README details and fix doc defects (bob-plugins)

The original README is `git show bd6ff0a:README.md`.

**Restore these details.** Check each claim against current source and tests before
writing it, and keep each page within budget. `bob-navigation-hotkeys.md` is about 2,880
words.

- **`docs/bob-navigation-hotkeys.md`** (base README table row, line 17):
  - **Project note from a task.** Checkbox-less, ALL-CAPS direct child bullets that have
    their own sub-bullets become title-cased `##` sections. A matching existing header
    is reused case-insensitively; otherwise the section is appended. Descendants are
    copied verbatim as notes, not `#task` lines. The source task's managed schedule and
    work logs move under the new `^prj` task in source order.
  - **Converting back.** The same gesture on a project note's `^prj` turns the note back
    into a parent-task block in the parent's `## Tasks` section. It carries tasks,
    uppercased section bullets, and the `^prj` subtree, repoints `#^prj` links, and
    trashes the note. When a rewrite fails, the note is kept and a notice is shown.
  - **The binding for this gesture.** The README calls it `Ctrl+Shift+Alt+N`, but the
    generated table shows no registered default for `create-project-note-from-task`. A
    keydown handler near `src/610-plugin-jumps-and-counted-keys.js:634` matches `KeyN`.
    Determine from source whether this is a handler or a custom vault binding and label
    it that way.
  - **Renaming managed references.** Managed `ref/` reference notes with `source_pdf`
    must be renamed with `bob ref rename` (see `src/210-rename-and-line-context.js`).
  - **Emptied Pomodoros.** Pomodoro link cleanup deletes the open entries it empties but
    keeps entries with prose children and untouched placeholders.
  - **`Alt+F` stamping.**
    - It stamps the cursor task, or a dedicated Task Link's target, plus the next N when
      counted. The stamp goes through `keepLine`: an exact due Ready keep counts once,
      and other stamps clear `keeps`.
    - A single press on an exact, due, at-limit task opens the approved-decay decision
      card and writes nothing.
    - Counted and Task Link sessions skip those targets with a `1 needs a decision`
      notice.
  - **The Depends on row.**
    - `Ctrl+D` deletes the Depends-On line, the `[dependsOn::]` field, and legacy
      children, with immediate recovery.
    - Hand edits mirror once the cursor leaves the edited line; malformed lines are left
      alone.
    - The BLOCKED badge reads `🔒 waits on N` (N ≥ 1), else `🔒 scheduled YYYY-MM-DD`,
      else `🔒 blocked`.
  - **Sub-project cascade.** Task Card Depends on and Schedule changes on a `^prj` with
    sub-projects cascade top-down and report
    `· N sub-projects Blocked / Ready again / not updated` notices.
- **`docs/develop/plugin-apis.md`** (base README lines ~165–200):
  - The `freshnessDecayCard.version >= 2` consumer gate: the leaf plus `Alt+F to decide`
    appears only with the nav capability and decay on, with counting-only wording
    otherwise.
  - The `stampLine` placement rule. Nav, task-status-cycler, and block-id-prompt call
    `api?.freshness?.stampLine?.(line, dateText) ?? line` (root api `version >= 3`)
    instead of placing `fresh` themselves.
  - `keepLine` saturates at 999 and requires a valid prior `fresh` before today.
  - `isToday` is also false when fields are missing. `todayRank` returns the ledger
    index or `Number.MAX_SAFE_INTEGER`.
  - `dashboardLaneBudget` lane mapping: `pending` uses `IN_PROGRESS`; `next` uses symbol
    `*`. It colors only a strict section excess, while the no-argument
    `pendingBudget()`/`nextBudget()` keep whole-lane semantics.
  - `completeTaskAtCursor` closes through the Tasks `set-status-symbol-to-x` command and
    never writes `[x]` raw.
  - `recoverBlockedDependents` never throws.
- **`docs/develop/validation.md`:** `npm run validate` exits non-zero when any plugin
  fails, so it is safe in CI or a pre-commit hook.

Then reread the base README's plugin table rows (lines 20–26 of the table) and its
non-table sections against the new pages. Restore any remaining substantive detail you
find orphaned: a URI parameter or notice, an API shape, a safety guard, or a config key.
Record what you restored in the epic close note.

**Fix these defects:**

- `docs/block-id-defaults.md:14`: a code span inside a table cell holds an unescaped
  `|`, which splits the row on GitHub. Escape it as `\|`. Replace the real host name
  `apollo` with a synthetic one.
- `docs/develop/validation.md:71-72`: `` `\\p` `` / `` `\\s` `` render with two
  backslashes. Make them `` `\p` `` / `` `\s` ``.
- `docs/task-status-cycler.md:59`: remove the stray `>` in `Ctrl+Shift+]>` and
  `Ctrl+}>`.
- `docs/open-notes-uri.md:7`: "processes entries serially" changed the README's meaning.
  `enqueueBobOpenNotesUri` (`src/676-plugin-open-notes-uri.js`) queues whole URLs, so
  each URL is handled only after the previous one finishes. State that, and also
  describe entry order within one URL if the source confirms it.
- `docs/develop/architecture.md:89`: "from a bob-cli checkout" reads as if you run the
  command inside bob-cli. Say it runs from this checkout against a resolved bob-cli
  checkout.
- `README.md`: "Get started" points at `https://bobs-org.github.io/getting-started/`,
  which is the bob-cli Getting started contract. Make the three documentation links:
  - Get started → `https://bobs-org.github.io/start/obsidian/`
  - Reference → `https://bobs-org.github.io/reference/plugins/`
  - Develop → `https://bobs-org.github.io/develop/plugins/`

  Keep the generated requirements markers and every other front-door element.

- `CLAUDE.md` is a tracked hand-maintained mirror of `AGENTS.md` and lacks the
  docs-mount sentence. Add the same sentence: `docs/` is mounted into the Bob site; keep
  the inventory generated and run `npm run docs:check`.

## Step 4 — Integrate pages added by parallel epics (bob-cli)

The README is now a front door, so bob-cli pages that send readers to it for plugin
details are stale.

- `docs/reference/configuration.md:9-10`: plugin settings are "described in the
  bob-plugins README". Point at the [Obsidian plugins](plugins/index.md) reference.
- `docs/reference/configuration.md:259`: replace the README link with the Obsidian
  plugins reference and the [plugin deployment](../develop/plugins/deployment.md) page.
  Keep the existing `bob completion` and `../plugins.md` links.
- `docs/capture.md:1339`: "whose contract lives in the bob-plugins README". Link the
  [Go to task URI](reference/plugins/goto-task-uri.md) page.
- `docs/start/obsidian.md`:
  - Keep the clone link.
  - Add a short pointer to the [Obsidian plugins](../reference/plugins/index.md)
    reference for each plugin's commands, default hotkeys, and settings. Put it in
    section 3 or under Next.
  - If the page's tutorial-skip markers or search smoke depend on the surrounding text,
    keep them intact.
- `docs/reference/compatibility.md:17-19`: keep the `1.8.7` floor statement. Add that
  the plugin inventory generated from the manifests in
  [Obsidian plugins](plugins/index.md) is the source of truth for the floor.
- `.github/workflows/docs-publish.yml`: `bob-docs check --production` now runs the
  bob-plugins Node inventory generator (`tools/docs/bob_docs/gen_plugins.py`). The
  publish job has no Node setup.
  - Add an `actions/setup-node@v7` step with `node-version: "24"` (v7.1.0 is the current
    release; Node 24 is the active LTS) before "Run the gated production checks".
  - Add `setup-node v7` to the header comment's verified-pins list.
  - Add `"actions/setup-node@v7"` to `ALLOWED_USES` in
    `tools/docs/tests/test_publish_workflow.py`.
  - Add a test asserting that a setup-node step precedes the gates step in the publish
    job.
  - Do **not** touch `runs-on`: the missing `runs-on` keys belong to active epic
    `bob-cli-a0.5.1` and are recorded there.

Rerun `just docs-gen` if any generated bob-cli content depends on these pages, so
`generated drift` stays clean. Do not commit mount copies; they stay gitignored.

## Step 5 — Verify

bob-plugins, from the opened checkout:

- `node scripts/docs-inventory.mjs --check`
- `npm run docs:inventory:check`
- `npm run validate`
- `node --test` on the scanner and wrapper test files
- `npm test`. Six or seven known pre-existing failures are expected and tracked:
  bob-cli-6v (four unblocked-glyph cases), bob-cli-6w, bob-cli-7i, and the stage-ranker
  timing flake bob-cli-3w. Any other failure is yours to fix.
- `BOB_CLI_DIR=<your bob-cli workspace checkout> npm run docs:check`
- `git diff --check`

bob-cli:

- `just docs-gen`, then `just docs-check`. Both plugin mounts must resolve to the opened
  bob-plugins checkout, which the linked-checkout discovery does by default, or export
  `BOB_DOCS_BOB_PLUGINS_DIR` to it.
- `just docs-build-material`
- `git status` showing only intended tracked changes, with no mount copies.
- Confirm that the search smoke cases `install plugins` and
  `Yank absolute path with tilde` still pass.

No Rust is planned to change. Run `just check` only if Rust changes.

Then run `bob plugins sync` without force, as bob-plugins' `AGENTS.md` requires, and
record the result. Sync reads the canonical plugins checkout and these are docs-only
changes, so 0 copied is expected.

## Step 6 — Close out epic `bob-cli-a0.10.1` and its parent phase `bob-cli-a0.10`

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-a0.10.1`; at planning time
   there were none. Resolve any entry that now appears (wire it up, privatize it, add a
   non-test pragma, or delete it under the Symvision epic-whitelist policy). Re-key a
   Justfile line only to a still-open bead that genuinely needs it.
2. **Close the epic.** Run `sase bead close bob-cli-a0.10.1 --note "<verification>"`.
   The note must summarize:
   - the phase work verified by the land review: hubs, inventory with 76 commands, seven
     pages plus topic and develop pages, both mounts, the wrapper, and search smoke;
   - the fixes from Steps 1–4, including the restored README details;
   - the commands run and their results, with bob-cli and bob-plugins SHAs;
   - the known npm test failures and the beads tracking them;
   - the follow-up triage outcomes already recorded in the epic's LAND TRIAGE note.

   Never use `--force` merely to make the close succeed. If the close is rejected
   because of leftover epic symbols, clean them up and close again.

3. **Symvision.** Run `just symvision` and confirm the whitelist is clean.
4. **Mark the plan done.** Open the plans sidecar (`sase repo open plans -r "..."`) and
   set `status: done` in the frontmatter of `202610/plugin_reference.md`. This is the
   EPIC PLAN path that `sase bead read bob-cli-a0.10.1` prints.
5. **Close the parent phase.** Run
   `sase bead read bob-cli-a0.10 -r "Verify parent phase scope before closing"`. It is a
   phase of epic `bob-cli-a0`, scope `plugins-ref` in
   `plan:202610/bob_public_docs_site.md`.
   - **Confirm its acceptance is met:**
     - bob-plugins docs with a generated inventory and command tables;
     - the README front door;
     - the `docs:check` wrapper and the `AGENTS.md` line;
     - both mounts with nav placeholders;
     - retargeted Reference hub and homepage card;
     - the `install plugins` and command-name search smoke cases;
     - `npm run validate`, `npm run docs:check`, and `just docs-check` green;
     - the inventory showing seven plugins with correct generated versions.

     `npm test`'s pre-existing failures are tracked and do not block, per the epic plan.

   - **Close it.** Run `sase bead epic-symbols bob-cli-a0.10`, resolve any entries, then
     run `sase bead close bob-cli-a0.10 --note "<what you verified>"`.
   - **Do not close the outer epic `bob-cli-a0`.** Its land agent owns it.

## Acceptance

- The wrapper works on repeated cache runs and rejects non-bob-cli or foreign-origin
  checkouts; its tests cover each case.
- The scanner fails loudly on every non-recognized `addCommand` form, and the real
  output is unchanged.
- Every listed orphaned detail has a home. Page budgets hold, and generated regions are
  untouched.
- bob-cli pages no longer send readers to the README for plugin details, and the publish
  job sets up Node before the production gate.
- All Step 5 gates pass, except the tracked npm test failures.
- Epic `bob-cli-a0.10.1` and phase `bob-cli-a0.10` are closed, `just symvision` is
  clean, and the plan file says `status: done`.
