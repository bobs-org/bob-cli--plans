---
tier: tale
size: medium
title: "bob plugins sync: deploy the enclosing checkout and fetch on pull failure"
goal:
  "Remove the two avoidable frictions agents hit when deploying plugins from a SASE
  linked worktree: a bare `bob plugins sync` run inside a bob-plugins checkout should
  deploy that checkout instead of refusing, and a failed repo-refresh `git pull` should
  fall back to `git fetch` so deploy-guard remedies can name the landed build."
decisions:
  cwd_scope:
    ask:
      Should enclosing-checkout repo resolution apply to the whole `bob plugins` family?
    choices:
      family:
        list, bare default, and sync all resolve the same repo from the cwd, so they
        never disagree
      sync_only:
        only sync resolves from the cwd; list keeps env/default resolution and may name
        a different repo
    default: family
    why:
      list and sync disagreeing about the repo from the same cwd is a worse trap than
      the original bug.
    answer: family
  fetch_fallback:
    ask: When the repo-refresh `git pull` fails, should sync fall back to `git fetch`?
    choices:
      fetch:
        refresh remote refs without touching the worktree so refusal remedies name the
        landed build
      status_quo:
        keep pull-or-nothing; a dirty linked worktree leaves remote refs stale and
        remedies generic
    default: fetch
    why:
      the command already chose network by not passing --no-pull; fetch is the safe
      subset of pull.
    answer: fetch
proposed_by: bbugyi200.apollo.dc.f0
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 15:35:15
status: wip
---

# `bob plugins sync`: deploy the enclosing checkout and fetch on pull failure

## Why (evidence from the 2026-11-11 `dc--code` run)

While deploying bob-ledger-tools 1.111.0 from the linked bob-plugins worktree
(`sase/repos/linked/bob-plugins`), the implementing agent hit, in order:

1. **Bare-sync refusal (exit 2).** `bob plugins sync` from inside the linked checkout
   refused: _"refusing bare sync from inside a different bob-plugins checkout (…); the
   sync would deploy ~/projects/github/bobs-org/bob-plugins instead. Run
   `bob plugins sync --repo <linked tree>`…"_. The command knew exactly which checkout
   the caller meant and made them retype it.
2. **Repo-refresh pull failure.** The retry with `--repo` warned _"git pull failed;
   using existing checkout: error: cannot pull with rebase: You have unstaged changes."_
   — expected in a worktree with a freshly built bundle — and left the remote-tracking
   refs stale.
3. **Dirty-stamp refusal with only the generic remedy.** The deploy guard correctly
   refused to overwrite bob-cli_11's uncommitted 1.108.0 deploy (stamp `0fd19de`), but
   because the refs were stale the landed-build hint degraded to the generic _"if that
   build has since landed, run `git fetch`…"_. The agent had to hand-run `git fetch`,
   verify commit `af6a346` on `origin/master` held the stamped bundle, update the
   branch, and re-sync (which then passed via the inclusion proof, no `-F`).

Not in scope because already handled:

- The `git check-ignore … pathspec magic not supported by this command: 'literal'`
  warning spam came from the installed `bob 0.5.0`; master already fixes it
  (`1a9b6828 fix(git): restore check-ignore probes`) and the fix rides the next release
  train.
- The deploy-protection guard itself behaved correctly: it protected another agent's
  unpushed dirty build and later permitted the overwrite through the inclusion proof.
  Its rules (HEAD-reachability for the proving commit, clean-incoming requirement, `-F`
  semantics) do **not** change in this plan.

## Fix 1 — resolve the repo from the enclosing checkout (retire the refusal)

Replace the foreign-checkout refusal (`src/native/plugins/guard.rs`, bob-cli-59) with
cwd-based resolution, matching git's own semantics and the documented workflow "make
changes to this linked repo and, when done, run `bob plugins sync`".

New repo resolution order (was: `--repo` → `BOB_PLUGINS_DIR` → default):

1. `-r/--repo` (unchanged, always wins),
2. **the enclosing bob-plugins checkout** when the cwd sits inside one (existing probes:
   `origin` remote points at `bobs-org/bob-plugins`, or the repo root holds `plugins/`
   plus the monorepo `package.json`),
3. `BOB_PLUGINS_DIR`,
4. `~/projects/github/bobs-org/bob-plugins`.

Details:

- When step 2 wins **and** its toplevel differs from what steps 3–4 would have resolved,
  print one stderr notice naming both paths, e.g.
  `bob plugins: using enclosing checkout <toplevel> (default is <resolved>; pass --repo to override)`.
  No notice when the cwd sits inside the env/default checkout (resolution is unchanged)
  or outside any bob-plugins checkout. Stderr keeps stdout JSON parseable, matching the
  pull-outcome convention.
- The enclosing checkout outranks `BOB_PLUGINS_DIR` deliberately: the bob-cli-59 hazard
  was precisely that the env/default repo silently beat cwd intent. The notice line is
  the guard against the reverse surprise.
- The exit-2 refusal path is retired entirely (`check_bare_sync`, `refusal_message`,
  `refuse_bare_sync`). Keep and repurpose the probes (`git_toplevel`,
  `is_bob_plugins_checkout`, `same_path`) as the resolution helper; `guard.rs` may be
  renamed (e.g. `resolve.rs`) if that reads better.
- Per `> [!decision] cwd_scope = family`: `repo_from_matches` in
  `src/native/plugins/cli.rs` performs the new resolution for `list`, the bare default,
  and `sync` alike, so the table header, JSON `repo`, and the deploy target always agree
  from any cwd.

