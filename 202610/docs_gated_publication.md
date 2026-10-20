---
tier: epic
title: Launch Bob documentation with gated hourly GitHub Pages publishing
goal:
  "Complete the publish scope of bob-cli-a0.5: ship reusable publication and weekly
  workflows, bootstrap bobs-org/bobs-org.github.io, verify the live site, and
  demonstrate a broken-link drill followed by recovery without replacing the last good
  site. Record the run evidence for the assigned phase's land agent."
phases:
  - id: source-plan
    title: Publication source selection and manifest contracts
    depends_on: []
    size: medium
    description:
      "source-plan: implement and test publish-plan, deterministic input fingerprints,
      production source manifests, and hold/pinned/force/drill selection semantics
      against local Git fixtures."
  - id: publish-workflow
    title: Gated reusable publication and recovery
    depends_on:
      - source-plan
    size: medium
    description:
      "publish-workflow: implement the workflow_call-only Pages publisher, staged
      broken-link drill, publication recording, live verification, and one blocked-issue
      lifecycle with focused failure-path tests."
  - id: host-controls
    title: Deploy repository templates and operator commands
    depends_on:
      - publish-workflow
    size: medium
    description:
      "host-controls: implement thin caller templates, idempotent preview/apply host
      bootstrap, publish dispatch/watch and status commands, just recipes, and the
      operator runbook without launching the live site yet."
  - id: weekly-checks
    title: Weekly fallback build and external link checks
    depends_on:
      - host-controls
    size: medium
    description:
      "weekly-checks: add the workflow_call-only weekly Material build and online lychee
      check, its separate issue lifecycle and caller, and validate the complete workflow
      set and production build path."
  - id: launch
    title: Bootstrap the public host and prove publication safety
    depends_on:
      - weekly-checks
    size: medium
    description:
      "launch: add the three README docs links, configure the authorized public host and
      homepages, run first publish, broken-link drill, normal recovery and weekly
      checks, and record live URL and provenance evidence."
proposed_by: bbugyi200.apollo.bob-cli-a0.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-a0.5
create_time: 2026-10-19 23:18:28
status: wip
---

- **PROMPT:**
  [prompts/202610/docs_gated_publication.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/docs_gated_publication.md)
- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)

# Complete the publication phase

This is a child implementation epic for **bob-cli-a0.5**, under the approved
`plan:202610/bob_public_docs_site.md`. Its five phases deliberately serialize the shared
CLI, workflow, and operator-documentation changes. Each is bounded work for one coding
agent; the launch phase owns external verification after all code lands. No
implementation has been made by this planner.

The parent phase is in progress, has no prior-work notes, and has closed prerequisites
`bob-cli-a0.1` through `.4`. This split is based on the missing implementation and
operational scope, not on an earlier incomplete attempt. Keep `.5` open through the
handoff. Child phase workers close only their own assigned beads. The child epic's land
agent prepares/verifies completion of `.5`; closure of `.5` belongs to the authorized
land continuation for that assigned phase, never a child phase worker. Do not close
`bob-cli-a0`, any other ancestor, or sibling phases.

## Binding decisions and scope

The parent decisions are final: MIT licenses, search indexing enabled, generated design
decisions published, and memory authorization limited to `docs.md`. This child plan
needs **no memory edits** and asks no new decisions.

Preserve the approved architecture:

- `bob-cli` owns all publishing logic, documentation, and the two reusable workflows.
- `bobs-org/bobs-org.github.io` holds only thin callers, README, and the publication
  manifest. It publishes the org-root site, using the caller repository's token.
- Use hourly polling and manual dispatch. No new secrets, tokens, domains, website
  platforms, push-triggered CI in bob-cli or bob-plugins, or private-vault access.
- Source code is public; source fetching needs no cross-repository credential.
- Publication runs the existing strict build and gates before artifact upload/deploy.
  Keep URL=path, all existing published URLs, source ownership, and privacy checks.
- The parent already authorizes public host creation, Pages and Actions setup,
  publication, drill/normal dispatches, and all three GitHub homepage updates. Do not
  ask for that authorization again.

Read the parent plan and the research through audited commands:

```bash
sase bead read bob-cli-a0.5 -r "Need the approved publication scope and current notes"
sase artifact read plan:202610/bob_public_docs_site.md "Need the governing publication design"
sase artifact read research:202610/bob_public_docs_site/bob_public_docs_site.md "Need the publication research"
```

If the research checkout is absent, use `/sase_repo` to open `bob-cli--research` then
retry the artifact read. Never read sidecar artifacts directly.

## Evidence and existing integration points

