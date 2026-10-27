---
tier: epic
title:
  "Backlog batch A1: red test gates, double-numbered headings, and stale walk/ref memory"
goal: "The five most important open task beads are resolved: just check is green on
  apollo and load-tolerant on athena, rendered research PDFs stop double-numbering
  headings, and the decisions roster and glossary describe today's review walk and
  reference tasks. The lander then starts the next backlog epic, so the chain continues
  five beads at a time until no open task beads remain.

  "
decisions:
  memory_walk_record:
    ask:
      Add a decisions strand for the current walk tiers and mark review-walk-is-tiered
      superseded in part?
    memory:
      - decisions:review-walk-is-tiered
    corrects:
      PRE → NEW → PROJECTS → PENDING → NEXT → TICKLER → REFERENCES → ROTTEN → POST
    default: true
    answer: true
  memory_glossary:
    ask:
      Correct glossary task-freshness, reference-note, and area-note to match today's
      walk and reading tasks?
    memory:
      - glossary:task-freshness
      - glossary:reference-note
      - glossary:area-note
    corrects: Its [[reference-task]] (`^ref`) represents progress through that material.
    default: true
    answer: true
phases:
  - id: snapshot-test-gate
    title: Skip the snapshot store test when note-profile LaTeX packages are missing
    depends_on: []
    size: small
    description:
      "snapshot-test-gate: move the note-profile package skip helper into shared test
      support, gate store_snapshot_create_records_note_snapshot_route and any other
      ungated real snapshot render with it, and close bob-cli-d6."
  - id: manual-heading-numbers
    title: Stop pandoc numbering headings the author already numbered
    depends_on: []
    size: medium
    description:
      "manual-heading-numbers: teach the number_from_top_level Lua pass to leave
      self-numbered Markdown documents unnumbered but listed, cover it with filter tests
      and a scratch render, update docs/highlights-create.md, and close bob-cli-58."
  - id: completion-load-flakes
    title: Make the completion PTY and zpty tests load-tolerant
    depends_on: []
    size: medium
    description:
      "completion-load-flakes: reproduce the bash and zsh completion driver failures
      under load, pin test-only deadlines and replace fixed sleeps with bounded
      readiness waits, prove repeated loaded runs pass, and close bob-cli-8q."
  - id: memory-corrections
    title: Correct the stale review-walk and reference-task memory
    depends_on: []
    size: small
    description:
      "memory-corrections: per the accepted memory decisions, add a decisions strand
      partly superseding review-walk-is-tiered, fix glossary task-freshness,
      reference-note, and area-note, republish memory and docs, and close bob-cli-5q and
      bob-cli-6c."
proposed_by: bbugyi200.athena.14v
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-27 14:54:30
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch_a1.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_a1.md)

# Backlog batch A1: red test gates, double-numbered headings, and stale walk/ref memory

## Context and triage record

This epic opens a chain of backlog epics. Each epic resolves at most five open task
beads, and its lander plans the next one. On 2026-10-27 the planner read all 112 open
task beads and verified the obsolete and duplicate candidates against bob-cli master
(`5bb53caa`), bob-mac-capture, bob-plugins, sase, athena services, and memory. Earlier
sweeps were 12v on 2026-10-21 and the bob-cli-an.6 lander on 2026-10-23.

Closed during triage, each with an evidence note on the bead:

- **bob-cli-b4** (done, already resolved). The Multivisor web UI on athena:8100 returns
  200 with its built frontend, installed 2026-10-26, and its assets load. `/api/data`
  still returns 401.
- **bob-cli-39** (superseded by epic bob-cli-c9). Bare `-f` rereads as `<root>__vN`
  reading families, and `-f overwrite` replaces a reading in place, keeping the note,
  id, and reading task. bob-cli-80 (`--absorb`) stays open.
- **bob-cli-3y** (superseded, consolidated into bob-cli-5q). Schema 13 retired the
  REFERENCES tier, and the walk-order item was already recorded. The only live part
  left, the PROJECTS `#hide` gate, supersedes the same record as 5q, so it moved into a
  CONSOLIDATED note on 5q.

Triage notes added without closing: bob-cli-58 (defect 1 fixed by `55334b09`),
bob-cli-cv (sase epic sase-1oh covers its first fix only), bob-cli-ar (the same
duplicated title block also appears in glossary:task-relation), bob-cli-1x (the new
`scripts/dev/macos-ci-check` gate exists, but there is no CI-watch guidance yet), and
bob-cli-ai (the crate is 7.84 MiB against an 8 MiB warning and a 9 MiB failure).

Why these five beads first:

