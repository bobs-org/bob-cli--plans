---
tier: epic
title: "Backlog batch 7: URL identity, Gist metadata, close summaries, and Mac CI"
goal:
  Close five existing task beads by fixing URL identity spelling, Gist author/title
  metadata, shifted Pomodoro close-summary headlines, and the Mac save-key test
  compilation and mounted documentation failures. Continue the bounded backlog campaign
  until all project task beads are resolved.
parent_bead: bob-cli-an.6.6.6.5.6
phases:
  - id: url-identity
    title: Normalize equivalent URL path spellings without merging distinct resources
    size: medium
    depends_on: []
    description:
      "url-identity: resolve bob-cli-7u in shared reference dedupe keys and verify
      lookup, duplicate detection, and collection consumers."
  - id: gist-metadata
    title: Give Gist clips a real author and a readable content title
    size: medium
    depends_on: []
    description:
      "gist-metadata: resolve bob-cli-86 with evidence-based Gist metadata extraction
      and offline adapter fixtures, preserving generic metadata and explicit overrides."
  - id: close-summary
    title: Resolve close-summary headlines after Work Log edits shift the ledger
    size: medium
    depends_on: []
    description:
      "close-summary: resolve bob-cli-2t so pomodoro_close.entry_line and task_line name
      the actual closed Pomodoro after same-day task logs shift its line."
  - id: mac-save-ci
    title: Restore Mac save-key test compilation and the mounted Refs docs gate
    size: medium
    depends_on: []
    description:
      "mac-save-ci: resolve bob-cli-bg and bob-cli-bh in bob-mac-capture, preserving
      event conversion and documentation contracts, with compiled Mac tests and the
      shared docs gate."
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.land--2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-24 12:31:53
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch7_url_identity_gist_metadata_close_summary_mac_ci.md)

# Backlog batch 7: URL identity, Gist metadata, close summaries, and Mac CI

## Scope and current evidence

This continues the bounded backlog campaign after batch 6 (`bob-cli-an.6.6.6.5.6`).
Fresh complete enumeration on 2026-10-24 after closing all five batch-6 tasks found
**124 unresolved tasks: 120 ready and four snoozed**; none are open, claimed, or in
progress. The four snoozed tasks are `3o`, `6q`, `aa`, and `ah`, with their recorded
prerequisites preserved. This epic owns exactly five existing tasks across four
independent phases:

| Priority | Task         | Current defect                                                                                                                       | Phase           |
| -------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ | --------------- |
| 1        | `bob-cli-7u` | Shared URL keys preserve path escapes verbatim, hiding the twin of a Medium URL and allowing duplicate capture.                      | `url-identity`  |
| 2        | `bob-cli-86` | Gist metadata can select a numeric author and owner/filename title.                                                                  | `gist-metadata` |
| 3        | `bob-cli-2t` | Close JSON reads a pre-edit line index after task logs move the Pomodoros section, silently returning a blank or unrelated headline. | `close-summary` |
| 4        | `bob-cli-bg` | The Mac save-key test helper cannot compile after commit `72cd721`.                                                                  | `mac-save-ci`   |
| 5        | `bob-cli-bh` | The same commit pushes the mounted Mac Refs documentation above its 6000-word cap.                                                   | `mac-save-ci`   |

Ranking follows silent identity/metadata and JSON defects, then deterministic CI
blockers. `2t` is 24 days old and its exact stale-index source remains. The two Mac
defects share a repository and originating commit, so one agent can repair and validate
both without inter-agent contention. No new feature or memory changes are planned.

Other candidates remain in the same queue. `bob-cli-3w` has 23 corroborations but is a
load-sensitive flake; `5l`, `4k`, `8q`, and `b9` follow it. `4e` is low-impact modal
cleanup (new modal instances normally mask it). `58` is PDF heading presentation, below
this batch's identity, semantic output, and compile defects. `b8` is a discretionary
live identity repair requiring its own implement/decline decision. Athena monitoring/UI
items (`b4`, `ax`, `8r`, `9x`) have no new security or content-loss evidence. The four
snoozed items retain their actual prerequisites.

All-status searches found no duplicate for the five assignments. The active plan
inventory includes `28`, `3j`, `5w`, `66`, `87`, `8b`, `9k`, `9k.8`, `bc`, and `bd`.
None owns these five tasks. Read current ownership again before implementation. `9k.8`
concerns derived homes and its live acceptance; `2t` predates it and repairs the close
headline index, without changing homes policy. `bc` touches theme/close code and `bd`
touches reference rendering: integrate their newer changes without absorbing their
remaining work.

## Working rules

Read the assigned task, every note, this epic's final decisions, and phase dependencies
with `sase bead read`. Reproduce against the current tree. If already fixed, record
independent acceptance evidence instead of reimplementing. Phase workers leave target
tasks open for the lander; record baseline, fix, tests, and any limitation on the phase.
Discovered out-of-scope issues are `PROPOSED FOLLOW-UP:` notes, never new phase tasks.