At planning time the local base is `f8ef1628`. There is no `.github/`,
`tools/docs/host/`, publication planner, bootstrap, or operator command. The mount table
is currently empty; later parent phases will add mounts, so the publisher must discover
them dynamically rather than hard-code three source checkouts.

Reuse these implemented interfaces:

- `tools/docs/bob_docs/assemble.py`: `assemble(..., production=True, sources=...)`
  verifies source cleanliness before and after mounting, stages files under
  `target/docs/`, and emits `build.json` into the served site.
- `mounts.py`: `load_mounts`, origin validation helpers, and production validation. The
  strict production input is exactly
  `{"sources": {"bob-cli": {"path": "...", "sha": "<40 hex>"}, ...}}`; each required
  repo appears once. Extra metadata belongs in another file. The bob-cli path must equal
  the actual build root. `BOB_DOCS_*_DIR` overrides fail.
- `check.py`: `run_check` assembles once, then runs lint, generated drift, strict
  Zensical build, URL preservation, offline lychee with fragments, search smoke, and
  pytest. Preserve that shared source context and first-failure behavior. Rust golden
  tests remain a local gate; production does not require Rust.
- `links.py`: use `resolve_lychee`, including its existing release/checksum metadata,
  for both offline and online checks. Do not add another downloader.
- `urls.py`: reuse HTML/fragment resolution and the URL contract where possible.
  `build.json` already records repo SHAs, dirty flags, builder, time, and versions.
- `cli.py`, lowercase `justfile`, `tools/docs/README.md`, and `docs/develop/build.md`
  are the operator integration points.

Read `docs.md` and `cli_rules.md` with `/sase_memory_read` before editing those
interfaces. Keep subcommands/options alphabetical and public long options paired with
short aliases. Keep generated source/reference documents unchanged.

Read-only GitHub preflight observed:

- `GET repos/bobs-org/bobs-org.github.io` returned 404.
- `GET orgs/bobs-org/actions/permissions` returned 403 with missing `admin:org`. This is
  an inability to inspect the org setting, not evidence Actions is disabled.
- `GET repos/bobs-org/bob-cli/actions/permissions` returned
  `enabled=true, allowed_actions=all, sha_pinning_required=false`. Repository metadata
  reports admin/push permission and default branch `master`.

Retry these during launch. Do not refresh authentication, create credentials, or change
organization-wide policies. If org inspection remains unavailable, record that fact and
inspect the deploy repo's effective Actions permissions; actual workflow acceptance is
the final proof. Only an actual permission rejection blocks the affected external
action.

