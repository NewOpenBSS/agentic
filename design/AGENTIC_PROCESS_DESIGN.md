# Agentic Process Design

This document captures the design decisions and principles agreed during discussion.
It is the basis for building the new Agentic repo from scratch.

---

## Vision

A generic, reusable agentic SDLC process that works across projects and languages.
The process lives in a dedicated **Agentic repo** within a GitHub organisation.
All domain code lives in separate **domain monorepos** within the same organisation.

---

## Repository Structure

### One GitHub Organisation

All repos — domain and agentic — live in a single GitHub organisation.
This is the defined boundary for now.

### Agentic Repo

The control plane. Owns the process, standards, reusable workflows, and recipes.
Also owns all **Requirements** as GitHub Issues.

```
agentic/
  AGENTS.md                        — process standards (language-agnostic)
  docs/
    SYSTEM_ARCHITECTURE.md         — overall system, cross-domain integration
    SETUP.md                       — one-time org setup guide
    ONBOARDING_DOMAIN.md           — per-domain onboarding guide
    domains/
      charging.md                  — domain description + linked repos
      billing.md
      ...
  .goose/recipes/                  — all recipes (requirements, scoping, design, dev)
  .github/workflows/               — reusable workflow definitions
  standards/
    go.md                          — Go coding standards and conventions
    java.md                        — Java/Quarkus/Maven standards
    ...
```

### Domain Repos

One monorepo per domain, regardless of language. Contains code, domain-specific
context, and **Feature and Task GitHub Issues**.

```
charging-domain/
  docs/
    ARCHITECTURE.md                — domain-specific architecture (version-controlled)
    PROJECT_BRIEF.md               — what this domain does and why
  .ai/
    context/                       — domain-specific standards overrides (if any)
    memory/
      DECISIONS.md                 — domain ADRs, version-controlled with code
  .github/workflows/               — thin callers of Agentic reusable workflows
  [code]
```

Note: `.ai/tasks/` no longer exists. Task tracking moves to GitHub Issues.
Note: `.ai/memory/STATUS.md`, `FEATURES.md`, and `REQUIREMENTS.md` do NOT exist.
GitHub Issues replace them entirely.

---

## GitHub Issues as Memory

### Hierarchy

Requirements live in the **Agentic repo**. Features and Tasks live in the
**domain repo**. Native GitHub sub-issues model the full hierarchy:

```
agentic#42      Requirement: Port Charging Backend to Go   (domain-agnostic)
  └── charging-domain#1   Feature: Permission Enforcement Framework
        ├── charging-domain#2   Task: JWT permissions claim extraction
        ├── charging-domain#3   Task: Implement SecureRouter
        ├── charging-domain#4   Task: Add @auth GraphQL directive
        └── charging-domain#5   Task: Wire unit tests
```

Cross-repo sub-issues are fully supported by GitHub — proven in practice.

### Why this split

**Requirements are domain-agnostic.** A requirement describes a business need
without knowing which domain or domains will deliver it. It belongs in the
central Agentic repo.

**Features are domain-specific.** A Feature describes what gets built in a
specific domain repo. It belongs in that repo so that:
- Branch linking is native and automatic (same repo)
- PR closing is simple: `Closes #N` not cross-repo syntax
- The agent works in one repo throughout the development session
- Task sub-issues are co-located with the Feature they belong to

**Tasks are ephemeral implementation detail.** They are AI-generated during the
Feature Design Session, processed during the Dev Session, and closed on completion.
They are sub-issues of the Feature issue and live in the domain repo.

### Labels

**Agentic repo:**
- Type: `requirement`
- Status: `backlog`, `in-design`, `in-development`, `in-review`, `done`

**Domain repos:**
- Type: `feature`, `task`
- Status: `backlog`, `in-design`, `in-development`, `in-review`, `done`

Requirements carry no domain label — they are domain-agnostic by definition.
Which domains a Requirement touches is visible from its Feature sub-issues.

### Feature Numbering

Features use the **GitHub issue number** within the domain repo. There is no
F-NNN scheme — that was a file-system artifact with no place in the GitHub model.

Issue numbers are unique within each domain repo. At the org Project level,
the issue title and Domain field disambiguate issues with the same number
across different domain repos.

Cross-repo references use GitHub's standard syntax: `charging-domain#1`.

### Branch Naming

