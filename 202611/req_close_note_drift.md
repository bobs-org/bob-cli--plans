---
tier: tale
title: Requirement close card survives unrelated note edits
goal:
  Drop and Carry choices on the managed Done requirement card apply even when another
  writer touches the owner's note while the card is open, and any refusal is shown,
  never silent.
size: medium
decisions:
  undo_scope:
    ask:
      Also make the card's Undo survive the frontmatter write that follows a Carry in a
      project note?
    choices:
      follow_up: Fix the apply path only and file a bead for the Undo refusal
      now:
        Also rework Undo to restore only req hunks; touches the documented Undo contract
    default: follow_up
    why:
      The apply bug loses choices on every project-note close; Undo only refuses visibly
    answer: follow_up
proposed_by: bbugyi200.athena.1fn
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 13:29:01
status: wip
---

# Requirement close card: unrelated note edits silently discard Drop and Carry

## Verdict on Bryan's suspicion

**Confirmed.** In the 2026-11-23 13:15 screencast, Bryan closes `foo` in `bob.md` with
Ctrl+Enter. The requirement card opens with `boom`, `bam`, and `slam`. He presses Space
on `bam` (Carry and pull in), then Ctrl+Shift+X on `slam` with the reason "some reason",
then Enter. The owner becomes `[x]`. Nothing else is written: `bam` is not a `[>]` stub,
there is no new `^bam` task, `slam` is not `[-]`, and there is no Cancel Log. No receipt
or notice appears either.

The vault history agrees. Commit `5df6ece4e` (13:15:31) is the only write during the
gesture. It holds the `[x]` plus `open_task_count: 19 → 18` in `bob.md`'s frontmatter,
and the requirement lines are unchanged. The earlier tries at 13:11:48 and 13:12:57 show
the same result.

## Root cause

The fault is in the `bob-plugins` repo, Task Status Cycler plugin, not in bob-cli. Bob
Mac Capture and the Rust engine check staleness per row
(`src/native/task_complete/req_writes.rs`) and are not affected.

1. **Whole-note staleness gate (the bug).**
   - `reqCloseWritePreflightExpected`
     (`plugins/task-status-cycler/src/198c-req-close-apply-plan.js`, about line 153)
     compares each whole note in `expectedNotes` to the live note.
   - `expectedNotes` is the note text from when the card opened, plus the gesture's own
     writes (`reqCloseComposeExpectedNotes` in `198h-req-close-card-wiring.js`, called
     from `137-plugin-successor-project.js`).
   - Any other byte change anywhere in the owner's note while the card is open makes it
     return `req_close_stale` for the whole pass ("stale requirement pick: bob.md").
   - In a project note this always happens if the card stays open past autosave. The
     `[x]` save fires `metadataCache` `changed`, and the **Bob Project Tasks** plugin
     (`plugins/bob-project-tasks/main.js`, `processFrontMatter`) then rewrites
     `open_task_count`.
   - Vault sync and other plugins can trigger the same refusal.
   - This contradicts the contracts:
     - bob-cli `docs/task-dependencies.md` §12.25 binds identity to the owner block, and
       stale means "reordered, re-indented, edited-duplicate, or disappeared rows".
     - bob-plugins `docs/task-status-cycler.md` §"Requirement close card" says the
       cycler re-reads "the owner and its req lines".
2. **Silent refusal (why nothing was shown).**
   - `applyManagedReqCloseCard` (`198g-req-close-integration.js`) passes the whole-pass
     refusal `{ok:false, code, message, selectors}` from `applyReqCloseWrites` through
     unchanged.
   - `planAndApplySuccessorsNow` (`137-plugin-successor-project.js`) only forwards
     `req_outcomes`, `met`, and `failures`. `planAndApplyIdlessRequirementOnlyCard`
     (`198i-req-close-idless.js`) does the same.
   - So the model is empty, `successorNoticeText` is empty, and `presentSuccessorModel`
     stays silent.

