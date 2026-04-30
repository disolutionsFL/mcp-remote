# disolutionsFL/mcp-remote — fork notes

This fork tracks `geelen/mcp-remote` and adds one targeted patch that we needed
for our local infrastructure. We are NOT diverging from upstream as a project —
we are just patching the one bug below until upstream lands its own fix.

## Patch: reinit-on-reconnect

**Branch:** `main` (patch is on the default branch so `npx github:disolutionsFL/mcp-remote` works without specifying a branch).

**Problem:** When the remote MCP server (e.g. `mcp-proxy` running on a different
host) restarts, the SDK's `EventSource` auto-reconnects after ~3s but neither
`SSEClientTransport` nor the wrapping `Client` learn that the server-side
session was wiped. The `Client` keeps `initialized = true`, and every
subsequent `tools/call` is rejected by the new server instance with
`-32602` / `Received request before initialization was complete`. From the
host's perspective, the MCP tool just stops working until the host process
is restarted (which kills + respawns mcp-remote with a fresh `initialize`
handshake).

**Fix:** In `mcpProxy()`, intercept incoming server messages. If we see a
`-32602` error whose message contains `"initialization was complete"`,
`"not initialized"`, or `"invalid session"`, forward the error once so the
host sees what happened, then `process.exit(2)`. Hosts that spawn mcp-remote
as a long-lived stdio child (Claude Desktop, OpenClaw, etc.) treat the exit
as "tool failed" and respawn the child on the next tool call. The respawned
child completes `initialize` cleanly against the new server, and the host's
natural tool-call retry covers the user-visible turn.

This is a "die-and-let-the-supervisor-respawn" recovery, not an in-place
reconnect. A proper in-place fix would require recreating the transport,
re-running `initialize`, and replaying pending requests — much more code, and
arguably belongs upstream in `SSEClientTransport` itself.

**Code:** `src/lib/utils.ts`, inside `mcpProxy`'s `transportToServer.onmessage`
handler. Marked with the comment `disolutionsFL fork patch (reinit-on-reconnect)`.

**Build hook:** the `prepare` script in `package.json` runs `tsup` on install
so `npx -y github:disolutionsFL/mcp-remote` auto-builds the `dist/` after
cloning (devDependencies are pulled by npm during git installs).

## Trying it

```jsonc
// e.g. in openclaw.json mcp.servers entry
{
  "command": "npx",
  "args": ["-y", "github:disolutionsFL/mcp-remote", "<sse-url>", "--allow-http"]
}
```

## Rolling back

If upstream lands its own fix or you want vanilla behavior, revert to:

```jsonc
{
  "command": "npx",
  "args": ["-y", "mcp-remote", "<sse-url>", "--allow-http"]
}
```
