# DOMAINS.md — Domain Registry

This file is the authoritative list of all domain repositories that make up the
NewOpenBSS solution. Each domain is an independently deployable monorepo with its
own codebase and GitHub Issues. Coding standards are global and defined in `standards/`.

The `domains/` directory is gitignored — each developer clones domain repos locally
on first use. At session start, agents check that all active domains are present and
prompt the user to clone any that are missing.

Only humans add or remove domains. Adding a domain is an architectural decision and
must be reflected in `docs/ARCHITECTURE.md`.

---

## charging-domain

- **Repo:** git@github.com:NewOpenBSS/charging-domain.git
- **Stack:** Go
- **Status:** active
- **Description:** Real-time quota charging, balance management, and quota expiry processing
