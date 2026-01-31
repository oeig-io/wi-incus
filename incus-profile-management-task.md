# Incus Profile Management Task

The purpose of this task is to manage incus profiles that define container configuration defaults.

## Scope

This task covers:

- Creating and modifying profiles
- Device assignments
- Resource limits

Standard host provisioning commands (run once per new host) are in incus-environment-management-task.

## View Profiles

### List All Profiles

```bash
incus profile list
```

### View Profile Details

```bash
incus profile show <profile-name>
```

## Edit Profiles

### Edit Profile Interactively

```bash
incus profile edit <profile-name>
```

### Set Configuration Value

```bash
incus profile set <profile-name> <key>=<value>
```

### Unset Configuration Value

```bash
incus profile unset <profile-name> <key>
```

## Device Management

### View Devices

```bash
incus profile device show <profile-name>
```

### Set Device Property

```bash
incus profile device set <profile-name> <device> <key>=<value>
```

## Apply Changes to Existing Containers

Profile changes apply automatically to new containers. For existing containers:

```bash
incus restart <container-name>
```

Tags: #role-system-admin
