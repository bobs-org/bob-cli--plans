---
tier: tale
title: Fix Bob narration on the Mac with the installed SASE plugin
goal: Make Bob's managed listen command launch the existing SASE plugin on the Mac
  and verify the repair.
size: small
proposed_by: bbugyi200.apollo.6m
status: done
---

# Fix Bob narration on the Mac by using the installed SASE command plugin

## Outcome and scope

Make Bryan's existing command use the narration implementation already installed on his
Mac:

```sh
bob ref create https://interconnected.org/home/2026/01/15/reminders -i -L -P sase
```

Correct the chezmoi-managed `highlights.listen_command` to
`sase listen render {target} -e full -o {audio}`, deploy that configuration on the Mac,
and document how plugin and standalone installations map to Bob's command setting. This
is a `small` tale: one agent can make and verify this bounded config and documentation
fix. No Rust behavior change or second narration installation is needed. Preserve the
existing full edition, placeholder quoting, credentials, feed configuration, and
PDF/audio transaction behavior.

## Confirmed diagnosis

The supplied output shows successful article capture followed by
`sh: sase-listen: command not found` and exit 127. Read-only investigation on 2026-10-10
established:

- On `mac`, a login shell finds `uv` and `ffmpeg`, but has no standalone `sase-listen`
  executable. `uv tool list` has no separate `sase-listen` tool.
- Both `sase listen --help` and `sase listen --version` work. The latter reports
  `sase-listen 0.1.1 (editable @ 1b82d27)`. Calling it through `/bin/sh` with the login
  shell's inherited PATH also works, matching Bob's execution model.
- The live Mac config and its managed source both contain
  `listen_command: sase-listen render {target} -e full -o {audio}`.
- `bob ref doctor --no-hooks` reports that `sase-listen` cannot be found. The same
  doctor invocation with
  `BOB_HIGHLIGHTS_LISTEN_COMMAND='sase listen render {target} -e full -o {audio}'`
  reports `listen_command: ok (...; executable: sase)`.
- `sase listen --version` also succeeds on apollo and athena, both reporting
  `0.1.1 (git 1b82d27)`. Thus the shared config can use the plugin on all three checked
  machines without introducing a macOS-specific template.

This is a command/configuration mismatch, not evidence that the narration engine is
missing. A doctor `ok` row checks the executable, not successful synthesis or credential
validity; the plugin help/version probes provide additional dispatch verification. No
source files or live configuration were changed during planning.

## Source context and repository access

Before work, use `/sase_repo` to open `chezmoi` from the current Bob workspace and read
its `AGENTS.md`. Use only the checkout path returned by `sase repo open`. Reopen
`gh:sase-org/sase-listen` through the same skill only if additional upstream source
inspection is needed. Do not edit a canonical checkout or live managed file directly.
For Mac access, use `/sase_memory_read` for `tailnet.md`; SSH alias `mac` reaches the
account `bbugyi`, and the Mac may be offline when its lid closes. Use a login shell for
remote Bob/SASE commands: bare SSH has a different PATH.

Relevant source, relative to each repository:

- chezmoi `home/dot_config/bob/config.yml`, around line 50: managed command and
  explanatory comment. `.chezmoiroot` is `home`.
- chezmoi `home/dot_config/sase-listen/config.yml`: existing Gemini/pass credentials and
  apollo auto-publish settings; these need no edits.
- Bob `src/native/highlights_ref/listen.rs`: `ListenCommand::resolve` honors the
  environment override; `run_listen` substitutes shell-quoted placeholders and executes
  the configured string through `sh -c`. It already accepts a command with a subcommand
  such as `sase listen`.
- Bob `README.md` dependency list, `docs/highlights-create.md` prerequisites and
  Configuration/Troubleshooting, and `docs/highlights-ref-sync.md`'s listen example.
- Upstream `sase-listen` `README.md`, `docs/getting-started.md`,
  `docs/multi-machine.md`, and `pyproject.toml`: `sase plugin install listen` provides
  `sase listen`; `uv tool install sase-listen` provides `sase-listen`. Both use the same
  implementation and config. The inspected upstream revision was `1b82d27`; Bob was
  `1870ac1`, and chezmoi was `1dcd13fc`.

## Implementation

1. Update chezmoi `home/dot_config/bob/config.yml` to use:

   ```yaml
   highlights:
     listen_command: sase listen render {target} -e full -o {audio}
   ```

   Change the existing value in place, preserving all other settings. Adjust the
   adjacent comment to name the plugin invocation and point to Bob's setup docs. Keep
   placeholders unquoted: Bob quotes their values itself. Do not install another tool,
   change PATH, add aliases, or add automatic backend fallback.

