---
tier: epic
title: Bob hand-written product reference
goal:
  Readers can look up capture syntax, vault structure, configuration, and machine
  outputs in four verified reference pages, with complete token search coverage and an
  environment-variable drift guard, completing bob-cli-a0.7.
phases:
  - id: capture-vault
    title: Capture syntax and vault format reference
    size: medium
    depends_on: []
    description:
      "capture-vault: write the capture token index and vault-format reference, add
      their navigation and token/lane search cases, and verify examples in a synthetic
      vault."
  - id: configuration-guard
    title: Configuration reference and environment drift guard
    size: medium
    depends_on: []
    description:
      "configuration-guard: document configuration keys and public environment
      variables, add a conservative production-source BOB_* inventory with reasoned
      exceptions to docs lint, and test the guard's detection and exclusions."
  - id: json-integration
    title: JSON output catalog and reference integration
    size: medium
    depends_on:
      - capture-vault
      - configuration-guard
    description:
      "json-integration: catalog every public machine-readable command output, finish
      the Reference hub and goal search cases, verify complete token coverage, and run
      the integrated documentation gates for the land agent."
proposed_by: bbugyi200.apollo.bob-cli-a0.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-a0.7
create_time: 2026-10-19 23:14:26
status: wip
---

- **PROMPT:**
  [prompts/202610/product_reference.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/product_reference.md)
- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)

# Hand-written product reference for Bob

## Outcome and ownership

Complete the remaining implementation of **bob-cli-a0.7**, the `reference-pages` phase
of `plan:202610/bob_public_docs_site.md`. This is a child epic of that assigned phase.
Its three phases each fit one coding agent; its land agent verifies and closes
**bob-cli-a0.7 only as the enclosing assigned phase**, in addition to completing the
normal lifecycle of this child epic. It must never close **bob-cli-a0** or any ancestor
above bob-cli-a0.7.

The deliverables are four hand-written pages:

- `docs/reference/capture-syntax.md`
- `docs/reference/vault-format.md`
- `docs/reference/configuration.md`
- `docs/reference/json-outputs.md`

They integrate into the existing Reference navigation and hub, with complete token
search coverage and a failing guard for undocumented production `BOB_*` names. No
product behavior or command-line interface changes are required.

## Evidence and size

The assigned bead is in progress, with no previous implementation or remaining-work
notes. Its two prerequisites are closed. The four pages do not exist in the inspected
tree; the Reference hub currently links only CLI, glossary, and decisions.
`sase bead epic-symbols bob-cli-a0.7` currently reports no entries.

The scope needs several distinct source audits:

- `docs/capture.md` contains more than 6,000 lines. Its two "Grammar at a glance" tables
  include overloaded tokens, incomplete picker states, and retired forms.
- A broad source scan finds 57 distinct `BOB_*` names, including test-only names,
  historical help strings, constant-based readers, and private process controls. A
  substring search alone cannot distinguish the public environment contract.
- Configuration is spread across `src/native/config/*.rs` and consumer parsers;
  machine-readable outputs span many command families. Version numbers must be verified
  individually: for example, freshness currently declares schema 14.

This is an epic, rather than one large tale, so each agent can verify a bounded catalog.
The last phase depends on both earlier phases to integrate their pages and tests. The
first two phases can proceed independently.

## Required context and inherited decisions

Each worker reads its assigned bead, this plan, the relevant existing contracts, and
`docs/develop/writing-docs.md`. Use audited artifact reads for the parent design and
research:

```bash
sase artifact read plan:202610/bob_public_docs_site.md "Need parent documentation requirements"
sase artifact read research:202610/bob_public_docs_site/bob_public_docs_site.md "Implementing the Bob public docs site epic"
```

If the research artifact is missing, open `bob-cli--research` with `/sase_repo` and
retry the audited read. Do not read sidecar files directly.

Use `/sase_memory_read` for `docs.md` and the glossary terms used by the assigned pages.
For behavioral distinctions, consult the relevant decision strands, especially sticky
lanes, ledger-derived Today, Depends-On links, project blocking, capture dependency
gestures, freshness, and successor offers. Describe current behavior from contracts and
code; do not turn old examples or superseded decisions into new policy.

