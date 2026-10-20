---
tier: tale
title: Goal-titled guides and component browsing
goal:
  Publish twelve verified how-to guides with component chips, a generated index, and
  connected reader paths, then close bob-cli-a0.8.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.8
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.8
create_time: 2026-10-20 01:50:00
status: wip
---

- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:**
  [bob-cli-a0.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.8.md)

# Goal-titled guides and component browsing

Implement the guides phase of the Bob public documentation epic, assigned as
`bob-cli-a0.8`. Readers should be able to find a short procedure by their goal or by the
component they use, then follow links to exact reference and contract details.

## Scope and evidence

This is one bounded documentation and Python-tooling change for one coding agent. There
are no previous implementation notes on the assigned phase. Its dependencies are closed,
the working tree was clean at investigation, and the checkout at `50d0e990` has the
tutorial, product reference, mounted plugin reference, and docs gates. It has no
`docs/guides/` directory, component index, or tag override. Recheck the current tree
before implementing so concurrent epic work is preserved.

Read the phase with
`sase bead read bob-cli-a0.8 -r "Implement the approved guides plan"`. The authoritative
epic is `plan:202610/bob_public_docs_site.md`; read it with `sase artifact read`. Its
required research is `research:202610/bob_public_docs_site/bob_public_docs_site.md`,
also read through `sase artifact read`. Opening `bob-cli--research` with `/sase_repo`
resolved an initial missing-artifact result during investigation.

The epic's decisions are settled: MIT license, search indexing enabled, published design
decisions, and memory authorization only for `docs.md`. This plan requires no memory
edits, license changes, deployment changes, or new reviewer decisions. Keep the contract
files at their existing URLs. The separate README phase owns removing material from the
README; this phase harvests it without deleting it. The concepts phase owns concept
prose, and the Mac reference phase owns Mac troubleshooting and reference expansion.

Read `docs.md` using `/sase_memory_read`, then `docs/develop/writing-docs.md`,
`docs/_templates/how-to.md`, and the local source pages below. Read the relevant
glossary and decision strands through the audited memory command as needed. Use
`/sase_repo` before consulting either linked source repository; edit no mounted copies.
No Rust, JavaScript plugin, or Swift behavior changes are needed.

## 1. Write twelve focused how-to guides

Create the following files directly under `docs/guides/`. The filename is the published
URL; navigation grouping must not move it into a group subdirectory. Use these goal
titles and the listed navigation groups. Tags describe the surfaces the procedure
actually uses; refine the initial assignments if the verified steps warrant it.

