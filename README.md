# TP IA — MCP server + Gemini in the Spring Boot API

> ## Before you start — every student needs a Google account
>
> This TP uses Google's AI, so **each of you needs a Google account** to
> obtain a **free API key**:
>
> - **Antigravity (Step 3)** is **Google's** code tool — it signs you in with
>   a Google account and lets you register the local MCP server.
> - **Gemini key (Step 5 / bonus)** — the *free-tier* key comes from
>   **Google AI Studio**: <https://aistudio.google.com/apikey> (same Google
>   account). Create it **once, before the session**; paste it into the
>   environment variable `GEMINI_API_KEY` (it is **never** committed).
>
> If you have **no Google account**, set one up before the TP — both the
> client and the bonus key depend on it. The free tier is generous enough for
> a whole class during the session.

Follow-up to the Spring Boot TP (TP1, `../springboot/`). We take the **TP1
listings API** and add an **AI layer**: the API becomes an **MCP (Model
Context Protocol) server** — its read/write methods are exposed as *tools* —
so an AI client can **call your code in natural language**. Then we **flip the
roles**: the API itself calls the **Gemini** model to **generate content**
for a listing.

What each piece is, and who owns it:

| Piece | What it is | Lives in |
|---|---|---|
| **MCP server** | Your Spring Boot app exposing `@Tool` methods over MCP 2026-07-28 (`POST /mcp`) | **`../springboot/`** (a new `ai` package) |
| **MCP client ("waouh")** | **Antigravity** (CLI or app; or any MCP 2026-07-28 client), a local app that talks MCP | on the student's machine — config provided in `client/` |
| **Gemini** | Google's model (free Google AI Studio key), called from the API | remote API, no local service |

There is **no new Docker stack** and **no port change**. The API stays on
**8080**; the MCP endpoint is an extra route on that same port. The only extra
artifact on the host is the **AI client** you point at `http://localhost:8080`.

> The full pedagogical statement is [`exercise.md`](exercise.md). This file is
> **operational only** — how to start, where to look. The two do not duplicate:
> the rules live in `exercise.md`, the commands live here.

## How the pieces fit

```
   Antigravity     ----  POST http://localhost:8080/mcp   (every JSON-RPC call:
   (your "waouh")         server/discover, tools/list, tools/call; headers
                          Mcp-Protocol-Version, Mcp-Method; no session)
                            ^
                            |  same Spring app as TP1 (port 8080), Basic auth
   +-----------------------------------------------------------+
   |                 listings-api (TP1, 8080)                  |
   |   ... TP1 REST endpoints (unchanged) ...                  |
   |   ... POST /listings/{id}/ai-description  (BONUS) ...     |
   +---------------------------^-------------------------------+
                               |  HTTPS (Google AI Studio key)
                               v
                          Gemini API  gemini-flash-latest
```

- **One endpoint**, `POST /mcp`, MCP revision **2026-07-28** (Spring AI
  `protocol: MODERN`). It is stateless: no `initialize`, no session; each
  request names its method (`Mcp-Method` header) and carries the protocol
  version (`Mcp-Protocol-Version` header and `_meta`). The client starts with
  `server/discover` (server name, versions, capabilities).
- The older session-based transports (`STREAMABLE`: `initialize` +
  `Mcp-Session-Id`; legacy HTTP+SSE) are **not** used: Antigravity CLI speaks
  2026-07-28 and never finishes connecting to a session-based server.

Services still talk over the shared `middleware` network exactly as in TP1
(`listings-postgres`, container name `listings-api`, host `8080`).

## Prerequisites

- **TP1 is done and green** (Steps 1·2·3·5·6·8: register, CRUD, ownership,
  `?version=V1|V2`). This TP **reuses the TP1 listing business methods**. If TP1
  is not at that state, finish it first — it is an independent TP.
