---
tier: tale
size: medium
title:
  Get bob ref create --listen episodes back into the AntennaPod feed and make a queued
  publish visible
goal:
  Every bob ref create --listen episode reaches the AntennaPod feed again, the three
  queued episodes are published, and a queued publish never passes silently in bob.
decisions:
  token_source:
    ask:
      Should apollo, the feed host, read its feed token from a mode-600 file instead of
      decrypting it with pass?
    choices:
      file:
        Copy the token once from athena's pass to ~/.sase on apollo; publishing survives
        gpg-agent restarts
      pass:
        Keep pass; Bryan unlocks apollo's gpg-agent by hand after each restart; bob's
        warning is the signal
    default: file
    why:
      The token is already plain text in apollo's feed.xml, and an OOM kill already
      broke this once
    answer: file
proposed_by: bbugyi200.athena.140
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 17:38:56
status: wip
---

# Get `bob ref create --listen` episodes back into the AntennaPod feed

## Diagnosis (verified on athena and apollo, 2026-10-24)

The symptom: since 2026-10-17, no episode rendered by `bob ref create … -L` has appeared
in AntennaPod. The most recent one is
`bob ref create https://arxiv.org/abs/2509.20497 -i -P sase_blog -L`, a PromptDebt
paper. bob did its own part of the job correctly. `ref/papers/promptdebt.md` exists with
`audio: "[[lib/papers/promptdebt.mp3]]"`, and that MP3 is byte-identical to the
sase-listen library episode. The failure happens after bob's part, in the sase-listen
publish step.

The causal chain:

1. **apollo's gpg-agent lost its passphrase cache.** On 2026-10-17 at 12:06 UTC, memory
   pressure on apollo (`sase.service`) caused the OOM killer to kill `gpg-agent.service`
   (`journalctl --user -u gpg-agent` shows `Failed with result 'oom-kill'`). systemd
   restarted the agent at 12:06:44, and the restarted agent had an empty cache. apollo
   has not rebooted since (`uptime` 15 days), and lingering is on.
2. **apollo, the feed host, resolves its feed token by decrypting it with pass.**
   apollo's `~/.config/sase-listen/config.yml` comes from chezmoi
   `home/dot_config/sase-listen/config.yml` and sets
   `feed.token_command: pass show sase_listen_feed_token`.
   - That entry is encrypted to the shared primary RSA key, which has a passphrase. Its
     subkey keygrip is `35C383FA…`, and `gpg-connect-agent keyinfo --list` on apollo
     shows it as `P` (protected) and not cached.
   - When athena publishes, the work runs on apollo as `sase listen feed receive` inside
     a non-interactive `BatchMode` SSH session. That session has no TTY, so
     `pinentry-curses` cannot ask for the passphrase.
   - Running `pass show` the same way exits 2 with `gpg: public key decryption failed`.
   - sase-listen's `publish_episode` calls `resolve_token` before it copies anything, so
     every receive fails with `No feed token: token_command produced no token.`
3. **athena queued every publish, and nothing failed loudly.**
   - Auto-publish on athena puts each failed publish in
     `~/.local/state/sase-listen/outbox/` (shown with timestamps in UTC):
     - `how-the-claude-code-team-does-code-review…` (2026-10-17 22:15, 3 attempts)
     - `the-ai-native-sdlc-playbook…` (2026-10-23 14:00)
     - `promptdebt-a-comprehensive-study-of-technical-debt-across-ll-d690ce` (2026-10-24
       21:11)
   - Every entry has
     `last_error: "on apollo: No feed token: token_command produced no token."`
   - The renderer prints one yellow `⚠ Publish queued` line and exits 0. bob then prints
     about 15 lines of success summary, which push the warning off screen.
   - Each later render first retries the outbox, and each retry fails the same way.
   - `sase listen doctor` on athena reports `FAIL: feed:host` but does not say why. On
     apollo, `feed --json` reports `configured: false`.
4. **The feed itself stopped changing.** apollo's `feed.xml` was last written on
   2026-10-16 at 21:13 UTC and holds 53 items. Every athena library episode created
   since then has `published: false`. The Mac has no outbox.

