---
tier: tale
size: medium
title:
  Fix the release train's binaries path, attach v0.2.0 archives, and land epic
  bob-cli-ac
goal:
  Every bob-cli release attaches working Linux x86_64 and macOS arm64 archives built
  from its own tag. v0.2.0 carries them now, cargo-binstall metadata resolves them, a
  failed upload can be retried for an existing tag, and the release-train epic
  bob-cli-ac is closed.
proposed_by: bbugyi200.apollo.bob-cli-ac.land
proposed_with: claude/opus
bead: bob-cli-ac
status: done
---

- **PARENT:**
  [202610/crates_io_release_train.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/crates_io_release_train.md)
- **BEAD:**
  [bob-cli-ac](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ac/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ac.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ac.land.md)
- **COMMITS:**
  - [9c1f9c6](https://github.com/bobs-org/bob-cli--plans/commit/9c1f9c6286fdec33d682f0e8cc35fea39afc2802)
    — docs(plan): mark crates_io_release_train epic as done

# Plan: fix the release binaries path and land epic bob-cli-ac

## Context

This tale holds the remaining work found while landing epic **bob-cli-ac** ("Daily
release train for bob-cli"). Its eight phase beads are closed and their notes check out:
the hermetic tests, the `bob-obsidian` package, green CI on Linux and macOS, the
release-plz `git_only` train, the GitHub environment, rulesets and arming, and the docs
and decision record. The first rehearsal (run 39540723503) created tag and GitHub
Release **v0.2.0**, but both `binaries` legs failed. The `ref:` fix in commit 1257c2e8
is not enough. Read the epic's two `DISCOVERED ISSUE` / `LAND TRIAGE` notes first:

```bash
sase bead read bob-cli-ac -r "Need the landing notes and the binaries defects before fixing release.yml"
```

The verified defects:

1. **The archive name is rejected.** `taiki-e/upload-rust-binary-action` v1.30.2
   (`main.sh` lines 169–170) bails "when multiple binary names are specified, default
   archive name or
   '$bin' variable cannot be used in 'archive' option".
   `.github/workflows/release.yml` passes four bins and leaves `archive` at its default
   `$bin-$target`. Every leg therefore fails even with the `ref:`
   fix.
2. **The binstall metadata cannot render.** `Cargo.toml`
   `[package.metadata.binstall].pkg-url` uses `{ bin }`, but cargo-binstall's `pkg-url`
   context has no `bin` key (`binstalk-fetchers/src/gh_crate_meta.rs`,
   `Context::get_value`). `{ bin }` is valid only in `bin-dir`, so the URL never
   renders.
3. **Binaries are built from the wrong commit.** The `binaries` job checks out
   `needs.plan.outputs.sha`, the pre-bump commit. release-plz tags the
   `chore(release): vX.Y.Z` bump commit, so a v0.2.1 archive would print `bob 0.2.0`.
4. **The release tag comes from the wrong source.** The publish step computes
   `TAG=v<Cargo.toml version>` instead of reading release-plz's output. That output is
   `{"releases":[{"package_name":"bob-obsidian","prs":[],"tag":"v0.2.0","version":"0.2.0"}]}`
   when something shipped and has an empty `releases` list otherwise. On a GO day with
   nothing user-visible, the step reports a fake release and re-runs binaries against
   the previous tag. The action uploads with `gh release upload --clobber`.
5. **A failed release-plz run can pass.** The publish step pipes
   `release-plz release … | tee`, and the default Actions shell is `bash -e` without
   `pipefail`, so a failed `release-plz release` can still pass the step.
6. **There is no recovery path.** Nothing can rebuild and upload archives for an
   existing tag, which v0.2.0 needs now.
7. **Override reasons are interpolated into shell.** The plan job interpolates
   `${{ inputs.override_reason }}` into the bash script, so a hotfix reason containing
   backticks or `$( )` is executed.

**Timing.** Today's scheduled slots (08:17 and 10:47 UTC, 2026-10-21) skip on the
same-day ceiling. The 2026-10-22 08:17 UTC slot would hit defects 1, 3 and 4, so this
should land today.

**Rules carried over from the epic.**

- Land only through the normal SASE commit flow and never push `master` by hand.
- Scratch branches `ci/<slug>` and `ci/release/<slug>` may be pushed to `origin` and
  must be deleted afterwards.
- Pin any new third-party action to a full SHA with a `# vX.Y.Z` comment, and reuse the
  pins already in `release.yml`.
- `tools/release/` is stdlib-only Python (≥ 3.11), with offline, injectable tests.
- Never edit `Cargo.toml` `version` or `CHANGELOG.md`; the train owns them. Metadata
  tables are fine.
- Read `sase memory read docs.md -r "<why>"` before touching `docs/`.

**Recommended commit type:** `ci(release): …`. It is release tooling, not a user-visible
change, so it does not trigger a release by itself.

## Step 1: Reusable binaries workflow

Add `.github/workflows/release-binaries.yml` (name `Release binaries`). Its only trigger
is `workflow_call`, with one required string input `tag`. Use top-level
`permissions: { contents: read }` and declare **no** `concurrency`: a reusable workflow
evaluates `github.workflow` as its caller's, and the caller's `release` group already
serializes. Add `defaults: run: shell: bash` so `pipefail` is on.

Job `binaries`:

- Matrix with `fail-fast: false`: `ubuntu-22.04` → `x86_64-unknown-linux-gnu` and
  `macos-latest` → `aarch64-apple-darwin`.
- `permissions: { contents: write }` and `timeout-minutes: 45`.
- Steps:
  1. Validate the tag. Pass it through `env:`, never inline `${{ }}` in the script.
     Require `^v[0-9]+\.[0-9]+\.[0-9]+$`, then require that a GitHub Release exists with
     `gh release view "$TAG" -R "$GITHUB_REPOSITORY"` (`GH_TOKEN: ${{ github.token }}`).
     Fail with a clear message otherwise.
  2. Check out `ref: refs/tags/${{ inputs.tag }}`, so the archives are built from
     exactly the tagged source, including the bump commit's version.
  3. Run `taiki-e/upload-rust-binary-action` at the existing pin
     `f0d45ae91ee7b8ee928de7a9d04d893a08bcbec6 # v1.30.2` with:
     - `bin: bob,bob_pomodoro,bob_notify,tmux_bob_pomodoro`
     - `archive: bob-$target`: a literal `bob`, not `$bin`. This keeps the documented
       names `bob-x86_64-unknown-linux-gnu.tar.gz` and `bob-aarch64-apple-darwin.tar.gz`
       with all four binaries at the archive root.
     - `target: ${{ matrix.target }}`
     - `tar: unix`
     - `locked: true`
     - `ref: refs/tags/${{ inputs.tag }}`

## Step 2: release.yml changes

- Add top-level `defaults: run: shell: bash` (`pipefail`). The preview step's pipes
  already end in `|| true`.
- Add a `workflow_dispatch` input `binaries_tag` (string, default `""`), described as:
  "Rebuild and upload the Linux and macOS archives for an existing vX.Y.Z Release; skips
  the guard, CI and release, and ignores the other inputs."
- Gate the `plan` job with `if: ${{ !inputs.binaries_tag }}`. Schedule and push events
  have no inputs, so the job still runs for them; step 5 proves this for push. Every
  other train job already depends on `plan` and is skipped with it.
- In the plan job, pass `override_reason` and `dry_run` through `env:` (for example
  `OVERRIDE_REASON: ${{ inputs.override_reason }}`) and reference `"$OVERRIDE_REASON"`
  in the script. No `${{ inputs.* }}` should remain inside any `run:` body.
- Publish step: keep `release-plz release --git-token "$GITHUB_TOKEN" -o json`, writing
  stdout to a file, and derive `tag` and `version` **from that JSON**. Use the release
  entry whose `package_name` is `bob-obsidian`. Both outputs are empty when nothing was
  released. Do the parsing in a new stdlib helper, `tools/release/release_outputs.py`.
  The helper:
  - accepts the JSON file path, and preferably the package name and mode as flags;
  - tolerates non-JSON log lines by parsing the last line that starts with `{`;
  - exits nonzero on missing or malformed output;
  - writes `tag=` and `version=` to `$GITHUB_OUTPUT` when that is set;
  - writes the step summary: `### Release: vX.Y.Z` with version, tag and mode, or "No
    release: release-plz found nothing new to release" when the list is empty.
- Replace the inline `binaries` job with a caller job. Keep the name `binaries`, with
  `needs: [release]`, `if: needs.release.outputs.tag != ''`,
  `permissions: { contents: write }`, `uses: ./.github/workflows/release-binaries.yml`
  and `with: { tag: ${{ needs.release.outputs.tag }} }`.
- Add a caller job `binaries-retry` with `if: ${{ inputs.binaries_tag != '' }}`, no
  `needs`, `permissions: { contents: write }`, the same `uses:`, and
  `with: { tag: ${{ inputs.binaries_tag }} }`.
- Update the header comment to name the new input, the reusable workflow, and why
  archives build from the tag.

## Step 3: Cargo binstall metadata and tests

- `Cargo.toml` `[package.metadata.binstall]`: set
  `pkg-url = "{ repo }/releases/download/v{ version }/bob-{ target }.tar.gz"`. Keep
  `bin-dir = "{ bin }{ binary-ext }"` and `pkg-fmt = "tgz"`. Do not touch `version`.
- Add `tools/release/test_release_outputs.py`, covering:
  - one release;
  - an empty `releases` list (empty outputs, plus the "No release" summary);
  - leading log lines before the JSON;
  - a release for a different package name only;
  - malformed or missing output (nonzero exit);
  - `$GITHUB_OUTPUT` and `$GITHUB_STEP_SUMMARY` written to temp files.
- Add a stdlib contract test, for example `tools/release/test_binaries_contract.py`. It
  reads `Cargo.toml` with `tomllib` and `release-binaries.yml` as text (regex is fine;
  no PyYAML). It asserts that:
  - the binstall `pkg-url` contains no `{ bin }`;
  - the `archive:` value contains no `$bin`, because more than one bin is listed;
  - for each matrix target, the rendered `pkg-url` file name equals
    `<rendered archive>.tar.gz`;
  - the bins listed in the action match the four `[[bin]]` targets in `Cargo.toml`.
- `just check-release-tools` discovers `test_*.py` automatically, so no justfile change
  is needed.

## Step 4: Docs

Read `docs.md` memory first. Change only what is now false or missing.

- `docs/develop/releases/reference.md`:
  - **Triggers table:** add the `binaries_tag` input.
  - **Job graph:** `binaries` and `binaries-retry` call `release-binaries.yml`. They run
    only when release-plz's JSON reported a release (or a `binaries_tag` was given), and
    they build from the release tag with `--locked`.
  - **Binaries section:** add that each archive holds all four binaries at the root.
    Correct the binstall pattern line to
    `{ repo }/releases/download/v{ version }/bob-{ target }.tar.gz`.
  - **Pinned-SHA sentence:** add `release-binaries.yml`.
- `docs/develop/releases/recover-a-failed-release.md`: add a symptom section, for
  example "## The Release is missing its archives":
  `gh workflow run release.yml -R bobs-org/bob-cli -f binaries_tag=vX.Y.Z`, then confirm
  with `gh release view vX.Y.Z -R bobs-org/bob-cli --json assets`. Note that it rebuilds
  from the tag, replaces existing archives of the same name, and skips the guard and CI.
- `docs/develop/releases/test-release-workflow-changes.md`: its "If it goes wrong" says
  a manual dispatch cannot test an unregistered definition. If step 5 proves that
  `gh workflow run release.yml --ref <scratch branch>` runs the scratch branch's
  definition, refine the text. A dispatch with `--ref` runs that branch's version of an
  already registered workflow (`ci.yml`, `release.yml`); only a brand-new workflow file
  needs a push trigger first.
- Leave the reader pages alone; their archive names stay correct:
  - `docs/reference/releases.md`
  - `docs/guides/install-a-tagged-release.md`
- No new URLs, so `docs/_meta/url_contract.txt` is unchanged.
- Run `just docs-gen` if generated blocks changed, then `just docs-check`.

## Step 5: Verify

**Local.**

- Run `just check-release-tools`, then
  `uvx --from actionlint-py actionlint .github/workflows/ci.yml .github/workflows/release.yml .github/workflows/release-binaries.yml`.
- Run `just package-check --no-verify` for the `Cargo.toml` metadata change.
- Run `just check`.

**Same-day guard (closes the epic's unverified github-phase step 7).**

- If a scheduled Release run already happened today after 08:17 UTC, read its verdict.
- Otherwise dispatch `gh workflow run release.yml -R bobs-org/bob-cli` on master with no
  inputs and watch it. Expect the plan job's SKIP verdict ("already released today"),
  skipped `ci`, `release` and `binaries` jobs, and no new tag or Release
  (`gh release list -R bobs-org/bob-cli` still shows only v0.2.0).

**Scratch branch.** Do not move the workspace branch. Build a scratch commit from the
working tree and push it:

```bash
git add -A
TREE=$(git write-tree)
C=$(git commit-tree "$TREE" -p HEAD -m "ci(release): scratch test of the binaries path")
git reset -q
git push origin "$C:refs/heads/ci/release/binaries-retry"
```

That one push triggers `ci.yml` (normal CI) and a forced dry run of `release.yml`.

- Watch the dry run with `gh run watch <id> -R bobs-org/bob-cli --exit-status` and a
  long timeout. Confirm the push event still runs `plan` (the `binaries_tag` gate holds
  for events without inputs). Then confirm `ci` (full) and `preview` succeed, and that
  `release`, `binaries` and `binaries-retry` are skipped.
- Dispatch the retry on the scratch ref:
  `gh workflow run release.yml -R bobs-org/bob-cli --ref ci/release/binaries-retry -f binaries_tag=v0.2.0`.
  It queues behind the dry run in the `release` group. Watch it to completion. Only the
  two `binaries-retry` legs should run.
- Confirm `gh release view v0.2.0 -R bobs-org/bob-cli --json assets` lists
  `bob-x86_64-unknown-linux-gnu.tar.gz` and `bob-aarch64-apple-darwin.tar.gz`.
- Download the Linux archive into a new empty temp dir:
  - `tar -tzf` must list exactly `bob`, `bob_pomodoro`, `bob_notify` and
    `tmux_bob_pomodoro`;
  - the extracted `./bob --version` must print `bob 0.2.0`.
- Optional: check that the binstall metadata resolves against the real Release. Install
  cargo-binstall into a temp root
  (`cargo install --locked cargo-binstall --root <tmp>`), then run it with
  `--manifest-path Cargo.toml --dry-run --no-confirm`, a temp `--root`, and
  `--disable-strategies quick-install,compile` for `bob-obsidian`. It should pick the
  v0.2.0 GitHub Release archive. Record it as skipped if it is impractical.
- Delete the branch: `git push origin --delete ci/release/binaries-retry`.
- Record every run URL and outcome in a note on bob-cli-ac.

**If the scratch-ref dispatch is refused.** For example, GitHub rejects `binaries_tag`
as an unexpected input because it validates against master. Then record the exact
post-landing command
(`gh workflow run release.yml -R bobs-org/bob-cli -f binaries_tag=v0.2.0`) in the epic
note and in the report in step 6, and continue. The next real release also exercises the
main path.

## Step 6: Revised handoff report

The existing handoff artifact `file:explicit:69aedc409000fc76103efc6e` tells Bryan to
re-dispatch with `override_reason`. After this fix, that would not attach binaries.
Write `crates_io_release_train_handoff_v2.md` in a temp dir outside the repo and
register it with `/sase_artifact`, attached to bob-cli-ac. Contents:

- **Status:**
  - v0.2.0 Release and its archives, with run URLs;
  - CI green on master;
  - the train armed (`RELEASES_PAUSED=false`), the `release` environment, and rulesets
    25521281 and 25521282;
  - the docs pages under Develop > Releases;
  - the decisions as implemented, and no crate-name substitution (`bob-obsidian`).
- **What changed since report v1:** the binaries defects and their fixes, and the new
  `binaries_tag` recovery path.
- **Rehearsal (1–2 weeks), what to watch:**
  - version math;
  - changelog grouping;
  - no second release per New York day;
  - how often an agent push races the gate;
  - the CI red rate;
  - archives on each Release, with `bob --version` matching the tag.
- **Go live:** follow `docs/develop/releases/publish-to-crates-io.md` in order. Re-check
  that the name is free right before publishing. The first `cargo publish` is manual,
  because Trusted Publishing cannot create a crate. Then add the Trusted Publisher
  (`bobs-org/bob-cli`, `release.yml`, environment `release`) and land
  `git_only = false`.
- **After go-live:**
  - switch to Trusted-Publishing-only mode after two automated releases;
  - the reader install-docs switch (task bob-cli-ah, snoozed until then);
  - a one-month review of downloads (if they are essentially Bryan's own, drop crates.io
    to weekly or occasional and keep daily GitHub Releases);
  - the size watch (task bob-cli-ai; the crate is ~7.79 MiB against the 8 MiB warning).
- **Deliberately not done:**
  - release PRs;
  - cargo-semver-checks and cargo-dist;
  - Windows and Intel macOS binaries;
  - automatic host upgrades;
  - companion-repo releases.

Add a note on bob-cli-ac naming the new artifact ref and saying it supersedes v1.

## Step 7: Close out epic bob-cli-ac (final step; do it in this turn)

1. Run `sase bead epic-symbols bob-cli-ac`. It listed no entries at planning time. For
   any entry now listed, resolve the symbol (wire it up, privatize it, add a non-test
   pragma, or delete it). Re-key the Justfile line only if a still-open later bead needs
   the exemption.
2. Close the epic. Complete the note with the actual run IDs and outcomes from step 5
   and the artifact ref from step 6:

   ```bash
   sase bead close bob-cli-ac --note "Verified all 8 phases closed with notes addressed and DECISIONS honored. crate_name: package bob-obsidian, lib bob_cli, version 0.2.0, include trimmed (~7.79 MiB). github_apply: release environment, rulesets 25521281/25521282, RELEASES_PAUSED=false. hotfix_override: input implemented. binaries: job and binstall metadata. memory: decisions:bob-cli-releases-ride-a-daily-train. Epic commits 9a0fb957 eabd2915 1d1b8fec 104cee99 fc6ffdc3 8789c12d ea7f5f0b 1257c2e8 plus this landing fix. CI green on master. Tag/Release v0.2.0 created by train run 39540723503. Landing fixed the binaries path: literal bob-\$target archive, build from the release tag, tag from release-plz JSON, pipefail, binstall pkg-url without { bin }, override_reason via env, binaries_tag retry through release-binaries.yml. Archives attached to v0.2.0 (run <id>); same-day SKIP verified (run <id>); scratch dry run green (run <id>). Integration: 062fc3da needed no change; bob-cli-9z closed superseded by bob-cli-ac.1; chezmoi and bob-mac-capture have no stale bob-cli install commands. Follow-ups: .8 #1 folded into this landing, #2 -> bob-cli-ah (snoozed), #3 -> bob-cli-ai. Handoff report v2: <artifact ref>."
   ```

   If the close is rejected for leftover `--epic-symbol` entries, clean them up and
   close again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` if the recipe exists (`just --list | grep -w symvision`). At
   planning time it did not; if it still does not, say so in the final response.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file, `plan:202610/crates_io_release_train.md`. Take its path from the PLAN line of
   `sase bead read bob-cli-ac`. That file lives in the plans repo, so `/sase_final` will
   list it as a repository obligation to commit.
5. bob-cli-ac has no `parent_bead`, so nothing further is needed upward.
6. Finish with `/sase_final`.
