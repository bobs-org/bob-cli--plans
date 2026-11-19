---
tier: epic
title: 'bob bank import -D: import the week''s downloads, then delete them'
goal: '`bank import -D ~/Downloads/*.qfx`, typed into Bob Command Mode or a shell,
  previews the import and the planned deletions. Ctrl+Enter imports the batch, then
  permanently deletes exactly the source files bob parsed. It deletes them only if
  the whole batch was recorded and each file is unchanged since bob read it. A refused
  or failed import keeps every file, and the output says so.

  '
phases:
- id: expand
  title: bob expands ~ and glob patterns in bank import FILE operands
  depends_on: []
  size: medium
  description: 'expand: add a shared native::host_paths operand expander (tilde, literal-first
    glob, dedupe, no-match refusal) and adopt it in bob bank import. Also light up
    the "bank" Command Mode invalidation tag, filter completion to exports, add a
    macOS privacy hint for unreadable files, and document file patterns.'
- id: delete
  title: The -D/--delete flag, its delete gate, output, JSON, and docs
  depends_on:
  - expand
  size: medium
  description: 'delete: add -D/--delete with an up-front check of each file, a gate
    that deletes only after the batch succeeds and only files whose bytes still match
    their digest, per-file outcomes, colored human output, an additive JSON `delete`
    object, exit codes, and the docs/bank.md and Monday guide rewrite.'
- id: card
  title: Bank card weekly-import button opens the Command Mode preview
  depends_on:
  - delete
  size: small
  description: 'card: in bob-plugins, turn the bank queue card''s static "Weekly import"
    footer text into a one-click chip. The chip opens the Command Mode preview of
    `bank import -D ~/Downloads/*.qfx` and shows a fallback notice only when Command
    Mode is unavailable.'
decisions:
  memory_record:
    ask: Add a decisions:host-file-operands-expand-in-bob record for bob-side ~/glob
      expansion?
    memory:
    - decisions:host-file-operands-expand-in-bob
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.5j.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 14:47:04
status: wip
bead_id: bob-cli-no
---

- **PROMPT:** [prompts/202611/bank_import_delete.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_import_delete.md)
- **BEAD:** [bob-cli-no](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-no/README.md)

# bob bank import -D: import the week's downloads, then delete them

## Why this shape

The Monday guide prescribes `bob bank import ~/Downloads/*.qfx && rm ~/Downloads/*.qfx`
because the QFX holds the full account number. That ritual has two problems. It cannot
run in Command Mode, which has no shell. And `rm` expands the glob a second time, so a
file that arrives during the import is deleted without ever being imported. `-D` moves
the cleanup into bob, the only component that knows exactly which files it recorded. The
research report `research:202611/bank_import_delete_flag/bank_import_delete_flag.md` is
the design basis. Bryan accepted its requirements. Read it with
`sase artifact read file:explicit:6333872361985e792db63a60 "<reason>"` if you need the
full rationale.

**The flag alone does not deliver the requested command.** Command Mode spawns `bob`
directly (`spawn(path, args)`; `cmTokenizeBobArgv` handles quotes only). So
`~/Downloads/*.qfx` reaches bob as a literal string, and `execute_import`
(`src/native/bank/cli.rs`) turns FILE operands into `PathBuf`s with no tilde or glob
expansion. Phase `expand` fixes that inside bob, so shells, scripts, and Command Mode
behave the same. RECENT also keeps storing the _pattern_, so next Monday's replay
re-expands against the new downloads.

Import is already all-or-nothing. Every source parses before the lock, and a single bad
member refuses the whole batch with no vault, journal, Git, or sync change. A successful
apply is one guarded transaction with one scoped commit. So "delete iff all imports were
successful" maps onto one boolean: `report.failure.is_none()` after a real
`apply_with_restore`. Deletion always happens **after** that transaction, outside the
vault lock. It is never part of `planned_txn`/`run_locked` or their rollback set.

### Design calls (final; workers do not reopen them)

