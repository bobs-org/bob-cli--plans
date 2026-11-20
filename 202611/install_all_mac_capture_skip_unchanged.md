---
tier: tale
title: Rebuild and restart Bob Mac Capture only when its app changed
goal:
  just install-all skips the Mac Capture build, swap, and restart when the installed app
  already matches the checkout.
size: medium
decisions:
  rebuild_trigger:
    ask:
      Should install-all also skip the rebuild when only docs, tests, or repo metadata
      changed since the last install?
    choices:
      app_inputs: Docs/tests/CI-only changes skip; any other changed path rebuilds
      any_change:
        Any new commit or local change rebuilds; skip only on the exact installed commit
    default: app_inputs
    why: Tests/docs-only commits are common there and never change the built app
    answer: app_inputs
proposed_by: bbugyi200.athena.1ek
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 09:16:17
status: wip
---

# install-all: rebuild and restart Bob Mac Capture only when its app changed

## Goal

Today every `just install-all` on macOS rebuilds, re-signs, swaps in, and **restarts**
Bob Mac Capture, even when nothing in `../bob-mac-capture` changed since the last
install. The restart quits the running app and stashes any open draft. On SSH sessions
it can also stop at the signing keychain preflight. After this change, `install-all`
runs the Mac Capture install (build, swap, and restart) only when the installed app no
longer matches what the checkout would build. Otherwise the step prints a skip row and
does not touch the app. The change is entirely in bob-cli. bob-mac-capture's
`Scripts/install.sh` stays the single install path whenever a rebuild is needed.

## Current behavior (read these first)

- `scripts/install_all`:
  - `mac_capture_step()` runs `just signing-check` (interactive only), then always runs
    `just --justfile ../bob-mac-capture/justfile install ~/Applications` through
    `install_ui_run`. Exit 0 or 72 records a watermark with
    `install_ui_mark_installed bob-mac-capture`.
  - The header pulls all three repos, then calls `print_digest`, then `print_plan`, then
    runs the steps. `print_plan` adds `--step bob-mac-capture=… --parser swiftpm`
    whenever `mac_capture_eligible`.
  - Arguments are parsed with a `shift` loop that consumes `"$@"`. A bob-cli self-update
    re-execs with `exec "$root/scripts/install_all" "$@"`, so **flags do not survive the
    re-exec**. `-v` survives only because it is exported as `BOB_INSTALL_VERBOSE=1`.
    Internal handoff state uses `BOB_INSTALL_ALL_*` env vars that are read, then unset,
    near the top of the script.
  - `mac_capture_is_adhoc()` already detects an ad-hoc-signed installed app.
- `scripts/lib/install_ui/state.py`: `installed.json` watermarks
  `{sha, at, dirty, path}` per repo (`get_watermark`, `write_watermark`,
  `snapshot_repo`). `write_watermark` keeps a caller-supplied `at`.
- `scripts/lib/install_ui/commits.py` uses the watermark as the base of the "What's new"
  digest range. When the watermark equals HEAD, the repo is skipped in the digest.
- `scripts/lib/install_ui/history.py` (`predict_kind`, `DEPENDENCY_FILES`,
  `_artifact_exists`) already has a `noop` build kind for "watermark == HEAD, clean, app
  exists".
- bob-mac-capture's `Scripts/install.sh` (linked repo, read-only for this plan) builds
  via `Scripts/bundle.sh`, swaps `~/Applications/Bob Mac Capture.app` in with a backup,
  and restarts only copies that were running from that path. `cp -R` gives the installed
  `Contents/MacOS/BobMacCapture` a fresh mtime on every install. The app embeds no git
  metadata. It re-probes `bob` capabilities whenever the bob executable changes, so a
  bob-cli upgrade by itself never requires an app restart.

## Design

### 1. A Python verdict: is the installed app current?

Add a new module `scripts/lib/install_ui/app_current.py`. Keep it Python 3.9 compatible,
with the same `from __future__ import annotations` header style the portability test
enforces. Expose it as a new `install_ui` subcommand in `__main__.py`, with subcommands
kept alphabetical in the parser:

