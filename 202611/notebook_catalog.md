---
tier: tale
title: Notebook catalogs, sync, creation, and nightly maintenance
goal: Complete bob-cli-j8.2 with stable per-host notebook catalogs, guarded sync and
  creation, and nightly repair.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.2
status: done
---

- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:**
  [bob-cli-j8.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.2.md)

# Notebook catalogs, sync, creation, and nightly maintenance

Implement the remaining scope of assigned phase `bob-cli-j8.2` in bob-cli. An area or
project should show a stable, generated tree of its notebooks; `bob notebooks sync`
repairs those catalogs, `bob notebooks create` creates a notebook and its catalog rows
together, and nightly maintenance keeps catalogs current.

## Authority, starting point, and scope

Read `bob-cli-j8.2` with `sase bead read` before implementing. Its parent design is
`plan:202611/bob_notebooks.md`; read it through `sase artifact read`. The relevant
research is `research:202611/bob_notebooks/bob_notebooks.md`, especially “The Notebooks
catalog” and the creation/read-guard discussion. The approved epic overrides the
research: in particular, write `[!notebooks]` without a color suffix. The research
checkout initially lacked the report during planning; opening `bob-cli--research`
through `/sase_repo` refreshed it, and the audited research read then succeeded.

At planning time, HEAD was `0a74731b`, which landed phase `bob-cli-j8.1`. The tree was
clean, `bob-cli-j8.2` had no prior-work notes, and its epic-symbol query found no
entries. The existing implementation has notebook recognition, ownership, body
validation, `list`, `check`, and `docs/notebooks.md`. `NotebookReport::to_json`
deliberately emits `catalogs: []`. There is no catalog renderer, notebook writer,
create/sync command, or nightly notebooks step. Recheck for subsequently landed work
before implementing.

This is one medium tale: extend one existing scanner and add a shared pure planner, then
two thin command entry points and the nightly adapter. It does not require a child epic.
Do not implement capture grammar, notebook note insertion, plugins, Mac Capture,
migration, or memory changes. The accepted epic decisions stay final; their memory
permissions do not require memory edits in this phase. Do not change release versions or
CHANGELOG.md.

Before changing CLI help or docs, read `cli_rules.md` and `docs.md` through
`/sase_memory_read`. Work on repository source and isolated fixture vaults, not the live
vault. The later capture-creation phase needs the renderer and catalog planning APIs
from this work, so keep those reusable without invoking the CLI or writing to disk.

## Existing integration points

- `src/native/notebooks/{scan,model,rules,cli,output}.rs`: the scanner already makes one
  sorted vault discovery pass, resolves path-aware parent links with `NoteIndex`, and
  uses `HomeResolver`/`IndexedHomeSource`. It currently retains notebook bodies but
  discards area/project bodies. Error-bearing notebooks have null owner/root.
- `src/native/pomodoro_homes/resolve.rs`: `Home` includes `residence_chain` and
  `root_parent`; reuse this authority for root boundaries instead of inventing another
  root algorithm.
- `src/native/capture/commit.rs`: exported `write_staged_files` and `StagedTextFile`
  provide disk-preimage checks, no-replace creation, and rollback. The guarded variant
  currently has private visibility; expose the minimum needed to guard read-only
  ancestors at both validation points.
- `src/native/vault_rewrite/mod.rs`: precedent for lock, vault identity/worktree checks,
  pre-sync, planning, stale checks, scoped commit, failure recovery, and post-sync. Its
  current `run` always acquires the lock, treats an empty commit as a failure, and
  installs through `atomic_write`, so it cannot be called unchanged for nightly, no-op
  sync, or no-replace creation.
- `src/native/ob.rs`: shared maintenance lock, child environment, Git helpers, and
  `commit_paths`, which commits only the supplied paths.
- `src/native/capture_project_note.rs`: `project_note_basename`,
  `format_created_timestamp`, and title helpers. `capture_block_ids` owns ID syntax;
  `records/render.rs` owns the typed YAML scalar serialization precedent.
- `src/native/records/cli.rs`: existing Obsidian-open helper; extract or expose it
  minimally rather than introducing another URI/opener implementation.
- Command registration is in `src/runner.rs`, dispatch in `src/native.rs`, completion in
  `src/native/completion/tree.rs`, and Command Mode metadata in
  `src/native/catalog/table.rs`.

## 1. Shared snapshot and catalog planner

Extend the notebook scan into a reusable snapshot-backed operation while preserving
existing list/check behavior and notebook fields. Retain original bytes for notebook and
host files, path identities, resolved parent edges, and the ancestry/root facts needed
to plan catalogs. Keep one discovery pass and indexed graph traversal, not a vault scan
per host. Propagate I/O failures rather than treating a partial scan as a complete empty
subtree and deleting catalogs.

