# cam — Claude Access Manager

cam runs each project under the right Claude account, using a separate
`CLAUDE_CONFIG_DIR` per account so logins don't collide.

Set up one account per login you use (most people need two: personal and
work), bind a directory to each as a project, then `cam <project>` cds in
and launches Claude with the matching config dir.

cam manages the `CLAUDE_CONFIG_DIR` directories, and gives each account its
own command, so you don't hand-edit either.

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

Then, once ever, put cam's own bin dir on your `PATH`:

```bash
cam shim sync
source ~/.zshrc          # or your shell's rc — see below
```

That `source` is a one-time step, not something you repeat per account: it adds
`~/.config/cam/bin` to your `PATH`. Every account you add later drops an
executable in that directory, which every shell — including ones already open —
picks up immediately.

Requires `jq` (`brew install jq` on macOS/Linuxbrew, or `apt install jq` /
`dnf install jq` on Linux) and bash 3.2+. Works on macOS (Intel or Apple
Silicon) and Linux; the `PATH` step supports both zsh and bash and targets
whichever one your `$SHELL` points at (override with `CAM_SHELL_RC`). The
keychain checks in `cam doctor` are macOS-only and are skipped automatically
elsewhere.

## Concepts

| Object       | Is                            | Lives in                       |
| ------------ | ----------------------------- | ------------------------------ |
| **Account**  | An isolated Claude login      | `~/.claude-<name>`             |
| **Project**  | An account + a directory      | cam's config                   |
| **Command**  | An executable per account     | `~/.config/cam/bin/claude-<n>` |

## Quick start

```bash
cam account add work
cam account login work        # run /login inside the session
claude-work                   # launch Claude as 'work', from anywhere

cam project add fintech --account work --directory ~/work/fintech
cam fintech                   # cd + right account + launch claude
```

Two ways in, for two different jobs: `claude-<account>` when you just want a
given login in the directory you're already standing in, `cam <project>` when
you want a specific directory *and* its account.

## Commands

### Accounts

```bash
cam account add <name>                 # create dir, write command, prompt to log in
cam account list                       # all accounts + the email each is logged in as
cam account login <name>               # open Claude in that account so you can /login
cam account remove <name>              # unregister (config dir kept)
cam account remove <name> --purge      # also delete the dir, with confirmation
```

You'll usually set these up once — most people need two, personal and work.
Each account is isolated: its own `CLAUDE_CONFIG_DIR`, its own login, and
multiple accounts can be logged in at the same time. On first run, cam
registers any pre-existing `~/.claude` as `default` automatically, so if you
already had Claude Code set up you don't need to run `account add` for it.

`cam account list` reads each account's logged-in email from its own
`.claude.json`, so you can see at a glance which real account a directory
currently holds. This catches the easy mistake of running plain `claude` and
completing `/login` as the wrong account — the identity in that dir changes,
and the list makes it visible instead of silent.

An account named `default` maps to `~/.claude` with the command `claude`; every
other account `<n>` maps to `~/.claude-<n>` with the command `claude-<n>`.

### Projects

A project is an account bound to a directory. `cam <project>` cds into the
directory and launches Claude under that account.

```bash
cam <project>                          # cd + right account + launch claude
cam project add <name> --account <a> --directory <path>
cam project list
cam project remove <name>              # never touches the directory itself
```

Re-running `project add` with an existing name updates it in place.

### Running

```bash
cam <project>                          # same as the line below
cam run <project>                      # cd + CLAUDE_CONFIG_DIR + exec claude
cam run <project> -- --model opus      # extra args pass through to claude
```

`project add` refuses to create a project whose name collides with a cam
command (`account`, `project`, `alias`, `run`, `list`, `status`, `doctor`,
`help`), so a bare `cam <name>` always resolves unambiguously.

Runs in your current terminal — no tmux, no new windows.

### Commands

Every account gets a command named after it, so switching login is just
choosing which one you type:

```bash
claude-work                            # Claude as the 'work' account
claude-personal                        # Claude as the 'personal' account
claude                                 # the 'default' account (~/.claude)
```

These work in any directory — the command only selects the config dir, it
doesn't `cd` anywhere. Arguments pass straight through, so `claude-work --model
opus` behaves exactly like the flags on plain `claude`. Use `cam <project>` when
you want a directory as well as an account.

```bash
cam shim list                          # each account's command and its target
cam shim sync                          # regenerate the commands and PATH entry
```

#### Why executables and not aliases

Each command is a small executable in `~/.config/cam/bin`, not a shell alias.
This is the difference between `cam account add work` being *usable* and merely
being *written down*:

- **It works in the terminal you're already in.** A brand-new command name has
  never been in your shell's hash table, so there is nothing stale to clear —
  no `source`, no `rehash`, no new tab. An alias, by contrast, only ever reaches
  shells started after it was written, because `cam` is a separate process and
  cannot reach into the shell that launched it. That limitation is what made the
  old approach need a `source` after every `account add`.
- **It works where aliases don't exist**: scripts, Makefiles, `xargs`, and any
  non-interactive shell.

The cost is one `PATH` entry, written to your shell rc once:

```bash
# >>> claude-manager >>>   (managed by cam — do not edit inside these markers)
# Puts cam's per-account commands (claude, claude-<name>, …) on PATH.
case ":$PATH:" in
  *":$HOME/.config/cam/bin:"*) ;;
  *) export PATH="$HOME/.config/cam/bin:$PATH" ;;
