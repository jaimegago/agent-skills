# Testing MCP servers and clients

Testing MCP code is not optional. The schemas, the error discipline, the cancellation behavior, and the capability advertisement are all easy to regress and painful to diagnose from the outside. In-memory transports make the tests fast and deterministic enough that there's no excuse.

## The three test layers

Organize MCP tests in three layers, not because the spec says so, but because each layer catches different bugs:

1. **Handler unit tests** — exercise handler functions directly with constructed requests. Fastest; catches logic bugs, schema-independent behavior.
2. **In-memory transport tests** — full protocol round-trip, client and server in the same process over `mcp.NewInMemoryTransports()`. Catches schema bugs, capability bugs, context propagation, error discipline. **This is the default layer — most tests go here.**
3. **Subprocess end-to-end tests** — launch the server as a subprocess via `CommandTransport`. Catches build / packaging / stdio framing issues. Keep these few and slow.

Do not skip layer 2. If you're writing any MCP tests at all, write the in-memory transport tests first. They're the highest-signal layer.

## In-memory transport — the standard setup

```go
func newTestSession(t *testing.T, build func(*mcp.Server)) (*mcp.ClientSession, *mcp.Server) {
    t.Helper()
    ctx := t.Context() // Go 1.24+; otherwise context.Background() with cleanup
    server := mcp.NewServer(&mcp.Implementation{Name: "test", Version: "v0"}, nil)
    build(server)
    client := mcp.NewClient(&mcp.Implementation{Name: "test-client", Version: "v0"}, nil)
    t1, t2 := mcp.NewInMemoryTransports()
    if _, err := server.Connect(ctx, t1, nil); err != nil {
        t.Fatalf("server connect: %v", err)
    }
    session, err := client.Connect(ctx, t2, nil)
    if err != nil {
        t.Fatalf("client connect: %v", err)
    }
    t.Cleanup(func() { _ = session.Close() })
    return session, server
}
```

(Verify method names and signatures against the pinned SDK — `Connect` and `NewInMemoryTransports` have been stable but argument shapes evolve.)

This helper is worth having project-wide. Put it in a `mcpservertest` package alongside the server code.

## Tool tests

For each tool, cover:

### 1. Happy path

```go
func TestSearchLogs_Match(t *testing.T) {
    session, _ := newTestSession(t, func(s *mcp.Server) {
        mcp.AddTool(s, &mcp.Tool{Name: "search_logs"}, buildSearchLogs(fakeStore))
    })
    res, err := session.CallTool(t.Context(), &mcp.CallToolParams{
        Name: "search_logs",
        Arguments: map[string]any{"query": "error", "limit": 10},
    })
    if err != nil { t.Fatal(err) }
    if res.IsError { t.Fatalf("unexpected tool error: %v", res.Content) }
    // assert structured content matches expectation
}
```

### 2. Invalid input → tool error (not Go error)

```go
res, err := session.CallTool(ctx, &mcp.CallToolParams{
    Name: "search_logs",
    Arguments: map[string]any{"query": "", "limit": -1},
})
if err != nil { t.Fatalf("want tool error, got protocol error: %v", err) }
if !res.IsError { t.Fatal("want IsError=true, got success") }
```

This is the canonical test for error discipline — if someone changes the handler to return a Go error for bad input, this test fails.

### 3. Cancellation

```go
ctx, cancel := context.WithCancel(t.Context())
done := make(chan struct{})
go func() {
    _, _ = session.CallTool(ctx, &mcp.CallToolParams{Name: "slow_tool"})
    close(done)
}()
time.Sleep(50 * time.Millisecond) // give the call time to start
cancel()
select {
case <-done:
case <-time.After(time.Second):
    t.Fatal("tool did not return after cancellation")
}
```

If the handler ignores `ctx`, this test hangs and then fails — which is exactly the signal you want.

### 4. Schema shape

```go
tools, err := session.ListTools(t.Context(), nil)
if err != nil { t.Fatal(err) }
// find the tool, assert InputSchema.Properties contains expected fields,
// assert required fields are in InputSchema.Required,
// assert descriptions are non-empty.
```

This is boring but catches silent regressions — someone renames a struct field, schema changes, LLM stops calling the tool correctly. Without this test, nothing fails and the tool quietly degrades.

