---
tier: tale
title: Cancel-label config and pure Rust cancellation engine
goal:
  Complete bob-cli-dl.2 with conservative policy parsing and a vector-tested cancel
  planner that prunes current Pomodoro links and recovers dependents without offers.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-dl.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-dl.2
create_time: 2026-10-28 09:17:11
status: wip
---

- **PARENT:**
  [202610/relation_cancel_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_cancel_labels.md)
- **BEAD:**
  [bob-cli-dl.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dl/bob-cli-dl.2.md)

# Implement the cancel-label policy and pure Rust cancellation engine

Complete assigned phase **bob-cli-dl.2** (`cancel-engine`) of
`plan:202610/relation_cancel_labels.md`. This is one bounded implementation:
configuration, pure cancellation planning, recovery reuse, and their tests. The
downstream capture and surface phases will call this API. Do not wire cancellation into
capture or reconcile in this phase.

## Authority and starting point

- Read `sase bead read bob-cli-dl.2 -r "Need the current phase scope and notes"` before
  implementation. Keep its runtime-owned status; do not claim it again.
- At planning time the checkout is clean at `52bff016`, the landed `cancel-contract`
  change. `bob-cli-dl.1` is closed. There are no earlier work or remainder notes on
  `.2`. Recheck the tree and notes before implementing to avoid repeating work that
  landed during approval.
- `src/native/task_cancel/mod.rs` exists but contains only a fixture smoke test.
  `src/native/config/relations.rs` still parses only `linked_from_lines`.
- Normative behavior is `docs/task-relations.md` §17, backed by
  `tests/fixtures/task_cancel/vectors.json` and its provenance README. The accepted
  research is
  `research:202610/cancel_labels_for_task_relations/cancel_labels_for_task_relations.md`;
  use audited `sase artifact read` for that report and the parent plan. If research
  initially resolves missing, open `bob-cli--research` through `/sase_repo` and retry
  the audited read; that recovered the document during planning.
- Final epic decisions remain: `cancel_stage_door=no`, `memory_decision_record=no`,
  `memory_glossary_relation=yes`, and `memory_glossary_term=no`. This phase edits no
  memory. The authorized `glossary:task-relation` correction belongs to rollout; do not
  create links to the rejected decision record or glossary term.
