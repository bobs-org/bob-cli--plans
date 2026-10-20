---
tier: tale
size: medium
title: Finish and land the Bob public docs site epic (bob-cli-a0)
goal: The six remaining defects the bob-cli-a0 docs-site epic caused are fixed and
  verified, and the epic is closed with its plan file marked done.
decisions:
  edit_link_check:
    ask: Should the weekly online link check stop fetching rendered GitHub edit links
      and verify them offline instead?
    choices:
      offline: Skip Bob edit URLs online; a new offline gate proves each targets a
        real source file
      throttle: Keep fetching edit links with lychee throttling; slower, and GitHub
        may still 429
    default: offline
    why: Edit URLs always 302 to /login, so fetching proves nothing; the offline gate
      checks more
    answer: offline
proposed_by: bbugyi200.apollo.bob-cli-a0.land
proposed_with: claude/opus
bead: bob-cli-a0
decided_by: auto
status: done
---

- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:**
  [bob-cli-a0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/README.md)

# Finish and land the Bob public docs site epic (bob-cli-a0)

## Context

Epic `bob-cli-a0` (plan `plan:202610/bob_public_docs_site.md`) built the public docs
site at <https://bobs-org.github.io/>. All 13 phases are closed, and its land agent has
verified the work and triaged every follow-up. The triage is recorded in the last note
on `bob-cli-a0` (read it with `sase bead read bob-cli-a0 -r "<why>"`).

Six defects caused by the epic remain. This tale fixes them and then closes the epic. It
has no land agent of its own, so **its last step is the epic's closeout**.

Do not run `just check-full`. Verification uses `just check` and `just docs-check`.

## 1. Material template tests use whatever `mkdocs` is on PATH

**Source:** epic note #7.

`tools/docs/tests/test_templates.py::_build` runs a bare `mkdocs`.
`test_templates_in_material` and `test_component_chips_in_material` are gated on
`shutil.which("mkdocs")`. The default uv groups (`dev`) do not install the optional
`material` group. On a host with a global Material install (athena:
`~/.pyenv/shims/mkdocs`, `~/.local/bin/mkdocs`), PATH falls through to that global
`mkdocs`. It lacks `markdown-callouts`, so both tests fail with
`Failed to load extension github-callouts / ModuleNotFoundError`.

- In `test_templates.py`:
  - Gate both Material tests on `pytest.importorskip("mkdocs")` and
    `pytest.importorskip("material")`, matching
    `tests/test_compatibility.py::test_compatibility_builds_in_material`.
  - Build Material with the project interpreter:
    `[sys.executable, "-m", "mkdocs", "build", "--strict", "--config-file", ...]`.
  - Delete the dead `args` list in `_build`, which is built and never used.
- Make the same interpreter change in `test_compatibility.py`'s Material build.
- Make `just docs-check` actually exercise the Material tests. In
  `tools/docs/bob_docs/check.py::_run_unit_tests`, run
  `uv run --project tools/docs --group material pytest tools/docs/tests`. Update
  `tests/test_check.py` if it pins the command.
- **Verify:**
  - Put a fake failing `mkdocs` first on PATH, for example a temp-dir script that prints
    `ModuleNotFoundError` and exits 1.
  - Run
    `uv run --exact --project tools/docs pytest tools/docs/tests/test_templates.py tools/docs/tests/test_compatibility.py`.
    `--exact` removes the material group. The Material tests must **skip**, not fail.
  - Run `uv run --project tools/docs --group material pytest` on the same files. They
    must **pass**.

## 2. Weekly online link check fails on GitHub 429s for edit links

**Source:** bob-cli-a0.13 note #2. Host issue `bobs-org/bobs-org.github.io#2` stays open
until a weekly run passes.

The weekly check runs `bob-docs links --online`
(`tools/docs/bob_docs/links.py::online_argv`, pinned lychee 0.24.2). Two runs failed:

