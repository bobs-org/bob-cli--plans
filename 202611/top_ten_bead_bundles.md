---
tier: epic
title: Work the top-ten task bead bundles from the two-host audit
goal: Every work item ranked by the 2026-11-04 two-host bead audit is done. bob-plugins
  `npm test` is green under UTC and New York time. `bob plugins sync` accepts a dirty
  deploy once its exact bundle is committed, so the vault stops running stale plugins.
  Obsidian closes keep recorded Pomodoro time. `bob periods brief` attributes archived
  and multi-link minutes correctly before the 2026-11-09 weekly Review. Vault edits
  stop leaking SASE plans. bob-mac-capture and bob-cli master CI are green again for
  the failures nobody else owns. athena alerts on hostile CPU, exec, and egress. Every
  link-consuming writer keeps notes, and generic ref PDFs render emoji.
decisions:
  g4_close_policy:
    ask: Which close behavior becomes canonical when a Pomodoro close defers every
      link?
    choices:
      keep: Obsidian keeps the closed timed row and its minutes, matching bob capture
        x
      delete: bob also deletes the closed row; the shared vector and capture docs
        change
    default: keep
    why: Recorded session time survives, per decisions:off-clock-pomodoros-count-apart
    answer: keep
  consumption_parity:
    ask: Should reconcile, task_complete, and plugin Depends-On consumption keep notes
      and delete only emptied entries?
    default: true
    why: Today one blocked link loses notes or an unrelated slot depending on the
      surface
    answer: true
  still_open_past:
    ask: How should the brief's Still open lists behave for periods that already ended?
    choices:
      rebuild: Rebuild Kept 3+ and No linked minutes as of span end; omit Blocked
        projects with a note
      omit: Omit every Still open list for ended periods and print one explanatory
        line
      label: Keep today's lists but title them "as of today" for ended periods
    default: rebuild
    why: Recovers what history allows and never prints a wrong Blocked list
    answer: rebuild
  mac_ci_repo_note:
    ask: Should bob-cli's always-loaded bob-mac-capture repo note tell agents to watch
      macOS CI after pushes?
    default: true
    why: bob-cli-1x asks for it; agents see it every turn, not only after opening
      the repo
    answer: true
  memory_consumption_record:
    ask: Should a decisions record capture the consumption-parity and timed-row retention
      rule?
    memory:
    - decisions
    requested: Can you help me complete the work corresponding with all of the task
      beads recommended in the task_bead_top10_with_two_host_agent_audit.md file in
      the research sidecar repo?
    default: true
    answer: true
phases:
- id: plugin-tests
  title: Make bob-plugins npm test green
  depends_on: []
  size: medium
  description: 'plugin-tests: in bob-plugins, fix the six test-harness defects behind
    bob-cli-eh, bob-cli-et (freshness half only), bob-cli-ff, bob-cli-f7, bob-cli-f1,
    and the bob-cli-ej wall-clock gates. Edit tests only. Prove the full suite is
    green under UTC and America/New_York, then close those beads and the already-fixed
    bob-cli-dx.'
- id: plugin-sync-proof
  title: Let bob plugins sync accept committed dirty deploys
  depends_on: []
  size: medium
  description: 'plugin-sync-proof: in src/native/plugins/deploy.rs, let a clean-checkout
    sync overwrite a dirty deploy stamp from another checkout or host when one commit
    reachable from HEAD holds the stamped bundle byte for byte. Keep every other refusal.
    Add tests, help text, and docs. Clear the live drift without --force, and close
    bob-cli-fy (and bob-cli-e4 once both hosts are synced).'
- id: brief-attribution
  title: Fix archived-task minutes and multi-link splitting in the periods brief
  depends_on: []
  size: medium
  description: 'brief-attribution: in src/native/periods/brief.rs, resolve original
    and done/ archive paths to one task identity (bob-cli-gs), and split a multi-link
    Pomodoro''s minutes with the stats root_parent splitter so task rows never outrank
    their root (bob-cli-gt). Land this before the 2026-11-09 Review.'
- id: vault-commit-guard
  title: Stop vault edits from leaking SASE plans into ~/bob
  depends_on: []
  size: small
  description: 'vault-commit-guard: rewrite ~/bob/AGENTS.md so agents let bob vault-sync
    commit, or use plain git for their own files, and never use /sase_git_commit.
    Replace the stale Obsidian Sync sentence. Make the vault .gitignore keep sdd/
    out. Close bob-cli-8n, and propose the SASE-side root cause of bob-cli-gy as a
    follow-up.'
- id: mac-ci-green
  title: Turn bob-mac-capture master CI green and record the CI-watch rule
  depends_on: []
  size: medium
  description: 'mac-ci-green: fix the root causes of the bob-cli-h7 Go To failures:
    the backslash insert helper, the delayed chip-refetch race, the leaked error wrapper,
    and the 15-minute hang. Coordinate with the 185 chain. Verify on the exact-SHA
    macOS run. Close h7, the six already-fixed Mac CI beads, and bob-cli-1x after
    recording the watch-CI rule.'
