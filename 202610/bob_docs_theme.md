---
tier: tale
title: Complete Bob visual identity and documentation homepage
goal:
  Ship the Bob documentation theme, homepage, task examples, deterministic assets, and
  browser verification in both supported builders.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.3
create_time: 2026-10-19 22:27:23
status: wip
---

- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:**
  [bob-cli-a0.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.3.md)

# Complete Bob's visual identity and documentation homepage

## Outcome and assignment

Complete the theme phase, `bob-cli-a0.3`, of `plan:202610/bob_public_docs_site.md`. The
existing documentation build should produce a calm, distinctly Bob homepage and readable
articles in light and dark themes, with accessible task examples, local fonts, accurate
ownership links, and visible build provenance. Deliver the supporting asset generators
and a repeatable browser audit.

This is one substantial but bounded coding assignment: the assembler, strict builders,
contract registry, lint, URL/link gates, and documentation test suite already work. No
prior implementation notes or unfinished attempts were present on this phase when
planning. Implementation is confined to the documentation shell and its Python tools; it
does not require product behavior changes or work in another source repository.

Read these audited references before implementation:

```bash
sase bead read bob-cli-a0.3 -r "Confirm assigned theme scope, notes, and final decisions"
sase artifact read plan:202610/bob_public_docs_site.md "Implement the approved theme phase"
sase artifact read research:202610/bob_public_docs_site/bob_public_docs_site.md "Use the canonical visual design research"
```

If the research reference is missing, open `bob-cli--research` with `/sase_repo` and
retry the audited artifact read. This recovered it during planning. Read `docs.md` and
`cli_rules.md` with `/sase_memory_read`, and the three display contracts
`docs/task-tag-marks.md`, `docs/date-marks.md`, and `docs/today-marks.md` before writing
the fence formatter. Consult glossary memory for Capture, Pomodoro, and Task Link before
authoring examples about them.

The parent decisions are final: MIT, search indexing allowed, publish decisions, and
only `docs.md` authorized for memory edits. This implementation needs no memory edits
and introduces no reviewer decisions. Keep every contract at its current URL; there are
currently 27 registered contracts. Preserve additions from other epic phases, especially
generated navigation, Reference content, and tooling dispatch.

## Existing integration points

- `mkdocs.yml` uses Zensical 0.0.71, Material's classic-compatible theme, no remote
  fonts, `docs/overrides`, and SuperFences with an existing Mermaid formatter. Extend
  these settings; keep the plugin allowlist and avoid hooks or build-time plugins.
- `tools/docs/bob_docs/assemble.py` stages sources into `target/docs/docs`, rebases
  paths, and injects `extra.contracts` and `extra.build`. The registry is
  `{total, groups: [{title, contracts: [{stem, summary, reader?, historical?}]}]}`.
  Match contract pages by their actual source path, not every root-level URL.
- `extra.build` contains `sources` keyed by repository, each with `sha`, `dirty`,
  `branch`, and `path`, plus `built_at`, `builder`, and `tools`. `/build.json` already
  publishes the same manifest. Use it rather than inventing a second manifest.
- Mounted page provenance is `page.meta.source = "<repo>:<repo-relative-path>"`. Mount
  configuration provides the remote and intended branch. Worktree branch names are not
  necessarily valid edit-link branches.
- `tools/docs/bob_docs/cli.py` is argparse-based. `check.py` already runs lint,
  generation drift, Zensical, URLs, offline lychee, search smoke, and pytest. `justfile`
  contains the five basic docs recipes. Normal checks must remain read-only for tracked
  sources.
- `tools/docs/pyproject.toml` already defines optional `fonts`, `a11y`, and `material`
  groups; `uv.lock` is committed. FontTools/Brotli and Playwright are declared, but
  Pillow and an axe browser bundle still need integration.
- The homepage is a small placeholder and overrides contain only `.gitkeep`.
  `docs/_meta/media.yml` maps each image path to exactly `source` and an ISO `date`.

