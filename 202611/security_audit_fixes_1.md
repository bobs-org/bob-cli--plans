---
tier: epic
title: Security and bug audit fixes
goal: 'The 19 verified findings from the five-area security/bug audit of bob-cli are
  fixed with regression tests: git pathspec globbing can no longer revert the wrong
  vault file, calendar titles can no longer inject tags/links, the web clipper''s
  private- network guard holds across redirects and WebSockets, vault writes are guarded
  and crash-durable, and the small verified defects are corrected.

  '
phases:
- id: git-literal-pathspecs
  title: Literal git pathspecs for vault filenames
  depends_on: []
  size: small
  description: 'git-literal-pathspecs: switch the highlights_ref recovery/mutation
    and collect_done git call sites that pass concrete vault filenames from ob::git_command
    to ob::git_command_literal, reject pathspec magic in recovery''s validated_paths,
    and add glob-filename regression tests.'
- id: gcal-title-hardening
  title: Gcal remote-text trust boundary
  depends_on: []
  size: medium
  description: 'gcal-title-hardening: extend gcal escape_title to neutralize mid-line
    tags, wikilinks, and Markdown links in organizer-authored event titles; fix the
    gog 429 substring misclassification; optionally gate #on/#off clock tags on organizer_is_self
    per the memory decision.'
- id: web-clip-guards
  title: Web-clip network guards and image handling
  depends_on: []
  size: medium
  description: 'web-clip-guards: re-validate the final navigation URL and redirect
    chain against host_is_global, block unguarded WebSocket handshakes, parse img
    attributes positionally so data-bob-origin query strings cannot drop images, and
    make the origin re.sub replacement literal.'
- id: durable-vault-writes
  title: Durable and race-free vault writes
  depends_on: []
  size: medium
  description: 'durable-vault-writes: bind bank ensure''s write-guard preimage to
    its single planning snapshot, install clip attachments via hard_link/create_new,
    fsync the three non-flushing atomic_write variants and the gkeep token write,
    and verify ownership of the shared-tmp script-cache fallback.'
- id: small-audit-fixes
  title: Small verified fixes
  depends_on: []
  size: medium
  description: 'small-audit-fixes: prefix-match the pomodoro OVERDUE status, close
    the perl two-arg open in bob_pomodoro, report damaged albums as io instead of
    note_changed, require FILE args on bank import, and strip migration anchors by
    match span.'
decisions:
  gate_organizer_clock_tags:
    ask: 'Should gcal ignore #on/#off clock tags on third-party-organized invites,
      superseding part of the off-clock record?'
    default: false
    memory:
    - decisions:off-clock-follows-tag-then-note-default
    - decisions
    answer: false
proposed_by: bbugyi200.athena.1bo
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 05:24:34
status: wip
bead_id: bob-cli-kq
---

- **PROMPT:** [prompts/202611/security_audit_fixes_1.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/security_audit_fixes_1.md)
- **BEAD:** [bob-cli-kq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kq/README.md)

# Security and Bug Audit Fixes

A five-agent audit of bob-cli (subprocess/shell integration, network/auth, filesystem
safety, helper scripts, and the five most recent commits) produced 19 verified findings:
four medium-severity security issues, one medium-severity data-clobber bug, and fourteen
low-severity bugs/hardening gaps. This plan fixes 18 of them; the nineteenth (albums
health findings advise a `bob albums sync` repair that does not consume owner-catalog
edits yet) is already owned by the in-flight epic bob-cli-k3.6.1 phase 2 and was
recorded there as a DISCOVERED ISSUE note instead. Every finding below was verified
against the code at commit 3dce8a9e (several empirically, in scratch git repos or by
running the scripts), with file:line references from that commit — anchor on the named
functions if lines have drifted.

Coordination note: epic bob-cli-k3.6.1 (owner album catalogs) has phases in flight that
touch `src/native/albums/`; the one albums-adjacent fix below edits
`src/native/capture/record_note.rs` only, but implementers should still rebase and check
for collisions before landing.

The five phases touch disjoint files, have no dependencies on each other, and can run in
parallel.

Shared constraints for every phase:

- Never edit `Cargo.toml`'s version or `CHANGELOG.md` (release-train rule). None of
  these fixes breaks a CLI or JSON contract, so no `!` commit markers are needed.
