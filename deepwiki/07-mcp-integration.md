# Chapter 7 — MCP Integration

Expert: **Tools & MCP Expert**. Sources: `pkg/tools/mcp/` (client), `pkg/mcp/` (server), `docs/features/mcp-mode`, `docs/configuration/mcp*`.

Docker Agent speaks Model Context Protocol in **both directions**.

## A. As MCP client (`pkg/tools/mcp`)

### Server kinds

```yaml
mcps:
  local:
    type: stdio            # spawn a process
    command: npx -y @modelcontextprotocol/server-filesystem /tmp
  remote:
    type: http             # remote HTTP/SSE server ("remote-mcp" feature)
    url: https://m.example.com/mcp
  catalog:
    type: docker           # run an MCP server from Docker's catalog image
    ref: docker:duckduckgo
```

- **stdio/local**: process supervision via StartableToolSet; parent contexts are cancellable so killing a session kills children (no orphan processes — explicit fix noted in HEAD commit message).
- **Docker-catalog servers**: images from the MCP catalog; `mcpcatalog` builtin toolset lets agents *discover* catalog entries at runtime; missing binaries auto-installable via `pkg/toolinstall` (`version: owner/repo@ver`).

### OAuth for remote servers

Full client-side OAuth implementation: `oauth.go`, `oauth_login.go`, `pending_oauth.go`, token persistence in OS keyring (`keyringstore`), dynamic client registration, and **elicitation round-trips** — the agent pauses, emits `Authorization`/`ElicitationRequestEvent`, the TUI opens a browser flow, resume continues. Interactive streams only; non-interactive sessions fast-fail (#3200, Ch. 5).

### Tool bridging semantics

MCP tools convert 1:1 into `tools.Tool`: JSON-Schema parameters pass through; **MCP annotations (readOnly/destructive) drive permission & safety classification** (Ch. 11); output content blocks (text/image/audio/resource) map to `ToolCallResult` incl. `MediaContent`. Large tool counts stay cheap thanks to deferral/catalog search (Ch. 6).

### Gateway routing

Remote MCP traffic can route through Docker's gateway (`pkg/gateway`) for auth/observability; bypass flags exist for direct connections.

## B. As MCP server (`pkg/mcp/server.go`)

```sh
docker agent serve mcp ./agent.yaml            # stdio transport
docker agent serve mcp --transport http ...    # HTTP transport
```

- Each configured **agent becomes an MCP tool** (name/description/input schema derived from the agent); multi-agent configs become multi-tool servers.
- Consumable by Claude Desktop / Claude Code / any MCP host — your docker-agent team runs *inside another vendor's client*.
- Internally it drives the same `RunStream` and translates events to MCP progress/results; approval flows surface as elicitation back to the MCP client.

## C. Related protocol modes

- `serve a2a` — Google Agent-to-Agent protocol (Ch. 15).
- `serve acp` — Agent Client Protocol for editors.
- The MCP *client* and *server* share transport helpers but have separate lifecycles; both participate in OTel tracing.

## Practical notes from the docs

- Image-based (`docker:`) MCPs give reproducible, sandboxed tool servers — arguably the flagship Docker angle: **MCP servers distributed like container images**.
- Toolset restart-on-crash means flaky npx-based MCPs degrade gracefully (status shown, retried with backoff).

Next: [Chapter 8 — Multi-Agent Orchestration](08-multi-agent.md)
