# How to use this client config

This is the [client-side config](./mcp-client.json) to paste into your
AI client (Gemini Desktop / Antigravity, Claude Desktop, or any SSE MCP
client). It registers **this TP's** MCP server at `http://localhost:8080/sse`,
with **Basic auth** (TP1 default-deny is **preserved** — Step 1.1-1.2).

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

> **Why a Basic header here at all:** TP1's `SecurityConfig` requires
> authentication on every non-permitted route; `/sse` and `/mcp/message`
> fall under that rule (they are *not* in the `permitAll` matcher). That is
> the **security decision of this TP** (see `exercise.md` Step 1.2): the
> AI client authenticates **as a SELLER**, so *the write tool still enforces
> TP1 ownership*. Opening these routes to the world would be a 4xx-level
> mistake (a 500 where it should be a 403).

## Where this config lives for the student

In your local AI client's MCP settings (not committed to any repo). The
team README (in `../springboot/`) keeps a screenshot of the transcript.
