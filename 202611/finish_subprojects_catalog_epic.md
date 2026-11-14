---
tier: tale
size: medium
title: Finish and close the sub-projects catalog epic (bob-cli-ld)
goal:
  Every remaining bob-cli-ld obligation is done or explicitly handed to an existing
  tracker, every gate this epic touched is green, and bob-cli-ld.9.1.3, bob-cli-ld.9.1,
  bob-cli-ld.9, bob-cli-ld.10, and bob-cli-ld are closed with evidence.
proposed_by: bbugyi200.athena.1cd
proposed_with: claude/opus
create_time: 2026-11-14 05:27:49
status: wip
---

# Plan: Finish and close bob-cli-ld

Bryan asked for the remaining bob-cli-ld work to be finished and the epic closed. Read
the epic first: `sase bead read bob-cli-ld -r "<why>"`, then `bob-cli-ld.9.1.3`,
`bob-cli-ld.9.1`, and `bob-cli-ld.10`. The epic plan is
`plan:202611/subprojects_catalog_section.md`, and the card's child epic plan is
`plan:202611/projects_catalog_card.md`. Read both with `sase artifact read`. This tale
is the epic's land step.

## Where things stand (verified 2026-11-14 on master 5b1b6d59 / bob-plugins 415a33d)

- **Phases.** `ld.1`–`ld.8` are closed. `ld.9` became child epic `ld.9.1`. Its phases
  `9.1.1` and `9.1.2` are closed. `9.1.3` is in progress. `ld.10` was never worked.