- No live vault/config writes, release version edits, or changelog edits. Tests use
  scratch vaults and explicit `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, and
  `BOB_CONFIG_FILE`. No linked-repo changes are needed.

## 1. Parse an independent, conservative cancel policy

Extend `src/native/config/relations.rs` with ordered, deduplicated cancel-label records
retaining a canonical key and a normalized display name, plus `warnings: Vec<String>` on
`RelationsConfig`. Preserve first occurrence order and spelling. Use
`task_relations::label::normalize_relation_label` for writer legality and
`relation_labels::relation_label_key` for identity; do not invent a second normalizer.
Export crate-visible types/helpers as required through `config/mod.rs`.

- Missing file/block/key, `null`, and `[]` produce an empty policy. A non-list value
  gives the exact §17 warning and an empty policy. Invalid elements each warn and are
  skipped; valid peers survive. Cancel-policy validation never returns a `ConfigError`
  and cannot change `linked_from_lines`.
- Provide `is_cancel_label(key)` and `cancel_label_for_inverse(key) -> Option<&str>`.
  The latter compares against `inverse_phrase` of configured labels and returns their
  stored display spelling. Inverses themselves never become cancel labels unless
  explicitly configured. Hidden/one-off state is not an input to policy.
- The config vector with invalid `linked_from_lines` requires retaining valid cancel
  labels, defaulting the mirror flag to true, and one warning. Parse the keys
  independently. Keep YAML syntax/read failures and invalid whole-section handling
  conservative. Update the existing tests that currently expect an invalid mirror flag
  to abort the combined parse.
- Preserve existing command warning behavior when changing that error path:
  `capture/relations_mirror.rs::load_linked_from_lines_enabled` and
  `task_status_hooks/sync.rs::load_relations_linked_from_enabled` currently turn a
  config error into the mirror-default warning. Ensure they still surface that warning
  exactly once after the parser starts returning it. Do not add cancel effects or
  surface/catalog JSON; the dependent phases own their command-wide cancel-warning
  presentation.
- Correct one fixture typo: in the `duplicate-keys-collapse` YAML string, quote
  `"Duplicate  Of:"`. The unquoted `- Duplicate  Of:` is a YAML mapping and contradicts
  its intended zero-warning duplicate-string case. Keep its expected keys/count
  unchanged, add a separate mapping-entry rejection test, and record this syntax repair
  in the phase note so the plugin phase copies the corrected source fixture. Do not
  relax the non-string rule. Do not alter generated `cancel` or `prune` expected bytes.

## 2. Implement pure verdict and source-note edits

Replace the smoke-only module with small `pub(crate)` modules under
`src/native/task_cancel/` for verdicts, line/log edits, prune, recovery, and tests. All
production functions consume snapshots and explicit date/settings; they perform no
filesystem writes, config loads, clock reads, notices, or reconcile calls.

- Expose the source verdict required by the parent plan:
  `cancel_verdict(source_line, tasks_settings, is_new_task)` returning
  `Cancels { previous_status, project }`, `AlreadyClosed { status }`, `Recurring`, or
  `NewTask`. Use the shared `is_recurring_task_line` and exactly the Task Card open set
  (` `, `*`, `/`, `?`); arbitrary custom open statuses do not qualify.
- Add a thin pure relation-policy verdict wrapper for the shared `rule` cases: only an
  explicit Added/Relabeled edge with a configured key can reach the source verdict;
  unchanged, unlink, relabel-away, reconcile, rename, hand/agent edits, mirrors, sync,
  and config changes return no effect. Represent item-action conflicts explicitly so all
  34 rule cases test production logic. Callers supply the already-resolved source; the
  engine never swaps S/T based on direction.
- Rewrite only an eligible source task's checkbox and upsert `[cancelled:: date]`,
  preserving unrelated fields, their spacing, block IDs, indentation, line endings, and
  the final newline. Closed sources preserve both status and old dates.
  Recurring/new/conflicting sources return a refusal before any planned effect.
- Reuse existing Cancel Log recognition (`task_dependencies::blocks::is_cancel_log` or
  the reconcile field helper). When a direct child Cancel Log exists, prepend the dated
  fallback reason under it; otherwise create no log. The fallback uses the exact LINKS
  label/link spelling, with multiple triggering edges joined in authored order by `·`.
  Cancel once and log once per source. A second call on the already-cancelled source
  must leave its bytes unchanged.
- Test all 15 `cancel` fixtures against the real writer, including the reported
  cancel/skip/log/refusal outcomes, rather than just parsing the fixture file.

## 3. Implement the immediate Pomodoro prune

Use `task_status_hooks::scan_pomodoros`, section/fence parsing, and
`vault_links::NoteIndex` for structural boundaries and identity resolution. Use the
existing retirement module as an API example, but do not invoke its completed task
retirement or reconcile's cancelled-subtree cleanup: those have different semantics.

- Operate only on the supplied current daily snapshot, and only children of open entries
  inside `## Pomodoros`.
- A dedicated matching link bullet disappears with its subtree. Mixed prose loses only
  matching live tokens and normalizes the resulting gap exactly as the generated
  fixtures prescribe. Preserve unrelated tokens/prose and line endings.
- Keep struck links, closed Pomodoros, top-level entry lines, fenced examples, links
  outside the section, and other-day files. Resolve path plus block ID; ambiguous or
  unresolved links must not cause deletion. Preserve empty entries according to the
  immediate planner's fixtures; do not add reconcile cleanup.
- Return the postimage and removed-link/bullet/entry and unresolved counts needed to
  assert all 9 `prune` fixtures. Count overlapping subtree removals once.
- Represent an absent daily snapshot distinctly from a failed read. Absence yields no
  daily postimage and zero links removed; a supplied read error propagates with no plan.
  A pure input enum or `Result<Option<...>, ...>` lets tests cover this without adding
  filesystem access to the engine. Capture will supply these states when its wiring
  lands.

## 4. Reuse recovery without offers or completion-only side effects

Implement `task_cancel/recovery.rs` on the production successor/project recovery logic.
`task_complete/recovery.rs` is test-only and remains a parity oracle, not new duplicated
production policy. Keep `task_dependency_states`, `blocking_open_dependency_ids`, and
`project_close`'s remaining-reason, inherited blocking, scope, tag, and unblocked-field
helpers authoritative.

Introduce a narrow cancellation/recovery-only entry point or internal mode in
`task_complete/successors.rs`, leaving the existing `plan_successors` entry point and
all its default completion behavior unchanged. `SuccessorSelection::None` alone is
insufficient: it still creates offer metadata. Even `link_unblocked=false` currently
permits project-finished pull-in detection, so explicitly suppress completion-only
pull-ins as well as offer construction, selection, key generation, minting, linking,
freshness stamps, and hide removal.

