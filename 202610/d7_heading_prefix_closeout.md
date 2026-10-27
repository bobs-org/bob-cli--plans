---
tier: tale
title: Finish bob-cli-d7 heading-prefix boundaries and landing
goal:
  Match the approved manual-number grammar, verify the combined tree, and complete
  bob-cli-d7 closeout and its required backlog continuation.
size: medium
proposed_by: bbugyi200.athena.bob-cli-d7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-d7
create_time: 2026-10-27 16:05:29
status: wip
---

- **PARENT:**
  [202610/backlog_batch_a1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_a1.md)
- **BEAD:**
  [bob-cli-d7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-d7/README.md)

# Finish bob-cli-d7: exact heading prefixes and epic closeout

## Outcome

Complete the remaining defect in bob-cli-d7's manual-heading detector, verify the
combined tree, and finish the epic's landing in this coder's own turn. This tale has no
land agent. Do not wait for this work's commit SHA, push, or CI before closeout: the
host commits after the turn ends. The approved parent plan also requires the next
backlog epic proposal after closeout; preserve that continuation.

## Audit already completed

The lander read bob-cli-d7, all four child beads and every child note, the approved plan
`plan:202610/backlog_batch_a1.md`, the five task beads, and the actual source and
commits. On 2026-10-27, HEAD and freshly fetched origin/master were `f00854be`. Only the
four epic commits landed since the epic began; there were no unrelated commits to
integrate:

- `f771b196`: self-numbered Markdown headings.
- `d208c595`: shared note-profile package gate, used by the create, diagram, and
  snapshot-store real-render tests.
- `afaacab0`: the authorized current-walk decision and reference glossary changes,
  generated instructions and public documentation. The accepted old decision body was
  preserved with partial-supersession metadata and a backlink.
- `f00854be`: test-only 10000 ms completion deadlines, bounded PTY readiness waits, and
  load-tolerant timing. Production completion code was unchanged.

The catalog's `code_catalog_report.lua` checks the `unnumbered` class before recording
section numbers, so the new classes keep its labels consistent. Note snapshots do not
pass `--number-sections`. Source review found no other integration gap. The shared
additions to `tests/cli/support.rs` coexist.

The original phases reported passing `just check`; d7.3 reported two passes and five
loaded completion-module runs, 116 tests each, with 0/580 failures. Its historical
baseline was 1/1 loaded failure and 0/1 isolated rerun. It could not obtain a fresh
pre-fix baseline because invalid old zpty calls silently skipped. Do not describe that
historical baseline as a fresh controlled experiment.

Task beads bob-cli-58, bob-cli-5q, and bob-cli-6c are closed with evidence. The code for
bob-cli-d6 and bob-cli-8q landed, but those tasks remain ready because their phase
workers closed only their assigned phases. The lander has already added coordination
notes to bob-cli-3j and bob-cli-b9. The aliases helper now uses the pinned deadline, but
b9 remains ready pending macOS evidence.

Follow-up triage is complete: d7.4 notes 1 and 2 repeat one proposal about the
pre-existing obsolete walk order in `docs/reference/vault-format.md:171-172`. Through
`/sase_new_task`, the lander searched all task statuses, swept the last week, inspected
active epics, and created small bug **bob-cli-d8**, marked ready, with evidence
`file:explicit:362600ca13c4854b1fed2c18`. No proposals were declined. The outcome is
recorded on bob-cli-d7. Do not create a duplicate or expand this repair to d8. No other
distinct proposed follow-ups were present.

`sase bead epic-symbols bob-cli-d7` reported no entries; this checkout has no
`symvision` recipe. Recheck both at closeout. bob-cli-d7 currently has no parent.

## 1. Repair the manual-number predicate

Read the parent bead and plan through their audited commands. In
`src/native/highlights_ref/create.rs`, the embedded Lua `has_manual_number` around line
180 currently trims the heading, takes its first nonspace token, strips one optional
trailing `.` or `)`, and accepts any remaining digits/dots. That admits invalid prefixes
and changes numbering for every heading in the document.

The lander extracted `PANDOC_CODE_BREAK_FILTER` into a temporary Lua file and ran
`pandoc -f markdown -t latex --number-sections --lua-filter <file>` on each heading
followed by `## Ordinary`. Both of these incorrectly produced starred sections for
`Ordinary`:

```markdown
## 1.

## Ordinary
```

```markdown
## 2.3.. Topic

## Ordinary
```

The approved grammar is a leading integer plus `.` or `)` followed by whitespace, or a
dotted sequence of integer components with at most one optional trailing `.` or `)`. A
bare year such as `2026 roadmap` is not a prefix. Match the heading's plain-text
stringification. For the simple `N.` / `N)` branch, require the following whitespace
specified by the approved plan. For dotted prefixes, validate complete integer
components instead of a loose digits-and-dots character class. Do not accept an empty
component or multiple trailing punctuation marks. Keep the author's text and identifiers
untouched.

Preserve document-wide unnumbering for valid prefixes, the lone-H1 title exclusion,
explicit unlisted-heading exclusion, TOC/bookmark membership, and unchanged
ordinary-document numbering. Change only the predicate and focused regression coverage.
No CLI options, memory edits, release/version edits, or existing PDF rewrites are
needed.

Extend the pandoc-backed section-numbering tests beside
`section_numbering_self_numbered_doc_prints_author_numbers_only`:

