---
tier: tale
title: Album and bank schema saves wait out vault-lock contention
goal:
  A previewed album or bank schema save that meets another Bob maintenance run waits
  visibly and saves on its own, never failing with lock_busy.
size: medium
decisions:
  wait_budget:
    ask: Should a waiting save give up on its own after 10 minutes of lock contention?
    choices:
      ten_minutes:
        The wait ends after 10 minutes with the preview kept, and Enter starts a fresh
        wait
      until_stopped:
        The save keeps retrying until it lands, Esc stops it, or the card closes
    default: ten_minutes
    why:
      Mac highlights scans run up to ~5 min; a bound stops endless bob spawns on a hung
      lock
    answer: ten_minutes
  scan_followup:
    ask:
      Should the implementer file a large follow-up task bead to shorten how long
      writing highlights scans hold bob_sync.lock?
    choices:
      file:
        One large task bead records the Mac cron evidence; this plan still changes no
        bob-cli code
      skip: No bead; only album and bank saves learn to wait
    default: file
    why:
      The scan holds the lock 80-300 s every 15 min with no writes, breaking every 3 s
      writer
    answer: file
proposed_by: bbugyi200.athena.1da
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 07:24:14
status: wip
---

# Album and bank schema saves wait out vault-lock contention

## Problem

Bryan's 2026-11-17 07:00:47 screencast (MacBook) shows the album Schema Card adding the
choice `foo` to a choices field. The dry-run preview succeeds. Enter locks the stage,
and about 3.5 s later the status rule shows a red refusal: "another Bob vault
maintenance run is already active (waited on /tmp/bob_sync.lock) · wait for the other
maintenance run to finish, then retry" with a **Preview again** button. The Emoji Slot
is not involved: this is the shared save path, and bank setup saves have the same
failure.

Root cause, verified in code and on the Mac:

- `bob albums update` and `bob bank config update` write through bob-cli's
  `host_catalog::run_locked` (`src/native/host_catalog/txn.rs`). It waits at most 3 s
  (`LOCK_WAIT`) for `bob_sync.lock`, then returns a JSON failure with code `lock_busy`.
  That is the only `lock_busy` source for these writers, and it comes before the
  pre-sync, the plan, and any write, so an attempt that returns it wrote nothing.
- On the Mac, the `bob-highlights-sync` cron (`:00/:15/:30/:45`) runs a writing
  `bob highlights scan`. That scan takes `bob_sync.lock` before its pre-scan hook and
  holds it through planning all 694 PDFs. The cron log shows each run lasts 78–307 s,
  even with `writes: none`. The run during the screencast went from 07:00:00 to
  07:01:32. The 15 s `bob vault-sync` LaunchAgent and the other crons also take the lock
  briefly.
- In bob-ledger-tools, the shared guarded update session
  (`plugins/bob-ledger-tools/src/242aa-plugin-guarded-update-session.js`) treats
  `lock_busy` like any refusal. It drops the preview, sets `refused`, and requires
  Preview again plus another Enter.

## Approach

Retry inside the shared session; do not lengthen bob's wait.

- Retrying is safe. A decoded `lock_busy` report proves nothing was written, the
  envelope stays identical, and bob's `expect` fingerprint is still the real stale guard
  on every attempt.
- A longer wait inside bob would force the 20 s plugin spawn timeout
  (`BOB_SPAWN_TIMEOUT_MS`) to grow past the longest scan. Killing bob on timeout after
  it took the lock gives an unknown outcome, and Esc could never stop such a wait
  safely. With short attempts, every point between attempts is a safe place to stop.
- One change in the shared controller covers every consumer: the album Schema Card
  (gesture edits, arrangement saves, and Undo applies), the Bank setup card, and the
  Category card's new-category stage.

No bob-cli code changes. All edits are in the **bob-plugins** linked repo (open it with
`/sase_repo`). Read its `AGENTS.md`, edit `src/` fragments only, keep each fragment ≤
1000 lines, and never hand-edit generated `main.js`.

## Changes

### 1. Shared session: `242aa-plugin-guarded-update-session.js`

- Add constants `GUARDED_UPDATE_LOCK_RETRY_MS = 1000` and
  `GUARDED_UPDATE_LOCK_WAIT_BUDGET_MS = 10 * 60 * 1000`. Allow `init` overrides so tests
  can shrink them.

> [!decision] wait_budget = until_stopped Drop the budget constant and the
> budget-exhausted branch. The wait ends only on success, a non-`lock_busy` result, Esc,
> a superseding draft, or close.

