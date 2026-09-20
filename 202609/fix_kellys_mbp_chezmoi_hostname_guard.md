---
tier: tale
title: Fix the stale Kellys-MBP chezmoi hostname guard
goal:
  chezmoi manages ~/.config/sase/sase_kellys_mbp.yml and ~/.ssh/tailnet.conf on the Mac
  again, so the deployed SASE overlay matches the chezmoi source and the mbp-chime
  notification rule takes effect.
size: medium
proposed_by: bbugyi200.kellys_mbp.17
create_time: 2026-09-20 17:25:31
status: wip
---

# Fix the stale `Kellys-MBP` chezmoi hostname guard

## Problem

`~/.config/sase/sase_kellys_mbp.yml` on the Mac is missing the `ace.notification_rules`
block (the `mbp-chime` Glass-sound rule) that exists in the chezmoi source at
`home/dot_config/sase/sase_kellys_mbp.yml`. The deployed copy is byte-identical to the
source as of commit `f2ed1c64` (2026-09-03) and has not been touched since 2026-09-06.

## Root cause

`home/.chezmoiignore` gates the overlay on a hostname literal that no longer matches
this machine:

```
{{ if ne .chezmoi.hostname "Kellys-MBP" }}
.config/sase/sase_kellys_mbp.yml
{{ end }}
```

`chezmoi data` reports `.chezmoi.hostname == "Kellys-MacBook-Pro"` (derived from
`LocalHostName`, which `scutil --get LocalHostName` confirms; `scutil --get HostName` is
unset, so macOS falls back to `Kellys-MacBook-Pro.local`). `Kellys-MacBook-Pro` !=
`Kellys-MBP`, so the guard is always true and chezmoi permanently ignores the overlay.

The machine was renamed after 2026-09-06. Evidence that chezmoi _used_ to manage the
file: `chezmoi state dump` still holds an `entryState` entry for
`/Users/bbugyi/.config/sase/sase_kellys_mbp.yml`, which chezmoi only records after an
apply. It is now absent from `chezmoi managed`.

The `ace` block was added on 2026-09-20 by an agent running on `athena` (commit
`f7da6820`), so it landed in the chezmoi source but could never reach this Mac.

### Confirmed user-visible impact

- `sase config layers` shows the overlay loading with keys `id, memory` only - no `ace`.
- `sase config show` lists only the `quiet-task-beads` notification rule; `mbp-chime` is
  absent, so SASE still uses the terminal bell instead of `Glass.aiff`.

### Second file hit by the same stale literal

The compound guard at the end of `home/.chezmoiignore` also ignores `.ssh/tailnet.conf`
on this Mac for the same reason, and
`home/.chezmoiscripts/run_onchange_after_ensure_ssh_tailnet_include.tmpl` is gated on
the same literal, so the run_onchange script never executes here either.
`~/.ssh/tailnet.conf` happens to still be byte-identical to source (it was applied on
2026-09-06 before the rename), so there is no content drift today - but it is unmanaged
and would silently stop tracking future edits.

## Why the fix is a literal hostname swap and NOT a refactor

Do **not** replace the guard with `.chezmoi.os`, `hasPrefix`, `.chezmoitemplates` /
`includeTemplate`, or any other shape. SASE parses `.chezmoiignore` and requires this
exact stanza shape. In the `sase` repo, `src/sase/main/_init_chezmoi_ignore.py` defines:

```python
_HOSTNAME_GUARD_RE = re.compile(r'^\{\{ if ne \.chezmoi\.hostname "([^"]+)" \}\}$')
```

`parse_hostname_ignore_entries()` uses it to build a `{target_entry: hostname}` map, and
`src/sase/amd/_config.py` feeds that map into `render_chezmoi_h1_template()` to emit the
machine H1 switch in `home/AGENTS.md.tmpl`, `CLAUDE.md.tmpl`, `GEMINI.md.tmpl`,
`OPENCODE.md.tmpl`, and `QWEN.md.tmpl`. Changing the stanza shape would make SASE drop
the machine mapping and regenerate those shims with only the fallback title.

This also means `.chezmoiignore` is the single source of truth for the hostname: the
five `*.md.tmpl` files are **generated**, so fix `.chezmoiignore` and regenerate them
with `sase init memory` rather than hand-editing them.

## Steps

1. Open the chezmoi repo and work only in the path it prints:

   ```bash
   CM="$(sase repo open chezmoi -r 'Fix the stale Kellys-MBP hostname guard')"
   ```

2. In `$CM/home/.chezmoiignore`, replace both occurrences of the literal `"Kellys-MBP"`
   with `"Kellys-MacBook-Pro"`. There are exactly two:
   - the `{{ if ne .chezmoi.hostname "Kellys-MBP" }}` stanza guarding
     `.config/sase/sase_kellys_mbp.yml`
   - the compound
     `{{ if and (ne .chezmoi.hostname "athena") (ne .chezmoi.hostname "apollo") (ne .chezmoi.hostname "Kellys-MBP") }}`
     stanza guarding `.ssh/tailnet.conf`

   Preserve the stanza shape exactly (see the section above).

3. In `$CM/home/.chezmoiscripts/run_onchange_after_ensure_ssh_tailnet_include.tmpl`,
   change `(eq .chezmoi.hostname "Kellys-MBP")` to
   `(eq .chezmoi.hostname "Kellys-MacBook-Pro")`.

