---
tier: tale
title: Count [s and ]s review-queue jumps
goal: "Make N]s and N[s jump N entries along the existing freshness review queue in one
  landing, while a bare chord stays a one-step jump.

  "
size: medium
proposed_by: bbugyi200.apollo.4x
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4x](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4x.md)
- **COMMITS:**
  - [10cfee3](https://github.com/bobs-org/bob-plugins/commit/10cfee33df7d8d1ea84a91358362dba0037cd79b)
    — feat(navigation): jump N review-queue entries with N\]s

# Count the Obsidian `[s` / `]s` review jumps

## Finding

`[s` and `]s` do not support counts. `10]s` does not jump forward 10 tasks.

The vault vimrc maps the chords to one-shot Obsidian commands:

```
exmap bob_next_due obcommand bob-navigation-hotkeys:jump-to-next-due-task
exmap bob_prev_due obcommand bob-navigation-hotkeys:jump-to-prev-due-task
nmap ]s :bob_next_due<CR>
nmap [s :bob_prev_due<CR>
```

`obsidian-vimrc-support`'s `obcommand` calls the command callback with no arguments.
Those callbacks are `jumpToDueTask(1)` and `jumpToDueTask(-1)`. `planReviewJump()` then
moves exactly one index. CodeMirror Vim does not repeat a `keyToEx` mapping for a typed
count, and it does not pass the count into the ex command. The digits sit in
`prefixRepeat` until input state is cleared after the command returns. The visible
result of `10]s` is the same one-entry jump as `]s`.

These chords walk the freshness review queue (due tasks plus project and reference
rows), not every open task in the current note. Counted in-note open-task movement
already exists as `N<Ctrl+Shift+J/K>` and is out of scope.

## Outcome

In Vim normal mode, `N]s` lands on the Nth next review-queue entry and `N[s` lands on
the Nth previous entry. `10]s` jumps forward 10 entries. `1]s` and bare `]s` stay the
current one-step jump. The jump is one queue read, one landing, one notice, and one
anchor update. It never stamps or rewrites a task.

This is a `tale` of `size: medium`: one plugin method, one pure planner, its existing
test file, a version bump, and two short doc sentences. There is no new command,
evaluator, or vimrc map. Planning follows `sase_sizes.md`. The approved implementation
is medium; this plan is the large planning handoff.

## Repository access

Open the linked plugin repo from the bob-cli workspace before editing:

```bash
sase repo open bob-plugins -r "Add counted [s and ]s review-queue jumps"
```

Use only the printed path. Read that repo's `AGENTS.md`. Paths below are relative to the
repo they name. Do not edit the installed copy under the vault plugin directory by hand.

Read `decisions:review-walk-is-tiered` and `glossary:task-freshness` with
`sase memory read` before implementing. The queue order and membership stay whatever
`api.freshness.queue()` already returns (today NEW → PROJECTS → PENDING → NEXT →
RETURNED → REFERENCES → ROTTEN). This change only selects a farther index in that list.

No chezmoi change. The bob vault vimrc stays as it is.

## Current behavior to preserve

- **bob-plugins** `plugins/bob-navigation-hotkeys/main.js`
  - `jump-to-next-due-task` / `jump-to-prev-due-task` call `jumpToDueTask(1)` /
    `jumpToDueTask(-1)`. Default hotkeys remain Ctrl+Alt+J / Ctrl+Alt+K.
  - `jump-to-first-due-task` / `jump-to-last-due-task` call `jumpToDueTask` with
    `endpoint: "first"` / `"last"`. `[S` and `]S` are absolute and ignore the cursor and
    the anchor.
  - `planReviewJump()` returns `{ kind: "empty" }` or
    `{ kind: "jump", entry, rank, total, wrapped, originTier }`. Repeat 1 must keep
    today's origin rules:
    - cursor on a live, unhandled entry → neighbor, wrapping at the ends
    - else a shaped anchor → first surviving successor (`]s`) or last surviving
      predecessor (`[s`) in the queue minus handled keys
    - else a legacy `{ keys, rank, count }` anchor → the existing rank-skip, mirrored
      backward
    - else the first entry forward or the last entry backward
  - `jumpToDueTask()` reads the queue, plans, lands, and on a stale land rereads and
    replans once. Success updates `reviewAnchor` and shows `buildReviewJumpNotice`, with
    `buildReviewBoundaryNotice` prepended only for a forward non-endpoint step into
    ROTTEN.
  - `getPendingVimRepeat()` / `resetPendingVimInputState()` / `normalizeVimRepeat()` are
    the count helpers. `normalizeVimRepeat` maps missing, non-finite, zero, and negative
    values to 1.
  - Alt+F / Alt+Shift+F already treat an explicit Vim count as N additional stamps,
    reset Vim input state before writing, and then advance with
    `jumpToDueTask(1, { fromStamp })` exactly once.