Separate disk loading from analysis sufficiently that an in-memory overlay can add a new
notebook and change existing staged files before catalog planning. The standalone create
command must use this overlay; the later capture phase can reuse it for a multi-item
draft. Return exact source snapshots/read dependencies with the plan so a caller can
enforce stale-input guards. Use relative paths as graph identities, not ambiguous stems,
while preserving the public stem-based report fields.

Add a pure catalog planner, preferably in `notebooks/catalog.rs` with region handling in
a small separate module. It returns host states, findings, and before/after edits; it
does not acquire locks, sync, write, commit, or print.

Catalog membership follows the approved contract:

1. Every area/project host lists notebooks whose owner is in that host's subtree,
   stopping at another root: any descendant area, any `root: true` project, or an
   outermost project. Reuse the resolved root boundary and ancestry. A non-root
   sub-project gets its own slice as well as appearing in its root's rollup.
2. Put the host's directly owned notebook forest first. Group notebooks owned by
   descendant non-root projects under `🧩 [[project_stem]]`, ordered by project stem.
   Preserve each notebook-parent tree within its owner group. Intermediate non-root
   hosts also include the relevant descendant-owner groups.
3. Within each notebook level, sort by case-insensitive display title, with a stable
   path tie-breaker. Invalid/unowned notebooks remain in check/list but never in a
   catalog. Avoid cycles and duplicate visits.
4. Plan hosts with expected rows or an existing managed region, including old owners
   after reparenting and hosts that have become empty. Hosts with neither need no
   catalog entry. Do not put catalogs inside notebooks.

The generated block uses this exact framing, adapted only to the host's line endings:

```markdown
## Notebooks

<!-- bob:notebooks:begin v1 -->

> [!notebooks]
>
> - **[[dev_tools|Developer tools]]** — Command-line tools and dotfile setup.
>   - **[[git|Git]]** — Git commands, configuration, and workflow tips.

<!-- bob:notebooks:end -->
```

Omit the em-dash tail when the summary is missing. Emit titles and summaries as inert
text, escaping link/wikilink, emphasis, and HTML openers so authored text cannot
introduce active markup or break a row. Keep wikilink destinations intact and valid. Use
deterministic formatting independent of terminal width, dates, and mtimes. Write no
colors, counts, recency, or generated timestamps in catalogs.

## 2. Preserve the host's bytes and position

Use a strict byte-span region parser modeled on Gmail's managed regions, independent of
the Gmail identity/body rules. Preserve a UTF-8 BOM, LF/CRLF, and all bytes outside the
owned region. Heading detection must ignore fenced examples and frontmatter.

- With no markers and nonempty rows, insert the complete block immediately before the
  first real `## Tasks`, or at EOF separated by one blank line. Preserve user-authored
  content, including an existing unmarked heading, rather than adopting its body as
  generated content.
- With one valid begin/end pair, replace only the bytes between the markers in place,
  regardless of where the block was moved. Leave the heading and surrounding prose
  alone. An identical result produces no edit.
- With an empty subtree, remove the managed region including its markers. Remove its
  `## Notebooks` heading only if that section then contains solely blank lines; keep any
  authored prose or other content.
- A lone marker, duplicate begin/end, reversed markers, or unsupported/malformed managed
  fence is `notebook_catalog_damaged`. Do not infer a repair range or overwrite that
  host. Include a useful path and line in the finding.

Catalog state is `missing` for required content without a region, `stale` for differing
or removable content, `current` when expected bytes match, and `damaged` when the region
cannot safely be edited. Moving an otherwise current block does not make it stale.
Reparenting updates both old and new catalog hosts.

## 3. Public report and section protection

Replace the placeholder `catalogs` with deterministic records containing at least
`host`, `path`, `state`, and `notebooks` (the expected recursive row count). Keep the
existing `schema_version: 1` envelope and notebook fields. Host findings may be an
additive `issues` array on each catalog record; do not misattribute them to an arbitrary
notebook. Count them once in report error/warning totals and print them under their host
path in human check output.

Both missing and stale required catalogs produce `notebook_catalog_stale` warnings with
the hint `bob notebooks sync`; damaged catalogs produce `notebook_catalog_damaged`
errors. Empty stale catalogs are also actionable. `list` remains read-only and exits
zero for findings; `check` exits one for errors and zero for warnings alone. Actual scan
I/O errors are failures. Root-filtered list output filters host catalogs by the same
computed root and recomputes counts consistently.