- Negative: bare `1.` and `1)` headings, malformed `2.3.. Topic`, and empty components
  such as `2..3 Topic` leave `Ordinary` numbered.
- Positive: `1. Evidence`, `10) Risks`, `2.3 Details`, `2.3. Details`, and
  `2.3) Details` leave every listed heading unnumbered and listed.
- Include a manually numbered lone H1 with ordinary H2s and an explicitly unlisted
  numbered heading, proving neither independently triggers the mode. Keep the existing
  text/id and TOC assertions.

## 2. Verify and finish the resolved task records

Run the focused section-numbering tests and the note-profile gate tests. Run
`cargo test --test cli highlights::store_gaps -- --nocapture` on athena and confirm the
snapshot test actually renders when packages are installed. The original phase verified
only the missing-package report parser. Also exercise the actual skip path: prepend a
temporary executable `kpsewhich` shim that exits 1 to PATH for the exact snapshot-route
test, while leaving real pandoc/xelatex available; assert its missing-package skip
message. Remove the shim afterward. A real apollo run is welcome but not necessary after
this deterministic skip-path proof; report honestly that the lander did not run apollo.
All tests use their scratch vaults.

Run `just check` as the file-change gate, using `/sase_monitor` if it needs a handoff;
the continuation must resume this tale's closeout. Do not run `just check-full`. Do not
broaden testing once the relevant checks pass without new evidence. Existing phase
evidence already supplies the five loaded runs and two completion full-suite passes;
this repair does not change completion code.

After verification, close bob-cli-d6 with the shared-gate, real-render, and skip
evidence; close bob-cli-8q with commit f00854be, the honest before/after record above,
and the phase's two passing checks. Supplement bob-cli-58 with the precise
prefix-boundary correction and new test evidence. Keep bob-cli-b9 open for macOS
reverification. Re-read readiness before closing anything, and never force.

## 3. Finish bob-cli-d7's landing and required continuation

This final step is owned by the tale coder, not a future land agent.

1. Re-read bob-cli-d7 and every descendant and note, plus the linked plan, and
   fetch/review any new origin/master commits since f00854be. Integrate any actual
   conflict with this feature and rerun affected checks. Confirm the accepted memory
   decisions were honored and all five original task outcomes are recorded.
2. Resolve the current tale's own completion record normally if it is a descendant that
   would otherwise block the parent close; all implementation and verification must be
   complete first. Mark this tale's linked plan done as appropriate. Never use `--force`
   to advance a successful nested landing or wait for this turn's future commit. If
   another unfinished descendant appears, complete it or record the genuine blocker
   instead of forcing it closed.
3. Run `sase bead epic-symbols bob-cli-d7`. Resolve every entry by wiring it,
   privatizing it, adding an appropriate non-test pragma, or deleting it under the
   Symvision policy; re-key only if a still-open later bead genuinely needs it.
4. Close normally with `sase bead close bob-cli-d7 --note "<verification>"`. The note
   must identify all four reviewed phases and commits, the prefix repair, current check
   results, post-start integration review, the original five task resolutions, the
   coordination notes, and the one follow-up outcome (d8, proposed by d7.4 notes 1/2,
   none declined). If rejected for symbols, clean them and retry. If rejected for
   unfinished phases, finish them; force is only for a deliberate canceled/superseded
   outcome with a specific reason, never to make close succeed.
5. Run `just symvision` when the recipe is available; otherwise record its absence. Open
   the plans repository via `/sase_repo`, use the printed path, and set `status: done`
   in **202610/backlog_batch_a1.md**, the plan shown by bob-cli-d7. Read its contents
   through `sase artifact read plan:202610/backlog_batch_a1.md`.
6. Run `sase bead read bob-cli-d7 -r "Need the parent link"`. Its current parent is
   null, so there is no ancestor to close. If that changed, follow the user's
   parent-phase/parent-plan readiness rules, closing only verified ancestors normally;
   stop and note any incomplete or ambiguous ancestor.
7. Fulfill the approved parent's required successor before ending: use `/sase_plan` to
   author, validate with `--explain`, revalidate without it, and propose the next
   backlog epic as the final handoff. This successor is a separate backlog batch, not
   another repair child of d7. List every open task (`sase bead list -t task -n 0`),
   read candidates with audited reads, and close only verified obsolete or duplicate
   tasks with evidence. Choose at most five important tasks in at most five phases, each
   with a unique slug and explicit dependencies. Rank red gates and flakes first,
   correctness/data-safety next, then misleading memory, then features. Recheck b9 after
   f00854be; suggested other candidates from batch A1 are d4, d5, ai, 5n, cv, and 5m,
   not a mandatory selection. Open any other repo with `/sase_repo` first; check active
   sase epic sase-1oh before planning cv. Features require yes/no decisions; memory
   changes require `/sase_memory_write` decisions. Skip snoozed tasks until their wake.
   Human Mac acceptance, cron, privileged installs, live write passes, and app UX checks
   need their actual gate/question/decision, never assumed results. Copy this successor
   rule into the new epic's Landing section so the chain continues until no open tasks
   remain. Wait for `sase plan propose` itself to exit; a successful handoff ends the
   turn without a final declaration. If no tasks remain, finish normally with
   `/sase_final`, declaring every repository changed by this turn.