- [39475174066](https://github.com/bobs-org/bobs-org.github.io/actions/runs/39475174066)
- [39499697687](https://github.com/bobs-org/bobs-org.github.io/actions/runs/39499697687),
  re-dispatched about 3 hours later.

Each failed only on four `[429] https://github.com/bobs-org/<repo>/edit/master/docs/...`
URLs, a different four each time, with zero Adobe errors. The site renders one edit link
per page, more than 100 in all. Anonymous GitHub edit URLs always answer
`302 -> /login`, even for a branch or path that does not exist:

- `curl -s -o /dev/null -w '%{http_code}' https://github.com/bobs-org/bob-cli/edit/no-such-branch/docs/no-such-file.md`
  prints `302`.

So fetching them online can only prove the repository exists.

> [!decision] edit_link_check = offline
>
> - **Offline gate.** Add an edit-link check that runs in `just docs-check` (a new gate
>   in `check.py` after the offline lychee gate, or inside offline `run_links`). It
>   scans the built site for `<a ... rel="edit">` hrefs and HTML-unescapes them:
>   Zensical/MiniJinja writes `&#x2f;` for `/`. Each href must parse as
>   `https://github.com/<owner>/<repo>/edit/<branch>/<path>`.
> - **What it checks.** Owner/repo and branch must match a known source:
>   - bob-cli: `repo_url` and `edit_uri` in `mkdocs.yml`;
>   - mounts: `docs/_meta/mounts.yml` and the assembler's `bob_edits` (`assemble.py`).
>
>   `<path>` must exist as a file in that source checkout: the repo root for bob-cli,
>   the mounted repo's resolved source directory for mounts. Report every failure with
>   its page; any failure fails the gate.
>
> - **Tests.** Add unit tests with a passing fixture site and failing fixtures: missing
>   file, wrong branch, unknown repo, and an escaped href.
> - **Online exclusion.** In `online_argv`, add exactly one more `--exclude`, next to
>   `ADOBE_OFL_URL_EXCLUDE`, as a named constant:
>   `^https://github[.]com/bobs-org/(bob-cli|bob-plugins|bob-mac-capture)/edit/`. Its
>   comment says anonymous edit URLs always 302 to /login, cites the two runs above, and
>   points at the offline gate.
> - **Keep everything else checked.** All other github.com URLs (blob, tree, repo,
>   issues) stay checked online.
> - **Pin it.** Update `tests/test_links_online.py` so the permitted exclusions are
>   exactly the Adobe URL and this edit-link pattern.
> - **Update the docs.** Update the `links.py` module docstring and the weekly-check
>   text in `docs/develop/build.md` and `tools/docs/README.md`.

> [!decision] edit_link_check = throttle
>
> Keep fetching edit links. In `online_argv`, add lychee's per-host rate limiting
> (`--host-concurrency 2 --host-request-interval 1s`; confirm the flags with the pinned
> binary's `--help`). Raise `ONLINE_RETRY_WAIT_SECONDS` so a 429 retries across GitHub's
> rate-limit window (for example 10s with the existing 3 retries). Pin the new argv in
> `tests/test_links_online.py`. Update the same docs text.

Either branch: the fix cannot be proven in CI this turn. The weekly workflow checks out
bob-cli `master`, and this work is committed only after the turn ends. So **do not**
dispatch the weekly workflow, and do not wait for one. The next run proves it: the
scheduled weekly at `41 6 * * 1`, or a later manual
`gh workflow run weekly.yml --repo bobs-org/bobs-org.github.io --ref main`. A pass
closes issue #2 automatically. Say this in the epic close note.

Local verification:

1. Run `just docs-build-material`.
2. Run `uv run --project tools/docs bob-docs links --online` against that Material build
   and record the summary. A failure that is not a GitHub 429 must be reported, not
   hidden.
3. Run `just docs-check`. It must pass, including the new edit-link gate for `offline`.

## 3. The demo vault's Tasks status registry breaks dependency capture

**Source:** bob-cli-a0.8 note #2.

The demo vault's settings are in
`examples/demo-vault/.obsidian/plugins/obsidian-tasks-plugin/data.json`. They declare
Blocked as `{"symbol": "?", "name": "Blocked", "type": "TODO"}` and omit
`nextStatusSymbol` and `availableAsCommand` on every status. That contradicts
`docs/reference/vault-format.md`, which says Blocked is `ON_HOLD`. It also breaks the
published guide `docs/guides/add-prerequisites.md` against the demo.

Reproduced on master c300239c:

1. Copy the demo vault and set `BOB_DIR` to the copy.
2. Run `bob capture 'Prep the bench @workshop^prep'`.
3. Run `bob capture --dry-run 'Sand the bench @workshop^sand &workshop:prep'`.

It fails:

```text
cannot reconcile Blocked [?] tasks: the Tasks status is incompatible (symbol="?",
name="Blocked", type=Todo, next="", availableAsCommand=false); expected symbol '?',
name Blocked, type ON_HOLD, next status ' ', and availableAsCommand true
```

With full entries, both the dry run and the real capture succeed, and the real capture
writes the BLOCKS / DEPENDS ON lines.

- Rewrite `statusSettings` with full entries, keeping the demo's names and the
  `dataview` / `#task` settings. Use
  `tests/fixtures/tasks_parity/vault/.obsidian/plugins/obsidian-tasks-plugin/data.json`
  as the reference:

  | Symbol | Name        | `nextStatusSymbol` | Type          |
  | ------ | ----------- | ------------------ | ------------- |
  | `" "`  | Ready       | `"x"`              | `TODO`        |
  | `"x"`  | Done        | `" "`              | `DONE`        |
  | `"*"`  | Next        | `"x"`              | `ON_HOLD`     |
  | `"?"`  | Blocked     | `" "`              | `ON_HOLD`     |
  | `"/"`  | In Progress | `"x"`              | `IN_PROGRESS` |
  | `"-"`  | Canceled    | `" "`              | `CANCELLED`   |

  Every entry also gets `"availableAsCommand": true`.

- Add a regression test under `tools/docs/tests/` that loads the demo file and asserts
  the Blocked and Next entries match what bob requires (symbol, name, type, next status,
  availableAsCommand).
- **Verify:**
  - Rerun the reproduction above with the checkout's `bob` (`cargo build`, then use the
    binary under `cargo metadata`'s `target_directory`). Both commands must succeed.
  - `just docs-check` must still pass. Its tutorial and CLI-media gates replay against
    the demo vault.
  - If the media gate reports drift, regenerate with `just docs-media` and review the
    diff before keeping it.

## 4. Copy polish in a Mac Capture recovery guide

**Source:** bob-cli-a0.13 note #3. Three of ten blind readers found
`docs/guides/fix-mac-capture-bob-resolution.md:49` garbled: "the `!` glyph returns to
the bullet-b".

`docs/reference/mac-capture/diagnostics.md` ("Menu-bar icon") defines the states:

- Ready is the bullet-b mark.
- "bob unresolved" turns the bullet into a `!`.

Reword the sentence to name the menu-bar icon. For example: "...matches `which bob` and
the menu-bar icon's `!` turns back into the Bob mark's bullet." Optionally link the
diagnostics page's menu-bar section, with a relative link that `bob-docs lint` accepts.
Labels and copy only; no page moves.

## 5. Authoring rule: close every fence on its own line

**Source:** bob-cli-a0.4 note #1.

In `docs/develop/writing-docs.md`, add a rule next to the existing fence rules (the
bullets that begin "In `bash`, `sh`, ..."). Close every fenced code block on its own
line with the same fence marker. If a fence is left unclosed, the strict Zensical build
scans the rest of the page as prose, and bare `[[#^block]]` links there fail the build.

## 6. Integration: `@@@` template start in the capture-syntax reference

Commit 0e9f915a landed after the epic started. It added the `@@@` template-start typing
assist to `docs/capture.md` and the `capture-rewrite` CLI page, but not to the epic's
`docs/reference/capture-syntax.md`.

Under the existing `### \`@@template:\`
{#template}`heading, add one or two sentences: editors that run`bob
capture-rewrite`, including the Mac capture panel, turn a typed `@@@`at the very start of a draft into the`@@template:`header and put the caret on the next line. Link the`../capture.md#bob-capture-rewrite`
contract.

Do **not** add a token-index row; it is an editor assist, not a capture token. Do not
touch the bare-Pomodoro-name docs (`=bugs`, `@route:id=bugs`). Active epic phase
`bob-cli-a7.3` owns them.

## 7. Verify everything

- `just docs-check` passes. The Material template tests now run, not skip.
- `just docs-build-material` passes.
- `just check` passes.
- `git diff --check` is clean.

## 8. Close out epic bob-cli-a0 (final step)

1. Run `sase bead epic-symbols bob-cli-a0`. If it lists any `--epic-symbol` entry,
   resolve it under the Symvision epic-whitelist policy: wire it up, privatize it, add a
   non-test pragma, or delete it. Only re-key a Justfile line to a bead that is still
   open and still needs the exemption. At planning time it reported none.
2. Close the epic, filling in your actual results:

   ```bash
   sase bead close bob-cli-a0 --note "<verification>"
   ```

   The note must cover:
   - This tale's six fixes and their verification: test results, the reproduction now
     passing, and the local online-links summary.
   - The land audit that preceded it: all 13 phases closed and reviewed; epic notes #1,
     #2, #4, #5, and #6 resolved by 1513a385; follow-ups triaged as in the land triage
     note (bob-cli-5q +1, notes on bob-cli-3y and bob-cli-8p, new bob-cli-aa snoozed to
     2026-11-06, human-only items left in the handoff artifact).
   - Integration since the epic started: task relations already wired in; `@@@` added
     here; bare Pomodoro names owned by bob-cli-a7.3.
   - Weekly issue #2 still awaits the next weekly run against the landed fix.

   Never pass `--force` just to make the close succeed. If the close is rejected for
   leftover epic-symbol entries, finish that cleanup and close again.

3. Run `just symvision` if the recipe exists. At planning time `just --list` had no such
   recipe; if so, say that in your final response.
4. In the epic's plan file, change frontmatter `status: wip` to `status: done`. That is
   `plan:202610/bob_public_docs_site.md`, the PLAN path that
   `sase bead read bob-cli-a0 -r "<why>"` prints.
5. `bob-cli-a0` has no `parent_bead`, so nothing further needs closing.
