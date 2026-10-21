---
tier: epic
title: Daily release train for bob-cli (CI, package hygiene, release-plz rehearsal,
  docs)
goal: 'bob-cli has hermetic per-push CI on Linux and macOS and a renamed, trimmed
  package under the crates.io size cap. A scheduled release-plz train releases at
  most once per New York day, only after full CI passes on the exact SHA, and runs
  in git_only rehearsal mode (tags and GitHub Releases), ready to switch to crates.io.
  The process is documented on the docs site in the Diátaxis structure, and Bryan
  gets a report of the manual go-live steps that only he can take.

  '
decisions:
  crate_name:
    ask: Which crates.io package name should replace the taken bob-cli (the bob binary
      keeps its name)?
    choices:
      bob_obsidian: Matches the repo's "Bugyi's Obsidian" description and reads clearly
        in search
      bob_vault: Shorter, but less clearly tied to Obsidian
      bobs_cli: One typo away from the unrelated bob-cli crate
    default: bob_obsidian
    why: Descriptive, free on crates.io today, and not a near-homograph of bob-cli
    answer: bob_obsidian
  github_apply:
    ask: Should agents apply the GitHub setup (environment, rulesets, arm variable)
      and dispatch the first rehearsal?
    default: true
    why: Each setting and a git_only rehearsal release can be undone
    answer: true
  hotfix_override:
    ask: Should a manual run be able to bypass the once-per-day guard when given a
      written hotfix reason?
    default: true
    why: Without it, a fix for a bad release waits until the next day
    answer: true
  binaries:
    ask: Should each release attach prebuilt Linux x86_64 and macOS arm64 binaries
      for cargo binstall?
    default: true
    why: Installs then skip compiling QuickJS, image, and SQLite from source
    answer: true
  memory_release_train:
    ask: Should docs-dev add a release-train decisions record telling agents never
      to edit version or CHANGELOG.md?
    memory:
    - decisions
    default: true
    requested: Can you help me implement the recommendations made by the crates_io_daily_release_train.md
      file in the research sidecar repo?
    answer: true
phases:
- id: hermetic
  title: Hermetic test suite and pinned toolchain
  depends_on: []
  size: small
  description: 'hermetic: fix the three tests that fail in a clean environment and
    add rust-toolchain.toml pinned to 1.95.0.'
- id: package
  title: Package rename, metadata, trim, size budget, and install migration
  depends_on: []
  size: medium
  description: 'package: rename the package, pin the lib name, seed 0.2.0 and CHANGELOG.md,
    trim the include list under the size budget, add package-check, migrate installs,
    and fix docs the rename makes false.'
- id: ci
  title: Reusable per-push CI workflow brought green on GitHub
  depends_on:
  - hermetic
  - package
  size: medium
  description: 'ci: add .github/workflows/ci.yml (push, ci/** branches, dispatch,
    workflow_call with sha/full) and iterate on a scratch branch until every Linux
    and macOS job is green.'
- id: train
  title: release-plz train in git_only rehearsal mode
  depends_on:
  - ci
  size: medium
  description: 'train: add release-plz.toml, the tested daily guard, release.yml (plan,
    CI, release, preview, optional binaries), and just release-preview, and validate
    them with release-plz locally and a dry run on GitHub.'
- id: github
  title: GitHub setup and first rehearsal release
  depends_on:
  - train
  size: small
  description: 'github: dry-run the train on master, then (per github_apply) create
    the release environment and rulesets, arm the train, dispatch the first rehearsal
    release, and verify the same-day guard.'
- id: docs-dev
  title: Maintainer release documentation and decision record
  depends_on:
  - train
  size: medium
  description: 'docs-dev: write the Develop > Releases section (hub, tutorial, explanation,
    how-to guides, reference) and the CI reference, add the release-train decision
    record, and regenerate the docs.'
- id: docs-user
  title: Reader-facing release documentation and site integration
  depends_on:
  - docs-dev
  size: medium
  description: 'docs-user: add the Releases and versions reference and the tagged-release
    install guide, update hubs, nav, the release checklist, and the URL contract,
    and pass the full docs gate.'
- id: handoff
  title: End-state verification and Bryan's go-live report
  depends_on:
  - github
  - docs-user
  size: small
  description: 'handoff: verify CI, the train, and the docs against what landed, finish
    any pending rehearsal dispatch, and publish an artifact report of Bryan''s remaining
    steps.'
proposed_by: bbugyi200.apollo.95.w1
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 18:36:50
status: done
bead_id: bob-cli-ac
---

- **PROMPT:** [prompts/202610/crates_io_release_train.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/crates_io_release_train.md)
- **BEAD:** [bob-cli-ac](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ac/README.md)

# Plan: Daily release train for bob-cli

## Context

This epic implements the recommended solution in the research report
`research:202610/crates_io_daily_release_train/crates_io_daily_release_train.md`. Every
phase agent should read it first:

```bash
sase artifact read research:202610/crates_io_daily_release_train/crates_io_daily_release_train.md "Implement the release-train epic phase"
```

The report recommends a scheduled release train instead of the release-plz quickstart or
a release-PR flow. Each train run applies a registry-backed once-per-New-York-day guard
and a pause switch. It then runs full CI on the exact SHA, runs `release-plz update`,
pushes the version bump fast-forward only, and runs `release-plz release`. The train
rehearses in `git_only` mode first and moves to crates.io Trusted Publishing only after
a manual first publish.

### Facts re-verified on 2026-10-20 (deltas from the report)

- **License: done.** `LICENSE` is MIT and `Cargo.toml` already has `license = "MIT"`
  (commit `c7df845f`). No license work remains.
