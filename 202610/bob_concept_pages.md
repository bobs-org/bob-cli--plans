---
tier: tale
title: Complete Bob's concept pages
goal: Publish six accurate concept pages with reader routing and verified documentation
  builds, links, search, and diagram rendering.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.9
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.9
status: done
---

- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:**
  [bob-cli-a0.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.9.md)

# Complete Bob's concept pages

## Outcome and scope

Complete the `concepts` phase, bead `bob-cli-a0.9`, of
`plan:202610/bob_public_docs_site.md`. A reader should be able to understand tasks,
today's work, review, dependencies, reading, and sync without starting in an
implementation contract. Deliver six short concept pages, wire them into Understand, the
homepage's “A Bob day” strip, and the relevant contract banners, and verify the rendered
result.

This is one bounded documentation implementation, sized medium. The phase had no earlier
implementation notes or unfinished-work list when inspected. Its dependency, the Learn
phase, is closed; the concepts hub, system overview, templates, strict docs pipeline,
generated decisions, and synthetic demo vault already exist. Six explanatory pages and
their shared navigation can be completed by one coder without another epic.

The epic's decisions are final: MIT, public search indexing, and generated public design
decisions are enabled; only `docs.md` memory changes are authorized. This plan requires
no memory edits. Preserve the existing page URLs, contract bodies, generated decision
records, publishing machinery, and ownership of other phases. Runtime code, new CLI
options, a new documentation framework, screenshots from the private vault, and
cross-repository content changes are outside this implementation.

## Read before implementing

Use the audited commands and the named skills:

```bash
sase bead read bob-cli-a0.9 -r "Need the assigned concept phase and latest notes"
sase artifact read plan:202610/bob_public_docs_site.md "Implement the approved concepts phase"
sase artifact read research:202610/bob_public_docs_site/bob_public_docs_site.md "Implementing the Bob public docs site epic"
sase memory read docs.md -r "Need the public documentation contract"
```

The research artifact was available after opening `bob-cli--research` through
`sase_repo`. If the artifact is missing in a fresh checkout, prepare that repository
with the skill, then retry the audited artifact read; do not read its files directly.

Read `docs/_templates/concept.md`, `docs/develop/writing-docs.md`,
`docs/concepts/{index,how-bob-fits-together}.md`, `docs/index.md`, `mkdocs.yml`, and
`docs/_meta/{contracts,mounts,search_smoke}.yml`. Consult the contracts listed below.
Use `sase_memory_read` for glossary terms and relevant decision strands, in batches;
never open canonical memory files directly. Public citations go to the existing relative
`../develop/decisions.md#<decision-slug>` anchors and contract pages.

### Source boundaries already checked

- Lane, Today, freshness state, and review tier are different dimensions.
  `docs/plan.md`, `docs/task-status-hooks.md`, and `docs/freshness.md` specify them.
- The current freshness contract and `src/native/freshness/state.rs` use PRE → NEW →
  PROJECTS → PENDING → NEXT → RECURRING → TICKLER → ROTTEN → POST. References
  participate as ordinary tasks. The older `decisions:review-walk-is-tiered` still lists
  REFERENCES and omits RECURRING. Use its rationale for structural review ordering, not
  its obsolete roster. Tests in `tests/cli/freshness.rs` and `state_tests.rs`
  corroborate current behavior. A `PROPOSED FOLLOW-UP:` note on this phase already cites
  the existing memory tasks `bob-cli-5q` and `bob-cli-3y`; do not duplicate it or
  attempt those edits.
- Some accepted decisions are explicitly superseded in part. In particular,
  `reference-ids-follow-file-stems` replaces the old claim that a reference ID never
  changes; `unblocked-tasks-are-offered` replaces automatic successor linking;
  `subprojects-inherit-project-blocking` extends immediate-residence blocking. Explain
  the resulting behavior, without the history, in reader pages.
- There is no git-sync-specific decision strand in the current decision roster. Ground
  that page's explanation in the observable guarantees and costs in
  `docs/vault-git-sync.md`. The `references-are-ingested-at-birth` decision provides
  relevant cross-host ingestion rationale, not proof of why Git was originally selected
  over another sync product. Do not invent such a historical reason.

## 1. Write the six pages

Follow the concept template's purpose → model → implications → Related shape, keeping
its search boost. Each page needs a title, a description of at most 160 characters, and
a purpose sentence immediately after its heading. Target roughly 500–900 words, with a
hard maximum of 3,000. End every page with Related links to relevant concepts, the
first-day tutorial or an existing guide, and the authoritative reference/contracts. Do
not link to a guide that has not landed.

