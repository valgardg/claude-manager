# cam — Claude Account Manager

Run multiple Claude Code accounts on one machine without repeatedly logging in
and out. Each account gets its own `CLAUDE_CONFIG_DIR`, so logins stay isolated
and simultaneously active.

Built on Claude Code's `CLAUDE_CONFIG_DIR` environment variable: point each
account at its own config directory and its login stays separate. cam manages
those directories and the shell aliases for you instead of you hand-editing
`~/.zshrc`.

## Install

```bash
chmod +x cam
BIN_DIR="$(command -v brew >/dev/null 2>&1 && brew --prefix)/bin"
BIN_DIR="${BIN_DIR:-$HOME/.local/bin}"
mkdir -p "$BIN_DIR"
ln -sf "$PWD/cam" "$BIN_DIR/cam"
```

`brew --prefix` resolves to `/usr/local` on Intel Macs and `/opt/homebrew` on
Apple Silicon, so this works on either without hardcoding a path. Without
Homebrew it falls back to `~/.local/bin` — make sure that's on your `PATH`
(add `export PATH="$HOME/.local/bin:$PATH"` to your shell rc if not).

Requires `jq` (`brew install jq` on macOS/Linuxbrew, or `apt install jq` /
`dnf install jq` on Linux) and bash 3.2+. Works on macOS (Intel or Apple
Silicon) and Linux; the shell-alias step supports both zsh and bash and
targets whichever one your `$SHELL` points at (override with
`CAM_SHELL_RC`). The keychain checks in `cam doctor` are macOS-only and are
skipped automatically elsewhere.

## Concepts

| Object      | Is                              | Lives in                    |
| ----------- | ------------------------------- | --------------------------- |
| **Account** | An isolated Claude login        | `~/.claude-<name>`          |
| **Project** | An account + a directory        | cam's config                |
| **Alias**   | Shell shortcut to an account    | A managed block in `~/.zshrc` |

## Quick start

```bash
cam account add work
cam account login work        # run /login inside the session
cam project add fintech --account work --directory ~/work/fintech
cam run fintech               # cd + right account + launch claude
```

## Commands

### Accounts

```bash
cam account add <name>                 # create dir, write alias, prompt to log in
cam account list                       # all accounts + the email each is logged in as
cam account login <name>               # open Claude in that account so you can /login
cam account remove <name>              # unregister (config dir kept)
cam account remove <name> --purge      # also delete the dir, with confirmation
```

`cam account list` reads each account's logged-in email from its own
`.claude.json`, so you can see at a glance which real account a directory
currently holds. This catches the easy mistake of running plain `claude` and
completing `/login` as the wrong account — the identity in that dir changes,
and the list makes it visible instead of silent.

An account named `default` maps to `~/.claude` with the alias `claude`; every
other account `<n>` maps to `~/.claude-<n>` with the alias `claude-<n>`.

### Projects

```bash
cam project add <name> --account <a> --directory <path>
cam project list
cam project remove <name>              # never touches the directory itself
```

Re-running `project add` with an existing name updates it in place.

### Running

```bash
cam run <project>                      # cd + CLAUDE_CONFIG_DIR + exec claude
cam run <project> -- --model opus      # extra args pass through to claude
```

Runs in your current terminal — no tmux, no new windows.

### Aliases

```bash
cam alias list                         # cam-managed aliases and their targets
cam alias sync                         # regenerate the managed ~/.zshrc block
```

cam only ever writes between its markers:

```bash
# >>> claude-manager >>>   (managed by cam — do not edit inside these markers)
alias claude='CLAUDE_CONFIG_DIR="$HOME/.claude" claude'
alias claude-work='CLAUDE_CONFIG_DIR="$HOME/.claude-work" claude'
# <<< claude-manager <<<
```

Everything outside the markers is preserved untouched, and `~/.zshrc` is backed
up to `~/.config/cam/backups/` before every write.

### General

```bash
cam list                               # accounts and projects together
cam status                             # which account is active in this shell
cam doctor                             # drift, missing dirs, stale aliases, logins
```

`cam doctor` also verifies each account's login: it reports the email in each
dir, warns about dirs that were never logged in, and cross-checks the macOS
Keychain. Every account *except* `default` stores its OAuth token under
`Claude Code-credentials-<hash>`, where `<hash>` is the first 8 hex of the
sha256 of the config dir path; `default` uses the plain, unsuffixed
`Claude Code-credentials` entry instead (see the design note below). Doctor
flags an account whose identity file has an email but no matching keychain
entry (a stale or hand-copied dir), and notes orphaned keychain credentials
left behind by a deleted config dir.

## Design notes

- **Config is the source of truth.** `~/.config/cam/config.json` holds
  everything; the `~/.zshrc` block is generated output. `cam doctor` reports
  drift between them.
- **Idempotent.** Re-running `account add` or `alias sync` converges rather
  than duplicating.
- **Non-destructive by default.** `remove` unregisters; deleting a config dir
  needs `--purge` *and* a confirmation prompt. Removing a project never touches
  its directory. An account bound to a project can't be removed until the
  project is repointed or deleted.
- **Adopts existing setups.** On first run, cam imports accounts from a prior
  `~/.claude-manager.json` and from any aliases already in the managed rc
  block, then reuses the same markers so you don't end up with two blocks. It
  also registers a bare, pre-existing `~/.claude` as the `default` account
  even without either of those, since that's Claude Code's own default
  location whether or not cam is involved.
- **`default` never sets `CLAUDE_CONFIG_DIR`.** Claude Code special-cases an
  *unset* `CLAUDE_CONFIG_DIR`: only then does it read the identity file from
  the legacy, top-level `~/.claude.json`. Exporting `CLAUDE_CONFIG_DIR`
  explicitly — even to that same `~/.claude` path — makes Claude Code look
  for `.claude.json` *inside* that directory instead, find nothing, and
  report the account as logged out. So the `default` alias, `cam run`, and
  `cam account login` all invoke `claude` bare for this one account instead
  of exporting the var, and `cam doctor`/`account list` read the identity
  and keychain entry from the unsuffixed, legacy locations for it.

## Not in scope

Credential handling (Claude's own `/login` does that), syncing MCP config
between accounts, and any GUI.

## Environment overrides

| Variable       | Default              | Purpose                          |
| -------------- | -------------------- | -------------------------------- |
| `CAM_HOME`     | `~/.config/cam`      | Config and backup location       |
| `CAM_SHELL_RC` | `~/.zshrc`           | Which rc file to manage          |

Useful for trying cam against a throwaway rc file before pointing it at your
real one.