- **Size is now over the cap.** `cargo package --no-verify` packages 1,147 files at 11.2
  MiB compressed, above crates.io's 10 MiB (10,485,760-byte) limit. Most of the growth
  is the docs site under `docs/`. These are compressed sizes by top-level directory:

  | Directory | Compressed size |
  | --------- | --------------- |
  | `docs`    | 2.9 MB          |
  | `assets`  | 2.8 MB          |
  | `scripts` | 2.8 MB          |
  | `src`     | 2.6 MB          |
  | `tests`   | 0.7 MB          |

  Keeping only `src/`, `assets/`, `scripts/` and the metadata files gives about 8.19 MB
  (7.8 MiB), which is under the 8 MiB warning line but close to it.

- **Clean-environment tests: 3 failures, not 2.** A run of
  `env -i HOME=<tmp> PATH=/usr/local/bin:/usr/bin:/bin:~/.cargo/bin TZ=UTC cargo test --locked --no-fail-fast`
  fails these three tests:
  1. `native::capture::task_blocks::tests::authored_children_and_schedule_log_are_added_in_order`
     (`src/native/capture/task_blocks.rs:568`) needs `$HOME/.config/bob/config.yml` for
     priority levels.
  2. `tests/gkeep_auth.rs::doctor_all_ok_reports_the_checklist` (line ~438) fails with
     `uv not found on PATH`.
  3. `tests/cli/completion/zsh_adapter.rs::no_color_uses_plain_header` (line ~435) fails
     when no UTF-8 locale is set: zsh prints the header's box-drawing characters as
     `$'\M-b…'` meta escapes.
- **Names.** On 2026-10-20 the sparse index returns 404 (free) for `bob-obsidian`,
  `bob-vault`, `bobs-cli`, `bob-gtd`, `bobctl` and `bobsidian`. `bob-cli` and `bob` are
  taken by unrelated projects.
- **Nothing depends on the package name.** The Mac app, the plugins and chezmoi never
  name the Cargo package. Runtime state paths are hard-coded `bob-cli` strings (for
  example `state_home().join("bob-cli")`), not `CARGO_PKG_NAME`, so they do not change.
  No test pins `0.1.0`; `CARGO_PKG_VERSION` is read dynamically. The only literal
  `bob 0.1.0` strings are sample output in `docs/start/install.md` and
  `docs/guides/install-everything.md`.
- **The rename breaks install commands.** `cargo install --git … --locked bob-cli`
  appears in `README.md`, `docs/start/install.md:80` and `docs/completion.md:229`.
  `cargo install --path .` will also refuse on every host that already has `bob-cli`
  installed ("binary `bob` already exists … as part of `bob-cli`").
- **Existing GitHub state.** The repo is public and its default branch is `master`.
  Workflows: `docs-publish.yml` and `docs-weekly.yml`, both `workflow_call` only; the
  `bobs-org/bobs-org.github.io` deploy repo calls them hourly and weekly and publishes
  the docs site from bob-cli `master`. There are no rulesets, no environments, no
  Actions variables and no tags.
- **release-plz v0.3.173, verified in source:**
  - With no matching tag in `git_only` mode, a package is treated as an initial release.
  - For an unreleased package, if `CHANGELOG.md`'s newest entry already names the
    current version, `update` leaves the changelog untouched (`updater.rs` around line
    1225).
  - Without that entry, the first changelog scans up to `max_analyze_commits` (default
    1,000) historical commits.

  Seeding `version = "0.2.0"` together with a hand-written `## [0.2.0]` entry therefore
  makes the first train release a clean, short v0.2.0.

- **GitHub only dispatches workflows that exist on the default branch.** So a workflow
  can be tested on GitHub before landing only through a `push` trigger on a scratch
  branch.

### Target end state

- `rust-toolchain.toml` pins 1.95.0, and `Cargo.toml` declares `rust-version = "1.95"`.
- The package uses the chosen name, has `[lib] name = "bob_cli"`, is at version 0.2.0,
  has a trimmed `include` list, and passes `just package-check` (fail above 9 MiB, warn
  above 8 MiB).
- `.github/workflows/ci.yml` runs on every push to `master` and to `ci/**` branches, and
  the train calls it through `workflow_call`.
- `release-plz.toml` sets `git_only = true` for the rehearsal.
- `.github/workflows/release.yml` holds the train:
  - two cron slots, at 04:17 and 06:47 New York time in summer;
  - `workflow_dispatch` with `dry_run` (and `override_reason` if accepted);
  - a forced-dry-run `push` trigger on `ci/release/**` scratch branches.
- `tools/release/` holds the stdlib Python guard, the package checker and their tests.
- `just` recipes: `package-check`, `check-release-tools` and `release-preview`.
- Docs: a Develop > Releases section, a Develop > CI page, a Reference > Releases and
  versions page and a Guides > Install a specific Bob release page.
- Memory: one new decision record, `decisions:bob-cli-releases-ride-a-daily-train`.

### Rules for every phase

- Land work only through the normal SASE commit flow. Never push to `master` by hand.
  This epic explicitly allows pushing **scratch branches** named `ci/<slug>` or
  `ci/release/<slug>` to `origin` to run Actions before landing. Delete each scratch
  branch when the phase finishes (`git push origin --delete <branch>`).
- Read `sase memory read docs.md` before touching `docs/`, `mkdocs.yml` or `README.md`.
  Read `sase memory read cli_rules.md` only if you add `bob` CLI surface; none is
  planned.
- Pin every third-party action in new workflows to a full commit SHA, with a trailing
  `# vX.Y.Z` comment. Resolve SHAs with
  `gh api repos/<owner>/<repo>/commits/<tag> --jq .sha`.
