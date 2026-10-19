---
tier: tale
title: Show bob ref create references in Bob Refs within seconds
goal:
  A ref created on athena or apollo appears in the open Mac Refs panel within seconds,
  with no reopen.
size: medium
decisions:
  arrivals:
    ask:
      Should references that arrive while the Refs panel is open join the visible list
      on their own?
    choices:
      join:
        New refs appear in the open panel within seconds; rows below an arrival may
        shift down one
      hint:
        Rows never move while open; a footer 'N new · ⌘R' hint shows until you press ⌘R
        or reopen
    default: join
    why:
      The pain is an open panel that never shows the new ref; selection stays id-based
    answer: join
  intake_cadence:
    ask:
      Should athena and apollo drain their xlib/ intake queue every 2 minutes instead of
      every 10?
    choices:
      two_min:
        Queued creates reach the Mac within ~2 min; one extra no-op bob spawn per host
        every 2 min
      keep: The rare create that falls back to the queue can still wait up to 10 minutes
    default: two_min
    why:
      An empty queue takes no lock and does no sync, so the faster cadence costs almost
      nothing
    answer: two_min
proposed_by: bbugyi200.athena.123
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 15:49:08
status: wip
---

# Make `bob ref create` references show up in Bob Refs within seconds

## Goal

A reference created by `bob ref create` on athena or apollo should appear in the Mac's
Bob Refs panel within a few seconds of the panel being open, without a reopen. Today it
can take 15 s to well over a minute, and up to 10 minutes when the create falls back to
the intake queue.

## What was measured (2026-10-18/19)

Ingest-at-birth works, and Git transport is not the bottleneck:

- Each create commits and pushes about 2 s after it starts writing. For example,
  `e11d401` was committed at 15:34:07 and pushed at 15:34:09.
- Across 12 create/ingest commits, the Mac's `origin/master` picked them up 5–28 s after
  the commit (median about 13 s). That lag is the Mac LaunchAgent's `StartInterval` of
  15 s plus a roughly 0.9 s `bob vault-sync` cycle.
- After the Mac's fast-forward, the Refs watcher refresh finishes in about 1 s. Mac
  signposts: fast-forward at 15:34:23, `refs-refresh` ended at 15:34:24.3. The watcher
  relevance filter added in bob-mac-capture `561d777` is not at fault, because checkouts
  touch visible `ref/` and `lib/` paths.

Where the time actually goes:

1. **The open panel never shows arrivals.** `RefsPanelModel` freezes its listing while
   visible ("late library data refreshes row content in place and never reorders"; see
   `libraryDidPublish`). In the 15:34 trace, Bryan opened the panel at 15:34:20, three
   seconds before the Mac pulled. The new ref stayed invisible until he reopened the
   panel at 15:34:58, about 51 s after the create.
2. **⌘R cannot pull.** ⌘R re-runs `bob ref list` against the Mac's local tree. Inside
   the 15 s poll window it re-ranks the same stale data.
3. **Queue fallback waits for a 10-minute timer.** On 10-19,
   `chat/mac_sshd_finder_automation_prompt` went to athena's `xlib/` instead of being
   ingested. It was filed by `bob-ref-intake.timer` at 12:30:40.

## Changes

### A. bob-mac-capture: pull the vault when the Refs panel opens, and on ⌘R

- Add a pull call to `RefsFetching` in `Sources/RefsCore/RefsFetching.swift`. Give it a
  protocol-extension default that throws `RefsFetchingError.unsupported`, so the
  existing fake fetchers keep compiling; `linkToToday` uses the same pattern.
  `BobRefsFetcher` runs `bob vault-sync -q` through `BobProcessClient` with the default
  20 s timeout.
  - This is plain process orchestration, so it stays within the thin-client decision
    (`decisions:mac-capture-is-a-thin-client`). The app already runs bob commands that
    sync, such as `bob ref rename`.
  - `bob vault-sync` returns quietly when `bob_sync.lock` is held, so the pull never
    fights the 15 s LaunchAgent, a scan, or a rename.
- Add a `refs-pull` lane to `RefsLibrary` in
  `Sources/BobMacCapture/Refs/RefsLibrary.swift`:
  - Single-flight: a trigger that arrives while a pull is running joins that pull.
  - Throttle panel-open pulls: skip one if the last pull started less than 5 s ago.
  - Skip while a scan or rename is in flight.
  - On failure, log a bounded `NSLog` and do nothing else. Never touch `refreshState`,
    banners, or the footer.
  - Wrap each pull in a `refs-pull` `CaptureSignpost` interval.
  - The pull does not refresh anything itself. The existing vault watcher sees the
    fast-forward and runs the normal snapshot refresh.
- Trigger points:
  - Every Refs panel show, including the Highlights Tabs source: start the pull from
    `RefsPanelModel.refreshForOpen`, alongside the Today refresh. Do not block
    presentation.
  - ⌘R (`case .refresh` in `RefsPanelModel.perform`): await the pull (single-flight, not
    throttled), then run the existing `library.refresh(reason: .manual)` with
    `pendingRefreshRerank`. If the pull fails or times out, still run the refresh.

### B. bob-mac-capture: let arrivals join the open listing (decision `arrivals`)

> [!decision] arrivals = join