| Question                                  | Call                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Spelling                                  | `-D, --delete`, exactly one spelling, no hidden aliases. On `bank import`, `-D` is free. Misuse such as `-D today` fails safe: `today` becomes a FILE, the read fails, exit 2, nothing written or deleted.                                                                                                                                                         |
| What gets deleted                         | Every unique source bob parsed in the batch: QFX, OFX, and the Wells Fargo CSV profile alike.                                                                                                                                                                                                                                                                      |
| "Successful"                              | `failure.is_none()` after a real apply. This **includes** the all-seen replay ("No changes; no import commit."). The leftover re-downloaded `Checking1 (1).qfx` case is the point. It also includes the post-sync-failure success, because the local commit is durable. It **excludes** every refusal (exit 2), every runtime failure (exit 1), and every dry run. |
| Unlink vs Trash                           | Permanent `fs::remove_file`. Trash would keep the account-number PII on disk. The journal (with `-F`) restores rows; nothing restores the QFX. Docs must say so plainly.                                                                                                                                                                                           |
| Cleanup failure after a successful import | Exit 1, top-level `ok: false`, error code `bank_delete_failed`. The import summary and commit sha still print. Re-running the same command is a seen no-op that retries only the cleanup.                                                                                                                                                                          |
| `-D` with no FILE                         | Refused, exit 2. This includes `-D -F` with no FILE. bob never scans for files the user did not name.                                                                                                                                                                                                                                                              |
| Pattern case                              | Case-sensitive, matching zsh. `*.qfx` does not match `.QFX`.                                                                                                                                                                                                                                                                                                       |
| Preview/apply drift                       | Accepted. A file arriving between Enter and Ctrl+Enter is still fully parsed, planned, verified, and committed before deletion. No fingerprint protocol.                                                                                                                                                                                                           |
| Glob engine                               | Add the rust-lang `glob` crate (`glob = "0.3"`, no dependencies). Do not hand-roll a matcher.                                                                                                                                                                                                                                                                      |

Release train rules (`decisions:bob-cli-releases-ride-a-daily-train`): never edit the
`Cargo.toml` version or `CHANGELOG.md`. Every change here is additive, so commit
subjects are `feat(bank): …` / `feat(catalog): …` with **no** `!`. Adding the `glob`
dependency to `Cargo.toml` `[dependencies]` (and `Cargo.lock`) is fine.

Every bob-cli phase gates on `just check`. Phases that touch docs or help also run
`just docs-gen` and then `just docs-check`. Follow the `cli_rules` and `docs` reference
memory: help must be excellent, options alphabetical, and every long option needs a
short alias.

## Phase `expand`: bob expands `~` and glob patterns in `bank import` FILE operands

### Shared helper: `src/native/host_paths.rs` (new, registered in `src/native/mod.rs`)

This is one small module with no bank knowledge, so `attachments import` and
`ref create` can adopt it later.

`pub(crate) fn expand_operands(operands: &[OsString]) -> Result<Vec<PathBuf>, OperandError>`

For each operand, in argv order:

1. **Tilde.** Expand a leading `~` / `~/` with `crate::native::env::expand_tilde`,
   exactly like a shell. No `~user`, `$VAR`, brace, or command substitution.
2. **Literal wins.** If the tilde-expanded path exists (`symlink_metadata` succeeds),
   use it as-is. A real file named `statement [1].qfx` is never reinterpreted as a
   pattern, and shell-expanded arguments behave exactly as before.
3. **Pattern.** Otherwise, if the path is valid UTF-8 and contains `*`, `?`, or `[`,
   expand it with `glob::glob_with` using
   `MatchOptions { case_sensitive: true, require_literal_separator: true, require_literal_leading_dot: true }`.
   This means no recursion across `/`, no `**` semantics beyond the crate's, and hidden
   entries only when the pattern component starts with `.`. Keep only matches where
   `Path::is_file()` holds, so directories are skipped. Matches keep the crate's sorted
   order. Argv order is preserved across operands.
   - A `GlobError`, such as permission denied while listing `~/Downloads`, is **not**
     "no match". Return `OperandError::Unreadable { path, error }`, so macOS privacy
     denials say what actually happened.
   - A valid pattern that matches no file returns `OperandError::NoMatch { operand }`,
     carrying the operand **as typed**.
   - An invalid pattern (`glob::PatternError`, such as an unclosed `[`) falls back to
     the literal path, which then fails as today's ordinary read error.