- Python tooling under `tools/release/` is stdlib-only (Python ≥ 3.11, for `tomllib` and
  `zoneinfo`). Every behavior that needs testing takes injectable inputs (clock, fetched
  data), so the tests are offline and deterministic.

**Crate name.** Throughout this plan, `<crate>` means the package name chosen in the
`crate_name` decision: `bob-obsidian` by default, otherwise `bob-vault` or `bobs-cli`.
If the chosen name is no longer free (a sparse-index lookup no longer returns 404) when
the package phase runs, use the next free name in that order. Flag the substitution in
the package bead note so the handoff report surfaces it.

## Phase hermetic: hermetic test suite and pinned toolchain

Goal: the full suite passes in a clean environment, so CI can be required from day one.
The pinned toolchain keeps a new stable clippy lint from turning `master` red overnight
and silently stopping releases.

1. Fix the three clean-environment failures listed under Context. Find each root cause
   instead of only hiding the symptom:
   - **task_blocks:** give the test its own priority-level config through an existing
     test seam (an injected config, or a temp config dir passed explicitly). Do not
     mutate process-global env from a multi-threaded unit test unless the module already
     serializes env access.
   - **gkeep doctor:** the test spawns `bob`. Give it a test-controlled `PATH` holding a
     stub `uv` executable, so the "all OK" checklist does not depend on the host having
     `uv`.
   - **zsh adapter `no_color_uses_plain_header`:** give the zsh subprocess an explicit
     UTF-8 locale that exists on both Linux and macOS (for example try `C.UTF-8`, then
     `en_US.UTF-8`), or make the assertion locale-independent. Check the test's intent
     first: it asserts on the plain header under `NO_COLOR`.
2. Add `rust-toolchain.toml` at the repo root:
   - `[toolchain] channel = "1.95.0"`
   - `components = ["rustfmt", "clippy"]`
   - `profile = "minimal"`

   Do not edit `Cargo.toml`; the package phase owns it and adds the matching
   `rust-version`.

3. Verify:
   - `just check` passes.
   - The clean-environment command from Context passes with 0 failures. Also run it once
     with `TZ=America/New_York` to catch date assumptions.
   - Record the clean-run wall time in the bead note.

## Phase package: package rename, metadata, trim, size budget, and install migration

Goal: a publishable package that is safe to release every day. This phase owns every
`Cargo.toml` change in the epic except release-plz-generated version bumps.

1. **`Cargo.toml` `[package]`:**
   - `name = "<crate>"`
   - `version = "0.2.0"`: a one-time seed. After this phase only the train changes it.
   - `rust-version = "1.95"`, matching `rust-toolchain.toml`.
   - A crates.io-quality `description`, for example "Command-line tools for Bob, a GTD
     workflow that lives in one Obsidian vault".
   - `homepage = "https://bobs-org.github.io/"` and
     `documentation = "https://bobs-org.github.io/"`.
   - `readme = "README.md"`.
   - At most 5 `keywords`: `obsidian`, `gtd`, `pomodoro`, `cli`, `tasks`.
   - `categories = ["command-line-utilities"]`.

   Add `[lib] name = "bob_cli"` with `path = "src/lib.rs"`, so `bob_cli::…` imports keep
   compiling. Keep the four `[[bin]]` targets unchanged.

2. **Trim `include`** to:
   - `/Cargo.toml`, `/Cargo.lock`, `/README.md`, `/LICENSE*`, `/CHANGELOG.md`
   - `/src/**`, `/assets/**`
   - only the `scripts/` files the binaries embed (`include_bytes!` / `include_str!`) or
     execute at runtime. Inventory them with
     `grep -rn 'include_bytes!\|include_str!' src` plus the script shims in
     `src/scripts.rs`.

   Keep every third-party license or notice that ships with embedded assets: OFL fonts,
   DejaVu, Mermaid, Defuddle, Moment and anything else the inventory finds. Drop
   `docs/`, `tests/`, `justfile`, `tools/`, `vault/`, `sase/` and `.github/`. Tests
   under `src/` that read `tests/fixtures` will not run from the extracted crate; that
   is acceptable because package verification only builds.

3. **`tools/release/package_check.py`** with unit tests in
   `tools/release/test_package_check.py`, run by **`just package-check`**:
   - Run `cargo package --locked`, with a `--no-verify` flag for quick local runs. Find
     the `.crate` under the Cargo target dir (use `cargo metadata`; do not assume
     `target/`).
   - Print its size and its share of the 10,485,760-byte cap.
   - Exit nonzero above 9 MiB (9,437,184 bytes). Emit a warning above 8 MiB (8,388,608),
     as a `::warning::` annotation when `GITHUB_ACTIONS` is set. On a warning or
     failure, print the largest contributors by directory.
   - Fail if `cargo package --list` contains any path under `docs/`, `tests/`, `tools/`,
     `vault/`, `sase/` or `.github/`.
   - Fail if a required license or notice file is missing.

   Add **`just check-release-tools`**, which runs
   `python3 -m unittest discover -s tools/release -p 'test_*.py'`. The ci phase wires it
   into CI.

4. **Install migration.** `just install` and `scripts/install_all` call
   `cargo install --path`. Before that call, both detect a stale `bob-cli` entry in
   `cargo install --list --root "$root"`:
   - **Path or git source:** run `cargo uninstall --root "$root" bob-cli` with a
     one-line notice.
   - **Registry source:** that is the unrelated crates.io project. Stop with a clear
     message; never uninstall it.

   Keep the detection in one shell function, for example in `scripts/lib/bob_shell.sh`.
   Cover it with a test that feeds fixture `cargo install --list` outputs through a fake
   `cargo` on `PATH`, and wire that test into `just check-scripts`. Never add `--force`.

