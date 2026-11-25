# TP IA — verification commands

Copy-pasteable commands for every `Verify` section of
[`../exercise.md`](../exercise.md): the MCP server (MCP 2026-07-28,
`POST /mcp`), the tools, the ownership rules, and the Gemini call (bonus).

> **Prerequisite:** TP1 is green and up on `http://localhost:8080`, with the
> TP1 accounts `alice@test.org` (owner of listing `1`) and `bob@test.org`,
> password `secret123`, and the **README §0** edits applied.

## 0) Variables and helpers (once per terminal)

```bash
export TP1_DIR="/path/to/your/tps-1-repo"     # the Spring Boot app repo
API=http://localhost:8080
ALICE='alice@test.org:secret123'
BOB='bob@test.org:secret123'
JSON='Content-Type: application/json'
ACCEPT='Accept: application/json, text/event-stream'

# MCP 2026-07-28 is stateless: no initialize, no session id. Every request carries
# Mcp-Protocol-Version, Mcp-Method (+ Mcp-Name for tools/call) and a _meta.
META='"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientInfo":{"name":"verification","version":"1.0"},"io.modelcontextprotocol/clientCapabilities":{}}'
# choose the caller for the next calls ($1 = a Basic "email:password")
mcp_init() { AS="$1"; echo "calls as ${AS%%:*}"; }
# one JSON-RPC request: mcp METHOD 'EXTRA-PARAMS' [TOOL-NAME]; prints the JSON answer
# (the server answers either plain JSON or one SSE frame "data:{...}")
mcp() {
  curl -s -X POST "$API/mcp" -u "$AS" -H "$JSON" -H "$ACCEPT" \
    -H "Mcp-Protocol-Version: 2026-07-28" -H "Mcp-Method: $1" ${3:+-H "Mcp-Name: $3"} \
    -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"$1\",\"params\":{${2:+$2,}$META}}" \
    | sed -n '/^data:/{s/^data://;p;d};/^{/p' | tail -1 | python3 -m json.tool
}
# call one tool: tool NAME 'JSON-ARGS'
tool() { mcp tools/call "\"name\":\"$1\",\"arguments\":$2" "$1"; }
```

## 1) Step 1 — the MCP server

```bash
# anonymous -> 401 (the route keeps Basic auth)
curl -s -o /dev/null -w "anonymous POST /mcp -> %{http_code}\n" -X POST "$API/mcp" -H "$JSON" -d '{}'

# server/discover: 200, the protocol version, serverInfo.name = listings-api
mcp_init "$ALICE"
mcp server/discover \
  | python3 -c 'import sys,json; r=json.load(sys.stdin)["result"]; print("versions", r["supportedVersions"], "| serverInfo", r["_meta"]["io.modelcontextprotocol/serverInfo"])'

# tools/list: 0 tools before Step 2, 2 after Step 2, 3 after Step 4
mcp tools/list \
  | python3 -c 'import sys,json; t=json.load(sys.stdin)["result"]["tools"]; print(len(t), "tool(s)"); [print(" ", x["name"], "|", x["description"], "|", list(x["inputSchema"].get("properties",{})), "required", x["inputSchema"].get("required")) for x in t]'
```

## 2) Step 2 — read tools vs the REST API

```bash
mcp_init "$ALICE"
tool get_listing '{"id":1}'                                   # result.content[0].text = the listing
curl -s -u "$ALICE" "$API/listings/1" | python3 -m json.tool   # same fields

tool list_listings '{"category":"SPORTS","maxPrice":500}' \
  | python3 -c 'import sys,json; print("tool ids:", sorted(i["id"] for i in json.loads(json.load(sys.stdin)["result"]["content"][0]["text"])))'
curl -s "$API/listings?category=SPORTS&maxPrice=500" \
  | python3 -c 'import sys,json; print("REST ids:", sorted(i["id"] for i in json.load(sys.stdin)["content"]))'
# the two id lists must be equal
```

## 3) Step 4 — the write tool and ownership

```bash
price() { curl -s "$API/listings/1" | python3 -c 'import sys,json; print("price of listing 1:", json.load(sys.stdin)["price"])'; }
show()  { python3 -c 'import sys,json; r=json.load(sys.stdin)["result"]; print("isError:", r.get("isError"), "| text:", r["content"][0]["text"])'; }

# (a) owner (alice) -> OK, price changed
price
mcp_init "$ALICE"
tool change_listing_price '{"id":1,"newPrice":320}' | show       # isError: False
price                                                              # 320

# (b) non-owner (bob) -> 403, price unchanged
mcp_init "$BOB"
tool change_listing_price '{"id":1,"newPrice":5}' | show          # isError: True | text: 403 ...
price                                                              # still 320

# (c) unknown id -> 404
tool change_listing_price '{"id":99999,"newPrice":5}' | show      # isError: True | text: 404 ...

# (d) bad price -> 422 newPrice
mcp_init "$ALICE"
tool change_listing_price '{"id":1,"newPrice":-5}' | show         # isError: True | text: 422 newPrice ...
price                                                              # still 320
```

