---
tier: epic
title: Implement note-pull capture in bob-cli
goal:
  Complete bob-cli-gj.2 with atomic explicit note-pull selection, grammar 7, shared
  pull-in reporting, NP1–NP28 coverage, and real-bob fixtures for the Mac phase.
phases:
  - id: core
    title: Note-pull grammar and single-item planner
    size: medium
    depends_on: []
    description:
      "core: implement sections 3 and 4, including the lone-header grammar and editor
      mode, staged note-pull planner, classification, placement, explicit picks,
      refusals, additive result model, and shared offer traversal. Cover the single-item
      NP vectors and parser regressions. Read section 2 constraints."
  - id: batch
    title: Note-pull batch identity and transaction semantics
    size: medium
    depends_on:
      - core
    description:
      "batch: implement section 5, including stable pre-pick keys, earliest-item
      ownership, draft numbering, same-draft creation provenance, per-row net filtering,
      inbox routing, and transaction rollback. Cover NP13–NP20 and adversarial mixed
      drafts without changing existing successor-group behavior."
  - id: presentation
    title: Note-pull copy, human output, and block reporting
    size: medium
    depends_on:
      - batch
    description:
      "presentation: implement section 6, including note-pull groups and copy, human
      success and refusal output, task and Pomodoro block reasons, and CLI help
      regeneration. Pin approved wording, lane names, mixed-group behavior, and additive
      JSON."
  - id: verification
    title: Note-pull conformance fixtures and performance evidence
    size: medium
    depends_on:
      - presentation
    description:
      "verification: complete section 7, audit NP1–NP28, add CLI fixture assertions,
      generate real-bob JSON fixtures for the Mac phase, verify dry-run parity and
      write-free refusals, measure release latency on an isolated vault copy, and run
      just check plus just docs-check. Prepare closure evidence for bob-cli-gj.2; this
      phase closes only its own bead."
proposed_by: bbugyi200.athena.bob-cli-gj.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-gj.2
create_time: 2026-11-03 12:31:12
status: wip
---

- **PROMPT:**
  [prompts/202611/note_pull_cli.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/note_pull_cli.md)
- **PARENT:**
  [202611/note_pull.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/note_pull.md)

# Implement note-pull capture in bob-cli

## 1. Scope, current state, and decomposition

Implement the CLI phase of the approved note-pull design. A lone `@sase` offers eligible
tasks residing in `sase.md`, previews their destinations, and writes only explicitly
picked rows. It uses the existing `-p/--pull-in` protocol and atomic capture batch. The
Mac implementation remains in `bob-cli-gj.3`.

The contract phase `bob-cli-gj.1` landed as `fa077d2f8eb8783f2c6d9a65147ba9ef6844bcd8`.
It added the behavior contract and NP1–NP28 vectors, and already updated `capture-parse`
help. Do not repeat it. At planning time `GRAMMAR_REVISION` is still 6, the executor's
zero-bullet branch still refuses `empty_list`, and no note-pull implementation or tests
exist. `bob-cli-gj.2` is in progress, has no previous implementation/remainder notes,
and has no `--epic-symbol` entries. Recheck history and current code before editing.

This child epic separates four bounded coding tasks. The core planner owns one item's
reads and writes; the batch phase owns identity and later-item effects; presentation
owns observable copy; verification owns executable interchange fixtures and performance
evidence. Dependencies are sequential because the phases share `plan.rs`, `output.rs`,
and offer types. Each phase must compile and pass its focused checks; there must be no
temporary panic, stub, broad lint suppression, or permanently unused implementation
awaiting a later phase. The complete product contract is accepted only after all four
phases finish.

Authoritative context, read through the audited commands:

- `sase bead read bob-cli-gj.2 -r "Need CLI phase scope and decisions"`.
- `sase artifact read plan:202611/note_pull.md "Need approved note-pull design"`.
- Local `docs/task-dependencies.md` §§12.20–12.21, `docs/capture.md`'s "Pulling ready
  tasks from a note", and `docs/reference/json-outputs.md`.
- Use `/sase_memory_read` for relevant lane, freshness, ledger, successor-offer,
  documentation, and release rules. This plan changes no memory.

## 2. Settled constraints