Use `/sase_repo` before accessing another repository and read its `AGENTS.md`. Use
`sase artifact read` for plan/research/evidence artifacts. No live vault edits are
authorized by this plan; use isolated fixtures. Build bob through `$CARGO_TARGET_DIR`,
not a stale workspace `target/debug/bob`. Follow the daily release train; do not edit
the package version or `CHANGELOG.md` or publish a manual release.

Rust phases run focused regressions then `just check`, using `/sase_monitor` for long
commands. Do not run `just check-full`. Read docs memory and run `just docs-check` for
documentation changes. Record owners for unrelated baseline failures rather than
weakening assertions. At batch-6 landing on `ae3a432b`, `just check` failed only on the
two `highlights::continuations::continuation_scan_*` nodes owned by active `bob-cli-bd`,
and `every_value_arg_has_a_decision` missing `themes rename:old/new`, recorded on active
`bob-cli-bc` (cause `4a6f4494`, phase `bc.6`). These are baseline evidence, not blanket
permission to ignore new failures.

## Phase url-identity — bob-cli-7u

At `ae3a432b`, `src/native/highlights_ref/clip_url.rs::dedupe_key_for` appends
`url.path().trim_end_matches('/')` verbatim. The recorded Medium article exists under
both `@edominguez.se` and `%40edominguez.se`, so `ref find` misses its twin.

1. Reproduce with offline URL pairs and two fixture reference notes. Trace every
   consumer of the shared key, including ref find/library duplicate detection, create
   dedupe, capture routing, and collection duplicate rejection.
2. Normalize percent-escape spelling safely in the shared key. Decode unreserved bytes
   and normalize retained escape hex case. Treat the verified Medium `@`/`%40`
   equivalence explicitly: `@` is reserved, so being legal literally in a path is not
   alone proof of universal equivalence. Keep host-specific alias handling narrow unless
   primary evidence establishes a broader safe rule. Never merge encoded `/`, `?`, `#`,
   or `%` with their structural/literal counterparts, or recursively decode. Keep the
   stored source URL and original requests unchanged.
3. Cover equivalent spellings, escape hex case, non-ASCII paths, malformed escapes as
   accepted/rejected by the existing parser, and distinct delimiter-encoded resources.
   Add a library/CLI regression proving both existing twins are discoverable and the
   duplicate classification is correct; test the collection consumer too.
4. Run focused URL/library/collection tests and `just check`. Document the actual
   identity normalization contract if needed; this is not a live rename/merge job.

Acceptance: the reported Medium pair has one identity and both records are found;
equivalent unreserved spellings agree; distinct encoded resources remain distinct.

## Phase gist-metadata — bob-cli-86

The recorded capture of
`https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f` produced
`author: "262588213843476"` and `title: karpathy/llm-wiki.md`. In
`scripts/web_clip/web_clip_adapter.py`, `layer_author` accepts the first nonblank
JSON-LD/meta/byline/Defuddle candidate; the caller around line 1555 has no Gist rule.
`layer_title` keeps OG/Defuddle unless H1 overlaps it. The root selection mechanisms are
known; trace the actual host facts before choosing a correction.

1. Capture or recover a representative Gist page through `/sase_repo` for its repository
   contents and approved clip/browser tools for page metadata. Keep a small offline
   fixture with the relevant metadata, visible owner, and Markdown heading; do not make
   correctness tests depend on the live service.
2. Prefer the verified Gist owner as author over irrelevant numeric metadata, and the
   content heading or a readable filename fallback as title. Handle Gists without an H1
   and multiple files deliberately. Preserve explicit CLI title/author overrides,
   provenance fields, and generic non-Gist metadata behavior. Do not globally reject
   numeric author names merely to suppress one host's bad metadata.
3. Add adapter tests for the exact bad candidate, title, missing-H1/multiple-file
   fallback, overrides, and a non-Gist control. Verify the Python-adapter-to-Rust
   metadata contract with a fixture capture.
4. Run `just check-web-clip-adapter`, relevant clip tests, and `just check`; use docs
   validation if the contract text changes. Do not rewrite existing references.

Acceptance: a new fixture capture reports the verified owner and readable title;
non-Gist metadata and explicit overrides retain their behavior.

## Phase close-summary — bob-cli-2t

`build_close_summary_json` in `src/native/capture/pomodoro_close.rs` still indexes
`ledger.contents` with `running.line`. `plan_pomodoro_close` in
`src/native/capture_pomodoro_close/linked_tasks.rs` has already replaced those contents
with the staged day file after Work Log insertion. `plan_pomodoro_close_item` copies
that stale `summary.entry_line` into `task_line`. `locate_close_headline` already
compensates for this when building block previews; avoid divergent duplicate logic.

