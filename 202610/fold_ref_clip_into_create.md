---
tier: tale
title: Fold bob ref clip into bob ref create
goal:
  bob ref create covers everything bob ref clip did (--html replay, --author and
  --published overrides), clip disappears from help, completion, and docs, and bob ref
  clip survives only as a permanent hidden alias of create.
size: medium
proposed_by: bbugyi200.athena.0xs
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0xs](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0xs.md)
- **COMMITS:**
  - [62fa615](https://github.com/bbugyi200/dotfiles/commit/62fa615c5916416b5648a780a0cdf3d3b101a4bf)
    — docs(skills): bob_ref forbids bob ref create instead of clip

# Fold `bob ref clip` into `bob ref create`

## Goal

Retire `bob ref clip` as a separate command. `bob ref create <URL>` already sends web
article URLs through the same clip engine (`clip::capture_article`). That routing landed
in commit 0779e7d (bead bob-cli-4s.4). After this change `create` is the only documented
way to capture a web article. It keeps every capability `clip` had, and `clip` survives
only as a permanent hidden alias of `create`, as the CLI rules require.

## Findings that shape this plan

- **No caller needs the `clip` subcommand.** Nothing in bob-plugins, bob-mac-capture,
  the vault's `.obsidian` config, or `~/.config` and `~/bin` runs `bob ref clip`.
  In-process URL ingest (`highlights_ref::ingest::ingest_url`, used by capture and
  `bob gkeep pull`) calls the clip adapter directly, not the subcommand. Retry hints
  already say `bob ref create <url>`. The only outside mention is the chezmoi `bob_ref`
  agent skill, which lists `bob ref clip` among commands an agent must never run.
- **`create` is missing three `clip` flags:**
  - `-H, --html FILE|-` replays a page saved from a real browser, including from stdin.
    It is the documented workaround for bot-walled and login-walled pages, and open bead
    bob-cli-37 (`--from-chrome`) plans to build on it.
  - `-A, --author NAME` overrides the extracted author.
  - `-p, --published DATE` overrides the extracted publish date.

  Today `create`'s article route deliberately leaves all three unset (see the comment in
  `create_article_route` and `ClipOptions::for_create`).

- **Routing differs.** `create` first probes the URL with curl
  (`target::fetch_and_route`). It sends only HTML 2xx and 403/429/503 to the clip
  engine; PDF and arXiv URLs go to `create`'s own routes, and any other status fails.
  `clip` hands every URL straight to the adapter. With `--html` there is no reason to
  probe at all.
- **CLI rule (memory `cli_rules.md`):** never hard-remove a command. The old spelling
  becomes a permanent hidden alias with byte-identical behavior and no deprecation
  output. Help and diagnostics print only the canonical path. So `bob ref clip …` must
  keep parsing and must behave exactly like `bob ref create …`.

## Decisions

1. **`clip` becomes a hidden clap alias of `create`.** Add `.alias("clip")` to
   `create::command()`; clap aliases are hidden by default. Dispatch sees `"create"`, so
   `bob ref clip <args>` is byte-identical to `bob ref create <args>`, including
   `--help`, which prints `create`'s help with the canonical `bob ref create` usage. Do
   not print any deprecation output. The root aliases (`bob highlights clip`,
   `bob highlights-ref clip`) keep working through the existing root rewrite.
2. **Port all three flags to `create`, so every old `clip` invocation still parses.**
   Old `clip` short flags do not collide with `create`'s (`-A` ≠ `-a`; `-H` and `-p` are
   unused).
3. **`--html` forces the web-article route.**
   - It requires an http(s) TARGET. A local TARGET with `--html` fails before any work
     with a clear error.
   - With `--html`, `create` still runs its pre-fetch dedupe, listen-attach, and
     legacy-warning logic.
   - It then skips arXiv detection and the curl probe, and goes straight to
     `create_article_route` with the saved page.
   - `-H -` still reads stdin, because `capture_article` already handles that.
