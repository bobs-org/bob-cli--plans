---
tier: tale
title: Give the research-file hook stable names and deliberate recapture modes
goal: Prevent hash-suffixed duplicate captures, select overwrite or reread from the
  newest reading, and record a safe proposal for the five existing references.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3
status: done
---

- **PARENT:**
  [202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3.md)

# Give the research-file hook stable names and deliberate recapture modes

Implement assigned phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3`, which owns the
work for `bob-cli-cm`. The parent decision `hook_dedupe = implement` is final. This is
one bounded coding task: a chezmoi hook adapter, its offline tests, and a read-only
proposal for the five existing hash-suffixed references. The phase had no earlier work
notes when inspected, and neither the adapter nor its tests had landed.

Leave `bob-cli-cm` and every ancestor open for the parent lander. Close only the
assigned phase after completing this plan. Do not create task beads or change bead
status by hand; put discoveries in `PROPOSED FOLLOW-UP:` notes on the assigned phase. No
memory edits are authorized or needed.

## Evidence and boundaries

Read the phase again before implementation, and use audited reads for its sources:

- `plan:202610/backlog_batch17_macos_gate_source_backfill_hook_dedupe_dirty_scans_swift_format.md`
- `plan:202610/ref_create_force_recapture.md`
- `research:202610/ref_create_recapture_modes/ref_create_recapture_modes.md`

Use `/sase_repo` to open `chezmoi`, then read its `AGENTS.md`. Paths below are relative
to the named repository. The planning baseline was bob-cli `c2b69089` and chezmoi
`027a28b9`. Check current history and dirty state before editing rather than replacing
concurrent work.

The cause and available interfaces are already established:

- Chezmoi `home/dot_config/sase/sase.yml` currently sets the research-highlights command
  to `bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT"`.
- Bob `src/native/highlights_ref/create.rs::plan_markdown` takes its default name from
  the input file stem. `-N` overrides it. Title identity is checked at the candidate
  stem, so `<report>-<12 hex>.md` escapes a capture at `<report>.md`.
- The current `sase-research-artifacts` plugin already excludes artifact-copy producers.
  Its hook accepts research `ADD` events from `commit`, `sdd`, and `finalizer`; its
  existing draft, narration, diagram, and infographic exclusions must stay intact. Do
  not claim the historical duplicate-producing route remains enabled today.
- SASE appends one shell-quoted absolute report path and exports
  `SASE_FILE_HOOK_PROJECT`. Its runner does not export an original basename or a
  separate artifact digest. The inspected sources were
  `gh:sase-org/sase-research-artifacts` (`README.md` and `provider.py`) and
  `gh:sase-org/sase` (`src/sase/file_hooks/runner.py`); open them through `/sase_repo`
  before reading if needed. Neither needs a code change.
- Bob's existing `ref show -f json` provides `refs`, `status`, `status_sync`,
  `snapshot.highlights_count`, `annotation_count`, `reading`, `earlier`, and `later`.
  `later` contains actual family paths ordered oldest first; a filename resembling
  `__v2` alone is not family membership. JSON failures distinguish `unknown_reference`,
  `ambiguous_reference`, `missing_ref_dir`, and `read_failed`.
- `-N` selects the exact overwrite member. Always sending the root with overwrite would
  replace v1 even after the family acquired a newer reading.

Keep this policy in the chezmoi hook; Bob's ordinary Markdown naming, family allocation,
write guards, and public CLI/JSON contracts already provide the necessary operations. Do
not expand the plugin's event filters or add a new recapture API.

## Implement the adapter

Add a small executable such as `home/bin/executable_bob_research_capture`, using
Python's standard library for JSON and subprocess argument arrays. Change only the
configured hook's `command` to call that executable. Preserve its `use` entry and all
provider filtering.

The adapter receives exactly one readable Markdown path. Reject invalid input and a
missing/blank project with a clear nonzero diagnostic. Derive a stable stem by removing
only a terminal `-[0-9a-fA-F]{12}` from the basename before `.md`; preserve the rest,
including ordinary dashes and `__` suffixes. Reject an empty or invalid result rather
than silently inventing a different name. Bob's `-N` currently accepts
`[A-Za-z0-9][A-Za-z0-9_.-]*`. Keep the original report path as TARGET so relative assets
and saved-source capture retain their context. Do not rename, move, copy, or rewrite the
input just to change its output stem.

