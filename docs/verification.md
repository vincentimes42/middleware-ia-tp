# TP IA — stack verification

This file is a **live verification** of the MCP server + the Gemini call.
After `docker compose up -d` in the TP1 repo, run each section in order.
Every section ends with a `curl` that should return data — that is the
evidence the pipeline actually works. All commands are **copy-pasteable**.

> **Prereq:** the TP1 app (your `springboot` repo) is up at
> `http://localhost:8080/actuator/health`, **TP1 is green**, and the **two
> required base edits** from **README §0** are done (the `pom.xml` dep + the
> `application.yml` `spring.ai.mcp.server` block). If you are verifying before
> Steps 2-4, Sections 1-2 work — the MCP server is up, but `tools/list` is
> still empty (you have added no tools yet).
>
> **Paths:** the commands below use two variables (set them once per terminal,
> from the "Run" box of [`README.md`](../README.md)):
>
> ```bash
> export TP1_DIR="/path/to/your/tps-1-repo"     # the Spring Boot app repo
> export IA_DIR="/path/to/ia-repo"              # this IA repo
> ```

---

## 1) App + SSE handshake (Step 1)

The MCP SSE transport is **two channels**: one long-lived **stream** you *read*
(`GET /sse`) and a **POST** endpoint you *talk to*
(`POST /mcp/message?sessionId=...`). The response to a POST arrives **on the
stream**, not in the POST body.

Terminal A — open the stream (Basic auth = a TP1 SELLER, e.g. `alice@test`):

```bash
curl -N -s \
  -H "Accept: text/event-stream" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  http://localhost:8080/sse
```

Expected: `event: endpoint` whose `data` is the message path
(`/mcp/message?sessionId=<id>`). Note the value (call it `$EP`).

Terminal B — open the MCP session with `initialize`:

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":1,"method":"initialize",
        "params":{"protocolVersion":"2025-06-18","capabilities":{},
                  "clientInfo":{"name":"verification","version":"1.0.0"}} }'
```

Expected: on **Terminal A**, a JSON-RPC `result` whose `serverInfo.name`
is `listings-api` (or your custom `spring.ai.mcp.server.name`).

Then list tools (should be **empty** before Step 2, **3** after Step 4):

```bash
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":2,"method":"tools/list","params":{} }' | python3 -m json.tool
```

Expected: `result.tools` is an array. Before Step 2 it is empty; after Step 4
it contains `get_listing`, `list_listings`, `change_listing_price`, each with
a non-empty `description` and an `inputSchema`.

---

## 2) Read a tool by hand (Step 2)

Once `tools/list` shows `get_listing` and `list_listings`, call them:

```bash
# get_reading {id:1}
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":10,"method":"tools/call",
        "params":{"name":"get_listing","arguments":{"id":1}} }'
    # -> result.content[0].text holds the JSON of listing 1
```

```bash
# list_listings {category, maxPrice}  (TP1 category enum: SPORTS, ELECTRONICS, ...)
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":11,"method":"tools/call",
        "params":{"name":"list_listings","arguments":{"category":"SPORTS","maxPrice":500}} }'
```

Expected: the tool returns **exactly the same data** as the REST endpoint you
get with `curl -s http://localhost:8080/listings` (same ids, same fields).

---

## 3) The write tool + ownership (Step 4)

Three cases the tool **must** distinguish: valid / non-owner / unknown. All three
return a **tool result** (never a raw 500):

```bash
# (a) owner writes — should succeed
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":20,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":1,"newPrice":320}} }'
    # -> result.content[0].text = ok / "updated"
#     -> then:
curl -s http://localhost:8080/listings/1 | python3 -c 'import sys,json;d=json.load(sys.stdin);print("price:",d["price"])'
#     # expected: 320

# (b) non-owner writes — should return "(403) ..." (the tool result, not a 500)
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":21,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":42,"newPrice":5}} }'
    # -> result.content[0].text starts with "403"
#     -> and a fresh GET /listings/42 shows the SAME price as before the call

# (c) unknown id — should return "(404) ..."
curl -s http://localhost:8080"$EP" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  -d '{ "jsonrpc":"2.0","id":22,"method":"tools/call",
        "params":{"name":"change_listing_price","arguments":{"id":99999,"newPrice":5}} }'
    # -> result.content[0].text starts with "404"
```

Note: the "non-owner" case depends on the credential **not** being the owner of
the listing. If `alice@test` *does* own listing 42, use a different id, or a
different SELLER credential. What matters is the **status prefix in the tool
result**.

---

## 4) Waouh — the AI client (Step 3)

Not curl-able — the point of this step is the AI client (Antigravity or
another MCP-over-SSE client). The **proof** is the transcript: it lists **your
3 tools** and renders the tool result in natural language. Paste the transcript
in the team README (`../springboot/README.md`) alongside a side-by-side
`curl GET /listings/{id}` so you can show the model did **not invent the data**.

If your client does not list tools, come back to Section 1 — the stream must
emit the `endpoint` event before any `tools/list` can be answered.

---

## 5) Gemini from the API (Steps 5-6, bonus)

Generate a description by calling the API (your Java code calls Gemini, not
the client). The key must be in the **env variable** `GEMINI_API_KEY`, not
anywhere in the repo:

```bash
export GEMINI_API_KEY="your-AI-Studio-key"

# from "$TP1_DIR" so the env reaches the container on a rebuild:
cd "$TP1_DIR" && docker compose up -d --build listings-api

# (Step 5: return-only)
curl -s -X POST http://localhost:8080/listings/1/ai-description \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  | python3 -m json.tool
# Expected 200: { "listingId": 1, "description": "..." }

# (Step 6: regenerate — re-runs + persists into the V2 payload)
curl -s -X POST http://localhost:8080/listings/1/ai-description/regenerate \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  | python3 -m json.tool

# Step 6 proof: the V2 payload now carries the stored text
curl -s "http://localhost:8080/listings/1?version=V2" \
  -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" \
  | python3 -m json.tool | grep -E "aiDescription"
```

Failure paths to spot (the endpoint must **not** 500):

| Condition | Expected |
|---|---|
| `GEMINI_API_KEY` not in env | **503** + `{ "error":"NO_API_KEY", ... }` |
| Model / network error | **503** + `{ "error":"LLM_UNAVAILABLE", ... }` |
| Non-owner write | **403** (TP1 rule, unchanged) |
| Unknown listing id | **404** (TP1 rule, unchanged) |

---

## 6) No secrets, no solution (cross-check)

```bash
cd "$TP1_DIR"
# no literal key anywhere in Java / yml / pom / compose
git grep -ni "GEMINI_API_KEY\|AI_STUDIO" src pom.xml docker-compose.yml
# expected: only System.getenv("GEMINI_API_KEY") matches in Java; zero in yml/pom/compose

# the container is not baked with a key even if you did `export` earlier
docker inspect listings-api --format '{{.Config.Env}}' | grep -i gemini || echo "ok"
```

---

## 7) Stop

```bash
cd "$TP1_DIR" && docker compose down              # keep the DB
# (add -v to wipe the Postgres data)
```
