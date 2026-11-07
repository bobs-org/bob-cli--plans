---
tier: tale
size: medium
title: "Vault sync: stop losing tasks to same-point insertion conflicts"
goal:
  Recover the tasks two sync conflicts dropped, and make vault-sync union same-point
  Markdown insertions instead of discarding the local file.
decisions:
  restore_october_task:
    ask:
      Also restore the Oct 23 highlight task the same bug dropped from sase.md, as an
      open task?
    default: true
    why:
      Sync dropped it without Bryan's input; restoring it puts the keep-or-cancel choice
      back with him
    answer: true
  notify_on_quarantine:
    ask:
      Send a desktop notification when a conflict still falls back to remote-wins
      quarantine?
    default: true
    why: Both losses so far went unnoticed until a task was found missing
    answer: true
  copy_on_auto_merge:
    ask: Also write a _conflicts/ copy of the local file when a conflict is auto-merged?
    default: false
    why: The invariant checks only accept unions that keep every line from both sides
    answer: false
  memory_obsidian_conflicts:
    ask:
      Update obsidian.md's Conflicts bullet to describe the same-point insertion
      auto-merge?
    memory:
      - obsidian.md
    default: true
    corrects:
      "**Conflicts:** supported conflicts keep the remote version in place and preserve
      local copies under `_conflicts/`, logged in `_conflicts/sync_conflicts.md`."
    answer: true
proposed_by: bbugyi200.apollo.cx
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 14:02:45
status: wip
---

# Vault sync: stop losing tasks to same-point insertion conflicts

## Background (what happened)

At 13:45:45 on 2026-11-07 the MacBook's vault-sync committed `a8de70d43`. In `bob.md`,
that commit:

- added `- [*] #task Change parent project of \`^gtd\` daily task to [[gtd]]! ...
  ^gtd-parent-is-gtd`plus three child lines, right after the`^successor-deps` block's
  ROTTEN LOG;
- applied the Task Merge of `^grey-gtd-daily` into it (`[?]` → `[-]`,
  `[cancelled:: 2026-11-07]`, `🪢 **LINKS:** merged into [[#^gtd-parent-is-gtd]]`);
- bumped the frontmatter counts.

At the same moment apollo's `bob ref create` (`bc9fe10d8`) appended
`- [ ] #task #ref [[ref/chat/gmail_thread_note_sync|...]] ... ^ref-gmail-thread-note-sync`
at the same spot. Git's merge (`ae99648c0`) reported a `UU` conflict whose only
conflicting hunk was two **pure insertions at the same point** (diff3 base section
empty). `resolve_conflicts` in `src/native/vault_sync.rs` then ran
`git checkout --theirs` on the **whole file**. That dropped the new task, the merge
cancellation of `^grey-gtd-daily` (a hunk that did not even conflict), and the count
bump. The full local file now sits in
`_conflicts/bob.kellys-macbook-pro-local-2026-11-07T134547-0500.md`. Today's daily note
`2026/20261107.md` still has a dangling Task Link `[[bob#^gtd-parent-is-gtd]]` under the
13:50 BUILD Pomodoro.

`_conflicts/sync_conflicts.md` shows exactly one other conflict, on 2026-10-23 in
`sase.md` (merge `ead6bb599`), with the same shape. A Mac-side highlight task
(`- [ ] #task Use “Paste-ready block” for tasks?! ... [h:: 98b0c781…]`) and a remote
`bob ref create` reading task were both appended at the end of the same open-task list.
The highlight task was never restored: it exists only in
`_conflicts/sase.kellys-mbp-2026-10-23T224729-0400.md`.

So every conflict logged so far is a same-point double insertion, typically a human
capture on the Mac racing a server-side `bob ref create` append. A union of the two
insertions resolves each one without data loss.

## Part 1: recover the lost vault content

Do this from the vault host you run on (`~/bob`) with plain file edits. The Mac edits
this vault live, so keep the window short: run `bob vault-sync`, make the edits, then
run `bob vault-sync` again right away. After a few minutes, re-check
`_conflicts/sync_conflicts.md` for any new entry. The code fix is not deployed yet, so a
new conflict would still quarantine.

1. **`bob.md`: restore `^gtd-parent-is-gtd`.** Insert these four lines, copied
   byte-exact from lines 27–30 of the bob.md conflict copy, directly above the
   `^ref-gmail-thread-note-sync` line, where the local side had them:
   ```
   - [*] #task Change parent project of `^gtd` daily task to [[gtd]]! [created::2026-11-07] ^gtd-parent-is-gtd
   	- 🪢 **LINKED FROM:** [[#^grey-gtd-daily]] (merged into)
   	- Make `gtd_daily` show as grey!
   		- IMPL: Add support for `color` property on area/project notes!
   ```
   Indentation is tabs. Copy from the conflict file rather than retyping.
