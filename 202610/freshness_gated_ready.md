---
tier: epic
title: Freshness-gated READY with NEW and ROTTEN review views
goal: The dashboard separates unconfirmed tasks into NEW, keeps confirmed and exempt
  tasks in READY, and links expired or returned confirmations to rotten.md, with matching
  live badges and one rotten vocabulary across human and machine views.
phases:
- id: ledger-bucket
  title: Add cached freshness buckets and matching dashboard models
  size: medium
  depends_on: []
  description: 'ledger-bucket: add the shared read-time bucket contract in bob-ledger-tools
    and bob-cli, memoize classification and config reads, gate the shared READY count,
    and implement availability-aware NEW/ROTTEN models and live refresh. Cover the
    partition, fallback, calendar boundaries, and performance; preserve v1 JSON state
    names until vocab-rotten. Follow the ledger-bucket section below.'
- id: dash-gating
  title: Roll out NEW and ROTTEN views, badges, docs, and decisions
  size: medium
  depends_on:
  - ledger-bucket
  description: 'dash-gating: add NEW between TODAY and PENDING, gate READY, rename
    freshness.md to rotten.md with RETURNED and ROTTEN groups, migrate links and badges,
    and publish the authorized decision/glossary updates. Deploy the plugin and vault
    changes together and verify the actual rendered views. Follow the dash-gating
    section below, including the two-week trial and rollout checks.'
- id: vocab-rotten
  title: Finish the rotten vocabulary and versioned contract migration
  size: medium
  depends_on:
  - dash-gating
  description: 'vocab-rotten: rename freshness-specific machine state/count/config
    names in Rust and JavaScript, publish bob freshness JSON schema 2, and support
    the old budget key with a deprecation lint for one release. Update every freshness
    consumer and conformance test, preserve the bucket contract and unrelated stale
    terminology, then release and verify the final integration. Follow the vocab-rotten
    section below.'
proposed_by: bbugyi200.athena.0uy
create_time: 2026-10-01 13:09:36
status: done
bead_id: bob-cli-3b
---

- **PROMPT:** [prompts/202610/freshness_gated_ready.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/freshness_gated_ready.md)
- **BEAD:** [bob-cli-3b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3b/README.md)

# Freshness-gated READY with NEW on the dashboard and ROTTEN review

## Outcome and scope

Make READY mean recently human-confirmed work, plus the existing freshness-exempt Ready
tasks. New captures appear on the dashboard immediately above PENDING (formerly WIP).
Expired confirmations and returned deferrals appear in `rotten.md`, reachable through a
counted ROTTEN badge. Classification is computed from the existing freshness evaluator
at read time: no `#rotten`/`#new` tag, persisted classification, new status, task
relocation, automatic confirmation, or hooks change.

Bryan explicitly accepted the recommendations in
`research:202610/freshness_gated_ready_dash/freshness_gated_ready_dash.md` and asked for
plan approval before implementation. This epic implements its ADJ-1 through ADJ-10,
including the separately sequenced machine vocabulary migration. Three bounded,
sequential `medium` phases fit the plugin, vault rollout, and breaking contract
boundaries. They deliberately do not run concurrently: phases touch the same plugin and
authoritative docs, and the queries depend on the new API.

The report's optional note-interval changes and inbox triage remain optional; this plan
does not change `sase.md`'s `task_refresh`, bulk-review tasks, or reseed. Freshness for
Next/Pending, dependency-specific return rules, a changed review walk order, and a
custom dashboard renderer are outside this work. Existing stamping gestures, capture
grammar, Bob Mac Capture, task-status-hooks, lane semantics, and the definition of Today
retain their contracts.

## Context verified during planning

- bob-cli at `d2aae45`: `docs/freshness.md` is the authoritative evaluator, placement,
  JSON, and display contract; `src/native/freshness/{state,cli,scan}.rs` and
  `src/native/config/freshness.rs` implement it. JSON is schema 1, with `stale`,
  `counts.stale`, and `stale_daily_budget`.
- bob-plugins at `854bdbe`: ledger-tools manifest is 1.10.0, top-level API v3, freshness
  namespace v1. `plugins/bob-ledger-tools/main.js` contains the evaluator, memo, READY
  models/renderers, status bar, and freshness marks.
  `scripts/test-ledger-tools-freshness.cjs` and
  `scripts/test-ledger-tools-ready-badge.cjs` provide the main test seams.
