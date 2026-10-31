---
tier: tale
title: Finish project-offer validation and land bob-cli-ex.2.1
goal:
  Repair the project schedule guard and final offer identity/state reporting, verify the
  combined tree, and close bob-cli-ex.2.1 and parent phase bob-cli-ex.2 normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ex.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ex.2.1
status: done
---

- **PARENT:**
  [202610/project_note_offer_form.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/project_note_offer_form.md)
- **BEAD:**
  [bob-cli-ex.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ex/bob-cli-ex.2.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ex.2.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ex.2.1.land.md)
- **COMMITS:**
  - [4d46180](https://github.com/bobs-org/bob-cli--plans/commit/4d461800f67260ed99d9ebd1510197189ed1875f)
    — docs(plans): mark project_note_offer_form done for bob-cli-ex.2.1 landing

# Finish project-offer validation and land bob-cli-ex.2.1

Complete only the remaining bob-cli work found by the land audit of `bob-cli-ex.2.1`.
This is a single-coder tale. Its coder must finish the landing in the same turn as the
final code; there is no separate tale land agent and no later resumption to rely on. Do
not wait for this work's own commit, push, SHA, or CI.

## Existing work and authoritative scope

Read `bob-cli-ex.2.1` and its notes with `sase bead read`, then use audited artifact
reads for `plan:202610/project_note_offer_form.md` and
`plan:202610/project_note_seals.md`. The parent phase is **bob-cli-ex.2**; its
containing epic is **bob-cli-ex**, which has its own land agent.

The three child phases are closed and their implementation is present:

- `9a30ec4c` / `bob-cli-ex.2.1.1`: rendered task metadata, frozen offer keys, selection,
  shared colon-bullet placement, scratch destinations, output plumbing.
- `e9b9cd1d` / `bob-cli-ex.2.1.2`: execution/editor grammar, completion, sealed
  separator rewrites, CLI help and protocol tests.
- `e8dd3b65` / `bob-cli-ex.2.1.3`: contracts, authorized glossary updates and acceptance
  tests, including byte-equal Offer versus authored colon-bullet writes.

Preserve settled decisions: offers start unselected; a pick uses the existing colon
writer and creates no freshness stamp; `^prj` and placeholders are not offered; Create
serialization remains unchanged. The two authorized glossary changes are already
complete. This repair needs no memory edits or new options. Do not edit package versions
or `CHANGELOG.md`.

At audit HEAD `e8dd3b65`, fetched `origin/master` was identical. Non-epic commits after
the first child commit were `36c48075` (remove `@@route+id`), `efe2ebe0` (Keep status),
and `17702dca` (Keep guide). Reviewed parser, rewrite and docs retain route-only `@@`
declarations and item-local project seals; the Keep changes require no integration.
Recheck drift before finishing and preserve these landed contracts.

The lander read every child note and completed follow-up triage, recorded on
`bob-cli-ex.2.1`. The sole proposal, `bob-cli-ex.2.1.3` note #1, asked for an exception
for trailing `#hide` in project bullets. It was declined: the existing documented edge
`#type` grammar deliberately requires escaping literal tags. The pre-seal installed bob
reproduces the same refusal using the old caret-plus spelling; `\#hide` and mid-text
`#hide` both preserve the tag. Same-type/all-type searches, last-week task sweeps and
the active-epic review found no duplicate or causal active owner requiring a task. Do
not create a new task for that proposal. Carry this outcome into the final close note.

## 1. Repair the future-project guard

In `src/native/capture/project_note.rs`, `project_link_guard` currently requires both
`project_blocked` and a rendered named task with `link = true`. Thus an Offer project
with only ordinary or caret-ID bullets has no `project_schedule` report, despite being
blocked. The approved child plan explicitly requires the guard even without an explicit
colon bullet. The existing `offer_blocked_project_offers_nothing_and_links_nothing` test
claims this case in its comment but actually includes `:collect-docs`.

Add a public CLI regression using fixed time and a future schedule:

```text
Finish packet s:3 @cash:pkt@
- Draft memo
- Collect docs ^collect-docs
```

Assert `successors: []`, no task links, zero offers/picks, and
`link_guard.reasons == ["project_schedule"]` for default selection and `-p all`. Verify
the project is created with its blocked residents and no daily-note write; also cover
absence of the daily note because this path has neither offers nor actual links. Add the
Offer branch to the guard without changing the existing Create behavior. Preserve the
existing refusal for an unused named destination.

## 2. Reconcile project offers against the final staged task identity and state

The remaining source-audited defects are in
`src/native/capture/task_complete.rs::apply_successor_net_post_pass`,
`keep_unblocked_row`, and `successor_task_lookup`, with offer construction in
`capture/project_note.rs`:

- An ID-less row accepts any task now occupying its old `row.line`. Its fallback
  compares a cleaned final description to raw `row.text` and ignores the frozen
  duplicate ordinal. Later insertions can therefore match a different task or drop a
  still-valid offer containing task fields.
- The net pass only retains/removes rows. It never refreshes the reported final line,
  even when a block-ID lookup or fallback found the task at a different line.
- The later-Blocked exclusion only applies when the reported status is Ready or Next
  (`' ' | '*'`). Newly supported Pending (`'/'`) project offers can remain offered after
  a later draft item makes the task Blocked.

These are source findings, not claims of a fresh runtime test failure. Reproduce them in
focused regressions first, using this checkout's binary. The installed bob predates
project seals and cannot verify this feature.

Use existing fixture helpers in `tests/cli/capture/project_note_offer.rs`. A useful
line-shift case is a project with `Anchor ^anchor` followed by an ID-less task, then a
separate `Add context @cash_pkt+anchor` item. Repeat with a typed ID-less task (for
example `[type:: FIX]` in the typed fixture), duplicate descriptions, and a selected
sibling. Verify every surviving row's `line` addresses its own task in the final written
note. Preview/submit and numeric/key picks must keep the same frozen identity;
unselected rows must not acquire committed IDs. Cover a stale line occupied by another
task, not only a line occupied by plain context.

For Pending blocking, create `- [/] Pending work ^pending` and then use a
relationship-only item such as `@cash_pkt+pending &cash:gate`, with an open `^gate`
prerequisite in the area fixture. Confirm the final task is Blocked and the project
offer disappears. Cover the Ready/Next/Pending variants and preserve existing
selected-row link-survival behavior. Keep the already-covered `s:0` case: a selected row
authored as colon under a resolved non-future project schedule legitimately reports `?`
and must still match Create byte for byte.

Implement a bounded internal identity reconciliation for project-created rows. Reuse
existing stable-key/cleaned-description conventions and available staging or task
tracking rather than treating line numbers as identity. Carry any additional identity
metadata internally only; preserve the public schema and the pre-pick offer key after ID
minting/retyping. Resolve against final staged contents after relevant mutations, and
update final line/status metadata coherently. Do not apply the Done-successor engine's
freshness mutations to newly created tasks. Keep behavior of unrelated
Done/relation/Pomodoro-close rows unchanged except for any shared correctness fix that
is necessary and regression-covered.

If a proposed regression does not fail, record the exact observed result and existing
protection; do not broaden into speculative refactoring. The guard fix and any
reproduced final-state defects remain epic work until verified.

## 3. Verify the repaired tree

Run focused project-note and successor tests while implementing, then run the canonical
**`just check`** gate (the named `sase tool run check` runs it and records evidence). Do
not run `just check-full`. A pass of `just check` with an independently observed
`check-full` failure is test-infrastructure work, not this epic's scope. Use
`/sase_monitor` if verification needs a handoff; carry these remaining closeout steps in
its continuation, rather than ending with a running command.

Retain the existing byte-equality tests for implicit, named, typed, and resolved
schedule destinations; Create's omitted fields; choice-required/unknown-pick atomicity;
draft-wide numbering; and no automatic linking or freshness stamps. Recheck origin/base
drift and integrate any relevant newer capture changes. If documentation changes become
necessary, first read `docs.md` through the memory skill, regenerate generated pages
instead of editing them, and run `just docs-check`. The prior phases' reported broad
passes are historical evidence, not a substitute for verification of the repairs.

## 4. Finish the epic and its parent phase in this turn

After the fixes and required verification pass, re-read `bob-cli-ex.2.1`, all three
child beads and any new descendants/notes. Confirm descendant and linked-plan readiness.
If this tale is represented by a descendant bead, finish and close that completed bead
normally before attempting its parent; do not leave an unfinished descendant that would
block the close. No part of this step depends on the coder's own post-turn commit, push,
SHA or CI result.

1. Run `sase bead epic-symbols bob-cli-ex.2.1`. Resolve every entry by wiring it,
   privatizing it, using an appropriate non-test pragma, or deleting it according to the
   Symvision policy. Re-key a Justfile exemption only when a still-open later bead truly
   needs it. The audit found no entries, but recheck now.
2. Close normally with `sase bead close bob-cli-ex.2.1 --note "<verification>"`. Write a
   concrete note naming the reviewed child commits, repaired guard/identity/state cases,
   test results, post-start integration review, inherited decision compliance, and the
   declined trailing-#hide proposal from `bob-cli-ex.2.1.3` with its reason above. If
   rejected for symbols, finish that cleanup and retry. If rejected for unfinished
   phases, complete or reopen the needed work. Never use `--force` merely to pass the
   close, and never force a successful nested landing. Only a deliberate
   canceled/superseded outcome permits
   `--force --reason ... --resolution canceled|superseded`.
3. Run `just symvision` if available. There is currently no symvision recipe; report
   that fact if still absent. Open the plans sidecar with `/sase_repo`, use its returned
   path, and set `status: done` in **`202610/project_note_offer_form.md`**, the file
   identified by **`plan:202610/project_note_offer_form.md`**. Include that modified
   repository in the final declaration.
4. Run `sase bead read bob-cli-ex.2.1 -r "Need the parent link"`. The expected direct
   parent is phase **bob-cli-ex.2**. Read that phase, verify this child plan completed
   its offer_form scope and every note, and run `sase bead epic-symbols bob-cli-ex.2`;
   resolve or legitimately re-key every entry. Then close only that phase normally with
   `sase bead close bob-cli-ex.2 --note "<implemented offer contract, child closure, repair verification and integration evidence>"`.
   Leave containing epic **bob-cli-ex** and `plan:202610/project_note_seals.md` to its
   waiting land agent. If the parent scope is incomplete or ambiguous, record the
   concrete blocker on bob-cli-ex.2 and report it; do not force closure.
5. Complete this tale's own lifecycle/status as required and finish through
   `/sase_final`. The final response must distinguish the closed child epic and parent
   phase from the still-owned containing epic.
