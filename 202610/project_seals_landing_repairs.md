---
tier: tale
size: medium
title: Repair project-offer identity and Mac reporting, then land bob-cli-ex
goal:
  Preserve duplicate project offers, report every picked link once, verify the Mac
  contract, and close bob-cli-ex.
proposed_by: bbugyi200.apollo.bob-cli-ex.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ex
create_time: 2026-10-31 19:19:05
status: wip
---

- **PARENT:**
  [202610/project_note_seals.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/project_note_seals.md)
- **BEAD:**
  [bob-cli-ex](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ex/README.md)

# Repair project-offer identity and Mac reporting, then land bob-cli-ex

Finish only the remaining work found while landing `bob-cli-ex`. This is one bounded
coding task across bob-cli and its linked bob-mac-capture repository. The coder must
finish the epic's closeout in this same work session; this tale has no later land agent.

## Evidence and constraints

Read `bob-cli-ex` and its landing audit note with `sase bead read`, and read
`plan:202610/project_note_seals.md` through `sase artifact read`. The original plan is
the contract. The lander reviewed all notes on ex.1–ex.4, ex.2.1 and ex.2.1.1–ex.2.1.3,
the child plan `plan:202610/project_note_offer_form.md`, and the implementation commits
`76f23471`, `9a30ec4c`, `e9b9cd1d`, `e8dd3b65`, `42dd92bc`, `e2838ac3` in bob-cli and
`6b94f06`, `891fe11` in bob-mac-capture. All descendants are closed. The epic has no
`parent_bead` as of this audit. Its plan is not done and the epic remains open.

The final DECISIONS remain `initial_marks = none` and `memory_glossary = yes`, limited
to successor-offer/task-successor. Those glossary changes are already present. This
repair requires no memory edits, new options, version changes, or CHANGELOG edits.
Preserve Create serialization, explicit picking, unchanged unpicked task IDs, no
freshness stamps on newly created tasks, and byte equality with authored colon bullets.

At audit time bob-cli HEAD and fetched origin/master were `e2838ac3`; the linked Mac tip
was `891fe11`. Non-epic commits after the first epic implementation are `ecffd6fa`,
`36c48075`, `efe2ebe0`, `17702dca`, and `6ae5df00`. Keep status/cache/docs changes are
independent. Route-only `@@` and `%c` clipboard parsing already integrate with sealed
project forms; `%c` remains refused on project-note captures. Mac has no non-epic
commits after its first epic implementation. Recheck subsequent drift before closing,
especially the active capture-directives work; do not implement that epic.

The lander built the current Bob and reproduced the identity bug below in a scratch
vault. No product files have been repaired yet. `just check` is the file-change gate; do
not run `just check-full`.

## 1. Preserve frozen identities when a duplicate is picked

In `src/native/capture/project_note.rs`, `digest_ordinal_for` counts ID-less rows in the
pre-pick render. `src/native/capture/task_complete.rs::successor_task_lookup` later
resolves an ID-less project offer by the frozen ordinal among only currently ID-less
tasks. Picking an earlier duplicate mints its ID and removes it from that population, so
later unpicked rows are lost or matched to the wrong row.

Reproduction with a scratch `cash.md` area, `day.md` containing
`## Pomodoros\n- [ ] (1330-1400) Work\n`, an empty config, `BOB_DAY_FILE` pointing at
that day, `BOB_NOW=2026-09-20 14:31:07`, and `TZ=UTC0`:

```text
Finish packet @cash:pkt@
- Draft memo
- Draft memo
```

Run `bob capture -b <scratch-vault> --no-clip --dry-run -f json` with that stdin. It
reports keys `cash_pkt.md@c0084bb3` and `cash_pkt.md@c0084bb3~2`, lines 13 and 14. With
`-p 1`, current Bob incorrectly reports `offer_count: 1` and omits the second row. With
`-p 2` or `-p all`, it reports two. The unselected row still exists in the staged note;
this is an output/identity defect, not an eligibility change.