- `freshnessEnsureMemo` rereads/parses config on each call, and `apiFreshnessState`
  ensures twice and reconstructs rows with `tasks.indexOf`. The existing memo key omits
  Today membership. Caching state therefore requires explicit Today invalidation, not
  merely a map added to the current memo.
- `readyCountFromTasks`, `readyBudgetFromTasks`, `readyBudget`, `planBlockModel`, and
  the dashboard inline fallback currently count ungated Ready tasks. `renderReadyBadge`
  already owns lifecycle-managed refresh for both dashboard and daily-note badges.
  Preserve its separate label/value spans and the recently fixed Obsidian `.text` setter
  behavior.
- The opened vault has TODAY / PENDING / NEXT / READY sections, a REVIEW chip, and one
  `freshness.md` review page. Both daily and weekly chores in `gtd_daily.md` link there.
  `blocked.md` promises tomorrow's tasks land on dash. Dashboard frontmatter and Tasks
  global query provide common visibility rules.
- `bob-cli-3a` (freshness marks) is CLOSED/done, verified with `sase bead read`. Its
  former external sequencing dependency is satisfied. Preserve mark rendering and its
  tests during the vocabulary rename.

Reopen needed repos in each implementing workspace via `/sase_repo`:
`sase repo open bob-plugins -r "Implement freshness-gated Ready"` and
`sase repo open gh:bobs-org/bob -r "Implement freshness dashboard views"`. Use only the
returned checkouts and read their AGENTS instructions. Read research with
`sase artifact read`, not direct sidecar file reads. Before editing, inspect each repo's
status and preserve unrelated work. Repo names and relative paths in this plan are
intentional; no implementing agent should reuse a planning workspace path.

## Shared contract and invariants

Keep the current evaluator's interval precedence, malformed/future-date handling, scope,
and schedule rule (`fresh < scheduled <= today`). RESURFACED still wins over age expiry.
New means no valid human confirmation, not a recent `created`.

| Existing evaluator state             | Stable bucket | Visible destination                                      |
| ------------------------------------ | ------------- | -------------------------------------------------------- |
| `new`                                | `new`         | dash NEW Tasks                                           |
| `resurfaced`                         | `rotten`      | rotten RETURNED Tasks                                    |
| `stale`, renamed `rotten` in phase 3 | `rotten`      | rotten ROTTEN Tasks                                      |
| `fresh`                              | null          | dash READY Tasks                                         |
| null (out of freshness scope)        | null          | READY only if the preexisting Ready predicate permits it |

Let B be the existing visible TODO Ready pool, excluding Today, dependency blocking,
future schedules, `#hide`, templates/conflicts, and the query note. It includes custom
TODO symbols and eligible recurring/daily-note tasks. With the supported plugin and a
ready cache:

`B = NEW ∪ RETURNED ∪ ROTTEN ∪ READY`, pairwise disjoint.

READY applies B plus `bucket !== "new" && bucket !== "rotten"`. Never filter with
`state === "fresh"`: that would lose exempt tasks. A null bucket alone is not proof that
a task is Ready. No freshness bucket may overlap Today, Next, Pending, Blocked, hidden,
done, or cancelled work under the current scope.

The ROTTEN badge total includes RETURNED. Machine `counts.rotten` in phase 3 counts only
age-expired confirmations, so badge total is `counts.resurfaced + counts.rotten`; name
this distinction in code and docs. `counts.due` continues to include NEW as well. The
review walk remains NEW by path/line, then DUE by due date/path/line; page grouping does
not reorder it.

Failure behavior is part of the contract:

- Missing/old/throwing freshness API: legacy READY visibility/count; empty NEW and
  rotten queries; NEW and ROTTEN badges `–`, not zero.
- Missing Tasks data, a non-Warm Tasks cache, or Today not initialized: retain READY's
  existing unavailable badge behavior. New models expose unavailable explicitly and
  never turn an incomplete cache into an empty success.
- Guard query calls with try/catch as well as optional chaining: optional chaining alone
  does not catch a throwing API. Keep fallback handling tiny; it must not become another
  freshness evaluator.
- A failure halfway through a batch must not leave a partially gated badge. Gate from
  one validated snapshot, or discard the partial count and recompute the legacy Ready
  count. Section and badge fallbacks must be tested together.
