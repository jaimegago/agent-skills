---
name: git-commit
description: Commit staged and unstaged changes with an auto-generated Conventional Commit message, then sync the branch with the remote. Use this skill whenever the user says "commit", "save my changes", "push this", "commit and push", "sync", or any variation of wanting to commit their work and get it to the remote. Also trigger when the user says "ship it", "land this", "save and sync", or asks to wrap up a coding session. Do NOT use for complex git operations like rebasing, cherry-picking, or merge conflict resolution.
metadata:
  version: "1.0.0"
---

## Workflow

Execute these steps in order. Stop and report to the user if any step fails.

### 1. Assess the working tree

Run git status and git diff (both staged and unstaged) to understand what changed. If there are no changes at all, tell the user there is nothing to commit and stop.

### 2. Clean dirty submodules

If `git status` shows submodules with "(modified content)" but no staged or intended submodule pointer changes, those are nested sub-submodules that drifted (typically from a recursive `--remote` update). For each such dirty submodule, run `git -C <submodule-path> submodule update --init` to restore the nested references to the commits expected by the parent submodule. After cleanup, re-check `git status` to confirm the dirt is gone. Do NOT stage submodule pointer changes unless the user explicitly changed them.

### 3. Stage everything unless the user specified otherwise

If the user did not specify particular files, stage all changes (tracked and untracked) with git add -A. If the user named specific files or paths, stage only those.

### 4. Lint and auto-fix

Before committing, detect the project's linter and run it with auto-fix on staged files. Detection order:

- **Project-level lint target first**: if Makefile has a `lint` or `lint:fix` target, or package.json has a `lint` or `lint:fix` script, prefer that — it respects the project's own conventions. Move on to per-language detection only if no project-level target exists.
- **Go** (go.mod present): run `gofmt -w` on staged .go files, then `go vet ./...`
- **JavaScript/TypeScript** (package.json present): run `npx eslint --fix` on staged JS/TS files if an ESLint config exists. If Prettier is configured, also run `npx prettier --write` on those files.
- **Python** (pyproject.toml, setup.py, or requirements.txt present): run `ruff check --fix` and `ruff format` on staged .py files if Ruff is configured. Fall back to `black` if installed. Fall back to `flake8` (report-only, no auto-fix).
- **Rust** (Cargo.toml present): run `cargo fmt` and `cargo clippy --fix --allow-dirty`
- **No linter detected**: skip this step and proceed.

After auto-fix, re-stage any modified files with `git add -u`. If the linter reports errors that could not be auto-fixed, stop and report them to the user — do not commit with known lint failures.

If a linter is detected but the binary is not installed, warn the user and ask whether to proceed without linting or abort.

### 5. Generate the commit message

Analyze the staged diff to produce a Conventional Commit message:

    type(scope): short summary

    Optional body with context on what changed and why.

Rules:
- type: one of feat, fix, refactor, docs, test, chore, ci, perf, style, build
- scope: optional but preferred — derive from the primary package, module, or directory touched. Omit only when changes span too many areas for a single scope to be meaningful.
- Summary line: lowercase, imperative mood, no period, max 72 characters.
- Body: add only when the diff is non-trivial and the summary alone would not give a reviewer enough context. Keep it concise.
- Multiple logical changes: pick the dominant one for type/scope, mention secondary changes in the body. Do NOT split into multiple commits unless the user explicitly asks.
- Breaking changes: add a BREAKING CHANGE footer or use the ! suffix (e.g., feat!: ...).
- If the linter auto-fixed files, mention it in the body (e.g., "Includes gofmt auto-fixes.").

### 6. Commit

Run git commit with the generated message. Use -m for summary-only messages. For messages with a body, write to a temp file and use git commit -F to preserve newlines, then clean up the temp file.

### 7. Sync with remote

Determine the current branch name. Then:
- If the branch has an upstream, run git push.
- If the branch has no upstream, run git push -u origin <branch-name>.
- If push is rejected because the remote is ahead, run git pull --rebase then push again. If the rebase produces conflicts, stop and report — do not auto-resolve merge conflicts.

### 8. Confirm

Report to the user:
- Commit hash (short form)
- Commit message used
- Branch name and remote sync status
- Files changed, insertions, deletions (from git diff --stat of the commit)
- Which linter ran (or that none was detected)

## Edge cases

- No remote configured: commit locally and tell the user.
- Detached HEAD: warn before committing, suggest creating a branch first.
- Only untracked files: stage and commit them normally.
- Respect .gitignore — never force-add ignored files unless the user explicitly asks.
