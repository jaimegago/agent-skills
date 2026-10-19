---
name: llm-docs-authoring
description: Create or edit AGENTS.md files. Use when asked to create a AGENTS.md, add or update sections, or apply findings from an llm-docs-audit run. Does not hunt for drift.
---

# LLM Docs Authoring

## Purpose

Create and edit AGENTS.md files. AGENTS.md is a repo-specific briefing for AI agents working in the repo. (`CLAUDE.md` is the older name of the same file. Claude Code reads `CLAUDE.md` instead of `AGENTS.md` when both exist, so a repo keeps exactly one, and the workstream's name for it is `AGENTS.md`.) It is NOT a human-readable doc — that is what README.md is for.

## When to use

Invoke this skill when the developer asks any of:
- "Create a AGENTS.md for this repo"
- "Add a section to AGENTS.md for [X]"
- "Update the build commands in AGENTS.md"
- "Apply the findings from the audit"

Do NOT invoke this skill to hunt for drift. That is llm-docs-audit's job.

## Termination rule

This skill terminates when the specific ask is complete. The ask defines the scope.

- Ask: "create AGENTS.md" → scope is the full file once; terminate when the standard sections are populated.
- Ask: "add a section for X" → scope is that section only; do not touch other sections.
- Ask: "update build commands" → scope is the Build/Test/Lint section only.
- Ask: "apply audit findings" → scope is the listed findings; do not add findings of your own.

If you find yourself wanting to "also fix" something outside the scope, STOP. Report it to the developer at the end and let them decide whether to invoke a new run.

## AGENTS.md philosophy

Three principles, in order of importance:

1. **AGENTS.md is a briefing, not a textbook.** It tells the agent things that are only true in THIS repo. General language conventions, framework patterns, and idiomatic style belong in installed skills, not here. If you can imagine the same sentence in ten other repos' AGENTS.md, it does not belong in any of them.

2. **Skills are authoritative for their domain.** AGENTS.md defers to installed skills for topics within their scope. AGENTS.md only overrides a skill when it explicitly marks the rule as a repo-specific override.

3. **Terse and imperative.** Every sentence is either a fact the agent needs to act on or an instruction. No motivation, no history, no "this repo was started because…" — that belongs in README.md.

## Standard structure

A AGENTS.md has these sections, in this order. Omit sections that do not apply to the repo. Do not add sections not on this list without explicit developer request.

### 1. Project identity

Two or three sentences. What this repo is, what it produces, and its role in the ecosystem. Nothing more.

### 2. Applicable skills

List of the skills vendored at `agent-skills/` (a pinned submodule; see `.gitmodules`) that apply to work in this repo, with a one-line note on what each governs. Example:

```
- dev-standards: universal — verification-before-claiming, read-before-editing, minimal-changes discipline
- go-backend-dev: authoritative for Go conventions, error handling, testing patterns
- frontend-design: authoritative for React component structure and styling
```

Universal skills (those that apply to every repo regardless of stack) are listed first, marked with "universal —". Stack-specific skills follow.

Precedence rule (include verbatim in every AGENTS.md):

> **Precedence.** When this AGENTS.md and a referenced skill give conflicting guidance, the skill wins for topics within its scope. This AGENTS.md only overrides a skill when it explicitly says "repo-specific override:" followed by the rule.

### 3. Architectural invariants

Principles and constraints that rarely change. Guardrails, not implementation details. Rule of thumb: if it changed in the last two weeks, it is not an invariant yet — leave it out until it stabilizes.

Examples:
- "Safety scenarios are binary pass/fail, never scored."
- "No CGO dependencies; pure Go only."
- "All public API changes require a spec version bump."

### 4. Repo dependencies

Git submodules, with path and purpose. For each submodule include a single instruction:

> When starting work that touches [area], check `git submodule status` and update with `git submodule update --remote [path]` if behind.

### 5. Build / test / lint commands

Minimum set to validate changes. No explanation, just commands:

```
go build ./...
go test ./...
golangci-lint run
```

### 6. Repo-specific conventions

Conventions unique to THIS repo that are NOT covered by the applicable skills. Anything that would duplicate a referenced skill belongs in the skill, not here. If you find yourself writing a convention that could apply to any Go repo, delete it and trust the Go skill.

### 7. File layout notes

Only if the repo has a non-obvious layout the agent needs to know about. Example: "Generated code lives in /gen; never edit by hand." Skip this section if the layout is standard for the language.

## Language and skill detection

When creating a new AGENTS.md or updating the Applicable Skills section:

1. Detect primary language(s) by looking for: `go.mod`, `package.json`, `pom.xml`, `Cargo.toml`, `pyproject.toml`, `requirements.txt`, `*.csproj`. A repo can have more than one.

2. List vendored skills with `ls agent-skills/` and read each SKILL.md's description field.

3. Match in three tiers:

   **Tier 1 — Universal skills (always include).** Any skill whose description contains the exact phrase `"applies to all repos"` is universal and must be referenced in every AGENTS.md regardless of the repo's stack. List these first in the Applicable Skills section, prefixed with `universal —`.

   **Tier 2 — Stack-specific skills (name-based match).** A skill named `go-*` matches a Go repo, `frontend-*`/`react-*`/`vue-*` matches a repo with package.json and a frontend framework, `python-*` matches a Python repo, `java-*` matches a Java repo, etc.

   **Tier 3 — Stack-specific skills (description-based fallback).** If a skill's name does not match any pattern, scan the description field for language or framework keywords that match the repo's detected stack.

4. Include matched skills in the Applicable Skills section in tier order: universal first, then stack-specific. If no skills match at all, omit the section entirely rather than leaving it empty.

### Note on universal-skill opt-in

The phrase `"applies to all repos"` is the opt-in marker for universal skills. If the developer has a skill that should be universal but does not contain this phrase in its description, flag it at the end of your response under "Noted but not addressed" so the developer can update the skill's description. Do not modify the skill yourself — editing other skills is out of scope.

## Writing style

- Imperative voice: "Use X" not "You should use X."
- No hedging: "Never commit generated files" not "Generally, it's best to avoid committing generated files."
- No motivation: "All tests must pass before merge" not "Because we value quality, all tests must pass before merge."
- Bullet points over prose for lists of facts.
- Code blocks for commands and paths, never inline.

## What NOT to do

- Do NOT copy content from installed skills into AGENTS.md. Reference them instead.
- Do NOT add sections for "recent changes" or "TODO" — AGENTS.md is not a changelog.
- Do NOT write onboarding narrative. That is README.md's job.
- Do NOT invent invariants. If the developer did not tell you something is invariant, it is not.
- Do NOT touch sections outside the scope of the current ask.

## Completion

When the ask is complete:
1. Show the developer the diff (or the full file for creation).
2. If you noticed anything outside the scope that might need attention, list it as a single "Noted but not addressed" section at the end of your response.
3. Stop. Do not continue looking for things to improve.