- **bob-cli-d6** (ci): `just check` fails every run on apollo.
- **bob-cli-8q** (flake): this is the completion-test flake family. 3j notes #4 and
  #7–#12 record about six separate landings tripping on it under parallel load.
- **bob-cli-58** (bug): about half of the research PDFs Bryan reads in Highlights
  double-number their headings.
- **bob-cli-5q** and **bob-cli-6c** (memory): both make every agent's instructions wrong
  today. The always-loaded decisions roster gives a walk order that no longer exists,
  and glossary:reference-note still describes in-note `^ref` trackers. Agents planning
  ref features were misled by the latter at least twice (6c +1s).

The four phases share no files and no ordering, so they run in parallel. Each phase
closes the task beads it resolves with `sase bead close <id> -n "<evidence>"`. Each
phase reads its own bead(s) first with `sase bead read <id> -r "<why>"`.

## Phase snapshot-test-gate (bob-cli-d6)

`tests/cli/highlights/store_gaps.rs` `store_snapshot_create_records_note_snapshot_route`
(around line 201) checks only for `pandoc --version` and `xelatex --version` before it
runs a real note-snapshot render. The note print profile
(`src/native/highlights_ref/note_snapshot/note.tex`, `profile.rs`) needs `titlesec` and
other packages. apollo's TeX Live has no `titlesec.sty`, so the test fails there every
run. bob-cli-d3.2 and .3 reproduced it on the clean base tree.

1. `skip_if_missing_note_profile_packages(vault)` exists twice, in
   `tests/cli/highlights/create.rs` (around line 42) and
   `tests/cli/highlights/diagrams.rs` (around line 39). It reads the `latex_packages:`
   row from `bob highlights doctor`. Move one copy into the shared test-support module
   the highlights CLI tests already use, keep its skip message, delete both duplicates,
   and update their callers.
2. Call it from `store_snapshot_create_records_note_snapshot_route`. Then find every
   other test that runs a real note-snapshot or note-profile xelatex render (for example
   grep for the pandoc/xelatex version gates in `tests/cli/highlights/`) and gate the
   ones that lack the helper. Pure pandoc/xelatex Markdown-route tests that do not use
   the note profile keep their current gates.
3. Verify. On athena, `cargo test --test cli highlights::store_gaps` must still run the
   real render and pass. Then show the skip path. Preferred: run the same test on apollo
   from a scratch copy (rsync without `target/` and `.git`; read the reference memory
   `tailnet.md` for SSH access), and delete the copy afterward. If apollo is
   unreachable, prove the skip with the helper's own seam (for example a doctor stub, if
   the existing tests already use one) and say the apollo run was not done. Finish with
   `just check`.

## Phase manual-heading-numbers (bob-cli-58)

Commit `55334b09` fixed the first defect: a lone H1 becomes an unnumbered, unlisted
title and H2s number from 1. The second defect remains. The `number_from_top_level` pass
in the embedded Lua filter in `src/native/highlights_ref/create.rs` (around lines
177–206) never inspects heading text, so an agent heading such as `## 1. Evidence`
prints as `1 1. Evidence` in the body, the Contents list, and the bookmarks. bob-cli-58
measured 147 of 291 chat PDFs double-numbered.

Design (decided here, not a reviewer decision): keep the author's numbers and drop
pandoc's.

1. A document is **self-numbered** when any listed heading carries a manual number
   prefix. The lone title H1 that is already excluded does not count. A prefix is a
   leading `N.` or `N)` followed by whitespace (`1. Evidence`, `10) Risks`), or a dotted
   multi-level number with optional trailing punctuation (`2.3 Details`,
   `2.3. Details`). A bare integer followed by a space, such as `2026 roadmap`, is not a
   prefix. Match against the heading's plain-text stringification, not raw inlines.
2. In a self-numbered document, add the `unnumbered` class (never `unlisted`) to every
   heading. Pandoc then prints no section numbers, while the Contents list and PDF
   bookmarks still show every heading with the author's text. Documents with no prefix
   keep today's numbering exactly.
3. Heading text and identifiers must not change, because the pass adds only classes.
   Return-link pills and code-catalog anchors depend on those ids. Read
   `code_catalog_report.lua` (it also reads `number_sections`) and keep any section
   labels it records consistent for self-numbered documents.
4. Scope: the plain-Markdown route and `-M` vault-note renders. Note snapshots do not
   pass `--number-sections` (see the `profile.rs` assertion). Never re-render existing
   library PDFs: Highlights annotations are bound to them.
5. Tests: extend the pandoc-backed filter tests next to `LONE_H1_SOURCE` (around line
   6215). Cover these cases:
   - a self-numbered document (no section numbers, all headings listed in the TOC);
   - a mixed document where only some headings are numbered (whole document unnumbered);
   - a year-led heading, which keeps numbering;
   - a lone H1 combined with numbered H2s;
   - a document with no manual numbers, unchanged from today.
