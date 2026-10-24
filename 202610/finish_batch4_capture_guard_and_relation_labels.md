---
tier: tale
title: Finish batch 4 capture safety and land the epic
goal:
  Close the remaining capture guard and relation-label gaps, integrate Mac tests, verify
  and close bob-cli-an.6.6.6, and preserve its backlog continuation.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6
status: done
---

- **PARENT:**
  [202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md)
- **BEAD:**
  [bob-cli-an.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.land.md)
- **COMMITS:**
  - [0a8ca46](https://github.com/bobs-org/bob-cli--plans/commit/0a8ca4610cc97eacdda6b71b01867d9c78035aca)
    — chore(plans): mark backlog batch 4 epic plan done

# Finish batch 4: capture option clusters, authoritative relation labels, and integrated Mac tests

This is the remaining work from the landing audit of **bob-cli-an.6.6.6**, not another
backlog batch. One coder can finish these bounded Rust/Swift repairs, verify them, and
complete the landing. There is no later land agent for this tale.

Read the epic and its latest landing note with `sase bead read`. Read the approved plan
through
`sase artifact read plan:202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md "Need the approved batch-4 scope and closeout contract"`.
The finalized decision is **capture_flag_guard = refuse**. Do not revisit it. Use
`/sase_repo` to open bob-mac-capture and bob-plugins before reading those repos, and
follow their AGENTS.md. Before relying on the grammar contract, read
`decisions:mac-capture-is-a-thin-client` with `/sase_memory_read`.

## Verified work and audit evidence

- All four phase beads `.1` through `.4` are closed. Every phase note was reviewed;
  **none contains a PROPOSED FOLLOW-UP**. Epic task beads **bob-cli-b5, bob-cli-6v,
  bob-cli-7i are now closed** by the lander with independently checked evidence. Leave
  their completed implementation intact. **bob-cli-8a and bob-cli-al remain open**, as
  does the epic.
- bob-cli `30f10f15` persists the scratch acceptance vault's git identity.
  [CI 40084439229](https://github.com/bobs-org/bob-cli/actions/runs/40084439229) is
  fully green at that SHA. Integrated `0afeb2f3` has a green lint job (120848942441),
  package job, and Ubuntu job in
  [CI 40085996699](https://github.com/bobs-org/bob-cli/actions/runs/40085996699).
- bob-plugins `e6b2ec2` fixes the frozen clock and the same-note reverse-conversion
  BLOCKS mirror. The lander reran both focused files: 29/29 pass. Full-suite monitor
  `yac9rg630jn6` is terminal success, 3529/3529 tests. The phase's sync record confirms
  all seven plugins synced, navigation-hotkeys 2.56.1, zero drift.
- bob-mac-capture `22598dc` ties suggestion responses to label/generation/draft.
  [CI 40084777082](https://github.com/bobs-org/bob-mac-capture/actions/runs/40084777082)
  actually executes the relation race tests successfully, resolving the phase's local
  XCTest limitation. Its only failures are the three known RefsLibraryTests tracked by
  **bob-cli-b7**.
- bob-cli `0afeb2f3` adds the flag guard. Mac `8400bd3` gates the pull-in guard on a
  positive preview from the same executable. These implementations have the gaps below.
- The lander fetched all three origin/master branches. Since the epic started at
  2026-10-24 05:39 EDT, all commits in those repositories belong to this epic; no
  unrelated post-start drift needed integration. Repeat this comparison when coding
  finishes, because more changes may land.
- `sase bead epic-symbols bob-cli-an.6.6.6` reported no entries. bob-cli currently has
  no `symvision` recipe. Recheck both at closeout.

Do not use an existing `target/debug/bob` blindly: this checkout contained an August
binary while Cargo's configured `target_directory` was elsewhere. The lander's first
binary probe was therefore discarded as current-tree evidence. Use `cargo metadata` to
locate the actual target, or `cargo run`/the integration-test binary after a build.
Every mutation probe must explicitly set a temporary BOB_DIR, config/state roots, and an
inbox fixture; do not use the live vault. One mistaken diagnostic wrote a unique
`--bogus` inbox task; the lander immediately removed its exact line and verified its
absence. There is no recovery work to repeat.

## 1. Finish the unknown-option guard without changing valid capture text

In `src/native/capture_flag_guard.rs`, the short-option scan recognizes an entire
cluster of switches, or a value-taking flag only when it is the first character. It
treats a valid `-nf json` cluster as TEXT and stops before `--bogus`. A rustc harness
importing the **current source module** confirms:

```text
["-nf", "json", "--bogus", "--", "hi"]: rejected=false
["-f", "json", "--bogus"]: rejected=true
```

First reproduce through the freshly built CLI in an isolated vault. Fix the scan to
consume valid short clusters exactly through their value-taking option, including
separate and attached values, and preserve format selection for the JSON error. Examples
to cover include `-nf json`, `-nfjson`, and an equals-only optional option combined with
another declared option where clap accepts it. Audit the five consumers (`capture`,
`capture-complete`, `capture-parse`, `capture-rewrite`, `capture-templates save`)
against their real clap declarations; do not assume every command accepts every cluster.
Prefer deriving needed metadata from the command if that simplifies keeping it correct,
or add a focused declaration-consistency check; avoid a broad CLI refactor.

Add integration regressions that assert an unknown alphabetic long flag in option
position exits 2, emits the correct human/JSON error, and leaves the temporary vault and
template shelf unchanged. Include capture-complete with its required cursor. Keep
declared options, option values, `--2`/`--1`, a bare `--`, literal text following `--`,
and flag-looking words after TEXT working as before. Preserve the accepted scope:
unknown short spellings and numeric capture grammar are not newly forbidden. Run the
focused tests before the final `just check`.

## 2. Keep relation-label submission authoritative in bob

`Sources/BobMacCapture/CapturePanelModel.swift` now contains
`relationLabelAuthoredReplacement`, a Swift port of label collapse and quoting. That
violates the accepted thin-client decision and fails validation parity. An extracted
harness running the exact current production submit method and helper shows `see (old)`
is inserted as `|"see (old)"` both while a response is pending and when a **fresh**
typed row has `replacement: ""` plus
`disabledReason: "label contains a forbidden character"`. Submission checks for a
nonempty replacement before consulting the disabled reason, then falls through to the
Swift formatter. Rust's `src/native/task_relations/label.rs` rejects parentheses,
several other characters, invalid starts, and labels over 40 characters. The current
Swift disabled-label test incorrectly gives the disabled candidate a nonempty
replacement, so it misses the real wire response.

Remove the local label formatter; do not repair it by porting more Rust rules. Use bob's
existing `capture-complete` typed candidate as the source of replacement and refusal. A
fresh response is identified by the request/generation and exact authored input, not by
comparing its normalized display label to raw authored text. Bob legitimately collapses
whitespace and strips a trailing colon.

Keep immediate Return useful: if the matching result is still pending, record one
pending submit for that prompt/draft/authored input, obtain the current result, and
commit it once only after bob supplies a valid replacement. A user edit, dismissal,
different prompt, draft change, or superseding request must cancel that pending intent.
An unavailable, failed, or old-bob response leaves the authored input visible with an
appropriate retry/error state; it must never synthesize syntax or silently insert an old
label. Consult `disabledReason` before using a replacement, including when that
replacement is empty. Retain the existing valid no-label gesture.

Audit both `applyRelationLabelSuggestions` and `handleRelationLabelCompletion`. Stale or
unverified results cannot replace current validation, including responses without a
query and a prompt reopened against the same draft. Keep visible candidate actions
consistent with the same freshness policy.

Extend the current race tests with realistic bob responses: empty replacement on a
disabled row, pending invalid input, normalized whitespace/trailing colon, >40 chars,
and punctuation that bob refuses. Keep delayed, out-of-order, immediate Return,
edit-after-Return, dismissal/reopen, and repeated Return covered with bounded async
waits. The `old` → `new` case must eventually insert only bob's `|new` replacement. No
local Swift quoting or validation implementation should remain.

## 3. Integrate the positive capability contract with Mac model tests

[CI 40086037387](https://github.com/bobs-org/bob-mac-capture/actions/runs/40086037387)
on `8400bd3` completes 2932 tests with **24 failed assertions**. Three are bob-cli-b7;
the other 21 are new assertions in:

- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`
- `Tests/BobMacCaptureTests/CapturePullInChoiceModelTests.swift`
- `Tests/BobMacCaptureTests/CaptureTemplateShelfModelTests.swift`

Those tests still expect `--require-pull-in-choice` for an unpreviewed draft or a
fixture lacking positive `pull_in.supported`.
`testGuardRefusalOpensChoiceWithoutFailureOrDraftLoss` also uses a failed preview rather
than positive capability proof before asserting the guard was sent. Review the actual CI
log and each fixture before updating it. The CapturePullInClientTests and
CaptureRelationPanelTests suites pass on this SHA.

Make expectations reflect the intended contract. Ordinary/no-capability captures must
omit the guard and preserve every other argv, draft, override, template-origin, and
open-intent assertion. Tests that exercise an actual guard refusal must first obtain a
successful capability-bearing preview from the same executable. Retain positive and
negative cases, including executable identity replacement and stale draft previews;
never restore an unsafe unconditional flag just to satisfy a test. Audit negative
`record.contains` assertions and argv-count substrings too, because an obsolete expected
string can make a no-submission test pass vacuously.

## 4. Verify the repaired tree

- Run bob-cli focused capture/guard tests, then **`just check`** on the integrated
  change. Never run `just check-full`. If help/docs change, read docs.md memory first
  and run `just docs-check` with synchronized golden references.
- Run Mac `just portable-check`. For the AppKit source/test changes, compile a scratch
  copy on `mac` per AGENTS.md (read tailnet memory): rsync without .build/.git, run
  `bundle.sh` and `just typecheck-tests`, then remove only that scratch copy. Never use
  Bryan's checkout or `just install`. Mac CLT has no XCTest; typechecking is not an
  executed XCTest run. Use the existing extracted-method approach for executable
  regression checks when AppKit tests cannot run locally, and report that limit. Do not
  make this tale's closeout depend on its own future commit, SHA, push, or CI. Existing
  landed CI evidence is usable; future CI is not a prerequisite for same-turn closing.
  The next CI run must have meaningful repaired tests, not disabled gates.
- All new verification failures caused by these repairs remain this tale's work. The
  known b7 set is preexisting. If any other failure recurs, establish its cause and use
  `/sase_new_task` for genuinely unrelated discoveries before closeout.
- Use `/sase_monitor` for long verification, with a continuation that resumes this tale
  **and its landing**, not a final-only pass message. A monitor successor must still
  execute step 5. Do not use a prepared completion that skips closeout.

## 5. Finish this epic's closeout and the inherited continuation

The coder owns this step; there is no later lander. Nothing here waits for this tale's
own commit. Its host commits only after the provider turn ends. Before closing, rerun
source/drift and descendant readiness checks and record verification accurately.

**Follow-up triage already completed by the lander:** there were zero child proposals.
The unrelated three RefsLibrary failures were deduplicated via `/sase_new_task` onto
**bob-cli-b7**, with a +1 and the CI runs recorded. No new task was created. The
integrated bob-cli macOS job also had one
`capture::related::draft_related_stays_under_the_debug_perf_gate` overrun (1.712423541s
versus its 500ms limit) in run 40085996699. No matching task was found in all-type
search or the weekly sweep; immediate filing was **declined** because one observation
does not establish a reproducible CI defect or a same-tree fail/pass flake. This epic
does not change the capture-related path. Do not silently promote that observation into
a proved flake or add unrelated performance implementation here. If verification
supplies further evidence, triage it through `/sase_new_task`. Include all these
outcomes in the close note; the epic's latest landing note also records them.

1. Verify `bob-cli-8a` and `bob-cli-al` against the repaired behavior and close them
   with `sase bead close <id> --note "<concrete evidence>"`. The other three assigned
   tasks are already closed; check their notes rather than recreating work. Read all
   current descendants and notes to ensure no additional work appeared.
2. Run **`sase bead epic-symbols bob-cli-an.6.6.6`**. Resolve each listed exemption
   (wire up, privatize, non-test pragma, or delete according to Symvision policy), or
   re-key it only to a still-open later bead that actually needs it. Never leave an
   exemption keyed to a closing bead.
3. Close normally with **`sase bead close bob-cli-an.6.6.6 --note "<verification>"`**.
   The note must cover the four original phases, the three repairs above, honored
   `capture_flag_guard=refuse` and thin-client decisions, exact checks and limits,
   post-start integration review, and every follow-up disposition. If close rejects
   leftover symbols, fix them and retry. If it rejects unfinished descendants, finish or
   reopen them. Never use `--force` merely to succeed or to advance this successful
   nested landing; cancellation/supersession requires a deliberate factual reason.
4. Run **`just symvision`** if that recipe is available. Open the plans repo with
   `/sase_repo` and set **`status: done`** in the frontmatter of
   **`202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md`**,
   the exact linked epic plan. Read artifacts through `sase artifact read`; use the path
   returned by `sase repo open` for the status edit.
5. Re-read the epic's parent link. The concrete parent is **bob-cli-an.6.6**, a plan
   bead, already closed when this audit ran. Review its previous landing notes, all
   descendants and notes, its linked plan
   `plan:202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md`,
   and post-child drift. Rerun descendant and linked-plan readiness checks. If still
   complete, retire symbols with `sase bead epic-symbols bob-cli-an.6.6`, invoke normal
   `sase bead close bob-cli-an.6.6 --note "<rechecked evidence>"` (already closed is a
   no-op; use `sase bead note` to preserve new evidence), run symvision if available,
   and ensure its plan status is done. Repeat this process through the directly parented
   plan ancestors **bob-cli-an.6**, then **bob-cli-an**, obtaining each plan path from
   its audited bead read. At the first incomplete or ambiguous parent, leave its status
   alone, record the blocker on that parent, and report it. Never force ancestors
   closed. If a link unexpectedly resolves to a phase instead, verify the child
   satisfied that phase and close only that phase normally, leaving its containing epic
   to its waiting lander.
6. Preserve the backlog campaign below. Complete this tale's authorized work and
   closeout before handing off the next batch; do not implement a sixth task here.
   Re-enumerate tasks because three were closed in this audit (133 before those
   closures). Record the successor ownership durably before proposal. If a plan proposal
   will terminate this turn, the original approved epic explicitly authorizes
   `/sase_git_commit` for every changed repository first, including the plans status
   edit; use it only under that explicit authorization. This is **after** closeout,
   never a prerequisite to it. Otherwise use `/sase_final` as the last action of the
   normal response and declare every changed repository.

### Mandatory continuation contract for the next bounded backlog epic

The approved batch-4 plan requires continuing until the entire project's task backlog is
resolved. The following contract must be copied into every successor backlog epic,
including its lander instructions. This repair tale must carry it through to that
successor; finishing this batch alone does not finish the campaign.

1. Enumerate the complete current unresolved set without a limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. A task is
   not resolved merely because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve unique evidence on its canonical surviving bead and name that
   survivor in the close reason.
2. If tasks remain, the current lander (this tale's coder for this closeout) must author
   and submit the next epic through `/sase_plan`. Assign the five most important
   remaining actionable tasks, or fewer if fewer remain. Every backlog epic has at most
   five task beads and five authored phases, explicit dependencies, unique phase slugs,
   and intentional sizes. Tasks may share a phase. Investigation counts toward the cap;
   do not implement extra tasks under landing. Reuse the existing inventory.
3. Rank from current evidence: security/content loss, silent corruption, deterministic
   CI blockers, recurring flakes, performance, then optional additions. Age and
   independent corroboration break ties. Previous candidates included 2l, 94, 95, 4r,
   79, b8 (write/identity defects), b7 (Mac CI), 3w/5l/b9/4k/8q (flakes), and
   b4/ax/8r/9x (Athena usability/detection). These are guidance, not preassignment;
   recheck concurrent work, current evidence, and authorization.
4. Keep decisions minimal and meaningful. Give discretionary features a local
   implement/decline decision; historical suggestions are not feature authorization.
   Before planning memory edits use `/sase_memory_write`, one decision per changed note
   with required selectors and justified requested/corrects quote for a true default.
   New guidance defaults false. Split at five decisions. Honor human declines with
   canceled tasks rather than recreating them; automatic false memory decisions are not
   human declines and must route to an actual reviewer.
5. Preserve future prerequisites. Snoozed 6q, aa, ah are not obsolete because their
   date/evidence/publication gate has not arrived; 3o's recorded one-week gate was met,
   so it is actionable once awake. Verify actual prerequisites, not only wake times.
   Prefer other actionable work. If only gated tasks remain, the next bounded epic owns
   the readiness check and supported `/sase_monitor` or `/sase_questions` handoff
   required to resume. Do not abandon the chain, spin on identical epics, close valid
   gated tasks merely to reach zero, or interfere with another worker's claim.
6. Phase workers record out-of-scope discoveries as PROPOSED FOLLOW-UP on their phase.
   The lander uses `/sase_new_task` to deduplicate and route before creating any task.
   Such tasks enter the future queue and do not enlarge the current assignment.
7. Complete permitted closure and repository obligations before the terminating proposal
   handoff. Validate the successor with `--explain`, correct it, revalidate without
   `--explain`, then `sase plan propose` and wait for the command itself to exit. Record
   successor ownership through SASE links or bead notes. A scratch plan or an intention
   to propose is insufficient.
8. End the chain only when a fresh enumeration of all unresolved statuses returns zero,
   with done/superseded/human-canceled evidence for closures. Until then every successor
   repeats this obligation and reports remaining counts honestly.