## 4) Step 3 — the AI client

Not curl-able: the proof is the client transcript (it lists **your 3 tools**
and answers from your tool results), next to a side-by-side
`curl GET /listings/{id}`. If the client shows no tools, replay §1 with the
**same** Basic credential as the client config.

## 5) Steps 5-6 (bonus) — Gemini from the API

```bash
export GEMINI_API_KEY="your-AI-Studio-key"
cd "$TP1_DIR" && docker compose up -d --build listings-api       # needs ${GEMINI_API_KEY:-} in the compose (README §0)

# Step 5: owner -> 200 {listingId, description}
curl -s -X POST -u "$ALICE" "$API/listings/1/ai-description" | python3 -m json.tool
# non-owner -> 403, unknown -> 404
curl -s -o /dev/null -w "bob -> %{http_code}\n" -X POST -u "$BOB" "$API/listings/1/ai-description"
curl -s -o /dev/null -w "unknown -> %{http_code}\n" -X POST -u "$ALICE" "$API/listings/99999/ai-description"

# no key -> 503 + JSON
unset GEMINI_API_KEY && docker compose up -d listings-api && sleep 20
curl -s -w "\n-> %{http_code}\n" -X POST -u "$ALICE" "$API/listings/1/ai-description"

# Step 6: regenerate (owner), then the V2 payload carries the stored text, V1 does not
curl -s -X POST -u "$ALICE" "$API/listings/1/ai-description/regenerate" | python3 -m json.tool
curl -s -o /dev/null -w "bob regenerate -> %{http_code}\n" -X POST -u "$BOB" "$API/listings/1/ai-description/regenerate"   # 403 (updateAiDescription: TP1 owner check)
curl -s "$API/listings/1?version=V2" | python3 -c 'import sys,json; print("V2 aiDescription:", json.load(sys.stdin).get("aiDescription"))'
curl -s "$API/listings/1"            | python3 -c 'import sys,json; print("V1 has aiDescription:", "aiDescription" in json.load(sys.stdin))'
(cd "$TP1_DIR" && docker compose exec -T listings-postgres psql -U listings -d listings \
  -c "SELECT id, filename FROM databasechangelog ORDER BY orderexecuted DESC LIMIT 1")   # v1.2-ai
```

| Condition | Expected |
|---|---|
| `GEMINI_API_KEY` not set or blank | **503** + `{ "error":"LLM_UNAVAILABLE", ... }` |
| Model / network error, timeout | **503** + `{ "error":"LLM_UNAVAILABLE", ... }` |
| Non-owner | **403** (TP1 rule, unchanged) |
| Unknown listing id | **404** (TP1 rule, unchanged) |

## 6) No secrets (cross-check)

```bash
cd "$TP1_DIR"
git grep -n "GEMINI_API_KEY"            # only the Java getenv read + ${GEMINI_API_KEY:-} in docker-compose.yml
git log -p | grep -c "AIza"             # 0: no Google API key ever committed
```

## 7) Stop

```bash
cd "$TP1_DIR" && docker compose down              # keep the DB
```

## 8) Common issues

| Symptom | Cause | Fix |
|---|---|---|
| Antigravity stays at `listings initializing...` | server on a session-based transport (`STREAMABLE`, Spring AI 2.0) | Spring AI 2.1 + `spring.ai.mcp.server.protocol: MODERN` (README §0) |
| a tool is named `listListings` | `@Tool` without `name` | `@Tool(name = "list_listings", …)` |
| `tools/list` is empty after Step 2 | no `ToolCallbackProvider` bean for `ListingTools` | Step 2, item 2; the startup log says `Registered modern tools: N` |
| `400 Missing required header: Mcp-Method` | a request without the 2026-07-28 headers (or a hand-written `initialize`) | use the helpers of §0 (`Mcp-Protocol-Version`, `Mcp-Method`, `Mcp-Name`, `_meta`) |
| app does not start: `found duplicate key spring` | README §0 block pasted as a second top-level `spring:` | put `ai:` under the existing `spring:` (and `listings:`) keys |
| `503 … high demand` / timeouts on every Gemini call | `gemini-flash-latest` overloaded on the free tier | `LISTINGS_AI_MODEL=gemini-flash-lite-latest` (README §0) |
| `{"description": null}` in `get_listing` but not in `GET /listings/1` | Spring AI's own Jackson mapper ignores TP1's `non_null` setting | `@JsonInclude(NON_NULL)` on the returned DTO (Step 2) |
| 403 text missing, raw exception message instead | no `AiErrorMapper` | Step 4, item 2 |
| `503` even with the key exported | the compose does not forward it | `GEMINI_API_KEY: ${GEMINI_API_KEY:-}` in the base compose, then `up -d` again |
| `404 … model … not found` | retired fixed model version | `listings.ai.model: gemini-flash-latest` |
| Basic header rejected by the client | base64 of a value ending with a newline | encode with `printf '%s'`, not `echo` |
