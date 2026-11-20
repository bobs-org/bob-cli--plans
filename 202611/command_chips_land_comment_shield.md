---
tier: tale
size: medium
title: 'Finish command chips: ⌘↵ never targets chips hidden in multi-line comments
  or math, then close bob-cli-nu'
goal: commandChipAt (the ⌘↵ target rule and keycap) ignores run chips and bob-run
  fences inside multi-line %% comments, HTML comments, and $$ math blocks, matching
  what Live Preview and Reading view render. The command-chips e2e asserts the Live
  Preview copy-card press, and the epic bob-cli-nu is closed.
proposed_by: bbugyi200.apollo.bob-cli-nu.land
proposed_with: claude/opus
bead: bob-cli-nu
status: done
---

- **PARENT:**
  [202611/command_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/command_chips.md)
- **BEAD:**
  [bob-cli-nu](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nu/README.md)

# Finish command chips and close bob-cli-nu

## Context

This is the remaining work of epic **bob-cli-nu**, "Command chips: one-gesture bob runs
from Obsidian notes" (plan `plan:202611/command_chips.md`). All eight phases are closed.
The land agent verified the shipped code: bob-plugins commits bf13f43 … ee941a1,
bob-chips 0.14.0, bob-command-mode 0.4.0, bob-navigation-hotkeys 2.92.0, and
bob-ledger-tools 1.143.0+. It also ran the chip-related unit suites (310/310 pass),
confirmed the deploy state, and triaged every follow-up. That triage is recorded on the
epic as a note, so do not redo it. Two epic-caused gaps remain, and this tale finishes
them and closes the epic. The epic has no parent bead.

Work in **bob-plugins**: open it with `/sase_repo` and read its `AGENTS.md`. The rules
from the epic plan still hold:

- edit only `src/` fragments, each at most 1000 lines;
- never hand-edit a generated `main.js`;
- run `npm run build`, `npm test`, `npm run validate`, `npm run docs:gen`, and
  `npm run docs:check` (with `BOB_CLI_DIR` set to a bob-cli checkout);
- bump the version of every plugin you change;
- deploy with `bob plugins sync`, and never pass `-F` on your own.

Copy chips' behavior and `api.copyChips` v2 must stay byte-stable (contract C11):
`copyChipAt`, the copy vectors, and `COPY_VECTORS_SHA256` must not change.

Known baseline failures that are not yours:

- `docs:check` (and bob-cli `just docs-check`) currently fails at lint on bob-cli
  `docs/reference/vault-format.md` (6016/6000 words). That failure is tracked on
  bob-cli-o6. If it is still failing, run the remaining gates the way bob-cli-nu.8 did
  (bob-docs build, offline links with fragments, and urls), and report them.
- The ledger-tools 25 ms wall-clock perf asserts flake under full parallel `npm test`.
  They are tracked by bob-cli-km and bob-cli-io and pass alone.

## Gap 1: `commandChipAt` ignores multi-line comments and math blocks

Reproduced on master ee941a1 with `scripts/copy-chips-test-helpers.cjs` helpers:

| Lines                                                                                    | Cursor      | Today                                   | Should be |
| ---------------------------------------------------------------------------------------- | ----------- | --------------------------------------- | --------- |
| `%%` / `` `run: bob task reconcile` `` / `%%`                                            | line 1 ch 3 | `ready via cursor`                      | `none`    |
| `<!--` / `` `run: bob task reconcile` `` / `-->`                                         | line 1 ch 3 | `ready via cursor`                      | `none`    |
| `$$` / `` `run: bob task reconcile` `` / `$$`                                            | line 1 ch 3 | `ready via cursor`                      | `none`    |
| `- [ ] Task` / `  %%` / `` `  run: bob task reconcile` `` / `  %%`, task scope lines 0–3 | line 0 ch 2 | `ready via task` (runs the hidden chip) | `none`    |
| `%% `` `run: bob task reconcile` `` %%` (one line)                                       | ch 5        | `none` (already correct)                | `none`    |

