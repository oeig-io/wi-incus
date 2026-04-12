---
name: incus-container-management
description: Instructions for how to configure incus containers
---

# Incus Container Management

## Set Container Description

```
incus config set <some.container> description="some description" -p
```
Note: The -p (or --property) flag is required because description is an instance-defined property/metadata, not a configuration setting

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
