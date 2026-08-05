# wi-incus

The purpose of this repo is to hold the work instructions for creating,
configuring, and observing [Incus](https://linuxcontainers.org/incus/)
containers and VMs. It matters because Incus is where our application payloads
actually run, and these documents keep that footprint consistent across hosts.

Authoring and formatting follow `wi-base/WORK_INSTRUCTIONS.md`.

## Work Instructions

**Roles**

- `incus-system-admin-role.md` — boundaries for the actor administering Incus

**Tasks**

- `incus-environment-management-task.md` — bring an Incus environment up/down
- `incus-profile-management-task.md` — apply and adjust instance profiles

**Tools**

- `incus-container-management-tool.md` — configure containers
- `incus-disk-management-tool.md` — map a host disk into a container, and
  observe ZFS disk usage across the shared storage pool
- `incus-instance-clone-tool.md` — clone an instance and scrub the identity and
  outbound credentials it inherits before first boot; the authority for
  authoring and auditing a repo's clone script; also how to read or edit a
  **stopped** container's filesystem offline
- `incus-profile-gui-tool.md` — run GUI desktop apps in a container (X11, audio)
- `incus-project-management-tool.md` — projects with dedicated networks/profiles
- `incus-remote-management-tool.md` — remotes and project-scoped user access
- `incus-windows-tool.md` — install Windows in an Incus VM from a repacked ISO
- `incus-desktop-vm-tool.md` — enable a Linux desktop environment (COSMIC, …) in
  an Incus **VM** and connect to its graphical (VGA) console; sibling to
  `incus-windows-tool.md` (Windows) and distinct from `incus-profile-gui-tool.md`
  (GUI apps in containers)

## Related

- `wi-base/WORK_INSTRUCTIONS.md` — authoring standard for these documents
- `wi-base/refresh-skills.sh` — publishes `*-tool` files as AI skills