- Native `bob query` has no Obsidian `app.plugins`: NEW/ROTTEN are empty and READY is
  ungated. `bob freshness list` is the headless review interface. Do not extend the
  native JavaScript runtime just to emulate the plugin.

## Phase ledger-bucket

### Classification and performance

1. Add a pure state-to-bucket helper in Rust and JavaScript. Expose synchronous,
   never-throwing `api.freshness.bucket(task)` returning `"new"`, `"rotten"`, or null.
   Increase the freshness namespace to v2; keep top-level API v3. Add `bucket` to CLI
   JSON queue rows without changing schema 1 yet.
2. Build a key-to-evaluated-result map once per freshness snapshot, using the existing
   normalized path + block ID or path + one-based line key. Include FRESH and
   out-of-scope rows, not just the review queue. Store enough evaluated data to serve
   state, bucket, due, tier, rank, and interval without re-parsing each task. Match the
   existing identity rules and preserve neutral behavior for ambiguous/missing identity.
   A miss can use the current per-row evaluator with the already acquired memo, not a
   second `ensure` call or normal-path linear `indexOf` scan.
3. Cache the freshness config snapshot. Check its file path and mtime/size once per
   existing tick or explicit invalidation; parse only on change. Handle config
   creation/deletion, invalid edits and recovery, environment path overrides, and
   mobile's current default-config behavior. No per-task disk reads, YAML parses, or new
   per-chip polling loops. A config edit must be visible within the existing 60-second
   tick.
4. Invalidate on Tasks cache generation (including a cache-update with the same array
   object), local day, relevant frontmatter, config, and Today membership or readiness.
   Use the same supplied local date throughout a rebuild rather than mixing an injected
   date with `new Date()`; test DST calendar boundaries.
5. Refresh open Tasks queries, READY badges, the status bar, and new freshness views
   after relevant invalidation. Reuse existing events and debounce paths. Notify on a
   bucket/state change even if the due key set is unchanged, and refresh
   calendar-derived labels/escalation each day even if every due task remains due.
   Prevent recursive rebuild/reload loops. A stamp, note override, Today link change, or
   day rollover must not require hooks or reopening notes.

### Counts, models, and presentation

6. Pass one snapshot-backed bucket predicate through `readyCountFromTasks`,
   `readyBudgetFromTasks`, `readyBudget`, and `planBlockModel` and their callers.
   Preserve the cap (100 by default), strict `count > cap` red boundary, current day
   semantics on old daily notes, and unchanged PLAN enforcement/status. READY tooltips
   include total lane pressure, e.g.
   `READY 120/100 · lane 210 = 3 new + 87 rotten + 120 ready`.
7. Add shared NEW/ROTTEN view models from the same snapshot, with an explicit
   availability bit or nullable counts. Expose an additive synchronous model API (e.g.
   `freshness.reviewModel()`) so dash and rotten summary share counts, meter, tooltip,
   and severity. Provide lifecycle-owned rendering/subscription using the existing READY
   widget pattern so DataviewJS-created chips and the rotten summary actually update
   without a note edit. Clean up subscriptions when their component unloads.
8. NEW displays 0 when empty and is red above 0. ROTTEN is neutral at 0, orange above 0,
   red if any returned or age-expired row has `daysOverdue >= interval` for that row.
   This uses `due_on` and each task's effective interval; it is not based on
   confirmation age or the global default. Badge: `ROTTEN 31 · ✓ 12` or
   `ROTTEN 31 · ✓ 12/15` with a budget. Tooltip: total, returned/rotten split, oldest
   days overdue, and today's meter. Describe an expired confirmation as "not confirmed
   in N days", not an invalid task. Budget met never hides/caps NEW or rotten rows.
9. Change the status bar to `⟳ 3 new · 31 rotten · ✓ 12 today` (budget variant
   supported). Change freshness-related human output/help/docs from stale to rotten now,
   including CLI human rows; leave machine names until phase 3. Keep `resurfaced`
   machine state but use RETURNED on the page. Status-bar navigation still invokes the
   next-due command; stage its fallback-target switch for the same rollout that creates
   `rotten.md` in phase 2 so this phase alone does not introduce a dead link.