Use concrete examples from `examples/demo-vault/` where helpful, not personal notes. Use
ordinary Markdown or `bob-task` fences for static examples. Prefer explanatory prose
over command recipes; exact syntax and exhaustive option/field tables belong in
reference. At most one Mermaid diagram per page; only the lane page needs a new one.
Tables and short examples are sufficient for the other five.

Use appropriate `CLI`, `Obsidian`, and `macOS` component tags. Tag the existing system
overview consistently if needed. The guides phase owns the tag plugin, chip override,
and generated component index; do not implement those features here. If that generator
has landed, regenerate its outputs through the normal docs tooling.

### `docs/concepts/tasks-and-lanes.md` — Tasks and lanes

Explain a task as an actionable Markdown checkbox, with its note giving context and a
block ID giving links a stable target. A block ID is not required for every newly
captured task. Include a “Capture makes a task” section for the homepage's Capture link,
and point to capture syntax for grammar.

Explain Ready `[ ]`, Next `[*]`, Pending `[/]` (the Tasks status name remains In
Progress), Blocked `[?]`, Done `[x]`, and Cancelled `[-]`. Distinguish the Ready lane
from the freshness-gated READY view. Explain sticky commitments: an eligible link
promotes Ready to Next, an in-progress outcome puts work in Pending, unlinking does not
release it, and an explicit release returns Next/Pending to Ready. The existing explicit
Next/Pending toggle can be mentioned without reproducing a keymap. Blocking is derived
from prerequisites, a future schedule, or project inheritance. Recovery recomputes an
eligible lane rather than restoring a hidden previous status. Link the
canonical-daily-note exception instead of asserting that every task is always sticky.

Include one compact `stateDiagram-v2` diagram of the main lane transitions. Label arrows
with the action or condition; keep Today and freshness out of the state nodes. State
that the diagram summarizes the main transitions, and make any Blocked recovery arrow
agree with recomputation, not saved-state restoration. Follow it with enough text to
understand the model without the diagram.

Sources: `plan.md`, `task-status-hooks.md`, `reference/vault-format.md`; decisions
`task-lanes-are-sticky`, `note-ready-cap-counts-the-lane`, and
`projects-block-from-prj-line` with its superseding records.

### `docs/concepts/today-and-the-pomodoro-ledger.md` — Today and the Pomodoro ledger

Explain the daily note's `## Pomodoros` ledger and dedicated Task Links: tasks stay in
their area/project/inbox notes while links select today's work. Today is derived from
open tasks linked under today's open entries; it is neither a lane nor a tag nor “tasks
stored in the daily note.” Closed entries are work history. Removing the last live link
removes Today membership but leaves a sticky Next/Pending lane. Starting or closing a
session and completing its task are distinct outcomes.

Use one small ledger example and explain the difference between the live plan, recorded
session history, and task commitment. Briefly distinguish Today membership from the
theme/link budget: exempt GTD entries can still contribute Today tasks; budget links and
Today use different counting rules. `bob plan` is read-only. Leave exact caps, syntax,
markers, and resolution rules in the contract.

Sources: `plan.md` (Today, ledger, budgets), `pomodoro-homes.md`, `today-marks.md`,
capture reference; decisions `today-is-read-from-the-ledger` and
`task-lanes-are-sticky`.

### `docs/concepts/freshness-and-the-review-walk.md` — Freshness and the review walk

Explain `[fresh::]` as a human confirmation date, not creation time, completion,
priority, or activity. Explain NEW, fresh work, scheduled/unblocked TICKLER returns, and
ROTTEN review without implying that classification edits task statuses. The READY view
excludes review buckets while whole-lane pressure remains visible. Freshness itself
never silently changes a lane, priority, or schedule.

Describe the current nine tiers in a short ordered list or table using the roster above.
Explain daily Pending/Next review, PROJECTS, completion-based PRE/POST, due recurring
occurrences that are never stamped, and references using ordinary tiers. Explain that
review order differs from dashboard grouping. Keep detailed eligibility, intervals, and
comparators behind contract links.

Briefly describe the human choices: confirm, change, defer, link, release, complete, or
drop. Ordinary resolving answers advance a landed walk row; a task move follows the
moved task and parks the walk, and Alt+F is the stay answer. Repeated unchanged keeps
can prompt an explicit approved decision; they do not authorize automatic decay. Do not
reproduce the full hotkey reference.

Sources: `freshness.md`, `plan.md`; decisions `review-walk-is-tiered` (rationale only
where outdated), `ready-is-freshness-gated`, `rotten-keeps-use-priority-decay`,
`decay-decisions-are-available-immediately`, `unblocked-tasks-resurface-as-ticklers`,
and `task-move-never-advances-the-walk`.

### `docs/concepts/projects-and-dependencies.md` — Projects and dependencies

