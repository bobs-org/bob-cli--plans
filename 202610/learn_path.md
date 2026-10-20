---
tier: epic
title: Complete the Bob Learn path
goal: New readers can follow a verified first Bob day in a synthetic vault, understand
  component requirements, and continue into Obsidian or Mac Capture; executable tutorial
  checks and deterministic CLI media keep the published path accurate.
phases:
- id: tutorial
  title: Synthetic demo vault and executable first-day tutorial
  size: medium
  depends_on: []
  description: 'tutorial: build the date-independent demo vault and isolated Bash-fence
    runner, write the checkpointed first-day tutorial and its hub, and install the
    tutorial command, recipe, tests, and docs-check gate.'
- id: reader-path
  title: Learn continuations, system concept, and compatibility
  size: medium
  depends_on:
  - tutorial
  description: 'reader-path: complete the Get started pages, concepts hub and system
    diagram, compatibility template, navigation, homepage doors, footer links, and
    search smoke queries. Keep every new shell fence covered by the tutorial runner
    or an explicit skip.'
- id: cli-media
  title: Deterministic CLI media and Learn visual verification
  size: medium
  depends_on:
  - reader-path
  description: 'cli-media: reuse the isolated tutorial environment for real PTY captures,
    render deterministic Rich SVGs, add media drift checking, replace the homepage
    terminal illustration, and verify the completed Learn pages in both builders.'
proposed_by: bbugyi200.apollo.bob-cli-a0.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-a0.6
create_time: 2026-10-19 23:17:14
status: wip
bead_id: bob-cli-a0.6.1
---