2. Update the Bob documentation at the source locations above. Explain both supported
   combinations clearly: the SASE plugin uses `sase listen render ...`, while a
   standalone installation uses `sase-listen render ...`. Use the plugin command in the
   managed-setup example while retaining an explicit standalone example for other users.
   Clarify that installing Bob does not itself install either narration frontend. Add an
   exit-127 troubleshooting entry with:
   - `command -v sase-listen`, `sase listen --help`, and `bob ref doctor --no-hooks` to
     distinguish the two installations.
   - The exact temporary override below, and the permanent managed-config repair.
   - The explanation that the failed command left the vault untouched and can be rerun
     after the command setting is corrected.

   ```sh
   BOB_HIGHLIGHTS_LISTEN_COMMAND='sase listen render {target} -e full -o {audio}' \
     bob ref create https://interconnected.org/home/2026/01/15/reminders -i -L -P sase
   ```

   Do not rewrite historic captured logs, change the CLI's generic standalone
   example/error hint, or add a new CLI flag for this config-only repair.

3. Validate the changed YAML and the effective chezmoi diff. Confirm that the only
   semantic config change is the command prefix from `sase-listen` to `sase listen`.
   Review the documentation for matching placeholders and options, and run
   `git diff --check` in both modified repositories. No new tests or Rust build are
   needed for this config/documentation change.

4. Deliver and apply the approved managed-source change to the Mac using the
   repository's supported publish/apply workflow. Follow chezmoi's `AGENTS.md`
   obligation to run `chezmoi update -a --force` after commits, including finalizer
   commits; ensure the Mac receives the revision containing this change. Use the SASE
   finalizer for commits, not raw `git commit`. If applying a reviewed source checkout
   before its finalizer commit, use chezmoi's targeted apply of the Bob config from that
   opened source rather than hand-editing the destination. Any repository accessed on
   the target machine must also be opened via `/sase_repo`. Verify the effective Mac
   setting after apply and that a subsequent chezmoi diff has no pending change to that
   setting. A session-only environment override is evidence of the diagnosis, not the
   permanent fix.

5. Verify the installed configuration on the Mac in a login shell, with
   `BOB_HIGHLIGHTS_LISTEN_COMMAND` unset for the permanent-config checks:
   - `sase listen --version` and `sase listen render --help` succeed, including a probe
     through `sh -c` with the same inherited PATH.
   - `bob ref doctor --no-hooks` reports `listen_command: ok` with the plugin template
     from config. Inspect the actual row; doctor's overall exit status can be successful
     even when optional dependencies warn.
   - Run `sase listen doctor` to assess runtime prerequisites. Its directory checks can
     create application directories, so this belongs after approval. Do not use
     `doctor --online`: upstream documents it as unimplemented. Do not print
     password-manager values or mistake a configured credential source for a successful
     API request.
   - Render a tiny local Markdown fixture through
     `sase listen render -n tone --no-publish` to a temporary MP3 using isolated
     temporary XDG cache/data/state directories. This exercises actual plugin execution
     and MP3 production without an API request or feed publication. Use `sh -c` and
     argument quoting equivalent to Bob's configured runner. Inspect the output as a
     nonempty MP3.
   - Run Bryan's exact Bob command with `--dry-run` appended. Confirm its
     `listen: would-run` line starts with `sase listen render`, retains `-e full`, and
     plans the companion MP3 without writing to the vault. A dry run can still
     fetch/capture in scratch; it never proves that full narration succeeded.

   Remove only temporary files created for verification. Provide Bryan the exact
   original retry command once the permanent config and checks pass. Do not run a paid
   full-article synthesis or publish an episode solely as this smoke test.

## Acceptance and operational boundaries

- Managed source and the effective Mac config use the installed `sase listen` plugin,
  and the fix survives a normal chezmoi apply.
- The plugin is callable by the shell Bob uses; doctor recognizes the configured
  executable; the isolated tone smoke produces an MP3; Bob's dry run expands the
  corrected command with the original options.
- Documentation explains the plugin/standalone distinction and an immediately usable
  recovery command. Both repo diffs pass whitespace validation.
- No unrelated dotfiles, Rust behavior, plugin code, credentials, feed settings,
  production vault files, or durable memory are changed by this repair.
- Report deployment and test results separately. If the Mac becomes unreachable, finish
  the source/docs work and identify deployment as outstanding; do not claim a live
  repair. If doctor or rendering exposes a separate credential/network failure, report
  its exact non-secret diagnostic separately from the fixed executable mismatch. Do not
  expand this fix into a toolchain migration.

Long-running checks or apply operations must use `/sase_monitor` when handing off
execution. Follow `/sase_final` for both modified repositories when ending the
implementation turn, including the required chezmoi post-commit application.