- **bob-plugins** `scripts/test-navigation-freshness.cjs` covers the one-step planner,
  anchors, notices, endpoint commands, and the `jumpToDueTask` method harness. Those
  tests must keep passing with no `repeat` argument.
- **bob vault** `obsidian_vimrc.md` lines 33–40 are the mappings quoted above. Leave
  them.
- **bob-cli** `docs/freshness.md` §6 is the ritual text that names `]s`, `[s`, `[S`, and
  `]S`.

## User-visible contract

1. `N` is an ordinary Vim count: the total number of queue steps, not N additional
   steps. `3]s` lands three entries forward from the same origin a bare `]s` uses.
2. The walk list is the list today's one-step jump already ranks against: the full queue
   for a cursor hit and for the no-origin fallback, and the queue minus handled keys for
   an anchor. Compute the current one-step result first. When `N` is 1, return that
   result unchanged. When `N` is greater than 1 and the one-step result is a jump, move
   `N - 1` further indexes in the same direction on that same list, modulo its length.
3. `wrapped` is true when the one-step result wrapped or when any further step crosses
   either end. Crossing more than once is still a single `wrapped: true`. `rank` and
   `total` describe the final entry on the walk list. `originTier` stays the one-step
   origin, so a jump from a commitment tier that lands on ROTTEN still gets the existing
   boundary preamble, and a jump that only passes through ROTTEN does not.
4. A count larger than the list wraps, including a one-entry queue. The review jump
   already lands on that sole entry with `wrapped: true`; a larger count keeps landing
   there. Do not copy the open-task jump's refusal to move when the only target is the
   current line.
5. From outside the queue, bare `]s` lands on entry 1, so `10]s` lands on entry 10,
   wrapping if the queue is shorter. From entry 3, `10]s` lands 10 entries after
   entry 3.
6. `[S` and `]S` stay absolute. A typed count does not change their destination and does
   not turn them into repeated relative jumps.
7. Alt+Shift+F still stamps the counted batch it already stamps, then advances exactly
   one queue entry. It must not also skip N entries.
8. One notice, using the existing builders. Do not add a "jumped N" line. An empty queue
   still shows the existing empty notice and does not land.

## Implementation

### Planner

Add `options.repeat` to `planReviewJump()`. Absent or invalid means 1, via
`normalizeVimRepeat`. `endpoint` still returns before any relative walk and ignores
`repeat`.

Implement the further steps only after the existing one-step selection, so the four
origin rules are not re-derived. Extra movement uses the index of that one-step entry in
its walk list:

```text
extra = repeat - 1
raw = firstIndex + direction * extra
wrapped = firstStep.wrapped or raw is outside [0, length)
finalIndex = modulo raw into the walk list
```

An anchor whose one-step result already wrapped continues the extra steps from that
wrapped index and stays `wrapped`.

### Command

In `jumpToDueTask()`, capture the repeat synchronously, before the first `await` and
before the stale replan:

- An explicit `options.repeat` is the whole count. Do not also read Vim.
- Otherwise, on a non-endpoint call, if the active editor is in Vim normal mode and
  `getPendingVimRepeat()` is `explicit`, that `repeat` is the count. Reset the pending
  input state immediately with reason `counted-review-jump`.