2. **`bob.md`: re-apply the merge cancellation of `^grey-gtd-daily`.** Make the block
   match lines 109–112 of the conflict copy: the task line becomes `- [-] ...` with
   `[cancelled:: 2026-11-07]` inserted after `[scheduled:: 2026-11-08]`. Its
   `\t- IMPL: ...` child is replaced by
   `\t- 🪢 **LINKS:** merged into [[#^gtd-parent-is-gtd]]`, and the SCHEDULE LOG lines
   stay unchanged.
3. **`bob.md` frontmatter:** set `task_count` / `open_task_count` to match the restored
   content. Confirm with `bob projects --json` or the project scan in
   `src/native/projects/scan.rs`. The expected values are 60 / 6, but trust the scan.
4. **`sase.md` (decision `restore_october_task`):**

   > [!decision] restore_october_task Copy line 30 of
   > `_conflicts/sase.kellys-mbp-2026-10-23T224729-0400.md` (the
   > `Use “Paste-ready block” for tasks?!` line with its `[h::]` hash) byte-exact into
   > sase.md's top open-task list, as the last line before `### Next & In Progress`.
   > Then update that note's frontmatter counts the same way as step 3. Check first that
   > the line is not already present anywhere outside `_conflicts/` (grep for
   > `98b0c781a40faa60`).

   > [!decision] restore_october_task = no Skip this step.

5. Leave both `_conflicts/` copies and `sync_conflicts.md` untouched as history.
6. Verify the recovery:
   `grep -rn 'gtd-parent-is-gtd' ~/bob --include=*.md | grep -v _conflicts` must show
   bob.md (task line plus the LINKS back-link) and the daily-note Task Link. Also run
   `rg -n '<<<<<<<|>>>>>>>' ~/bob/bob.md ~/bob/sase.md`, which must find nothing, and
   `git -C ~/bob status --short`, which must be clean after the sync. Then
   `git -C ~/bob log -1` must show your sync commit pushed
   (`bob vault-sync status --json`).

## Part 2: fix the resolver (bob-cli)

### Behavior

For a `UU` conflict on a **`.md`** path that has all three index stages, first try a
_same-point insertion union_. Only when that is refused, fall back to today's whole-file
remote-wins plus `_conflicts/` quarantine. `AA`, `UD`/`AU`, `DU`/`UA`,
`.canvas`/`.base`, and binary handling do not change. (`.canvas` is JSON, and a line
union can corrupt it.)

The union is accepted only when **every** check below passes. Any failure, or any
unexpected git or parse error, returns "not unionable" with a reason string. The caller
then quarantines exactly as today and puts the reason in the warning.

1. Run
   `git merge-file -p --diff3 --object-id --marker-size=<M> -L local -L base -L remote <stage2> <stage1> <stage3>`.
   All hosts run git ≥ 2.43 (apollo 2.43.0, athena 2.47.3, Mac 2.50.1), so `--object-id`
   is available. Use a large `M` (for example 41). Refuse the union if any of the three
   blobs has a line starting with `M` copies of `<`, `|`, `=` or `>`. A non-negative
   exit status is the conflict count; a negative status (or a signal) is an error. All
   three blobs and the output must be valid UTF-8.
2. Parse the output strictly into clean segments and conflict blocks
   `{local, base, remote}`. Refuse on any malformed or unterminated marker. Run the
   parse and every check below on a pure, unit-tested function over strings, with no git
   or filesystem access.
3. Every conflict block must have an **empty base** section, with non-empty local and
   remote sides.
4. No block may start inside the leading YAML frontmatter (the region between the
   opening `---` on line 1 and its closing `---`), because a union there can duplicate
   keys.