Explain the difference between residence in a project note, a parent project
relationship, and a prerequisite. A project's `^prj` task represents the outcome. One
managed first-child Depends-On line holds readable prerequisite links; fields such as
`dependsOn` and target `id` are derived projections, and the Blocks mirror is a reverse
view rather than another source of dependency intent.

Explain future scheduling and open prerequisites on `^prj`, inherited blocking through
child projects, and why blocked work cannot remain planned under today's open Pomodoros.
Make clear that a manually set blocked checkbox alone is not a project blocking reason.
Explain the importance of recomputing from causes, without teaching manual edits to
derived fields or tags.

Include a “Finishing a prerequisite” section: managed completion can recover dependents
automatically, but offers require explicit selection before work is linked into Today.
Unpicked work follows its ordinary recovery/review rules; descendant recovery can
require reconcile. Do not claim every close immediately repairs the entire project tree.
Avoid the old blanket rule that every capture link on a blocked task raises
prerequisites: direct capture has a documented own-prerequisites exception, and exact
gesture semantics belong in the contract.

Sources: `projects.md`, `task-dependencies.md` sections 2/12/13/14,
`task-status-hooks.md`; decisions `task-deps-are-depends-on-links`,
`projects-block-from-prj-line`, `subprojects-inherit-project-blocking`,
`unblocked-tasks-are-offered`, and `capture-link-drops-blocking-deps`.

### `docs/concepts/reference-library.md` — The reference library

Distinguish the source/PDF, reference note and annotations, and the reading task in an
area/project/inbox note. Explain derived queued/started/finished/dropped reading state
without creating a second independent task workflow. A reading task can be linked into
Today and reviewed by the same ordinary freshness rules as other tasks. Blocked is a
separate overlay on reference reading state.

Explain why intake is not yet the indexed library, why the producing host normally files
a reference at creation, and how unsafe ingestion is queued. Annotation synchronization
remains separate; annotations are a scan snapshot. Keep this explanation distinct from
the detailed import/job recovery runbooks.

Explain identity carefully: URLs support deduplication, a recorded reference ID follows
its note stem (with a uniqueness suffix when needed), and the reading-task block ID
remains stable across supported bundle renames. Scope rename guarantees to supported
unannotated references rather than promising arbitrary file moves.

Sources: `ref.md` (reading state and identity), `ref-jobs.md`, `highlights-create.md`,
`highlights-ref-sync.md`; decisions `references-are-ingested-at-birth`,
`reference-renames-keep-identity`, and `reference-ids-follow-file-stems`.

### `docs/concepts/why-the-vault-syncs-through-git.md` — Why the vault syncs through git

Explain the practical model: each machine has the Markdown vault, commits record
changes, and Bob's sync cycle reconciles copies. Describe history, coordination with
maintenance writers, and bounded recovery as the useful properties of this design. Git
does not provide real-time collaborative editing or a lock across all hosts. The shared
maintenance lock serializes cooperating operations on one host; do not imply that
editors or remote machines share it.

Explain supported conflicts concretely: remote content stays at the working path, the
local version is preserved under `_conflicts/`, and quarantined copies are excluded from
active task views. Unhandled or repeated interrupted merges can halt sync and need
attention. Convey these tradeoffs and link the runbook for status, credentials,
triggers, recovery, and rollback, instead of copying personal machine names, private
paths, setup commands, or unsafe recovery recipes.

Use the reference-ingestion decision only to explain how reference creation fits the
synchronized vault and why annotation ownership stays separate. Link `vault-git-sync.md`
as the behavioral authority.

## 2. Connect the reader paths

1. Add all six pages once under the existing Understand section in `mkdocs.yml`, after
   the overview and system diagram, in the order above. Preserve generated nav regions,
   other sections, and mount placeholders.
2. Refresh `docs/concepts/index.md` within its 400-word ceiling. Keep the eight nouns,
   link each to its explanation, and offer all seven concept pages plus the glossary.
   Describe Plugin as an Obsidian extension and Mac Capture as a sibling program. Avoid
   claiming all newly captured tasks already have block IDs or that the vault is the
   only cache/state anywhere in the implementation.
3. Retarget the homepage strip:
   - Morning review → `concepts/freshness-and-the-review-walk.md`
   - Capture → `concepts/tasks-and-lanes.md#capture-makes-a-task`
   - Pomodoros → `concepts/today-and-the-pomodoro-ledger.md`
   - Close → `concepts/projects-and-dependencies.md#finishing-a-prerequisite`
   - Sync → `concepts/why-the-vault-syncs-through-git.md` Replace “close links what
     unblocked” with language about finishing work and choosing what comes next. Keep
     the homepage within 400 words and preserve changes from the guides, plugin, and Mac
     phases.
