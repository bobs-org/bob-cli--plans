---
tier: tale
title: Repair album model validation and finish its landing
goal:
  Correct the album resolver and reports, verify the final tree, and close
  bob-cli-k3.2.1 and parent phase bob-cli-k3.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-k3.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-k3.2.1
status: done
---

- **PARENT:**
  [202611/album_model.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_model.md)
- **BEAD:**
  [bob-cli-k3.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.2.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-k3.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-k3.2.1.land.md)
- **COMMITS:**
  - [ba3f51b](https://github.com/bobs-org/bob-cli/commit/ba3f51bad3bd491d66e739a895ce4de99149329f)
    — fix(albums): confine routes, recover nouns, and restore report diagnostics

# Finish album-model validation and land bob-cli-k3.2.1

## Objective and authority

Repair the bounded defects found while landing `bob-cli-k3.2.1`, then finish that epic
and its directly parented phase `bob-cli-k3.2` in the same implementation turn. This
tale has no separate land agent. Its last implementation step is the concrete closeout
below; do not wait for this tale's own commit, SHA, push, or CI before closing the epic.

Read these through the audited commands before implementing:

- `sase bead read bob-cli-k3.2.1 -r "Need the landing audit and follow-up outcomes"`
- Every child `.1`, `.2`, and `.3` with
  `sase bead read ... -r "Need child scope and notes"`.
- `sase bead read bob-cli-k3.2 -r "Need parent phase scope and final decisions"`.
- `sase artifact read plan:202611/album_model.md "Need the album-model contract"`.
- `sase artifact read plan:202611/album_and_record_notes.md "Need authoritative parent requirements"`.

The approved requirements are unchanged. This tale makes no memory edits, migrates no
real vault, changes no chezmoi configuration, and does not implement the later
capture/records consumer migration, generated indexes/catalogs, or album mutation
commands. Keep `Cargo.toml`'s version and `CHANGELOG.md` unchanged. The parent DECISIONS
permit ledger mining and specific glossary work elsewhere, and reject
`decisions:albums-own-record-schemas`; none of that work belongs in this tale.

This is one coding agent's direct implementation task: the API and report already exist,
the affected seams and failure cases are identified, and no new architecture or
multi-repository implementation is required.

## Verified starting point

The lander reviewed every child note and actual implementation at `ba9867a5`. Epic
commits are `f977009a` (schema), `b6f3b92c` (registry), and `ba9867a5` (reports). A
fetch showed `origin/master` equal to that HEAD. Recheck later drift.

Two intervening commits were reviewed and are compatible:

- `d0ada555` extracts notebook region and transaction behavior into `host_catalog`. Keep
  that engine and the snapshot integration; this tale adds no catalog writer.
- `01c8e822` restores disjoint capture record spans. Keep the single `record_route` span
  and its tests; do not reintroduce overlapping `record_stem` spans.

Both `sase bead epic-symbols bob-cli-k3.2.1` and `sase bead epic-symbols bob-cli-k3.2`
were empty. The current justfile has no `symvision` recipe. These facts must be
rechecked at closeout. The lander ran `just docs-check`; it failed at the six word
budgets described below. No passing Rust gate is claimed for this landing.

## Remaining implementation

### 1. Confine route resolution before opening a file

In `src/native/albums/registry.rs::resolve_route_head`, `route` is interpolated into
`PathBuf::from(format!("{route}.md"))` and joined before validation. The declaration
parser validates only the resulting file stem. Thus `../outside` or an absolute route
can open an album outside the supplied vault and return `AlbumValid` with `TargetOnly`
health. This violates the approved bounded, vault-scoped resolver.

Validate the route as one permitted album stem before any target I/O; reuse the existing
route/stem rules. Reject traversal, absolute paths, separators, dot and reserved routes.
Apply the existing record catalog's confinement policy to symlinked targets so a
valid-looking stem cannot read an external file. Return an explicit refusal/error, never
a valid claim or config fallback for unsafe input. If a supplied known registry belongs
to another effective vault/config, reject that mismatch or load the correct target
rather than returning the other vault's schema. Keep the documented distinction between
full discovery and target-only health. Do not turn the hot path into a vault walk.

Add focused tests with an external sentinel, traversal/absolute routes, a symlink
escape, two vaults, and a normal bounded head read. Verify no unrelated body or record
folder is opened. Keep parent validation based on raw top-level parent facts so local
validation does not lose quote or duplicate-key information.

### 2. Preserve only the actual noun claim through malformed YAML

In `src/native/albums/mod.rs`, `recover_noun_from_yaml` scans every indented `noun` and
requires a closing brace for flow recovery. A source-extracted Rust probe at the
reviewed HEAD produced:

| Record section                                                          | Actual claim           | Required claim |
| ----------------------------------------------------------------------- | ---------------------- | -------------- |
| `record:\n  noun: restaurant # comment\n  fields: [`                    | `restaurant # comment` | `restaurant`   |
| `record: {noun: restaurant, fields: [`                                  | none                   | `restaurant`   |
| `record:\n  fields:\n    thing:\n      noun: restaurant\n    broken: [` | `restaurant`           | none           |

These defects either permit a broken album to fall back to a matching config noun in
another folder or wrongly suppress an unrelated config type. Recover only a
syntactically intact direct `record.noun`, with YAML scalar/comment handling and scope
awareness, independently of later broken syntax. Do not guess a noun from nested field
properties. Preserve the folder claim even when no noun is recoverable.

Test parser results and registry fallback together: use a different config directory so
a surviving folder claim cannot hide a broken noun-recovery test. Cover quoted nouns,
comments, incomplete flow syntax, nested-only nouns, and a healthy unrelated
album/config kind. Keep the normal ordered serde YAML schema parser and all current
schema/config validation semantics.

### 3. Keep report diagnostics and runtime failure state intact

`AlbumIssue::from_finding` in `albums/report.rs` drops `AlbumFinding.path`.
`album_child_reparent` points at the child file, so both JSON and human output now
attribute that child line to the album without saying which child needs attention. Human
`print_issues` also drops schema key paths and correction suggestions.

Carry source path into each report issue and render path, physical line, key path, and
suggestions where available. Keep stable codes and severities. For generated schema
conflict findings, supply the relevant declaration location rather than discarding
information already available from the declaration. Make the informational
child-reparent message suggest the resolved owner, as the approved contract requests;
non-record children outside the album folder remain legal.

`project_album` reads `catalog.folder(...).notes()` but ignores folder diagnostics. A
root `eat.md` album plus a regular file named `eat` gives a complete snapshot while
`RecordCatalog` emits `NoteRead` (not a directory); the report currently presents a
healthy empty album. A record-folder symlink escape can fail similarly. Preserve
folder/member runtime read failures in the report's I/O channel, distinguishing them
from record schema/value findings owned by the later records phase. Missing folders must
remain legitimate empty albums. Invalid Markdown members must remain counted.

`AlbumReport::ok()` currently checks only contract error counts. Make runtime I/O
failure set JSON `ok: false`, including after root filtering. Keep the command exit
rules: runtime failures exit 1; list exits 0 for contract findings; check exits 1 for
contract errors. Keep I/O details observable and do not manufacture a clean empty
report. Avoid converting schema errors into global filesystem failures or blocking
unrelated notebook catalogs.

Add CLI cases for the non-directory folder, confined/symlink failures where supported,
an unreadable/missing vault, a healthy album beside a broken one, a child info finding
with its real source path, and an invalid schema with a spelling suggestion. Assert both
JSON and human output and the corresponding exit codes.

### 4. Honor invalid-album placement and preserve grouping semantics

Preserve the existing typed value-validation semantics in `albums/grouping.rs`: the
legacy records checker also accepts scalar input for list fields. The lander's initial
grouping-shape concern is therefore not a required behavior change. Cover
missing/unknown values and the existing invalid-shape cases without introducing a new
coercion policy. Preserve ordered choices, deterministic title/path ordering, and the
existing large-group `index_by` rules.

The human list groups solely by computed root, so an album with a valid owner and broken
schema stays in the healthy root group. Place entries with error findings in the final
invalid/unowned group, as the approved human contract requires. Retain their computed
owner/root in JSON and root-filter calculations; warnings alone do not make an album
invalid. Cover this with a mixed healthy/invalid fixture.

### 5. Restore legacy diagnostic context

The `f977009a` change to `records/schema.rs::RecordDiagnostic`'s Display implementation
prints `field` and `record_type` only when `config_path` exists. Existing note/value
diagnostics therefore lost field context. A source-extracted probe with field `status`
now prints `record value: unknown choice`, where the old Display included
`, field status`. Restore the historical non-album formatting while keeping album
file/line/key-path formatting. Add a focused regression for config, value/note, and
album diagnostics and retain existing schema tests unchanged.

### 6. Complete and fit the album documentation

Read `docs.md` and `cli_rules.md` via `/sase_memory_read` before editing docs/help. The
current `docs/records.md` adds the album example and report synopsis but omits the
explicit album-wins transition and failure-isolation rules and the full album key
contract. Clarify that canonical records use `[[record]]`, with the noun naming the
record kind and legacy type spelling; current capture/records consumers remain
config-based until `bob-cli-k3.4`. Document required/reserved nouns and stems, optional
fields, label/plural defaults, derived folder/parent, and album-only failure isolation
concisely, linking to the shared field semantics already documented.

Fix the album-caused `docs/reference/json-outputs.md` budget regression without removing
required contract facts, changing the fixed budget, or moving published URLs. The
canonical `bob_docs.markdown.count_words` gives **5983** for both `05ec8253` and
`ba9867a5^`, then **6109** at `ba9867a5`: the album commit added 126 counted words and
crossed the 6000 cap. The child note's raw count of 6357 is not this gate's count. Use
concise descriptions and links to the canonical album contract and retain exact JSON
fields and issue location behavior. Update schema documentation for additive issue-path
changes. Run `just docs-gen` and inspect all generated changes.

## Follow-up triage already completed

The sole proposal was `bob-cli-k3.2.1.3` note #1. The lander invoked `/sase_new_task`,
searched ci tasks and all task types, swept last-week tasks, and reviewed active epic
scopes. The current epic has a durable note recording every outcome:

- Four inherited mounted-page overages were corroborated on **bob-cli-jz**: Mac
  capture-panel 6036, capture-previews 6177, keyboard 6060; plugin navigation-hotkeys
  6186 (source tips Mac `ee26941`, plugins `66841b1`).
- The inherited Mac protocol page at 6149 was corroborated on **bob-cli-h9**.
- Deferring `json-outputs.md` was **declined** because this epic crosses its cap; fix it
  in this tale. No new follow-up tasks were created.

Do not spend this tale editing the linked Mac/plugin pages or creating duplicate tasks.
If those same independently tracked failures remain after generation, record the exact
gate output and source revisions. They are not remaining album work.

## Verification and integration

Re-read current source and recent commits before editing; integrate any new changes that
touch these APIs. Keep the approved parser, registry, notebook isolation, root-walk,
read-only reports, bare/explicit list equivalence, help, command catalog, and completion
contracts intact. Preserve the album-before-area/project/notebook predicates already
present; the mixed-type snapshot regression already covers them.

Run focused album/schema/notebook and album CLI tests while implementing. Strengthen the
read-only fixture to include invalid albums and records and verify vault paths, bytes,
git HEAD/index/worktree state (including no new directories or locks) before and after
both commands. Explicitly test that a broken album's immediate Markdown members still
count, including malformed member frontmatter. These were required acceptance cases
absent from the current CLI fixture.

Then run **`just check`**, **`just docs-gen`**, and **`just docs-check`** on the final
tree. Never run `just check-full`. Use `sase tool run check` and the SASE monitor
procedure if verification escalates; preserve a continuation that finishes the closeout
below. A check-full-only failure would be test infrastructure, not album work. All
failures caused by these changes must be fixed. The five known mounted budget failures
above may be reported as independently tracked baseline failures; the album's own
JSON-reference overage must be gone. Do not claim a full docs pass when its lint gate
still stops on inherited failures.

## Final step: close this epic and only its parent phase

Complete these actions in the implementation turn after the final tree is verified; none
depends on this work's future commit, SHA, push, or CI result:

1. Re-read `bob-cli-k3.2.1`, every child and note, linked plan
   `plan:202611/album_model.md`, and `bob-cli-k3.2`. Verify all descendants remain
   completed, all planned acceptance work is addressed, and intervening commits are
   integrated. Carry the audit and follow-up outcomes above into the close note, adding
   the actual passing checks and remaining independent docs failures.
2. Run `sase bead epic-symbols bob-cli-k3.2.1`. Resolve each listed symbol by wiring it,
   privatizing it, adding a justified non-test pragma, or deleting it; re-key a justfile
   exemption only to a concrete still-open later bead that still needs it. Do not leave
   any exemption keyed to this epic or its closing phases.
3. Close normally with
   `sase bead close bob-cli-k3.2.1 --note "<what was verified, repaired, integrated, tested, and triaged>"`.
   If closure rejects stale symbols, fix them and retry. If it names unfinished phases,
   actually finish or reopen them. Never force closure merely to succeed, and never use
   `--force` to advance a successful nested landing.
4. Run `just symvision` if available. If still absent, explicitly record that fact. Open
   the plans repository with `/sase_repo` and use the exact path it prints; set
   `status: done` in the frontmatter of the epic's PLAN file, `202611/album_model.md`
   (canonical ref `plan:202611/album_model.md`). Read artifact contents only through
   `sase artifact read`. Do not update another plan's status.
5. Inspect the parent link again with
   `sase bead read bob-cli-k3.2.1 -r "Need the parent link"`. The expected parent is
   phase **bob-cli-k3.2**. Verify the now-complete child satisfies that phase's
   album-model scope, including the API/test handoff to `bob-cli-k3.4` already recorded
   in its notes. Run `sase bead epic-symbols bob-cli-k3.2` and resolve or legitimately
   re-key any entries, then close **only** that phase with
   `sase bead close bob-cli-k3.2 --note "<verified album-model scope, registry API handoff, integration and checks>"`.
   Run symvision if available. Leave containing epic **bob-cli-k3** and its linked
   `plan:202611/album_and_record_notes.md` to its already-waiting land agent. If the
   actual parent differs or its readiness is ambiguous, record the precise blocker on
   that parent and report it instead of forcing a close.
6. Finish through `/sase_final`, declaring every edited repository, including the plans
   sidecar. The host creates the code/plan-status commits after finalization; that is
   compatible with closing the epic and phase now. Report the actual closure results and
   verification limits to the user.
