---
name: incus-windows-tool
description: Install Windows in an incus VM using a driver-injected (repacked) ISO volume — reuse an existing project-scoped volume when present, or build and publish one with distrobuilder when not — plus the TPM 2.0 (vtpm), local-account, and guest-driver steps a Windows 11 install needs
compatibility: opencode
metadata:
  type: tool
  original_file: incus-windows-tool.md
  category: provisioning
  scope: incus
---

# Incus Windows VM Provisioning

The purpose of this tool is to make a Windows install ISO usable inside incus VMs.

This is important because a stock Microsoft ISO does not contain virtio drivers, so the Windows installer cannot see the VM's virtio disk and the install dead-ends at "Where do you want to install Windows?". We solve this once by repacking the ISO with the virtio drivers baked in, then publish it as a reusable storage volume.

## TOC

- [Decide: Build or Use](#decide-build-or-use)
- [Why Repack](#why-repack)
- [Build the Repacked ISO](#build-the-repacked-iso)
  - [Repack on the incus Host (Debian/Ubuntu)](#repack-on-the-incus-host-debianubuntu)
  - [Create a Repack VM (not a Container)](#create-a-repack-vm-not-a-container)
  - [Obtain the Source ISOs](#obtain-the-source-isos)
  - [Repack](#repack)
  - [Verify Driver Injection](#verify-driver-injection)
- [Publish the ISO as a Storage Volume](#publish-the-iso-as-a-storage-volume)
  - [Import Into the First Project](#import-into-the-first-project)
  - [Copy to Additional Projects](#copy-to-additional-projects)
- [Install Windows From the Volume](#install-windows-from-the-volume)
  - [Create the VM](#create-the-vm)
  - [TPM 2.0 Is Required](#tpm-20-is-required)
  - [Boot the Installer](#boot-the-installer)
  - [Finish Setup Without a Microsoft Account](#finish-setup-without-a-microsoft-account)
- [After Install](#after-install)
  - [Install Guest Drivers](#install-guest-drivers)
  - [Protect the VM From Deletion](#protect-the-vm-from-deletion)
- [Cleanup](#cleanup)

## Decide: Build or Use

Start here. The repacked volume uses an `-incus` suffix (e.g. `win11-25h2-incus`)
to signal that virtio drivers are already injected. Check whether one exists in the
target project before doing anything else:

```bash
incus storage volume list <pool> --project <project> --format csv | grep -i -- '-incus'
```

Route based on where the volume already lives — **rebuilding is the last resort**:

| Situation | Do this |
|-----------|---------|
| `-incus` volume is in the **target project** | Skip to [Install Windows From the Volume](#install-windows-from-the-volume) |
| `-incus` volume exists in **another project** | [Copy it in](#copy-to-additional-projects) server-side — do **not** rebuild |
| No `-incus` volume **anywhere**, or a new Windows build is needed | [Build the Repacked ISO](#build-the-repacked-iso), then publish |

```bash
# Find the volume across all projects (decide whether to copy vs. build)
for p in $(incus project list --format csv | cut -d, -f1 | sed 's/ (current)//'); do
  incus storage volume list <pool> --project "$p" --format csv 2>/dev/null \
    | grep -iq -- '-incus' && echo "present in: $p"
done
```

## Why Repack

Two facts about our environment drive this whole workflow:

- **Stock ISOs lack virtio drivers.** `distrobuilder repack-windows` injects the
  virtio storage and network drivers (`vioscsi`, `viostor`, `netkvm`, `vioser`)
  into both `boot.wim` (the installer) and `install.wim`, so the disk is visible
  during setup with no manual "Load driver" step.
- **Storage volumes are project-scoped.** Every project here has
  `features.storage.volumes=true`, so a volume in one project is invisible to
  instances in another. The repacked ISO must be copied into each project that
  needs it — see [Publish the ISO as a Storage Volume](#publish-the-iso-as-a-storage-volume).

## Build the Repacked ISO

The repack loop-mounts the source ISOs, so it needs kernel mount privileges.
Where that is possible decides where you run it:

- **Client is the incus host itself** — repack directly on the host. On
  Debian/Ubuntu: [Repack on the incus Host (Debian/Ubuntu)](#repack-on-the-incus-host-debianubuntu).
  On NixOS: run the [Repack](#repack) `nix shell` command on the host instead
  (with `sudo`, per the NixOS warning below), skipping the apt build and the VM.
- **Client is our unprivileged LXC container** (cannot loop-mount; a privileged
  container still lacks the host loop devices) — use a VM, the only reliable
  place to run the repack: [Create a Repack VM (not a Container)](#create-a-repack-vm-not-a-container).

### Repack on the incus Host (Debian/Ubuntu)

Install the repack dependencies and build distrobuilder from source (it lands
in `~/go/bin/distrobuilder`):

```bash
sudo apt install --no-install-recommends \
  libhivex-bin libwin-hivex-perl wimtools genisoimage rsync libguestfs-tools
sudo apt install -y golang-go make git

tar xzf distrobuilder-3.3.1.tar.gz && cd distrobuilder-3.3.1/ && make
```

Put the Windows ISO and the virtio-win ISO (see [Obtain the Source ISOs](#obtain-the-source-isos))
in the working directory, then repack with `sudo` (it loop-mounts) and import
straight into the pool — because the client is the server, no remote/token
round-trip is needed:

```bash
sudo ~/go/bin/distrobuilder repack-windows win11.iso win11-25h2-incus.iso
incus storage volume import <pool> win11-25h2-incus.iso win11-25h2-incus --type=iso
```

Verify the result before publishing — see [Verify Driver Injection](#verify-driver-injection)
(for the xorriso/wimlib commands, `apt install libisoburn wimtools` on the host).

### Create a Repack VM (not a Container)

`distrobuilder repack-windows` loop-mounts the source ISO, which requires kernel
mount privileges. Our incus client runs inside an unprivileged LXC container that
cannot loop-mount (no `/dev/loop*`), and a privileged container still lacks the
host loop devices. A **VM** has its own kernel and full loop support, so when the
client is a container it is the only reliable place to run the repack.

```bash
incus launch images:nixos/26.05 repack-vm --vm \
  -c limits.cpu=4 -c limits.memory=6GiB \
  -c security.secureboot=false \
  -d root,size=50GiB
```

> 📝 **Note** - The NixOS VM image requires `security.secureboot=false`. Wait for the
> agent before running `incus exec` (poll `incus exec repack-vm -- true`).

> ⚠️ **Warning** - On NixOS, `incus exec` starts with a minimal `PATH`. Prefix
> interactive work with `export PATH=/run/current-system/sw/bin:$PATH`, and always
> run `nix` through `sudo` (even as root) or the Nix store throws permission errors.
> When using `sudo nix`, pass `--extra-experimental-features "nix-command flakes"`
> on the command line because `sudo` strips the `NIX_CONFIG` environment variable.

### Obtain the Source ISOs

Get both ISOs onto the repack machine:

- **Windows ISO** — download from Microsoft, or attach an existing raw ISO volume
  to the VM and copy it to a file (`dd if=/dev/sr0 of=/root/win11.iso bs=4M`).
  Attaching a volume is host-local and far faster than pushing a multi-GB file
  through the client. See `incus-disk-management`.
- **virtio drivers ISO** — download the stable `virtio-win.iso` from the
  fedorapeople virtio-win archive directly on the repack machine (it has internet):
  `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso`.
  Keep it in the working directory — the repack uses it next, and the guest-driver
  install reuses it later.

### Repack

```bash
incus exec repack-vm -- bash -c '
export PATH=/run/current-system/sw/bin:$PATH
sudo nix --extra-experimental-features "nix-command flakes" \
  shell nixpkgs#distrobuilder nixpkgs#wimlib nixpkgs#hivex nixpkgs#cdrkit nixpkgs#libisoburn nixpkgs#rsync \
  --command distrobuilder repack-windows \
    /root/win11.iso /root/win11-25h2-incus.iso \
    --drivers /root/virtio-win.iso \
    --windows-version w11 --windows-arch amd64
'
```

### Verify Driver Injection

Confirm the install images survived and the storage driver is present in the
installer (`boot.wim` index 2 = Windows Setup) before trusting the ISO:

```bash
incus exec repack-vm -- bash -c '
export PATH=/run/current-system/sw/bin:$PATH
sudo nix --extra-experimental-features "nix-command flakes" shell nixpkgs#libisoburn nixpkgs#wimlib --command bash -c "
  xorriso -indev /root/win11-25h2-incus.iso -lsl /sources/ 2>/dev/null | grep -iE \"install.wim|boot.wim\"
  xorriso -osirrox on -indev /root/win11-25h2-incus.iso -extract /sources/boot.wim /tmp/boot.wim 2>/dev/null
  wimdir /tmp/boot.wim 2 2>/dev/null | grep -iE \"vioscsi|viostor|netkvm\"
"'
```

A genisoimage `install.wim.staging... No such file` warning during repack is benign
as long as `install.wim` (multi-GB) is present in the verification output.

> 💡 **Tip** - `--drivers` is optional: it defaults to `virtio-win.iso` in the
> working directory, so the flag can be omitted when the ISO is already there
> (distrobuilder resolves or fetches it when absent). Pass `--windows-version` and
> `--windows-arch` only when the ISO filename does not already encode them —
> distrobuilder auto-detects both from the filename.

## Publish the ISO as a Storage Volume

### Import Into the First Project

`incus storage volume import` uploads from the client filesystem to the server.
When you repacked **on the incus host**, the import shown there already published
the volume — skip straight to [Copy to Additional Projects](#copy-to-additional-projects)
if needed. Otherwise the file lives on the repack VM, and pulling it back through
the client is slow. Instead, let the **VM import the volume itself** over the
server's LAN. Grant the VM trust with a token and add the server as a remote
(see `incus-remote-management`), then:

```bash
incus exec repack-vm -- bash -c '
export PATH=/run/current-system/sw/bin:$PATH
sudo nix --extra-experimental-features "nix-command flakes" shell nixpkgs#incus --command \
  incus storage volume import <remote>:<pool> /root/win11-25h2-incus.iso \
    win11-25h2-incus --type=iso --project <project>
'
```

> 💡 **Tip** - When adding the remote inside the VM, use the server's IP, not the
> hostname embedded in the token (the VM may not resolve it), and remove the
> temporary trust certificate afterward.

### Copy to Additional Projects

Because volumes are project-scoped, copy the published volume server-side into every
other project that needs it (no re-upload):

```bash
incus storage volume copy <pool>/win11-25h2-incus <pool>/win11-25h2-incus \
  --target-project <project>
```

See `incus-project-management` for project concepts.

## Install Windows From the Volume

### Create the VM

Create an empty VM and attach the repacked ISO as a bootable CD-ROM:

```bash
incus init <vm> --empty --vm \
  -c limits.cpu=4 -c limits.memory=8GiB -d root,size=128GiB \
  -c security.secureboot=true -c image.os=Windows

incus config device add <vm> vtpm tpm

incus config device add <vm> install-iso disk \
  pool=<pool> source=win11-25h2-incus boot.priority=10
```

> ⚠️ **Warning** - Set `image.os=Windows` so incus disables unsupported virtual
> devices, uses local-time RTC, and switches IOMMU handling. Without it Windows 11
> commonly stalls or bluescreens. `boot.priority=10` is required or the firmware
> boots the empty disk and falls through to PXE.

> 💡 **Tip** - The root disk is the one size worth over-provisioning: 64GiB is the
> Microsoft minimum and fills up once Windows Update and Office land, and a root disk
> cannot be shrunk afterward. Start at 128GiB or more.

### TPM 2.0 Is Required

Windows 11 setup halts without TPM 2.0. The bare `vtpm tpm` device above hands the VM a
software TPM (swtpm); together with `security.secureboot=true` and `image.os=Windows`
that is the combination verified against Windows 11 Pro here.

> ⚠️ **Warning** - Do **not** add `path=/dev/tpm0` to a VM's tpm device. `path` and
> `pathrm` are container-only settings naming the device nodes bind-mounted into a
> container; incus accepts them on a VM and then ignores them. Recipes carrying
> `path=/dev/tpm0` succeed in spite of it, not because of it — a VM reaches its TPM
> through a swtpm socket incus creates on its own.

> ⚠️ **Warning** - `swtpm` must be installed on the **incus host** (the server), not on
> the operator's workstation. Without it the VM refuses to start with
> `Failed to validate environment: Required tool 'swtpm' is missing`.

> ⚠️ **Warning** - Removing the vtpm device **destroys the TPM state** — incus deletes
> the device's state directory, so the guest boots with a blank TPM: BitLocker drops to
> the recovery-key prompt and Windows Hello PINs are gone. Renaming the device does the
> same, because the state is keyed to the device name. Never "remove and re-add the
> vtpm" as a troubleshooting step on a VM already in use.

> 📝 **Note** - If Windows reports "No TPM found" despite the device being present, the
> host's EDK2/OVMF firmware may not perform the PCR measurements Windows expects — an
> upstream issue; try updating incus or the host firmware package.

### Boot the Installer

Start the VM **with the VGA console attached from power-on** so the SPICE viewer
opens immediately and you can catch the boot prompt on the very first attempt:

```bash
incus start <vm> --console=vga   # powers on AND attaches the SPICE display
```

Expect the installer image to take **three minutes or more** to reach the boot
prompt — this is normal, not a hang.

When the `remote-viewer` window appears, **click into it** to grab the keyboard, then
**hold the spacebar** so key-repeat lands inside the brief "Press any key to boot
from CD or DVD..." window. Release once "Windows Setup is loading files" / the spinner
appears, then complete the graphical installer.

> ⚠️ **Warning** - The "Press any key to boot from CD..." prompt is **graphical** — it
> renders only on the VGA/SPICE display. The serial console (`incus start <vm>
> --console`, no type) shows *only* the `BdsDxe ... Time out` / PXE-fallback lines and
> gives you **no way to catch the prompt**, so do not use it for the install. If you
> miss the prompt, the firmware loops (CD → empty disk → PXE → back to CD), giving
> unlimited retries — just keep the spacebar held.

> ⚠️ **Warning** - `--console=vga` (like `incus console <vm> --type=vga`) opens a
> **SPICE** display and needs a SPICE viewer installed on the machine that runs the
> command (the operator's workstation, not the server). incus auto-launches
> `remote-viewer` (from the `virt-viewer` package) or `spicy` (from `spice-gtk`) when
> present. If neither is installed, incus prints only a raw socket path
> (`spice+unix:///.../sockets/<id>.spice`) and no window opens. Install a viewer first:
>
> | Workstation OS | Install command |
> |----------------|-----------------|
> | Windows | Install **virt-viewer** from virt-manager.org (provides `remote-viewer`) |
> | macOS | `brew install virt-viewer` |
> | Debian/Ubuntu | `sudo apt install virt-viewer` |
> | Fedora/RHEL | `sudo dnf install virt-viewer` |

> 💡 **Tip** - To reattach the display to an **already running** VM, use
> `incus console <vm> --type=vga`. `incus start --console=vga` only works from a
> stopped VM.

The virtio disk is visible immediately because the drivers are baked in.

### Finish Setup Without a Microsoft Account

Current Windows 11 Pro builds offer no visible local-account path and dead-end at
"Let's add your Microsoft account". Bypass it from the installer itself:

1. [ ] At the Microsoft account screen, press **Shift+F10** to open a command prompt
1. [ ] Run `ms-cxh:localonly`
1. [ ] Enter the local user name, password, and security questions

Setup then completes to the desktop. Detach the ISO so the VM boots from disk:

```bash
incus storage volume detach <pool> win11-25h2-incus <vm>
```

## After Install

### Install Guest Drivers

Windows boots with a low fixed resolution until guest drivers are installed.
Import the virtio-win ISO as a project volume (it is the same ISO used for the
repack — see [Obtain the Source ISOs](#obtain-the-source-isos)), attach it as a
CD-ROM, run its guest-tools installer, reboot, then detach:

```bash
incus storage volume import <pool> virtio-win.iso virtio-win --type=iso
incus config device add <vm> drivers disk pool=<pool> source=virtio-win
# in Windows: run virtio-win-guest-tools.exe from the attached CD, then reboot
incus config device remove <vm> drivers
```

Alternatively download the ISO inside the guest (see `incus-disk-management` for
volume attach/detach mechanics).

> ⚠️ **Warning** - SPICE Guest Tools alone does **not** fix the display in our VMs; the
> virtio-win guest tools do. Install virtio-win first, and add SPICE Guest Tools only
> for the extras it provides (clipboard and folder sharing).

### Protect the VM From Deletion

A Windows VM is hand-built and cannot be recreated from an image, so guard it once the
install is good:

```bash
incus config set <vm> security.protection.delete=true
incus config get <vm> security.protection.delete   # expect: true
```

Clear it deliberately (`=false`) only when deletion is the confirmed intent.

## Cleanup

- Delete the repack VM once the volume is published (`incus delete repack-vm --force`).
- Remove any temporary trust certificate created for the VM import.
- Delete superseded raw (non-`-incus`) ISO volumes once the repacked volume is verified.

Tags: #role-incus-system-admin
