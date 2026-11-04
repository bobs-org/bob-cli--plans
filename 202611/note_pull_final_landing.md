---
tier: tale
title: Finish note-pull source reporting and land bob-cli-gj
goal:
  Preserve source-note groups through moves, finish the Mac copy contract, verify both
  clients, and close bob-cli-gj.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gj.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gj
create_time: 2026-11-04 10:07:11
status: done
---

- **PARENT:**
  [202611/note_pull.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/note_pull.md)
- **BEAD:**
  [bob-cli-gj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gj/README.md)

# Finish note-pull source reporting and land bob-cli-gj

This tale repairs the remaining work in **bob-cli-gj** and closes that epic in the same
coding turn. It has no land agent afterward. All implementation phases are already
closed; do not repeat the feature or reopen completed phases just to run this repair.
The epic has **no parent_bead** as of this audit.

## 1. Accepted scope and audit evidence

Read these through the audited interfaces:

- `sase bead read bob-cli-gj -r "Need landing audit, triage, and final decisions"`.
- `sase artifact read plan:202611/note_pull.md "Need the approved note-pull contract"`.
- `sase artifact read plan:202611/note_pull_cli.md "Need CLI acceptance requirements"`.
- `sase artifact read plan:202611/note_pull_latency_move_landing.md "Preserve the completed move and latency repair"`.
- `sase artifact read file:explicit:2d4e46033f084dad02ac11a0 "Inspect the reproduced source-group drift"`.

The land audit used bob-cli **27feff87** and bob-mac-capture **206c4b8**. It read every
note on gj.1, gj.2, gj.3, nested epic gj.2.1, and gj.2.1.1 through gj.2.1.4, the seven
matching CLI commits, and the Mac commit. Relevant later integrations are target runs,
project-note moves, and capture transfer moves/merges. The fixed initializer collisions
and existing NP1–NP28 coverage remain in place. The current tree builds with
`cargo build --locked --bin bob`. Grammar is **10**, superseding the epic's earlier 7
and coordination notes' 9; preserve any newer revision that has landed when this tale
starts.

Keep the final decisions: `lanes = open_lanes`, `empty_pull = refuse`,
`memory_decision_record = no`, and `memory_successor_offer = no`. Make **no memory
edits**. Keep explicit picks, stable pre-pick keys, Pending/Next/Ready ordering, atomic
writes, unchanged schedules, and the same-draft creation stamp exemption. Do not edit
Cargo versions or CHANGELOG.md. Do not add commands or flags.

Use `/sase_repo` to open `bob-mac-capture` and `plans`; use only their returned checkout
paths. Read the Mac AGENTS.md. Read `docs.md` through `/sase_memory_read` before
changing contracts. Never run capture against the live vault.

## 2. Preserve the source note when an offered task moves

The completed repair cbc35adf makes a note-pull row follow a later project or transfer
move: `row.note_path` is its final location, while `offer.key` remains frozen. This is
correct for links, task blocks, selection, and deduplication. However,
`pull_in_copy.rs::group_key` and `output.rs::print_human_note_pull_groups` still group
by `row.note_path`. Consequently a single pull becomes multiple source groups, with
wrong titles, counts, and potentially missing source fold counts.

Reproduction on the built audit tree: an isolated area `work.md` contains Pending Alpha
`^alpha`, Next Beta `^beta`, and Ready Gamma `^gamma`; area `bob.md` is empty; a fixed
day contains one open FIX Pomodoro. Use fixture Tasks settings and explicit
`BOB_DAY_FILE`, `BOB_NOW=2026-10-05 09:30:00`, and a missing `BOB_CONFIG_FILE`. Run each
as a fresh dry-run with `-p all`:

```text
@work

@work+gamma > @bob
```

```text
@work

Docs push @work^docs-push@
- ^gamma
```

Both currently report `Pull from 2 notes`, and human output splits three picks into
`Pull from work · 2 of 2 pulled in` plus a destination header with `1 of 1`. There was
only one `@work` pull. Existing tests in
`tests/cli/capture/note_pull.rs::note_pull_move_project_and_transfer_follow_picked_row`
check final identities and total counts but miss group provenance.

Make source provenance explicit across the existing JSON protocol:

1. Add optional **`offer.note`** to `TaskCompleteOfferJson`, populated only for
   note-pull offers with the source `NotePullSummaryJson.note`. Omit it for other
   groups. Keep it frozen through net filtering and moves, exactly like the key. Update
   every Rust constructor consistently; keep the known leading
   `CaptureItemResult.note_pull` initializers in `plan.rs` where they are.
