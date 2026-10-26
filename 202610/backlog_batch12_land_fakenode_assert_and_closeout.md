---
tier: tale
size: medium
title: "Finish backlog batch 12: real FakeNode assert fix, epic closeout, and batch 13"
goal:
  Make a failing assert.equal / deepEqual on two live ledger-tools FakeNodes report
  instead of exhausting the heap, close bob-cli-c7 and epic
  bob-cli-an.6.6.6.5.6.5.5.6.6.6.6, and propose the batch-13 successor epic.
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.land
proposed_with: claude/opus
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6
status: done
---

- **PARENT:**
  [202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.land.md)
- **COMMITS:**
  - [74eb763](https://github.com/bobs-org/bob-cli--plans/commit/74eb763d63c8736e36b40d1cdf136d737bac7ab6)
    — docs(plan): mark batch 12 done and author batch 13

# Finish backlog batch 12: real FakeNode assert fix, epic closeout, and batch 13

You are finishing the landing of epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6` ("Backlog batch
12"). Its lander verified everything except the `fakenode-inspect` phase and handed the
remaining work to you. Nothing resumes the landing after you, so this tale ends with the
epic closeout and the successor-epic proposal. Read these first:

```bash
sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6.6 -r "Need the epic scope, lander notes, and plan path"
sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.3 -r "Need the fakenode-inspect phase notes"
sase bead read bob-cli-c7 -r "Need the FakeNode OOM task report and the lander probe note"
```

All work in step 1 happens in the `bob-plugins` linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`), use only the path it prints, and read its
`AGENTS.md` first.

## Already done by the lander (do not redo)

- Verified and closed `bob-cli-ca` (bob-cli `7b8c5ddb`), `bob-cli-8z` (bob-plugins
  `d9823d1`), `bob-cli-c6` (bob-plugins `8866540`, bob-cli docs `3633a10a`), and
  `bob-cli-c8` (bob-plugins `41f2093` by concurrent agent 14i plus phase commit
  `abb6ea6`). The vault's bob-navigation-hotkeys 2.56.4 and bob-ledger-tools 1.73.0 are
  byte-identical to bob-plugins master. `just check` on bob-cli master `01f7ec97` passed
  at load average 21–25, and both `capture_related` perf gates passed.
- Integration review of everything that landed after the epic started is finished. Only
  14i's Day strip legend work touched this epic's area. It is the `c8` fix, and
  `docs/stats.md` matches it.
- Follow-up triage is finished and recorded in the epic's lander note. The only
  `PROPOSED FOLLOW-UP` (phase `.3` note #1: future Day strip DOM asserts must use the
  `===` + `nodeNote` form) was declined as a new task. Step 1 below removes that trap at
  its root.
- `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6` reported no entries.
- Backlog at the lander's audit: 117 unresolved tasks, 113 ready and four snoozed (`3o`,
  `6q`, `aa`, `ah`). Closing `c7` makes it 116.

## Step 1: Make failing node asserts report (finish `bob-cli-c7`)

### Why phase `.3` did not resolve it

Phase `.3` (bob-plugins `0ccb3ac`) gave every cyclic ledger-tools fake a short
`util.inspect.custom` descriptor. It kept the plan-host identity checks in the
`assert.ok(a === b, nodeNote(...))` form. Its note claims `assert.equal` on two live
nodes cannot be made to report. That is only true for descriptors. Node v22's
`AssertionError` builds its diff with `customInspect: false`, so it ignores the
descriptor. The new regression only exercises the `===` form, which never diffs. As a
result, `assert.equal` / `assert.deepEqual` on two live FakeNodes still exhausts the
heap.

The lander measured the real cause with a scratch copy of the verbatim plan-host
`FakeNode` class (from `scripts/test-ledger-tools-day-strip-plan-host.cjs`) on an
85-node tree: a root with two depth-3, fan-out-4 subtrees, `alpha` and `beta`. Assert
renders values with these options:
`{compact: false, customInspect: false, depth: 1000, maxArrayLength: Infinity, showHidden: false, sorted: true, getters: true}`.

- Rendered with those options, one 42-node subtree is 14,771 lines (417 KB). The
  `inserted` and `removed` bookkeeping arrays re-list nodes that `children` already
  lists, so the dump doubles at every level. `parentNode` and `ownerDocument` walk back
  up the tree.
- Node's line diff of two dumps that size exhausts a 1 GB heap: OOM, exit 134, after
  about 8.5 s.
- Defining `ownerDocument`, `parentNode`, `_closest`, `inserted`, and `removed` as
  **non-enumerable** own properties (`writable: true, configurable: true`) drops the
  dump to 1,976 lines. `assert.equal` and `assert.deepEqual` on the two subtrees then
  throw `AssertionError` in 256 ms and 514 ms under `--max-old-space-size=1024`. The
  diff is readable: `_text: 'alpha'` vs `'beta'`, plus class and attrs lines.