esac
for _cam_f in "$HOME/.config/cam/bin"/*; do
  [ -f "$_cam_f" ] && unalias "${_cam_f##*/}" 2>/dev/null
done
unset _cam_f
# <<< claude-manager <<<
```

Adding or removing accounts never touches this block — it only adds or deletes
files in the bin dir — so the one-time `source` really is one-time. The `case`
guard keeps `PATH` from growing if your rc is sourced more than once. The
`unalias` loop retires aliases left by cam ≤ 1.0, and only for names cam owns (a
name is cam's only if it has a shim for it), so your own aliases are left alone.
Both are POSIX, so the block works in zsh and bash alike.

Everything outside the markers is preserved untouched, and your rc file is
backed up to `~/.config/cam/backups/` before every write. `cam alias list` and
`cam alias sync` still work as synonyms for the `shim` subcommands.

#### Upgrading from the alias-based version

Run `cam shim sync` once. It adopts the accounts named in your old alias block,
converts the block into the `PATH` entry above, and writes a command per
account. Terminals opened before that still hold the old aliases in memory;
`source` your rc in them (or open a new one) and the loop above clears them.
This matters more than it sounds: an alias takes precedence over a `PATH`
lookup, so a stale `claude-personal` alias would expand to plain `claude` and
land on the *default* account's command — the wrong account, silently.

### General

```bash
cam list                               # accounts and projects together
cam status                             # which account is active in this shell
cam doctor                             # drift, missing dirs, missing commands, logins
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

- **Projects for daily use, accounts for setup.** `cam <project>` (or `cam
  run <project>` in scripts) is what you'll run day to day. Accounts are
  configured once — most people need two — and are what let projects
  switch identity safely: each one is an isolated, simultaneously active
  Claude login.
- **Config is the source of truth.** `~/.config/cam/config.json` holds
  everything; the bin dir and the rc block are generated output. `cam doctor`
  reports drift between them.
- **Idempotent.** Re-running `account add` or `shim sync` converges rather
  than duplicating.
- **Non-destructive by default.** `remove` unregisters; deleting a config dir
  needs `--purge` *and* a confirmation prompt. Removing a project never touches
  its directory. An account bound to a project can't be removed until the
  project is repointed or deleted. Pruning the bin dir only ever deletes files
  carrying cam's own `# managed by cam` marker, so anything else you keep there
  survives.
- **cam bypasses its own commands.** `cam run` and `cam account login` resolve
  the real `claude` binary with the bin dir stripped from `PATH`. Otherwise the
  `default` account — whose command is plain `claude` — would intercept them and
  re-decide `CLAUDE_CONFIG_DIR` after cam had already chosen it.
- **Adopts existing setups.** On first run, cam imports accounts from a prior
  `~/.claude-manager.json`, from any aliases already in the managed rc block,
  and from any shims left in the bin dir, then reuses the same markers so you
  don't end up with two blocks. It also registers a bare, pre-existing
  `~/.claude` as the `default` account even without any of those, since that's
  Claude Code's own default location whether or not cam is involved.
- **`default` never sets `CLAUDE_CONFIG_DIR`.** Claude Code special-cases an
  *unset* `CLAUDE_CONFIG_DIR`: only then does it read the identity file from
  the legacy, top-level `~/.claude.json`. Exporting `CLAUDE_CONFIG_DIR`
  explicitly — even to that same `~/.claude` path — makes Claude Code look
  for `.claude.json` *inside* that directory instead, find nothing, and
  report the account as logged out. So the `default` command, `cam run`, and
  `cam account login` all invoke `claude` bare for this one account instead
  of exporting the var, and `cam doctor`/`account list` read the identity
  and keychain entry from the unsuffixed, legacy locations for it. As
  executables they go one better than the old alias: they actively `unset` an
  inherited `CLAUDE_CONFIG_DIR`, so `claude` selects the default account even
  in a shell where something else exported it. An alias could only avoid
  setting the variable, not clear it.

## Not in scope

Credential handling (Claude's own `/login` does that), syncing MCP config
between accounts, and any GUI.

## Environment overrides

| Variable       | Default                                    | Purpose                          |
| -------------- | ------------------------------------------ | -------------------------------- |
| `CAM_HOME`     | `~/.config/cam`                            | Config, backups, and the bin dir |
| `CAM_SHELL_RC` | auto-detected from `$SHELL` (see Install)  | Which rc file to manage          |

Useful for trying cam against a throwaway rc file before pointing it at your
real one — point both at a scratch directory and nothing outside it is touched.
