---
tier: tale
title: Correct tracker review eligibility, cadence, and the old-reference backlog
goal:
  Review projects only without other open tasks, configure project/reference reviews to
  1/3 days, and complete references proven older than the seven-day cutoff.
size: medium
proposed_by: bbugyi200.apollo.4r.f1
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4r.f1](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4r.f1.md)
- **COMMITS:**
  - [c0f0039](https://github.com/bobs-org/bob-plugins/commit/c0f0039cbed9714a79f6c4b494ff124cb98b92c3)
    — feat(freshness): mirror tracker intervals and open-task occupancy

# Correct tracker review eligibility, cadence, and the old-reference backlog

## Outcome and scope

Correct the behavior introduced by `plan:202610/project_reference_freshness.md`:

1. Review an eligible `^prj` only when its own note contains **no other open tasks**.
2. Add independent `freshness.project_interval` and `freshness.reference_interval`
   configuration, and set Bryan's chezmoi values to **1 day** and **3 days**.
3. Perform one evidence-based cleanup that completes open `^ref` tasks created more than
   seven days before this request. Preserve newer references and report any whose age
   cannot be established.

Keep the walk **NEW → PROJECTS → PENDING → NEXT → RETURNED → ROTTEN**. An unstamped
Ready reference still enters NEW immediately. A project stamped today can return
tomorrow if it still has no other open tasks. Existing stamps stay intact.

This is a medium tale: the evaluator changes, their JavaScript mirror, two personal
settings, and a bounded one-time data cleanup can be implemented and verified together
by one agent. Separate language phases would create an unnecessary contract mismatch.
The cleanup is an execution step after validation, not an ongoing auto-completion rule.
Do not modify canonical SASE memory or generated instruction files under this plan.

## Inspected context and repositories

In **bob-cli**:

- `src/native/config/freshness.rs` owns the normalized config and invalid-value
  contract.
- `src/native/freshness/state.rs`, `scan.rs`, and `cli.rs` own interval selection,
  tracker context, evaluation, queue/count metadata, and schema 5 output.
- The current `project_ready_count` comes from the filtered READY query. Its helper
  `is_counted_ready_row` is also used by note-ready capacity; changing that helper's
  meaning would incorrectly change capacity counts.
- `src/native/dataview/tasks/mod.rs` provides the all-task scan, `OPEN_QUERY`, and
  status-aware task data. `task.rs` treats DONE, CANCELLED, and NON_TASK as terminal;
  ON_HOLD includes both Next and Blocked in this vault.
- `docs/freshness.md` is the shared behavior contract. Also update tracker descriptions
  in `docs/projects.md`, `docs/highlights-ref-sync.md`, and `docs/plan.md` as
  applicable.
- `tests/cli/freshness.rs`, state/scan/config tests, and note-ready/plan tests cover the
  existing contract.

Open other repositories through `/sase_repo`, read their AGENTS.md, and use only the
paths the command returns. Do not hard-code a numbered workspace:

- **bob-plugins**: `plugins/bob-ledger-tools/main.js` contains config normalization,
  `freshnessEvaluate`, `freshnessIntervalForLine`, row adapters, marks/tooltips,
  `freshnessProjectReadyCounts`, and memo/fallback construction. Navigation's `main.js`
  still says “No Ready tasks in this project.” Tests live in
  `scripts/test-ledger-tools-freshness*.cjs`, dashboard/note-ready tests, and
  `scripts/test-navigation-freshness.cjs`. Preserve the subsequently landed keep/decay
  and decision-card functionality; freshness namespace v5 is now implemented.
- **chezmoi**: `home/dot_config/bob/config.yml` currently has interval 7 and lane
  intervals 1. Its `sase/sase.yml` already declares the post-commit hook
  `chezmoi update -a --force`; use that existing application path.
- **gh:bobs-org/bob**, opened as an external repository, is the vault source for the
  cleanup and its history. Use `bob query` for the live census and the opened checkout
  for direct file/history work. Current audited obsidian memory and
  `docs/vault-git-sync.md` establish Git sync as the active transport, despite the vault
  AGENTS.md's stale Obsidian Sync sentence. Preserve unrelated dirty files.

### Planning-time census, not a fixed mutation list

On 2026-10-03, a native `bob query` of exact `task.blockId === "ref"`, `not done`,
excluding templates and conflicts, found **226 open references: 213 Ready and 13
Pending. None had a task-level `created` date.** The generator in
`src/native/highlights_ref/note.rs` currently omits that field, so a cleanup based only
on `[created::]` would close zero tasks.

The opened vault has full history and matched the live sync SHA
`17d339660a3c709103460fe3e3367a79a168f1d3` when inspected. In the last inspected
snapshot before the UTC cutoff, `ae87ef5e5ec947a2056cb5195ded11caa14ef6f1` (2026-09-25),
**152 current open references already had the same exact tracker and PDF target**: 141
Ready and 11 Pending. This is positive historical age evidence, not a claim that Git
records the exact creation instant. The remaining 74 need boundary/history
classification, not blanket completion. Some commits dated September 25 with a -04:00
offset are September 26 in UTC; compare parsed timestamps, never date prefixes.

Recompute the manifest against current content during implementation; these counts are
useful sanity checks, not an instruction to overwrite intervening changes.

## Behavior contract

### 1. Project occupancy is independent of review eligibility

Define `project_open_count(path)` over actual Tasks rows physically resident in the
tracker's own vault-relative file:

- Include every open status according to the configured Tasks status type, including
  Ready, dependency-blocked or `[?]`, Pending `[/]`, Next `[*]`, and custom open types.
- Include hidden, recurring, future-scheduled, Today-linked, fresh, NEW, RETURNED, and
  ROTTEN tasks. Review eligibility, tags, freshness, priority, and the note's Ready cap
  do not decide whether work exists.
- Exclude DONE, CANCELLED, NON_TASK, and every exact `^prj` row. The tracker must never
  suppress itself; excluding all exact project anchors is neutral under malformed
  duplicate anchors. An open `^ref` in that file does count.
- Include nested real task rows in the file. A plain bullet, link, embed, backlink, or
  an open child project living in a different file contributes nothing. Do not roll up
  parents/children or deduplicate different files by basename.
- Use the unfiltered task inventory/status semantics, not freshness lane queries or the
  note-ready count. A global query that hides an open row must not turn a project empty.
  Preserve the separate Ready-cap predicate unchanged.

Positive count means null project state/bucket/tier. Zero means use existing tracker
eligibility and cadence checks. Preserve the exact `prj`/`ref` identity rule and
hide-only review bypass, closed/blocked/Today/recurring/daily-note exclusions for the
tracker itself, project note scheduling and malformed-schedule diagnostics, and checkbox
lifecycle authority. The broader occupancy rule is not permission to walk blocked
trackers or future-scheduled projects.

Adding, reopening, or moving in any open task suppresses the project immediately.
Completing/cancelling or moving out the last open task restores eligibility according to
the existing stamp. Hiding, scheduling, blocking, or changing an open task's lane does
**not** make the note empty. Neither transition writes freshness metadata.

### 2. Tracker configuration and precedence

Add exactly two optional keys under `freshness`:

```yaml
freshness:
  interval: 7
  project_interval: 1
  reference_interval: 3
  pending_interval: 1
  next_interval: 1
```

Each new key accepts an integer 1–365. **Absent/null means inherit the existing
behavior**, so configurations that omit the new keys preserve their previous cadence.
Reject booleans (including `false`), zero, negatives, values above 365, fractional
numbers, strings, and containers. Follow the existing failure contract: Rust reports a
config error, while the plugin uses the complete default config and flags invalid. Other
config loaders still ignore malformed freshness blocks. Mobile/missing-file fallback has
neither tracker override; document that it cannot see Bryan's desktop settings. No
second copy of those personal values belongs in plugin defaults.

When present, the matching tracker setting is an explicit type cadence and overrides
task `refresh`, note `task_refresh`, global `interval`, and lane interval **for that
tracker**. This mirrors the existing lane-overrides-Ready-chain model and ensures
Bryan's 1/3-day request applies to all eligible trackers, including the 13 Pending
references in the census. Preserve any stored overrides; the UI must describe the
effective value, not imply that an overridden value is active.

| Row                                  | Effective cadence with new key present   | Without new key                    |
| ------------------------------------ | ---------------------------------------- | ---------------------------------- |
| `^prj` in Ready, Pending, or Next    | `project_interval`, source `project`     | Existing Ready chain in every lane |
| `^ref` in Ready                      | `reference_interval`, source `reference` | Existing Ready chain               |
| `^ref` in a walked Pending/Next lane | `reference_interval`, source `reference` | Existing lane interval             |
| Ordinary task                        | Unchanged                                | Unchanged                          |

Eligibility and cadence remain distinct. `pending_interval: false` or
`next_interval: false` still disables reference review in that lane; it does not disable
PROJECTS, consistent with the current project exception. Reference lane rows keep their
actual tier and null state/bucket. Ensure their due arithmetic uses the effective
reference interval, not the original `lane_days` variable. Projects always walk in
PROJECTS. With no key, preserve current source labels and due calculations.

For eligible trackers, missing/invalid/future `fresh` retains existing handling.
Unstamped Ready references are NEW regardless of age; a valid stamp is due at
`today >= fresh + interval`, with current resurfacing precedence unchanged. Existing
`fresh` dates are never flattened, backdated, reseeded, or reset by configuration.

### 3. Shared API and display agreement

Rename project context/count helpers to reflect **open** counts, including pure
evaluator inputs, Rust scan contexts, plugin memo construction, fallback/cloned-task
lookups, and test fixtures. Retain one inventory pass per snapshot and O(1) warm
lookups. Missing/unavailable/ambiguous cache context must stay neutral, never prove
emptiness. Reuse one shared occupancy predicate in normal and fallback plugin paths.

Both new config fields must participate in config snapshots and memo invalidation.
Invalidate on same-array task mutations/generation, completion/reopen/moves, config
edits, note schedule edits, and day rollover. Do not call note-ready from freshness.

Thread the effective cadence/source through state, tier, `due_on`, `days_overdue`,
marks, tooltips, line-based interval lookup, and navigation's refresh picker. Add
explicit human labels for `project` and `reference` sources. The picker's returned
`ready` interval must describe the cadence after release, including a configured tracker
override. Replace project-specific “No Ready tasks”/“has Ready tasks” prose with “No
open tasks in this project”/“project has open tasks.” Leave ordinary Ready labels and
capacity terminology alone.

Keep all six tiers and the state-versus-tier counts contract, hidden review-only
trackers, visible dashboard projection, commitment boundary, progress budget, legacy
navigation fallback, keep/decay eligibility, and stamp writers unchanged. Expose the two
normalized optional config fields in CLI JSON (null means inherit) and add the two
source strings. Advance CLI schema 5 to 6 for the expanded interval contract, including
the shared seed envelope version without changing seed selection/writes. Keep the
existing plugin namespace and tracker-review capability truthful; no new
navigation-owned evaluator or invented keep/decay capability is needed.

## One-time reference cleanup

This plan authorizes completing the eligible old references after the code and cleanup
checks pass; no second generic permission gate is required. Do not run it while
authoring this plan, and do not turn it into a recurring freshness feature.

### Selection and age evidence

- Freeze the request's calendar date as **2026-10-03**, using the supplied **UTC**
  timezone, and its exclusive cutoff as **2026-09-26**. Select `created < cutoff`;
  exactly seven days old is excluded. A later implementation date must not silently
  expand this approved selection. The completion date is the actual execution date.
- Start with a fresh `bob query` census of all open exact `ref` anchors, including
  hidden, blocked, Pending, Next, and scheduled references. Exclude templates,
  conflicts, closed/non-task rows, and lookalike IDs. Do not use the freshness queue as
  the cleanup universe.
- Prefer a valid explicit task creation date. A present but invalid/ambiguous date is an
  exception for review, not permission to fall back to an older estimate.
- For missing dates, examine history in the opened vault repo. A historical snapshot
  strictly before cutoff containing the same real `^ref` tracker and stable reference
  identity/PDF target proves it is old enough. Follow renames when needed; do not use
  note creation alone if the tracker was inserted later. Detect deletion/recreation or
  target replacement rather than borrowing the age of a different reference. Recent-only
  history does not prove the exact creation time; report it separately as not proven
  old. Missing/shallow/ambiguous history remains an explicit exception.
- Never substitute `fresh`, modification time, clone/birth time, PDF publication date,
  or `highlights_synced_at` for task creation. No retrospective creation fields are
  written. Do not add a new creation-stamping feature under this corrective plan.

### Guarded application and evidence

Implement a small one-time helper with a preview/default-no-write mode and explicit
apply mode; keep it scoped to this migration rather than adding a public `bob`
subcommand. A checked-in script under `scripts/migrations/` with fixture tests is
appropriate. If it exposes command-line options, follow `cli_rules.md`: alphabetic,
clear help, short aliases for public long options.

The preview manifest records the frozen cutoff/timezone, observed HEAD, every source
path/line, raw-line/file hash, previous status, creation evidence (date or commit/path),
decision/reason, and exact before/after line. Deduplicate by actual source identity;
refuse duplicate `^ref` anchors in one file rather than choosing an arbitrary one. Save
the manifest and diff with `/sase_artifact` so the final report can identify what was
closed and why. Avoid new vault notes solely for this audit.

Apply only reviewed manifest entries to the clean/current files in the opened vault
checkout. Re-read and compare source bytes, exact anchor/identity, open status, and age
evidence before each guarded atomic file replacement; preserve concurrent changes and
report stale candidates. A retry needs a regenerated preview, not forced writes.
Completion means `[x]` with one canonical `[completion:: YYYY-MM-DD]`, not cancellation
or deletion. Preserve `fresh`, `refresh`, `keeps`, other fields, tags, IDs, PDF links,
children, surrounding text, newline style, and unrelated work. Reuse applicable
completion formatting rather than a vault-wide regex replacement.

Complete the tracker only. Do not mass-edit PDF bytes, highlights, reference
frontmatter, or annotation-derived tasks. The existing Highlights contract makes the
checkbox authoritative and projects completion to `status: read` during its normal
targeted sync; verify that this result cannot be reopened by unchanged sync inputs. Do
not launch a broad `scan --write-pdfs` as part of this cleanup.

Afterward, requery the changed checkout: every applied ID is DONE, every noncandidate is
unchanged, rerunning the migration proposes no changes to already-completed rows, and
the completed trackers disappear from freshness regardless of stamps. Finalize only the
changed vault files through SASE. The existing vault Git-sync service brings the
published commit into the live vault; verify live convergence through `bob query` and
`bob vault-sync status` when available, and distinguish verified source changes from any
still-pending live synchronization in the handoff/report. Do not directly edit `~/bob`
or bypass the repository-open rule to deploy the migration.

## Implementation sequence

1. Pin the revised contract and shared vectors in `docs/freshness.md`; update the
   relevant cross-links/prose. Keep the distinction between open-count occupancy,
   tracker review exclusions, and Ready-cap counts explicit.
2. Extend Rust config parsing, typed interval sources/resolution, the all-task project
   count, evaluator due arithmetic, and CLI report metadata. Update every row/context
   constructor and downstream bucket consumer; preserve seed behavior.
3. Mirror the changes in ledger-tools and update navigation wording/interval display.
   Cover both memo and fallback paths; bump affected plugin manifests and README
   versions in the normal repository convention.
4. Add the two explicit values to chezmoi's managed Bob config with accurate comments
   (they are personal settings, not new built-in defaults). Keep ordinary/lane settings
   unchanged. Verify parsing from this exact source file and the existing post-commit
   apply hook.
5. Implement and fixture-test the one-time helper. Rebuild the current cleanup manifest,
   inspect the exact diff, then apply the authorized entries to the opened vault repo.
   If any age/source remains uncertain, finish the provable entries and enumerate
   exceptions instead of guessing or declaring the cleanup universally complete.
6. Run the verification below, install the corrected CLI with the documented
   `just install` workflow, and verify that the resolved `bob` executable uses the new
   contract. Deploy the two affected plugins with
   `bob plugins sync --repo <opened-plugin-path> --no-pull` and explicit plugin
   selection, respecting destination safeguards, and verify deployed-file reporting. Use
   `/sase_monitor` for checks/install commands expected to outlast a provider turn, with
   enough continuation context to finish the remaining rollout steps.
7. Include bob-cli, bob-plugins, chezmoi, and the edited vault checkout in the normal
   SASE final declaration. Do not manually commit without the required authorization.
   Let chezmoi's declared after-commit hook apply its configuration; confirm/report its
   outcome through the host completion evidence. Report actual tests, deployment,
   cleanup counts/exceptions, audit artifact, and unavailable interactive checks.

## Verification and acceptance

Use fixed injected local dates for evaluator tests. Mirror the core vectors in Rust
state/CLI and plugin evaluator/memo tests; include integration coverage so a correct
pure predicate cannot mask a still-filtered input scan.

| Cases                                                                                                                                                                    | Required result                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| Empty project, unstamped; stamp today; stamp yesterday with project interval 1                                                                                           | PROJECTS; no tier; PROJECTS with accurate due date                                                          |
| One other open Ready, Blocked, dependency-blocked, Pending, Next, custom open task, hidden task, recurring task, future-scheduled task, Today task, or reference tracker | Each individually suppresses project review                                                                 |
| Only terminal/non-task rows, the tracker itself, plain link/embed, or another-file child project                                                                         | None supplies own-file open work                                                                            |
| Last open task is hidden, deferred, blocked, or changes lane                                                                                                             | Project stays suppressed                                                                                    |
| Last open task completes/cancels/moves; task reopens/moves back                                                                                                          | Existing stamp determines eligibility; reopening suppresses again; no stamp writes                          |
| Ready-cap note with new/rotten/blocked/Pending/Next rows                                                                                                                 | Existing per-note Ready counts/caps remain unchanged                                                        |
| Hidden/visible trackers, exact IDs versus lookalikes, missing optional tags                                                                                              | Existing identity and hide-only bypass preserved; no duplicate queue rows                                   |
| Tracker future schedule, malformed project schedule, Today membership, recurrence, blocked/closed status                                                                 | Existing review gates retained despite zero occupancy                                                       |
| Reference unstamped; stamped today, 2 days ago, 3 days ago with reference interval 3                                                                                     | NEW immediately; fresh; fresh; ROTTEN; resurfacing still wins when applicable                               |
| Configured tracker plus task refresh 30, note refresh 14, global 7, lane 1                                                                                               | Matching tracker cadence wins consistently in queue, due metadata, mark, and picker                         |
| Reference in Pending/Next with interval 3; same lane disabled                                                                                                            | Due after 3 days in its lane with null state; disabled lane remains unwalked                                |
| Project in Pending/Next with interval 1 and lane disabled                                                                                                                | PROJECTS cadence still works; actual lane preserved                                                         |
| Absent/null new keys; ordinary task with both keys set                                                                                                                   | Exact prior cadence/source fallback; ordinary task unchanged                                                |
| New key bounds 1/365 and invalid scalars/types                                                                                                                           | Valid endpoints accepted; invalid Rust config exits 2/plugin whole-default fallback; unrelated loaders work |
| Config edits, same-array mutation, move/reopen/close, midnight, cloned lookups, unavailable cache                                                                        | Correct invalidation and neutral uncertainty; no per-project vault scans or warm-read regression            |
| Six-tier histogram, limit, visible dashboard projection, budget/commitment boundary, keep/decay regressions                                                              | Existing invariants remain true with new eligibility/cadence                                                |
| Cleanup created cutoff-1, cutoff, cutoff+1; old tracker without created; newly inserted tracker in old note                                                              | Only strictly older explicit dates or proven old matching trackers close                                    |
| Git timezone boundary, rename, replacement, duplicate ID, invalid date, unavailable history                                                                              | Correct date conversion and identity; uncertain candidates are explicit exceptions                          |
| Cleanup already done/cancelled, recent task, ordinary task mentioning ref, stale file, CRLF, no final newline, second run                                                | No unintended changes; stale guard; preserved formatting; idempotence                                       |
| Closed generated reference through existing lifecycle projection                                                                                                         | Maps to read; unchanged metadata does not resurrect it                                                      |

Run targeted checks while implementing, then the repo's normal Rust gate (`just all`:
fmt, clippy all targets/features, cargo test) and bob-plugins `npm test` plus
`npm run validate`. Run the cleanup helper's fixture suite explicitly. Do not assume the
predecessor's reported clippy failure still exists; recheck and distinguish a
pre-existing failure from a regression with concrete evidence.

Use temporary fixture vaults for destructive migration tests. Real-vault planning and
queue/list checks remain read-only until the inspected migration apply step. Finish with
a real-source-config CLI JSON smoke check (effective project 1/reference 3), plugin
deployment verification, cleanup before/after counts and exceptions, and an Obsidian
`]s`/Alt+Shift+F smoke test if the app is accessible. Never claim interactive validation
or post-commit synchronization that was not actually observed.