5. Each side of each block must have balanced code fences: an even count of lines whose
   trimmed start is a ` ``` ` or `~~~` fence.
6. The two sides of a block must share no identical non-blank line. This catches both
   hosts writing the same Task Link, Pomodoro entry, or task, where a union would
   duplicate it.
7. **Outline-safe order.** Let `indent` be the leading-whitespace width (tab = 4
   columns), skipping blank lines. "X then Y" is safe when every non-blank line of X has
   `indent >= indent(first non-blank line of Y)`, which keeps Y's lines attached to the
   same parents they had in Y's own version. Try local-then-remote first, then
   remote-then-local, and refuse if neither order is safe. Also, if the first non-blank
   line after the block is indented (`k > 0`), refuse when either side has a line with
   `indent < k`, since such a line would re-parent that trailing line.
8. **Block-id invariant** on the assembled result: for every Obsidian block id (a
   `^[A-Za-z0-9-]+` token ending a line), its count in the result must be ≤ its max
   count in the local and remote blobs. This catches two sides minting the same id.

On success, write the assembled text to the working-tree path and stage it, as the
existing code does with `git_add_path`. Append a line to `_conflicts/sync_conflicts.md`:
`- {timestamp} {path} auto-merged {n} same-point insertion hunk(s); nothing dropped`.
Print a non-red `print_log`/`warn` line, and push the path onto a new status-record
field (below).

> [!decision] copy_on_auto_merge = no Write no `_conflicts/` copy for an auto-merged
> path.

> [!decision] copy_on_auto_merge Also write the local stage with the existing
> `write_conflict_copy` and list it in `conflicts` as today.

Several conflicted paths in one merge are each decided on their own. Some can auto-merge
while others quarantine, and the single `git commit --no-edit` that follows covers both.

**Rejected alternatives** (record these in the doc change, briefly):
`git merge -X theirs` and the global `merge=union` gitattribute. The first keeps local
non-conflicting hunks but still drops the conflicting insertion, leaving half-applied
gestures (here a `merged into` link to a task that would not exist). The runbook already
forbids `-X ours/theirs`. The second applies union even to modify/modify hunks, keeping
both old and new versions of an edited line. Partial-hunk remote-wins in the fallback
was also rejected: the whole-file quarantine keeps the conflict copy a complete,
restorable local version.

### Notification (decision `notify_on_quarantine`)

> [!decision] notify_on_quarantine When the fallback quarantines any `UU`/`AA` path,
> send one desktop notification per cycle through the existing `BOB_VAULT_SYNC_NOTIFY`
> gate and sender. Rename `clobber_notifications_enabled` / `send_clobber_notification`
> to sync-generic names and keep the env var and test default (off under `cfg(test)`).
> The body is `Sync conflict: kept remote <path>; local copy at <copy>` for one path,
> and `Sync conflicts: quarantined N local copies` for several. Auto-merges send no
> notification.

> [!decision] notify_on_quarantine = no Leave notifications clobber-only.

### Code layout

- New module `src/native/vault_merge_union.rs`, declared next to `mod vault_sync;` in
  `src/native.rs`. It holds the pure parse, checks, order selection, and assembly, plus
  a `#[cfg(test)]` unit-test module. Return a small enum or struct such as
  `UnionOutcome::Merged { text, hunks } | UnionOutcome::Refused(reason)`.
- `src/native/vault_sync.rs`: in `resolve_conflicts`' `"UU" | "AA"` arm, for a `UU`
  `.md` path with stages 1/2/3, call a thin git wrapper (`git merge-file`, reusing
  `git_output`/`ChildEnv`) and then the pure function. Fall through to the existing
  quarantine code on `Refused`. Add `auto_merged: Vec<String>` to `StatusRecord` with
  `#[serde(default)]`, cleared per cycle like `conflicts`, and extend the
  `legacy_status_record_defaults_new_fields` test. Show auto-merged paths in
  `print_status_panel` (non-red) and in `--json`.

### Tests

Unit tests in `vault_merge_union.rs`:

- the exact `bob.md` shape from Background, which unions local-then-remote;
- the October `sase.md` shape (one line each);
- local adds a top-level task while remote adds an indented child of the preceding task
  at the same point, which picks remote-then-local;
- a case where neither order is safe;
- the indented trailing-line re-parent refusal;
- refusals for: non-empty base, a block in frontmatter, an unbalanced fence, a shared
  identical line, a duplicate block id, and marker-like content;
- CRLF content round-trips unchanged.

Integration tests in `tests/cli/vault_sync.rs`, modeled on
`vault_sync_same_line_edit_quarantines_local_copy_and_keeps_remote` with
`init_vault_sync_pair`:

- same-point insertions auto-merge: both blocks present, no `_conflicts/` copy, the log
  line is written, the status `auto_merged` field lists the path, and the remote has the
  merge;
- a file with one unionable hunk **and** one modify/modify hunk still quarantines the
  whole file (the copy has all local content, and remote wins in place);
- a `.canvas` both-insert still quarantines;
- two paths in one merge, one auto-merged and one quarantined, end in one commit;
- the existing quarantine, binary, ignored-path, and delete/modify tests stay green.

### Docs and memory

- `docs/vault-git-sync.md`: update **Sync cycle** step 6 and **Conflict policy** to say
  that same-point insertions in Markdown are unioned in place in an outline-safe order
  and logged as `auto-merged`, that everything else keeps remote-wins plus quarantine,
  and that a quarantine now notifies. Add a short "lost content after a conflict"
  recovery note: the copy is the full local file, so diff it against the note and
  re-apply the local-only hunks (avoid copying it back wholesale once remote edits have
  landed). In the status section, name the `auto_merged` field. Run `just docs-check`.
- Memory (decision `memory_obsidian_conflicts`): rewrite the **Conflicts** bullet of
  `sase/memory/obsidian.md` to say that same-point Markdown insertions merge in place,
  that other supported conflicts keep the remote version and preserve local copies under
  `_conflicts/`, and that both are logged in `_conflicts/sync_conflicts.md`. Then run
  `sase memory init`. Skip this when the decision is off.

### Gates and release

- Run `just fmt`, `just lint`, and `just test` (or `just check`), and `just docs-check`.
- Use the commit type `fix(vault-sync): ...` (no `!`: no CLI or JSON contract breaks;
  the new status field is additive). Per the release-train decision, do **not** touch
  the `Cargo.toml` version or `CHANGELOG.md`. The fix reaches the hosts with the next
  daily release, and conflicts keep the old behavior until each host upgrades.