4. **Anything else** (no glob characters, or non-UTF-8) passes through literally. A
   missing literal path keeps today's `could not read source` error.
5. **Dedupe** the final list by `fs::canonicalize` (falling back to the path itself when
   canonicalization fails). The first appearance wins, so overlapping patterns or a
   repeated FILE neither parse twice nor, in phase `delete`, delete twice. This applies
   with or without `-D`.

`pub(crate) fn display_path(path: &Path) -> String` returns an absolute path under
`$HOME` as `~/…`. Any other path (including relative ones) is returned unchanged (lossy
UTF-8). Phase `delete` uses it for every human-facing path. Several modules carry ad-hoc
copies of this logic (`randomize.rs`, `gkeep/list.rs`, …). Leave them alone; do not
refactor them here.

`pub(crate) fn permission_hint() -> &'static str` returns the shared hint for an
`io::ErrorKind::PermissionDenied` on a host file:
`bob cannot access this file; on macOS, allow the app running bob (Obsidian or your terminal) to use this folder in System Settings → Privacy & Security → Files and Folders`.
Keep it short and platform-honest. Wording may be tightened, but keep the System
Settings path.

Unit tests (in-module, temp dirs) cover:

- tilde expansion with a temp `HOME`
- literal-exists-wins for `statement [1].qfx`
- `*`, `?`, and `[ab]` patterns
- names with spaces
- a directory matched by `*` is skipped
- a hidden file is skipped by `*` but matched by `.*`
- a no-match error carries the typed operand
- an unreadable directory gives `Unreadable` (skip when `geteuid() == 0`)
- an invalid pattern falls back to a literal
- overlapping patterns dedupe with first-wins order
- non-UTF-8 operands pass through (Unix only)
- `display_path` for paths inside and outside `$HOME`

### Adopt it in `bob bank import`

In `execute_import` (`src/native/bank/cli.rs`), replace the `files.map(PathBuf::from)`
collection with `host_paths::expand_operands`. Map errors onto the existing failure path
(`ImportReport::failed` + JSON/human print + exit code), before the lock, sync, or any
parse:

- `NoMatch` → new code `bank_import_no_match` (exit 2 via the existing `bank_import_`
  prefix). Message: `no files match ~/Downloads/*.qfx`, showing the operand as typed.
  Hint: `download an export first, or check the pattern`.
- `Unreadable` → `bank_import_source`, message
  `could not read ~/Downloads: Permission denied (os error 13)`. Use `display_path` for
  the path. On `PermissionDenied`, the hint is `permission_hint()`.

In `import_run::prepare_sources`, when `fs::read` fails with `PermissionDenied`, set
`SourceError.hint` to `permission_hint()` instead of the generic
`fix the source file and retry`.

Labels stay `path.to_string_lossy()` of the expanded path. That is the same string a
shell expansion produced before, so JSON `files[].file` does not change. Labels are
never journaled. Verify this: the journal keeps `file_sha256`, not paths.

Help (`build_cli_import`): append to `long_about`: _"FILE accepts `~` and glob patterns
(`_`, `?`, `[...]`), which bob expands itself, so callers without a shell, such as command mode, can pass `~/Downloads/_.qfx`;
an existing file always wins over pattern interpretation, and a pattern that matches
nothing is refused."_ Update the FILE help to
`QFX/OFX or the supported Wells Fargo CSV export; repeatable; ~ and glob patterns expand`.

### Command Mode surfaces

- `src/native/catalog/table.rs`: add `"bank"` to `KNOWN_INVALIDATES`. Set
  `invalidates: &["bank"]` on the `bank-import`, `bank-ack`, and `bank-ensure` rows
  (`Row { invalidates: &["bank"], ..writes(…) }`). bob-ledger-tools already maps the
  `bank` tag to `apiBankInvalidate("command-mode")` on `bob-command-mode:run-finished`,
  and the event fires for every outcome. But no row emits the tag today, so a Command
  Mode import never refreshes Bank Badges, the queue card, or `]b` state. Add
  `"downloads"` to the `bank-import` keywords.
