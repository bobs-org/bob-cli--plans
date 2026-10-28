---
tier: epic
title: Repair the clobbered sase_agent_recovery note and guard against note clobbers
goal: '`bob ref scan -w` runs clean on the MacBook again with the lost agent-recovery
  project restored; the review walk never leaves an unsaved plugin write behind when
  it switches notes; and any future whole-note clobber is caught by vault-sync and
  named plainly in bob''s output instead of surfacing days later as cryptic ref-scan
  refusals.

  '
decisions:
  clobber_response:
    ask: When vault-sync detects a clobbered note, should it restore the note and
      quarantine the bad bytes?
    choices:
      quarantine: Restore from HEAD, keep the clobbering bytes in _conflicts/, log,
        alert; bad text never syncs
      warn: Commit the note as-is, then log and alert only; you restore it by hand
    default: quarantine
    why: Matches the existing _conflicts/ policy and stops the loss before it reaches
      other hosts
    answer: quarantine
phases:
- id: vault-repair
  title: Restore the clobbered note and verify the scan
  depends_on: []
  size: small
  description: 'vault-repair: restore sase_agent_recovery.md from 8ca09ee75^, let
    bob task reconcile drop 8 stale mirror links, verify ref scan on athena and the
    MacBook, sync.'
- id: walk-save-guard
  title: Save before the review walk switches notes
  depends_on: []
  size: medium
  description: 'walk-save-guard: in bob-navigation-hotkeys, flush and verify the active
    note before any same-pane note switch, add a clobber tripwire on landing, test,
    build, deploy.'
- id: sync-clobber-guard
  title: Vault-sync clobber detection and quarantine
  depends_on: []
  size: medium
  description: 'sync-clobber-guard: detect displaced-plus-foreign-identity rewrites
    of tracked notes before staging, quarantine or warn per the decision, record in
    status, alert, document, test.'
- id: ref-scan-copy-hint
  title: Name duplicated-task copies in ref scan failures
  depends_on:
  - sync-clobber-guard
  size: small
  description: 'ref-scan-copy-hint: make multiple_open_ref_tasks say when the claimants
    are copies of one task, and group repeated refusals into one cause line pointing
    at the clobber runbook.'
proposed_by: bbugyi200.athena.157.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 10:46:05
status: done
bead_id: bob-cli-do
---

- **PROMPT:** [prompts/202610/clobbered_note_repair_and_guards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/clobbered_note_repair_and_guards.md)
- **BEAD:** [bob-cli-do](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-do/README.md)

# Plan: Repair the clobbered note and make note clobbers impossible to miss

## Background: what happened (verified)

`bob ref scan -w` on the MacBook refuses 13 PDFs with `multiple_open_ref_tasks` because
every open reading task in `sase.md` also exists, byte-identical, in
`sase_agent_recovery.md`. A previous research report
(`research:202610/bob_ref_scan_duplicate_reading_tasks.md`) diagnosed this. It was
re-verified for this plan against the live vault and a scratch `git clone --shared` of
it:

| Report claim                                                                                                                                                     | Verdict                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The duplicate tasks cause the 13 refusals                                                                                                                        | **Confirmed.** Scratch clone at vault HEAD: 13 refusals. Same clone with the note restored: 549 PDFs, 0 failures.                                                                                                                                                                      |
| Vault commit `8ca09ee75` (MacBook, 2026-10-28 09:31:10) replaced the note with sase content; do not `git revert` it because it also carries real `sase.md` edits | **Confirmed.**                                                                                                                                                                                                                                                                         |
| The agent-recovery `^prj` line and its 7 tasks now exist only in `8ca09ee75^`                                                                                    | **Confirmed.** This is real data loss, not just a scan failure.                                                                                                                                                                                                                        |
| Every line of the current `sase_agent_recovery.md` also exists in `sase.md`                                                                                      | **Confirmed.** Restoring discards nothing unique.                                                                                                                                                                                                                                      |
| Mechanism unknown: maybe a stale editor buffer or a second sync engine                                                                                           | **Wrong.** See root cause below.                                                                                                                                                                                                                                                       |
| One dangling link (`sase_pypi_releases.md:35`), to be fixed by hand                                                                                              | **Incomplete.** There are 8 stale `[[sase_agent_recovery#^…]]` mirror links on 7 lines in 6 notes. They are managed LINKED FROM / BLOCKS mirrors, and after the restore `bob task reconcile` removes all of them (verified by dry run on the scratch clone). No hand edits are needed. |

### Root cause (strongly supported, not reproduced inside Obsidian)