```text
install_ui app-current -r/--repo NAME -d/--dir DIR -a/--artifact PATH [-A/--advance]
```

- `--artifact` is the installed file whose presence and mtime stand for the installed
  app. install_all passes
  `~/Applications/Bob Mac Capture.app/Contents/MacOS/BobMacCapture`.
- Stdout is exactly one human-readable reason line, without a trailing period.
- Exit codes:
  - `0`: current; skip the install.
  - `1`: rebuild needed.
  - Anything else: treat as a rebuild. Internal exceptions go through the existing
    `_crash()` and return 70.

Evaluate these rules in order. The first match decides. `<short>` is the 7-character
watermark SHA.

1. The artifact is missing → `1`, `app not installed at <display path>`.
2. There is no usable watermark for `--repo` (missing, no `sha`, or unparsable `at`) →
   `1`, `no install record`.
3. The watermark `path` differs from the resolved `--dir` → `1`,
   `last install came from another checkout`.
4. The watermark `dirty` is true → `1`, `last install had local changes`.
5. The artifact's mtime is more than 2 s after the watermark `at` → `1`,
   `app was replaced outside install-all`. This catches a direct `just install` from the
   mac-capture checkout, or any other out-of-band reinstall, after the recorded one.
6. The watermark SHA does not resolve to a commit in `--dir` → `1`,
   `last install @<short> is not in this checkout`.
7. Build the changed-path set:
   - `git diff --name-only --no-renames <watermark sha>` (the watermark tree compared
     with the working tree, covering both new commits and tracked local edits);
   - plus `git ls-files --others --exclude-standard` (untracked files).

   Use a **tree diff, not ancestry**, so a rollback or rebase is judged by content.
   `--no-renames` makes a file moved between an app path and an ignored path count on
   both sides.
   - Drop every path that matches the repo's ignore list (see below).
   - Any path left → `1`, `N app file(s) changed since @<short>`. Use singular "file"
     for 1.

8. Otherwise → `0`:
   - If HEAD equals the watermark SHA and the tree is clean: `app current @<short>`.
   - Otherwise: `only docs/tests changed since @<short>`.

`--advance`, applied only on exit 0: when HEAD differs from the watermark SHA **and**
`git status --porcelain` is empty, rewrite the watermark to
`{sha: HEAD, at: <original at>, dirty: false, path: <resolved dir>}` with
`state.write_watermark`.

- Keeping the original `at` keeps rule 5 and the digest's "since your last install
  <when>" truthful.
- The advance makes the next run's digest stop repeating docs-only commits as "since
  your last install".
- A dirty tree is never recorded as clean. Its watermark stays put.
- An advance failure is silent and does not change the exit code.

Ignore list: a module constant keyed by repo name. A repo without an entry ignores
nothing.

```python
APP_IGNORED_PATHS = {
    "bob-mac-capture": (
        ".github/*", ".gitignore", ".swift-format", "AGENTS.md", "LICENSE", "README.md",
        "Scripts/docs-check.sh", "Scripts/docs-media.sh", "Scripts/docs_media.py",
        "Scripts/generate-*-fixtures.sh", "Scripts/macos-ci-check", "Scripts/test_*.py",
        "Scripts/typecheck*", "Tests/*", "docs/*",
    ),
}
```

Match repo-relative POSIX paths with `fnmatch.fnmatchcase`. `Package.swift`, `Sources/`,
`Resources/`, `justfile`, and every other script stay app inputs. Unknown paths always
rebuild. Reuse the git subprocess helpers in `commits.py` (`_run_git`, `is_commit`,
`get_head`) instead of adding new ones.

> [!decision] rebuild_trigger = any_change Drop `APP_IGNORED_PATHS` and rule 7's filter:
> any changed or untracked path returns `1` with `checkout changed since @<short>`, and
> rule 8 only ever reports `app current @<short>`. `--advance` then never has work to
> do, so omit it from the subcommand and from install_all. Drop the docs-only test cases
> below and the ignore-list sentence in the docs.

### 2. Wire it into `scripts/install_all`

Keep everything bash 3.2 safe; `test_portability.py` enforces this.

