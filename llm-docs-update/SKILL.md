---
name: llm-docs-update
description: One-shot AGENTS.md update. Runs a session-reflection pass AND a repo-drift audit, merges findings, and applies approved changes via llm-docs-authoring. Use when asked to "update AGENTS.md", "refresh AGENTS.md", "sync AGENTS.md", or at session end when you want to capture both session learnings and repo drift in one go.
metadata:
  version: "1.0.0"
---

# LLM Docs Update

## Purpose

Single invocation point for keeping AGENTS.md current. Runs two independent passes — session reflection and repo drift audit — merges their findings, shows them to the developer, and hands approved changes to llm-docs-authoring.

Use this when you don't want to decide whether a session revealed learnings vs. whether the repo drifted. Run this, review findings, approve.

## Termination rule (read this first)

This skill orchestrates but does not hunt. It is bounded by construction:

1. **Exactly two passes, in this order.** Reflection pass, then audit pass. Each pass has its own fixed scope. Do not add a third pass.

2. **Each pass runs once.** Do not re-run reflection after audit. Do not loop.

3. **Merging is mechanical.** Concatenate findings, de-duplicate obvious overlaps, present. Do not "improve" findings during merge.

4. **You do not edit AGENTS.md yourself.** Only llm-docs-authoring edits. Your job ends when you hand off the approved findings to it.

5. **No findings is a valid outcome.** If both passes come back empty, report "AGENTS.md is current" and stop. Do not manufacture findings.

If you catch yourself wanting to do a third pass, re-verify, or start editing: STOP. Finish the handoff and end the skill.

## Pass 1: Session reflection

Source of truth: **this conversation's history only**. You are looking for knowledge that surfaced in the session but is not in the repo state (and therefore would be invisible to the audit pass).

### What reflection catches

Look for these patterns in the conversation:

- **Environment or config requirements the agent had to be told.** "You need `POSTGRES_URL` set for DB tests." "The dev server needs `--zone=dev` locally or it panics."
- **Workarounds the agent discovered.** "Running `go generate` before `go build` is required." "Kill port 3000 before `npm run dev` or Next.js auto-switches to 3001."
- **Gotchas hit during the session.** "This linter rule has a known false positive here; use `//nolint:gosec` with a comment."
- **Non-obvious ordering or dependencies.** "The auth package must initialize before crypto." "Regenerate protos before touching the handler."
- **Corrections the developer gave the agent.** If the developer said "no, we do X here, not Y," that is a repo convention the agent did not know. Flag it.
- **Tools or commands the agent had to be told about.** "We use `just` not `make` here." "Tests go through `scripts/test.sh`, not `go test` directly."

### What reflection does NOT catch

- Things visible in the repo state (those are the audit pass's job).
- Style preferences the developer expressed for this one message ("be more concise"). AGENTS.md is not a tone guide.
- One-off debugging back-and-forth where nothing durable was learned.
- General language conventions the agent should know from a skill. If the reflection finding would duplicate a referenced skill, drop it.
- Emotional or conversational content. "I was frustrated with the output" is not a learning.

### Reflection output

Produce a list of candidate findings, each with:

- **What was learned:** one sentence.
- **Where it belongs:** which AGENTS.md section (Architectural invariants / Repo-specific conventions / Build commands / File layout notes / etc.).
- **Evidence:** brief reference to the turn(s) in the session where this surfaced.

If no durable learnings surfaced, emit an empty list and move on. Do not pad.

## Pass 2: Repo drift audit

Delegate the full audit scope and rules to the llm-docs-audit skill's specification. Specifically, check these five surfaces, once each, in order:

1. Build/test/lint commands vs Makefile, package.json scripts, language entry points.
2. Submodule list vs `.gitmodules`.
3. Referenced file paths — do they exist?
4. Applicable skills — stale references (skills named in AGENTS.md but not in the vendored `agent-skills/`).
5. Applicable skills — missing references (vendored skills matching the repo's stack via the three-tier rule: universal marker `"applies to all repos"`, name-based, description-based).

Follow llm-docs-audit's termination rules: fixed scope, single pass, no re-verification. The rules in llm-docs-audit SKILL.md are authoritative for this pass — if they conflict with anything here, they win.

## Merge

Combine the two finding lists into a single document. Group by target section of AGENTS.md, not by which pass produced the finding. The developer does not care whether a finding came from reflection or audit; they care what AGENTS.md needs.

De-duplication: if reflection and audit produced the same finding (rare but possible — e.g., reflection notes "we use `just`" and audit notes `justfile` exists but AGENTS.md says `make`), merge into one finding and cite both sources.

## Output format

```
# AGENTS.md Update — [repo name]

## Summary
[One sentence: "N findings from reflection, M findings from audit" or "No findings; AGENTS.md is current."]

## Findings

### Architectural invariants
- [finding]  _(source: reflection, turn N)_
- [finding]  _(source: audit — referenced paths)_

### Repo-specific conventions
- [finding]  _(source: reflection)_

### Build / test / lint commands
- [finding]  _(source: audit — build commands)_

### Applicable skills
- [finding]  _(source: audit — missing skill references)_

### File layout notes
- [finding]  _(source: reflection)_

## Approval

Which findings do you want applied? Reply with:
- "all" to apply everything
- A list of finding numbers or section names to apply a subset
- "none" to discard
```

Omit sections with no findings.

## Handoff

Based on developer's reply:

- **"all" or a subset:** Invoke llm-docs-authoring with the approved findings as the scope. Phrase the handoff as "apply these findings:" followed by the selected finding list. llm-docs-authoring will produce the diff and apply it after its own confirmation step.
- **"none":** Stop. Report "No changes made."

## What this skill does NOT do

- Does NOT edit AGENTS.md directly. That is llm-docs-authoring's job.
- Does NOT re-verify findings after the developer approves them.
- Does NOT run a third pass if the first two come back empty.
- Does NOT reflect on sessions older than the current one.
- Does NOT reason about what the developer "really" wanted; the findings list is the findings list.

## Completion

After the handoff to llm-docs-authoring (or after "none"), stop. Do not offer a follow-up audit. Do not propose future improvements. The loop is closed.
