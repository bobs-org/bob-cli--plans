---
tier: tale
title: Repair Learn gaps and finish its landing
goal: Correct Learn runtime and reader defects, integrate mounted references, verify
  the result, and close bob-cli-a0.6.1 and phase bob-cli-a0.6.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.6.1
status: done
---

- **PARENT:**
  [202610/learn_path.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/learn_path.md)
- **BEAD:**
  [bob-cli-a0.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.6.1.md)

# Repair the remaining Learn gaps and finish its landing

Finish **bob-cli-a0.6.1**, then close its containing **phase bob-cli-a0.6**. This is one
bounded implementation and landing task, not another epic. The original Learn
implementation is present; do not rebuild it or wait for another land agent. This tale's
coder must complete the closeout in the same turn as the last code changes, before the
host creates that turn's commit.

## Context and verified baseline

Read these through the audited bead/artifact commands:

- `sase bead read bob-cli-a0.6.1 -r "Need Learn landing findings and follow-up outcomes"`
- Each of `bob-cli-a0.6.1.1`, `.2`, and `.3`, with
  `-r "Need the child scope and notes"`.
- `sase bead read bob-cli-a0.6 -r "Need the containing phase and final decisions"`
- `sase artifact read plan:202610/learn_path.md "Need the original Learn acceptance"`
- `sase artifact read plan:202610/bob_public_docs_site.md "Need enclosing Learn and handoff requirements"`

The land audit read all three children's notes, their source and tests, and commits
`7751f505` (tutorial), `48f3ef22` (reader pages), and `50d0e990` (media). All three
phases are closed. At audit time HEAD and remote master both resolved to
`50d0e990138287513304fbde2487cf54defbb86c`.

The date-independent fixture, persistent Bash replay, nine state checkpoints, two-date
integration tests, deterministic PTY/Rich SVG generation, shared binary preparation in
docs-check, navigation, search smoke, and compatibility template exist. The media check
renders twice and compares committed bytes without writes. The children report
successful full documentation gates and both builders' visual inspection. The lander
independently ran:

```bash
uv run --project tools/docs pytest tools/docs/tests/test_tutorial.py tools/docs/tests/test_media.py tools/docs/tests/test_compatibility.py tools/docs/tests/test_check.py -m 'not integration' -q
```

Result: **29 passed, 3 deselected**. This was not a fresh full-gate or GUI pass. Two
standalone reproductions nevertheless confirmed the runtime defects below.
`sase bead epic-symbols bob-cli-a0.6.1` reported no entries; repeat at closure.

Intervening work reviewed includes the publication planner/workflows/operator commands
and weekly link checks (`d18b45de`, `9ea12d2d`, `d5c318a9`, `dadf0734`), product
references (`5d3ad7d0`, `6f6841b5`, `f30389c7`, `cf7a9e98`), and mounted plugin docs
(`f3678291`, `b1f98d99`). The normal docs-check path preserves tutorial/media and the
mount-aware generated check. `cf7a9e98` already added product-reference links to the
first-day page; retain them. `b1f98d99` creates the plugin reference that Learn must now
link to. Check for additional landed changes before editing and again before closing.

## Constraints and decisions

Use `/sase_memory_read` for `docs.md` and relevant glossary/decision strands. The lander
read `decisions:mac-capture-is-a-thin-client`, `decisions:task-lanes-are-sticky`,
`decisions:today-is-read-from-the-ledger`, and the Capture, Task Link, and Task
Freshness glossary definitions. Preserve ledger-derived Today, sticky lanes, and
freshness as distinct concepts.

The enclosing DECISIONS are final: MIT license, indexing enabled, generated public
decisions, and memory-edit permission limited to `docs.md`. No memory edit is needed. Do
not change capture semantics, private vault files, Swift or plugin implementations,
publication machinery, existing public URL paths, or the broader theme to make this tale
pass.

Open linked repositories through `/sase_repo` before reading their current requirements
or instructions; use only its returned checkout paths. The plugin reference source is
`bob-plugins/docs/`, mounted at `reference/plugins/`. The Mac README remains the
authoritative destination until its mount actually lands. Read source for technical
facts instead of guessing a GUI workflow.

## 1. Make the selected tutorial binary authoritative

In `tools/docs/bob_docs/sandbox.py`, `isolated_environment` currently puts
`bob_bin.parent` first on PATH. The tutorial executes literal `bob` commands, while
checkpoints and media use the explicit absolute executable. An executable selected via
`BOB_BIN=/.../versioned-bob` therefore does not govern the tutorial: an ambient PATH
entry named `bob` wins. Resolving a symlink can expose this even when the supplied
symlink was named `bob`.

The lander reproduced this with two temporary shell executables, `versioned-bob` and
`ambient/bob`: after sandbox construction, `shutil.which("bob", path=env["PATH"])`
resolved to the ambient decoy.

Create a sandbox-owned bin directory containing a `bob` entry that runs the resolved
selected executable, and put that directory first on PATH. Keep checkpoints/media on the
same selected executable. Preserve explicit invalid BOB_BIN errors, Cargo
target-directory resolution, single preparation per gate, and cleanup. Never fall back
to an installed Bob.