4. **`--author` and `--published` apply to every `create` route,** not just articles, so
   the help can say "Override the derived author/publish date" without a route-specific
   exception.
   - Validate `--published` as `YYYY-MM-DD` in `create::run` with
     `clip_url::validate_published` before any work.
   - Article route: pass the values through as the existing clip overrides. The dry-run
     source labels already read "override".
   - Local PDF, PDF URL, and arXiv routes (`pdf_target::PdfPlan`): the override wins
     over PDF-info and arXiv API values. Record the source as `override` for author, and
     add a matching published-source label, so dry-run stops hard-coding
     `published: … (arxiv api)` when the value was overridden.
   - Markdown route: put the overrides into the marker; today they are always `None`.
5. **Keep the engine's internal names.** `clip.rs`, `clip_adapter.rs`, `clip_url.rs`,
   the `BOB_WEB_CLIP_*` environment variables, the "web clip adapter" wording,
   `scripts/web_clip/`, and the doctor's web-clip rows all stay, because they name the
   engine, not the subcommand. Remove only the subcommand surface.
6. **Keep `create`'s probe policy.** Do not widen it. Statuses other than
   2xx/403/429/503 still fail on the probe. The error should point at `--html` as the
   escape hatch for a walled page (see Changes).

## Changes (bob-cli)

### `src/native/highlights_ref/create.rs`

