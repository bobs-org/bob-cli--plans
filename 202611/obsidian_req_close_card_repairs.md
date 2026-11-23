---
tier: epic
title: Repair the Obsidian requirement close card and prove it end to end
goal:
  Make explicit requirement verdicts, jump pins, staleness guards, counted and idless
  closes actually work on the Obsidian managed Done card, prove every path through
  public entry points and real Obsidian, and finish the contract docs.
parent_bead: bob-cli-on.11.1
phases:
  - id: verdict_wiring
    title: Fix verdict, landing, staleness, and range wiring with public-path tests
    size: medium
    depends_on: []
    description:
      "verdict_wiring: map picker reqs to resolver selectors, feed the landing resolver
      synchronous final texts with exact line text, guard owner blocks from card open
      through apply, fix counted-range capture, and add the public-path integration
      matrix."
  - id: idless_acceptance
    title: Open the card for idless owners, add real-Obsidian e2e, and finish docs
    size: medium
    depends_on:
      - verdict_wiring
    description:
      "idless_acceptance: route idless owners with Pending reqs through a
      requirement-only card pass, add real-Obsidian close-card e2e scenarios, finish the
      bob-cli and bob-plugins docs, run the combined gates, and deploy both plugins."
proposed_by: bbugyi200.athena.bob-cli-on.11.1.land
proposed_with: claude/opus
create_time: 2026-11-22 20:02:18
status: wip
---

- **PROMPT:**
  [prompts/202611/obsidian_req_close_card_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_req_close_card_repairs.md)
- **PARENT:**
  [202611/obsidian_requirement_close_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/obsidian_requirement_close_card.md)

# Repair the Obsidian requirement close card and prove it end to end

This child epic finishes the remaining work of **bob-cli-on.11.1**, the epic "Obsidian
requirement close card and carry-now"
(`plan:202611/obsidian_requirement_close_card.md`). Its six phases are closed, but the
land agent found that the managed Done wiring from the `close_wiring` phase (bob-plugins
`c91ebae`) does not work end to end. This plan covers only that remaining work. The
parent epic's plan, decisions, and contract stay authoritative. Read the parent plan
with `sase artifact read plan:202611/obsidian_requirement_close_card.md "<why>"` before
you start, and read the parent epic bead's two `LAND` notes
(`sase bead read bob-cli-on.11.1 -r "<why>"`).

The child land agent verifies this epic, closes it, and then resumes the interrupted
landing of `bob-cli-on.11.1` through `parent_bead`. No phase closes `bob-cli-on.11.1`,
`bob-cli-on.11`, or `bob-cli-on`. No phase marks the parent plan done.

## What is already verified (do not redo)

At bob-plugins `55bf2ed` and bob-cli `7f104e52`:

- SP1-SP51 task-split vectors, RV1-RV44 requirement-close vectors, twin-copy parity, and
  the 101 focused req-close/carry tests pass.
- Nav `api.taskSplit` v1 `carryReq`, picker `api.successors` v6, notice v6, cycler
  `planReqCloseWrites` / `applyReqCloseWrites`, `createReqPullInPreflight`,
  `planReqPullInLinks`, and `resolveReqLanding` exist and pass their own tests.
- Both plugins are deployed and synced (nav 2.107.0, cycler 1.58.1).
- There are no `--epic-symbol` entries for `bob-cli-on.11.1` or `bob-cli-on.11`.
- Commits since the epic began (Pomodoro mention history `55bf2ed`/`7f104e52`, Mac card
  docs `9d0cbfbe`, ledger-tools property cards, props, gmail, tick, highlights) neither
  conflict with nor duplicate this work.

## Defects to fix (reproduced by the land agent)

All paths are in bob-plugins `plugins/task-status-cycler/src/` unless named otherwise.

