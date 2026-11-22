---
tier: tale
size: medium
title:
  Finish landing bob-cli-oz — post-NAME Enter, in-flight keys, S6 siblings, live Esc
  ladder, T9 gate
goal:
  Fix the five gaps the bob-cli-oz land agent found in the prompt (( task links so the
  epic's T2, T11, and T9 rules hold and the live walk passes without workarounds. Then
  close epic bob-cli-oz.
proposed_by: bbugyi200.apollo.bob-cli-oz.land
proposed_with: claude/opus
bead: bob-cli-oz
status: done
---

- **PARENT:**
  [202611/prompt_task_links.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/prompt_task_links.md)
- **BEAD:**
  [bob-cli-oz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oz/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-oz.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-oz.land.md)
- **COMMITS:**
  - [65fce22](https://github.com/bobs-org/bob-plugins/commit/65fce2281dca17551bed7b6ae1985196f66a5ff0)
    — feat(nav): land prompt task-links follow-ups items 1-3 plus tests, docs, perf
    harness

# Plan: finish landing `bob-cli-oz` (`((` task links in Bob's prose prompts)

## Context

Epic `bob-cli-oz` shipped `((` task links in every Bob prose prompt. Its seven phases
are closed. The epic plan is `plan:202611/prompt_task_links.md`; run
`sase bead read bob-cli-oz -r "<why>"` to see its PLAN path and the landing triage note.
The land agent verified the work and found the gaps below. All of them are in
**bob-plugins**. bob-cli and bob-mac-capture need no changes.

Already handled by the land agent, so do not redo them:

- **Follow-up triage.** Every `PROPOSED FOLLOW-UP` note is triaged and recorded on the
  epic. The perf flakes were +1'd on `bob-cli-km` and `bob-cli-io`, and the docs budget
  on `bob-cli-ob`. Already-fixed items were declined.
- **bob-mail.** The S13 wiring is deployed to the vault: commit `c2e28ca` held the old
  stamped bundle byte for byte, so `-F` was not needed.
- **`--epic-symbol` entries.** `sase bead epic-symbols bob-cli-oz` reports none.
- **Parent bead.** The epic has none.

Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`). Read its
`AGENTS.md`/`CLAUDE.md`, and follow them:

- Edit only `src/` fragments.
- Keep every fragment at or under 1000 lines. nav's `189b-prose-field-controller.js` is
  at exactly 1000, so put new code in `189e` or a new fragment, never in `189b`.
- Run `npm run build`, and never hand-edit `main.js`.

Paths below are relative to the bob-plugins checkout. `nav` means
`plugins/bob-navigation-hotkeys/src` and `bip` means `plugins/block-id-prompt/src`.

apollo's root disk was 100% full at landing (about 132 MB free). If a build or test hits
`ENOSPC`, run the heavy runs on athena, as `bob-cli-oz.7` did. Never delete other
workspaces' files. See `sase memory read tailnet.md` for access.

## Work

### 1. A NAME accept must never eat the next Enter (T2)

**Root cause.** In `nav/189e-prose-task-mode.js`, the NAME-stage Enter keydown runs
`void commitName()` without arming the held-Enter swallow. `commitName` then calls
`options.armSwallow()` only after `await session.name(...)` resolves. nav's controller
(`189b`) clears `swallowEnter` only on an Enter `keyup`.

When the write takes longer than the keypress, the order is:

1. Enter keydown.
2. Enter keyup (nothing to clear yet).
3. The write resolves and arms the swallow.

The armed swallow then eats the user's next Enter, which was meant to submit or save.
This happens whenever the task note is not open in an editor, because `vault.process` is
slower than a keypress. That is exactly why the live walk needed workarounds:

- S14 (Intention Card) retried Enter.
- S15 (Record Card) "Enter timed out" and fell back to `recordApplyTyped`
  (`bob-cli-oz.7` note #2).

**Fix.**

- Arm the swallow synchronously in the NAME Enter keydown branch, before
  `void commitName()`, so it covers the whole press.
- Remove the post-`await` `armSwallow()` in `commitName`'s success path.
- Leave the list-stage arms (identified accept, ID-less → NAME, disabled row) as they
  are.

**Tests.** Add these to `scripts/test-navigation-prose-task-band.cjs`, driving the real
controller with a deferred `session.name` promise:

- **Write resolves after the keyup.** Enter keydown in NAME, then Enter keyup, then
  resolve the write. The link is inserted, and the next fresh Enter keydown is not
  consumed: `handleKeydown` returns `false`, so the host sees it.
- **Write resolves while Enter is held.** A repeat or second keydown before the keyup is
  swallowed. After the keyup, Enter is free.
- **S15 Record Card.** Extend `scripts/test-ledger-tools-record-card-prose-field.cjs`:
  after a NAME accept whose write resolves after the keyup, one Enter reaches the card's
  save path (`recordSaveTyped`).
- **S14 Intention Card.** Extend
  `scripts/test-ledger-tools-intention-card-prose-field.cjs` the same way.

### 2. Every key is swallowed while the NAME write is in flight

**Root cause.** The plan's keyboard contract says "Any key while the write is in flight:
Swallowed". In `189e`, the NAME branch of `handleKeydown` and `backOut()` ignore
`namePending`:

- A second, non-repeat Enter during the write starts a second `commitName()` and a
  second `session.name()`.
- Escape or Ctrl+[ moves the stage back to the list mid-write. The pending success then
  still inserts the link.

**Fix.**

- While `namePending` is true, NAME `handleKeydown` returns `true` and does nothing
  else. This includes Enter, Esc, Ctrl+[, Tab, and the arrows.
- `backOut()` in NAME also returns `true` without changing the stage. nav's `backOut`
  already marks the event as peeled.
- `handleBeforeinput` already cancels input in NAME, so leave it as it is.

Also check, and test, that a blur or `detach` during an in-flight write never inserts
the link into the field. The success path must take the same branch as "field changed":
only the `Added ^id to <note>` Notice.

**Tests** (same suite):

- A second Enter during a pending write calls `session.name` exactly once.
- Esc or Ctrl+[ during a pending write leaves `isNaming()` true until the write settles.
- Blur during a pending write inserts nothing and still shows the Notice.

### 3. S6: ID-less successor rows are subjects of the cancel reason (T11)

**Root cause** (`bob-cli-oz.4` note #2). `nav/206d-successor-picker-verdict-render.js`
`successorReasonSubjects(row)` declares only the row being cancelled. The successor
picker can write to every row it lists. An ID-less row's verdict key and preimage come
from its line digest. If the cancel reason's `((` names an ID-less sibling row, that
row's staged verdict, or one staged later, refuses with `changed_identity` (see
`task-status-cycler` `089g-successor-offer-close.js`).

**Fix.** Have `successorReasonSubjects(row, modal)` return:

- the cancelled row, as today;
- plus every other row in `modal.model.groups[*].rows` whose `task` has a valid path and
  a 1-based line, and whose `task.preimage` (or `task.text`) has no trailing block ID.
  Use the existing `getTrailingBlockId` / `proseTaskSubjects` helpers, and convert the
  1-based line to the 0-based guard line as today.

Identified sibling rows stay linkable, because linking one writes nothing and "dup of
((other successor))" is a real use. Keep the one-argument form working, since existing
tests call `helpers.successorReasonSubjects(row)`. Pass the modal at the `attach` call
site: `subjects: () => successorReasonSubjects(row, modal)`. The guard wording stays
bip's `The task this prompt is about` / `this prompt`.

**Tests.**

- **Subject list.** In `scripts/test-navigation-successor-verdicts.cjs`, build a picker
  with three rows: the cancelled row, an identified sibling, and an ID-less sibling. The
  subjects include the cancelled row and the ID-less sibling, but not the identified
  sibling.
- **Host safety.** In `scripts/test-task-status-cycler-successor-apply.cjs` (T11
  section), use `scripts/prompt-sibling-naming-helpers.cjs`. Show that the namer refuses
  the ID-less picker row as subject-guarded. Then show that naming an ID-less task in
  the same note that is not a picker row still lets the cancel, plus a staged verdict on
  another picker row, apply without `changed_identity`.

**Docs.** Add one sentence to `docs/block-id-prompt.md`'s "Task links in prompts"
section: in the successor picker's cancel reason, ID-less successor rows are also shown
disabled, because naming one would make its own verdict stale. Stay within the page's
docs lint budget.

### 4. The live Esc ladder in NAME, and walks without workarounds

`bob-cli-oz.7` note #1: the real-Obsidian walk never pressed Escape in NAME. Under CDP,
a real Escape was observed to peel 0 NAME layers, and only the unit test covers the
same-event, one-layer peel.

Extend `scripts/obsidian-prose-task-e2e.cjs`. Keep the offline
`scripts/test-obsidian-prose-task-e2e.cjs` green and in `test:unit`.

- **The ladder, on every walked surface.** After `enter-on-idless-opens-name`, press a
  real Escape (CDP `Input.dispatchKeyEvent`):
  - assert NAME closed, the band still open in the tasks list with the query intact, and
    the field byte-identical;
  - then re-enter NAME on the same row and continue the walk.
- **The full ladder, on S1 and S15-text.** After the walk, type `((` again and open
  NAME. Then check that each press peels exactly one layer:
  - Escape → list;
  - Escape → band closed, with the field unchanged;
  - Escape → the host's own Esc (S1 modal closes; S15 backs out of the stage).

  Do the same once with Ctrl+[ on one surface.

- **Remove the swallowed-Enter workarounds.** Remove the S14 `press("Enter", 2)` retry
  and the S15 `recordApplyTyped(null)` / CTA-click fallback in `saveRecordStage`. A
  single Enter must save, and the check fails otherwise. Review S6's second Enter: keep
  it only if it is the picker's own documented apply step, and name the reason in a
  comment.

If a real Escape does not peel exactly one layer on some host, fix the routing:

- nav's `189b` per-event guard (`markPeeled` / `escLayerEvents`);
- the host's capture listener vs its Obsidian `Scope` / `onEscapeKey` path;
- or the host wiring in `185`, `185a`, `206d`, `242sa`, `297cd`, or bip `090`.

Each real Escape must peel exactly one layer. Add a unit test for each fix. Keep `189b`
at ≤1000 lines.

**Run the walk** on athena under `xvfb-run`, per `docs/develop/validation.md`
"Prose-task walkthrough". Use the current build and a bob that has `capture-complete -T`
(`~/.cargo/bin/bob` 0.5.0 there was enough for `bob-cli-oz.7`). It must report
`pass: true` with no failed checks and no console exceptions, and include the mention
round-trip. Record the report path and the check count in the closing note.

### 5. The T9 perf gate for `api.taskLinks`

The plan's T9 requires these budgets in `test:perf`:

- keystroke-to-rows p95 ≤ 12 ms over 5,000 tasks;
- session open (snapshot plus first rank) ≤ 50 ms;
- no vault reads while filtering.

`scripts/test-block-id-prompt-task-wikilink-index-performance.cjs` gates only the editor
index path. `bip/132-plugin-task-links-api.js`'s `session.rank()` also maps every ranked
row through `publicRow` (title parts and guards) on each keystroke, and nothing gates
that.

- Add `scripts/test-block-id-prompt-task-links-api-performance.cjs`. Build it on
  `scripts/task-wikilink-index-perf-support.cjs`, follow the existing CPU-time
  measurement style there (not single-sample wall clock; see `bob-cli-io`), and add it
  to `test:perf` in `package.json`. Assert the three budgets through the real
  `api.taskLinks` (`open({ sourcePath, subjects })`, then `rank(q)` over a realistic
  query sequence), plus zero vault reads during `rank`.
- If a budget fails, make the mapping cheaper without changing the API shape
  (`{ rows, total, indexing }`, every ranked row returned). For example, cache
  `publicRow` per row key and query inside the session, or build `titleParts` once per
  row. Never raise the budgets.

### 6. Ship

1. Bump `bob-navigation-hotkeys` by a minor version (from 2.106.0), and any other
   changed plugin too (bip if item 5 changed it, ledger-tools if item 4 changed it).
   Then run `npm run build` and `npm run docs:gen`.
2. Run:
   - `npm test`; the only known flakes are the ledger-tools 25 ms budgets tracked by
     `bob-cli-km` and `bob-cli-io`;
   - `npm run test:perf`;
   - `npm run validate`;
   - `npm run docs:check`. The `plugin-apis.md` word warning is pre-existing and tracked
     by `bob-cli-ob`.
3. Run `bob plugins sync`. If it refuses, follow bob-plugins `CLAUDE.md`. Never pass
   `-F` on your own.

## Final step: close epic `bob-cli-oz`

Do this in the same turn as the code. It does not wait for a commit SHA, a push, or CI.

1. **Epic symbols.** Run `sase bead epic-symbols bob-cli-oz`. For each listed entry,
   either resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete
   it per the Symvision epic-whitelist policy), or re-key the Justfile line to a
   still-open bead that needs it. There were none at landing.
2. **Close.** Run `sase bead close bob-cli-oz --note "<verification>"`. The note must
   summarize:
   - the land agent's verification: all seven phases checked against source;
   - the prose vectors byte-identical across bob-cli and bob-plugins;
   - the post-epic commits reviewed with no conflicts;
   - the bob-mail deploy;
   - the follow-up triage outcomes, already recorded in the epic's landing note;
   - what this tale fixed (items 1–5), with test counts and the athena walk report.

   Never use `--force` to make the close succeed.

3. **Symvision.** Run `just symvision` in bob-cli to confirm the whitelist is clean.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file: the
   PLAN path that `sase bead read bob-cli-oz -r "<why>"` prints,
   `plan:202611/prompt_task_links.md` in the plans repo.
5. **No parent bead.** The epic has none, so nothing else to close.
