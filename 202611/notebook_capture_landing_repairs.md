---
tier: tale
size: medium
title: Repair notebook capture integrity and finish its landing
goal:
  Fix the remaining notebook capture contract violations, verify the integrated tree,
  and close bob-cli-j8.3.1 and its delegated parent phase bob-cli-j8.3.
proposed_by: bbugyi200.apollo.bob-cli-j8.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.3.1
status: done
---

- **PARENT:**
  [202611/notebook_capture_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/notebook_capture_notes.md)
- **BEAD:**
  [bob-cli-j8.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.3.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-j8.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-j8.3.1.land.md)
- **COMMITS:**
  - [843642f](https://github.com/bobs-org/bob-cli/commit/843642f9fc34e0edf60b126751bb221d2fe6283c)
    — feat(capture)\!: repair notebook capture landing

# Repair notebook capture integrity and finish its landing

This is the remaining work from the landing audit of **bob-cli-j8.3.1**, not a
replacement implementation. One coding agent can finish these bounded corrections in the
existing grammar, policy, writer, and tests. The tale coder also performs the landing in
its final step; no land agent resumes after this tale.

## Context and boundaries

Read the current bead and all later notes with
`sase bead read bob-cli-j8.3.1 -r "Need landing findings and current descendants"`. Read
these accepted plans through `sase artifact read`:

- `plan:202611/notebook_capture_notes.md`: the child epic and its complete acceptance
  tables; its plan file is `202611/notebook_capture_notes.md` in the plans repository.
- `plan:202611/bob_notebooks.md`: the binding parent contract, especially Capture and
  `capture-notebook-notes`.

Audit baseline: `12692ebb`, equal to freshly fetched `origin/master`. All four original
phases are closed and have one implementation note each. Their commits are `d9066754`,
`3f8c49d4`, `8d098e55`, and `12692ebb`. Individual/list/global/forced notebook routes,
note writes, summaries, opt-in targets, and Similar-task suppression already exist.
There is no transitional execution refusal. Preserve that work.

The parent phase is **bob-cli-j8.3**, not the containing epic **bob-cli-j8**. The
accepted `collapse_assist=yes` decision belongs to sibling j8.5; creation belongs to
j8.4; Mac, plugins, migration, catalogs, and memory wrap-up remain sibling scope. Do not
re-ask those decisions or implement those features here. No memory edits are planned.
The research artifact `research:202611/bob_notebooks/bob_notebooks.md` was still missing
on the audited retry; the two approved plans supply the contract.

Read `docs.md`, `cli_rules.md`, `decisions:capture-sigils-say-where-what-how`, and
`decisions:capture-bullets-mirror-the-task-block` through `/sase_memory_read` before
changing the relevant contract/help. No package-version or CHANGELOG edits.

## Findings to repair

The lander executed the exact source functions in small standalone Rust probes. These
were helper reproductions, not a full CLI build or `just check` pass:

| Finding                                       | Current implementation and evidence                                                                                                                                                                                                                                                                          | Required result                                                                                                                           |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| CRLF corruption                               | `capture/notebook_note.rs::split_lines_keep` retains `\r`, then `join_lines` joins with `\r\n`. Round-tripping `# Notebook\r\n\r\n- Existing\r\n` yields `# Notebook\r\r\n\r\r\n- Existing\r\n`.                                                                                                             | Preserve original bytes and add new lines in the document's line ending.                                                                  |
| Insertion inside code                         | `region_bounds` and `last_note_block_end_in_region` scan raw lines without fence/frontmatter context. For `# Notebook\n\n```md\n- Example\n## Fake\n```\n\n## Real\n`, the extracted helpers choose insertion before `## Fake`, inside the fence.                                                            | Only real note blocks/headings determine placement; preserve whole nested blocks.                                                         |
| Sub-note detail is flattened                  | `plan_notebook_sub_note` renders authored detail at `parent_indent + unit * depth`, the same level as the newly inserted child. Clipboard planning receives that same child indent.                                                                                                                          | The new child is one level below its parent, and its authored/clipboard detail is another level below it.                                 |
| Anchor identity is inconsistent               | `find_notebook_parent` counts only eligible bullet matches, so a heading and note sharing an ID can still select the note. It also rejects plain notes with `logical.contains("#task")`. New sub-note content has no within-block duplicate set. The top-level and sub-note paths use different ID scanners. | Resolve one actual note anchor, reject ambiguous live anchors and repeated newly written anchors, and permit literal code/tag/link prose. |
| Task-shaped text can be written               | `notebook_sub_bullets` preserves child bodies and the writer emits `- {body}` without a checkbox guard. Lists and forced routes reach the same path.                                                                                                                                                         | Never author checkbox rows into notebooks; fail before any mutation with teaching copy.                                                   |
| Leading clip loses text                       | `body_without_marker` returns only the prefix whenever a route starts after byte zero; `%c @nb Keep this` therefore loses `Keep this` after directive stripping.                                                                                                                                             | Shared strict/editor classification preserves the full note body for valid leading/trailing directive combinations.                       |
| Global inheritance claims explicit operations | `classify_item` falls back to the notebook global for any item without an edge route; its early operator exclusion covers only transfer/merge. Standalone `%n` and session operators can therefore reach notebook refusals or note creation.                                                                 | Only eligible unrouted text/URL items inherit a notebook global; explicit local operation modes retain their meaning.                     |
| Incomplete preimage guards                    | `capture_notebooks.rs` reads notebook parents/owners into a temporary cache but stores only target notebook bytes in `NotebookRoutePolicy.sources`. The writer checks the target, never those metadata inputs.                                                                                               | Every consulted routing metadata preimage is guarded through the existing batch transaction.                                              |
| Resolver and wire drift                       | Section exact/near-miss/create resolution is private to the writer. `capture-parse` can report no near-miss diagnostic for `@nb#gi` with `## Git`. The wire emits new `notebook_*` span labels, despite the accepted existing labels plus notebook modifier contract.                                        | Share section semantics across strict/editor/native planning and restore the accepted additive span contract.                             |

The existing `tests/cli/capture/notebook_notes.rs` has 11 tests. Its CRLF,
nested-detail, clipboard rollback, and retype acceptance cases are absent. Several tests
exercise only dry-run summaries. Add real behavior assertions, not tests that reproduce
the helper implementation.

## Implementation

1. **Replace the writer's ad hoc structural editing with byte-preserving placement.**
   Work primarily in `src/native/capture/notebook_note.rs`. Reuse the repository's
   line-span, heading, fence, indentation, and insertion utilities where their semantics
   fit. Do not fix CRLF by normalizing the entire file. The pre-H2 region skips YAML,
   preserves its H1/preamble, and appends after the last real top-level note's whole
   block. A section ends at the next real H1/H2, not a fenced heading; deeper headings
   remain inside it. Preserve no-final-newline behavior and existing nested detail. Fix
   the relative indentation of authored and clipboard children under sub-notes at
   arbitrary depths, retaining the file's established indentation unit.

2. **Use consistent actual-anchor and notebook-body validation.** Share an anchor
   collector/lookup between note and sub-note planning. Count actual anchors across all
   eligible Markdown blocks, not caret words, inline code, or link destinations; skip
   fenced examples. Before choosing a parent, detect a duplicated anchor even if the
   other occurrence belongs to a heading or another block kind. Reject task/checkbox and
   heading parents, but accept a plain note that discusses `#task` or contains code.
   Verify uniqueness against staged bytes and within each new block, including authored
   children and sub-notes. Do not reuse the older notebook checker's known-bad caret
   scanner (see follow-up triage below).

   Validate the text that will actually be inserted so a list item, ordinary note,
   sub-note, forced route, or clipboard expansion cannot introduce a live checkbox.
   Protect legitimate fenced code and escaped/literal prose. Reuse the existing
   notebook/task shape helpers where correct, with `notebook_holds_no_tasks` or
   `notebook_marker_unsupported` teaching diagnostics and the resolved owner route. An
   invalid owner chain must not invent an owner: use the established area/project and
   `bob notebooks check` guidance. Align strict/editor refusals for authored syntax;
   clipboard validation occurs after its content becomes available but before commit.
   Keep child bullets opaque to task drawer normalization.

   In `capture_language/notebook.rs`, repair body extraction by preserving the ranges
   around the selected marker and removing only recognized syntax. Cover
   `%c @nb Keep this`, `@nb Keep this %c`, and `Keep this @nb %c`, including literal
   sigils and inline code. Do not weaken ordinary task, record, or URL grammar.

   Restrict global notebook claiming to the already-approved inheritable item classes.
   Test `%n`, close/start/adjustment operators, and explicit task links in a draft with
   `@@nb` against their existing operation semantics. Do this before note claiming,
   using shared lexical/operation classifiers rather than parsing a task and repairing
   it afterward. Keep unrouted URLs eligible as plain notes and local routes dominant.

3. **Finish the shared policy, section, and span contract.** Retain the
   invocation-local, pure grammar policy. The native builder must keep preimages of
   every metadata file used for notebook classification/title/owner/ breadcrumb,
   including notebook ancestors. Register them with `BatchReadGuards` using the
   classification-time bytes, not fresh bytes that hide drift. Reuse the existing
   transaction validation at planning/commit boundaries; staged writes within the batch
   must not spuriously fail. Retain zero notebook I/O for an unrouted draft and one read
   per distinct candidate/shared ancestor within the invocation. Templates already
   preflight expanded routes: preserve that and test it.

   Move exact/whole-slug/near-miss/create resolution into one reusable pure operation,
   supplied with H2s derived from the same snapshot as route metadata. Parse diagnostics
   and the writer consume it; the writer resolves against staged contents so later items
   can see newly added sections. Expose the shared result to native completion without
   implementing j8.5's section/create or note-parent picker UI.

   Restore the promised wire labels: `route`, `section`, `task_block_id`, and
   `sub_bullet_block_id`, with an additive notebook modifier on notebook spans. Preserve
   byte ranges, incomplete states, quoted carets, list/global inheritance, and existing
   non-notebook JSON. Keeping internal notebook span kinds is fine if one common wire
   projection emits the accepted labels and modifier. Update `capture-parse` help and
   documentation that currently advertise `notebook_*` labels. Check current HEAD's
   grammar revision before any required breaking-contract bump; never overwrite a
   concurrent revision or edit Cargo.toml/CHANGELOG.md.

4. **Integrate the intervening work and prove the contract.** The lander reviewed all 12
   non-epic commits since `d9066754`: `d7906813`, `a846a23b`, `b97d8d4c`, `0ed552a5`,
   `b5196709`, `0996111c`, `076fdc48`, `2327590f`, `8728fe28`, `03dfe6de`, `6d063ff5`,
   and `eba2bd59`. Archive pairing/readers/stats changes need no new notebook task
   destinations. Keep using the current capture transaction and its shared guarded
   writer; do not add another filesystem commit path. The `6d063ff5` merge-copy
   correction distinguishes literal code from live identities and task rows: notebook
   parent/duplicate handling must obey the same distinction. Reuse appropriate common
   primitives without copying private merge-specific policy. Retain `8728fe28`'s lexical
   dot-separator behavior and its tests. Do not make capture-rewrite access the vault or
   take over j8.5's `@@` collapse.

   Recheck commits landing after `12692ebb` and reconcile any actual overlapping edits.
   Keep `scan_capture_targets` task-only: only capture-targets and route candidates opt
   into notebooks. Retain notebook-only Similar-band suppression and mixed drafts'
   task/record bands.

   Add focused library tests for metadata drift between policy/planner and
   planner/commit, including a changed owner and a retyped target. Add temporary-vault
   CLI tests with fixed `BOB_NOW` for the findings table, exact resulting bytes, LF and
   CRLF, empty/populated regions, fenced headings/bullets, nested parents, duplicated
   IDs, details containing code/tag prose, and later items referring to earlier writes.
   Verify real writes and dry runs agree on notebook summaries and placement. Exercise
   `%c` and `%t` using existing mocks; prove clipboard/write failure rolls back a mixed
   task/note draft and leaves no files or ref jobs behind. Complete any still-missing
   rows of the approved success/refusal table, including forced `--route/--section/-t`,
   ordinary-area `#sec^id`, Unicode offsets, task-syntax refusals in lists, URL notes,
   and target inclusion/exclusion. Prefer table-driven fixtures where behavior aligns.

5. **Validate the final implementation and update only affected documentation.** Run
   focused tests, then **`just check`**. Never run `just check-full` for this tale; a
   check-full-only failure is infrastructure, not remaining epic scope. Run
   `just docs-gen` for changed help and `just docs-check` for the contract/reference
   changes. Keep docs/capture.md, docs/notebooks.md, capture-syntax, JSON, and
   compatibility copy consistent without broad rewrites. Use `/sase_monitor` for
   commands that exceed the synchronous limit and resume this same plan with results. Do
   not claim success from a running process or stale binary.

6. **Perform the complete delegated landing in the same turn as the final code.**
   Re-read `bob-cli-j8.3.1`, every descendant and note (including this tale), and
   `bob-cli-j8.3`. Verify implementation, all original acceptance requirements, the
   fixed findings above, decisions, and post-audit drift. This closeout does not depend
   on this turn's future commit SHA, push, or CI result. Finish and close this tale's
   own assigned bead normally after its implementation and verification are complete, so
   it cannot remain an open descendant when the epic closes; continue with the delegated
   closeout before `/sase_final`.

   Run `sase bead epic-symbols bob-cli-j8.3.1`. Resolve every listed entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting it according to the
   Symvision policy; re-key only when a specific still-open later bead needs it. Repeat
   the command until none remain. Then run:

   `sase bead close bob-cli-j8.3.1 --note "<source/commit/decision verification, fixes, just check and docs results, integration findings, and every follow-up outcome>"`.

   The note must include the triage below. Run `just symvision` if available (there is
   no such recipe at the audit baseline; record that fact if still absent). Open the
   plans repo with `/sase_repo`, use only its printed path, and set **`status: done`**
   in **`202611/notebook_capture_notes.md`**, the plan linked by the epic read. Never
   edit the plan or a sidecar artifact through a guessed checkout path.

   After the epic closes, run `sase bead read bob-cli-j8.3.1 -r "Need the parent link"`.
   Its parent is phase **bob-cli-j8.3**. Verify this child completed that phase's full
   capture scope, retire entries from `sase bead epic-symbols bob-cli-j8.3`, and close
   **only that parent phase** with
   `sase bead close bob-cli-j8.3 --note "<verified capture scope, child completion, integration and test evidence>"`.
   Leave containing epic **bob-cli-j8** and `plan:202611/bob_notebooks.md` for its
   waiting land agent. Do not mark the containing plan done.

   If a normal close is rejected, finish the named symbol/descendant work and retry;
   never use `--force` to advance a successful nested landing or simply to make a
   command succeed. If parent scope is incomplete or ambiguous, keep that parent open
   and append a concrete blocker note. Finish through `/sase_final`, declaring commits
   for every changed repository, including the plans repo. Use a breaking Conventional
   Commit subject if the wire/grammar correction requires it.

## Follow-up triage already completed by the lander

None of j8.3.1.1 through j8.3.1.4 contains a `PROPOSED FOLLOW-UP:` entry. All findings
in the repair table were introduced by this capture epic and remain its work, not new
tasks.

One separate pre-existing defect was discovered in `notebooks/rules.rs`: its
`extract_block_ids` reads inline-code and wikilink carets as anchors. Executing the
exact function on this line returns `["x", "x", "real"]`, a false duplicate:

```markdown
- Discuss `^x` and [[nb#^x]] ^real
```

It came from notebook-core commit `0a74731b`, phase **bob-cli-j8.1**, before this child
epic. `/sase_new_task` searched bug tasks and all types, swept the past week's tasks,
and inspected active epics and j8.1: no semantic duplicate exists. The lander recorded a
**DISCOVERED ISSUE on active causal parent bob-cli-j8**, per the active-epic rule; no
separate task was created. Preserve that outcome in the close note. That checker repair
belongs to the wider epic; the capture writer must not inherit its flawed scanner.
