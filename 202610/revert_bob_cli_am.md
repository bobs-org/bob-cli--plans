---
tier: tale
title: Revert the landed bob-cli-am undirected task relations work
goal: bob-cli, bob-plugins, the installed bob binary, and the vault plugins carry
  no bob-cli-am behavior, docs, or glossary changes, matching their pre-epic state.
size: small
decisions:
  glossary_relations:
    ask: Should the glossary go back to its pre-epic relation strands (restore task-relation,
      delete the two new ones)?
    memory:
    - glossary:task-relation
    - glossary:undirected-relation
    - glossary:directed-relation
    default: true
    requested: Can you help me revert any work that was already submitted related
      to the bob-cli-am epic bead?
    answer: true
proposed_by: bbugyi200.apollo.9y
proposed_with: claude/opus
decided_by: auto
status: done
---

# Revert the landed bob-cli-am (undirected task relations) work

## Context

Epic `bob-cli-am` ("Undirected task relations: a plain link has no direction, a label
gives it one") was **canceled** on 2026-10-21 at 16:00Z. Bryan's close reason: too
little value for how rarely task relations are used. Three of its eight phases finished
and pushed commits before the cancel. Phases 3, 4, 5, 7, and 8 were force-canceled and
left no commits. This plan backs out the landed work in every repo and redeploys this
host so the tools match the pre-epic behavior.

### Landed epic work (each commit carries a `SASE_BEAD=[bob-cli-am.N]` trailer)

| Repo            | Commit                                                                           | Phase | What it did                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------- | -------------------------------------------------------------------------------- | ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob-cli         | `39e1d48a` docs(relations): define pair contract and glossary                    | am.1  | Rewrote `docs/task-relations.md` around the pair model; touched glossary/vault-format/contracts docs, the contracts snapshot (`docs/_meta/contracts.yml`, `tools/docs/tests/test_contracts.py`), and memory: rewrote `sase/memory/glossary/task-relation.md`, added `glossary/directed-relation.md` and `glossary/undirected-relation.md`, plus the generated `sase/memory/glossary.md` strands block, `sase/memory/README.md`, `AGENTS.md`, `CLAUDE.md` |
| bob-cli         | `dec8f879` feat(relations): add pair planner, projection, and redundant warnings | am.2  | Added `src/native/task_relations/pair.rs` and tests. Reconcile now omits plain LINKED FROM mirrors that the target already names, and a new `redundant_relation` warning kind was added (`src/native/task_status_hooks/relations.rs`, `tests/cli/task_status_hooks/relations.rs`, `docs/task-status-hooks.md`)                                                                                                                                           |
| bob-plugins     | `0bfae66` feat(ledger-tools): one aggregated links block per task                | am.6  | Added `related()` to `api.relationIndex` and one `🪢 links` block per task. Touches the `src/` fragments `145`, `146`, `153`, `171`, `278`, `282`, `350`, the generated `plugins/bob-ledger-tools/main.js`, `scripts/relation-inverse-vectors.cjs`, two relation test suites, `docs/bob-ledger-tools.md`, and `docs/develop/plugin-apis.md`                                                                                                              |
| bob-mac-capture | none                                                                             | —     | am.8 was canceled before it started                                                                                                                                                                                                                                                                                                                                                                                                                      |

Facts checked while planning:

- Both bob-cli commits are the most recent touches of every file they changed. The only
  commit between them (`c81f8fd2`, stats) touches unrelated files. `0bfae66` is
  bob-plugins `master` HEAD. All three reverse-apply cleanly (`git apply --check -R`).
- Neither bob-cli commit has shipped: the newest tag, `v0.2.0` (`ea7f5f0b`), predates
  both. So the revert breaks no released CLI or JSON contract and takes no `!`.
- `~/.cargo/bin/bob` on this host was rebuilt at 11:58 EDT, after `dec8f879` landed, so
  it runs the am.2 reconcile behavior. am.6 ran `bob plugins sync`, so the vault's
  bob-ledger-tools has the links block.

### Out of scope (not bob-cli-am work; keep it)

- Earlier task-relation work: the Oct 20 relation commits, and `bob-cli-aj`'s
  relation-pipe/card work (`bcc201fe`, `8bcc3e2b` in bob-cli; `bf40f48`, `d73d943` in
  bob-mac-capture; `4ca3b50`, `d4927af`, … in bob-plugins).
- Commits interleaved with the epic: `c81f8fd2` (stats), `55470e65` (date-marks), and
  bob-plugins `105f2b7` (recurrence marks).
- The beads: the epic and its phases are already closed. Do not change their status. No
  open follow-up bead belongs to this epic.
- The archived plan `plan:202610/undirected_task_relations.md`. It stays as a record.
- `CHANGELOG.md` and the `Cargo.toml` version. The release train owns them.

## Steps

### 1. Re-confirm the scope

Open `bob-plugins` and `bob-mac-capture` with `/sase_repo`. In all three repos, run
`git fetch` and then `git log --all --format='%h %s' --grep='bob-cli-am'`. If any
epic-tagged commit beyond the three above has landed since planning, revert it too
(newest first, same method). For each extra commit, check that no later non-epic commit
touched its files. If one did, resolve the conflict so the later non-epic work survives.

### 2. bob-cli: stage the inverse of both commits (one revert, no direct commit)

From the bob-cli workspace root:

```bash
git revert --no-commit dec8f879 39e1d48a   # newest first
```

Do not run plain `git revert` or `git commit`. Commits go through the SASE final
declaration (step 6).

> [!decision] glossary_relations = no After the revert, run
> `git checkout HEAD -- sase/memory/glossary/` so the three relation strands keep their
> current text and files, and nothing under `sase/memory/glossary/` changes.

With `glossary_relations` at `yes` (the default), keep the reverted memory files:
`task-relation.md` gets its pre-epic text back, and `directed-relation.md` and
`undirected-relation.md` are deleted. Either way, run `sase memory init` to regenerate
`sase/memory/glossary.md`'s strands block, `sase/memory/README.md`, `AGENTS.md`, and
`CLAUDE.md`. The regenerated output is authoritative. If it differs slightly from the
reverted copies, keep the regenerated version. Never hand-edit the generated files.

### 3. Verify bob-cli

- No-leftover check:
  `git diff 55470e65 --stat -- <every path touched by 39e1d48a or dec8f879>` must be
  empty, except for memory files kept by `glossary_relations = no` and any generated
  file `sase memory init` legitimately changed. Then confirm that
  `src/native/task_relations/pair.rs`, `sase/memory/glossary/directed-relation.md`, and
  `sase/memory/glossary/undirected-relation.md` no longer exist (unless
  `glossary_relations = no`), and that `git grep -n redundant_relation` finds nothing.
- `just fmt` (expect no changes), `just lint`, `just test`, and `just docs-check`. All
  must pass. If a failure looks unrelated, rerun it on a clean `origin/master` checkout
  to show it predates the revert. Record any such failure through `/sase_new_task`
  instead of fixing it here.

### 4. bob-plugins: revert, rebuild, test, and sync

In the `bob-plugins` checkout that `/sase_repo` printed:

```bash
git revert --no-commit 0bfae66
npm run build          # must leave the generated main.js identical to the reverted copy
npm run build:check
npm run docs:check
npm test
```

`git diff 0bfae66~1` must be empty after the build. For `npm test`, all relation suites
must pass: `test-navigation-relation-*`, `test-relation-index-linked-from-stage`,
`test-ledger-tools-relations`, `test-ledger-tools-relation-index`,
`test-ledger-tools-relation-chips`, `test-task-status-cycler-relations`, and
`test-block-id-prompt-relations`. The only acceptable failures are the ones already
known before the epic:

- `test-ledger-tools-unblocked-glyph` ×4 (`bob-cli-6v`)
- the decision-card Keep count (`bob-cli-6w`)
- the forward-promotion conversion
- the stage-ranker timing flake

Treat any other failure as a regression and fix it.

Then run `bob plugins sync` so the vault's bob-ledger-tools loses the links block.

### 5. Redeploy `bob` on this host and restore relation mirrors

From the bob-cli workspace, with the step-2 revert in the working tree, run
`just install`. That rebuilds `~/.cargo/bin/bob` without the am.2 pair code. Then:

```bash
bob task reconcile --dry-run --format json
```

The output must contain no `redundant_relation` warning. If it lists
`linked_from_line_updates`, run `bob task reconcile` once. Those updates are plain
LINKED FROM mirrors that the am.2 binary omitted, and LINKED FROM lines are derived, so
this restores them. A second `--dry-run` must report no relation changes.

Do not touch other hosts (athena, mac). They pick up the revert on their next
`just install-all`. Say so in the final report.

### 6. Commit (one commit per repo, through the SASE final declaration)

- bob-cli:
  `revert(relations): back out the bob-cli-am pair contract, glossary, and pair planner`.
  The body names the reverted commits `39e1d48a` and `dec8f879` and the canceled epic
  `bob-cli-am`.
- bob-plugins: `revert(ledger-tools): back out the bob-cli-am aggregated links block`.
  The body names `0bfae66` and `bob-cli-am`.

Neither message takes `!`, because neither reverted change ever shipped in a release.
The final report lists both new commits, the redeploy steps that ran (`just install`,
`bob plugins sync`, and reconcile, with its relation-update count), and any pre-existing
test failures recorded as beads.

## Acceptance

- bob-cli and bob-plugins `master` match their pre-epic state on every path the epic
  touched (subject to `glossary_relations`). No other files change.
- `just test`, `just lint`, `just docs-check`, and the bob-plugins build check, docs
  check, and relation suites pass.
- The installed `bob` and the vault plugins on this host carry no bob-cli-am behavior.
  Reconcile reports no `redundant_relation` warning and no pending relation changes.