- **Review journal** (`.bob/stats/review/2026-10.kellys-mbp.jsonl`, plugin
  `bob-navigation-hotkeys@2.56.4`, ]s walk, TICKLER tier):
  - 09:30:59: `done` on `sase.md:36`.
  - 09:31:01: `defer` on `sase.md#q-weights-metadata`, a Task Card recommended roll
    (`🎲 P2 roll`).
  - 09:31:16: next answer, on `sase_art.md:17`.
  - 09:31:45: `defer` on `sase.md#q-weights-metadata` **again**.
- **Commit `8ca09ee75`:** the new `sase_agent_recovery.md` is exactly `sase.md` plus the
  09:31:01 roll (row flipped to `[?]`, `[scheduled:: 2026-11-19]`, new
  `🎲 P2 roll · in 22` log line). `sase.md` itself never got that roll, which is why the
  walk offered the same row again 44 s later.
- **Pre-overwrite `sase_agent_recovery.md:16`** was a TICKLER row due 2026-10-28. In
  path order (`sase.md` < `sase_agent_recovery.md` < `sase_art.md`) it is the walk's
  next stop after sase.md's last tickler.
- **Plugin code** (bob-plugins `plugins/bob-navigation-hotkeys/src/`):
  - The roll writes only through a CodeMirror editor transaction
    (`setInlineBulletPropertyValues` in `219-plugin-project-subtree-writers.js`). Saving
    to disk is left to Obsidian's debounced save.
  - The walk advance (`536-plugin-review-advance.js`) then lands cross-note through
    `openMarkdownFileWithLeafReuse` (`480-review-jump-and-nav-api.js`). That calls
    `app.workspace.getLeaf(false).openFile(file)`, which reuses the same pane with no
    save first.
  - The unsaved sase.md text was persisted under the newly opened file.

The plugin's own pre-write guards (path and text re-checks in 219/550) were satisfied:
the damage happened after the write, during the pane switch. Because the trigger is
inferred from Obsidian internals, the fix is layered:

1. Prevent it in the plugin.
2. Catch any clobber at sync time, before it spreads to other hosts.
3. Make the downstream symptom name its cause.

### Vault-safety rules for every phase

- Never `git revert`, `reset --hard`, or force-push the vault.
- Never run a non-dry-run `bob vault-sync`, `bob task reconcile`, or `bob ref scan -w`
  against a _clone_ of `~/bob`. Its origin is the real vault. Tests use temp repos with
  a temp bare remote.
- Do not touch `_conflicts/`.
- Leave the unrelated duplicate alone. `^docs-like-zen` sits in both `bob.md:55` and
  `sase_pre_blog.md:40`, from a 10-20 double capture. Bryan chooses which copy to keep;
  it is out of scope here.

## Phase vault-repair: Restore the clobbered note and verify the scan

Run on athena against `~/bob`. The installed `bob` (0.4.0) is sufficient; this phase
needs no code changes.

1. Run `bob vault-sync` to pull the latest. Confirm the note is still clobbered: its
   frontmatter has `created: 2026-06-01T03:30:02-04:00` and its `^prj` line is the sase
   "≥1000 stars" line.
   - If someone already repaired it, skip to step 5.
2. Confirm the current file holds nothing unique. This must print nothing:

   ```bash
   grep -vxFf ~/bob/sase.md ~/bob/sase_agent_recovery.md | grep -v '^\s*$'
   ```

   If it prints lines, stop and report them; they would be sase edits made in the copy
   after 09:45 and must be carried into `sase.md` first.

3. Restore the pre-overwrite note:

   ```bash
   git -C ~/bob show 8ca09ee75^:sase_agent_recovery.md > ~/bob/sase_agent_recovery.md
   ```

   Check that `parent: "[[sase]]"` is present and the `^prj` line reads "Failed, killed,
   or interrupted agents … can be recovered in one action without losing work!".

4. Run `bob task reconcile --dry-run --format json`. Expect these removals (live daily
   churn may add unrelated entries; nothing may touch `sase.md` task lines):
   - `blocks_line_updates`:
     - `[[sase_agent_recovery#^time-all-of-turn]]` in `sase_pre_blog.md`.
     - `[[sase_agent_recovery#^ref-gascity-advantages-over-sase-and-what]]` in
       `sase_pypi_releases.md`.
   - `linked_from_line_updates`:
     - `^web-pdf-artifact-viewer` in `dev.md`.
     - `^create-new-project` and `^snipit-tool` in `sase_art.md`.
     - `^task-plugins` in `sase_memory_plan.md`.
     - `^create-new-project` in `sase_post_blog.md`.
     - `^usage-tracking` in `sase_pre_blog.md`.

   Then run `bob task reconcile`. `done/sase_done.md`'s link to
   `[[sase_agent_recovery#^restart-survival]]` is valid again and stays.