6. Real check: write a fixture report like bob-cli-58's repro (`# Report title`,
   `## 1. Summary`, `## 2. Details`), plus one real agent report fetched with
   `sase artifact read`. Render both with `bob ref create <file> -o /tmp/<name>.pdf`.
   Never write into the vault. Confirm with `pdftotext -l 2 -layout` that the TOC reads
   `1. Summary`, not `1 1. Summary`.
7. Update the numbering paragraph in `docs/highlights-create.md`, then run
   `just docs-check` and `just check`.

## Phase completion-load-flakes (bob-cli-8q)

`tests/cli/completion/bash.rs::bash_readline_inserts_what_bob_returned` fails under
parallel `just check` but passes alone in about 21 s. The sibling
`completion::zsh_adapter::real_zsh_first_tab_completes` shows the same signature. 3j
notes #4 and #7–#12 also saw protocol and vault completion subsets fail, in disjoint
sets, on clean trees. They name three suspects:

- the spawned `bob` inherits the production completion deadline (no
  `BOB_COMPLETE_DEADLINE_MS`, so 150 ms);
- wall-clock budgets;
- fixed sleeps in the PTY and zpty drivers.

1. Reproduce first. Run the completion test module repeatedly under deliberate CPU load:
   for example several concurrent `cargo test --test cli completion::` runs, or a busy
   loop on every core. Record the failure rate and which suspect fires. Instrument
   temporarily if needed; never commit debugging output.
2. Fix in the tests, not in production behavior:
   - pin an explicit, generous test-only deadline in the environment of every spawned
     completion `bob`;
   - replace fixed sleeps in the PTY and zpty drivers with bounded readiness waits: poll
     for the prompt or marker the driver already prints, under an overall timeout of
     about 30 s;
   - make any wall-clock performance budget load-tolerant, following the
     `7b8c5ddb`/`0e75ca56` precedent for capture perf gates.

   The production default deadline stays unchanged. Do not weaken correctness
   assertions, add `#[ignore]`, or wrap tests in retry-on-failure loops.

3. bob-cli-b9 (macOS CI: `complete_values(&["bob", "task", ""])` returned `[]` in
   `tests/cli/aliases.rs`) is not this phase's bead. If `complete_values` goes through
   the same deadline path and your pinning covers it, apply the pin there as well. Then
   add a phase note saying so, so the next epic can re-check b9. Do not close b9.
4. Verify: at least five consecutive loaded runs of the completion module pass where the
   baseline failed, then `just check` twice. Put the before and after failure rates in
   the closing note for bob-cli-8q.

Epic bob-cli-3j (shell completion) has every phase closed but has sat in landing since
2026-10-02. Its notes say it owns these flakes, but no 3j work is in flight. This epic's
lander records a coordination note on 3j (see Landing).

## Phase memory-corrections (bob-cli-5q, bob-cli-6c)

Use `/sase_memory_write` and edit only what the accepted memory decisions cover. Read
5q, including its CONSOLIDATED note from 3y, and 6c, including its three +1s, before
editing.

> [!decision] memory_walk_record

The current contract is in `docs/freshness.md`: schema 14, the queue order around line
430, the vocabulary and schema history around lines 511–513, and the PROJECTS gate
around line 503. The walk is PRE → NEW → PROJECTS → PENDING → NEXT → RECURRING → TICKLER
→ ROTTEN → POST.

1. Add one decisions strand that records the current walk; bob-cli-5q suggests
   `recurring-occurrences-walk-when-due`. It covers three changes:
   - due recurring occurrences walk in RECURRING between NEXT and TICKLER. They are
     never stamped and resolve through completion, a reschedule past today, Today
     linkage, or cancellation. 5q lists the selected `tier_position` and `ctrl_alt_f`
     answers;
   - schema 13 retired the REFERENCES tier. References are ordinary tasks in the NEW,
     lane, TICKLER, and ROTTEN groups, and `^prj` stays the only tracker exception;
   - a `^prj` walks in PROJECTS only when it lacks `#hide`, which `bob projects sync`
     owns. There is no open-task occupancy count and no frontmatter `scheduled` gate.

   Give the strand Applies to, Claim, Why (with rejected alternatives), Cost, Reopens
   when, and Evidence: the plans, commits, and docs that 5q and its notes name. Where a
   reason is not recorded in those sources, ask Bryan rather than infer it.

2. Mark `decisions/review-walk-is-tiered` with `metadata.status: superseded-in-part`,
   `superseded_by`, and a back-link stating what was retired. Do not reword its accepted
   body.