- id: highlights-tz-tests
  title: Make the highlights overwrite tests timezone-independent
  depends_on: []
  size: small
  description: 'highlights-tz-tests: the two highlights::overwrite_local failures
    (bob-cli-h6) assert a hardcoded -0400 offset, while CI stamps in UTC. Pin the
    zone for those bob subprocesses, sweep sibling assertions, and confirm the overwrite-refreshes-captured
    rule in docs. Then close h6 and the no-longer-reproducing bob-cli-fh.'
- id: athena-hostile-alerts
  title: Alert on hostile CPU, writable-dir execution, and mining egress on athena
  depends_on: []
  size: medium
  description: 'athena-hostile-alerts: in the tailnet repo, add per-user CPU telemetry
    (process-exporter on loopback), a textfile collector for writable-dir and deleted-binary
    execution plus nft mining-port egress counters, and Prometheus rules with promtool
    tests and staleness alerts. Route them through the existing Alertmanager email
    path. Deploy through /sase_sudo, prove that synthetic alerts fire and resolve,
    and close bob-cli-8r.'
- id: cycler-keeps-timed-close
  title: Align task-status-cycler's deferred-only close with bob
  depends_on:
  - plugin-tests
  size: small
  description: 'cycler-keeps-timed-close: apply the chosen bob-cli-g4 policy. By default,
    remove the task-status-cycler deferredOnlyClose/deletedClosedEntry special case
    so a closed timed row survives while its links still defer. Assert the full expected_day
    of the shared vector, update the docs, and close g4.'
- id: brief-polish
  title: Brief truncation counts, Bryan's words, and Markdown glitches
  depends_on:
  - brief-attribution
  size: medium
  description: 'brief-polish: in the same brief.rs, report totals and +N more behind
    the Finished/Dropped caps (bob-cli-gv). Make Bryan''s words deduplicated, date-ordered,
    fair across days, and link-safe (bob-cli-gw). Drop empty Part date segments and
    raw block ids in Still open aliases (bob-cli-gx).'
- id: brief-still-open
  title: Make the brief's Still open lists honest for ended periods
  depends_on:
  - brief-polish
  size: medium
  description: 'brief-still-open: apply the chosen bob-cli-gu semantics so a past
    period''s Still open section stops reporting today''s vault. By default, rebuild
    Kept 3+ and No linked minutes as of span end, and omit Blocked projects with a
    note.'
- id: ref-pdf-emoji
  title: Render emoji and uncovered symbols in generic ref PDFs
  depends_on:
  - highlights-tz-tests
  size: medium
  description: 'ref-pdf-emoji: route the generic pandoc-to-XeLaTeX render in highlights_ref/create.rs
    through the note-snapshot Noto Emoji fallback, so uncovered codepoints print and
    extract correctly. Add a golden extraction test, propose the PDF backfill as a
    follow-up, and close bob-cli-e9.'
- id: consumption-bob
  title: Bring reconcile and task_complete consumption to capture's rules
  depends_on: []
  size: medium
  description: 'consumption-bob: after bob-cli-h2 and bob-cli-h4 land (or their files
    go quiet), move reconcile''s blocked-link consumption and task_complete retirement
    onto capture''s keep-notes, only-emptied, and identity rules, never deleting a
    closed timed row. Add the §13.14 parity vectors and write the decisions record
    (bob-cli-7q, bob half).'
- id: consumption-plugins
  title: Bring bob-plugins Depends-On consumption to the same rules
  depends_on:
  - consumption-bob
  - cycler-keeps-timed-close
  size: medium
  description: 'consumption-plugins: move bob-plugins Depends-On link consumption
    off whole-subtree removal onto the same keep-notes, only-emptied, and identity
    rules. Assert the new §13.14 vectors in the plugin harness, sync the plugins,
    and close bob-cli-7q.'
proposed_by: bbugyi200.athena.18c
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 16:39:08
status: wip
bead_id: bob-cli-h8
---

- **PROMPT:** [prompts/202611/top_ten_bead_bundles.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/top_ten_bead_bundles.md)
- **BEAD:** [bob-cli-h8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h8/README.md)

# Plan: Work the top-ten task bead bundles from the two-host audit

## Why

Bryan asked for the work behind every bead the consolidated audit ranked. The source is
`research:202611/task_bead_top10_with_two_host_agent_audit/task_bead_top10_with_two_host_agent_audit.md`.
The audit ranks ten bundles by harm to Bryan's data and daily workflow, then by the tax
each bundle puts on every agent's gate. It also lists beads to close instead of work.
Two items there needed planning-time checks, and both changed the plan:

- **`bob-cli-h6` is a timezone bug, not a contract question.** CI run 41913160920 shows
  the marker stamped `- captured: 2026-10-16T12:00+0000`. The tests assert
  `2026-10-16T12:00-0400` (`tests/cli/highlights/overwrite_local.rs:655` and `:1658`).
  `BOB_NOW="2026-10-16 12:00:00"` is naive local time, and CI runs in UTC. That makes it
  small, and it no longer has to wait on anything.
- **The Mac error-prefix failure is a product regression.** The
  `bob command failed (exit N): <argv>. <stderr>` wrapper
  (`Sources/CaptureCore/BobErrors.swift:33`) is old. A Go To refetch path recently
  started showing the whole wrapper in the panel, while the test keeps the long-standing
  contract that the panel shows bob's own message.

State at planning time (2026-11-04 ~21:30 UTC):

- apollo runs `bob-cli-h2.5` and `bob-cli-h0.land` (h0.land will fix `bob-cli-h5`).
- athena runs `bob-cli-h4.4`, `185.f0--code` (the Mac sigil-alias work), and `sase-*`
  agents. `185.f0.f1` and `185.f0.f2` (backslash handling) are queued behind it.
- The latest Mac run (41912472770) also fails three `CaptureSeparatorToggleTests` from
  185's commit `3cf0c6d`. They are not part of `h7`, and their owner is still running.
- Both hosts' vaults show `bob-ledger-tools` and `bob-navigation-hotkeys` in `drift`
  from dirty deploys. athena's comes from a bob-cli_18 build at `7ad20f5`; apollo's
  comes from a different build. This is the live case `fy` must clear.

## Shared rules for every phase

- Read your beads first with `sase bead read <ids> -r "<why>"`. Open every repo other
  than bob-cli through `/sase_repo` (`bob-plugins`, `bob-mac-capture`, `tailnet`), and
  read that repo's `AGENTS.md` before editing.
- Right before you edit, refresh both rosters: `sase agent list -j` locally, plus
  `ssh <other-host> 'zsh -lc "sase agent list -j"'` for athena or apollo. If a running
  agent now owns a file you need, work around it or record that on your phase bead. Do
  not race it.
- bob-cli: never edit the `Cargo.toml` version or `CHANGELOG.md`. Mark a CLI or JSON
  contract break with `!` (decisions:bob-cli-releases-ride-a-daily-train). Prefer
  additive JSON. Read the `docs.md` reference memory before changing `docs/` or CLI
  help, and `cli_rules.md` before adding an option. Run the repo's normal gate
  (`just check`) before you finish. Failures owned by `h2` (fmt/clippy) or `h0` (`h5`)
  are not yours. Name them and move on.
- bob-plugins: edit `src/` fragments, then `npm run build`. Run `npm test` and
  `npm run docs:check`, then `bob plugins sync` as its AGENTS.md requires. Never use
  `bob plugins sync --force` without Bryan's explicit OK.
- Close every task bead you resolve with `sase bead close <id> --note "<evidence>"` only
  after verifying the fix. Phase workers never create beads. Put
  `PROPOSED FOLLOW-UP: <summary — detail>` notes on your own phase bead. An issue caused
  by another active epic goes on that epic as a `DISCOVERED ISSUE:` note.
- Inside a phase, do not re-ask this epic's DECISIONS. The callouts below say what each
  answer changes.

| Phase                    | Beads                                   | Repo                            | Waits for                                 |
| ------------------------ | --------------------------------------- | ------------------------------- | ----------------------------------------- |
| plugin-tests             | eh, et, ff, f7, f1, ej (+ close dx)     | bob-plugins                     | —                                         |
| plugin-sync-proof        | fy, e4                                  | bob-cli                         | —                                         |
| brief-attribution        | gs, gt                                  | bob-cli                         | —                                         |
| vault-commit-guard       | gy, 8n                                  | `~/bob` vault                   | —                                         |
| mac-ci-green             | h7, 1x (+ close f3, fu, dz, eo, fj, fp) | bob-mac-capture, bob-cli memory | —                                         |
| highlights-tz-tests      | h6 (+ close fh)                         | bob-cli                         | —                                         |
| athena-hostile-alerts    | 8r                                      | tailnet (athena)                | —                                         |
| cycler-keeps-timed-close | g4                                      | bob-plugins                     | plugin-tests                              |
| brief-polish             | gv, gw, gx                              | bob-cli                         | brief-attribution                         |
| brief-still-open         | gu                                      | bob-cli                         | brief-polish                              |
| ref-pdf-emoji            | e9                                      | bob-cli                         | highlights-tz-tests                       |
| consumption-bob          | 7q (bob half)                           | bob-cli                         | `h2`/`h4` quiet (self-checked)            |
| consumption-plugins      | 7q (plugin half)                        | bob-plugins                     | consumption-bob, cycler-keeps-timed-close |

The three bob-plugins phases run in sequence, so only one agent edits that repo at a
time. The three brief phases are also sequential, because they all edit `brief.rs`.