## 1. Local fonts and deterministic identity assets

Add `bob_docs/fonts.py` and a small asset module for `bob-docs social-card`. Wire both
commands into the CLI, keeping imports of optional packages inside their commands. Add
`just docs-fonts` using the fonts group and a similarly clear regeneration recipe for
the identity images. Add Pillow to the appropriate generation dependency group and
update the lockfile. Keep new subcommands/options alphabetized in help; give every new
public long option a short alias.

Use the pinned files in `assets/print_fonts/`, preserving those source files. Ship a
compact set of WOFF2 faces in `docs/assets/fonts/`: Inter body/UI including emphasis,
Inter Display SemiBold headings, and JetBrains Mono code. Treat the display face as the
heading face of the sans stack, and keep article typography to sans plus mono. Omit the
optional serif hero face. Use `font-display: swap`, local URLs only, and ship the
existing OFL notices beside the generated fonts. The total WOFF2 payload must be less
than 450,000 bytes; print and record individual sizes and the total.

Subset for Latin/Latin-1, punctuation, arrows, and box drawing, plus `•` and `⌗`. There
is a verified source-font gap: a read-only scan of all bundled font cmaps found no
U+2317 (`⌗`); JetBrains Mono does contain the box-drawing characters. Do not claim that
requesting a missing Unicode point from the subsetter adds it. Resolve this locally by
adding one docs-only hash outline, based on the task-tag contract's geometry, to the
generated mono derivative before subsetting. Give the derivative a Bob-specific font
name and preserve the upstream notices. Do not change the source TTF. Verify its cmap
and actual nonempty glyph, and let the rich task display use the contract's SVG mask if
that is visually more faithful. Other coverage should retain the available glyphs in the
requested ranges; validate representative punctuation/arrows/box drawing and explicitly
report unexpected missing required characters.

