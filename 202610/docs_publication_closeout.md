---
tier: tale
size: medium
title:
  Close out gated docs publication — rollback, queued-run record, bootstrap staging,
  verify retry, runbook accuracy
goal:
  Fix the remaining epic-caused publication defects, so that the documented rollback
  works, queued runs record safely, bootstrap commits only host files, verify retries,
  and the runbooks state the real failure boundaries. Then close epic bob-cli-a0.5.1 and
  its parent phase bob-cli-a0.5.
proposed_by: bbugyi200.apollo.bob-cli-a0.5.1.land
proposed_with: claude/opus
bead: bob-cli-a0.5.1
create_time: 2026-10-20 09:49:54
status: wip
---

- **PARENT:**
  [202610/docs_gated_publication.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/docs_gated_publication.md)
- **BEAD:**
  [bob-cli-a0.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.5.1.md)

# Close out epic bob-cli-a0.5.1 (gated hourly GitHub Pages publishing)

The epic's five phases are closed and the live launch is proven. The evidence is first
publish 39422837056, no-op 39423369898, drill 39423449416, recovery 39423607269 (issue
#1 closed), and weekly 39427654125. The land review then found a handful of defects that
the epic itself caused in the shipped publisher, operator commands, and runbooks. This
tale fixes them, then performs the epic's closeout and the closure of its parent phase
`bob-cli-a0.5`. It is the final landing step; nothing resumes after it.

Follow-up triage is already done and recorded on `bob-cli-a0.5.1` (the LAND TRIAGE
note). Two items went to the active parent epic `bob-cli-a0` as DISCOVERED ISSUE notes:
the Adobe HTTP 403 in the weekly online-link check, and the hourly schedule that rarely
fires. Do **not** fix either of them here, and do not file tasks for them.

Read before editing:

```bash
sase bead read bob-cli-a0.5.1 -r "Need the epic scope, land triage note, and plan path"
sase memory read docs.md --reason "Editing tools/docs, docs/develop/build.md and CLI help"
sase memory read cli_rules.md --reason "Touching bob-docs CLI help text"
```

## Fixes

### 1. Documented rollback always fails (docs + test)

Both `tools/docs/host/README.md` ("Rollback with held pinned SHAs") and
`docs/develop/build.md` ("### Rollback with held pinned SHAs") tell the operator to edit
only `"hold"` and the source SHAs in `published.json`, then run
`just -- docs-publish --pinned`. The pinned path in
`tools/docs/bob_docs/publish_plan.py`, inside `plan_publication`'s `if pinned:` branch,
rebuilds inputs at those SHAs. It then requires `fingerprint == state.fingerprint`, and
`_load_published` also validates the fingerprint against the stored `inputs`. Edited
SHAs therefore always fail with "pinned source contents do not match the published
fingerprint", or an earlier inputs/fingerprint validation error.

Keep the code: the fingerprint check is a deliberate integrity check. Rewrite both
rollback sections to the procedure that works:

1. In the deploy repository, find the manifest commit to roll back to:
   `git log --format='%h %ad %s' -- published.json`. Each
   `Record documentation publication` commit names its run.
2. Restore that record whole: `git show <commit>:published.json > published.json`. Then
   set `"hold": true`, commit only `published.json`, and push.
3. `just -- docs-publish --pinned` builds exactly those SHAs and keeps the hold.
   Scheduled and unpinned runs, including `--force`, skip while held.
4. Release it by setting `"hold": false`, commit, and push. The next ordinary run
   publishes the current heads.

Say explicitly that hand-editing SHAs is rejected, and why.

Add a regression test to `tools/docs/tests/test_publish_plan.py` using the existing
helpers (`_fixture`, `_plan`, `_record_publication`, `_commit_and_push`):

- Record publication A.
- Change a docs file, push, plan, and record publication B as the manifest.
- Write A's full recorded manifest back with `hold: true`.
- Run `_plan(fixture, pinned=True)`. Assert `should_build`, reason `pinned`, the bob-cli
  SHA equal to A's, and a candidate `hold` of true.
- Assert that the hand-edit variant raises `DocsError`: B's manifest with only the
  bob-cli SHA swapped to A's and hold set to true.

### 2. A queued run's manifest push is rejected after deploy (workflow + tests)

The `record` job in `.github/workflows/docs-publish.yml` uses `actions/checkout@v6` with
no `ref`, so it checks out the caller's trigger commit (`github.sha`). The `publish`
job, however, plans from `ref: ${{ github.ref }}`, the branch head at plan time.

That breaks when a run waits in the `pages` concurrency group behind another run that
records a manifest commit. The waiting run then plans from the newer manifest and
deploys, but `record` commits on top of its stale trigger SHA. The push is rejected only
_after_ the new site is live, and a spurious blocked issue opens.

Fix it with an explicit compare-and-swap on the revision the plan actually read:

- In the `publish` job, after the deploy checkout, emit
  `deploy_sha=$(git -C "$GITHUB_WORKSPACE/deploy" rev-parse HEAD)` to `$GITHUB_OUTPUT`.
  A small step with an `id` works, or the plan step. Expose it as a `publish` job output
  named `deploy_sha`.
- In `record`, check out `ref: ${{ needs.publish.outputs.deploy_sha }}` and keep
  `fetch-depth: 0`. That leaves a detached HEAD, so push with
  `git push origin "HEAD:refs/heads/${{ github.ref_name }}"`. Never force.
- Any change to the deploy branch between planning and recording is then rejected as a
  non-fast-forward. That includes an operator's hold commit, which must never be
  overwritten. A run queued behind an earlier recorder now succeeds.
- Keep the existing guarantees and their tests: only `git add published.json`, no force
  push, the no-change skip, and the bot identity args.

Update `tools/docs/tests/test_publish_workflow.py`:

- `test_record_script_pushes_only_manifest_and_preserves_remote_changes` must also
  substitute `${{ github.ref_name }}` with `main`. Detach the record checkout at the
  planned deploy SHA to mirror the new checkout.
- Add a "queued behind an earlier recorder" case. Another commit to `published.json`
  lands on the remote _before_ the deploy SHA is taken, and is newer than the run's
  original trigger commit. The record must succeed and push exactly one manifest-only
  commit on top.
- Keep the concurrent-hold case: an operator commit after the deploy SHA must produce
  `[rejected]` and leave the operator's manifest on the remote.
- Add a YAML-aware assertion that the `record` checkout `ref` is
  `${{ needs.publish.outputs.deploy_sha }}` and that `publish.outputs` contains
  `deploy_sha`.

### 3. `host-bootstrap --deploy-dir` commits the whole index (code + test)

`_commit_host_files` in `tools/docs/bob_docs/host.py` stages the maintained paths. It
then checks `git diff --cached --quiet` and runs a plain `git commit`, so anything the
operator had already staged in that checkout is committed and pushed as "Sync
documentation host templates". That could include an edited `published.json`. The plan
forbids staging unrelated files through the bootstrap.

Fix:

- Scope the change check to `git diff --cached --quiet -- <paths>`.
- Commit with a pathspec (`git commit -m ... -- <paths>`, which implies `--only`).
- Add a `tools/docs/tests/test_host.py` apply test using the real-git `FakeHost`
  pattern. Pre-stage an unrelated change, for example a modified `published.json` or a
  new file, in the deploy checkout. Apply. Assert that the pushed commit contains only
  the maintained files, and that the unrelated change is still staged locally and absent
  from the remote.

### 4. Live `/urls.txt` precheck has no retry (code + test)

`verify_publication` in `tools/docs/bob_docs/publish_verify.py` calls
`fetch_published_urls(origin, timeout=timeout)` once. A transport blip, a 5xx (which
raises `DocsError`), or a transient 404 right after deploy fails live verification and
opens the blocked issue, even though every per-page check already retries through
`get_with_retries`.

- Wrap that precheck in the same bounded policy: `retries + 1` attempts, retrying on the
  `"offline"` status, on `DocsError`, and on `"first-publish"` (404), with the same
  `min(2.0 * attempt, 10.0)` back-off.
- Make `sleep` injectable as `get_with_retries` already does, and leave
  `fetch_published_urls` itself unchanged, because `run_urls` in docs-check relies on
  its single-shot first-publish semantics.
- Add focused tests in `tools/docs/tests/test_publish_verify.py`: a transient failure
  then success passes, and persistent failure still returns 1 after the bounded
  attempts.

### 5. Runbook accuracy (docs only)

These apply to both `tools/docs/host/README.md` and `docs/develop/build.md`. Keep them
consistent with each other and with the plan's failure boundaries.

- **Failure boundaries / Recovery.** They currently say a deploy or record failure
  "keeps the previous live site", and that a deploy failure "leaves the last good site
  live". The rule is: only failures _before_ deploy (planner, hold skip, gates, and the
  drill) promise the last good site and manifest are untouched.
  - A Pages deploy failure normally leaves the previous deployment serving, but the
    workflow does not prove it. Tell the operator to check `just docs-status`, which
    reads live `/build.json`.
  - A record failure happens after a successful deploy. The new site is live, but
    `published.json` still names the previous publication. This includes the
    compare-and-swap rejection from fix 2.
  - A live-verification failure means the new site may be serving.
  - All of these keep the blocked issue open. The next ordinary run takes the recovery
    path: it rebuilds, deploys, records, verifies, and closes the issue.
  - Mention that `record` refuses to overwrite a deploy-repo change made after planning.