Carry enough pre-pick source identity through planning and final reconciliation to
resolve every original offer correctly after ID minting and subsequent line shifts. Do
not replace this with stale line-number lookup, mint IDs for unselected tasks, or count
every authored-ID task as an original ID-less duplicate. Keep public keys frozen.
Preserve final status/line refresh and the previous landing's later-blocked
Ready/Next/Pending suppression.

Extend `tests/cli/capture/project_note_offer.rs`: two and three equal descriptions; pick
first/middle/last by number and opaque key; preview and commit; unselected IDs stay
empty; all expected offers and their original keys/numbers survive; actual lines match
final note rows. Include a later item inserting context before the duplicates, plus an
authored-ID row with the same description so identity domains cannot be conflated. Keep
existing byte-equality and blocked-row tests passing.

## 2. Count every linked project task once and repair the Mac tests

Open bob-mac-capture with `/sase_repo` and read its `AGENTS.md` before changes.

`Sources/CaptureCore/CaptureProjectNotePresentation.swift` currently removes all picked
successor IDs from `note.taskLinks`. `NotificationService.pulledInSummary` also excludes
project-note successors, as the original design correctly requires. Together these
suppress picked tasks from both notification summaries. An offer-only project has no
Linked summary; one authored colon task plus one picked task says Linked 1 instead of
Linked 2. The original contract is to count all `task_links` once in the project
notification, leaving project successors out of the general Pulled-in count. Restore
that behavior. Keep any desired preview deduplication separate from the notification
count so it cannot drop committed links.

Update the incorrect expectations in
`Tests/CaptureCoreTests/CaptureProjectNotePresentationTests.swift` and
`Tests/BobMacCaptureTests/NotificationServiceTests.swift`. Cover picked-only,
authored-colon-plus-picked, and unselected offers: the notification lists the actual
linked count exactly once and never adds a second Pulled-in project count.

Mac CI run <https://github.com/bobs-org/bob-mac-capture/actions/runs/41251130345> at
`891fe111f61a40af9917480184e7260588bb3967` passed app Build but failed Test at
compilation: `NotificationServiceTests.swift:1590` and `:1595` omit required
`CaptureProjectTaskLink.blockLink`. Supply the real links in those initializers; do not
weaken the production initializer. This is caused by ex.4, not an unrelated CI task.
Also fix the new
`CapturePullInChoiceModelTests.testProjectNoteOfferFixtureOpensNewProjectChoiceUnmarked`
context expectation: it expects `Finish the exit packet!`, but the generated fixture's
`unblocked_by.text` is `Finish the exit packet`. Keep the generated contract
authoritative.

## 3. Finish the real-Bob fixture contract and project choice flow

The ex.3 note #2 proposal remains incomplete: the new generator saves only
`project-note-successors.json`; it asserts but discards a parse response and never
generates the existing sealed-toggle fake replies. `Tests/Fixtures/fake-bob` still
labels those branches hand-authored. A fresh Bob comparison found both rewrite replies
identical, but the `Finish it @cash:goog-exit@#admin` parse fixture omits the real
`pomodoro_name.themes` metadata.

Extend `Scripts/generate-project-note-offer-fixtures.sh` to persist deterministic
real-Bob parse and both sealed-toggle responses, and have fake-bob consume the generated
responses. Normalize only unstable scratch paths and use an isolated config/environment.
Include the named offer parse so its themes and unwashed name span are exercised.
Continue verifying dry-run, required-choice refusal without writes, and explicit
selection/commit. Keep old-Bob missing-field decoding tests.

Use the generated project-offer response for panel coverage of Skip (`-p none`, create
note but no picked links), Escape (unchanged draft, no submit), and marked opaque-key
submission with Control-Return open intent. Reuse existing test helpers and keep waits
bounded. Preserve empty `markedKeys`, project context and glyphs, mixed-set context, and
the unchanged generic successor flow. Recheck the manila chip tests, especially theme
metadata after the seal and picker-wash restoration.

