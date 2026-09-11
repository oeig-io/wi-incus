---
name: incus-environment-management-task
description: Provision and maintain host-level Incus artifacts — storage pools, networks, profiles, images, projects — and the standards every container on the host inherits, including stable IPv6 addressing, incus-admin access, and the /opt/changelog.md record. Use when setting up a new Incus host, and when a container accumulates unexpected or churning IPv6 addresses.
---

# Incus Environment Management Task

The purpose of this task is to manage the host OS and Incus infrastructure artifacts.

## Scope

This task covers host-level Incus resources:

- Storage pools (e.g., local S3 endpoints)
- Networks
- Profiles
- Images
- Projects

For project creation and configuration, see incus-project-management-tool. For remote access and project-scoped user management, see incus-remote-management-tool.

Container-level management is handled by application-specific tasks (see idempiere-environment-management-task).

## Host Provisioning

Run these commands once when setting up a new incus host.

### Disable IPv6 Temporary Addresses

Containers should use stable IPv6 addresses, not random temporary ones. This
takes **two** settings, one per layer — the host profile alone does not hold.

Set the host default profile, once per host:

```bash
incus profile set default linux.sysctl.net.ipv6.conf.all.use_tempaddr=0 linux.sysctl.net.ipv6.conf.default.use_tempaddr=0
```

Then every NixOS payload must disable them in its own configuration:

```nix
networking.tempAddresses = "disabled";
```

The payload half is the load-bearing one. NixOS defaults
`networking.tempAddresses` to `"default"` whenever IPv6 is enabled — generate
temporary addresses *and prefer them* as the source address — and re-applies
that at every boot and `nixos-rebuild switch`, overriding whatever the profile
set when the container started. A payload that skips it grows a fresh IPv6
address on every rotation interval no matter how the host is provisioned.

Older payloads express the same thing as `boot.kernel.sysctl` entries forcing
`net.ipv6.conf.{all,default}.use_tempaddr` to `0`. That form is equivalent and
compliant; prefer the option above in new work, because it owns the setting
rather than fighting the module that owns it.

## User Access Management

To allow a user to run incus commands without sudo, add them to the `incus-admin` group.

### Add User to Incus Group

```bash
sudo usermod -aG incus-admin <username>
```

The user must log out and back in for the group membership to take effect.

### Verify Group Membership

```bash
groups <username>
```

The output should include `incus-admin`.

## Changelog Requirement

The host must maintain a changelog at `/opt/changelog.md`. This file tracks all Incus infrastructure changes.

### Changelog Format

```markdown
# Changelog

## YYYY-MM-DD
- Description of change
- Another change made on this date

## YYYY-MM-DD
- Earlier changes
```

### Changelog Guidelines

- Add entries at the top (newest first)
- Use date headers for each day changes are made
- Itemize each infrastructure change under the date
- Keep descriptions concise but specific

Tags: #role-system-admin