## plugin-tests

All six defects are in test harnesses. Do not change production behavior.

- **`eh`:** `scripts/test-ledger-tools-note-ready.cjs` (~211) and
  `scripts/test-ledger-tools-relation-index.cjs` (~98) model `workspace.onLayoutReady`
  as one slot. `setupGcalEvents` (`src/291-plugin-gcal-events.js`) registers a second
  callback that overwrites it. Make the fakes keep an ordered callback list and fire
  each callback once, as Obsidian does. Grep `scripts/` for other single-slot fakes and
  fix them the same way. Keep the production registration.
- **`et`:** the gcal-events half already passes (25/25 under both zones). Narrow it to
  `scripts/test-ledger-tools-freshness-namespace.cjs` › "the reload event fires only
  when the due key set changes". Run the initial memo and the later refresh on one
  injected clock, through the plugin's `now()` seam. Do not just move fixture dates.
- **`ff`:** `scripts/test-navigation-hotkeys-schedule-work-log.cjs:145` hardcodes
  `2026-11-02` as the "future" date. Derive it from the harness clock (now + 7 days), or
  freeze the clock. Keep the Pending→Blocked and Work Log assertions.
- **`f7`:** in `scripts/test-ledger-tools-keep-inbox.cjs` (~1020–1041), build the
  expected notice with the same local-time formatter that production uses. Keep the
  success, floor, backoff, and busy cases.
- **`f1`:** `scripts/test-command-mode-sheet-sequence-dom.cjs`: when no Chromium is
  found, skip with a clear reason. Fail hard when `CHROMIUM_PATH` is set but invalid, or
  when `CI` is set.
- **`ej`:** replace raw `Date.now()` budgets that flake under full-suite load. These are
  "Live Preview performance stays bounded on a large child block" in
  `test-ledger-tools-freshness-mark-surfaces.cjs` (~939) and the warm viewport build
  gate in `test-ledger-tools-task-link-marks-live.cjs`. Prefer asserting work counts the
  harness can observe. Otherwise take the best of three timed runs against the existing
  budget. Never just raise the number.
- **Verify:** `TZ=America/New_York npm test` and `TZ=UTC npm test`, each run twice, all
  green. Run `npm run docs:check`. Then close `eh`, `et`, `ff`, `f7`, `f1`, and `ej`,
  each with its own evidence. Close `dx` once `test-ledger-tools-lane-walk.cjs` passes
  (the audit saw 10/10 at `b9f47f3`). Finish with `bob plugins sync`. Report any plugin
  it refuses, but do not force it; `plugin-sync-proof` owns that.

## plugin-sync-proof

`evaluate_guard` (`src/native/plugins/deploy.rs` ~830–960) handles a
`PriorStamp::Complete { matches_vault: true }` stamp with `dirty == Dirty`. Unless the
stamp is from the same checkout and host, it refuses with "HEAD ancestry cannot prove
inclusion…". Agents sync before SASE commits their turn, so every agent deploy is
stamped dirty and stays refused after its content is merged.

1. In that arm, when the incoming source is a clean worktree, look for an **inclusion
   proof**. Walk the commits reachable from the incoming HEAD that touched the plugin's
   source directory (`git rev-list HEAD -- <plugin dir>`), newest first, bounded (for
   example 500 commits). Find one commit whose tree holds every stamped managed asset
   with exactly the stamped `sha256`. Assets the stamp marks `present: false` must be
   absent. Hash blob contents in one `git cat-file --batch` session, cached by blob id.
   Read `scan.rs` / `CapturedPlugin` for how assets map to repo paths.
2. When a commit qualifies, `Permit { stamp_action: Write }`: a normal overwrite without
   `--force`. The sync output names the proving commit. Never accept per-file matches
   drawn from different commits; such hashes can describe a bundle that never existed.
   Keep every other refusal from `bob-cli-cu`: same-checkout divergence, digest
   mismatch, pending stamps, and unknown provenance.
3. Tests in `src/native/plugins/tests.rs` and the CLI suite:
   - A dirty deploy from checkout A whose bytes are then committed, followed by a later
     commit, syncs from clean checkout B without `--force`.
   - Bytes that match no single commit still refuse.
   - A bundle mixed from two commits refuses.
   - A vault edited after deploy (digest mismatch) refuses.
4. Update the refusal and help text and `docs/plugins.md`.
5. Live check, without installing over `~/.cargo/bin/bob`: run the freshly built binary
   from your tree as `bob plugins sync` (default bob-plugins checkout, no `--force`).
   Confirm `bob plugins list` shows 8/8 `synced` on your host. Check the other host over
   ssh. If its installed `bob` predates the fix, leave it for the land agent.
6. If a host's dirty deploy proves against no commit, do not force it. Record the bytes
   and the stamp on `bob-cli-e4` for Bryan. Close `fy`. Close `e4` only when both hosts
   show 8/8 `synced`.