Resolve the existing reference through Bob's JSON interface, with a precise stem or path
query and validation of the returned reference. A broad title search or a fuzzy match
must never choose an overwrite target. Check successful envelopes and field types; only
a real `unknown_reference` is an ordinary miss. Ambiguity, inaccessible vaults,
malformed JSON, unsupported schemas, and failed lookups stop visibly before any create
call. Do not reinterpret such failures as an empty library.

When a reference exists, follow Bob's explicit `later` paths and select the newest live
reading using the returned reading numbers. Query those paths precisely and reject
inconsistent or ambiguous results. Do not calculate a new `__vN`, infer family
membership from filename suffixes, or assume an exact root lookup returns the newest
member. Use the following policy:

| State                                                                                                                   | Create invocation                                                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No filed reference                                                                                                      | Original report, `--include-id`, `-N <stable-stem>`, `-P <hook-project>`, and bare `-f`; on a fresh capture force is a no-op, and on a queued-only collision it keeps Bob's existing replacement behavior. |
| Newest filed reading is `ready`, synchronized, and has a known numeric zero highlight count with no indexed annotations | Original report, `--include-id`, `-N <newest-member-stem>`, `-f overwrite`.                                                                                                                                |
| Newest filed reading is in another status, is annotated, or has unknown highlight metadata                              | Original report, `--include-id`, `-N <family-member-stem>`, bare `-f`; let Bob allocate the next reading.                                                                                                  |

Do not treat null, absent, boolean, or malformed highlight values as zero. Existing
readings inherit their parent through Bob's recapture defaults; avoid forcing the hook
project onto an overwrite whose reading task has since moved. The hook project continues
to supply the parent for a first capture. Preserve Bob's precondition failures,
diagnostics, and exit status. In particular, do not retry a refused overwrite with a new
name or bypass dirty-path, annotation, lock, or coordination checks.

Print one concise policy diagnostic identifying the stable stem and chosen mode so
SASE's hook log makes the action reviewable. Never parse Bob's human hint prose to
choose the mode, and never shell-evaluate a path, project name, or JSON field.

## Regression coverage

Add an offline test under chezmoi `tests/bash/` (for example
`bob_research_capture_test.sh`), which `just test-bash` already discovers. Stub `bob` on
PATH, return realistic JSON envelopes, and record argv without losing argument
boundaries. The tests must never invoke a real capture or access the live vault.

Cover these observable behaviors:

1. Two different terminal artifact hashes yield the same `-N`; an un-hashed name and a
   nonmatching dash suffix remain unchanged. Test paths with spaces in their parent
   directory, preserved `__` components, and invalid/empty inputs.
2. First capture and queued-only capture use the stable name, original source path,
   include-id, and exact project argument; missing project causes no writes.
3. A ready reading with confirmed zero highlights selects overwrite; next, wip, read,
   abandoned, positive highlights, and unknown counts select reread. A status conflict
   must not qualify for overwrite.
4. With a read root and a ready, unannotated v2, overwrite targets v2. With an unread
   root but a read or annotated v2, use reread. Natural version-looking filenames
   without lineage remain standalone, and the hook does not allocate v3 itself.
5. Ambiguous/malformed/error responses, a vanished later member, and unsupported schemas
   produce no create call. A create failure retains its exit status and does not trigger
   a fallback write.
6. Paths and project strings containing shell metacharacters remain single literal
   arguments. The configured command points to the helper, and the provider entry and
   event exclusions remain intact.

Run the focused test, `just test-bash`, and `just check` in chezmoi. Run bob-cli's
`just check` as required by the parent epic; use `$CARGO_TARGET_DIR` for any Rust build.
Use `/sase_monitor` for long checks rather than leaving a process running at turn end.
Never use `just check-full`. If a failure reproduces identically on the clean base,
record its evidence and any existing task reference as `PROPOSED FOLLOW-UP:` on this
phase; it is not a reason to leave the phase open. Do not fix unrelated failures.

