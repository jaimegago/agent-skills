# MCP primitives — behavior reference

This file describes the behavior contract of each MCP primitive and client feature as implemented by the official Go SDK. For when to use each, see `greenfield.md`. For common audit pitfalls, see `review-checklist.md`. Verify the specific API surface against `pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp` at the pinned SDK version — details below have been stable in recent v1.x releases but are not frozen.

## Tools

### Registration

Two registration paths:

**Generic (preferred)** — `mcp.AddTool[In, Out any](server, tool, handler)`. The SDK infers input and output schemas from the type parameters. `In` and `Out` must be map or struct types so the schema is `type: object`; if `In` is `any`, the schema is an empty object; if `Out` is `any`, the output schema is omitted.

**Low-level** — `server.AddTool(tool, handler)` where the tool carries explicit `InputSchema` / `OutputSchema` and the handler takes `json.RawMessage`. Use this when:

- The schema can't be expressed as a Go type (e.g. heavily dynamic).
- You want to hand-author the schema for precision the inferrer can't match.
- You're migrating existing code from another SDK.

### Handler signature

The generic path requires:

```
func(ctx context.Context, req *mcp.CallToolRequest, in In) (*mcp.CallToolResult, Out, error)
```

- `ctx` — cancellable. Every I/O call inside must honor it.
- `req` — access to session (`req.Session()`), raw arguments, metadata. Most handlers don't touch this; typed `in` is sufficient.
- `in` — unmarshaled from the client's arguments according to the inferred schema.
- Return `(nil, out, nil)` for the common success case; the SDK synthesizes the `CallToolResult`.
- Return `(&CallToolResult{...}, zeroOut, nil)` to take manual control (custom content, `IsError: true`).
- Return `(nil, zeroOut, err)` for protocol-level failures.

### Schema inference

Internally the SDK uses `github.com/google/jsonschema-go`. Key behaviors:

- `json:"field"` tag controls the JSON field name.
- `jsonschema:"..."` tag provides the description and constraints. Multiple values can be combined.
- Optional fields use `,omitempty` on the JSON tag and become non-required in the schema.
- Pointer types indicate optionality in the schema.
- Enums: `jsonschema:"enum=a,enum=b,enum=c"` (verify exact syntax against the jsonschema-go docs for the pinned version).

### Structured vs unstructured results

Tools have two result pathways:

- **Structured** — `Out` type is non-`any`. Output schema is published. Clients get both a JSON structured result and a synthesized text representation.
- **Unstructured** — `Out` is `any` or the handler returns a manual `CallToolResult`. Only content blocks (text, images, embedded resources) are returned.

Prefer structured for anything programmatic. Use unstructured when the tool's primary output is unstructured content (e.g. rendered images, streams of text).

### Tool errors vs Go errors

- `IsError: true` on `CallToolResult` — the tool ran and produced a user-visible failure. LLM sees it and can react.
- Go `error` return — protocol-level failure. LLM often can't meaningfully recover; the client may surface it as a red error rather than tool output.

Default to `IsError: true` for anything the LLM might reasonably retry or work around. Use Go errors for "something is broken in the infrastructure".

### Cancellation

When the client cancels a tool call, the SDK cancels the handler's `ctx`. The handler must:

1. Stop doing work.
2. Return promptly.
3. Clean up any resources it acquired (closing connections, releasing locks).

Tests should cover this — spawn a slow tool, cancel client-side, assert the handler observed cancellation.

## Resources

### Static resources

```
server.AddResource(&mcp.Resource{
    URI: "docs://readme",
    Name: "Project README",
    MIMEType: "text/markdown",
}, handler)
```

Handler signature (verify against pinned SDK):

```
func(ctx context.Context, req *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error)
```

Return contents via `ReadResourceResult.Contents` — a slice of `ResourceContents` (text or blob).

### Resource templates

For parameterized URIs:

```
server.AddResourceTemplate(&mcp.ResourceTemplate{
    URITemplate: "users://{id}/profile",
    Name: "User Profile",
    MIMEType: "application/json",
}, handler)
```

The handler receives the matched URI and extracts `{id}` from it. Use a URI template library matching the spec's RFC 6570 subset; don't hand-parse.

### Subscriptions

Clients can subscribe to resources they care about. When a resource's *content* changes, notify subscribers via the session's notification method (exact API varies by SDK version; check `ServerSession` methods).

Key discipline:

- Track subscriptions per session, not globally. When a session closes, drop its subscriptions.
- The goroutine watching the backing store for changes must observe session-close or an explicit teardown signal. Orphaned watchers are a leak.
- Only support subscriptions if your data actually benefits from push. Polling-based subscriptions that fire on a timer are usually worse than letting the client re-read on demand.

### List-changed notifications

When the *set* of resources changes (not the content of one), notify via the server's list-changed mechanism. Typical triggers: files created/deleted in a watched directory, records added/removed in a backing store.

## Prompts

### Registration

```
server.AddPrompt(&mcp.Prompt{
    Name: "summarize",
    Description: "Summarize a document",
    Arguments: []mcp.PromptArgument{
        {Name: "document", Description: "Text to summarize", Required: true},
        {Name: "style", Description: "brief | detailed", Required: false},
    },
}, handler)
```

### Handler