2. Group note-pull rows and their human output using this source field, with the
   previous `row.note_path` fallback for old/synthetic payloads. Join folded counts by
   the source note. Keep the row's final note path, block id, line, and link truthful;
   never revert the move-aware identity fix to repair the title.
3. In Mac `CaptureModels.swift`, decode the additive field optionally on the offer.
   `CapturePullInChoice.swift::PullInSectionIdentity.init(row:)` must use this source
   note for note-pull membership, falling back to the row note path when absent.
   Preserve Bob's group copy and order. Do not parse opaque offer keys or recompute
   eligibility in Swift.
4. Update `docs/task-dependencies.md` §12.20 and the relevant JSON reference to specify
   that `offer.note` and `pull_in.groups[].note` identify the requested source, while
   `note_path` identifies the final resident location. This is an additive report
   repair; no grammar or schema revision is needed.

Extend the existing move tests with both forms, none/all/numeric/key selections, one
source split across two destinations, and two requested sources moved into a shared
destination. Assert one group per requested source in draft order, exact source titles
and pick counts, each source's fold counts exactly once, stable offer numbers/keys,
correct final paths/links, and unchanged dry-run bytes. Include an
all-offered-rows-moved case with a hidden source resident so lost fold counts are
observable. Preserve existing repeated-pull, rollback, and final-identity dedup tests.
Use a real CLI output fixture to test the same grouping on the Mac, including the flat
`pull_in_choice_required` offers path; keep old-field-absent decoding covered.

## 3. Finish the Mac copy contract

These are narrow omissions in 206c4b8, not a new design:

- `CapturePanelModel.primaryActionTitle` checks `previewResults.count == 1` for
  `note_pull`. A nonempty draft containing two or more note-pull results must also say
  **Pull In**. Mixed drafts keep their existing action rules. Use decoded results, not
  draft parsing. Cover single, two-note, mixed, and empty preview states in model tests.
- `CaptureNotePullPresentation.notificationTitle` currently adds `task`/`tasks`, and
  `NotificationServiceTests` pins that divergent copy. Match the approved title
  **`📥 Pulled in k from ‹N›`** exactly for one and several selected rows. Keep the
  selected-text/destination body and mixed-draft summary behavior.
- The pull-only prompt currently names only its first note even when Bob's groups name
  several. Use the source group identities to say `Choose tasks to pull in from 2 notes`
  for multiple distinct sources, retaining `Choose tasks to pull in from ‹N›` for one.
  Document this plural extension next to the existing prompt copy and test
  VoiceOver/status copy, Bob group order, and distinct source memberships. Skip stays
  hidden on pull-only cards; Esc returns to the unchanged draft; Shift-Return on that
  card remains explanatory.

Retain the existing calm empty refusal, list-start helper, tray glyph, Next tint, and
inbox routing behavior. Existing Mac NP1, empty, and inbox fixtures are present; refresh
them from real Bob output only where the additive field changes their bytes. Add the
move and two-source cases to the appropriate decoding, presentation, model, and
notification tests. Do not rewrite unrelated views.

## 4. Verification and drift

First inspect commits since the audit SHAs in both checkouts, including any newer
transfer and move-choice work. Integrate conflicts within the source-reporting scope. Do
not lower the current grammar revision, duplicate offer traversal, or undo the cached
theme/catalog and deterministic note traversal optimizations.

- Run focused note-pull unit/CLI tests, relevant target-run and transfer integration
  tests, and `cargo check --all-targets --locked`. Run the file-change gate
  **`just check`**, using `sase tool run check` as appropriate. Never run
  `just check-full`. A check-full failure after check passes is infrastructure work, not
  this epic's unfinished feature.
- Run `just docs-check` in each repository whose docs changed. The current
  `task-merge.md` frontmatter blocker is already routed to bob-cli-go; if it still
  occurs, establish the same unrelated failure on the base and retain its owner. Fix any
  new note-pull documentation failure.
- Run Mac `just portable-check`. Compile the changed app and tests through a scratch
  rsync copy on the Mac with `bundle.sh` and `just typecheck-tests`, then remove the
  scratch copy, following the Mac AGENTS.md and audited tailnet memory. Never use
  Bryan's checkout or install there. Never close with uncompiled Swift. If this cannot
  run, keep the epic open and record the concrete blocker.
