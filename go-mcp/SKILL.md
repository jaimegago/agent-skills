---
name: go-mcp
description: Use this skill whenever the user is building, extending, or reviewing a Model Context Protocol (MCP) server or client in Go using the official modelcontextprotocol/go-sdk. Triggers include: creating a new MCP server from scratch, adding tools/resources/prompts/sampling/elicitation/roots to an existing server, choosing a transport (stdio, Streamable HTTP, SSE), reviewing MCP-specific Go code, wiring up testing with in-memory transports, diagnosing MCP handshake or capability bugs, or embedding an MCP server inside a larger Go binary as a subcommand. Apply this skill any time the user mentions "MCP", "mcp-go", "modelcontextprotocol/go-sdk", or references files containing mcp.NewServer, mcp.AddTool, CallToolRequest, StreamableHTTPHandler, or equivalent symbols — even if they don't explicitly ask for "MCP help".
metadata:
  version: "1.0.0"
---

# go-mcp

Build and review MCP servers and clients in Go using the official `github.com/modelcontextprotocol/go-sdk`. This skill covers what is specific to MCP — protocol shape, SDK API surface, transport tradeoffs, primitive semantics, testing patterns. It deliberately does not cover general Go idioms.

## Companion skill: go-backend

For anything not specific to MCP — package layout, error handling, context propagation, struct tag style, logging with slog, graceful shutdown, dependency injection, goroutine hygiene, test organization — defer to `go-backend` if it is loaded. If `go-backend` is not available in the session, apply standard idiomatic Go and flag that the companion skill is recommended.

Do not duplicate guidance from `go-backend` here. When a question spans both (e.g. "how should I structure a cobra subcommand that runs an MCP server over stdio"), use `go-backend` for the cobra/command structure and this skill for the MCP server wiring.

## SDK version discipline

The Go SDK is still evolving and has had breaking changes as recently as the v1.x series (ClientCapabilities.RootsV2, Capabilities fields on ServerOptions/ClientOptions, elicitation defaults, URL-mode elicitation, tool name validation). Before giving any authoritative answer about SDK behavior:

1. Check the repo's `go.mod` for the pinned SDK version.
2. Consult the current API surface at `https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp` (and adjacent subpackages: `jsonrpc`, `auth`, `oauthex`, `jsonschema`).
3. For spec-level questions, cross-reference `https://modelcontextprotocol.io/specification/` at the version advertised by the pinned SDK release.
4. If uncertainty remains about whether an API exists at the pinned version, say so explicitly rather than guessing.

## Mode selection

Decide early which mode this task is in and stay there:

**Greenfield** — creating a new MCP server or client from scratch, or adding a new primitive (tool, resource, prompt, etc.) that doesn't yet exist in the codebase. Read `references/greenfield.md`.

**Review** — the user is asking you to audit existing MCP code, find bugs, suggest improvements, or check conformance. Read `references/review.md` and `references/review-checklist.md`.

**Both** — some tasks (e.g. "extend this server with a new tool") are review plus greenfield. Do the review pass first on the surrounding code so the new addition fits the existing patterns, then apply greenfield guidance.

If the user's intent is ambiguous between these, ask one short clarifying question before proceeding.

## Scope — what this skill covers in depth

All MCP primitives and client features, because the spec is deeply interconnected and skipping any of them leaves blind spots:

- **Tools** — typed handlers via generic `mcp.AddTool[In, Out]`, schema inference, structured vs unstructured results.
- **Resources** — static resources, resource templates, subscriptions, list-changed notifications.
- **Prompts** — prompt templates with arguments, message construction.
- **Sampling** — server-initiated LLM calls via the client.
- **Elicitation** — form-mode and URL-mode (URL mode is mandatory for credentials; form mode is forbidden for them).
- **Roots** — client-side filesystem roots, roots-changed notifications, the `RootsV2` compatibility workaround.
- **Logging, progress, cancellation** — the utilities that cut across primitives.

See `references/primitives.md` for the behavior contract of each.

## Scope — what this skill covers at surface level

- **Transports** — stdio, Streamable HTTP, SSE (legacy), in-memory. See `references/transports.md` for when to pick which and what each costs.
- **Testing** — in-memory transports, deterministic fixtures, golden-file patterns for tool schemas. See `references/testing.md`. This section is mandatory for any greenfield task.
- **Auth** — the `auth` and `oauthex` subpackages exist for OAuth 2.1 and Protected Resource Metadata. This skill points at them but does not teach OAuth.

## Scope — what this skill deliberately excludes

- Go idioms, package layout, error wrapping, slog setup — `go-backend`.
- OAuth 2.1 flow design — defer to the MCP security spec and `auth`/`oauthex` docs.
- Alternative SDKs (`mark3labs/mcp-go`, `mcp-golang`, `go-mcp`) — this skill is specific to the official SDK. If the repo uses a different SDK, say so and stop.
- Client UX for MCP hosts (Claude Desktop, Cursor, etc.) — out of scope.

## Embedded vs standalone servers

MCP servers in Go come in two shapes, and the guidance differs slightly:

**Standalone** — a single-purpose binary whose `main` just builds a server, registers features, and blocks on `server.Run(ctx, transport)`. Usually distributed via `go install` or as a prebuilt binary invoked over stdio by a host.

**Embedded** — an MCP server is one subcommand inside a larger Go binary (typically cobra-based), sharing the host process's dependency graph, configuration, logging, and lifecycle. The MCP server is constructed from injected collaborators rather than global state.

