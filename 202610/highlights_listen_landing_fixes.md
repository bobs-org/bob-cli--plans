---
tier: tale
size: medium
title: Finish bob-cli-4s — fix the create/clip --listen landing defects, then close
  the epic
goal: Every epic-caused defect and spec gap from the bob-cli-4s land audit is fixed
  with regression tests; listen, attach, PDF, arXiv, and article routes are all-or-nothing
  as specified, create's TARGET completes in bash and zsh, the epic leaves no dead
  code, and epic bob-cli-4s is closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-4s.land
bead: bob-cli-4s
status: done
---

- **PARENT:**
  [202610/highlights_create_listen.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/highlights_create_listen.md)
- **BEAD:**
  [bob-cli-4s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4s/README.md)

# Finish epic bob-cli-4s: highlights create/clip `--listen` landing fixes

## Context

Epic **bob-cli-4s** ("bob highlights create --listen and every sase-listen target", plan
`plan:202610/highlights_create_listen.md`) has all six phases closed and its code on
master (commits `476f4ae`, `fb77b56`, `fa7c7b0`, `0779e7d`, `acceed2`, `fe1c05f`). The
land agent's audit found defects and spec gaps that this epic caused. This tale fixes
them and then closes the epic. It is the epic's last piece of work; nothing resumes the
landing after it.

Read the epic plan for the contract before changing behavior:
`sase artifact read plan:202610/highlights_create_listen.md "<why>"`. Its sections
"Dedupe and attach mode", "Listen command contract", "Ordering and failure semantics",
"Output", and "PDF route" are the spec. The code lives in `src/native/highlights_ref/`
(`create.rs`, `clip.rs`, `attach.rs`, `listen.rs`, `companion.rs`, `pdf_target.rs`,
`target.rs`, `fetch.rs`, `doctor.rs`) and `src/native/completion/`. The CLI tests are in
`tests/cli/highlights/{create,listen,clip}.rs`, `tests/cli/help_options.rs`, and
`tests/cli/completion/`.

Follow-up triage is already done and recorded on the epic (bob-cli-4u, bob-cli-4j and
bob-cli-40 were corroborated, bob-cli-4v was filed, and the TTS-stall follow-up was
declined). Do not refile any of them.

Rules for every fix below:

- Write a failing regression test first where the item says so, then fix the code.
- Keep the existing `key: value` report lines and their order. Errors stay
  `bob highlights: error: <message>` plus an optional separate `hint:` line.
- Update `docs/highlights-create.md`, `docs/highlights-clip.md`, and the help text
  wherever a behavior below changes.

## 1. Correctness bugs (each needs a regression test)

1. **An interrupted listen on the article route exits 1.**
   - `impl From<ClipError> for CommandError` in `clip.rs` drops `exit_code`. So
     `bob highlights create <article URL> -L` exits 1 when the listen command is
     interrupted, while `clip -L` correctly exits 130.
   - Fix: preserve `exit_code` (`CommandError::with_exit_code`).
   - Test: a fake listen command that exits 130 on the `create` article route (fake curl
     serving `text/html`, fake clip adapter) makes bob exit 130.
2. **A stamp failure after the listen leaves orphan audio in the vault.**
   - Affected: the local PDF, PDF URL, and arXiv routes in `create.rs`, the three nearly
     identical `Some((command, flow))` blocks.
   - Today they run the listen command, install the audio with `install_listen_audio`,
     and only then call `pdf_target::stamp_pdf_to_scratch`. If stamping fails, the error
     path never calls `companion::cleanup_audio_on_failure`, so `xlib/…/<stem>.mp3`
     stays behind and the next run refuses.
   - Fix: stamp and verify into scratch **before** running the listen command. That
     fails fast before the paid render, and the spec says to produce the PDF before the
     listen. `{pdf}` stays the unstamped scratch copy.
   - After the listen, the only steps left are:
     1. the collision re-check;
     2. the audio install;
     3. `atomic_copy` of the stamped PDF, which deletes only audio this run created if
        the copy fails.
   - Factor the three duplicated blocks into one shared helper.
3. **Attach mode never re-checks before installing.**
   - `attach.rs::run_attach` runs `refuse_existing_audio` only before the listen.
     Afterwards `atomic_copy` silently overwrites a `xlib/<rel>.mp3` that appeared
     during the run.
   - Fix: re-run `refuse_existing_audio` after the listen and before the copy. On
     refusal, return through `listen::post_listen_error`: keep the scratch audio, print
     `kept:`, and give the attach hint `copy it to <xlib dest> for scan to pair`.
   - Test: the fake listen creates `xlib/<rel>.mp3` before exiting 0. bob refuses,
     prints `kept:`, and leaves the fake's file byte-for-byte unchanged.