- Record the set of library item ids each listing was ranked from. Every re-rank path
  that calls `libraryListing()` or rebuilds the Tabs listing must set it:
  `rerankSelectingFirst`, `freshListingKeepingSelection`, `freshListing(selecting:)`,
  `queryDidChange`, scope/Newest changes, and the post-scan/rename paths. Centralize
  this in one place.
- In `libraryDidPublish`, compute arrivals as current `library.items` ids minus that
  set. Re-rank with `freshListingKeepingSelection()` when all of these hold:
  - arrivals is non-empty;
  - the Tabs filter is off;
  - no rename, Today-link, or copy card is open or in flight, and there is no pending
    open;
  - the selected id is nil or still available.

  Otherwise keep the frozen listing; the arrivals join at the next re-rank. Post a
  VoiceOver announcement ("1 new reference" / "N new references") when arrivals join.

- Content-only publishes (same ids) still update in place and never reorder.
  Vanished-row dimming and "No longer in your library" keep working as they do today.

> [!decision] arrivals = hint

- Use the same arrival detection, but never re-rank. Publish an arrival count and have
  the footer show `N new · ⌘R` (follow the existing `RefsFooter` styling). ⌘R, a query
  edit, or a reopen re-ranks and clears the count.

Either branch: update the README `### Updates` paragraph in bob-mac-capture, which
currently says late data "never reorders". Document the new arrival rule, the
`refs-pull` lane (panel show and ⌘R, 5 s throttle, failures only logged), and the new
signpost in the Signposts list.

### C. Intake cadence and docs (decision `intake_cadence`)

> [!decision] intake_cadence = two_min

- chezmoi `home/dot_config/systemd/user/bob-ref-intake.timer`: set
  `OnUnitActiveSec=2min` and update `Description`.
- In `home/.chezmoiscripts/run_onchange_after_enable_bob_ref_intake_timer.sh.tmpl`, keep
  `enable` and add `systemctl --user restart bob-ref-intake.timer` after
  `daemon-reload`, so an already-active timer is rescheduled. The embedded sha256 lines
  already make the script re-run when the timer changes.
- bob-cli docs: change "every ten minutes" to "every two minutes" in
  `docs/vault-git-sync.md` (Highlights intake) and `docs/highlights-ref-sync.md` (intake
  timer paragraph).

> [!decision] intake_cadence = keep

- Leave the timer and the cadence docs alone.

Either branch: in bob-cli `docs/vault-git-sync.md` under Background triggers (the
MacBook part), add one sentence. Bob Mac Capture also runs `bob vault-sync -q` when the
Refs panel opens (throttled to one pull per 5 s) and before a ⌘R refresh, so references
pushed from athena or apollo show up without waiting for the 15 s LaunchAgent.

## Out of scope (considered and rejected)

- **A publisher-side SSH nudge to the Mac after each create push.** It adds cross-host
  coupling and does nothing while the Mac sleeps. Pull-on-show covers the moment that
  matters. This leaves `decisions:references-are-ingested-at-birth` untouched.
- **Lowering the Mac LaunchAgent interval below 15 s.** That triples GitHub `ls-remote`
  traffic and Mac wakeups all day just to help the rare moment the panel is open.

## Verification

- Tests: add XCTest cases in `Tests/BobMacCaptureTests/RefsLibraryTests.swift` and
  `RefsPanelModelTests.swift`, using the existing fake fetcher pattern.
  - Pull lane:
    - a panel show calls pull once;
    - a second show within 5 s does not;
    - concurrent triggers share one pull;
    - ⌘R awaits the pull before the refresh;
    - a failed or unsupported pull still refreshes and leaves `refreshState` and banners
      untouched;
    - no pull runs during a scan or rename.
  - Arrivals (`join` branch):
    - a publish with a new id re-ranks and keeps the selected id;
    - a content-only publish does not re-rank;
    - an open rename, Today-link, or copy card blocks the re-rank;
    - the Tabs filter blocks it;
    - a vanished selection blocks it.
  - Arrivals (`hint` branch): assert the count, the footer text, and that ⌘R clears it.
  - Async tests must not await untimed continuations (see bob-mac-capture `AGENTS.md`).
- Compile and check, following bob-mac-capture `AGENTS.md`:
  - Run `just portable-check` on athena.
  - Then compile a scratch rsync copy on the mac (without `.build`/`.git`), run
    `bundle.sh` and `just typecheck-tests`, and delete the copy.
  - Never use Bryan's checkout or `just install` on the mac.
  - CI runs the tests. If the mac is offline, say in the final report that the macOS
    targets are unverified.
- bob-cli: docs-only. Run `just check` (or at least the docs and format checks it
  includes).
- chezmoi (`two_min`): run `chezmoi apply` on athena and `ssh apollo chezmoi update` (or
  apply), then confirm the 2-minute cadence with
  `systemctl --user list-timers bob-ref-intake.timer` on both.
- Manual acceptance, for Bryan after he installs the app:
  1. Run `bob ref create` on athena.
  2. Open Bob Refs on the Mac right away.
  3. The new row should appear within about 3 s without a reopen.
  4. `log show --signpost --predicate 'subsystem == "org.bobs.bob-mac-capture"'` should
     show `refs-pull`, then `refs-refresh`.