- `docs/command-catalog.md`: the `invalidates` row is already stale (it lacks `gkeep`).
  List every `KNOWN_INVALIDATES` value, including `gkeep` and `bank`.
- `src/native/completion/kinds.rs`: change the `["bank", "import"]` / `files` entry to
  `Kind::Files(Some("*.qfx *.ofx *.csv"))`, so zsh/bash Tab offers exports. Update any
  completion test that pins the kind.

### CLI tests (`tests/cli/`)

Prefer a new focused module, such as `tests/cli/bank_import_files.rs` registered in
`tests/cli/main.rs`, for this phase's and phase `delete`'s import-operand tests. Share
the `BankVault` / `bank_command` / `ofx` helpers from `tests/cli/bank.rs` by moving them
into a small shared module rather than copying them. If that extraction is
disproportionate, extend `tests/cli/bank.rs` instead. `bank_command` already points
`HOME` at a temp `home-dir`, so the Command Mode shape is a literal argv
`["import", "-a", "checking", "~/Downloads/*.qfx"]`, with QFX files written under
`$HOME/Downloads/`.

Pin these:

- a literal `~/Downloads/*.qfx` imports every match in sorted order
- the literal-exists-wins case
- a zero match exits 2 with `bank_import_no_match` in JSON, and the vault, HEAD, and
  journal stay untouched (no lock or sync attempted)
- overlapping patterns parse each file once (counts are not doubled)
- the catalog JSON shows `invalidates: ["bank"]` on the three rows
  (`tests/cli/catalog.rs`)

### Memory

> [!decision] memory_record When accepted, use `/sase_memory_write` to add one decision
> record, `decisions:host-file-operands-expand-in-bob`. Claim: bob, not its callers,
> expands `~` and glob patterns in host-file operands, literal path first, because
> Command Mode spawns bob without a shell. Plugin-side expansion was rejected: the
> binary is the source of truth, expansion is unsafe for non-path args, and RECENT would
> store stale expanded names. Cite the research report and this phase's commit. When
> declined, record nothing (a human reviewer declined it).

Record a `PROPOSED FOLLOW-UP:` note on this phase's bead. Follow-up: adopt
`host_paths::expand_operands` in `bob attachments import` and `bob ref create`, the
other host-file operands Command Mode cannot glob.

## Phase `delete`: the `-D/--delete` flag

### CLI contract

```text
bob bank import [FILE...] [-a ALIAS] [-b DIR] [-D] [-d] [-F] [-f human|json]

  -a, --account <ALIAS>  Choose a configured importable account
  -b, --bob-dir <DIR>    Bob vault root; defaults to BOB_DIR or ~/bob
  -D, --delete           Delete each FILE after the whole batch is recorded
  -d, --dry-run          Preview diffs without writing
  -F, --force            Restore journaled transactions whose rows are missing
  -f, --format <FORMAT>  Output format: human or json [default: human]
  -h, --help             Show help
```

- Add a `delete_arg()`
  (`Arg::new("delete").long("delete").short('D') .action(ArgAction::SetTrue)`). Declare
  it between `bob_dir_arg()` and `dry_run_arg()` so help lists `-a -b -D -d -F -f -h`.
  Do **not** use clap `requires`: the no-FILE refusal must go through `ImportReport` so
  `-f json` gets a JSON envelope.
- `-D` with an empty FILE list, with or without `-F`, is refused before anything else
  with `bank_import_missing_files` (exit 2). Message:
  ` -D needs at least one FILE to delete`. Hint:
  `` name the exports to import and delete, e.g. `bob bank import -D ~/Downloads/*.qfx` ``.
- `long_about` gains: _"`-D` permanently deletes every FILE only after the whole batch
  is recorded (committed, or already seen) and only while the file still matches what
  bob read; a refused or failed import keeps every file, and a file bob could not delete
  is reported and exits 1."_
- `after_help` examples become: `bob bank import ~/Downloads/*.qfx`,
  `bob bank import -D ~/Downloads/*.qfx`, `bob bank import -d ~/Downloads/checking.qfx`,
  `bob bank import -a checking -f json ~/Downloads/export.csv`, `bob bank import -F`.
