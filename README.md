# TP IA — MCP server + Gemini in the Spring Boot API

Follow-up to the Spring Boot TP (TP1, `../springboot/`). We take the **TP1
listings API** and add an **AI layer**: the API becomes an **MCP (Model
Context Protocol) server** — its read/write methods are exposed as *tools* —
so an AI client can **call your code in natural language**. Then we **flip the
roles**: the API itself calls the **Gemini** model to **generate content**
for a listing.

What each piece is, and who owns it:

| Piece | What it is | Lives in |
|---|---|---|
| **MCP server** | Your Spring Boot app exposing `@Tool` methods over SSE | **`../springboot/`** (a new `ai` package) |
| **MCP client ("waouh")** | **Gemini Desktop / Antigravity**, a local app that talks MCP | on the student's machine — config provided in `client/` |
| **Gemini** | Google's model (free Google AI Studio key), called from the API | remote API, no local service |

There is **no new Docker stack** and **no port change**. The API stays on
**8080**; the MCP endpoints are extra routes on that same port. The only extra
artifact on the host is the **AI client** you point at `http://localhost:8080`.

> The full pedagogical statement is [`exercise.md`](exercise.md). This file is
> **operational only** — how to start, where to look. The two do not duplicate:
> the rules live in `exercise.md`, the commands live here.

## How the pieces fit

```
   Gemini Desktop /
   Antigravity    ----  GET  http://localhost:8080/sse          (open the stream)
   (your "waouh") ----  POST http://localhost:8080/mcp/message  (every JSON-RPC call:
                         initialize, tools/list, tools/call)   read tools + write tool)
                            ^
                            |  same Spring app as TP1 (port 8080)
   +-----------------------------------------------------------+
   |                 listings-api (TP1, 8080)                  |
   |   ... TP1 REST endpoints (unchanged) ...                  |
   |   ... POST /listings/{id}/ai-description  (BONUS) ...     |
   +---------------------------^-------------------------------+
                               |  POST (Google AI Studio)
                               v
                          Gemini API  gemini-2.0-flash
```

- **SSE stream** = `GET /sse` (server→client events, keeps the session open).
- **JSON-RPC calls** = `POST /mcp/message` — this is where `tools/list` and
  `tools/call` actually go. (Spring AI 2.0 also offers a single-endpoint
  **Streamable HTTP** variant at `POST /mcp`; this TP standardizes on **SSE**.)

Services still talk over the shared `middleware` network exactly as in TP1
(`listings-postgres`, container name `listings-api`, host `8080`). Nothing about
TP1's ports or network changes for this TP.

## Prerequisites

- **TP1 is done and green** (Steps 1·2·3·5·6·8: register, CRUD, ownership,
  `?version=V1|V2`). This TP **reuses the TP1 listing business methods**. If TP1
  is not at that state, finish it first — it is an independent TP.
- `docker compose version` (the base, `../springboot`, must run).
- JDK 27 + Maven 3.9+ (already installed for TP1).
- **One** **AI client** that speaks MCP over HTTP/SSE and that you can register
  a local server in: **Gemini Desktop** (with its code space
  **Antigravity**) or an equivalent MCP desktop client.
- **Ports free on the host:** `8080` (the API + MCP) · `5435` (TP1 Postgres).
- A **free Google AI Studio API key** for the bonus (Step 5):
  <https://aistudio.google.com/apikey>. Keep it in an **environment variable**
  (`GEMINI_API_KEY`) — **never** in code (anti-pattern, see `exercise.md`).

> **Tip:** Keycloak, Monitoring and IA are **alternatives** — all start the same
> base `listings-api` (same container name), so run **one** at a time. Do not run
> two AI/identity/observability TPs against the same base simultaneously.

## Run

### 1) Start the TP1 base (unchanged)

```bash
cd ../springboot
docker compose up -d
curl -fsS http://localhost:8080/actuator/health     # API on 8080
```

### 2) Make the API an MCP server (Steps 1-2)

