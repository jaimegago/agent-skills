---
name: go-code-review
description: Code quality review for Go repositories. Spawns a thorough Explore agent to scan for context, error-handling, antipattern, nil/goroutine, and convention issues, spot-checks the top findings, then writes an ordered fix plan under docs/analysis/. Picks scope (uncommitted changes, branch diff, or full repo) and filename based on git state. Use when the user says "review this code", "review this MR", "review this branch", "code quality check", "find issues in this code", or after a refactor lands before opening a PR. Do NOT use for single-function reviews or chat-style "is this idiomatic" questions.
metadata:
  version: "1.0.0"
---

# go-code-review

A repo-agnostic Go code review skill. Workflow: orient → pick scope → spawn thorough Explore agent → spot-check HIGH findings → write ordered fix plan → short summary.

## When to invoke

- The user asks to review code, an MR, a branch, or "find issues" in a Go repo.
- A refactor or feature phase is complete on a branch, before opening the PR.
- Re-entering a Go repo after a gap and wanting to know what has accumulated.

Do not invoke for single-function reviews, "is this idiomatic" chat questions, or on a clean branch with no changes since the last review.

## Repo prerequisite

This skill is for Go repos. Before starting, confirm a `go.mod` exists at the repo root (or in a recognized subdirectory). If not, stop and tell the user this skill is Go-specific.

---

## Workflow

Execute these steps in order.

### Step 1 — Orient: load conventions

Look for a repo conventions document, in this order:

1. `AGENTS.md` at repo root
2. `CLAUDE.md` at repo root (the older name of the same file)
3. `CONTRIBUTING.md` at repo root
4. `docs/conventions.md`
5. `.cursor/rules/` directory

Read the first one that exists. Note any patterns that should be enforced as part of this review (naming, layering, instrumentation placement, interface assertions, etc.).

If none of those exist, ask the user once: "I couldn't find a conventions document (checked AGENTS.md, CLAUDE.md, CONTRIBUTING.md, docs/conventions.md, .cursor/rules/). Where are this repo's Go conventions documented? If there isn't one, say 'skip' and I'll review against generic Go idioms only."

Do not invent conventions. If there is no doc and the user says skip, the convention category in Step 3 is dropped.

### Step 2 — Determine scope and output filename

Pick the review scope and the output filename together, based on local git state. Run the checks in this order; stop at the first match.

Determine the default branch first (typically `main`, fall back to `master` if `main` doesn't exist locally). Compute `<branch-slug>` as the current branch name lowercased with non-alphanumerics collapsed to hyphens.

**Scope decision tree:**

1. Working tree is dirty (`git status --porcelain` returns anything):
   - Scope: `git diff HEAD` plus any untracked Go files listed by `git status`.
   - Filename: `docs/analysis/pre-commit-<YYYY-MM-DD>.md`.

2. Clean tree, branch is ahead of the default branch (`git rev-list --count <default>..HEAD` > 0):
   - Scope: `git diff $(git merge-base HEAD <default>)..HEAD`.
   - Filename: `docs/analysis/branch-<branch-slug>.md`.

3. Clean tree, on the default branch with no commits ahead, or the user explicitly said "review the whole codebase":
   - Scope: all non-vendor Go source. Discover via `go list -f '{{.Dir}}' ./...` (excluding vendor) or fall back to scanning everything outside `vendor/`, `node_modules/`, `.git/`.
   - Filename: `docs/analysis/review-<YYYY-MM-DD>.md`.

**Filename collisions:**

If the chosen filename already exists, append a new section dated today (`## <YYYY-MM-DD> follow-up`) rather than overwriting.

State the chosen scope and filename to the user in one sentence before spawning the Explore agent.

### Step 3 — Spawn the Explore agent

Spawn a single Explore agent with search breadth **very thorough**. The agent must read whole files, not excerpts. Pass it the scope from Step 2 (specific file list, not "the whole repo" unless that is the chosen scope).

Instruct it to look for:

**1. Context discipline**
- `context.Background()` created inside a function whose caller already has a real context (propagation broken)
- `context.TODO()` left in non-placeholder code
- `context.Context` stored in a struct field

**2. Error handling**
- Errors silently discarded: `_ = someFunc()`
- Log-and-return double handling: `log.Error()` immediately followed by `return err` at the same call site (logging belongs at one layer)
- `log.Fatal` or `panic` outside `cmd/main.go`
- `fmt.Errorf` without `%w` when wrapping (loses the error chain)

**3. Go antipatterns**
- Inconsistent receiver names for the same type across files (e.g., `a` in some files, `s` in others)
- Returning a typed nil from an interface-returning function (callers' `!= nil` check will be wrong)
- Named return values used with bare `return` statements (obscures what is returned)
- Logger formatting via `fmt.Sprintf` inside a logging call when the logger has a native formatted variant (allocates for nothing)

**4. Stdlib reinvention**
- Duplicated string-manipulation patterns across files that should be a shared helper
- Chained `strings.ReplaceAll` where `strings.NewReplacer` or `strings.FieldsFunc` would be cleaner
- `err.Error() == "some string"` instead of `errors.Is(err, sentinel)` or type-asserting the error

**5. Nil pointer and concurrency hazards**
- Ranging over a channel that is never closed (blocks forever)
- Unguarded map write from multiple goroutines (data race)
- Dereference of a pointer returned from a function that could be nil without a nil check

**6. Goroutine discipline**
- Goroutines launched without a `sync.WaitGroup` or done-channel (no way to join)
- Goroutine that writes to a channel after the receiver has already returned (write to abandoned channel)
- Missing context cancellation propagation into a goroutine's blocking call

**7. Magic values**
- Hardcoded values inside function bodies that should be package-level constants or config fields (string keys, identifiers, paths, addresses, role names)
- Inline time arithmetic (e.g., `* 7 * 24 * 60 * 60`) instead of `time.Duration` helpers

**8. Convention violations** (only if Step 1 found a conventions doc)
- Pass the conventions doc to the agent and have it flag deviations specifically. Do not invent rules. If no conventions doc was found, skip this category entirely.

For each issue: file path, line number, category, exact code snippet, explanation, suggested fix, and severity (HIGH / MEDIUM / LOW). Severity guidance: HIGH = correctness, data race, or production failure mode; MEDIUM = idiom or maintainability; LOW = style.

### Step 4 — Spot-check

Before writing the plan, directly read the file lines for **every HIGH-severity finding** the agent reported. Confirm the code is exactly as described. Correct or drop any finding where the agent misread the file.

Spot-check at least the top 3 findings by severity even if all seem credible.

### Step 5 — Write the plan

Write the verified findings to the filename chosen in Step 2 as an ordered task list.

Header: scope reviewed (commit range or file list), date, conventions doc consulted (if any).

Ordering rules:
1. Prerequisite changes first (e.g., receiver name rename before any other edit, interface changes before implementation changes, signature changes before caller updates).
2. Within the same priority tier, group by file to minimize context switching.
3. Each task must include: the issue, the file path and line number(s), the exact before/after change, and any callers or tests that also need updating.
4. End with a verification checklist appropriate to the repo: typically `go build ./...`, `go vet ./...`, the repo's lint command (e.g., `make lint` or `golangci-lint run`), and `go test ./...`.

### Step 6 — Report to the user

Summarise in one short paragraph:
- How many issues found, by severity (HIGH / MEDIUM / LOW)
- The single most impactful fix (and why)
- Confirm the plan was written to the chosen filename
- Any finding that was dropped after spot-checking (and why)

Do not list every finding in the summary — the plan file is the record.