10. Extend `docs/freshness.md` with bucket semantics and conformance vectors: S1 new;
    S11 rotten; S3/S4 rotten; S2/S5/S7/S12 null; all S13 rows null. Add the same
    assertions to Rust and JS tests; document the temporary state-name compatibility and
    namespace v2. Update plugin README and release metadata consistently. Follow the
    linked repo's required `bob plugins sync` workflow with an explicit repo path; test
    deployment in a disposable vault before the coordinated live cutover.

### Phase validation

Run focused Rust freshness and CLI integration tests plus Node freshness, READY-badge,
Today, and plan-budget suites. Cover bucket boundaries, recurring and daily-note
exemptions, custom TODO, Today exclusion, hidden/conflict/template paths, dependency
blocking, future schedules, malformed/future/duplicate stamps, task/note overrides, and
absence/old/throwing APIs. Assert set membership and badge equality, not only count
totals.

Use read/parse/evaluation counters in a representative Tasks fixture to prove warm
per-row lookups do not repeatedly read config or re-evaluate the vault. Record
before/after render timing with fixture size; avoid a flaky wall-clock test threshold.
Test single-edit refresh, config/frontmatter changes, same-array cache updates,
Today-link-only changes, and midnight expiry/escalation without any vault write.
Preserve all freshness placement and mark regressions.

## Phase dash-gating

### Vault and plugin rollout

1. Work in the opened vault repo, after checking status and current dashboard contents.
   Retain frontmatter, unrelated sections, Tasks global filtering, and existing chip
   styling/accessibility. Apply this section order: TODAY → NEW → PENDING → NEXT →
   READY.
2. NEW is TODO plus `bucket === "new"`, always visible and never limited. It needs no
   separate Today filter because the evaluator excludes Today. Keep the section heading
   and a real zero count when empty. READY retains its old TODO/Today predicates and
   excludes the two review buckets. Align shared visibility rules with the badge model,
   including global template/conflict filtering. Use guarded one-line function
   expressions compatible with Tasks.
3. Replace REVIEW with NEW and ROTTEN. Chip order is NEW, PENDING, NEXT, READY, BLOCKED,
   ROTTEN, TODAY. NEW targets `dash#NEW Tasks`; ROTTEN targets `rotten` with
   external-link affordance and explicit destination `Rotten Tasks`. Keep one shared
   READY badge. Update its inline fallback to the same guarded bucket predicate and
   whole-lane tooltip; do not leave a second legacy count when a capable freshness API
   is available. Preserve badge text spans and keyboard/click/hover behavior. Use the
   phase-1 live models/renderers for NEW, ROTTEN, and their severity classes; delete
   REVIEW-only code/CSS.
4. Rename `freshness.md` to `rotten.md`, rather than creating two review pages. Keep
   `parent: "[[gtd]]"`, aliases `Review` and `Freshness review`, and add `Rotten Tasks`.
   Move the decision/key table. Explain that tasks stay in their source notes, rendered
   rows are click-through views, and `]s`/Alt+Shift+F performs review on source/editor
   lines. Link back to dash and NEW.
5. Give rotten a shared live summary and two always-present sections. RETURNED selects
   bucket rotten + state resurfaced, sorted by queue rank. ROTTEN selects bucket
   rotten + state not resurfaced, sorted by queue rank and grouped by path so one note
   can be cleared at a time. Both use `not done`, `short mode`, and `hide toolbar`; do
   not hardcode TODO in this review page. Return empty safely on missing/old/throwing
   API. State current exclusions (recurring, daily-note, Today, Blocked); render `–`
   when unavailable.
6. Update daily review to clear `[[dash#NEW Tasks|NEW]]` to 0 first, then
   `[[rotten|ROTTEN]]` until 0 or budget, then PENDING → NEXT. Update weekly prune's
   link too. Correct `blocked.md` TOMORROW copy to explain that returned deferrals whose
   scheduled date follows confirmation require RETURNED review; unconfirmed returns are
   NEW. Do not promise every scheduled task is rotten. Search live note/template
   wikilinks, Markdown links, and code targets for `freshness`; update navigation
   references, not archival research or literal field/config names. Aliases do not
   replace updating `openLinkText` targets.
