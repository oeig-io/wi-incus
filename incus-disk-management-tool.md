---
name: incus-disk-management-tool
description: Configure Incus disks — map a host directory into a container, and observe ZFS disk usage across the Incus storage pool
compatibility: opencode
metadata:
  type: tool
  original_file: incus-disk-management-tool.md
  category: configuration
  scope: incus
---

# Incus Disk Management

The purpose of this document is to configure and observe Incus disks. This is
important because every container we run is backed by one shared ZFS pool, so
knowing how to map a disk in — and how to read real headroom out — keeps that
single pool from surprising us.

## Map Disk Between Host and Container

```
X_INCUS_INSTANCE=container-name
X_DISK_SOURCE=/path/in/host
X_DISK_DESTINATION=/path/in/container
X_DISK_LABEL=some-description    #no spaces
incus config device add $X_INCUS_INSTANCE "$X_INCUS_INSTANCE"-"$X_DISK_LABEL" disk source=$X_DISK_SOURCE  path=$X_DISK_DESTINATION shift=true
```

## Observe Disk Usage

Our Incus storage pool (`default`) is a single ZFS pool named `tank`, with every
container, VM, image, and custom volume living under `tank/incus/...` and mounted
beneath `/var/lib/incus/storage-pools/default/`. Because it is one shared pool,
generic per-filesystem tools mislead — read usage from ZFS, not from `df`.

> ⚠️ **Warning** - `df -h` reports the same pool-wide free space against every
> dataset, making each container look like it has the whole pool free. Trust
> `zfs list` AVAIL (which honors a dataset's quota) and `zpool list` CAP/FRAG
> instead. Profiling is read-only — none of the recipe below changes state.

Run this read-only recipe to profile the pool consistently:

```bash
# 1. Pool-level headroom: CAP is % full, FRAG erodes usable free space
zpool list -v tank

# 2. Live datasets by space, largest last (skip snapshots and soft-deleted)
zfs list -t filesystem -o name,used,avail,refer,compressratio,quota tank/incus \
  | grep -v '/deleted/' | sort -k2 -h

# 3. Space held by snapshots (unique bytes, summed)
zfs list -t snapshot -H -o used -p tank \
  | awk '{s+=$1} END{printf "snapshots hold %.2f GiB\n", s/1024/1024/1024}'

# 4. Space stranded in Incus soft-delete tombstones (see note below)
zfs list -o name,used tank/incus/deleted
```

Reading the output against our setup:

- **`tank/incus/deleted/`** holds datasets Incus has soft-deleted (removed
  containers and old images) but not yet purged. This space is not reclaimable by
  normal use and does not appear in any container — treat it as a cleanup
  candidate, not free space, and reclaim it through Incus, never with a raw
  `zfs destroy`.
- **AVAIL varies by dataset** because some containers carry a ZFS quota and some
  do not: a quota'd dataset shows its remaining allowance, an unquota'd one shows
  the whole pool's free space. Compare against the `quota` column, not against
  each other.

For everything general about ZFS (what fragmentation is, snapshot vs. clone
accounting, capacity thresholds), rely on standard ZFS knowledge rather than
repeating it here. To attach or detach the underlying disk device, see the
Map Disk Between Host and Container section above.
