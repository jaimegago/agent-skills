---
name: llm-docs-audit
description: Audit AGENTS.md for drift against the repo and installed skills. Use when asked to check, audit, or review AGENTS.md for staleness. Produces a findings list only, does not edit.
metadata:
  version: "1.0.0"
---

# LLM Docs Audit

## Purpose

Hunt for drift in AGENTS.md. Produce a findings list the developer can review. Do not edit anything.

## Termination rule (read this first)

This skill is strictly bounded. You will thrash if you do not follow these rules:

1. **Fixed scope.** You check only the surfaces listed in "Audit scope" below. Anything outside that scope is out of scope, period. Do not expand scope because a finding "seems related" to something else.

2. **Single pass.** You walk the scope exactly once, collecting findings as you go. You do not re-verify findings. You do not go back to check something you already checked. You do not search the codebase for "one more thing."

3. **Report only, no edits.** Your output is a findings list. You do not produce a diff. You do not modify AGENTS.md. If the developer approves findings, they invoke llm-docs-authoring separately.

4. **No findings is a valid outcome.** If AGENTS.md is current, say so and stop. Do not manufacture findings to feel productive.

If you catch yourself wanting to do a second pass or expand scope: STOP. Report what you have and let the developer decide.

## Audit scope

You check these surfaces, in this order, once each:

### 1. Build/test/lint commands

Compare the commands listed in AGENTS.md's Build/Test/Lint section against:
- The repo's Makefile (if present)
- The `scripts` section of package.json (if present)
- Common language-specific entry points (`go test ./...`, `cargo test`, `pytest`, etc.)

Finding format: "AGENTS.md says `X`, actual is `Y`" or "AGENTS.md is missing command `Y` that exists in Makefile."

### 2. Submodule list

Compare the submodules listed in AGENTS.md's Repo Dependencies section against `.gitmodules`.

Finding format: "AGENTS.md lists submodule `X` which is not in .gitmodules" or "Submodule `Y` exists in .gitmodules but is not documented in AGENTS.md."

### 3. Referenced file paths

For every file path mentioned in AGENTS.md, check whether it exists.

Finding format: "AGENTS.md references `path/to/thing` which does not exist."

### 4. Applicable skills — stale references

For every skill named in AGENTS.md's Applicable Skills section, check whether `agent-skills/[skill-name]/SKILL.md` exists at the pinned submodule.

Finding format: "AGENTS.md references skill `X` which is not in the vendored `agent-skills/`."

### 5. Applicable skills — missing references

Detect the repo's primary language(s) and frameworks by looking for: `go.mod`, `package.json` (and its dependencies), `pom.xml`, `Cargo.toml`, `pyproject.toml`, `requirements.txt`, `*.csproj`.

List vendored skills with `ls agent-skills/` and read each SKILL.md's description field.

Match in three tiers:

**Tier 1 — Universal skills.** Any skill whose description contains the exact phrase `"applies to all repos"` is universal and must be referenced in every AGENTS.md regardless of the repo's stack. If a universal skill is installed but not referenced, flag it.

**Tier 2 — Stack-specific skills (name-based).** `go-*` skills match Go repos, `frontend-*`/`react-*`/`vue-*` match repos with package.json and a frontend framework, `python-*` matches Python repos, `java-*` matches Java repos, etc.

**Tier 3 — Stack-specific skills (description-based fallback).** If a skill's name does not match any pattern, scan its description field for language or framework keywords matching the repo's detected stack.

For any installed skill that matches (via any of the three tiers) but is NOT referenced in AGENTS.md's Applicable Skills section, flag it.

Finding format:
- Universal missing: "Universal skill `X` is installed but not referenced in AGENTS.md. All repos should reference it."
- Stack-specific missing: "Installed skill `X` matches this repo's stack but is not referenced in AGENTS.md."

## What is NOT in scope

Do not check:
- Whether invariants are "still true" — you cannot tell, and the developer defines invariants.
- Whether the writing style is good.
- Whether sections are "complete enough."
- Conventions for drift against the actual code — the Go skill (or equivalent) is authoritative for conventions, not AGENTS.md.
- Anything in README.md or /docs — that is human-docs-authoring's job.
- Recent commits, changelogs, or git history.

If a finding does not fit one of the five scope categories above, it is out of scope.

## Output format

Produce a single markdown document with this structure:

```
# AGENTS.md Audit — [repo name]

## Summary
[One sentence: "N findings across M categories" or "No findings."]

## Findings

### Build/test/lint commands
- [finding 1]
- [finding 2]

### Submodules
- [finding 1]

### Referenced paths
- [none]

### Stale skill references
- [finding 1]

### Missing skill references
- [finding 1]

## Next step
[If findings: "Review and invoke llm-docs-authoring with the findings you want applied."]
[If no findings: "AGENTS.md is current. No action needed."]
```

Omit empty categories rather than listing "none."

## Completion

After producing the findings list: stop. Do not offer to apply findings. Do not re-verify. Do not expand scope. The developer's next move is either "apply these" (which triggers llm-docs-authoring) or "ignore" — that is their call.
