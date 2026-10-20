---
tier: epic
title: Finish the Bob docs toolchain landing fixes
goal:
  Fix the verified defects and acceptance gaps that the bob-cli-a0.2.1 landing audit
  found in bob-docs mounts, serve, lint, URL/link gates, and authoring templates, so the
  toolchain epic can close honestly.
parent_bead: bob-cli-a0.2.1
phases:
  - id: mount-safety
    title: Make mount assembly and serve safe and complete
    size: medium
    depends_on: []
    description:
      "mount-safety: fix front-matter stamping, destination symlink and gitignore
      validation, stale-deletion safety, cache freshness, serve build.json and live
      reload, and add the missing mounts acceptance tests."
  - id: lint-contract
    title: Lint the assembled site, not the authored approximation
    size: medium
    depends_on:
      - mount-safety
    description:
      "lint-contract: lint the expanded nav and verified mount provenance from the
      assembled context, stop lint discovery, fix privacy placeholders, line numbers,
      nested fences, and ratchet consistency, and add the missing negative fixtures."
  - id: links-authoring
    title: Check fragments offline, widen mention scans, and finish templates
    size: medium
    depends_on: []
    description:
      "links-authoring: enable offline lychee fragment checking, scan every text source
      for docs mentions, harden the URL resolver and live manifest, and give the
      authoring templates and guide Bob examples and Output fences."
proposed_by: bbugyi200.apollo.bob-cli-a0.2.1.land
proposed_with: claude/opus
create_time: 2026-10-19 21:02:57
status: wip
---

- **PROMPT:**
  [prompts/202610/docs_toolchain_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/docs_toolchain_landing_fixes.md)
- **PARENT:**
  [202610/docs_toolchain.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/docs_toolchain.md)

# Finish the Bob docs toolchain landing fixes