- **PROMPT:** [prompts/202610/learn_path.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/learn_path.md)
- **PARENT:** [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:** [bob-cli-a0.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.6.1.md)

# Complete the Bob Learn path

This child epic implements the full `learn` scope assigned to **bob-cli-a0.6** in
`plan:202610/bob_public_docs_site.md`. It creates a safe first encounter with Bob:
readers copy a synthetic vault, capture a task, choose today's work, finish a Pomodoro,
and inspect the review queue. Automated checks execute those same commands and verify
the resulting notes and ledger. The surrounding pages explain installation, optional
desktop interfaces, and compatibility.

## Scope and evidence

Read the enclosing phase with
`sase bead read bob-cli-a0.6 -r "Need the enclosing Learn scope and decisions"`. Its
status was `in_progress`, with no earlier-work notes, during planning. Neither
`examples/demo-vault/` nor `docs/start/` nor `docs/concepts/` exists yet.
`sase bead epic-symbols bob-cli-a0.6` reported no entries.

The complete scope benefits from three coding agents: the tutorial runner and media
generator are new executable tooling, while the remaining reader pages and compatibility
template form a separate bounded deliverable. Each phase is sized `medium`; dependencies
are deliberately serial because they share the docs CLI, gate, navigation, and homepage.
Each phase must leave a useful site and passing checks, without links to pages that a
later phase has not created.

Sources used:

- `plan:202610/bob_public_docs_site.md`, especially the global documentation rules,
  build/provenance contract, and `learn` phase.
- `research:202610/bob_public_docs_site/bob_public_docs_site.md`, the required research
  report. It is available through `sase artifact read` after preparing the research
  sidecar with `/sase_repo`.
- `docs/getting-started.md`, `docs/capture.md`, `docs/plan.md`, and generated CLI
  references; these remain authoritative for behavior.
- `tools/docs/bob_docs/{cli,check,markdown,assemble,a11y}.py`, their tests,
  `mkdocs.yml`, existing page templates, and `docs/_meta/`.
- `vault/.obsidian/plugins/obsidian-tasks-plugin/data.json`, `tests/cli/support.rs`, and
  capture tests for block IDs, named starts, and close outcomes.
- Linked bob-plugins manifests currently require Obsidian **1.8.7**. The linked Mac
  README requires **macOS 26** and Apple **Xcode 26+ Command Line Tools or Xcode 26+**.
  Recheck these sources through `/sase_repo` when writing. `Cargo.toml` declares Rust
  edition **2024**.

The enclosing decisions are final: MIT license, search indexing enabled, public
generated design decisions, and memory-edit authorization limited to `docs.md`. This
work needs no memory edits. Do not reopen those decisions. Use the audited memory-read
skill for `docs.md`, `cli_rules.md`, relevant glossary terms, and the governing
decisions before implementing. In particular, retain the distinctions between sticky
lanes, ledger-derived Today, and freshness; Mac Capture owns its UI and delegates
capture semantics and vault writes to Bob.

All implementation belongs to bob-cli. The linked component repositories are read-only
sources for requirements and installation instructions; open them through `/sase_repo`.
The Mac and plugin documentation mounts belong to sibling phases. Do not change Swift,
plugin implementation, the private vault, publication machinery, existing contract URLs,
or product capture semantics to make a tutorial pass.

## Shared implementation contract

### Documents and URLs

Use these source paths and navigation order:

| Page                           | Source                                   |
| ------------------------------ | ---------------------------------------- |
| Get started                    | `docs/start/index.md`                    |
| Is Bob for me?                 | `docs/start/is-bob-for-me.md`            |
| Install Bob                    | `docs/start/install.md`                  |
| Your first Bob day             | `docs/start/first-bob-day.md`            |
| Continue in Obsidian           | `docs/start/obsidian.md`                 |
| Continue with Mac Capture      | `docs/start/mac-capture.md`              |
| Concepts                       | `docs/concepts/index.md`                 |
| How Bob fits together          | `docs/concepts/how-bob-fits-together.md` |
| Requirements and compatibility | `docs/reference/compatibility.md`        |

Add nav entries only with their pages. Preserve existing generated nav blocks. Follow
the existing tutorial/concept/hub templates, required front matter, portable Markdown,
and word budgets: hubs/homepage at most 400 words, tutorials at most 2,500 each and
10,000 total, concepts at most 3,000. Keep implementation and tracker terminology out of
reader prose. Use same-repo relative links. Link the public component READMEs until
their mounted reference pages actually exist.

Shell commands use `bash` fences without prompts. Output uses a separate
`text { .no-copy title="Output" }` fence. GUI, network, installation, and plugin sync
steps have explicit `tutorial-test: skip` reasons and remain real, complete instructions
for readers. The runner must report each skip with its source.

### Demo and isolation

Use a cheerful generic gardening/workshop persona without real names, private paths,
remotes, or copied vault content. Include an inbox (`mac_inbox.md`), two lowercase area
notes (for example `garden.md` and `workshop.md`), one lowercase project note with an
appropriate parent and a `^prj` task, and a synthetic reference note. Include a short
purpose/license README. Ship no binaries, Git repository, scheduled/created/freshness
timestamps, or dated daily notes.

The minimal Tasks settings must explicitly set `taskFormat: "dataview"` and
`globalFilter: "#task"` and provide the compatible status registry. Do not copy `vault/`
verbatim: its settings omit `taskFormat` and use a historical Blocked type. Check
`docs/getting-started.md` and the live parser when choosing the fixture's settings.

One shared Python sandbox helper should serve tutorial and media execution. It creates a
temporary home, XDG directories, temporary working directory, and copy of the demo
vault. Set the subprocess environment's `HOME` and XDG paths, `BOB_DIR`,
`BOB_CONFIG_FILE`, `BOB_NOW`, `TZ=UTC`, `BOB_REF_JOBS_KICK=off`, and any day-file
override consistently inside that sandbox. Clear inherited Bob overrides and
desktop/clipboard state that could redirect execution. Do not repurpose the agent
shell's `HOME`.

Resolve `$BOB_BIN` explicitly when provided and fail clearly if unusable. Otherwise
build the current checkout with `cargo build --locked --bin bob` before switching to the
isolated environment; resolve the executable through Cargo's actual target directory
rather than assuming `target/debug`. Put that binary first on sandbox `PATH`, and never
fall back to an unrelated installed Bob. Prepare once per gate invocation and share the
prepared binary.

The fixed automated clock can be `2026-10-20 09:02:00`; choose a fixed UTF-8 locale
available on Linux and macOS. Human instructions use the current local date. Daily-note
setup must honor the harness's fixed clock when replayed, so wall-clock `date` cannot
silently select a different file. Test a second date. All temporary files stay under the
sandbox and are cleaned up.

## Phase tutorial

1. Add the demo fixture and `bob_docs/tutorial.py` with any small shared helper needed
   for isolation. Implement extraction using the existing Markdown fence helpers where
   appropriate. Read `start/*.md` in the Get started nav order and preserve each page's
   fence order. Reject uncovered/orphan start pages rather than silently omitting them.
2. Execute the published Bash bodies in a persistent `bash` session with
   `set -euo pipefail`, or generate an equivalent single driver. Variables and `cd` must
   survive across fences. Report a failed page, fence line, command status, and useful
   output; add a bounded timeout and reap subprocesses.
3. Define unambiguous marker semantics:
   - `<!-- tutorial-test: skip <reason> -->` immediately before a Bash fence skips that
     fence only. Reasons must be nonempty.
   - `<!-- tutorial-test: checkpoint <id> -->` immediately after a Bash fence validates
     the just-completed state, before subsequent mutations.
   - Reject malformed markers, unknown IDs, unconsumed markers, and missing required
     checkpoints. Treat an unmarked Bash fence as executable.
   - Only fences actually parsed as Markdown code count; example comments and nested
     illustrative fences must not acquire execution semantics.
4. Write `start/index.md` and `start/first-bob-day.md` together with the runner. Make
   fixture copying and vault selection visible, safe to paste, and reproducible by the
   harness, without requiring its hidden setup for success. The documented setup must
   always choose a new demo copy even if the reader already has `BOB_DIR` set. Use a
   task-specific temporary-directory variable, not a redefinition of the reader's home.
5. Use a narrow scenario with explicit IDs: capture an inbox task, capture a routed area
   task such as `Water the seedlings @garden^seedlings`, and inspect `bob plan`. Explain
   that new unstamped tasks enter review. Create today's daily-note skeleton with an
   empty `## Pomodoros` section explicitly in the tutorial. Bob does not bootstrap this
   file. Then use a named `bob capture '=#focus'` to create/start a session, and
   `bob capture '@garden:seedlings'` to link the existing task. Verify this exact
   sequence against the current binary before settling prose. Close with
   `bob capture 'x!'` for the one-task completion outcome. Finish with the read-only
   `bob freshness list -f json` review inspection, explaining that answering the review
   is a desktop continuation.
6. Implement reusable real-effect checkpoint assertions, exercised through pytest: the
   captured task exists exactly once in the correct note; its explicit ID is unique; the
   daily path matches the run date; the Task Link is a direct child of the intended open
   Pomodoro; linking produces Next/Today as reported by `bob plan -f json`; starting
   alone does not invent a Pending transition; completion leaves a Done task with a
   completion date and a closed ledger entry; Today no longer contains that completed
   task; the other new task remains in the review queue. Assert semantic fields using
   the current JSON schema rather than copying human output. If closing immediately has
   a zero-duration result, test and explain the actual result; do not sleep or invent
   elapsed work.
7. Wire `bob-docs tutorial`, `just docs-tutorial`, and the docs-check gate. Respect the
   CLI help conventions. With Cargo or an explicit usable `BOB_BIN`, docs-check runs the
   real tutorial and propagates failures. With neither, print an explicit capability
   skip. Direct `docs-tutorial` fails with an actionable dependency error. Remove only
   the obsolete tutorial skip message; retain unrelated gate behavior.
8. Add focused parser, skip, checkpoint, shell-state, timeout, and isolation tests.
   Include a failed pipeline, bad checkpoint, and hostile inherited Bob environment. A
   real integration test runs the actual published first-day commands and assertions; do
   not substitute a parallel handwritten scenario. Keep unit tests fast and avoid
   recursively running docs-check or rebuilding Bob for every test.
9. Update tooling usage documentation and add the populated Get started nav.

Acceptance: pytest for the new runner and existing check integration passes;
`just docs-tutorial` works with an isolated home and on a second simulated date;
`just docs-check` passes. No tracked fixture changes or outside-vault writes occur
during a run. Record the executable and commands used in the child phase note.

## Phase reader-path

1. Complete the hub with time estimates and the remaining four Get started pages.
   - **Is Bob for me?** State the opinionated GTD workflow and intended technically
     capable Obsidian audience. Explain what the Rust CLI alone can exercise, what
     requires Tasks and custom plugins, and that Mac Capture is optional.
   - **Install Bob.** Start from Git, a current stable Rust toolchain, and `just`; give
     the public source checkout, `just install`, PATH guidance, and a meaningful
     verification step. Installation/network fences are explicitly skipped in replay;
     safe Bob verification runs using the prepared binary. Link Compatibility rather
     than duplicating the whole requirements matrix.
   - **Continue in Obsidian.** Open the copied vault, install/enable Tasks, confirm
     Dataview format/status settings, clone the public plugin source, and run
     `bob plugins sync` explicitly against the demo `BOB_DIR`. Explain that sync deploys
     assets but does not enable them or create dashboards. Show the rendered task lines
     and the supported review gesture with Bob Ledger Tools and Bob Navigation Hotkeys
     enabled.
   - **Continue with Mac Capture.** Describe its source build, selecting Bob and the
     demo vault, the first capture and result, and the required Accessibility
     interaction with the current README's actual purpose. Link detailed
     signing/toolchain instructions rather than copying the protocol changelog. Do not
     claim the Linux runner exercised macOS or Obsidian.
2. Complete the Concepts hub with the eight nouns Vault, Task, Lane, Pomodoro, Capture,
   Freshness, Project, and Plugin. Present Mac Capture as a sibling program. Link one
   substantive system page and the existing glossary for terms; later concept pages are
   owned by the enclosing epic's concepts phase.
3. Write `how-bob-fits-together.md` with one Mermaid diagram showing Markdown vault
   ownership and the Mac capture round trip: draft, Bob parsing/completion/ preview,
   rendered UI, one capture submission, Bob writes, success response. Explain the role
   of Obsidian plugins without making an inaccurate claim that they never write the
   vault. Link the generated thin-client decision anchor.
4. Add `reference/compatibility.md` with `template: compatibility.html` and a matching
   override. Render the tested rolling snapshot from `config.extra.build.sources`,
   `.tools`, `.built_at`, and `.builder`, then display the authored minimum
   requirements. Reuse the provenance conventions; escape values and handle absent
   mounts or optional tools truthfully. Do not infer a tested Obsidian/Tasks version
   from a builder version or repository SHA. Rust is a current stable compiler
   supporting edition 2024; do not invent an MSRV. State the supported Linux/macOS CLI
   baseline, manifest-derived Obsidian floor, and optional Mac requirements.
5. Establish the Tasks version evidence from existing parity tooling or a documented
   fixture source and record it accurately. `docs/dataview.md` currently mentions Tasks
   8.4.0 behavior; that alone does not prove a new GUI run. Distinguish a tested
   headless/parity baseline from desktop verification still awaiting a human. If no
   verifiable tested version is available, state that limitation and record a proposed
   follow-up instead of inventing one.
6. Add Learn and Understand homepage doors, update its introductory Get started link,
   and add footer links for **Is Bob for me? · Requirements & compatibility**. Preserve
   the homepage budget and the existing Look up/Develop doors. Add all page nav entries,
   link Compatibility from the Reference hub, and extend `search_smoke.yml` with
   first-day, install, requirements, and system concept queries targeting real pages.
7. Update the tutorial template and `develop/writing-docs.md` to explain markers,
   executable fence ordering, and when a skip is appropriate. Ensure the runner
   discovers every new start page. Do not add unapproved front-matter keys.
8. Test the compatibility override with complete and partial build provenance, including
   dirty sources, using the actual supported template environment. Run strict Zensical
   and Material builds to verify the custom template and Mermaid rendering. Avoid
   prose-pinning Rust/JS tests.

Acceptance: `just docs-tutorial`, the docs unit suite, `just docs-check`, and
`just docs-build-material` pass. Every command is verified from current source or has an
explicit manual/network skip. Both builders render the generated compatibility block and
system diagram with valid links.

## Phase cli-media

1. Add `bob_docs/media.py`, `bob-docs media [-c|--check]`, and `just docs-media`. Add
   Rich as an explicit dependency in `tools/docs/pyproject.toml` and update `uv.lock`.
   Reuse the prepared executable and isolation helper from tutorial; keep the capture
   command list small and declared in code.
2. Capture actual Bob output under a POSIX PTY with fixed terminal dimensions, `TERM`,
   locale, UTC clock, and a fresh demo. Capture at least the simple creation command
   used by the homepage and a useful plan view after linking. Ensure Bob actually emits
   ANSI color. Give the child a timeout, preserve its exit status, handle PTY EOF
   correctly on Linux and macOS, and reap it.
3. Convert ANSI output through Rich into `docs/assets/media/cli/capture.svg` and
   `plan.svg`, using Bob's existing palette and bundled JetBrains Mono. Ensure the font
   resolves inside an SVG loaded as an image, without a remote font request. Use
   deterministic SVG IDs, fixed titles, and fixed generator metadata; normalize only
   known sandbox-path/terminal transport variation, never semantic output. Reuse
   existing theme/font resources rather than inventing another brand.
4. Register the two images in `docs/_meta/media.yml` with real provenance (commands,
   fixture, renderer) and the intentional generation date. Ordinary `--check` must not
   refresh that date or write any tracked file. Render twice from independent sandboxes
   and byte-compare both runs plus the committed assets. Drift reports the affected path
   and regeneration command.
5. Add media determinism to docs-check and remove the unimplemented media skip. It runs
   with Cargo or `BOB_BIN`, otherwise explicitly reports the same unavailable-runtime
   reason as the tutorial. Unit tests still run without Cargo. Cover ANSI preservation,
   stable SVG generation, failure propagation, and nonmutating drift detection with
   meaningful tests.
6. Replace the homepage terminal illustration with the real capture SVG and descriptive
   alt text. Retain an adjacent copyable command/text explanation so the workflow is
   accessible without reading the image. The image must remain readable on narrow
   layouts in both themes.
7. Extend the existing a11y page matrix with the first-day tutorial and Compatibility;
   include the concepts diagram in visual inspection. Use the existing Playwright
   tooling for 360/768/1280 widths in light/dark, verify keyboard focus, overflow, 200%
   zoom, copy buttons, footer links, and compatibility table wrapping. Inspect the
   actual screenshots. Store audit outputs under `target/docs/`, not as public images.
8. Update tooling documentation with media generation/check instructions. Record the
   required enclosing-phase note:
   `PROPOSED FOLLOW-UP: Clean-machine tutorial walkthrough — Bryan should follow /start/ from a fresh install, open the demo in Obsidian, and try the Mac continuation; automated Linux checks cannot verify those GUI interactions.`

Acceptance: unit tests, `just docs-tutorial`, media `--check`, and `just docs-check`
pass; regeneration is byte-stable without tracked modifications. Strict Material also
builds, and both builders' representative Learn screenshots have been inspected. Record
actual checks and any environmental limitations.

## Integration, follow-ups, and closure

Every coding phase runs its relevant pytest tests and `just docs-check`. Run
`just check` if any Rust changes turn out to be necessary; the intended implementation
needs none. Use `/sase_monitor` for long builds/checks and wait for each handoff command
to exit as required by the runtime.

Children close only their own assigned child phase, after checking
`sase bead epic-symbols <their-id>`. They do not close bob-cli-a0.6 or any other
ancestor. Discovered out-of-scope work is recorded as `PROPOSED FOLLOW-UP:` on the
enclosing bob-cli-a0.6 phase for its parent epic's land agent to triage; do not create
task beads.

The child epic's land agent integrates the three phases and verifies the complete scope
against the enclosing `learn` acceptance criteria. It consolidates evidence on
bob-cli-a0.6, including the clean-machine follow-up, commands, meaningful checkpoint
results, media determinism, and screenshot findings. Before closure, run
`sase bead epic-symbols bob-cli-a0.6`; resolve any entries or re-key remaining Justfile
entries to a still-open owner.

The user's explicit child-epic instruction designates this child land agent to finish
and close **bob-cli-a0.6** once its descendant work is complete:
`sase bead close bob-cli-a0.6 --note "<specific verification evidence>"`. Use the normal
child-epic settlement flow first so no unfinished descendants block that close. Do not
close **bob-cli-a0**, the public-docs parent epic, or any other ancestor. Never hand-set
status.

A failure reproduced identically on the clean base is recorded with exact evidence and
any already-tracking task ID as a `PROPOSED FOLLOW-UP:`; it does not strand the
enclosing phase open. New failures caused by this work must be fixed. Preserve other
agents' changes to shared navigation and tooling during integration.
