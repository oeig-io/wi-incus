---
name: incus-project-management
description: Instructions for creating and configuring incus projects with dedicated networks and profiles
compatibility: opencode
metadata:
  type: tool
  original_file: incus-project-management-tool.md
  category: configuration
  scope: incus
---

# Incus Project Management

The purpose of this tool is to create and configure incus projects with dedicated networks and default profiles.

This is important because projects provide resource isolation between teams or workloads while sharing the same incus host.

## Create a Project

A project requires three resources: the network, the project itself, and a default profile with root disk and NIC.

### Naming Convention

| Resource | Pattern | Example |
|----------|---------|---------|
| Project | `<name>` | `acme` |
| Network | `<name>-net` | `acme-net` |

### Step 1: Create the Network

```bash
incus network create acme-net
```

This creates a managed bridge in the `default` project with auto-assigned IPv4/IPv6 subnets and NAT enabled.

### Step 2: Create the Project

```bash
incus project create acme \
  --config features.images=true \
  --config features.profiles=true \
  --config features.storage.volumes=true \
  --config features.storage.buckets=true
```

> **📝 Note** - `features.networks` is intentionally left at the default (`false`). The network lives in the `default` project so the admin manages all networks in one place.

### Step 3: Configure the Default Profile

New projects with `features.profiles=true` get an empty default profile. Containers need a root disk and NIC to launch:

```bash
incus profile device add default root disk path=/ pool=default --project acme
incus profile device add default eth0 nic network=acme-net --project acme
```

### Verify

```bash
incus project list
incus network list
incus profile show default --project acme
```

## List Projects

```bash
incus project list
```

## Switch Projects

```bash
incus project switch acme
```

## Restrict a Project

To limit resource consumption or block security-sensitive features:

```bash
incus project set acme restricted=true
incus project set acme limits.containers=10
```

See `incus project set --help` for available options.

## Delete a Project

A project must be empty (no instances, images, profiles beyond default, or volumes) before deletion:

```bash
incus project delete acme
```

> **⚠️ Warning** - Deleting a project does not remove the companion network. Remove it separately with `incus network delete acme-net` if no longer needed.

Tags: #role-system-admin