- `src/runner.rs` `bob bank` group examples: replace
  `bob bank import ~/Downloads/*.qfx  Import posted transactions` with
  `bob bank import -D ~/Downloads/*.qfx  Import, then delete the downloads`. Keep the
  column alignment the surrounding lines use.

### Lifecycle and the delete gate

Keep `plan_import` and `import_plan.rs` pure. Put the delete logic in a focused new
module, `src/native/bank/import_delete.rs`. `import_run.rs` is already about 2,100
lines; only touch it for the report field and printing hooks.

1. **Expand** (phase `expand`).
2. **Up-front check (only with `-D`), before parse, lock, or sync.** For each expanded
   path, `symlink_metadata` must say regular file. Symlinks, directories, FIFOs, and
   sockets are refused. The canonical path must not lie inside the canonical vault root.
   A violation refuses the batch with `bank_import_delete_unsupported` (exit 2), naming
   the offending path via `display_path`:
   - `-D deletes only regular files; ~/Downloads/link.qfx is a symlink`, hint
     `pass the real file, or drop -D`.
   - `-D never deletes vault files; ~/bob/inbox/x.qfx is inside the vault`, hint
     `move the export out of the vault, or drop -D`.
3. **`prepare_sources`** as today. `PreparedSource` gains `path: PathBuf`. The digest is
   already in `statement.file_digest` (SHA-256 of the bytes read; reuse
   `identity::sha256_hex`).
4. **Dry run:** plan as today. On success, every file is `would_delete`. Nothing is
   removed. Exit 0.
5. **Apply:** the unchanged `apply_with_restore` / `run_locked` lifecycle, then
   `refresh_accounts` as today.
6. **If the run failed** (`failure.is_some()`, or any earlier refusal): every file is
   `kept` with reason `import_failed`. Print the failure, plus the kept line.
7. **Else, cleanup.** Visit each unique prepared file in batch order and keep going
   after a failure, because one locked file must not strand the rest:
   - `symlink_metadata` NotFound → `deleted`. Something else already removed it, and the
     promise "this download is gone" holds.
   - No longer a regular file, or the re-read bytes' SHA-256 ≠ `file_digest` → `kept`,
     reason `changed_since_read`.
   - `fs::remove_file` error → `kept`, reason `io`, carrying the error text (and
     `permission_hint()` on `PermissionDenied`).
   - Otherwise → `deleted`.

   Write the cleanup as
   `remove_sources_with(sources, remove: impl Fn(&Path) -> io::Result<()>)` so unit
   tests can inject an unlink failure. The public entry point passes `fs::remove_file`.

8. **Exit code.** `ImportReport` gains `delete: Option<DeleteReport>`, which is `None`
   without `-D`. `exit_code()` keeps today's failure mapping. With no failure, any
   `kept` entry gives exit 1. The cleanup failure is **not** a `TxnFailure`: every
   `bank_import_*` code maps to exit 2, and `failure.is_none()` must stay true so the
   import is still reported as committed.

Deletions are never journaled (raw host paths are banned from journal payloads), and the
commit subject does not change. Crash safety needs no new machinery. Before the commit,
nothing was removed. After the commit, a re-run is a seen no-op that finishes the
cleanup.

### Human output (beautiful, through the existing `Styler`, honoring `NO_COLOR` and pipes)

Use `display_path` everywhere a path is shown. Also use it for the per-file block header
in `print_human` (`~/Downloads/Checking1.qfx · checking · …` instead of
`/home/bryan/Downloads/…`). JSON keeps absolute labels.

The delete phrase, shared by every line below:

- one file: `~/Downloads/Checking1.qfx`
- several files with one parent:
  `2 files from ~/Downloads: Checking1.qfx, Checking1 (1).qfx`
- several parents: `2 files: ~/Downloads/a.qfx, ~/tmp/b.csv`

Always list every name. These are destructive actions, and a weekly batch is small.

**Dry run.** Print the line after `Affected:` and before the diffs, so the Command Mode
preview shows it in the summary at the top:

```text
Dry run — nothing written
~/Downloads/Checking1.qfx · checking · 2026-11-01–2026-11-14
  New 9 · matched 23 · seen 41 · ambiguous 0 · before start 4
  …
Affected: .bob/bank/apollo.jsonl, bank/2026-11.md
Would delete 2 files from ~/Downloads: Checking1.qfx, Checking1 (1).qfx
--- a/bank/2026-11.md
…
```

