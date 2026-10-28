---
tier: tale
title: Provision the isolated plans sidecar for external SASE validation
goal:
  sase validate passes in an external sase checkout opened from another project's
  workspace by validating that project's own provisioned plans sidecar, never the host's
  plans and never a silenced stage.
size: medium
proposed_by: bbugyi200.athena.bob-cli-dp.1
proposed_with: claude/opus
bead: bob-cli-dp.1
status: done
---

- **BEAD:**
  [bob-cli-dp.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dp/bob-cli-dp.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-dp.1](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-dp.1.md)
- **COMMITS:**
  - [7531093](https://github.com/sase-org/sase/commit/75310937136049b2f90eb37f5304fff7be3c52ef)
    — feat(repo): provision nested plans sidecar on registered-project open

# Provision the external project's own plans sidecar and validate against it

Phase `bob-cli-dp.1` of epic `bob-cli-dp` (plan `202610/backlog_batch_b5.md`), resolving
task `bob-cli-dg`. All source changes land in the **sase** repository, opened with
`/sase_repo` (`sase repo open sase -r "<reason>"`); use only the path it prints. No
bob-cli source, bob-cli memory, or sase memory note changes are planned.

## Root cause (verified while planning)

From a numbered bob-cli workspace, `sase repo open sase` clones the registered `sase`
project into the host-scoped tree `<host>/sase/repos/external/projects/sase`. Inside
that checkout, `sase validate` passes every stage except `plan links validate`, which
reports `invalid-root` for `<external>/sase/repos/plans`. This reproduces with the
checkout's own source, not only the globally installed `sase`.

1. **Identity leak.** The external checkout has no `.sase/checkout.json`.
   `sase.sdd._paths.get_primary_workspace_dir` → `_resolve_primary_from_marker` →
   `find_marker_from_cwd` walks up and finds the **host bob-cli workspace's** marker. So
   `resolve_sdd_store(<external>, 1)` reads **bob-cli's** store record (plans remote
   `bobs-org/bob-cli--plans`). It also builds `plans_dir` with
   `sidecar_clone_root(<external>, "plans")`, a path under the nested checkout that
   nothing ever creates.
2. **Preparation and resolution disagree.** Commit `36b962ad9f` deliberately anchors
   sidecar materialization for nested repos to the owning workspace
   (`sase.sdd._store_workspace._inherited_sdd_record_owner_anchor`), so nothing ever
   provisions the nested path that resolution points at.
3. **What a correct context looks like.** A clean clone of sase resolves the GitHub
   provider policy `separate_repo` (`<checkout>/.sase/sdd`), which is also absent. CI
   makes `just validate` meaningful only by provisioning the project's **own** store
   first. `tools/ci_bootstrap_sidecars` clones the sidecars declared in `sase/sase.yml`
   into `sase/repos/<role>` and writes a checkout-local `.sase/sdd-store.json`. The
   correct isolated context for an external sase checkout is therefore sase's own plans
   sidecar, never bob-cli's plans and never a silenced stage.

The external checkout already git-excludes `.sase/` and `/sase/repos/`. The current
`ensure_sase_git_info_excludes` patterns plus sase's `.gitignore` cover them, so
provisioning there does not dirty the checkout or create a commit obligation.

## Design

Two parts, both required:

### A. `sase repo open` provisions the registered project's own plans sidecar

The scope is tier-2 opens: registered SASE projects, meaning
`_ExternalRepoTarget.project_source_dir is not None` in
`src/sase/main/repo_open_external.py`. After `_materialize_external_repo` and
`ensure_sase_git_info_excludes`, run an idempotent provisioning step. Put it in a new
focused module, for example `src/sase/main/_repo_open_external_sdd.py`, rather than
growing `repo_open_external.py`. It runs on a fresh clone **and** on every valid reopen,
so checkouts opened before this fix are upgraded by the next `sase repo open`.

- **Source of truth.** Read the registered project's materialized store record from its
  primary checkout (`read_sdd_store_record(project_source_dir)`). That primary is
  already the clone source for this open. This is sase reading the canonical source
  during an audited open; it never reads another numbered workspace. Provision only when
  the record is materialized, `is_sidecar_storage`, and names `plans`. Otherwise skip
  silently, which leaves today's behavior.
- **Clone first, then record.** Materialize only the `plans` sidecar, the role numbered
  workspaces auto-clone, at `sidecar_repo_clone_dir(<external>, "plans")` with
  `sase.sdd._store_link.ensure_sidecar_sdd_clone(..., strict=True)`. Use the same
  reference-repo policy numbered workspaces use: the source primary's plans clone when
  it is a matching directory, else none. Write the checkout-local record
  (`write_sdd_store_record(<external>, record)`) only after the clone succeeds, so a
  record never names a missing clone. Do not clone research, beads, or hidden
  (agents/attachments) sidecars.
- **No reset of existing state.** If a matching plans clone already exists, leave it as
  is: no forced fetch, reset, or `fresh` integration. This honors the documented
  `repo open` contract of not synchronizing an existing checkout. Rewrite the
  checkout-local record only when it differs from the source record.
- **Failure policy.** Provisioning failure (network, auth, mismatched clone) prints one
  actionable warning on stderr naming the sidecar, the error, and the remedy (rerun
  `sase repo open <name>`). It writes no record, and the open still prints its path and
  succeeds. Validation stays fail-closed, so nothing is silenced.
- **Unchanged.** Tier-3 provider refs (`gh:owner/repo`) and linked repos are out of
  scope. They have no registered primary record to derive from. Record this as a
  `PROPOSED FOLLOW-UP:` note on `bob-cli-dp.1` if it still matters after this change.