Branches are named `feature/N-short-description` where `N` is the GitHub issue
number in the domain repo. This naming causes GitHub to auto-link the branch
to the Feature issue natively — no API call required.

Example: `feature/1-permission-enforcement-framework` auto-links to
`charging-domain#1`.

### STATUS.md

Dropped. The agent establishes session context by querying the org Project at
startup via the GitHub Projects v2 GraphQL API:
- Open Requirements → active work
- Features with `in-development` → current branch
- Features with `backlog` under open Requirements → next up

No stale file. Live data from GitHub.

---

## GitHub Organisation Project

An org-level GitHub Project (v2) aggregates issues from all repos — Agentic and
domain — into a single board. Provides cross-domain visibility without duplicating
issue data.

### Views

| View | Type | Purpose |
|---|---|---|
| All Work | Table, grouped by Status | Full backlog across all domains |
| By Domain | Board, filtered by Domain field | Domain-level kanban |
| Roadmap | Timeline, grouped by parent Requirement | Planning view |

### Fields

| Field | Location | Owner |
|---|---|---|
| Status | Project field (single select) | Synced from issue labels via automation |
| Domain | Project field (single select) | Set on Feature issues — not on Requirements |
| Parent issue | Built-in | Populated from sub-issue relationships |
| Sub-issues progress | Built-in | Auto-calculated from child issue state |

**Labels are the source of truth.** The Project Status field is a view on top
of labels, kept current by GitHub's built-in automation. Agents write to labels
only — the Project reflects this automatically.

### Automations

- Issue opened → Status = `Backlog`
- Issue closed → Status = `Done`
- PR merged (closes issue) → Issue closes → Status = `Done`

---

## Workflow Triggers — Label-Driven State Machine

The Feature label IS the state. A label change IS the trigger.
No sentinel files. No polling. Pure GitHub event-driven execution.

```
Human applies in-design on Feature issue
  → Feature Design Session workflow triggers (issues: labeled)
  → Decomposes Feature into Task sub-issues
  → Creates branch via auto-link naming convention
  → Applies in-development label on Feature issue

  → Dev Session workflow triggers (issues: labeled)
  → Queries open Task sub-issues ordered by issue number
  → Implements → commits → closes Task issue → repeat
  → When all Tasks done → opens PR → applies in-review label

Human reviews and merges PR
  → Feature issue closes automatically (Closes #N in PR body)
  → Status → Done via Project automation
```

### Workflow Configuration (domain repos)

```yaml
on:
  issues:
    types: [labeled]

jobs:
  dispatch:
    if: contains(github.event.issue.labels.*.name, 'feature')
    steps:
      - if: github.event.label.name == 'in-design'
        # invoke Feature Design Session recipe

      - if: github.event.label.name == 'in-development'
        # invoke Dev Session recipe
```

---

## Domain Monorepo Structure

### Principle

One repo per domain. All applications and internal libraries for a domain
live together. The "one repo per library" Java pattern is abandoned in
favour of monorepos with smart builds.

**Rule:** if nothing outside this domain consumes it, it lives in the
domain monorepo. Truly shared cross-domain libraries remain as separate
repos with independent versioning.

### Naming Convention

Domain repos are named `{domain}-domain` — e.g. `charging-domain`,
`billing-domain`, `notification-domain`. The name reflects the domain,
not the language or project.

### Go Domains

Standard Go monorepo layout:

```
charging-domain/
  cmd/
    app-1/
    app-2/
  internal/
    shared-package/
```

### Java/Quarkus Domains

Maven multi-module layout reflecting the standard library structure:

```
java-domain/
  pom.xml                         — parent POM, dependency management only, no version
  domain-api/
    pom.xml                       — cross-domain contracts, published to Nexus, versioned
    .github/project.yml           — current-version + next-version
  domain-common/                  — internal shared lib, reactor-only, no Nexus publish
    pom.xml                       — no .github/project.yml needed
  domain-operator/                — K8s operator + Flyway DB scripts, internal
    pom.xml
  app-1/
    pom.xml
    .github/project.yml           — current-version + next-version
  app-2/
    pom.xml
    .github/project.yml
  app-3/
    pom.xml
    .github/project.yml
```

**Module types:**
- `domain-api` — cross-domain contracts (models, events). Published to Nexus for other
  domains to consume. Has its own version. Changes here are contract changes — require
  explicit approval (see Contract Rules).
- `domain-common` — shared code internal to this domain only. Never published to Nexus.
  Consumed via Maven reactor. No independent versioning needed.