1. **Every explicit verdict is refused.** `applyManagedReqCloseCard`
   (`198g-req-close-integration.js`) passes the picker's
   `reqs: [{key, verdict, reason?}]` straight through as `requests`.
   `reqCloseResolveVerdicts` (`198b-req-verdicts-twin.js`) reads `request.selector`, so
   every Drop, Carry, and Carry+pull-in fails `req_close_stale` with
   `selectors: ["undefined"]`. Only implicit Met (`reqs: []`) works. The req-apply tests
   passed only because they build `{selector}` themselves.
2. **Every requirement jump pin is lost.** `resolveManagedReqCloseLanding` passes an
   `async` reader to the synchronous `resolveReqLanding` (`198f-req-close-landing.js`).
   `reqLandingLocateBlock` and `searchOwner` see a Promise, not a string, and return
   `req_jump_lost`. Separately, the returned target is not landable by nav `landOnTask`
   (`bob-navigation-hotkeys/src/676a-plugin-task-landing.js`, which matches by block ID,
   then exact `text` at `line`, then unique exact text, then the task-only description
   fallback). `moved_with_ancestor` returns a 1-based `line` with no `text`, so it falls
   to the task-only description fallback that the parent plan forbids for requirement
   landings. Only `requirement` converts to 0-based with text.
3. **Staleness guards always pass.** `applyManagedReqCloseCard` re-reads live notes,
   then builds both `expectedNotes` and `expectedOwnerPostimages` from those same live
   texts. External edits made inside the owner block while the card was open are
   therefore accepted, whether a new req, a sibling, title, or note edit, or a re-indent
   elsewhere in the block. Only the per-row structural lookup catches changes to the
   chosen row. The parent plan requires: store the exact post-close expected bytes at
   card open, compose the gesture's own later writes explicitly, and refuse
   `req_close_stale` on any external edit in a guarded owner block.
4. **Counted ranges capture no owners.** `cycleTaskStatusRange`
   (`150-plugin-commands.js`) calls `captureReqCloseOwnersFromText(text, path)` with no
   identities. In that branch `selected` is a flat task list, but the loop expects
   arrays of matches, so every task is skipped (0 owners).
5. **Idless owners never open the card.** `closedTaskIdentity` (`080-dependencies.js`)
   returns `null` without a block ID or `[id::]`. `normalizeClosedTaskIdentities` and
   `finalizeClosedTasks` (`130-plugin-references.js`) then drop or short-circuit, and
   local close callers skip finalize when no identity exists. bob-cli §12.25 makes an
   idless owner eligible, with its canonical identity from the existing deterministic
   line/digest convention (`reqCloseStableTaskKey`). The parent `req_model` phase
   collects unanchored owners.
6. **Coverage gaps.** `scripts/test-task-status-cycler-req-close-integration.cjs` holds
   six unit-level tests. None drives a public managed Done entry point with an explicit
   verdict, a pin, staleness, Undo, or exclusions. The real-Obsidian harness
   `scripts/obsidian-task-reqs-e2e.cjs` has no close-card scenario.
7. **Docs contradict the code.** bob-cli `docs/task-requirements.md` §5 says
   "Ctrl+Shift+T on a requirement line keeps its existing task-split behavior". Nav
   actually carries through `api.taskSplit.carryReq` (`653-plugin-task-split.js`,
   `splitRequirementCarry`), as `docs/bob-navigation-hotkeys.md` states. §7 credits only
   the `close-engine` and Mac shipping. It does not describe the Obsidian card's
   receipts, one guarded Undo, Esc, or same-day reopen. bob-plugins
   `docs/task-status-cycler.md` has no close-card section.

## Phase verdict_wiring

Work in bob-plugins task-status-cycler (and nav only if a landing adapter needs it).
Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Edit `src/` fragments and
`fragments.json`, never generated `main.js`. Keep every hand-edited fragment at or below
1000 lines: `160-plugin-completion.js` is already at 1000, and `137`, `137a`, and `150`
are 726-934. Add new fragments, for example `198h-...`, instead of growing near-cap
files, and register them in `fragments.json` after the code they use.