Keep generated bytes stable: fixed font timestamp handling, deterministic file order, no
current date or absolute machine paths embedded in fonts/images, and stable PNG
encoding. FontTools documents timestamp preservation and explicit Unicode selection in
its
[subsetting reference](https://fonttools.readthedocs.io/en/latest/subset/index.html).
Provide a `-c/--check` mode that compares generated files without rewriting them, and
test it with the generation dependencies installed.

Use one geometry definition for the bullet-b mark: an 18×18 y-down viewBox; a 1.5 px
round-capped vertical stroke from (4.25, 3.25) to (4.25, 10); a stroked circle centered
at (9, 10), radius 4.75; and a filled center circle of radius 1.6. Produce
`docs/assets/bob-mark.svg`, an SVG favicon, a 32×32 PNG favicon, and
`docs/assets/social-card.png` at 1200×630. Draw the raster assets deterministically with
Pillow and the bundled fonts; use paper, ink, teal, the mark, and short Bob copy.
Include all published image files in `docs/_meta/media.yml` with generator provenance
and the actual creation date. This is geometric asset generation, not AI image work.

## 2. Static, accessible `bob-task` examples

Implement `tools/docs/bob_docs/fences.py` and register its formatter and option
validator alongside the existing Mermaid fence using `!!python/name:`. Support normal
`bob-task` fences and an explicit `today` option; document the supported header syntax
after verifying it in both builders. The
[SuperFences custom-fence API](https://facelessuser.github.io/pymdown-extensions/extensions/superfences/#custom-fences)
provides the formatter and validator extension points; test actual Markdown conversion,
not just direct Python calls.

The formatter is a display-only subset of task syntax, not another capture parser:

- Render `[ ]`, `[*]`, `[/]`, `[?]`, `[x]`, and `[-]` as distinct status marks with
  accessible names Ready, Next, In Progress, Blocked, Done, and Cancelled. Display
  status semantically without suggesting the documentation has editable checkboxes.
- An exact `#task` token becomes the teal identity hash, with an accessible tracked-task
  label. Do not transform `#tasks`, `#Task`, escaped tokens, or inline-code contents.
  Render other tags as readable text pills; `#prj` uses violet. Color alone never
  carries meaning.
- The canonical `created`, `scheduled`, `completion`, and `cancelled` fields become
  monochrome date marks with their established shapes and absolute labels including the
  year. Expose the exact ISO date accessibly. Do not use the build date, relative
  labels, or inferred urgency. Preserve unsupported fields such as `due` and `start`
  literally; malformed or duplicate fields must not be presented as valid dates.
- Dim `[id:: ...]` and `[dependsOn:: ...]` while retaining readable contrast. Keep
  indentation, blank lines, and child lines such as Depends-On; never discard text.
- `today` is an explicit illustration annotation for open task rows, not a status or
  stored tag. Add the tomato mark in the identity slot and an accessible Today label;
  omit it from completed/cancelled rows. It must not be inferred from dates or lanes.
- Escape every user-supplied segment and attribute. Unknown/unparseable lines appear as
  raw escaped text with their whitespace, rather than failing the build. Keep unknown
  Markdown/wiki syntax literal instead of interpreting arbitrary HTML.

Add focused tests for all statuses, token boundaries/code spans, dates including leap
days and malformed values, children, Today, and HTML injection. Assert time-independent
output and that a real fence invocation reaches the custom formatter. GitHub should
continue to show the original, accurate task source inside an ordinary code fence.

## 3. Notebook theme and shared page overrides

Add `docs/stylesheets/bob.css`, load it from `extra_css`, and configure accessible
light/slate palette toggles. Use the parent design's tokens:

| Purpose              | Light                 | Dark                  |
| -------------------- | --------------------- | --------------------- |
| Paper / body text    | `#FBF8F2` / `#171C1A` | `#121816` / `#EEF3ED` |
| Muted text           | `#5C5F5A`             | `#A3A9A2`             |
| Links / active tab   | `#0F6B63`             | `#79DEDC`             |
| Identity glyph       | `#0A9391`             | `#79DEDC`             |
| Today/Pomodoro text  | `#9B3D2D`             | `#F58B72`             |
| Today/Pomodoro glyph | `#D9472B`             | `#F58B72`             |

Choose and record an AA-compliant non-tomato crimson pair for caution/warning styles.
Use tomato only for Today/Pomodoro. Map tokens onto Material variables for the header,
search, nav, content, and footer; ensure code syntax tokens remain readable in both
schemes. The header is paper with ink text, the mark, the Bob wordmark, and a teal
active-tab underline. Article text is 17–18 px, about 1.65 line height, and 65–75ch
measure. Scope the measure so the homepage and navigation can use their intended width.

Style tables, code, all five GitHub-callout types, tag/component chips, task lines, and
keycaps. Keycaps use a 1 px border, 2 px bottom edge, 4 px radius, and 0.85em mono.
Output blocks (`text`, title `Output`, `.no-copy`) look quieter than commands and have
no copy action. Prose links are underlined. Preserve visible teal focus indicators and
the skip link. Long tables/code scroll inside their own containers, never widening the
whole document. Navigation and the right rail collapse on mobile. Theme Mermaid from the
same tokens and honor reduced motion. Core content must render without JavaScript.

Use `docs/overrides/main.html` and small partials shared by Jinja2 and MiniJinja:

1. Above registry-listed contracts, show an inline icon and **Contract · written for
   implementers and tests**, followed by the registry summary. When `reader` exists, add
   the reader-oriented link using the registry's target; when absent, omit that
   sentence. Render historical entries truthfully. Never apply the banner to the
   homepage or an arbitrary typed page.
2. Make **Edit this page** use the owning repository/path from mounted provenance.
   Ordinary bob-cli pages keep `edit_uri`. Use the mount table's branch for mounted edit
   URLs; commit links in provenance use the recorded SHA. Avoid duplicate edit buttons
   or links back to the assembled copy. If template portability needs derived URLs,
   compute only that small additive mapping in the assembler, retaining its existing
   metadata shapes.
3. Render **Tested rolling snapshot**, source-repo short SHAs linked to commits, build
   date, and `/build.json`. Label dirty/unknown local provenance honestly instead of
   fabricating a commit. Add Compatibility only when its real page exists; later phases
   own that page. Do not display local filesystem paths in the footer.
4. Emit escaped Open Graph and Twitter metadata from page title/description and the
   absolute social-card URL, with sensible contract fallbacks. Use `site_url` rather
   than repeating a hardcoded host in templates. Supply the SVG and PNG favicon paths.
5. Honor `search_indexing = yes`: `robots.txt` allows crawling and points to the
   sitemap; pages must not emit `noindex`. Keep the parent MIT license choice.

Test actual output from both builders for a contract, a normal page, a registry entry
with/without `reader`, and a mounted-page fixture with a different repo/branch. Fixtures
must use local synthetic repositories, not require real siblings to be mounted.

## 4. An honest, useful homepage

Replace `docs/index.md` with portable Markdown and small `md_in_html` sections, with
`hide: [navigation, toc]`, title/description, a single main heading, and at most 400
words. Keep all reader-facing copy in Markdown rather than only in a template.

- Hero: **Capture a thought. Choose today's work. Keep an honest record.** A brief lede
  names the CLI, Obsidian plugins, and Mac Capture; explain the Obsidian workflow and
  that the Mac panel is optional. Show **Linux & macOS · Obsidian**, primary and
  secondary links to useful pages that actually exist. At the current baseline those can
  lead to `getting-started.md` and `develop/contracts.md`; do not invent Learn
  destinations before the learn phase creates them.
- **One thought · three surfaces · one vault**: a terminal card, an Obsidian task card
  using `bob-task`, and a simple menu-bar card using the mark. Obtain the terminal
  command and its actual output by running the current bob binary against a minimal
  synthetic scratch vault with an isolated HOME, XDG directories, BOB_DIR, fixed
  BOB_NOW/TZ, and reference-job kicks disabled. Use an ordinary local task capture
  without network/sync, and inspect the resulting note. Publish no real-vault paths or
  invented success output. Put the command in a prompt-free bash fence and output in its
  own no-copy block. Keep the Mac card illustrative; the later Mac phase owns the real
  panel screenshot. The learn phase will replace the terminal with its SVG.
- **A Bob day**: Morning review → Capture → Pomodoros → Close → Sync, linked to existing
  contract pages/anchors verified by the build. Use meaningful link labels.
- Show doors only for populated sections. Develop is present now; retain or add Look up
  only if the generated phase has landed real Reference content by implementation. Leave
  Learn/Do/Understand to their owning phases. Use small readable HTML sections so those
  phases can extend the homepage without replacing the design.
- Footer row: the three public source repositories and the actual MIT license link.

Keep current and concurrently added nav sections intact. Do not rewrite contracts, move
pages, introduce empty hubs, or change the primary README in this phase.

## 5. Repeatable browser verification and tooling guidance

Add `bob_docs/a11y.py`, `bob-docs a11y`, and `just docs-a11y` using the a11y dependency
group. The command checks an existing built site and reports its recorded builder; if
missing, fail with the build command to run. Serve it on an ephemeral loopback port
inside the runner and always stop the server and browser when finished. Use Python
Playwright with Chromium. Missing browser/tool dependencies are actionable failures, not
green skips. Document the browser install command in `tools/docs/README.md`.

Use a pinned official axe-core browser bundle, vendored under `tools/docs/vendor/` with
its license, version, source, and checksum. Inject it locally for audits; it is not a
public site script and QA must not download an unpinned CDN script at runtime.

The default matrix covers the homepage, the short `today-marks.md` contract, long
`capture.md`, and a real CLI reference page if present. Explicitly report when that
later-phase page does not yet exist. Test 360, 768, and 1280 px in light and dark. Save
screenshots and a machine-readable report under `target/docs/a11y/<builder>/`. For the
very long capture contract, use viewport images at its top and a representative deep
section rather than a huge full-page bitmap. Require zero serious/critical axe
violations, reporting selectors and rules on failure. Do not disable rules wholesale to
make the run green.

Also verify actual reflow at 200% zoom with a 360 px viewport, allowing horizontal
scroll only within code/table containers; do not mistake device pixel ratio for zoom.
Exercise keyboard-only skip-link, nav, palette, search, and content navigation with
visible focus and no trap. Run a no-JavaScript browser context to verify readable core
content and working ordinary links. Check that fonts load locally without external font
requests. Include a representative scratch fixture for every status, dates, keycaps,
Output blocks, five callouts, and a Mermaid diagram so checks do not depend on later
phases adding examples.

Open the generated screenshots with the image-viewing tool, inspect both schemes and all
widths, and iterate on actual layout issues. Save useful evidence and record the audit
matrix/results on the phase bead. If screenshots need durable publication, use the
artifact skill; ignored output paths alone are not durable evidence.

Update `tools/docs/README.md` and the existing `docs/develop/writing-docs.md` and
`docs/develop/build.md` only where needed to explain the fence syntax, fonts/assets, QA
setup, and recipes. Correct the tooling README's claim that only `docs-gen` and
`lint --fix` write tracked files: the explicit asset generators now do too. Browser
audits remain dev-only as specified by the parent epic; do not make ordinary
`docs-check` require Chromium. Formatter/template tests and cheap committed-asset
validation belong in the normal documentation pytest suite. Optional generation tests
run explicitly with the fonts group and report their dependency skips when absent.

## Validation and completion

Run the focused formatter and template tests first, then:

1. Regenerate fonts and identity assets; run their check modes and confirm a second
   generation is byte-stable. Inspect dimensions, glyph coverage, font total, and media
   registration. Record the WOFF2 byte sizes in completion evidence and the eventual
   commit description when the host prepares it.
2. Run `just docs-check`, including all current gates and Python tests. Verify relevant
   tests with the fonts dependency group installed. No Rust changes are planned; if
   implementation nevertheless changes Rust, `just check` is also required.
3. Run `just docs-a11y` against the Zensical site and inspect its images. Verify the
   real homepage capture command/result and resulting vault task in isolation.
4. Run `just docs-build-material`, then the browser audit on that built site. Confirm
   banners, ownership links, task rendering, fonts, metadata, and provenance actually
   work in Material, not merely that the process exits successfully. Preserve each
   builder's reports separately.
5. Check the diff for generated-nav loss, memory changes, real personal data, leaked
   scratch outputs, unexpected dependency changes, and moved URLs. Use `/sase_monitor`
   when a long command needs a SASE handoff; never end a turn with a yielded command
   still running.

This plan does not complete the phase by itself. The approved coder must finish and
verify the implementation before the phase is closed. Record discovered out-of-scope
work only as `PROPOSED FOLLOW-UP:` notes on `bob-cli-a0.3`; do not create task beads. If
a check fails identically on the clean base, record the exact reproduction and any
existing tracking bead as such a follow-up rather than leaving the phase open solely for
that failure.

Before completing the assigned phase, run:

```bash
sase bead epic-symbols bob-cli-a0.3
```

There were no entries at planning time. Resolve any that now exist, or re-key them to a
still-open appropriate phase/epic. Record what was implemented and verified, then close
only the assigned phase with
`sase bead close bob-cli-a0.3 --note "<implementation and verification evidence>"`.
Never close `bob-cli-a0` or another ancestor plan bead. If the handoff runtime assigns
the coding turn to a child plan instead, supply this phase's completion evidence to the
authorized host/land flow rather than treating this paragraph as permission to close an
ancestor. Follow `/sase_final` for the final declaration; do not manually commit unless
separately instructed.