Assigning to an existing writable non-enumerable own property keeps it non-enumerable,
so `this.parentNode = x` elsewhere in each class keeps working.

### Changes

1. **Hide back-references and bookkeeping in every fake.** The inventory is the
   `expected` map in `scripts/test-ledger-tools-fakenode-inspect.cjs`: 11 cyclic fakes
   across the ledger-tools suites (7 `FakeNode`, 2 `MiniNode`, and 2 `Node` in
   `pomodoro-home-chips`), plus the two regression files. In each class, find every
   field that does one of the following:
   - points up or sideways: `parentNode`, `ownerDocument`, `doc`, `_closest`, and any
     owner, view, root, or window reference;
   - re-lists nodes that `children` / `childNodes` already holds: `inserted`, `removed`,
     and similar logs.

   Make those fields non-enumerable at the end of the constructor, through a small
   helper local to each suite. Keep `children` / `childNodes`, attributes, classes,
   text, and style enumerable so a diff still identifies the node. Keep the change local
   to each suite. The shared harness belongs to `bob-cli-4g` and stays out of scope.
   Keep the existing `inspect.custom` descriptors: `util.inspect` messages and the
   `node:test` reporter still use them.

2. **Restore the plan-host identity asserts (original plan step 2).** In
   `scripts/test-ledger-tools-day-strip-plan-host.cjs`, convert the
   `assert.ok(a === b, nodeNote(...))` / `assert.ok(a !== b, ...)` checks back to
   `assert.equal` / `assert.notEqual`, keeping each message. Delete `nodeNote` and its
   `bob-cli-c7` comment if nothing else uses them. Then break one converted assert on
   purpose (compare against the wrong node) and run that suite under
   `node --max-old-space-size=1024 --test`. It must report an `AssertionError` that
   identifies both nodes within a few seconds. Revert the break and record the observed
   time.
3. **Make the regression test the real defect.** Rework
   `scripts/test-ledger-tools-fakenode-inspect.cjs` and its `-child.cjs` probe. Their
   `FakeNode` must mirror the verbatim plan-host class, including `_classSet`,
   `classList`, `_closest`, `inserted`, `removed`, listener maps, and the new hidden
   fields, and build the same 85-node tree. In the child under
   `--max-old-space-size=1024` with a timeout, a deliberately failing
   `assert.equal(alphaSubtree, betaSubtree)` and a failing `assert.deepEqual` must each
   throw `AssertionError` within a few seconds, and each message must distinguish the
   nodes (for example by containing both `alpha` and `beta`). Keep the inventory test
   that every suite carries its descriptor(s). Add a guard that fails if any suite's
   fake exposes a hidden field again. Prefer a cheap in-suite check, for example
   `Object.keys(node)` containing none of that suite's hidden fields. Extending the
   source-inventory test is the fallback. Show the regression fails without the fix:
   temporarily make the regression's fields enumerable, observe the OOM (exit 134) or
   timeout, revert, and record it.
4. **Fix the comments.** The header comment in `test-ledger-tools-fakenode-inspect.cjs`
   says node-identity checks "must therefore stay in the
   `assert.ok(left === right, message)` form". Every descriptor comment says the
   descriptor makes a failing assert report "instead of diffing the whole cyclic graph".
   Both claims are now false. Rewrite them: hidden back-references keep assert's diff
   bounded, and descriptors keep `util.inspect` output and reporter output short.
5. **Verify.** Run the touched suites, then `npm test` and `npm run validate`. These are
   test-only changes with no plugin source edits, so do not bump a version, rebuild an
   entrypoint, or deploy. `bob plugins list` must still report 0 drift. No bob-cli or
   docs change is expected. If one turns out to be needed, run `just check`, never
   `just check-full`.

## Step 2: Close `bob-cli-c7`

Once step 1 is green, run:
`sase bead close bob-cli-c7 --note "<root cause, fix, tests, and observed timings>"`.

Name phase commit `0ccb3ac` plus your change. Give the before and after evidence: the
regression with enumerable fields OOMs or times out, and with the fix both asserts
report in N ms. Include the deliberately broken plan-host assert's report time and the
`npm test` / `npm run validate` results.

## Step 3: Close epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6`

1. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6`. For each listed
   `--epic-symbol` entry, resolve the symbol: wire it up, privatize it, add a non-test
   pragma, or delete it per the Symvision epic-whitelist policy. Re-key a Justfile line
   to a still-open bead only if that bead still needs the exemption. The lander saw no
   entries.
2. Close the epic normally, never with `--force`:
   `sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6 --note "<verification>"`. Summarize
   the lander's verification note:
   - `ca`, `8z`, `c6`, and `c8` were verified and closed;
   - `just check` passed under load;
   - the vault plugins match master;
   - the integration review found no conflicts;
   - the `.3` follow-up was declined;
   - there were no epic symbols.

   Add your step 1–2 evidence. If the close is rejected, fix the cause and close again.

3. Run `just symvision` if the recipe exists (`just --list | grep -i symvision`). At
   this landing bob-cli has no such recipe. If it is still absent, say so in your notes.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/backlog_batch12_perf_baseline_fakenode_daystrip_blocks_notice.md`. Use
   the PLAN path printed by `sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6.6`; it lives
   in the plans sidecar.
