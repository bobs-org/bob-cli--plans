---
tier: tale
size: medium
title: Finish bob-cli-a0.13 (docs polish) and hand the epic to its land agent
goal:
  bob-cli-a0.13 is closed with its acceptance met. The salvaged work is on master,
  llms.txt is live, the weekly check is green, live reader-test results and the handoff
  artifact are recorded, and bob-cli-a0.land has started.
decisions:
  weekly_exclusion:
    ask:
      How should the weekly link check stop failing on the Adobe URL in the vendored
      font license?
    choices:
      url:
        Skip only the exact http://www.adobe.com/ URL; other OFL.txt links stay checked
      file:
        Skip the whole vendored assets/fonts/OFL.txt file; none of its URLs are checked
    default: url
    why: This is the narrowest possible exclusion, and the epic plan forbids broad ones
    answer: url
proposed_by: bbugyi200.apollo.9m
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.9m](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.9m.md)
- **COMMITS:**
  - [1513a38](https://github.com/bobs-org/bob-cli/commit/1513a385a691bedbafe13c36de3205e83ce7c7d0)
    — feat(docs): add llms.txt, reader-testing protocol, and final accessibility fixes

# Finish bob-cli-a0.13 (docs polish) and hand the epic to its land agent

## Why

Epic `bob-cli-a0` (Bob public docs site, `plan:202610/bob_public_docs_site.md`) has 12
of 13 phases closed. Only `bob-cli-a0.13` ("llms.txt, reader tests, final sweep, and
handoff") is still `in_progress`. Its waiting land agent `bob-cli-a0.land` starts only
after every phase bead closes, so the whole epic is stuck on this one phase.

What happened to a0.13 (verified 2026-10-20 ~17:50Z):

- Its worker (codex, run `bob-cli-a0.13--plan`, chat
  `gh_bobs_org__bob_cli-tmp_261020_101518-main-261020_111554`) did most of the phase:
  - It enabled `llmstxt`.
  - It wrote `docs/develop/reader-testing.md`.
  - It fixed the shared-theme accessibility defects from epic note #4.
  - It got `just docs-check` green (434 passed).
  - It reported a clean screenshot-free axe, reflow, keyboard, and no-JS sweep on both
    builders.
- It then requested a 10-agent reader-test launch through `/sase_run`. That launch
  handoff ended its turn, so the commit finalizer was skipped
  (`phase_skipped handoff:gate`). None of its work reached master.
- The ten readers ran against a site without the change:
  - All five `llms.txt` readers got HTTP 404.
  - Two site readers could not open the home page with their browser tool.
  - One site reader failed on model capacity.
  - Two site readers finished all six tasks.

  Those results do not count as the phase's reader test.

- The gate's follow-up agent `bob-cli-a0.13--1` has been `WAITING` on its own session
  `bob-cli-a0.13` since 16:04Z, two hours after that session finished. It is stranded.
  It still holds the old workspace, whose uncommitted tree matches the saved diff below.
- The saved diff of that work is stable at
  `~/.sase/projects/gh_bobs-org__bob-cli/artifacts/ace-run/202610/19/20261019185631/main_diff.txt`
  (584 lines, 13 files). `git apply --check` passes on current master (`eefdeb47`).

Other epic-owned follow-through still open, from the epic's notes:

- **Note #1/#2:** `docs/highlights-create.md` ends a sentence with "; see the PDFKit and
  text-layer research." That clause has no public target.
- **Note #5:** the weekly online link check is permanently red. `http://www.adobe.com/`
  in `docs/assets/fonts/OFL.txt` returns 403. Managed issue
  `bobs-org/bobs-org.github.io#2` stays open until a weekly run passes.
- **Note #6:** hourly scheduled publishes are sparse. Scheduled runs exist only at
  10:09Z (which failed on a since-fixed `runs-on`) and 15:10Z. The live `/build.json` is
  bob-cli `a02f9b33`, behind master.

## Ground rules

- Honor the epic's final DECISIONS shown by `sase bead read bob-cli-a0` (MIT, indexing
  yes, decisions page yes; only `sase/memory/docs.md` may be edited, and this plan needs
  no memory edit).
- Fix reader-test misses with **labels and hub copy only, never page moves or URL
  changes**.
- **Do not use `/sase_run`, `sase launch request`, or any SASE handoff for the reader
  test.** A launch handoff is exactly what stranded the last attempt. Run every reader
  inside your own turn, using one of these two paths:
  - **(a) Native subagents.** In Claude Code, use the Agent tool with
    `subagent_type: "web-fetch"`. It has only WebFetch, so readers cannot see the
    checkout.
  - **(b) Fallback for harnesses with only shell and edit tools** (the earlier codex
    worker had only these). Run each reader as a synchronous one-shot
    `claude -p --allowedTools "WebFetch(domain:bobs-org.github.io)" "<prompt>"`.
    - Run it from a fresh empty temporary directory outside every repository.
    - Save its stdout to a scratch file.
    - Every other tool is denied in print mode, so the reader stays blind.
- Run long commands (`just docs-publish`, workflow watches, a11y sweeps) synchronously
  with explicit long timeouts.
- Do not create task beads. Record unresolved work as `PROPOSED FOLLOW-UP:` notes on
  `bob-cli-a0.13` with `sase bead note`, so the epic's land agent triages it.
- **Do not close `bob-cli-a0` or any ancestor.** `bob-cli-a0.land` closes the epic.
- This plan explicitly instructs you to use `/sase_git_commit` to land each bob-cli
  change on master mid-turn. Publication builds origin/master, so the commit must be
  pushed before you publish.

## Steps

### 1. Stop the stranded waiter

1. Confirm the waiter is still `WAITING`: `sase agent list -a -j`, row
   `bob-cli-a0.13--1`.
2. Stop it with `sase agent kill -n bob-cli-a0.13--1`. If it woke up and is doing work,
   stop and report instead of racing it.
3. Re-check that the waiter is gone, that `bob-cli-a0.land` is still `WAITING`, and that
   `bob-cli-a0.13` is still `in_progress`.
4. Do not touch the stranded agent's workspace.

### 2. Salvage the a0.13 diff onto current master

1. Apply the diff: `git apply <saved main_diff.txt path above>`. It contains:
   - **llms.txt:** a `llmstxt` plugin in `mkdocs.yml` with sections Get started, Guides,
     Concepts, Reference, and "Contracts (optional)". The last one lists the 28 registry
     contracts explicitly.
   - **Reader testing page:** `docs/develop/reader-testing.md` (the six-task protocol,
     4-of-5 pass bar, door mapping, and results table), plus its Develop nav entry.
   - **Accessibility fixes:**
     - New `docs/overrides/partials/header.html` and `search.html`: visually hidden
       header labels instead of prohibited `aria-label`, and a named search dialog.
     - `docs/overrides/main.html`: a script that focuses and labels scrollable tables.
     - `bob-provenance.html`: drops its duplicate `contentinfo` role.
     - `bob.css`: Pygments contrast tokens, inline-code and task-card wrapping at 200%
       zoom, and TOC code ink.
     - `docs/index.md`: named image links.
   - **Audit matrix:** `a11y.py` gains a Guides page and a Develop page.
   - **Link gate:** `links.py` excludes llmstxt's duplicate per-directory `index.md`
     copies from the offline and online lychee runs (`(^|/)index[.]md$`), with matching
     test updates.
   - **URL contract:** `docs/_meta/url_contract.txt` gets the `bob-docs urls --refresh`
     additions.
2. Reconcile the diff with master's newer commits:
   - Build, then run `uv run --project tools/docs bob-docs urls --refresh` again. This
     adds pages that landed after the diff's base, such as
     `/guides/link-related-tasks/`. Keep only additions.
   - Re-check that the "Contracts (optional)" list still equals the stems in
     `docs/_meta/contracts.yml` (28 today).
   - Add a test to `tools/docs/tests/test_contracts.py` asserting that the llmstxt
     Contracts section lists exactly the registry stems as `<stem>.md`. A new contract
     must not silently drop out of `llms.txt`.
3. Confirm **both** builders emit the file:
   - `just docs-build` must produce `target/docs/site/llms.txt`. Zensical is the publish
     builder, so this is the critical check.
   - `just docs-build-material` must produce it too.
   - In both, check that the five sections appear and that the links resolve to
     published pages.

### 3. Close the remaining epic-owned defects

1. **Weekly Adobe 403** (epic note #5): add one commented, narrowly scoped lychee
   exclusion to `online_argv` only, per the decision below.
   - Name the vendored OFL boilerplate as the reason.
   - Update the `links.py` module and function docstrings, which currently promise no
     `--exclude`.
   - Update `tools/docs/tests/test_links_online.py` so the only permitted exclusions are
     the llmstxt `index.md` path and this one.
   - Update the weekly-check text in `docs/develop/build.md` and `tools/docs/README.md`.
   - Do not edit `OFL.txt`: it is upstream license text kept verbatim.

   > [!decision] weekly_exclusion = url Add `--exclude '^http://www[.]adobe[.]com/?$'`,
   > with a comment saying the URL is Adobe's copyright line in the vendored Source
   > Serif OFL text and returns 403 to automated clients. Other OFL links, such as
   > `scripts.sil.org/OFL`, stay checked.

   > [!decision] weekly_exclusion = file Add
   > `--exclude-path '(^|/)assets/fonts/OFL[.]txt$'`, with a comment saying the file is
   > vendored third-party license text that Bob does not author. No URL inside it is
   > fetched.

2. **Dangling PDFKit citation** (epic notes #1/#2): in `docs/highlights-create.md`, end
   the sentence at "...damages some extracted words." The finding is already summarized
   inline, so the "see ..." clause adds nothing and points at a private report.
   - Mention in the handoff that, once `bob-cli-8p`'s decision record is accepted and
     published, this sentence can link to its `/develop/decisions/#<slug>` anchor.
3. **Sparse hourly schedule** (epic note #6):
   - Re-run
     `gh api "repos/bobs-org/bobs-org.github.io/actions/runs?event=schedule" --jq '.workflow_runs[] | [.id,.created_at,.conclusion] | @tsv'`
     and record the result.
   - Keep cron `17 * * * *`. In `tools/docs/host/README.md` and `docs/develop/build.md`
     publishing text, document that:
     - GitHub runs schedules on a best-effort basis and may delay or drop them under
       load;
     - every run is idempotent behind the gates;
     - `just docs-publish` publishes current master on demand;
     - `just docs-status` shows what is live.
   - Run `just docs-host-bootstrap` (preview). If the only drift it reports is the host
     `README.md`, apply it with `just -- docs-host-bootstrap --yes`. Otherwise do not
     apply, and record the drift in the handoff.

### 4. Gate, land, publish

1. Run the gates: `just docs-check` (must be green), `just docs-build`, and
   `just docs-build-material`.
2. Run the accessibility sweep on both builders with `bob-docs a11y` (`-b` per builder;
   see `just docs-a11y`).
   - The earlier run found that Playwright's `page.screenshot` times out on this
     machine. If that recurs, run the same axe, reflow, keyboard, no-JS, and mobile
     checks with screenshots disabled through an uncommitted throwaway harness, as the
     earlier worker did.
   - Record the limitation in the handoff, and as a `PROPOSED FOLLOW-UP:` if it
     reproduces.
   - Fix any serious or critical finding before landing.
3. Land the change on master with `/sase_git_commit`, for example
   `feat(docs): add llms.txt, reader-testing protocol, and final accessibility fixes`.
   Push it.
4. Publish with `just docs-publish`, and watch the run it starts.
5. Verify the result:
   - `curl -sSf https://bobs-org.github.io/llms.txt` returns 200 with the five sections.
   - The live `/build.json` bob-cli SHA equals the commit you landed.
   - `just docs-status` shows the live site matching the latest `published.json`.
6. Run the weekly check and watch that exact run:
   `gh workflow run weekly.yml --repo bobs-org/bobs-org.github.io --ref main`.
   - A pass closes issue #2. Confirm that it is closed.
   - If it fails on anything other than Adobe, record the failure; do not add more
     exclusions.

### 5. Run the agent reader test against the live site

1. Run **ten fresh readers** through path (a) or (b) from the ground rules. With path
   (a), start them in one message so they run in parallel:
   - **S1–S5** get only `https://bobs-org.github.io/`.
   - **L1–L5** get only `https://bobs-org.github.io/llms.txt`.
2. Base each prompt on the earlier worker's wording. Each prompt must say:
   - "You are fresh independent reader <ID> in a blind findability trial."
   - Use only the given entry point. Site readers start each task from the home page and
     follow visible links or tabs. They do not use site search, and they do not type or
     guess URLs. `llms.txt` readers pick a section first, then follow its links.
   - Do not read local files, use web search, or use outside knowledge.
   - Work the six tasks one at a time, in order, with up to three minutes each:
     1. Is Bob for me?
     2. Capture your first task.
     3. Sync the vault across machines.
     4. What is a lane?
     5. Find an environment variable.
     6. Fix "bob not resolved" in Mac Capture.
   - Return one row per task: first door or section chosen, URL reached, whether they
     reached the intended information and can explain it in one line, elapsed time, and
     an observation.
   - Never mention expected doors or intended pages in the prompt.
3. Score the results against `docs/develop/reader-testing.md`:
   - Doors map as Learn → Get started, Do → Guides, Understand → Concepts, and Look up →
     Reference. `llms.txt` section titles map the same way, and choosing "Contracts
     (optional)" first counts as a miss.
   - A task passes when at least 4 of 5 readers on that entry point choose the expected
     door first.
   - If a reader fails for tooling reasons (fetch error or capacity), replace it once
     with a new fresh reader, and record both.
4. **One fix round at most.** For any failing task:
   - Change only labels or hub copy: homepage door copy, hub intros, nav titles, or
     llmstxt section names or descriptions.
   - Gate the change, land it with `/sase_git_commit`, and publish with
     `just docs-publish`.
   - Re-run that task alone with five new fresh readers on the failing entry point.
   - If it still fails, stop. Record it in the handoff and as a `PROPOSED FOLLOW-UP:`.

### 6. Write and register the handoff

1. Read the SASE artifacts reference memory
   (`sase memory read sase_artifacts.md -r "..."`).
2. Open the `bob-cli--research` sidecar with `/sase_repo`.
3. Write `202610/bob_public_docs_site/bob_public_docs_site__handoff.md` there, then
   register it with `/sase_artifact` and its matching `sase artifact create` command.
4. The handoff must contain:
   - **The seven expected items from the epic plan.** Each one says what is needed, why
     an agent could not do it, and the exact next command or step:
     1. Bryan walking `/start/` on a clean machine, including Obsidian with the demo
        vault and a Mac panel capture (epic note #3).
     2. Human reader tests with 3–5 people, using `develop/reader-testing.md`.
     3. A dated Obsidian screenshot set from the demo vault, which needs GUI Obsidian.
     4. Whether to add a custom domain. The plan authorizes none.
     5. A Zensical `0.1.x` bump. The live build uses `zensical 0.0.71`.
     6. Watching for 60-day scheduled-workflow inactivity.
     7. Unwritten guide candidates.
   - **Every `PROPOSED FOLLOW-UP:` and `DISCOVERED ISSUE:` from the epic and all its
     descendant beads**, including nested child-epic phases. Find them with
     `sase bead search "PROPOSED FOLLOW-UP" --format json`,
     `sase bead search "DISCOVERED ISSUE" --format json`, and `sase bead read` on each
     `bob-cli-a0*` bead. Mark each one resolved here (cite the commit), resolved
     earlier, or open.
   - **The reader-test results tables** for both entry points and any fix round, with
     pass or fail per task. Include a one-paragraph note on the invalid 15:15Z attempt
     and why it was rerun.
   - **The final-sweep evidence:** a11y results per builder, the `urls --refresh`
     result, live versus `published.json`, the weekly run URL and result, the issue #2
     state, and the schedule observations, with the operator-timer option left as
     Bryan's call.

### 7. Close the phase, not the epic

1. Add `PROPOSED FOLLOW-UP:` notes to `bob-cli-a0.13` for anything still open. Always
   include one for the stranded launch-gate follow-up: `bob-cli-a0.13--1` waited on its
   own finished session after the gate settled. That is a SASE runtime defect for the
   land agent to route.
2. Run `sase bead epic-symbols bob-cli-a0.13`. It must report none.
3. Close the phase:
   `sase bead close bob-cli-a0.13 --note "<what you verified: commits, llms.txt 200, reader results, a11y, weekly run, artifact ref>"`.
4. Confirm that `bob-cli-a0.land` leaves `WAITING` (`sase agent list -a -j`). It will
   triage the follow-ups and close `bob-cli-a0`. Report its state. Do not close the epic
   yourself.

## Acceptance

- `https://bobs-org.github.io/llms.txt` is served (200) from a Zensical build of the
  landed commit, and the Material build also emits it.
- `just docs-check` is green, and the llmstxt Contracts list is test-pinned to the
  registry.
- The a11y sweep shows no serious or critical findings on either builder (or records a
  screenshot-only limitation).
- A post-change weekly run passed, and issue #2 is closed (or its non-Adobe failure is
  recorded).
- The agent reader results are recorded in the handoff, and any label fixes have landed.
- The handoff artifact is registered.
- `bob-cli-a0.13` is closed with a verification note, and `bob-cli-a0.land` has started.
  `bob-cli-a0` is left to that land agent.

## Outward-facing actions this plan authorizes

- Pushing bob-cli commits to master through `/sase_git_commit`.
- Dispatching `just docs-publish` (one, plus one more for a fix round) and one weekly
  workflow run on `bobs-org/bobs-org.github.io`.
- Applying a README-only `docs-host-bootstrap --yes`.
- Writing and registering the handoff in the research sidecar.
- Stopping the stranded `bob-cli-a0.13--1` waiter.

Nothing else: no custom domain, no new secrets, and no new CI in bob-cli.