1. **Force flag.**
   - Add `-f|--force-mac-capture`, sorted before `-h`. Usage line:
     `-f, --force-mac-capture  Rebuild, reinstall, and restart Bob Mac Capture even when its app is current.`
   - Update the `usage()` paragraph to say that, on macOS, Mac Capture is rebuilt and
     restarted only when its app files changed since its last install.
   - Read `BOB_INSTALL_ALL_FORCE_MAC_CAPTURE` into `force_mac_capture` alongside the
     other `BOB_INSTALL_ALL_*` handoff vars, and add it to the `unset` list. The flag
     sets `force_mac_capture=1`.
   - In the self-update re-exec block, export `BOB_INSTALL_ALL_FORCE_MAC_CAPTURE` when
     it is 1, so the flag survives the re-exec.
2. **Decide once, after the digest.**
   - Add `mac_capture_decide()`. Call it only when `mac_capture_eligible`, **after
     `print_digest` and before `counting_rows=1` / `print_plan`**. The digest must read
     the old watermark first, so this run's digest still shows the new docs-only
     commits.
   - It sets `maccap_rebuild` (default 1) and `maccap_reason`, in this order:
     - `force_mac_capture=1` → rebuild with reason `forced by --force-mac-capture`.
     - The installed app is ad-hoc signed (`mac_capture_is_adhoc`) and
       `CODESIGN_IDENTITY` is not `-` → rebuild with reason
       `installed app is signed ad-hoc`. This keeps today's remedy ("rerun from a GUI
       Terminal … to sign with the stable identity") working, because only a reinstall
       can upgrade the signature.
     - Otherwise run
       `install_ui app-current --repo bob-mac-capture --dir "$mac_capture_dir" --artifact "$HOME/Applications/Bob Mac Capture.app/Contents/MacOS/BobMacCapture" --advance`.
       - Exit 0 → `maccap_rebuild=0` and the reason from stdout.
       - Exit 1 → rebuild with the stdout reason.
       - Any other status, or an unavailable helper (`BOB_INSTALL_UI=0`, no Python 3.9+)
         → rebuild with an empty reason. This is exactly today's behavior.
3. **Plan line.** In `print_plan`, add the `bob-mac-capture` `--step/--parser` pair only
   when `mac_capture_eligible` **and** `maccap_rebuild == 1`. This keeps the ETA honest.
4. **Step.**
   - In `mac_capture_step`, keep the existing banner, the eligibility and clone checks,
     and the `pull_detail` / `step_warning` bookkeeping.
   - Then, if `maccap_rebuild == 0`:
     - Print one row and return **before** `signing-check`, so no keychain prompt
       appears and nothing is built, swapped, restarted, or watermarked.
     - Detail: `"$pull_detail · $maccap_reason · not rebuilt"`.
     - Use the `–` skip glyph and an `add_summary 'bob-mac-capture' skip … ''` row.
     - When the pull warned (`step_warning > 0`), use the `⚠` warn row and summary, and
       add the warning to `WARNINGS`, matching how the other steps treat a failed pull.
   - When rebuilding with a non-empty `maccap_reason`, print
     `dim_row "rebuild  $maccap_reason"` before the signing check, mirroring the
     existing `signing  <identity>` row. Then run today's flow unchanged.
5. Leave the bob-cli and plugin steps, the Obsidian restart logic, and the digest code
   unchanged.

Example summary rows:

```text
  –  bob-mac-capture   up to date · 11dcba7 · app current @11dcba7 · not rebuilt
  –  bob-mac-capture   2 new commits · a2aab30 → 11dcba7 · only docs/tests changed since @a2aab30 · not rebuilt
```

## Tests

Run the suite with
`python3 -B -m unittest discover -s scripts/lib/install_ui/tests -t scripts/lib`, which
`just check-scripts` also runs. It must pass on Linux.

