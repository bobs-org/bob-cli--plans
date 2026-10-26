---
tier: tale
size: small
title: "Land Bob command mode: make `:` work on every host, then close bob-cli-ce"
goal:
  Enable the deployed bob-command-mode plugin in the vault so that `:` opens the bob
  command sheet instead of doing nothing. Deploy it on the MacBook when reachable, fix
  the one epic-caused test fragility, sharpen the troubleshooting docs, and close epic
  bob-cli-ce.
proposed_by: bbugyi200.athena.bob-cli-ce.land
proposed_with: claude/opus
bead: bob-cli-ce
status: done
---

- **PARENT:**
  [202610/bob_colon_command_mode.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_colon_command_mode.md)
- **BEAD:**
  [bob-cli-ce](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ce/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ce.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ce.land.md)
- **COMMITS:**
  - [86f9e61](https://github.com/bobs-org/bob-plugins/commit/86f9e613ff235c8f8890419dca080e209a946117)
    — fix(command-mode): drop inherited NO_COLOR in test runner, add \`:\`
    troubleshooting

# Land Bob command mode (epic `bob-cli-ce`)

This is the landing tale for epic `bob-cli-ce`. The land agent verified all 11 phases
and their commits. The lander has already triaged every proposed follow-up and recorded
the outcomes on the epic as a `LANDER FOLLOW-UP TRIAGE` note, so do not re-triage them.
Two pieces of epic work remain, plus the closeout.

## Why `:` does nothing (Bryan's epic note #1)

The lander confirmed the root cause on athena:

- The vault vimrc (`~/bob/obsidian_vimrc.md`, lines 53–54) maps `:` correctly:
  `exmap bob_command obcommand bob-command-mode:open` and `nnoremap : :bob_command<CR>`.
  - Obsidian 1.14.5 bundles CodeMirror Vim in `lib/codemirror/vim.js`.
  - Its `doKeyToKey` replays the noremapped `:` into the default ex prompt through
    `virtualPrompt`, then submits `bob_command` on `<CR>`.
  - The other `nnoremap X :bob_*<CR>` maps use the same path.
- obsidian-vimrc-support 0.10.2 passes each vimrc line verbatim to `handleEx`. Its
  `obcommand` throws "Command … was not found" when the id is not registered.
- **The plugin is not enabled.** `bob-command-mode` is missing from the vault's
  git-tracked `.obsidian/community-plugins.json`.
  - On athena, `bob plugins list` reports `bob-command-mode 0.1.0 synced disabled`.
  - Because that file syncs through vault git, every host has the plugin disabled.
- **The MacBook may not have the plugin files.** Custom plugin folders are gitignored
  (`.obsidian/plugins/bob-command-mode/`), so each host needs its own
  `bob plugins sync`.
  - `docs/vault-git-sync.md` § Custom plugins documents the Mac step as
    `ssh mac 'bob plugins sync'`.
  - The Mac was offline when the lander checked (SSH connect timeout); it is reachable
    only when powered on with the lid open.
- No other plugin intercepts `:`.
- The installed `bob` on athena already serves `bob __catalog --protocol 1`, with
  features `clicolor-force`, `progress-env`, and `run-receipt`.

## Step 1 — Enable the plugin in the vault

1. Edit `~/bob/.obsidian/community-plugins.json` and append `"bob-command-mode"` to the
   array after `"obsidian-linter"`.
   - Keep Obsidian's exact format: a 2-space-indented JSON array of strings, with **no
     trailing newline** (the file currently ends in `"\n]`).
   - Change nothing else.
   - Read the file immediately before editing. If someone has changed it since, keep
     their entries.
2. Deliver the change through the git vault sync workflow (`docs/vault-git-sync.md`):
   - Run `bob vault-sync run`, then `bob vault-sync status --json`.
   - Confirm that `local_sha == remote_sha` and that the commit touches only
     `.obsidian/community-plugins.json`.
   - Preserve unrelated vault changes. Never re-enable Obsidian Sync.
3. Verify with
   `bob plugins list --no-pull --repo <bob-plugins path from 'sase repo open bob-plugins'>`.
   It must show `bob-command-mode … synced enabled`.

## Step 2 — Deploy on the MacBook (best effort)

1. Try
   `ssh -o ConnectTimeout=10 -o BatchMode=yes mac 'PATH="$HOME/bin:$HOME/.cargo/bin:/opt/homebrew/bin:/usr/bin:/bin" bob plugins sync --plugin bob-command-mode --dry-run'`.
   If it connects, run the same command without `--dry-run`. Never use `--force`.
2. If the Mac is reachable, also run these with the same `PATH` prefix:
   - `bob vault-sync run`, so the Mac pulls the Step 1 commit;
   - `bob plugins list`, which should report `bob-command-mode` as `synced enabled`;
   - `bob __catalog --protocol 1 | head -c 200`, which shows whether the Mac's bob
     already has the catalog. An older bob leaves the sheet in its "Update bob" state;
     that is not a blocker.
3. Agents cannot drive the Obsidian UI, so never claim a UI pass. Whether or not the Mac
   was reachable, the final report must list these manual steps for Bryan:
   - If the Mac was unreachable: run `ssh mac 'bob plugins sync'` (or `bob plugins sync`
     on the Mac) once it is online.
   - Reload Obsidian on the Mac (command palette → "Reload app without saving", or quit
     and reopen). Obsidian reads `community-plugins.json` only at startup. Reload before
     toggling any other plugin, or the running app may write back its old enabled list.
     If that happens, enable **Bob Command Mode** under Settings → Community plugins.
   - Smoke test: in a note's Vim normal mode, `:` opens the bob command sheet, `g:`
     opens Vim's ex line, and `-`, `]s`, and `Z` still work.

## Step 3 — Fix the NO_COLOR-sensitive runner test (bob-plugins)

Open bob-plugins with `sase repo open bob-plugins` and work only at the printed path.

- **Problem:** `scripts/test-command-mode-runner.cjs`, test "child env carries vault,
  features, receipt, and tool paths", fails whenever the caller's shell sets `NO_COLOR`.
  - Reproduce it with `NO_COLOR=1 node --test scripts/test-command-mode-runner.cjs`.
  - The cause is `runnerWith()` (around line 14), which seeds the runner env with
    `...process.env`. An inherited `NO_COLOR` then correctly suppresses
    `CLICOLOR_FORCE`, so the test's `CLICOLOR_FORCE === "1"` assertion fails.
- **Fix:** `runnerWith()` must drop an inherited `NO_COLOR` unless the test's `extraEnv`
  sets it explicitly. Do not change plugin behavior: `NO_COLOR` precedence is correct
  and covered in `scripts/test-command-mode-resolver-env.cjs`.
- **Verify** that `node --test scripts/test-command-mode-*.cjs` passes both with
  `NO_COLOR=1` and with `NO_COLOR` unset.

## Step 4 — Troubleshooting docs for a `:` that does nothing

- **bob-cli** `docs/guides/run-bob-from-obsidian.md`, section "Keep Vim's ex line
  available": replace the last two sentences ("If `:` does nothing, confirm …") with a
  short troubleshooting paragraph. It must say:
  - plugin folders are host-local (gitignored in the vault), so each desktop host needs
    `bob plugins sync`;
  - `bob plugins list` must show `bob-command-mode` as `synced` and `enabled`;
  - after enabling or deploying it, reload Obsidian;
  - `g:` still opens Vim's ex line meanwhile.

  Match the guide's voice and keep it to 3–4 sentences.

- **bob-plugins** `docs/bob-command-mode.md`, section "Keyboard and Vim": after the
  sentence ending "The plugin warns if a recursive map replays `:`.", add one sentence
  with the same check: run `bob plugins sync` on the host, confirm `bob plugins list`
  shows it enabled, then reload Obsidian.
- **Gates:**
  - **bob-cli:** run `just docs-gen` and `just docs-check`.
  - **bob-plugins:** run `npm run build`, `npm test`, `npm run validate`,
    `npm run docs:inventory:check`, and `npm run docs:check`.
  - Then run
    `bob plugins sync --repo <bob-plugins path> --no-pull --plugin bob-command-mode --dry-run`
    followed by the same command without `--dry-run`, as bob-plugins' CLAUDE.md
    requires. No manifest bump is needed, because plugin code does not change.
- **Known unrelated failures:** these may still be red and do not block this landing.
  Report them, but do not fix them here:
  - The `BOB_TEST_FIXTURE_PDF` docs lint
    (`src/native/highlights_ref/source_store.rs:194`), which belongs to epic
    `bob-cli-cc`.
  - The bob-cli lib test `fenced_column_zero_content_does_not_end_dependency_scan`.
  - 18 cases in bob-plugins `scripts/test-dependency-line-migration.cjs`.
  - "stage collections reuse one snapshot per parent".

  The last three belong to batch-14 epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6`. Both
  epics already carry `DISCOVERED ISSUE` notes. Any other new failure in the files you
  touch is yours to fix.

## Step 5 — Close epic `bob-cli-ce`

This step is the epic's closeout and must run in the same turn as Steps 1–4.

1. Run `sase bead epic-symbols bob-cli-ce`. The lander saw "No --epic-symbol entries".
   If entries have appeared, resolve each one per the Symvision epic-whitelist policy:
   wire it up, privatize it, add a non-test pragma, or delete it. Re-key a Justfile line
   to a still-open bead only when that bead still needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-ce --note "<verification>"
   ```

   The note must summarize:
   - All 11 phases were verified against their commits:
     - bob-cli: 92706cb4, 0bec19d8, b5ba95c9, 60eb7812;
     - bob-plugins: 7d005fb, 90eb3c0, c036597, 5790364, e3b2763, 6f8f1c5, 6b1b6ec;
     - vault: 402d1f7b and 92c3a71e.
   - The DECISIONS were honored: the four Pomodoro capture rows ship, `capture` stays
     excluded, and no decisions record was added.
   - The catalog covers the post-epic `ref code` and `ref attach-source` leaves.
   - Root cause of the `:` note: the plugin was disabled in the synced
     `community-plugins.json`. Name the vault commit that enabled it, and give the Mac
     deploy outcome or the pending manual step.
   - The NO_COLOR test fix.
   - The docs additions.
   - Gate results, naming the known unrelated failures.
   - Follow-up triage: no new tasks. `BOB_TEST_FIXTURE_PDF` was noted on `bob-cli-cc`,
     the legacy-dependency-reader failures were noted on batch-14, and the env test was
     fixed here.

   Never use `--force`. If the close is rejected, fix the stated cause and close again.

3. Run `just symvision` if the recipe exists (`just --list | grep -i symvision`). The
   lander found no such recipe in bob-cli's justfile; say so if it is still absent.
4. Set `status: done` in the frontmatter of the epic's plan file. It is the PLAN path
   that `sase bead read bob-cli-ce -r "Need the plan path"` shows:
   `plan:202610/bob_colon_command_mode.md` under `sase/repos/plans/`.
5. Epic `bob-cli-ce` has no `parent_bead`, so no parent handling is needed.

## Acceptance

- `bob plugins list` on athena shows `bob-command-mode` as `synced enabled`, and the
  vault commit is pushed.
- On the Mac: if it was reachable, the plugin is deployed and enabled there. Either way,
  the final report lists the manual steps from Step 2.
- `NO_COLOR=1 node --test scripts/test-command-mode-runner.cjs` passes in bob-plugins.
- Both docs pages carry the troubleshooting text. The docs gates report only the known
  unrelated failures, or none.
- `bob-cli-ce` is closed with the verification note, and its plan file says
  `status: done`.