**The cause is apollo's dependence on an interactively unlocked gpg-agent.** The
`[!decision] token_source` callout covers the fix: a mode-600 token file on apollo.
bob-cli also never surfaced the queued publish, so a broken feed went unnoticed for a
week. This plan fixes both.

## Ground rules for the implementer

- Never print the feed token. Never put it in a command-line argument (argv is visible
  in `ps`). Never write it to logs, plan notes, or commit messages. Move it only through
  stdin pipes and files with mode 0600.
- Do not edit the password store (`~/.password-store` is a shared git repo, and its
  recipient was deliberately pinned to the shared primary key on 2026-10-03). Do not
  re-encrypt any entry. Do not flush or reload any gpg-agent.
- Open the chezmoi repo with `/sase_repo` (`sase repo open chezmoi -r "<why>"`) and read
  its `AGENTS.md` before you edit it. Use only the path that command prints.
- On apollo, call binaries by absolute path, because the non-interactive SSH PATH does
  not include them: `~/bin/chezmoi` and `~/.local/bin/sase`.

## Step 1: Restore the feed (operations)

> [!decision] token_source = file
>
> 1. **Provision the token file on apollo from athena.** athena's gpg-agent currently
>    has the passphrase cached, so `pass show` works there without a prompt. Run this on
>    athena, exactly as written:
>
>    ```sh
>    pass show sase_listen_feed_token </dev/null | head -n 1 | ssh -o BatchMode=yes apollo 'umask 077 && mkdir -p ~/.sase && cat > ~/.sase/.sase_listen_feed_token.tmp && mv ~/.sase/.sase_listen_feed_token.tmp ~/.sase/sase_listen_feed_token'
>    ```
>
>    If `pass show` on athena fails or would prompt, stop here. Ask Bryan to run that
>    exact line in a terminal, and record that you did so.
>
> 2. **Verify the file without revealing it.** On apollo, run a `python3 -I` one-liner
>    that prints only booleans:
>    - the file is mode 0600 and holds one non-empty line;
>    - `"/" + token + "/"` appears in `~/.local/share/sase-listen/feed/feed.xml`.
>      feed.xml's enclosure URLs embed the live token, so a match proves the copy is
>      correct.
>
>    If the match is false, stop and report. Do not guess.
>
> 3. **Write the Step 2 template first, then render it on apollo** and install the
>    result atomically with mode 0644:
>
>    ```sh
>    ssh apollo 'umask 022 && ~/bin/chezmoi execute-template > ~/.config/sase-listen/.config.yml.new && mv ~/.config/sase-listen/.config.yml.new ~/.config/sase-listen/config.yml' < <chezmoi checkout>/home/dot_config/sase-listen/config.yml.tmpl
>    ```
>
>    apollo renders the template with its own hostname. Once apollo pulls the commit,
>    `chezmoi update` will find this file already up to date.

> [!decision] token_source = pass
>
> Change nothing in chezmoi and provision no file. The agent cannot type the passphrase.
> In the final summary, give Bryan these two commands to restore the feed, then go
> straight to Step 3:
>
> - `ssh -t apollo 'pass show sase_listen_feed_token >/dev/null'`
> - then, on athena, `sase listen publish --pending`
>
> Skip items 4–6 below. Bryan runs the equivalent after he unlocks apollo.

4. **Check the host.** Run `ssh apollo '~/.local/bin/sase listen doctor'`. It should
   report `feed:token` ok and `feed:feed.xml` ok. Then run `sase listen doctor` on
   athena. `feed:host` should now be ok, and `feed --json` on apollo should report
   `configured: true`.
5. **Flush the outbox.** Run `sase listen publish --pending` on athena. All three queued
   episodes should publish, and `~/.local/state/sase-listen/outbox/` should end up
   empty.
6. **Confirm the feed.** On apollo, `feed.xml` should have a fresh mtime and 56 `<item>`
   elements (53 + 3; report any other count and its reason, such as same-title
   superseding or retention). It should contain the PromptDebt title. The athena library
   manifests for the three episodes should now say `published: true`. Do not print feed
   URLs that contain the token.

