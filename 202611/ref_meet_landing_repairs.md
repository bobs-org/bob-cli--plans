---
tier: tale
title: Finish and land the meeting notes reference command
goal: Repair the remaining ref meet contract gaps, verify the complete command, and
  close bob-cli-pw.5.1 plus parent phase bob-cli-pw.5.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-pw.5.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-pw.5.1
status: done
---

- **PARENT:**
  [202611/ref_meet_cli.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/ref_meet_cli.md)
- **BEAD:**
  [bob-cli-pw.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pw/bob-cli-pw.5.1.md)

# Finish and land the meeting notes reference command

Repair the remaining work in epic **bob-cli-pw.5.1**, then complete its landing and
close its parent phase **bob-cli-pw.5** in this same implementation turn. This is one
bounded coding-agent task, not another phase graph. There is no later land agent for
this tale. Do not wait for this tale's commit SHA, push, or CI before closing the epic:
normal same-turn verification and closeout precede the host commit.

## Context and boundaries

Read `sase bead read bob-cli-pw.5.1 -r "Need landing audit and triage outcomes"`, its
four closed children `.1` through `.4`, and the audited artifact
`plan:202611/ref_meet_cli.md`. The epic's plan file is `202611/ref_meet_cli.md` in the
plans repository, as resolved by the bead's PLAN link. Use `/sase_repo` before editing
that repository and `/sase_memory_read` for applicable CLI/docs memory. Do not hard-code
any numbered workspace path.

The land audit was on bob-cli `e4c96b05` and fetched origin/master at that same commit.
The child implementations are `15fd03b2` (preparation), `bf309445` (first filing),
`c5fa6da6` (replacement), and `e4c96b05` (CLI/docs). Preserve their working shared
resolver, normalizer, PDF profile, marker birth, recap, source-store, intake, and
overwrite services. The repairs below complete their existing approved contracts.

Fixed decisions remain `recap_scope = full`, `scaffolding = rehang`, and both memory
decisions `no`. **Do not edit memory, versions, or CHANGELOG.md.** Synthetic meetings,
people, notes, and temporary vault-sync pairs only. Do not read the real first-use notes
or change the real vault. First use, Obsidian, and Mac work belong to outer phases
pw.6/pw.7/pw.8. Do not close the outer epic bob-cli-pw.

The audit found no non-epic commits after the first child commit. It also reviewed
`69e5fcb9` (Gmail post-filing observations) and `447ef1a5` (props settings report),
which landed after epic creation but before that first child commit. Gmail needs no
meeting integration. The props addition shares the JSON documentation page; retain that
contract while repairing the shared page budget. Recheck new commits at start and before
close for relevant drift, rather than assuming this audit remains current.

## 1. Complete public argument, input, editor, and selection handling

Work in `src/native/highlights_ref/meet_cli.rs` and the existing supporting modules.

- `command()` leaves `parent`, `producer`, `status`, and `title` as clap `String`, but
  `run_file()` reads `OsString`. An offline harness using the unchanged production
  builder and clap 4.6.6 reproduced a downcast panic for all four. Make parser and
  accessor types agree and test each public long and short flag through the real
  command. Audit the other value flags too. Reuse the existing directory arguments; the
  new public `--lib-dir` needs an available short alias (use `-L`, since `-l` is list)
  if retained. Keep the approved flags and sorted help.
- The `ref-meet` catalog writes row in `catalog/table.rs` currently supplies no notes
  source, so its ordinary preview/action can only list. Supply a concrete source
  consistently, using clipboard `-c` for this row, and verify both assembled argv paths.
  Keep `ref-meet-list` read-only with fixed `-l` and preserve existing slot, adapter,
  policy, and invalidation contracts.
- Clipboard intake currently calls `read_clipboard()`, which normalizes CRLF and strips
  trailing blank lines before `read_notes()` hashes the string. Add a narrow
  raw-byte/snapshot seam in the existing clipboard helper and pass the single snapshot
  through the shared bounded notes reader. Do not change ordinary capture clipboard
  normalization. File/stdin/clipboard receipts and `--expect-sha256` must use the exact
  source bytes before normalization. Empty clipboard notes must be `no_notes` rather
  than a generic read failure; retain operational clipboard failures as unreadable.
- Editor subprocess stdout currently inherits JSON stdout. Keep the editor usable while
  keeping stdout exactly one JSON envelope; direct editor diagnostics to stderr or the
  controlling terminal using existing editor conventions. Reuse the bounded
  UTF-8/binary/empty notes validation on accepted editor content. Keep scissors and
  trusted instructions out of accepted notes, retain the original source digest after
  edits, set edited provenance correctly, and reject a hash guard that cannot actually
  be checked rather than silently bypassing it. Dry-run must never launch the editor or
  create scratch files. Do not consume notes stdin as interactive picker input; off-TTY
  and JSON choices return structured errors promptly.