7. Switch the status-bar fallback to `rotten` in this rollout and update plugin
   README/release metadata. Install from the opened source via
   `bob plugins sync --no-pull --repo <opened-bob-plugins> --plugin bob-ledger-tools`
   (dry-run first). Never patch deployed plugin files or force past dirty-file guards.
   Land the tracked vault rename/edits through its normal Git workflow and use the
   documented `bob vault-sync` path so the result reaches `~/bob`; an edited external
   clone alone is not completion. Verify the destination, checked-out changes, and
   successful sync. Plugin bundles are gitignored and must be synced separately on each
   relevant machine. If remote access is needed, read `tailnet.md` with
   `/sase_memory_read` first.
8. Once the corresponding behavior is verified, close vault tasks
   `bob_gtd.md#^hide-rotten-tasks` and `#^scheduled-are-stale` using the existing
   task-completion conventions. Preserve their IDs, children, and live-link consistency.
   The S11 returned-deferral test is required before closing the latter. Leave
   `^wip-next-refresh` and unrelated tasks unchanged.

### Docs, authorized memory, and trial

9. Update bob-cli `docs/freshness.md` (scope/buckets, human vocabulary, ritual,
   surfaces, JSON addition, fallback, mobile config caveat) and `docs/plan.md` (gated
   READY, full-lane tooltip, ROTTEN counterweight, section order and shared daily
   badge); update README examples as needed. Explain that READY's cap now bounds the
   confirmed/exempt pullable backlog, and skipping review can lower READY without
   lowering total lane pressure.
10. Bryan's acceptance of the report explicitly includes these memory changes. Use
    `/sase_memory_write` again before writing, read canonical content only with
    `/sase_memory_read`, then:
    - Add `sase/memory/decisions/ready-is-freshness-gated.md`: READY's read-time
      partition and exempt tasks; section order; reject tags/fields/status changes,
      duplicate inline evaluators, and dim-not-hide as initial policy; costs include
      review-dependent visibility and ungated headless queries; reopen on a required
      plugin-free surface or failed trial. Cite this plan, the accepted research, and
      implementation evidence without inventing why.
    - Mark `decisions:today-is-read-from-the-ledger` superseded-in-part only for its
      section list using `metadata.status`, `superseded_by`, and a backlink; do not
      rewrite its accepted claim or weaken ledger-only Today.
    - Update `glossary:freshness` (`sase/memory/glossary/task-freshness.md`) to rotten
      terminology and the NEW/ROTTEN surfaces. Link related memory. Run
      `sase memory init`; never hand-edit generated instruction shims.
11. Document the accepted trial (2026-10-05 through 2026-10-18) in `docs/freshness.md`
    and a brief rotten-page review note, with a lightweight daily tally of NEW,
    RETURNED, expired ROTTEN, confirmed FRESH, READY, and whether the chip was red. Keep
    if red on no more than 3 mornings, at least about 30 confirmed tasks on most
    mornings, and no lost-needed-task case. Track confirmed FRESH separately from exempt
    READY for that rule. If rollout misses the start, record actual dates for a full
    14-day trial. Completion means the trial is ready to run, not that an agent waits
    two weeks or claims its outcome. If it fails, Bryan can first adjust intervals or
    budget, then reconsider dim-not-hide; never adopt a stored rotten tag.

### Phase validation

Build a fixture with all four destinations and exemptions, execute the actual Tasks
predicate strings with a stubbed Obsidian API, and verify the sections' pairwise
disjoint union against B by path/line/status. Exercise missing, old, and throwing
plugins and restore them. Assert NEW has no limit, old REVIEW targets are gone, and
badges and both rotten groups match classification.

Cross-check one pinned-date Obsidian snapshot against `bob freshness list -f json` on
the same vault/config/date: NEW = new, RETURNED = resurfaced, expired group = stale,
ROTTEN chip = resurfaced + stale, READY = fresh plus eligible exempt tasks. Do not
hardcode the research's historical 199/11 counts, and do not use headless `bob query` as
proof of rendered partition correctness.

Verify in Obsidian on athena or apollo and on the Mac where available: all chips contain
text, navigation/hover/keyboard links work, fresh marks survive, Alt+F on a rotten
source moves it to READY, NEW review moves it to READY, and a local day rollover updates
queries/counts/severity without hooks or task-file writes. Simulate time in a test
harness rather than changing a machine clock or restamping real tasks. Compare
`task-status-hooks --dry-run` with its baseline: no feature-induced rewrites, even if
unrelated existing repairs are reported. Record actual live verification evidence; if a
GUI or machine is unavailable, state exactly what remains unverified and use a concrete
verification gate, not a fabricated pass. Measure the live rerender before/after the
memo change.