## brief-attribution

Target: landed before Monday 2026-11-09.

- **`gs`:** archived tasks are indexed with `source_path` (the original note) and
  `archive_path` (`done/<note>_done`, `src/native/stats/index.rs`). `linked_minutes`
  matches session links only against `source_path`, and `task_link` always renders it.
  Add one resolver that maps either path plus the block id to a single task identity,
  keyed by full vault-relative path plus block id. Use it in `linked_minutes`,
  `linked_for`/`task_key`, ranking, and `task_link`, and render `archive_path` when it
  is set. Count each session–task match once, even when two link spellings resolve to
  the same task.
- **`gt`:** task rows add the whole `session.counted_minutes()` to every linked task,
  while root rows come from the stats `root_parent` lanes, which split the session.
  Split task minutes with the same splitter the lanes use (see
  `src/native/stats/homes.rs`). Do not duplicate the logic. Task rows under a root then
  sum to no more than that root. This changes values, not JSON shape. Say so in
  `docs/periods.md`.
- **Tests:**
  - A W41-shaped regression: a root and its own task, where the task must not exceed the
    root.
  - A 20-link session.
  - An archived task, which must get its minutes and a `done/` link.
- **Verify:** read-only `bob periods brief 2026-W41 -f md` and `2026-W43 -f md` against
  the live vault. In W43, the bob root lists its archived tasks. In W41, no task
  outranks its root. Spot-check that the printed links resolve. Close `gs` and `gt`.

## vault-commit-guard

`~/bob` is Bryan's git-synced vault, not a SASE repo. Read the `obsidian.md` reference
memory first. Inspect `git -C ~/bob status` and touch only your two files.

1. `~/bob/AGENTS.md`:
   - Replace line 5 ("…actively synced by Obsidian Sync") with: the vault is synced by
     git through `bob vault-sync` on athena, apollo, and the Mac; check it with
     `bob vault-sync status --json`. Keep the advice that pre-existing changes may be
     legitimate (this fixes `8n`).
   - Replace line 11 (commit with `/sase_git_commit`): `bob vault-sync` commits and
     pushes note edits automatically. If an explicit commit is needed, run
     `git add <your files> && git commit`. Never use `/sase_git_commit` here, because it
     archives SASE plan files into the vault.
   - The `CLAUDE.md`/`GEMINI.md`/… files are `@AGENTS.md` shims. Leave them alone.
2. `~/bob/.gitignore` starts with `*`, re-includes `!*/`, and later re-includes `!*.md`.
   Add an anchored `/sdd/` exclusion placed so it wins. Verify with
   `git -C ~/bob check-ignore -v sdd/plans/202611/x.md` and with
   `git -C ~/bob ls-files sdd` (expected empty).
3. Let vault-sync commit, then confirm both files with
   `git -C ~/bob log -1 -- AGENTS.md` and `bob vault-sync status --json`. Close `8n`.
4. Do not fix SASE here. Add `PROPOSED FOLLOW-UP:` for the sase project: SASE must never
   archive a plan into a repository that is not that plan's home. The likely trigger is
   the legacy gitignored `~/bob/sdd/beads/` store from June 2026. Leave that directory
   in place. The land agent files the bead and closes `gy`.

## mac-ci-green

Linux hosts cannot run the macOS targets. Follow bob-mac-capture's `AGENTS.md`: run
`just portable-check`, and compile a scratch copy on the mac when it is online. Use
bob-cli's `scripts/dev/macos-ci-check` (see `docs/develop/ci.md`) as the pre-push gate.
After pushing, watch the exact-SHA run through `/sase_monitor`
(`gh run watch <id> --repo bobs-org/bob-mac-capture --exit-status`). Keep iterating
until Test, Bundle, Launch smoke, and Install are green.

1. Start from the latest finished master run's failures (`gh run view --log-failed`).
   Fix each root cause and never delete or weaken a test.
   - **Backslash insert helper** (`CaptureBackslashGotoTests.swift:415`, from
     `fde8211`): the helper must decline while composing, submitting, or locked. First
     check athena's roster. If `185.f0.f2` (backslash handling) has started or landed,
     leave this test to it and re-check after it lands. Otherwise fix it here with a
     minimal change and note it on your phase bead.
   - **Delayed chip refetch** (`CaptureTaskGotoPanelTests.swift:487–501`, from
     `70c0839`): a stale refetch must not replace a reopened snapshot. Guard the result
     with a generation token or an equivalent check.
   - **Error wrapper** (`:605`): the panel shows bob's own message
     (`controlled Go To fixture failure`), not BobErrors'
     `bob command failed (exit N): <argv>. ` wrapper. Fix the Go To path that now
     stringifies the whole error. Keep the full wrapper in logs. If `git log -p` shows
     the wrapper was a deliberate user-facing change, update the assertion instead and
     cite that commit.
   - **15-minute hang:** download the run's xctest samples (`gh run download`) and fix
     the deadlock or wait they show.
