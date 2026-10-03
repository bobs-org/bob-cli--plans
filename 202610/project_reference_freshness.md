---
tier: tale
title: Add project and reference tracker freshness review
goal: Review empty projects after NEW and review reference tasks from creation using
  the existing freshness intervals in the CLI and Obsidian.
size: medium
proposed_by: bbugyi200.apollo.4r
status: done
---

# Review project and reference tracking tasks

## Outcome

Make the existing freshness walk remind Bryan to replenish projects that have no Ready
tasks and to review unfinished reading references. Implement one consistent contract in
bob-cli and bob-plugins, including the `]s` / Alt+Shift+F walk:

**NEW → PROJECTS → PENDING → NEXT → RETURNED → ROTTEN.**

An eligible `^prj` goes in PROJECTS only while its own note has no counted Ready tasks
and its confirmation is missing or due. An eligible Ready `^ref` starts in NEW
immediately, then uses ordinary freshness. Both reuse the normal Ready interval chain
(`refresh` → `task_refresh` → `freshness.interval` → 7 days).

This is one medium tale: one implementation agent can make the bounded Rust and
JavaScript changes together and verify their shared contract. Splitting by language
would create an avoidable intermediate mismatch in queue semantics.

## Context and implementation boundaries

Relevant source in **bob-cli**:

- `docs/freshness.md`: authoritative state, bucket, queue, count, mark, and stamp
  contract; currently JSON schema 4 with Rust keep/decay support.
- `src/native/freshness/state.rs`, `scan.rs`, `cli.rs`, `seed.rs`, and their tests.
- `src/native/dataview/tasks/mod.rs`: `RichTask`, all-task scan, and the existing
  READY/NEXT/PENDING queries.
- `src/native/note_ready/{mod,scan}.rs`: per-note counts of the Ready lane, excluding
  recurring tasks and `^prj`, independently of freshness and `ready_cap`.
- `src/native/projects/{scan,model,sync}.rs`, `docs/projects.md`: project anchor
  parsing, project-frontmatter scheduling, and lifecycle ownership.
- `tests/cli/freshness.rs`, `tests/cli/plan.rs`, note-ready tests, capture creation
  fixtures, and `src/native/highlights_ref/tests/`.

Open **bob-plugins** through `/sase_repo` with an implementation-specific reason; read
its AGENTS.md and use only the returned path. Relevant source there:

- `plugins/bob-ledger-tools/main.js`: `freshnessRowFromTask`, evaluator, queue, counts,
  mark resolution, memo construction/fallbacks, API and review model.
- `plugins/bob-navigation-hotkeys/main.js`: review tier recognition, commitment
  boundary, jump notice, handled-row anchor, and stamp-and-advance.
- `scripts/test-ledger-tools-freshness.cjs`, freshness-mark tests, dashboard-parity and
  note-ready tests, and `test-navigation-freshness.cjs`.
- Both affected manifests and README.md.

Current behavior explains the missing coverage: freshness excludes `#hide`, while
generated project/reference trackers carry it. Rust list also constructs its review
input exclusively from the three already-filtered lane queries. Merely changing the pure
evaluator would therefore miss hidden tasks. Project sync hides `^prj` whenever there
are any non-hidden open tasks or open subprojects; this is broader than having Ready
tasks and cannot serve as the freshness predicate.

The accepted freshness and tiered-walk decisions govern the baseline. This plan changes
their review eligibility/order as explicitly requested, while retaining ordinary-task
semantics, sticky lanes, and read-time classification. Product docs are in scope.
Canonical memory files and generated instruction shims are outside this plan; the
memory-write skill requires separate authorization for those edits.

## Behavior contract to implement

### Recognition and scope

1. Recognize tracking tasks by the parsed, exact trailing block ID `prj` or `ref` on a
   real task. Tags alone, `^prj-extra`, description text, and `[[x#^prj]]` links/embeds
   are not tracking-task identities. Keep legacy lines without the cosmetic
   `#prj`/`#ref` tag supported. The corresponding project note is the tracker’s own
   vault-relative path, not its frontmatter parent or a linked note.
