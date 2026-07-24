---
name: incus-desktop-vm
description: Enable a desktop environment in an Incus Linux virtual machine and connect to its graphical (VGA) console — COSMIC option paths and the cold-start attach pattern
compatibility: opencode
metadata:
  type: tool
  original_file: incus-desktop-vm-tool.md
  category: configuration
  scope: incus
---

# Incus Linux Desktop VM Tool

The purpose of this tool is to get you into a COSMIC desktop login screen inside an Incus **virtual machine** (not a container) as fast as possible.

This is important because two non-obvious details block the way: the NixOS COSMIC option paths differ from the wiki's, and the graphical greeter only appears if you attach the VGA console **as the VM boots** — attaching to an already-running VM drops you on a text login instead.

> 📝 **Note** — This skill covers COSMIC on NixOS, the first desktop VM we documented. Add sibling `### <DE> (<distro>)` sections as new scenarios come up. For GUI apps in a **container**, use `incus-profile-gui-tool.md`; for a **Windows** VM, use `incus-windows-tool.md`.

## Launch the VM

A NixOS `--vm` needs **both** `security.secureboot=false` and `security.nesting=true` to launch — omit either and the VM never comes up. Everything else (profile, memory, CPU, disk) is machine-specific sizing and contributes nothing to COSMIC.

```bash
incus launch images:nixos/26.05 cosmic-01 --vm \
  -c security.secureboot=false \
  -c security.nesting=true \
  -c limits.memory=6GiB -c limits.cpu=2 \
  -d root,size=30GiB
```

> 📝 **Note** — `incus-windows-tool.md` owns the `security.secureboot=false` requirement for the NixOS VM image; this skill adds that `security.nesting=true` is also required to launch.

## The three critical lines

On NixOS, enable COSMIC with these exact option paths and add Flatpak so the app store installs:

```nix
services.desktopManager.cosmic.enable = true;      # COSMIC desktop + greeter
services.displayManager.cosmic-greeter.enable = true;
services.flatpak.enable = true;                    # gates cosmic-store (the Flathub app browser)
```

Then `sudo nixos-rebuild switch`.

- **Use these paths, not the wiki's `services.cosmic.*`** — the merged nixpkgs module (PR #267099) registers under the conventional `services.desktopManager.*` / `services.displayManager.*` namespaces. The wiki shorthand fails with `The option 'services.cosmic' does not exist.`
- **`cosmic-store` is gated on Flatpak.** It lives in a `lib.optionals config.services.flatpak.enable […]` block, so without `services.flatpak.enable` the app store is missing. The other COSMIC apps (files, edit, term, settings, launcher) install regardless.
- **Do not** also enable a competing desktop/display manager (gnome, gdm, sddm, lightdm) — they fight cosmic-greeter.

Add the Flathub remote once (runtime command, not a NixOS option) so `cosmic-store` can browse Flathub:

```bash
flatpak remote-add --user flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

## Create a login user

The greeter needs an account to log into. Create one imperatively — see
`wi-nixos/nixos-best-practices-tool.md` → "Creating Users as Data (Imperative useradd)".

## Connect: cold-start with the VGA console

This is the piece that made the login screen appear. Attach the VGA console **on start**, not after boot:

```bash
incus stop cosmic-01
incus start cosmic-01 --console=vga
```

`--console=vga` attaches the graphical framebuffer as the VM boots, so the kernel messages give way to the cosmic-greeter login. The default `incus console <vm>` is the **serial** console (text getty on `ttyS0`) — a graphical greeter never appears there. Attaching `incus console … --type vga` to an already-running VM can strand you on the tty1 text-login fallback.

If you still land on a text login after a clean cold boot, the greeter genuinely failed — confirm over serial (`incus console cosmic-01`, `Ctrl-q` to detach):

```bash
systemctl status display-manager.service
journalctl -b -u display-manager.service --no-pager | tail -50
```

Renderer/EGL/DRM errors mean cosmic-comp could not get a graphical context — a VM-GPU limitation (no `virgl` 3D, no working DRM render node), not a config error. The practical answer is to test COSMIC on bare metal.

## Remote desktop over NetBird — state of affairs (2026-07-24)

The use case: the remote host runs COSMIC and is joined to NetBird, the user's machine is joined to NetBird, and the user opens ordinary remote-desktop software and logs into the remote COSMIC desktop over its NetBird address — **no Incus client, no SPICE-over-API bridge**. This needs a remote-desktop *server* listening inside the VM on a NetBird-reachable TCP port.

As of this date there is **no vanilla/upstream way to serve a stock COSMIC session over RDP or VNC.** COSMIC's compositor does not yet expose the `org.freedesktop.portal.RemoteDesktop` portal (pop-os/xdg-desktop-portal-cosmic #23, still open), and that portal is what a remote-desktop server needs for input injection. Consequently:

- **wayvnc** targets wlroots compositors; `cosmic-comp` is smithay-based and lacks the virtual-pointer protocol wayvnc needs (pop-os/cosmic-comp #2094) — not reliable.
- **xrdp** requires an X11 session; COSMIC is Wayland-only, so xrdp cannot launch a COSMIC session.
- **gnome-remote-desktop** relies on the RemoteDesktop portal — it will not drive COSMIC.

Three paths exist today:

| Path | Fits the use case? | Notes |
|------|--------------------|-------|
| `cosmic-ext-rdp-server` (olafkfreund) | Yes, natively | RDP server on `:3389` for Remmina/FreeRDP/`mstsc`; PAM, TLS; ships a NixOS module. **Experimental** — requires a forked compositor (`cosmic-comp-rdp`) and forked portal, replacing core COSMIC components. v0.3.0, single maintainer. |
| GNOME headless RDP (`gnome-remote-desktop`) or KDE + KRDP | Yes, on a **different** DE | Mature, listens on `:3389`, any RDP client connects to the VM's NetBird address. Drops COSMIC for the remote use case. |
| SPICE via the Incus client (`incus console --type vga`) | No | Requires the Incus client on the user's machine — see [Connect: cold-start with the VGA console](#connect-cold-start-with-the-vga-console). Fine for admins, not the end-user use case above. |

All of these still ride on `cosmic-comp` getting a graphical context in the VM — the same VM-GPU limitation noted above applies.

## References

- NixOS COSMIC wiki (note: shows shorthand `services.cosmic.*`, not the real paths): https://wiki.nixos.org/wiki/COSMIC
- nixpkgs merge PR (real option paths): https://github.com/NixOS/nixpkgs/pull/267099
- Incus console how-to: https://linuxcontainers.org/incus/docs/main/howto/instances_console/
- SPICE is Incus-API-only, no per-VM port (maintainer confirmation): https://discuss.linuxcontainers.org/t/vm-on-headless-unbuntu-server-connecting-windows-spice-client-for-vga-console/13544
- Remote SPICE via the Incus client (confirmed working): https://discuss.linuxcontainers.org/t/remote-viewer-for-vm-console-spice-over-ip/21480
- COSMIC RemoteDesktop portal gap (open): https://github.com/pop-os/xdg-desktop-portal-cosmic/issues/23
- Experimental COSMIC RDP server (NixOS module): https://github.com/olafkfreund/cosmic-ext-rdp-server
- Login user creation — `wi-nixos/nixos-best-practices-tool.md`
- Sibling tools — `incus-profile-gui-tool.md` (containers), `incus-windows-tool.md` (Windows VM)
- Authoring standard — `wi-base/WORK_INSTRUCTIONS.md`

Tags: #tool #incus #vm #desktop #cosmic #nixos