## 4. Verification and drift

Build Bob from the repaired tree and regenerate the Mac fixtures against that binary.
Run focused project-offer tests, then one `just check` in bob-cli. Run docs checks for
any affected docs; do not repeat equivalent broad gates without a new change or failure
to investigate. Format only changed Swift lines.

Run the linked repo's `just portable-check` (on athena if this host has no Swift). For
changed macOS targets, use the required scratch rsync copy on `mac`, excluding
`.build`/`.git`, then `Scripts/bundle.sh` and `just typecheck-tests`, and remove the
scratch copy. Never install or use Bryan's checkout. The lander's SSH probe timed out.
If the Mac stays offline, use the repo's existing `Scripts/macos-ci-check` precommit
snapshot gate to compile and run the changed macOS tests. It verifies a throwaway
working-tree snapshot without changing HEAD or the index and does not require this
tale's eventual host commit. Use `/sase_monitor` for that long check. Do not order
closeout after the finalizer's commit, its SHA/push, or master CI. Inspect later CI
steps as well as lint and distinguish confirmed unrelated failures from these repairs.
The known epic-caused compilation/test errors must be fixed and checked before close.
When rendering is available, inspect the chip in light/dark and increased contrast;
otherwise explicitly report the visual tuning limitation, as the original plan allows.

Fetch/review post-audit changes in both repositories and resolve actual integration
conflicts with this feature. Record check evidence and any remaining external limits.
The article completion flake is separately tracked as `bob-cli-f2`; its omission is not
remaining project-seal work.

## 5. Complete bob-cli-ex landing in this same session

Carry the lander's follow-up dispositions into the close note: ex.1 #1 became
`bob-cli-f2`; ex.3 #1 compilation/visual evidence is handled by this repair and its
verification (not a new task); ex.3 #2 fake-bob contract is completed above; ex.2.1.3 #1
trailing `#hide` exception was deliberately declined because edge `#type` syntax already
requires escaped or mid-text literal tags and predates this epic.

Read
`sase bead read bob-cli-ex -r "Verify final descendants, notes, and plan readiness"` and
any newly added descendants/notes. Confirm the repairs and original DECISIONS, all child
resolutions, and linked-plan readiness. Run:

```sh
sase bead epic-symbols bob-cli-ex
```

Resolve every listed entry (wire up, privatize, add a justified non-test pragma, or
delete under Symvision policy), or re-key only a genuinely needed exemption to its
still-open later bead. The audit currently lists none. Then close normally:

```sh
sase bead close bob-cli-ex --note "<original audit, integration review, repairs, exact verification, and all follow-up dispositions>"
```

If symbols reject the close, clean them and retry. If a named phase is incomplete,
finish/reopen it. Never force merely to succeed or to advance a successful nested
landing; canceled/superseded work requires a deliberate reason and resolution.

After closing run `just symvision` when available (there is currently no recipe). Open
the plans sidecar with `/sase_repo`, resolve/read `plan:202610/project_note_seals.md`
through `sase artifact read`, and set only its frontmatter `status: done`. Use the PLAN
path shown by the bead, never a hard-coded workspace path. If a closed child's linked
plan is still marked active, verify its readiness and finish that status too, without
reopening completed work.

Finally run `sase bead read bob-cli-ex -r "Need the parent link"`. No parent exists at
audit time. If one has been attached, follow the landing prompt's exact parent rules:
close a verified parent phase normally and leave its epic to its lander; for a parent
plan, re-audit all descendants, notes, plan, drift and symbols before normal closure,
symvision and plan status, repeating only while complete. Record and report the first
incomplete or ambiguous parent. Submit `/sase_final` with commit decisions for every
changed repository, including the plans sidecar.
