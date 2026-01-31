# Incus Environment Management Task

The purpose of this task is to manage the host OS and Incus infrastructure artifacts.

## Scope

This task covers host-level Incus resources:

- Storage pools (e.g., local S3 endpoints)
- Networks
- Profiles
- Images
- Projects

Container-level management is handled by application-specific tasks (see idempiere-environment-management-task).

## Host Provisioning

Run these commands once when setting up a new incus host.

### Disable IPv6 Temporary Addresses

Containers should use stable IPv6 addresses, not random temporary ones:

```bash
incus profile set default linux.sysctl.net.ipv6.conf.all.use_tempaddr=0 linux.sysctl.net.ipv6.conf.default.use_tempaddr=0
```

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
