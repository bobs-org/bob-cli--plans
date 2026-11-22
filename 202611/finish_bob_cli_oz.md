---
tier: tale
size: medium
title:
  Finish and close bob-cli-oz — pass the api.taskLinks keystroke gate, re-verify,
  deploy, close
goal:
  Make api.taskLinks keystroke ranking fit the T9 12 ms budget over 5,000 tasks without
  weakening the gate or changing the API. Then re-run the full verification on athena,
  deploy nav and Block ID Prompt, and close epic bob-cli-oz with its plan marked done.
bead: bob-cli-oz
proposed_by: bbugyi200.apollo.fm
proposed_with: claude/opus
create_time: 2026-11-22 11:20:45
status: wip
---

- **BEAD:**
  [bob-cli-oz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oz/README.md)

# Plan: finish and close `bob-cli-oz` (`((` task links in Bob's prose prompts)

## Where things stand

Run `sase bead read bob-cli-oz -r "<why>"` first. Its three notes are the history. The
epic plan is `plan:202611/prompt_task_links.md`. The landing tale is
`plan:202611/prompt_task_links_landing.md`, which is already marked `status: done`. Read
it with `sase artifact read`. Its "Final step" section lists what the close note must
contain.

What has landed, all in **bob-plugins**:

- All seven phases are closed.
- Landing items 1–4 are in `65fce228` and `6b829ab1`:
  - post-NAME Enter;
  - keys swallowed while the NAME write is in flight;
  - S6 ID-less sibling subjects;
  - the real Esc/Ctrl+[ ladder with the walk workarounds removed.
- `6b829ab1` also fixed stale `is-naming` styling (`189d`) and the successor reason
  field's native Scope Escape (`206`/`206d`).
- The versions are prepared but not deployed. The repo has nav `2.107.0` and
  block-id-prompt `1.45.0`. The vault still has `2.106.0` and `1.44.0`.

The last full verification ran on athena against `6b829ab1`:

| Check                                             | Result                                                                                                                                                                                                                                                         |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `test:unit`                                       | 8,652 tests, 2 failures. Both are the known ledger-tools 25 ms budgets (`test-ledger-tools-event-link-times.cjs`, `test-ledger-tools-frees-marks.cjs`), tracked by `bob-cli-km` and `bob-cli-io`. These two are allowed.                                       |
| `test:perf`                                       | 16 tests, **1 failure**: `api.taskLinks keystroke-to-rows p95 takes at most 12 ms over 5,000 tasks` measured **21.0 ms** (reproduced at 17.5 ms on apollo). Session open was 45.3 ms against a 50 ms budget, which passes but is tight. No-vault-reads passes. |
| `validate`                                        | 12/12 plugins valid.                                                                                                                                                                                                                                           |
| Live walk (`scripts/obsidian-prose-task-e2e.cjs`) | `pass: true`: 8 surfaces, 142 checks, 0 failed, 0 console exceptions, mention round-trip included.                                                                                                                                                             |
| Isolated startup smoke                            | PASS.                                                                                                                                                                                                                                                          |
| `docs:check`                                      | Failed only on a local environment collision. A stale, ignored, generated `docs/reference/plugins/albums.md` had lost its mount ownership record in that workspace. This is not a source defect.                                                               |

`sase bead epic-symbols bob-cli-oz` reports none. bob-cli has no `symvision` just
recipe.

**Remaining work:** the T9 keystroke budget, a fresh verification, the deployment, the
plan status, and the close.

### Why the keystroke gate fails

Profile on apollo, warm session, CPU ms per uncached `rank(q)` over 5,000 tasks:

| Query             | Matches | Scoring loop alone | Full `rank` |
| ----------------- | ------- | ------------------ | ----------- |
| `task`            | 5,000   | 4.0                | 16.9        |
| `task number 1`   | 2,239   | 10.9               | 24.8        |
| `task number 12`  | 736     | 11.0               | 13.0        |
| `trains`          | 5,000   | 6.0                | 16.9        |
| `trains topic-3`  | 2,090   | 13.0               | 16.0        |
| `task number 412` | 150     | 12.0               | 10.0        |
| `""`              | 5,000   | 0                  | 5.0         |

Two costs dominate in `bip/132-plugin-task-links-api.js` `session.rank`:

1. **Scoring.** Every keystroke rescores every term of the query over all rows.
   Multi-term queries cost 10–13 ms of scoring alone, although typing changes only the
   last term.
2. **Row materialization.** Each keystroke builds `titleParts` (highlighted title spans)
   for every ranked row, up to 5,000. The per-query title cache keeps only 4 queries,
   and nav's band (`nav/189e`, `189d`) reads `titleParts` only for the handful of
   visible slot rows.

## Ground rules

- Open bob-plugins with `/sase_repo`: `sase repo open bob-plugins -r "<why>"`. Read its
  `AGENTS.md` and follow it.
- Edit only `src/` fragments. Run `npm run build`, and never hand-edit `main.js`.
- Every hand-edited fragment must stay at or under 1000 lines. `132` is at 777, so put
  new helpers in a new fragment, for example `bip/132a-task-links-term-cache.js`,
  registered in `plugins/block-id-prompt/src/fragments.json`.
- Paths: `bip` means `plugins/block-id-prompt/src` and `nav` means
  `plugins/bob-navigation-hotkeys/src`.
- **Never raise a budget, cap or drop ranked rows, or weaken an assertion.**
- The API contract must not change:
  - `rank(q)` returns `{ rows, total, indexing }` with every ranked row;
  - each row keeps all current fields;
  - row order stays identical to `orderForWikilinkPicker`;
  - guards (`disabled`) stay fresh when the subject signature changes;
  - filtering does no vault reads.
- Other agents may be landing bob-plugins changes right now. Fast-forward to
  `origin/master` before you verify and again before you deploy.
- **apollo's root disk is 100% full** (about 300 MB free). Run builds and small focused
  tests on apollo if they fit. Run the heavy runs on athena: full `npm test`, perf, the
  live walk, smoke, and docs. Keep screenshots and large evidence on athena, and cite
  their paths. Never delete another workspace's files to make space. See
  `sase memory read tailnet.md` for SSH access.
- This is a single-turn session. Run long commands in the foreground with explicit large
  timeouts. Do not hand verification off to a monitor and end the turn: the last attempt
  stalled exactly there.

## Work

### 1. Lazy, memoized `titleParts` on public rows

In `session.rank`, stop computing `titleParts` for every ranked row.

- Make `titleParts` an **own, enumerable accessor** on each row object. It computes on
  first read, then memoizes. Use the existing `taskLinksTitlePartsFast` logic with the
  row's `base` and the terms captured at rank time.
- Because the accessor is own and enumerable, `JSON.stringify`, object spread,
  `Object.keys`, and `structuredClone` all still see it.
- A row from an earlier `rank` must keep its own query's highlights and its snapshot's
  title, even after later ranks and an index refresh. The existing test "title parts
  keep their query and snapshot after later ranks and refresh" pins this. So capture the
  terms and `base`. Do not look anything up lazily in session state that can change.
- Keep `parsedTitles` (the plugin-lifetime parse cache) and its bound. The per-query
  `taskLinksTitlePartsCache` can stay as a memo, or go if the accessor memo makes it
  redundant.
- Keep every other row field eager. `disabled` is already a cheap cache lookup.

### 2. Per-term tier cache with prefix narrowing

The scorer (`bip/032-task-wikilink-rank.js` `scoreLoweredFieldsWithTerms`) is
decomposable:

- a row's score is the sum, over query terms, of that term's best tier across the row's
  `loweredSearchFields` (3 prefix, 2 word-boundary substring, 1 substring, 0
  subsequence);
- the row is excluded if any term has no tier.

Use that decomposition:

- **Per-term tier function.** Add one that returns exactly the inner loop's best tier
  for one term, with the same prunes and the same surrogate handling. Leave
  `scoreLoweredFieldsWithTerms` untouched: it is the editor picker's hot loop, and the
  wikilink-index perf gate covers it. A differential test pins the two together (see
  item 4).
- **Session cache.** Cache per-term tiers in the session: a `Map` from term to an
  `Int8Array` aligned with `snapshotRows`, with `-1` for no match.
  - Bound it as an LRU of about 32 terms.
  - Clear it wherever `refreshRows()` replaces `snapshotRows`.
- **Prefix narrowing.** When you compute a new term `T`, look for the longest cached
  term `P` for which `T.startsWith(P)`. Rows where `P` is `-1` are `-1` for `T` without
  being scored.
  - Why this is sound: any tier for `T` implies `T` is a subsequence of some field, so
    its prefix `P` is too.
  - Narrow by prefix only. The differential test must cover surrogate-bearing terms and
    lone surrogate halves.
- **Ranking.** Compute each row's score by summing the cached arrays, and skip the row
  if any term is `-1`. Duplicate terms (for example `a a`) count once per occurrence, as
  today. Then fill the same stable score buckets over `snapshotRows` order. The empty or
  whitespace-only query keeps the plain snapshot order.
- **Keep** the "same query → reuse last result" fast paths and the guard-signature
  invalidation exactly as they behave today.

### 3. Make the T9 gate measure real typing

Edit `scripts/test-block-id-prompt-task-links-api-performance.cjs`. Today's keystroke
sample warms every query first, so with term caching it would never measure scoring.
Keep the existing three tests unchanged, and add one:

- **"api.taskLinks typed keystrokes p95 takes at most 12 ms over 5,000 tasks"**.
  - For each of `task number 412`, `trains topic-3`, and `task number 12`, open a fresh
    session. Do not time the open.
  - Then time `rank(prefix)` for every prefix of the query, one character at a time, as
    a user types it. Read the three visible rows' `titleParts` in each sample, as the
    existing test does.
  - Repeat each typed sequence on 5 fresh sessions. Take the per-keystroke minimum
    across the runs, which follows the file's `bestOf` CPU-noise convention.
  - Assert that the p95 over all keystrokes is ≤ 12 ms. Print it with `t.diagnostic`.
- Extend the no-vault-reads test so it also covers a typed sequence.

### 4. Tests

Add these tests to `scripts/test-block-id-prompt-task-links-api.cjs`, or to the wikilink
model or rank test files where they fit better:

- **Differential.** The sum of per-term tiers, or `null` when any term misses, equals
  `scoreLoweredFieldsWithTerms` over randomized rows and queries. Include:
  - Unicode (`é`, CJK);
  - surrogate pairs and lone halves;
  - empty and whitespace queries;
  - repeated terms;
  - terms that are prefixes of each other.
- **Narrowing equivalence.** On one session, `rank(q)` keys equal
  `orderForWikilinkPicker(rowsForPicker(...), q, sourcePath)` for all of these:
  - typing `task number 412` one character at a time;
  - backspacing it out;
  - editing a middle term;
  - switching to an unrelated query;
  - an `ingestNote` plus `emitForced()` refresh in the middle of the sequence. The cache
    must clear.
- **Lazy titles.** For one row:
  - `titleParts` equals the editor's `taskWikilinkHighlightedTitleParts` flattening;
  - a second read returns the same array;
  - `JSON.stringify` and `{ ...row }` include it.
- **Unchanged tests.** The existing guard-freshness, title-cache, score-bucket,
  serialization, and nav band tests stay green without edits:
  - `scripts/test-navigation-prose-task-band.cjs`;
  - `scripts/test-navigation-prose-field.cjs`;
  - `scripts/test-prompt-subjects-host-safety.cjs`.

### 5. Build and version

- Keep nav at `2.107.0` and block-id-prompt at `1.45.0`. Neither has been deployed, so
  this change rides in those versions. If `origin/master` has meanwhile moved either
  manifest past those values, bump that plugin one minor version above master's value.
- Run `npm run build`, then `npm run docs:gen`.
- If the `api.taskLinks` prose in the docs states anything the change makes false,
  update it. Stay within the docs lint budget. The `plugin-apis.md` word warning is
  pre-existing and tracked by `bob-cli-ob`.

### 6. Verify on athena

Set up a fresh `mktemp -d` directory on athena. Copy each tree with
`git ls-files -z --cached --others --exclude-standard`. That keeps new fragments, which
plain `ls-files` would drop. Copy:

- the bob-plugins working tree;
- the bob-cli tree;
- bob-mac-capture, opened read-only with `sase repo open bob-mac-capture`, for the docs
  check.

Run these and record the results:

1. **`npm test`.** The only allowed failures are the two ledger-tools 25 ms budgets
   named above.
2. **`npm run test:perf`, three consecutive runs.** All must pass. Record the
   session-open time, the warm keystroke p95, and the typed keystroke p95 from each run.
   Also run the perf file once on apollo and record its numbers, for information only.
3. **`npm run validate`.**
4. **The live walk.** Follow `docs/develop/validation.md`, "Prose-task walkthrough":
   `BOB_BIN=$HOME/.cargo/bin/bob xvfb-run -a node --experimental-websocket scripts/obsidian-prose-task-e2e.cjs -r <dir>/walk.json -o <dir>/shots`.
   It needs `pass: true`, no failed checks, no console exceptions, and the mention
   round-trip. Record the report path and the surface and check counts. Expect at least
   8 surfaces and 142 checks.
5. **The isolated startup smoke:**
   `xvfb-run -a npm run smoke:obsidian -- --report <dir>/smoke.json`. It must PASS.
6. **`npm run docs:check`** from the bob-plugins copy, with `BOB_CLI_DIR`,
   `BOB_DOCS_BOB_PLUGINS_DIR`, and `BOB_DOCS_BOB_MAC_CAPTURE_DIR` pointing at the fresh
   copies. A fresh copy cannot hit the stale ignored-page collision. If athena lacks the
   docs toolchain, run it on apollo only from a checkout whose ignored generated pages
   are consistent, and only if disk allows.

If any check fails for a reason this epic caused, fix it and rerun the affected checks.
For a pre-existing failure, use `/sase_new_task` to corroborate or file it.

### 7. Deploy, on apollo

From the bob-plugins checkout, fast-forwarded first, deploy each plugin separately:

```bash
bob plugins sync --repo "$PWD" --no-pull --dry-run -p bob-navigation-hotkeys
bob plugins sync --repo "$PWD" --no-pull -p bob-navigation-hotkeys
bob plugins sync --repo "$PWD" --no-pull --dry-run -p block-id-prompt
bob plugins sync --repo "$PWD" --no-pull -p block-id-prompt
```

- If a sync refuses, follow bob-plugins `CLAUDE.md` and `docs/develop/deployment.md`,
  "When sync refuses an overwrite". Never pass `-F` on your own.
- Confirm with `bob plugins list --json`: both rows must show `sync: synced` at the
  expected versions.
- Run the mandatory post-sync gate:
  `npm run smoke:obsidian -- --deployed-vault ~/bob --report /tmp/oz-post-sync.json`.
  apollo has no Obsidian or `xvfb-run`, so it is expected to exit unrun, as `bob-cli-ln`
  recorded. If it does, say so in the close note and rely on the list JSON plus the
  athena isolated smoke of the same tree. Never claim a deployed-vault pass that did not
  run.

### 8. Close epic `bob-cli-oz` in this same turn

1. Run `sase bead epic-symbols bob-cli-oz`. Resolve any entry per the landing plan. None
   are expected.
2. **Plan status.** Open the plans sidecar with `sase repo open plans -r "<why>"`. In
   `202611/prompt_task_links.md`, change only the frontmatter `status: wip` to
   `status: done`.
3. **Close the bead.** Write the note to a file and run
   `sase bead close bob-cli-oz --note @<file>`. Never use `--force`. The note must
   summarize:
   - the land agent's verification: all seven phases checked against source;
   - the prose vectors byte-identical across bob-cli and bob-plugins;
   - the post-epic commits reviewed with no conflicts;
   - the bob-mail deploy;
   - the follow-up triage outcomes, which are in the epic's note #1;
   - landing items 1–4 (`65fce228`, `6b829ab1`): the Escape layer fixes and the walk
     with its workarounds removed;
   - this tale's T9 work, with test counts, the three perf runs' numbers, and the
     typed-keystroke gate;
   - the athena walk report path and its surface and check counts;
   - the startup smoke result;
   - the docs check result;
   - the deployed versions and the post-sync gate outcome;
   - that bob-cli has no `symvision` just recipe, and epic-symbols are none.

## Acceptance

- In `npm run test:perf` on athena, all four `api.taskLinks` perf tests pass in three
  consecutive runs: open ≤ 50 ms, warm keystroke p95 ≤ 12 ms, typed keystroke p95 ≤ 12
  ms, and zero vault reads. No budget changed.
- The differential, narrowing, and lazy-title tests pass. All existing `api.taskLinks`,
  prose-band, and host-safety tests pass unedited.
- In full `npm test`, the only failures are the two known ledger-tools 25 ms budgets.
  `validate` passes, and `docs:check` passes.
- The live walk reports `pass: true` with 0 failed checks and 0 console exceptions, and
  the isolated startup smoke passes.
- The vault runs nav `2.107.0` and block-id-prompt `1.45.0`, or the bumped versions, as
  `synced`.
- `bob-cli-oz` is closed with the note above, and `plan:202611/prompt_task_links.md`
  shows `status: done`.