> [!decision] memory_glossary

3. In `glossary/task-freshness`, replace the tier sentence with the current nine-tier
   order. Link the new strand if memory_walk_record was accepted.
4. `glossary/reference-note`: a reference note holds no open tasks. It shows its reading
   task (`#task #ref`, living in the parent area, project, or inbox note's `## Tasks`)
   through the managed embed below the H1. Frontmatter `parent` is projected from where
   that task lives, and annotation follow-up tasks default to the parent note. Keep a
   short mention of the frozen in-note `^ref` form as the legacy v1 shape. Rework the
   phrase "gathers source links, notes or highlights, and any follow-up work" so it no
   longer implies follow-up tasks live in the note. That concern was consolidated from
   bob-cli-4n.
5. `glossary/area-note`: drop "a reference note's own [[reference-task]]" from the
   residence exceptions. Do not add inbox-routing wording; bob-cli-4t owns that.
6. Leave `glossary/reference-task` alone; bob-cli-c9.6 (`94cd2d95`) already fixed it.

> [!decision] memory_walk_record = no

> [!decision] memory_glossary = no

If a reviewer turned a memory decision off, skip its steps and file nothing for them.
Close each bead once every accepted part is done. If a bead lost only some of its parts
(5q spans both decisions), close it `done` and name the declined part in the closing
note. If every part a bead needs was declined (6c needs only memory_glossary), close it
`-R canceled` with a reason citing the declined decision.

Finish with `sase memory init`, `just docs-gen` (the public glossary and decisions pages
are generated from memory), and `just docs-check`. Close 5q and 6c with evidence.

## Landing

1. Confirm that bob-cli-d6, bob-cli-58, bob-cli-8q, bob-cli-5q, and bob-cli-6c are
   closed, each with evidence or with the reviewer's declined decision. Triage every
   phase's `PROPOSED FOLLOW-UP:` notes through `/sase_new_task`.
2. Add a note on bob-cli-3j recording that completion-load-flakes fixed the completion
   driver load flakes (bash and zsh nodes), with its commits and loaded-run evidence.
   Add a note on bob-cli-b9 if that phase recorded `complete_values` evidence.
3. **Successor epic (required).** As the lander's final action, author and propose the
   next backlog epic with `/sase_plan`, following the same rules:
   - List every open task bead (`sase bead list -t task -n 0`). Read candidates with
     audited `sase bead read`. Close only verified duplicates or obsolete beads, with
     evidence notes: `-R superseded` for duplicates or replaced work, `done` for work
     already fixed. Re-check this plan's triage record rather than trusting it.
   - Choose the at most five most important remaining beads, in at most five phases,
     with explicit `depends_on` and unique slugs. One phase may work several beads. Rank
     red gates and flakes, then correctness and data-safety bugs, then memory that
     misleads agents, then Bryan-facing features.
   - Add plan decisions that let Bryan opt out of feature work or memory changes that
     may not be wanted. Feature beads get a yes/no toggle. Memory changes get
     `/sase_memory_write` memory decisions. Keep other decisions to a minimum.
   - Skip snoozed beads until they wake: bob-cli-ah 2026-11-04, bob-cli-aa 2026-11-06,
     bob-cli-cp 2026-11-16, and bob-cli-6q, whose four weeks of journal data arrive
     around 2026-11-09. Beads that need Bryan's hands get a gate, question, or decision
     in the plan, never a guess. Those are Mac acceptance runs (bob-cli-ay, bob-cli-cx),
     the Mac crontab (bob-cli-30), sudo installs (bob-cli-ax), approved write passes
     (bob-cli-cm, bob-cli-co), and the running-app UX review (bob-cli-20).
   - Copy this successor rule into the new plan's own Landing section, so each lander
     keeps the chain going until no open task beads remain.
4. Suggested starting order for the next batch, from this triage (re-verify first):
   - bob-cli-ai: crate split, now 7.84 MiB;
   - bob-cli-5n, bob-cli-cv, bob-cli-5m: sase-runner bugs. Open the sase repo with
     `/sase_repo`. 5n is still live in `plan_propose_handler.py`, which exits without
     restoring the scratch plan when inlet publication fails after the archive move.
     Check sase epic sase-1oh before cv. Consider asking Bryan to move these to the sase
     project;
   - bob-cli-b9 (re-check after completion-load-flakes);
   - bob-cli-d4 (Mac CI flake), bob-cli-d5 (Mermaid class), bob-cli-ax, and bob-cli-8n;
   - the small memory corrections: bob-cli-ar, bob-cli-4o, bob-cli-7a, bob-cli-bu, and
     bob-cli-1x;
   - the decision-record memory backlog, then features.