- Recover only affected Blocked dependents whose other open prerequisites, future
  schedule, and project blocking reasons are all gone in the post-cancel snapshot.
  Unrelated tasks and sticky Next/Pending lanes stay unchanged.
- Apply the existing `[unblocked:: today]` WRITE rules and skips, and project-tag
  cleanup through shared helpers. Never transfer prerequisites to T.
- When a dependent `^prj` recovers, recover eligible same-project residents through the
  existing project recovery logic. Also explicitly seed local resident recovery when S
  itself is a cancelled `^prj`: the current successor planner's gate requires
  predecessor `[id::]` or a pull-in and otherwise misses this case. It must work even
  when the cancelled `^prj` lacks `[id::]`. Residents remain open, with their own
  outstanding dependency/date reasons still honored.
- Keep inherited/subproject recovery boundaries from
  `decisions:subprojects-inherit-project-blocking`; do not expand a close gesture into
  descendant-tree reconciliation.
- Test direct recovery parity with completion's unselected recovery, while asserting
  zero offers, successor links, minted IDs, and completion-only pull-in writes. Include
  an additional open prerequisite, a future date, another project block, inherited
  blocking, a recovered dependent project, and a cancelled source project with both
  eligible and still-blocked residents.

## 5. Compose one usable engine entry point

Expose `plan_task_cancel(input) -> Result<TaskCancelPlan, ...>` with the parent plan's
result information: source postimage, optional daily postimage, cancel-log outcome,
prior status, removed Pomodoro-link count, recovered task identities, and recovery-file
postimages. Inputs include source identity/line, already-staged source text (including
LINKS), ordered reasons, explicit date, note index, settings, daily read state, and
dependency snapshots.

Compose source edit, prune, and recovery over the resulting snapshots in that order.
When paths overlap (source is daily, or recovery edits source/daily), fold the edits
into one coherent postimage; do not return competing versions that callers can
overwrite. Test this overlap explicitly. AlreadyClosed/no-effect cases must not prune or
recover anything, and refusals return no partial plan.

Pass the configured day into every dated write. Test New York midnight by deriving the
engine date using the existing `native::env` clock/config facilities with `BOB_NOW`
before/after local midnight; use the repository's scoped env helpers. The pure engine
itself must not discover today or create a daily file.

## 6. Documentation, verification, and completion

Read `docs.md` via `/sase_memory_read` before editing docs. Remove only
`planned (phase cancel-engine)` markers for implemented config/engine rules in
`docs/task-relations.md` §17, preserving remaining client-phase markers. Update
`docs/reference/configuration.md` only where the actual warning behavior needs
clarification. No CLI command/help/JSON changes are in scope.

Verification must include:

- Typed fixture-driven assertions for all five sections: 15 cancel, 9 prune, 34 rule, 10
  key, and 10 config cases (plus the mapping-entry regression).
- A scratch-vault CLI pin in `tests/cli/task_status_hooks/relations.rs`: with cancel
  labels configured, an already-normalized open task carrying a `duplicate of` LINKS
  entry stays open through reconcile. Supply canonical mirrors or disable them, so
  unrelated reconciliation does not invalidate the byte-stability assertion; repeat
  reconciliation to pin idempotence.
- Existing relation/config, task-completion/successor/project-recovery, and capture
  relation suites unchanged, with no cancel config, remain green.
- Targeted tests first, then `just check` and `just docs-check`. For long checks use
  `/sase_monitor` as required by the runner and carry the completion steps into its
  continuation. Do not finish while an ordinary exec session is still running.
- Reproduce failures on the clean base before classifying them as pre-existing. Record
  identical base failures as `PROPOSED FOLLOW-UP:` on `.2`, citing an existing tracking
  bead when known; those failures do not prevent phase closure.

Keep production API lint coverage while these entry points await downstream callers. If
temporary unused-symbol tracking is necessary, use the project's epic-symbol mechanism
with the still-open parent `bob-cli-dl` or appropriate consumer phase
(`bob-cli-dl.3`/`.4`), not an expiring `.2` waiver or blanket lint suppression.

Record verified APIs, fixture correction, checks, and any implementation caveats on
`bob-cli-dl.2`. Create no beads; discoveries go only in
`sase bead note bob-cli-dl.2 'PROPOSED FOLLOW-UP: ...'`. Before closing run
`sase bead epic-symbols bob-cli-dl.2` and resolve every remaining entry or re-key it to
a still-open owner. Then close **only**
`sase bead close bob-cli-dl.2 --note "<concrete verification evidence>"`. Do not close
`bob-cli-dl` or any ancestor plan bead. Submit the normal root `/sase_final` declaration
at implementation completion.
