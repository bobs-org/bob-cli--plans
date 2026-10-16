---
tier: tale
title: Protect intraword underscores from Prettier in sase-formatted Markdown
goal: sase's Prettier formatting never rewrites an intraword underscore such as `memory_glossary`
  into `*`, formatting a plan twice gives the same text, and `sase plan propose` rejects
  any plan that a second formatting pass, like the one archival runs, would make invalid.
size: medium
proposed_by: bbugyi200.apollo.81
status: done
---

# Plan: Stop Prettier from rewriting intraword underscores in sase-formatted Markdown

## Context

### What failed

The `research.1o.linker.w0` epic launch monitor ran
`sase bead work ~/.sase/plans/202610/article_notes.md --yes-to-all …` and exited 1 while
archiving the approved plan:

```text
Error: could not archive epic plan …/article_notes.md: committed plan validation failed:
…/article_notes.md:888: error [decision-branch-unknown] decision callout names unknown decision `memory*glossary`
```

The plan on disk said `memory_glossary`, which is a valid decision ID.

### Root cause

`archive_plan_file` (`src/sase/sdd/plan_archive.py`) runs `format_with_prettier` on the
plan before committed-plan validation. Prettier's Markdown parser treats the intraword
`_` in `memory_glossary` as an emphasis opener. It pairs that `_` with a later `_`
closer in the same paragraph, then reprints the span with `*` delimiters, because `_`
cannot sit inside a word:

```text
in:  > [!decision] memory_glossary Add … (also called an _article_) as a Bob
out: > [!decision] memory*glossary Add … (also called an _article*) as a Bob
```

