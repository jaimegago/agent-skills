---
name: save-prompt
description: Save the prompt that produced generated code before committing. Use when the user says "save prompt", "capture prompt", "save the prompt and commit", or wants to preserve the specification that led to LLM-generated code. Also trigger when the user says "save what we did", "capture this session", or "preserve the spec". Do NOT use for general documentation, READMEs, or non-LLM-generated code.
disable-model-invocation: true
allowed-tools: Bash(git *) Read Write
argument-hint: [target-file-or-directory]
---

## Purpose

Preserve the prompt specification that produced LLM-generated code. The prompt is a source artifact — it captures the intent, constraints, and design decisions that led to the generated code and enables future regeneration, auditing, and iteration.

## Workflow

Execute these steps in order. Stop and report to the user if any step fails.

### 1. Identify the target

Determine which generated file(s) this prompt produced:
- If the user passed an argument ($ARGUMENTS), use that as the target file or directory.
- If no argument was given, examine the current conversation to identify which files were generated or substantially modified by LLM output during this session. If ambiguous, ask the user.

### 2. Determine prompt file location

Place all prompt files in `docs/prompts/` at the repo root. Create the directory if it does not exist.

Naming convention (in priority order):
- Use the current git branch name: branch `feature/auth-service` becomes `docs/prompts/feature-auth-service.prompt.md` (replace slashes with hyphens).
- If on `main` or `master`, fall back to the primary target: `foo.go` becomes `docs/prompts/foo.prompt.md`. If the target is a directory or multiple files, use the most descriptive name: `docs/prompts/auth-service.prompt.md`.
- If a file with that name already exists, ask the user whether to overwrite or version it (e.g., `foo.prompt.v2.md`).

### 3. Distill the effective prompt

Analyze the full conversation history for this session. Produce a single, self-contained prompt that captures the final state of the specification — what you would hand to an LLM to regenerate this code from scratch.

The distilled prompt must:
- Be written as a specification: requirements, interfaces described in prose, behaviors, constraints, acceptance criteria.
- NOT contain markdown code fences or code implementations.
- NOT contain conversational artifacts (debugging back-and-forth, false starts, abandoned approaches).
- Include any constraints or decisions that emerged during iteration but were not in the original request.
- Be concise. Aim for the minimum text needed to reproduce the output.

### 4. Ask about transcript

Ask the user whether they also want to save the full session transcript alongside the distilled prompt. If yes:
- Save it as `<same-basename>.prompt-session.md` in the same location.
- The transcript should be a faithful capture of the conversation, including the back-and-forth, not just the final prompt.

### 5. Write the prompt file

Create the prompt file with this structure:

```
# Prompt: <short description>

Generated: <date>
Model: <model used, if known>
Target: <relative path(s) to generated file(s)>

## Specification

<the distilled prompt>
```

Do not add any other sections unless the user requests them.

### 6. Stage the prompt file(s)

Run `git add` on the prompt file (and transcript file if created). Do NOT commit — the user will invoke the git-commit skill separately, or chain it after this one.

### 7. Confirm

Report to the user:
- Prompt file path created
- Transcript file path (if created)
- Target file(s) the prompt is associated with
- That files are staged and ready to commit

## Edge cases

- If the conversation history is too short or trivial to distill (e.g., a single one-liner), save the original prompt verbatim — no distillation needed.
- If a prompt file with the same name already exists, this is handled in step 2.
- If the user invokes this skill but no code was generated in the session, tell them and stop.
- Respect .gitignore — do not stage prompt files in ignored directories.