## Step 2: chezmoi — the feed host reads the token from a file (`token_source = file` only)

In the chezmoi checkout, rename `home/dot_config/sase-listen/config.yml` to
`home/dot_config/sase-listen/config.yml.tmpl` with `git mv`. Change only the
`token_command` line:

- When `.chezmoi.hostname` is `apollo`, emit
  `token_command: cat {{ .chezmoi.homeDir }}/.sase/sase_listen_feed_token`.
  - Use an absolute path: sase-listen runs `token_command` through `shlex.split` with no
    shell, so `~` would not expand.
  - Put a short YAML comment above that line, in this order:
    - why the feed host cannot use pass: `feed receive` runs over non-interactive SSH,
      where pinentry cannot prompt, and an OOM-killed gpg-agent lost its cache on
      2026-10-17;
    - that the file is a mode-600 copy of `pass show sase_listen_feed_token`;
    - that this follows the `~/.sase/telegram_bot_token` precedent.
- Every other host keeps `token_command: pass show sase_listen_feed_token`.
- Render the template on athena (`chezmoi execute-template < …tmpl`) and diff the output
  against the current athena file. It must be **byte-identical**, so athena and the Mac
  pick up no change. Use `{{-` trimming carefully. Put any explanation for the template
  branch in `{{/* */}}` comments, which do not render.
- Render it on apollo, as in Step 1 item 3. The only differences from the old file
  should be the token line and its comment.

Do not run `chezmoi update` or `chezmoi apply` from the workspace checkout on athena.
athena's rendered file is unchanged, so it has nothing to apply. The chezmoi `AGENTS.md`
asks for `chezmoi update -a --force` after a commit lands. The commit lands when the
turn finalizes, so put that step in the final summary for Bryan to run on apollo,
athena, and the Mac. It is a no-op for this file on every host.

## Step 3: bob-cli — never let a queued publish pass silently

Today a `-L` run that queues its publish looks exactly like one that published. Fix that
in bob-cli without coupling to sase-listen internals beyond what bob already uses:

- bob already reads sase-listen library manifests.
- The outbox location `$XDG_STATE_HOME/sase-listen/outbox/` is documented in
  sase-listen's `docs/multi-machine.md` and `docs/architecture.md`.

### New module `src/native/highlights_ref/listen_feed.rs`

Register it in `src/native/highlights_ref/mod.rs`. Match the style of `listen.rs` and
`audio.rs`: `pub(super)` items, a `//!` module doc, and the same comment density.

- **`QueuedPublish`**: `episode_id`, `queued_at` (string), `attempts` (u64),
  `last_error` (string).
- **`outbox_dir() -> PathBuf`**: `bob_env::state_home().join("sase-listen/outbox")`.
  `state_home` already honors `XDG_STATE_HOME` and the thread-local test overrides.
- **`read_outbox(dir: &Path) -> Vec<QueuedPublish>`**:
  - Read regular `*.json` files whose names do not start with `.`. This skips
    sase-listen's `.tmp-*` atomic-write files.
  - Parse them leniently with `serde_json::Value`.
  - Skip unreadable files, invalid JSON, and entries whose `episode_id` fails
    `audio::is_valid_episode_id`.
  - Sort by `(queued_at, episode_id)`.
  - A missing directory returns an empty list.
- **`FeedNotice { this_episode: Option<QueuedPublish>, others: Vec<QueuedPublish> }`**
- **`feed_notice(audio: &Path, library: &Path, outbox: &Path) -> Option<FeedNotice>`**:
  - Return `None` when the outbox is empty.
  - For each entry, resolve `audio::episode_audio_source(library, &entry.episode_id)`.
  - An entry is this run's episode when that file's bytes equal `audio`. Check the size
    first, then call `companion::files_have_identical_bytes`. This works because
    sase-listen's `-o` copies the library MP3 byte for byte.
  - Every other entry, including one whose library episode is missing, goes into
    `others`.
  - Only the small outbox gets scanned, never the whole library.