| File                             | Title and nav group                                           | Starting tags        | Sources and outcome                                                                                                                                                                                                                                                                                                                |
| -------------------------------- | ------------------------------------------------------------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `install-everything.md`          | Install everything with one command — Set up                  | CLI, Obsidian, macOS | README “Install everything”, `scripts/install_all`, installation and compatibility pages. Establish the checkout/vault/tool prerequisites, run `just install-all` from bob-cli, inspect its per-component results, and explain platform and missing-sibling skips. Link to signing details instead of copying the signing runbook. |
| `point-bob-at-your-vault.md`     | Point bob at your vault — Set up                              | CLI                  | Configuration reference, getting-started contract, CLI help. Show an explicit vault path with a command that supports `-b`, then `BOB_DIR` for the shell; verify a read-only result. Do not imply every subcommand supports the same short option.                                                                                 |
| `deploy-obsidian-plugins.md`     | Deploy the Obsidian plugins — Set up                          | CLI, Obsidian        | `plugins.md`, generated CLI help, mounted plugin reference. List, preview sync, deploy, and verify/reload; explain the source-repository and dirty-vault safeguards without suggesting routine force overrides.                                                                                                                    |
| `capture-from-your-mac.md`       | Capture a task from anywhere on your Mac — Capture            | macOS, CLI           | `start/mac-capture.md`, current Mac Capture README/docs, capture reference. With the app installed, verify its Bob/vault settings, invoke the configured global shortcut, inspect the preview, submit, and find the task. Verify exact UI names and shortcut behavior in the source repo.                                          |
| `drain-google-keep.md`           | Drain Google Keep into tasks — Capture                        | CLI                  | `gkeep.md`, README Gkeep, CLI help. Check adapter/auth prerequisites, inspect the queue, preview and pull using supported flags, and confirm the imported task and remote-note disposition. Explain credentials and deletion effects from current behavior.                                                                        |
| `run-morning-review.md`          | Run the morning review walk — Plan the day                    | CLI, Obsidian        | `freshness.md`, `dashboard.md`, current plugin navigation reference, README freshness. Inspect the queue, enter the walk, make explicit keep/resolve choices, and observe progress. Link to detailed tier and gesture rules rather than reproducing the whole state machine.                                                       |
| `keep-ready-under-cap.md`        | Keep Ready lanes under their cap — Plan the day               | CLI, Obsidian        | `plan.md`, `ready` CLI reference, configuration and vault-format reference. Inspect per-note counts, review over-cap tasks, make deliberate workflow choices, then check counts; explain `ready_cap` overrides and the distinction from dashboard READY.                                                                           |
| `add-prerequisites.md`           | Add prerequisites while capturing — Projects and dependencies | CLI, macOS           | `capture.md`, `task-dependencies.md`, capture-syntax reference. Use a synthetic prerequisite with a stable block id, preview and submit a dependent capture, and confirm its Depends-On child line and blocked state. Distinguish `&` and `*` directions briefly and link their exact rules.                                       |
| `save-link-to-reading-queue.md`  | Save a link to your reading queue — Reading                   | CLI, macOS           | Capture bare-URL reference, `ref-jobs.md`, `ref.md`. Queue a supported public URL, inspect job completion and the reading queue, and explain queued versus completed success. Link URL exclusions and immediate-import alternatives.                                                                                               |
| `sync-vault-across-machines.md`  | Sync the vault across machines — Set up                       | CLI, Obsidian        | `vault-git-sync.md`, CLI help. Assume the reader has their own Git remote and credentials; verify checkout/branch prerequisites, preview/run a cycle, inspect status, and repeat from the other machine. Explain one active sync engine and excluded content.                                                                      |
| `resolve-vault-conflict-copy.md` | Resolve a vault-sync conflict copy — Fix problems             | CLI, Obsidian        | `vault-git-sync.md` conflict policy. Inspect status and the conflict log, compare the saved local copy with the current note, retain wanted edits, verify, and sync. Keep halted merges distinct and link their runbook; never prescribe reset-hard or force-push.                                                                 |
| `recover-reference-job.md`       | Recover a stuck reference job — Fix problems                  | CLI                  | `ref-jobs.md`, reference doctor and jobs help. Inspect job state/logs, diagnose the failure, invoke `bob ref jobs run`, and confirm recovery. A stuck job retries fallback writing without clipping again; do not invent retry/delete subcommands.                                                                                 |

Each page must have:

- `title`, a one-line description of at most 160 characters, and `tags` front matter
  using only `CLI`, `Obsidian`, and `macOS`.
- A brief outcome, a “Before you start” section, ordered steps with an observable
  confirmation for **each** step, exactly one “If it goes wrong” section, and “Related”
  links.
- Short copyable examples, synthetic names, no shell `$` prompts, and separate
  non-copyable output fences when output is useful. Link existing references for
  exhaustive flags, grammar, schemas, and settings.
- A target below 1,500 words and a hard ceiling of 2,500; aim substantially shorter
  where possible. Reader pages must satisfy hygiene and privacy checks.

Translate personal runbook assumptions into reader prerequisites. Do not copy host
names, private remotes, home paths, key names, service labels, or personal account
details into the sync guide. Do not present personal background services as shipped Bob
setup. Show the supported manual sync workflow and link exact behavior.

