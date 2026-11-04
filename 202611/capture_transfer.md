---
tier: epic
title: Capture transfer grammar and execution
goal: Complete bob-cli-go.10 with one-item capture transfers that move to notes or
  merge into tasks, drawers, and sections, preserving atomicity, editor contracts,
  offers, shell safety, and existing capture behavior.
parent_bead: bob-cli-go.10
phases:
- id: transfer-lexer
  title: Shared transfer lexer and shell redirect guard
  size: medium
  depends_on: []
  description: 'transfer-lexer: implement the pure span-aware transfer recognizer
    and its complete grammar vectors, plus the stdout regular-file shell guard. Keep
    the recognizer test-gated until the next phase wires both consumers; do not bump
    grammar yet.'
- id: transfer-resolvers
  title: Execution and editor transfer parsing with the single grammar bump
  size: medium
  depends_on:
  - transfer-lexer
  description: 'transfer-resolvers: wire the shared recognizer before target-run fan-out
    in both resolvers, introduce CaptureKind::Transfer and the editor transfer contract,
    and bump grammar once. Explicitly refuse execution until its adapter lands.'
- id: transfer-moves
  title: Staged transfer moves and shared result fields
  size: medium
  depends_on:
  - transfer-resolvers
  description: 'transfer-moves: resolve and validate transfer identities, execute
    bare-note destinations through batch_move, and report moves and repairs with human
    output. Preserve atomic draft planning, link relocation, freshness, and dry-run
    parity.'
- id: transfer-merges
  title: Staged transfer merges and draft-level offers
  size: medium
  depends_on:
  - transfer-moves
  description: 'transfer-merges: execute task, drawer, and section destinations through
    plan_task_merge with Explicit semantics, preserve guarded write ordering, and
    integrate records, repairs, warnings, and picks into capture''s existing pull_in
    machinery.'
- id: transfer-delivery
  title: Transfer documentation and integrated verification
  size: medium
  depends_on:
  - transfer-merges
  description: 'transfer-delivery: publish quoted help and transfer contracts, regenerate
    documentation, fill cross-item rollback and compatibility coverage, and run scratch-vault
    and repository gates. Supply closeout evidence for this child epic''s land agent;
    no phase worker closes bob-cli-go.10 or bob-cli-go.'
proposed_by: bbugyi200.apollo.bob-cli-go.10
proposed_with: codex/gpt-6-astra
create_time: 2026-11-04 01:50:42
status: wip
bead_id: bob-cli-go.10.1
---

- **PROMPT:** [prompts/202611/capture_transfer.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_transfer.md)
- **PARENT:** [202611/task_move_merge_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_move_merge_split.md)
- **BEAD:** [bob-cli-go.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-go/bob-cli-go.10.1.md)

# Capture transfer lines for bob-cli-go.10

## Ownership and outcome

This is a child implementation epic under the assigned phase **bob-cli-go.10**, not a
replacement for its parent epic **bob-cli-go**. The host creates the child epic and
phase beads through approval; do not create beads by hand. Its land agent finishes the
assigned phase after all five phases and integrated checks complete. No child phase
worker may close an ancestor, and nobody in this child epic closes bob-cli-go. Transfer
completion/rewrite belongs to bob-cli-go.11; Mac rendering and the parent epic's memory
closeout also remain outside this plan.

The deliverable is one pure capture directive:

```text
@inbox+bench @work+pgdump > @db
@inbox+bench @work+pgdump > @db+upgrade
@inbox+bench > @db+upgrade#future-work
@mac_inbox+ws-proto > @dev#future-work
```

A bare-note destination moves tasks, preserving identity. A task, drawer, or section
destination merges their authored content and retires the sources through the existing
merge engine. Every directive is one capture item in the existing atomic draft
transaction.

## Authority and inspected state

Read `plan:202611/task_move_merge_split.md` through `sase artifact read`; its shared
sections 2, 3, 9, and 10 and `transfer-capture` phase are authoritative. The accepted
report is
`research:202611/task_move_merge_split_design/task_move_merge_split_design.md`; its
merge and split sections were read. The report initially failed to resolve, then
resolved after `sase repo open bob-cli--research` refreshed the managed checkout. The
parent plan wins over research where they differ, including how the target's own
dependency on a source recovers.