- In `apply()`, after `deps.spawnApply` returns: if `outcome.ok === true`,
  `report.ok !== true`, and `failureFromReport(report).code === "lock_busy"`, do **not**
  refuse. Instead:
  - keep `session.preview`;
  - keep `status: "applying"`, so every existing `applying` guard in the cards still
    locks the stage and gestures;
  - set `session.lockWait = { since, attempts, message, stopping: false }`;
  - notify;
  - wait `GUARDED_UPDATE_LOCK_RETRY_MS` with the injected `timers`;
  - call `spawnApply` again with the same `envelopeText` and `ownPath`.

  Only a decoded-report `lock_busy` qualifies. Transport failures, timeouts, and
  `unknown` outcomes keep today's never-blind-retry handling.

- Before each new attempt:
  - Return quietly, as today, if the session closed or the epoch or revision changed.
  - Do not re-run `saveEditors`; the first attempt already saved.
  - If `vaultGeneration()` moved since the confirmed preview, dry-run the same
    `envelopeText` through `deps.spawnPreview` (no lock). Compare the result with the
    confirmed report using a small `guardedUpdateSameConsequence(a, b)` helper. It
    compares what the status rule paints: consequence text, affected and newly-invalid
    counts, derived keys, and per-op outcomes.
    - If the dry run succeeds with the same consequence, adopt it as the preview (new
      generation) and keep waiting.
    - Otherwise, end the wait with the fresh result as an ordinary `preview-ready` (or
      refusal) and `copy.saveInvalidated`, so applying takes another Enter.
- While a wait is active, notify at least once per second with injected timers, so hosts
  can repaint the elapsed time without their own intervals.
- Add `stopWaiting()` to the returned API.
  - It sets `lockWait.stopping` and clears any pending retry or ticker timer.
  - If no attempt is in flight, the wait ends at once.
  - If an attempt is in flight, its result is handled as usual: success is a normal
    save, `lock_busy` ends the wait, and anything else takes today's path.
- When the wait ends without a write (stop, or budget exhausted):
  - set `status: "preview-ready"` with the preview intact, so `canApply` is true again;
  - set `refusal: null`;
  - set a new snapshot field `lockNotice: { message, hint }`. The message is bob's lock
    message (or a stop/budget message); the hint is "Nothing was written. Press Enter to
    try again."

  Enter then goes through the normal `apply()` path and starts a fresh wait if the lock
  is still busy. `lockNotice` clears on the next apply, draft change, or rebase.

- Snapshot: add `lockWait: { since, elapsedMs, attempts, stopping } | null` and
  `lockNotice`. `busy` stays true while waiting.
- `close()` clears the retry and ticker timers, and no attempt starts after close. An
  attempt already in flight keeps today's close-during-save semantics.
- If `setDraft()` or `rebase()` arrives mid-wait, it supersedes the wait: clear the
  timers and drop `lockWait`.

### 2. Apply services: skip refreshes after unwritten attempts

In `242a-plugin-album-update-service.js` (`apiAlbumUpdateApply`) and
`248e-plugin-bank-config-update-service.js` (`apiBankConfigUpdateApply`), when the
decoded report's failure code is `lock_busy`:

- skip the forced post-apply revalidation (`albumRevalidateAfterSync` /
  `bankConfigRevalidateAfterApply`);
- return the current report state as `refreshed`;
- release own-path suppression as today.

Otherwise each retry would force a full `bob albums list` or bank report and could also
bump the generation.

### 3. Hosts: waiting paint and Esc

- **Status line while `lockWait` is set:** "Waiting for another Bob vault maintenance
  run… 42s · saves when it finishes · Esc stops waiting". While `stopping`: "Stopping
  after the current attempt…". Use the neutral pending-style rule, not the red refusal.
  - Album: the status region in `241yc-plugin-album-schema-card-regions.js`, where
    "Saving the schema edit…" paints today, plus
    `241yd-plugin-album-schema-card-paint.js`.
  - Bank setup: the status line in `248kb-plugin-bank-setup-card-regions.js`.
  - Category card new-category stage: `248g-plugin-bank-category-new-stage.js`, painted
    from `onNewCategorySessionChange` state.
- **`lockNotice`:** paints as a neutral notice with **Save ⏎** enabled. It never shows
  the **Preview again** button and never triggers the stale refresh path. The bank setup
  card's `onSessionChange` refreshes on `refusal.kind === "stale"`, so a `lockNotice`
  must not set `refusal`.