- Linux or WSL2, Bash terminal ([details](../springboot/README.md#operating-system-and-shell)).
- `docker compose version` (the base, `../springboot`, must run).
- JDK 27 + Maven 3.9+ (already installed for TP1).
- **Antigravity** (recommended — Step 3) on the student's machine: the CLI
  (`agy`) or the app. It signs in with a Google account and lets you register
  a local MCP server. (Any MCP client that speaks the 2026-07-28 revision over
  HTTP works, but the lab is built around Antigravity.)
- **Ports free on the host:** `8080` (the API + MCP) · `5435` (TP1 Postgres).
- A **free Google AI Studio API key** for the bonus (Step 5):
  <https://aistudio.google.com/apikey>. Keep it in an **environment variable**
  (`GEMINI_API_KEY`) — **never** in code (anti-pattern, see `exercise.md`).

> **Tip:** the four TP2s (Keycloak, Monitoring, IA, Messaging) are
> **alternatives** — they all use the base `listings-api` (same container
> name), so run **one** TP2 at a time.

## Run

> **Two repos, two folders.** The TP1 repo (the Spring Boot app) and this
> IA repo are **side by side** (any folder names). Set these once per
> terminal so the commands below do not need hard-coded paths:
>
> ```bash
> export TP1_DIR="/path/to/your/tps-1-repo"     # the Spring Boot app repo
> export IA_DIR="/path/to/ia-repo"              # this IA repo
> ```

### 0) Required base edit — `../springboot/pom.xml`, `application.yml` (and the compose, bonus)

The TP's Java lives in your copy of `../springboot` (this repo ships **no Java**;
`client/` and `docs/` here are the config + verification, the way `../keycloak/`
ships `realm/` and `../monitoring/` ships the config YAMLs).

Two **additive** edits are required — `pom.xml` and `application.yml` — plus
one line in the base compose for the bonus:

**`pom.xml` — add both dependencies** (the Spring AI MCP server, and the Gemini
SDK for the bonus):

```xml
<!-- (a) MCP server — Spring AI 2.1.0-RC2 (Boot-4 line, works with TP1's Boot 4.1.1):
     the first line with the MCP 2026-07-28 revision (protocol MODERN) that Antigravity speaks -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
    <version>2.1.0-RC2</version>
</dependency>

<!-- (b) Google Gemini SDK — used by Steps 5-6 (BONUS) to call the model -->
<dependency>
    <groupId>com.google.genai</groupId>
    <artifactId>google-genai</artifactId>
    <version>1.78.0</version>
</dependency>
```

**`application.yml` — add the `spring.ai.mcp.server` block** (+ optional AI
settings). TP1's file **already has** top-level `spring:` and `listings:`
keys: put `ai:` **under each existing key** (indented), do **not** paste a
second `spring:` / `listings:` at the end (Spring Boot refuses the file:
`found duplicate key spring`).

```yaml
spring:
  ai:                               # NEW, under the existing spring: key
    mcp:
      server:
        name: listings-api          # shown in serverInfo of server/discover
        protocol: MODERN            # MCP 2026-07-28 on POST /mcp (what Antigravity speaks)
  application:                      # (existing TP1 keys, unchanged)
    name: listings
  # ...

listings:
  ai:                               # NEW, under the existing listings: key (bonus Steps 5-6)
    model: gemini-flash-latest      # Google's alias for the current Flash model
    timeout-ms: 10000
  # ... (existing TP1 keys, unchanged)
```

**Base compose — bonus only (Step 5):** in `listings-api` → `environment`,
forward the Gemini key from your shell to the container:

```yaml
      GEMINI_API_KEY: ${GEMINI_API_KEY:-}   # a reference, never the value
      # the model is a deployment choice (listings.ai.model): switch to
      # gemini-flash-lite-latest when flash-latest answers "high demand"
      LISTINGS_AI_MODEL: ${LISTINGS_AI_MODEL:-gemini-flash-latest}
```

`docker compose` does not pass host variables to a container unless they are
listed like this. The value comes from `export GEMINI_API_KEY=…` in the shell
that runs `docker compose up`, and stays out of the repo.

No new container, network, or port (unlike the Keycloak / Monitoring /
Messaging TPs).

### 1) Start the TP1 base (unchanged)

```bash
cd "$TP1_DIR"
docker compose up -d
curl -fsS http://localhost:8080/actuator/health     # API on 8080
```

### 2) Make the API an MCP server (Steps 1-2)

The work is **code in `../springboot`** (a new `com.app.listings.ai` package).
The **required `pom.xml` + `application.yml` additions** are the dedicated
[`0) Required base edit`](#0-required-base-edit----springspringbtpomxml-and-applicationyml)
above; the **how/why per step** (the `@Tool` methods, the `ToolCallbackProvider`
bean) is in `exercise.md` Steps 1-2.

After editing, restart the base:

```bash
cd "$TP1_DIR"
docker compose up -d --build listings-api && docker compose ps
```

Then prove the MCP server answers **without the AI client** (raw `curl`). The
MCP route keeps **HTTP Basic** (a TP1 seller):

```bash
curl -s -X POST http://localhost:8080/mcp -u 'alice@test.org:secret123' \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H 'Mcp-Protocol-Version: 2026-07-28' -H 'Mcp-Method: server/discover' \
  -d '{"jsonrpc":"2.0","id":1,"method":"server/discover","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientInfo":{"name":"curl","version":"1"},"io.modelcontextprotocol/clientCapabilities":{}}}}'
# -> 200, supportedVersions ["2026-07-28"], _meta serverInfo.name = listings-api
```

`tools/list` and hand-written `tools/call` are in
[`docs/verification.md`](docs/verification.md).

### 3) Connect the AI client (Step 3 — the "waouh")

Point **Antigravity** (or any MCP client) at the local server. The exact
config — **including the HTTP Basic header** (the MCP routes keep TP1's
default-deny and are **not** opened to the world) — is in
[`client/mcp-client.json`](client/mcp-client.json) (app) and
[`client/README.md`](client/README.md) (CLI: one `agy mcp add` command). The client authenticates as
a specific **SELLER** (a TP1 account you already have). Then ask, in natural
language, e.g. *"what do you have in sports under 500?"* or
*"change the price of my first listing to 320"*. The client picks the right
`@Tool`, your code runs, and the answer comes back in natural language.

> Security lesson — **the write tool respects TP1 ownership**: even though the
> AI *chooses* to call `change_listing_price`, TP1's owner check
> (`listing.owner.id == caller id`) still runs inside the tool. An AI acting as
> `alice` cannot change a listing owned by `bob`.

### 4) Flip roles — call Gemini from the API (Step 5, BONUS)

```bash
export GEMINI_API_KEY="your-AI-Studio-key"
# needs the GEMINI_API_KEY line of §0 in the base compose:
cd "$TP1_DIR" && docker compose up -d --build listings-api
curl -s -X POST -u 'alice@test.org:secret123' http://localhost:8080/listings/1/ai-description
```

### 5) Stop

```bash
cd "$TP1_DIR" && docker compose down              # keep the DB
# (add -v to wipe the Postgres data)
```

## Tree

```
IA/
├── .gitignore
├── README.md                 # this file
├── exercise.md               # the statement (core 1-4 + bonus 5-6 + delivery 7)
├── client/
│   ├── README.md             # the "waouh" client setup (Step 3)
│   └── mcp-client.json       # Antigravity mcp_config.json entry (serverUrl + Basic)
└── docs/
    ├── primer.md             # MCP, Spring AI, the Gemini SDK, what not to do
    └── verification.md       # raw MCP calls (curl) + Gemini curl, copy-pasteable

../springboot/ (the code you extend — NOT in this repo)
├── pom.xml                   # +2 deps (spring-ai mcp server, google-genai) [Step 1 / 5]
├── src/main/java/com/app/listings/ai/
│   ├── tools/...             # your @Tool methods (Step 2, 4)
│   └── GeminiClient.java     # the Gemini call (Step 5, BONUS)
├── src/main/resources/application.yml   # + spring.ai.mcp.server block [Step 1]
├── docker-compose.yml        # + GEMINI_API_KEY: ${GEMINI_API_KEY:-} [Step 5, BONUS]
└── src/main/java/.../config/SecurityConfig.java  # MCP routes keep Basic auth (TP1 default-deny)
```

> **Solutions** (supervisor only, NOT shipped): none in this repo. Like the other
> TPs, the Java lives in your copy of `../springboot`, and the statement +
> provided config live here.

## Useful links (version-pinned)

- Spring AI 2.1 — MCP server (Boot starters):
  <https://docs.spring.io/spring-ai/reference/2.1/api/mcp/mcp-server-boot-starter-docs.html>
- MCP revision 2026-07-28 (the protocol of this TP): <https://modelcontextprotocol.io/specification/2026-07-28>
- Spring AI 2.0 — tools (`@Tool`, `@ToolParam`): <https://docs.spring.io/spring-ai/reference/2.0/api/tools.html>
- MCP (the protocol): <https://modelcontextprotocol.io/docs/learn/server-concepts>
- Antigravity — MCP configuration: <https://antigravity.google/docs/mcp>
- Gemini models (current names): <https://ai.google.dev/gemini-api/docs/models> ·
  free key: <https://aistudio.google.com/apikey>
- Google `google-genai` Java SDK: <https://github.com/googleapis/java-genai>

> TP rule: no `TODO`, no stub, no secret in production code.
> The AI client is a **viewer** for your API — the **code** (the `@Tool`s, the
> Gemini call) is in `../springboot`, and that is what you are graded on.