The parent decisions are final: MIT license, search indexing enabled, design decisions
published, and only `docs.md` authorized for memory edits. **This child plan needs no
memory changes** and does not reopen any of those choices.

## Shared documentation contract

- Preserve existing contract URLs and contents. Add the four pages at the specified
  paths; do not move pages, rewrite the README, or change sibling-repo behavior.
- Use the existing reference template and portable Markdown. Each new page needs `title`
  and a description of at most 160 characters, and stays within 6,000 words. Keep the
  Reference hub within 400 words.
- Commands use `bash` fences without shell prompts. Separate output into `text` fences
  with the existing Output/no-copy convention. Use `bob-task` for task results. Examples
  must be synthetic; never consult or mutate the personal vault.
- Verify command examples using the current checkout's binary, an isolated temporary
  HOME/XDG environment, a synthetic vault, fixed `BOB_NOW` and `TZ`, and
  `BOB_REF_JOBS_KICK=off`. Network, clipboard, sync, and hook execution must be
  explicitly stubbed or avoided. Explain fixture prerequisites near examples.
- Link within bob-cli docs relatively. Link to exact contract sections rather than
  copying large specifications or regenerating CLI flags by hand.
- Each page phase adds only its own navigation entries outside generated blocks.
  Preserve other phases' entries and reconcile additive conflicts. Generated blocks stay
  owned by `just docs-gen`.
- Do not create task beads. Record out-of-scope discoveries as `PROPOSED FOLLOW-UP:`
  notes on the working child phase and summarize them on bob-cli-a0.7 for the parent
  land agent.

## Phase capture-vault

### Capture syntax

Build `reference/capture-syntax.md` from both tables under "Grammar at a glance" in
`docs/capture.md`, then check the detailed sections for constraints and examples.

At the top, place a table of **token / meaning / linked heading**. Group equivalent
spellings into useful families, but account for every distinct operation in the
contract. Each indexed token has a named heading containing its literal spelling,
syntax, one or two examples, a result illustration when it changes task text, and a link
to its authoritative contract section.

Cover routes and global destinations; URL routing and audio; templates; sections and
block IDs; linked tasks and active-task aliases; task pickers and parent selection;
prerequisites and blocked-task links; completion; adjustments and shifts; session start,
naming, and drop selections; closing, per-task outcomes, and work logs; operator chains;
project captures; task children and link toggles; Pomodoro notes; scheduling, priority,
and clipboard capture. Explain overloaded `+`, `#`, `@`, `*`, `!`, and `%` in context.
Separate incomplete picker input from executable syntax, and identify rejected/retired
spellings without presenting them as valid commands.

Add stable, descriptive anchors where punctuation would otherwise create collisions. For
each token listed in the index, add a literal query pointing at its corresponding
heading in `docs/_meta/search_smoke.yml`. Quote YAML punctuation safely; do not aim all
queries at the page root. Preserve existing search cases.

### Vault format

Write `reference/vault-format.md` using README "Vault layout", `vault/`, synthetic test
fixtures, and the contracts for capture, tasks, dependencies, projects, freshness,
dates, Today, references, and Pomodoro homes.

Include an annotated task-line anatomy: a valid `bob-task` example, numbered callouts
adjacent to it, and a legend connecting each number to the exact token/field. Keep
annotation markers out of the task's canonical Markdown. Reuse the existing renderer; no
new image assets or renderer changes are needed.

Provide lookup tables for checkbox characters and lane names, authored and derived
fields (`scheduled`, `due`, `created`, `completion`, `id`, `dependsOn`, `fresh`,
`refresh`, `keeps`, `unblocked`, and other currently supported fields), and the managed
Depends-On child line. Distinguish lane status, freshness buckets, and Today; explain
authoring ownership instead of suggesting edits to derived fields.

Describe area, project, inbox, day/ledger, and reference notes with their real
frontmatter/identity conventions and a synthetic directory tree. Distinguish default
paths from required structure and configurable locations. Add named Pending and Next
sections or headings and corresponding search cases.