4. **Clip's post-listen dedupe re-check is a no-op.**
   - `clip.rs` (the `--listen` path, after `refuse_target_collisions`) calls
     `check_dedupe` with the `recorded` list gathered before the listen.
   - Fix: re-collect the recorded sources after the listen. Do the same for any `create`
     route that re-checks dedupe after the listen.
   - Test: the fake listen writes a ref note whose `source_url` matches before it exits.
     bob refuses with `kept:`.
5. **The arXiv PDF download ignores the HTTP status and content.**
   - `create_arxiv_route` hands any downloaded body to PDF validation. A 404 or 503, or
     an HTML page, therefore produces the misleading
     `TARGET must be a Markdown file … /tmp/bob-create-…/arxiv.pdf`. The curl error hint
     is also dropped.
   - Fix:
     - Require a 2xx status; otherwise error with `server returned HTTP <n> for <url>`.
     - Require `%PDF-` in the first 1024 bytes; otherwise error with
       `server said PDF but sent something else`.
     - Keep the fetch error's hint as its own `hint:` line.
   - Test: fake curl returns 404 for `/pdf/<id>`.
6. **Shell completion of create's TARGET is broken in both shells. This is a regression
   from `*.md`.**
   - `src/native/completion/kinds.rs` gives the `target` positional
     `Kind::Files(Some("*.{md,pdf}"))`. Neither adapter brace-expands that pattern:
     - bash's adapter (`adapters/bob.bash`) matches with `[[ "$val" == $files_glob ]]`,
       so `a.md` and `b.pdf` do not match.
     - zsh's `_files -g` expands the pattern through `$~`, which also does not
       brace-expand: `zsh -fc 'p="*.{md,pdf}"; print -l ${~p}'` prints "no matches
       found".
   - Fix: emit space-separated patterns (`*.md *.pdf`).
     - zsh's `_path_files` already splits `-g` into words.
     - Make the bash adapter split `files_glob` on whitespace and keep a value that
       matches any of the patterns.
   - Check other `!files <glob>` consumers and the protocol docs and tests
     (`src/native/completion/protocol.rs`, `tests/cli/completion/`).
   - Test: in a directory with `a.md`, `b.pdf`, and `c.txt`, the bash adapter keeps
     `a.md` and `b.pdf` and drops `c.txt`. Add a zsh test too if the zsh adapter tests
     can express it.
7. **A companion beside a local PDF source is reported but never copied.**
   - `companion::plan_reused_companion` looks beside the target, then beside
     `extra_beside` (the source file). It marks both cases `reused: true`, so
     `copy_audio_for_install` skips the copy. The report still prints
     `audio: xlib/papers/<stem>.mp3 (from existing companion)`, but that file never
     exists.
   - Fix: only a companion already at the target's `<stem>.<ext>` counts as reused.
     Audio found beside the source is planned through `plan_audio_copy_for_source`,
     which handles the copy, reuse of identical bytes, the `--force` rules, and the
     library-destination refusal. The Markdown route uses the same helper, so it is
     fixed too.
   - Test: `paper.pdf` with a sibling `paper.mp3` outside the vault. After `create`,
     `xlib/papers/<stem>.mp3` exists and has the same bytes.
8. **`-N` alone embeds the marker id** on the Markdown and local PDF routes: in
   `create.rs`, `id = if options.include_id || options.name.is_some()`.
   - The spec and the command's own help say the id is set only with `-i`; with `-N`,
     `-i` embeds the name. URL routes always embed `id`.
   - Fix the condition and update any test that relied on the old behavior.
9. **A PDF that already carries a Highlights marker gets a second one.**
   - `pdf_target.rs` checks "already captured" only inside `lib/` or `xlib/`. A marked
     PDF outside the vault, or a downloaded PDF (PDF URL or arXiv) with a marker, gets a
     second marker prepended.
   - Per the spec, a page-1 note that parses as a Highlights marker means "already
     captured".
   - Fix: refuse before any write. Say that the PDF already carries a Highlights marker,
     and add a hint pointing to the library copy (`bob highlights sync`, or
     `create <library PDF> --listen` to add audio).
   - Test: an out-of-vault marked PDF is refused and nothing is written.
