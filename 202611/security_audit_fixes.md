---
tier: epic
title: Security and bug audit fixes
goal: "Confirmed findings from a five-area audit (gcal sync injection, web-clip SSRF
  guard gaps, vault-sync data-loss races, bank parser bugs, subprocess and script
  defects) are fixed with regression tests.

  "
decisions:
  partial_registry_targets:
    ask:
      Should bob capture targets keep config-defined record routes when the album
      registry walk is partial?
    choices:
      drop_all:
        Keep the current drop — a partial walk hides every record route row; add a
        policy comment only.
      config_fallback:
        Emit config-defined record types on a partial walk; withhold only album-derived
        rows.
    default: drop_all
    why:
      The completion provider mirrors the same never-act-on-incomplete-registry policy;
      looks intended.
    answer: drop_all
  dns_rebinding_depth:
    ask:
      Should web clip pin subresource connections to the vetted IP to fully close the
      DNS-rebinding window?
    choices:
      cache_scope:
        Scope the host-verdict cache to one capture and accept the residual rebinding
        race.
      pin_ip:
        Pin vetted hosts to their resolved IPs; heavier machinery that may break real
        captures.
    default: cache_scope
    why:
      Single-user tool; pinning adds a proxy layer whose breakage cost outweighs the
      narrow race.
    answer: cache_scope
phases:
  - id: gcal-escaping
    title: Gcal hostile title and badge URL escaping
    depends_on: []
    size: small
    description:
      "gcal-escaping: collapse newlines/control whitespace in calendar titles and
      percent-encode whitespace in badge URLs so a synced event can no longer forge
      daily-note Markdown; extend the hostile-title e2e test with multi-line cases."
  - id: web-clip-fetch-guard
    title: Web clip out-of-band fetch guard
    depends_on: []
    size: medium
    description:
      "web-clip-fetch-guard: route image/PDF APIRequestContext fetches through the
      private-network guard with per-hop redirect checks, escape < in the reader CSS
      title, and fix the sanitize-order bug that disabled inline-SVG externalization."
  - id: vault-sync-durability
    title: Vault sync and rewrite write durability
    depends_on: []
    size: medium
    description:
      "vault-sync-durability: literal git pathspecs, a pre-restore byte recheck in
      clobber quarantine, conflict copies before merge aborts, fsync before rename in
      the atomic-write helpers, and per-file stale rechecks in the rewrite transaction."
  - id: bank-parser-fixes
    title: Bank month-note parser fixes
    depends_on: []
    size: small
    description:
      "bank-parser-fixes: validate trailing tokens before the out-of-month early return
      in day-heading parsing, parse frontmatter from BOM-stripped text, and correct the
      latent write-planner anchors to match the documented contract."
  - id: subprocess-hardening
    title: Subprocess reaper and notify hardening
    depends_on: []
    size: small
    description:
      "subprocess-hardening: bound the post-exit pipe-drain phase with a deadline, fix
      the straggler-kill PID-reuse ordering, and reject huge sleep values that panic
      Duration::from_secs_f64 in bob pomodoro notify."
  - id: script-small-fixes
    title: Shell and migration script fixes
    depends_on: []
    size: small
    description:
      "script-small-fixes: honor bob_notify's --no-timeout escalation, make the tracker
      migration CRLF-safe and atomic with working duplicate-anchor detection, and settle
      the decision-gated partial-registry capture-targets behavior."
proposed_by: bbugyi200.athena.1bj
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 13:33:42
status: wip
---

- **PROMPT:**
  [prompts/202611/security_audit_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/security_audit_fixes.md)

# Plan: Security and bug audit fixes

A five-area audit of bob-cli (scripts, process execution, network/credential code, vault
file handling, and the most recent feature commits) produced the confirmed findings
below. Each phase fixes one area. All six phases are independent — they touch disjoint
files and may run in parallel. Every finding was verified against the current `master`
source; cited line numbers are from that tree and may drift, so locate by symbol name
first.

Shared constraints for every phase:

- These are behavior fixes for hostile or edge-case input; do not change any CLI or JSON
  contract. None of these fixes warrants a `!` commit marker.
- Never edit the `Cargo.toml` version or `CHANGELOG.md` (the release train owns both).
- Rust phases: run the targeted `cargo test` suites for touched modules and `just lint`.
  Script phases: run the adjacent test files / self-tests named in the phase.
- Every fix lands with a regression test that fails before the fix and passes after,
  unless the phase explicitly says a test is impractical.

## Gcal hostile title and badge URL escaping

