---
tier: epic
title: Keep → Inbox footer indicator
goal: "Obsidian's desktop status bar shows, beside the review footer, a gold bulb pill
  counting the Google Keep notes a default `bob gkeep pull` would act on (only once Keep
  has verifiably worked on this host) and a plain inbox-tray count of open top-level
  tasks in gkeep_inbox.md. bob owns the Keep count, its cache, and its check budget; the
  footer is a calm, attention-gated client.

  "
decisions:
  auto_checks:
    ask:
      Turn attention-gated automatic Keep checks on by default (gkeep.check_interval
      30m)?
    default: true
    why:
      bob's gates cap it near 16 checks per attended day, and check_interval off is a
      kill switch
    answer: true
  decision_record:
    ask:
      Add a decisions-web record that bob owns the Keep count, its cache, and its check
      budget?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: keep-adapter-hardening
    title: Keep adapter hardening
    depends_on: []
    size: small
    description:
      "keep-adapter-hardening: bound gkeepapi's endless 429 retry, classify rate-limit
      403s as rate_limit, stop network blips from forcing a full resync, and drop the
      redundant state save in the Python adapter, with self-tests."
  - id: keep-status-cache
    title: Host-local Keep status cache and its writers
    depends_on: []
    size: medium
    description:
      "keep-status-cache: add the 0600 status.json cache under bob's state dir with a
      default-pull observation helper, and make list, pull (dry and real), doctor, and
      login record observations, failures, and pull invalidation."
  - id: keep-status-command
    title: bob gkeep status with bob-owned check gates
    depends_on:
      - keep-adapter-hardening
      - keep-status-cache
    size: medium
    description:
      "keep-status-command: add `bob gkeep status [-b] [-f] [-m] [-r]`, the
      never-prompting token read, freshness/floor/single-flight/busy/budget/backoff
      gates, gkeep.check_interval, the JSON and human output, the catalog row and gkeep
      invalidation tag, and docs."
  - id: command-mode-preview-api
    title: Command Mode preview API
    depends_on: []
    size: small
    description:
      "command-mode-preview-api: add `api.commandMode.preview(argv)` to bob-command-mode
      so a sibling plugin can open the sheet straight into a row's preview stage, with
      tests and docs."
  - id: keep-indicator-view
    title: Ledger-tools Keep inbox status-bar item
    depends_on:
      - keep-status-command
    size: medium
    description:
      "keep-indicator-view: add the bob-keep-inbox status-bar item to bob-ledger-tools
      with a pure view model, the live Tasks-cache inbox count, cache-only Keep reads,
      the full visual design, the inbox click, tests, and docs."
  - id: keep-indicator-actions
    title: Attention-gated Keep checks and indicator actions
    depends_on:
      - command-mode-preview-api
      - keep-indicator-view
    size: medium
    description:
      "keep-indicator-actions: add attention-gated `bob gkeep status -r` triggers, Check
      now, the bulb click into a previewed pull, the context menu and commands, the
      sign-in notice, docs, and the optional decision record."
proposed_by: bbugyi200.apollo.research.2y.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-31 13:32:39
status: wip
---

- **PROMPT:**
  [prompts/202610/gkeep_footer_indicator.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/gkeep_footer_indicator.md)

# Plan: Keep → Inbox footer indicator

## Context

Bryan wants an Obsidian footer indicator showing how many Google Keep notes
`bob gkeep pull` would sync, shown only when Keep auth works, checked rarely enough not
to burden the unofficial Keep API or the MacBook, plus a visually distinct, much more
frequent count of non-closed tasks in `~/bob/gkeep_inbox.md`. He accepted every
requirement adjustment (A1–A9) in the research report
`research:202610/gkeep_inbox_footer_indicator/gkeep_inbox_footer_indicator.md` (read it
with `sase artifact read` for background, figures, and evidence). This plan is the
design of record: where it differs from the report, follow this plan.

Deliberate refinements over the report:

- The status cache lives in bob's **state** dir (beside `pull.lock` and
  `journal.jsonl`), not the cache dir, because it holds backoff, budget, and
  last-known-good auth that must survive cache cleaners.
- `busy` and `in_flight` are transient booleans, not states, so they never overwrite the
  persisted state.
- Writers use an ordered, lock-protected read-modify-write with pull invalidation, so a
  slow status check can never overwrite a newer pull result.
- Each footer segment has its own tooltip and click: what you hover is what you learn
  about and what you click.
- No shared eligibility helper is extracted: every observation runs the real planner
  (`plan::classify_with_imports`) with default options, so the footer can never disagree
  with `bob gkeep list`.

## Design shared by every phase

### What Bryan sees

```text
⟳ Review 4 due · 2 commitments · ]s next            (💡 3) › 📥 1      312 words …
└──────────── existing review footer ────────────┘   └─ new item ─┘
```

- One status-bar item, `bob-keep-inbox`, registered immediately after the review footer.
  The review footer has `margin-right: auto`, so the new item is the first item of the
  right-hand cluster: review on the left, intake on the right. It never flexes
  (`flex: 0 0 auto`); the review footer shrinks first.
- **Keep segment** (remote, not yet in the vault): Lucide `lightbulb` plus a tabular
  numeral in a pill. Gold-filled when there is work, outlined when clear, dashed when
  last-known.
- **Connector**: Lucide `chevron-right` in `--text-faint`. It says what a pull does:
  things move from the left box into the right one. It shows only when both segments
  show.
- **Inbox segment** (local, awaiting triage): Lucide `inbox` in `--text-muted` plus a
  numeral in `--text-normal`, weight 600, no fill.
- Distinct beyond color: different icons, filled pill versus plain numeral, left versus
  right of the arrow, and staleness styling only on the remote side. In grayscale the
  pill alone still separates them. No words on the bar; words live in tooltips and
  `aria-label`s. No red anywhere. Background checks never animate.

