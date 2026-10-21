# Review checklist

Walk this checklist when auditing an MCP server or client. Each item has a failure mode — if the failure mode can occur in the code under review, it is a finding.

## 1. SDK and spec version

- [ ] `go.mod` pins a specific SDK version. **Failure mode:** floating minor/major can surface breaking changes on rebuild.
- [ ] Code matches the SDK version's API. **Failure mode:** using `RootsV2` on a pre-RootsV2 SDK or vice versa produces compile errors or runtime misbehavior.
- [ ] Features used match the MCP spec version the SDK targets. **Failure mode:** depending on URL-mode elicitation on a spec version that predates it.

## 2. Context discipline

- [ ] Every handler signature accepts `ctx context.Context` as first parameter. **Failure mode:** the SDK signature won't match and the code won't compile, but older hand-rolled wrappers sometimes drop it.
- [ ] Every I/O call inside a handler takes `ctx`. `http.NewRequestWithContext`, `sql.QueryContext`, `db.ExecContext`, etc. **Failure mode:** client cancellation leaves goroutines running and external resources pinned.
- [ ] Handlers with timers or sleeps use `select` with `<-ctx.Done()`, not bare `time.Sleep`. **Failure mode:** slow tool that ignored cancellation keeps running long after client gave up.
- [ ] Background goroutines spawned from handlers observe `ctx.Done()` or are bounded by an explicit shutdown. **Failure mode:** per-call goroutine leak.

## 3. Capabilities

- [ ] Capabilities are not hand-authored when the SDK infers them from registration. **Failure mode:** drift between advertised and implemented capabilities; usually the advertisement is stale after a refactor.
- [ ] `ClientCapabilities.Roots` vs `RootsV2` matches the SDK version. **Failure mode:** silently ignored capability on post-#607 SDKs.
- [ ] The client does not advertise `roots` if it has no roots to expose. **Failure mode:** servers query `ListRoots` and get empty lists rather than "not supported".
- [ ] Server code checks capability presence before calling client-side features. Before calling `Elicit`, verify the client advertised elicitation. Before `ListRoots`, check roots. **Failure mode:** runtime error when talking to clients that don't support the feature.

## 4. Tool schemas

- [ ] Every tool input struct field has a `json` tag. **Failure mode:** schema generation uses Go field names, which are capitalized and misleading for the LLM.
- [ ] Every tool input struct field has a `jsonschema` tag with a useful description. **Failure mode:** LLM calls the tool with wrong values or doesn't call it at all; symptoms look like "tool isn't working" when it's "tool isn't understood".
- [ ] Required fields do not have `,omitempty` on their JSON tag. **Failure mode:** schema marks them optional; LLM omits them; handler validation catches it but wasted a round trip.
- [ ] Enum-like fields use `jsonschema:"enum=..."` or equivalent. **Failure mode:** LLM passes freeform strings that don't match any valid value.
- [ ] Tool names conform to the spec's naming rules (recent spec versions enforce validation). **Failure mode:** server registration succeeds locally but some clients reject the tool.
- [ ] Tool descriptions explain *when* to use the tool, not just what it does. **Failure mode:** LLM calls it at wrong times or not at all.

## 5. Error discipline

The distinction matters and is frequently wrong:

- **Go `error` returned from a handler** → JSON-RPC error. The LLM typically doesn't see this in a useful way. Reserve for: protocol errors, panics you caught, dependency failures the LLM can't recover from.
- **`*CallToolResult{IsError: true, Content: [...]}` with `nil` Go error** → tool-level failure the LLM sees and can react to. Reserve for: expected failure modes like "no results", "invalid input", "resource not found", "operation would exceed quota".

Check:

- [ ] Expected-failure cases (empty results, invalid args, not-found) are returned as `IsError: true` tool results with explanatory text, not as Go errors. **Failure mode:** LLM sees an opaque "tool failed" and can't recover.
- [ ] Infrastructure failures (DB down, auth service unreachable) are returned as Go errors. **Failure mode:** LLM gets a "success" with error text in content and doesn't retry or escalate.
- [ ] Error messages returned to the LLM don't leak sensitive data (internal paths, credentials, PII from other users). **Failure mode:** data exposure through tool errors.
- [ ] Panics in handlers are recovered somewhere (SDK-level middleware or explicit `defer recover`). **Failure mode:** one bad tool call takes down the session.

## 6. Resources

- [ ] URI scheme is namespaced and unlikely to collide with other servers the host might aggregate. **Failure mode:** host gets two servers claiming `file://`; routing breaks.
- [ ] Resource templates use URI template syntax correctly and the handler matches what's declared. **Failure mode:** client constructs URIs the server can't parse.
- [ ] If subscriptions are supported, per-session goroutines watching resources are stopped when sessions close. **Failure mode:** goroutine leak per session.
- [ ] List-changed notifications fire when the resource set changes, not when resource content changes. **Failure mode:** clients refresh lists needlessly or miss actual list changes.
- [ ] Resource content types (`MIMEType`) are accurate. **Failure mode:** clients misinterpret bytes (e.g. treating JSON as plain text).

