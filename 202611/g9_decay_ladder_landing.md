---
tier: tale
size: medium
title:
  "Land the task-decay ladder: P-level rung labels, a passing live e2e, and the epic
  closeout"
goal:
  P-level keeps, cards, and Rotten Log entries name the configured level label (P1–P4),
  never the raw [priority::] value; the real-Obsidian decay-ladder e2e passes live on
  athena with reviewed light/dark card screenshots recorded in docs/decay-ladder.md; and
  epic bob-cli-g9 is closed.
proposed_by: bbugyi200.apollo.bob-cli-g9.land
proposed_with: claude/opus
bead: bob-cli-g9
create_time: 2026-11-02 21:59:34
status: wip
---

- **PARENT:**
  [202611/task_decay_ladder.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_decay_ladder.md)
- **BEAD:**
  [bob-cli-g9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-g9/README.md)

# Plan: land the task-decay ladder (epic bob-cli-g9)

## Context

Epic `bob-cli-g9` ("Task Decay: one ladder, two stays per rung, one card") has all seven
phases closed. Its plan is `plan:202611/task_decay_ladder.md`; run
`sase bead read bob-cli-g9 -r "<why>"` to see the PLAN path, DECISIONS, and notes. The
land agent's verification found two pieces of unfinished epic work. Both break the
epic's acceptance, so this tale fixes them and then closes the epic.

The work spans three repos:

- **bob-plugins** (linked repo): navigation is `plugins/bob-navigation-hotkeys`, ledger
  is `plugins/bob-ledger-tools`. Open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first.
- **bob-cli** (this repo): docs only.
- **athena** (a tailnet host): only to run the live e2e. Read
  `sase memory read tailnet.md -r "<why>"` for SSH access.

### Defect 1: P-level rungs are named by the priority value, not the level label

`rottenKeepRungForTarget(raw, entry, priorityName)` in navigation
`src/195-rotten-keep.js:4` returns the raw `[priority::]` value for a `[ ]` task with a
priority. Bryan's ladder (`~/.config/bob/config.yml`, `properties: - name: priority`)
uses labels `P1`–`P4` with values `high`, `medium`, `low`, and `lowest`. The plan's
grammar says `<Rung>` is "the configured level label" (`kept 1× at P2`, `Kept 2× at P2`,
`🍂 P2 → P3 · after 2 keeps`). Today the code produces:

- in the live e2e, the P4 keep logged `bottom of the ladder · kept 1× at lowest`;
- the keep prompt asks "Why keep it at high?";
- the card title reads "Kept 2× at high";
- `052-decay-ladder-step-planner.js` mixes the two. It passes
  `fromLevel: { label: rung }` (the value) and `toLevel.label` (the label), so the
  detail line, the Schedule Log reason, and the `🍂` step entry read `high → P2`. The
  stay entry reads `stayed at high`.

Unit tests missed this because their fixtures use ladders whose values equal their
labels.

Call sites to fix:

- `195-rotten-keep.js:107`, inside `classifyRottenKeepTarget`, which has no property in
  scope today;
- `195-rotten-keep.js:331` (the batch `rungForLine` map) and `:425`;
- `530-plugin-freshness-refresh-and-decay.js:214` (card context) and `:428` (the
  revalidate rung comparison);
- the `052` P-level branch, which looks the current level up by value (keep that) and
  must use the current level's label as the "from" rung;
- the counted-keep callers in `520-plugin-lane-links-and-review.js`
  (`refreshTaskFreshnessOnTasks`) and `532-plugin-freshness-link-refresh.js`, which call
  `classifyRottenKeepTarget`.

The land agent checked the vault (`~/bob`). It holds no `× at high|medium|low|lowest`
entries yet, only WIP and Next entries, so no migration or legacy parsing is needed.

### Defect 2: the real-Obsidian e2e has never passed

Phase `live-e2e` added bob-plugins `scripts/obsidian-decay-ladder-e2e.cjs` (plus
`-lib.cjs` and `test-obsidian-decay-ladder-e2e.cjs`). It only ran `--self-test`, because
apollo has no Obsidian. The land agent ran it live on athena (Obsidian 1.15.0 at
`/snap/bin/obsidian`, `xvfb-run`, bob-plugins `8275198`). **11 checks passed and 25
failed.**

What passed:

- plugins loaded, the note opened, and the Tasks cache was ready;
- each rung's **first** keep (WIP, Next, P4) counted and logged correctly;
- no Work Log was written;
- `batch-skip-no-write`.

What failed:

- every second keep (`fresh` advanced to the new day while `[keeps::]` stayed at 1, with
  no new entry), so no card ever opened and every card, step, stay, drop, and undo check
  failed;
