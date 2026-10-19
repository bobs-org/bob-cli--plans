---
tier: epic
title: Complete the Bob documentation toolchain and contract site
goal:
  Finish bob-cli-a0.2 with an in-place contract site, reproducible assembly, executable
  documentation gates, and the authorized authoring guidance.
parent_bead: bob-cli-a0.2
phases:
  - id: foundation
    title: Build the docs package and contract site foundation
    size: medium
    depends_on: []
    description:
      "foundation: create the locked uv package, tagged-YAML/config support, strict
      builders, contract registry and generators, minimal Home/Develop pages, and
      build/gen/serve recipes."
  - id: mounts
    title: Assemble mounted sources with verified provenance
    size: medium
    depends_on:
      - foundation
    description:
      "mounts: implement sibling discovery, safe copying, mount nav expansion,
      provenance, and explicit clean-SHA production assembly with fixture-repository
      tests."
  - id: lint
    title: Enforce Bob's documentation contract
    size: medium
    depends_on:
      - mounts
    description:
      "lint: port the editorial checks and add root registration, nav completeness,
      portability, media provenance, privacy, and budget ratchets with negative fixtures
      for every rule."
  - id: url-links
    title: Protect published URLs and verify built links
    size: medium
    depends_on:
      - mounts
    description:
      "url-links: implement built-site URL and anchor resolution, live and curated URL
      contracts, source mention scanning, and checksum-verified lychee acquisition and
      execution."
  - id: authoring
    title: Document authoring and add the approved memory note
    size: small
    depends_on:
      - lint
    description:
      "authoring: finish Writing docs and Build each repo, port five authoring
      templates, and create only docs.md under the inherited accepted memory decision,
      then republish memory."
  - id: integration
    title: Wire search and prove the complete docs gate
    size: medium
    depends_on:
      - lint
      - url-links
      - authoring
    description:
      "integration: add search smoke, compose docs-check including production mode,
      prove strict and fallback builds and injected-failure diagnostics, and leave
      complete evidence for the land agent."
proposed_by: bbugyi200.apollo.bob-cli-a0.2
proposed_with: codex/gpt-6-astra
create_time: 2026-10-19 19:05:36
status: wip
---

- **PROMPT:**
  [prompts/202610/docs_toolchain.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/docs_toolchain.md)
- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)

# Complete the Bob documentation toolchain and contract site

This child epic implements the **toolchain** phase of
`plan:202610/bob_public_docs_site.md`, assigned as `bob-cli-a0.2`. It leaves a complete
small site with Home and Develop, plus the machinery the outer epic's theme,
generated-reference, and publish phases need.

The work spans independent build, filesystem, validation, and link-checking systems. Six
direct implementation phases keep each coding assignment bounded. The phase notes were
empty at planning time; this is a decomposition of the original scope. The graph is
foundation → mounts → {lint, url-links}; lint → authoring; {lint, url-links, authoring}
→ integration. Only lint and url-links run in parallel; their implementations and test
files are separate.

## Authority, evidence, and scope

Before implementing, read the assigned bead and these audited artifacts:

```bash
sase bead read bob-cli-a0.2 -r "Need toolchain scope and inherited decisions"
sase artifact read plan:202610/bob_public_docs_site.md "Implement the toolchain phase of the public docs epic"
sase artifact read research:202610/bob_public_docs_site/bob_public_docs_site.md "Need the public docs research underpinning the approved design"
```

The parent plan wins over the research. Its accepted choices are final: MIT, search
indexing enabled, public design decisions enabled, and `memory_docs_note = yes`,
authorizing **`sase/memory/docs.md` only**. The memory authorization is inherited
through `parent_bead`; it is not a new reviewer choice. Use `/sase_memory_write` before
creating that note and `sase memory init` afterward.

Verified on bob-cli `8b0f3782`:

- There is no `tools/` docs project, `mkdocs.yml`, or `docs/_meta/` yet.
- The actual recipe file is lowercase `justfile`. Its `check` recipe runs Rust
  formatting, Clippy, and all tests; retain it unchanged.
