---
name: Incus Container Management
description: Instructions for how to configure incus containers
---

# Incus Container Management

## Set Container Description

```
incus config set <some.container> description="some description" -p
```
Note: The -p (or --property) flag is required because description is an instance-defined property/metadata, not a configuration setting