Planning began on clean commit `3741ce2e`. The assigned phase was already `in_progress`,
with no prior work notes and no epic-symbol entries; no reopen or status change was
needed. Audited prerequisite reads confirmed bob-cli-gm and bob-cli-gl.2 are closed.
Their target runs and move-bullet lexer are present. `497db446` supplies task move/merge
CLI support; `3741ce2e` supplies capture merge-label support. There is no transfer
recognizer or execution adapter yet.

Relevant integration points:

- `capture_language/draft.rs::parse_capture_item_members` selects a target run before
  calling the singleton parser. `editor_parse.rs::parse_editor_item_with` likewise calls
  `try_parse_editor_target_run` first. Both require an earlier transfer claim; inserting
  only into `line.rs` would be too late.
- `capture_language/model.rs`, `editor_model.rs`, `editor_classify.rs`, and
  `capture_parse.rs` own the semantic/editor/JSON shapes. Shared tokenization already
  preserves byte spans and quoted selector text.
- `capture/batch_move.rs` supplies `destination_snapshot`, `apply_batch_move`, freshness
  stamping, snapshot-bound staging, and destination-before-source edges. Reuse it rather
  than invoking a CLI subprocess.
- `capture/relations_merge.rs` demonstrates the merge context, staged overlay, and
  offer-row conversion. Reuse/extract small helpers as needed. The pure merge result
  includes `write_order`, `read_paths`, records, and repair data; its caller must
  preserve these rather than iterating a path-sorted map alone.
- `CaptureItemResult` already flattens a defaulted `RelationMergeOutput` holding
  `merges` and internal offer rows. Extend grouped/defaulted result fields rather than
  duplicating the public `merges` key or copying every result literal.
- `task_move::MoveRecord` is not currently a serde JSON contract by itself:
  `task_move/command.rs` maps it in its JSON printer. Share that exact mapping or a
  common DTO instead of deriving a different wire shape accidentally.

The current grammar constant and current-version prose are 9, with some older example
JSON still claiming 6. Re-scan the tree when the grammar-changing phase starts and
choose one greater than the highest **capture grammar** revision claimed by code or
docs. Do not hard-code the new number here or confuse it with unrelated browser, schema,
or package versions.

All parent DECISIONS stand. This child epic edits **no memory notes**. Authorized
glossary changes remain with the parent closeout; the three declined decision records
must not be created. No Cargo package version or CHANGELOG changes.

## Contract every phase must preserve

### Recognition and refusals

Recognize a leading run of one or more complete `@route+id` sources followed immediately
by a standalone `>` before terminal-marker consumption, target-run fan-out, and
`split_capture_targets` checks. One source is sufficient. Once claimed, a bad or
incomplete destination can never become authored note text or an Ensure Next operation.

The four destinations are `@route`, `@route+id`, `@route+id#drawer`, and
`@route#section`, including quoted `#"Exact Title"` selectors. Preserve source order,
raw markers, absolute byte offsets, and selector literal-vs-slug intent. Incomplete RHS
forms produce the appropriate `transfer_destination`, `block_id`, `drawer`, or `section`
need in the editor; execution refuses without mutation.

Required coded errors are `transfer_arrow_spacing` for glued arrows;
`transfer_extra_text` for extra text or modifiers (including schedule, priority, type,
percent, dependency, and relation markers); `transfer_children`;
`transfer_source_suffix` for source drawers or toggles; `transfer_source_marker` for
`@route:id`, `@route^id`, or `^route:id` sources; and `transfer_destination_marker` for
other destination shapes. Near-transfer recognition must claim malformed spellings to
teach these errors, without claiming ordinary prose, code, or a line-initial blockquote.

`\>` at the arrow position escapes the transfer and drops exactly that backslash in
authored text. Elsewhere `\>` stays verbatim. Mid-line comparisons such as
`called Bob > 3 times` and code stay unchanged. Forced `-r`, `--task`, and `--drawer`
retain existing literal behavior. `@@` never supplies a transfer destination. Transfers
in bullets under a lone route header fail with `list_bullet_marker`.

