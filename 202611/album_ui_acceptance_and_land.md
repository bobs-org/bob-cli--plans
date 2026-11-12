---
tier: tale
size: medium
title: Album UI acceptance proof and landing for bob-cli-k3.7.1
goal:
  Build and run the album real-Obsidian acceptance proof that the bob-cli-k3.7.1 plan
  assigns to its land agent, fix any defects it finds, deploy, then close bob-cli-k3.7.1
  and its parent phase bob-cli-k3.7.
proposed_by: bbugyi200.athena.bob-cli-k3.7.1.land
proposed_with: claude/opus
bead: bob-cli-k3.7.1
status: done
---

- **PARENT:**
  [202611/album_plugin_ui.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_plugin_ui.md)
- **BEAD:**
  [bob-cli-k3.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.7.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-k3.7.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-k3.7.1.land.md)
- **COMMITS:**
  - [48505cc](https://github.com/bobs-org/bob-cli--plans/commit/48505cc0413332a3cba361aab1ac5ba3f1b53aa8)
    — docs(plans): mark album plugin UI epic and acceptance plans done

# Album UI acceptance proof and landing

## Context and what is already settled

Epic **bob-cli-k3.7.1** ("Obsidian album cards, live chips, headers, and navigation",
plan `plan:202611/album_plugin_ui.md`) has all five phases closed. Their commits are on
bob-plugins master: `9c79123` (album-data), `bd6fc41` (owner-cards), `ed5e648`
(records-cards), `c2f6834` (album-header), and `2d10fca` (album-navigation). The land
agent has already done the following. Do not redo it.

- Verified the phase deliveries against the plan: shared `apiAlbum*` service, 3 s sync
  debounce, owner chips with peek and group jump, fold-after-12 record index, visited
  and watched_on age chips, `Name @<stem>/` copy, scoped `record` property hiding, album
  and record parent rules, and lifecycle wiring. The 129 focused album and navigation
  tests pass. The real-bob fixture test in `scripts/test-ledger-tools-albums-data.cjs`
  passes against current bob-cli master.
- Integrated drift. bob-cli `1174a17b` (bob-cli-kr.1) adds only additive album-list JSON
  fields, and the plugin decoder accepts them. Consuming those fields belongs to the
  follow-up epic **bob-cli-kr** (kr.3 reshapes the header into a collapsed schema
  summary). Do not build kr.3 behavior here. Assert the header as it ships today.
- Ran a normal `bob plugins sync` (no `-F`). All 10 plugins are synced.
- Triaged every phase `PROPOSED FOLLOW-UP:` and recorded the outcomes on bob-cli-k3.7.1.
  These known clean-base failures are **not** blockers and need no new beads:
  - `test-ledger-tools-task-tag-marks.cjs` "session toggle and onload wiring" (2 !== 1)
    is tracked by bob-cli-kw.
  - The load-sensitive 200-link Live Preview budgets under the full suite are tracked by
    bob-cli-ki (keep-moving surface) and bob-cli-ju (off-clock link chips).
  - The strict docs-check warning at `develop/plugins/validation.md:55`
    (`../bob-vim-yank-flash.md#verification`) is tracked by bob-cli-kk and bob-cli-kb.

The one piece of epic work still missing is the plan's "Integration and land-agent
acceptance" proof. Before starting, read that section with
`sase artifact read plan:202611/album_plugin_ui.md "<why>"`. `sase bead epic-symbols` is
currently empty for both bob-cli-k3.7.1 and bob-cli-k3.7.

## Repository discipline

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. All plugin paths below
are relative to that checkout. Edit `src/` fragments, never `main.js`, and keep each
hand-edited fragment at or below 1000 lines. Rebuild with `npm run build`. Read
`docs.md` with `/sase_memory_read` before any documentation edit, and read
`sase_artifacts.md` before registering artifacts. Use `/sase_monitor` for any command
that can run for minutes (full `npm test`, `just check`). Never run `just check-full`.

## Step 1: Album e2e script and offline wrapper

Add `scripts/obsidian-album-ui-e2e.cjs`. If it would be unwieldy, also add an
`scripts/obsidian-album-ui-e2e-lib.cjs` for fixture building and pure probe helpers.
Model both on `scripts/obsidian-bank-ui-e2e.cjs` and its lib, and on
`scripts/obsidian-notebook-ui-e2e.cjs`. Reuse their pieces: the scratch vault and
profile, the owned Obsidian process, the CDP client, the `-e/-o/-r/-s/-h` options,
real-bob discovery, and `Page.captureScreenshot`. The script must never open the
personal vault.

Build the fixture vault at run time with the **real bob binary** in a temporary
`BOB_DIR`, with a fixed `BOB_NOW` and a missing `BOB_CONFIG_FILE`. Author the album and
record notes, then let `bob albums sync` write the generated `[!albums]` and
`[!records]` bytes. Prefer a fresh build of the primary bob-cli checkout
(`cargo build --bin bob`; find the binary through the `target_directory` that
`cargo metadata` reports), because the installed `~/.cargo/bin/bob` can lag master.
Allow an override through `BOB_BIN`/`BOB_CLI_BIN`. Wire the in-Obsidian plugin to the
same binary the way the bank e2e does.

The fixture must include:

- a healthy grouped album (choice `status` group field) under an area owner, plus a
  project owner with its own album, so nested catalog rows exist;
- a later top-level group with more than 12 records and subgroups (`index_by`), and a
  first group that starts expanded;
- an empty album, an invalid album (broken `record:` schema), and an owner whose
  generated catalog is missing or stale after sync;
- a notebook neighbor under the same owner;
- records with valid `visited` and `watched_on` dates, one record with both, and one
  with an invalid or future date;
- an H1-less album;
- an unrelated non-album note that carries a `record` frontmatter property.

The offline `--self-test` path never launches Obsidian. It builds the fixture with real
bob when a binary is available. It asserts the generated callout structure, the
`bob albums list -f json` shape (index state objects, issues, groups, peek, and
`last_record`), and the probe helpers. If bob is unavailable, it skips that part with an
explicit message, like the real-fixture test in `test-ledger-tools-albums-data.cjs`. On
success it prints `offline self-test complete`. Add
`scripts/test-obsidian-album-ui-e2e.cjs` as a copy of the bank wrapper pattern (20 s
timeout) and register it explicitly in `package.json` `test:unit`, next to
`scripts/test-obsidian-bank-ui-e2e.cjs`.

## Step 2: Real-Obsidian run, screenshot review, and fixes

Run `xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs -o <outdir> -r <report.json>` on
athena. Pass `-e` with the Electron binary if the default is the CLI shim. Probe and
screenshot each of the following in **Reading view and Live Preview**. Cover light and
dark themes, and a narrow pane of about 375 px.

- Owner `[!albums]` card: count, lead tally, peek titles (as text), last-added age, and
  health chips. Check stale, missing, and invalid states, and that data is never shown
  as zero when unavailable. Check that chips wrap in the narrow pane.
- Lead-chip group jump by click and by keyboard: it opens the album, expands a folded
  target group, and scrolls to the group line.
- `[!records]` card: the first group is expanded; a later group with more than 12
  records is folded; fold choices survive a report refresh and stay independent per
  pane.
- Date chips with distinct visited and watched tooltips. Changing a record date in
  frontmatter refreshes the chip. A simulated day rollover updates the ages.
- Album header: breadcrumb links resolve to the real owner path, and the H1-less album
  still gets one header. The copy action puts exactly `Name @<stem>/` on the clipboard.
  On the invalid album the copy action is disabled and shows its reason. The schema card
  lists fields in declaration order with choice labels.
- Property hiding: the `record` property row is hidden only on album notes. It is
  visible on the unrelated note, and visibility comes back after a file switch.
- Report failure and retry: make the bob wrapper fail or emit malformed JSON, check the
  error state and its Retry button, then restore bob and retry to a healthy card.
- Cleanup on file switch and plugin unload: no leftover decorations, hidden rows, or
  scoped classes.
- Byte preservation: hash every fixture note before and after the presentation-only
  interactions (peek, fold, group jump, breadcrumb, copy, file switch, retry). The
  hashes must be identical.
- Debounced sync: make two quick frontmatter edits to a record, then count `albums sync`
  invocations through a logging bob wrapper. Expect exactly one sync, starting about 3 s
  after the last edit.

Open every screenshot with the image reader and inspect it. Fix any overlap, clipping,
focus, contrast, or visibility defect in the album fragments or `styles.css`, rebuild,
and rerun until the walkthrough passes and the screenshots look right. Any defect caused
by this epic's album UI is in scope and must be fixed here. For a genuinely unrelated
defect, use `/sase_new_task` and identify bob-cli-k3.7.1 as the proposer.

Register the JSON report and the key screenshots as SASE artifacts with
`sase artifact create`, following `sase_artifacts.md`, and keep their refs for the notes
in step 5.

## Step 3: Documentation integration

- bob-plugins `docs/develop/validation.md`: add an album paragraph next to the bank e2e
  paragraph. Name the offline wrapper and the
  `xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs` command. Use a link form that
  resolves both in the plugin docs and in the mounted bob-cli site. Do not copy the
  broken `../bob-vim-yank-flash.md#…` pattern at line 55.
- bob-plugins `docs/albums.md`: add a short Verification section if it fits the page
  budget.
- bob-cli `docs/records.md`: add one sentence in the albums section pointing readers to
  the Obsidian presentation at `[Album cards](reference/plugins/albums.md)`, in the same
  style as `docs/notebooks.md` links to its plugin page. Respect the docs word caps.
- Run `npm run docs:gen` when the inventory changes.

## Step 4: Gates and deployment

In bob-plugins, run `npm run build`, `npm run docs:gen`, `npm test` (through
`/sase_monitor`), `npm run validate`, and
`BOB_CLI_DIR=<primary bob-cli checkout> npm run docs:check`. Inspect changes to bundles,
inventory, and manifests, and keep only the intended generated updates. The known
clean-base failures listed above may remain only if they reproduce identically without
your changes. Record that evidence. In the primary bob-cli checkout, run `just check`.

Run `bob plugins sync` from the opened bob-plugins checkout. If it refuses, follow that
repo's `AGENTS.md` and `docs/develop/deployment.md`. Never pass `-F` on your own.
Confirm that `bob plugins list` reports bob-ledger-tools and bob-navigation-hotkeys as
synced.

## Step 5: Record and close out (final step)

1. Add a note to **bob-cli-k3.7** recording the implemented features, the exact checks
   and their results, the artifact refs (report and screenshots), the deployment
   outcome, and any clean-base failures with their tracking beads.
2. Run `sase bead epic-symbols bob-cli-k3.7.1`. Resolve every listed `--epic-symbol`
   entry: wire it up, privatize it, add a non-test pragma, or delete it. Re-key an entry
   only when a still-open later bead needs it.
3. Close the epic:
   `sase bead close bob-cli-k3.7.1 --note "<verification: phases verified, drift integrated, follow-ups triaged (bob-cli-kw new; +1 bob-cli-ki, bob-cli-ju; bob-cli-kk existing), album e2e results and artifact refs, gates, deployment>"`.
   If the close is rejected for leftover epic symbols, finish that cleanup and close
   again. Never use `--force` to make the close succeed.
4. Run `just symvision` in the primary bob-cli checkout and confirm the whitelist is
   clean.
5. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   that `sase bead read bob-cli-k3.7.1 -r "<why>"` shows
   (`plans/202611/album_plugin_ui.md` in the plans repo).
6. The parent bead **bob-cli-k3.7** is a **phase** of epic bob-cli-k3. Check that its
   description is fully met: hue-accented `[!albums]`/`[!records]` cards; live count,
   lead-group, peek, and freshness chips from `bob albums list -f json`; long-group
   folding; the album header with breadcrumb, add-record affordance, and schema card;
   and a debounced `bob albums sync`. Run `sase bead epic-symbols bob-cli-k3.7` and
   resolve anything listed. Then close only that phase with
   `sase bead close bob-cli-k3.7 --note "<what you verified>"`. Do **not** close
   bob-cli-k3 or any other ancestor or sibling phase; bob-cli-k3's own land agent owns
   it.
7. Finish through `/sase_final`, declaring every changed repository: bob-plugins,
   bob-cli, and the plans repo.