4. Make a narrow accuracy correction in the existing system overview if its present
   wording remains: the CLI owns capture grammar/planning/writes; Obsidian plugin
   gestures also mutate notes. Do not broaden the Mac thin-client decision into a claim
   that every plugin delegates every vault write to the CLI. Preserve that page's
   diagram, URL, and focus.
5. Add docs-root-relative `reader:` strings to `docs/_meta/contracts.yml`:

   | Contract stems                                                | Reader page under `concepts/`        |
   | ------------------------------------------------------------- | ------------------------------------ |
   | `task-status-hooks`                                           | `tasks-and-lanes.md`                 |
   | `plan`, `pomodoro-homes`, `today-marks`                       | `today-and-the-pomodoro-ledger.md`   |
   | `freshness`                                                   | `freshness-and-the-review-walk.md`   |
   | `projects`, `task-dependencies`                               | `projects-and-dependencies.md`       |
   | `ref`, `ref-jobs`, `highlights-create`, `highlights-ref-sync` | `reference-library.md`               |
   | `vault-git-sync`                                              | `why-the-vault-syncs-through-git.md` |

   Example value: `reader: concepts/tasks-and-lanes.md`. Preserve stems, orders,
   summaries, and historical flags. If a parallel phase has already supplied a focused
   guide as a banner destination, retain that guide and ensure it has a Related link to
   the concept; do not replace useful navigation blindly.

6. Add six concept-title smoke queries to `docs/_meta/search_smoke.yml`, keeping
   existing reference token queries. Use ordinary prose search terms and the new page
   locations. This checks discoverability without pinning paragraphs in tests.

## 3. Verify the documentation

Before editing, capture the clean base revision and run the docs gate to establish any
existing failure with the same mounted source revisions. The build currently mounts
bob-plugins. Open every needed mounted repository through `sase_repo` before tooling
consumes it, and use the returned path for any explicit `BOB_DOCS_<REPO>_DIR` override.
Do not let a build clone an alternate repository outside that workflow. Recheck the
mount table if another phase has landed.

After writing:

```bash
uv run --project tools/docs bob-docs lint
just docs-check
just docs-build-material
```

Use `just docs-gen` when generation is needed, then inspect generated diffs and rerun
the gate. Do not hand-edit generated decision/glossary pages or weaken hygiene, privacy,
budgets, link checks, or search checks. No new tests should pin concept prose or
duplicate existing linter coverage. Rust checks are only needed if implementation scope
unexpectedly includes Rust; this plan does not call for that.

The docs gate covers front matter, budgets, hygiene, generated drift, strict Zensical,
URLs and anchors, links, search, tutorial replay, media determinism, and tooling tests.
Confirm the six pages are in nav and the rendered contract banner links resolve to the
intended reader pages; relative template URLs need a rendered check as well as source
lint.

Verify the lane diagram in a local browser on both the Zensical and Material builds, in
light and dark themes at 360px and 1280px. Save scratch screenshots/report under
`target/docs/`; inspect them for clipped labels, poor contrast, unreadable text, or page
overflow. Confirm Mermaid produced an SVG with no parse-error panel rather than treating
a successful Markdown build as rendering proof. Review each new page for readable
headings, tables/examples, and Related links in both themes. The default `bob-docs a11y`
page matrix does not include these new pages, so explicitly visit them with the existing
Playwright dependency or a scratch audit; do not claim the default matrix alone proves
the lane diagram. Existing browser/a11y helpers can be reused without growing a new
committed testing framework.

Use `sase_monitor` before starting checks that need a long unattended run; do not leave
a live tool session running at turn end. Keep actual test/build/browser results in the
phase note, including any environmental limitations.

If a gate failure reproduces identically on the clean base with the same source set,
record it as `PROPOSED FOLLOW-UP:` on this phase, citing an existing tracking bead when
known. Continue validating the new pages with available targeted gates. Such a baseline
failure does not keep this phase open, per the task instruction. Failures introduced by
this implementation must be fixed.

## 4. Finish the assigned phase

Review the final diff for all six pages, routing, current behavior, and accidental
changes from generators or concurrent phases. Append verification evidence to
`bob-cli-a0.9`. Record any newly discovered out-of-scope work only as
`PROPOSED FOLLOW-UP:` notes on this phase; do not create task beads.

Immediately before closing, run:

```bash
sase bead epic-symbols bob-cli-a0.9
```

The planning read found no entries. If entries now exist, resolve them or re-key the
relevant justfile line to a still-open parent/later phase; verify no phase-owned entries
remain. Then close only the assigned phase:

```bash
sase bead close bob-cli-a0.9 --note "Implemented six concept pages and reader routing; verified <actual docs, search, link, and theme results; cite any baseline follow-up>."
```

Replace the placeholder with concrete evidence. Do not hand-set status, close the parent
epic `bob-cli-a0`, or close an ancestor plan bead. Finish through the required
`sase_final` workflow.