1. Extend the isolated fixture in
   `tests/cli/capture/pomodoro_close.rs::capture_pomodoro_close_tracks_blocks_past_day_file_work_log`:
   day-file task `^build` above `## Pomodoros`, CAPTURE 0920–0950, pinned
   `BOB_NOW=2026-09-28 09:37:00`, close `=x`. Assert both summary fields equal the
   resulting closed headline, not only that block previews exist.
2. Resolve the closed entry against post-edit content using its actual identity and
   timing/location mapping; a name-only search cannot safely distinguish repeated names.
   Make summary fields and block location use one consistent result without altering
   close writes, duration rounding, homes, or JSON field names.
3. Test no shift, multiple inserted log lines, same-name entries, and named/unnamed
   sessions. Preserve preview versus commit behavior. Check all callers of the summary
   builder so fixes to one close variant do not leave another stale.
4. Run focused close/capture regressions and `just check`. Keep this a bob-side fix;
   clients render Bob's result under the thin-client decision.

Acceptance: `pomodoro_close.entry_line` and `task_line` exactly match the intended
closed entry after the write; block references still point to that same entry.

## Phase mac-save-ci — bob-cli-bg and bob-cli-bh

On Mac commit `72cd721`, `HighlightsSaveKeyEvent` declares `init(_ event: NSEvent)`
inside its struct. That suppresses the memberwise initializer called by the `commandS`
helper in `Tests/BobMacCaptureTests/HighlightsSaveKeyMonitorTests.swift:145`. CI
`40107118870` fails with extra-argument and String-to-NSEvent errors before XCTest. The
same commit adds save-scan docs; the mounted `reference/mac-capture/refs.md` has 6027
words against a 6000-word cap. These are distinct task causes.

1. Open bob-mac-capture with `/sase_repo`, read `AGENTS.md`, and reproduce on its
   current source. Add an explicit value initializer or move the NSEvent initializer
   into an extension, keeping event conversion and all test assertions intact.
2. Condense source-owned `docs/refs.md` below the existing cap with reasonable margin.
   Retain save-scan defaults, permissions, limitations, and key contracts. Do not raise
   the cap, edit generated mounted copies, or broadly reformat Swift.
3. Run `just portable-check`. Read `tailnet` memory, rsync a scratch copy to `ssh mac`
   excluding `.build` and `.git`, run `bundle.sh` and `just typecheck-tests`, and clean
   up that scratch copy. The Mac has Command Line Tools without XCTest: CI executes the
   tests; local typecheck is not an XCTest pass. Never use Bryan's checkout or
   `just install`. Use `/sase_monitor` for waits; do not call an uncompiled change done.
4. Run `just docs-check` with the active bob-cli and opened Mac source explicitly
   mounted (`BOB_CLI_DIR`, `BOB_DOCS_BOB_MAC_CAPTURE_DIR` as the tooling requires).
   Record the exact test/compile/docs evidence and handed-off CI SHA/run.

Acceptance: all XCTest targets compile, HighlightsSaveKeyMonitorTests execute and pass
in macOS CI, and the shared docs gate passes with the real Mac source mounted.

## Lander: verification, closure, and mandatory next epic

Read all phase notes and proposals, source, commits, and post-start drift in every
affected repository. Recheck the five target tasks independently. In particular, require
URL lookup/duplicate behavior, fixture Gist metadata, exact post-log close headlines,
successful Mac save-key XTests, and the mounted docs budget. Monitor the macOS Actions
run on the landed Swift SHA; prior batch-6 Mac SHA `9759771` passed all 2966 tests (run
`40105623623`), but that cannot verify this phase's newer Swift. Run appropriate
integrated verification once; new epic-caused failures remain work. Do not hide baseline
failures or rerun `check-full`.

Triage phase follow-ups through `/sase_new_task`, preserving proposing bead and every
outcome. Close only verified target tasks, resolve or re-key all `epic-symbols` entries
to a still-open owner, close this epic normally, run `just symvision` if available, and
mark its linked plan `status: done`. Follow the invoking land prompt's parent readiness
traversal; never force a successful nested landing. A completed batch does not mean the
backlog campaign is finished.

Because `sase plan propose` terminates the runner without the commit finalizer, the
lander must commit every repository it changed, including the plans sidecar status edit,
with `/sase_git_commit` before proposing the successor. This approved plan explicitly
authorizes that skill for those repository obligations.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include `bob-cli-b8` for live identity repair (needs an implement/decline
   decision); `bob-cli-86` and `bob-cli-7u` for silent metadata defects; `bob-cli-2t`
   for an empty close-summary line; flakes `bob-cli-3w`, `bob-cli-5l`, `bob-cli-4k`,
   `bob-cli-b9`, and `bob-cli-8q`; Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and
   `bob-cli-9x`. This is guidance, not a preassigned batch, and concurrent work may
   obsolete it.
4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.