Exclude the generated `## Notebooks` section from area/project section-bullet routing
and section completion. The shared predicate is `is_bullet_section` in
`src/native/capture/sections.rs`; trace explicit selection as well as candidate
generation so `@dev#notebooks` cannot silently write before the first heading or into
the callout. Preserve other section behavior. Add a `bob projects sync` regression
showing notebook children never enter a Sub-projects line.

## 4. Guarded sync and a lock-free nightly core

Add `bob notebooks sync [-b DIR] [-d] [-f human|json]`, with alphabetical options
`--bob-dir/-b`, `--dry-run/-d`, `--format/-f`, and `--help/-h`.

Build a small notebook transaction wrapper using the existing maintenance helpers and
the shared capture staged-file writer. Follow `vault_rewrite::run`'s lifecycle without
copying its rename/journal assumptions or performing a broad rewrite of unrelated
commands. A narrow extraction of common orchestration/recovery helpers is acceptable.

- Dry-run performs read-only planning and prints per-host unified diffs in human output.
  JSON embeds structured changes/diffs. It must not acquire a write lock, sync, commit,
  open Obsidian, or change vault bytes/index/HEAD.
- A real standalone run verifies the requested vault identity/worktree, acquires the
  shared bounded maintenance lock, pre-syncs, then plans from the synced snapshot.
  Revalidate file preimages and relevant read dependencies before installing files. Use
  the existing no-replace/rollback writer; never use unguarded `fs::write`.
- Skip damaged hosts and report them. Healthy host edits may be applied together; the
  overall command still exits one when any host is damaged. I/O failure must not trigger
  catalog deletions based on incomplete discovery.
- Commit changed hosts once, scoped to those paths, with
  `notebooks: sync catalogs (N notes)`, where N is the number of changed host files.
  Unchanged planning is successful and creates no catalog write or catalog commit.
  Ordinary pre/post vault-sync behavior must not be mislabeled as a catalog commit.
- Follow existing recovery semantics for stale inputs, install failure, verification
  failure, and commit failure. Preserve unrelated files/index entries and foreign edits;
  do not blindly restore over a later edit. Distinguish a committed local result whose
  post-sync failed from an unapplied operation, with a retry hint.
- JSON stdout must be one machine-readable envelope, including failures; route sync
  chatter and human progress to stderr. Include `schema_version`, `ok`, dry-run state,
  changed paths, findings, and commit/sync outcome so callers need not parse prose.

Expose a lock-free `sync_with_existing_lock` core accepting the vault and `ChildEnv`. It
plans, guards, applies, and commits catalogs but does not acquire the lock or run extra
pre/post sync cycles. Standalone sync wraps that core. In `nightly.rs`, add `notebooks`
between `period notes` and the final `vault-sync`; nightly supplies its existing
lock/environment and continues later steps after failure as it does today. Update
nightly help, root command description, Command Mode confirmation text, and the order
assertions to six steps.

## 5. Shared notebook renderer and standalone creation

Add
`bob notebooks create PARENT ID -s SUMMARY [-t TITLE] [-b DIR] [-d] [-f human|json] [-o]`.
Options are alphabetical and every public long option has its short alias (`--open/-o`,
`--summary/-s`, `--title/-t` plus the shared options).

Create a pure `render_new_notebook` and a reusable creation planner. Resolve the parent
path with the existing index. It must be an area, a project of any status (including
done/canceled), or a valid notebook chain reaching an owner. Reject missing, ambiguous,
unsupported, or cyclic parents. Validate the ID with the existing block-ID alphabet,
name the new root file through `project_note_basename` (parent + `_` + ID, hyphens
converted to underscores), and refuse case-insensitive collisions against disk and
overlay/staged paths. Recheck absence at install; never replace an existing file,
including one that appeared after planning.

Require a nonblank, single-line summary; more than 140 characters is a warning rather
than a refusal. Title is the explicit one-line title or the humanized ID. Render keys in
order `parent`, `template`, `type`, `summary`, `created`, then a blank line and the H1.
Parent/template/type use quoted wikilinks. Use typed YAML string serialization following
the records renderer, quoting summaries when needed (including strings that look like
YAML booleans/numbers), not raw interpolation. Expose the smallest shared serialization
helper needed without changing existing record rendering. Use Bob's resolved clock and
timestamp helper so `BOB_NOW` fixtures stay deterministic. The renderer may accept a
prepared body for later seeded capture, but this phase implements no seed grammar or
capture dispatch.

Plan the new notebook in the snapshot overlay, then derive every affected host edit from
that same post-create view. Stage the new file and affected catalog hosts as one guarded
transaction. Guard the parent and owner/root ancestry even when those notes are not
rewritten. A damaged affected host refuses the entire create; an unrelated damaged
catalog must not prevent creating under a healthy owner. Creation does not
opportunistically repair unrelated hosts.