Add a meaningful regression using an executable with a different basename and an ambient
decoy: execute a published-style `bob` fence and prove only the selected executable ran.
Cover a selected symlink whose resolved basename differs as appropriate. Do not merely
assert the new PATH string.

## 2. Bound PTY execution through early EOF

In `tools/docs/bob_docs/media.py`, `capture_pty` breaks on PTY EOF/EIO and then calls
unbounded `proc.wait()`. A process may close stdin/stdout/stderr without exiting. The
lander ran a child that closes descriptors 0/1/2 and sleeps one second with
`timeout=0.1`; capture returned status 0 after 1.03 seconds instead of timing out.

Apply the remaining deadline to process completion after EOF as well as to terminal
reads. On timeout/error, terminate and reap the process group, close both PTY
descriptors, and raise the existing actionable DocsError. Preserve normal Linux
EIO/macOS EOF handling, ANSI bytes, and nonzero exit status. Add a short regression for
early descriptor closure followed by a sleeping child; prove timeout and cleanup without
timing-sensitive narrow thresholds. Retain the existing ordinary timeout and
nonzero-status tests.

## 3. Repair reader instructions and integrate mounted references

Keep these edits concise and within the existing page types and budgets.

- **`docs/start/obsidian.md`:** install Tasks from Community plugins, then clone/deploy
  the custom Bob plugins from their public source and enable them after deployment. Do
  not tell readers to find the custom plugins in the Community directory. Give an actual
  clone command with a known path and
  `bob plugins sync --repo <that-path> --bob-dir "$BOB_DIR"`. The CLI's
  `src/native/plugins/cli.rs` defaults to BOB_PLUGINS_DIR or a personal checkout path;
  simply saying “clone, then sync” is insufficient and can refuse from a foreign
  checkout. Preserve runtime Tasks settings and the fresh demo vault. Link
  `../reference/plugins/index.md` and the specific mounted navigation/ ledger reference
  pages for review details. Verify an accessible review gesture from the plugin
  source/reference, including editor/Vim prerequisites if using `]s`. Explain the inbox
  routing/confirmation interaction that may precede the freshness stamp instead of
  promising Alt+F only adds a stamp.
- **`docs/start/mac-capture.md`:** the current page stops at `just build` before
  discussing an app that has not been launched, and only mentions the README without a
  clickable link. Provide a source-verified build/bundle and launch path, actual
  settings labels for executable and demo vault selection, and links to authoritative
  signing/toolchain instructions. Preserve the accurate limited purpose of Accessibility
  (Highlights tabs; ordinary capture does not require it). Its proposed first draft
  currently repeats `Water the seedlings @garden^seedlings`, whose ID already exists
  after the first-day tutorial. Use a distinct draft/ID and clearly show the text to
  submit in the panel, then explain how to confirm the result. Validate that draft's CLI
  semantics against the post-tutorial demo state; do not pretend Linux ran the Mac GUI.
- **`docs/concepts/how-bob-fits-together.md` and `docs/concepts/index.md`:** limit the
  thin-client/no-direct-vault-write claim to Mac Capture. The current prose wrongly says
  the plugins delegate all vault mutation to Bob, and the glossary-style hub calls Mac
  surfaces plugins that read the vault. Obsidian plugins implement editor/review
  gestures and write Markdown; Mac Capture is a separate application that calls Bob.
  Keep a single diagram and show the successful capture response returning to the panel,
  not just the submit arrow to the vault.
- **`docs/start/first-bob-day.md` and hub:** make the end of the tutorial lead directly
  to the desktop continuations, retaining the reference links added by `cf7a9e98`. Say
  the zero-minute outcome belongs to automated fixed-clock replay; a human following the
  instructions normally uses their local clock. Make any Bash-shell prerequisite
  explicit where Bash-specific syntax is used.
- **`docs/reference/compatibility.md`:** keep verified minimum floors and the
  distinction between headless Tasks parity and unperformed desktop testing. Link
  current authoritative component requirements instead of referring to an unlinked
  “repository README.” Prefer mounted docs when actually available.

Every new Bash fence in `start/` must either execute in the tutorial's persistent shell
or have an adjacent `tutorial-test: skip <specific reason>`. Do not silently hide
prerequisites in the sandbox or turn executable first-day steps into skips. Do not add
prose-pinning tests.

## 4. Keep local checkout paths out of the compatibility page

`docs/overrides/partials/bob-compatibility.html` currently renders `info.path`.
`assemble._manifest` can put an absolute checkout path there when a mounted source is
outside the primary root. The existing partial-provenance test asserts no local path
leaks but supplies only `"."` and `"link"`.

Omit checkout filesystem paths from reader-facing compatibility rendering. Continue
showing repository, revision, dirty/branch status, tools, build time, builder, and the
build.json link. Do not expand into a provenance-format or publication refactor. Add a
rendering regression with an absolute private source path and untrusted text that
verifies paths are absent and displayed values remain escaped. Exercise the supported
builders as part of the gate.