Live Preview hides chips in these blocks through the syntax tree, and Reading view
strips comments. But the pure ⌘↵ rule only shields comments and math that open on the
same line: `chipShieldedRanges` in `plugins/bob-chips/src/021-run-grammar.js` handles a
single line only. So `targetAt` arms a target the user cannot see, the keycap advertises
it, and ⌘↵ runs it. The worst case is the task fallback: `⌘↵` on a task line runs a
commented-out chip. This was proposed by bob-cli-nu.3 (note #2) for the keymap phase,
which never picked it up.

### Fix (in `plugins/bob-chips`)

1. **Cross-line block state.** Add one pure, total helper (never throws), for example
   `chipBlockShieldState(lines, uptoLine)` or a per-line shield map. It walks the lines
   from the top of the note, with the same frontmatter and fence handling that
   `chipScanTaskScope` and `copyChipFindFenceBlock` (`065-cursor.js`) already use. While
   walking, it tracks whether a `%%` comment, an HTML `<!-- … -->` comment, or a `$$`
   display-math block is open. Only delimiters outside code spans and outside fences
   count; reuse `chipFindOutsideCode`, `copyChipFindCodeSpans`, and the opener rules of
   `chipShieldedRanges`, so one-line and multi-line behavior agree. For each line, it
   returns the `[from, to)` ranges hidden by a block carried in from an earlier line or
   opened on this line and left unclosed. An unclosed block runs to the end of the note.
   Put it beside `chipShieldedRanges` or in a new fragment listed in
   `src/fragments.json`; keep every hand-edited fragment at or under 1000 lines.
2. **Use it in `commandChipAt`** (`066-command-cursor.js`):
   - drop inline run chips on the cursor line that fall inside a carried or unclosed
     block range, before rules 2 and 4, so they never count, arm, or go ambiguous;
   - rule 1: a `bob-run` fence whose opening line starts inside an open comment or math
     block is `none`, as is the cursor inside such a fence;
   - rule 6: the task-scope scan ignores run chips and `bob-run` fences in hidden
     ranges. Pass the shield through `chipScanTaskScope` only for `kind === "run"`, so
     the copy path stays byte-identical.

   Do not change `copyChipAt`, `copyChipScanLine`, the copy vectors, or any Live Preview
   or Reading rendering. Link handling (rules 3 and 5) is unchanged.

3. **Vectors.** Add `chip_at` cases to `scripts/fixtures/command-chips-vectors.json` for
   every row of the table above. Also add:
   - a cursor below a closed multi-line comment (resolves normally);
   - an unclosed `%%` that hides the rest of the note;
   - a `bob-run` fence inside a `%%` block (`none`);
   - `%%` inside a code span or inside a fence (no shield);
   - a task whose visible chip sits beside a commented-out one (`ready` on the visible
     one, not `ambiguous`).

   `scripts/test-command-chips-grammar.cjs` must still pin the copy vectors' sha256
   unchanged. Extend `scripts/test-command-chips-keymap.cjs` with one `targetAt` case on
   a commented-out task chip (`none`, no keycap).

4. **Docs.** In `docs/bob-chips.md`, extend the paragraph that says the pure scanner
   skips command chips in inline math and comments ("An unclosed comment opener shields
   the rest of its line"). Say that the ⌘↵ target rule also skips chips and `bob-run`
   fences inside multi-line `%%`, HTML-comment, and `$$` blocks. Keep the reference
   page's word count about neutral (it is at about 4,900 of 6,000 lint words), and do
   not touch `docs/develop/plugin-apis.md` (bob-cli-ob tracks its headroom). Run
   `npm run docs:gen`.
5. **Version.** bob-chips 0.14.0 → 0.14.1, or relative to whatever master holds when you
   land.

## Gap 2: the e2e Live Preview copy-card press is only diagnostic

In `scripts/obsidian-command-chips-e2e.cjs`, the case "copy span and bob-copy card still
copy beside run chips" (about lines 565–589) asserts the span copy and the Reading-view
card copy. The Live Preview card press is recorded only as a diagnostic:
`livePreviewHit: notice-message`. Persistent Command Mode failure toasts from earlier
cases cover the card's right-hand Copy button (bob-cli-nu.5 note #1). Before the Live
Preview press, clear `.notice-container .notice` when `elementFromPoint` misses the
button, just as `clickFence` (about line 722) already does. Then add
`liveCardCopy === "card body"` to the assertion, and remove the "diagnostic only"
comment. Keep the offline self-test wrapper
(`scripts/test-obsidian-command-chips-e2e.cjs`) passing.

## Verification

- `npm run build`, `npm run build:check`, and the focused suites must all pass:
  - `node --test scripts/test-command-chips-*.cjs scripts/test-copy-chips-*.cjs`
  - `node --test scripts/test-command-mode-chip-contract.cjs`
  - `node --test scripts/test-navigation-command-chip-dispatch.cjs`
  - `node --test scripts/test-ledger-tools-gmail-open-hint.cjs`
  - `node --test scripts/test-navigation-copy-card-view.cjs`
  - `node --test scripts/test-obsidian-command-chips-e2e.cjs`
  - `node --test scripts/test-obsidian-command-chips-keymap-e2e.cjs`
- Run full `npm test`; only the known flakes listed in Context may fail. Also run
  `npm run test:perf` (the Live Preview budgets must still hold), `npm run validate`,
  and `npm run docs:check`.
- **Real-Obsidian e2e on athena** (xvfb, isolated scratch vault, stub bob), following
  the harness notes in the scripts and in bob-cli-nu.6 and nu.7. Run both
  `scripts/obsidian-command-chips-e2e.cjs` (it must now assert the Live Preview card
  copy) and `scripts/obsidian-command-chips-keymap-e2e.cjs`, and add one keymap e2e case
  for a task whose only run chip sits in a multi-line `%%` comment: no keycap, ⌘↵ falls
  through to the existing chain, and the stub sees zero invocations. If athena cannot
  run them, say so in the close note.
- Commit in bob-plugins. Then run `bob plugins sync` (no `-F`; follow `AGENTS.md` on a
  refusal) and confirm that `bob plugins list` shows bob-chips at the new version,
  synced and enabled. Ask Bryan to reload Obsidian.

## Closeout: close epic bob-cli-nu (final step)

Run these steps in the same turn as the code, after the bob-plugins commit and deploy:

1. Run `sase bead epic-symbols bob-cli-nu`. At planning time it reported no entries. If
   any appear, resolve each one: wire it up, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy. Re-key a Justfile line only to a
   still-open bead that needs the exemption.
2. Close the epic with a verification note, for example:
   `sase bead close bob-cli-nu --note "<what you verified>"`. The note should cover:
   - the land agent's verification of all 8 phases and the follow-up triage (already in
     the epic's notes);
   - the vimrc fix (vault 46c02b4db);
   - this tale's comment and math shield, its vectors and tests, the e2e results, the
     bob-chips version you deployed, and the bob-plugins commit SHA.

   Do not use `--force` merely to make the close succeed. If the close is refused, fix
   the cause.

3. Run `just symvision` in bob-cli and confirm that the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file
   `plan:202611/command_chips.md`. `sase bead read bob-cli-nu -r "<why>"` prints its
   PLAN path.
5. Report to Bryan:
   - reload Obsidian;
   - athena's vault still has bob-chips 0.10.0 until `bob plugins sync` runs there;
   - the new follow-up tasks bob-cli-o7, o8, o9, oa, and ob are ready for triage.