- **Selector mapping (defect 1).** Normalize card results once, at the cycler boundary:
  `{key, verdict, reason}` becomes `{selector: key, verdict, reason}` before
  `planReqCloseWrites` / `applyReqCloseWrites`. Keep explicit `met` distinct from an
  absent row. Pass the jump pin as its key. Do not change the resolver or RV vectors.
- **Landing (defect 2).** Before resolving, read fresh final texts for every note the
  resolver may need: owner paths, each outcome's `carried.path`, route destinations, and
  the daily path. Invalidate the successor cache first. Then call `resolveReqLanding`
  with a synchronous `(path) => text | null` over that map. Hand nav `landOnTask` a
  target it lands exactly: a 0-based `line` plus the exact final line `text` for all
  three kinds, and `blockId` for `carried_task`. Use the requirement's text as
  `label`/`description`. Never rely on the task-only description fallback. A
  `req_jump_lost` / `req_jump_ambiguous` refusal adds a receipt failure line and lands
  nothing. Keep the `successorJumpTarget: "return"` contract
  (`decisions:successor-jump-parks-the-walk`): a pinned walk close returns the one jump
  to its walk owner, with no intermediate advance or second landing, and Ctrl+O history
  is recorded once by the lander.
- **Staleness (defect 3).** At card open in `planAndApplySuccessorsNow`
  (`137-plugin-successor-project.js`), snapshot each attached owner's exact post-close
  owner block from the post-close note texts the card was planned from. Close
  bookkeeping already on disk at that point is part of the expected bytes. When the
  picker settles, compare it with the revalidated post-picker text from
  `revalidateSuccessorPickerChoices`, before any of this gesture's writes. On any
  difference in a guarded owner block, refuse that owner's requirement choices with
  `req_close_stale`: no requirement bytes, a receipt failure line, and the owner stays
  Done. Ordinary offers keep their own revalidation and still apply. Then follow every
  write this gesture makes before the requirement apply (successor note edits, daily
  insertions, offer-close rewrites). Use each write's actual postimage per path as the
  composed expected text, and pass it as `expectedNotes` / `expectedOwnerPostimages`. A
  live text that differs from the composed expectation refuses `req_close_stale`. Never
  derive both sides of a guard from one read.
- **Counted ranges (defect 4).** Capture owners from the pre-loop editor text only for
  anchored tasks whose line lies in the cycled range. Fix the identity-less branch of
  `captureReqCloseOwnersFromText` so each matching task forms its own single-match
  group, or remove that branch and pass explicit identities.