- **Inactivity.** "after 60 days without a run" is wrong. GitHub disables scheduled
  workflows in a public repository after 60 days with no repository activity. No-op
  publish runs commit nothing, so they do not count.
- **`publish-verify --expected` help** in `tools/docs/bob_docs/cli.py` still says
  "planner next-publication.json". The workflow passes the recorded `published.json`
  from the publication handoff, so say that. Also update any matching text in
  `tools/docs/README.md`.

### 6. Recovery count ignores the managed title (workflow + test)

The plan step in `docs-publish.yml` counts every open issue labelled `docs-publish`.
Match `publish_issue.py`'s managed identity instead:
`--json number,title --jq '[.[] | select(.title == "Docs publish blocked")] | length'`.
Extend `test_plan_reads_blocked_issues_and_recovers` to assert the title filter.

### 7. Dead code

Before deleting each symbol, grep all of `tools/docs`, tests included, to confirm it is
unused:

- the `timezone` import in `publish_plan.py`; the `"timezone"` in an error string is not
  a use
- the `DRILL_MARKER` import in `check.py`
- `_FRAGMENT_PATTERNS` in `publish_verify.py`
- `REUSABLE_WORKFLOW` and `WEEKLY_REUSABLE_WORKFLOW` in `host.py`
- `inspect_workflow_state` and `inspect_weekly_state` in `host.py`
- the never-false `ensure_managed_label` parameter of `run_publish_report` in
  `publish_issue.py`; keep the label provisioning itself unconditional