- `ready-keep-1` (stamped but uncounted);
- `batch-skip-notice` (captured notice text was `""`);
- no screenshots were captured.

Likely harness cause, to confirm first: navigation's `readFreshnessQueue` reads ledger
`api.queue()`, which is built from the Tasks plugin cache. That cache only updates after
Obsidian autosaves the editor buffer and re-indexes the file. The harness sleeps 400 ms
after each write. The next gesture's `matchFreshStampExactEntry` then misses the exact
row in two ways:

- the raw text is stale;
- line numbers shifted after the Rotten Log child lines were inserted.

So the keep takes the documented "unresolved cache match → uncounted" path. That
explains the second-keep failures. It also explains `ready-keep-1`: it ran right after a
Next keep added log lines above it. P4 keep 1 passed only because earlier 10 s card
timeouts had given the cache time to catch up.

Two more harness points to check:

- `pinToday` overrides `ledger.apiFreshnessConfig` with a hand-built object. Confirm its
  shape matches what `090-freshness-config.js` produces since ledger-mirror (per-rung
  `decay` fields), or drive ledger from the scratch `bob/config.yml` instead.
- The batch-skip notice may be gone before `lastNotice` reads `.notice` elements.

## Steps

1. **Fix the P-level rung label (navigation).**
   - Give `rottenKeepRungForTarget` the loaded priority property, not just its name.
   - For a `[ ]` row whose priority value matches a level, return that level's
     normalized `label`.
   - With no loadable ladder or an unknown value, keep today's fallback: the raw value,
     or `Ready` when there is no priority field. The card's step is already unavailable
     in that case.
   - Thread the property to every call site listed under Defect 1. Load it once per
     gesture through the existing `loadFreshnessDecayPrioritySnapshot()` and never read
     `this.config`.
   - Make the `052` P-level step use the current level's label as the "from" rung, so
     the title, detail, Schedule Log `🎲` reason tail (`kept 2× at P1`), and `🍂` entry
     all agree.
   - Make sure `freshnessDecayAllowanceFor` still maps every P-level label to the Ready
     allowance.
   - Add navigation unit tests with a Bryan-style ladder (labels P1–P4, values
     high/medium/low/lowest). They must cover:
     - a counted keep on `[priority:: high]` logs `… · kept 1× at P1`, and its prompt
       asks "Why keep it at P1?";
     - the card is titled `Kept 2× at P1`;
     - Enter steps to `medium` with `🍂 P1 → P2 · after 2 keeps` and a Schedule Log tail
       `kept 2× at P1`;
     - Stay logs `stayed at P1 after 2 keeps`;
     - `[priority:: lowest]` rolls again with `🍂 rolled again at P4 · after 2 keeps`;
     - a counted batch logs label tails.
   - Check that ledger-tools' Rotten Log tallies and tooltips show `P1 2` style keys.
     The shared grammar fixture `scripts/fixtures/rotten-log-grammar.json` already uses
     labels. Add a ledger test only if ledger derives a rung name itself.

2. **Make the e2e harness wait for the cache.**
   - After every write (keep, card answer, undo), flush the editor (for example
     `await view.save()`).
   - Poll until ledger `api.queue()` (or the Tasks cache) holds the task's current
     editor line before the next gesture. Do this inside `keepOn`, `openCardOn`, and
     after each `cardKey` / `undoEditor`.
   - Fix notice capture so `batch-skip-notice` reads the notice the batch actually
     showed. For example, patch `Notice` or observe `.notice` insertions from before the
     gesture.
   - Add `-log` assertions for the P4 keeps and roll that expect `at P4`.
   - If a failure remains after the harness waits correctly, treat it as a product bug.
     Fix it in navigation or ledger with a focused unit test. Do not loosen the e2e
     assertion.
   - Keep `--self-test` green.

3. **Run the live e2e on athena.** Apollo has no Obsidian. Never use or change athena's
   own `~/projects/github/bobs-org/bob-plugins` checkout, and never open the live vault.
   - Your changes are uncommitted, so copy the working tree, not `HEAD`:
     `D=$(ssh athena mktemp -d /tmp/bob-decay-e2e.XXXXXX)`, then from the bob-plugins
     checkout run
     `tar --exclude=.git --exclude=node_modules -cf - . | ssh athena "tar -xf - -C $D"`.
   - Run
     `ssh athena "cd $D && mkdir -p shots && xvfb-run -a node --experimental-websocket scripts/obsidian-decay-ladder-e2e.cjs --out shots -r report.json"`.
     Non-interactive `node` on athena is v20.19, which needs `--experimental-websocket`
     for the CDP client.
   - Pass criterion: every check PASS and exit 0, two runs in a row.
   - `scp` the `shots/*.png` and `report.json` back to a fresh local temp dir.
   - Look at each WIP, Ready, and P4 card screenshot in light and dark. Check the card
     anatomy: title `Kept 2× in/at …`, the focused recommended Enter row naming its
     destination, Stay, Drop, and a footer built from the rows; lane cards have no Less
     often or 1–4 rows.
   - Register the PNGs with `/sase_artifact` so the close note can cite them. Do not
     commit binaries.
   - Remove the athena temp dir afterwards.

