# Greenfield: building a new MCP server or client

Use this reference when creating a new MCP server, a new MCP client, or adding a previously-absent primitive to an existing server. For audits of existing code, use `review.md` instead.

## Before writing any code

Four decisions need to be made up front because each one constrains the others:

1. **SDK version.** Check `go.mod`. If the project is new, pick the latest stable release of `github.com/modelcontextprotocol/go-sdk` and record which MCP spec version it implements. The SDK's README has a compatibility matrix.

2. **Embedded or standalone?** Embedded means the MCP server is one subcommand inside a larger binary and receives its dependencies via constructor injection. Standalone means a single-purpose binary whose `main` builds the server and blocks on `server.Run`. Default to embedded when the MCP server exists to expose functionality of a larger system you're already building. Default to standalone when the MCP server is the product.

3. **Transport.** Default to stdio. Choose Streamable HTTP only if there's a concrete requirement for network exposure, multi-client fan-out, or hosts that can't spawn subprocesses. See `transports.md` for the tradeoff matrix.

4. **Primitive inventory.** List every tool, resource, prompt, sampling hook, elicitation, and root you plan to expose. For each, write a one-sentence behavior spec and name the input/output types. Do this before touching handler code — if you can't describe the primitive in one sentence, the LLM won't know how to call it either.

## Package layout for embedded servers

When the MCP server is part of a larger binary, prefer this shape:

- A dedicated package (e.g. `internal/mcpserver`) that exports a constructor taking the host's dependencies and returning a configured `*mcp.Server`.
- The constructor registers tools, resources, and prompts. It does not own the transport or the lifecycle.
- The cobra subcommand (or equivalent) wires the constructor to a transport and calls `server.Run(ctx, transport)`.
- Handler implementations live in the same package as the constructor, as unexported functions or methods on a holder struct that owns the injected collaborators.

This keeps the MCP layer testable in isolation with in-memory transports, and keeps `main` uncluttered.

For standalone servers the shape collapses: `main` builds the server, registers features, and runs the transport. Still put handlers in their own file — `main.go` should not contain business logic.

## Registering tools

Use the generic `mcp.AddTool` unless there is a specific reason not to:

```go
mcp.AddTool(server, &mcp.Tool{
    Name: "search_logs",
    Description: "Search application logs by query string and time window.",
}, searchLogs)
```

The function `searchLogs` has this shape:

```go
type SearchLogsInput struct {
    Query string    `json:"query" jsonschema:"the search term, case-insensitive substring match"`
    Since time.Time `json:"since" jsonschema:"earliest log timestamp to include (RFC3339)"`
    Limit int       `json:"limit,omitempty" jsonschema:"max results, default 50"`
}

type SearchLogsOutput struct {
    Matches []LogEntry `json:"matches"`
    Total   int        `json:"total"`
}

func searchLogs(ctx context.Context, req *mcp.CallToolRequest, in SearchLogsInput) (*mcp.CallToolResult, SearchLogsOutput, error) {
    ...
}
```

Key points:

- **Every field needs a `json` tag.** The schema inference follows JSON tags.
- **Every field should have a `jsonschema` tag describing it.** This is what the LLM reads when deciding how to call the tool. Terse tags produce bad tool calls. Spend words here; it pays back immediately.
- **Return `(nil, out, nil)` on success.** The SDK builds the `CallToolResult` from the typed output. Only construct a `*CallToolResult` explicitly when you need custom content types (images, embedded resources), `IsError: true`, or to suppress the structured output.
- **Use `omitempty` judiciously.** A required field should not have `omitempty` — the schema should reflect that it's required.
- **Reject invalid input in the handler, not via schema gymnastics.** The schema can express "this is a number"; it cannot express "this must be a valid ISO-3166 country code". Validate in code and return a meaningful tool error (see § Error discipline in `review-checklist.md`).

See `primitives.md` § Tools for advanced cases: raw JSON input, custom schemas, custom marshaling, complex nested types.

## Registering resources

Two variants:

- **Static resource** — a known URI pointing at a specific piece of data. Register via `server.AddResource` with a handler that reads and returns contents.
- **Resource template** — a URI pattern (e.g. `users://{id}/profile`) that matches many resources. Register via `server.AddResourceTemplate`. The handler receives the matched URI and extracts parameters.

Resources support subscriptions (clients ask to be notified when a resource changes). If your data is genuinely dynamic and clients benefit from push updates, support subscriptions. If not, don't — subscriptions add state management burden.

List-changed notifications (`server.SendResourceListChanged` or equivalent on the current SDK version) are for when the set of resources itself changes, not when an individual resource's content changes.

See `primitives.md` § Resources for URI template syntax, content type selection, and subscription wiring.

## Registering prompts

Prompts are templated conversation starters. They take structured arguments and produce a list of messages. Register via `server.AddPrompt` with a handler that returns `*mcp.GetPromptResult` containing the messages.

Prompts are simpler than tools — they don't execute, they just shape input for the LLM. But they share the schema hygiene concern: argument descriptions matter because the host UI may render them.

See `primitives.md` § Prompts.

## Client-side features: sampling, elicitation, roots

These are features the *client* registers, typically used by servers during tool execution.

**Sampling** — the server asks the client to run an LLM inference. Useful when a tool needs the LLM to produce text as part of its work (e.g. "summarize this document"). Wire a `SamplingHandler` on the client. The handler receives a `CreateMessageParams` and returns a `CreateMessageResult`. Be explicit about model selection — clients may reject requests for models they can't serve.

**Elicitation** — the server asks the client to collect input from the user. Two modes:

- **Form mode** (default) — structured input with an optional JSON Schema. The client shows a form, the user fills it, the response goes back to the server. Form mode MUST NOT be used for credentials, tokens, or payment information.
- **URL mode** — the server returns a URL the client should direct the user to. Used for OAuth flows, API key entry, payment. The client's bearer token is not affected; URL mode elicitation is not MCP authorization.

If your server needs any sensitive input from the user, it must use URL mode. Mixing form mode with credentials is a spec violation and a security bug.

**Roots** — the client tells the server what filesystem paths it has access to (typically the user's open project). Servers that operate on files should consult the roots list. Clients register roots via `client.AddRoots` and may send roots-changed notifications.

Note: `ClientCapabilities.Roots` has a known design bug (issue #607) fixed by `ClientCapabilities.RootsV2`. On recent SDK versions, use `RootsV2`. On older versions, consult the SDK's CHANGELOG for the right field.

See `primitives.md` for full behavior details.

## Transport wiring

Once the server is built, wire it to a transport:

**Stdio** (default for embedded hosts that spawn subprocesses):

```go
err := server.Run(ctx, &mcp.StdioTransport{})
```

**Streamable HTTP** (for network-exposed servers):

```go
handler := mcp.NewStreamableHTTPHandler(func(r *http.Request) *mcp.Server {
    return server // or construct per-request for session isolation
}, nil)
http.ListenAndServe(":8080", handler)
```

The `getServer` factory allows per-request server construction when you need per-session state isolation. If a single `*mcp.Server` instance is safe to share across concurrent sessions, return it directly.

See `transports.md` for when to use which, and for event store / replay / stateless mode tradeoffs.

## Minimum viable client

```go
client := mcp.NewClient(&mcp.Implementation{Name: "my-client", Version: "v0.1.0"}, nil)
transport := &mcp.CommandTransport{Command: exec.Command("myserver")}
session, err := client.Connect(ctx, transport, nil)
// ...
defer session.Close()
```

Then call methods on `session`: `session.CallTool`, `session.ListTools`, `session.ReadResource`, etc.

For clients that *provide* features to the server (sampling, elicitation, roots), set the corresponding options in `*mcp.ClientOptions` before `NewClient`.

## Before declaring done

- Tests exist for every registered tool, resource, and prompt, using in-memory transports. See `testing.md`.
- The server handles context cancellation at every I/O point.
- Schemas have descriptive `jsonschema` tags on every field.
- Error discipline is correct (Go errors vs `IsError: true` tool results).
- The server advertises only capabilities it actually implements.
- For HTTP transports: graceful shutdown is wired (`http.Server.Shutdown` with the same `ctx` the server is running on).