- `domain-operator` — Kubernetes operator and Flyway migration scripts. Internal.
- Apps — Quarkus applications. Each has its own version. Build produces a container image.

---

## Build Strategy

### Core Principle

> Only build what changed. Build shared dependencies once.

This holds regardless of where the build runs (self-hosted, cloud CI, local).

### Version Bump as Build Trigger

The developer (human or AI) explicitly signals build intent by bumping the version
in `.github/project.yml`. CI detects which `*/.github/project.yml` files changed
and builds only those modules.

**This is intentional by design** — a developer changing code without bumping the
version signals "not ready to release." A version bump signals "build and publish this."

`.github/project.yml` structure:
```yaml
current-version: "1.1.1"
next-version: "1.1.2-SNAPSHOT"
```

### Java Release Flow

```
Developer bumps current-version and next-version in module/.github/project.yml
  → PR raised and merged
  → maven-release-prepare workflow triggers (on PR merge)
      → mvn release:prepare  — creates git tag at current-version
      → mvn release:perform  — publishes artifact to Nexus (for API libs)
  → tag creation triggers release workflow
      → native image built and pushed to container registry (for apps)
      → JVM image built as fallback if native flag not set
```

### Native First

Quarkus native is the default build target. JVM image is supported as an explicit
opt-in (dev convenience or fallback). The release workflow builds native by default.

### Shared Lib Build Cost

Internal shared libs (no `.github/project.yml`) are built as part of the Maven
reactor whenever an app that depends on them releases. They compile fast — the cost
is acceptable compared to a native image build.

### Go Build Strategy

Go's toolchain handles dependency awareness natively. Container images pushed only
for binaries whose source paths changed, detected via `dorny/paths-filter`.

### Reusable Workflows

Build and release logic lives in the Agentic repo as reusable workflows
(`workflow_call`). Domain repos contain thin caller workflows only.

Existing workflow templates (from `platform-templates`):
- `build-jar.yml` — build and test a JAR library
- `build-quarkus-app.yml` — build and test a Quarkus application
- `maven-release-prepare.yml` — prepare and perform Maven release on PR merge
- `release-java.yml` — build and push JVM Docker image
- `release-native.yml` — build and push native Docker image (GraalVM)
- `release-helm.yml` — release Helm chart

---

## Recipe Bootstrap Process

Every recipe that runs on a branch follows this bootstrap sequence:

```
1. git fetch origin
2. git checkout main
3. git pull origin main
4. Check if the recipe file itself changed in the pull
     YES → "Recipe was updated. Please re-launch." — stop cleanly
     NO  → continue
5. Query GitHub for session context (replaces STATUS.md read)
     - Open Requirements
     - Features in-development (active branch)
     - Features in backlog (next up)
6. Do all work and commits on the feature branch
7. Confirmation step — summarise changes, ask human to approve
8. End session cleanly — workflow handles push and PR
```

Main branch protection is enabled. No direct commits to main.

---

## Phase 1 — Requirements Session

**Goal:** Capture one or more requirements as GitHub Issues in the Agentic repo.

**Process:**
- Human starts the recipe
- Recipe bootstraps
- Capture first requirement → create GitHub Issue in `agentic` repo with structured template
- Ask: "Do you have more requirements to capture?"
  - YES → capture next requirement on the same session
  - NO  → proceed to confirmation
- Confirmation: summarise all issues created, ask "Done?"
- Session ends

Requirements are created directly as GitHub Issues — no branch, no PR, no merge.

---

## Phase 2 — Scoping Session

**Goal:** Decompose a Requirement into one or more scoped Feature Issues in the
appropriate domain repo.

**Process:**
- Human starts the recipe, specifying which Requirement to scope
- Recipe bootstraps
- Recipe reads the parent Requirement issue from `agentic` repo
- Recipe reads existing Feature sub-issues for this Requirement (if any)
  — treats documented content as established context, only asks about gaps
- Work through scoping artefacts
- Create Feature Issue(s) in the relevant domain repo(s)
- Wire sub-issue relationship: Feature → parent Requirement (cross-repo)
- Add Feature to org Project, set Domain field
- Confirmation: "Ready to kick off design and development?"
  - YES → apply `in-design` label on Feature issue → triggers Feature Design Session
  - NO  → Feature issue exists in `backlog`, development deferred

