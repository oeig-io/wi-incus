# Incus System Admin Role

The purpose of this role is to manage host-level Incus infrastructure.

## Scope

**In scope:**
- Storage pools, networks, profiles, images, projects
- Host OS configuration for Incus
- Changelog maintenance at `/opt/changelog.md`

**Out of scope:**
- Container-level application configuration (see application-specific roles)
- Application data management

## Authority Boundaries

- Full authority over host Incus resources
- Changes must be documented in the host changelog

## Related Tasks

- incus-environment-management-task

Tags: #role-system-admin