CommonMark never lets an intraword `_` run open or close emphasis, so this is a parser
bug, not a style choice. The same bug reproduces on Prettier 3.6.2 (the global binary
sase shells out to) and on 3.8.1 (sase's `package.json` pin), so upgrading Prettier does
not fix it. The closer can even sit inside a later code span, which then gets a `*` too.

### Why `sase plan propose` did not catch it

The planner wrote `(also called an *article*)`. Propose formats the plan once and
re-validates it in `_formatted_plan_failure` (`src/sase/main/plan_propose_handler.py`).
On that first pass Prettier only normalized `*article*` to `_article_`, which is
harmless, so propose passed. After approval, the archive formatted the stamped plan a
second time. That second pass paired the new `_article_` with `memory_glossary`. Raw
Prettier is not idempotent on this text, so validating one formatting pass does not
guarantee that the archive's next pass stays valid.

### Already handled, and not a one-off

Bryan's agent `82` worked around this for the incident. It moved each
`> [!decision] <id>` header into its own paragraph and relaunched the epic, which is now
running as `bob-cli-8g`. This plan is the systemic fix.

The same corruption has landed silently wherever it did not hit a decision ID, because
validation still passed:

- bob-cli's archived plan `202610/web_url_highlights_clip.md` contains `snake*case`,
  `` `*` ``, and `` `<host>*<YYYYMMDD>` ``.
- `202609/core_ref_resolution.md` contains `doc*kind … _notification id*`.
- In sase's own docs, raw Prettier would rewrite `$SASE_TMPDIR` and `'sase_tmux_<N>'` in
  `docs/reference/cli/tui.md`.

### Prior art and evidence for the fix

`src/sase/markdown_literals.py` already protects every `_` and `*` in agent prompts
(`format_agent_prompt_markdown`) with width-preserving Unicode private-use (PUA)
sentinels. Plain `format_with_prettier` deliberately keeps raw-Prettier parity for other
Markdown, so it got no protection.

A prototype protected only intraword underscore runs, replacing each `_` with one PUA
sentinel before Prettier ran and restoring it afterwards. It was run against 308
archived plans and 250 tracked Markdown files in sase:

- The output was byte-identical to raw Prettier everywhere except the corrupted
  documents above.
- The one other difference was a table in `docs/reference/cli/restore.md`. Raw
  Prettier's backslash escapes, once unescaped, leave that table's separator row two
  characters too wide. The protected output matches the committed file.
- The protected output was idempotent on all 558 files. Raw Prettier was not idempotent
  on one.

The protection therefore changes output only where raw Prettier would damage the text.

## Where the work happens

All code changes are in the **sase** repo. Open it with your `/sase_repo` skill
(`sase repo open sase -r "<why>"`), work only in the printed path, and read that
checkout's `AGENTS.md` before editing. bob-cli needs no changes.

## Changes

### 1. Intraword-underscore protection (`src/sase/markdown_literals.py`)

- Add `protect_intraword_underscores(text) -> tuple[str, str]`, which returns the
  protected text and its sentinel. Add `restore_intraword_underscores(text, sentinel)`.
- Protect runs of `_` that have a Unicode letter or digit on both sides:
  `(?<=[^\W_])_+(?=[^\W_])`. Replace every underscore in a run with the same
  single-code-point sentinel. That keeps widths identical, because Prettier measures a
  PUA character as width 1.
- Choose the sentinel by the same rule as `_choose_sentinels`: the first PUA code point
  absent from the text. Factor the scan into a shared helper rather than duplicating the
  loop.
- Apply the replacement to the whole text, including code spans, fences, URLs, HTML, and
  YAML front matter. The replacement is a bijection, so those regions round-trip exactly
  without a Markdown tokenizer.
- Leave every other `_` alone. That covers `_article_`, `_leading`, `trailing_`,
  `__init__.py`, `self.__dict__`, and an authored backslash-escaped underscore. These
  are real emphasis delimiters or escapes, and Prettier must still see them.
- In the module docstring, state three things:
  - The CommonMark rule: an intraword `_` run can never open or close emphasis, so
    protecting it cannot change the rendered meaning.
  - The Prettier behavior it defends against: the parser pairs such a run with a later
    `_` closer, even one inside a code span, and reprints the span with `*`.
  - The invariants: protect then restore is the identity for every string, and no
    intraword `_` run survives protection.
- Export both functions in `__all__`.

### 2. Protect at every sase Prettier entry point

These three functions are the only places sase invokes Prettier.

- **`format_with_prettier`** (`src/sase/file_references.py`)
  - Protect before the subprocess runs.
  - On success, restore after the existing backslash-underscore unescape loop.
  - Every fallback (Prettier missing, `CalledProcessError`, timeout) must still return
    the caller's original text, never the protected text.
- **`format_markdown_files_with_prettier`** (same file)
  - This is the bulk path that plan-links refresh (`src/sase/sdd/plan_links_refresh.py`)
    and artifact-link refresh (`src/sase/sdd/_artifact_link_refresh.py`) use to
    re-format archived plans, so it needs the same protection.
  - Read each file, keep its original text, and write the protected text.
  - Run the existing single `prettier --write` process.
  - Then read each file, unescape, restore, and write it back.
  - If Prettier fails or times out, write every file's original text back before
    returning `False`, so no sentinel ever persists on disk.
  - The missing-Prettier early return happens before any file is touched, as today.
- **`format_unique_skill_outputs_batch`** (`src/sase/main/_init_skills_rendering.py`)
  - Protect each output before writing its temp file.
  - Restore after `_unescape_prettier_underscores`.
  - Keep the existing fallback to the single formatter.
- **`format_agent_prompt_markdown`**
  - Its behavior does not change, because its protected text has no `_` left.
  - Its docstring says plain `format_with_prettier` "keeps raw prettier parity". Reword
    it to say that parity holds except for intraword underscores, which are protected
    because raw Prettier corrupts them.

### 3. Propose also validates the text that archival will produce

Change `handle_plan_propose_command` (`src/sase/main/plan_propose_handler.py`):

- Once the existing `_formatted_plan_failure(validation, formatted, …)` check passes,
  run the same check on `format_with_prettier(formatted)`. That text is what
  `archive_plan_file` produces when it formats the plan again at approval or epic
  launch.
- On failure, behave exactly like the existing first-pass failure: print the message,
  exit non-zero, and leave the scratch file, the archive, and the runner untouched.
- After Change 1, this check should never fire for intraword underscores. It closes the
  general gap where propose passes but the approved plan fails at launch, for any
  remaining Prettier instability. It costs one extra Prettier run per propose.

In `docs/sdd.md`, find the paragraph that begins "`sase plan propose` formats the plan
with prettier and re-validates the formatted copy". Add two points: propose also checks
that formatting the plan a second time, as archival does, keeps it valid; and sase
protects intraword underscores (`snake_case`) from Prettier's emphasis parsing. Follow
the sase repo's docs-writing guidance (its `docs_writing` reference memory) for that
edit.

### 4. Tests

`tests/test_markdown_literals.py` (pure, no Prettier):

- Protect then restore round-trips the existing `TABLE_ROWS` and these extra cases:
  - text that already contains a PUA character;
  - a fenced block, a code span, a URL, and YAML front matter, each containing
    `snake_case`.
- Only intraword runs are replaced:
  - `memory_glossary`, `a___b`, and `é_ü` are protected;
  - `_article_`, `_leading`, `trailing_`, `__init__.py`, and an authored
    backslash-escaped underscore are untouched.

`tests/test_format_with_prettier.py`:

- Unit tests that need no Prettier binary:
  - A fake Prettier asserts that no intraword `_` run reaches its stdin.
  - A failing fake proves that `format_with_prettier` returns the original text, not the
    protected text.
  - A failing fake proves that `format_markdown_files_with_prettier` restores each file
    to its original bytes, with no sentinel left behind.
- Real-Prettier tests, behind the existing `_prettier_required` marker:
  - The incident paragraph,
    `> [!decision] memory_glossary Add … (also called an *article*) …`, formatted once
    and then twice keeps `memory_glossary` and never contains `memory*glossary`. The
    second pass equals the first.
  - `foo_bar baz _qux_\n` comes back unchanged.
  - The code-span closer case from `web_url_highlights_clip.md` keeps `snake_case` and
    `` `_` ``.
  - `format_markdown_files_with_prettier` and `format_unique_skill_outputs_batch` both
    keep `memory_glossary`.
  - The existing wrap-parity and literal-table tests still pass unchanged.

`tests/test_plan_command_handler_formatting.py`:

- Propose fails, and proposes nothing, when the second formatting pass breaks
  validation. Simulate it with a formatter stub that returns its input on the first call
  and corrupts a decision ID on the second.
- With real Prettier, an epic plan shaped like the incident proposes successfully.
  Running `archive_plan_file` on that proposed plan passes committed-plan validation.

## Verification

- Run sase's checks from the opened sase checkout, as its `AGENTS.md` and
  `lint_and_test` reference memory direct (`just check` through `sase tool run`).
- Reproduce the incident by hand from the sase checkout. The output must keep
  `memory_glossary` and `_article_`:

  ```bash
  printf '> [!decision] memory_glossary Add `/sase_memory_write`. (also called an _article_) here\n' |
    python -c 'import sys; from sase.file_references import format_with_prettier; print(format_with_prettier(sys.stdin.read()), end="")'
  ```

## Out of scope

- **Relaunching the `article_notes` epic.** It is already running as `bob-cli-8g`.
- **Repairing already-corrupted archived plans.** These are `web_url_highlights_clip.md`
  and `core_ref_resolution.md`. They are historical records whose meaning is still
  recoverable.
- **Repo-run Prettier** (`just fmt-md` and similar). It still uses raw Prettier and
  still has this bug. Files sase writes into repos now differ from repo-run Prettier
  only where that Prettier would corrupt them.
- **Reinstalling sase globally.** That step is human-only. Bryan runs it after this
  lands.