### Two counts, two clocks

- The **inbox count** never leaves Obsidian: it is recomputed from the Tasks plugin
  cache on every `obsidian-tasks-plugin:cache-update`, with no subprocess and no timer.
- The **Keep count** comes only from bob. `bob gkeep status` prints a host-local cached
  observation; `bob gkeep status -r` contacts Google only when bob's gates say a check
  is due and allowed. Bryan's own `list`, `pull`, `doctor`, and `login` runs refresh the
  cache for free.
- The plugin never reads bob's cache file, never reads a credential, and never decides
  backoff. It asks bob, and only while someone is using Obsidian.

### Count definitions

- **Keep count** = the actionable set of a _default_ `bob gkeep pull` (no `-i`, `-p`,
  `-S`, `-l`, `-R`): live Home notes classified by `plan::classify_with_imports` with
  `PlanOptions::default()` plus the default gkeep URL routing, counting states `new`,
  `revised`, and `pending`. This equals `list`'s footer total, so Keep 0 ⇔
  `✓ Keep inbox is clear`. Breakdown (a partition of the total):
  - `new`: `NoteState::New` notes whose action is not `CreateRef` (become tasks).
  - `links`: `CreateRef` notes (URL-only, clipped into the reading queue).
  - `revised`: `NoteState::Revised`.
  - `pending`: `NoteState::Pending` (already imported; archive only).
  - Stays (not counted): `pinned`, `shared`, `empty`.
- **Inbox count** = top-level Tasks-plugin tasks (they already pass the `#task` global
  filter) in the configured `gkeep.target` (default `gkeep_inbox.md`, vault-relative)
  whose status symbol is not `x`, `X`, or `-`. Blocked `[?]`, Next `[*]`, and In
  Progress `[/]` count. Untagged checklist children never count. bob's equivalent is
  `ledger::read_target_tasks_with_imports` filtered by `list.rs`'s `is_open_status`.
- Unknown is never 0: JSON uses `null`, the bar shows `–`.

**Parity fixture** (identical text in a bob-cli test and a bob-plugins test; both must
count **5**: the bank, blocked, next, in-progress, and grocery rows):

````markdown
---
parent: "[[inbox]]"
---

# Keep inbox

- [ ] Call the bank #task
- [?] Blocked thing #task
- [*] Next thing #task
- [/] In progress thing #task
- [x] Done thing #task
- [x] Done uppercase #task
- [-] Cancelled thing #task
- [ ] Grocery list #task
  - [ ] milk
  - [ ] eggs
  - [ ] Hand-nested subtask #task
- [ ] Untagged top-level line

```text
- [ ] Fenced #task line
```
````

### `bob gkeep status` contract

```text
bob gkeep status [-b|--bob-dir DIR] [-f|--format human|json] [-h|--help] [-m|--max-age DURATION] [-r|--refresh]
```

- Default is **cache only**: no token read, no adapter, no network, no ledger scan. It
  prints the cached Keep observation and a live inbox count read from the target file
  alone.
- `-r` runs at most one never-prompting Keep check if the gates below allow it.
- `-m DURATION` (`0`, `90s`, `15m`, `2h`) overrides the freshness gate for this call;
  `-r -m 0` is a **forced** check.
- Exit 0 whenever a status document prints, because states are data. Exit 2 for clap
  usage errors and for unreadable or invalid gkeep config (with a JSON error document
  under `-f json`:
  `{"schema_version":1,"ok":false,"error":{"kind":"config","message":…,"hint":…}}`). A
  missing config file, missing `gkeep:` section, or missing `gkeep.email` is not an
  error: it prints `configured: false`, exit 0.

JSON (`schema_version: 1`, additive only, no titles, bodies, or secrets):

```json
{
  "schema_version": 1,
  "ok": true,
  "configured": true,
  "keep": {
    "state": "ok",
    "verified": true,
    "stale": false,
    "in_flight": false,
    "busy": false,
    "checked_at": "2026-10-31T14:03:11Z",
    "next_check_at": "2026-10-31T14:33:11Z",
    "source": "status",
    "pull": { "total": 3, "new": 1, "links": 1, "revised": 0, "pending": 1 },
    "stays": { "pinned": 1, "shared": 0, "empty": 0 },
    "error": null,
    "refresh": { "ran": true, "skipped": null }
  },
  "inbox": { "path": "gkeep_inbox.md", "exists": true, "open": 1 }
}
```

- `state`: `unconfigured`, `unverified` (never succeeded for this config), `ok`,
  `offline` (network or deadline), `locked` (token unreadable without a prompt),
  `rate_limited`, `auth` (credentials rejected), or `error` (adapter crash, protocol,
  internal, dependency).
- `verified`: a Keep check has succeeded on this host for the current config key.
- `stale`: the count is last-known, not current: a failure state with a prior
  observation, or a pull changed Keep after the observation started.
- `in_flight`: another `status -r` check holds the single-flight lock. `busy`: a
  `bob gkeep pull` holds `pull.lock`. Both are transient and never persisted.
- `checked_at`: when the last successful observation's snapshot started; `null` when
  none.
- `next_check_at`: the earliest time an automatic `-r` could contact Keep; `null` when
  automatic checks will not run (`check_interval: off`, `auth`, `unconfigured`). Clients
  schedule from it and never re-implement backoff.
- `source`: which command produced the observation (`status`, `list`, `pull`, `doctor`,
  `login`).
- `pull`/`stays`: `null` until an observation exists.
- `error`: `null` in `ok`, else `{"kind","message","hint","since","failures"}`; the
  message is one scrubbed line.
- `refresh`: present only with `-r`:
  `{"ran": bool, "skipped": null | "unconfigured" | "auth" | "busy" | "in_flight" | "floor" | "backoff" | "off" | "fresh" | "budget"}`.
- `inbox.open`: `null` when the target is missing (`exists: false`).