### Acceptance

Both pages appear once in navigation, all indexed tokens have matching search cases,
examples match the fixture results, and `just docs-check` passes. Inspect rendered task
anatomy and token headings in the built HTML. Do not add brittle tests that pin ordinary
prose.

## Phase configuration-guard

### Configuration and environment reference

Write `reference/configuration.md`, harvesting README "Environment" and "Runtime
dependencies", then verifying against current source and contracts. Include:

- Config path precedence: `BOB_CONFIG_FILE`, then `XDG_CONFIG_HOME`, then the default
  `~/.config/bob/config.yml`; distinguish empty values and tilde handling as
  implemented.
- Tables of keys, types/allowed values, defaults, effects, and contract links for
  `properties`/priority windows, `capture`, `plan`, `dependencies`, `freshness`,
  `highlights` (including URL routing/audio/intake), `gkeep`, and `stats`. Inspect
  consumer parsers too, such as the freshness engine, rather than stopping at
  `RawConfig`. Include supported nested keys and explain rejected obsolete settings only
  where useful. Do not expose private tokens or personal configuration.
- Every public environment variable with name, default/unset behavior, effect, and link.
  Cross-check README against actual readers, including shell completion, reference
  rendering/audio, and adapters; README alone is incomplete.
- Standard variables such as XDG paths, `DATE`, `NO_COLOR`, `COLUMNS`, and relevant
  platform/runtime discovery variables, plus command-specific runtime requirements.
  Cross-link the existing contracts and generated CLI pages for exact flags.

Plugin-only configuration belongs to plugin documentation. Link a sibling reference if
available, or its public README; do not claim the Rust CLI validates every plugin
setting. Open any sibling source through `/sase_repo` before inspecting it.

### Environment drift guard

Add a focused helper, for example `tools/docs/bob_docs/environment.py`, invoked by
`lint.check_tree`. It must run through both `bob-docs lint` and `just docs-check`
without adding a new public command.

The guard scans non-test Rust source beneath the checkout's `src/`, compares every
production `BOB_*` name against documented identifiers in the configuration page, and
fails with name plus source file/line when missing. A small,
`docs/_meta/environment_ignore.yml` mapping may exempt exact names with nonempty,
specific reasons. Public runtime knobs must be documented, not hidden in that list.

Use a **conservative lexical inventory** of `BOB_*` identifiers in production Rust
string literals, including constants and help strings, rather than attempting
whole-program call resolution. This catches literals passed directly to `std::env`,
`bob_env` wrappers, constants, and names passed through helpers such as `env_name`. Any
extra historical/help-only names need explicit reasons. Document the conservative scope
in `tools/docs/README.md`.

Exclude comments and test-only items/modules and test source files, while retaining
production code after an inline `#[cfg(test)]` item. In this tree
`src/native/config/mod.rs` has a test-only method before much of its production
implementation, so truncating at the first `#[cfg(test)]` is wrong. Respect Rust
string/raw-string/comment boundaries when counting item braces. Do not exclude a
production reader merely because its variable has `TEST` in its name.

Read documentation from the assembled docs context when lint is assembled, and source
from its repository root. Count complete documented identifiers in actual Markdown
content, not comments or substring matches. Fail on malformed or duplicate ignore
entries, blank reasons, and unknown/stale names. A missing configuration page must fail
when source variables need coverage. Preserve small test fixtures with no Rust
environment names.

### Acceptance

Focused pytest fixtures prove: an added undocumented literal or constant fails with
provenance; exact documentation or a reasoned exemption passes; comments, inline test
items, and test modules do not become public obligations; later production code is still
scanned; raw strings and braces do not corrupt scope; invalid/stale exceptions fail; and
the check is invoked by normal lint, including assembled context. Keep tests about these
observable failure modes, not implementation details.

Run the focused tests, all `tools/docs/tests`, and `just docs-check`. Add the page to
Reference navigation. No Rust code changes or broad compiler checks are required.

