# How to use this client config

[`mcp-client.json`](./mcp-client.json) is the entry to add to
**Antigravity**'s `mcp_config.json` (*… → MCP Servers → Manage MCP Servers →
View raw config*). It registers **this TP's** MCP server at
`http://localhost:8080/mcp`, over the **Streamable HTTP** transport, with
**Basic auth** (TP1's default-deny is **preserved** — exercise Step 1).

> Antigravity supports **only** the Streamable HTTP transport for remote
> servers (`serverUrl`). The legacy HTTP+SSE transport (`/sse`) is not
> supported: your server must use `spring.ai.mcp.server.protocol: STREAMABLE`
> (README §0). Reference: <https://antigravity.google/docs/mcp>

## Required change before pasting

**Replace the credentials** in the `Authorization` header with *your* TP1
`SELLER` account. The value is `Basic` + base64 of `email:password`:

```bash
printf '%s' 'alice@test.org:secret123' | base64     # YWxpY2VAdGVzdC5vcmc6c2VjcmV0MTIz
```

(`printf`, not `echo`: `echo` adds a newline that ends up in the encoded value.)

## Another client

Keep the **same three facts**, in that client's format:

- the **Streamable HTTP** transport (some clients call it `http`),
- the URL `http://localhost:8080/mcp`,
- the `Authorization: Basic …` header above.

> **Why there is an `Authorization` header at all:** the MCP route is **not**
> open to the world — it keeps TP1's default-deny. The header means "act as
> this SELLER", so the write tool still enforces TP1 ownership. The *why* is
> in [`exercise.md`](../exercise.md); this file is only the *how*.

## Where this config lives

In your local AI client's settings (not committed to any repo). The team
README (in `../springboot/`) keeps a screenshot of the transcript.
