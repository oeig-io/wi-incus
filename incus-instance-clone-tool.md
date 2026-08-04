---
name: incus-instance-clone
description: Clone an Incus instance and scrub the runtime identity it inherits before first boot, so the copy cannot impersonate its source or rejoin a mesh/VPN. Also the way to read or edit any STOPPED container's filesystem offline. Use when copying a production container, when a clone must not join a network as its source, or when a container will not boot and needs an offline fix.
compatibility: opencode
metadata:
  type: tool
  original_file: incus-instance-clone-tool.md
  category: provisioning
  scope: incus
---

# Incus Instance Clone

## TOC

- [Summary](#summary)
- [Two Phases: Machine Identity, Then Sanitization](#two-phases-machine-identity-then-sanitization)
- [Check for a Dedicated Clone Script First](#check-for-a-dedicated-clone-script-first)
- [Naming Clones](#naming-clones)
- [Land Clones in Their Own Project](#land-clones-in-their-own-project)
- [Offline Access to a Stopped Instance](#offline-access-to-a-stopped-instance)
- [Copy Flags That Matter](#copy-flags-that-matter)
- [What a Clone Inherits](#what-a-clone-inherits)
- [Scrub Data, Never Shadow Config](#scrub-data-never-shadow-config)
- [Verify the Clone](#verify-the-clone)

## Summary

The purpose of this tool is to produce an Incus clone that is a *new* machine rather than a second copy of an existing one, and to give you offline access to a stopped container's filesystem.

This is important because `incus copy` reproduces the source's **runtime identity** along with its software. Peer keys, SSH host keys, machine IDs, and credentials were established once, by hand or by a daemon, and nothing regenerates them just because the volume was duplicated. Boot such a clone unmodified and it authenticates to your infrastructure *as the source* — joining a mesh under the source's key, answering to the source's SSH fingerprint, or claiming to be a production system. Scrubbing happens **before first boot**, because the damage is done in the first few seconds of the daemon starting.

## Two Phases: Machine Identity, Then Sanitization

Cloning has two distinct phases with different owners. Conflating them is how a clone ends up looking finished while still being unsafe.

| Phase | Makes it | Scope | Owner |
|-------|----------|-------|-------|
| **1 · Machine identity** | a distinct *machine* | peer identity, SSH host keys, machine-id, hostname, production flags | generic — this skill |
| **2 · Application sanitization** | a safe *non-production system* | credential rotation, PII obfuscation, environment badging, integration credentials, background processors | application-specific — the owning repo |

**Phase 1 completing does not make a clone safe to expose.** A clone that has passed phase 1 is still holding the source's database passwords, un-obfuscated production data, and production badging. Say so explicitly when you report a clone as ready, so nobody infers more safety than was delivered.

## Check for a Dedicated Clone Script First

⚠️ **Warning** - Before cloning any long-lived or production instance, look for a clone script that its owning repo ships. Ad-hoc cloning of an instance that has one will miss steps that repo knows about and you do not.

A generic clone handles what is generic. An application-specific instance carries more — an application-level production flag, a licence registration, a scheduled job that writes to a shared destination. Only the owning repo knows that list.

Check in this order:

1. **The owning `host-*` or `install-*` repo** for a clone script (e.g., `host-idempiere/clone.sh`) — read its `--help` and use it.
2. **That repo's `AGENTS.md` / `CLAUDE.md` / `README.md`** for cloning guidance or prohibitions.
3. **Only if neither exists**, clone ad-hoc with the steps below — and report to the owner that a dedicated script would be worth writing.

| Instance | Dedicated script |
|----------|------------------|
| `idempiere-00` (production iDempiere) | `host-idempiere/clone.sh` |

> 📝 **Note** - Add a row when a repo gains a clone script, so the next actor finds it instead of improvising.

## Naming Clones

```
<source-base>-clone[-<label>]-NN
```

- **source-base** is the source name minus any trailing `-NN`, so `idempiere-00` yields `idempiere-clone-NN` and `id-01` yields `id-clone-NN`.
- **NN** starts at `01` and takes the lowest free value, allocated per prefix (a labelled series numbers independently).
- **Never `-00`.** House convention reserves that suffix for a blessed production singleton; a clone must never wear it.

Incus enforces its own rules on instance names (`shared/validate.IsHostname`), so validate before copying rather than failing mid-operation:

| Rule | Value |
|------|-------|
| Length | 1–63 characters |
| Charset | letters, digits, `-` only — **no underscores, no dots** |
| Must not start or end with | `-` |
| Must not be | only digits |

63 characters is generous: `idempiere-clone-uat-01` is 22, leaving 41 for a longer label.

## Land Clones in Their Own Project

Put clones in a dedicated project rather than beside their source:

```bash
incus --project <source-project> copy <source> <target> \
  --target-project <clone-project> --instance-only
```

**What the project buys you.** A name filter in the source's project stops matching clones, so bulk operations cannot reach production. Without this, adopting a `<source>-clone-*` name means `incus list <source>` returns production *and* every clone — and a loop over that list will stop, restart, or reconfigure the real system.

**What the project does *not* buy you.** Projects isolate the management API, not packets. Nor does giving the project its own bridge isolate traffic on its own: Incus managed bridges route to each other through the host, so a clone on its own subnet can still reach the source's services. Verify rather than assume — a container on a separate bridge reaching the source's database port is the expected result, not a misconfiguration. Packet isolation requires explicit network ACLs (`incus network acl`).

Create the clone project once, mirroring the source project's feature flags. Per house convention the network lives in the `default` project so one admin manages all networks:

```bash
incus network create <clone-project>-net
incus project create <clone-project> \
  --config features.images=true --config features.profiles=true \
  --config features.storage.volumes=true --config features.storage.buckets=true
incus --project <clone-project> profile device add default root disk path=/ pool=default
incus --project <clone-project> profile device add default eth0 nic network=<clone-project>-net
```

> ⚠️ **Warning** - Put `--project` **before** the subcommand, as above. Every project has its own profile named `default`; a trailing `--project` that gets lost off the end of a long command edits the *current* project's `default` profile instead — which may be the one production depends on.

## Offline Access to a Stopped Instance

`incus file` works against a **stopped** container. This is the mechanism that makes pre-first-boot scrubbing possible, and it is equally the way to repair a container that will not boot.

The daemon does not need a guest agent for this: when the instance is not running it mounts the root filesystem host-side, serves SFTP over it, and unmounts when the session ends.

**Targeted edits (preferred in scripts)** — no mount lifecycle to clean up on failure:

```bash
incus file delete <instance>/var/lib/some-daemon/identity.json
incus file pull   <instance>/etc/some.conf ./some.conf
incus file push   ./some.conf <instance>/etc/some.conf
incus file edit   <instance>/etc/some.conf
```

**Browsable mount (preferred for exploration)** — needs `sshfs`, and the instance path must end in `/`:

```bash
mkdir -p "${TMPDIR:-/tmp/$USER}/rootfs"
incus file mount <instance>/ "${TMPDIR:-/tmp/$USER}/rootfs"   # blocks; background it
# ... inspect and edit ...
umount "${TMPDIR:-/tmp/$USER}/rootfs"
```

> ⚠️ **Warning** - `incus storage volume file ...` looks like the same capability but is **not**. It only serves `type=custom` volumes; against an instance rootfs it fails or resolves the wrong volume. Use `incus file ...` for instances.

Scratch mount points belong under your private `TMPDIR`, never bare `/tmp` — see the `host-ai-user` shared-`/tmp` standard.

## Copy Flags That Matter

```bash
incus copy <source> <target> --instance-only
```

| Flag | When |
|------|------|
| `--instance-only` | **Normally yes.** Omits the source's snapshots; without it every source snapshot is copied and carries the source's name. |
| `--target-project <p>` | **Normally yes.** Lands the clone in its own project — see above. |
| `--no-profiles` | **Only when you want an isolated clone with no NIC.** It is *not* the safety mechanism and is not the default. |

`--no-profiles` is easy to over-reach for. The scrub is protected by the clone being **stopped** — a NIC merely *defined* on a stopped container carries no traffic. Withholding the network protects nothing during the scrub and only adds a step afterward. Use it when a permanently networkless clone is the goal, not as caution.

Copying a **running** instance is fine; Incus snapshots it internally. Real cost tracks used space, not the disk's declared size.

> ⚠️ **Warning** - **Look at what is scheduled to fire before you copy.** The copy itself is atomic, so a job running mid-copy cannot corrupt the source or the clone's kept data. What it does do is capture that job's *partial working state* — a half-written staging directory, a temp file whose cleanup handler will never run on the clone because the process does not continue there — and compete for I/O with the job on an instance you care about. Neither is dangerous; both make the clone messier and the copy slower.

```bash
# What is armed, when it last ran, and what fires next
incus exec <source> -- systemctl list-timers --all

# Is anything heavy running right now?
incus exec <source> -- systemctl list-units --type=service --state=running
```

Prefer a window with no imminent fire. When one is close, either wait it out or expect to tidy the clone afterward — and say which you chose when you report the clone. This is a courtesy check, not a gate: do not build a hard refusal around it, because a gate on a non-safety issue only teaches people to bypass gates.

> ⚠️ **Warning** - **A copy inherits the source's instance config.** Anything that must differ has to be set *explicitly* — never inferred from absence. The trap is `security.protection.delete`: clone a protected production singleton and the clone arrives protected too, so a script that only sets the flag "when asked" silently produces protected clones. State the intended value on every path, then verify it.

```bash
# State intent explicitly, both ways
incus --project <p> config set <target> security.protection.delete=false   # disposable clone
incus --project <p> config set <target> description="clone of <source> taken YYYY-MM-DD; <purpose>" -p
```

Keep delete protection **rare**. Its entire value is that encountering it makes someone stop; if clearing it becomes routine muscle memory it protects nothing, including production. Default clones to unprotected.

## What a Clone Inherits

Scrub each of these **while the clone is stopped**. Deleting a regenerable artifact is correct: the platform recreates it, keyed to the new machine.

| Artifact | Why it must go | Regenerated by |
|----------|----------------|----------------|
| Mesh/VPN peer identity (e.g., `/var/lib/netbird/config.json`, `state.json`, `active_profile.json`) | Holds the private key and management URL that let the clone rejoin the mesh **as the source** | Daemon, on next start — fresh key, no registration |
| SSH host keys (`/etc/ssh/ssh_host_*`) | Any client that trusted the source's fingerprint silently trusts the clone | `sshd` / NixOS activation, on next boot |
| `/etc/machine-id` | systemd identity; duplicates confuse journald and DHCP | systemd, on next boot |
| Stored credentials (e.g., a `.pgpass`, API keys, service tokens) | **Inherited credentials are inherited identity.** They still authenticate against the *source's* systems | Nothing — rotation is phase 2 work |
| Application-level production flags | A clone that believes it is production hardens its UI and arms guards meant for the real system | Nothing — must be set deliberately |
| Scheduled jobs (e.g., a `systemd` backup timer) | **An armed timer fires on the clone too.** It writes real data — including anything it dumps, such as credentials — onto a box that phase 2 has not yet sanitized, and burns disk on a throwaway | Nothing — deactivation is phase 2 work |

**A declaratively-defined scheduled job cannot be switched off imperatively.** On a config-managed host (NixOS `wantedBy = [ "timers.target" ]`, or any unit a config manager owns), `systemctl disable` is undone by the next rebuild — and clone scripts commonly *run* a rebuild to reconcile the clone's rendered state. Deactivation therefore has to be declarative, which on NixOS means importing an override module on the clone; see the `nixos-best-practices` skill for why an added module stays inert until it is wired.

Stopping the service on the source is **not** equivalent to scrubbing. A mesh client's `down` verb stops the tunnel and deliberately leaves the identity on disk so a later `up` needs no new key. That surviving identity is exactly what a clone inherits — see the `netbird-connect` skill for the NetBird specifics.

Scrubbing a mesh identity also removes the **management URL**, not just the key. A self-hosted deployment therefore needs it supplied again on the clone (`netbird up --management-url ... --setup-key ...`), or the daemon falls back to the vendor's public default.

**Do not scrub what the platform already fixed.** Incus rewrites what it owns during a copy. On NixOS it regenerates the autogenerated hostname module, so the clone's *declarative* hostname is already correct while the *rendered* `/etc/hostname` is still stale. The fix is to rebuild, not to edit the rendered file — see the next section.

## Scrub Data, Never Shadow Config

**Invariant: remove the offending runtime data; never add runtime data that contradicts declarative config.**

On a declaratively-managed host, config is the single source of truth and the files on disk are its output. Two ways to stop a cloned daemon from connecting:

| Approach | Verdict |
|----------|---------|
| Delete the runtime identity the daemon would have reused | ✅ Correct — restores the state a fresh install would have |
| Drop in an override file that contradicts the rendered config | ❌ Wrong — declared config and observed behaviour now disagree, with the reason in an unversioned file |

The second is tempting because it looks like defence in depth. It is drift: the next person reads the config, predicts one behaviour, and gets another. If a clone genuinely must behave differently *by policy*, change the declarative config and rebuild.

The same logic applies to stale rendered artifacts. When a clone's hostname is wrong because the rendered file predates the copy, **rebuild to regenerate it** — do not hand-edit. Hand-editing produces the right value with no provenance, and the next rebuild silently reverts it.

On NixOS the system enforces this for you: `/etc/hostname` is a symlink into the read-only `/etc/static/` tree, so it cannot be edited at all. When a file resists being written, that is usually the platform telling you it is output rather than input — find the input.

> ⚠️ **Warning** - Rebuilding is necessary but **not sufficient** for the hostname. systemd reads `/etc/hostname` once, at boot, to set the running (*transient*) hostname. A clone's first boot therefore adopts the source's name, and `nixos-rebuild switch` then corrects the *static* hostname without touching the transient one. `hostnamectl` reports the split plainly:
>
> ```
> Transient hostname: <source-name>   ← running kernel hostname
>    Static hostname: <clone-name>    ← /etc/hostname, correctly rendered
> ```
>
> **Restart the clone after the rebuild** so the system applies its own declared config. `hostnamectl set-hostname` also works but imperatively duplicates what boot does, and the restart additionally proves the clone boots clean under its new identity.

Order matters for anything generated *from* the hostname. Host keys created on the first boot carry the source's name in their comment (`root@<source>`), which looks alarmingly like a failed scrub even though the key material is unique. Re-delete them just before the restart so they regenerate under the correct name.

A clone with no NIC still rebuilds fine. Nix logs failed `cache.nixos.org` lookups and then builds the handful of changed derivations from the local store — the warnings are noise, not failure.

```bash
# Reconcile rendered state with declarative config (NixOS example)
incus --project <p> exec <clone> -- sudo nixos-rebuild switch
```

## Verify the Clone

Prove the clone is a new machine rather than asserting it. Compare each value against the source and report a mismatch loudly:

```bash
incus --project <p> exec <clone> -- hostname                    # the clone's name, not the source's
incus --project <p> exec <clone> -- cat /etc/machine-id         # differs from source
incus --project <p> exec <clone> -- ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
incus --project <p> config get <clone> security.protection.delete   # matches what you intended
```

For a mesh client, the daemon should be **running but unregistered** — enabled by config, with no valid identity. A clone reaching the *source's* management URL means the scrub missed something.

> ⚠️ **Warning** - An unregistered mesh client reports one of **two** shapes, and a check that matches only one will silently pass while showing nothing. NetBird prints `Daemon status: NeedsLogin` when there is no config at all, or a `Management:` / `Signal:` table when config exists but is not connected. Match both, and print an explicit "could not read" rather than emitting nothing.

A verification that can silently output nothing is worse than none, because it reads as success. The same applies to any state you *set*: assert the outcome afterward instead of trusting the command that set it.

Tags: #incus #clone #provisioning #nixos
