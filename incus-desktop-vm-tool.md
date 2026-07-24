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

The purpose of this tool is to enable a desktop environment inside an Incus **virtual machine** (not a container) and connect to its graphical console.

This is important because an Incus VM is a separate OS in a separate kernel — a Wayland compositor (like COSMIC) renders to a VGA framebuffer, not to the serial line, so the default `incus console` shows a text getty instead of the greeter. Getting both the config and the connection right is what this skill captures.

> 📝 **Note** — This skill is intentionally sparse. It is a starting point for documenting Linux desktop-VM scenarios as we encounter them. The COSMIC section below is the first entry; add siblings (GNOME, Plasma, etc.) as they come up.

## TOC

- [Scope and Siblings](#scope-and-siblings)
- [Console Types: VGA vs Serial](#console-types-vga-vs-serial)
- [COSMIC (NixOS)](#cosmic-nixos)
- [Future Scenarios](#future-scenarios)
- [References](#references)

## Scope and Siblings

This skill covers **Linux** desktop VMs. Sibling skills cover adjacent ground and should be used instead where they fit:

| Need | Use |
|------|-----|
| GUI apps inside a **container** (X11/PulseAudio forwarding) | `incus-profile-gui-tool.md` |
| A **Windows** VM install from a repacked ISO | `incus-windows-tool.md` |
| A **Linux** desktop VM (COSMIC, …) | this skill |

The line is the instance type: `incus-profile-gui` is for containers; this skill is for VMs (`incus launch … --vm`, or `incus info` showing `VIRTUAL-MACHINE`).

## Console Types: VGA vs Serial

An Incus VM exposes two consoles:

| Console | Command | What it shows |
|---------|---------|---------------|
| Serial | `incus console <vm>` (default) | Text only — kernel and getty on `ttyS0`. A graphical greeter will **never** appear here. |
| VGA | `incus console <vm> --type vga` | The graphical framebuffer — where a Wayland greeter renders. This is how you see a desktop login. |

If you see a "Welcome to NixOS … tty1" text login on the VGA console, the greeter did **not** take over VT 1 and agetty fell back to a text login — see the scenario sections for why and how to confirm.

## COSMIC (NixOS)

### Enable COSMIC

On NixOS, COSMIC is enabled with two options. Use these exact paths — the NixOS wiki shows shorthand `services.cosmic.*`, but the actual merged module (nixpkgs PR #267099) registers the options under the conventional `services.desktopManager.*` and `services.displayManager.*` namespaces. The wiki paths fail with `The option 'services.cosmic' does not exist.`

```nix
services.desktopManager.cosmic.enable = true;
services.displayManager.cosmic-greeter.enable = true;
```

Do **not** also enable a competing desktop or display manager (`services.xserver.desktopManager.gnome`, `services.displayManager.gdm`, `sddm`, `lightdm`) — they will fight the cosmic-greeter.

A minimal `configuration.nix` for a NixOS-in-Incus VM:

```nix
{ modulesPath, ... }:

{
  imports = [
    "${modulesPath}/virtualisation/incus-virtual-machine.nix"
    ./incus.nix
  ];

  # COSMIC desktop
  services.desktopManager.cosmic.enable = true;
  services.displayManager.cosmic-greeter.enable = true;

  networking = {
    dhcpcd.enable = false;
    useDHCP = false;
    useHostResolvConf = false;
  };

  systemd.network = {
    enable = true;
    networks."50-enp5s0" = {
      matchConfig.Name = "enp5s0";
      networkConfig = {
        DHCP = "ipv4";
        IPv6AcceptRA = true;
      };
      linkConfig.RequiredForOnline = "routable";
    };
  };

  system.stateVersion = "26.05";
}
```

### Connect to the COSMIC greeter

After `nixos-rebuild switch`, attach the **VGA** console and **cold-cycle** the VM so you catch the greeter from boot. Attaching to an already-running VM with `incus console … --type vga` can land you on the tty1 getty fallback if the greeter has already tried and the framebuffer state is stale:

```bash
incus stop cosmic-01
incus start cosmic-01 --console=vga
```

`--console=vga` on `start` attaches the VGA console immediately as the VM boots, so you see the kernel messages and then the cosmic-greeter login screen replace the text VT.

If you still land on the tty1 text login after a clean cold boot, the greeter genuinely failed to start — confirm over the serial console:

```bash
incus console cosmic-01            # serial; Ctrl-q to detach
# after login:
systemctl status display-manager.service
journalctl -b -u display-manager.service --no-pager | tail -50
journalctl -b --no-pager | grep -iE 'cosmic|smithay|drm|egl|vulkan' | tail -40
```

A `display-manager.service` that is `failed` or `inactive` with renderer/EGL/DRM errors in the journal means cosmic-comp could not get a graphical context in the VM. That is a VM-GPU limitation (no `virgl` 3D acceleration, no working DRM render node), not a config error — and the practical answer is usually to test COSMIC on bare metal instead.

### Verify the option exists on your channel

```bash
nixos-option services.desktopManager.cosmic.enable
nixos-option services.displayManager.cosmic-greeter.enable
```

Both should return `false` (the default). If `nixos-option` is deprecated on your channel, a successful `nixos-rebuild switch` is itself confirmation.

## Future Scenarios

Intentionally sparse. Add a new `### <DE> (<distro>)` section under here for each Linux desktop-VM scenario we document (GNOME on NixOS, Plasma on Debian, etc.), keeping the same shape: enable options, connect pattern, failure-mode notes.

## References

- NixOS COSMIC wiki page (note: uses shorthand `services.cosmic.*`, not the real option paths): https://wiki.nixos.org/wiki/COSMIC
- nixpkgs COSMIC module source: `nixos/modules/services/desktop-managers/cosmic.nix` in `NixOS/nixpkgs`
- Merge PR (option paths landed here): https://github.com/NixOS/nixpkgs/pull/267099
- Tracking issue: https://github.com/NixOS/nixpkgs/issues/259641
- Incus console how-to: https://linuxcontainers.org/incus/docs/main/howto/instances_console/
- Sibling: GUI apps in containers — `incus-profile-gui-tool.md`
- Sibling: Windows VM install — `incus-windows-tool.md`
- Authoring standard — `wi-base/WORK_INSTRUCTIONS.md`

Tags: #tool #incus #vm #desktop #cosmic #nixos