The work is **code in `../springboot`** (a new `com.app.listings.ai` package,
`@Tool` methods + config), exactly the "required TP1 edit" pattern used by the
other TPs. The concrete additions are documented per step in `exercise.md`; the
canonical `application.yml` block and the two `pom.xml` dependencies are in
**Appendix A/B of `exercise.md`** and mirrored in `docs/verification.md`.

After editing, restart the base:

```bash
cd ../springboot
docker compose up -d --build listings-api && docker compose ps
```

Then prove the MCP handshake works **without the AI client** (raw `curl`). The
MCP routes keep **HTTP Basic** (a TP1 seller):

```bash
# open the SSE stream (Basic = alice@test:secret123, a TP1 seller account)
curl -N -s -H "Authorization: Basic YWxpY2VAdGVzdDpzZWNyZXQxMjM=" http://localhost:8080/sse
```

The first event is the **endpoint** (the POST target for the JSON-RPC calls).
The full handshake, `tools/list`, and hand-written `tools/call` are in
[`docs/verification.md`](docs/verification.md).

### 3) Connect the AI client (Step 3 — the "waouh")

Point **Gemini Desktop / Antigravity** at the local server. The exact config
snippet to paste — **including the HTTP Basic header** (we are **not** opening
the MCP routes to the world; they keep TP1's default-deny) — is in
[`client/mcp-client.json`](client/mcp-client.json). The client authenticates as
a specific **SELLER** (a TP1 account you already have). Then ask, in natural
in natural language, e.g. *"what do you have in sports under 500?"* or
*"change the price of my first listing to 320"*). The client picks the right
`@Tool`, your code runs, and the answer comes back in natural language.

> Security lesson — **the write tool respects TP1 ownership**: even though the
> AI *chooses* to call `change_listing_price`, TP1's
> `listing.getSeller().getId() == currentUserId` check still runs inside the
> tool. An AI acting as `alice` cannot change a listing owned by `bob`.

### 4) Flip roles — call Gemini from the API (Step 5, BONUS)

```bash
export GEMINI_API_KEY="your-AI-Studio-key"
# from ../springboot so the env reaches the container on a rebuild:
docker compose up -d --build listings-api
curl -s -X POST http://localhost:8080/listings/1/ai-description
```

### 5) Stop

```bash
cd ../springboot && docker compose down          # keep the DB
# (add -v to wipe the Postgres data)
```

## Tree

```
IA/
├── .gitignore
├── README.md                 # this file (operational)
├── exercise.md               # the statement (5-6h, core 1-4 + bonus 5-6)
├── client/
│   └── mcp-client.json       # the config to paste in the AI client (Step 3)
└── docs/
    └── verification.md       # raw-MCP handshake + Gemini curl, copy-pasteable

../springboot/ (the code you extend — NOT in this repo)
├── pom.xml                   # +2 deps (spring-ai mcp server, google-genai) [Step 1 / 5]
├── src/main/java/com/app/listings/ai/
│   ├── tools/...             # your @Tool methods (Step 2, 4)
│   └── GeminiClient.java     # the Gemini call (Step 5, BONUS)
├── src/main/resources/application.yml   # + spring.ai.mcp.server block [Step 1]
└── src/main/java/.../config/SecurityConfig.java  # MCP routes keep Basic auth (TP1 default-deny)
```

> **Solutions** (supervisor only, NOT shipped): none in this repo. Like the other
> TPs, the Java lives in your copy of `../springboot`, and the statement +
> provided config live here.

## Useful links (version-pinned)

- Spring AI MCP server (Boot 4 line, matches TP1):
  <https://github.com/spring-projects/spring-ai/tree/main/models/> + the
  `spring-ai-starter-mcp-server-webmvc` artifact (`2.0.1`).
- MCP (the protocol): <https://modelcontextprotocol.io/introduction>
- Gemini (Google AI Studio, free tier): <https://aistudio.google.com> ·
  REST reference <https://ai.google.dev/api>
- Google `google-genai` Java SDK: <https://github.com/googleapis/java-genai>

> TP rule: no `TODO`, no stub, no secret in production code.
> The AI client is a **viewer** for your API — the **code** (the `@Tool`s, the
> Gemini call) is in `../springboot`, and that is what you are graded on.