GitHub's
[Pages workflow documentation](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
requires a Pages environment and deploy permissions; its current examples use checkout
v6, configure-pages v5, upload-pages-artifact v4, and deploy-pages v4. Check release
tags during implementation, including setup-uv, then pin release tags as the parent
requires.
[Reusable workflow documentation](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
describes typed inputs and caller permissions, which cannot be elevated by the callee.
Use explicit caller/source checkout paths: checkout's default repository in a reusable
workflow is the caller.

## Shared publication contracts

### State and immutable inputs

Implement versioned, validated JSON records, with these distinct purposes:

1. **`published.json`**, durable in the deploy repo: schema version, `hold` boolean,
   source identities and exact commit SHAs, per-input Git tree/blob IDs, deterministic
   fingerprint, toolchain versions, build time, and run URL. A missing file means first
   publish; malformed/unsupported data is an error, never an empty state or permission
   to publish.
2. **Plan output**, transient: whether to build, a reason, flags, fingerprint, selected
   source data, and paths to the sources/next-publication files. Emit a compact JSON
   result plus stable `GITHUB_OUTPUT` fields for workflow conditions; never emit
   unchecked multiline values into that file.
3. **Production sources JSON**, transient: the exact assembler schema above. Keep it and
   other transient files outside tracked source inputs or under ignored outputs.

Freeze source SHAs before checking out content. Read the mount table from the selected
bob-cli revision, deduplicate repos with multiple mounts, and reject conflicting
remote/branch declarations for one repo. Resolve refs with `git ls-remote`, fetching the
observed SHA rather than a branch that can advance during the run.

For a normal run, compare resolved repo membership and SHAs with the manifest first. If
unchanged, skip further source work. When sources changed, compute a SHA-256 fingerprint
of canonical, sorted input identities and Git object IDs:

- bob-cli `docs`, `mkdocs.yml`, `tools/docs`;
- each `.github/workflows/docs-*.yml` file, including additions/removals;
- every declared mounted source directory, with its repo/source/target identity.

Use Git objects, not mtimes, local absolute paths, or timestamps. A source commit that
changes none of those inputs must not trigger a deployment or a manifest commit. Do not
update `published.json` merely to remember the last poll.

The initial tooling checkout needed to run the planner is separate from the selected
build sources. Execute pinned builds with compatible current orchestration and the
selected sources' locked dependencies; do not assume an old rollback revision contains
newly added helper entry points. Reject an unsupported rollback clearly rather than
silently building current content.

### Flag precedence and recovery

| Situation                                            | Required behavior                                                                                       |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Scheduled or ordinary unpinned run while `hold=true` | Skip; even force does not release the hold                                                              |
| `pinned=true`                                        | Require a valid manifest and build its exact SHAs, preserving hold                                      |
| `force=true`, with no unpinned hold                  | Rebuild despite SHA/fingerprint equality                                                                |
| `drill=true`, with no unpinned hold                  | Run the real gates despite equality; never deploy or record                                             |
| Same SHAs or same docs fingerprint                   | Skip without a manifest commit, unless forced, pinned, drilling, or recovering a blocked publication    |
| Open managed publication-blocked issue               | Normal run revalidates/builds even with equal inputs, then verifies live state before closing the issue |
| Failed planning/build/gate/upload/deploy             | Fail the run; report the failing step; no successful publication record                                 |

An unchanged fingerprint is not proof a previously blocked gate is repaired. Recovery
must exercise the gates. Held skips must not close an unresolved blocked issue.

Use a single publish concurrency group `pages`, with `cancel-in-progress: false`,
covering source selection through deploy, record, and live verification. There must not
be both caller and callee holding the same group and deadlocking. The weekly workflow
has a separate group and cannot publish or change publication state.

### Failure boundaries

A failure before deployment preserves the last good artifact and manifest. The
broken-link drill must demonstrate this boundary. A live verification or manifest push
failure occurs after deployment and therefore cannot honestly promise the old site is
still live. Report the actual state and failing step; preserve evidence, keep the
blocked issue open, and provide retry/hold/pinned recovery. Do not disguise post-deploy
failures as successful no-ops or claim automatic transactional rollback.

## Phase source-plan

Add `publish_plan.py` and narrowly scoped supporting modules/tests. Wire the
`publish-plan` command into the existing CLI with flags for the publication manifest,
output/work directories, force, drill, pinned, and recovery. Public flags follow the CLI
rules. Git subprocesses use argument arrays and bounded error handling.

Make acquisition usable in GitHub Actions and testable with local bare Git remotes.
Production source directories must be clean and pinned; never reuse the developer's
dirty checkout as production. Under SASE, agents must open any real external repo
through `/sase_repo` before inspecting it; fixture repos are synthetic test inputs.

Test behavioral boundaries: first publish; equal SHAs; code-only vs docs changes; two
mounts from one repo; mount add/remove; workflow add/remove; SHA movement after
discovery; unavailable refs; malformed manifest; all flag precedence rows; and
determinism independent of paths/order. Run a resulting sources manifest through the
existing production assembler tests to prove schema compatibility.

Acceptance: focused tests and the full docs Python suite pass; planner dry-runs do not
mutate publication state; documented outputs let the next phase wire workflow conditions
without redesigning the format. Update the tool README with the schema.

## Phase publish-workflow

Add `.github/workflows/docs-publish.yml` with **only `workflow_call`**, with boolean
force/drill/pinned inputs. Guard the expected caller repo and its default branch. Give
the caller the required contents/pages/id-token/issues permissions, reducing permissions
within jobs where practical. Use `github-pages` for the deploy job.

The workflow must:

1. Check out the deploy repo and bob-cli tooling into separate explicit paths. Read
   current publication state and any managed blocked issue. Plan immutable sources.
2. Skip on a legitimate no-op. Otherwise install locked dependencies and run the
   existing `bob-docs check --production --sources ...` on the selected build root.
3. For drills, inject a known broken internal link into the **assembled staging tree**,
   after clean-source verification and before the strict-build gate. Add a narrow drill
   mode to the check orchestration if needed. Do not dirty tracked source files, disable
   cleanliness, or merely exit with an artificial failure. Verify the expected
   broken-link diagnostic. If the injected link passes the gate unexpectedly, fail
   explicitly and still prohibit upload/deployment.
4. Upload only a successfully gated site and deploy only when drill is false.
5. After successful deploy, record `published.json` with the selected SHAs and build
   metadata. Preserve hold and unrelated deploy-repo files. Commit only the manifest,
   skip a no-change commit, and fail/report push conflicts rather than force-pushing or
   overwriting a concurrent operator hold change.
6. Verify the live URL set and curated fragment obligations with bounded retries,
   confirm `/build.json` matches expected provenance, then close the managed issue.

Add reusable Python support for publication record preparation, live checks, and
single-issue reporting if that keeps workflow YAML small. A failure handler must work
even when planning/building fails, identify the failed step and run URL, and
create/update exactly one `Docs publish blocked` issue with label `docs-publish`.
Provision the label idempotently. Weekly will reuse the reporting helper with a
different title/label. Use structured API input for issue bodies.

Test gate failure prevents deployment/recording, drill mutation stays in staging,
unexpected drill success still blocks deploy, blocked recovery bypasses no-op, and live
verification retries/fragment checks/mismatched provenance fail correctly. Mock GitHub
interactions; no live issues from unit tests. Inspect workflow conditions with
YAML-aware tests rather than fragile string matching, including YAML `on`.

Acceptance: Python suite and `just docs-check` pass; workflow syntax is validated; no
workflow can trigger within bob-cli itself. Record any clean-base gate failure exactly
as described in the verification rules below.

## Phase host-controls

Add `tools/docs/host/publish.yml`, the host README, and the host/operator Python module.
The publish caller runs on `17 * * * *` and manual dispatch with boolean
force/drill/pinned inputs, calling
`bobs-org/bob-cli/.github/workflows/docs-publish.yml@master`. Keep logic in bob-cli and
verify the caller/callee input contract.

Implement `host-bootstrap` (preview by default, apply with `-y/--yes`):

- Print the concrete create/sync/Pages/workflow-enable operations.
- Create the exact public host repo if absent, with description and homepage.
- Synchronize only the maintained caller files and README; never overwrite, initialize,
  remove, or reset `published.json` during bootstrap.
- Standalone use may clone over SSH into a temporary directory as the parent specifies.
  For agent use, support an explicit deploy checkout opened through `/sase_repo`,
  avoiding any untracked second checkout. The bootstrap program's narrowly scoped
  commit/push of its rendered host files is part of the approved bootstrap behavior.
  Source edits use normal SASE finalization; agents must not issue manual git commits or
  stage unrelated files through the bootstrap.
- Configure Pages with `build_type=workflow`, ensure effective Actions permission, and
  enable installed workflows. Handle an empty repo/default-branch discovery and re-entry
  after a partially completed bootstrap.
- A second identical invocation makes no file commit. Distinguish 404 from 403 and
  transport errors; never treat arbitrary API failure as permission to recreate.

Implement publish dispatch/watch and status commands, wrapped by the themed recipes
`docs-host-bootstrap`, `docs-publish`, and `docs-status`. Publish enables the workflow
before dispatching, supports the three flags, and watches the run it actually
dispatched, not an unrelated newest scheduled run. Use bounded run discovery, matching
repository/workflow/ref/time and a correlation identifier if needed. Preserve nonzero
run failure status.

Status reports host manifest, live build manifest, recent publish/weekly runs, workflow
enabled/disabled state including `disabled_inactivity`, hold, and open managed issues.
The publication manifest lives in the deploy repo; do not assume it is automatically
served as `/published.json`. A missing first site is a clear status, while
authentication failure is an error.

Document the command examples, rollback with held pinned SHAs, recovery, inactivity, and
pre/post-deployment failure boundaries in the host README and existing Develop build
page. Avoid adding a new nav destination unnecessarily.

Test preview makes no mutations, apply is idempotent, existing manifest and unrelated
files survive byte-for-byte, partial bootstrap recovers, and status/dispatch handle API
failures and run-selection races. Do not run the real bootstrap in this phase.

Acceptance: unit suite and `just docs-check` pass; command helps and just recipes are
usable; the host files exactly match their declared workflow inputs.

## Phase weekly-checks

Add `.github/workflows/docs-weekly.yml` with only `workflow_call` and
`tools/docs/host/weekly.yml` with schedule `41 6 * * 1` plus manual dispatch. Build
strict Material with the locked material dependency group from explicit, clean source
revisions, always running the weekly checks rather than skipping on the publication
fingerprint. Weekly checks can examine current source heads while a publish hold remains
in place, but cannot release the hold or deploy.

Extend the link tool with an online mode using the same pinned lychee resolver. Check
external HTTP(S) destinations with bounded retries/timeouts. Keep offline fragment
checking on the built tree and avoid treating local/source/command URLs as external
requests. Do not add broad exclusions to turn a broken check green.

Keep a separate managed weekly failure issue, with its own label/title, and close only
that issue on weekly success. Reuse publication reporting infrastructure. Weekly has no
Pages deployment, no publication-manifest write, and no unnecessary write permissions
beyond issue reporting.

Test online invocation/exit behavior and issue isolation. Validate both reusable
workflows and both callers with a workflow-aware validator (and shell validation where
applicable). Validate available tool versions without introducing an unnecessary
dependency framework.

Acceptance: docs Python tests and `just docs-check` pass; strict Material build passes;
weekly checks have meaningful local coverage and the host sync includes both callers.
Record concrete external-link failures for launch to repair or triage, not false green
skips.

## Phase launch

Start from landed source changes on the source default branch. Open bob-plugins and
bob-mac-capture with `/sase_repo`, read their instructions, and add the short docs link
near the top of each README if absent. Add the same link in bob-cli. Preserve ongoing
README reorganizations by sibling parent phases. No Swift or plugin runtime changes are
required. Set each public repo's GitHub homepage to the approved URL.

Before creating the host, recheck whether another attempt already created it. Run the
concrete bootstrap preview and apply within the parent's existing authorization. For the
first bootstrap under SASE, create the missing public repo, open it via `/sase_repo`,
and pass that checkout to the bootstrap's sync/apply operation. If an applicable host
rule instead requires source landing through finalization before dispatch, complete that
handoff and resume launch through the host workflow. Do not use a temp clone to bypass
repository auditing or hand-commit rules.

Record evidence in the launch bead and a consolidated note on bob-cli-a0.5:

1. **First publish:** dispatch, wait for completion, record URL/run ID, deployed source
   SHAs, fingerprint, manifest commit, and clean `build.json` source flags. Verify every
   `/urls.txt` page returns 200, each curated fragment is in its live page, footer SHAs
   match `/build.json`, and robots permits indexing.
2. **No-op:** an ordinary unchanged run does not deploy or commit the manifest.
3. **Failure drill:** capture the live manifest and representative page hashes; dispatch
   `drill=true`; verify the real link gate fails, upload/deploy/record are skipped, the
   live site/manifest remain unchanged, and one blocked issue opens.
4. **Recovery:** dispatch normally, without relying on force; verify gates run even with
   the previous fingerprint, deployment/live verification succeeds, and that issue
   closes. Record its URL.
5. **Weekly:** dispatch once and verify successful Material and external checks; record
   the run URL. Confirm the separate issue state.
6. **Bootstrap idempotence/status:** preview or reapply unchanged host templates and
   show no manifest rewrite; confirm hourly/weekly workflows are enabled and status
   accurately reflects live provenance.

Use `/sase_monitor` before long `gh run watch` commands or scheduled waits. A yielded
tool session is not completion: poll the handoff command until it exits. Continue from
the exact run rather than dispatching duplicates after a handoff.

Fix publication defects in scope and rerun only affected checks. If unrelated source
content or a real external permission prevents an acceptance item, record the exact
failure and next command as `PROPOSED FOLLOW-UP:` on the assigned bead and `.5`,
preserving all successfully completed evidence. A local test pass alone does not prove
launch or the drill occurred.

## Verification and land responsibility

Each implementation phase runs focused meaningful tests plus the full docs Python suite,
and `just docs-check` for changes to docs/tooling. Run `just check` only if Rust changes
become necessary. Honor linked repositories' own checks for README edits. Use frozen
dependency resolution for production.

Before treating a check failure as caused by this work, reproduce it on the clean base
in an isolated approved checkout. If it reproduces identically, record a
`PROPOSED FOLLOW-UP:` note with the exact command, diagnostic, base SHA, and an existing
task ID if known. Such a failure does not keep an otherwise completed assigned phase
open. Never create task beads from these phase workers.

Before any assigned phase closes, run `sase bead epic-symbols <assigned-id>`. Resolve
entries or re-key them to a still-open appropriate bead. At planning time `bob-cli-a0.5`
has no entries; recheck before its eventual close.

The child land agent verifies all five outputs, confirms landed source/default branch
state and the live evidence, and consolidates any proposed follow-ups for the parent
epic's land agent. Its normal child-epic closure must finish before the authorized `.5`
land continuation attempts `.5` closure, to respect descendant guards. That continuation
runs:

```bash
sase bead epic-symbols bob-cli-a0.5
sase bead close bob-cli-a0.5 --note "<verified run URLs, live provenance, drill/recovery, weekly result, and any clean-base failures>"
```

This is preparation and evidence for that land continuation. It never authorizes a child
worker to close ancestors, and it never authorizes closure of `bob-cli-a0`. Use
`/sase_final` as the last action of normal provider turns; a successful
`sase plan propose` or monitor handoff follows its own termination protocol.