## Phase vocab-rotten

1. Reconfirm `bob-cli-3a` is landed (it was already closed at plan time) and integrate
   against the completed prior phases. Rename freshness-specific `FreshState::Stale` to
   `Rotten`, JS state `"stale"` to `"rotten"`, and freshness count members to `rotten`.
   Update evaluator/queue/counts, pure helpers, freshness marks and their state
   comparisons, status/model code, fixtures, and consumers such as navigation's
   freshness fallback counts. Preserve unrelated stale-session, stale-preimage,
   Pomodoro, sync, and other meanings; this is not a repository-wide search/replace.
2. Publish JSON schema 2 for `bob freshness` (`list` and `seed` share the envelope
   constant today). List rows output state `rotten`; list counts use `rotten`; config
   uses `rotten_daily_budget`; `bucket` remains unchanged. Seed has no rotten
   classification but publishes the documented schema-2 envelope consistently. Keep
   `resurfaced`, `tier: due`, `fresh`, `refresh`, `task_refresh`, date rules, queue
   order, and line-number conventions.
3. Rename config to `freshness.rotten_daily_budget` in Rust and JS (`rottenDailyBudget`
   internally). Accept `stale_daily_budget` for one release with one nonfatal
   `freshness_stale_daily_budget_deprecated` diagnostic per loaded config, not one per
   task. Surface it in CLI human/JSON warnings and plugin lints/config status without
   marking a valid legacy config invalid. Canonical key presence wins, including null
   (budget off); if only legacy is present use it. If both occur, warn and ignore
   legacy; reject invalid values of the selected key under current validation rules.
   Test equal/conflicting values, null, absent, invalid, and recovery; unknown-key
   handling and other Bob config loaders retain their behavior.
4. Bump the freshness namespace to v3 to signal changed state/count/config names while
   keeping top-level API v3 and stamping signatures unchanged. Document the intermediate
   v2 and final v3 contracts and the one-release legacy-key window; do not silently
   publish both old/new count names forever. Recheck every in-repo freshness consumer.
   The bucket-based vault predicates should need no state rename because their only
   special case is resurfaced.
5. Update docs, README, examples and the authorized freshness glossary as needed, using
   `/sase_memory_write` and regeneration for memory. Only deliberate
   compatibility/migration references should still say stale about freshness. Bump
   relevant plugin manifest/README versions, run manifest validation, deploy changed
   plugins with `bob plugins sync` from the opened source, and install the updated bob
   binary through the repo's existing release workflow. Verify
   `bob freshness list -f json` from the deployed binary reports schema 2.

### Phase validation and epic completion

Run `cargo fmt --check`, targeted freshness/config/CLI/help/capture-stamp tests, then
`cargo test` and `cargo clippy --all-targets --all-features`. In bob-plugins run focused
freshness, READY, Today, plan-budget, mark, and navigation freshness suites, then
`npm test` and `npm run validate`. Use `/sase_monitor` before any long-running command
that cannot be completed within the current agent turn. Report unrelated known baseline
failures accurately rather than broadening the feature to fix them or treating them as
passes.

Repeat the partition/count/fallback checks against final schema/API names and test the
old-config/new-config compatibility matrix. Confirm rendering and review gestures after
plugin reload, note/frontmatter edits, Tasks reload, Today changes, and midnight. Check
the supported files' diffs for accidental task stamps, status changes, copied task
lines, or tag writes; the only intended task-content edits are review-chore text and
verified completion of the two named vault tasks. Read-only classification itself must
write zero task bytes.

The epic is complete when implementation, tests, docs/memory, and deployment are
accounted for; NEW/READY and rotten groups obey the partition; dashboard and daily READY
agree; badges accurately distinguish unavailable/zero and escalate; schema 2 and the
legacy budget warning are verified; and the trial instructions are present. The landing
report must name any live-platform checks still gated and their exact reproduction
steps. Rollback restores the old queries/badge semantics and review-note links together,
keeps all `[fresh::]` stamps and task content, and can leave the additive bucket API
installed. Never roll back by bulk restamping, tagging, or moving tasks.
