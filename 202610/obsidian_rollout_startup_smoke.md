---
tier: tale
title: Package the real-Obsidian plugin rollout smoke check
goal:
  Plugin rollouts verify startup and four editor mounts in isolated Obsidian with
  reproducible evidence and a deployment gate.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5
status: done
---

- **PARENT:**
  [202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5.md)
- **COMMITS:**
  - [88f0cdd](https://github.com/bobs-org/bob-plugins/commit/88f0cddaec4907623b2d4498183d1884be1b58a5)
    — feat(plugins): package the real-Obsidian startup smoke check

# Package the real-Obsidian plugin rollout smoke check

Implement phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5` (`plugin-startup-smoke`)
from `plan:202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md`.
The final parent decision is `startup_smoke = implement`. Deliver the reusable check,
offline tests, deployment documentation, and a concrete gate for the parent lander's
re-sync. Leave target task `bob-cli-a6` open for that lander. Close only the assigned
phase after verification; never close an ancestor.

## Evidence and size

The phase is in progress and has no prior-work or remainder notes. At planning, bob-cli
is at `2f004d77`, bob-plugins at `0b02193`, both clean. Searching the plugin scripts and
history found Chromium preview utilities but no packaged Obsidian startup check. No
phase-owned `--epic-symbol` entries were present. This is one bounded tooling change in
bob-plugins, a guide change in bob-cli, and verification; one medium coding agent can
finish it.

Read these audited sources for context:

- `bead:bob-cli-a6`, including the independent editor-mount failure evidence.
- `plan:202610/obsidian_startup_relation_index_oom.md` and completed agent `12h`
  (`sase chat show --agent 12h`): the original check ran on the Mac through CDP
  port 9333. Version 1.60.0 replayed 8,414 creates and read roughly 3.5 GB of mixed file
  types before layout-ready. Fix `b61f214` reached layout-ready in 2.6 seconds with zero
  early ledger reads, peak heap about 793 MB, and no new renderer reports. Its
  permission to stop Bryan's live Obsidian does **not** apply to this phase.
- bob-cli-a6's +1 records that 1.67.0 passed 3,647 stubbed unit tests while real
  CodeMirror editor construction threw `isEmpty` errors. Both note kinds and both
  editing modes are necessary.

Athena currently has `obsidian`, `xvfb-run`, and Node 22.14.0 with native
`fetch`/`WebSocket`. A bounded SSH attempt to `mac` timed out. Prefer a local isolated
Obsidian run under Xvfb; availability on PATH is not proof that it launches. The parent
permits shipping tested tooling with an explicit unrun real-app limitation when no
reachable host can run Obsidian.

## 1. Add the runner in bob-plugins

Use `/sase_repo` to open bob-plugins and read its `AGENTS.md`. Recheck current history
and phase notes before implementing to avoid duplicating concurrent work. Use
`/sase_memory_read` for `docs.md`, `cli_rules.md`, `obsidian.md`, and any required
glossary terms. No memory changes are authorized or required. Keep plugin source,
generated entrypoints, manifests, versions, and public Bob CLI behavior unchanged.

Add `scripts/obsidian-startup-smoke.cjs`, small supporting modules if useful, and an
`npm run smoke:obsidian` entry. Use Node 22 built-ins, following the CDP client pattern
in `scripts/preview-day-strip.cjs`, but add bounded RPC/connect timeouts, disconnect
handling, and protocol-event collection. Do not refactor the existing preview scripts as
part of this work.

Provide a minimal documented interface: an Obsidian executable override, an optional
JSON report path, `--self-test`, and `--deployed-vault PATH` for post-sync verification.
The plugin checkout is resolved from the script's location. Give developer-facing long
options short aliases and alphabetized, complete `-h`/`--help`, following CLI memory.
Set fixed default acceptance thresholds (30 seconds to layout-ready, 1.5 GiB sampled
peak heap); do not quietly relax thresholds to obtain a pass. Use clear nonzero
failures, including missing prerequisites and unrun checks. Self-tests must identify
themselves as offline tests, never as an Obsidian pass.

For `--deployed-vault`, invoke the existing read-only
`bob plugins list --repo <this-checkout> -b <vault> -f json` and require every expected
repository plugin to be present, have the expected version, and have `sync: synced`. The
command's exit code or `ok` alone is insufficient: `list` exits zero on drift. Fail on
missing or malformed rows. Repeat this verification after the smoke check and reject
source changes during the run. This ties the tested build bytes to the deploy without
ever opening the live vault in Obsidian. Document that the fixture verifies the managed
Bob plugin build, not the user's full third-party plugin/settings combination.

### Disposable fixture and process ownership

- Create a fresh directory owned by this run for the vault and a separate fresh
  application profile. Copy/generate only synthetic fixture notes and the checkout's
  managed plugin files; enable all its Bob plugins in the scratch vault. Never copy live
  settings, credentials, symlinks, or a live workspace.
- Include a canonical `YYYY/YYYYMMDD.md` daily note, an ordinary task note, a second
  note with a resolvable task relation, and several small binary files with non-Markdown
  extensions. Give fixture notes valid parent links. The relation must actually enter
  the index; an empty-index pass is inadequate. A bounded synthetic set of extra files
  should exercise startup create replay without allocating gigabytes. Do not require
  downloaded third-party plugins.
- Launch an owned Obsidian process with an isolated user-data/profile directory,
  loopback CDP, and scratch-only configuration/cache/state paths. Set child `BOB_DIR`
  and cwd to the scratch vault so plugin subprocesses cannot fall back to the real
  vault. Do not repurpose `HOME` or `CODEX_HOME`. On Linux use an independent Xvfb
  display, not the inherited desktop display. Verify the installed executable's profile
  routing before relying on it; never fall back to the normal profile or an existing
  Obsidian session.
- Select only the CDP target belonging to the owned process/profile. Verify the
  renderer's vault path equals the real path of the generated fixture before driving it.
  A missing target, wrong vault, reused profile, or failed instrumentation must fail,
  not attach elsewhere. No hard-coded shared 9333 port, `open -a Obsidian`, `pkill`, or
  global app restart.
- Always close CDP and terminate/reap only the process tree this run created, including
  on failure, timeout, and signals. Remove only run-owned temporary files after
  collecting diagnostics. Keep an overall watchdog independent of renderer promises so a
  hung renderer cannot hang the gate.

## 2. Measure startup and all four editor mounts

Reuse the proven injection approach from the OOM plan: enable CDP Page/Runtime, install
`Page.addScriptToEvaluateOnNewDocument`, then reload the **scratch** renderer for an
instrumented startup. Record the initial launch separately; an early crash or process
exit is also a failure. The injection must observe the app before plugin startup, wrap
vault `read`/`cachedRead` and create replay, and record layout-ready timing. Verify
hooks installed early enough; zero counters without a working hook do not establish a
pass.

Collect read counts before/after layout-ready, non-Markdown reads attributed to
`plugin:bob-ledger-tools`, create replay counts, runtime exceptions/rejections,
plugin-load errors, heap samples, and renderer/process crashes. Check crash artifacts in
the owned profile where available; retain CDP crash/disconnect and process-exit evidence
as well. Report the crash observation method instead of claiming a report-directory
check that was never performed.

Acceptance must require:

1. Layout-ready within 30 seconds of instrumented startup, with all expected plugins
   loaded and matching source manifest versions.
2. Ledger relation index ready; zero non-Markdown keys; the fixture relation indexed;
   zero ledger reads before layout-ready and zero non-Markdown ledger reads. Current
   probes are `relationIndexApi().ready()` and `relationIndexFiles` in
   `src/171-relation-index-lifecycle.js`.
3. Sampled peak JS heap below 1.5 GiB throughout the run, not just a final sample.
   Missing/non-finite samples are failures. Observe a short bounded settling interval
   after startup and editor operations so delayed exceptions surface.
4. Open the canonical daily note and ordinary note in **Live Preview** and **Source**
   mode (four cases) through the real workspace/Markdown view. For each, wait for the
   expected file and actual mode, a connected CodeMirror editor DOM, and a functioning
   editor with the expected document text. Await mount/paint and observe exceptions. A
   successful `openFile` return alone, an old editor still present, or a Reading-view
   leaf is not success. Do not replace CodeMirror, disable the Day strip, or stub Bob
   plugin implementations.
5. No new renderer crash artifacts, crash events, plugin-load failures, uncaught
   exceptions/rejections, or editor mount errors. Capture console errors where Obsidian
   catches and reports a failure; avoid broad exception allowlists.

Write a compact human summary and a structured report on both success and failure:
source SHA, plugin ids/versions and managed-file hashes, executable/app version where
available, durations, counters, heap peak, each editor result, crash/exception evidence,
deployment match result, and exact failed checks. No real-vault note content belongs in
the report.

## 3. Offline regression tests and rollout wiring

Add `scripts/test-obsidian-startup-smoke.cjs` to the existing explicit `npm test` list;
`--self-test` runs these offline checks without needing Obsidian. Separate assertion
logic from orchestration to test evidence instead of mirroring code. Cover at least:

- a complete passing observation;
- timeout, absent/late instrumentation, empty-index false positive, missing or wrong
  plugin version, early/non-Markdown ledger reads, high/missing heap;
- each of the four editor cases missing or failed, wrong mode/path, disconnected editor,
  a swallowed mount error, asynchronous renderer exception or crash;
- malformed/missing/drifting deploy rows despite a successful `list` exit;
- wrong-vault rejection, bounded RPC failure/disconnect, and cleanup of only owned
  temporary paths/processes with a fake transport/process.

Update bob-plugins `docs/develop/deployment.md` and bob-cli
`docs/guides/deploy-obsidian-plugins.md` (read docs memory first). Merge with the
concurrent plugin-ignore phase's edits; do not overwrite them. Include:

- prerequisites and the isolated Linux/Xvfb invocation;
- the offline self-test command, explicitly distinguished from a real pass;
- build/check, sync preview, scoped sync, then the mandatory post-sync command:
  `npm run smoke:obsidian -- --deployed-vault <vault> --report <report.json>`;
- threshold meanings, report location, failure handling, and the limited fixture
  coverage. No rollout-success claim on a skipped/unrun or failing check.

Do not change `bob plugins sync` itself or `scripts/install_all` to restart or
instrument Bryan's session. This is the explicit rollout gate selected by the parent,
usable by humans and land agents.

Record the exact command, executable/profile prerequisites, report reference, and
verdict on the assigned phase via `sase bead note`. Label the note
`LANDER ROLLOUT GATE:` and state that the parent lander must run it after its master
re-sync and zero-drift check, before declaring rollout verified. A current failure or
missing host must stay visible as a limitation; the gate must not convert it into
success. This durably wires the step into the parent lander's existing phase-evidence
review without editing an approved ancestor plan.

## 4. Verification and phase completion

Run the offline suite first, then a real isolated Athena Obsidian run if it can launch.
Record measured values. If safe launch is unavailable, document the specific observed
reason and leave exact rerun instructions; the approved fallback permits completing this
phase with offline validation and a clear unrun real-app limitation. Do not spend the
turn waiting for the Mac's lid.

Run full `npm test`, `npm run build:check`, and `npm run validate` in bob-plugins. Run
`just check` (never `check-full`) in bob-cli, honoring `CARGO_TARGET_DIR`. Run the
mounted docs gate using `BOB_CLI_DIR=<current bob-cli checkout>` for
`npm run docs:check` so it does not clone an unapproved cache repository; this runs
`just docs-check` against these plugin docs. Use `/sase_monitor` for long commands and
finish the handoff command before ending a turn.

Honor bob-plugins' required `bob plugins sync` verification. Before syncing,
fetch/rebase onto current origin/master and rebuild; preview first. Use the scratch
fixture for this tooling-only change, since no managed plugin payload is being changed.
Leave the live all-plugin master re-sync to the parent lander, as the parent design
specifies. If implementation unexpectedly needs a plugin payload change, keep it within
scope, regenerate from fragments, and follow the parent's changed-plugin-only sync and
deploy-stamp rules; do not force past someone else's deploy.

For unrelated check failures, reproduce on the clean base and append
`PROPOSED FOLLOW-UP: <summary — evidence and existing task id if known>` on this phase.
Such baseline failures do not keep this phase open. Create no task beads. Preserve every
changed repository through the normal `/sase_final` flow; do not manually commit unless
separately authorized.

Before closing, rerun
`sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5`. Resolve any entries
or re-key them to a still-open owner. Then close **only**
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.5` with a note naming the delivered
runner/docs, validation evidence, real-app result or limitation, and lander gate. Leave
`bob-cli-a6` and all ancestor beads for the parent lander.