- The prior Mac phase compiled and typechecked but its landed CI run **41851831313** for
  **206c4b8** was queued at audit time. Inspect its settled result, distinguish
  unrelated existing failures from note-pull defects, and fix note-pull defects. The
  tale's closeout must not depend on a SHA, push, or CI run for this tale's own
  uncommitted changes. Use the SASE monitor workflow for long commands or pending
  external checks; do not end a turn with a live command.
- The committed `tests/fixtures/note_pull/performance.md` records a 6,907-note isolated
  release measurement at p95 **50.41 ms** for `sase` and **30.37 ms** for `mac_inbox`,
  both below 70 ms. The latency proposal is resolved. This repair adds no scans;
  preserve that property. If implementation changes the read/placement path, rerun the
  existing isolated benchmark and meet the unchanged budget.

## 5. Follow-up outcomes already settled by the lander

The root epic has a `FOLLOW-UP TRIAGE` note recording the `/sase_new_task` search,
last-week task sweep, causal-epic review, and these three outcomes. Carry all three into
the close note; do not create duplicate tasks:

1. **gj.2.1.2 note #1 — Clippy warnings:** declined. `nonminimal_bool` and
   `result_large_err` are advisory under the canonical explicit deny list; it does not
   use `-D warnings`. Historical bob-cli-v deliberately resolved the mechanical gate
   only. No evidence supports reopening or corroborating it.
2. **gj.2.1.4 note #3 — latency:** was epic acceptance work and is now resolved by
   cbc35adf and the passing committed performance evidence, not a follow-up task.
3. **gj.3 note #2 — docs generator:** independently reproduced on clean 27feff87 with
   `uv run --project tools/docs bob-docs gen --check`: exit 2 because
   `sase/memory/glossary/task-merge.md` has no frontmatter. Corroboration is on causal
   active epic **bob-cli-go**; **bob-cli-go.13** owns its accepted memory output. No
   standalone task and no memory edits here.

Use `/sase_new_task` for any genuinely new unrelated finding. This tale's source
identity and Mac copy defects are caused by this epic and must be completed here.

## 6. Close bob-cli-gj in this same turn

This is the final implementation step and cannot be deferred to another lander or
ordered after this work's own commit, push, or CI result.

1. Re-read `bob-cli-gj`, gj.1/.2/.3, gj.2.1, and gj.2.1.1 through gj.2.1.4 through
   `sase bead read ... -r ...`. Check for new notes, open descendants, and linked plan
   readiness. Include the verified repairs, integration review, checks, and all three
   triage outcomes above in the close note. If an actual blocker remains, leave the epic
   open and record it rather than forcing a successful close.
2. Run **`sase bead epic-symbols bob-cli-gj`**. Resolve every listed exemption by
   wiring, privatizing, adding an appropriate non-test pragma, or deleting it; re-key
   only when a specifically identified still-open later bead needs it. This audit found
   none. Never leave an exemption keyed to a bead about to close.
3. Close normally with
   **`sase bead close bob-cli-gj --note "<what was verified, repaired, integrated, tested, and triaged>"`**.
   If stale symbols reject it, finish their cleanup and retry. If a named phase is
   unfinished, finish it or reopen deliberately. Never use `--force` to make a
   successful landing advance; cancellation/supersession requires its actual
   justification and resolution.
4. After closing, run **`just symvision` if available**; neither audited checkout
   currently has that recipe. If available at implementation time, it must pass.
5. Open `plans` via `/sase_repo` and set **`status: done`** in the frontmatter of the
   epic's linked plan **`plan:202611/note_pull.md`**, the file **`202611/note_pull.md`**
   under that returned checkout. Read its content via `sase artifact read` before
   editing. Do not leave the status update for a later turn.
6. Run `sase bead read bob-cli-gj -r "Need the parent link"`. This epic has no parent,
   so finish normally. If the link has changed, follow the user's parent rule: close
   only a verified parent phase normally, or fully re-audit a parent plan's descendants,
   notes, linked plan, drift and symbols before closing it; stop and record a blocker on
   the first incomplete or ambiguous ancestor.
7. Finish through **`/sase_final`**, declaring every changed repository (bob-cli,
   bob-mac-capture, and plans) for host commit. Closing before this host commit is the
   normal flow; do not commit manually or wait for the new SHA.