Source reconciliation matters: the current `freshness.md` contract describes schema 14
with recurring review and no separate reference tier, while the glossary memory still
names a REFERENCES tier. Check current implementation and plugin reference for guide
steps, keep the guide focused on gestures and visible progress, and record
contradictions for the epic land agent. Do not modify decision records or glossary
memory to resolve them in this phase.

## 2. Generate the component index and render text chips

Implement a small `tools/docs/bob_docs/gen_components.py` module using the existing safe
YAML/front-matter utilities. Have it discover authored Markdown under `docs/guides/` and
`docs/concepts/`, so later concept additions automatically join the index without a
hardcoded page list.

Define the component names and anchors once in the generator: `CLI` → `cli`, `Obsidian`
→ `obsidian`, `macOS` → `macos`. Require non-hub guide and concept pages to declare a
nonempty list drawn from that vocabulary. Hubs may omit tags and are excluded from the
index. Fail with a source path for malformed metadata or unknown components; do not
silently drop misspelled tags. De-duplicate membership so a page appears at most once
per group. Do not index templates, references, mounted pages, or the generated index
itself.

Generate `docs/reference/by-component.md` as a complete generated page with normal
title/description front matter and the standard
`<!-- generated by bob-docs; run just docs-gen -->` marker. Give each of the three
sections an explicit stable anchor, and list links using page titles and paths relative
to the generated page. Sort deterministically by title, then path. Pages with multiple
components appear in each relevant group. Finish with related links to the guide and
concept hubs.

Integrate it into `gen_contracts.check_or_generate`, which already coordinates contract,
memory-reference, and plugin-inventory generation. Preserve its drift return contract:
`gen --check` reports a missing/stale index and exits nonzero without writing; normal
generation writes it; the next check succeeds. This automatically includes the index in
`just docs-gen` and the existing generated drift gate in `just docs-check`. Update the
`gen` help summary and the short generation description in the docs authoring/build
instructions as appropriate. No new CLI option or generic guide-execution framework is
necessary.

Tag the existing non-hub concept page `how-bob-fits-together.md` with its three
components. If the concepts phase has landed additional pages, tag those according to
their actual scope; otherwise keep discovery ready for future pages. Add a brief
component-tag convention to `docs/develop/writing-docs.md` and the how-to and concept
templates so future additions know the vocabulary and hub exception.

Enable the allowlisted `tags` plugin in `mkdocs.yml`. Add
`docs/overrides/partials/tags.html` to render text chips from page metadata with links
to `/reference/by-component/#cli`, `#obsidian`, and `#macos`. Resolve links through the
builders' URL handling so nested pages and a configured URL prefix work. Ensure the
partial renders once in both pinned Zensical and Material builders; if their inclusion
paths differ, adjust the existing override minimally. Use compatible MiniJinja/Jinja
syntax, escaped text, and no JavaScript dependency. Use existing teal tokens and visible
focus styling if custom CSS is needed.

## 3. Connect the reader paths

Add a Guides tab with an Overview entry and the six populated groups in this order: Set
up, Capture, Plan the day, Projects and dependencies, Reading, Fix problems. Each guide
appears once in nav. Create `docs/guides/index.md` with one sentence of orientation and
compact goal links under those same groups. Keep the hub within 400 words. Link the
component index from the hub, the Reference hub, and Reference navigation.

Add the homepage **Do** door pointing to the Guides hub with useful example goals.
Preserve unrelated homepage work and keep the homepage within 400 words. Do not rename
or move existing pages to achieve the grouping.

Set `reader:` in `docs/_meta/contracts.yml` where these guides provide a direct reader
path. Suitable mappings include `plugins` → deploy guide, `gkeep` → Keep guide,
`freshness` → morning review, `plan` → Ready-cap guide, `task-dependencies` →
prerequisites guide, `ref-jobs` → recovery guide, `ref` → reading-queue guide, and
`vault-git-sync` → sync guide. Choose the most representative guide for `capture`;
preserve any more useful existing mapping. The registry supports one reader path per
contract, so use guide Related links for additional goals. Do not add registry entries
for typed pages.