5. **Seed `CHANGELOG.md`.** Use a Keep a Changelog style header and a hand-written
   `## [0.2.0]` entry, a short "first versioned release" summary. Earlier builds all
   reported 0.1.0. Use a heading format that release-plz's changelog parser recognizes
   as version 0.2.0; the train phase re-verifies this with release-plz itself.
6. **Docs made false by this phase:**
   - **Install commands:** in `README.md`, `docs/start/install.md` and
     `docs/completion.md`, change the `cargo install --git … --locked bob-cli` lines to
     `<crate>`. Update the `bob 0.1.0` sample output to `bob 0.2.0` in
     `docs/start/install.md` and `docs/guides/install-everything.md`.
   - **`docs/reference/compatibility.md`:** replace "No minimum version is pinned here"
     with the 1.95 minimum and the `rust-toolchain.toml` pin.
   - **`docs/develop/migration-notes.md`:** add a "Package name" section. Explain:
     - the Cargo package is `<crate>` while the binaries keep their names;
     - `just install` uninstalls an old path or git `bob-cli` install automatically;
     - Git installs need `cargo uninstall bob-cli` once;
     - `cargo install bob-cli` installs an unrelated project.
   - **`docs/develop/release-checklist.md`:** add `just package-check` next to
     `just package-list`.
   - Do not use history wording ("renamed", "formerly", "no longer") in `start/`,
     `guides/` or `concepts/` pages; docs lint rejects it there.
7. **Verify:**
   - `cargo package --locked` with full verification succeeds.
   - `just package-check` passes; record the size in the bead note.
   - `cargo install --path . --locked --root <tmp>` installs all four executables
     without `--force`.
   - `just install-smoke` and `just check-scripts` pass, and the migration test is
     green.
   - `just check-release-tools`, `just check` and `just docs-check` pass.

## Phase ci: reusable per-push CI workflow brought green on GitHub

Goal: "all CI green" becomes a concrete, named set of checks that runs on every push and
on the exact SHA the train releases. A missing or skipped check is never green.

1. Add `.github/workflows/ci.yml` (name `CI`):
   - **Triggers:**
     - `push` to branches `master` and `ci/**`;
     - `pull_request`;
     - `workflow_dispatch`;
     - `workflow_call`, with inputs `sha` (string, default `''`) and `full` (boolean,
       default `false`).
   - **Checkout:** `ref: ${{ inputs.sha || github.sha }}`.
   - **Permissions:** top-level `contents: read`.
   - **Concurrency:** cancel superseded runs for `push` only. The group must never equal
     the train's `release` group: a reusable workflow evaluates `github.workflow` as the
     caller's, and docs-publish.yml documents the deadlock that colliding groups cause.
   - **Job `lint` (ubuntu-latest):** `just fmt`, `just lint`, `just check-scripts` and
     `just check-release-tools`. Install `just` with a pinned `taiki-e/install-action`.
   - **Job `test` (matrix `ubuntu-latest`, `macos-latest`):**
     - `astral-sh/setup-uv`, then `cargo test --locked --no-fail-fast`.
     - The macOS leg also runs `just lint`, so macOS-only code is clippy-checked the
       same way `just check` checks it on the Mac.
     - When `full` is true, the Linux leg first installs `zsh`, `pandoc`,
       `poppler-utils` and `texlive-xetex`, plus any fonts those tests need, so
       self-skipping tests actually run.
   - **Job `package` (ubuntu-latest):** `just package-check`, with full verification.
   - **All jobs:** `Swatinem/rust-cache`, job timeouts, and `CARGO_TERM_COLOR=always`.
     The toolchain comes from `rust-toolchain.toml`.
2. **Bring it green on real runners.** Push the work to a scratch branch
   `ci/<short-slug>` and watch the runs with `gh run list` and `gh run watch`. Fix
   failures until every job is green on both operating systems, including one
   `workflow_dispatch` run with `full=true` on the scratch branch. Fix macOS-only
   failures at their root cause in code or tests. Do not skip them, mark them ignored,
   or drop the macOS leg. If a fix grows beyond this phase, stop and record a
   `PROPOSED FOLLOW-UP:` note with the failing test and evidence. Do not land a red
   required job.
3. Record per-job wall times, for both the normal and the `full` run, in the bead note.
   docs-dev uses them.
4. Do not add `docs-check` to CI; the deploy repo's publish pipeline already gates the
   docs. Do not add branch protection here.

## Phase train: release-plz train in git_only rehearsal mode

Goal: one serialized workflow makes every release decision in order:

1. guard;
2. CI on the exact SHA;
3. `release-plz update`;
4. fast-forward push;
5. `release-plz release`;
6. optional binaries.

It needs no PAT and no `CARGO_REGISTRY_TOKEN`.

### 1. `release-plz.toml`

```toml
[workspace]
git_only = true            # rehearsal: tags + GitHub Releases only; never set publish = true alongside
semver_check = false       # src/lib.rs is an internal seam, not the product's contract
dependencies_update = false
publish_timeout = "30m"
release_commits = "^(feat|fix|perf|revert)(\\(.+\\))?!?:|^[a-z]+(\\(.+\\))?!:"
```

Add a `[changelog]` configuration with `commit_parsers`. Group breaking changes,
features, fixes, performance and reverts. Skip `chore(release)` and other `chore`,
`test`, `ci`, `style` and `build` commits. Choose deliberately whether `docs` and
`refactor` appear, and justify the choice in the bead note. Leave the tag name at the
default `v{{ version }}`, but state it explicitly in the file.

### 2. `tools/release/train_guard.py` (stdlib) and `tools/release/test_train_guard.py`

