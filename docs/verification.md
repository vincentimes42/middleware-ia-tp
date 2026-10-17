# TP IA — verification (copy-pasteable evidence)

This is the **proof layer** for the acceptance checklists in `exercise.md`.
You should be able to run each block and see the expected result **without**
trusting the AI client. The AI client (Step 3) is the *waouh*; these raw calls
are the *proof it is real*.

Base facts (do not change):
- Base / ports: API on **`8080`** (TP1 base, unchanged), Postgres **5435**.
- MCP transport: **SSE**. Stream = `GET http://localhost:8080/sse`; JSON-RPC = `POST http://localhost:8080/mcp/message`.
- **Auth**: the MCP routes keep **HTTP Basic** (TP1 default-deny is preserved).
  Add `-H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM="` (= `alice@test:secret123`, a
  TP1 **seller**) to every `curl` below. That is the seller the AI acts as.
- Tool names (defined in `exercise.md`): `get_listing`, `list_listings` (Step 2, read),
  `change_listing_price` (Step 4, write).

## 1. The MCP handshake (proof the server is up — Step 1)

The SSE transport is **two channels**: one long-lived stream you *read*, and a
POST endpoint you *post* JSON-RPC to. The response to a POST arrives **on the stream**, not in the POST body.

**Terminal A — open the stream** (it will not return; that is normal):

```bash
curl -N -s \
  -H "Accept: text/event-stream" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  http://localhost:8080/sse
```

The first event must be the **endpoint** event, e.g.:

```
event: endpoint
data: /mcp/message?sessionId=<some-id>
```

> Note the value of `data` (call it `$EP`). **Without** the `Authorization`
> header you get a `401` — that is *correct* (TP1 default-deny). The MCP routes
> are **not** opened to the world in `exercise.md` Step 1; you authenticate the
> way a normal API client already does (Basic, a TP1 **seller**).

**Terminal B — talk JSON-RPC to it.** Replace `$EP` with the endpoint value
above and the session id if it is relative (prefixed `http://localhost:8080`).

(a) `initialize` — open the MCP session:

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{
    "jsonrpc":"2.0","id":1,"method":"initialize",
    "params":{"protocolVersion":"2025-06-18","capabilities":{},
              "clientInfo":{"name":"verification","version":"1.0.0"}}
  }'
# -> on Terminal A: a JSON-RPC result with "serverInfo" (name "listings-api",
#    capabilities.tools). This is handshake proof.
```

(b) `tools/list` — your tools must be present:

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":2,"method":"tools/list","params":{} }'
# -> on Terminal A: result.tools = [ get_listing, list_listings,
#    (change_listing_price after Step 4) ] each with a "description" and "inputSchema"
```

### Acceptance (Step 1)

- [ ] `GET /sse` **without** a Basic credential → `401` (routes are **not** opened to the world)
- [ ] `GET /sse` **with** a `SELLER` Basic credential → `text/event-stream` and the **`endpoint` event** is emitted
- [ ] `initialize` returns a `result` with `serverInfo.name` = `listings-api` (or your `spring.ai.mcp.server.name`)
- [ ] `tools/list` returns **at least** `get_listing` and `list_listings`, each with a non-empty `description`

## 2. Calling a read tool by hand (proof the tool runs Java — Step 2)

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":3,"method":"tools/call",
        "params":{"name":"get_listing","arguments":{"id":1}} }'
# -> on Terminal A: result.content[0].text holds the JSON of listing 1
#    (title, category, price, …). This proves the @Tool ran your ListingService.
```

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":4,"method":"tools/call",
        "params":{"name":"list_listings","arguments":{"category":"SPORTS","maxPrice":500}} }'
# -> returns only SPORTS listings priced <= 500
```

### Acceptance (Step 2)

- [ ] `tools/call get_listing` returns the actual DB row for that id (compare with `GET http://localhost:8080/listings/1`)
- [ ] `tools/call list_listings` with filters returns the filtered set
- [ ] Every tool you exposed has a `description` that a human (and an LLM) can read — this is the *contract*, not decoration

## 3. Calling the write tool + bad input (proof of control — Step 4)