10. **curl reads `~/.curlrc`.**
    - `fetch.rs` never passes `-q`, so a `location`/`-L` line in `~/.curlrc` makes curl
      follow redirects itself and bypass the per-hop `validate_and_clean` check.
    - Fix: pass `-q` as curl's **first** argument.
    - Test: the fake curl logs its argv; assert that the first argument is `-q`.
11. **`create <library PDF> -L` misses attach with a relative or symlinked vault.**
    - `attach::attach_for_local_pdf` compares the canonicalized target with the raw
      `config.lib_dir`/`xlib_dir`, and so does its `vault_relative_path_value` lookup.
      `pdf_target::check_local_pdf_identity` also tries the canonicalized directories.
    - So with `-b vault` (relative) or a symlinked vault, attach returns `None`, and the
      identity check then refuses with "add --listen" even though `-L` was given.
    - Fix: use one shared containment helper that compares canonical directories.
    - Test: run from the vault's parent directory with `-b <relative vault>`.

## 2. Spec gaps

12. **Missing hint.** Markdown targets and out-of-vault local PDFs whose planned library
    or intake destination already exists must keep refusing. Their error must add the
    hint
    `to add audio to that capture, run bob highlights create <library PDF> --listen`,
    naming the existing PDF path when it is known. Add a test.
13. **The Markdown route creates `xlib/<ref-type>/` before the listen.** It calls
    `fs::create_dir_all` before the listen, so a failed listen leaves an empty
    directory. Create the parent directory only at install time (`atomic_copy` already
    creates parents; check the `stamp_and_install` path). Test: after a listen failure,
    `xlib/` has no new entries.
14. **Listen failure hints.**
    - On the `NoAudio` exit-0 failure, use `LISTEN_FAILED_HINT`
      (`nothing was written to the vault; rerun the same command once the listen error above is fixed`).
      The `{audio}` advice may follow it.
    - The `Interrupted` hint must also say that nothing was written to the vault.
15. **Attach output.**
    - `attach.rs` prints a plain `ok attached …` and `would attach …`. Use
      `Styler::success_prefix` like every other route.
    - Clip's attach source line must be `source: <url>`, not `source_url: <url>`.
16. **arXiv dry-run provenance.** The dry run always prints `author: … (pdf info)`.
    `PdfPlan.author_source` is never set or read. Wire it up so the line says
    `(arxiv api)` when the author came from the API.
17. **arXiv fallbacks use the versioned id.** They produce `arxiv_2608_04278v2` and
    `arXiv 2608.04278v2`. The spec says `arxiv_<id>` and `arXiv <id>`, without the
    version.
18. **Fetch hints are folded inline.** On generic URL routes, `target::fetch_and_route`
    folds the fetch hint into the message as ` (hint: …)`. Emit it as a separate `hint:`
    line.
19. **Doctor `curl` row.** Check `BOB_HIGHLIGHTS_CURL` first, which is the fetcher's
    precedence. Report `warn` when the override does not exist or is not executable.
20. **PDF download timeout.** `--max-time 30` caps the whole transfer, which is too
    short for PDFs approaching the 95 MiB limit on slow links. Use a generous cap, such
    as 300 s, for PDF and article downloads. Keep 20 s for the arXiv API query.
21. **`target::file_starts_with_pdf` reads the whole file.** Read only the first 1024
    bytes.
22. **Plausible Info title check.** It compares the title with the final stem
    (snake-cased or from `-N`). The spec says to reject a title that equals the source
    file stem or URL stem, compared case-insensitively.
23. **Help order and layout** (`sase/memory/cli_rules.md`: excellent, scannable help):
    - In both `create` and `clip`, put `-L, --listen` before `-l, --lib-dir`. That
      matches the plan's CLI surface and the commands' own `-N/-n` and `-T/-t` order.
      Update the help-order tests in `tests/cli/help_options.rs`.
    - Order create's after-help Targets, Listen, Audio, Examples. Fold the trailing
      legacy prose paragraph after Examples into a short **Output:** section placed
      before Examples. Keep its facts: the default target path, the `-o` rules and
      conflicts, `-N`/`-T`, scan/sync discovery, and the listen card.

## 3. Dead code the epic left behind

