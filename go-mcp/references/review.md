# Review: auditing existing MCP code

Use this reference when reviewing, auditing, or improving existing Go MCP code. For the item-by-item audit, see `review-checklist.md`. For creating new code, use `greenfield.md`.

## Orient before judging

Before filing any finding, establish the baseline:

1. **SDK version** — read `go.mod`. The SDK API has changed across releases (capability configuration, roots V2, elicitation modes, tool name validation). A pattern that looks wrong on a recent release may be correct for an older one.

2. **MCP spec version the SDK targets** — the SDK's README has the compatibility matrix. A server targeting spec 2024-11-05 legitimately doesn't have URL-mode elicitation or tool icons. Don't flag the absence of features that don't exist in the targeted version.

3. **Server shape** — standalone or embedded? The review bar differs. Embedded servers must integrate with the host's lifecycle, logging, and DI; standalone servers must own those concerns themselves.

4. **Transport** — stdio, Streamable HTTP, SSE, or custom? Each has different failure modes and different review priorities (see `transports.md`).

5. **Primitive inventory** — which tools, resources, prompts are registered? Which client-side features (sampling, elicitation, roots) are wired? This tells you where to look.

Record these five things before starting the line-by-line review. If any of them is unclear, ask — guessing wastes the user's time.

## How to structure findings

For every finding, include three things:

- **What** — the specific code location and the specific behavior.
- **Why it matters** — the failure mode. Under what condition does this cause a problem? For whom? At what severity? A finding without a failure mode is a style preference and should be labeled as one.
- **What to do** — a concrete change, or a direction when the right change depends on context you don't have.

Findings that only name a violation ("this doesn't follow the spec") without tying it to runtime consequences are easy to dismiss and frequently get dismissed. Findings that lead with blast radius ("if the client cancels during this handler, the goroutine leaks and the backing DB connection stays pinned until the server restarts") get acted on.

## Prioritize by blast radius

Rank findings:

1. **Spec violations that can cause client-visible failure** — e.g. form-mode elicitation for credentials, advertising a capability that isn't implemented, ignoring context cancellation in a long-running tool.
2. **Resource and goroutine leaks** — handlers that don't respect cancellation, HTTP servers without shutdown wiring, subscription handlers that never stop their backing goroutines.
3. **Correctness bugs in tool behavior** — incorrect `IsError` vs Go error usage, silently swallowing errors, race conditions on shared state between handlers.
4. **Schema quality** — missing or unhelpful `jsonschema` descriptions, wrong required/optional markers, unconstrained strings where enums apply. These are correctness issues for the LLM even when they're cosmetic for humans.
5. **Style and structure** — package layout, naming, idiom. Defer to `go-backend` for this tier; don't re-litigate here.

## Common patterns worth flagging

These come up repeatedly. None are automatic fails, but all warrant inspection:

- **Handlers that don't take `ctx` seriously** — calling `http.Get` instead of `http.NewRequestWithContext`, running `time.Sleep` instead of `select { case <-time.After: ; case <-ctx.Done(): }`, doing DB queries without `ctx`. Any of these means client cancellation doesn't propagate.

- **Pre-capability-inference struct authoring** — hand-written `ServerCapabilities` or `ClientCapabilities` structs on current SDK versions. The SDK infers capabilities from what you register. Hand-authored capability structs are either redundant or wrong (advertising something you don't implement).

- **Global state closures** — tool handlers that reach into package-level variables for DB handles, config, loggers. Works but makes testing harder and makes per-session isolation impossible. Prefer constructor injection.

- **Subprocess testing as the default** — tests that launch the server as a subprocess via `CommandTransport` when they could use `NewInMemoryTransports`. In-memory is faster, deterministic, and lets you assert on the server's internal state. Reserve subprocess tests for end-to-end sanity.

- **Missing `jsonschema` descriptions on tool input/output fields** — a field with a `json` tag but no `jsonschema` tag is invisible to the LLM beyond its name and type. If a tool is being called wrong, this is the first thing to fix.

- **Conflating tool errors with Go errors** — returning a Go `error` for conditions the LLM should see and recover from (e.g. "no matching records found"), or returning `IsError: true` for conditions that are actually protocol failures (e.g. database connection lost). See `review-checklist.md` § Error discipline.

- **HTTP servers without shutdown wiring** — `http.ListenAndServe` with no way to call `Shutdown`. On SIGINT the server will drop in-flight requests and fail to close sessions cleanly.

- **Resource subscriptions without teardown** — when a client subscribes to a resource, the server often spawns a goroutine watching the backing store. When the session closes, that goroutine must stop. Missing teardown is a leak that accumulates per-session.

- **Elicitation used for credentials** — form-mode elicitation requesting API keys, passwords, tokens, or payment info is a spec violation in recent versions and a security hazard in all versions. URL mode exists for this.

- **Capability mismatch between client and server** — a server that calls `ListRoots` on a client that doesn't advertise roots, or `Elicit` on a client that doesn't advertise elicitation, will fail at runtime. Check that server logic degrades gracefully when a capability is absent.

## Review output format

When producing a review, structure it as:

- **Summary** — one paragraph: SDK version, spec version, server shape, transport, primitive inventory. Then the top two or three highest-priority findings.
- **Findings** — grouped by priority tier (blast radius order above). Each with what / why / what-to-do.
- **Non-findings** — brief list of things you verified are correct. This calibrates the reader's trust in the review.
- **Open questions** — things you couldn't resolve from the code alone. Ask them.

Keep findings precise. A review with twelve specific high-signal findings beats a review with forty mixed-signal ones.
