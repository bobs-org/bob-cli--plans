---
tier: tale
title: Add first and last review-stack keymaps in Obsidian
goal:
  Add normal-mode [S and ]S mappings that jump to the first and last entries of the
  shared review queue while preserving existing review navigation and task content.
size: small
proposed_by: bbugyi200.apollo.4n
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4n](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4n.md)
- **COMMITS:**
  - [ac5419c](https://github.com/bobs-org/bob-plugins/commit/ac5419c39ea0812a98c5b0e8e6d169ff26e72a70)
    — feat(nav): add first/last review-stack jumps with \[S/\]S keymaps (1.66.0)

# Jump to the first and last Obsidian review entries with [S and ]S

## Outcome and scope

Add normal-mode `[S` and `]S` mappings that jump directly to the first and last entries,
respectively, of the same vault-wide review queue used by `[s` and `]s`. An endpoint is
global across the whole queue, not the first or last task in the current note or tier.
Navigation never reviews a task or writes task content.

This is a `tale` with `size: small`: one coding agent can implement a focused extension
to the existing navigation pipeline, its tests, and the vault mappings. There is no new
evaluator, API, CLI option, or independent implementation phase. Planning follows the
canonical `sase_sizes.md` guidance; planning itself is large work, while the approved
implementation is small.

## Repository access and established behavior

Before implementation, use `/sase_repo` to open `bob-plugins` and `gh:bobs-org/bob` from
the bob-cli workspace. Use only the returned paths, read each repository's `AGENTS.md`,
and inspect its working-tree status. All paths below are relative to the named
repository. Do not edit the installed plugin as source. No chezmoi changes are needed.

Read `decisions:review-walk-is-tiered` and `glossary:freshness` with `sase memory read`
before implementing. The queue remains NEW → PENDING → NEXT → RETURNED → ROTTEN with the
existing within-tier ordering and eligibility rules; `bob-cli/docs/freshness.md` §4
defines those rules.

Verified starting points:

- **bob-plugins:** `plugins/bob-navigation-hotkeys/main.js` registers
  `jump-to-next-due-task` and `jump-to-prev-due-task` in `onload()`. Their callbacks
  call `jumpToDueTask(1/-1)`; existing default hotkeys are Ctrl+Alt+J/K.
- **bob-plugins:** `planReviewJump()` selects a relative destination, using the cursor
  and a `reviewAnchor`. `buildReviewAnchor()` records an anchor on every successful
  landing as well as after stamping. Its `keys` therefore mean more than just tasks that
  have been reviewed.
- **bob-plugins:** `jumpToDueTask()` gets the ledger-tools API and queue, resolves a
  plan, calls `landOnReviewQueueEntry()`, rereads and replans once on a stale result,
  updates the anchor after success, and shows the review notice. Landing already
  resolves moved lines, reuses open leaves, and centers/defer-positions the destination
  cursor.
- **bob-plugins:** `scripts/test-navigation-freshness.cjs` has 34 passing tests at
  planning time. It covers pure selection, anchors, stamps, notices, and stale-line
  resolution. Command-registration harness examples are in
  `scripts/test-navigation-hotkeys.cjs`.
- **Bob vault:** `obsidian_vimrc.md` defines `bob_next_due` and `bob_prev_due` with
  `exmap ... obcommand ...`, then maps `]s` and `[s` with `nmap`. Neither uppercase
  mapping exists. The tracked `.obsidian/plugins/obsidian-vimrc-support/data.json`
  selects this file and disables JavaScript vimrc commands; keep those settings.

## User-visible contract

1. `[S` selects queue index 0; `]S` selects index `length - 1`. Read the current queue
   on each invocation and retain its existing order.
2. Absolute selection ignores cursor position and the previous anchor, including its
   handled keys, rank, and saved neighbors. Repeated `[S` or `]S` stays at that endpoint
   while the queue is unchanged. A one-entry queue lands on that entry for either key,
   even if already there. Do not emulate an endpoint with repeated relative jumps.
3. Reuse the existing landing guards. A stale endpoint triggers one queue reread and
   selection of that same requested endpoint in the new queue. An empty reread uses the
   existing empty notice; an unresolved stale target uses
   `Review queue changed — try again`. This also safely handles cache lag after stamping
   an endpoint: do not bypass content validation or invent another eligibility filter.
4. A successful endpoint jump installs the usual anchor at the selected entry. Lowercase
   navigation and Alt+Shift+F then continue from that location with their existing
   behavior. Failed or empty jumps must not replace the previous anchor.
5. Preserve the ordinary rank/total and tier-aware destination notice, lane action
   hints, legacy v3 fallback, missing-API notice, empty queue notice, and file-open
   failure handling. Endpoint jumps never report wrapping. Suppress the relative-walk
   boundary preamble for endpoint jumps: skipping to the last entry does not establish
   that commitments have been reviewed. Lowercase boundary notices stay as-is.
6. Expose both actions in the command palette. Only the requested Vim mappings are
   added; no additional default modifier hotkeys or count semantics are introduced.
   Existing lowercase mappings keep working.

## Implementation

1. In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`, add an explicit
   `endpoint: "first" | "last"` option to `planReviewJump()`. Handle recognized endpoint
   values immediately after the empty-queue check, before relative cursor/anchor logic.
   Return the existing jump shape with the selected entry, 1-based rank, full queue
   total, `wrapped: false`, and `originTier: null`. Keep the default relative path
   unchanged, including its legacy `stamped` option.
2. Thread that option through `jumpToDueTask()` on both initial planning and
   stale-target retry. Reuse the existing landing, anchor update, and notice builders.
   Gate the relative boundary preamble to relative jumps. Avoid duplicating the
   asynchronous landing implementation.
3. Register commands next to the existing review commands:
   - `jump-to-first-due-task`, named `Jump to first task due for freshness review`.
   - `jump-to-last-due-task`, named `Jump to last task due for freshness review`. Their
     callbacks invoke the shared jump method with the corresponding explicit endpoint.
     First/last must not accidentally use the current relative fallback, which maps
     forward to first and backward to last.
4. In the Bob vault's `obsidian_vimrc.md`, add alongside the existing review mappings:

   ```vim
   exmap bob_first_due obcommand bob-navigation-hotkeys:jump-to-first-due-task
   exmap bob_last_due obcommand bob-navigation-hotkeys:jump-to-last-due-task
   nmap [S :bob_first_due<CR>
   nmap ]S :bob_last_due<CR>
   ```

   Use the existing `obcommand` bridge; do not enable vimrc JavaScript or add another
   plugin-owned mapping for these same keys.

5. Update the navigation description in `bob-plugins/README.md` and the review ritual in
   `bob-cli/docs/freshness.md` to distinguish previous/next `[s`/`]s` from first/last
   `[S`/`]S`. Bump the navigation plugin manifest version and matching README version
   according to repository convention, using the current version at implementation time
   (1.65.0 was inspected). No memory-note edits are part of this plan.

## Validation and acceptance

Extend the existing review test suite with behavior-focused coverage:

- First/last selection from a middle entry, from either endpoint, from an unrelated note
  or absent editor, and with a conflicting saved anchor. Verify full-queue rank/total
  and no wrapping. Include a mixed tier queue so endpoints cannot accidentally be scoped
  to one tier.
- Empty and one-entry queues; repeated endpoint calls with the anchor produced by the
  previous successful landing. A still-due visited endpoint must never disappear merely
  because its key is in the anchor.
- Method-level coverage with stubbed API/landing: a stale first or last target followed
  by a changed queue reselects the requested endpoint; retry-empty, retry-stale, missing
  API, and non-stale landing failure preserve the anchor and return existing notices.
  Retry is bounded.
- A successful jump updates the anchor; subsequent lowercase steps continue from the
  selected location. Existing post-stamp/release behavior remains covered by the
  original tests. Include a stale stamped endpoint to ensure it cannot be landed on from
  outdated text.
- Destination notices retain v4 tier details and legacy v3 text; endpoint-to-ROTTEN does
  not emit a completion/boundary preamble. Existing relative boundary behavior remains
  covered.
- Capture `onload()` registrations using the existing harness pattern and invoke both
  new command callbacks to verify correct routing. Keep navigation-only tests equipped
  with write/stamp spies that fail if a jump attempts to mutate task content.

Run in bob-plugins:

```sh
node --test scripts/test-navigation-freshness.cjs scripts/test-navigation-hotkeys.cjs
npm test
npm run validate
git diff --check
```

Run `git diff --check` in each other modified repository. Inspect the vimrc
mapping-to-command chain for both uppercase and lowercase pairs; check exact case and
command IDs. No Rust implementation changes or Rust suite run are needed for the
prose-only bob-cli edit.

## Deployment and completion

After approval, implementation, and passing checks, run `bob plugins sync` as required
by bob-plugins' instructions. Use `--repo` with the path returned by
`sase repo open bob-plugins`, `--no-pull`, and `--plugin bob-navigation-hotkeys`; first
preview with `--dry-run`, then run the same scoped sync without it. Do not add `--force`
to bypass an unexpected dirty-destination refusal.

Include the plugin, vault mapping, and documentation changes in the appropriate SASE
finalization obligations. Preserve unrelated vault changes. Deliver the tracked vimrc
change through the existing git vault sync workflow described in
`docs/vault-git-sync.md`; do not re-enable Obsidian Sync (the vault AGENTS text about
that service is historical). Plugin deployment must precede use of the new mappings.

Reload Bob Navigation Hotkeys and the vimrc in Obsidian when a running UI is available.
Smoke-test `[S`, `]S`, repeats, then `[s`/`]s` from the landed endpoints, including a
cross-note landing. Confirm task text and freshness stamps are unchanged. If the UI
cannot be exercised from the implementation environment, report that limitation
explicitly alongside the automated results and deployment status; do not claim a UI
pass.