## 7. Prompts

- [ ] Prompt argument schemas describe each argument well enough that the host UI can render sensible controls. **Failure mode:** unusable prompts in hosts that generate UI from the schema.
- [ ] Prompt handlers return a coherent message list. **Failure mode:** malformed prompt results that some clients reject.

## 8. Sampling (server-to-client)

- [ ] Before calling sampling, the server verifies the client advertised the capability. **Failure mode:** runtime error on clients that don't support sampling.
- [ ] Sampling requests specify model preferences that real clients can actually serve. **Failure mode:** clients refuse the request.
- [ ] Sampling is scoped to a current request (per the spec, server-initiated requests must associate with an originating client request). **Failure mode:** spec violation; clients may reject the request.

## 9. Elicitation

- [ ] Sensitive data (passwords, API keys, tokens, payment info) uses URL mode, not form mode. **Failure mode:** spec violation and security hazard.
- [ ] Form-mode requests include a `requestedSchema` when structured input is expected. **Failure mode:** client can't validate or render a form.
- [ ] Server tolerates the user declining or canceling elicitation. **Failure mode:** tool hangs or errors unhelpfully when the user clicks cancel.

## 10. Roots

- [ ] Servers that operate on files consult the roots list rather than hardcoding paths or trusting tool arguments. **Failure mode:** server operates outside the user's intended scope; privacy and security concern.
- [ ] Servers handle zero-roots gracefully. **Failure mode:** a client that advertises roots but has registered none crashes the server's file operations.

## 11. Transport wiring

- [ ] Stdio servers use `server.Run(ctx, &mcp.StdioTransport{})` (or equivalent at the pinned version) and propagate cancellation from `ctx`. **Failure mode:** server doesn't shut down on signal.
- [ ] HTTP servers wrap `StreamableHTTPHandler` in an `http.Server` with `Shutdown` wiring and a reasonable timeout. **Failure mode:** ungraceful termination; in-flight requests dropped.
- [ ] `getServer` factory in `NewStreamableHTTPHandler` returns a per-session server if sessions carry state; returns a shared server only if the server is genuinely stateless. **Failure mode:** state bleeding between sessions, or resource waste from per-session construction when it wasn't needed.
- [ ] For SSE transports: the code is aware SSE is legacy (2024-11-05 spec version). New deployments should use Streamable HTTP. **Failure mode:** building on a transport that's being phased out.

## 12. Lifecycle

- [ ] The server shuts down cleanly on `ctx` cancellation or `SIGINT`. **Failure mode:** stuck processes, dropped connections, corrupted state.
- [ ] Sessions close their transports (`defer session.Close()`). **Failure mode:** connection leaks.
- [ ] For HTTP: `http.Server.Shutdown(ctx)` is called with a bounded timeout, not the same context being cancelled to trigger shutdown. **Failure mode:** shutdown returns immediately with no grace period.

## 13. Logging and observability

- [ ] Handlers log at sensible levels via the injected logger (not `log.Println` or `fmt.Print`). **Failure mode:** observability gap; logs don't route with the rest of the system's output.
- [ ] Sensitive data is not logged (tool arguments may contain user PII or credentials). **Failure mode:** data exposure.
- [ ] The SDK's `LoggingTransport` or equivalent wrapping is available for diagnosing protocol-level issues but is not enabled in production by default. **Failure mode:** logging noise and potential data exposure in prod.

## 14. Testing

- [ ] Tool, resource, and prompt handlers have tests using `mcp.NewInMemoryTransports()`. **Failure mode:** no regression coverage for the primary contract.
- [ ] Tests assert on tool schemas (at minimum, that they exist and have the expected fields). **Failure mode:** silent schema regressions that degrade LLM performance without breaking tests.
- [ ] Error paths are tested. **Failure mode:** untested `IsError: true` vs Go error paths are precisely where the discipline slips.
- [ ] Tests don't launch subprocesses for common cases. **Failure mode:** slow test suite, nondeterminism, flakes in CI.

See `testing.md` for the patterns.

## 15. Security surface

- [ ] Tool arguments that become filesystem paths, URLs, SQL, or shell commands are validated against a safe allowlist or a root-relative constraint. **Failure mode:** path traversal, SSRF, injection, RCE.
- [ ] Credentials are never accepted via form-mode elicitation. **Failure mode:** credential exposure through MCP transcripts.
- [ ] OAuth flows use the `auth` / `oauthex` packages rather than hand-rolled token handling. **Failure mode:** subtle OAuth bugs, token leakage.
- [ ] Error messages don't expose implementation details usable for attack (DB type, schema, internal endpoints). **Failure mode:** information disclosure.

## Final gate

Before signing off a review as complete, verify three things:

- Every finding has a named failure mode.
- Findings are ordered by blast radius.
- At least one non-finding is listed (what you checked that was correct).