24. `cargo build --lib` currently prints 9 warnings: unused `target::resolve_target`,
    `resolve_url_target`, `CreateSource::Arxiv`, the `fetch` field, `with_hint`,
    `plan_create`, and `author_source`, plus the unused `pdf_target::*` and `target::*`
    re-exports.
    - `create.rs` also carries `#[allow(dead_code)]` stubs the epic added:
      - `plan_create_legacy_markdown_only`, `create_pdf_legacy_shim`, and
        `dispatch_create_pdf_after_scratch`;
      - `render_temp_path` and `code_break_filter_path`, now unused;
      - duplicate copies of `plan_audio_copy_for_source`, `resolve_audio_arg_path`, and
        `files_have_identical_bytes` that `companion.rs` replaces.
    - Fix:
      - Prefer routing create's dispatch through `target::resolve_target` if that
        removes the duplicated dispatch in `create.rs`; otherwise delete the unused
        items.
      - Delete the dead stubs and duplicates.
      - Item 16 wires up `author_source`.
    - Done when `cargo build --lib` and `cargo clippy --all-targets --all-features`
      print no warnings from `src/native/highlights_ref/`.
    - The pre-existing `#[allow(dead_code)]` items in `clip_adapter.rs` (`44bfe589`) are
      out of scope.

## 4. Strengthen the listen CLI tests (`tests/cli/highlights/listen.rs`)

25. Make the fake listen script log its environment as well as its argv, as the epic
    plan required. Then tighten these tests:
    - **PDF URL + `-L`:** assert the `{target}` and `{title}` values (including a title
      with `'`) from the fake's argv log, not from bob's stdout. The URL also appears on
      the `source:` line, so the current stdout check proves nothing.
    - **Markdown + `-L`:** prove that `{target}` exists while the command runs (the fake
      runs `test -f` and logs the result) and that it is outside `xlib/`.
    - **Listen failure:** assert that `xlib/` contains no files at all after exit 4, and
      that the exit-0-with-no-audio case also leaves no PDF.

## 5. Verification

- This repo has no `just check` or `just symvision` recipe; `just all` (fmt, clippy,
  `cargo test`) is the file-change verification.
- `cargo test` stops after the lib binary fails. Its only allowed failures are three
  known pre-existing lib tests that this tale must not try to fix:
  - `native::completion::kinds::tests::every_value_arg_has_a_decision`
    (`highlights create:audio`, tracked by bob-cli-4j);
  - `native::highlights_ref::create::tests::listen_filter_renders_card_and_encoded_play_link`
    (tracked by bob-cli-4u);
  - `native::capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
    (a parallel-only flake, tracked by bob-cli-40).
- So also run `cargo test --no-fail-fast` and confirm that every other test passes,
  including the full `cargo test --test cli` suite (1036 tests before this tale, plus
  the new ones).
- `cargo fmt --check` and clippy must be clean.
- Manually check completion: in a scratch directory with `a.md`, `b.pdf`, and `c.txt`,
  check the bash adapter's filtering and zsh's `_files -g '*.md *.pdf'` behavior.

## 6. Close out epic bob-cli-4s (final step, same turn as the code)

1. `sase bead epic-symbols bob-cli-4s`. It listed no entries at landing time. If any
   appear, resolve each one: wire it up, make it private, add a non-test pragma, or
   delete it. Re-key one to another bead only if a still-open bead needs the exemption.
2. Close the epic:

   ```bash
   sase bead close bob-cli-4s --note "<verification>"
   ```

   The note must summarize:
   - **Land audit:** all 6 phases verified against the plan and commits. No integration
     was needed: the only non-epic commits since the epic started, `0ca5a13` and
     `16aba77`, are docs that do not touch highlights.
   - **Triage:** recorded in an epic note; A→bob-cli-4u, B→bob-cli-4j, C→bob-cli-40, D
     declined, E→new bob-cli-4v.
   - **This tale:** the fixes it made and the `just all` / `cargo test --no-fail-fast`
     results, with the 3 known pre-existing lib failures named.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, clean them up and close again.

3. Run `just symvision` if the recipe exists. It did not exist at landing time; if it
   still does not, say so in the close note or the final report.
4. Set `status: done` (it is `wip` now) in the frontmatter of the epic's plan file. That
   file is the PLAN path printed by `sase bead read bob-cli-4s -r "<why>"`
   (`plan:202610/highlights_create_listen.md`).
5. bob-cli-4s has no `parent_bead`, so there is no parent to close.
6. In the final report, tell Bryan to run `just install` (or `just install-all`) to put
   the fixes on his `PATH`. Do not install them yourself.