**Problem 1 (HIGH, confirmed end-to-end).** A Google Calendar event title containing
newlines is written raw into the daily note, forging arbitrary Markdown lines.
`escape_title` (`src/native/gcal/render.rs:160`) runs
`task_text_escape::escape_task_text` plus gcal-specific passes, but nothing neutralizes
`\n`/`\r` (title source: `src/native/gcal/model.rs:259-263`; sink:
`src/native/gcal/plan.rs:2348,2384` → `WorkingNote::finish`, which joins lines with
`\n`). A probe sync with summary
`"Standup\n## Pomodoros\n- [x] 🍅 **FORGED** [t:: 25m]\n- injected raw line"` exited 0
and produced a forged `## Pomodoros` heading plus a forged column-0 ledger row; bob then
filed its real MEET entry under the attacker's heading, and the row's `^block-id` landed
on the injected line, corrupting journal span tracking for later syncs. Reachable with
zero interaction via `fromGmail` auto-created events (`src/native/gcal/classify.rs:134`
includes them; no attendees → `MyAnswer::Yes` → task row at
`src/native/gcal/model.rs:523-527`). The sibling importers already handle this (gkeep
collapses whitespace in `normalize_body`; gmail's `inert_single_line` collapses it) —
gcal titles are the outlier.

**Problem 2 (MEDIUM, confirmed).** An event location beginning with `http(s)://` becomes
the 📹 join-link badge URL, and `escape_badge_url` (`src/native/gcal/render.rs:260-267`)
encodes only space/parens/`<>`/backtick — not `\n`, `\r`, or tab — while `location_url`
(`src/native/gcal/model.rs:258,575-579`) keeps interior newlines. A location like
`"https://evil.test/\n<payload>"` splits the Event Task row across physical lines
mid-link, moving trailing fields and the block id to a separate line (journal/lifecycle
mismatch → held/duplicated rows on later syncs); tab is also unencoded. Contrast gkeep's
`encode_source_url` (`src/native/gkeep/render.rs:340-362`), which percent-encodes all
whitespace, and gmail's `SafeUrl::parse` (`src/native/gmail/body/inert.rs:119-142`),
which rejects control characters.

**Fix.**

1. In `escape_title`, collapse all whitespace runs (including `\r\n`, `\r`, `\n`, tab,
   and other control whitespace) to a single space before the existing escapes,
   mirroring gkeep's `normalize_body`. Apply the same treatment to every other gcal
   string that reaches a single-line sink (audit `escape_task_text` callers in `gcal/` —
   e.g. location text that lands in the maps badge label).
2. In `escape_badge_url`, percent-encode all ASCII whitespace and control bytes (`\n` →
   `%0A`, `\r` → `%0D`, tab → `%09`, other C0 bytes likewise), or route badge URLs
   through the bullets renderer's `one_line` + `allowed_url`
   (`src/native/gcal/description/bullets.rs:238-255`). Either way the resulting row must
   be provably single-line.
3. Extend the existing `hostile_titles_stay_data` e2e test (`tests/gcal_sync.rs:1404`) —
   it covers `#task`/emoji/`#hashtag` but not newlines. Add cases: a title embedding
   `\n## Pomodoros\n- [x] 🍅 ...` and a location of `https://evil.test/\n<x>` plus a tab
   variant. Assert the daily note gains no headings or column-0 rows beyond the expected
   Event Task rows, and each written row is one physical line ending with its block id.

## Web clip out-of-band fetch guard

All work in `scripts/web_clip/`. The adapter's private-network guard (`BLOCKED_NETWORKS`
/ `PRIVATE_HINT` / `host_is_global` at `web_clip_adapter.py:678-703`, installed via
`context.route("**/*", _handle)` at `:1035-1042`) only intercepts page-originated
browser requests.

**Problem 1 (HIGH, confirmed — SSRF guard bypass).** Several fetches go through
Playwright's `APIRequestContext` (`context.request` / `pw.request.new_context()`), which
the route guard does not intercept, and none re-check `host_is_global()`:

- `_localize_images` → `api.get(raw_src, ...)` (`web_clip_adapter.py:1331`): a clipped
  page containing `<img src="http://192.168.1.1/cgi/reboot">` (nh3 keeps `http:` img
  srcs) gets fetched directly against the LAN/localhost address, and any internal
  response is embedded in the saved article/PDF.
- `_probe_pdf` / `_download_remote_pdf` (`:1474-1510`) and the direct-PDF re-fetch
  (`:1670`): `APIRequestContext` follows redirects (default up to 20), so the single
  up-front `host_is_global(host)` check at `:1601` is defeated by a page that 302s to
  `http://127.0.0.1:9000/x.pdf`, saving an internal file as the reference PDF.