The guard decides `go` and a human-readable `reason`. It writes `go`, `reason`, `mode`
and `sha` to `$GITHUB_OUTPUT` and a short Markdown verdict to `$GITHUB_STEP_SUMMARY`.
The verdict links the matching docs page; see "Docs URLs emitted by the train" below.

- **Mode** comes from `release-plz.toml`: `git_only = true` is `rehearsal`, otherwise
  `live`.
- **Armed** only when the repository variable `RELEASES_PAUSED` is exactly `false`.
  Unset or any other value means paused and fails closed, so the train never runs until
  someone explicitly arms it.
- **Ref:** a non-dry-run release from any ref other than `refs/heads/master` is refused.
  A `push` event (the `ci/release/**` hook) is always a dry run.
- **Already released:** if the SHA already carries a `v*` tag, skip.
- **Last release time:**
  - **live:** the maximum `pubtime` over every line of the crates.io sparse index entry
    for `<crate>`, yanked versions included. Compute the sparse-index path with the
    general Cargo rules for 1-, 2- and 3-character names and for names of 4 or more
    characters. Send a descriptive `User-Agent`. A 404 or any lookup error fails the job
    (fail closed).
  - **rehearsal:** the newest GitHub Release `createdAt`, drafts and prereleases
    included, from `gh release list` or the REST API. Document this known gap: a run
    that created a tag but not its Release is not counted.
- **Daily ceiling:** compare the America/New_York calendar date of the last release with
  today's. On the same date, skip, unless an override is given.

> [!decision] hotfix_override With a non-empty `override_reason` dispatch input, a
> same-day release proceeds and emits a `::warning::` with the reason, which also
> appears in the summary. The pause, the ref check and the already-released check still
> apply.

> [!decision] hotfix_override = no There is no `override_reason` input. Same day always
> means skip.

Tests cover each case with an injected clock and injected fetch results:

- unarmed, paused and armed;
- no previous release;
- released yesterday;
- released today;
- override with and without a reason (if accepted);
- 23:30 ET yesterday versus 00:30 ET today;
- both 2026 DST transitions (2026-03-08 and 2026-11-01);
- a yanked newest version;
- index 404 and network error;
- tagged SHA;
- non-master ref;
- push-event dry run;
- `git_only` true, false and missing.

Add a consistency test that the pinned release-plz version in `release.yml` equals the
one `just release-preview` expects.

### 3. `.github/workflows/release.yml` (name `Release`)

- **Triggers:**
  - `schedule` at `17 8 * * *` and `47 10 * * *` (UTC). The second slot is the retry,
    and the guard makes it a no-op after a release.
  - `workflow_dispatch` with inputs `dry_run` (boolean) and, if accepted,
    `override_reason` (string).
  - `push` to `ci/release/**`, always a dry run.
- **Concurrency:** `concurrency: { group: release, cancel-in-progress: false }`.
- **Permissions:** top-level `permissions: { contents: read }`.
- **Job `plan`:** checks out `github.ref` with `fetch-depth: 0`, resolves the SHA, runs
  the guard and outputs `sha`, `go`, `mode` and `dry_run`.
- **Job `ci`:** `uses: ./.github/workflows/ci.yml` with `sha` and `full: true`. It runs
  when `go == 'true'` or in a dry run.
- **Job `release`** (real runs only):
  - Settings: `needs: [plan, ci]`, `environment: release`,
    `permissions: { contents: write, id-token: write }`. It checks out the planned SHA
    with `fetch-depth: 0` and installs release-plz pinned to `0.3.173` with
    `taiki-e/install-action`.
  - **Bump step:** run `release-plz update`. If the tree changed, commit as
    `github-actions[bot]` with the subject `chore(release): v<version>` and
    `git push origin HEAD:refs/heads/master`, never forcing. If `master` moved, the push
    is rejected, the job fails and nothing ships; the retry slot tests the new HEAD.
  - **Publish step:** runs whenever the bump step succeeded, even with no bump, so that
    a bump left over from a failed run still ships. It runs
    `release-plz release --git-token "$GITHUB_TOKEN" -o json`, outputs `tag` and
    `version`, and writes them to the summary.
  - Never set `CARGO_REGISTRY_TOKEN` and never add `rust-lang/crates-io-auth-action`. In
    live mode release-plz performs the Trusted Publishing OIDC exchange itself.
- **Job `preview`** (dry runs only): no environment, `contents: read`. It runs
  `release-plz update` and writes the next version, the new changelog section and
  `git diff --stat` to the summary. It never pushes, tags or publishes. Use
  `release-plz release --dry-run` only if it works in `git_only` mode without write
  credentials.

> [!decision] binaries Add a job `binaries` with `needs: [plan, release]` and
> `if: needs.release.outputs.tag != ''`:
>
> - It runs a matrix of `ubuntu-22.04` → `x86_64-unknown-linux-gnu` and `macos-latest` →
>   `aarch64-apple-darwin`.
> - It uses `taiki-e/upload-rust-binary-action` at the tag to upload all four binaries
>   to that GitHub Release, with `contents: write`.
>
> Add the matching `[package.metadata.binstall]` (`pkg-url`, `bin-dir`, `pkg-fmt`) to
> `Cargo.toml`, so `cargo binstall <crate>` resolves the archives once the crate is on
> crates.io. Skip Windows and Intel macOS.

> [!decision] binaries = no No `binaries` job and no binstall metadata.

### Docs URLs emitted by the train

These URLs are fixed now; docs-dev creates the pages and docs-user adds them to the URL
contract:

- `https://bobs-org.github.io/develop/releases/pause-and-resume-releases/`
- `https://bobs-org.github.io/develop/releases/recover-a-failed-release/`
- `https://bobs-org.github.io/develop/releases/reference/#daily-guard`