Consider golden-file testing for schemas: serialize the schema to JSON, check it into `testdata/`, assert equality. Updates become deliberate (`go test -update`) rather than accidental.

## Resource tests

Cover:

- **Static resource read** — `session.ReadResource(ctx, &mcp.ReadResourceParams{URI: "docs://readme"})` returns expected content and MIME type.
- **Template resolution** — `session.ReadResource(ctx, params{URI: "users://42/profile"})` routes to the template handler and extracts the parameter correctly.
- **Missing resource** — reading an unknown URI returns an appropriate error.
- **Subscriptions** (if supported) — subscribe, trigger a change, assert the client receives the notification, close the session, assert the backing watcher stopped.

Subscription teardown is hard to test directly but important. One approach: inject a watcher factory that returns instrumented watchers, then assert the instrumentation shows `Stop()` was called.

## Prompt tests

Cover:

- **Get with required args** — returns expected message list.
- **Get with missing required args** — returns appropriate error.
- **Argument description** — list prompts, assert descriptions are non-empty.

## Sampling tests (server calling client)

The server-to-client direction requires a sampling handler on the client side:

```go
client := mcp.NewClient(&mcp.Implementation{Name: "test", Version: "v0"}, &mcp.ClientOptions{
    SamplingHandler: func(ctx context.Context, req *mcp.CreateMessageRequest) (*mcp.CreateMessageResult, error) {
        return &mcp.CreateMessageResult{
            Role: "assistant",
            Content: &mcp.TextContent{Text: "stubbed response"},
            Model: "stub-model",
        }, nil
    },
})
```

Then register a tool on the server that calls `serverSession.CreateMessage(...)` and assert the result flows through correctly.

Test the failure modes too:

- Client with no sampling handler — server calls `CreateMessage`, gets an error, handles it gracefully.
- Sampling handler returns an error — server propagates it appropriately.

## Elicitation tests

Same pattern as sampling — put a handler on the client, exercise a tool that elicits.

Critical cases to cover:

- User fills the form → tool gets the values.
- User declines → tool handles the decline without hanging or panicking.
- Client lacks elicitation capability → server-side check works; server doesn't attempt to elicit.
- URL-mode elicitation → the returned URL is what the server expects, and the tool flow completes correctly.

## Roots tests

```go
client.AddRoots(&mcp.Root{URI: "file:///tmp/test-workspace", Name: "workspace"})
// ...
roots, err := serverSession.ListRoots(ctx, nil)
```

Test:

- Server correctly retrieves roots.
- Server degrades gracefully when client has no roots.
- If server supports roots-changed notifications, trigger `client.AddRoots` / `RemoveRoots` and assert the server observed the change.

## Capability tests

For each capability the server uses:

- Connect a client that advertises it → server uses it normally.
- Connect a client that doesn't advertise it → server's capability check prevents the call; tool either errors clearly or degrades.

This catches the capability-mismatch bug class — server tries to use a feature the client doesn't support.

## Subprocess tests — use sparingly

A handful of subprocess tests catch build/packaging bugs:

```go
transport := &mcp.CommandTransport{Command: exec.Command("./myserver")}
session, err := client.Connect(ctx, transport, nil)
```

Good uses:

- "Does the binary actually start?"
- "Does stdio framing work?"
- "Does the published Docker image respond to a basic tool call?"

Bad uses:

- Anything you can do with in-memory transports. It'll be slower and flakier.

Tag these tests appropriately (`//go:build e2e`) and run them separately from the fast suite.

## Test data hygiene

- Every test should use its own server instance. Don't share servers across tests — state leaks.
- Use `t.Cleanup` for session close, not `defer`. Cleanups run in reverse registration order and are reliable across subtests.
- Prefer `t.Context()` (Go 1.24+) over `context.Background()` — it's cancelled on test completion.
- Avoid sleeping more than tens of milliseconds. If you need to wait for something, poll or use channels.

## What this looks like when done right

A well-tested MCP server has:

- One in-memory-transport test file per primitive (tools, resources, prompts, etc.).
- Schema golden files in `testdata/` regenerated only by an explicit flag.
- A handful of subprocess tests behind a build tag.
- A capability-mismatch test or two.
- Cancellation tests for any tool that does I/O.

Total: usually a few hundred lines of tests per tool file. That's the right order of magnitude.