2. Add freshness-specific review eligibility. For these exact tracking IDs, allow the
   conventional `#hide` tag without changing global lane queries, dashboard visibility,
   task tags, or project/highlights synchronization rules. All other exclusions still
   apply: unsupported/closed/blocked status, dependency blocking, recurring task,
   template/conflict path, canonical daily note, Today membership, and future
   scheduling. Ordinary hidden tasks remain out.
3. Honor a project's valid frontmatter `scheduled` date as an additional review gate.
   Project sync deliberately keeps that date off the `^prj` task line, so testing inline
   `scheduled` alone is insufficient. A future note schedule suppresses project review;
   once due, it participates in the existing resurfacing rule. Honor a future legacy
   inline schedule too, without writing or reconciling either date. Reuse existing
   project date parsing; invalid project schedules produce a diagnostic and suppress
   that tracker until fixed, consistent with project sync declining invalid schedule
   input. Do not invent frontmatter scheduling for reference tasks.
4. The task checkbox remains authoritative for lifecycle. Done/canceled trackers
   disappear; stale terminal frontmatter alone must not hide an explicitly reopened
   tracker. Missing optional type/tag metadata does not disable an exact tracking
   anchor. Do not create tasks for notes missing their tracker.

### Project emptiness and cadence

Define `project_ready_count(path)` using the existing per-note Ready-lane counting
predicate: visible TODO-type tasks physically resident in that path, not done,
dependency-blocked, or future-scheduled, excluding recurring tasks and every `^prj` row.
It includes unconfirmed NEW, RETURNED, ROTTEN, and fresh tasks, including ordinary Ready
rows linked Today. It ignores `ready_cap` (including `off`), counts no Next/Pending
rows, and never rolls up child projects, embeds, backlinks, or parent membership. Use
Tasks status types so custom TODO symbols behave like Ready. This is the deliberate
interpretation of “no ready tasks”: the whole Ready lane, as in the existing per-note
count, not the freshness-gated READY dashboard subset. A populated note does not become
empty merely by aging.

- If that count is positive, `^prj` has no freshness state/bucket or walk tier.
- If the count is zero and all other scope checks pass, use the normal Ready interval
  chain. A missing/invalid/future stamp is immediately due; an existing stamp becomes
  due at `today >= fresh + interval`, with the existing scheduled resurfacing exception.
  A stamp today removes it from the queue.
- Every due `^prj` uses machine tier `projects`, label PROJECTS, even if its underlying
  Ready state is NEW or RESURFACED. It never precedes NEW references or ordinary NEW
  tasks. Within PROJECTS: never-confirmed first, then due date ascending, created date
  ascending (missing last), path and line ascending.
- Project cadence also uses that Ready interval when the tracker is `[*]` or `[/]`: the
  requested weekly project reminder must not become a daily lane review. Preserve the
  actual lane and null state/bucket of such lane rows; assign only the PROJECTS tier and
  its due metadata. A disabled lane walk does not disable this project reminder.
  Blocked/custom non-TODO statuses stay out.
- Adding a counted Ready task suppresses review immediately. Completing, moving, hiding,
  blocking, or scheduling the last one away makes the tracker eligible again, using its
  existing stamp. Neither transition writes a stamp or resets the interval: a recently
  reviewed project waits until due; an overdue one appears immediately. Reopening/adding
  a task reverses that result.
- A parent with only open subprojects still gets this reminder when its own counted lane
  is empty. That differs deliberately from sync's hide rule.

### Reference cadence and stamps

For `^ref`, bypass only the freshness hide exclusion; use the existing evaluator
otherwise. An unstamped Ready reference is NEW regardless of `created` or PDF import
age. Confirmation makes it fresh, day 7 makes it rotten under defaults, and existing
task/note/config overrides and resurfacing apply. If moved to Next or Pending, it
follows ordinary lane-review intervals/off-switches; this is the same behavior as any
other task in that lane.

