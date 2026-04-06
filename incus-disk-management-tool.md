---
name: incus-disk-management
description: Instructions for how to configure incus disks
---

# Incus Disk Management

## Map Disk Between Host and Container

```
X_INCUS_INSTANCE=container-name
X_DISK_SOURCE=/path/in/host
X_DISK_DESTINATION=/path/in/container
X_DISK_LABEL=some-description    #no spaces
incus config device add $X_INCUS_INSTANCE "$X_INCUS_INSTANCE"-"$X_DISK_LABEL" disk source=$X_DISK_SOURCE  path=$X_DISK_DESTINATION shift=true
```