The parent DECISIONS are final: `lanes = open_lanes`, `empty_pull = refuse`,
`memory_decision_record = no`, `memory_successor_offer = no`. Do not re-ask them. No
memory note, descriptor, strand, or generated `AGENTS.md` may be edited.

- Offer Pending `[/]`, Next `[*]`, and Ready `[ ]` in that order, then source line
  order. Ready includes NEW and ROTTEN tasks; this is not the READY bucket.
- Read only residents of the named root-level `<route>.md`, including nested subtasks,
  reference tasks, and eligible recurring occurrences. Do not traverse sub-projects. Do
  not offer or count the source's own `^prj` lifecycle task.
- No automatic selection, interactive CLI prompt, new subcommand, flag, or config
  switch. A typed pull ignores `plan.link_unblocked` and `plan.keep_moving` switches.
- Reuse the existing explicit-pick writers, status conventions, live-link parser,
  deterministic block-ID minting, and freshness writer. Never call `plan_task_link` for
  picks: it retires schedules. Do not mutate `[id::]` or `[dependsOn::]`.
- No capture command may target the live vault. Tests and timings use isolated fixture
  vaults or the approved temporary vault copy, with explicit vault/day/time settings.
  Use `/sase_repo` before inspecting any other repository, including a vault repository;
  use its returned path. Do not edit the Mac or plugins here.
- Keep capture JSON unversioned/schema 1. Change `GRAMMAR_REVISION` to 7. The
  grammar-changing commit must use `feat(capture)!:` and a `BREAKING CHANGE:` footer
  explaining lone `@route` and editor mode changes. Leave the Cargo version and
  `CHANGELOG.md` to the release train.

## 3. Core phase: grammar and result plumbing

In `src/native/capture_language/model.rs`, add `CaptureKind::NotePull { route }`. In
`item.rs::parse_list_bullet_item`, change only the `list.count == 0` branch for a task
header without a section. Existing header validation must continue rejecting trailing
markers and malformed headers before they can become pulls. Preserve lowercase route
resolution and `[A-Za-z0-9_-]`; aliases still do not resolve.

In `editor_parse.rs`, report mode `note_pull`, `route`, no needs or diagnostics, and the
existing route spans and task-list metadata (`count: 0`, `header_line`, `header_range`).
Keep Shift-Return's list metadata usable by the later Mac phase. Set grammar 7 in
`capture_language/mod.rs`; the already-landed help is the target.

Pin NP24–NP27 and adjacent grammar cases: a header plus bullets stays a list;
`@route#section` alone still needs `list_item` and refuses `empty_list`; a header
immediately followed by another header is judged as its own item; `@@route`,
existing-task link/note/toggle/project forms, and forced destination literals retain
their previous behavior. Cover whitespace, route case, and byte spans.

Add `src/native/capture/note_pull.rs`, register it in `capture/mod.rs`, and dispatch the
new kind from `plan_capture_item` before ordinary task formatting or target creation.
Cover exhaustive `CaptureKind` matches in `batch.rs`, `plan.rs`, and other callers found
by the compiler. `capture_kind_label` returns `note_pull`.

Add the optional `CaptureItemResult.note_pull` payload with `note`, `route`,
`successors: Vec<TaskCompleteUnblockedJson>`, and folded counts. Reuse
`PullInFoldedJson`'s seven keys: `already_planned`, `blocked`, `scheduled`, `hidden`,
`sub_projects`, `already_started`, `waits_for`; the last three are zero here. Add
`OfferGroup::NotePull` and `NOTE_PULL_TRIGGER`. Rows have no predecessor:
`trigger: "note_pull"`, `unblocked_by: []`, `offer.group: "note_pull"`, and
`offer.project: null`. Preserve `inbox` information.

Include these rows exactly once in `all_offer_rows`, the matching mutable row traversal,
and `completion_summary_count`. This makes the existing single-item numeric/key
selection, choice-required refusal, counts, and unknown-pick teaching error work
immediately. Keep all old traversal paths and serialized fields stable when no pull is
present. Final mixed-item ordering is the batch phase's job.

## 4. Core phase: staged planner and picks

Use `CaptureBatchPlanner::current_contents`, `read_existing`, `loaded_texts`, and
`stage`; never directly write files from the planner. Read the source before any pick,
today's staged daily note, and the staged `NoteIndex`. A missing source must not create
the note. Missing daily text or an unusable Pomodoros section uses the existing error
behavior. Freeze eligible rows and keys before writes.

