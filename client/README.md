# How to use this client config

[`mcp-client.json`](./mcp-client.json) is the entry to add to
**Antigravity**'s `mcp_config.json` (*… → MCP Servers → Manage MCP Servers →
View raw config*). It registers **this TP's** MCP server at
`http://localhost:8080/mcp` (MCP revision 2026-07-28 over HTTP), with
**Basic auth** (TP1's default-deny is **preserved** — exercise Step 1).

> Antigravity (app and CLI 1.5) speaks the **2026-07-28** revision of MCP
> (`server/discover`, no session): your server must use
> `spring.ai.mcp.server.protocol: MODERN` (Spring AI 2.1, README §0). A server
> on the older session-based `STREAMABLE` transport stays at
> `initializing...` in the client. Reference: <https://antigravity.google/docs/mcp>

**Antigravity CLI** (`agy`): one command instead of the JSON file —

```bash
agy mcp add --header "Authorization: Basic $(printf '%s' 'alice@test.org:secret123' | base64)" \
  listings http://localhost:8080/mcp
agy mcp list                 # listings  http  enabled
agy                          # then /mcp: listings, your tools
```

## Required change before pasting

**Replace the credentials** in the `Authorization` header with *your* TP1
`SELLER` account. The value is `Basic` + base64 of `email:password`:

```bash
printf '%s' 'alice@test.org:secret123' | base64     # YWxpY2VAdGVzdC5vcmc6c2VjcmV0MTIz
```

(`printf`, not `echo`: `echo` adds a newline that ends up in the encoded value.)

## Another client

Keep the **same three facts**, in that client's format:

- an HTTP server entry (some clients call the type `http`) speaking MCP 2026-07-28,
- the URL `http://localhost:8080/mcp`,
- the `Authorization: Basic …` header above.

> **Why there is an `Authorization` header at all:** the MCP route is **not**
> open to the world — it keeps TP1's default-deny. The header means "act as
> this SELLER", so the write tool still enforces TP1 ownership. The *why* is
> in [`exercise.md`](../exercise.md); this file is only the *how*.

## Where this config lives

In your local AI client's settings (not committed to any repo). The team
README (in `../springboot/`) keeps a screenshot of the transcript.
