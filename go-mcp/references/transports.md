# Transports — choosing and using them

The Go SDK ships four transports. Each has a different shape of failure, a different deployment story, and a different testing story. Picking the right one before writing code saves rework.

## Quick picker

| Situation | Use |
|---|---|
| Host spawns the server as a subprocess (Claude Desktop, most IDE integrations) | Stdio |
| Server runs as a network service with one-to-many client connections | Streamable HTTP |
| Maintaining an existing deployment that already runs SSE | SSE (legacy) |
| Tests | In-memory |
| Custom protocol over your own `io.ReadWriteCloser` | IOTransport |

Default greenfield: stdio for locally-spawned servers, Streamable HTTP for networked ones. Don't invent reasons to use SSE for new code — it is the 2024-11-05 spec's transport and is being phased out in favor of Streamable HTTP.

## Stdio

### When

- The server will be spawned as a child process by the host.
- Single client per server process.
- No network exposure needed.

This is the most common shape and the default assumption for most MCP hosts.

### Server side

```go
err := server.Run(ctx, &mcp.StdioTransport{})
```

`Run` blocks until the transport closes or the context is cancelled. On context cancellation, the server shuts down gracefully.

### Client side

```go
transport := &mcp.CommandTransport{Command: exec.Command("myserver")}
session, err := client.Connect(ctx, transport, nil)
```

`CommandTransport` manages the subprocess lifecycle. When the session closes, the subprocess is terminated.

### Framing

Stdio uses newline-delimited JSON-RPC (RawFramer in the SDK). Every log line the server writes to stderr is fine; anything it writes to stdout that isn't a JSON-RPC message will desynchronize the transport. This is the single most common stdio bug.

Discipline:

- Route all logging to stderr. Use `slog.New(slog.NewJSONHandler(os.Stderr, ...))`.
- Audit for stray `fmt.Println`, `log.Println` (default goes to stderr, but verify the `log` package's output destination).
- If the server embeds a third-party library that writes to stdout, reroute or silence it.

### Signal handling

In standalone servers, `main` should set up signal handling so the context passed to `server.Run` cancels on SIGINT/SIGTERM:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
if err := server.Run(ctx, &mcp.StdioTransport{}); err != nil {
    // handle
}
```

Embedded servers inherit the parent command's context and don't set this up themselves.

## Streamable HTTP

### When

- Server runs as a network-exposed service.
- Multiple clients may connect concurrently.
- You need features like session resume, event replay, or stateless request handling.

### Server side

```go
handler := mcp.NewStreamableHTTPHandler(
    func(r *http.Request) *mcp.Server { return server },
    nil, // *mcp.StreamableHTTPOptions
)

srv := &http.Server{Addr: ":8080", Handler: handler}
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Error("listen", "err", err)
    }
}()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)
```

### Per-session servers vs shared server

The `getServer` factory is the key customization point.

- **Return the same `*mcp.Server` every call** when the server is fully stateless across sessions (or state is safe to share concurrently). Cheapest option.
- **Return a per-request server** when each session needs isolated state — for example, when tools maintain a session-scoped cache, when auth context varies per request, when resource subscriptions need per-session goroutines.

Construct the per-session server from injected collaborators, not from package-level state. Otherwise you have "per-session" in name only.

### Authentication

Streamable HTTP typically uses bearer tokens. The `auth` subpackage supports this; add a receiving middleware that extracts and validates the token:

```go
server.AddReceivingMiddleware(authMiddleware(verifier))
```

For full OAuth 2.1 with Protected Resource Metadata, use the `auth` and `oauthex` packages together. This is beyond this skill's scope — consult the SDK's OAuth examples and the MCP security spec.

### Event store and replay

Streamable HTTP supports event store / replay for session resumability — a client that disconnects and reconnects can pick up where it left off. This requires a persistent event store; the SDK provides an interface for plugging one in.

Use this when:

- Sessions are long-lived.
- Network interruptions are expected.
- You can afford the persistence cost.

Skip it for simple request/response workloads.

### Stateless mode

For fully-stateless deployments (each request treated as an independent interaction), Streamable HTTP has a stateless mode. Useful when scaling horizontally behind a load balancer without session affinity.

Stateless mode disables features that require session continuity — subscriptions, progress notifications tied to prior requests, etc. Check the SDK docs for the exact constraints.

## SSE (legacy)

### When

- Never for new code.
- Only if maintaining an existing deployment that has clients hard-coded to the SSE endpoint from the 2024-11-05 spec.

SSE in MCP uses a hanging GET for server-to-client events plus a POST endpoint for client-to-server messages. It has been superseded by Streamable HTTP.

### Server side

```go
handler := mcp.NewSSEHandler(func(r *http.Request) *mcp.Server { return server })
http.Handle("/mcp", handler)
```

### Client side

```go
transport := mcp.NewSSEClientTransport("https://host/mcp", nil)
session, err := client.Connect(ctx, transport, nil)
```

### Migration

If you're on SSE and considering migration: Streamable HTTP is strictly more capable. Plan the cutover, operate both endpoints during a transition period, deprecate the SSE endpoint once clients have moved. Don't rush it, but don't let it linger.

## In-memory

Only for tests. See `testing.md`.

```go
t1, t2 := mcp.NewInMemoryTransports()
server.Connect(ctx, t1, nil)
session, _ := client.Connect(ctx, t2, nil)
```

Fast, deterministic, no framing or serialization overhead. If a production scenario would be well-served by an in-memory transport, the question is probably "should this be an MCP server at all", not "how do I use in-memory in production".

## IOTransport

For custom underlying `io.ReadCloser` / `io.WriteCloser` pairs — e.g. wrapping a WebSocket, a Unix domain socket, or a message queue. Rare. If reaching for this, verify the spec allows the transport you're implementing; most custom transports are cargo-culted and shouldn't exist.

## Transport-level review items

See `review-checklist.md` § 11 for the audit checklist. The high points:

- Stdio servers: nothing writes to stdout except JSON-RPC.
- HTTP servers: graceful shutdown wired with a bounded timeout.
- HTTP `getServer` factory: per-session vs shared is a deliberate decision, not a default.
- SSE deployments: migration to Streamable HTTP is on the roadmap.
- Custom transports: scrutinized for spec compliance.