- `filter_list_by_query()` currently searches substrings in a concatenated title/root/
  path/ref. Resolve explicit supported tokens through the existing event-token parser
  and resolver (wikilinks, note:id, leading `~`, Google eid URLs, and paths), while
  retaining title-query filtering and day/window restrictions. Honor the documented
  list-versus-filing modes and conflicts without reading notes sources in list mode.

Keep changes local to these seams. No new capture grammar, parser family, network
service, or independent vault writer.

## 2. Preserve overwrite intent and report actual transaction results

Preparation in `meet/prepare.rs` currently derives a fresh title even when overwriting
an existing custom-titled reference without `-T`. Carry explicit-title intent and
default to the selected reference's live title. Preserve the live task's residence,
block ID, status, and exact LINKS bytes unless the approved explicit override applies.
Do not let an unsupported parent move become a misleading success or proposed re-root.
Keep recap ownership, annotated history separation, coordination, and recovery inside
the existing overwrite transaction.

`FileSuccessPayload` currently flattens `MeetingPrepareReport` (which already contains
`outcome` and `warnings`) alongside duplicate fields of the same names. Build one
unambiguous envelope, with one occurrence of every key. Use `IntakeOutcome::Ingested`
and `OverwriteResult` to replace planned note/PDF/task fields with the actual committed
facts. This matters when post-sync task allocation or existing status differs from
preparation. Preserve vault-relative path and task-locator conventions.

Do not discard `commit_warning`, intake progress/ignored-path warnings, sync status, or
overwrite `push_pending`. Surface them in human receipts and JSON warnings or the
established result fields without claiming a successful commit/push when it failed. Keep
incomplete writes and refused overwrites failures. Preserve typed overwrite codes and
hints through `OverwriteRefusal::command_error()` instead of labeling every code-less
failure `render_failed`.

Queued receipts describe planned paths and the actual queued PDF, say nothing was filed,
and report skipped re-root; they must not imply that a planned root mutation landed.
Success should include the existing mirror/reconcile and re-root/reset guidance required
by the approved contract. Preserve schema version 1.

## 3. Finish the locked occurrence and event-edit guard

In `intake.rs`, `verify_event_root_edit()` currently contains the only locked meeting
identity/dedupe recheck, and it runs only when `event_root_edit` exists. An
ordinary-root filing therefore does not recheck the occurrence after pre-sync. Decouple
the meeting occurrence guard from optional root mutation: every fresh meeting ingest
must recheck Google event ID or resolved manual event identity and occupied destinations
under the existing locks after pre-sync, before publishing note/task/root mutations.
Reuse the existing filed index and marker identity, not title identity. Test a competing
filing with a different live stem so simple filename-collision guards cannot hide the
bug.

The new daily-note preimage is currently restored only if `apply_event_root_edit()`
itself errors. A later incomplete return, including persisted source-pointer
verification failure, can leave the re-root applied. Keep the root edit within the
successful intake boundary: restore the checked preimage on incomplete failure before
commit, retain honest commit-warning behavior, and avoid overwriting any independently
changed bytes. Use the existing transaction/rollback mechanisms or a narrow checked
preimage guard, never a post-success write. Preserve `set_row_metadata` badge/buffer
bytes and the single scoped commit. Queued intake and subsequent drain must not replay
an abandoned root-edit intent. Keep other producers' intake behavior unchanged.

## 4. Add the missing acceptance coverage and integrate docs

The current public `tests/cli/highlights/meet_cli.rs` has 11 mostly read-only tests;
there is no CLI success/overwrite/queue/editor/input/picker matrix promised by the
approved plan. Add focused end-to-end cases using the existing synthetic fixtures and
renderer/sync stubs, reusing service tests where they already prove internals:

- Every repaired value flag, catalog preview/action source, explicit list identities,
  query/auto/picked evidence, noninteractive choice errors, and parent selection.
- Single-read stdin/file/clipboard inputs; exact BOM/CRLF/trailing-newline digests and
  mismatch refusal before editor/render; editor success/change/cancel/failure/empty and
  invalid or oversized output; a noisy editor in JSON mode; dry-run no-write behavior.
- Successful filing and inbox re-root with one commit/path set; duplicate refusal;
  ordinary-root competing occurrence after pre-sync; owned and edited recap overwrite;
  preserved custom title/status/task/LINKS identity; explicit overrides; transcript
  exclusion; queued receipt/drain without re-root; incomplete-root rollback; final
  receipt paths and commit/push diagnostics. Assert JSON has no duplicate keys, not
  merely that a permissive parser accepts it.

