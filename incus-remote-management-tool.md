---
name: incus-remote-management
description: Instructions for how to configure incus remotes and project-scoped user access
compatibility: opencode
metadata:
  type: tool
  original_file: incus-remote-management-tool.md
  category: configuration
  scope: incus
---

# Incus Remote Management

## Install Incus Server and Client

https://github.com/zabbly/incus

- for desktop: best to use Stable
- for server: best to use LTS

## Add a New Remote Server

On the client - get/generate a certificate. Note this is a public key. It is OK to pass in plain text.

```
incus remote get-client-certificate | tee /tmp/cert.txt
```

On the server - add the client certificate

```
incus config trust add-certificate /path/to/cert.txt
```

On the client - give the remote a meaningful name

```
incus remote add <remote-name> https://<server-ip>:<port:8443>
```

## Connect and Manage Remote Clusters

Connect to a remote server:

```
incus remote switch some-remote
```
Confirm looking at remote to ensure you see different instances:

```
incus list
```

While in the remote cluster, you might belong to more than one project. To list projects:

```
incus project list
```

The table will indicate the current project. To switch projects:

```
incus project switch some-other-project
```

Switch back to your local cluster/server:

```
incus remote switch local
```

## Project-Scoped User Access

To give a user access to a specific project without granting full admin privileges, add their client certificate as restricted.

### Get the User's Certificate

The user runs this on their client machine and shares the output with the admin:

```
incus remote get-client-certificate | tee /tmp/cert.txt
```

### Add Restricted Certificate

The admin adds the certificate to the server, confined to one or more projects:

```
incus config trust add-certificate /path/to/user-cert.txt --projects acme --restricted
```

To grant access to multiple projects, use a comma-separated list:

```
incus config trust add-certificate /path/to/user-cert.txt --projects acme,plantos --restricted
```

> **📝 Note** - You can also use token-based authentication (`incus config trust add --projects acme --restricted`) where the user redeems a one-time token instead of sharing their certificate file.

### Verify Restricted Access

List trusted certificates and confirm the restriction:

```
incus config trust list
```

To inspect or modify restrictions on an existing certificate:

```
incus config trust edit <fingerprint>
```

Ensure `restricted` is `true` and the `projects` list contains only the intended projects. To grant access to additional projects later, add them to the `projects` list — no client-side changes are needed.

### User Connects to the Remote

The user adds the remote in the usual way:

```
incus remote add <remote-name> https://<server-ip>:8443
```

The user will only see resources in their assigned project(s). If the user has access to multiple projects, they switch between them:

```
incus project switch acme
incus list

incus project switch plantos
incus list
```

Or target a project directly without switching:

```
incus list --project acme
```