4. In `$CM/tests/bash/poseidon_chezmoi_isolation_test.sh`, update the two `Kellys-MBP`
   fixture arguments (`render_ignore Kellys-MBP` and `render_cargo Kellys-MBP`) to
   `Kellys-MacBook-Pro` so the fixtures name a hostname that actually exists. These
   assertions are about Poseidon/Cargo isolation and pass either way, but the fixture
   should not encode a dead hostname.

5. Verify the guard now resolves correctly, before regenerating anything:

   ```bash
   chezmoi execute-template < "$CM/home/.chezmoiignore"
   ```

   `.config/sase/sase_kellys_mbp.yml` and `.ssh/tailnet.conf` must **not** appear in the
   output, while `.config/sase/sase_athena.yml`, `.config/sase/sase_apollo.yml`, and the
   Poseidon entries must still appear.

6. Regenerate the provider shims (do not hand-edit them). From `$CM`:

   ```bash
   sase init memory -c -d
   ```

   Confirm the plan now includes H1 changes to `home/AGENTS.md.tmpl`,
   `home/CLAUDE.md.tmpl`, `home/GEMINI.md.tmpl`, `home/OPENCODE.md.tmpl`, and
   `home/QWEN.md.tmpl` swapping `Kellys-MBP` for `Kellys-MacBook-Pro`. Note that this
   repo already has unrelated pending memory drift (`sase/artifact_relations.json`,
   `sase/memory/README.md`, `sase/memory/task_types.md`); that is pre-existing and will
   be regenerated too. Then apply without letting it commit:

   ```bash
   sase init memory -C
   ```

7. Commit the change in the chezmoi repo with `/sase_git_commit`. Call out in the body
   that the machine was renamed from `Kellys-MBP` to `Kellys-MacBook-Pro` and that the
   five `*.md.tmpl` shims are regenerated output.

8. Sync the live chezmoi source dir and deploy. The linked checkout is a separate clone
   from the source dir `chezmoi apply` actually reads (`chezmoi source-path`), so the
   commit must reach `~/.local/share/chezmoi` first:

   ```bash
   git -C ~/.local/share/chezmoi pull --ff-only
   chezmoi diff
   ```

   Review the diff - it should show only `~/.config/sase/sase_kellys_mbp.yml` gaining
   the `ace` block, the five `~/AGENTS.md`-style shims unchanged in rendered output
   (their H1 already renders to the Mac title via the `{{ else }}` fallback), and the
   `ensure_ssh_tailnet_include` run_onchange script re-running. Then:

   ```bash
   chezmoi apply
   ```

## Verification

Run all of these after `chezmoi apply`:

1. `diff ~/.config/sase/sase_kellys_mbp.yml "$CM/home/dot_config/sase/sase_kellys_mbp.yml"`
   exits 0 (no output).
2. `chezmoi managed | grep -E 'sase_kellys_mbp\.yml|tailnet\.conf'` lists both entries.
3. `chezmoi status` is empty.
4. `sase config layers` shows the `overlay:sase_kellys_mbp.yml` layer with `ace` among
   its keys (currently it shows only `id, memory`).
5. `sase config show | grep -A6 notification_rules` includes a rule named `mbp-chime`
   with `sound: /System/Library/Sounds/Glass.aiff`, alongside the existing
   `quiet-task-beads` rule.
6. `head -1 ~/CLAUDE.md` still reads `# kellys_mbp - Kelly's MacBook Pro` (regression
   check on the regenerated shims - this must not fall back to a wrong title).
7. `grep -rn 'Kellys-MBP' "$CM" --exclude-dir=.git` returns nothing.
8. From `$CM`, `just test-bash` passes (bashunit; covers
   `poseidon_chezmoi_isolation_test.sh`).
9. `sase init -c` reports `init memory  memory files are current` again.

## Risks and notes

- **Recurrence.** macOS derives `LocalHostName` from `ComputerName` ("Kelly's MacBook
  Pro"), and `HostName` is unset, so a future rename or a DHCP-supplied name can break
  the guard again. SASE's `ensure_machine_ignore_entry()` only _appends_ a guard when
  the entry is missing; it never repairs a stale hostname, so nothing self-heals. Do not
  try to fix this inside `.chezmoiignore` (see the refactor warning above). Two options
  worth raising with the user afterwards, both out of scope here:
  - pin the name with `sudo scutil --set HostName Kellys-MacBook-Pro` so it stops
    tracking `ComputerName`;
  - file a task bead against the `sase` project for a `sase doctor` check (or a repair
    path in `ensure_machine_ignore_entry`) that flags a `.chezmoiignore` hostname guard
    naming the current machine's overlay but not its current hostname.
- **Do not rename the Mac back to `Kellys-MBP`.** The Tailscale node is
  `kellys-macbook-pro` and `~/.ssh/tailnet.conf` points `Host mac` at
  `kellys-macbook-pro.tail297af1.ts.net`; renaming would desync the tailnet identity.
- **`chezmoi apply` blast radius.** `chezmoi status` is currently empty, so the apply
  should be limited to the files above. Still review `chezmoi diff` before applying and
  stop if anything unexpected appears.
- The `ensure_ssh_tailnet_include` run_onchange script will re-run because its content
  hash changes. It is idempotent (it checks for an existing `Include tailnet.conf` line
  in `~/.ssh/config`, which is already present as line 1), so it should be a no-op.
- `home/.chezmoiignore` also gates `.config/sase/sase_work.yml`, which does not exist in
  the source. That is harmless and unrelated; leave it alone.
