---
tier: tale
title: Install the bank-journal vault git policy and close the import phase
goal:
  Apply and verify the narrow .bob/bank/*.jsonl allowlist and union-merge rule in the
  live Bob vault repository, then normally close bob-cli-kd.4.1 and bob-cli-kd.4.
size: small
bead: bob-cli-kd.4.1
proposed_by: bbugyi200.apollo.df
proposed_with: claude/claude-fable-5
create_time: 2026-11-12 06:21:36
status: wip
---

- **BEAD:**
  [bob-cli-kd.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kd/bob-cli-kd.4.1.md)

# Finish the bank-import landing closeout

Epic `bob-cli-kd.4.1` (weekly bank import) and its parent phase `bob-cli-kd.4` are
complete except for one setup step and the closeout that was gated on it. The landing
repairs from tale `202611/bank_import_landing_repairs.md` are implemented and landed on
`master` (`ae6a28d0 fix(bank): harden import planning and recovery`, docs in `b8c71181`
and `9662ce54`). The lander's note #3 on `bob-cli-kd.4.1` records fresh passing results
(`native::bank` 106/106, `host_catalog::txn` 13/13, notebooks 49/49, albums 65/65,
`just docs-gen` pass) and its note #4 records the required monitored `just check` gate:
red, exit 101, with exactly two failures outside the bank diff, both already tracked
(`bob-cli-kl` for the `subprocess_runner` pipe-drain truncation flake; the
`capture::complete_themes` completion-deadline omission corroborated on `bob-cli-g8`).

The one blocker: the live Bob vault repository still lacks the bank-journal git policy
documented in `docs/vault-git-sync.md#bank-import-journal`, and the lander could not
broker that repository (its attempt used the unknown inventory name `bob`). Planning for
this tale verified the working reference: `sase repo open gh:bobs-org/bob` resolves the
vault's GitHub remote and returns a checkout whose `.gitignore` has no
`!/.bob/bank/*.jsonl` line and whose `.gitattributes` has no bank union rule. Without
them, the first real `bob bank import` refuses its journal destination.

This tale installs that policy, re-verifies, and performs the closeout steps that the
repair tale reserved for after journal readiness. It makes **no bob-cli source, docs,
version, or CHANGELOG changes** and **no SASE memory edits** (the approved
`glossary:bank-month-note` / `glossary:bank-row` strands belong to the outer
`bob-cli-kd` wrap-up, per the phase's final DECISIONS).

## Read first

1. `sase bead read bob-cli-kd.4.1 -r "Need landing audit, triage outcomes, and gate result"`
   and `sase bead read bob-cli-kd.4 -r "Need phase scope and final decisions"`. The
   close notes below must carry these recorded outcomes accurately.
2. `sase memory read obsidian.md -r "Vault git conventions before editing the vault repo"`.
3. `docs/vault-git-sync.md#bank-import-journal` in the bob-cli checkout for the exact
   policy lines and verification commands.

## 1. Install and verify the vault git policy

Open the live vault repository only through the broker:
`sase repo open gh:bobs-org/bob -r "Install the documented bank-journal ignore allowlist and union merge rule"`.
Use only the printed path. Read that checkout's `AGENTS.md` before editing. Inspect
`git status` first; pre-existing changes may be legitimate sync activity — never revert,
stage, or touch anything except the two files below.

Make both edits idempotent (skip any line that already exists):

- Vault root `.gitignore`: after the Gmail journal block (`!/.bob/gmail/*.jsonl`), add:

  ```gitignore
  # Bank import journal, one file per host.
  !/.bob/bank/*.jsonl
  ```

- Vault root `.gitattributes`: after the Gmail union rule, add:

  ```gitattributes
  # Union-merge concurrent bank-import-journal appends from different hosts.
  .bob/bank/*.jsonl merge=union
  ```

Preserve every existing rule and comment. Verify effective rules in that checkout,
including ancestor-directory ignores, for at least `apollo.jsonl` and `athena.jsonl`:

```bash
git check-ignore -v .bob/bank/apollo.jsonl   # last match must be the negated !/.bob/bank/*.jsonl (exit 1: not ignored)
git check-attr merge -- .bob/bank/apollo.jsonl   # must print: merge: union
```

The `.bob/bank/` directory does not exist yet; that is expected (it is created on the
first import) and does not affect either check. Do **not** commit manually and do
**not** use `/sase_git_commit` in the vault checkout (its `AGENTS.md` forbids it: that
flow archives SASE plan files into the vault). The prepared, verified tree rides this
tale's final declaration. Closeout must not wait on this change's commit SHA, push, CI,
or on `bob vault-sync` picking it up. Do not access `~/bob` directly, import real bank
data, or place raw exports anywhere.

## 2. Fresh observed verification in bob-cli

In the bob-cli checkout (unchanged by this tale), run the focused suites and record the
observed counts for the close note:

```bash
cargo test --lib 'native::bank::'
cargo test --lib 'host_catalog::txn::'
```

This tale changes no bob-cli files, so the file-change gate `just check` is not
triggered; do not rerun it and never run `just check-full`. Cite the recorded gate
result (`bob-cli-kd.4.1` note #4) as-is: red, both failures outside the bank diff and
already tracked. Do not claim any pass that was not observed.

The weekly Monday PRE chore already exists in `gtd_daily` (one match, `#gtd #pre`,
`repeat = every week on Monday when done`, scheduled 2026-11-16, verified read-only by
the lander). Preserve it; do not capture a duplicate or mutate it. Use the `bob_query`
skill only if a fresh read-only confirmation is needed.

## 3. Mark the plans done

Open the plans sidecar through
`sase repo open plans -r "Mark the bank import plans done"` and edit only frontmatter
`status:` fields:

- `202611/bank_import.md`: set `status: done` (currently `wip`).
- `202611/bank_import_landing_repairs.md`: set `status: done` if not already flipped by
  host tooling.

## 4. Close the beads

Perform these in order, without waiting on any commit, push, or CI:

1. `sase bead epic-symbols bob-cli-kd.4.1` — empty at planning time; resolve any new
   entries by wiring, privatizing, deleting, or a non-test pragma before closing.
2. `sase bead close bob-cli-kd.4.1 --note "<note>"`. The note must state: vault
   journal-readiness verified (both policy lines installed in the brokered vault
   checkout, `check-ignore`/`check-attr` outputs), the observed focused test counts, the
   landed repair commits, the recorded red-but-unrelated `just check` gate with its two
   tracked failures, and the carried follow-up triage outcomes from note #1
   (`.1 #1`/`.2 #2` fixed by `6e3d8837`; `.2 #1`/`.3 #1` corroborated on `bob-cli-g8`
   and `bob-cli-3j`; `.4 #1` corroborated on `bob-cli-kk` and `bob-cli-kb`; `.4 #2`
   resolved by this tale's vault policy step). Never use `--force`; if the close is
   rejected, fix what it names and retry.
3. `just symvision` only if the recipe exists (absent at planning time).
4. Re-read phase `bob-cli-kd.4`, confirm the child fulfilled the import scope and the
   inherited final decisions (checking-only accounts; subscriptions out of scope; no
   memory edits here), run `sase bead epic-symbols bob-cli-kd.4` (also empty at
   planning) and resolve any entries, then
   `sase bead close bob-cli-kd.4 --note "<verified child import scope, policy install, and checks>"`.
5. Leave the outer epic `bob-cli-kd` and `202611/bank_daily_acknowledgment.md`
   untouched: the waiting outer land agent owns them, including the approved glossary
   strands. If the phase has gained genuinely incomplete work since planning, append the
   blocker as a bead note and report it instead of forcing closure.

## 5. Finish

End through `/sase_final`, declaring every changed repository: the vault external repo
(`gh:bobs-org/bob`) and the plans sidecar. No manual git commits anywhere.