2. `CaptureSeparatorToggleTests` (from 185's `3cf0c6d`) and failures that `h4.4`–`h4.6`
   or `h2.6` introduce belong to their owners while those agents run. Note them on the
   owner's epic, or for the 185 chain on your phase bead. Fix them only if their owner
   has finished and they are still red.
3. Close `h7`. Close `f3`, `fu`, `dz`, `eo`, `fj`, and `fp` after confirming that each
   bead's named tests pass in your green run (the audit cites green run 41888215951 at
   `4906d94`).
4. **`1x`:** add an "After you push" paragraph to bob-mac-capture's `AGENTS.md`: run
   `scripts/dev/macos-ci-check` before landing, and watch the exact-SHA run to green
   after every push.

> [!decision] mac_ci_repo_note Also append one sentence to the `bob-mac-capture`
> repository description in `sase/sase.yml`. That is project config: the generated
> `sase.md` note is rebuilt from it and must not be hand-edited. The sentence: GitHub
> Actions is the only gate for the macOS-only targets, so agents watch the exact-SHA
> macOS run to green with `/sase_monitor` after every push, or gate first with
> `scripts/dev/macos-ci-check`. Then run `sase memory init` to regenerate `sase.md` and
> `AGENTS.md`, and close `1x`. If this decision is off, make only the AGENTS.md edit and
> close `1x` with that scope noted.

## highlights-tz-tests

- In `tests/cli/highlights/overwrite_local.rs`, pin `TZ=America/New_York` on the bob
  subprocesses whose markers the tests read. Do it in the shared command helper where
  possible, so `FIXED_NOW` keeps its meaning. Grep `tests/cli/highlights/` for other
  hardcoded `-0400`/`-0500` stamps and treat them the same way.
- Verify with `TZ=UTC cargo test --test cli highlights::` and
  `TZ=America/New_York cargo test --test cli highlights::`.
- Confirm that `docs/highlights-ref-sync.md` states the rule `5c9b6e42` established: an
  overwrite refreshes `captured` to the minute stamp. Add one sentence if it is missing.
- Close `h6` after the next master CI run shows both tests passing, or with local
  evidence from both zones plus the run id once it has started. Close `fh` after
  `just check` (clippy) shows no `needless_return` in `task_theme.rs`.

## athena-hostile-alerts

Work in the `tailnet` repo (`/sase_repo open tailnet`, then read its `AGENTS.md`). Model
the work on `athena/etc/prometheus/rules/athena-apt.rules.yml`, its promtool test, and
`hosts/athena/monitoring/` (the loopback nft boundary and the scrape config). Evidence:
`research:202610/athena_plex_compromise_incident_report.md` ("Preventing a repeat" #6),
plus the `bob-cli-8v.land` note on `8r`.

1. **Per-user CPU:** install `prometheus-process-exporter`, grouped by username and
   bound to `127.0.0.1`. Add its port to the `athena_monitoring` nft table and a scrape
   job.
2. **Textfile collector**, run as a root systemd timer every minute and writing to
   node_exporter's textfile directory:
   - Count processes executing from `/tmp`, `/var/tmp`, `/dev/shm`, or `memfd:`, and
     deleted binaries whose original path was in those locations. A `/usr` binary
     deleted by an upgrade is benign.
   - Read nft output counters for TCP to 3333, 4444, 5555, and 14444 on non-loopback
     interfaces.
   - Write a last-run timestamp.
3. **Rules** (`athena-hostile.rules.yml`), each with promtool unit tests:
   - Service-account CPU above 2 cores for 10m. Tune the per-user thresholds against
     real athena load: Plex transcodes and `bryan`'s agent builds must not page, and the
     rule file records why.
   - Any writable-dir exec.
   - Any mining-port egress increase.
   - Collector stale for more than 5m.
   - process-exporter `up == 0`.
4. Reuse the existing route: Alertmanager `127.0.0.1:9093` → Postfix/Mailjet → Gmail.
   Never add a second delivery stack.
5. Deploy only through `/sase_sudo` gates, using the README's install pattern (back up,
   install, `promtool check`, restart). Acceptance:
   - A copy of `sleep` run from `/tmp` fires and then resolves by email.
   - A connect attempt to `192.0.2.1:3333` fires the egress alert.
   - Rules and collector survive `systemctl restart prometheus` and a timer restart.
6. Document install, verify, and tuning in a monitoring README, commit in tailnet, and
   close `8r`. Propose pairing `bob-cli-9x` (vault-sync halt alerts) onto this route.

## cycler-keeps-timed-close