Returns `*mcp.GetPromptResult` containing a list of messages. Each message has a role (`user` or `assistant`) and content. The host typically presents these to the user as a starting point, or sends them to the LLM directly depending on the UX.

### Descriptions matter

Prompt argument descriptions frequently surface in host UI. Write them as if a user will read them in a form field, because they probably will.

## Sampling (client → LLM, triggered by server)

### What it is

The server asks the connected client to perform an LLM inference on the server's behalf. Useful when a tool needs the LLM to do something textual as part of its work — summarize a long document, extract structured data, generate a description.

### Client side

Client registers a `SamplingHandler` in `ClientOptions`:

```
func(ctx context.Context, req *mcp.CreateMessageRequest) (*mcp.CreateMessageResult, error)
```

The handler decides whether and how to fulfill the request. It may reject it, modify model selection, or proxy to whatever LLM provider the host uses.

Setting a `SamplingHandler` automatically advertises the `sampling` capability (per recent SDK versions that infer capabilities from registration).

### Server side

Server calls `serverSession.CreateMessage(ctx, params)` (or equivalent) from within a tool handler. The request must be associated with an originating client request — sampling is not for out-of-band server activity.

### Gotchas

- Model preferences should be reasonable. Clients can reject requests for models they can't serve.
- Always check the client advertised sampling before calling it.
- Expect latency. Sampling round-trips an entire LLM inference through the client's provider.

## Elicitation (server asks user for input via client)

### Form mode

Server requests structured input; client shows a form; user fills it; response returns to server.

Request includes `requestedSchema` (a restricted JSON Schema subset). Server receives the filled values in `ElicitResult`.

**Form mode MUST NOT be used for sensitive data** — passwords, API keys, tokens, payment credentials. The spec categorically forbids it in recent versions; clients may reject such requests.

### URL mode

Server returns a URL; client navigates the user to it; user interacts with an external system (OAuth provider, payment portal, credential form); server waits out-of-band for a result it fetches through another channel.

URL mode is the correct path for:

- OAuth flows.
- API key or token entry.
- Payment processing.
- Any interaction where sensitive data must not pass through the MCP transport.

URL mode is **not** the same as MCP authorization. The client's bearer token for the MCP server stays unchanged. URL mode elicitation is about the server gathering something *from the user* that doesn't involve granting MCP access.

### Capability

Setting an `ElicitationHandler` on the client advertises elicitation. Server must check the capability before calling `Elicit`.

### Handler shape (client)

```
func(ctx context.Context, req *mcp.ElicitRequest) (*mcp.ElicitResult, error)
```

The handler may:

- Return a filled result.
- Return a result indicating the user declined.
- Return an error for protocol-level issues.

Servers must handle all three outcomes gracefully.

## Roots (client-side filesystem scopes)

### What it is

The client tells the server what paths it has access to — usually the user's open workspace, project, or selected folders. Servers that operate on files should consult this list rather than hardcode paths or trust arguments.

### Client side

```
client.AddRoots(&mcp.Root{URI: "file:///home/user/project", Name: "project"})
```

Advertising roots is controlled by `ClientOptions.Capabilities.RootsV2` on current SDK versions. (Older versions used `ClientCapabilities.Roots`; there was a design bug — issue #607 — that led to `RootsV2`.)

### Server side

Server calls `serverSession.ListRoots(ctx, nil)` to retrieve current roots. If the client supports `roots.listChanged`, the server also receives notifications on root changes.

### Discipline

- Servers doing file operations should treat roots as the authoritative scope boundary.
- Handle empty root lists gracefully — either error clearly or degrade to a sensible no-scope mode.
- Cache root lookups per session but refresh on list-changed notifications.

## Logging

The SDK supports structured logging via `slog`. Levels map to MCP log levels (debug, info, notice, warning, error, critical, alert, emergency) — see the `mcp.Level*` constants.

Servers can send log messages to the client via `serverSession.Log(ctx, params)`. Clients register a logging handler in `ClientOptions` to receive them.

Production discipline:

- Use the injected logger, not package-level loggers.
- Don't log tool arguments at info level — they may contain user data.
- Use debug level for protocol-level tracing (request/response); gate it behind an env var.

## Progress

Long-running operations can emit progress notifications via `serverSession.NotifyProgress(ctx, params)` if the client included a progress token in the original request (`req.Params.Meta.ProgressToken`).

Discipline:

- Check for the progress token before emitting notifications. If absent, the client didn't ask for progress; emitting anyway is wasted work.
- Throttle notifications. One per 100ms is usually enough; per-byte emits overwhelm clients.

## Middleware

The SDK supports middleware on both sending and receiving paths for both client and server:

- `AddSendingMiddleware(middleware...)` — wraps outgoing requests (e.g., progress tokens, tracing, metrics).
- `AddReceivingMiddleware(middleware...)` — wraps incoming requests (e.g., auth, logging, rate limiting).

Applied right-to-left: `AddMiddleware(m1, m2, m3)` results in `m1(m2(m3(handler)))`.

Common middleware uses:

- **Auth** — receiving middleware that extracts and validates a bearer token before dispatch.
- **Request logging** — receiving middleware that logs method and duration.
- **Panic recovery** — receiving middleware that converts handler panics into Go errors instead of taking down the session.
- **Tracing** — sending middleware that injects trace context, receiving middleware that extracts it.