Human output (colored through `Styler`, plain when piped):

```text
💡 Keep    3 to pull · 1 new · 1 link → reading queue · 1 pending archive · 1 pinned stays   checked 4m ago
📥 Inbox   1 open in gkeep_inbox.md
→ bob gkeep pull
```

Other Keep lines: `✓ clear · 1 pinned stays   checked 4m ago`;
`3 to pull, as of 14:02 · offline since 14:05 · next try about 14:45` plus a `hint:`
line; `not checked yet · run bob gkeep status -r`;
`sign-in expired · run bob gkeep login`;
`not set up on this host · run bob gkeep login`. With `-r`, one dim line says what
happened (`checked Keep just now` or `skipped: fresh · next check 14:33`). The
`→ bob gkeep pull` line prints only when the total is above 0.

### Status cache

- File: `${XDG_STATE_HOME:-~/.local/state}/bob-cli/gkeep/status.json` (via
  `bob_env::bob_cli_state_dir()`), dir `0700`, file `0600`, written by temp file plus
  same-dir rename. Host-local; never in the vault. It never holds a token.
- Locks in the same dir: `status.lock` (short, blocking with a 5 s cap) guards every
  read-modify-write; `status-check.lock` (non-blocking, held for a whole `-r` check) is
  the single-flight lock. Create both like `bob_env::create_state_lock_file`.
- Persisted document (`cache_version: 1`; a missing, corrupt, or mismatched-config file
  loads as empty):

  ```json
  {
    "cache_version": 1,
    "config_key": "<sha256 hex of email, device_id, token_command, target joined by NUL>",
    "verified": true,
    "observation": {
      "started_at": "…",
      "source": "list",
      "pull": { "total": 3, "new": 1, "links": 1, "revised": 0, "pending": 1 },
      "stays": { "pinned": 1, "shared": 0, "empty": 0 }
    },
    "invalidated_at": null,
    "failure": {
      "state": "offline",
      "kind": "network",
      "message": "…",
      "hint": "…",
      "since": "…",
      "last_at": "…",
      "count": 2
    },
    "last_contact_at": null,
    "budget": { "day": "2026-10-31", "used": 3 }
  }
  ```

- Ordering rules, applied under `status.lock`:
  - An observation applies only if its `started_at` is at least the current
    observation's `started_at` and not before `invalidated_at`. Applying sets
    `verified`, clears `failure`, and clears `invalidated_at`.
  - A failure applies only if its start time is at least the current observation's
    `started_at`. Same state increments `count`; a new state resets it to 1.
  - `invalidate(at)` sets `invalidated_at`; it never clears the count.
- Recording never changes a command's exit code or stdout. A cache write failure prints
  one stderr warning and moves on.
- Interactive token-read failures from human-run commands are never recorded. Only
  adapter outcomes are, plus `locked` from `status -r`.

### Refresh policy and gates

The plugin asks, bob decides. `bob gkeep status -r` evaluates, in order:

1. Not configured → `skipped: unconfigured`.
2. State `auth` → `skipped: auth`, forced included. Recovery is `login`, or any
   human-run command that succeeds.
3. `pull.lock` held (probe with a non-blocking shared lock, released at once) →
   `busy: true`, `skipped: busy`.
4. `status-check.lock` held → `in_flight: true`, `skipped: in_flight`.
5. Floor: a status-initiated Keep contact in the last 60 s → `skipped: floor`, forced
   included.
6. `rate_limited` backoff window open → `skipped: backoff`, forced included.
7. Not forced and an `offline`, `locked`, or `error` backoff window open →
   `skipped: backoff`.
8. Freshness: max age is `-m` if given, else `gkeep.check_interval`. With `off` and no
   `-m` → `skipped: off`. Observation newer than max age and not invalidated →
   `skipped: fresh`.
9. Not forced and 48 automatic checks already used this local day → `skipped: budget`.

Then: set `last_contact_at`, count the budget when not forced, read the token without a
prompt (10 s), snapshot with a 30 s adapter deadline, classify with default options, and
record the observation or failure.

Backoff, from `failure.count` (n) and `failure.last_at`, with ±10% jitter derived
deterministically from `last_at` so repeated renders agree:

- `offline`: 5 min × 2^(n−1), capped at 2 h.
- `error`: 30 min × 2^(n−1), capped at 2 h.
- `rate_limited`: 60 min × 2^(n−1), capped at 6 h. Check now cannot bypass it.
- `locked`: a flat 30 min. Any human-run gkeep success clears it.
- `auth`: no automatic or forced checks.

`next_check_at` = the latest of the observation's `started_at` + max age (now if
invalidated or none), the backoff end, `last_contact_at` + 60 s, and the next local
midnight when the budget is spent; `null` per the contract above.

Plugin triggers (phase `keep-indicator-actions`); a local task edit never touches Keep:

- Layout ready: cache read now, then `-r` after 15 s if attended.
- Window `focus`, `visibilitychange` → visible, or `online`: `-r`, coalesced to at most
  one per 15 s.
- One timer at `next_check_at`, clamped to 60 s–2 h: `-r` if attended, else mark due and
  fire on the next focus or input.
- Bulb click or **Check Google Keep now**: `-r -m 0`.
- Command Mode `run-finished` carrying the `gkeep` tag, or a modify/rename of the target
  note: cache read only.
- Attended = `document.visibilityState === "visible"`, `document.hasFocus()`, and a
  `keydown`, `pointerdown`, or `wheel` in the last 10 minutes.

Expected volume: eight attended hours at 30 minutes is about 16 checks; overnight and
while Bryan works in other apps, zero.

### Visibility states