> [!decision] g4_close_policy = keep bob-plugins only. In
> `plugins/task-status-cycler/src/060-pomodoro.js` (~847–973), remove the
> `deferredOnlyClose` / `deletedClosedEntry` special case, plus its hints in
> `170-plugin-pomodoro.js` (~463–479) and `135-plugin-successors.js` (~299–307). Keep
> deferring every kept link to the new placeholder. Keep deleting genuinely empty,
> **untimed** entries under the emptied-Pomodoro rule. In
> `scripts/test-task-status-cycler-pomodoro-clock.cjs`, drop the
> `close-defers-not-started-off-clock` exception and assert the full `expected_day`,
> covering worked and off-clock rows. Remove the "deliberate difference" from
> `plugins/task-status-cycler/scripts/fixtures/pomodoro_clock/README.md` and update
> `docs/task-status-cycler.md`. bob's behavior and vector stay unchanged.

> [!decision] g4_close_policy = delete bob changes instead.
> `src/native/capture_pomodoro_close/` deletes a closed entry whose kept links were all
> deferred. Update the `close-defers-not-started-off-clock` vector in
> `tests/fixtures/pomodoro_clock/vectors.json` and the closing section of
> `docs/capture.md`. Copy the vector into the plugin fixture and assert the full
> `expected_day` there. This touches `h2`'s close path, so first apply the wait rule
> from consumption-bob.

Then `npm run build`, `npm test` (both zones), `npm run docs:check`, `bob plugins sync`,
and close `g4`.

## brief-polish

Same file as brief-attribution, rebased on it.

- **`gv`:** `finished_section` and `dropped_section` truncate to `finish_cap` without
  keeping the count. Add additive JSON `total` and `more` for Finished and Dropped, and
  print `+N more` in `-f md`.
- **`gw`** (`bryans_words_section`, `work_log_entries`, `pomodoro_notes`):
  - Deduplicate by (date, normalized text), preferring the block-anchored daily-note
    link.
  - Use a stable sort by date only.
  - Trim only a checkbox prefix, and skip children that are only a wiki-link.
  - Strip wiki-link brackets inside alias text.
  - Replace the global `truncate(40)` with a per-day cap, so later days are never
    starved, and report what was cut (JSON count, `+N more` in Markdown).
- **`gx`:** `render_part` skips an empty dates segment. The Still open "No linked
  minutes" alias uses `task_link` / `clean_task_text` instead of raw scan text.
- Add focused tests for each defect, shaped like the beads' W37, W32, W27, and W30
  repros. Update `docs/periods.md`, then close `gv`, `gw`, and `gx`.

## brief-still-open

`still_open_section` builds every list from the live vault: `freshness::scan`,
`is_open`, `is_blocked` (today's `#prj_is_blocked`), and `parse_keeps` (today's
`[keeps:: N]`). It never looks at `ctx.observed_end`. The current, still-running period
keeps today's behavior. For periods that ended before today:

> [!decision] still_open_past = rebuild Title the section "Still open, as of
> <span end>".
>
> - Kept 3+ counts dated Rotten Log entries on or before the span end.
> - No linked minutes includes only tasks whose `[created::]` falls on or before the
>   span end, that were still open at the span end, and that had no linked minutes in
>   the span.
> - Blocked projects is omitted with one line: blocked state has no history. In JSON,
>   use an additive `omitted_reason`.

> [!decision] still_open_past = omit For ended periods, emit no Still open lists: one
> Markdown line, and an additive JSON `still_open.omitted_reason`.

> [!decision] still_open_past = label Keep today's lists, but title them "Still open, as
> of today (<date>)" and add JSON `as_of`.

Add tests at a past span with keeps and creations after its end. Update
`docs/periods.md` and close `gu`. If you have to remove or rename an existing field,
mark the commit `!`.

## ref-pdf-emoji

`render_temp_pdf_from` (`src/native/highlights_ref/create.rs` ~5890–5953) passes
`mainfont=DejaVu Serif`, `sansfont=DejaVu Sans`, and `monofont=DejaVu Sans Mono` with no
fallback. MuPDF finds U+FFFD in 193 of 498 `lib/chat/*.pdf` files. Note snapshots
already solve this:

- `note_snapshot/fonts.rs` ships a pinned `NotoEmoji-Regular.ttf`.
- `note.tex` declares `\BobFallbackFont`.
- `note.lua` routes uncovered codepoints through a glyph table.

Reuse that path for the generic route. Share the glyph-table logic rather than copying
it, and wrap codepoints DejaVu lacks in the fallback family, covering symbols as well as
emoji. Add a golden test that renders 🍅 ✅ ⚠️ next to a DejaVu-covered control. It
asserts that Noto Emoji is embedded and that the extracted text contains the characters,
and it skips when the TeX toolchain is absent, as sibling render tests do. Update
`docs/highlights-create.md` and close `e9`.

Do not re-render existing PDFs. Highlights PDFs carry annotations
(decisions:open-pdf-writes-are-coordinated). Propose a previewed backfill as a
follow-up.