Lexical duplicate sources use the parent contract's lower-cased route and ID, with
`duplicate_capture_target`. Restrict that rule to transfers: ordinary target runs
deliberately distinguish exact block IDs such as `A` and `a`. Planning checks resolved
`(path, id)` identities to catch aliases, rejects a destination task also present as a
source (`transfer_destination_is_source`), and refuses overlapping source subtrees
(`transfer_overlapping_sources`) before any source is staged. Map these
transfer-specific errors at the adapter boundary without changing the standalone merge
CLI's established error codes.

### Public parse and capture results

Keep `schema_version: 1`. A complete transfer's mode is `task_move` or `task_merge`;
legacy `route` and `block_id` describe the destination. Add the following optional
object on the same per-item and singleton surfaces used by the existing parser, omitted
on ordinary items:

```text
transfer: {
  sources: [{start, end, marker, route, block_id}],
  arrow: {start, end},
  destination: {start, end, kind: note|task|drawer|section,
                route, block_id?, drawer?, section?}
}
```

Add `transfer_source`, `transfer_arrow`, and `transfer_destination` spans; reuse
drawer/section sub-spans. Partial items retain usable spans and needs, and errors have
exact byte ranges and codes. Ordinary singleton/target-run payloads stay unchanged apart
from the intentional top-level grammar revision.

`bob capture -f json` transfer items expose `moves[]` or `merges[]`, plus
`repaired_files[]`, using existing engine/CLI wire shapes and stable ordering. Ordinary
results omit empty additive fields. Human output uses `↪ moved …` and `⤵ merged …`,
with dry-run wording and the canonical copy from the parent plan and
`docs/task-merge.md`. Merge warnings and offers must remain visible.

### Execution and transaction semantics

- Resolve all sources against the staged snapshot before mutation. Later draft items see
  earlier edits, and any parse, plan, or guarded commit failure leaves no partial
  successful capture. Use the existing rollback machinery.
- Bare-note moves call `batch_move` once per source in typed order. Preserve open,
  column-zero, non-`^prj` eligibility, blocked-project restrictions, existing same-note
  refusal (`move_same_note`), link/ID repair, Task Link relocation, and freshness
  stamping.
- Merge into a task's plain-child anchor, a resolved drawer (H2 on `^prj`), or a strict
  eligible H2 section. Use `MergeGesture::Explicit` and the first configured merge
  label, falling back to `merged into`. Retain quoted-selector intent; use the engine's
  strict section creation/near-duplicate behavior, never capture's loose section
  fallback.
- Let `plan_task_merge` own payload cuts, tombstones, dependency rewrites, managed logs,
  provenance, refusals, idempotency, and source-link pruning. Retain configured
  successor labels and relation policy in its context.
- Adapt every engine read dependency into capture's available guard mechanism, including
  relevant scanned path membership. Preserve merge write order: target, dependents,
  repairs, daily note, then sources. Same-file operations stay one staged file. Extend
  batch ordering/guards only as necessary; do not duplicate the mutation engine.
- Reuse draft-level `pull_in` numbering and `hand_off` offers with `merged_from`. Offer
  T once at the earliest lost source-link slot only when eligible; never auto-link.
  Selected `-p` rows link, raise to Next, and stamp fresh through existing pick
  semantics. Engine calls must not apply picks a second time or interpret item-local
  numbers as draft-global numbers.
- Dry-run and real planning return the same result aside from `dry_run`. An unchanged
  re-run has zero writes. Merge followed by reconcile is a no-op.

## Phase transfer-lexer

Add a single pure span-aware transfer scanner and result types under
`capture_language/transfer.rs`, with table tests in
`capture_language/tests/transfer.rs`. Model unclaimed, escaped-literal, incomplete,
valid, and invalid input distinctly. Both future consumers must use this scanner, not
parallel regexes or re-tokenization with lost offsets. Use existing token and selector
primitives where possible.

Keep this otherwise-unused module test-gated until transfer-resolvers consumes it; do
not add permanent dead-code suppression. This phase does not change public grammar
recognition, CaptureKind, or the grammar revision.