Fix: add one guarded-fetch helper used by every `api.get`/`api.head` on a page-derived
or user-given URL: resolve and require `host_is_global()` before the request, disable
automatic redirect following (`max_redirects=0`) and walk redirects manually with a
per-hop `host_is_global()` re-check (mirroring the hardened Rust fetcher in
`src/native/highlights_ref/fetch.rs`), with a sane hop cap. Tests: a hostile fixture
page whose `<img>` points at a private address must produce a "dropped image" warning
with no fetch, and a redirect-to-private PDF probe must be refused (unit-test the helper
directly if network-level testing is impractical; the self-test pattern at
`web_clip_adapter.py:3292` shows the local style).

**Problem 2 (LOW, confirmed — `</style>` breakout).** `css_escape_string`
(`web_clip_render.py:358-361`) escapes `"` and `\` but not `<`; the page-controlled
footer title is interpolated into a `<style>` block (`content: "{footer_title}";` at
`:740` in `assemble` and `:837` in `assemble_collection`), so a title containing
`</style><img ...>` injects raw markup past the nh3 boundary into the reader/print DOM
(script is still blocked by the meta CSP, so impact is markup/PDF content spoofing).
Fix: escape `<` (CSS `\3c `) — and `>` for symmetry — in `css_escape_string`; add a
render test with a `</style>` title.

**Problem 3 (LOW, confirmed — dead SVG externalization).** snapshot.js marks live SVGs
with `data-bob-svg="1"` (`snapshot.js:537`), but that attribute is not in the nh3
allowlist (`_SVG_ATTRS`, `web_clip_adapter.py:482-489`), and `sanitize_html` runs before
`_localize_images` (`:1826-1834`), so `_SVG_TAG_RE` (`:1222-1225`) can never match in
production and inline SVGs are never externalized to `assets/svg-NNN.svg`. The self-test
at `:3292` feeds the attribute straight to `_localize_images`, masking this. Fix: either
run the SVG externalization pass before `sanitize_html`, or allowlist `data-bob-svg` on
`svg` and strip it after externalization; update the self-test to exercise the real
pipeline order (sanitize → localize) so the regression cannot return.

**Problem 4 (LOW, decision-gated — DNS rebinding).** The route guard resolves hostnames
with Python `getaddrinfo` (verdict cached process-wide forever in `_host_global_cache`),
then Chrome re-resolves on continue — a 0-TTL attacker name can alternate
public→127.0.0.1 between the two resolutions.

> [!decision] dns_rebinding_depth = cache_scope Clear/scope `_host_global_cache` to a
> single capture and add a code comment stating the accepted residual race.

> [!decision] dns_rebinding_depth = pin_ip Additionally pin vetted hosts to their
> resolved IPs (Chrome host-resolver rules, or fulfill subresource requests via a
> guarded API fetch to the pinned address).

## Vault sync and rewrite write durability

Five confirmed data-loss/corruption risks in the vault write paths. The guarded archive
engine (`src/native/task_status_hooks_write/`) is the gold standard here — exclusive
staged temps with `sync_all`, byte-identical re-preflight immediately before each
install — and these fixes bring the weaker paths up to it.

1. **Git wildcard pathspecs (MEDIUM).** `ob::git_command` (used by
   `src/native/ob.rs:318-349` `commit_paths`, `src/native/vault_sync.rs` `git_add_path`
   at `:1777-1788`, and the `git checkout --theirs/--ours -- <path>` calls at
   `:1390-1451`) and the raw git call in `src/native/vault_rewrite/mod.rs:640` pass real
   filenames as pathspecs, but `--` does not disable git's fnmatch wildcards. A vault
   file named `What is Rust?.png` (bob's own `capture_clip` sanitizer allows `?`/`*`) or
   an externally synced `Screenshot [2].png` makes `git checkout --theirs` exit 128
   mid-merge — after one auto-abort retry, vault-sync enters its permanent "halted:
   resolve by hand" state — and `*`/`?` names make `git add -f` over-match and silently
   force-stage unrelated (including gitignored) siblings. Fix: set `--literal-pathspecs`
   (or `-c core.literalpathspecs=true`) in `ob::git_command` and the `vault_rewrite` raw
   git invocation, exactly as `src/native/highlights_ref/guard.rs:561` already does.
   Regression test: a sync/commit over a tracked file whose name contains `[`, `]`, `?`.
2. **Clobber-quarantine race (MEDIUM).** `detect_clobbers` reads `work_bytes`, then
   spends time in `git grep` + candidate reads
   (`src/native/vault_clobber.rs:107,141-183`); if the user saves a newer edit in that
   window, `restore_head_bytes` (`src/native/vault_sync.rs:599-606`, impl at `:796-819`)
   renames HEAD content over the newer save, which is also absent from the conflict copy
   (it holds the stale `work_bytes`) — silent loss of the newest edit in the code path
   that exists to preserve user bytes. Fix: immediately before the rename, re-read the
   live file; if its bytes differ from `hit.work_bytes`, skip the quarantine for this
   cycle (or refresh the conflict copy from the fresh bytes first, then restore).
3. **Merge-abort destroys hand-resolutions (MEDIUM).** Interrupted-merge recovery
   (`src/native/vault_sync.rs:990`, also `:1001`, `:1016`) runs `git merge --abort`
   (rebase/cherry-pick likewise) before anything is staged; if the user hand-resolved a
   conflicted note in Obsidian between cycles, the abort reverts it and the re-merge
   quarantines only the committed local side — the hand-merged text is gone. Fix: before
   aborting, write `_conflicts/` copies of the worktree versions of unmerged/modified
   paths, reusing the mechanism `resolve_conflicts` already uses
   (`write_conflict_copy_bytes`).
4. **Missing fsync before rename (LOW-MEDIUM).** `atomic_write`
   (`src/native/collect_done/archive.rs:570-588`) — the write path for the vault-rewrite
   transaction (`src/native/vault_rewrite/mod.rs:271`) and capture writers
   (`src/native/capture_task_id.rs:405`, `src/native/capture_pomodoro_name.rs:315`) —
   renames without `sync_all`, so a power loss can install an empty/truncated note. Same
   gap in `restore_head_bytes` (`vault_sync.rs:807-818`) and `write_status_record`
   (`vault_sync.rs:2318-2322`). Fix: `sync_all()` the temp file before `fs::rename` in
   all three (match `task_status_hooks_write/staging.rs:343`). No automated power-loss
   test needed; a unit test asserting the helper still round-trips is enough.
5. **Remaining non-atomic / unguarded writes (LOW).** (a) `write_union_merge`
   (`vault_sync.rs:1619-1631`) writes a live note with plain `fs::write`
   (truncate-then-write); switch to the temp+rename pattern used by
   `restore_head_bytes`. (b) The vault-rewrite transaction
   (`src/native/vault_rewrite/mod.rs:219-281`) stale-checks all files, then writes all
   files, so a concurrent Obsidian save landing between a file's check and its own write
   is silently overwritten and survives nowhere; fix by re-comparing each file's bytes
   against its `before_contents` immediately before its own `atomic_write`, failing
   through the existing `stale_note` path.

## Bank month-note parser fixes

All in `src/native/bank/` (landed in the newest commit). `docs/bank.md` is the contract.

1. **Out-of-month day headings accept trailing junk (MEDIUM, reproduced).**
   `parse_iso_heading` (`src/native/bank/grammar.rs:541`) early-returns
   `Some(DayHeading)` on the `!month.contains(date)` branch before the trailing
   token-count/weekday validation that the in-month branch applies. So in
   `bank/2026-11.md`, `## 2026-11-12 trip retro` is correctly prose, but
   `## 2025-12-31 year-end review` parses as a day heading → spurious `bank_day_invalid`
   error (exit 1 on a valid note) and rows under the prose heading attach to the bogus
   day instead of reporting `bank_row_outside_day`. Fix: perform the trailing-token
   validation (at most one token, weekday-shaped) before the out-of-month early return,
   mirroring the in-month branch. Add both reproduction cases as unit tests.