### B. Link validation reads the nested checkout's own provisioned store

`sase.sdd._link_files._resolve_project_sdd_root` is used by `resolve_sdd_root` and so by
`plan links validate/list/repair/refresh`, plus the doctor fallback. Change it to
resolve an explicit read context before the generic `resolve_sdd_dir(project_root, 1)`:

- Find the nearest enclosing git checkout root of `project_root` (the first ancestor
  with a `.git` entry, no subprocess). If that root sits inside a host checkout's
  repo-open clone tree and has its own materialized checkout-local record, return
  `resolve_sdd_store(root, 1, primary_workspace_resolver=<checkout-local>).sdd_dir`. The
  repo-open clone tree is `sase/repos/external/…` or `sase/repos/linked/…` under the
  owner found by `find_marker_from_cwd`; reuse the constants in
  `sase._linked_repo_paths`. The checkout-local resolver returns the checkout root
  itself as primary, so the host's marker-derived record is never inherited.
- Every other case keeps today's resolution byte-for-byte: primary checkouts, numbered
  workspaces, the owner's own `sase/repos/<role>` sidecar clones, and nested clones
  without their own record. Do **not** change `get_primary_workspace_dir`,
  `resolve_sdd_store`'s default resolver, or bead/plan write paths. Agents running
  `sase bead …`/`sase plan propose` inside an external checkout must keep acting on the
  host project.
- When the resolved root is missing for an unprovisioned nested repo-open clone, keep
  the `invalid-root` error and exit status. Append a remedy that rerunning
  `sase repo open` for that checkout provisions its plans sidecar. Never downgrade it to
  a warning or skip.

## Tests (sase repo)

Use temp git repos and bare remotes in the style of
`tests/sdd_store/test_workspace_clone.py::test_nested_repo_inherits_owner_sdd_record_without_nested_sidecar`
and `tests/main/test_repo_handler_open_external.py`.

1. Opening a registered project whose primary has a sidecar record clones `plans` into
   `<external>/sase/repos/plans` from the project's own plans remote. It writes
   `<external>/.sase/sdd-store.json` equal to the source record, and the external
   checkout's `git status --porcelain` stays empty.
2. Reopen is idempotent. It upgrades an unprovisioned earlier checkout and never resets
   an existing matching plans clone.
3. No provisioning happens for a source with no record, a non-sidecar record, or a
   tier-3 provider ref.
4. Provisioning failure warns on stderr, writes no record, and still returns the path.
5. With a host workspace marker whose primary record names **different** plans, link
   validation run from inside the provisioned external checkout (and from a subdirectory
   of it) validates the external project's own plans clone. A deliberately broken plan
   there fails; the host's plans are never read.
6. An unprovisioned nested clone still fails `invalid-root`, now with the remedy text.
   Primary, numbered-workspace, and owner-sidecar-clone resolution are unchanged and
   pinned by assertions.
7. An end-to-end clean-checkout test: a fresh registered project, open, then
   `validate_sdd_tree(None)` from the new checkout passes.

## Docs

Read `sase memory read docs_writing.md` and `cli_rules.md` first. Update the
`sase repo open` help/description (`src/sase/main/parser_repo.py`) and the generated
`docs/reference/cli/repo.md`, plus `docs/reference/config/repositories-behavior.md` if
it describes external opens. State that opening a registered SASE project also
provisions that project's own plans sidecar inside the external checkout for validation.
Regenerate generated docs with the repo's documented tool; do not hand-edit generated
output.

## Verification and closeout

1. Open sase with `/sase_repo`. If the checkout is clean, fast-forward it to
   `origin/master` before editing. Read its `AGENTS.md` and the `lint_and_test` memory
   note before finishing. Run every guarded recipe only through `sase tool run`, never
   raw `just`, and never `check-full`.
2. Baseline: in the external checkout, `sase plan links validate` reports `invalid-root`
   for `sase/repos/plans`. Record the exact output.
3. Run the focused suites through `sase tool run test -- <test paths>`.
4. Real-checkout acceptance. Global reinstall is human-only, so run the **modified**
   code's open from the host workspace root using the external checkout's own editable
   venv: `<external>/.venv/bin/sase repo open sase -r "<reason>"`. Create the venv first
   with `sase tool run install-venv` in the external checkout if it is missing. Then,
   inside the external checkout:
   - `.venv/bin/sase validate` shows every stage `ok`;
   - `.venv/bin/sase plan links validate` passes against sase's own plans;
   - `git status --porcelain` shows only intended source edits.

   Use `sase validate` or `sase plan links validate`; `sase validate plan-links` is not
   valid syntax.

5. Run the governed gate `sase tool run check` in the external sase checkout. Never
   claim green on NEW/UNKNOWN failures. A failure that reproduces identically on the
   clean base tree is recorded as `PROPOSED FOLLOW-UP:`, citing any existing task bead.
   Do not close `bob-cli-dg` or any ancestor; the epic lander closes `bob-cli-dg`.
6. Record evidence: the baseline failure, ToolRun ids, the validate output, and the
   provisioned remote. Use `sase bead note bob-cli-dp.1 '…'`, plus a short corroborating
   note on `bob-cli-dg`. Record discovered follow-up work only as `PROPOSED FOLLOW-UP:`
   notes on `bob-cli-dp.1`, for example tier-3 SASE-managed externals, or the
   store-resolution versus owner-anchor inconsistency for nested repos that inherit a
   record. Do not create beads.
7. Run `sase bead epic-symbols bob-cli-dp.1` and resolve or re-key any entries. Then run
   `sase bead close bob-cli-dp.1 --note "<what you verified>"`. The sase repo edits are
   a commit obligation of the turn; declare them through `/sase_final`.