## Phase json-integration

### JSON outputs

Write `reference/json-outputs.md` as a command-level lookup: command/subcommand,
enabling flag or default mode, output shape, actual schema version (or explicitly
unversioned), and contract link. Split rows whenever flags, envelopes, or versions
differ. Explain success/error framing and stdout/stderr only as far as the contracts
actually promise.

Audit `src/main.rs`/command registration, generated CLI pages, serialization code, and
tests. Include capture and all public `capture-*` protocol helpers, saved templates,
plan/ready/freshness/stats, task/project/maintenance outputs, query engines, articles,
plugins, vault-sync, completion status, gkeep, ref operations and ref jobs where they
expose machine-readable output. Public ref subcommands need direct source inspection; do
not infer coverage from a search for `json` in generated pages alone. Inventory other
public structured formats, such as CSV/TSV where supported, in a clearly labeled
companion section. Hidden internal adapters and on-disk state files are not public
command outputs.

Do not assume all JSON uses schema 1, copy stored-file versions onto command output, or
add version fields to make the table uniform. Use existing output fixtures/tests and
safe invocations in the isolated environment to verify representative envelopes from
each family. Do not run live migrations, sync, login, or reference network jobs.

### Integration and search

Update `reference/index.md` so readers can choose among the four new references,
generated CLI, glossary, decisions, and any sibling references that actually exist.
Replace its claim that all reference material is generated.

Add the JSON navigation entry and reconcile the earlier phases' additive nav changes.
Add goal smoke cases for **complete a task**, **hotkey conflict**, and **install
plugins**, preserving the token and lane cases.

- Completion should resolve to the capture completion section.
- Plugin installation should resolve to meaningful existing plugin instructions or a
  linked, concise configuration/runtime subsection.
- A Mac hotkey conflict should resolve to a relevant mounted reference/guide if one has
  landed. Otherwise add a short discoverability subsection on the configuration page
  that directs readers to the actual Mac Capture keyboard/settings instructions in its
  public README. Inspect those instructions via `/sase_repo` before writing the summary.
  Do not create a broken link to a future guide or invent troubleshooting.

Check every literal token in the capture index against a smoke case with a matching
heading location. A small structural check in docs-tool pytest may enforce this
cross-file coverage; it must not freeze prose or require a new token registry. Preserve
the existing search matching semantics, especially punctuation-only queries requiring a
literal heading match. The smoke tests verify indexed discoverability, not browser
search ranking.

### Acceptance

Run focused docs tests and `just docs-check`, followed by `just docs-build-material` to
prove the new markup/anchors work in the fallback. Verify the four page URLs, rendered
anatomy, named token anchors, all token/lane/goal search cases, and the environment
guard. Record commands and results on the child phase and bob-cli-a0.7. Regenerate only
when a source-owned generated block needs it; do not mask generator drift with hand
edits.

## Land and close

The child epic's land agent checks that all three child phases completed and their
changes landed together, reviews the four pages against the parent phase checklist, and
verifies the integrated docs gate. Follow the SASE finalization workflow; manual git
commits are outside this plan.

Before closing any assigned child phase, run its `sase bead epic-symbols` check. Before
closing the enclosing assigned phase, run:

```bash
sase bead epic-symbols bob-cli-a0.7
```

Resolve its remaining symbols or re-key their Justfile lines to a still-open owning
bead. Close this child epic through its normal land lifecycle before closing its
enclosing phase, so the descendant guard is satisfied. Then:

```bash
sase bead close bob-cli-a0.7 --note "<pages, token coverage, environment guard, and checks actually verified>"
```

Do not set status by hand, reopen already completed work, or close bob-cli-a0 or an
ancestor above the assigned phase. If work has landed while waiting, verify and reuse it
instead of duplicating it.

A check failure reproduced identically on the clean base tree is recorded as
`PROPOSED FOLLOW-UP:` on bob-cli-a0.7, citing an existing task if one is already known;
it does not keep this phase open. Changes introduced here must pass their relevant
checks. Use `/sase_monitor` for commands too long to finish in the provider turn.