- **Public-path integration matrix (part of defect 6).** Drive real entry points
  (`finalizeClosedTasks` / `planAndApplySuccessorsNow`, Ctrl+Enter
  `toggleActiveCheckboxOpenDoneAndPropagate`, `completeTaskAtCursor`, a single cycle,
  and `cycleTaskStatusRange`). Use the existing fake-vault and editor harnesses
  (`task-status-cycler-harness.cjs`, the successor test files). Use a stub v6 picker
  that returns scripted results, and the **real** nav carry API from
  `navigation-hotkeys-harness.cjs` (as `test-task-status-cycler-req-close-apply.cjs`
  does). Assert exact bytes, receipts, and restores. Cover:
  1. Ctrl+Enter, then an explicit Drop with a reason: drop bytes plus reason log,
     `⊘ Dropped` receipt, owner stays `[x]`, no `[fresh::]` / `[unblocked::]`.
  2. Carry and Carry+pull-in for a daily-note owner (cross-note): the created task
     routes per Task Split, the `[>]` stub, the back-link, and the pull-in link at the
     existing successor destination raised to Next without freshness.
  3. Several carries plus a drop on one owner in one file, and two owners in one file.
  4. Jump pins: a selected carry lands on the created task, a descendant under an
     ancestor carry lands on its moved line (exact text), and a Met or dropped row lands
     on its final line. Equal labels never substitute.
  5. A req-only card on a task with no `[id::]` and no ordinary offers: the card opens,
     and the receipt has no `unblocked` or `offer_closes` rows.
  6. Mixed ordinary offers and reqs: requirement groups come first. An ordinary pick and
     a req carry both apply, and an ordinary write in the owner's file never targets a
     pre-carry line offset.
  7. Embedded multi-owner close (local plus transclusion): one card, groups in closing
     order.
  8. External edits while the card is open (new req, sibling or title edit, re-indent,
     edited duplicate label): `req_close_stale`, no requirement bytes, ordinary offers
     unaffected.
  9. Older picker (v5) or missing nav: no groups, the close succeeds, reqs implicitly
     Met, no requirement bytes.
  10. Exclusions: a recurring owner, a cancel, a parked event, an automated close, and
      tasks closed by the card's own offer-close verdicts never open a requirement
      audit.
  11. Esc (picker `null`): no requirement bytes, the owner stays Done, and ordinary
      reopen still works.
  12. One guarded Undo: the merged restores revert requirement and ordinary writes, the
      owner stays Done, and a late user edit to any recorded postimage refuses the whole
      Undo before any write.
  13. A counted-range close of an anchored owner opens the card.
  14. A walk-owned close (`successorJumpTarget: "return"`) returns exactly one
      requirement jump.
- Run focused tests, `npm run build`, `npm run build:check`, and `npm test` (add new
  test files to the explicit `test:unit` list). Known 25 ms timing flakes under full
  load are tracked as `bob-cli-km` (event-link-times) and `bob-cli-io` (frees-marks).
  Confirm an isolated pass and cite the ID; do not re-file them. Deploy the changed
  plugin IDs only: `bob plugins sync -p task-status-cycler`, plus
  `-p bob-navigation-hotkeys` if nav changed. Never pass `-F`; follow the repo
  `AGENTS.md` reconciliation steps if sync refuses.
- In the completion note, publish the exact seam that `idless_acceptance` builds on:
  where the owner snapshot, expected texts, and the request mapping live.

## Phase idless_acceptance

Work in bob-plugins (cycler, and nav only if needed), the real-Obsidian harness, and
bob-cli docs. Follow the same fragment, cap, and deploy rules.

- **Idless owners (defect 5).** A managed Done of an idless owner with at least one
  Pending req opens the same requirement-first card. Cover the local Ctrl+Enter toggle,
  `completeTaskAtCursor`, single and counted cycles into Done, and nav callers that pass
  `context.reqCloseOwners`. Capture the owner from the pre-close editor text. Give it a
  close identity from its path, its pre-close 1-based line, and its canonical `task_key`
  (`reqCloseStableTaskKey`; never mint an ID). Order it with the anchored owners in
  actual closing order. When no anchored identity was closed, run a requirement-only
  pass through `planAndApplySuccessorsNow`, the way `offerParkedSuccessors` calls it
  with `[]`. Do not run dependency recovery, retirement, or successor-link planning for
  the idless owner. Req-less idless closes keep today's no-extra-scan behavior. The
  staleness, landing, and Undo rules from `verdict_wiring` apply unchanged. Add an
  idless owner to an RV-style injected test, plus public-path tests: drop, carry, a pin,
  Esc, a recurring idless owner excluded, and a mixed anchored/idless closing order.