Extend `docs/_meta/search_smoke.yml` with representative new goals such as “sync the
vault across machines”, “resolve a vault-sync conflict copy”, “add prerequisites”, and
“recover a stuck reference job”, pointing to the guide URLs. Preserve existing contract
search checks.

Cross-repository detail links should target already available source or mounted pages.
Until the Mac reference mount exists, link the existing Mac continuation page and the
relevant public repository README anchors. Do not introduce broken future URLs or create
placeholder Mac reference pages.

## 4. Verification

Test the generator with small temporary document trees, not assertions on live guide
prose. Cover component membership, deterministic ordering, duplicate tags, hub
exclusion, bad/missing tags, valid relative links and explicit anchors, and non-writing
drift detection followed by successful generation. Extend the existing generation-gate
fixture tests as needed.

Extend the existing dual-builder template tests with a synthetic tagged page and
component-index fixture. Verify chips exist exactly once, retain readable text, link to
the correct built anchors from a nested page, and disappear on an untagged page. Test
both builders, including a non-root URL prefix where supported. Tests may assert
generated metadata and link structure; do not pin editorial guide sentences.

Verify commands against a freshly built Bob from the current implementation checkout,
recording its SHA. Use the existing sandbox helpers and copied demo vault for CLI
examples, with isolated home/config/state/cache/spool directories; never use the real
vault or credentials. Use synthetic local Git repositories and a local bare remote for
sync/conflict examples. Use adapter/job fixtures for Keep and reference-job behavior, so
verification neither drains an account nor starts real background jobs against user
data.

Record what ran and what could only be inspected: Mac GUI actions, real account
authentication, signing, and a host-wide install cannot be claimed as executed on a
Linux sandbox. Validate their command syntax/current source and provide explicit manual
confirmation steps. Existing install-script tests or a stubbed disposable checkout can
verify orchestration without replacing user installs. There is no need to add a
permanent test runner for ordinary guide prose.

Run these checks after implementing:

1. Focused component-generator and template tests, with the Material dependency group
   enabled so the fallback tests execute.
2. `just docs-gen`, then `uv run --project tools/docs bob-docs gen --check`.
3. `just docs-check`, covering lint, budgets, generated drift, strict Zensical, URLs,
   offline fragments, search smoke, tutorial/media, and docs tests.
4. `just docs-build-material` to verify the real site and chips under the fallback
   builder; inspect a nested guide and the component page in both themes.
5. `git diff --check` and a final scope/diff review. Run `just check` if any Rust
   changes became necessary; otherwise the documentation gates are the relevant suite.

Open required mounted repositories through `/sase_repo` before the docs tools need them;
point the documented source-directory overrides at those checkouts. Use `/sase_monitor`
before starting long checks that need a handoff. A reproduced clean-base failure is
recorded with exact evidence and an existing tracking bead if known; it does not keep
the assigned phase open. Fix regressions introduced by this work before closure.

## Completion

The phase is complete when all twelve guides are present and navigable, guide and
concept component chips reach a generated deterministic index, contract reader links and
the homepage Do door work, and the relevant gates pass or have the user-authorized
clean-base exception recorded.

Record any discovered out-of-scope work only with
`sase bead note bob-cli-a0.8 'PROPOSED FOLLOW-UP: <summary — evidence and detail>'`. Do
not create task beads. Before closing, run `sase bead epic-symbols bob-cli-a0.8`;
resolve each remaining symbol or re-key its Justfile entry to a still-open parent/later
phase. Then close only the assigned phase with
`sase bead close bob-cli-a0.8 --note "<guides, tags, command verification, and gate results>"`.
Do not close `bob-cli-a0` or any ancestor plan bead, and do not hand-edit bead status.
Finish the implementation turn with `/sase_final`.