### 4. `just release-preview`

This recipe has no side effects outside a temp dir:

1. Clone `origin/master` into a `mktemp -d` directory.
2. Require `release-plz` at the pinned version. If it is missing or a different version,
   print the exact `cargo binstall` / `cargo install --locked` command and exit.
3. Run `release-plz update` in the clone.
4. Print the next version (or "no release: nothing user-visible since vX"), the new
   changelog section and `git diff --stat`.
5. Run the guard in an explain-only mode against live data, to show what the next
   scheduled run would decide. Degrade gracefully without `gh` auth.

Extend `just check-release-tools` with the guard tests.

### 5. Validate

- Run the unit tests and `actionlint` over both workflows. Use
  `uvx --from actionlint-py actionlint` or a pinned release binary.
- **release-plz in scratch clones**, with release-plz installed locally at the pinned
  version:
  1. Current HEAD: `update` makes no changes, because the seeded 0.2.0 entry matches. If
     it rewrites the changelog, fix the seed format.
  2. Simulate a released tag `v0.2.0` plus a `feat:` and a `docs:` commit: `update`
     produces 0.2.1 with the expected sections and no `chore(release)` entry.
  3. `docs:` commits only: no bump.
  4. A `feat!:` commit: 0.3.0. Confirm against the source what release-plz does for 0.x;
     the report says breaking changes bump the minor version and features bump the patch
     version.

  Record the observed results in the bead note; they become reference facts.

- Push a `ci/release/<slug>` scratch branch to run the whole train as a forced dry run
  on GitHub: plan → CI (full) → preview. It must succeed and show the expected guard
  verdict, which is "unarmed" until the github phase. Delete the branch afterwards.

## Phase github: GitHub setup and first rehearsal release

Goal: the train runs on `master` for real in rehearsal mode, and its guard is observed
working.

1. Confirm that CI is green on the current `master` HEAD
   (`gh run list -w ci.yml -b master`). If it is red, stop and record why.
2. Dispatch a dry run: `gh workflow run release.yml -f dry_run=true`, then
   `gh run watch`. Expect the guard verdict "unarmed", green CI, and a preview showing
   an initial v0.2.0 release with the seeded notes.

> [!decision] github_apply 3. **Create the `release` environment** with a deployment
> branch policy limited to `master`: `PUT /repos/bobs-org/bob-cli/environments/release`
> with custom branch policies, then add a `master` branch policy. No required
> reviewers. 4. **Add two active rulesets**, with no bypass actors:
>
> - branch `~DEFAULT_BRANCH`: block `deletion` and `non_fast_forward`;
> - tags `refs/tags/v*`: block `deletion` and `update`.
>
> They must not block the bot's fast-forward push or its tag creation; step 6 proves it.
> Never require PRs or status checks on `master`, because agents push directly. 5. **Arm
> the train:** `gh variable set RELEASES_PAUSED --body false -R bobs-org/bob-cli`. 6.
> **Dispatch the first real rehearsal:** `gh workflow run release.yml`, then verify:
>
> - tag `v0.2.0` exists;
> - a GitHub Release v0.2.0 carries the seeded notes;
> - there is no bump commit, because this is an initial release;
> - with `binaries`, archives for both targets are attached.
>
> 7. **Dispatch again the same New York day:** expect a "skip" verdict (already released
>    today, or the SHA is already tagged) and no new tag or Release.
> 8. **If a run fails because of a workflow bug:**
>    1. fix it in code;
>    2. prove the fix with a `ci/release/<slug>` dry run;
>    3. land it through the normal flow;
>    4. write `PENDING REDISPATCH:` in the bead note, so the handoff phase re-runs steps
>       6 and 7 on the fixed `master`.
>
>    A rehearsal tag or Release left inconsistent may be deleted (temporarily disable
>    the tag ruleset). Record every such action.
>
> If any `gh api` call is refused (for example a 403), record the exact command for the
> handoff report and continue with what is possible.

> [!decision] github_apply = no Stop after step 2. The train stays unarmed, because
> `RELEASES_PAUSED` is unset. Write the exact commands for steps 3–7 into the bead note;
> the handoff report gives them to Bryan.

Record every run URL and outcome in the bead note.

## Phase docs-dev: maintainer release documentation and decision record

Goal: excellent maintainer documentation for the release process, organized by Diátaxis
inside the Develop audience section. The four types are nested under one audience, the
way `develop/plugins/` and `develop/mac-capture/` are.

**Before you write:**

- Read `docs/develop/writing-docs.md` and the templates in `docs/_templates/`.
- Classify each page with the Diátaxis compass (action or cognition, acquisition or
  application).
- Keep each page to one type and one purpose, and do not create empty structure.
- Document only behavior that exists in the landed `release.yml`, `ci.yml`,
  `release-plz.toml`, `tools/release/` and justfile. Check every command and flag
  against the files and `--help`, and use the facts recorded in the ci, train and github
  bead notes (`sase bead show`).
- Develop pages may cite repo paths. Do not put bead IDs in reader-facing pages.

Create these pages, using these slugs; the train already links three of them.