**Reasons to defer development:**
- Dependency on another feature not yet complete
- Waiting for external input (e.g. permission names from Keycloak admin)
- Strategic timing / resource availability

---

## Phase 3 — Feature Design Session

**Goal:** Decompose a Feature into ordered Task sub-issues. Triggered automatically
by the `in-design` label being applied to a Feature issue.

**Process:**
- Workflow triggers on Feature issue labeled `in-design`
- Recipe reads the Feature issue spec from the domain repo
- Analyses the codebase to understand what exists and what must be built
- Creates Task sub-issues under the Feature issue (ordered by creation = issue number)
- Creates the feature branch: `feature/N-description` (auto-links to Feature issue)
- Applies `in-development` label on Feature issue → triggers Dev Session
- Session ends cleanly

---

## Phase 4 — Dev Session

**Goal:** Process Task sub-issues to completion. Triggered automatically by the
`in-development` label being applied to a Feature issue.

**Process:**
- Workflow triggers on Feature issue labeled `in-development`
- Recipe reads the Feature issue and its open Task sub-issues (ordered by issue number)
- For each Task: implement → build → test → commit → close Task issue → next
- If build or tests fail → stop. Workflow turns red. Human investigates and re-triggers
  by re-applying the `in-development` label.
- When all Tasks closed → open PR with `Closes #N` → apply `in-review` label
- Session ends cleanly — PR awaits human review

---

## File Artefacts

The following files remain in the domain repo. All tracking artefacts have moved
to GitHub Issues.

| File | Purpose | Lives in |
|---|---|---|
| `AGENTS.md` / `CLAUDE.md` | Agent process instructions | Agentic repo (inherited) |
| `.ai/context/*.md` | Language/framework standards | Agentic repo `standards/` |
| `.ai/memory/DECISIONS.md` | Domain ADRs | Domain repo |

**Removed artefacts — replaced by GitHub Issues:**

| Was | Replaced by |
|---|---|
| `.ai/memory/REQUIREMENTS.md` | Issues in `agentic` repo |
| `.ai/memory/FEATURES.md` | Issues in domain repo |
| `.ai/memory/STATUS.md` | Project GraphQL query at session start |
| `.ai/tasks/queue/NNN-name.md` | Task sub-issues in domain repo |
| `.ai/tasks/done/NNN-name.md` | Closed Task sub-issues |
| `.ai/tasks/READY` | `in-development` label on Feature issue |
| F-NNN numbering | GitHub issue number within domain repo |

---

## Documentation

### What lives where

| Document | Location | Rationale |
|---|---|---|
| System architecture | Agentic repo `docs/SYSTEM_ARCHITECTURE.md` | Spans all domains |
| Org setup guide | Agentic repo `docs/SETUP.md` | One-time setup instructions |
| Domain onboarding | Agentic repo `docs/ONBOARDING_DOMAIN.md` | Per-domain setup |
| Domain registry | Agentic repo `docs/domains/` | Cross-repo metadata |
| Domain architecture | Domain repo `docs/ARCHITECTURE.md` | Version-controlled with code |
| Domain brief | Domain repo `docs/PROJECT_BRIEF.md` | Version-controlled with code |
| Language standards | Agentic repo `standards/` | Reusable across projects |
| ADRs | Domain repo `.ai/memory/DECISIONS.md` | Version-controlled with code |
| Process standards | Agentic repo `AGENTS.md` | Language-agnostic |

### Wiki

Scrapped. Version control of technical docs is non-negotiable.
Human-readable entry points are handled by GitHub repo descriptions,
the org Project board, and the Agentic repo README.

---

## Project Adoption Strategy

### Greenfield Projects
Start with the monorepo structure from day one. No compromise, full process applies.

### Well-Structured Existing Projects
Restructure once into monorepo layout. Effort is manageable. Full process applies
after restructure.

### Legacy / Messy Projects
Use the **strangler pattern** — build the monorepo alongside the existing structure,
migrate modules piece by piece, enable agentic processes only for migrated modules.
No expectation of full coverage on day one. The agentic process does not attempt to
work with fragmented multi-repo legacy structures.

---

## Open Questions

- **DECISIONS.md** — could ADRs move to GitHub Issues with a `decision` label?
  Searchable and linkable, but loses version-control co-location with code.
  Decision deferred.

- **Scoping Session save-and-continue** — for long scoping sessions that cannot
  complete in one context window, the save-and-continue mechanism is not yet designed.