`Would delete` is yellow, marking an attention-worthy destructive step.

**Real run, success.** Print the line after the `Sync:` line, or after
`No changes; no import commit.` for a seen replay, and before warnings and `Next:`. The
last side effect is printed last:

```text
Committed bank: import 9 rows (checking) (1a2b3c4)
Sync: pre ok, post ok
Deleted 2 files from ~/Downloads: Checking1.qfx, Checking1 (1).qfx
Next: file the 9 waiting rows in bank/2026-11.md — ]b walks them oldest first, …
```

`Deleted` is green. Command Mode's completed toast stays the `Next:` line.

**Refused or failed batch (stderr),** inside the existing failure block, after the dim
hint:

```text
✗ bob bank import: <message>
  <hint>
  Kept 2 files — nothing is deleted until the whole batch imports.
```

The kept line is dim. Omit it when no file was expanded, as with `no files match`.

**Cleanup failure after a successful import.** Stdout carries the full normal summary,
including a `Deleted …` line for any files that did go and the `Next:` line. Stderr
starts with the error line, because Command Mode's failed toast shows the first stderr
line:

```text
✗ bob bank import: import succeeded, but could not delete 1 file
  ~/Downloads/Checking1 (1).qfx — changed since bob read it
  Re-run the same command to retry, or delete it by hand — it holds your account number.
```

Per-file reasons render as `changed since bob read it` or the io error text (plus the
permission hint). The `✗` is red and the hint is dim.

### JSON (additive; `schema_version` stays 1)

`"delete": null` without `-D`. With `-D`, it is always an object, including on refusals:

```json
"delete": {
  "files": [
    { "path": "/home/bryan/Downloads/Checking1.qfx", "status": "deleted", "reason": null, "message": null },
    { "path": "/home/bryan/Downloads/Checking1 (1).qfx", "status": "kept", "reason": "changed_since_read", "message": "changed since bob read it" }
  ]
}
```

- `status` ∈ `would_delete | deleted | kept`.
- `reason` (only for `kept`) ∈ `import_failed | changed_since_read | io`.
- `message` is the human detail for `kept`, otherwise `null`.
- `files` is empty when expansion itself failed.
- On a cleanup failure, the top-level `ok` is `false`, `issues.errors` is 1, and `error`
  is `{code: "bank_delete_failed", message, hint}`. `commit`, `files[]`, and `rows[]`
  stay intact.
- Never echo FITID, ACCTID, or raw OFX.

The research sketch had a `requested: true` key. It is dropped as redundant with
`null`-vs-object.

### Tests

**Unit tests (`import_delete.rs`):**

- the delete phrase in all three shapes
- the up-front check: symlink, directory, FIFO (Unix), and inside-vault are refused; a
  regular file outside the vault passes
- cleanup: all deleted; already-gone counts as deleted; a changed digest is kept; an
  injected unlink failure on the second of three files deletes the first and third,
  keeps the second with `io`, and the run continues
- `exit_code()`: 0 when all deleted; 1 with any kept and no failure; the failure codes
  are unchanged