Independently add the shell guard at the capture entry point before planning. Only
stdout attached to a regular file whose basename starts with `@` triggers
`shell_ate_transfer_arrow`. Obtain the path with `/proc/self/fd/1` on Linux and
`fcntl(F_GETPATH)` on macOS; skip unsupported platforms. Do not close stdout or treat
pipes/TTYs as files. Use the existing error printer and canonical quote-the- draft
message. Help and pure expansion should retain their existing behavior.

Tests cover one/many sources, four destinations, partial selectors, exact error
codes/ranges, UTF-8 and CRLF offsets, quotes, escaping, modifier positions, duplicate
identity, code/prose/blockquotes, and guarded stdout. A child process redirected to `@c`
must refuse before vault mutation; normal files and pipes must still work.
Platform-specific path acquisition is cfg-gated, with actual Linux coverage and macOS
coverage runnable on macOS.

Done when focused lexer/guard tests and `just check` pass (or an independently
reproduced clean-base failure is recorded under the policy below). No grammar bump or
user-facing transfer execution yet.

## Phase transfer-resolvers

Promote the scanner into production and introduce
`CaptureKind::Transfer { sources, destination }` plus the four destination variants.
Wire it at the top of `parse_capture_item_members` and before
`try_parse_editor_target_run`, with matching integration in `line.rs`, item
classification, declaration inheritance, and list handling as required. Both resolvers
use the same claim and error data before masking modifiers.

Implement the parse contract above across editor models and `capture_parse.rs`. Update
exhaustive matches throughout the crate without implementing transfer
completion/rewrite, which belongs to bob-cli-go.11. Transfer execution receives one
temporary explicit coded refusal at early planner dispatch, preventing any fallthrough
to note insertion or Ensure Next. Test this refusal and no writes; transfer-moves/merges
replace it branch by branch.

This phase alone bumps `GRAMMAR_REVISION`, after re-checking the highest claimed
revision. Update current revision assertions, current-version prose, and example JSON
across tests/docs, while preserving statements about historical revisions. Read
`docs.md`, regenerate any affected references, and use a `feat(capture)!:` commit
declaration with a `BREAKING CHANGE:` footer describing that formerly-authored `> @dest`
text now claims a transfer directive.

Add execution/editor agreement tests for the full scanner matrix, forced flags, lists,
`@@` shadowing, multiple items, no fan-out, and no Ensure Next. Test singleton and
multi-item JSON, exact offsets after non-ASCII preceding items, and omission of transfer
fields on ordinary target runs. The worked parse example must already report
`task_merge` and the transfer object.

Done when these tests and `just check` pass, grammar has changed exactly once, and
complete but unimplemented execution fails explicitly with zero writes.

## Phase transfer-moves

Add `capture/transfer.rs` and early dispatch in `capture/plan.rs`. Build shared
source/destination resolution and preflight validation against staged snapshots,
including resolved duplicate aliases, overlapping sources, and target-is-source. Adapt
route resolution instead of assuming routes equal filesystem paths.

Implement bare-note transfers via `apply_batch_move` sequentially. Integrate existing
task relocation/freshness trackers and net item results so subsequent capture items see
the correct identities and paths. Add grouped/defaulted `moves` and `repaired_files`
output fields, sharing the standalone move JSON mapping. Do not alter existing
relation-merge output shape. Merge destinations retain their explicit temporary refusal
until the next phase.

Register `tests/cli/capture/transfer.rs`. Cover byte-for-byte same/multiple source note
moves, typed order, IDs and link repair, source eligibility and blocked destination
rules, same-note refusal, aliases/duplicates/overlap, dry-run parity, later-item access
to the moved task, later-item rollback, human output, and no Ensure Next side effects.
Include compatibility cases mixed with ordinary target runs and project-note moves.

Done when focused transfer/move and affected composition tests plus `just check` pass;
bare-note transfers work through the public CLI and remain one item.

## Phase transfer-merges