Existing unstamped trackers become eligible naturally on rollout. Preserve all existing
confirmation dates. Creation/import/sync must not auto-confirm either kind. Use the
shared stamping APIs for human actions, preserving block IDs, `#hide`, links, completion
criteria, fields, and line endings. No migration, backdating, seed rerun, automatic task
creation, or auto-completion is involved.

### States, counts, and visible dashboard surfaces

Keep freshness state and walk tier distinct. An eligible Ready project can have
`state: new|resurfaced|rotten|fresh` and its usual bucket; the due ones all walk in
PROJECTS. An eligible reference uses ordinary state/bucket/tier. Hidden special tasks
may therefore have review state even while absent from dashboard sections.
Populated/ineligible projects have null state/bucket/tier.

Make the resulting count semantics explicit rather than assuming tiers equal states:

- `new`, `resurfaced`, `rotten`, `fresh`, and `due` count evaluated Ready states over
  the full review universe; `due = new + resurfaced + rotten`. These include eligible
  Ready trackers, once each. Pending/Next rows retain null state.
- Add `by_tier` in CLI counts (`byTier` in JavaScript counts), with all six machine tier
  keys and zeroes for empty tiers. It counts the actual full queue, and
  `walk = sum(by_tier.values())`. Add `projects_due` / `projectsDue` for symmetry with
  existing pending/next due fields; each equals its respective tier count.
- Human section counts, the REVIEW summary, status-bar walk totals, per-tier ranks, and
  commitment-boundary logic use tier counts, not state counts. A NEW project is
  shown/counts once in PROJECTS. `--limit` only truncates rows.
- PROJECTS belongs before the “Commitments done” boundary. Meeting the upkeep budget
  must not signal that commitments are finished while projects remain; keep the
  established raw `budget_met` formula and meter semantics, and give outstanding
  commitment tiers precedence in notices/status mode.
- Keep `refreshed_today` and `upkeep_today` meanings and all-status scan intact. Do not
  broaden keep/decay eligibility: a PROJECTS row does not acquire a decay decision
  simply because its underlying state is rotten. Preserve current Ready RETURNED/ROTTEN
  logic for references and ordinary tasks.
- Preserve the visible dashboard pool and its NEW/RETURNED/ROTTEN/READY partition.
  `freshness.reviewModel()` and its NEW/ROTTEN chip counts must project the same
  memoized evaluated states onto the existing visible Ready pool. Do not feed hidden
  review-only rows into a badge for a section that excludes them. Keep the global
  progress meter global. Visible project/reference rows use their updated buckets
  normally. This intentionally distinguishes dashboard counts from full
  `freshness.counts()`/CLI review counts; document and test it. Hidden references still
  participate in NEW review through `]s`, the status bar, and the CLI.

No new dashboard section, chip, vault query edit, CLI subcommand/flag, or config key is
required. Preserve `bob plan`, per-note capacity counts, ordinary hidden tasks, and
legacy plugin fallback behavior.

## Implementation steps

1. **Specify and pin the contract.** Update `docs/freshness.md` (scope, formulas,
   interval exception, tier order, counts, human/JSON examples, review ritual, marks,
   and conformance vectors). Briefly link the tracker review behavior from
   `docs/projects.md`, `docs/highlights-ref-sync.md`, and relevant `docs/plan.md` text.
   Define shared PR/RF vectors before implementing. Publish CLI schema 5 for the
   expanded tier vocabulary/counts; its shared seed envelope version advances too, with
   seed behavior unchanged. Expose a small explicit plugin capability for tracker review
   and update documentation/tests; keep namespace version semantics truthful. The
   inspected plugin is namespace v4, while the Rust docs reserve v5 for keep/decay work
   that has not landed in this checkout. Do not claim that unfinished capability by
   unconditionally setting version 5 or reimplementing it. Preserve any already-landed
   keep/decay work encountered at implementation time.