Classify each actual `#task` checkbox using the existing task scanners and configured
status knowledge, with this first-match precedence:

1. Own `^prj`: ignore. Done, cancelled, unknown, or custom-status rows: ignore.
2. Blocked status, an unproven/open prerequisite, or whole-token `#prj_is_blocked`:
   folded `blocked`.
3. Future scheduled date: folded `scheduled`; today is eligible.
4. Whole-token `#hide`: folded `hidden`.
5. Live Task Link under today's open Pomodoros: folded `already_planned`.
6. Pending, Next, or Ready: offer.

Reuse `keep_moving`'s tag and lane helpers and `successors::live_link_keys`; a struck
link or a link under a closed Pomodoro does not make a row planned. Do not inherit
keep-moving's predecessor, leaf-project, configuration, or last-link gates.

Keep the KM31 prerequisite gate: only dependency-bearing open source tasks require the
vault-wide dependents snapshot; held `^prj` dependency metadata may require the same
proof for a blocked-project reason. Overlay staged text through
`DependencyContext::dependents_snapshot`, and respect snapshot completeness. Unavailable
proof folds affected candidates as blocked. No prerequisite snapshot is built for an
ordinary dependency-free pull. A no-pull draft gains no new reads. Use existing
project-block reason wording for a blocked project's empty refusal.

Base placement on `project_note.rs`'s `place_project_task_links`/`offer_entry_role` and
`CaptureTaskThemeContext::placement_for_task`. Share a small helper only where it
preserves existing behavior; do not duplicate or refactor the whole project planner. Use
the normal typed placement rules without predecessor or anchor: matching open type gives
`typed`; a new typed placeholder gives `created`; no type uses the current/next open
entry with `implicit` and `fallback: true`.

Maintain separate scratch and actual day text. Scratch placement runs sequentially
through all offers, so a second DOCS row sees the placeholder the first would create.
Actual placement processes only picks; an unpicked row cannot create a placeholder. Pin
subset selection where the row selected was preceded by an unselected same-type offer,
as well as NP21's ordinary all/none paths.

For selected rows, Ready becomes Next; Pending and Next retain their symbols. Mint a
missing block ID using the shared deterministic algorithm and reservations, stamp
freshness with the standard writer, and insert a plain Task Link. Preserve task
children, schedules, dependency IDs, and unrelated bytes. Unselected rows have
`not_linked: "not_selected"` and no writes or minted IDs. Same-draft creation's stamp
exemption is completed and tested by the batch phase.

Use `CaptureError`'s code and extra-payload mechanism for:

- `pull_note_missing`: teach that `@foo` pulls from absent `foo.md`, bullets start a
  list there, and `bob capture-targets` lists routable notes. Use the contract's exact
  message.
- `nothing_to_pull_in`, regardless of selection flags: additive
  `note_pull: {note, route, folded}`. Distinguish folded residents, no open tasks, and
  blocked-project reasons. Reuse the contract's exact copy.

Test NP1–NP12, NP15, NP21–NP23 at planner level where appropriate, with actual file-byte
assertions for no-pick success, empty/missing refusals, and successful picks. Use the
parser tests for NP24–NP27. Ordinary creation and existing successor tests must remain
green.

## 5. Batch phase: identity, numbering, and atomicity

The current `pull_in.rs::plan_with_pull_in` probes unselected batches for numeric picks,
then re-plans with keys and restores inbox-route answers. Keep that protocol. The batch
loops through staged items, runs `apply_successor_net_post_pass`, detects and applies
inbox routes, then finalizes block trackers. Integrate at those seams.

- Keys use §12.2 KEYS: `<note>#^<id>` or `<note>@<digest>[~k]` from the staged source
  before any pick. Preserve duplicate-description ordinals through minting, line shifts,
  and later row lookup; `project_note.rs`'s frozen/settled ordinal logic is a reference.
  Test selecting only the second of two identical ID-less rows.
- One key has one offering owner per draft. Track earlier offers across every successor
  class, not just pulls. Later pulls omit a row already owned by an earlier item even
  when it was unselected; selected predecessors must not create a second identity after
  minting. Do not write a duplicate pick and merely hide its report.