- Gate with `just check` (fmt + clippy with the repo's deny-list + full test suite).
- Each fix should land with a regression test where the module has a test harness (all
  the Rust modules do; the web-clip Python scripts have self-test patterns).

## Literal git pathspecs for vault filenames

**Problem (MEDIUM, security/data-loss).** `ob::git_command` deliberately does not pass
`--literal-pathspecs` (see the doc comment in `src/native/ob.rs:26-41`, which says
callers passing real filenames must use `ob::git_command_literal`). Several modules
rolled their own git calls with the globbing builder while passing concrete vault
filenames. Vault filenames legitimately contain `*?[]` (e.g. `What is Rust?.pdf`,
`Screenshot [2].png`). Empirically verified with git 2.47: when the literal path is
absent from the tree, the glob silently matches a sibling —
`git checkout HEAD -- "What is Rust?.md"` with no such file **reverted the sibling
`What is RustX.md` from locally edited content to the committed version, exit 0**.
Mapped onto `restore_from_base()` in the overwrite-recovery flow: an interrupted
`bob ref create` recovery whose intent `touched` list contains a `?`/`[...]` path not
tracked at `base_head` (the `paths_tracked_in_commit` doc anticipates exactly such stale
entries) glob-matches a sibling in `ls-tree`, passes the filter, and `git checkout` then
silently overwrites the sibling note's uncommitted edits — vault data loss during a
recovery that is supposed to be conservative.

**Fix.** Switch these call sites (all pass concrete filenames, never intentional globs)
from `ob::git_command` to `ob::git_command_literal`:

- `src/native/highlights_ref/overwrite/recovery.rs:1902`
  (`git checkout <base> -- <paths>`), `:1957` (`git diff --quiet <base> -- <paths>`),
  `:1986` (`git ls-tree --name-only <base> -- <path>`)
- `src/native/highlights_ref/reference_mutation.rs:121` (`git reset HEAD -- <paths>`)
- `src/native/highlights_ref/overwrite/execute.rs:980`
  (`git ls-files --error-unmatch -- <path>`)
- `src/native/collect_done/git.rs:200, 210, 235` (`git add -- <paths>`,
  `git diff --cached -- <paths>`, `git commit -- <paths>`)

Then audit every remaining `ob::git_command` caller in those four files (and any other
module passing filename pathspecs — `rg 'git_command\b'`) and switch any that pass
concrete filenames; keep `git_command` only for intentional globs such as `*.md`
pathspecs and for `git check-ignore` (which rejects `--literal-pathspecs`).
`src/native/vault_sync.rs` and `ob::commit_paths` already do this correctly and are the
pattern to follow.

Additionally, in `validated_paths` in `src/native/highlights_ref/overwrite/recovery.rs`,
reject pathspec magic (a leading `:`) alongside the existing absolute/`.`/`.git`
rejections, mirroring the guard `src/native/vault_git.rs` applies for `check-ignore`.

**Tests.** A regression test per repaired flow with a tracked filename containing `?` or
`[2]` plus an edited sibling that the old glob would have matched: recovery must leave
the sibling untouched, and the collect_done add/diff/commit trio must stage exactly the
named file.

## Gcal remote-text trust boundary

**Problem A (MEDIUM, security).** `escape_title` in `src/native/gcal/render.rs:162-167`
routes calendar event titles through the shared `escape_task_text`
(`src/native/task_text_escape.rs:20-25`), which only neutralizes `%%`, `::` field runs,
the `#task` token, and a trailing `^block-id`. Mid-line `#tags`, `[[wikilinks]]`, and
`[label](url)` Markdown links pass through intact. Titles are authored by the event
**organizer**, and sync re-applies organizer edits to untouched rows. Verified
consequences: a title renamed to `Budget sync #gtd #pre` writes a daily-note Event Task
row whose `#gtd`/`#pre` tags are parsed as exact task tags
(`src/native/dataview/tasks/task.rs:27-33,613-617`) and enroll the row in the PRE
checklist tier of every morning review walk via `checklist_from_tags`
(`src/native/freshness/state.rs:324-334`), where it "resolves only by completion". A
title of `[join here](obsidian://...)` renders a clickable arbitrary-scheme link,
bypassing the http/https/mailto/tel allowlist that gcal's own description renderer
enforces (`src/native/gcal/description/bullets.rs:238-255`); `[[Some Note]]` mints real
vault links and backlinks.

**Fix A.** In `escape_title` (after `strip_google_clock_tokens` has consumed the
intended directives), additionally neutralize the mid-line hazards the description path
already handles: escape mid-line `#tags` (reuse the tag-hash escaping used by
`escape_note_inline`), and break `[[` and `](` sequences the way
`escape_note_inline`/`escape_link_text` do, so no tag, wikilink, or Markdown link
survives in a title. Keep this in gcal's `escape_title`, not in the shared
`escape_task_text` (gkeep input is Bryan's own notes by design). Note: rendered titles
change, so previously synced, untouched rows will be rewritten once on the next sync
(one-time journal span churn); verify sync treats them as untouched Google-span updates,
and update the golden expectations in `tests/gcal_*.rs`.

**Problem B (LOW, bug).** `classify_gog_failure` in `src/native/google/gog.rs:614`
treats any failure output containing the substring `"429"` as rate limiting — checked
before the not-found/invalid-grant branches — so an error echoing an id like
`team-429@group.calendar.google.com` is misreported as "Google throttled; wait and run
again", and retry wrappers retry a permanent failure forever.

**Fix B.** Match 429 only as an HTTP-status shape (e.g. `"code":429` / a word-bounded
`429` status token), the way the same function already shapes its 5xx matching. Add a
unit test with "429" embedded in an identifier.

**Decision branch.**

> [!decision] gate_organizer_clock_tags Honor title/description `#on`/`#off` (and
> deprecated off aliases) only when `organizer_is_self` (already computed in
> `src/native/gcal/model.rs:285-288`; tag parsing at `:263-276`). On a third-party
> invite carrying clock tags, ignore them and surface a one-time per-row report,
> following the existing `kept_edit`/`kept_root` reporting pattern. Do NOT touch
> `@route` tokens or description buffer directives — those follow Google by accepted
> decision (`gcal-root-follows-google-until-edited`,
> `buffer-reasons-ride-description-bullets`). Author a new decisions strand under
> `sase/memory/decisions/` recording the organizer gate, mark
> `off-clock-follows-tag-then-note-default` superseded-in-part with `superseded_by` and
> a back-link per the supersession convention, and run `sase memory init`; use the
> `/sase_memory_write` skill before touching the memory files. Rationale for the default
> being off: the exposure is Worked-metrics misclassification only, while a wrong gate
> breaks legitimate invite workflows (e.g. a partner-organized recurring event relying
> on a title `#off`), and the off-clock record explicitly includes Google Calendar clock
> tags with no organizer distinction.

> [!decision] gate_organizer_clock_tags = no No change to clock-tag handling and no
> memory edits; implement only Fix A and Fix B.

## Web-clip network guards and image handling

All in `scripts/web_clip/`. The adapter's in-browser private-network guard
(`_install_guard`, `web_clip_adapter.py:1096-1133`) uses Playwright `context.route()`,
and the repo pins Playwright 1.62.

**Problem A (MEDIUM, SSRF).** `context.route()` is invoked only for the first URL of a
redirect chain (the pinned Playwright docs state this), and `run_capture` never
re-validates `page.url` after navigation (`web_clip_adapter.py:1809`). A clipped URL
answering `302 Location: http://192.168.1.1/admin` sails past the guard: the public
first hop is approved and Chromium follows the redirect internally, so bob snapshots and
archives the internal page into the vault; a hostile page's `<img>` pointing at a
redirecting URL fires blind GET requests into the LAN. The adapter's own `guarded_fetch`
walks redirect hops manually with per-hop `host_is_global` checks precisely because of
this; the browser side has no equivalent.

**Fix A.** After every `goto` (headless and headed retry paths), fail closed when
`host_is_global(urlsplit(page.url).hostname or "")` is false; additionally walk
`nav.request.redirected_from` and refuse chains that touched a non-global host.
Optionally enable Chrome's Private Network Access enforcement flags at launch to reduce
the subresource-redirect case.

**Problem B (MEDIUM, SSRF).** `context.route()` does not intercept WebSocket handshakes
at all; Playwright 1.62 has a separate `context.route_web_socket()` the adapter never
installs. A hostile page can open `ws://10.0.0.5:8080/` (or scan an RFC1918 range)
during the 45s navigation + 20s settle window, reaching internal hosts unchecked.

**Fix B.** Next to each `_install_guard` call, install
`context.route_web_socket("**/*", ...)` closing the socket (or allowing it only after a
`host_is_global` check on `ws.url`).

**Problem C (MEDIUM, bug — reproduced).** `_SRC_ATTR_RE` (`web_clip_render.py:73`)
matches the first `src=` anywhere in the serialized `<img>` tag, and the adapter writes
`data-bob-origin="<original url>"` _before_ the `src` attribute
(`web_clip_adapter.py:1477-1481`). Any original image URL whose query contains `src=` —
the standard image-proxy/CDN shape, `...?src=https%3A%2F%2F...` — is misparsed at render
time: the extracted "src" becomes a fragment of the origin URL, fails the `assets/`
check, and the image is silently deleted from the reader PDF and compose output
(`normalize_images` at `web_clip_render.py:262`, `prefix_part_assets` at `:577`). The
adapter-side `_localize_images` (`web_clip_adapter.py` near line 1408) has the same
first-match weakness when `alt`/`title` text containing ` src=` precedes the real
attribute.

**Fix C.** Parse attributes positionally (reuse the existing `_IMG_ATTR_RE`/`_img_attr`
iteration used by `source_images`) instead of regex-searching the whole tag, in
`normalize_images`, `prefix_part_assets`, and `_localize_images`; or insert
`data-bob-origin` after `src` and anchor the regex on attribute boundaries. The
positional parse is the robust option.

**Problem D (LOW, bug — reproduced).** `web_clip_adapter.py:1478-1481` uses the escaped
origin URL as a `re.sub` _replacement string_; a URL containing `\1` or `\g` in its
query raises `re.error: invalid group reference` after the image was already fetched,
aborting the entire capture as an `internal` error.

**Fix D.** Use a callable replacement (`lambda m: ...`) so the origin URL is inserted
literally.

**Tests.** Follow the scripts' existing self-test patterns: a mocked redirect-to-private
navigation refusal, a WebSocket-blocked case if testable, an `<img>` whose
`data-bob-origin` query contains `src=` surviving render, and a `\1`-bearing image URL
completing capture.

## Durable and race-free vault writes

**Problem A (MEDIUM, data clobber).** `bank ensure`'s planner reads the target month
note twice: `src/native/bank/ensure.rs:40` snapshots `existing` and derives the edited
contents from it, but `:75` fills the write-guard preimage `original` from a **second**
`fs::read_to_string` of the same file. `execute_ensure`
(`src/native/bank/cli.rs:722-731`) then installs via `write_staged_files_guarded`
against that second read. Anything that writes the note between the two reads (Obsidian
sync, `bob vault-sync`, a concurrent maintenance run — `bank ensure` does not take the
maintenance lock) makes the guard compare the disk against the _newer_ text, pass, and
install contents derived from the _stale_ first read — silently reverting the concurrent
edit. This is exactly the clobber class `validate_disk_preimages` exists to refuse.

**Fix A.** Derive `original` and `existed` from the same `existing` snapshot taken at
line 40 and delete the second read. Do not add the maintenance lock (the corrected guard
now refuses concurrent changes, which is the designed behavior). Regression test: mutate
the note between `plan` and `execute`; the guarded write must refuse, not revert.

**Problem B (LOW, race).** `write_new_file_atomically` in
`src/native/capture_clip/persist.rs:106-113` installs clip/snippet attachments with
`if destination.exists() { refuse } ... fs::rename(temp, destination)`. `fs::rename`
replaces existing files, so a file landing between the check and the rename (a second
capture process — the Mac capture app plus a CLI capture both planning `img/photo.png` —
or Obsidian sync) is silently overwritten. The main note writer explicitly closes this
hole with `hard_link` + `create_new` fallback (`install_new_file`,
`src/native/capture/commit.rs:481-507`, including a "plant before install" test).

**Fix B.** Install via `fs::hard_link(temp, destination)` then unlink the temp, with the
`create_new` fallback, exactly as `install_new_file` does; add the analogous
plant-before-install regression test.

**Problem C (LOW, durability).** Three `atomic_write` variants rename without fsync, so
a crash can install an empty/truncated note over good content (the repo's own standard,
stated at `src/native/collect_done/archive.rs:584`, is to flush the temp before the
rename): `src/native/periods/draft.rs:572` (period-note restamps),
`src/native/highlights_ref/io.rs:173` (reference-note and inbound-link rewrites via
`highlights_ref/sync.rs` and `rename/plan.rs`), and
`src/native/completion/manifest.rs:142`.

**Fix C.** In each, write the temp through a `File` with `write_all` + `sync_all` before
the rename (or route the callers through a shared fsyncing `atomic_write`).

**Problem D (LOW, durability).** `src/native/gkeep/login.rs:334` writes the stored token
file with `create(true).truncate(true)` in place; a crash between truncate and write
destroys the credential.

**Fix D.** Temp file (0600) + rename, like every other state writer in the repo.

**Problem E (LOW, hardening).** When `HOME`/`XDG_CACHE_HOME` are unset (stripped
cron/daemon env), the script cache and state home fall back to a _shared_ temp path,
`env::temp_dir().join("bob-cli-cache")` (`src/runner.rs:2112-2122`,
`src/native/env.rs:311-322`). Another local user who pre-created that directory owns the
scripts bob executes; `write_asset`'s content check is TOCTOU, and the next `bob gkeep`
pipes the Keep master token into whatever `gkeep_adapter.py` sits there (the gkeep state
cache under the same fallback also holds full note content).

**Fix E.** Make the fallback per-uid and verified: create the directory `0700`, and
before executing cached scripts (and before writing state) verify it is owned by the
current uid with no group/other write access, refusing with a clear error otherwise.
Apply to both `runner.rs`'s `cache_home` and `env.rs`'s `bob_cli_cache_dir`.

## Small verified fixes

Five independent one-to-few-line fixes, each with a focused test where the module has a
harness:

**A. Pomodoro "done" false positive (LOW).** `src/native/notify.rs:59` and
`scripts/bob_notify:51` decide the Pomodoro is done via a substring match
(`contains("OVERDUE")`) on the whole status line, which embeds the vault task text — an
open task named `Chase OVERDUE invoices` triggers endless critical "Pomodoro is done!"
notifications while the session is still running. Fix: match the prefix —
`starts_with("[OVERDUE")` in Rust, `[[ "${bob_pomodoro_output}" == '[OVERDUE'* ]]` in
the script.

**B. Perl two-arg magic open (LOW, hardening).** `scripts/bob_pomodoro:144-195` reads
the day file with the diamond operator over ARGV, i.e. 2-arg open semantics: a
`BOB_DAY_FILE` ending in `|` executes a command. Script-fallback only
(`BOB_CLI_USE_SCRIPT=1`) and the value is the user's own env, but the fix is one line:
feed the file on stdin (`perl -e '...' < "${day_file}"`) or use 3-arg open.

**C. Damaged album misreported as transient (LOW).**
`src/native/capture/record_note.rs:492` maps `IndexState::Damaged` (persistently
unreadable album folder/member) to `CaptureFailureKind::NoteChanged`, the kind reserved
for "vault changed after planning". Clients treat `note_changed` as transient and retry
(e.g. the gmail save flow, `src/native/gmail/save.rs:382`) — forever, since the
condition is persistent. Fix: return a plain `CaptureError::io(reason)` (an existing
JSON kind, so no contract break) for `Damaged`, keeping `NoteChanged` for the genuine
stale-preimage refusals in the same function.

**D. `bob bank import` with no files silently succeeds (LOW).**
`src/native/bank/cli.rs:165-170` leaves the `files` arg optional and
`prepare_sources(&[])` plans a no-op, so plain `bob bank import` (or a mistyped shell
variable expanding to nothing) takes the maintenance lock, runs a pre-sync, and exits 0
with "No changes; no import commit." — contradicting the command's documented "Refused
input exits 2" contract. Fix: `.required(true)` on the clap arg, plus an empty-list
refusal in `execute_import` for non-CLI callers.

**E. Tracker migration corrupts anchors with trailing whitespace (LOW — reproduced).**
`scripts/migrations/tracker_review_corrections.py:274-284`: selection tolerates trailing
whitespace after `^ref` (`BLOCK_ID` ends `\s*$`) but `complete_line` strips the anchor
only via `head.endswith(" ^ref")`, so `- [ ] ... ^ref␠` gains a completion field and a
_second_ `^ref` while the original anchor becomes mid-line literal text. Fix: strip the
anchor using the `BLOCK_ID` match span (`raw_line[:match.start()]`).