- **What 9.1.3 has done.** It landed in bob-plugins as 703014d, fe7ffb7, and 415a33d:
  - the card and its real-Obsidian acceptance, passing 43/43, with screenshot artifacts
    attached to `ld.9.1`;
  - the "Projects catalog card" section in `docs/bob-ledger-tools.md`;
  - the `docs/develop/validation.md` entry;
  - the Bob Ledger Tools 1.116.0 deploy.

  It stays open only because of gates (its note #1).

- **No agents are running.** No bob-cli-ld agent is running, and the land agents for
  `bob-cli-ld` and `bob-cli-ld.9.1` are gone. Bryan explicitly asked for this, so this
  tale closes those phase and epic beads itself, overriding the usual "land agent closes
  the epic" convention.
- **Memory is done.** The epic's DECISIONS stay final:
  - `memory_decision_record = no`: write no decisions strand.
  - `memory_capture_glossary = yes`: already done in f9b61593.
    `glossary:project-note-capture` now says "the same batch lists it in the parent's
    `## Projects` catalog".

  No other memory note states the retired line. **This tale makes no memory edits.**

- **The real vault is already migrated.** Vault commit `b32bbc026`, "projects: sync
  lifecycles and catalogs (21 notes)" (2026-11-13 16:50 -0500), installed the catalogs.
  Read-only checks with a master build:
  - `bob projects sync -d -b ~/bob` prints `✓ projects catalogs are current`.
  - `bob projects list -b ~/bob -f json` reports `ok`, 21 catalogs all `current`,
    `findings: []`, and 100 projects.
  - Legacy `🧩 **Sub-projects:**` lines survive only in `_conflicts/` and `done/`
    copies. That is expected, because legacy readers stay forever.

  The epic's "dry-run preview before migration" step is therefore replaced by
  post-migration verification (Step 4).

- **The CLI is already updated.** `bob projects sync` help and output already describe
  the catalog (`src/native/projects/mod.rs`, `sync_engine.rs`). The CLI needs no change.
- **Bug bob-cli-la is already fixed.** It reports that `^prj` drawer selectors
  `#notebooks` and `#albums` write into generated catalogs. `ld.1`'s commit 2709046a
  fixed its root cause: `is_structural_h2_key` now consults
  `host_catalog::GENERATED_CATALOGS`. The bead is still `ready`.

### Gate failures today and who owns each

| Gate                   | Failure                                                                                                                                            | Cause                                                                                                                                               | Disposition                                                               |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| bob-plugins `npm test` | `test-ledger-tools-task-link-marks-live.cjs` "the lifecycle calls one setupMarkFamilies and stays within the fragment limit"                       | `src/170-plugin-lifecycle.js` reached 1000 newline-terminated lines in 4e197dd (`bob-cli-kx.4`, active Gmail epic), so `split("\n").length` is 1001 | **Fix here** (Step 1a), and note it on `bob-cli-kx`                       |
| bob-plugins `npm test` | `test-ledger-tools-off-clock-link-chips.cjs` 200-link build took 63 ms (limit 35 ms)                                                               | wall-clock assertion under full-suite load                                                                                                          | Tracked by `bob-cli-ju` (siblings `bob-cli-ki`, `bob-cli-km`); do not fix |
| docs-check             | `docs/reference/json-outputs.md` 6025/6000                                                                                                         | `ld.7` (7cd57603) took it from 5989 to 6025                                                                                                         | **Fix here** (Step 2c)                                                    |
| docs-check             | `reference/plugins/bob-ledger-tools.md` 6260/6000                                                                                                  | already 6115 from bank work; `ld.9.1.3` added +145                                                                                                  | **Fix here** (Step 1b); also satisfies `bob-cli-il`                       |
| docs-check             | `reference/plugins/bob-navigation-hotkeys.md` 6019/6000                                                                                            | copy-card work plus `ld.3` (+27)                                                                                                                    | **Fix here** (Step 1c); the nav part of `bob-cli-jz`                      |
| bob-cli `cargo test`   | `native::gcal::cli::tests::sync_args_read_last_n_days_aliases_and_default_to_none`                                                                 | deterministic on master; unrelated                                                                                                                  | `bob-cli-l2`                                                              |
| bob-cli `cargo test`   | `native::subprocess_runner::tests::run_bounds_pipe_drain_when_detached_grandchild_holds_output`                                                    | load flake                                                                                                                                          | `bob-cli-kl`                                                              |
| bob-cli `cargo test`   | `capture::complete_themes::{configured_shell_describes_never_used_and_unconfigured_create, theme_recent_shell_description_and_quoted_replacement}` | 150 ms completion deadline under load                                                                                                               | `bob-cli-g8`                                                              |

**Closing bar.** Every failure this epic caused, plus the deterministic lifecycle
failure, gets fixed. Each remaining red test must be one of the tracked rows above. Each
tracked flake must pass when rerun in isolation, with that evidence recorded on its
tracker. Do not fix the tracked beads here.

Measure word counts the way the docs lint does. Run this from the bob-cli root:

```bash
uv run --project tools/docs python -c 'import sys; from bob_docs.markdown import count_words; print(count_words(open(sys.argv[1]).read()))' <file>
```

## Step 1 — bob-plugins (open it with `/sase_repo`; read its `AGENTS.md`)

### 1a. Bring the lifecycle fragment back under the fragment limit

- **Where the code lives.** `plugins/bob-ledger-tools/src/170-plugin-lifecycle.js` opens
  `class BobLedgerToolsPlugin extends Plugin {`. It holds `onload()` from line 2 and
  `onunload()` from about line 682 to the end of the class.
- **The move.** Move one cohesive block out into a new fragment listed right after
  `170-plugin-lifecycle.js` in `src/fragments.json`. A good candidate is the bulk of
  `onunload()`'s teardown into a prototype method such as
  `BobLedgerToolsPlugin.prototype.teardownLedgerToolsSurfaces`, with `onunload()`
  calling it. Follow how sibling fragments add prototype methods.
- **Target.** Leave real headroom: about 950 lines or fewer, not 999. The test's other
  assertions must still hold: exactly one `this.setupMarkFamilies();`, and none of the
  listed per-family setup calls inside the lifecycle fragment.
- **Behavior.** Unchanged. The teardown order is preserved exactly.
- **Build and test.** Run `npm run build`; never hand-edit `main.js`. Then run the
  focused test file.
- **Record the cause.** Add a note on `bob-cli-kx`: "DISCOVERED ISSUE: 4e197dd (`kx.4`)
  pushed `170-plugin-lifecycle.js` to the fragment limit; fixed in <sha> during the
  bob-cli-ld land step." Do not create a task bead for it.
- **Version.** A behavior-neutral refactor needs no manifest bump (precedent: 415a33d).

### 1b. `docs/bob-ledger-tools.md` to at most 5700 words

It is mounted as `reference/plugins/bob-ledger-tools.md`.

- **Target.** 5700 or fewer, not just under 6000. That leaves the ≥300-word headroom
  `bob-cli-il` asks for.
- **How to trim.** Condense duplicated or wordy hand-written prose. The generated facts
  and commands blocks are not counted. Never drop a behavior, default, keybinding,
  toggle name, or contract link.
- **Extracting a section.** If trimming alone cannot get there, extract one coherent
  section into its own page in bob-plugins `docs/`. The precedent is `albums.md`, linked
  from the intro. Update every inbound link and anchor in bob-plugins and bob-cli
  (`grep -rn 'bob-ledger-tools.md#'`). Docs memory requires preserved published URLs for
  IA moves, and this approved plan covers that one extraction.

### 1c. `docs/bob-navigation-hotkeys.md` to at most 5900 words

Use the same rules as 1b.

### 1d. bob-plugins gates and deploy

Run, in order:

- `npm run build`
- `npm test`
- `npm run validate`
- `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`

The only allowed `npm test` reds are the timing tests tracked by `bob-cli-ju`,
`bob-cli-ki`, and `bob-cli-km`. Rerun each red file alone with
`node --test scripts/<file>.cjs`; it must pass. Then run `bob plugins sync`. If it
refuses an overwrite, follow `AGENTS.md`: no `-F`. Commit the bob-plugins changes (for
example `fix(ledger-tools): move teardown out of the lifecycle fragment` and
`docs(plugins): bring ledger-tools and navigation references under budget`).

## Step 2 — bob-cli

### 2a. Remove ld-introduced dead-code allowances (epic rollout rule 6)

- **Which ones.** The wiring phase (`ld.7`) left
  `#[cfg_attr(not(test), allow(dead_code))]` on 4 items in
  `src/native/projects/catalog.rs` and 5 in `src/native/projects/catalog_rows.rs`.
  Several are now used in production code (for example, `render_row` at `catalog.rs`
  ~857).
- **The fix.** Remove all nine. If
  `cargo clippy --workspace --all-targets --all-features` then flags an item as unused
  outside tests, make that item `#[cfg(test)]`, or delete it if nothing uses it. Do not
  restore the allow.
- **Leave alone.** The two in `src/native/host_catalog/region.rs` predate this epic.

### 2b. Docs sweep: retire "Sub-projects line" statements

**The rule.** `bob projects sync` no longer writes a `🧩 **Sub-projects:**` line. It
retires legacy lines into the host's `## Projects` / `> [!projects]` catalog. Readers
still recognize the legacy line forever.

- If a sentence describes sync writing, anchoring, or moving that line, or describes
  readers reading only it, rewrite it to the catalog.
- If it describes a recognizer, mark it legacy instead of deleting it.

Verify each claim against the code before rewriting it: `src/native/projects/*`, and the
plugin readers in task-status-cycler `089a-project-catalog-rows.js` and nav.

Known stale spots on master:

- **`docs/task-dependencies.md`**
  - §2.1 Position (~line 72): drop the "On `^prj`, the machine-owned Sub-projects line
    follows…" bullet, or mark it legacy.
  - Managed-line table row `🧩 **Sub-projects:**` (~119): "Writer" becomes none (legacy;
    `bob projects sync` retires it into the `## Projects` catalog). "Placement" becomes
    legacy-only. Keep "Recognised by".
  - Rows ~120/122/123: change "(and Sub-projects on `^prj`)" to legacy wording.
  - Nav conversion row ~521.
  - MC7, MC8 (sync no longer places a marker; legacy lines are retired), MC15, MC16, and
    MC19.
  - The ~1107 close-group text and SO24: say what the cycler actually counts (catalog
    with legacy fallback, per the `SUBOPEN` definition at ~2172).
  - PP6: catalog row states plus the legacy fallback.
  - PDS14: "marker states" becomes catalog rows.
  - The §13 project example (~3845) and its structure bullet (~3858).
  - The §14.5 child-order tree (~4640) and the "`bob projects sync`. Sub-projects
    anchors…" bullet (~4652).
  - BR8: order Depends-On then BLOCKS. Sync writes no Sub-projects line, and
    `bob projects sync --dry-run` then changes nothing.
- **`docs/task-relations.md`**: the child-order tree (~226), the Slots and "Existing
  lines are never moved" bullets (~233–245), and LW7 (~688). The LINKS anchor on a
  Sub-projects line is legacy-only.
- **`docs/projects.md`**
  - ~221 "sub-projects are read from the marker line" becomes the parent's catalog, with
    legacy-marker fallback.
  - The ~354 and ~455 nav conversion sentences: the skipped child is a legacy line.
  - The "Typical action output" block (~1425–1440): replace the
    `added/updated/removed [[child]] … on ^prj` and `updated sub-projects on ^prj` lines
    with real current output. Take it from `SyncEvent`'s display in
    `src/native/projects/sync_engine.rs`, or run `bob projects sync -d` on a scratch
    fixture vault.
- **`docs/notebooks.md`** ~271: "never lists a notebook in a `## Projects` catalog".
- **`docs/pomodoro-homes.md`** ~83: "Sub-projects lines" becomes "Projects catalogs".
- **`docs/keep-moving-marks.md`**
  - ~281: "whose Sub-projects marker lists N" becomes "whose `## Projects` catalog lists
    N".
  - Leave the dated 2026-11-11 record (~387) as history.
- **`docs/task-link-marks.md`** (~81, 87, TC5, TC20) and
  **`docs/guides/see-task-mentions.md`** (~67): the recognizers stay. Say "legacy
  Sub-projects rows" where it reads as a live writer.
- **Leave alone**:
  - `docs/develop/decisions.md` and `docs/reference/glossary.md`, which are generated
    from memory;
  - concept-only uses of the word "sub-projects" (getting-started, stats,
    task-status-hooks, capture §routing, glossary-style prose);
  - `docs/reference/vault-format.md` and `docs/capture.md` ~6025, which are already
    correct.

Finish with `rg -n -i 'sub-projects (line|marker)|Sub-projects:\*\*' docs/`. Every hit
must be a legacy or recognizer statement, or generated.

Do not grow these pages past their tier caps. Keep the edits neutral to slightly shorter
in word count.

### 2c. `docs/reference/json-outputs.md` to at most 5900 words

Condense prose without dropping any field or contract.

### 2d. bob-cli gates

- **Docs.** Run `just docs-gen`, then `just docs-check` with bob-plugins mounted from
  the opened checkout (`BOB_DOCS_BOB_PLUGINS_DIR=<bob-plugins path>`). It must pass with
  zero budget errors.
- **`just check`.** Must be green except for the tracked rows in the table above. It can
  exceed one tool timeout, so run it in the foreground with a large timeout, or run
  `cargo fmt --check`, `cargo clippy …`, and `cargo test --workspace --no-fail-fast`
  separately.
- **Rerun tracked failures alone.** Use `cargo test --lib <name>` and
  `cargo test --test cli capture::complete_themes`. Flakes must pass alone; `l2` stays
  deterministic.
- **Commit.** For example, `docs(projects): retire Sub-projects line statements for the
  ## Projects catalog`and`refactor(projects): drop stale dead-code allowances`. Never
  edit the `Cargo.toml` version or `CHANGELOG.md`. Nothing here is a contract break, so
  use no `!`.

## Step 3 — verify and close `bob-cli-la`

In a scratch git vault, reproduce `bob-cli-la`'s steps with the workspace build, for
example `BOB_DIR=<scratch> bob capture "drawer note text @proj+prj#notebooks"`. Do the
same for `#albums` and `#projects`.

- **If every case is refused** (or never treated as a drawer) and the generated
  section's bytes are unchanged, close it:
  `sase bead close bob-cli-la --note "Fixed by 2709046a (bob-cli-ld.1): …"`. Include the
  exact refusal codes observed.
- **If any case still writes into a generated section**, add a note with the
  reproduction and leave the bead open. Do not fix it here.

## Step 4 — vault verification note on `bob-cli-ld.10`

Never run a real sync. Run only these read-only commands with the workspace build:
`bob projects sync -d -b ~/bob`, and `bob projects list -b ~/bob -f json`. Then add a
`sase bead note bob-cli-ld.10` for Bryan's review. It records:

- the migration commit `b32bbc026` and its 21 notes;
- the host count from `catalogs`, split by kind (area or project);
- the catalog states;
- `findings` (expected `[]`);
- the dry-run result (expected `✓ projects catalogs are current`);
- that legacy lines remain only in `_conflicts/` and `done/`.

Mention any host that looks surprising, for example `mac_inbox` (an area with one open
child). If a check disagrees with these expectations, record the exact output instead.

## Step 5 — follow-ups and closures

1. **Triage every `PROPOSED FOLLOW-UP` note** on the epic's phase beads. Each one maps
   to an existing tracker:
   - Gmail clippy: `bob-cli-kx`; the clippy errors are already fixed on master.
   - Docs budgets: fixed in this tale.
   - Timing flakes: `ju`, `ki`, `km`.
   - Library reds: `l2` and `kl`.
   - `complete_themes`: `g8`.

   Use `/sase_new_task` before creating anything. Expect only `+1` corroborations with
   this run's evidence. Each tracker gets at most one `+1` from this agent; use
   `sase bead note` for anything more. Create no new beads unless a failure matches no
   tracker.

2. **Close `bob-cli-il`** when Step 1b ends at 5700 words or fewer. The note gives the
   before and after counts.
3. **Check `bob-cli-jz`.** Its other pages are bob-mac-capture's `capture-panel.md`,
   `capture-previews.md`, and `keyboard.md`. If none is over the cap in a bob-cli
   `just docs-check` that mounts bob-mac-capture (open it with `/sase_repo`), close jz
   with the counts. Otherwise add a note that the navigation page is fixed and list the
   pages still over.
4. **Close the epic's beads, in order, each with a `--note`** naming its commits and the
   gate evidence:
   - `bob-cli-ld.9.1.3`: acceptance artifacts already attached; gates now green except
     tracked flakes.
   - `bob-cli-ld.9.1`, then `bob-cli-ld.9`.
   - `bob-cli-ld.10`: docs-sweep commits, memory already done in f9b61593, decision
     record declined by `memory_decision_record = no`, and the Step 4 vault note.
   - Finally `bob-cli-ld`.

   Closing never cascades. If a close is refused, read the named unclosed beads and fix
   the cause; never use `--force`.

## Verification

- **bob-plugins:**
  - `npm run build:check`, `npm test`, `npm run validate`, and `npm run docs:check`
    pass, except for the tracked timing flakes, which pass alone;
  - `bob plugins sync` succeeds without `-F`;
  - `170-plugin-lifecycle.js` is at about 950 lines or fewer;
  - the ledger-tools page is at 5700 words or fewer, and the navigation page at 5900 or
    fewer.
- **bob-cli:**
  - `just docs-check` passes;
  - `just check` is green apart from the `l2`/`kl`/`g8` rows, with the flakes passing
    alone;
  - `json-outputs.md` is at 5900 words or fewer;
  - the `rg` sweep shows only legacy, recognizer, or generated hits;
  - none of the nine ld-introduced dead-code allowances remain.
- **Beads:**
  - `sase bead read bob-cli-ld -r "<why>"` shows the epic and all phases closed;
  - `bob-cli-la` and `bob-cli-il` are closed or annotated per Steps 3 and 5;
  - `bob-cli-kx` carries the lifecycle note.
- **Vault:** unchanged by this tale (`git -C ~/bob log` shows no commit from it).

## Out of scope

- Fixing the tracked flakes and `bob-cli-l2`.
- Any memory edit.
- Any real `bob projects sync`, `bob nightly`, or capture against `~/bob`.
- Bob Mac Capture code changes.
- New catalog behavior.
