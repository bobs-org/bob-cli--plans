---
tier: epic
title:
  "Backlog batch 4: green bob-cli CI, capture flag guard, bob-plugins red tests, and
  relation-label race"
goal:
  Close five prioritized task beads. Restore bob-cli master CI so the release train can
  ship and regressions stop hiding, stop bob capture from silently filing unknown flags
  as task text, fix bob-plugins' two deterministic test failures including the dangling
  BLOCKS link they expose, and stop Bob Mac Capture committing a stale relation label.
  The lander continues the backlog through further epics of at most five phases and five
  task beads until every project task bead is closed.
decisions:
  capture_flag_guard:
    ask:
      Should bob capture reject unknown alphabetic long options before TEXT, in addition
      to Bob Mac Capture's capability gate?
    choices:
      refuse:
        Unknown --flags exit 2 and write nothing; text starting with --word must follow
        --
      client_only:
        Only the app gate ships; bob still files an unknown --flag and draft as a task
    default: refuse
    why:
      Fixes the root cause for every future internal capture flag, not just
      --require-pull-in-choice
    answer: refuse
phases:
  - id: ci-scripts-identity
    title: Give the coordination acceptance self-test its own git identity
    size: small
    depends_on: []
    description:
      "ci-scripts-identity: resolve bob-cli-b5 by persisting the scratch vault's git
      identity so just check-scripts passes on an identity-less CI runner."
  - id: capture-flag-guard
    title: Stop bob capture filing unknown flags as task text
    size: medium
    depends_on: []
    description:
      "capture-flag-guard: resolve bob-cli-8a by applying the capture_flag_guard
      decision in bob capture and gating Bob Mac Capture's --require-pull-in-choice on a
      positive capability."
  - id: plugins-red-tests
    title:
      Fix bob-plugins' frozen-clock glyph tests and the reverse-conversion BLOCKS link
    size: medium
    depends_on: []
    description:
      "plugins-red-tests: resolve bob-cli-6v and bob-cli-7i by deriving the ledger-tools
      test daily path from the frozen clock and re-keying the prerequisite's BLOCKS
      entry when a project converts back to a task."
  - id: relation-label-race
    title: Commit only the relation label the user authored
    size: medium
    depends_on: []
    description:
      "relation-label-race: resolve bob-cli-al by tying Bob Mac Capture relation label
      suggestions to the request and authored label that produced them."
proposed_by: bbugyi200.athena.bob-cli-an.6.6.land
proposed_with: claude/opus
parent_bead: bob-cli-an.6.6
decided_by: auto
create_time: 2026-10-24 05:39:20
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch4_ci_identity_capture_flags_plugin_tests_relation_race.md)
- **PARENT:**
  [202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md)

# Backlog batch 4: green bob-cli CI, capture flag guard, bob-plugins red tests, and relation-label race

## Scope and triage

This continues the backlog campaign of bounded epics. Batch 1 (`bob-cli-an`), batch 2
(`bob-cli-an.6`), and batch 3 (`bob-cli-an.6.6`, plan
`202610/backlog_batch3_rfserver_backups_day_zone_webclip_fonts_ref_paths.md`) are
closed. Batch 3's lander closed `bob-cli-b3`, `b2`, `2q`, `85`, and `9a` with live
evidence. It closed `bob-cli-b6` as resolved by a08581d8 and fixed four epic-caused
gaps:

- 339578cb: ref and Keep date stamps now use the configured zone, and web-clip font
  scratch no longer leaks into `/tmp`.
- 05443159: a `DATE=date` clock command now runs in the configured timezone. The
  dotfiles export it, so apollo still used UTC.
- c7e0dab6: relative ref PDFs now map correctly through a symlinked or macOS
  `/private/var` cwd.

The lander also filed flake `bob-cli-b9`. That last fix was only found because master CI
had been red since 134016ce, which masked a macOS-only regression for five commits. This
epic owns exactly these five task beads; no phase may add a sixth:

| Rank | Task         | Reason for priority                                                                                                                                                                                                                              | Phase                 |
| ---- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| 1    | `bob-cli-8a` | Silent vault corruption: an unknown `--flag` before `--` becomes task text, so an older bob files a junk inbox task holding the flag and the entire Mac Capture draft and reports success. Every future internal capture flag shares the hazard. | `capture-flag-guard`  |
| 2    | `bob-cli-al` | Silent corruption: pressing Return before a pending suggestion fetch completes commits the previous relation label instead of the one just authored. Late responses can also overwrite current validation.                                       | `relation-label-race` |
| 3    | `bob-cli-7i` | A dangling write plus a deterministic test failure: reversing a project conversion leaves the prerequisite's BLOCKS entry pointing at the trashed project's `^prj`. 19 independent reports.                                                      | `plugins-red-tests`   |
| 4    | `bob-cli-b5` | bob-cli master CI lint has been red on every commit since 134016ce, so the daily release train cannot ship and new CI regressions stay hidden (batch 3's macOS regression was one).                                                              | `ci-scripts-identity` |
| 5    | `bob-cli-6v` | A deterministic bob-plugins test failure since 2026-10-09: four unblocked-glyph tests build the daily path from the wall clock while fixtures freeze the date. 24 independent reports, the most corroborated task in the backlog.                | `plugins-red-tests`   |

Ranking: no open security or content-loss task is actionable now. Silent corruption
comes first (`8a`, `al`), then `7i`, which is both a dangling write and a deterministic
test failure. The CI blockers follow: `b5` blocks releases and hides regressions, and
`6v` leads on corroboration and age. These were deferred with reasons, not silently
absorbed:

- `bob-cli-b7` (bob-mac-capture RefsLibraryTests, deterministic) ties with `6v` as a CI
  blocker. `6v` wins on corroboration (+24 vs none) and age. Phases that touch
  bob-mac-capture treat those three tests as known failures, not regressions.
- `bob-cli-79` fails loudly and keeps the source task.
- `bob-cli-b8` repairs live vault identities and needs its own reviewed mutation.
- `bob-cli-4r` affects two notes.
- `bob-cli-2l`, `bob-cli-94`, and `bob-cli-95` are next in line for write defects.
- `bob-cli-7q` remains a design change to accepted decision
  `decisions:capture-link-drops-blocking-deps` and needs its own reviewed decision.

On 2026-10-24, after the batch-3 landing, a fresh enumeration found **133 unresolved
tasks: 129 ready and four snoozed** (`bob-cli-3o`, `bob-cli-6q`, `bob-cli-aa`,
`bob-cli-ah`). None were claimed or in progress. This is a snapshot, not a substitute
for the lander's fresh enumeration. Open epic plans at this snapshot were `bob-cli-28`,
`bob-cli-3j`, `bob-cli-5w`, `bob-cli-66`, `bob-cli-87`, `bob-cli-8b`, `bob-cli-9k`, and
`bob-cli-9k.8`. None owns this batch's tasks, and `bob-cli-9k.8`'s plan explicitly
leaves `6v` and `7i` to their own beads. Recheck ownership before implementation.

## Working rules and handoffs

Read the assigned task, this epic's finalized decisions, and phase dependencies through
`sase bead read`. Reproduce against the current tree before changing anything. A task
that has meanwhile been fixed can be closed with independently checked evidence instead
of reimplemented. Record baseline, chosen fix, validation, and remaining constraints on
the task and phase. Use `sase bead close` with `done`, `superseded`, or a
human-authorized `canceled` resolution; never hand-edit the store or runtime-owned
statuses. Phase workers leave task beads open for the lander; the lander closes them
after rechecking evidence. Phase workers never create beads: record out-of-scope
discoveries as `PROPOSED FOLLOW-UP:` notes on your own phase.

Use `/sase_repo` before accessing any other repository (`bob-plugins`,
`bob-mac-capture`). Read each opened repo's `AGENTS.md` and follow its build and test
commands. Read sidecar evidence and attached artifacts only through
`sase artifact read`. bob-mac-capture builds and tests on the Mac (`ssh mac`, see the
`tailnet` reference memory); if the Mac is unreachable, use `/sase_monitor` to wait and
retry, and do not claim a Swift change verified without a Mac build and test run. Bob
Mac Capture is a thin client of bob (`decisions:mac-capture-is-a-thin-client`), so
coordinate capture-contract changes across bob-cli and bob-mac-capture.

bob-cli phases run the focused tests for the code they touch, then `just check` once on
their integrated change (use `/sase_monitor` if it runs long), and `just docs-check`
when they change docs. Follow the daily release train: do not edit package versions or
`CHANGELOG.md`, and do not publish a manual release. bob-plugins phases run the plugin
test suite their `AGENTS.md` names, and deploy only through `bob plugins sync`. Never
mutate the live vault during validation except through a step the phase explicitly
allows.

The four phases are independent and can run in parallel. `capture-flag-guard` and
`relation-label-race` both commit to bob-mac-capture, in different files
(`Sources/CaptureCore/BobProcessClient.swift` versus
`Sources/BobMacCapture/CapturePanelModel.swift`). Each rebases through
`sase stitch create`; resolve doc-index conflicts by keeping both sides.

## Phase ci-scripts-identity — bob-cli-b5

Evidence (2026-10-24, CI run 40081848753 on c7e0dab6): the `lint` job fails at
`just check-scripts` (justfile line 44) with
`bob ref: pre-sync failed: git commit failed: Author identity unknown`. The ubuntu,
macOS, and package jobs pass. `scripts/dev/highlights_coordination_acceptance` passes
identity to its own git calls with `-c user.email=... -c user.name=...` (around lines
135-153). But bob's internal vault-sync pre-sync commit inside the scratch vault reads
only repository, global, or system config. The runner has none, and athena's GECOS name
hides the failure locally.

1. Reproduce locally with identity inference disabled. For example, run
   `GIT_CONFIG_GLOBAL=/dev/null GIT_CONFIG_NOSYSTEM=1 GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=user.useConfigOnly GIT_CONFIG_VALUE_0=true bash scripts/dev/highlights_coordination_acceptance --self-test`.
   It should fail with the same `Author identity unknown`.
2. After the scratch vault's `git init`, persist the scratch identity in that repository
   (`git -C "$vault" config user.name "Acceptance Kit"` and `user.email`). Do the same
   for any other scratch repository that bob itself commits in. Keep the explicit `-c`
   identities or drop them once repository config covers them, whichever is clearer.
3. Rerun the reproduction command, then `just check-scripts` and `just check`. Record
   any other `lint` job failure you can reproduce locally. Do not fix bob-cli-v's
   preexisting clippy warnings here. They are warnings, not the failing node.

Acceptance: the self-test passes with no ambient git identity. `just check-scripts` and
`just check` pass. The lander confirms the GitHub Actions `lint` job is green on the
landed SHA.

## Phase capture-flag-guard — bob-cli-8a

Evidence (bob-cli-8a): `bob capture --format json --require-pull-in-choice-x -- hi`
exits 0 and files `- [ ] #task --require-pull-in-choice-x -- hi`, because
`src/native/capture/cli.rs` `text_arg()` is `trailing_var_arg(true)` plus
`allow_hyphen_values(true)`. Bob Mac Capture's
`Sources/CaptureCore/BobProcessClient.swift` sends `--require-pull-in-choice` and relies
on `isUnsupportedGuardFlagFailure`, which expects exit 2 plus "unexpected argument".
That fallback can never fire for `bob capture`. bob-cli-8b already solved the same
hazard for `--template-origin` with a positive capability (a successful
`capture-templates list` from the same executable identity proves support).

1. Reproduce both halves against a throwaway vault (`BOB_DIR` and `XDG_STATE_HOME` in a
   temp dir) and with a fake older bob in the Mac client tests.

> [!decision] capture_flag_guard = refuse In bob, reject an unknown alphabetic long
> option (`--` followed by a letter) that appears where TEXT would begin without a
> preceding `--`, with exit 2 and clap's unexpected-argument wording; honor
> `--format json` for the error envelope. Keep working unchanged: every declared flag,
> Pomodoro shift tokens such as `--2` and `--1`, a bare `--`, and everything after `--`
> (literal text, so `bob capture -- --anything` still files `--anything`). Apply the
> same rule to every command that reuses `text_arg()` only where it is a capture-style
> TEXT positional; list each user you checked. Update the capture docs and help text
> that describe hyphenated text, and run `just docs-check`. Also ship the Mac-side gate
> described below.

> [!decision] capture_flag_guard = client_only Leave bob's TEXT parsing unchanged and
> ship only the Mac-side gate below. Record on the phase that bob still files unknown
> flags as text.

2. In Bob Mac Capture, send `--require-pull-in-choice` only after a positive capability
   from the same executable identity proves support. Follow bob-cli-8b's
   `--template-origin` pattern, for example `pull_in.supported` in a preview from that
   executable. Delete the dead `isUnsupportedGuardFlagFailure` fallback for capture, or
   keep it only where it can actually fire. Add client tests: an older bob without the
   capability gets no guard flag and no junk task, and a current bob gets the flag.
3. Run bob-cli focused capture tests and `just check`, plus the bob-mac-capture build
   and test commands from its `AGENTS.md` on the Mac. The three `RefsLibraryTests` scan
   failures tracked by `bob-cli-b7` are known and are not regressions.

Acceptance: no supported bob and Bob Mac Capture combination can file an unknown
internal flag plus the draft as a task. Regressions cover both repositories, and under
`refuse` an unknown alphabetic long option before TEXT exits 2 without writing.

## Phase plugins-red-tests — bob-cli-6v and bob-cli-7i

Evidence: in bob-plugins, `scripts/test-ledger-tools-unblocked-glyph.cjs` builds
`const DAILY = todayDailyPath(new Date(), {})` from the wall clock (around line 179).
Every fixture freezes `TODAY = new Date(2026, 9, 9, 12)` (around line 188), so four
tests have found 0 links instead of 1 since 2026-10-09 (`bob-cli-6v`, +24). In
`scripts/test-navigation-hotkeys-forward-promotion.cjs`,
`forward conversion carries a Depends-On line and reverse restores it` fails at
`assert.doesNotMatch(restored, /Work_ship#\^prj/)` (around line 734). After a forward
then reverse conversion, the prerequisite keeps `🧱 **BLOCKS:** [[Work_ship#^prj]]`
instead of `[[#^ship]]`. That failure bisects to bob-plugins 8406ebb (bob-cli-7e.8,
which routed project conversion through the shared BLOCKS seam): the forward path
re-keys the entry, and the reverse path does not (`bob-cli-7i`, +19).

1. Reproduce both with the focused `node --test` commands on current bob-plugins master,
   then run the full plugin test suite and record every other failure with its existing
   task owner.
2. `6v`: derive `DAILY` from the frozen `TODAY`, or inject the clock, so the tests pass
   on any date. Change only the test unless a product clock seam is genuinely needed.
3. `7i`: make the reverse conversion re-key the prerequisite's BLOCKS entry from the
   trashed project's `^prj` to the restored task's block id. The prerequisite and the
   restored task share a note, so the re-key belongs in the parent write, per the
   writers section of `docs/task-dependencies.md` (§14). Keep the test's expectation,
   which already states the intended contract. Respect
   `decisions:task-deps-are-depends-on-links`: the Depends-On line is the source of
   truth, and BLOCKS is its mirror. Add a regression for a cross-note prerequisite if
   the reverse path handles that case differently.
4. Rerun the focused tests and the full suite, then deploy with `bob plugins sync` (the
   one vault mutation this phase allows) and record the synced plugin versions.

Acceptance: both focused tests pass on any wall-clock date. The full plugin suite has no
failures beyond recorded preexisting owners. A forward-then-reverse conversion leaves no
BLOCKS link to the trashed project, and the deployed plugins match the commit.

## Phase relation-label-race — bob-cli-al

Evidence (bob-cli-al, artifact `file:explicit:dd078a92f55551184b77dfc7`, a Swift
extracted-method harness): in bob-mac-capture `CapturePanelModel.swift`,
`updateRelationLabelPromptLabel` keeps old suggestions while it starts an asynchronous
fetch. `submitRelationLabelPrompt` then selects the first typed candidate without
checking that it matches `authoredLabel`. The response handler checks only
`draftSnapshot`, not the authored label, request generation, or prompt identity. Typing
`old`, changing it to `new`, and pressing Return before the new response arrives commits
`|old`. Git blame traces this to a85b1f86 (closed `bob-cli-a2`).

1. Read the harness with `sase artifact read` and reproduce: `authored=new`, then
   `committed=|old`.
2. Tie each suggestion fetch to a request generation, plus the prompt and authored label
   it was issued for. Drop responses that no longer match. Invalidate or visibly gate
   stale candidates as soon as the label changes. On submit, commit only a candidate
   produced for the current authored label. Otherwise commit bob's authored replacement
   for the current input, because bob owns relation grammar. Never commit a stale
   candidate.
3. Add regressions for a delayed response, an out-of-order older response arriving after
   a newer one, and an immediate Return before any response. Build and run the
   bob-mac-capture tests on the Mac per its `AGENTS.md`. The three `bob-cli-b7`
   `RefsLibraryTests` failures are known.

Acceptance: the harness scenario commits `|new`, never `|old`. Stale responses cannot
change the suggestions or validation of the current prompt, and Mac tests cover delayed,
out-of-order, and immediate-Return cases.

## Lander: verification, closure, and mandatory next epic

The lander owns integrated verification and the continuation, not a sixth implementation
task or a fifth authored phase. Read every phase's notes, acceptance evidence, and
`PROPOSED FOLLOW-UP` entries. Recheck the current tree and CI state, including whether
another epic changed capture flags, relation labels, project conversion, or the
acceptance kit. Run appropriate integrated checks once and document genuine preexisting
failures with their existing task owners. Require real evidence before closing each
task:

- `b5`: a green GitHub Actions `lint` job on the landed bob-cli SHA. Use `/sase_monitor`
  for the wait.
- `8a`: two-repo regressions, plus a throwaway-vault reproduction that no longer files a
  junk task.
- `6v` and `7i`: focused and full bob-plugins test runs, plus the `bob plugins sync`
  record.
- `al`: the harness scenario and the Mac test run.

Check bob-mac-capture CI on its landed SHAs, and confirm no failures beyond the known
`bob-cli-b7` set.

Because `sase plan propose` terminates the runner without running the commit finalizer,
the lander must commit every repository it changed (bob-cli, bob-plugins,
bob-mac-capture, the plans sidecar status edit, and any other opened repo) with
`/sase_git_commit`, which this approved plan explicitly authorizes for that purpose,
before proposing the successor.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include `bob-cli-2l`, `bob-cli-94`, `bob-cli-95`, `bob-cli-4r`,
   `bob-cli-79`, and `bob-cli-b8` for identity/write defects; deterministic CI blocker
   `bob-cli-b7` (bob-mac-capture RefsLibraryTests) unless already fixed through its
   triage gate; recurring flakes `bob-cli-3w`, `bob-cli-5l`, `bob-cli-b9`, `bob-cli-4k`,
   and `bob-cli-8q`; `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x` cover
   Athena usability and detection. This is guidance, not a preassigned batch, and
   concurrent work may obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes. Verify real prerequisites, not just wake timestamps. Work
   other actionable tasks first. If only gated tasks remain, the next bounded epic must
   own the readiness check and supported `/sase_monitor` or `/sase_questions` handoff
   needed to resume; do not abandon the chain, spin on fresh identical epics, or close a
   valid task merely to reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.