- `docs/README.md` lists **27 contracts**, including the newly landed `task-move.md`.
  The parent research predates that addition and says 26. Register every current
  contract, preserve its path and prose, and derive displayed counts from the registry.
- The five escaping links match the parent design: README links in `getting-started.md`,
  `stats.md`, and `capture.md`, and fixture links in `stats.md` and `pomodoro-homes.md`.
  The excluded `docs/README.md` has another README link which remains relative.
- `src/native/vault_sync.rs` prints `docs/vault-git-sync.md#halted-sync`; it must
  resolve after building.
- Existing `++`/`!!!` strings in `capture.md` are literal grammar examples; `$`
  transcripts in `completion.md` are in `text` fences. Example image syntax in
  `highlights-ref-sync.md` is inside code. Do not flag these as rendered markup.

The SASE source was opened through `/sase_repo`; the useful implementation is the
749-line executable **`tools/check_docs`**, not a package directory. Also read
`tests/test_check_docs_tool.py`, `docs/contribute/writing-docs.md`, and the five
`docs/_templates/*.md` files. Each worker needing that repo must open `gh:sase-org/sase`
with `/sase_repo` and use the returned path. Port the relevant rules and template
shapes, not SASE's hosting, blog, or PDF stack. Retain applicable source-license
attribution for copied code.