**CLI tests (alongside phase `expand`'s module):**

- Help lists `-D, --delete` in order `-a -b -D -d -F -f -h`. `-D` without FILE, and
  `-D -F` without FILE, exit 2 with the JSON envelope.
- Gate:
  - a real import unlinks every file and prints the `Deleted` line
  - an all-seen replay (no commit) unlinks the re-download
  - a mixed QFX + CSV batch unlinks all of them
  - the Command Mode shape `-D ~/Downloads/*.qfx` with a temp `HOME` unlinks the matches
  - a malformed member, an unresolved account, and a no-match pattern keep every byte
    and print the `Kept` line
  - `-d -D` keeps every byte and fills `would_delete`
- Safety:
  - a symlink operand is refused with exit 2 before any write
  - a file inside the vault is refused with exit 2
  - an unlink failure via a read-only parent directory (skip when `geteuid() == 0`)
    gives a committed import, the file kept, exit 1, `bank_delete_failed`, and the
    `commit` sha present; making the directory writable again and re-running the same
    command is a seen no-op that deletes it with exit 0
- JSON: `delete` is `null` without the flag, and the documented object shape with it.

### Docs

- `docs/bank.md`:
  - Add `[-D]` to the synopsis.
  - Add an exit-code bullet: `import -D` exits 1 when the import succeeded but a file
    could not be deleted.
  - Under **Import**, add a short `### Deleting imported downloads (-D)` subsection. It
    covers the gate (recorded batch, digest match, files outside the vault only), that a
    seen replay counts, that deletion is permanent and never journaled (`-F` restores
    rows, never the QFX file), the retry story, and the JSON `delete` object.
  - Extend the human-output paragraph with the `Would delete` / `Deleted` / `Kept`
    lines.
  - Phase `expand` adds a **File patterns** paragraph here first; extend it rather than
    duplicating it.
- `docs/guides/acknowledge-bank-transactions.md` §1: replace the `&& rm` command with
  `bob bank import -D ~/Downloads/*.qfx`. Keep the account-number reason as _why_ `-D`
  exists. Say it also works from Command Mode (Enter previews the diffs and the
  would-delete list; Ctrl+Enter imports and deletes). Replace the sample output block,
  which has drifted from what `print_human` prints: regenerate it from a real run
  against a test vault so it matches today's shape, including the `Deleted` line.
- `just docs-gen`, then `just docs-check`. `docs/reference/cli/bank.md` regenerates from
  clap.

## Phase `card`: the bank card's weekly-import chip (bob-plugins)

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and
read its `AGENTS.md`. `bob-ledger-tools` uses the committed source build: edit `src/`
fragments, run `npm run build`, never hand-edit `main.js`, and keep each fragment ≤ 1000
lines.

Today, `bankCardRenderQueueFooter`
(`plugins/bob-ledger-tools/src/241m-plugin-bank-cards.js`) renders a static muted
`<p class="bob-bank-import-command">Weekly import: bob bank import ~/Downloads/*.qfx</p>`.
Make it the one-click entry to the Monday ritual:

- Render `Weekly import` as a label, followed by a focusable `<button>` chip whose text
  is the exact command, `bob bank import -D ~/Downloads/*.qfx`. Give it monospace text,
  the muted color at rest, and `--text-accent` on hover and focus, consistent with
  `.bob-bank-teaching code`. Set
  `aria-label="Preview the weekly import in command mode"`. Wire it through
  `bankCardListen` so it is cleaned up on unload.
- Click (or Enter/Space) calls
  `api.commandMode.preview(["bank", "import", "-D", "~/Downloads/*.qfx"])` on the
  `bob-command-mode` plugin. Discover it the same way `keepInboxActionCommandMode` does
  in `295-keep-inbox-actions.js`; factor a shared helper if that stays small. Command
  Mode then shows the diffs and the `Would delete …` line, and Ctrl+Enter applies. The
  `bank` invalidation tag from phase `expand` refreshes the card afterwards.
- Fallback: only when Command Mode is missing, or the call throws, show
  `new Notice("Run: bob bank import -D ~/Downloads/*.qfx")`. When the preview _ran_ and
  bob refused (for example `no files match`), Command Mode already shows that result, so
  do not add a second notice. Check what `preview()` resolves with in `bob-command-mode`
  before deciding how to tell these cases apart.
- Tests in `scripts/test-ledger-tools-bank-cards.cjs`: replace the current
  `/Weekly import: bob bank import/` text assertion with assertions for the label and
  the chip text, that a click calls preview with that exact argv, the fallback notice
  when Command Mode is absent, and no notice when preview resolves with a refused run.
- `npm run build`, `npm test`, and `npm run docs:check` (update any bob-ledger-tools doc
  that describes the bank card footer). Then run `bob plugins sync` to deploy, following
  the repo's "When sync refuses an overwrite" rules and never passing `-F` on your own.
- Live check, when a vault and desktop are available: the chip opens a Command Mode
  preview listing the would-delete files. This needs a `bob` built from master with
  phases `expand` and `delete`. If the installed `bob` predates them, the preview shows
  bob's own refusal, and that is acceptable.