The land agent for epic `bob-cli-a0.2.1` ("Complete the Bob documentation toolchain and
contract site", plan `plan:202610/docs_toolchain.md`) verified its six closed phases on
master `cb42d654`. `just docs-check` passes all implemented gates and 121 tests, and
both strict builders pass. However, reading the code against the plan's acceptance
criteria found real defects in machinery that epic owns. Most are latent until the outer
epic's reference-mount phase configures `docs/_meta/mounts.yml`, which would then break
immediately. This child epic fixes them. Its `parent_bead` link hands control back to
the `bob-cli-a0.2.1` landing once it lands.

Read the governing plan before starting:

```bash
sase bead read bob-cli-a0.2.1 -r "Need the toolchain epic scope, decisions, and landing notes"
sase artifact read plan:202610/docs_toolchain.md "Need the toolchain contracts these fixes must honor"
```

That plan's "Shared implementation contracts" section and phase sections remain the
specification. Its inherited decisions are final: MIT, search indexing on, public design
decisions on, and the docs memory note limited to `sase/memory/docs.md`. **This epic
edits no SASE memory.** No Rust source changes are expected, and none are added to the
`just check` recipe.

All code is in `tools/docs/` (package `bob_docs`, tests in `tools/docs/tests/`), with
metadata in `docs/_meta/` and authoring pages in `docs/develop/` and `docs/_templates/`.
Run commands from the bob-cli checkout. Keep the existing module layout and idioms. CLI
help changes follow `cli_rules.md` (alphabetized options, short aliases).

## Phase mount-safety

Owns `bob_docs/mounts.py`, `bob_docs/assemble.py`, `bob_docs/build.py`, their tests, and
the serve/production sections of `tools/docs/README.md` and `docs/develop/build.md`.

1. **Stamping keeps front matter well-formed.** `stamp_markdown` drops the closing
   delimiter when a page already has front matter. For example,
   `---\ntitle: FM\n---\n\n# FM\n` becomes `---\nsource: …\ntitle: FM\n\n# FM\n`, so the
   page has broken front matter and loses its provenance. Re-emit the original closing
   delimiter line. Add a test that compares the **full** stamped output for a page with
   front matter and one without. Also assert that the stamped page parses with the same
   front-matter parser lint uses, keeping its title and gaining `source`.
2. **No destination escapes, checked before any write.** `validate_mount_targets` only
   checks whether the final destination file is a symlink. A symlinked `docs/reference`
   or target subdirectory therefore lets `sync_mount_copies` write outside `docs/`
   before `_stage_docs` notices. Before the first write or delete:
   - Reject any symlink in the path components from `docs/` down to each destination.
   - Require each destination's resolved parent to stay under the resolved docs root.

   Test with a fixture where nothing is written outside `docs/` and earlier mount
   outputs are untouched.

3. **Targets must be gitignored at assembly.** Today only lint checks this. Assembly
   writes copies into non-ignored targets. Move the `git check-ignore` helpers
   (`_gitignore_ignored`, `_gitignore_file_ignored`, `_path_ignored` in `lint.py`) to
   one shared place both modules import. Make `validate_mount_targets` fail, before
   writing, when a target directory (apart from its committed `index.md`) is not
   ignored. Test it.
4. **Stale deletion stays inside owned, ignored copies.** `sync_mount_copies` deletes
   every path listed in `target/docs/mount-manifest.json` that is no longer expected,
   with no other checks. Delete a stale path only when it is under `docs/`, not reached
   through a symlink, untracked, and gitignored. Otherwise fail with an actionable
   message before any write. Test that a tracked file at a previously owned path
   survives.
5. **No partial replacement.** Make sure every validation runs before the first write or
   delete: targets, destinations, stamping and spoof checks, and stale paths. Add a test
   where the second of two mounts has a spoofed `source:` page. Assembly must fail and
   leave the first mount's previous outputs byte-identical.
6. **Cache clones stay fresh and trustworthy.** A `~/.cache/bob-docs/<repo>` clone is
   reused forever and never checked against its remote. In non-production discovery:
   - Verify the cache's `origin` URL matches the mount `remote`. Fail if it differs.
   - Refresh the cache to the remote branch head with a shallow fetch and hard reset.
   - When genuinely offline, keep the existing cache with an explicit warning; the SHA
     still goes into `build.json`.

   Production never touches the cache. Test with a local fixture remote.

7. **Serve publishes `/build.json` and live-reloads authored edits.** Today `serve`
   neither copies the manifest nor sees edits: it serves a one-time copy of `docs/`
   staged under `target/docs/docs/`. Yet `tools/docs/README.md` and
   `docs/develop/build.md` promise live reload. Do two things:
   - Write the manifest into the staged docs tree during assembly, so build and serve
     both publish `/build.json` as a static file. Keep one copy path.
   - While `zensical serve` runs, mirror changes to authored `docs/` into the staged
     tree with a stdlib polling loop in a background thread. Cover additions, edits, and
     deletions, apply `_stage_docs`'s symlink rules, and preserve the staged
     `build.json` and anything assembly writes into the stage. Stop the loop when the
     server exits.

   Unit-test one mirror pass: a new file appears, an edited file updates, a deleted file
   disappears, and `build.json` stays. No test launches a real server. Confirm
   `urls.txt` and lint still ignore `build.json`.

8. **Record the builder actually used.** `build.json` always reports the Zensical
   version, even for `--material` builds. Record the tool versions of every installed
   builder (Zensical, plus MkDocs and mkdocs-material when installed) and the builder
   used. Test that the fields are present.
9. **Correct the production sources example.** The README example puts the JSON at
   `docs/_meta/production-sources.json` with `"bob-cli": {"path": "."}`. Paths resolve
   against the JSON file, so that path fails the build-root check. Make the example
   correct for its stated location.
10. **Add the missing mounts acceptance tests**, using fixture Git repositories:
    - An existing but invalid sibling or override (not a Git repo, or the wrong remote)
      fails without falling through.
    - A repo shared by two mounts is resolved once.
    - A `../` or absolute target is rejected.
    - A `..` source and a symlinked source directory are rejected.
    - Tracked files inside a target are rejected.
    - A destination file symlink and a parent-directory symlink are rejected.
    - A non-ignored target is rejected.
    - Production inputs: a missing `sha` field, a missing `path`, an untracked-only
      dirty tree, and dirt introduced after copying (the post-copy recheck) all fail.
    - `build` publishes `build.json` into the site.

Acceptance: all new and existing tests pass. `just docs-check` and
`just docs-build-material` are green with the empty mount table. Building and checking
leave tracked files unchanged.

## Phase lint-contract

Depends on mount-safety: it consumes that phase's assembled context, shared ignore
helpers, and mount manifest. Owns `bob_docs/lint.py`, `bob_docs/markdown.py`,
`docs/_meta/privacy.yml`, `tools/docs/tests/test_lint.py`, and lint wiring in
`bob_docs/cli.py` and `bob_docs/check.py`.

1. **Lint the expanded nav and copied mount pages.** Lint reads the authored config's
   nav and skips every mounted page, so an unlisted copied page passes. The empty-tab
   check also falsely fails a tab that holds only mount placeholders. Change three
   things:
   - Lint the nav that assembly expanded.
   - Have standalone `bob-docs lint` assemble first, exactly as `check` does, so it
     never lints an unexpanded or stale tree. `check` keeps passing its one context
     through.
   - Run the nav rules on the expanded nav. Every copied mount page appears exactly
     once; an unlisted copy fails and names its source repo's `nav.yml`. A tab qualifies
     when its expanded mount pages include a non-hub page.

   Add fixture-mount tests for each case.

2. **`source:` must match the stamp assembly wrote.** `_is_verified_copy` checks only
   that the page sits under a mount and the repo name matches.
   `source: bob-plugins: nowhere/at/all.md` therefore passes. Record each copy's exact
   `repo:path` stamp in the mount manifest. Accept `source:` only when the value equals
   that stamp; any other value fails as forged. Test both outcomes.
3. **Lint never discovers or fetches repos.** `check_test_pins` calls `discover_repo`,
   which can `git clone`. It ignores the context's resolved source dirs, so it breaks
   production mode, and it swallows `DocsError`. Scan JS test files only from the
   assembled context's `source_dirs`, and let errors propagate. Test that lint never
   calls `discover_repo` (monkeypatch it to raise) and that a JS test pin in a fixture
   mount repo fails.
4. **Placeholders pass their own rules.** The documented `vault-repo` placeholder,
   `git@github.com:example/vault.git`, matches the `vault-repo` pattern. Keep the broad
   non-`bobs-org` GitHub net and add an `allow` entry for the `example` owner. Document
   in the file header that a link to a public third-party repo needs its own narrow
   `allow` entry with a comment. Add a test asserting that every `placeholder` in the
   real `docs/_meta/privacy.yml` passes every pattern.
5. **Accurate line numbers.** Privacy findings always report line 1. Report the 1-based
   line of each reported hit's first occurrence, and test a hit below line 1.
6. **Fences nested in list items.** `FENCE_OPEN_RE` accepts at most three leading
   spaces, so a fence inside a list item goes unrecognized: `$ just x` is not flagged
   and its contents are not masked for hygiene. Recognize indented fence openers and
   their matching closers, the way pymdownx superfences nests them. Test both effects.
7. **Ratchet consistency.** `lint --fix` must prune exactly the ceilings that
   `check_stale_ceilings` reports, including excluded or support pages such as
   `_templates/t.md` and missing pages. After `lint --fix`, `lint` must never report a
   stale ceiling. Test it.
8. **Remove dead code**: `_mount_targets` and `lint_tree` in `lint.py`, and `mask_code`
   in `markdown.py`, unless a fix above uses them.
9. **Add the missing negative fixtures:**
   - **Budgets:** start page and start total, concepts, hub and homepage, handwritten
     reference including mounts, the develop warning severity, and guides warn versus
     fail.
   - **Ceilings:** a ceiling that is exceeded, and a stale ceiling.
   - **Mounts:** a mount-root collision and a non-gitignored mount target.
   - **Other rules:** a duplicate nav entry and a JS test pin.

Acceptance: the real site passes `bob-docs lint`, and every rule family has a failing
fixture. The capture.md grammar and completion.md transcripts still pass.
`just docs-check` and `just docs-build-material` are green.

## Phase links-authoring

Independent of the other phases. Owns `bob_docs/links.py`, `bob_docs/urls.py`,
`bob_docs/search_smoke.py` if touched, their tests, `docs/_meta/url_*`, the link and URL
sections of `tools/docs/README.md`, `docs/_templates/*.md`, and
`docs/develop/writing-docs.md`. Coordinate on `tools/docs/README.md` by editing only the
link/URL sections; mount-safety edits only serve and production sections.

1. **Offline lychee checks fragments.** The url-links phase concluded that lychee 0.24.2
   cannot check `file://` directory-URL fragments. The real cause was a missing
   `--index-files index.html`. On the landed strict build,
   `lychee --offline --include-fragments --index-files index.html --root-dir <site> --remap '<origin>/ file://<site>/' <site>`
   reports 3557 links and 0 errors. The audit also showed it catches injected broken
   relative and org-root anchors. Do this:
   - Add `--include-fragments --index-files index.html` to `lychee_argv`, checking the
     flags against the pinned binary's `--help`.
   - Update the `links.py` module docstring, the test that pins the omission (around
     `test_links.py:219`), the `check` gate message, and `tools/docs/README.md`.

   Keep `bob-docs urls` as the mention, curated, and live-URL resolver. Record evidence
   on the phase bead: the real-build run, plus one injected broken anchor in built
   output under a temporary site copy that fails.

2. **Scan every text source for docs mentions.** `scan_source_mentions` reads only
   `src/**/*.rs` and `tests/**/*.rs`. It misses mentions in, for example:
   - `src/native/highlights_ref/note_snapshot/note.lua` and `note_snapshot/note.tex`
   - `src/native/highlights_ref/diagrams.lua`, `diagrams_collect.lua`,
     `return_links.lua`, and `return_links.tex`
   - `tests/fixtures/freshness_keeps/vectors.json`

   Scan every UTF-8 text file under `src/` and `tests/`, skipping undecodable files and
   the usual build/vendor directories. Each new mention must resolve, or get a narrow
   reason-bearing ignore entry only if it is demonstrably non-public. Test with a
   fixture.

3. **Resolver robustness.** A URL with a fragment that resolves to an existing file
   outside the page index (for example `/404.html#x`, or an asset) raises `KeyError`
   (`urls.py` `resolve`). Raise a `DocsError` naming the URL, and test it.
4. **Live manifest strictness.** Three fixes, each tested with mocked `urlopen`:
   - An HTTP 200 `/urls.txt` with no URLs is malformed and must fail.
   - After HTTP redirects, the final response URL's origin must equal the configured
     origin, or the check fails.
   - Fix the `fetch_published_urls` return annotation for its offline result.
5. **Narrow ignore entries.** Only scope `*` is rejected today. Also reject:
   - scopes containing `**`;
   - scopes that are only wildcards;
   - scopes ending in `/`;
   - a MATCH that names no page (`docs/` alone, or with no `.md`).

   Test each.

6. **Refresh never curates ignored mentions.** `urls --refresh` collects candidates
   before ignore entries apply. Apply ignores first, and test it.
7. **Remove dead code**: `SiteResolver.page_paths`, the unused canonical-link collection
   in the HTML indexer, and the unused `uses` result of `apply_ignores`, unless used.
8. **Templates with Bob examples and Output fences.** The five templates in
   `docs/_templates/` are all placeholders. None shows the parent's Output convention:
   commands in `bash` fences without `$`, and output in a separate `text` fence titled
   `Output` and marked `.no-copy`. Make these changes:
   - Give each template at least one concrete Bob example. Use a real command or
     contract link whose flags you verify with `cargo run -q -- <cmd> --help`, and run
     any command only under a temporary `HOME` and vault.
   - In the tutorial and how-to, follow a command with its Output fence. Use the
     attribute syntax that both strict Zensical and Material 9.7 render. Prove it by
     temporarily building a scratch page that uses the fence, then removing it.
   - Keep the existing search boost, the one-diagram concept guidance, the confirmation
     steps, and the Related/Next structure.
9. **Writing docs completeness.** Expand `docs/develop/writing-docs.md` with the
   concrete rules lint enforces:
   - **Portable Markdown:** no `$` prompts, Output fences marked `.no-copy`, no `!!!`
     admonitions, `===` tabs, or `++key++` (use GitHub callouts), and no hooks or
     unlisted plugins.
   - **Hygiene:** no bead IDs, `src/` paths, epic or phase labels, or
     formerly/legacy/renamed wording in start, guides, and concepts; no bead IDs in
     reference.

   Also drop "when it is available" from the `just docs-check` sentence. In Related,
   match the link text for `build.md` to that page's title. Stay within the develop
   budget.

Acceptance: `just docs-check` is green, with lychee checking fragments offline and zero
errors. All printed `docs/…md` mentions across every text source resolve.
`just docs-build-material` is green. The templates and the writing guide pass lint.

## Landing and bead lifecycle

Each phase closes only its own bead, after checking `sase bead epic-symbols <id>`. Each
records evidence (commands, counts, injected-failure output) and any
`PROPOSED FOLLOW-UP:` notes on its own bead. Phases never create task beads and never
close ancestors. Do not run `just check-full`.

This child epic's land agent:

1. Verifies each phase's acceptance on the combined tree with `just docs-check`,
   `just docs-build-material`, and `just check`; the last confirms the unchanged Rust
   gate.
2. Confirms `git diff` shows no generated drift.
3. Triages follow-ups, then closes this child epic.

Its `parent_bead` link then resumes the interrupted `bob-cli-a0.2.1` landing. That
landing closes `bob-cli-a0.2.1` and then its parent phase `bob-cli-a0.2`, and marks
their plan files done. Those closure steps are not phases of this epic.
