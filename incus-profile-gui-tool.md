---
name: incus-profile-gui
description: Incus profile configuration for running GUI desktop applications inside containers with X11 and PulseAudio forwarding
compatibility: opencode
metadata:
  type: tool
  original_file: incus-profile-gui-tool.md
  category: configuration
  scope: incus
---

# Incus GUI Profile Tool

The purpose of this tool is to provide a reusable incus profile for running GUI desktop applications inside Debian containers.

This is important because some workloads (like running a browser or desktop application for testing) require access to the host's display server and audio system.

> **📝 Note** - This profile is designed for Debian-based containers. The cloud-init configuration and default user (`debian`) are specific to Debian images.

## Getting Started

The purpose of this section is to identify which values need updating for your environment before applying the profile.

### Variables to Update

| Variable | Default Value | How to Find Your Value |
|----------|---------------|------------------------|
| `DISPLAY` | `:1` | Run `echo $DISPLAY` on host |
| `idmap` | `both 1000 1000` | Run `id -u` to get your UID |
| X11 socket number | `X1` | Match the socket in `/tmp/.X11-unix/` |
| Container user | `debian` | Default user for Debian containers |

### Quick Check

```bash
# Find your display number
echo $DISPLAY

# Find your user ID
id -u

# List available X11 sockets
ls /tmp/.X11-unix/
```

> **💡 Tip** - Update these values in the profile before creating containers. Search for `TODO` in the configuration below.

## Host Prerequisites

Before applying this profile, ensure the host has:

1. **X11 server running** with socket at `/tmp/.X11-unix/`
2. **PulseAudio** running with native protocol enabled
3. **User ID mapping** (typically your logged-in user, e.g., UID 1000)

### Verify Host Prerequisites

```bash
# Check X11 socket exists
ls -la /tmp/.X11-unix/

# Check PulseAudio native socket exists
ls -la /run/user/1000/pulse/native
```

> **📝 Note** - If PulseAudio native protocol is not enabled, add `enable-native-protocol = yes` to `/etc/pulse/default.pa` or `~/.config/pulse/default.pa`

## GUI Profile Configuration

Apply this profile to containers that need GUI access:

```yaml
config:
  environment.DISPLAY: :1  # TODO: Update to match host (e.g., :0 or :1)
  raw.idmap: both 1000 1000  # TODO: Update to match host user (both HOST_UID CONTAINER_UID)
  security.nesting: "true"
  security.syscalls.intercept.mknod: "true"
  security.syscalls.intercept.setxattr: "true"
  user.user-data: |
    #cloud-config
    runcmd:
      # Create symlinks in expected locations (works even if /tmp is tmpfs)
      - mkdir -p /tmp/.X11-unix
      - ln -sf /mnt/host-x11/X1 /tmp/.X11-unix/X1  # TODO: Update X1 to match host X socket
      - mkdir -p /tmp/.pulse-native
      - ln -sf /mnt/host-pulse/native /tmp/.pulse-native/
      - 'sed -i "s/; enable-shm = yes/enable-shm = no/g" /etc/pulse/client.conf'
      - 'echo export PULSE_SERVER=unix:/tmp/.pulse-native | tee --append /home/debian/.profile'
      - [su, debian, -c, "git clone https://github.com/chuboe/chuboe-system-configurator /home/debian/chuboe-system-configurator"]  # TODO: Update debian to match container user
      - [su, debian, -c, "cd /home/debian/chuboe-system-configurator/; ./init.sh"]  # TODO: Update debian to match container user
    packages:
      - x11-apps
      - mesa-utils
      - pulseaudio
      - git
      - ufw
description: GUI LXD profile
devices:
  PASocket:
    path: /mnt/host-pulse/
    source: /run/user/1000/pulse/  # TODO: Update 1000 to match host user UID
    type: disk
  X0:
    path: /mnt/host-x11/
    source: /tmp/.X11-unix/
    type: disk
  mygpu:
    type: gpu
```

## Apply Profile to Container

### Create Container with GUI Profile

```bash
incus launch images:debian/13 <container-name> --profile default --profile gui
```

### Add GUI Profile to Existing Container

```bash
incus profile add <container-name> gui
```

## Add Profile to Incus Host

The purpose of this section is to import the GUI profile into your local incus host.

### Option 1: Create from File

Save the profile YAML to a file, then import it:

```bash
# Save the profile (see GUI Profile Configuration section above) to gui.yaml
incus profile create gui < gui.yaml
```

### Option 2: Create Interactively

```bash
incus profile create gui
incus profile edit gui
```

Paste the YAML configuration (from the GUI Profile Configuration section) into the editor.

### Option 3: Set Individual Keys

Create the profile and set each configuration option:

```bash
incus profile create gui
incus profile set gui environment.DISPLAY=:1
incus profile set gui raw.idmap="both 1000 1000"
incus profile set gui security.nesting=true
incus profile set gui security.syscalls.intercept.mknod=true
incus profile set gui security.syscalls.intercept.setxattr=true
incus profile set gui user.user-data="<cloud-init-data>"

# Add devices
incus profile device add gui X0 disk path=/mnt/host-x11/ source=/tmp/.X11-unix/
incus profile device add gui PASocket disk path=/mnt/host-pulse/ source=/run/user/1000/pulse/
incus profile device add gui mygpu gpu
```

### Verify Profile

```bash
incus profile show gui
```

## Key Components Explained

| Component | Purpose |
|-----------|---------|
| `environment.DISPLAY` | Sets container's DISPLAY variable to match host (`:1` = first X server) |
| `raw.idmap` | Maps host user (UID 1000) to container for file ownership |
| `security.nesting` | Required for nested containers or certain system calls |
| `X0` device | Passes X11 socket from host to container |
| `PASocket` device | Passes PulseAudio socket from host to container |
| `mygpu` device | GPU passthrough for hardware acceleration |

## Adjust Display Number

If your host uses a different X display number (e.g., `:0` instead of `:1`):

1. Change `environment.DISPLAY: :1` to match your host
2. The X0 device source path should point to your actual socket

### Find Your Display Number

```bash
echo $DISPLAY
```

Tags: #role-system-admin