**Reproduced.** The test harness ran Ctrl+Enter on Bryan's exact pre-close `bob.md`
(vault commit `1ede9f368`) and daily note, with a stub v6 picker returning
`bam=carry_pulled` and `slam=drop("some reason")`:

- Without concurrent edits, both HEAD and the deployed build write the correct bytes.
- With one `open_task_count` bump written while the picker is open, both write nothing,
  make no `carryReq` call, and show nothing. This matches the screencast.
- With an owner-scoped preflight (prototype), the race case writes the correct bytes,
  and all 108 existing req-close tests pass.

## Changes (all in the bob-plugins repo unless noted)

Open the repo with `sase repo open bob-plugins -r "..."` and read its `AGENTS.md`. Edit
`src/` fragments only, never `main.js` by hand.

### 1. Owner-scoped preflight (`plugins/task-status-cycler/src/198c-req-close-apply-plan.js`)

Rewrite
`reqCloseWritePreflightExpected(notes, owners, verdicts, expectedNotes, expectedOwnerPostimages)`
so staleness is checked per owner:

- Only owners named by a verdict (`verdicts[].row.owner_key`) are checked.
- The expected block for each owner is `expectedOwnerPostimages[owner.key]` when
  present. Otherwise it is
  `reqCloseWriteOwnerBlock(expectedNotes.get(owner.path), owner)` when `expectedNotes`
  has that path. With neither, skip the check for that owner.
- Refuse with `reqCloseWriteStale([owner.key])` only when the live note is missing or
  `reqCloseWriteOwnerBlock(live, owner)` differs from the expected block.
  `reqCloseWriteOwnerTask` finds the owner by block ID or unique description, so line
  shifts from frontmatter edits are fine.
- Remove the whole-note `live !== expected` loop. Add a short comment explaining why:
  bytes outside an owner block (frontmatter counts, other tasks, vault sync) are never
  staleness (§12.25).

The other guards stay unchanged and keep correctness:

- row identity (`reqCloseWriteFindPending`);
- nav `carryReq`'s `expectedLine` / `expectedOwnerPreimage`;
- the write-time compare-and-swap in `reqCloseWritePersistDrop` /
  `persistReqPullInPlan`;
- `reqCloseRefuseStaleOwnerChoices`.

Do not change the RV twin or vectors.

### 2. Never refuse silently (`198g-req-close-integration.js`, `137a-successor-offer-close-integration.js`)

- In `applyManagedReqCloseCard`, when `applyReqCloseWrites` returns `ok === false`, turn
  it into a normal result:
  - Set `outcomes: []`, `req_outcomes: []`, `met: []`, and `restores: []`.
  - Add one `failures` entry `{ kind: "requirement", code, reason, key }` for each owner
    named by a request (every owner when none can be resolved; use
    `reqCloseOwnerKeyForSelector`).
  - For `reason`, `req_close_stale` uses the existing copy
    `requirement choices went stale for “{owner.text}”`. Any other code uses
    `result.message`.
  - Keep `code`, `message`, `reqJump`, `index`, `completed`, and `closing`.
  - Both callers (`137-plugin-successor-project.js` and `198i-req-close-idless.js`)
    already merge `reqCloseResult.failures` into the receipt. Confirm they now do, and
    that `finishManagedReqPullIn` is skipped.
- In `successorOfferCloseNoticeText`, render `failure.kind === "requirement"` as
  `⚠ Requirements unchanged: {reason}` instead of `⚠ Close write failed: …`. Other kinds
  keep their current copy.
- Check with the nav harness that a failure-only requirement model is not silent. The
  nav card must show it, or `showUnblocked` returns false and the cycler shows its plain
  `Notice`. If the nav card swallows it, make the smallest change in
  `plugins/bob-navigation-hotkeys/src/713a-nav-req-close-receipt.js` so requirement
  failures render as lines, and bump nav's manifest patch version too.

### 3. Regression tests (`scripts/test-task-status-cycler-req-close-public-paths.cjs`)

Add tests using the file's existing `installReqCloseNav` / `exporterNote` helpers. The
note uses `type: "[[project]]"` and `open_task_count:` frontmatter to mirror `bob.md`:

- **Frontmatter write during the card:**
  - Inside `pick()`, rewrite the owner note's `open_task_count`, through both the editor
    and `vault.modify`, the way Bob Project Tasks does.
  - Then return a Drop with a reason and a Carry+pull-in on a depth-2 req.
  - Assert the `[-]` + Cancel Log bytes, the `[>] [[#^…|…]]` stub, the created `#task`,
    the pull-in link in the daily note, and the receipts.
- **Unrelated task edit during the card:** an edit to a different task in the same note
  does not refuse.
- **Whole-pass refusal is visible:**
  - An owner-block edit during the card that the row checks do not catch first (or a
    forced `applyReqCloseWrites` refusal) produces `failures` with `req_close_stale`.
  - `helpers.successorNoticeText(model)` (export it if needed) contains
    `⚠ Requirements unchanged:`.
  - Cover the idless requirement-only path too.
- Keep the existing staleness tests passing:
  - public-paths test 8;
  - `scripts/test-task-status-cycler-req-close-apply.cjs` "a dirty editor that no longer
    matches expectedNotes is stale". Its appended line falls inside the owner block.
  - If needed, rename the latter so it says the edit is inside the owner block.

  Add a sibling apply-level test: an edit outside the owner block (frontmatter) with
  `expectedNotes` supplied still returns `ok: true`.

### 4. Docs, version, build, deploy

- bob-plugins `docs/task-status-cycler.md` §"Requirement close card": say that only the
  owner block and its req rows count as stale. Edits elsewhere in the note while the
  card is open (Bob Project Tasks refreshing `open_task_count`, a vault sync) never
  refuse, and a refusal always shows a `⚠ Requirements unchanged` receipt.
- bob-cli `docs/task-requirements.md` §7: add the same one-sentence clarification after
  the sentence on `req_close_stale` staleness, then run `just docs-check`. This is a
  contract clarification only, with no change to vectors or Rust.
- Bump `plugins/task-status-cycler/manifest.json` 1.58.2 → 1.58.3, plus nav if it
  changed. Run `npm run docs:gen` if the inventory changes.
- Run `npm run build`, then the req-close suites:
  - `test-task-status-cycler-req-close-{apply,public-paths,integration,pull-in,vectors,landing}.cjs`
  - `test-req-close-twin-copies.cjs`
  - `test-navigation-req-close-receipt.cjs`
- Then run `npm test`.
- Deploy with `bob plugins sync`. The vault's current build came from a dirty
  bob-plugins tree at `3141cf64`. If sync refuses, follow the bob-plugins `AGENTS.md`
  procedure exactly (no `-F` on your own).

### 5. Follow-up (decision `undo_scope`)

> [!decision] undo_scope = follow_up Use `/sase_new_task` to file a bug bead in
> bob-plugins (size `large`). After a Carry in a project note, Bob Project Tasks
> rewrites `task_count` / `open_task_count`. The close card's whole-note guarded Undo
> then refuses ("Could not undo requirement writes"), even though only frontmatter
> changed. Reference this fix.

> [!decision] undo_scope = now Also make the req-close Undo restore only the requirement
> hunks: the owner block and the created task block, each guarded by its own postimage.
> Leave other bytes alone. Update the Undo wording in both docs, add a public-path test
> where frontmatter changes after a Carry and Undo still succeeds, and skip the bead.

## Acceptance

- Reproduce Bryan's gesture: in a project note, Ctrl+Enter on a task with nested reqs,
  wait past autosave so Bob Project Tasks updates `open_task_count`, Carry+pull-in a
  depth-2 req, Drop another with a reason, then Enter.
- Expected result:
  - a `[>]` stub and a new task linked under today's Pomodoro;
  - `[-]` plus Cancel Log on the dropped req;
  - receipts shown.
- Any genuine refusal shows `⚠ Requirements unchanged: …` and never fails silently.
- No change to bob-cli Rust code, RV/RQ/RL/RW/SP vectors, or Bob Mac Capture.
