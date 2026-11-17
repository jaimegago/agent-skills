---
name: dev-standards
description: Load general development standards; applies to all repos. Invoke at the start of a session or when standards need reinforcing.
user-invocable: true
metadata:
  version: "1.2.0"
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

## TODO and DECISIONS discipline

When writing a TODO comment in code, a DECISIONS.md entry, or any code comment that references an external contract — library API, protocol field name, tool flag, schema shape, version-specific behavior — either verify the specific claim at write time via doc fetch or source read, or mark the unverified phrase inline with "— unverified, from training."

A TODO that embeds a confident-sounding but unverified specific is a trap for future sessions: it gets read later as established fact, not as a hypothesis. A TODO that flags its own uncertainty is self-defending.

Examples.

Bad: `// TODO(large-output): add anthropic/maxResultSizeChars annotation when implemented.`
Good: `// TODO(large-output): add size annotation when implemented. Field name unverified — check MCP spec / SDK source before using.`

Bad: `// TODO: use github.GetRepoContents() to walk the tree.`
Good: `// TODO: walk the tree via GitHub Contents API. Endpoint shape and rate limit behavior — unverified, from training; verify before implementing.`

The rule fires whenever you're about to commit *written* text (code comment, doc entry, spec file) that names an external thing you haven't verified this session. "From training, unverified" is a valid thing to write; fabricating confidence is not.

## Pin a behavior fix

A commit that corrects behavior ships a test that **fails without the fix and passes with it**. Prove that by revert-run-restore, not by reading the test:

1. Revert the fix in the working tree, leaving the test in place.
2. Run that specific test and confirm it fails for the reason the fix addresses — not a compile error, not an unrelated assertion.
3. Restore the fix and confirm the test passes.

A test written against a fix that was never reverted is an assumption, not a pin: it can assert something the fix did not cause, or something that was already true. The pin is part of the fix, so "Minimal changes only" does not excuse leaving it out.

**Revert the fix's hunks, not its files, when later commits touched the same file.** A whole-file revert is the obvious first attempt, and it removes the later changes along with the fix: the build breaks, and a test that fails because nothing compiles has proven nothing. Reverse only the lines the fix introduced. In joe, an audit that re-proved the pins on a run of bug fixes found two of them revertible only this way.

**A change that corrects no behavior gets a recorded not-pinned rationale, not a contrived test.** Write down, where the repository records such things (commit message, pull request, decision log), that the change is not pinned and why. Two that joe exempted:

- Swapping a static favicon asset. There is no runtime surface, and the only possible test asserts the file's contents back at itself.
- Renaming UI label strings. Labels changed, behavior did not, and a test over the strings pins the copy against the very edits it should permit.

The record is what lets a reviewer tell "no test" from "forgot the test". A silent omission reads as the second.

## Pin wiring at the composition root

Unit-level pins do not protect the wiring. A unit test constructs its own subject, so it is blind to how production assembles it: a dead consumer, a duplicate instance and a discarded argument all pass every unit test, because that seam is exactly what unit tests abstract away.

The worked example, from joe (D-0107). Governance resolvers were wired into a policy engine whose only transport consumer was a middleware that had long since become a pass-through discarding its engine argument. The engine actually gating requests was a second, bare one built inside the API constructor, carrying neither resolver. The resolvers, the engine, the middleware and the accessor all had passing tests, and a read the launch default should have admitted returned `403` on the real transport path. Nothing tested the wiring.

When a defect's cause is "which instance was injected where", the fix owes both of these:

- **A static construction guard**: a test asserting the constructor is called only where it may be. Joe's `TestGuard_PolicyEngineConstructedOnlyAtCompositionRoot` fails on any call to the engine's constructors outside the binary's main package and test files. That makes a second instance impossible to reintroduce, rather than merely absent today.
- **An end-to-end pin through the production composition path**: extract the assembly into a named seam that the entrypoint and the test both call (joe's `buildHTTPHandler`), drive a real request through it, and assert the governed outcome. A pin that constructs its own subject proves nothing about the binary.

Ask of any test for a wiring-shaped fix: **would this test fail if the object under test were never wired into the binary at all?** If not, it is not a pin.
