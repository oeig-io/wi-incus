---
name: Incus Remote Management
description: Instructions for how to configure incus remotes
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