| Condition                                                         | Keep segment                                            | Inbox segment                               |
| ----------------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------- |
| `configured: false`                                               | whole item hidden                                       | (hidden with the item)                      |
| Before the first status read                                      | whole item hidden (no flash)                            | —                                           |
| `unverified` (never succeeded)                                    | hidden                                                  | live                                        |
| `ok`, total > 0                                                   | gold pill `💡 3` (tone `fresh`)                         | live                                        |
| `ok`, total 0                                                     | outline pill, faint `0` (tone `zero`)                   | live; faint `0` when zero                   |
| `stale` (failure state or invalidated), checked ≤ 24 h ago        | dashed pill, muted ink, same number (tone `stale`)      | live                                        |
| Verified, not `auth`, last success > 24 h ago                     | dashed pill, `–` (tone `unknown`)                       | live                                        |
| `auth` and verified                                               | dashed pill, `lightbulb-off`, no number (tone `signin`) | live                                        |
| `bob gkeep status` unsupported (old bob, non-JSON, bob not found) | hidden                                                  | live only if the default target file exists |
| Tasks plugin not ready                                            | —                                                       | bob's `inbox.open`, else `–`                |
| Target note missing                                               | —                                                       | `–`, tooltip "gkeep_inbox.md not found"     |

### Visual spec

All rules live under `.bob-keep-inbox` in `plugins/bob-ledger-tools/styles.css`:

- Item:
  `display: inline-flex; align-items: center; gap: 0.3em; flex: 0 0 auto; font-size: var(--font-ui-smaller, 12px); font-variant-numeric: tabular-nums`.
- Both segments are `<button type="button">` reset like `.bob-freshness-main`
  (appearance, background, border, box-shadow, font), `cursor: pointer`, hover
  `background: var(--background-modifier-hover)`. Focus uses the same `:focus-visible`
  ring as `.bob-freshness-main`. Icons are `setIcon`, about `1.05em`.
- Keep pill:
  `border-radius: 999px; padding: 0 0.5em 0 0.4em; gap: 0.3em; line-height: 1.35; border: 1px solid transparent`
  on every tone, so tones never shift layout. Ink:
  `--bob-keep-ink: oklch(from var(--color-yellow) var(--bob-keep-ink-l) min(c, 0.16) h)`
  with `--bob-keep-ink-l` 0.52 (light) and 0.80 (`.theme-dark`), the review footer's
  normalization for at least 4.5:1 contrast.
  - `fresh`: background `color-mix(in oklch, var(--color-yellow) 17%, transparent)`,
    border `color-mix(in oklch, var(--color-yellow) 30%, transparent)`, icon and numeral
    in `--bob-keep-ink`, numeral weight 600.
  - `zero`: no fill, border `var(--background-modifier-border)`, icon and numeral
    `--text-faint`.
  - `stale` and `unknown`: no fill, `border-style: dashed`, border
    `color-mix(in oklch, var(--color-yellow) 45%, var(--text-faint))`, ink
    `--text-muted`.
  - `signin`: like `stale`, `lightbulb-off` icon, no numeral.
  - Checking (user-initiated only, `data-checking`): the bulb pulses opacity 1 → 0.35 →
    1 over 1.2 s ease-in-out; under `prefers-reduced-motion: reduce` it holds at 0.6.
    When a user-initiated check changes the number, the numeral fades in over 240 ms
    (none under reduced motion).
- Connector: `chevron-right`, about 0.85em, `--text-faint`, `aria-hidden="true"`.
- Inbox: icon `--text-muted`, numeral `--text-normal` weight 600; zero → numeral
  `--text-faint` weight 500; unknown → `–`.
- If the yellow reads as related to the review footer's WIP ink, tint only the pill and
  keep the numeral `--text-normal`. Check light, dark, high contrast, and 200% zoom.

Tooltips use Obsidian's `setTooltip(el, text, { placement: "top" })` (fallback:
`title`), one per segment, recomputed on `mouseenter` so ages are current; never set
both `title` and a tooltip. Line 1 is a full sentence. Relative ages: `just now` (< 1
min), `N min ago`, `N h ago`, else `Oct 30, 18:02`. Copy:

- `fresh`: `Google Keep · 3 notes to pull` / nonzero parts of
  `1 new · 1 link → reading queue · 0 revised · 1 archive-only` / nonzero
  `1 pinned stays in Keep · 1 shared stays in Keep` / `Checked 4 min ago` /
  `Click to preview the pull · right-click for more`.
- `zero`: `Google Keep · nothing to pull` / stays line / `Checked 12 min ago` /
  `Click to check now`.
- `stale` (offline): `Google Keep · 3 notes to pull, as of 14:02` / breakdown /
  `Offline since 14:05 · next try about 14:45` / `Click to check now`.
- `stale` (locked): third line `Can't read the Keep token without a passphrase prompt`,
  plus `Run bob gkeep list in a terminal once to unlock`.
- `stale` (rate_limited): `Google asked Bob to slow down · next check after 15:30`.
- `stale` (error): `Keep check failed: <one-line message>` / `Run bob gkeep doctor`.
- `stale` (invalidated by a pull):
  `A pull changed Keep since this count · rechecking soon`.
- `unknown`: `Google Keep · count unknown` / `Last checked Oct 29, 18:02` /
  `Click to check now`.
- `signin`: `Google Keep sign-in expired` / `Run bob gkeep login in a terminal` /
  `Click for the Keep setup check`.
- Inbox: `gkeep_inbox.md · 1 open task to triage` (or `· inbox is clear`, or
  `gkeep_inbox.md not found`, or `Inbox count appears once Tasks loads`) /
  `Click to open the inbox`.
- `aria-label`s: `3 Google Keep notes to pull, checked 4 minutes ago`;
  `1 open task in gkeep_inbox.md`. No live-region announcements.

### Interaction

- 💡 click, tone `fresh`: open Command Mode straight into the `gkeep pull` preview (dry
  run); Ctrl+Enter applies there. The footer never writes on a click.