- No explicit prefix, a non-Vim editor, or no editor means 1. Leave Vim state alone in
  that case.
- Endpoint calls do not read or apply a count.

Pass that captured number into both `planReviewJump()` calls (the first plan and the
single stale replan). Do not read Vim state again after an `await`. CodeMirror clears
`prefixRepeat` when the ex command returns, which is after the first await yields.

Keep the command callbacks as `jumpToDueTask(1)` and `jumpToDueTask(-1)` with no repeat
argument. An explicit `1` would suppress the pending-count read. The same rule is why
`jumpToOpenObsidianTask` omits its repeat at the command boundary.

Do not add a capture-phase listener for `]` / `s`. Those keys are a Vim mapping, not a
chord CodeMirror swallows. A listener would race the vimrc command and could jump twice.
Do not `mapCommand` `]s` or `[s` in the plugin; the vimrc mapping remains the only key
binding.

Ctrl+Alt+J / Ctrl+Alt+K go through the same callbacks, so a count already typed in
normal mode applies to them too. With no typed count they stay one step. Insert mode and
non-Vim editing stay one step.

### Release notes and docs

- Bump `plugins/bob-navigation-hotkeys/manifest.json` from 1.73.0 to **1.74.0**.
- In `bob-plugins/README.md`, set that row's version to 1.74.0 and add one clause: `N]s`
  / `N[s` jump N review-queue entries in one landing, and a bare chord stays one entry.
- In `bob-cli/docs/freshness.md` §6, in the paragraph that already names `]s` and `[S` /
  `]S`, add that `N]s` / `N[s` move N entries along that same queue and wrap with the
  existing notice, while `[S` / `]S` stay the first and last entries.

### Deploy

`bob-plugins/AGENTS.md` requires `bob plugins sync` after plugin edits so the vault copy
matches the source. Run it after the version bump. Do not hand-edit the vault plugin.

## Tests

Extend `scripts/test-navigation-freshness.cjs`. Existing one-step and endpoint tests
stay as they are.

Planner:

- `repeat` omitted or 1 matches the current neighbor, wrap, no-origin, shaped-anchor,
  and legacy-rank results, including `wrapped` and `originTier`.
- Cursor on index 0 of a queue of length at least 11, `repeat: 10` forward, lands on
  index 10 with `wrapped: false`.
- Cursor on the last entry, `repeat: 2` forward, wraps to index 1 and `wrapped: true`.
- `repeat` greater than the length wraps with modulo and one `wrapped: true`, forward
  and backward.
- A one-entry queue with the cursor on it stays on that entry with `wrapped: true` for
  repeat 1 and repeat 10.
- No cursor and no anchor, `repeat: 10`, lands on the 10th entry, or wraps when the
  queue is shorter.
- Shaped anchor: repeat 3 lands on the third surviving successor, not on the first
  successor three times. A wrapped anchor continues from the wrapped entry.
- `endpoint: "first"` with `repeat: 10` still returns index 0 and ignores the count.
  Same for `"last"`.

Command harness, reusing `endpointMethodHarness`:

- A Vim-normal editor whose `prefixRepeat` is `["1", "0"]` makes `jumpToDueTask(1)` land
  on the 10th entry, reset that prefix, and show one notice.
- The registered `jump-to-next-due-task` callback, not a direct repeat argument, is what
  performs that read.
- `options.repeat` skips the Vim read.
- A non-normal editor with a pending prefix still jumps one entry and does not clear the
  prefix.
- The stale replan uses the same captured repeat and does not ask Vim again.
- An endpoint jump with a pending prefix still lands on the endpoint.
- A forward counted landing on ROTTEN from a commitment origin still includes the
  existing boundary preamble. An endpoint landing still does not.

Run from the bob-plugins checkout:

```bash
node --test scripts/test-navigation-freshness.cjs
```

## Out of scope

- Changing queue membership, tier order, stamps, or the Alt+F count (that count is
  additional stamps, not additional jumps).
- Counted `Ctrl+Shift+J/K`, `[S` / `]S` behavior beyond ignoring a count, or the vimrc
  text.
- A new notice, command id, or hotkey.