5. Parent handling: the epic's `parent_bead` is the plan bead
   `bob-cli-an.6.6.6.5.6.5.5.6.6.6` (batch 11). Re-read it with
   `sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6 -r "Need the parent link"`. Its lander
   closed it (done, 2026-10-26T12:22:56Z) before proposing batch 12, and its own parent
   (batch 10, `bob-cli-an.6.6.6.5.6.5.5.6.6`) is also closed, so there is nothing to
   close. If either is unexpectedly open, follow the standard plan-parent readiness
   traversal instead:
   - recheck its descendants, notes, and linked plan;
   - retire its epic symbols;
   - close it normally only if it is complete, then mark its plan done.

   Stop at the first incomplete ancestor and record a blocker note on it.

## Step 4: Commit, then propose the successor epic (mandatory continuation)

`sase plan propose` terminates the runner without the commit finalizer. Before
proposing, commit every repository you changed with `/sase_git_commit`: bob-plugins for
the test changes, the plans sidecar for the `status: done` edit, and bob-cli only if you
changed it. This approved plan explicitly authorizes `/sase_git_commit` for exactly
these repository obligations, as batch 12's own plan does for its lander.

Then carry out the continuation contract below as the effective lander of batch 12.
Author the next backlog epic ("Backlog batch 13") through `/sase_plan` with
`parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6`. Copy the contract into it verbatim,
including its own lander instructions. Refresh only the candidate guidance in item 3.
Before proposing, record the successor's reference and ownership with
`sase bead note bob-cli-an.6.6.6.5.6.5.5.6.6.6.6 "CONTINUATION: ..."`. Include the fresh
unresolved count.

Enumerate again even though the lander already counted. Candidate notes beyond the
contract text below:

- Active epics `bob-cli-c9` (owns `create --force`, so `az` waits on it) and
  `bob-cli-3j` (owns `8q`) were still in progress at the lander's audit. Recheck both.
- `bob-cli-cb` (small, the four remaining popout body-class gates) and `bob-cli-ax`
  (small, Athena Prometheus `clipster_server` scrape job down) are small deterministic
  bugs.
- `bob-cli-b4` still needs Bryan's choice between reinstalling the web assets and
  dropping the UI.

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
   candidates include:
   - the remaining popout body-class gates `bob-cli-cb`;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-az` once active epic `bob-cli-c9` has landed;
   - flakes `bob-cli-5o`, `bob-cli-b9`, and `bob-cli-67`;
   - a resized `bob-cli-v`.

   Security `bob-cli-8r` and content-loss alerting `bob-cli-9x` outrank these. Each is
   large and should get its own planned launch rather than a backlog-batch phase.
   `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic owns the PTY
   completion drivers. This is guidance, not a preassigned batch, and concurrent work
   may obsolete it.

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
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
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

The successor epic must also carry the working rules batch 9 used:

- Phase workers leave target tasks open for the lander and record baseline, fix, tests,
  and limitations on their phase.
- Use `/sase_repo` before touching another repository, and `sase artifact read` for
  artifacts.
- No live vault edits, live feed publishes, or credential handling. There are two
  exceptions:
  - bob-plugins phases deploy only through `bob plugins sync`.
  - A live-vault repair runs only as a command of a Bryan-approved `/sase_gate` turn.
- Build bob through `$CARGO_TARGET_DIR`.
- Follow the daily release train: no version or `CHANGELOG.md` edits.
- Run `just check`, never `just check-full`.
- The lander must commit every changed repository, including the plans sidecar status
  edit, with `/sase_git_commit` before proposing its own successor. This plan explicitly
  authorizes the skill for those obligations.

## Acceptance

- A deliberately failing `assert.equal` and `assert.deepEqual` on two live 42-node
  FakeNode subtrees (verbatim plan-host shape) throw `AssertionError` within a few
  seconds under `--max-old-space-size=1024`, and their output distinguishes both nodes.
  The same regression OOMs or times out with the fields enumerable.
- Every cyclic ledger-tools fake hides its back-references and bookkeeping, and a guard
  catches a regression. The plan-host identity checks use `assert.equal` /
  `assert.notEqual` again, and the stale "must stay `===`" comments are gone.
- `npm test` and `npm run validate` pass. No plugin version, entrypoint, or deploy
  change.
- `bob-cli-c7` and epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6` are closed normally with
  evidence. Epic symbols are clean, the batch-12 plan file says `status: done`, and
  every changed repository is committed.
- Batch 13 passes validation and is proposed through `sase plan propose`, with its
  reference recorded on the batch-12 epic bead.