## consumption-bob

**Wait rule.** Run `sase bead show bob-cli-h2 bob-cli-h4`. If either is still open, hand
a poll loop (every 10 min, up to 24 h) to `/sase_monitor` and resume when both are
closed. If 24 h pass, proceed only if `git log --since=6.hours` on master shows no
commit touching your files, and note that on your phase bead.

> [!decision] consumption_parity Capture already applies PDL8–PDL10 through
> `task_status_hooks::apply_blocked_consumption_with_options` (`structure.rs` ~561–628):
> it keeps notes, removes only what it emptied, and matches by `(path, block id)`. Bring
> the other two paths onto those rules.
>
> - **Reconcile:** `bob task reconcile` blocked-link consumption
>   (`task_status_hooks/sync.rs` ~483 and ~1416) moves to the `_with_options` calls. A
>   consumed link-only bullet keeps its children as plain text. Only entries that this
>   consumption emptied are removed.
> - **task_complete:** retirement (`task_complete/retirement.rs:168`) restricts
>   `plan_empty_pomodoro_removals` to entries this batch emptied. It never deletes
>   untouched placeholders. Under every path, a closed Pomodoro row with recorded time
>   (`[t::]` or `[off::]`) is never deleted; this matches `g4_close_policy = keep`.
> - **Parity vectors:** add them to `docs/task-dependencies.md` §13.14, and to the
>   fixture that both writers' harnesses read, if one exists. Cover nested notes,
>   aliased links that resolve to one task, untouched empty placeholders, and timed
>   rows. Show that a second reconcile is a no-op.
>
> Capture still drops prerequisites, and plugin gestures still consume
> (decisions:capture-link-drops-blocking-deps).
>
> If this decision is off, change no code. Add §13.14 vectors that pin today's intended
> difference, and let consumption-plugins close `7q` as `canceled`.

> [!decision] memory_consumption_record Use `/sase_memory_write` to add one strand to
> the `decisions` web, for example `consumption-keeps-notes-and-recorded-time`. It
> records the accepted rule for every writer that consumes or retires Task Links:
> capture, reconcile, task_complete, and bob-plugins. The rule covers keeping notes,
> removing only emptied entries, identity matching, and timed-row retention, or, if
> parity is off, the PDL7 difference. Cite this epic's approved decisions, `bob-cli-7q`,
> `bob-cli-g4`, and `plan:202610/capture_star_blocks.md` §8, then run
> `sase memory init`.

Run the focused `task_status_hooks`, `task_complete`, and reconcile tests, then
`just check`. Update the reconcile and task-complete docs.

## consumption-plugins

> [!decision] consumption_parity bob-plugins Depends-On link consumption currently
> removes the whole subtree (see
> `scripts/test-navigation-dependency-link-consumption.cjs` for the gesture under test).
> Move it to the same rules:
>
> - A consumed link-only bullet keeps its children as plain text.
> - An entry is removed only when this gesture emptied it.
> - Links match by resolved `(path, block id)`, so aliases count.
> - A closed timed row always survives.
>
> Assert the new §13.14 vectors in the plugin harness. If parity is off, assert only the
> documented difference.

Update the bob-plugins dependency docs. Run `npm run build`, `npm test` (both zones),
`npm run docs:check`, and `bob plugins sync`. Close `7q`: `done`, or `canceled` if
parity is off.

## Land agent

- Re-run the bob-cli gate on master, and bob-plugins `npm test` under UTC and
  America/New_York. Attribute any remaining red to its owner (`h2`, `h0`/`h5`, the 185
  chain).
- With a bob that contains plugin-sync-proof, run a normal `bob plugins sync` on athena
  and on apollo (over ssh). Close `e4` once both list 8/8 `synced`. Never use `--force`
  without Bryan.
- Confirm that the latest bob-cli master CI passes `highlights::overwrite_local`, and
  that the bob-mac-capture master Test step is green or red only for failures owned
  elsewhere.
- Turn phase follow-ups into beads with `/sase_new_task`:
  - In the **sase** project: plans must never be archived into a repo that is not their
    home. Then close `gy`, citing the vault mitigation and the new bead.
  - The `e9` PDF backfill.
  - Pairing `9x` onto the athena alert route.
- Add the audit's narrowing notes: on `8a`, long flags are refused now; re-check short
  flags after `h0` lands. On `ge`, after `933091b`, only the grammar-3/4 fixture JSONs
  remain. Leave `h5` to `h0.land`.

## Out of scope

- The audit's runners-up: `gd`/`f2`/`g8`, `9x`, `f8`, `fz`, `dq`, `h1`/`gc`, `e0`/`ei`,
  `30`/`5g`, and the batched "record decision" memory beads.
- The SASE-side fix for `gy` (filed, not fixed here).
- Re-rendering existing PDFs and re-drafting past weekly Reviews.
- The `gp` Monday automation.