4. **Validate bob-plugins.**
   - `npm run build`, the touched decay suites (decay planner, decision-card, handlers,
     keep, rotten-keep, rotten-log, keep-counting, refresh-modal, review-journal, ledger
     mark and keeps), then `npm test` and `npm run docs:check`.
   - These full-suite reds are known, pre-existing, and tracked elsewhere: bob-cli-eh,
     bob-cli-ff, bob-cli-et, bob-cli-f1, bob-cli-fz, bob-cli-ej, bob-cli-f8, and the
     relation_successor vectors drift noted on bob-cli-fs. Confirm any other red also
     fails on a clean tree before calling it pre-existing.
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.73.0 to 2.73.1, and
     ledger-tools (1.95.0) only if you touched it.
   - Update `docs/bob-navigation-hotkeys.md` where it names P-level rungs.
   - Run `bob plugins sync` from the linked checkout.

5. **Update bob-cli docs.** Read `sase memory read docs.md -r "<why>"` first.
   - In `docs/decay-ladder.md`, replace "This host has no Obsidian…" with the live
     result:
     - date, host athena, Obsidian version, and the `xvfb-run` command;
     - the pass count of both runs and that the screenshots were reviewed in both
       themes, with what you checked;
     - any harness fixes, and the P-level label fix.
   - Keep the "Pending (Bryan, in Obsidian)" list down to the human-only checks: the Mac
     smoke and iOS glyph rendering.
   - Add one sentence to `docs/freshness.md` §2a and `docs/reference/vault-format.md`
     where the Rotten Log grammar describes `<Rung>`: P-levels use the configured level
     **label**, never the stored `[priority::]` value.
   - Run `just docs-check`, then `just check`. Do not run `just check-full`.
   - Never edit `Cargo.toml`'s version or `CHANGELOG.md`.

6. **Close out epic bob-cli-g9.** This is the final step and it runs in this same turn.
   - Run `sase bead epic-symbols bob-cli-g9`. It was empty at landing. If it now lists
     entries, resolve each one: wire it up, privatize it, add a non-test pragma, or
     delete it per the Symvision epic-whitelist policy. Re-key a Justfile line only to a
     still-open bead that needs the exemption.
   - Run `sase bead close bob-cli-g9 --note "<verification>"`. The note must cover:
     - the P-level label fix and its tests;
     - both live athena e2e runs (pass counts) and the screenshot artifact refs;
     - the validation commands and results;
     - the follow-up triage already recorded on the epic by the land agent: +1 on
       bob-cli-eh/ff/et/f1/fz/ej/f8, a DISCOVERED ISSUE on bob-cli-fs for the vectors
       drift, and g9.6#1 handled here as epic work;
     - integration: commits since the epic started (capture clock tags, relation
       successors, gcal) do not conflict, and capture's `plan_task_link` raise already
       stamps fresh, which clears `[keeps::]`.
   - Never use `--force`. If the close is refused, fix the cause and close again.
   - Run `just symvision` and confirm the whitelist is clean.
   - Set `status: done` in the frontmatter of the epic's plan file (the PLAN path from
     `sase bead read bob-cli-g9`, `plan:202611/task_decay_ladder.md` in the plans
     sidecar).
   - bob-cli-g9 has no `parent_bead`, so nothing further is closed.
   - In the final response, tell Bryan that athena (chezmoi still at `baf49514`) and the
     Mac need `chezmoi update -a --force` to pick up the final per-rung decay block from
     chezmoi `4854d40b`. Apollo already has it.

## Acceptance

1. On Bryan's ladder, a P-level keep, prompt, card title, step detail, Schedule Log
   tail, and every Rotten Log entry name `P1`–`P4`, never
   `high`/`medium`/`low`/`lowest`. This is pinned by unit tests.
2. `xvfb-run -a node --experimental-websocket scripts/obsidian-decay-ladder-e2e.cjs`
   passes every check twice in a row on athena. The WIP, Ready, and P4 cards were
   reviewed in light and dark.
3. `docs/decay-ladder.md` records the live pass. `just docs-check` and `just check`
   pass. bob-plugins builds and `docs:check` passes, and the deploy is synced.
4. Epic bob-cli-g9 is closed with a verification note, `just symvision` is clean, and
   its plan file reads `status: done`.
