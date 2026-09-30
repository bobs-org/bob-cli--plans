---
tier: tale
title: Remove the 73 recently added
goal: The 73 identified Bob vault tasks no longer carry
size: medium
proposed_by: bbugyi200.apollo.3e
create_time: 2026-09-30 07:57:53
status: wip
---

# Remove the 73 recently added `#now` task tags

## Goal and scope

Remove the exact `#now` token from the 73 affected task lines in Bryan's default Bob
Obsidian vault. Preserve every task's checkbox/status, `#task` tag, description, inline
fields, block ID, whitespace outside the removed ` #now` token, line ending, and
surrounding note content. Do not remove `#now` from prose, code blocks, frontmatter,
other task populations, or a different token such as `#now/subtag`. This is a vault data
correction; no bob-cli product change is needed.

## Evidence available at planning time

On 2026-09-30, `bob query --format json --tasks 'tags include #now'` returned exactly 73
tasks in nine notes: `sase.md` 58, `bob.md` 6, `sase_memory.md` 3, and one each in
`sase_clean.md`, `sase_agent_history.md`, `sase_goals.md`, `sase_pager.md`,
`sase_remote.md`, and `sase_art_links.md`. All 73 task records had `tags == ["#now"]`
and their `originalMarkdown` had exactly one literal `#now` sequence. Forty-eight were
In Progress and 25 were On Hold; removal must not alter those states.
`bob vault-sync status --json` reported a successful sync with matching local and remote
SHA and no conflicts at planning time. Treat these as a scope baseline, not as a
guarantee that the vault will remain unchanged during approval.

## Implementation

1. Before writing, re-run the read-only Tasks query and build an explicit manifest keyed
   by vault-relative note path plus task block ID and current line content (retain line
   number as a diagnostic only). Confirm it still describes the intended 73 rows, with
   one exact `#now` sequence per task. Check for new or changed `#now` tasks and recent
   vault changes. If the target set is no longer unambiguous, stop and report the
   difference rather than applying a broad replacement. Preserve a recoverable preimage
   of only the nine affected notes.
2. Prepare and inspect an in-memory patch that removes only the five characters ` #now`
   from each manifested task line, keeping the following space before its metadata.
   Verify that the patch has exactly 73 such removals in the nine expected notes, with
   no other changed bytes. Do not use a whole-commit Git revert: recent commits or
   pending edits may also contain unrelated work.
3. Apply the reviewed patch to the live default vault while holding Bob's shared
   maintenance lock (`BOB_VAULT_SYNC_LOCK_FILE`, or
   `${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock`) and checking the original note snapshots
   again immediately before each guarded/atomic write. If Obsidian or another process
   changed any target note, abort and rebuild the manifest and patch from current
   content; never overwrite a concurrent edit. Limit writes to the nine manifested
   notes. Release the lock when the writes finish.
4. Let the vault's normal Git sync reconcile and publish the scoped note changes. Verify
   sync status and inspect any conflict copies before declaring success. Do not manually
   commit unrelated vault files.

## Verification and recovery

- Compare the resulting 73 task lines with the manifest: each differs only by deletion
  of ` #now`; all statuses, IDs, fields, and other text remain identical. Confirm no
  other lines or notes changed as part of this operation.
- Re-run `bob query --format json --tasks 'tags include #now'`; the targeted cohort
  should be absent. If a new unrelated `#now` task appeared meanwhile, identify it
  separately instead of removing it automatically. Confirm the task counts and status
  distribution outside the tag are unchanged, and check `bob vault-sync status --json`
  for a successful sync without conflicts.
- If validation fails, restore only the affected task lines from the saved preimages
  under the same lock and snapshot checks, then sync again. Preserve any intervening
  user edits; do not reset the vault or revert an entire mixed-purpose commit.

## Done when

The 73 identified task lines no longer carry `#now`, the vault notes have no unintended
changes, and vault sync reports a healthy state.
