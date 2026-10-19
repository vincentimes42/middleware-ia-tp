# How to use this client config

This is the [client-side config](./mcp-client.json) to paste into your
AI client — **Antigravity** (recommended; Google's tool, signs in with a
Google account) or any other MCP-over-SSE client (Claude Desktop, …). It
registers **this TP's** MCP server at `http://localhost:8080/sse`,
with **Basic auth** (TP1 default-deny is **preserved** — Step 1.1-1.2).

> **Note:** Antigravity ships its own MCP registration UI — if it accepts
> the JSON directly (name/url/headers), paste the block from
> [`mcp-client.json`](./mcp-client.json) as-is. If it uses a different field
> layout, keep the **same three facts** listed in section 2 below.

## Required changes before pasting

1. **Replace the credentials** in the `Authorization` header with *your*
   TP1 `SELLER` account:
   - current value: `alice@test:secret123` (the `alice@test` TP1 scaffold
     account)
   - base64 of `yourEmail:yourPass` → paste under `"Authorization"`.
   The header is `Basic <base64>`, e.g.
   `Basic $(echo -n "alice@test:secret123" | base64)` =
   `Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=`.

2. **If your client uses a different JSON shape** (Claude Desktop writes
   `claude_desktop_config.json`, e.g.), keep these three facts:
   - `type: "sse"`,
   - `url: "http://localhost:8080/sse"`,
   - the `Authorization` header above.

> **Why there is an `Authorization` header at all:** the MCP routes are **not**
> opened to the world — they keep TP1's default-deny (Step 1.2). The header means
> "act as this SELLER", so the write tool still enforces TP1 ownership. Read the
> *why* in [`exercise.md` Step 1](../exercise.md); this file is only the *how*.

## Where this config lives for the student

In your local AI client's MCP settings (not committed to any repo). The
team README (in `../springboot/`) keeps a screenshot of the transcript.