5. Run `bob ref scan --dry-run`. Expect no `multiple_open_ref_tasks` and 0 failures.
6. Sync:

   ```bash
   bob vault-sync run --message "vault: restore sase_agent_recovery.md clobbered by 8ca09ee75"
   ```

   If the background service commits first with a generic message, that is fine.

7. On the MacBook (best effort; `mac` is offline unless its lid is open):
   - Run `ssh mac 'bob vault-sync -q && bob ref scan --dry-run'` and expect 0 failures.
   - Confirm `ssh mac 'git -C ~/bob status --short sase_agent_recovery.md'` is empty.
   - Confirm the Mac copy has the agent-recovery `^prj` line about a minute later.
     - If Obsidian re-saved the stale buffer, the file reverts to sase text. Report that
       and ask Bryan to close the tab before repeating step 3.
   - The Mac cron (`~/bin/maybe_bob_highlights_sync -w`, every 15 min) performs the real
     write scan. If time allows, check `/var/tmp/maybe_bob_highlights_sync.log` after
     its next run. Do not block the turn waiting for it.

## Phase walk-save-guard: Save before the review walk switches notes

Repo: **bob-plugins** (open it with `/sase_repo`; read its `AGENTS.md`). Edit `src/`
fragments of `plugins/bob-navigation-hotkeys`, run `npm run build`, and never hand-edit
`main.js`. Keep each hand-edited fragment ≤ 1000 lines.