| Path under `docs/develop/`                        | Type        | Title and content                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `releases/index.md`                               | hub         | "Releases": one-sentence orientation, then links grouped Learn / Do / Understand / Look up                                                                                                                                                                           |
| `releases/preview-the-next-release.md`            | tutorial    | "Tutorial: Preview the next release". A safe, side-effect-free path: `just release-preview`, `just package-check`, then read the guard verdict and the changelog section tomorrow's train would produce. Ends by naming what the reader learned and where to go next |
| `releases/how-the-release-train-works.md`         | explanation | "How the release train works". See the outline after this table                                                                                                                                                                                                      |
| `releases/pause-and-resume-releases.md`           | how-to      | Pause, resume and check the state: `RELEASES_PAUSED`, unarmed versus paused                                                                                                                                                                                          |
| `releases/ship-a-hotfix.md`                       | how-to      | Ship a fix the same day. With `hotfix_override`: dispatch with a reason. Without it: the retry slot if nothing shipped today, otherwise tomorrow, plus withdrawing the bad version                                                                                   |
| `releases/withdraw-a-bad-release.md`              | how-to      | Live: `cargo yank --version X.Y.Z`; yanking never resets the guard. Rehearsal: mark or edit the GitHub Release; never move or delete tags                                                                                                                            |
| `releases/recover-a-failed-release.md`            | how-to      | One section per symptom: CI red; push rejected; bump pushed but release or publish failed; published but tag or Release missing (create both by hand for that version at the release commit; the guard blocks a second publish); guard lookup failed                 |
| `releases/publish-to-crates-io.md`                | how-to      | "Publish the first crates.io release". See the steps after this table                                                                                                                                                                                                |
| `releases/keep-the-package-under-the-size-cap.md` | how-to      | Read `just package-check`, trim `include`, and escalate to splitting the embedded assets into a separate `<crate>-assets` crate. Never use `publish_no_verify`                                                                                                       |
| `releases/update-the-rust-toolchain.md`           | how-to      | Bump `rust-toolchain.toml`, `rust-version` and the compatibility page together, then confirm on a `ci/**` branch                                                                                                                                                     |
| `releases/test-release-workflow-changes.md`       | how-to      | Use `ci/**` and `ci/release/**` scratch branches and dry-run dispatches to test workflow edits before landing                                                                                                                                                        |
| `releases/reference.md`                           | reference   | "Release train reference". See the list after this table                                                                                                                                                                                                             |
| `ci.md`                                           | reference   | "bob-cli CI". See the list after this table                                                                                                                                                                                                                          |

**Outline for "How the release train works":**

- What a train day looks like, with one mermaid diagram.
- Why a scheduled train, rather than the quickstart, a release PR, or a cron "cap".
- What "once a day" and "green" mean precisely.
- Why the guard reads registry `pubtime`.
- Why the bump commit is safe even though CI ran on its parent.
- Rehearsal versus live mode.
- Trusted Publishing.
- The one-way doors of crates.io: permanence, the name, the size cliff.
- Publishing is not deploying: hosts and companion repos track HEAD.
- Costs: the push race, the ~20-minute gate, lag behind HEAD.
- When to revisit: about one month after go-live, review downloads.
- A link to the decision record.

**Steps for "Publish the first crates.io release":**

1. Pause the train.
2. Re-check that the name is free.
3. Create a crates.io token scoped `publish-new` + `publish-update` to `<crate>`.
4. From a clean clone, check out the latest rehearsal tag and run
   `cargo publish --locked --dry-run`, then `cargo publish --locked`.
5. Revoke the token.
6. Add a Trusted Publisher: owner `bobs-org`, repo `bob-cli`, workflow `release.yml`,
   environment `release`.
7. Land `git_only = false`; the guard switches to registry mode by itself.
8. Resume the train.
9. Watch two automated releases, then enable Trusted-Publishing-only mode.
10. Switch the reader install docs to `cargo install <crate> --locked` (list the files).

**Contents of "Release train reference":** one table or list per topic, with
`## Daily guard` as a stable heading.

- schedule (UTC, EDT, EST);
- triggers and dispatch inputs;
- repository variable and environment;
- the job graph and per-job permissions;
- the guard's decision table (inputs → verdict → summary text);
- the data source per mode;
- `release-plz.toml` keys and values;
- version bump rules on 0.x, with the observed results;
- the commits that trigger a release;
- changelog sections;
- tag and Release naming;
- the files and identity of the bot commit;
- pinned tool versions;
- size thresholds;
- scratch-branch hooks;
- binaries targets and archive names, if accepted.

**Contents of "bob-cli CI":**

- triggers;
- jobs and OS legs, each with its local `just` equivalent;
- what `full` adds;
- hermetic-test rules for contributors: tests must not depend on the developer's `HOME`
  config, `PATH` tools, locale or `TZ`, with how to give a test its own;
- the measured job times;
- running CI on a `ci/**` branch.

Also:

- **Navigation:** add a Develop > Releases nav group to `mkdocs.yml` in Diátaxis order:
  hub, tutorial, explanation, how-tos, reference. Add Develop > CI. Link both from
  `docs/develop/index.md`.
- **Decision record.**

  > [!decision] memory_release_train Use `/sase_memory_write`, then add
  > `sase/memory/decisions/bob-cli-releases-ride-a-daily-train.md`, following the
  > existing records' frontmatter and their Applies to / Claim / Why / Cost / Reopens
  > when body.
  >
  > The roster `summary` must carry the agent rule in one sentence: bob-cli ships only
  > through the scheduled release train, at most one version per New York day after full
  > CI on the exact SHA, with no release PR. Agents never edit `Cargo.toml` `version` or
  > `CHANGELOG.md`, and they mark CLI or JSON contract breaks with `!`.
  >
  > The body covers:
  >
  > - **Why:** cite the research report and name the rejected alternatives: the
  >   quickstart, a release PR with a cron merge, an hourly guarded publisher,
  >   cargo-semver-checks and cargo-dist.
  > - **Cost:** the push race, lag behind HEAD, and permanence once live.
  > - **Reopens when:** the repo moves to PRs, crates.io shows no outside users after a
  >   month, or the size cap forces the assets split.
  >
  > Then run `sase memory init`. Never hand-edit `AGENTS.md` or `CLAUDE.md`.

  > [!decision] memory_release_train = no Write no memory. Record the proposed rule in a
  > `PROPOSED FOLLOW-UP:` note instead.

