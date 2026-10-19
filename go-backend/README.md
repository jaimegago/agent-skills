# Go Backend Standards Skill

Go backend development standards for any Agent Skills consumer, applied when working on Go code.

## Installation

This skill is vendored with the rest of `agent-skills`: the consuming repository carries the repository as a git submodule at `agent-skills/`, pinned to a tag, and its `AGENTS.md` tells the agent to read `agent-skills/go-backend/SKILL.md` when working on Go code. See the top-level README for the contract.

## What's Included

**SKILL.md** (read when working on Go):
- Architecture: `cmd/internal/pkg`, interface-based design, domain organization
- Testing: Table-driven, mocks, >80% coverage, integration tests
- Observability: OTel via middleware/decorators, trace-enriched logging
- Security: OWASP-based (SQL injection, XSS, bcrypt, input validation)
- Code style: Effective Go, Google Style Guide, golangci-lint

**Reference files** (loaded on-demand when Claude needs detail):
- `references/architecture.md` - DI, boundary separation, interface design
- `references/testing.md` - Test containers, coverage, mock generation
- `references/observability.md` - Full OTel patterns, gRPC interceptors
- `references/security.md` - OWASP Go-SCP practices
- `references/cicd.md` - Versioning, ldflags, Makefile, Docker builds, release pipelines

## Usage

The consuming repository's `AGENTS.md` names the trigger (any Go work) and the path. Nothing auto-discovers the skill from `agent-skills/`; the instruction file is the discovery.

## Structure

```
go-backend/
├── SKILL.md
└── references/
    ├── architecture.md
    ├── testing.md
    ├── observability.md
    ├── security.md
    └── cicd.md
```

## Updating

Advance the submodule pointer in the consuming repository to a newer tag of `agent-skills`; a copy edited in place is drift.