The SDK API is identical between the two. The differences are integration concerns:

- **Embedded servers** get their `context.Context` from the parent command and must respect its cancellation. They share the parent's logger (usually an `*slog.Logger` passed in). Their registered tools close over injected collaborators, not package-level singletons. Test doubles for those collaborators are already established in the host's test suite — reuse them.
- **Standalone servers** own their lifecycle, typically setting up `signal.NotifyContext` in `main` and passing that `ctx` to `server.Run`.

When writing a new MCP server as part of a larger binary, default to embedded: construct the `*mcp.Server` inside a factory function that takes the host's dependencies and returns the server, then register tools/resources/prompts in that factory. Keep `main` thin.

## Minimum viable server (orientation only)

This is not a template — it exists so the rest of the skill has a referent when it says "the server". Real code should follow `go-backend` for structure and error handling.

```go
import "github.com/modelcontextprotocol/go-sdk/mcp"

server := mcp.NewServer(
    &mcp.Implementation{Name: "my-server", Version: "v0.1.0"},
    nil, // *ServerOptions
)
mcp.AddTool(server, &mcp.Tool{Name: "greet", Description: "say hi"}, greetHandler)

// Then run over a transport, e.g. stdio:
// err := server.Run(ctx, &mcp.StdioTransport{})
```

Handler shape:

```go
func greetHandler(ctx context.Context, req *mcp.CallToolRequest, in GreetInput) (*mcp.CallToolResult, GreetOutput, error) { ... }
```

The generic `AddTool` infers both input and output schemas from the type parameters (map or struct). Returning `(nil, out, nil)` is the common success path — the SDK synthesizes the `CallToolResult` from the typed output. Return a non-nil `*CallToolResult` only when you need fine control (custom content blocks, `IsError: true`, etc.).

## Invariants that apply to every MCP server

These are non-negotiable regardless of mode:

1. **Context is sacred.** Every tool, resource, and prompt handler receives a `ctx`. If the handler does any work that can block (I/O, sleeps, external calls), it must be cancellable via `ctx`. The SDK cancels the handler's context when the client cancels the request; a handler that ignores this leaks goroutines and violates the spec's cancellation semantics.

2. **Schemas are the contract.** A tool's input and output schemas are what the LLM sees. If the struct tags are wrong or the description is vague, the tool will be called wrong or not at all. Invest in `jsonschema` struct tags and clear `Description` fields — see `references/primitives.md` § Tools.

3. **Handlers are pure with respect to the session.** Don't stash per-call state in package-level variables. If a handler needs session-scoped state, use the request's session (`req.Session()`) or close over injected collaborators that are themselves safe for concurrent use.

4. **Errors on the wire vs errors in Go.** A handler returning a Go `error` terminates the request with a JSON-RPC error. Returning `&CallToolResult{IsError: true, Content: [...]}` with a nil Go error signals a *tool-level* failure the LLM should see and potentially recover from. These are different and must not be conflated — see `references/review-checklist.md` § Error discipline.

5. **Capabilities are advertised by what you register, not by what you claim.** Adding a tool implicitly adds the `tools` capability. Setting an `ElicitationHandler` on the client adds `elicitation`. Do not hand-author capability structs except to *disable* a default (e.g. suppressing the default `roots` advertisement on a client that doesn't support roots).

6. **Testing uses in-memory transports, not subprocess launches.** `mcp.NewInMemoryTransports()` returns a connected pair. Subprocess tests are reserved for a handful of end-to-end sanity checks. See `references/testing.md`.

## Workflow — greenfield

1. Confirm SDK version from `go.mod`. If absent, propose the latest stable and note it.
2. Decide embedded vs standalone (see above).
3. Decide transport (see `references/transports.md`).
4. Enumerate the primitives needed. For each: draft the input/output types, the description, and a one-line behavior spec.
5. Write the typed handlers. Keep them thin — orchestration in the handler, work in collaborators.
6. Register everything on the server via the generic `mcp.AddTool[In, Out]` and the equivalents for resources/prompts.
7. Write tests with in-memory transports before wiring the real transport. See `references/testing.md` — this step is not optional.
8. Only then wire the real transport and run end-to-end.

See `references/greenfield.md` for the expanded version.

## Workflow — review

1. Identify the SDK version and the MCP spec version it targets.
2. Map the code: which primitives are registered, which transport is used, which capabilities are advertised vs used.
3. Walk the checklist in `references/review-checklist.md`.
4. For each finding, explain the failure mode (what breaks, under what condition) — not just "this is wrong". Reviewers who only name violations without tying them to runtime consequences are easy to ignore.
5. Prioritize findings by blast radius: spec violations > resource leaks / goroutine leaks > schema quality > style.

See `references/review.md` for the expanded version.

## When stuck

If the question can't be answered from the references plus the current code, the likely next moves are:

- Check the SDK's `examples/` directory — it is the authoritative source of idiomatic usage for new APIs.
- Check `docs/` in the SDK repo for feature-level documentation mapped to spec sections.
- Check recent releases for API changes: `https://github.com/modelcontextprotocol/go-sdk/releases`.
- For spec questions: `https://modelcontextprotocol.io/specification/` at the version your SDK targets.

Cite what you found and where. Do not present SDK behavior as established fact from memory when the SDK is this young.