```bash
# valid: change price of listing 1 to 320 (acting as alice@test)
curl -s http://localhost:8080"$EP" -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":5,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":1,"newPrice":320}} }'
# -> result content says the update succeeded; GET /listings/1 now shows price 320

# invalid: a listing id that does not exist
curl -s http://localhost:8080"$EP" -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":6,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":99999,"newPrice":1}} }'
# -> the tool returns an MCP **error** (isError / error content) carrying 404 —
#    NOT a 500 crash, NOT a silent success

# forbidden: alice changes a listing owned by someone else (TP1 ownership)
curl -s http://localhost:8080"$EP" -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":7,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":42,"newPrice":1}} }'
# -> 403 surfaced as a tool result error (42 is not alice's listing)
```

### Acceptance (Step 4)

- [ ] `change_listing_price` actually persists (confirm with a fresh `GET /listings/{id}`)
- [ ] Unknown id → a clean `404` surfaced as a tool result error (not a stack-trace `500`)
- [ ] Non-seller / forbidden → `403` surfaced the same way (reuse TP1's ownership rules — the tool **delegates to the TP1 service**, it does not bypass them)

## 4. The "waouh" via the AI client (Step 3)

1. Start the base, register the server from [`../client/mcp-client.json`](../client/mcp-client.json)
   (it already carries the **Basic** header — the AI acts as that specific SELLER).
2. In the client, ask in plain language, e.g.:
   - *"List everything in sports under 500 euros."*
   - *"Change the price of my first listing to 320."*
3. The client must **list tools** (`get_listing`, `list_listings`, `change_listing_price`), **pick the right one**, fill the arguments, and **render the answer** in natural language.

> Ownership check: if the account behind the Basic header does **not** own the
> listing you tell it to change, the write must **fail with 403** (TP1 rule),
> even though the model confidently *tried*. That is the point: the model
> *requests*, your code *decides*.

### Acceptance (Step 3)

- [ ] The client shows your 2-3 tools in its tool list (not a built-in)
- [ ] A natural-language question results in **your Java running** (you see the DB change / correct data) — show the client transcript + one `GET /listings/{id}` before/after for the write
- [ ] The client does NOT invent the data — the price/category it reports match the TP1 API's own `GET` response
- [ ] A write on a listing the authenticated seller does **not** own → `403` (TP1 ownership enforced *inside* the tool)

## 5. Gemini from the API — BONUS (Step 5)

Generate a description by calling the API (your code calls Gemini, not the client):

```bash
export GEMINI_API_KEY="your-key"          # from Google AI Studio; NEVER in code
cd ../springboot
docker compose up -d --build listings-api

curl -s -X POST http://localhost:8080/listings/1/ai-description
# { "listingId": 1, "description": "A tidy 26\" gravel bike, recently serviced…" }
```

Raw Gemini call (what your Java should effectively do — for reference only, keep using the SDK in code):

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent" \
  -H "Content-Type: application/json" \
  -d '{
    "contents":[{"parts":[{
      "text":"Write a 2-sentence, friendly, specific listing description for a SPORTS item titled \"Gravel bike\" priced at 320 EUR. No fluff."
    }]}]
  }' \
  -G --data-urlencode "key=$GEMINI_API_KEY" | python3 -c "import sys,json;print(json.load(sys.stdin)['candidates'][0]['content']['parts'][0]['text'])"
```

### Acceptance (Step 5)

- [ ] `POST /listings/{id}/ai-description` returns **201/200** with a non-empty `description` derived from the listing's own `title`/`category`/`price`
- [ ] `GEMINI_API_KEY` absent → a clean **503/4xx** (a sensible message), **not** a 500 stack trace, and the key is **nowhere in source, compose, or logs**
- [ ] The response is stable to re-runs (deterministic-ish; same listing in, coherent marketing text out), and the generated text never invents a different price or category than the listing

---

## Appendix — quick reference

| What | Where | Value |
|---|---|---|
| MCP stream (SSE) | host | `http://localhost:8080/sse` |
| MCP JSON-RPC | host | `http://localhost:8080/mcp/message` (+`?sessionId=`) |
| Read tools | code (`../springboot`) | `get_listing`, `list_listings` |
| Write tool | code (`../springboot`) | `change_listing_price` |
| Gemini (BONUS) | remote | `…/v1beta/models/gemini-2.0-flash:generateContent` |
| Gemini key | env only | `GEMINI_API_KEY` (never committed) |

> Anti-patterns (from `exercise.md`): no secret in code/compose; no bypassing TP1's
> ownership rules from a tool; no 500 where a 4xx is correct; the AI client is a
> viewer, not the deliverable.