1. New `scripts/lib/install_ui/tests/test_app_current.py`. Use temp git repos and an
   isolated `BOB_INSTALL_STATE_DIR`, following the setup in `test_state.py`. Cover:
   - every rule's exit code and reason text;
   - a rename from `Sources/` into `docs/` counts as an app change;
   - an untracked `Sources/` file rebuilds, while an untracked `docs/` file stays
     current but is not advanced;
   - a rollback (HEAD moved to an older commit whose tree differs only in ignored paths)
     is current and is judged by tree diff;
   - `--advance` moves `sha` to HEAD, keeps `at`, writes `dirty: false`, and is a no-op
     when HEAD already equals the watermark;
   - artifact mtime more than 2 s after `at` rebuilds;
   - the CLI's exit 0, exit 1, and exit 70 on an injected exception.
2. `scripts/lib/install_ui/tests/test_install_all.py`, with Darwin runs via
   `FAKE_UNAME=Darwin`. First update the fixtures:
   - `FAKE_MAC_JUSTFILE`'s `recipe_install` creates
     `$HOME/Applications/Bob Mac Capture.app/Contents/MacOS/BobMacCapture` on success,
     like a real install;
   - give `Sandbox.commit_and_push` an optional repo-relative path so tests can push
     `Sources/…` versus `docs/…` commits. Its default top-level `note-N.txt` counts as
     an app change.

   Add these cases:
   - **Second run with nothing new skips.** `mac_order` still lists one `install`.
     Output has `app current @` and `not rebuilt`. The final line is `ALL INSTALLED`,
     and the exit status is 0.
   - **Docs-only push skips.** This run's digest still lists the commit. The watermark
     advances to the new HEAD, and the next run's digest shows no bob-mac-capture
     section.
   - **A `Sources/` push rebuilds.** A `rebuild  1 app file changed since @…` row
     appears, the install runs, and the watermark moves.
   - **`-f` / `--force-mac-capture` reinstalls an unchanged app**, including across a
     self-update re-exec: push a bob-cli commit first, so the flag reaches the re-exec'd
     script only through the env handoff.
   - **A deleted app bundle reinstalls.**
   - **An artifact mtime in the future reinstalls** with `replaced outside install-all`.
   - **A skip on a PTY never runs `signing-check`.**
   - **`BOB_INSTALL_UI=0` always reinstalls**, as today.
   - **A failed mac-capture pull plus a current app** gives a warn row and
     `INSTALLED WITH 1 WARNING`.
   - **A skipped step is left out of the plan record.** Check `runs/<run>/plan.json`
     when history is seeded, as `EarnedEtaTests` does.

   Keep every existing test green. In particular,
   `test_exit_72_warns_advances_watermark_and_posts_banner` and the signing-preflight
   tests still install, because their sandboxes have no prior watermark.

3. `bash -n scripts/install_all` and the full `just check-scripts`.

## Docs

Follow the docs memory: behavior changes go to the reference/guide that owns them.

- `docs/guides/install-everything.md`:
  - In step 1, say that on macOS it rebuilds and reinstalls `../bob-mac-capture` only
    when its app files changed since the last install.
  - Add a `## Restart Mac Capture` section next to `## Restart Obsidian`. It covers:
    - the skip rule, including what does not count as an app change (the ignore list in
      plain words);
    - that a reinstall restarts a running copy and stashes an open draft;
    - that a missing app, an ad-hoc signature, an out-of-band reinstall, or a dirty last
      install always reinstalls;
    - `just install-all -f` to force it;
    - that without the helper (`BOB_INSTALL_UI=0` or no Python 3.9+) it reinstalls every
      run.
  - Add one skip summary row to the example output, or show it in the new section.
- Run `just docs-check`; it must pass.
- No mounted bob-mac-capture page becomes false. `docs/install.md`'s "bob-cli's
  `just install-all` does the same" about identity creation still holds, because a
  missing or ad-hoc app always reinstalls. So no change in the linked repo.

## Out of scope

- Skipping the bob-cli cargo install or the plugin sync when unchanged.
- Any change to bob-mac-capture's `Scripts/install.sh`, `bundle.sh`, or the app.
- Restarting the app when only `bob` changed. The app already re-probes a replaced bob
  executable.

## Done when

- A repeat `just install-all` on the Mac with no relevant bob-mac-capture changes leaves
  the running app untouched and reports the skip row.
- Any app-input change, a forced run, a missing app, or an ad-hoc app still installs and
  restarts exactly as today.
- `just check-scripts` and `just docs-check` pass.