Use the same maintenance wrapper for committed standalone creation, with one scoped
creation commit. Validation/refusal is exit two. Dry-run shows the new file plus host
diffs without writes. JSON identifies the new path, title, parent, warnings, planned
changes, and committed outcome with `schema_version: 1`. `--open` uses the shared
Obsidian helper only after successful real creation; failure to open is a warning and
never rolls back a committed file. A failed guarded write leaves no new notebook or
partially updated catalog.

Export the renderer, overlay-based catalog planner, and required request/result types
through `notebooks/mod.rs` with crate-level visibility. Their interfaces must allow the
later capture phase to supply staged contents and compose these edits into its existing
transaction without a second commit or nested maintenance lock.

## 6. Wire commands and document shipped behavior

Register native dispatch, notebook group members (`check`, `create`, `list`, `sync`),
completion descriptors, and appropriate Command Mode entries/policies. Keep bare
`bob notebooks` as read-only `list`. Extend help examples and reference-generation
coverage for both new commands rather than hand-editing generated CLI pages.

Update `docs/notebooks.md` as the contract hub: scope/root boundaries, fence ownership,
placement/deletion/damage handling, all commands and exit behavior, no-op/dry-run
semantics, JSON host findings, and nightly integration. Update
`docs/reference/json-outputs.md`, the section-bullet exclusion in `docs/capture.md`, and
the nightly sequence wherever described in `docs/vault-git-sync.md` and generated CLI
docs. Run `just docs-gen` after CLI help changes, inspect its changes, and keep
documentation additions within the repository's budgets. Do not edit memory files.

## Verification and completion

Add meaningful pure and CLI tests, using temporary vaults, `BOB_DIR`, deterministic
`BOB_NOW`, isolated maintenance-lock paths, and local Git remotes where sync/commit is
tested. Reuse existing fixtures in `tests/cli/notebooks.rs` and
`tests/cli/vault_sync.rs`; add focused modules if needed.

- Golden tree: directly owned and nested notebooks, title ordering, non-root project
  groups and intermediate slices, a descendant `root: true` project, an outermost
  project, and an area beneath an area. Prove each notebook appears in exactly one root
  catalog while permitted non-root slices also show it.
- Region lifecycle: insertion with/without Tasks, moved block, summary/title/reparent
  changes, deletion with and without authored prose, duplicate/lone/reversed markers,
  LF/CRLF/BOM, fenced examples, escaping, and exact surrounding-byte preservation.
- Reports: all four states, host findings/counts, filtered roots, invalid notebooks,
  read-only list/check, stable schema, and no ANSI or progress noise in JSON.
- Transactions: dry-run preserves files/index/HEAD; a second sync preserves bytes and
  creates no commit; real sync creates one scoped catalog commit; lock contention, stale
  ancestors/preimages, concurrent destination creation, injected install failure, and
  commit/post-sync failure preserve truthful outcomes and guarded rollback.
- Creation under an area, active/closed project, and nested notebook; default/custom
  titles; required/long/YAML-sensitive summaries; invalid IDs/parents; mixed-case
  collisions; affected damaged host refusal; unrelated damaged host tolerance; new file
  and catalog consistency; dry-run parity and stubbed `--open` behavior.
- Section routing and completion cannot target the catalog; project sync ignores
  notebook children; nightly has six ordered steps, calls the lock-free core once, and
  still reaches its final sync after a catalog failure.

Run focused notebook/transaction/section/nightly tests while developing. Format with
`cargo fmt --all`, regenerate docs/help goldens as needed, then pass the full `just`
gate (fmt, clippy, all tests) and `just docs-check`. Use `/sase_monitor` for long checks
when handing verification off; wait for the monitor staging command to exit and finish
through `/sase_final`. Do not repeatedly rerun full gates without a new change or
failure to investigate.

Before closing, reread the assigned phase and run `sase bead epic-symbols bob-cli-j8.2`;
resolve any new entries or re-key them to the still-open epic/later phase. Record
verified commands and outcomes, then close only `bob-cli-j8.2` with
`sase bead close bob-cli-j8.2 --note "<what was verified>"`. Never close its parent or
any ancestor plan. Do not hand-set status or reopen the assigned phase. Record
discoveries only as `PROPOSED FOLLOW-UP:` notes on this phase; do not create task beads.
A failure reproduced identically on the clean base belongs in such a note and does not
keep completed phase work open. End the coding turn through `/sase_final` so the host
commits the work.
