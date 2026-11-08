---
name: incus-backup-task
description: Choose how to back up an Incus instance or custom storage volume — snapshot, copy to a second Incus server, or export to a file — by what the restore has to survive. Use when someone asks how to back up or protect a container, get it off-host, or whether snapshots are enough. To confirm an existing backup actually ran, use backup-audit-task.
compatibility: opencode
metadata:
  type: task
  original_file: incus-backup-task.md
  category: backup
  scope: incus
---

# Incus Backup Task

The purpose of this task is to choose the right way to back up an Incus instance or custom storage volume.

This is important because Incus offers three methods, and each restores differently. A backup chosen for convenience rather than for what it must survive can vanish along with the thing it protects.

## Options

| Option | Protects against | Restore | Tool |
|--------|------------------|---------|------|
| Snapshot | A bad change. Not pool or host loss — it lives in the same pool | Seconds, in place | 🔜 `incus-snapshot-tool` |
| Copy to a second Incus server | Host loss | Fast — start the copy. `--refresh` keeps it current by sending only changes | `incus-remote-management-tool`, `incus-instance-clone-tool` |
| Export to a file | Host loss, without needing Incus at the destination | `incus import` on any Incus host | `incus-export-import-tool` |

> 📝 **Note** - 🔜 marks a tool not yet written. Until it exists, use the upstream docs: for snapshots, `doc/howto/instances_backup.md` in the `incus` repo; for borg, rsync.net's [borg usage notes](https://www.rsync.net/resources/howto/remote_commands.html).

## Choosing

- **Rollback before a risky change** — snapshot.
- **Recovery after losing the host** — copy or export. A snapshot alone is never a backup.
- **Off-host storage with no Incus at the other end** (e.g. rsync.net) — export, always encrypted. The account setup lives in `rsync-net-backup-tool`.
- **Large data volumes that change slowly, off-host** — a full export on every run gets expensive. Use an incremental, encrypted file-level backup instead: 🔜 `borg-rsync-net-tool` (in `wi-backup`, since borg is not Incus-specific).
- **Most production instances need two** — scheduled snapshots for rollback, plus a copy or export for host loss.

## Decision Points

- **Attached custom volumes are not part of an instance backup.** Back each one up on its own, using the same option.
- **Snapshots inside a copy or export** — include them to carry history, or leave them out (`--instance-only`) for a smaller artifact that restores only the current state.
- **Optimized storage** — faster and smaller, but restores only to a pool with the same storage driver.
- **A copy that may ever boot** must be scrubbed first — see `incus-instance-clone-tool`.

## Completion

A backup is not done until it is verified. Run `backup-audit-task` against it.

Tags: #task #incus #backup