- Encode note-pull rank after all existing offer groups in `OfferOrderKey`. Within that
  rank, draft item order precedes lane, path, and line; the current final `then(item)`
  tie-break alone is insufficient. Preserve existing groups' ordering and probe numbers.
  NP20 must work with route names in reverse lexical order.
- An earlier explicit link folds that resident `already_planned`. Earlier task creation
  makes it visible, including a newly staged source note. Picking that new row must not
  stamp freshness. Track creation provenance or compare against original rows with
  stable occurrence identity; raw whole-line inequality is insufficient because an
  existing row may have been edited earlier in the same draft.
- Extend the shared net pass to these rows: later link, completion, re-block, and drop
  remove the affected offer. Own picks survive. Note pulls never inherit keep-moving's
  whole-group invalidation or predecessor predicates. Refresh final row positions as
  needed, including ID-less unselected siblings.
- An initially empty source refuses atomically. A previously nonempty offer group
  emptied by later valid items is simply pruned by the net pass; do not retroactively
  turn that valid batch into an initial-empty refusal.
- Preserve existing inbox detection: Ready → Next in an inbox is a route candidate;
  already-Next or Pending picks are not. Cover no answer, required-choice refusal,
  numeric probe with `-i`, and routing the chosen row in the same transaction. Use the
  existing moved-task/block remapping; avoid a second routing engine.
- Revalidate stale, unknown, and out-of-range keys through the existing teaching errors.
  No earlier staged item may commit if any later pull or choice fails.

Own NP13–NP20, plus mixed groups, repeated source pulls, duplicate descriptions,
selected/unselected later-item effects, same-draft source creation, and rollback after a
prior successful staged edit. Reuse the test support's
`capture_json_dry_run_matches_real_flags` where suitable.

## 6. Presentation phase: one copy contract

In `pull_in_copy.rs`, add `GroupKey::NotePull` with source-note identity and correct
first-item ordering. `PullInGroupJson.note` is additive and omitted for other groups.
Attach the source's folded counts and contract copy:

- Title `Pull from ‹N›`, tag `ready in N`, glyph `note_pull`, capsule `Pull in`.
- Fold parts, omitting zeroes: already planned, blocked, scheduled later, hidden.
- Pull-only top copy: `Pull from ‹N›` or `Pull from 2 notes`; mixed groups retain their
  current top-copy logic. Count distinct notes. Empty/pruned groups do not contribute
  phantom titles or folds.

In `output.rs`, render the contract's CLI header, numbered offers with Pending / Next /
Ready lane names, note and optional ID, destination, fold line, picked rows, and
unpicked rows. Use existing status colors and dim fold/hint styling. Pull-only no-pick
output ends with `nothing pulled in · choose with -p 1,3 or -p all`, adapted to
available offer numbers; it replaces the generic successor link-later hint. Honor the
exact singular/plural and post-pick header rules in the contract. Render choice-required
refusals with the same group/fold information. Pin NP1, NP2, partial/none selection,
empty/missing refusals, two-note and mixed-group output.

Add `TaskBlockRole::NotePull` and carry `reason: "note_pull"` onto picked Pomodoro-link
diff lines through `surviving_successor_link_reasons` and the block tracker. Include
routed picks and ensure removed net-pass rows leave no stale reason. Only touched tasks
and picked links produce blocks.

Read `docs.md` through memory before help/doc edits. Add the approved
`bob capture @sase` and `bob capture -p 1,3 -- @sase` help examples and the pull-in
explanation in `capture/cli.rs`. Run `just docs-gen`; inspect generated changes, keeping
source-memory untouched. The already-landed contract is authoritative; fix an actual
inconsistency in the same change, without redesigning settled choices.

## 7. Verification phase: conformance and Mac delivery

Add `tests/cli/capture/note_pull.rs` and register it with the existing CLI test modules
if prior phases have not already done so. Use fixed `BOB_NOW`, deterministic settings,
explicit `-b` and `BOB_DAY_FILE`, and `NO_COLOR` for human snapshots. Maintain this
coverage checklist in the fixture README with actual test names:

| Vectors   | Owner and evidence                                                      |
| --------- | ----------------------------------------------------------------------- |
| NP1–NP2   | Core classification/picks; CLI JSON and human snapshots                 |
| NP3–NP9   | Core liveness, fold precedence, resident exclusions/inclusions          |
| NP10–NP12 | Core refusal codes, payloads, blocked-project reasons, unchanged files  |
| NP13–NP16 | Batch choice protocol, invalid picks, no selection, inbox routing       |
| NP17–NP20 | Batch creation, net filtering, cross-group dedup, item ordering         |
| NP21–NP23 | Core scratch placement, unavailable dependency proof, daily-note errors |
| NP24–NP27 | Core grammar/editor compatibility and revision                          |
| NP28      | Verification: dry-run and real-run parity and atomic writes             |

Generate actual CLI outputs under `tests/fixtures/note_pull/` for NP1, NP2 (`-p all`),
NP10, NP12, NP13, and NP16. Include both useful NP10 empty variants and NP16's
inbox-choice/routed response as needed by the contract. These are synthetic fixture
vaults, never personal vault exports. Capture stdout, nonzero exit expectations for
refusals, exact argv, fixed time, fixture source inputs, and reproduction instructions.
Do not hand-author outputs and label them real-bob. Normalize only nondeterministic
absolute temporary paths using a documented substitution shared with assertions; do not
normalize keys, offer numbers, status, copy, destination roles, counts, or missing
fields. Assert NP1, NP2, and NP13 via the CLI against the saved fixture expectations.
Record the fixture paths and producing revision in the phase note for `bob-cli-gj.3` to
consume after closure.

For NP28 compare the same initial vault/flags under dry run and real execution, allowing
only the documented dry-run/preview wording differences. Verify file preimages stay
unchanged in previews, no-pick commands, choice-required refusals, invalid picks, and
failed multi-item drafts. Reuse existing parity helpers rather than a permissive
whole-payload scrubber.

Read-count tests must show zero new note-pull work on non-pull drafts and no dependents
snapshot for dependency-free pulls. Include unavailable-snapshot and staged-overlay
proof. This is correctness/performance gating, not timing tests whose speed depends on
CI hardware.

Build release and measure `bob capture --dry-run -f json @<route>` on the approved
temporary copy of Bryan's vault, using the largest area/project note and `mac_inbox`.
Follow `/sase_repo` and vault instructions to obtain the isolated source; never invoke
capture on the live vault or commit personal data. Fix the clock/day paths to that copy
and use a day with an eligible Pomodoros section. Record build revision, host, dataset
size, selected route sizes, command, warm-up, sample count, median, p95, and maximum.
Verify the parent design's ≤70 ms target; if missed, profile the new path and remove its
avoidable reads/scans. Report any unavailable representative dataset or unresolved
timing limit accurately to the land agent; do not substitute tiny-fixture timing as
proof.

Run `cargo fmt --all` before the final gates, then `just check` and `just docs-check`.
Focused checks during development are `cargo test --lib capture_language`,
`cargo test --lib note_pull`, `cargo test --test cli note_pull`, and relevant existing
successor/project/inbox tests selected by their actual names. Long builds/checks must
use `/sase_monitor` when required by the runner; wait for the staging command itself to
finish.

## 8. Completion and ownership

Every child phase closes only its own assigned bead after verifying its scoped work.
Before closure, run `sase bead epic-symbols <assigned-id>` and resolve each entry or
re-key it to a still-open owner. Do not add speculative suppressions. Do not create
follow-up beads: append `PROPOSED FOLLOW-UP: <summary — evidence>` to the assigned
phase. A check failure reproduced identically on a clean base is recorded with its
reproduction and existing tracking bead, and does not prevent phase closure. Preserve
evidence for the final land agent.

The child epic's land agent is responsible for assembling all four phases and completing
the delegated `bob-cli-gj.2` scope after its descendants settle. Its acceptance
checklist is the original phase's entire contract: NP1–NP28, fixtures, copy, grammar 7,
atomicity, performance evidence, and both final gates. It must run
`sase bead epic-symbols bob-cli-gj.2`, resolve/re-key leftovers, and close
`bob-cli-gj.2` with `sase bead close bob-cli-gj.2 --note "<verified evidence>"` only
when that work is complete. This is land-agent responsibility, never an instruction for
a child phase worker to close an ancestor. Neither this child epic nor any worker may
close `bob-cli-gj` or another ancestor plan bead.