2. **BOM month note loses its frontmatter (LOW, reproduced).** `parse_month_note`
   (`src/native/bank/month.rs:121`) feeds raw `contents` (BOM included) to
   `parse_frontmatter`, which requires the first line to be exactly `---`, while the
   body scan uses the BOM-stripped `PhysicalDoc`. A BOM month note (supported elsewhere
   — see `insert_day_preserves_bom_and_crlf`) reports `type_is_bank_month == false` and
   `frontmatter_month == None` → two spurious `bank_month_mismatch` warnings per check.
   Fix: parse frontmatter from the BOM-stripped `rest` that `PhysicalDoc::parse` already
   computes; add the BOM round-trip test.
3. **Latent write-planner contract violations (LOW, cfg-gated dead code today).** The
   `#[cfg_attr(not(test), allow(dead_code))]` planners in
   `src/native/bank/month.rs:289-358` will back the future ensure/ack phase and already
   violate the documented contract: (a) `plan_append_row` anchors to the day's last row
   line, reparenting indented free-note children under the appended row — compute day
   end from the next heading / end of the contiguous block including children; (b)
   appending to an empty day glues the row to the heading with no blank line — the
   contract says one blank line after the heading; (c) `plan_insert_day`'s fallback
   `after_fence_or_title` picks the last fence in the file, so a code fence inside a
   day's notes misplaces an inserted earliest-day heading — anchor to the first
   fence/title. Fix all three now with pinned unit tests so the ensure/ack feature
   inherits correct planners.