Replace all remaining execution refusals with merge-engine integration for task, drawer,
and section destinations. Resolve all sources and call the engine on the current staged
snapshot in typed order, retaining its complete multi-source validation. Share narrow
snapshot/context/offer-conversion helpers with `relations_merge.rs` where practical;
preserve Label versus Explicit behavior.

Stage postimages and register ordering and read guards from the actual merge plan. Feed
the existing flattened `merges` output and shared repaired files, and integrate
warnings/offer rows into `pull_in`. Review both planning passes used by pull-in
selection so `-p` applies exactly once and the preview's numbers remain stable across
multiple items. Handle unchanged results without spurious freshness writes or commits.

Extend transfer CLI tests with task, existing/new drawer, quoted selector, `^prj` H2
drawer, existing/new area/project section, and strict section refusals. Assert actual
block contents, cancelled source histories, moved child anchors, dependent repointing,
preserved target properties, source-link pruning, and configured/default labels. Test
unchanged reruns, explicit closed-source refusal, the engine's key semantic refusals,
one deduplicated offer, unpicked and picked results, multiple-item offer numbering, and
reconcile convergence. Prove dry-run parity, staged merge then later-item editing,
rollback on later failure, guarded stale input refusal, and destination-before-source
ordering with existing transaction fixtures. Keep existing relation-label tests green.

Done when all destination forms work, no temporary execution refusal remains, and
focused tests plus `just check` pass.

## Phase transfer-delivery

Read `docs.md` before edits. Publish the new “Moving and merging tasks with `>`” section
under capture's existing-task sub-bullets, plus parse and result details. Update
`docs/reference/capture-syntax.md` token index and scratch example,
`docs/_meta/search_smoke.yml`, `docs/reference/json-outputs.md`, and
`docs/reference/compatibility.md`. Revise `capture/cli.rs` help and the task move
after-help with quoted capture examples; then run `just docs-gen`. Remove only obsolete
wording about this phase's transfer availability in relevant contracts. Leave unfinished
Mac/completion promises scoped to their owning phases.

Audit the parent's transfer-capture acceptance list against the preceding tests; fill
only genuine missing integration cases. In an isolated scratch vault run all four
examples above and:

```sh
bob capture-parse -f json -- '@inbox+bench @work+pgdump > @db+upgrade#future-work'
```

Check expected blocks, modes, spans, results and repairs, no Ensure Next, dry-run
parity, selected/unselected offers, rollback, and post-merge reconcile. Use the
just-built binary with isolated vault/home/config, never Bryan's live vault or an
installed binary with an older grammar. Keep the shell redirect smoke test inside that
scratch environment.

Run `cargo fmt --all`, `just check`, `just docs-gen`, `just docs-check`, and
`git diff --check`; after generation, rerun only checks affected by changes.
`just check` is the repository's fmt, clippy, and full workspace test gate. Use
`/sase_monitor` for long commands and wait for the handoff command itself to finish.
Record actual results and any unavailable macOS-only checks.

## Land-agent closeout and known baseline handling

Read every child phase note, inspect the combined changes, and verify the whole scope
above. Confirm the single grammar increment, no memory/version/changelog edits, no
transfer fallback or temporary refusal, and preserved ordinary capture contracts. Check
`sase bead epic-symbols` for each closing phase and for `bob-cli-go.10`; resolve entries
or re-key them to a still-open owner. Never leave stale Justfile symbols behind.

The closed gm epic records a docs-generator failure caused by missing glossary
frontmatter, already owned by bob-cli-go's memory closeout. Reproduce on the current
clean base if it still blocks documentation gates; do not fix it by editing memory in
this child epic. Any check failure reproduced identically on the clean base is recorded
with exact command/evidence and an existing task ID when known, and does not keep a
finished phase open. Newly discovered unrelated work goes only to a
`PROPOSED FOLLOW-UP:` note on the worker's assigned phase, for the parent land agent to
triage; create no task beads here.

The child epic land agent closes its own completed child epic and then only the
delegated original phase with:

```sh
sase bead epic-symbols bob-cli-go.10
sase bead close bob-cli-go.10 --note '<implemented scope and verified results>'
```

Closing bob-cli-go, another ancestor, or a sibling remains outside this work's
authorization. Finish each root turn through `/sase_final`.