## 5. Verify the integrated result

Run the focused regression tests, then `just docs-check` (includes actual tutorial
replay, both date integrations, and media determinism) and `just docs-build-material`.
Confirm both builders render the revised concept diagram, compatibility snapshot, and
continuation links. Use the existing Playwright tooling and inspect representative Learn
screenshots at narrow and desktop widths in light/dark; fix any regression this tale
introduces. Shared pre-existing a11y failures have already been routed as described
below.

Run **`just check`** for file-change verification. Never run `just check-full`. Do not
treat a check-full-only failure as missing epic functionality. Use `/sase_monitor` for
long commands, with a concrete successor instruction to return to this tale's closeout;
wait for each monitor handoff command to exit. Do not wait for this turn's own SHA,
push, or CI: the host commits after the turn ends. No approval or second land agent is
needed for ordinary closeout.

Confirm verification does not modify demo fixtures or committed media. If failures are
pre-existing, establish exact cause and use the required follow-up workflow; failures
introduced by Learn or these repairs remain work to fix before closure.

## Follow-up triage already completed by the lander

The lander used `/sase_new_task`, read the size/lifecycle policies, searched all task
statuses, swept every task from the last week, and inspected the active public-docs epic
and its theme/polish phases. No task duplicate was found. Outcomes are recorded on
bob-cli-a0.6.1 and bob-cli-a0:

1. **bob-cli-a0.6.1.2 note #1** (human Obsidian rendering/review pass) and
   **bob-cli-a0.6.1.3 note #1** (clean-machine tutorial/Obsidian/Mac pass) are one
   manual verification deliverable. A standalone task was declined because the active
   outer epic explicitly assigns this to its final handoff, **bob-cli-a0.13**. A
   DISCOVERED ISSUE on bob-cli-a0 preserves exact steps and the fact that Linux has not
   performed this human pass.
2. **bob-cli-a0.6.1.3 note #2** reports shared-theme axe/zoom findings already present
   in **bob-cli-a0.3 note #2**, predating Learn. A DISCOVERED ISSUE on the causal active
   epic **bob-cli-a0** assigns them to its final accessibility sweep; no standalone task
   was created. This includes header/search ARIA, syntax-token contrast, scrollable
   contract regions, and 200% reflow. Any genuinely new Learn-specific regression still
   belongs in this tale.

Do not reopen this triage or create duplicate tasks absent new evidence. Include these
outcomes and their reasons in the final close note.

## 6. Finish the landing in this coder turn

This step is mandatory and is the end of the tale.

1. Re-read **bob-cli-a0.6.1**, its descendants/notes, and `plan:202610/learn_path.md`;
   confirm all original Learn requirements and the repairs above are complete. Review
   post-audit commits for drift and preserve other agents' navigation/reference work.
   Ensure every descendant is completed normally; if approval created a descendant bead
   for this tale, close that completed bead before closing the epic.
2. Run `sase bead epic-symbols bob-cli-a0.6.1`. Resolve every entry by wiring,
   privatizing, a justified non-test pragma, or deletion under the Symvision policy;
   re-key only to a still-open later bead that actually needs it.
3. Close normally:
   `sase bead close bob-cli-a0.6.1 --note "<specific scope/code/commit review, integration changes, successful verification, and all follow-up triage outcomes>"`.
   If rejected for symbols or incomplete phases, fix the actual blocker and retry. Never
   use `--force` to make a successful nested landing advance. Cancellation/supersession
   requires a genuine deliberate outcome, explicit reason, and resolution; it is not the
   route for these completed repairs.
4. Run `just symvision` if available; report explicitly if there is no recipe. Use
   `/sase_repo` to open the plans sidecar before editing, resolve the original epic's
   PLAN path through its bead/artifact reference, and set **`status: done`** in
   **`plan:202610/learn_path.md`** frontmatter. Preserve all other plan content.
5. Re-read the parent link with
   `sase bead read bob-cli-a0.6.1 -r "Need the parent link"`. The expected parent is
   **phase bob-cli-a0.6**. Verify that this child completed its entire Learn scope.
   Check and retire `sase bead epic-symbols bob-cli-a0.6`, then close only that phase
   normally:
   `sase bead close bob-cli-a0.6 --note "<Learn evidence, integration and triage outcomes>"`.
   Record the required
   `PROPOSED FOLLOW-UP: Clean-machine tutorial walkthrough — Bryan should follow /start/ from a fresh install, open the demo in Obsidian, and try the Mac continuation; automated Linux checks cannot verify those GUI interactions.`
   on bob-cli-a0.6, identifying its existing outer-epic ownership rather than filing a
   duplicate. Leave **bob-cli-a0** open for its waiting land agent; do not mark
   `plan:202610/bob_public_docs_site.md` done.

If the parent relationship or completeness has changed, stop at the first incomplete or
ambiguous parent, record the specific blocker on it, and report that state. Finish with
`/sase_final`; include all changed repositories, including the plans sidecar, in the
host's commit declaration. The absence of this turn's commit in git log is expected and
does not delay closure.