- **Generate and check:** run `just docs-gen`, which regenerates
  `docs/develop/decisions.md` among other files, then `just docs-check`. Respect the
  Develop budget of 8,000 words per page.

## Phase docs-user: reader-facing release documentation and site integration

Goal: people who install Bob can find out what a release is and install one. The site
also links the new material from the right doors.

1. **`docs/reference/releases.md`**, "Releases and versions" (reference):
   - where releases appear: GitHub Releases, `vX.Y.Z` tags, `CHANGELOG.md`, and
     crates.io status (state plainly that it is not yet published, and what to use
     instead);
   - cadence: at most one release per New York day, and only when something user-visible
     landed;
   - 0.x version rules, and what a `!` breaking marker means for CLI and JSON consumers;
   - package name versus binary names, and the fact that `cargo install bob-cli`
     installs an unrelated project;
   - how to check your version;
   - compatibility with Mac Capture and the plugins: they track bob HEAD, and the JSON
     contract grows additively;
   - prebuilt archives, if `binaries` was accepted.
2. **`docs/guides/install-a-tagged-release.md`**, "Install a specific Bob release"
   (how-to, `tags: [CLI]`):
   - Before you start, then steps:
     `cargo install --git https://github.com/bobs-org/bob-cli.git --tag vX.Y.Z --locked <crate>`
     followed by `bob completion install`. With `binaries`, also cover downloading the
     prebuilt archive.
   - Confirm with `bob --version`.
   - Add one "If it goes wrong" section for the
     `binary bob already exists … as part of bob-cli` error, worded without history
     language.
   - End with Related links.
3. **Hubs and nav:**
   - Add Reference > "Releases and versions" and Guides > Set up > "Install a specific
     Bob release" to `mkdocs.yml`.
   - Update `docs/reference/index.md` and `docs/guides/index.md`.
   - Add a Next link from `docs/start/install.md` to the new guide. Do not add
     alternatives inside the tutorial's steps.
4. **`docs/develop/release-checklist.md`:** keep the URL and title. Reframe it as the
   manual smoke checks that complement the train (link the Releases hub), and make it
   consistent with `just package-check` and the train's automated gate.
5. **`docs/_meta/url_contract.txt`:** add the three train-emitted URLs under a comment
   naming `.github/workflows/release.yml`.
6. Keep the README a short front door; change only what is now false. Run
   `just docs-gen` if nav or generated blocks changed, then `just docs-check`, which
   covers lint, the strict build, links, the URL contract and budgets.
7. **Fresh-eyes check:** build the site and confirm each Diátaxis door leads a reader to
   the new pages in one click from the relevant hub: Learn → tutorial, Do → how-tos,
   Understand → explanation, Look up → references. Fix labels that mislead.

## Phase handoff: end-state verification and Bryan's go-live report

Goal: confirm what landed actually works, and hand Bryan a precise list of what only he
can do. The epic lander closes the epic bead whatever the report says.

1. **Verify:**
   - CI is green on `master` HEAD.
   - The train's latest scheduled or dispatched runs show sensible verdicts.
   - If the github bead note contains `PENDING REDISPATCH:` and `github_apply` was
     accepted, re-run that phase's steps 6–7 now.
   - `just docs-build` contains every new page, and the release reference matches the
     landed `release.yml` and `release-plz.toml`; fix drift.
   - If the deploy repo has published since docs-user landed, spot-check the live URLs.
2. **Write the report.** Create `crates_io_release_train_handoff.md` in a temp dir
   outside the repo. Register it with `/sase_artifact`, and add a note on the epic bead
   that names the artifact reference. Contents:
   - **Status:** what is done, with links to runs, the v0.2.0 Release and the docs
     pages, plus the approved decision values and any crate-name substitution.
   - **Now (if `github_apply` was declined or partly refused):** the exact `gh` commands
     to create the environment and rulesets and to arm the train.
   - **Rehearsal (1–2 weeks):** what to watch:
     - version math;
     - changelog grouping;
     - that no second release happens in one day;
     - how often an agent push races the gate;
     - the CI red rate, which predicts skipped days.
   - **Go live:** the ordered steps, linking "Publish the first crates.io release". Note
     that the name must be re-checked right before publishing, and that the first
     publish is manual because Trusted Publishing cannot create a crate.
   - **After go-live:**
     - Trusted-Publishing-only mode after two automated releases;
     - ask an agent to switch the reader install docs to crates.io (list the files);
     - a one-month review of downloads (if they are essentially Bryan's own, drop
       crates.io to weekly or occasional and keep daily GitHub Releases);
     - the size watch, with the assets-crate split when the 8 MiB warning trips.
   - **Deliberately not done:** release PRs, cargo-semver-checks, cargo-dist, Windows
     and Intel macOS binaries, automatic host upgrades and companion-repo releases.
3. Record `PROPOSED FOLLOW-UP:` notes on this phase bead for the post-go-live docs
   switch and for the assets-crate split. Phase workers never create task beads
   directly.

## Out of scope

- The manual first `cargo publish` and the crates.io-side settings (Trusted Publisher,
  Trusted-Publishing-only mode). Both need Bryan's crates.io account; the handoff report
  covers them.
- A release-PR flow, cargo-semver-checks, cargo-dist, Windows binaries, splitting the
  assets into a separate crate, automatic upgrades of Bryan's hosts, and releases of
  bob-plugins or Bob Mac Capture.