- **`notice_lines(&FeedNotice) -> (String, String)`** returns the warning text and the
  hint text, without their prefixes:
  - **This episode queued**:
    `podcast feed: this episode was not published (sase listen queued it: <last_error>)`.
    When `others` is non-empty, append
    `; N earlier episode(s) are also queued, oldest since <YYYY-MM-DD>`.
  - **Only older entries**:
    `podcast feed: N episode(s) are still queued for publishing, oldest since <YYYY-MM-DD> (last error: <newest entry's last_error>)`.
  - **Formatting**:
    - pluralize "episode" correctly;
    - `<YYYY-MM-DD>` is the first 10 characters of `queued_at` when they parse as a
      date, otherwise omit the clause;
    - collapse whitespace and newlines in `last_error` and cap it at 240 characters with
      `…`.
  - **Hint**:
    `they will not appear in AntennaPod until the feed host is fixed; check `sase listen
    doctor`, then run `sase listen publish --pending``.
- **`after_listen_success(audio: &Path, output: ListenOutput)`**:
  - Compute the library root as `audio::audio_library_root(...)`. Load
    `bob_config::load_highlights_config(&bob_config::config_path())` best-effort: on
    error, pass `None`.
  - Compute the notice. It is best-effort and must never fail or slow the command:
    swallow every I/O or parse error.
  - **`ListenOutput::Print`**: push the notice onto a `thread_local!`
    `RefCell<Vec<FeedNotice>>` for deferred printing.
  - **`ListenOutput::Quiet`** (background ingest): `eprintln!` both lines at once
    (`warning: …` / `hint: …`). Never print to stdout.
- **`print_deferred_notices()`**: drain the thread-local and print each notice to stderr
  as `{styler.warning_prefix()}: <warning>` followed by `hint: <hint>`, with
  `Styler::detect()`.
- **`append_listen_feed_doctor_row(warnings: &mut Vec<String>)`**:
  - Print nothing unless `ListenCommand::resolve()` returns `Ok(Some(_))`.
  - Otherwise print one row:
    - `listen_feed: ok (no queued feed publishes)`, or
    - `listen_feed: warn (N episode(s) queued for the podcast feed, oldest since <date>; last error: <err>)`.
  - When the row is `warn`, push a warning that names the outbox path and the same hint.
    Doctor warnings never fail the doctor, so keep it that way.

### Wiring

- **`listen.rs`**: in `run_listen_with_output`, once `audio_is_mp3(&values.audio)`
  succeeds, call `super::listen_feed::after_listen_success(&values.audio, output)` just
  before `return Ok(values.audio.clone())`. Change no failure paths.
- **`create.rs`**: rename the body of `pub(super) fn run(matches)` to
  `fn run_create(matches) -> i32`. The new `run` calls it, then calls
  `super::listen_feed::print_deferred_notices()`, then returns the code. Every
  interactive `-L` route reaches `run_listen` from `create::run`: single target,
  collection (`collection::run`), and attach. The warning therefore always prints as the
  last lines, after the `ingested:` and `sync:` summary, whether the route succeeded or
  failed after the listen.
- **`doctor.rs`**: call `append_listen_feed_doctor_row(&mut warnings)` right after
  `append_listen_doctor_row(&mut warnings)`.

### Tests

- **Unit tests in `listen_feed.rs`** (explicit temp paths; no real `$HOME`):
  - `read_outbox` skips dotfiles, non-JSON, invalid JSON, and invalid ids, and sorts its
    results;
  - `feed_notice` returns `None` for an empty or missing outbox;
  - a matching library MP3 becomes `this_episode`, and the other entries are counted;
  - non-matching entries, and entries whose library episode is missing, go only into
    `others`;
  - wording in both `notice_lines` modes, plus pluralization, date omission, whitespace
    collapse, and truncation;
  - the doctor row's ok and warn text, using `TestEnvGuard` with `XDG_STATE_HOME` and
    `BOB_HIGHLIGHTS_LISTEN_COMMAND`.
