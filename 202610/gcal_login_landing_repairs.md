---
tier: tale
title: Repair Calendar login and finish bob-cli-eg landing
goal:
  Both login flows match real gog, the accepted setup and account contracts hold, and
  bob-cli-eg closes with its original plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-eg.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-eg
status: done
---

- **PARENT:**
  [202610/gcal_login_redesign.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/gcal_login_redesign.md)
- **BEAD:**
  [bob-cli-eg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-eg/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-eg.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-eg.land.md)
- **COMMITS:**
  - [f7642c7](https://github.com/bobs-org/bob-cli/commit/f7642c7a5bad190d7e2d94fb6a9a426c84e96d99)
    — feat(gcal): match real gog OAuth and finish guided login

# Finish and land the guided Calendar login

Repair the remaining acceptance gaps in `bob-cli-eg`, then finish its landing in this
tale's coding turn. This is bounded work in the existing Rust login/transport, Python
test fakes, and documentation; it needs one coding agent. Do not restart the original
epic or wait for this tale's eventual commit, push, or CI SHA to close it.

## Context and completed audit

Read `sase bead read bob-cli-eg -r "Need landing notes and final decisions"` and
`sase artifact read plan:202610/gcal_login_redesign.md "Need the accepted login contract"`.
All four original phases are closed. The lander reviewed every phase note, the epic's
screenshot, source, tests, and commits `2ca76ca0`, `00c44ab4`, `18d3f97a`, and
`7254e106`. Client normalization/reuse, captured subprocesses, guided setup, the shared
resolver, and additive JSON fields are present. The reviewer chose
`memory_signin_record = no`: **do not edit any memory or decision files**.

The audited base is `7328fab2`, also fetched `origin/master`. Intervening commits
`b081097c`, `5e4bee79`, `60c77634`, `0e178061`, `3c9536a7`, `03a3497f`, `4b4a4a34`,
`a844d3e1`, and `7328fab2` concern task mentions/relations, capture, PDF tests, and CI
repairs. They introduce no new consumer or conflicting version of the login, runner, or
spinner. Shared documentation changes remain present. The extracted spinner keeps
gkeep's behavior and the runner has no inherited-stdio path. Recheck later drift before
closing, especially the shared docs and transport.

Follow-up triage is complete and recorded on `bob-cli-eg`: eg.1 note 1, eg.2 note 1, and
eg.3 note 1 all describe the same unrelated PDF snapshot assertion, already owned by
bob-cli-87 note 13. `/sase_new_task` search, recent-task sweep, and active epic review
found no separate defect to file. Commit `a844d3e1` repaired the assertion; the exact
lib test
`native::highlights_ref::note_snapshot::profile::tests::table_layout_records_prose_tables_on_snapshot_route`
passed on `7328fab2` (1 passed, 4117 filtered, 13.67 seconds). The lander recorded the
resolution on bob-cli-87 too. Decline a duplicate task for all three proposals. There
are no other child `PROPOSED FOLLOW-UP:` entries.

## 1. Make both OAuth steps obey real gog's contract

The epic screenshot's failure is real. An isolated smoke with installed gog v0.44.0
(`cf42bb7c`), a dummy Desktop client, and a deliberately blocked loopback HTTP proxy
established:

| Attempt                                                      | Result                                                    |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| Step 1 with `--force-consent`, step 2 without it             | `manual auth state mismatch`, before token exchange       |
| Both steps with `--force-consent`, callback missing its port | Same mismatch                                             |
| Matching flags and full callback origin/path                 | Reaches token exchange, then the deliberate proxy refusal |
| Repeating step 1 without a redirect override                 | Reuses the same state; `state_reused: true`               |

The source at that gog commit checks client, scopes, force-consent, and redirect URI in
`internal/googleauth/oauth_flow_manual.go::validateManualState`. If more upstream
inspection is needed, first use `/sase_repo` to open `gh:openclaw/gogcli`; inspect the
installed version's commit, not just upstream HEAD. No real Google authorization or
credential is needed to reproduce this contract.

In `src/native/gcal/gog.rs`, make force-consent part of the common remote auth options
so begin and finish always agree. Keep captured `--no-input --json`, `--readonly`,
`--services calendar`, and the isolated `bob-gcal` client. Client secrets stay on stdin
and refresh/access tokens never enter argv or output.

In `signin.rs`, carry each attempt's actual redirect URI through the listener and
exchange. `handle_connection` currently reconstructs `http://127.0.0.1/oauth2/callback`
without the bound port and interpolates decoded code/state without encoding. Build the
callback against the known loopback origin, preserving the query bytes or encoding with
`url::Url` APIs. Do not trust a request Host header to choose the origin. Continue
rejecting wrong paths and stale states, and show success only after exchange and account
confirmation succeed.

Strengthen `tests/gcal_support/fake_gog.py` to persist and validate the same state tuple
as real gog, including redirect URI and force-consent. It must reject the current
implementation. Model state reuse and consumption accurately enough to test retries;
retain deterministic fixture inputs where helpful. Extend `fake_browser.py` and the gcal
integration tests to cover the complete callback port and percent-encoded code/state
values, success, mismatch, and exchange failure.

## 2. Finish retry, cancellation, and presentation behavior

Make a retry advertised as a fresh link actually create a fresh OAuth attempt. Do not
delete or edit gog's state files. For a paste retry, supply a newly selected loopback
redirect port via gog's supported `--redirect-uri`; compare against the previous URI so
an allocator reuse cannot silently retain the same state. The browser path already
creates a listener per attempt; ensure its redirect differs when rotation is required.
Preserve the single automatic retry limit for stale state/invalid_grant. Keep explicit
consent-denial retries distinct from that automatic budget. Reprint the current link
after a stale pasted address instead of the current empty
`if message.contains("older sign-in")` branch.

Fix these specific gaps in `login.rs`, `signin.rs`, and `login_ui.rs`:

- Downloads confirmation accepts every answer except n/no, including q. Recognize q/quit
  and EOF as `cancelled`; only affirmative/default confirmation stores the client.
  Re-prompt on unrecognized answers. Explain skipped keyring-only clients in the
  interactive setup as well as the existing non-TTY error.
- A healthy rerun currently prints step 1 then step 3, with no `already connected` step
  or summary. Render step 2 before Check and the summary/Next/Undo/writer or account tip
  whenever login succeeds. Reuse the existing calendar probe; do not add duplicate API
  calls. Preserve JSON-only stdout and suppress status lines in JSON mode.
- Check currently warns that a configured primary calendar's real ID is missing after
  normalizing that calendar to `primary` in the report. Determine presence from actual
  calendars using `matches_configured_id`; keep existing JSON fields compatible. The
  API-disabled non-TTY/quit hints must explain that sign-in is kept and `bob gcal login`
  can resume after enabling the printed project URL.
- Browser opener spawn failure currently disappears as `None`. Print the promised
  fallback hint while continuing to accept callbacks. Drop/pause the spinner before
  interactive denial prompts so it does not overwrite input instructions.

Add focused regressions for these observable cases and for retry exhaustion, fresh-state
success, old-state rejection, and denied-then-retry. Keep fake network activity local;
no test uses Google or the real vault/keyring.

## 3. Complete the shared account resolver integration

The new resolver is called by `snapshot::fetch_day`, but the selected account is only
passed to `gog.calendars`. `Gog::events` and `Gog::event` still call
`common_flags(None)`, so list/sync and sync's absent-event lookup can select a different
gog account or fail as ambiguous. This predates the resolver, but now prevents the
epic's shared-account contract from working throughout those commands.

Carry the resolved account through the snapshot and subsequent event lookups; pass it to
every related gog query. Keep one stable account selection for a sync transaction. Do
not change Google write policy, journal semantics, or incomplete snapshot behavior. The
governing read-only/propose-once decision remains intact. Also share case-insensitive
email deduplication with logout while retaining its existing
`-a > config > sole bob-gcal account` order and confirmation behavior.

Make the fake assert the expected account for event list/single-event requests. Cover
configured account among multiple grants, sole Bob account among multiple default gog
accounts, and sync's absent-event lookup. Existing list/sync/lifecycle tests must keep
passing, including no vault writes on a failed lookup.

## 4. Finish the documentation items the docs phase omitted

Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before editing. Update
login's about text to "Connect Google Calendar (read-only) in one guided step", explain
idempotent reruns, and make -c replacement and automatic SSH paste selection explicit in
option help. Keep existing flags and alphabetical ordering. Run `just docs-gen` for
generated references.

Correct `docs/gcal.md` and the setup guide to match the repaired behavior: full SSH and
fresh-desktop transcripts, accurate rerun output, the new error kinds and recovery
actions, the single-use PKCE-bound code in step-2 argv, client JSON on stdin,
`DISPLAY`/`WAYLAND_DISPLAY` precedence, and current `BOB_GCAL_ASSUME_TTY` semantics.
Remove the duplicated "Login's Login" sentence. The guide currently lacks the approved
transcripts and several troubleshooting rows. Preserve the intervening
configuration/JSON documentation changes; make the account-resolution table describe
Bob-bucket preference, not merely a sole gog account. Do not change version or changelog
files.

## 5. Verify the remaining work

Run focused gcal tests first, then `just docs-check` and **`just check`**. Do not run
`just check-full`. A `check` pass with a `check-full` failure is infrastructure work,
not an unfinished part of this epic. Use `/sase_monitor` for a long gate when required,
with a follow-up that explicitly resumes this tale's closeout.

Repeat the safe real-gog contract smoke in a throwaway `GOG_HOME`/`--home`, using only
dummy credentials and a blocked local proxy. Verify that the fixed options and full
callback pass state validation and that a retry produces a new state. Keep raw auth
URLs, state/verifiers, secrets, and codes out of notes. Never inspect or mutate the real
gog configuration. This smoke proves local protocol matching, not live Google consent;
report that limit accurately. Exercise the original NO_COLOR terminal acceptance (paste
instructions, junk re-prompt, EOF cancellation) with isolated fixtures and confirm
stdout/stderr and exit codes.

## 6. Finish bob-cli-eg's landing in this same turn

No separate land agent will resume this tale. After verification, re-read `bob-cli-eg`
and its four phase beads and verify any new notes and commits since `7328fab2`. All old
phases were closed, and this tale completes their gaps. If the host creates a bead for
this tale under the epic, close that completed descendant normally before the epic so
the descendant-readiness check can pass; do not wait for the host's eventual code
commit.

Run `sase bead epic-symbols bob-cli-eg`. Resolve every listed exemption by wiring,
privatizing, adding an appropriate non-test pragma, or deleting the symbol per policy;
re-key only when a named still-open later bead needs it. At audit there were no entries,
and `just --summary` had no `symvision` recipe. Check again.

Close normally with `sase bead close bob-cli-eg --note "<verification>"`. The note must
record the real-gog mismatch repair and regression coverage, all acceptance repairs,
account propagation, docs/gate results, drift integration, honored memory prohibition,
and the completed follow-up triage: all three PDF proposals declined because a844d3e1
fixed the same assertion and the targeted test passed. If close rejects leftover symbols
or unfinished named phases, finish them and retry. Never use `--force` to make a
successful nested landing advance. A genuinely canceled/superseded phase requires
deliberate resolution with a documented reason, not a forced done close.

After closing, run `just symvision` if the recipe is available; otherwise record that it
is absent and that `epic-symbols` was empty. Open the plans sidecar with `/sase_repo`
(`sase repo open plans -r "Mark the landed gcal login plan done"`), read the original
artifact through `sase artifact read`, and set `status: done` in
`202610/gcal_login_redesign.md` under the returned plans root. Include that sidecar
change in the final declaration.

Run `sase bead read bob-cli-eg -r "Need the parent link"` after close. This epic has
**no parent_bead** at the audited state, so no ancestor close is required. If a parent
link has appeared, follow the original landing rules: close only a fully verified parent
phase, or re-audit all descendants, linked plan and drift for a directly parented plan
before normal close, symbol cleanup, symvision, and its plan status update. Stop and
note any ambiguous/incomplete ancestor.

Finish with `/sase_final` for all changed repositories. The host commits after the turn;
the completed epic and original plan must already be closed/done.
