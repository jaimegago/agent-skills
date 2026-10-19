---
name: dev-standards
description: Load general development standards. Applies to all repos. Invoke at the start of a session or when standards need reinforcing.
user-invocable: true
metadata:
  version: "1.0.0"
---

You are now operating under the following non-negotiable development standards. Apply them for the remainder of this session without exception.

## Verify before claiming

Never make any statement about code — "this is unused", "this is dead code", "X calls Y", "this is never referenced" — without first running the appropriate search or read tool to confirm it. Every factual claim about the codebase must be grounded in evidence you have collected in this session.

If you have not yet searched for it, you do not know it.

## Read before editing

Always read a file before proposing or making changes to it. Never edit based on assumptions about what a file contains.

## Minimal changes only

Only change what is directly requested. Do not refactor, reformat, clean up, add comments, add error handling, or "improve" anything beyond the explicit ask. Three similar lines of code is better than a premature abstraction.

## Dead code deletion procedure

When asked to remove dead or unused code:
1. Search the entire codebase for all references to every symbol being removed.
2. Show the search results to the user.
3. Only then make the deletion.
Never skip step 1 and 2.

For step 1, an LSP reference search (`findReferences`) can **undercount** on some language servers (see "Code navigation" below) — for a deletion decision, confirm with a text search (`grep`) as well. An undercount here means deleting code that is actually used.

## Code navigation

When navigating or understanding existing code, prefer a language server (the `LSP` tool) over text search where one is available for the file type. A language server understands the type graph — definitions, references, implementations, call hierarchy — whereas text search matches strings, including comments, docs, and unrelated identifiers.

- "Where is this defined / what's its type?" → `goToDefinition` / `hover`
- "Who uses this?" → `findReferences`
- "Find a symbol by name across the project" → `workspaceSymbol`
- "What implements this interface?" → `goToImplementation`
- "Who calls this / what does it call?" → `incomingCalls` / `outgoingCalls`

Mechanics:
- LSP operations are position-based (`filePath` + `line` + `character`), so they need a starting location. Efficient pattern: one text search (or `workspaceSymbol`) to find the anchor, then LSP for the semantic graph from there.
- In some harnesses the `LSP` tool is **deferred** — load its schema (e.g. via `ToolSearch`) before the first call. Load it once at the start of a navigation task; don't let that one-time setup push you back to `grep` reflexively.
- Keep text search for what it's actually best at: non-code files, config, log/string literals, and raw text.

**Completeness varies by language server, and this interacts with "Verify before claiming."** Some servers index eagerly and return complete results; others index lazily and can return a confident-but-incomplete answer (e.g. a cross-file `findReferences` that misses files not yet loaded). Treat a suspiciously low cross-reference count as unverified, not as fact. The language-specific skill (e.g. go-backend, frontend-dev) documents the per-server behavior — defer to it.

## Git discipline

Never commit, amend, push, or perform any destructive git operation unless the user explicitly asks for it in that message. Approving a push once does not authorize future pushes.

## When blocked

If a tool call fails, is denied, or an approach is blocked: stop, diagnose why, consider an alternative. Do not retry the same action. Do not brute-force past safety checks.

## Honesty over fluency

Do not generate plausible-sounding answers ahead of evidence. If you are uncertain, say so. If you need to search first, search first — then answer.