- **Existing `listen.rs` unit tests**: any test whose `run_listen` call succeeds must
  pin `XDG_STATE_HOME` and `BOB_HIGHLIGHTS_AUDIO_LIBRARY` to scratch directories with
  `TestEnvGuard`. Otherwise those tests would read athena's real outbox, which is not
  hermetic.
- **Integration tests in `tests/cli/highlights/listen.rs`**:
  - Extend the fake `sase` with `FAKE_LISTEN_QUEUE_EPISODE=<id>`. With it set, after
    writing the MP3 to `-o`, the fake:
    - copies the MP3 to `$BOB_HIGHLIGHTS_AUDIO_LIBRARY/<id>/<file>.mp3`;
    - writes a `manifest.json` with `audio.file`;
    - writes `$XDG_STATE_HOME/sase-listen/outbox/<id>.json` with a fixed `last_error`.

    The harness already isolates `XDG_STATE_HOME`. Set `BOB_HIGHLIGHTS_AUDIO_LIBRARY` to
    a temp dir.

  - (a) A Markdown `-L` create with a queued episode exits 0. Its stderr contains
    `podcast feed: this episode was not published`, the fixed error, and
    `sase listen publish --pending`.
  - (b) The same create with no queue says nothing about the podcast feed.
  - (c) `bob ref doctor --no-hooks` with one outbox entry prints the
    `listen_feed: warn (1 episode queued` row.
  - Update any doctor-output assertions that the new row changes.

### Docs and help

Read the `docs.md` reference memory with `/sase_memory_read` first.

- **`docs/highlights-create.md` § Listen, step 6**: say that bob ends with a
  `warning: podcast feed: …` line plus a hint when this episode or an earlier one is
  still in the sase-listen publish outbox.
- **`docs/highlights-create.md`, troubleshooting**: rewrite the "A queued publish"
  bullet. The current text says "publish drains later", which is wrong while the host is
  broken. Explain:
  - the episode still binds and the vault is fine;
  - nothing reaches AntennaPod until the feed host is fixed (`sase listen doctor`);
  - `sase listen publish --pending` flushes the queue, and a refresh in AntennaPod then
    picks it up;
  - for example, a feed host whose `token_command` needs an unlocked gpg-agent fails
    every non-interactive publish.
- **`docs/highlights-create.md`, configuration**: extend the "`bob ref doctor` reports
  the `listen_command` row" sentence to also name the `listen_feed` row.
- **CLI help**: in the long help's `Listen:` paragraph in `create.rs`, add one sentence
  about the end-of-run podcast-feed warning. If generated CLI reference pages embed that
  help, run `just docs-gen`.
- Run `just docs-check`.

Never edit `Cargo.toml`'s version or `CHANGELOG.md`; the daily release train owns them.
Suggested commit subjects:

- bob-cli: `fix(listen): warn when a listen episode is queued instead of published`
- chezmoi: `fix(sase-listen): read the apollo feed token from a mode-600 file`

## Validation

- `just check` passes in bob-cli (fmt, clippy, the full test suite).
- `just docs-check` passes.
- With `cargo run -- ref doctor --no-hooks` on athena:
  - after Step 1 has emptied the outbox, it shows
    `listen_feed: ok (no queued feed publishes)`;
  - before that (or if the `pass` decision left the queue in place), it shows the `warn`
    row with the apollo token error.
- Do not run a new paid `-L` render just to test. The integration tests cover the
  warning path.
- `token_source = file`: Step 1 items 4–6 all pass. `chezmoi execute-template` output is
  byte-identical to the live config on athena and equal to the installed file on apollo.

## Final summary for Bryan

Report:

- the root cause in two sentences;
- the three episodes that were republished (or, for `pass`, the two commands to run);
- that he should pull to refresh the SASE Listen feed in AntennaPod;
- the post-commit `chezmoi update -a --force` on apollo, athena, and the Mac.

Also list these observed problems outside this plan's scope as recommendations, not work
done:

- sase-listen's `doctor` `feed:host` row and the host's `feed --json` do not say why the
  host is unconfigured. They should carry the token error.
- apollo's `sase.service` drove the 2026-10-17 OOM kill.