## Sync the host README

`tools/docs/host/README.md` is rendered into the public host repo by the bootstrap. The
parent plan already authorizes the bootstrap's narrowly scoped commit and push. After
fixes 1, 3, and 5 pass their tests:

1. Open the host with
   `sase repo open gh:bobs-org/bobs-org.github.io -r "Sync corrected host README via bootstrap"`
   and confirm its checkout is clean and at `origin/main`.
2. Run `just docs-host-bootstrap` to preview. It must show only `README.md` changing,
   with `published.json` untouched.
3. Run `just -- docs-host-bootstrap --yes --deploy-dir <printed path>`.
4. Rerun the preview. It must be a no-op, and `published.json` must be byte-identical.

Do not dispatch any publish or weekly run in this tale.

## Verification

- `uv run --project tools/docs pytest tools/docs/tests -q`: the full docs suite. Run the
  focused files first.
- `just docs-check`, all gates. It takes several minutes, so run it in the foreground
  with a generous timeout.
- `just check`, the file-change gate. The PTY test
  `bash_readline_inserts_what_bob_returned` is a known parallel-run flake (task
  `bob-cli-8q`). If it alone fails, rerun it in isolation and record that; do not chase
  it.
- `git diff --check`.

## Final step: close the epic and its parent phase

1. Run `sase bead epic-symbols bob-cli-a0.5.1`. It currently has no entries. If any
   appear, resolve each one: wire it up, privatize it, or delete it. Re-key one only if
   a still-open later bead genuinely needs it.
2. Close the epic:

   ```bash
   sase bead close bob-cli-a0.5.1 --note "<verification>"
   ```

   The note must cover:
   - all five phases verified against source and commits (d18b45de, 9ea12d2d, d5c318a9,
     dadf0734, 110624b8, 8783d498, 1b1758fd, d51c160e, 2f9ad5bd);
   - the live evidence: first publish 39422837056 with manifest 5910ffb, no-op
     39423369898, drill 39423449416, recovery 39423607269 closing issue #1 with manifest
     38773b6 and an unchanged fingerprint, weekly 39427654125 with Material passing and
     issue #2 open on the Adobe 403;
   - live `/build.json` provenance (bob-cli d51c160e, bob-mac-capture 532c376c,
     bob-plugins ff369873, all clean);
   - the host templates matching the bob-cli templates;
   - this tale's fixes and their test, docs-check, and just check results;
   - the host README sync commit;
   - the triage outcomes: runs-on resolved; Adobe 403 and hourly schedule routed to
     bob-cli-a0; the pinned-tooling risk declined as by design;
   - the current count of scheduled runs, from
     `gh api "repos/bobs-org/bobs-org.github.io/actions/runs?event=schedule" --jq '.workflow_runs | length'`.
     If scheduled runs have appeared since 13:43Z on 2026-10-20, append a short
     `sase bead note bob-cli-a0` update to its hourly-schedule DISCOVERED ISSUE.

   Never use `--force` to make the close succeed.

3. Run `just symvision` if that recipe exists. The justfile currently has none, so
   record that it is unavailable.
4. Set `status: done` in the frontmatter of the epic's plan file
   (`plan:202610/docs_gated_publication.md`, at the EPIC PLAN path printed by
   `sase bead read bob-cli-a0.5.1`).
5. The epic's `parent_bead` is the phase `bob-cli-a0.5` ("Deploy repo, hourly gated
   publish, and launch"). Its required work is done: the reusable publish and weekly
   workflows, the bootstrapped `bobs-org/bobs-org.github.io`, the launched
   https://bobs-org.github.io/, and the drill-proven fail-safe path.
   - Run `sase bead epic-symbols bob-cli-a0.5` and resolve any entries.
   - Run
     `sase bead close bob-cli-a0.5 --note "<verified run URLs, live provenance, drill/recovery, weekly result with the Adobe 403 clean-base exception, and this closeout>"`.
   - Close only that phase. Leave `bob-cli-a0` and `bob-cli-a0.13` to their own land
     agents.