## Subprocess reaper and notify hardening

1. **Unbounded pipe drain (LOW/probable but contract-breaking).**
   `src/native/subprocess_runner.rs` — after the leader exits or is killed on timeout,
   `run` (`:100-118`) and `run_raw` (`:151-168`) `join()` the reader threads with no
   deadline. A grandchild that re-execs into its own session (e.g. a daemon spawned by a
   user-configured `token_command` run via `sh -c` in `src/native/gkeep/config.rs`)
   escapes the process-group SIGKILL while holding the inherited stdout write end, and
   `read_to_string` blocks forever — contradicting the module doc ("an orphaned
   grandchild cannot hold the pipes open") and hanging unattended `bob gkeep` cron runs
   past the 10 s `UNATTENDED_TOKEN_DEADLINE`. Fix: drain with a deadline — e.g. reader
   threads send results over a channel and the parent `recv_timeout`s up to a bound
   after leader exit, abandoning the pipe (and noting truncation) when the bound passes.
   Add a test with a spawned child whose grandchild holds stdout open past the timeout.
2. **PID-reuse race in straggler kill (LOW).** `kill_stragglers(child.id())`
   (`:179-185`) sends `kill -KILL -- -pid` after `try_wait` has already reaped the
   leader; once the group is empty the pid can be recycled as a new group leader, which
   then receives the stray kill. Fix: send the group kill before reaping the leader, or
   skip it when the leader is already reaped and no stragglers were observed.
3. **Panic on huge sleep argument (LOW, confirmed).** `src/native/notify.rs:229` guards
   `is_finite() && >= 0.0` but `Duration::from_secs_f64` panics for values ≥ 2^64 —
   `bob pomodoro notify 1e20 300` aborts with exit 101. Fix: reject values above a sane
   bound in the parse guard (usage error, not a silent clamp) and unit-test it.

## Shell and migration script fixes

1. **`bob_notify` persistent-notification escalation is dead (LOW, confirmed).**
   `scripts/bob_notify:59-63` passes `--no-timeout` to `notify_me` on the 10th
   consecutive overdue alert, but `notify_me` (`:70-77`) shifts the count and ignores
   remaining args, and the `-eq 10` comparison fires only on exactly the 10th loop. Fix:
   handle `--no-timeout` in `notify_me` (`notify-send -t 0` and/or `-u critical`) and
   use `-ge 10`. Keep parity with the Rust port in `src/bin/bob_notify.rs` if it shares
   this logic.
2. **Migration apply corrupts CRLF files and is not atomic (LOW, confirmed).**
   `scripts/migrations/tracker_review_corrections.py:244-268` (`apply_manifest`) splits
   on `"\n"` (leaving `\r` on every line) and, when the text ends with `\r\n`, re-joins
   with `"\r\n"`, producing `\r\r\n` file-wide; the write is a plain in-place
   `write_text` despite the docstring's "guarded atomic file replacement". Fix: split
   handling line endings explicitly (preserve the file's ending style without doubling)
   and write via temp file + `os.replace`. Also fix the dead duplicate-`^ref` detection
   at `:117-120` (`seen_ref_lines` is keyed by `lineno`, unique per iteration — key by
   the anchor/block id per file so the "duplicate ^ref anchor in one file" guard can
   fire). Extend `scripts/migrations/test_tracker_review_corrections.py` with a CRLF
   round-trip case and a duplicate-anchor case.
3. **Partial album-registry walk hides all record capture targets (decision-gated).**
   `append_record_targets` (`src/native/capture_targets.rs:609-618`) returns early on
   `DiscoveryHealth::Partial`, dropping even the config-defined record targets that
   predate the album registry.

   > [!decision] partial_registry_targets = drop_all Make no behavior change; add a
   > short comment at the early return stating the policy is intentional and mirrored by
   > the completion provider.

   > [!decision] partial_registry_targets = config_fallback On `Partial`, still emit the
   > config-defined record types (skip only album-derived rows), report the walk errors
   > as today, and add a test covering one unreadable vault directory.