- **Real-Obsidian e2e (defect 6).** `scripts/obsidian-task-reqs-e2e.cjs` is 973 lines,
  so add a sibling scenario file, for example `scripts/obsidian-req-close-card-e2e.cjs`,
  that reuses its setup and fixture vault. Add a `scripts/test-...-e2e.cjs` wrapper
  registered like `test-obsidian-task-reqs-e2e.cjs`. Run real keys in the isolated
  fixture vault:
  - Ctrl+Enter on an owner with nested reqs opens the card with requirements first.
  - Ctrl+Shift+X with a reason, then Enter, writes drop bytes and the receipt.
  - Notice Undo restores the reqs and keeps the owner Done.
  - Ctrl+Shift+T carry, then Enter, creates the task and the `[>]` stub.
  - A Ctrl+J pin on a nested req lands on its final line, and Ctrl+O returns.
  - Esc writes nothing.
  - Ctrl+Shift+T on a requirement line (outside the card) carries now and lands on the
    new task. Record any environment restriction honestly in the phase note. Never claim
    live-vault testing that was not performed.
- **Docs (defect 7).** Read bob-cli `docs.md` memory first
  (`sase memory read docs.md -r "<why>"`).
  - In bob-cli `docs/task-requirements.md` §5, replace the Ctrl+Shift+T sentence:
    Obsidian Ctrl+Shift+T on a req carries it now through nav `api.taskSplit`, with §6
    bytes, landing on the new task with Ctrl+O back, and the close card offers the same
    carry after a managed Done.
  - In §7, add the Obsidian card: shipped in `obsidian-close-card`, gated on picker v6.
    List the managed Done paths it covers, including idless owners. Describe receipts,
    one guarded Undo that keeps the owner Done and refuses after any later edit, and Esc
    writing nothing. State exactly what a same-day Ctrl+Enter reopen does to card
    verdicts, as observed in the code. Note that every exported line is 1-based in the
    API and 0-based only at the editor boundary.
  - Check `docs/task-dependencies.md` §12.13 and §12.25 for any remaining "later phase"
    wording.
  - In bob-plugins `docs/task-status-cycler.md`, add a "Requirement close card" section:
    eligible paths, exclusions, ordering (carries, drops, close bookkeeping, pull-in
    links, final jump), staleness, Undo, Esc, and older-picker fallback.
  - Update `docs/develop/plugin-apis.md` and `docs/bob-navigation-hotkeys.md` only where
    behavior or API shape changed. Bump cycler `api.version` only if a method was added.
  - Run `npm run docs:gen` (if inventory changed) and `npm run docs:check` in
    bob-plugins, and `just docs-gen`, `just docs-check`, and `just check` in bob-cli.
- **Final gates and deploy.** Run `npm run build`, `npm run build:check`, `npm test`,
  and the new e2e on the combined tree. Treat `bob-cli-km` / `bob-cli-io` the same way
  as in `verdict_wiring`. Deploy both changed IDs with `bob plugins sync -p ...` (no
  `-F`) and confirm `bob plugins list` shows both synced. Leave evidence for the land
  agent: tests run, e2e outcome or restriction, deployed versions.

## Rules for every phase

- Do not edit Cargo.toml version or CHANGELOG.md. No new CLI flags. Contract breaks
  follow the release-train `!` convention.
- This epic edits no SASE memory.
- Phase workers record unrelated discoveries only as
  `PROPOSED FOLLOW-UP: <summary — evidence>` notes on their own phase bead, never as new
  task beads.
- Each phase runs `sase bead epic-symbols <its-id>` before closing and resolves or
  re-keys leftovers. Each phase closes only its own phase bead.

## Child land agent

Verify both phases against the defects above by rerunning the reproductions. An explicit
Drop must apply, a pin must land, and an external owner-block edit must refuse. Confirm
the public-path matrix, the e2e result, the docs, and both deploys. Close this child
epic. Its `parent_bead` handoff then resumes the `bob-cli-on.11.1` landing: re-run
`sase bead epic-symbols bob-cli-on.11.1`, close `bob-cli-on.11.1` with a verification
note, run `just symvision`, and set `status: done` in
`plan:202611/obsidian_requirement_close_card.md`. Then close the parent phase
`bob-cli-on.11` (a phase of epic `bob-cli-on`) with a note naming verification and
deploy evidence. Leave `bob-cli-on` itself to its own land agent.
