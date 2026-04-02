# Project Brief: Agentic Repo

## Purpose

This is the **Agentic repo** — the control plane for an AI-driven software
development lifecycle (SDLC) process applied across all domain repos in the
NewOpenBSS GitHub organisation.

It owns the process, not the code. Domain repos own the code.

## What This Repo Contains

| Directory | Purpose |
|---|---|
| `AGENTS.md` | Agent session protocol — governs all AI agent behaviour |
| `.goose/recipes/` | Goose recipes for each session phase |
| `.github/workflows/` | Reusable GitHub Actions workflows (called by domain repos) |
| `standards/` | Language and framework coding standards (Go, Java, etc.) |
| `docs/` | Process and architecture documentation |
| `design/` | Design decisions and proposals |

## What This Repo Does NOT Contain

- Application code — that lives in domain repos
- Feature or Task issues — those live in domain repos
- Domain-specific architecture — that lives in each domain repo

## The Process

The agentic SDLC process has four phases:

**Phase 1 — Requirements Session (interactive)**
A human runs the requirements recipe conversationally. Raw ideas are
distilled into Requirement GitHub Issues in this repo. Requirements are
domain-agnostic — they describe business needs, not solutions.

**Phase 2 — Scoping Session (interactive)**
A human runs the scoping recipe for a specific Requirement. The session
decomposes it into one or more Feature Issues in the relevant domain repo(s).
Features are domain-specific — each belongs to exactly one domain repo.

**Phase 3 — Feature Design Session (automated)**
Triggered automatically when a Feature issue is labelled `in-design`.
The recipe analyses the codebase, decomposes the Feature into ordered
Task sub-issues, creates the feature branch, and applies `in-development`.

**Phase 4 — Dev Session (automated)**
Triggered automatically when a Feature issue is labelled `in-development`.
The recipe processes Task sub-issues in order: implement → build → test →
commit → close task → next. Opens a PR when all tasks are done.

## GitHub Issue Hierarchy

```
agentic repo: Requirement   (domain-agnostic business need)
  └── domain repo: Feature  (domain-specific, scoped deliverable)
        └── domain repo: Task sub-issues (implementation steps)
```

## Label-Driven State Machine

Labels on Feature issues drive the automated workflow:

```
[human] backlog → in-design     triggers Feature Design Session
[agent] in-design → in-development  triggers Dev Session
[agent] in-development → in-review  PR opened, awaits human review
[github] PR merged → done           issue closes automatically
```

## Organisation Structure

```
NewOpenBSS/
  agentic/            ← this repo — process control plane
  charging-domain/    ← domain monorepo (Go)
  billing-domain/     ← domain monorepo (Java/Quarkus)
  ...
```

## Key Design Decisions

- Requirements live here. Features and Tasks live in domain repos.
- GitHub Issues replace all file-based tracking artefacts.
- The org-level GitHub Project aggregates issues from all repos.
- Recipes live here and are referenced by reusable workflows.
- Domain repos contain only thin caller workflows.
- Branch naming: `feature/N-description` — auto-links to Feature issue.
- No F-NNN numbering — GitHub issue numbers are the identifiers.