- Add the args in alphabetical-by-short order, matching how `clip` sorted them
  (uppercase before lowercase):
  - `author` (`-A, --author NAME`, "Override the derived author"), placed before
    `-a, --audio`.
  - `html` (`-H, --html FILE`, `OsStringValueParser`, "Replay a page saved from a real
    browser (Save Page As, SingleFile); `-` reads stdin; forces the web-article route"),
    placed between `-f` and `-i`.
  - `published` (`-p, --published DATE`, "Override the derived publish date
    (YYYY-MM-DD)"), placed right after `-P, --parent`.
- Add `.alias("clip")`.
- Add `author`, `html`, and `published` to `CreateOptions`. Validate them in `run`:
  `--published` format, and `--html` needs a URL TARGET.
- In `create_pdf`, after the URL dedupe, listen-attach, and legacy-warning block:
  `if options.html.is_some()`, call `create_article_route` directly and skip both the
  arXiv branch and `fetch_and_route`.
- In `create_article_route`, pass `author`, `published`, and `html` into the clip
  options, and update the "stay unset on this route" comment.
- Apply the author/published overrides on the Markdown, local-PDF, PDF-URL, and arXiv
  plans as described in Decision 4, including their dry-run printers.
- Rewrite the `after_help`:
  - Replace "same PDF, marker, and report as `bob ref clip`" with a short description of
    the article route.
  - Add a "Web articles" paragraph carrying the useful parts of `clip`'s old
    `after_help`: reader-mode re-typesetting, headed retry (Xvfb on Linux, off-screen
    window on macOS), `--html` replay, and the `BOB_WEB_CLIP_ADAPTER`, `BOB_CHROME`,
    `BOB_WEB_CLIP_TIMEOUT_SECS`, and `BOB_WEB_CLIP_KEEP_WORKDIR` variables.
  - Add one example with `-H saved.html` and one with `-A`/`-p` overrides.
  - Keep the help excellent and easy to scan, per `cli_rules.md`.
- Optional, small: when the probe fails with a non-2xx status on what looks like an
  article URL, add `hint: save the page from a browser and pass --html FILE`.

### `src/native/highlights_ref/target.rs`

- Add the `--html` hint to the HTTP-status error from `fetch_and_route`, if you do it
  here rather than in `create.rs`.

### `src/native/highlights_ref/clip.rs`

- Delete the subcommand surface: `command()`, `run()`, `clip_options()`, `clip_pdf()`,
  and the `DEFAULT_*` constants used only by them.
- Replace `ClipOptions::for_create` with whatever constructor `create` now needs; for
  example, make the fields `pub(super)` and build the struct in `create_article_route`
  with the same `validate_name` and `validate_ref_type` checks. Avoid growing the
  `too_many_arguments` constructor.
- Keep `capture_article`, `Companion`, the dedupe and attach helpers, the dry-run
  report, `format_bytes`, and the doctor rows.
- Rename `bind_hint_for_clip` and `check_dedupe_for_clip` only if it reads better. Their
  hint text already points at `bob ref create`.
- Update the module doc comment to say this is the web-article capture engine behind
  `bob ref create <article URL>`.
- Update the uv-missing message ("bob ref clip cannot run its capture adapter") to name
  `bob ref create`.

### Other `highlights_ref` files

- `mod.rs`: drop `Some(("clip", …)) => clip::run(…)`. Change `pub(crate) mod clip` to
  `mod clip` if nothing outside needs it.
- `cli.rs`: remove `"clip"` from `HELP_GROUPS` and `clip::command()` from
  `all_subcommands()`.
- `clip_adapter.rs`, `clip_url.rs`, `stamp.rs`, `ingest.rs` ("mirroring `clip`"): fix
  doc comments that name `bob ref clip` so they name the engine or `bob ref create`.

### Completion and environment

- `src/native/completion/kinds.rs`:
  - Delete the `["ref", "clip"]` / `output` entry and its `lookup_exact` assertion.
  - Delete the generic `url` entry: `clip`'s `<URL>` was its only user, and
    `every_table_entry_matches_the_tree` will flag it as stale.
  - Keep the generic `html`, `author`, and `published` entries; `create` now uses them.
  - Keep the generic `clip` entry, which belongs to capture's `--clip`.
  - Fix the module doc comment that says "`ref clip` / `ref create`".
- `src/native/env.rs`: update the comment at ~line 175 that names `bob ref clip`.

## Tests (bob-cli)

- Delete `tests/cli/highlights/clip.rs` and move every scenario `create` does not
  already cover into `tests/cli/highlights/create.rs`:
  - overrides plus `--html` FILE
  - `--html -` from stdin
  - dry run writes nothing and prints the fidelity report
  - adapter failures with hints
  - pre-adapter rejections and option conflicts
  - library and dedupe collisions
  - legacy-only URL warns and captures
  - PDF-backed URL note still refuses
  - `--force` over the same intake target
  - protocol and garbage-adapter errors
  - round trip through scan

  `--html` cases need no fake curl, because the probe is skipped. Assert that
  explicitly: no curl log, or point `BOB_HIGHLIGHTS_CURL` at a binary that fails. Other
  cases use the existing `write_fake_curl`, `article_env`, and `FakeClip` helpers.
  Update `mod.rs` and the `fake_clip.rs` header comment.

- New create tests:
  - `--html` with a local TARGET is rejected before any work.
  - A bad `--published` is rejected before any work.
  - `-A` and `-p` override author and published on the local-PDF route, and the arXiv
    route shows them in the dry run with an `override` source.
  - The Markdown marker carries `-A` and `-p`.
- Alias tests: `bob ref clip <args>` is byte-identical to `bob ref create <args>` on an
  error path and on `--help`. `bob ref --help` no longer lists `clip`. `ref` completion
  candidates do not include `clip`.
- `tests/cli/highlights/listen.rs`: switch the two `highlights clip` invocations to
  `create`, with fake curl or `--html`. Keep `clip_listen_captures_and_binds_episode`'s
  assertions under a create-named test.
- `tests/cli/help_options.rs`:
  - Turn `highlights_clip_help_lists_options_alphabetically` into a `create` help-order
    test that includes `-A, --author`, `-H, --html`, and `-p, --published` in their
    sorted positions.
  - Update the `bob ref`/`highlights` help listing assertion that expects `"\n  clip "`
    (around line 676) to expect it absent.
- `tests/cli/ref_library/mod.rs`: remove `"clip"` from `VERBS` and its error-path case.
  Keep alias coverage through the new alias test above.
- `tests/cli/completion/protocol.rs` (around line 242): drop `"clip"` from the loop.
- `tests/cli/support.rs`: keep the fake-adapter guard, and only reword the comment if
  needed.
- Unit tests in `cli.rs` (`help_groups_cover_every_subcommand_exactly_once`) must keep
  passing. The hidden alias is not a separate subcommand.

## Docs and build (bob-cli)

- `justfile` `install-smoke`: replace `ref clip --help` and `highlights clip --help`
  with nothing, or keep one `ref clip --help` line explicitly labeled as the alias
  smoke. Keep the `create` lines.
- `README.md`:
  - Remove the `bob ref clip …` usage line (~830).
  - Add `[-A|--author NAME]`, `[-H|--html FILE]`, and `[-p|--published DATE]` to the
    `bob ref create` usage line in sorted order.
  - Reword the mentions at ~221, ~1047–1049, ~1075, ~1111, ~1173, and ~1212–1218 to name
    `bob ref create` (web article targets).
- `docs/highlights-clip.md`: keep the file to avoid link churn, but retitle it "Web
  article capture (`bob ref create <URL>`)".
  - Rewrite every example to `bob ref create … `.
  - Drop the sibling-command framing.
  - Keep the pipeline, hosts/browsers, `--html` escape hatch, environment,
    marker/dedupe, failure kinds, and adapter protocol sections.
  - Note in one line that `bob ref clip` is a hidden alias. That is acceptable in
    reference docs; help output still names only the canonical path.
- `docs/highlights-create.md`: remove "same … as `bob ref clip`" (lines ~9 and ~46) and
  "`clip` accepts the same flag" (~151). Add the three flags to the options and
  examples, and link to `highlights-clip.md` for article-route details.
- `docs/highlights-ref-sync.md` (~88–96): remove the `clip` usage and paragraph, or
  point them at `create`.
- `docs/getting-started.md` (~218): drop `ref clip` from the row.
- `docs/README.md` (~23): update the `highlights-clip.md` description.
- `scripts/web_clip/web_clip_adapter.py` and `scripts/web_clip/template/reader.css`:
  update the header comments that say "for `bob ref clip`". Do not change behavior; the
  adapter is pinned, so make sure the adapter checks still pass
  (`just check-web-clip-adapter` / `check-adapter`).

## Cross-repo: chezmoi `bob_ref` agent skill

- Open the linked `chezmoi` repo with `/sase_repo`.
- In the canonical skill source `home/sase/skills/bob_ref.md`, change "Never run
  `bob ref clip`, `create`, `scan`, or `sync`" to "Never run `bob ref create`, `scan`,
  or `sync`".
- Regenerate the provider copies. They live under `home/dot_claude/skills/`,
  `home/dot_codex/skills/`, `home/dot_config/{muse,opencode}/skills/`,
  `home/dot_gemini/antigravity-cli/skills/`, `home/dot_grok/skills/`, and
  `home/dot_qwen/skills/`. Use the repo's normal generator (see `sase skill list`,
  `sase skill init`, and chezmoi's `AGENTS.md`); do not hand-edit them unless that is
  the repo's convention.
- Commit that repo too; `/sase_final` will list it as an obligation.

## Bead bookkeeping

Add a note to each open feature bead whose title still says `bob highlights clip`:

- bob-cli-37: `--from-chrome` should now feed `bob ref create`'s `--html` replay path.
- bob-cli-38: page mode, image rescue, and `--keep-source`.
- bob-cli-39: recapture and versioning.

Each note should say that the entry point is now `bob ref create <URL>` (web-article
route) and that `clip` is only a hidden alias. Use `sase bead note`, and do not change
their status.

## Verification

- `just all` (fmt, lint, test) is green, apart from failures that already fail on master
  before this change. Name any such failures by test and bead.
- `just install-smoke` passes.
- Manual checks with the built binary:
  - `bob ref --help` lists no `clip`.
  - `bob ref clip --help` and `bob ref create --help` produce byte-identical output.
  - `bob ref create https://example.com/x -H saved.html -A "Jane" -p 2026-01-02 -d`
    prints a plan whose author, published, and title sources read `override`, and it
    never runs curl.

## Out of scope

- Widening `create`'s probe policy to send other HTTP statuses to the adapter.
- Renaming the engine modules, `BOB_WEB_CLIP_*` variables, or `scripts/web_clip/`.
- Implementing bob-cli-37, bob-cli-38, or bob-cli-39.
