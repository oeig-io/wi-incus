---
name: incus-export-import-tool
description: Export an Incus instance or custom storage volume to a backup file — or stream it straight to rsync.net, age-encrypted, with no local copy — and import it back to restore or recover. Use when someone asks to export, back up to a file, send a container off-host, or restore/import a backup. To choose between export, snapshot, and copy, start with incus-backup-task.
compatibility: opencode
metadata:
  type: tool
  original_file: incus-export-import-tool.md
  category: backup
  scope: incus
---

# Incus Export and Import

The purpose of this tool is to turn an Incus instance or custom storage volume into a portable backup file, and to turn that file back into a running instance or volume.

This is important because an export is the only Incus backup that leaves the host and restores on any other Incus host. Its most important rules hold for both directions, so this tool covers both. Everything here was tested against an Incus 6.0 server.

## Before You Start

- **Client version matters.** Check with `incus version`. Clients from 7.0 on accept `-` for stdin/stdout everywhere. A 6.0 client accepts `-` only for instances. For volumes, use `/dev/stdout -q` and `/dev/stdin`, or the client writes a file literally named `-`.
- **Server disk.** If the server lists `direct_backup` (`incus query <remote>:/1.0 | grep -w direct_backup`), it streams the export without buffering it. Otherwise it writes the full tarball under `/var/lib/incus/backups/` first, which needs free space the size of the backup.
- **Projects.** Add `--project <project>` to every command when the instance does not live in the remote's current project.

## What an Export Contains

| | Instance export | Volume export |
|---|---|---|
| Data | Root filesystem | Volume contents |
| Snapshots | Included as **full copies**. With `--optimized-storage`, only the changes between them | Same |
| Config | Included, **with disk devices pointing at the original volumes** | Volume config |
| Attached custom volumes | **Not included.** Export each one separately | — |

| Flag | Effect |
|------|--------|
| `--instance-only` / `--volume-only` | Leave out snapshots: a smaller file that restores only the current state |
| `--optimized-storage` | Driver-native format, which restores only to a pool with the same driver (e.g. ZFS) |
| `--compression none\|<algorithm>` | Default is gzip |
| `--root-only` | Leave out dependent volumes. Requires a 7.0 client |

## Export

```bash
# To a file
incus export <remote>:<instance> <instance>.tar.gz --instance-only
incus storage volume export <remote>:<pool> <volume> <volume>.tar.gz --volume-only

# To stdout (7.0+ client: "-" works for volumes too)
incus export <remote>:<instance> - --instance-only
incus storage volume export <remote>:<pool> <volume> /dev/stdout -q --volume-only
```

## Stream to rsync.net, Encrypted

An off-host export is always encrypted. `age` encrypts inside the pipe, so rsync.net, and anyone who reaches the account, holds only ciphertext. Encrypting needs only public keys, so the host that runs the export holds no secret.

Stream into `dd` on rsync.net. Plain `rsync` cannot read a stream, and rsync.net's restricted shell rejects pipes and redirects, so the pipe stays on your side. `tee` hashes the ciphertext in flight. The transfer is done only when the stored file's hash matches, because `dd` leaves a truncated file if the pipe breaks.

```bash
set -o pipefail
RN=ACMEs1@ACMEs1.rsync.net
F=<machine>/incus/<instance>-$(date -u +%Y%m%dT%H%M%SZ).tar.gz.age   # dir must exist: ssh $RN mkdir <machine>/incus

incus export <remote>:<instance> - --instance-only \
  | age -R <recipients-file> \
  | tee >(sha256sum | cut -d' ' -f1 > local.sha) \
  | ssh "$RN" dd of="$F" bs=4M

ssh "$RN" sha256 "$F"    # must equal: cat local.sha
```

Stream volumes the same way, swapping in the volume export command above. Account setup and remote path rules live in `rsync-net-backup-tool`.

> ⚠️ **Warning** - An encrypted backup is only as recoverable as its keys. Encrypt to at least two recipients, including an escrow key kept off any working machine (see `age-repo-encrypted-file-tool` → "The Escrow Recipient"). Prove a restore with the escrow key, not just your own. Key provisioning lives in `age-shared-secret-tool`.

## Import

```bash
# From a file, or from stdin with "-"
incus import <remote>: <file> [<new-name>]
incus storage volume import <remote>:<pool> <file> [<new-name>]      # 6.0 client: /dev/stdin, not "-"

# Straight from rsync.net: decrypt inside the pipe
ssh "$RN" cat "$F" | age -d -i ~/.config/age/keys.txt | incus import <remote>: - <new-name>
ssh "$RN" cat "$F" | age -d -i ~/.config/age/keys.txt | incus storage volume import <remote>:<pool> /dev/stdin <new-name> -q
```

A key that is not a recipient fails with `no identity matched any of the recipients`, before Incus reads anything.

An imported instance is created **stopped**. Before starting it:

> ⚠️ **Warning** - Its disk devices still name the **original** volumes. On the same host, starting it mounts the original's live drives. Import the volumes first, then re-point each device (`incus config device remove`, then `incus storage volume attach` with the restored volume).

> ⚠️ **Warning** - It keeps the original's MAC address. If the original is still running on the same network, the start fails with "MAC address … already defined". Clear it with `incus config unset <instance> volatile.eth0.hwaddr`.

If the copy may ever run alongside the original, it also inherits the original's identity and credentials. Scrub them first with `incus-instance-clone-tool`.

## References

- Upstream how-tos, in the `incus` repo: `doc/howto/instances_backup.md`, `doc/howto/storage_backup_volume.md`
- Export code paths: `cmd/incus/export.go`, `cmd/incus/storage_volume.go`

Tags: #tool #incus #backup #export #import