- 💡 click, tone `zero`, `stale`, or `unknown`: Check now.
- 💡 click, tone `signin`: open the `gkeep doctor` report in Command Mode.
- 📥 click: open the target note with the cursor on its first open top-level task.
- Right-click on either segment: **Check Google Keep now** · **Preview Keep pull…** ·
  **Open gkeep_inbox.md** · **Keep setup check** · **Hide until restart**.
- Palette commands: **Check Google Keep now**, **Toggle Keep inbox indicator** (session
  toggle, default on, matching the plugin's other `Toggle …` commands).
- Desktop only, like the review footer.

## Phase keep-adapter-hardening

Repo: bob-cli. Files: `scripts/gkeep_adapter.py` (protocol stays 1; the dependency pins
stay), plus a Rust test touch only if the message mapping needs it.

1. **Bound the 429 loop.** gkeepapi 0.17.1 `API.send` retries HTTP 429 forever (2 s → 60
   s sleeps), so throttling hangs until the 300 s adapter timeout. At adapter import,
   replace `gkeepapi.API.send` with a bounded copy of the 0.17.1 loop: at most 3
   attempts on 429 (sleeps 2 s, then 4 s), then raise
   `gkeepapi.exception.APIException(429, error)`. Keep the 401 → `refresh()` retry
   behavior byte-for-byte. Guard the patch by asserting the installed gkeepapi version
   is `0.17.1`, and fail loudly in `--self-test` otherwise. Upstream reference: open
   `gh:kiwiz/gkeepapi` with `/sase_repo` (`src/gkeepapi/__init__.py`, `API.send`).
2. **Rate-limit 403s.** In `classify_error`, a 401/403 whose error payload carries a
   reason of `rateLimitExceeded` or `userRateLimitExceeded`, or whose message contains
   "rate limit" (case-insensitive), maps to `rate_limit`, not `auth`. Apply the same
   rule in `_auth_error` so `connect()` agrees.
3. **Narrow the stateless retry.** `connect()` retries with `state=None` only when the
   first failure classifies as something other than `auth`, `network`, or `rate_limit`
   (for example `ResyncRequiredException` or a structurally bad cached state). A network
   blip must not turn into a full download.
4. **One state save per snapshot.** `op_snapshot` re-saves state that `connect()` just
   saved, with no sync in between; drop that second `save_state`. Keep the save after
   `op_archive`'s sync.

Tests: extend `self_test()` with stubs covering: 429 three times → `rate_limit` without
long sleeps (monkeypatch `time.sleep`); 429 then success; 403 with `rateLimitExceeded` →
`rate_limit`; plain 403 → `auth`; a network error on the first `authenticate` → no
stateless retry; a resync error → one stateless retry; snapshot saves state once. Run
`just check-adapter` and `cargo test gkeep`. In `docs/gkeep.md` Known limits, note that
throttling now surfaces as a `rate_limit` error within seconds. Commit as
`fix(gkeep): …`.

## Phase keep-status-cache

Repo: bob-cli. New module `src/native/gkeep/status_cache.rs`; touches `list.rs`,
`pull.rs`, `doctor.rs`, `login.rs`, and `mod.rs`. No new CLI surface and no user-visible
output change.

1. Implement the persisted document, path, locks, load rules, atomic `0600` write,
   `config_key`, and the ordering rules from **Status cache** above. API sketch (names
   are suggestions): `StatusCache::record_observation(&config, Observation)`,
   `record_failure(&config, &GkeepError, started_at)`, `invalidate(&config, at)`, and
   `load(&config) -> CacheDoc` for the next phase. Use `chrono` UTC RFC 3339 timestamps.
2. `Observation::from_default_plan(&Plan, started_at, source)` builds the counts in
   **Count definitions**, so `new` excludes `CreateRef`. Add
   `observe_default(notes, bob_dir, started_at, source)`: scan `Ledger`,
   `imports::read_all`, and the journal (as `list::read_vault` does, with no warnings
   printed), then classify with `PlanOptions::default()` plus the default gkeep routing.
   Consolidate `pull.rs`'s `load_gkeep_routing(false)` and `list.rs`'s inline routing
   load into one shared helper.
3. Map adapter error kinds to states: `network` and `timeout` → `offline`; `rate_limit`
   → `rate_limited`; `auth` → `auth`; `adapter`, `protocol`, `internal`, and
   `dependency` → `error`. Setup errors (config, token, uv) are not recorded.
4. Writers:
   - `list` (Keep shown): snapshot success → `Observation::from_default_plan` on its own
     plan (`-a` only adds archived notes, which never count); snapshot failure →
     `record_failure`. `-s vault` records nothing.
   - `pull`, dry run or real: snapshot success → `observe_default` on the snapshot (pull
     flags never change the cached default count); snapshot failure → `record_failure`.
   - real `pull`: immediately before the first mutating step (clip pre-pass, vault
     write, or archive call) → `invalidate(now)`. On completion with every archive
     outcome known (including `-n`, where nothing is archived) → re-read evidence and
     record `observe_default(snapshot notes minus confirmed-archived ids)`, which is
     what `list` would report next. Any failure, interruption, or unknown archive result
     leaves the cache invalidated (stale), never an optimistic 0.
   - `doctor`: a successful `keep` row snapshot → `observe_default`; an adapter failure
     there → `record_failure`.
   - `login`: its verifying snapshot succeeds → `observe_default` (this also clears
     `auth` and backoff); an adapter failure → `record_failure`.
5. Move `is_open_status` to `ledger.rs` (or another shared spot) so the next phase
   counts the inbox with the same predicate.

Tests (fake adapter through `BOB_GKEEP_ADAPTER`, isolated `XDG_STATE_HOME`): file mode
`0600`; corrupt or mismatched-key files load empty; the newest observation wins; a
failure older than the observation is ignored; invalidation then an older observation is
ignored; a pull with `-n` records the written notes as `pending`; a pull with an archive
failure stays invalidated; `-l 1` and `-i` pulls still record the default count;
`list -s vault` writes nothing; `doctor` and `login` record. Add the **Parity fixture**
as a `ledger.rs` test asserting 5 open. Never run a live Keep command. Commit as
`feat(gkeep): …`.

## Phase keep-status-command

Repo: bob-cli. New `src/native/gkeep/status.rs`; touches `cli.rs`, `mod.rs`,
`config.rs`, `src/native/config/mod.rs` (`RawGkeep`), `adapter.rs`, `pull.rs`,
`src/native/catalog/table.rs`, docs, and generated CLI references.

1. **CLI.** Add the `status` subcommand to `build_cli()` (subcommands stay alphabetical)
   with `-b/--bob-dir`, `-f/--format human|json` (default human), `-h/--help`,
   `-m/--max-age DURATION`, and `-r/--refresh`, alphabetical and each with a short
   alias. About: "Show how many Keep notes a pull would take and how many gkeep_inbox.md
   tasks are open". Give it a long help explaining cache-only versus `-r`, plus examples
   (`bob gkeep status`, `bob gkeep status -r`, `bob gkeep status -r -m 0 -f json`). Add
   a `StatusArgs` typed struct and arg tests like the existing ones. Bare `bob gkeep`
   stays `list`.
2. **Config.** Add `gkeep.check_interval` to `RawGkeep` and `GkeepConfig`: a duration
   `Nm` or `Nh` between 5m and 24h, or `off` (also accept YAML `false`). Out-of-range
   values are config errors.

   > [!decision] auto_checks The default is `30m`.

   > [!decision] auto_checks = no The default is `off`; the Status docs explain opting
   > in with `check_interval: 30m` after measuring check latency and CPU on the Mac.

3. **Never-prompting token read.** Add `GkeepConfig::read_token_unattended()`: run
   `sh -c token_command` with stdin `/dev/null`, stdout piped, and stderr captured
   (never inherited; keep only a scrubbed first line). Remove `GPG_TTY` from the env.
   Set `PASSWORD_STORE_GPG_OPTS` to its current value plus
   `--pinentry-mode error --batch`. Run it in its own process group with a 10 s deadline
   that kills the group (reuse `subprocess_runner`). Any failure, timeout, or empty
   output is a `GkeepError` of kind `locked`; a sign-in cookie shape is `auth`. Never
   log or persist the token.
4. **Adapter deadline.** Let status checks use a 30 s adapter deadline independent of
   `gkeep.timeout_secs` (for example `AdapterClient::with_timeout`), with no spinner.
5. **Gates and run.** Implement **Refresh policy and gates** exactly: gate evaluation as
   a pure, unit-tested function of (cache doc, config, now, `-m`, forced) → run or
   skipped reason; the backoff, jitter, budget, and `next_check_at` math; the busy
   probe; and the single-flight lock. Forced means `-m 0`. `off` blocks only `-r`
   without `-m`. On run, record via `status_cache` (a `locked` failure included).
6. **Pull-lock tolerance.** Make `pull.rs`'s `acquire_pull_lock` retry `WouldBlock` for
   up to 2 s (100 ms steps) before reporting "another bob gkeep pull is already
   running", so the busy probe can never fail a real pull.
7. **Output.** Render the public JSON and human output from the **contract** (the inbox
   via the target file and `is_open_status` only; no ledger scan). Configuration errors
   print the JSON error document and exit 2.
8. **Catalog.** Add a `gkeep-status` row (reads, `Presentation::Report`,
   `network: true`, keywords `keep inbox google gkeep status count`, invalidates
   `["gkeep"]`). Add `"gkeep"` to `KNOWN_INVALIDATES` and to the `gkeep-pull`,
   `gkeep-list`, and `gkeep-doctor` rows, so ledger-tools re-reads the cache after any
   Command Mode Keep run.
9. **Docs** (first read the docs reference memory with `/sase_memory_read`): in
   `docs/gkeep.md`, add `status` to the Commands block, add a "Status" section
   (contract, states, gates, cache path, which commands refresh it, the volume estimate,
   the "never prompts" guarantee, and the Mac Keychain `token_command` hardening tip),
   add `check_interval` under Configuration, and update Exit status and JSON output. Add
   the row to `docs/reference/json-outputs.md`, update `docs/reference/configuration.md`
   if it lists gkeep keys, then run `just docs-gen` and `just docs-check`. The "no
   scheduled runner" statement stays true: status observes and never pulls.

Tests: pure gate tests for every skip reason, forced versus automatic, the budget day
rollover, backoff growth, caps, and jitter bounds; `next_check_at` cases. Integration
tests with a fake adapter that fails if invoked (proves cache-only runs never spawn it)
and one that counts invocations. Cover: fresh skip, forced run, floor, single flight
(test holds `status-check.lock`), busy (test holds `pull.lock`), `auth` refusing a
forced check, `rate_limited` refusing a forced check, the budget, `locked` from a token
command that reads stdin or sleeps past the deadline, a token command asserting
`GPG_TTY` is unset and the pinentry opts are present, `configured: false` with exit 0,
invalid YAML with exit 2, and the JSON schema. Seed `status.json` with timestamps
relative to now. Never run a live Keep command. Commit as
`feat(gkeep): add bob gkeep status …`.

## Phase command-mode-preview-api

Repo: bob-plugins (open with `/sase_repo`: `sase repo open bob-plugins`; read its
`AGENTS.md`). Plugin `plugins/bob-command-mode` (fragment build).

1. Add `preview(argv, opts)` to `cmCreateApi` (`080-api.js`). It asks the plugin to open
   the command sheet (reuse `commandModeOpenSheet`, focusing an open sheet), resolves
   argv's catalog entry (`cmFindEntryForArgv`), builds the same row object the sheet
   builds for that entry, and calls the sheet's `runPreview(row, argv)`. Rows without a
   preview already fall through to `runReport` (report rows) inside `runPreview`; keep
   that, so `preview(["gkeep","doctor"])` opens the doctor report in the sheet's output
   stage. Excluded, unknown, unavailable, and Windows cases settle
   `{ok: false, reason}`. Success settles
   `{ok: true, runId, stage: "preview" | "report"}`. It never applies a write and never
   rejects.
2. Keep `CM_API_VERSION` at 1: the method is additive and consumers feature-detect
   `typeof api.commandMode.preview === "function"`.
3. Tests: extend `scripts/test-command-mode-outcomes-api.cjs` (or the sheet tests) for a
   preview-first row, a report row, an excluded row, an unknown argv, and an
   already-open sheet.
4. Document it in `docs/bob-command-mode.md` beside `run`, `openOutput`, and
   `onRunFinished`.
5. `npm run build`, `npm test`, `npm run docs:check`, then `bob plugins sync`. Commit as
   `feat(command-mode): add api.commandMode.preview`.

## Phase keep-indicator-view

Repo: bob-plugins, `plugins/bob-ledger-tools` (fragment build; each hand-edited fragment
≤ 1000 lines). This phase ships the visible item with **cache-only** Keep reads and no
network.

1. **Fragments**, listed in `src/fragments.json` after `291-plugin-gcal-events.js`:
   - a pure model fragment (for example `292-keep-inbox-model.js`):
     `keepInboxDecodeStatus` (the `schema_version: 1` decoder; anything else is
     `unsupported`); `keepInboxOpenCount(tasks, path)` (the inbox predicate); and
     `keepInboxViewModel({status, inboxCount, tasksReady, defaultTargetExists, enabled, now})`
     →
     `{visible, keep: {visible, tone, icon, text, tooltip, aria}, connector: {visible}, inbox: {visible, tone, text, tooltip, aria}, paintKey}`.
     It implements **Visibility states** and the tooltip copy.
   - a render fragment: DOM creation, `setIcon`, `setTooltip` with fallback,
     `data-keep-tone` and `data-inbox-tone` attributes, and paint-key skipping (like the
     review footer).
   - a mixin fragment, `BobLedgerToolsKeepInboxMixin`, installed in
     `310-install-methods.js`.
2. **Wiring.** `170-plugin-lifecycle.js` is at exactly 1000 lines. Replace its
   `this.setupFreshnessStatusBar();` line with `this.setupStatusBarItems();` (same
   count) and define `setupStatusBarItems()` in the new mixin: it calls
   `setupFreshnessStatusBar()` and then `setupKeepInboxIndicator()`, each in its own
   try, so the Keep item registers immediately after the review footer. Desktop only,
   same platform gate as the review footer. Release timers and listeners through
   `this.register(...)`, `registerEvent`, `registerDomEvent`, and `registerInterval`; no
   `onunload` edits.
3. **Inbox count.** Recount from `planBlockTasks(app)` on
   `obsidian-tasks-plugin:cache-update`, debounced 150 ms, keyed on the configured
   target path (`inbox.path` from the last status document, default `gkeep_inbox.md`).
   Before Tasks is ready, use bob's `inbox.open`. Read only `task.path`,
   `task.indentation` (empty means top-level), and `task.status.symbol`, the Tasks 8.4
   `Task` fields existing ledger-tools code already relies on; shape the test fixtures
   the same way.
4. **Keep reads (cache only).**
   `bobSpawnJson(["gkeep","status","-f","json"], keepInboxDecodeStatus, {commandLabel: "bob gkeep status", resolveBobPath: apiStatsBobPath, getVaultPath: statsVaultBasePath, timeout: 15000})`,
   like `gcalSyncCreatedDailyNote`. Read on layout ready; on window focus or
   `visibilitychange` → visible (at most once per 10 s); on a modify or rename of the
   target note (debounced 2 s, which catches terminal pulls); and on Command Mode
   `run-finished` carrying the `gkeep` tag (add a `gkeep` branch to
   `refreshCachesForCommandModeRun` in `283-plugin-command-mode-refresh.js`). Keep one
   read in flight and coalesce extras. A one-minute `registerInterval` repaint keeps
   ages and the 24 h `unknown` flip honest (the paint key makes it free). When
   `configured: false`, re-read on focus at most every 10 minutes so new setup appears
   without a reload.
5. **Visuals.** Implement **Visual spec** in `styles.css` under `.bob-keep-inbox`.
6. **Inbox click.** Open the target note in the active leaf and put the cursor on the
   first open top-level task line (Tasks' line number); a missing note shows a Notice
   `gkeep_inbox.md not found` and never creates it. The Keep segment has tooltips but no
   click action yet.
7. **Tests**: new `scripts/test-ledger-tools-keep-inbox.cjs` (added to `package.json`'s
   `test` list): every **Visibility states** row and tone; decode of good, unsupported,
   and garbage output; the **Parity fixture** as Tasks-shaped task objects counting 5;
   tooltip and aria copy with a fixed `now`; the paint key skipping unchanged frames;
   DOM render with the repo's fake node; and spawn wiring with an injected `execFile`.
8. **Docs**: a "Keep inbox indicator" section in `docs/bob-ledger-tools.md` (what each
   segment means, states, where numbers come from, and that the plugin only reads bob's
   cached status in this release), then `npm run docs:gen` and `npm run docs:check`.
9. `npm run build`, `npm test`, `bob plugins sync`. Commit as
   `feat(ledger-tools): add the Keep inbox status-bar item`.

## Phase keep-indicator-actions

Repo: bob-plugins `plugins/bob-ledger-tools` (new fragment(s) ≤ 1000 lines; mixin
installed in `310`), plus one bob-cli guide paragraph.

1. **Attention tracking.** Passive `keydown`, `pointerdown`, and `wheel` listeners on
   `document` record `lastInputAt`. Implement the attended predicate from **Refresh
   policy and gates**.
2. **Refresh scheduler.** Implement the plugin triggers in **Refresh policy and gates**
   with `bob gkeep status -r -f json` (60 s spawn timeout). Hold one spawn in flight;
   queue at most one follow-up, where `-r` beats cache-only and forced beats both. Arm a
   single timer from each document's `next_check_at` (clamped 60 s–2 h); a `null` value
   arms nothing. When the indicator is toggled off or the item is hidden for
   `configured: false`, issue no `-r`.
3. **Check now.** Bulb click (tones `zero`, `stale`, `unknown`), the context menu item,
   and the palette command run `-r -m 0` with the checking pulse. Feedback by `refresh`
   result: ran and succeeded → the number updates (fade-in if changed), no Notice; ran
   and failed → Notice `Couldn't check Google Keep: <reason>`; `floor` →
   `Google Keep was checked N s ago`; `backoff` while `rate_limited` →
   `Google asked Bob to slow down · try after 15:30`; `in_flight` →
   `A Keep check is already running`, then one cache re-read 10 s later; `busy` →
   `A Keep pull is running`.
4. **Bulb click, tone `fresh`.**
   `app.plugins.plugins["bob-command-mode"].api.commandMode.preview( ["gkeep","pull"])`.
   Without Command Mode or without `preview`, show the Notice
   `Preview with: bob gkeep pull -d`. Tone `signin` → `preview(["gkeep","doctor"])`
   (report). After the pull applies, the `gkeep` run-finished re-read shows the
   post-pull count.
5. **Context menu.** Obsidian `Menu` on `contextmenu` for both segments, with the items
   in **Interaction**; hide items that cannot work (no Command Mode). **Hide until
   restart** flips the session toggle.
6. **Commands.** `check-google-keep-now` ("Check Google Keep now") and
   `toggle-keep-inbox-indicator` ("Toggle Keep inbox indicator", session-only, default
   on).
7. **Sign-in notice.** One Notice per session on a transition from a verified non-`auth`
   state to `auth`: `Google Keep sign-in expired · run bob gkeep login in a terminal`.
   Never on startup when already `auth`, never repeated.
8. **Tests** (extend `scripts/test-ledger-tools-keep-inbox.cjs` or add a sibling): fake
   clock and injected spawn. Cover: attended versus unattended timer fire and the
   deferred fire on the next input; focus coalescing; single flight and queue upgrade;
   forced feedback per `refresh` reason; tone-dependent clicks, including the
   missing-API fallback; menu items; the toggle stopping all `-r`; and the sign-in
   notice firing once.
9. **Docs**: extend the bob-ledger-tools section (triggers, attention gate, clicks,
   menu, commands, volume) and regenerate the command inventory (`npm run docs:gen`). In
   bob-cli `docs/guides/drain-google-keep.md`, add a short "See the count in Obsidian"
   section linking `docs/gkeep.md#status`; run `just docs-check`.
10. `npm run build`, `npm test`, `npm run docs:check`, `bob plugins sync`. Commit per
    repo (`feat(ledger-tools): …`, `docs(gkeep): …`).

> [!decision] decision_record With `/sase_memory_write`, add a `decisions` web record,
> "Keep Count Is bob-Owned And Attention-Gated". Applies to: bob-cli and bob-plugins.
> Claim: only bob decides when to contact Google Keep and caches the result
> host-locally; clients run `bob gkeep status [-r]`, never poll on a fixed timer, and
> never read the token. Why: cite the research ref and this epic's commits. Cost and
> reopen conditions: from **Context** and **Risks**. Then run `sase memory init`.

## Rules for every phase

- Never contact live Google Keep or read a real credential: no
  `bob gkeep list|pull|doctor|login` or `status -r` against Google. Use
  `BOB_GKEEP_ADAPTER` fakes, temp `XDG_STATE_HOME`, and temp `BOB_CONFIG_FILE` and
  `BOB_DIR`. Live verification is Bryan's.
- bob-cli: Conventional Commits; never edit the `Cargo.toml` version or `CHANGELOG.md`
  (release-plz owns both); all changes are additive `feat`/`fix`. Follow
  `sase/memory/cli_rules.md` (alphabetical options, short aliases, excellent help,
  colored human output). Run `just check` (or the repo's equivalent fmt, clippy gate,
  and tests) before committing.
- bob-plugins: open it with `/sase_repo`, edit `src/` fragments only, never hand-edit
  generated `main.js`, keep fragments ≤ 1000 lines, and run `npm run build`, `npm test`,
  and `bob plugins sync`.
- Calm by default: no red, no badges, no motion for background work, no repeated
  Notices.

## Out of scope (record as `PROPOSED FOLLOW-UP:` on the phase bead if still wanted)

- A `-m 2m` check when the `]s` walk lands on the daily "Import inbox tasks from Google
  Keep" PRE row.
- Access-token reuse to cut master-token exchanges to about one per hour.
- A Mac Keychain `token_command` (documented as a tip only).
- tmux or menu-bar consumers of `bob gkeep status`.
- Auto-pull: revisit only after weeks of real counts.

## Risks

- **Google flags frequent token exchanges.** About 16 a day, a 48 budget, a 60 s floor,
  and rate-limit backoff that Check now cannot bypass; `check_interval: off` is the kill
  switch.
- **`state.json` grows with Keep's archive.** Fix 4 halves its writes; the 30 s deadline
  turns slow checks into `offline` with backoff instead of a hang.
- **GPG locked after a reboot.** By design the footer shows `locked` (stale count,
  unlock hint) and never prompts.
- **A gkeepapi bump re-materializes the uv env.** The status check times out to
  `offline`; the next human `pull` materializes it. The version guard in fix 1 forces a
  deliberate re-check of the send patch.
- **bob/plugin version skew.** The plugin treats a missing or old `bob gkeep status` as
  unsupported (Keep segment hidden); the JSON grows additively under
  `schema_version: 1`.
