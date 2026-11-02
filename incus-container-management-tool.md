---
name: incus-container-management-tool
description: Configure Incus containers and enter them as a specific user — use when you need to run a command inside a container as another account, change instance configuration, or reach a container's systemd user services, which need a real login session rather than `su`.
compatibility: opencode
metadata:
  type: tool
  original_file: incus-container-management-tool.md
  category: configuration
  scope: incus
---

# Incus Container Management

The purpose of this tool is to configure Incus containers and to get inside one as a particular user — including inside one in a way that can reach that user's own systemd services.

## TOC

- [Set Container Description](#set-container-description)
- [Run as User](#run-as-user)
  - [A Shell Is Not a Session](#a-shell-is-not-a-session)
  - [Commands That Need a Session](#commands-that-need-a-session)

## Set Container Description

```
incus config set <some.container> description="some description" -p
```

> ⚠️ **Warning** - The `-p` (or `--property`) flag is required. `description` is an instance-defined property/metadata, not a configuration setting.

## Run as User

> ⚠️ **Warning** - The `--login` flag is required to load environment variables from the login shell chain (`/etc/profile`, `~/.bashrc`, etc.). Without it, environment variables will be empty.

Interactive session:

```
incus exec <container> -- su --login <username>
```

Single command:

```
incus exec <container> -- su --login <username> --command "<command>"
```

> 📝 **Note** - `incus exec` allocates a TTY only with `-t`. Without it the shell has no terminal, so an interactive `su --login` is not interactive.

### A Shell Is Not a Session

`su --login` gives the user a login *shell* — the environment of `/etc/profile` and `~/.bashrc` — but not a systemd *session*. The difference decides whether the user's own services exist at all:

| | `su --login` | a real session (ssh, `machinectl shell`) |
|---|---|---|
| `XDG_RUNTIME_DIR` | unset | `/run/user/<uid>` |
| `systemctl --user` | fails: *"Failed to connect to user scope bus via local transport"* | works |
| user manager (`user@<uid>.service`) | never starts | starts, and its units run |
| user unit state (a runtime directory, an unlocked secret) | none | exists for the session |

The reason is `pam_systemd`, the PAM module that registers a login with systemd-logind and creates the runtime directory. On a NixOS host it is wired into `/etc/pam.d/login` and `/etc/pam.d/sshd` — and not into `/etc/pam.d/su` — so `su` cannot create a session, however much it looks like a login.

### Commands That Need a Session

Anything that reaches the user's own systemd units — `systemctl --user`, a runtime directory under `/run/user/<uid>`, a user service such as a secret store's unlock cache — needs a session. systemd ships the tool that makes one.

From inside the container, as root:

```
machinectl shell <username>@
```

From the Incus host:

```
incus exec -t <container> -- machinectl shell <username>@
```

> 📝 **Note** - The `-t` is required. `machinectl` allocates a PTY for the session, and without a TTY on the Incus client it connects and terminates immediately.

`machinectl shell` takes the command path first — there is no `--` separator:

```
machinectl shell <username>@ /bin/bash -c "<command>"
```

Confirm the session exists:

```
loginctl show-session self -p Name --value
systemctl --user is-system-running
```

The session lasts as long as the shell does. When you exit, the user manager stops with it and the user units' state goes away — nothing survives the logout unless the account lingers:

```
loginctl enable-linger <username>
```

Lingering keeps the user manager alive across logouts so its units run with nobody logged in. It also keeps them alive after the account is offboarded, so it has to be cleared as part of offboarding.