Existing service tests in `create/meet_route.rs`, `meet/tests.rs`, `meeting_birth`, and
`overwrite/execute.rs` remain useful controls. Do not claim a real renderer passed based
only on stubs. Outer epic bob-cli-pw already has a note about a possible meeting-profile
LaTeX preamble failure from pw.3; read its current disposition and verify the real
profile tests. If still reproducible, coordinate that recorded owner and assess the
required command acceptance honestly; do not create a duplicate task or silently waive a
failing required check.

Update existing help/contract/guide/JSON documentation only where these fixes change or
clarify behavior. Keep the shared props update settings-contract description. Use
canonical `bob_docs.markdown.count_words`: the audit measured configuration.md 5997
before this epic and 6098 now; json-outputs.md was 6000 before props `447ef1a5`, 6005
after it, and 6030 after this epic. Trim redundant prose to bring both under 6000
without losing either contract; do not just raise the cap. Run `just docs-gen` and
`just docs-check`, with mounted repositories opened through `/sase_repo` as needed.

Run focused suites, then **`just check`**, never `just check-full`. Use `/sase_monitor`
for long verification and wait for its staging command to exit before finalizing. Any
verification continuation must retain the remaining closeout below; do not use a
completion flow that ends after tests without closing the epic. A completed `just check`
pass paired with a `just check-full` failure would be test infrastructure, not residual
epic work. Record a synthetic dry-run timing measurement against the approved
under-300-ms target without adding a brittle timing assertion.

## 5. Complete this epic's landing and its parent phase

The lander has already finished all three child follow-up proposals. Carry these
outcomes into the epic close note; do not delegate or repeat task creation:

- pw.5.1.3 note #1: `useless_format` introduced by bf309445 was fixed in e4c96b05;
  separate task declined as already addressed epic work.
- pw.5.1.4 note #1: configuration budget belongs to this epic; JSON page's initial
  five-word overage was recorded as `DISCOVERED ISSUE` on active bob-cli-pl.6.1 and is
  integrated by this tale. Mounted navigation hotkeys independently measured 6029 at
  bob-plugins a22851fd; existing **bob-cli-jz** received corroboration (+18). No new
  task or plugin edit belongs here solely for that known unrelated budget node.
- pw.5.1.4 note #2: nine-minute test timeout had no established failing assertion; new
  task declined, final `just check` verification retained using the monitor.

Re-read current epic/child notes, check relevant post-audit commits, and confirm the
approved command contract and these repairs are complete. If a genuinely new unrelated
follow-up appears, use `/sase_new_task`; epic-caused defects remain here. Record actual
verification results and any documented baseline issue explicitly.

Then finish these actions, without waiting for this work's own future commit or CI:

1. Run `sase bead epic-symbols bob-cli-pw.5.1`. Resolve every reported exemption by
   wiring, privatizing, adding a justified non-test pragma, or deleting; re-key only to
   a still-open later bead that actually needs it. Also inspect
   `sase bead epic-symbols bob-cli-pw.5` for the parent phase. Both the child audit and
   the current Justfile had no entries/`symvision` recipe; recheck rather than assume.
2. Close **bob-cli-pw.5.1** normally with
   `sase bead close bob-cli-pw.5.1 --note "<scope/child-note/decision verification, repaired defects, integration, actual checks, and all follow-up dispositions>"`.
   Resolve any stale symbol or incomplete descendant named by rejection and retry. Never
   use `--force` merely to succeed or to advance a successful nested landing.
3. Run `just symvision` if available and confirm it passes; if the recipe is absent,
   record that fact. Open the plans repo through `/sase_repo` and set `status: done` in
   the frontmatter of **202611/ref_meet_cli.md**, using the PLAN path resolved from this
   epic. Do not mark the outer plan done.
4. Run `sase bead read bob-cli-pw.5.1 -r "Need the parent link"` and re-read parent
   **bob-cli-pw.5**. It is a phase of outer epic **bob-cli-pw**. Verify this child plan
   completed that phase's entire CLI/intake/overwrite/docs scope, retire any remaining
   parent-phase symbol entries, and close only that phase normally:
   `sase bead close bob-cli-pw.5 --note "<verified complete ref-meet command and checks>"`.
   Leave bob-cli-pw, its outer plan, and pw.6/pw.7/pw.8 to their waiting agents. If
   parent readiness has become incomplete or ambiguous, note the exact blocker on
   bob-cli-pw.5 and report it rather than forcing closure.

Finish through `/sase_final` with all this turn's repositories declared. The final
response must say whether both the child epic and parent phase closed and identify any
genuine remaining blocker.