> [!decision] cwd_scope = sync_only Only `run_sync` uses the new resolution; `run_list`
> keeps the current env/default order. The docs must then call out that `bob plugins`
> and `bob plugins sync` can name different repos from inside a worktree.

- Add an additive `repo_source` field
  (`"flag" | "enclosing_checkout" | "env" | "default"`) to both JSON result shapes
  (`PluginsResult`, `SyncResult` in `src/native/plugins/model.rs`) so automation can
  assert which rule fired. Additive JSON only; existing fields unchanged.
- This is a CLI contract change (a formerly refused invocation now deploys, and exit
  code 2 is retired): the commit subject takes the `!` break marker per the
  release-train policy. Never touch `Cargo.toml` version or `CHANGELOG.md`.

## Fix 2 — `git fetch` fallback when the repo-refresh pull fails

Per `> [!decision] fetch_fallback = fetch`: in `src/native/plugins/git.rs`, when
`pull_repo`'s `git pull` fails, run a non-interactive `git fetch`
(`GIT_TERMINAL_PROMPT=0`, default remote/refspec) before giving up:

- Extend `PullOutcome` with a fetched-fallback variant carrying the pull failure reason
  (e.g. `FetchedOnly { pull_error: String }`).
- `print_pull_outcome` reports it as one stderr warning, e.g.
  `bob plugins: warning: git pull failed (<reason>); fetched remote refs instead — checkout files unchanged`.
  If the fetch also fails, keep today's single pull-failure warning.
- `--no-pull` still skips both; non-worktree repos and missing git still skip silently.
  No guard or remedy code changes: the landed-build lookups (`find_landed_dirty_commit`,
  `find_clean_landed` in `src/native/plugins/deploy.rs`) already read remote-tracking
  refs and run after the refresh, so they see the fresh refs automatically and can name
  the proving commit and ref instead of the generic fetch hint.
- The contract sentence "Protection never depends on a remote fetch" stays true: the
  refresh fetches, the guard still only reads local state.

> [!decision] fetch_fallback = status_quo Skip this section's code changes; update only
> the docs wording from Fix 1 and leave `pull_repo` untouched.

## Non-goals

- No change to deploy-protection permit rules: HEAD-reachability for the inclusion
  proof, the clean-incoming requirement, and `-F` semantics stay.
- Sync never mutates the source checkout beyond refresh (no auto-stash, auto-merge, or
  branch updates).
- No new CLI options; no bob-plugins repo changes; no vault-side changes.

## Implementation steps

1. **Resolution helper.** Rework `src/native/plugins/guard.rs` into the
   enclosing-checkout resolver returning the toplevel plus a source tag; delete the
   refusal API. Unit-test the probe edge cases that exist today (non-git cwd,
   non-bob-plugins checkout, cwd inside resolved checkout).
2. **CLI wiring.** In `src/native/plugins/cli.rs`, make `repo_from_matches` return
   `(PathBuf, RepoSource)` using the decision-selected scope; print the stderr notice;
   drop `refuse_bare_sync` and the exit-2 path from `run_sync`; thread `repo_source`
   into both report structs and JSON (`model.rs`, `render.rs`, `scan.rs`, `sync.rs` as
   needed).
3. **Fetch fallback.** Implement the `PullOutcome` variant and fallback in
   `src/native/plugins/git.rs` plus `print_pull_outcome` wording.
4. **Help text.** Update the `sync` `long_about`/`after_help` and the `--repo` help
   string in `cli.rs` (resolution order, no more refusal), then `just docs-gen` for the
   generated clap pages.
5. **Contract doc.** Rewrite `docs/plugins.md`: Discovery's repo-root resolution list;
   Repo Refresh (fetch fallback); replace the "Foreign-checkout guard" section with the
   enclosing-checkout resolution contract (notice line, precedence over
   `BOB_PLUGINS_DIR`); Exit status (retire the exit-2 case); JSON output
   (`repo_source`); adjust the deploy-protection remedy paragraph to note the refresh
   keeps remote refs fresh even when the pull cannot advance the worktree.
6. **Tests.** In `tests/cli/plugins.rs`, replace
   `plugins_sync_bare_from_foreign_checkout_refuses_before_pull_or_copy` with a test
   asserting the bare sync deploys the enclosing checkout, prints the notice, and
   reports `repo_source: "enclosing_checkout"`; keep/adapt
   `plugins_sync_bare_from_resolved_checkout_is_allowed`,
   `plugins_sync_bare_from_unrelated_cwd_is_allowed`,
   `plugins_sync_explicit_repo_from_foreign_checkout_is_allowed`; add a list test for
   the family scope (per decision); extend the `pull_repo` tests in
   `src/native/plugins/tests.rs` (`pull_repo_skips_non_git_directory`,
   `pull_repo_fast_forwards_from_remote`) with a dirty-worktree fixture whose pull fails
   and whose fetch still updates a remote-tracking ref, and assert a dirty-stamp refusal
   in that fixture now names the landed commit and ref.
7. **Gates.** `just check` and `just docs-check` must pass; the pre-existing 6000-word
   reference budget failures (`bob-cli-jz` / `bob-cli-h9`) are known and must not grow
   from this change.

## Acceptance

- From inside a linked bob-plugins worktree, bare `bob plugins sync -p <id>` deploys
  that worktree with a notice; `bob plugins` names the same repo (family scope).
- With `BOB_PLUGINS_DIR` set and cwd outside any checkout, behavior is byte-identical to
  today except the additive `repo_source` JSON field.
- In a worktree where `git pull` fails with unstaged changes, sync still refreshes
  remote refs, and a dirty-stamp refusal names the landed commit and remote ref when one
  holds the stamped bundle.
- `--no-pull` performs no network calls at all.