2. **Gather scope inputs in Rust.** Extend the freshness row/context with parsed tracker
   identity, freshness-specific visibility, own-note Ready count, and project schedule
   context. Build the per-path count once from the unchanged Ready-lane snapshot. Use
   the existing rich-task/all-task scan to include hidden tracker candidates; preserve
   Tasks dependency/status parsing and deduplicate by `(path, line)` when combining with
   ordinary lane rows. Do not expand READY/NEXT/PENDING query constants or
   `snapshot.ready` to hold hidden tasks. Cache note metadata once per file. Thread
   these inputs through every row-construction consumer, including `row_bucket`,
   warnings and seed helpers. Share/factor the small Ready-row counting predicate with
   note_ready as useful, without calling `note_ready::scan` from freshness (it already
   calls freshness).

3. **Evaluate and report in Rust.** Add `Tier::Projects`, project gate/interval
   behavior, the deterministic comparator and full tier histogram. Decouple state totals
   from tier totals. Use the same review input for queue and counts. Update CLI rows,
   headings, summary, JSON, schema/help prose and `due_on` / `interval_source` metadata.
   Preserve original source coordinates and actual lane. Keep list read-only and seed
   candidate selection unchanged. Verify `row_bucket` and `bob plan`/note-ready
   consumers see identical decisions for visible trackers without changing their lane or
   capacity predicates.

4. **Mirror the contract in bob-ledger-tools.** Extend the row adapter and pure
   evaluator rather than special-casing navigation. Build/cache the per-path Ready
   counts during memo construction; reuse them for fallback/cloned task lookups and
   marks. Avoid one vault/task scan per project or per API lookup. Do not call
   `api.noteReady` from freshness: note-ready already depends on the freshness memo.
   Preserve all ordinary hide and dependency predicates by extracting a narrowly scoped
   common predicate or composing equivalent freshness inputs; never mutate cached Tasks
   objects to strip tags.

   Include project frontmatter `scheduled` in relevant cache keys and metadata
   invalidation, alongside `task_refresh`. Rebuild when Tasks array identity OR
   generation changes, tasks move/delete/change lane/blocking, Today changes, local day
   rolls over, or interval/schedule config changes. Same-array task updates must work.
   Per-task warm `state/bucket/tier/rank/intervalFor` reads remain O(1). Update the
   line-based interval API used by the refresh picker so `^prj` also displays the
   Ready-chain interval in Next/Pending. Keep missing or ambiguous row/context
   resolution neutral, never borrowing another tracker's count/state or interpreting an
   unavailable task cache as an empty project.

   Update queue tier ranks/labels, API capability, full-review counts/status view, and
   the visible-pool projection for `reviewModel`. Marks/tooltips must agree with the
   queue: a populated project reads as not due because it has Ready tasks; a project
   review reads as PROJECTS with its effective interval, not a daily lane review or a
   generic hidden-task exemption. Update affected manifests and README coherently.

5. **Integrate the navigation tier.** Teach `reviewEntryMachineTier`, commitment checks,
   tier labels, jump notices, and boundary counts about `projects`. Display “No Ready
   tasks in this project” with confirmation/due detail. Continue to use the ledger-tools
   queue for `]s`, `[s`, endpoints, and Alt+Shift+F. Keep legacy v3/v4 queues working
   when the capability is absent. Preserve the handled-row anchor and skip behavior
   after stamping/removing/reordering a project. Source resolution/stale-line guards and
   shared stamping remain the write path. No read/navigation action stamps or changes
   task status.

6. **Verify, then deploy both affected plugins.** Run the checks below; fix regressions
   in the agreed scope. Run `bob plugins sync` from the opened source checkout using its
   explicit `--repo` path and `--no-pull`, selecting the two modified plugin IDs so the
   verified code is deployed. Use the ordinary dirty destination safeguards and report
   an actual refusal if encountered; do not overwrite unrelated vault edits. Confirm
   managed files match via the existing plugin list/sync reporting. Report any
   interactive Obsidian smoke check that could not be performed. Follow the normal SASE
   final declaration for both repositories; no manual git commit is requested by this
   plan.

## Verification and acceptance

Use fixed local dates (`BOB_NOW` / injected JavaScript dates); normal default interval
7; include non-default config 10, note 14, task 3 to prove precedence and that no second
tracker interval setting was introduced. Mirror these vectors in Rust state/CLI and
JavaScript evaluator/memo tests:

| Cases                                                                                                      | Required result                                                                                                                     |
| ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Empty project, unstamped, both with and without `#hide`                                                    | PROJECTS immediately, once per source row; no NEW tier entry                                                                        |
| Empty project stamped today / 6 days / 7 days / longer ago                                                 | No tier / no tier / PROJECTS / PROJECTS, accurate due metadata                                                                      |
| A Ready task that is NEW, fresh, RETURNED, or ROTTEN                                                       | Each suppresses the project's reminder, even with `ready_cap: off`                                                                  |
| Only recurring, hidden, blocked, Next/Pending, closed, future-scheduled work, or open child projects       | None fills the own-note counted Ready lane                                                                                          |
| A custom TODO task; same basename in another directory; a parent/backlink/embed                            | Custom TODO counts in its own path; other-note relationships do not                                                                 |
| Tracker itself, including a visible `^prj`                                                                 | Never fills its own lane; no self-suppression                                                                                       |
| Remove/complete/move/defer/block last Ready task and restore it                                            | Queue updates both directions using the old project stamp, no write by evaluation                                                   |
| Note-frontmatter future project schedule, due date rollover, invalid schedule                              | Excluded until due; resurfacing/date semantics honored; malformed diagnostic with no accidental reminder                            |
| Ready/Next/Pending project tracker, walked lanes enabled/disabled                                          | Project interval always Ready chain; actual lane retained; one PROJECTS entry when due                                              |
| Hidden/unhidden Ready `^ref` with old `created` but no stamp                                               | NEW immediately; creation/import never stamps                                                                                       |
| Reference after stamp, interval boundary, override, resurfacing, Next/Pending, close/reopen                | Ordinary behavior under its actual lane; closed reference drops out                                                                 |
| Both tracker kinds Today-linked, recurring, daily, template/conflict, dependency-blocked, future-scheduled | Existing scope exclusions win over hide exception                                                                                   |
| Ordinary hidden task, tag-only tracker, near-match ID, embedded tracking link                              | No unintended tracking exception                                                                                                    |
| Ordinary NEW + NEW reference + due projects + due Pending/Next + RETURNED + ROTTEN                         | Exact six-tier order; stable ties and missing-created order; sum of tier counts equals full walk                                    |
| Counts, JSON, human output, status bar, limit=1, upkeep budget reached                                     | No duplicate totals; scalar state totals and histogram follow documented meanings; projects prevent premature commitment completion |
| Visible project/ref plus hidden project/ref                                                                | Full review includes eligible hidden trackers; dashboard visible-pool counts/rows agree; lane/capacity counts unchanged             |
| Rust keep/decay already active, due project with high keeps                                                | PROJECTS never creates a decay decision; ordinary/reference rotten behavior retained                                                |

Add plugin integration coverage for task-cache unavailable vs genuinely empty,
same-array generation updates, last-task edits in a different line/note than the
tracker, frontmatter schedule-only changes, cloned and ambiguous task identities,
midnight/config/Today invalidation, and no extra scans on warm API lookups.

Exercise navigation through NEW → PROJECTS → PENDING, backwards and endpoints; stamp a
project and reference with Alt+Shift+F and verify the correct next entry, source block
ID and `#hide` preservation, commitment notices, stale queue refusal, and unchanged
no-project/legacy behavior. Keep placement, dashboard partition, note-ready,
project/highlights sync, capture-no-stamp, seed, and keep/decay tests passing. Use
existing fixture helpers and extend related suites, not a parallel test harness.

Commands from bob-cli: `cargo test freshness`, the relevant note-ready/project/
highlights/capture regression tests, then the required `just all` checks (format,
clippy, full Rust tests) once the implementation is settled. From bob-plugins: focused
`node --test` on the affected suites, then `npm test` and `npm run validate`. Use
`/sase_monitor` if a check needs a long-running handoff. Final acceptance is matching
Rust/JavaScript behavior and successful deployment of both changed plugins, with any
unavailable live Obsidian verification explicitly reported.