Current upstream evidence:
[PyPI lists Zensical 0.0.71](https://pypi.org/project/zensical/) at planning time.
Recheck when foundation starts, then pin the newest compatible release exactly and
commit the lockfile. The parent requires a Material 9.7 fallback.
[Zensical configuration](https://zensical.org/docs/setup/basics/) and
[lychee local link checking](https://lychee.cli.rs/recipes/local-folder/) are the
primary documentation for path and local URL behavior.

The outer epic retains visual identity, fonts and a11y implementation, generated CLI
reference/glossary/decisions, real sibling reference pages, deployment, tutorials,
guides, and README restructuring. This child epic supplies their extension points. It
adds no Actions workflows or publishing side effects.

## Shared implementation contracts

### Files, outputs, and interfaces

`tools/docs/` is a Python ≥3.11 uv project named `bob-docs`, with package `bob_docs`, an
argparse console script `bob-docs`, and pytest tests under `tools/docs/tests/`. Run
commands from the bob-cli checkout; use explicit root/context objects internally so
tests can use fixture repositories. Commands raise/report actionable errors and return
nonzero; do not turn malformed input into an empty successful configuration.

Use a small common context containing repo root, docs root, assembled config, output
directory, and resolved sources. Keep command registration in one CLI module, with
separate implementation modules (`assemble`, `gen_contracts`, `lint`, `urls`, `links`,
`search_smoke`, `check`). Shared helpers cover tagged YAML, generated blocks, Markdown
parsing, and source inventory. Avoid a generic plugin framework.

Only `gen` and explicitly requested lint ratchet updates modify tracked source. Assembly
may replace its own **gitignored** mount copies. All other outputs are under
`target/docs/`: `mkdocs.yml`, `site/`, `build.json`, and gate reports. Building and
checking never regenerate tracked pages or refresh baselines silently. Ignore the local
uv environment and the four mount destinations:

```text
/docs/reference/plugins/
/docs/reference/mac-capture/
/docs/develop/plugins/
/docs/develop/mac-capture/
```

CLI help follows `cli_rules.md`: alphabetized options and subcommands, clear help, and
short aliases for public long options. Suggested scoped aliases are `-c/--check`,
`-f/--fix`, `-m/--material`, `-p/--production`, `-s/--sources`, `-r/--refresh`; each
belongs only on the commands that need it. No Rust CLI subcommand is added.

### Registry and generated blocks

Document and test these small metadata schemas at their owners:

- `contracts.yml`: ordered groups containing entries with `stem`, `summary`, optional
  `reader`, and optional `historical: true`. Seed summaries verbatim from the current
  README table; mark `obsidian-sync-exclusions` historical. Preserve the original table
  order independently of grouped nav if necessary, using an explicit order field. Reject
  duplicate/missing stems and unsafe paths.
- `mounts.yml`: an initially empty list; entries contain `repo`, `remote`, `branch`,
  `source`, `target`, `nav_title`. Targets are relative to `docs/`.
- `budgets.yml`: `ceilings: {relative-page.md: integer}`; start empty.
- `search_smoke.yml`: a nonempty list of `{query, location}` cases.
- `media.yml`: entries keyed by asset path, with provenance/source and ISO date;
  initially empty because this phase creates no images.
- The URL ignore file contains narrowly scoped source/mention matches with a reason, not
  broad path or domain exclusions. The curated URL file contains absolute site URLs,
  with optional fragments and comment lines.

Generated content uses exact `BEGIN GENERATED: name` / `END GENERATED: name` comments
(HTML in Markdown; `#` in YAML). Whole generated pages have
`<!-- generated by <tool>; run just docs-gen -->` as their first body line. Generation
is deterministic, validates marker pairing, preserves unrelated bytes, and supports
`--check` with file-specific drift diagnostics and no writes.

## Phase foundation

Implement the package, the empty-mount build path, and the initial useful site.

1. Pin Zensical, with `markdown-callouts`, `pyyaml`, and `pytest`. Add optional groups
   `material` (`mkdocs-material==9.7.*`, `mkdocs-redirects`, `mkdocs-llmstxt`), `fonts`
   (`fonttools`, `brotli`), and `a11y` (Playwright). Axe-core is JavaScript: reserve its
   actual pinned asset/runtime integration for the outer theme phase rather than adding
   an unrelated Python package with a similar name. Commit `uv.lock`.
2. Create `mkdocs.yml` exactly within the parent compatibility subset: Bob site name,
   org-root URL, bob-cli repo/edit links, strict mode, validation keys at `warn`,
   directory URLs, material theme, `variant: classic`, `font: false`, and an existing
   `docs/overrides` directory. Exclude `README.md`, `_meta/`, `_templates/`, and
   `overrides/`. Include the parent's navigation, code-copy/edit, search, and toc
   features; enable `search` and empty `redirects`; use the listed standard/PyMdown
   extensions plus `github-callouts`, with the Mermaid superfences formatter. Add only
   Home and Develop to nav. Keep generated nav markers in their own region.
3. Handle `!!python/name:` safely as inert YAML nodes while assembling; never execute
   arbitrary constructors. Preserve tags when re-emitting the config, including the
   Mermaid formatter and a future `bob_docs.fences` formatter. Resolve path settings
   from the authored config's location, then rebase them to `target/docs/mkdocs.yml` so
   both builders find `docs/`, overrides, assets, and `target/docs/site/`.
4. Add the full current registry, empty mounts, and `gen_contracts`. Render the
   generated `develop/contracts.md`, grouped `contracts nav` block, and marked table in
   `docs/README.md`. The README table's content stays byte-equivalent. Convert only the
   five escaping contract links to absolute `github.com/bobs-org/bob-cli/blob/master/…`
   URLs, preserving fragments.
5. Add an honest minimal `docs/index.md`, `develop/index.md`, and `develop/build.md`:
   explain the currently available contracts and locally usable build commands. Add
   front matter and live links. The authoring phase expands build guidance and adds
   Writing docs. No empty tabs, dead links, or placeholder reader doors.
6. Implement `assemble`, `build`, `serve`, and `gen`. At this stage `assemble` supports
   the empty table and fails clearly if nonempty mounts are supplied before the mounts
   implementation. Write basic bob-cli build provenance and inject `extra.build` and
   `extra.contracts`; keep that shape extensible. A successful build copies `build.json`
   into the site output. `serve` assembles then invokes Zensical serve; `build` invokes
   strict Zensical or strict Material on the same assembled config, propagating exit
   codes.
7. Add themed recipes `docs-serve`, `docs-build`, `docs-build-material`, `docs-gen` with
   `uv run --project tools/docs …`; the Material recipe selects its dependency group.
   Add `docs-check` only when integration can make it real. Later generators and CLI
   goldens are added by their owning outer phases.

Acceptance: locked package installs; CLI help is readable; generator drift tests pass;
each registered contract appears exactly once in nav; strict Zensical and Material build
this slice; tags survive assembly; build leaves tracked files unchanged. Snapshot the
original README table in a focused test, not guide prose.

## Phase mounts

Complete `assemble.py` and source discovery. Keep the real mount table empty.

1. Resolve each unique repo once, reusing it for several mounts. Local discovery order:
   `BOB_DOCS_<REPO_WITH_UNDERSCORES>_DIR`, then `../<repo>`, then
   `sase/repos/linked/<repo>`, then a shallow cache clone under
   `~/.cache/bob-docs/<repo>`. An explicitly supplied invalid override fails; it does
   not silently fall through. Report the attempted locations on failure. Test network
   behavior with local fixture remotes or mocks. Agents open real repos with
   `/sase_repo`; these fixtures do not need real linked repos.
2. Validate all mounts before writing: target beneath docs, source beneath its owning
   repo, target gitignored and untracked, no overlapping targets, source escapes,
   escaping symlinks, or destination symlink escape. Never overwrite authored files.
   Allow repeat assembly of previously owned mount outputs, remove stale copied pages,
   and fail without partial replacement if validation fails.
3. Copy permitted source files without link rewriting. Add `source: <repo>:<path>` to
   copied Markdown front matter while preserving the body. Reject authored source
   spoofing. Treat `nav.yml` and `media.yml` as support metadata, not public pages. For
   multiple mounts whose source directories nest, partition copying by the most specific
   source root so `docs/develop` is not duplicated under both reference and develop.
   Document that links crossing mount boundaries need absolute site URLs; validate them
   through the URL gate. Fixture-test that partition explicitly.
4. Expand each committed `<target>/index.md` nav placeholder using its source `nav.yml`,
   whose page paths are relative to that source root. Preserve titles and order, qualify
   paths, reject escaping/missing/duplicate entries and unresolved placeholders, and
   include the index exactly once. The lint later checks the full assembled nav against
   published Markdown.
5. Define production sources JSON as
   `{"sources": {"bob-cli": {"path": "…", "sha": "…"}, "bob-plugins": …}}`.
   `assemble/build/check --production --sources FILE` requires explicit entries for
   bob-cli and each mounted repo. Check exact HEAD and clean tracked/untracked state
   (ignored build outputs do not count); reject missing repos and discovery env
   overrides. Production never discovers, fetches, or changes these inputs. Resolve
   relative paths against the JSON file; verify the supplied bob-cli path is the actual
   build root. Communicate this schema in `tools/docs/README.md` for publish.
6. Write repo-keyed SHA, dirty flag, branch, UTC build time, and installed tool versions
   to `target/docs/build.json`; inject it and the registry into the assembled config.
   Serve the same manifest at `/build.json`. Build time is allowed to change; generated
   tracked content is deterministic. Confirm final clean source state after copying
   ignored mounts as well as before assembly.

Acceptance: pytest covers discovery precedence, missing/invalid siblings, repeated
mounts, nested sources, stable reruns/stale removal, all collision/escape cases,
provenance and nav expansion, YAML tags, missing/wrong SHA/dirty production inputs, and
rejection of env overrides. Both real empty-mount builds still pass.

## Phase lint

Port the useful upstream `check_docs` routines into Bob's package, with accurate
`path:line: rule: explanation` diagnostics. Preserve source line positions when masking
comments or code. Add a pure `lint` command and explicit `--fix` ratchet mode; normal
lint and `check` never fix.

Enforce all these rule families with positive and negative fixtures:

- **Typing and registry:** only root `index.md` and registered contracts are content at
  root; typed dirs are start/guides/concepts/reference/develop; support paths are
  excluded. Every registry page exists; no duplicate stem, typed-dir collision, or
  mount-root collision. Validate mount targets are gitignored.
- **Front matter:** typed pages need string title and a one-line description ≤160
  characters. Allowed keys are title, description, hide, search, tags, template, icon,
  status. `source` is legal only on verified assembled copies. Contracts do not need
  front matter. Generated pages remain subject to metadata, links, and privacy checks;
  their exemption is from word budgets.
- **Budgets:** count outside front matter, fenced code, comments, and generated blocks.
  Start ≤2,500/page and ≤10,000 total; guides warn at 1,500 and fail above 2,500;
  concepts ≤3,000; homepage and all hubs ≤400; handwritten reference, including mounts,
  ≤6,000; develop >8,000 warns. Contracts are exempt. Page ceilings can only ratchet
  down; `--fix` may lower/prune existing entries, never inflate limits. Do not import
  SASE's concept-total cap or blog rules.
- **Hygiene:** start/guides/concepts forbid Bob bead-ID shapes, src paths, epic/phase
  labels, and formerly/no longer/legacy/renamed. Reference forbids bead IDs. Skip code
  and link destinations for hygiene. Do not import broader upstream reference
  prohibitions not present in the parent contract.
- **Test pins:** scan `src/**/*.rs`, `tests/**/*.rs`, and available mounted JS test
  sources for reads of `docs/{start,guides,concepts}/`. Keep tutorial/lint exemptions
  narrow and explicit, never a broad filename substring loophole.
- **Portability:** fail `hooks`, plugins outside search/redirects/tags/llmstxt, rendered
  `!!!` admonitions, `===` tabs, and `++key++`; reject `$` prompts in bash/sh/shell
  fences. Inline grammar and text transcripts stay valid. Authoring templates
  demonstrate separate `text` output fences titled Output with `.no-copy`.
- **Navigation:** enumerate publishable Markdown using the same exclusions as assembly.
  Every page appears once in expanded nav; no nonexistent targets or unresolved mount
  placeholders. Each displayed typed tab has a non-hub page; Home is special and
  contracts count under Develop. Treat configured redirects as compatibility endpoints,
  not nav orphans.
- **Privacy and media:** add `privacy.yml` patterns for real home paths, private vault
  repository URLs, personal LaunchAgent labels, and personal email addresses with
  documented synthetic placeholders. Scan code/examples and link destinations too:
  privacy is not hygiene. Reader pages and mounts have no baseline allowance. Bootstrap
  `privacy_baseline.txt` from existing contracts only, using path, rule, matched
  content, and occurrence counts so moved lines do not cause false drift and a second
  copy is a new hit. Normal checks cannot add baseline entries; explicit ratchet updates
  only remove obsolete hits. Every actual published image asset needs source/date in the
  local or mounted media manifest; code examples of Markdown image syntax are not
  assets. Check manifest paths and real asset existence.

Acceptance: the real site passes lint; every family has a failing fixture, including nav
omission (Zensical misses this), empty tab, malformed YAML, forged source, new privacy
hit in code, duplicate baseline hit, unregistered image, generated-page metadata/privacy
violations, and a shell prompt. Contract grammar examples survive.

## Phase url-links

Implement the three-layer URL contract and the independent link checker.

1. Build an index from actual output HTML paths and IDs/named anchors. Resolve directory
   URLs, index pages, encoded paths/fragments, and redirects with bounded loop
   detection. Resolve fragment-bearing redirects to the real destination/anchor, not
   just the existence of the stub. Reject path escapes and broken targets. Reuse this
   resolver for every URL source and future live-publish checks.
2. Write sorted `/urls.txt` from built content/redirect pages, excluding support outputs
   and error pages. Compare the live org-root `/urls.txt` against the new build: every
   prior page URL must still resolve. A genuinely offline request or first-publish 404
   warns explicitly; malformed successful responses and missing previously published
   pages fail. Bound network timeouts and validate the fetched site's origin rather than
   accepting arbitrary external URLs.
3. Scan every literal `docs/<path>.md(#fragment)` mention in bob-cli src, tests, README,
   and docs, preserving filename/line provenance. Also scan absolute site URLs in all
   assembled docs and each discovered source repo's README. Do not clone unused repos
   when the mount table is empty. Check all three READMEs once the three repos are
   sources. Parse Markdown destinations plus prose/code mentions without swallowing
   trailing punctuation.
4. Seed `url_mentions_ignore.txt` only for demonstrated non-public references, with a
   reason and source scope: notably README's link to the intentionally excluded
   repository hub `docs/README.md`, fixture-vault paths, or schema/template examples.
   Never exclude a real path printed by bob, a broken anchor, or an entire subtree. Seed
   curated `url_contract.txt` with the actual code-emitted halted-sync deep link and
   other verified inbound links. `urls --refresh` validates and adds observed curated
   candidates deterministically; it never silently deletes old obligations or freezes
   every contract heading. Emit clear before/after output.
5. `links` resolves lychee from `$LYCHEE`, then PATH, then a pinned upstream release for
   Linux x86_64/aarch64 and macOS arm64 into `~/.cache/bob-docs/bin/`. Commit
   version/platform/checksum metadata. Verify SHA-256 before extraction or execution;
   reject unsupported platforms and corrupt cached/downloaded binaries with actionable
   errors. Agents obtaining upstream repository material must use `/sase_repo`; official
   release assets can supply binaries/checksum evidence.
6. Run lychee offline with fragment checking over `target/docs/site/`, using the built
   directory as root and remapping absolute org-root URLs to local output. Verify the
   exact chosen release flags with its help. Propagate nonzero status; a missing
   mandatory checker is not a green skip. Keep online external checks available to the
   outer weekly publish phase without performing them in local check.

Acceptance: synthetic HTML and fixture Git repositories exercise every scan source,
anchors, encoding, redirects, loops, stale published URLs, offline/404 versus bad
manifests, narrow ignore reasons, refresh preservation, acquisition precedence,
checksum/platform failures, and correct subprocess flags/status. Run URLs and lychee
against the real strict build with every bob-printed docs path resolved.

## Phase authoring

1. Add `develop/writing-docs.md` explaining the parent rules: directory/type model,
   two-question placement test and contracts as specs, five templates, budgets,
   generated markers and drift checks, hygiene/test pins, portable Markdown, media and
   privacy, nav and mount ownership, stable URLs/redirects, and approved-plan
   requirement for IA moves. Add it to Develop nav.
2. Expand `develop/build.md` with docs commands and actual build/test instructions for
   each repo. Open siblings with `/sase_repo` and read their instructions/README before
   describing commands. Link their existing sources until live reference mounts exist.
   Verify help/examples without mutating a real vault; use temporary HOME/environment
   for commands that may touch user state.
3. Port the tutorial, how-to, concept, reference, and hub templates to
   `docs/_templates/`, replacing SASE nouns and commands with Bob examples. Keep
   tutorial/concept search boost, one-diagram concept guidance, confirmation steps, and
   Related/Next structure. They are excluded from the site, not placeholder nav.
4. Use `/sase_memory_write` under inherited `memory_docs_note = yes`; create
   `sase/memory/docs.md` with `type: reference` and this description: "Read before
   changing docs/, mkdocs.yml, tools/docs, README front doors, or user-visible CLI
   help." Include a short URL=path/typed-directory/contracts model and the routing
   table: new flag → `just docs-gen` and clap help; new hotkey → plugin reference;
   behavior change → contract/reference and only guide steps made incorrect; new goal →
   guide candidate; design decision → decision record. State `just docs-check`, sibling
   ownership of mounts, and approved IA moves. Link memory references with `[[...]]` if
   named, and run `sase memory init`.

Acceptance: authoring pages pass lint and both builders; all linked instructions exist;
the memory projection contains only the authorized note addition and its generated
instruction/README updates. No other canonical memory note changes.

## Phase integration

1. Add `search_smoke.py` and cases including `halted sync` at
   `/vault-git-sync/#halted-sync` and `conflict copy` under `/vault-git-sync/`.
   Zensical's `search.json` contains `config.separator` and `items` entries with
   location/title/text/path/tags. Require a case-insensitive literal query in a matching
   item's title, or for a multi-word goal all query words across title and text; the
   same item must have the expected location prefix, on a path/fragment boundary.
   Punctuation tokens match literally, not as empty normalized strings.
   Missing/empty/bad indices and zero cases fail. Do not claim this tests UI ranking.
2. Implement `check` as an explicit gate sequence: assemble/validate source context,
   lint, `gen --check`, strict Zensical build, URLs, offline lychee, search smoke, and
   pytest. Use an acyclic command graph: unit tests do not call the entire check
   pipeline recursively. Print each gate and propagate its exit status. Add themed
   `just docs-check`. Keep `just check` unchanged.
3. `check --production --sources FILE` threads the same explicit source context through
   all gates; no gate independently discovers a different repo. Check must never fetch
   sibling sources or generate tracked content in production. Tests that deliberately
   create dirty fixture repos stay separate from real-source verification. Expose
   straightforward call sites for future CLI goldens, tutorial, and media checks; each
   outer phase adds its gate when it implements it. Before those phases land, print
   precise `skipped: ... not implemented in this slice` messages rather than successful
   empty checks. Once installed, errors cannot be downgraded to skips; an absent
   optional Rust runtime also prints its reason.
4. Run full `just docs-check` and `just docs-build-material`, then confirm `git diff`
   shows no generated drift from building/checking. In disposable fixtures, insert a
   broken relative link and an unknown fragment into a contract and prove check exits
   nonzero with the source path and target. Restore fixtures; do not leave faults in
   tracked pages. Test `gen --check` detecting changed registry output and search smoke
   detecting an unfindable/mislocated query.
5. Record installed tool versions, actual checks and skip reasons, contract/page counts,
   negative-test evidence, production fixture evidence, and preserved `just check`
   semantics on the phase bead. If Zensical genuinely cannot build the approved subset,
   use the parent's Material fallback through one shared builder setting and record a
   proposed follow-up, keeping all mandatory gates active.

Acceptance: all parent toolchain acceptance criteria hold on the combined tree: full
docs-check green, both strict builders green (or the evidenced parent fallback), all
printed docs URLs resolve, every lint rule has a negative fixture, deliberate bad
links/anchors fail precisely, and no changes to `just check`. No Rust source changes are
expected; if any become necessary, run `just check` too.

## Landing and bead lifecycle

Every child phase closes only its assigned bead after checking its own
`sase bead epic-symbols <id>` entries. It records discoveries as `PROPOSED FOLLOW-UP:`
notes, never creates task beads, and leaves ancestor closure to the land agent. Prefer
finishing defects in the machinery owned here rather than deferring required acceptance
work.

The child epic's land agent integrates the six phases, verifies their recorded
acceptance on the landed tree, and collects follow-ups onto `bob-cli-a0.2` for the outer
land agent. It closes this child epic according to its launch instructions, then runs:

```bash
sase bead epic-symbols bob-cli-a0.2
sase bead close bob-cli-a0.2 --note "Verified the documentation toolchain, current contracts in place, strict and fallback builds, all gates and negative fixtures, authorized docs memory, and unchanged Rust check recipe; see attached evidence."
```

Resolve every leftover symbol or re-key it to the still-open owning bead before closing.
These closure steps are the nested land agent's responsibility; phase workers only
prepare evidence. **Never close `bob-cli-a0` or any other ancestor.** If a check failure
reproduces identically on the clean base, record the exact commands, failure signature,
and existing tracker reference as a proposed follow-up, then close the assigned
completed work as the user requested. Do not treat a new toolchain defect as a
pre-existing base failure.