1. **Shared leaf-switch guard.** Add a helper, a new fragment or inside
   `480-review-jump-and-nav-api.js`:

   ```text
   prepareActiveLeafSwitch(plugin, targetFile)
     -> { ok, sourcePath, sourceText, targetDiskText }
   ```

   When the active MarkdownView shows a different file than the target:
   - `await view.save()`.
   - Then, if the view exposes a debouncer (`view.requestSave?.cancel`), cancel it so no
     delayed save can fire after the swap.
   - Then confirm `await vault.read(sourceFile) === editor.getValue()`.
   - On a save error or mismatch, show a Notice ("Could not save <path> before switching
     notes; staying here") and return `ok: false`.
   - Also `vault.read` the target's current disk text for the tripwire.

2. **`openMarkdownFileWithLeafReuse`.** Call the guard only on the
   `getLeaf(false).openFile(file)` branch. The same-file early return and existing-leaf
   activation do not swap a pane's file and need no save. Abort the navigation
   (`return false`) when the guard fails.
3. **Clobber tripwire.** It has two parts:
   - After `openFile` resolves: if the new view's editor text equals `sourceText` while
     `targetDiskText !== sourceText`, the pane opened the target with the previous
     note's text. Show a sticky Notice (timeout 0): "bob: <target> opened with
     <source>'s text — it may have been overwritten while switching notes. Walk stopped;
     see vault-git-sync docs, Clobbered notes". Then return `false`. The review advance
     in `536-plugin-review-advance.js` already treats a failed open as a stopped
     landing.
   - About 2.5 s later (past Obsidian's 2 s save debounce), re-read the target from
     disk. If the disk text now equals `sourceText` (and `targetDiskText` did not), show
     the same sticky Notice.
   - Do not auto-rewrite either note from the plugin. Vault-sync's guard owns recovery.
4. **Other same-pane switches.** Route them through the same guard: every site that
   replaces the file shown in an existing leaf. Audit and cover what applies:
   - `620-plugin-dash-and-vim-jumps.js:55`
   - `630-plugin-vim-link-and-leaves.js:220`
   - `675-plugin-goto-task-uri.js:472` (`setViewState`)
   - `676-plugin-open-notes-uri.js:427`
   - `680-plugin-link-parse.js:213/223` (`openLinkText` in the same leaf)
   - `285-unblocked-notice.js:507-509`
   - the single `openFile` in `plugins/bob-ledger-tools/main.js`

   Leave sites that always open a new leaf alone.

5. **Tests.** Add `scripts/test-navigation-leaf-switch-guard.cjs`, register it in
   `package.json`'s `test` list, and use `scripts/navigation-hotkeys-harness.cjs`.
   Cover:
   - (a) A cross-note review landing right after an editor transaction calls the source
     view's `save()` before `openFile`, and cancels a pending debounced save.
   - (b) A rejected save, or a disk/editor mismatch, means no `openFile` call, a Notice,
     and the walk stops.
   - (c) The clobber signature on landing produces a sticky Notice and `false`.
   - (d) Same-note landings and existing-leaf activation never call `save()`.

   Extend `scripts/test-navigation-review-advance.cjs` where its harness already drives
   cross-note landings.

6. Bump the plugin's manifest version per repo convention. Run `npm run build`,
   `npm test`, and `npm run docs:check`. If the plugin docs describe walk landings,
   document the tripwire Notice there.
7. Deploy:
   - Run `bob plugins sync` on athena.
   - The MacBook deploys from its own checkout via `ssh mac 'bob plugins sync'` once the
     commit is on origin. Do that if possible (best effort), otherwise report it as
     Bryan's remaining step.

## Phase sync-clobber-guard: Vault-sync clobber detection and quarantine

Repo: **bob-cli**. Put the detector in a new module (for example
`src/native/vault_clobber.rs`, declared in `src/native.rs`). Call it from
`run_cycle_inner` in `src/native/vault_sync.rs` after `working_tree_status` and before
`preflight_large_files` / `git add -A`. Dry runs call it too, report-only.

**Candidates** (cheap pre-filter; usually a handful of paths per cycle):

- Include tracked `.md` paths that exist in `HEAD` and are modified in the index or work
  tree.
- Skip added, deleted, renamed, and untracked paths.
- Exclude:
  - `_conflicts/`, `done/`, `_templates/`, and dot-directories (`.bob/`, `.obsidian/`,
    `.trash/`).
  - Canonical daily notes (`YYYY/YYYYMMDD*.md`).
- Reuse existing vault-walker exclusion helpers where they exist.

**Signature.** Let H be the HEAD blob and W the working-tree text. A note is flagged
only if both conditions hold:

1. _Displaced:_ H has ≥ 5 distinct non-blank lines, and ≥ 50% of them are absent from W.
2. _Foreign identity:_ some other live note B (same exclusions, read from the working
   tree) matches at least one of:
   - (a) W's frontmatter `created:` differs from H's and equals B's `created:`.
   - (b) W's `^prj` task line differs from H's and is identical (trimmed) to B's.
   - (c) ≥ 3 task block IDs (excluding `prj`) on W's task lines are absent from H and
     present on task lines in B.

   Find B candidates with `git grep -F -l` over tracked working-tree files, and only
   after (1) holds.

On the incident all three match. H's `created: 2026-10-19T19:32:43-0400` became sase's
`2026-06-01T03:30:02-04:00`, W carries sase's `^prj` line, and all 43 task block IDs new
to W are in `sase.md`, while 88% of H's 66 distinct lines were displaced. Legitimate
flows do not match:

- `bob task move`: the IDs leave B, and it makes its own scoped commits.
- Deleting half a note: no foreign identity.
- New-note copies: added, not modified.
- The `^docs-like-zen` single duplicate: one ID, no displacement.

**Response.** Applied to each flagged note A, naming B and an evidence string such as
`created …, ^prj line, 43 task block IDs shared with sase.md`.

> [!decision] clobber_response = quarantine
>
> - Write W's bytes to a conflict copy, reusing `conflict_copy_path` /
>   `unique_conflict_path` (the existing `write_conflict_copy` reads a git object, so
>   add a bytes variant or hash-object W first).
> - Restore H's bytes into A (temp file plus rename).
> - Append an entry to `_conflicts/sync_conflicts.md` in the existing format, plus the
>   matched note and evidence.
> - Then continue the cycle, so the restore, conflict copy, and log commit together and
>   the clobbered text never reaches other hosts.

> [!decision] clobber_response = warn Leave A as-is so it commits, and append the
> `_conflicts/sync_conflicts.md` entry without a conflict copy, so the log names the
> HEAD commit to restore from.

Both choices also:

- Print a red warning on stderr, even under `-q`.
- Add an additive, `serde(default)` status-record field. `clobbers` holds this cycle's
  detections: `path`, `matched_note`, `evidence`, `action` (`quarantined` / `warned`),
  `conflict_copy?`, `detected_at`. Detections must not vanish 15 s later when the next
  cycle resets per-cycle fields, so also keep a persistent capped `recent_clobbers`
  (last 10).
- Show both in `bob vault-sync status`: in red in the human panel, and in `--json`.
- Fire a best-effort desktop notification that never fails the cycle: macOS
  `osascript -e 'display notification …' with title "bob vault-sync"`, Linux
  `notify-send` when on PATH. Suppress it with an env var in tests. The MacBook runs
  vault-sync headless every 15 s under launchd, so stderr alone is invisible.
- In dry runs, print `would quarantine A (matches B: evidence)` (or `would warn`) and
  change nothing.

**Docs.** Follow the docs route (behavior change → contract/reference; `just docs-gen`
if clap help changes; `just docs-check`). In `docs/vault-git-sync.md`:

- Add the guard as a step in "Sync cycle".
- Mention it in "Conflict policy" and the new fields in "Status checks".
- Add a `## Clobbered notes` section: what it detects, what it does, where to look
  (`_conflicts/sync_conflicts.md`, `bob vault-sync status`), and how to undo a false
  positive (copy the conflict copy back).
- In that section, add the manual runbook for a clobber that was already committed:
  1. Find the commit with `git -C ~/bob log -p -- <note>`.
  2. `git show <commit>^:<note> > <note>`; never `git revert` a mixed vault commit.
  3. `bob task reconcile --dry-run`, then live, to drop stale LINKED FROM / BLOCKS
     mirrors.
  4. `bob ref scan --dry-run`.
  5. `bob vault-sync`.

**Tests.**

- Unit tests for the signature, using a fixture modeled on the incident: a sase-like B
  and a recovery-like A whose W is B plus one rolled row.
- Negative cases:
  - A task-move-style transfer.
  - Deleting most of a note.
  - Daily, `done/`, and `_conflicts/` paths.
  - Notes under 5 lines.
  - A single duplicated task.
  - Added files.
- A cycle test in a temp repo with a temp bare remote, asserting the chosen response:
  - The conflict copy, restore, log entry, status fields, and commit contents.
  - That a dry run writes nothing.

Run `just check`. Do not edit the `Cargo.toml` version or `CHANGELOG.md`; bob-cli ships
through the daily release train, which delivers this to the MacBook.

## Phase ref-scan-copy-hint: Name duplicated-task copies in ref scan failures

Repo: **bob-cli**. Depends on `sync-clobber-guard` for the
`docs/vault-git-sync.md#clobbered-notes` anchor.

1. **Diagnostic detail.** In `src/native/ref_tasks/select.rs` rule 2, when every open
   claimant carries the same `block_id`, keep the existing
   `N open reading tasks claim this ref: a:L, b:L` text and append:
   `; all are copies of one task ^<id> — keep the copy in the ref's parent note and delete the others`.
   Name the parent note when the caller already has the ref's `parent:` (the
   `parent_mismatch` path shows where). Check that callers and tests match on the
   diagnostic `code`, not the detail text (`ref_library/show.rs`, `ref_library/mod.rs`,
   `ref_library/today.rs`, `highlights_ref/overwrite.rs`). Update text-asserting tests.
2. **Scan-level cause line.** In `src/native/highlights_ref/report.rs`, both in the
   human report and in `scan_partial_failure_error`:
   - Group `multiple_open_ref_tasks` plan failures by their sorted claimant-note set.
   - For any group with ≥ 2 PDFs, print one line:
     `hint: 13 refusals share one cause: sase.md and sase_agent_recovery.md carry the same open reading tasks. If one note is an accidental copy, restore it (docs/vault-git-sync.md#clobbered-notes).`
   - Carry the claimant paths structurally on `ScanFailure` instead of parsing error
     strings.
   - Leave the `--json` envelope's fields unchanged; its field order is a documented
     contract. JSON consumers get the richer per-failure text only.
3. **Docs.** Update the diagnostics paragraph in `docs/ref.md` (around the
   `multiple_open_ref_tasks` entry) and link the runbook. Run `just docs-check`.
4. **Tests.**
   - A `select.rs` test: same-block-ID copies get the hint, while distinct tasks
     claiming one ref do not.
   - A report test: 3 refusals over one note pair print one cause line; refusals over
     different pairs print one line per pair.
   - Run `just check`.

## Validation (epic)

- **athena:** `bob ref scan --dry-run` reports 0 failures; `bob vault-sync status` shows
  no error.
- **MacBook** (best effort): `bob ref scan --dry-run` reports 0 failures, and
  `sase_agent_recovery.md` carries the agent-recovery `^prj` line.
- **Plugin:** `npm test` passes in bob-plugins; `bob plugins list` on athena shows the
  new navigation-hotkeys version deployed.
- **bob-cli:** `just check` passes. The temp-repo cycle test shows an incident-shaped
  clobber is handled per `clobber_response`, and legitimate rewrites pass untouched.
