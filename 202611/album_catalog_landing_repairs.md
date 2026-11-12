---
tier: tale
title: Finish guarded album creation and land the catalog epic
goal:
  Close the destination and claim guard gaps, verify integrated album behavior, and
  normally close bob-cli-k3.6.1 and parent phase bob-cli-k3.6.
size: medium
proposed_by: bbugyi200.athena.bob-cli-k3.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-k3.6.1
status: done
---

- **PARENT:**
  [202611/album_catalog_and_create.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_catalog_and_create.md)
- **BEAD:**
  [bob-cli-k3.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.6.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-k3.6.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-k3.6.1.land.md)
- **COMMITS:**
  - [7454ff5](https://github.com/bobs-org/bob-cli--plans/commit/7454ff50e20e9faa1a30077fe9fa255cd22f12ef)
    — chore(plans): mark album_catalog_and_create done

# Finish guarded album creation and land the catalog epic

## Scope and verified context

Finish the remaining work in epic `bob-cli-k3.6.1` (Owner album catalogs and guarded
album creation), then complete its landing and close its directly enclosing phase
`bob-cli-k3.6`. This tale has no separate land agent. Its coder must perform the final
closeout in this plan, in the same turn as the last implementation changes when checks
allow it. Do not make closeout depend on this work's future commit SHA, push, or CI.

Read `sase bead read bob-cli-k3.6.1 -r "Need the landing audit and follow-up outcomes"`
and its latest notes first. The approved design is
`plan:202611/album_catalog_and_create.md`; read it with `sase artifact read`. Its parent
`plan:202611/album_and_record_notes.md` governs the catalog, transaction, and
album-catalog phase contracts. The original research is
`research:202611/album_and_record_notes/album_and_record_notes.md`; if artifact access
reports missing, `sase repo open research -r "Refresh the required research context"`
followed by the audited artifact read worked for the lander. The approved plans win
where they refine the research.

The lander read all three child beads and every note, the epic's own discovered issue,
both plans and relevant research, and commits `3dce8a9e`, `0d2f433d`, `aef23542`.
Catalog rows, shared region placement, list/check reports, combined standalone/nightly
sync, section protection, command registration, creation rendering, and the shared
transaction wiring are present. The epic's old no-op-sync issue is addressed by
`sync::plan_sync` consuming `CatalogPlan::edits`, and `report_of` clears installed
catalog stale warnings. Existing acceptance tests cover those normal paths. Preserve
them; do not reimplement the feature.

The current integrated source inspected was `ae6a28d0`. The non-epic changes since the
first epic commit are `b8c71181` (bank docs) and `ae6a28d0` (bank import repairs). The
latter was fetched and fast-forwarded into this checkout. Its restaurant path and
directory guards use the registry's configured directory and its shared transactions
retain their read dependencies. It requires no separate album behavior change. Preserve
all `ReadDependencies` variants, including `directory_entries`, when changing shared
code. Recheck later base drift before finishing.

The findings below are confirmed from the source paths and control flow, not a fresh
runtime reproduction. The lander did not run a current build, `just check`, or
`just docs-check`: the workspace's reported Cargo target has no `debug/bob`, and the
installed binary predates `albums create`. Build the current tree for verification;
never treat the installed binary as evidence for these repairs.

## Constraints

- Keep the existing region and transaction engines: `host_catalog::RegionSpec`,
  `ReadDependencies`, `PlannedTxn`, `apply_and_commit`, `run_locked`, and guarded staged
  writes. Preserve exact authored bytes and intervening edits during rollback.
- This is maintenance/create validation work. Record capture and `records new|update`
  must retain bounded reads and must not touch owner notes or gain whole-vault scans.
- Preserve schema version 1, the existing `index: {state: ...}` JSON shape, CLI aliases,
  notebook behavior, and structural catalog membership for schema-invalid albums.
- Parent decisions remain final: ledger mining accepted elsewhere; only the five named
  glossary strands are authorized elsewhere; `albums-own-record-schemas` was rejected.
  This tale edits no memory, live vault, plugins, Mac app, chezmoi config, release
  version, or changelog. Config retirement remains outer-epic wrap-up work.
- Use temporary vaults, explicit `BOB_DIR`, `BOB_CONFIG_FILE`, isolated cache/config,
  and `BOB_NOW`, with only local test Git remotes. Never use the personal vault.

## 1. Check every destination spelling and guard the inventory

`albums/create.rs::plan_create` currently combines
`snapshot.contains_ignore_case(destination)` with exact-path `symlink_metadata`.
`notebooks/snapshot.rs::collect_markdown` includes only regular Markdown files, so the
first check cannot see `.base` files, directory destinations, or symlinks. On a
case-sensitive filesystem, `EAT.base`, an empty `EAT.md/`, or `EAT.base/` is therefore
missed while creating `eat`. This violates the approved case-insensitive collision rule
including non-file paths.

Read actual vault-root directory entries for collision validation, compare names
case-insensitively, and inspect entries without following symlinks. Refuse either
destination's existing spelling regardless of entry kind. Preserve exact-path no-replace
installation. Handle directory-read/metadata failures explicitly rather than treating
them as absence. Preserve the current refusal of a pre-existing record folder and the
rule that creation does not create an empty record folder.

Carry the namespace evidence into the shared dependencies so an uppercase `.base` or
directory appearing after planning also invalidates installation. Reuse the existing
directory-entry guard where it suffices; do not make the Markdown snapshot pretend
arbitrary files are Markdown notes. Add Linux-effective tests for mixed-case `.md` and
`.base` files, directories and symlinks (including dangling links where supported),
exact collisions, no writes on refusal, and a collision introduced after the real plan
was captured. Test dry-run and real refusal behavior.

## 2. Guard complete catalog/claim inputs and verify actual postconditions

`albums/catalog.rs::catalog_read_dependencies` currently guards existing album bytes,
area/project bytes, and only root Markdown membership. It does not guard the frontmatter
of an existing ordinary root note becoming an album without a filename change. That
changes the claim inventory without changing any recorded dependency. It also omits
non-area/project intermediate ancestry inputs and nested path membership used by
path-aware link resolution. These are missing inputs to the new catalog/create
transactions, not follow-up work outside this epic.

Derive dependencies from the actual catalog and claim inputs used by planning, covering
negative claim evidence as well as current declarations, relevant ancestry headers, and
discovery membership needed to detect changed link identity. Bound the guard scope to
those inputs; record bodies remain irrelevant. Keep root boundaries and path-qualified
identity rules unchanged. Existing ordinary notes changing into albums must be detected
even if the new declaration is broken and contributes no owner row.

Strengthen post-install verification as needed to enforce the planned declaration's
valid claim/schema and the affected catalogs' required state. In `create_run.rs`,
verification currently checks owner/root presence and exact new-file bytes, but ignores
new registry error findings. Both its and sync's host checks currently test only
`edit.is_some()`: damaged hosts have no edit and must not be accepted as current. An
absent host plan after legitimate last-row removal is allowed; distinguish that from
disappearance of a required host or unresolved ownership.

Add deterministic tests that capture a plan, mutate a dependency, then exercise
`ReadDependencies::validate` or `apply_and_commit` on that same plan. In particular:

- an ordinary root note becomes a broken competing noun claim after planning;
- a relevant ancestor header changes or a nested identity becomes ambiguous;
- a destination or host preimage changes after planning;
- verification/commit failure rolls back the newly created album and Base plus owner
  edits together, and preserves any intervening user edit.

Reuse shared transaction failure hooks/test scaffolding. Existing
`create_replans_and_refuses_a_concurrent_album` only changes a file between a dry-run
and a separately replanned real command; its second half actually creates a still
unclaimed noun. Keep valid replan coverage but rename/correct that misleading case and
add real stale-plan and rollback coverage. Do not add a production-only test bypass.

## 3. Make records init prove the config kind is unclaimed

`records/cli.rs::album_claimant` returns `None` whenever the snapshot has any I/O error,
causing `execute_init` to proceed through config scaffolding. A valid or broken album
can thus lose ownership merely because another Markdown note is unreadable or invalid
UTF-8. Replace that ambiguity with an explicit error outcome and refuse before planning
or writing when discovery cannot prove absence. Preserve known-claim teaching refusals
with exit 2 and a useful partial-discovery diagnostic.

The current preflight checks only the requested TYPE token. A config kind whose ID is
`eatery` but whose `dir` is `eat` is still album-claimed when `eat.md` declares noun
`restaurant`; `claimants("eatery")` alone misses that case. Reuse the registry's
existing id-or-directory ownership rule (`claimed_config_type` or an appropriately
scoped shared API), including broken claims. Do not duplicate a divergent ownership rule
in init. A complete, genuinely unclaimed config kind keeps its existing scaffolding
behavior; do not hide/remove init or retire config in this tale.

Add CLI regressions for folder-only overlap, valid and broken declarations, and partial
discovery via a deliberately invalid-UTF-8 unrelated `.md` file. Exercise dry-run and
real runs and assert the exact unchanged file set. Retain the existing unclaimed-config
success case.

## 4. Verify the integrated behavior

Run focused album create/catalog/sync, records init, shared transaction, notebook,
capture-section/completion, projects sync and nightly tests as appropriate to the final
diff. Run a temporary-vault walkthrough (a persistent regression fixture is suitable):
create; capture and `records new|update`; list/check JSON; `@owner#albums` refusal;
reparent plus combined sync; second sync is a no-op. Assert exact changed paths,
owner-row byte stability across record writes, stale findings cleared by real sync, and
unchanged notebook regions. A created album's Base and catalogs must match the dry-run
bytes and the real scoped commit. Use the current compiled binary.

Read `docs.md` through `/sase_memory_read` before documentation/help changes. Update
only observable contracts made inaccurate by these repairs. If CLI help changes, read
`cli_rules.md` too and regenerate with `just docs-gen`.

Required gates are `just check` and `just docs-check`; never run `just check-full`. Use
`/sase_monitor` for long checks and provide the exact remaining closeout steps in its
continuation. A failure independently reproduced on the unchanged base is not remaining
album work: record precise evidence and use existing triage outcomes below instead of
weakening tests. Finish epic-owned failures before closing. Do not claim historical
phase passes as a new integrated gate result.

## Follow-up triage already completed by the lander

Carry these outcomes into the eventual epic close note; do not repeat task creation:

- `bob-cli-k3.6.1.1` #1, `.2` #1, `.3` #1: the two strict-build anchors were
  corroborated on existing `bob-cli-kk` (+1); the mounted yank-flash link was recorded
  on its causal active epic `bob-cli-kb`. Duplicate tasks declined.
- `.2` #2: corroborated existing `bob-cli-kl` (+1) for detached-grandchild pipe-drain
  truncation flakes. Duplicate declined.
- `.2` #3: new large flake task `bob-cli-kp`, live Mermaid render reports zero of four
  under full-suite load but passes isolated runs. The BOB_DIR symptom is included as
  evidence without an invented cause. Related closed `bob-cli-9i` concerns the different
  fixed raw-PDF/pdftotext problem and was not reopened.

These relay independent phase observations, not fresh lander reproductions. The lander
searched matching types and all types, swept all recent tasks, reviewed every active
epic scope, and recorded this full triage on `bob-cli-k3.6.1` before proposing. No child
proposal remains untriaged. The repair items above remain epic-owned.

## 5. Complete the landing yourself

This is the tale's final step, not work for a later land agent.

1. Re-read `bob-cli-k3.6.1` and all three children (`bob-cli-k3.6.1.1`, `.2`, `.3`)
   through `sase bead read ... -r "Need final scope and note readiness"`. Check new
   notes and descendants, the linked plan, final DECISIONS, and any post-audit commits.
   Confirm this tale and the existing phases satisfy every remaining acceptance item.
2. Run `sase bead epic-symbols bob-cli-k3.6.1`. Resolve every listed symbol (wire up,
   privatize, add an allowed non-test pragma, or delete); re-key an exemption only to a
   still-open later bead that really needs it. The lander's earlier result was empty; do
   not rely on that result after edits.
3. Close normally with
   `sase bead close bob-cli-k3.6.1 --note "<concrete source, integration, regression and gate verification, plus all follow-up outcomes above>"`.
   If symbol cleanup blocks close, finish it and retry. If incomplete phases block
   close, finish or reopen them. Never force to make a successful nested landing
   advance. A deliberate cancellation/supersession alone can use the explicit
   user-authorized `--force --reason ... --resolution canceled|superseded` path, never
   to label work done.
4. Run `just symvision` if the recipe is available; the reviewed tree has none. Open the
   plans repo with `/sase_repo` before editing, use the PLAN path reported by the epic
   read (relative artifact `202611/album_catalog_and_create.md`), and set its YAML
   frontmatter `status: done`. Do not edit other frontmatter status or memory.
5. Run `sase bead read bob-cli-k3.6.1 -r "Need the parent link"`. Its parent is the
   phase `bob-cli-k3.6`, not a plan ancestor. Read that phase and verify this child plan
   completed the whole album-catalog scope. Run `sase bead epic-symbols bob-cli-k3.6`
   and resolve/re-key leftovers under the same rule. Close only that phase normally:
   `sase bead close bob-cli-k3.6 --note "<complete catalog/create phase verification and child epic completion>"`.
   Leave the containing epic `bob-cli-k3` and its plan to its already-waiting land
   agent. If the parent is incomplete or ambiguous, note the concrete blocker there and
   report it; never force it closed.
6. Finish through `/sase_final`, declaring every repository changed, including the plans
   sidecar. The host commits after this turn, so do not wait for this tale's commit,
   push or CI before doing the normal closes above.