- **Esc while `lockWait`:** call `session.stopWaiting()`, consume the key, and keep the
  card open. A later Esc keeps today's meaning.
  - Album `schemaHandleEscape` (`241y-plugin-album-schema-card.js`) returns `"ignore"`
    while applying today. Return `"stay"` after stopping when a wait is active.
  - Bank setup `onKeyDown` (`248k-plugin-bank-setup-card.js`) returns early while
    `busy`. Stop the wait first.
  - The Category card new-category stage closes on Esc even while applying. During a
    wait, Esc stops waiting instead, and the next Esc closes as today.
  - Update the footer and hint-strip key labels where they list Esc, if they are
    computed from state.

### 4. Tests (offline, part of `npm test`)

Session behavior, with fake timers and spawns, goes in
`scripts/test-ledger-tools-album-update.cjs`. Bank wiring goes in
`scripts/test-ledger-tools-bank-config-update.cjs`. Cover:

1. `lock_busy`, then success: the session waits, retries the identical envelope, saves
   once, has Undo, and never calls `saveEditors` twice.
2. Repeated `lock_busy`, then success after several attempts: `attempts` and `elapsedMs`
   advance, and notifications tick every second.
3. `stopWaiting()` with no attempt in flight ends at once with `preview-ready`,
   `canApply`, and `lockNotice`. Enter then retries.
4. `stopWaiting()` during an in-flight attempt that then succeeds: a normal saved
   result.
5. Budget exhausted: `preview-ready` plus `lockNotice` (skip this case under
   `until_stopped`).
6. Generation moves with the same consequence: the wait continues. With a different
   consequence: the wait ends, the fresh preview shows, and nothing is applied.
7. `close()` mid-wait: no further `spawnApply` calls.
8. Transport failure or timeout after a `lock_busy`: today's unknown/transport refusal,
   with no retry.
9. The service skips the forced revalidate on `lock_busy`.

Paint and Esc go in `scripts/test-ledger-tools-album-schema-card-shell.cjs`,
`scripts/test-ledger-tools-bank-setup.cjs`, and
`scripts/test-ledger-tools-bank-category-card-ui.cjs`: the waiting text, Esc stopping
without closing, and `lockNotice` with Save enabled and no Preview again.

If `flock(1)` is available, add a held-lock case to the real-writer acceptance
`scripts/obsidian-album-schema-writer-e2e.cjs`. Hold `BOB_VAULT_SYNC_LOCK_FILE` in a
child for about 6 s, apply through the session, and expect a wait followed by exactly
one commit. Report the case as skipped without `flock` or `BOB_BIN`.

### 5. Docs, version, deploy

- `docs/albums.md`, in the apply paragraph: a save that meets another Bob maintenance
  run keeps its preview, shows the waiting line, retries until the lock frees (bounded
  per `wait_budget`), and Esc stops waiting with nothing written. Add the held-lock case
  to the writer-acceptance sentence if it was added.
- `docs/bob-ledger-tools.md`: add a short clause to the Bank setup line.
- Bump the bob-ledger-tools version in `manifest.json` (patch, since this is a fix) and
  update its row in the `docs/index.md` inventory.
- Run `npm run build`, `npm test`, `npm run docs:check`, then `bob plugins sync` as the
  repo requires. Follow the repo's no-`-F` rule if sync refuses.

### 6. Follow-up bead (`scan_followup`)

> [!decision] scan_followup = file Use `/sase_new_task` to file one `large` task bead:
> "Writing highlights scans hold bob_sync.lock for the whole 80–300 s scan even with
> nothing to write". Include:
>
> - The Mac `~/.local/state/cron/bob-highlights-sync.log` durations.
> - `scan_library` in bob-cli's `src/native/highlights_ref/sync.rs`, which takes the
>   maintenance lock before the pre-scan hook, intake, and parallel planning.
> - The bob-cli-9c.1 rationale: keep vault-sync from pushing a library PDF without its
>   ref note.
> - The collateral damage: every 3 s-budget writer (album and bank updates, themes,
>   relation labels, attachments, periods, task move) fails during the window.
>
> Name the design constraints: preserve the 9c.1 guarantee and the `bob_sync.lock` →
> `scan.lock` order, or justify a new order. The likely direction is lock-free planning,
> then taking the lock only when intake or writes exist, with guarded re-validation.

## Acceptance

- If another process holds `bob_sync.lock` for any time within the budget, Enter on a
  previewed album or bank schema edit shows the waiting line with elapsed seconds. The
  edit saves on its own once the lock frees: one bob commit, no Preview again, and no
  second Enter.
- Esc during the wait stops it with nothing written and the card open. The preview stays
  applicable, and Enter tries again.
- Non-`lock_busy` failures, timeouts, and unknown outcomes behave exactly as before.
- `npm run build:check`, `npm test`, and `npm run docs:check` pass, and the plugin is
  deployed with `bob plugins sync`.