## Propose the five existing-reference repairs

Use `/bob_ref` for read-only library queries and the audited memory reads for
`decisions:reference-ids-follow-file-stems`,
`decisions:reference-renames-keep-identity`, and the reference note/task glossary. The
planning survey used `bob ref list -t chat -R all -A -f json`, filtered locally before
printing; all 493 matching rows were returned without truncation. Recheck these five
rows and their counterparts before writing the proposal:

| Hash-suffixed stem                                         | Verified planning state                                                                                |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `conditional_launch_admission-ad048d84997e`                | Read, zero indexed annotations; same-title clean-stem reference exists and has a saved-source pointer. |
| `finalizer_integrity_and_capabilities-460733ec5bb6`        | Read, zero indexed annotations; same-title clean-stem reference exists and has a saved-source pointer. |
| `agent_data_decks_and_cards__gem-4de479cbc0d1`             | Read, zero indexed annotations; no same-title counterpart in the current chat library.                 |
| `memory_history_versioning_and_pager_ux__gem-51b8163fc3b3` | Read, zero indexed annotations; no same-title counterpart in the current chat library.                 |
| `finalizer_completion_contracts-9c652d326895`              | Abandoned, one annotation; no same-title counterpart in the current chat library.                      |

All five currently have PDF bundles, no saved-source pointers, and no recorded family
lineage. Do not repeat the older survey's assumption that every row has a surviving
same-title counterpart. Do not infer equal PDF content from equal titles or hashes in
filenames.

Write a durable Markdown proposal, register it with `sase artifact create --bead` for
this phase, and cite it in the phase note. For each row specify the surviving reference
or intended family, the proposed rename or archive disposition, evidence, affected
companions/tasks/links, and the exact preview or gated repair steps. Use `bob ref show`
to inspect authored content where needed. A read-only rename preview is appropriate for
an eligible unannotated reference.

Renaming a duplicate to `<root>__v2` alone does not establish a family: `earlier` links
are the source of truth and birth derivation applies to newly ingested readings. Do not
propose that a rename alone fixes lineage. Prefer an archive proposal when there is no
justified family merge; preserve unique content, annotations, reading task identity, and
inbound links in the proposed disposition. The annotated fifth row cannot use the
ordinary full-rename route. State its preservation and coordination requirements
explicitly rather than supplying a command known to refuse. This work does not add a
generic archive command or perform a vault migration.

Execute no live repair in this coding task. Such writes require commands of a
Bryan-approved `/sase_gate` turn under the parent epic. A complete, evidence-backed
proposal left for the lander satisfies this phase's repair obligation; lack of repair
approval does not leave the code phase open.

## Landing, deployment, and closure

Review and record the final diff and test outcomes. The approved parent specifically
requires the chezmoi change to be committed through `/sase_git_commit`; use that skill
for this repository obligation, never raw git commit. Do not commit unrelated work. Do
not edit Bob's version, `CHANGELOG.md`, generated skills, or any `AGENTS.md`.

Before applying anything, run `chezmoi diff` for the hook config and new executable
using the opened source checkout (respect its `.chezmoiroot`, which is `home`). Verify
that the proposed deployed diff contains only this change. If so, apply only those
targets and verify the installed helper/config. Otherwise leave deployment to Bryan and
record the exact target diff and commands for the lander. The parent's specific
diff-and-target restriction takes precedence over chezmoi's generic whole-home
`chezmoi update -a --force` instruction; do not deploy unrelated home changes. Do not
smoke-test deployment by capturing a live report.

Append a phase note containing baseline, files changed, commit, tests, deployment
verdict, repair proposal reference, and limitations. Leave `bob-cli-cm` to the parent
lander. Immediately before closing run:

```bash
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3
```

The planner found no entries. Resolve any that have appeared, or re-key them to a
still-open owning bead, before closing only:

```bash
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.3 --note "<implementation and verification evidence; deployment and repair-proposal disposition>"
```

Use `/sase_final` as the last action before a normal final response. Do not close the
parent epic, an ancestor, or the underlying task, and do not inherit the backlog
campaign's successor-planning obligations from its lander.
