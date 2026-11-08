# TP IA — MCP server + Gemini in the Spring Boot API (5-6h)

## Overview

This is a **follow-up** to the Spring Boot TP (TP1, `../springboot/`). We reuse
the **TP1 listings API** — unchanged domain, unchanged port (8080) — and add an
**AI layer** with two small skills:

1. **The API becomes an MCP server.** Its business methods are exposed as
   *tools* over the **Model Context Protocol (MCP)**, so an **AI client** can
   call your Java and report back in **natural language** (the "waouh").
2. **We flip the roles.** The API itself calls the **Gemini** model (free
   Google AI Studio key, `google-genai` SDK) to **generate content** for a
   listing.

There is **no new Docker stack and no port change**: the MCP endpoints are
extra routes on the TP1 app (8080); the only new component on the student
machine is the **AI client** you point at `http://localhost:8080`.

**Validation method (the contract):** you prove each step with **`curl`**
(the raw MCP handshake, `tools/list`, hand-written `tools/call`), the **AI
client transcript**, and a **grep of your own Java** — **not** with a Java test
suite (writing tests is optional and not part of the deliverable). The
per-step `Verify` section **is** the "definition of done". Every `curl`
you need is copy-pasteable in [`docs/verification.md`](docs/verification.md).
The **code** lives in **your copy of `../springboot`** (a new
`com.app.listings.ai` package); **this repo** (`IA/`) only owns the statement,
the client config, and the verification curls.

> **Prerequisite: TP1 is done and green.** The TP1 API passes its core
> checklists (register, CRUD, ownership 401/403/404, `?version=V1|V2`) and runs
> on port 8080. Finish TP1 first — this TP only adds an AI layer on top of it.
>
> This is an **alternative** to [TP Keycloak](../keycloak/) and
> [TP Monitoring](../monitoring/) for the second session. All three start the
> same base `listings-api`, so run **one** at a time.

---

## What is MCP? (one minute)

**MCP is an open protocol that lets an AI application call your program through
standard "tools".** Your Spring Boot app is the **server**; an AI app
(**Antigravity**, or any other MCP client) is the **client**.

| Concept | What it is here |
|---|---|
| **Server** | your `listings-api` (TP1) exposing MCP over **SSE** |
| **Tool** | one Java method (`get_listing`, `list_listings`, `change_listing_price`) an LLM can call |
| **Transport** | **SSE**: `GET /sse` (stream) + `POST /mcp/message` (JSON-RPC: `initialize`, `tools/list`, `tools/call`) |
| **Client** | an AI app that lists your tools and calls them in natural language |

You are **not** writing a chatbot and you are **not** hosting an LLM. You are
**exposing your existing business code as callable tools** (Steps 1-4) and, in
the bonuses, **calling an LLM from your code** (Steps 5-6). That split is the
whole idea of the day.

---

## Before you start — the two libraries

New to MCP or to Spring AI? This ~3-minute model makes the steps below click.

### The stack, in one picture

```
   Antigravity (the "waouh" client)                        [Step 3]
        |   speaks MCP over SSE, authenticates as a SELLER
        v
   listings-api (TP1, port 8080)
     /sse, /mcp/message                   <- the MCP server   [Steps 1, 2, 4]
     com.app.listings.ai.*                <- your @Tool Java   [Steps 2, 4]
        |   POST (bonus, only in Steps 5-6) v
     Gemini (Google AI Studio, gemini-2.0-flash, key = env GEMINI_API_KEY)  [Steps 5, 6]
```

### The two libraries (versions pinned — do not guess)

| Library | Artifact · version | Used by |
|---|---|---|
| Spring AI (Boot 4 line) | `org.springframework.ai:spring-ai-starter-mcp-server-webmvc:**2.0.1**` | Steps 1-4 (the MCP server + tools) |
| Google Gemini SDK | `com.google.genai:google-genai:**1.78.0**` | Steps 5-6 (calling Gemini) |

> **Why two different things:** Spring AI 2.0 does **not** ship a Gemini model
> starter, and it is *not* how you "prompt Gemini". Spring AI = **exposing
> tools**. The **Gemini SDK** = **asking the model**. Keep the two separate in
> your head. Spring AI **2.0.1** targets the Boot 4 line and works with TP1's
> Boot **4.1.1** — no version conflict.

---

## Domain & data model (this TP)

The **business domain does not change** from TP1 (actors, the `Listing` entity,
ownership, `active=false`, V1/V2 visibility). What **changes** is only the AI
layer:

| Concern | TP1 | This TP |
|---|---|---|
| How you read/write listings | REST endpoints | **+ MCP tools** (same service, a new `ai` package) |
| Who can call the write path | HTTP Basic SELLER | **same** — the AI client also authenticates as a SELLER; **TP1 ownership (403) still applies inside the tool** |
| Who generates listing text | (none) | **Gemini**, called by your Java (bonus) |
| What the "AI" decides | — | nothing — it *requests*; **your code decides** (404/403/422) |

> **Key idea for the day:** exposing your API to an LLM does **not** remove any
> security or validation. The model picks a tool and fills the arguments;
> **your Java still enforces 404 / 403 / 422.** If the model *requests* a change
> to a listing it does not own, it gets a clean `403` — not a crash, not a
> success.

---

## Scope

| Track | Steps | Target (2nd-session binôme) |
|---|---|---|
| **Core** (required — the waouh must work) | 1 · 2 · 3 · 4 | ~3h30 |
| **Bonus** (start only when the core checklists are green) | 5 (Gemini text generation) · 6 (persist the text + harden the prompt) | ~2h |
| **Delivery** | 7 (final integration) | ~0h30 |

Work the core **in order**; each step's `Verify` section is your "definition of done".
**Core ~ 3h30, bonuses ~ 2h** — the binôme should finish in **5-6 h**. If the
core is not green, do not start the bonuses.

---

## Conventions

- **Service name** (used in `serverInfo.name`, tool names, logs): `listings-api`.
- **Port:** API + MCP on **8080** (unchanged from TP1). Postgres **5435**.
- **MCP transport:** **SSE**. Stream = `GET http://localhost:8080/sse`;
  JSON-RPC = `POST http://localhost:8080/mcp/message`.
- **Auth on MCP routes:** **HTTP Basic** (TP1's default-deny is **preserved**).
  The AI client authenticates as a **SELLER**; anonymous → **401**.
- **Tool names (fixed — do not rename):** `get_listing`, `list_listings` (read
  tools, Step 2) and `change_listing_price` (write tool, Step 4).
- **Category values (TP1, 9):** `ELECTRONICS`, `FURNITURE`, `CLOTHING`, `BOOKS`,
  `VEHICLES`, `SPORTS`, `HOUSEHOLD`, `GARDEN`, `OTHER`.
- **Gemini model** (bonus): any `gemini-*-flash` you have on your free AI Studio
  key (default in this TP: `gemini-2.0-flash`).
- **The key** is the **environment variable `GEMINI_API_KEY`** only — never in
  code, compose, the repo, or the git history.
- **Error JSON:** the TP1 error body shape is **unchanged** (no `fields` map
  needed for tools). MCP **tool results** carry the **HTTP status as text**
   (see the table below and Step 4).

---

## Conventions — how to validate each step

Every step is proven with **observable data**, not a Java test:

| Level | What you assert | How |
|---|---|---|
| **Handshake** | the MCP server is up **and** keeps auth | `curl` the SSE `endpoint` event — [`docs/verification.md` §1](docs/verification.md) |
| **Tools** | `tools/list` returns your tools with a readable `description` + `inputSchema` | [`docs/verification.md` §1](docs/verification.md) |
| **Call** | `tools/call` runs **your Java** and returns the DB data | [`docs/verification.md` §2-3](docs/verification.md) |
| **Client** | the AI client lists **your** tools and answers in natural language | the **client transcript** (Step 3) |
| **LLM** (bonus) | the API returns Gemini text for a listing | `curl POST /listings/{id}/ai-description` (Step 5) |

> **Rule:** you may not *pass* a step by only "the client said something".
> You must be able to show the **raw MCP call result** (or a `GET` readback)
> **and** a client transcript that agrees with it.

### What NOT to do

- **Never** put an API key (Gemini or any) in code, compose, the repo, or logs.
- **Never** open the MCP routes to the world — they **keep Basic auth**.
- **Never** bypass TP1 ownership from a tool — a **403** for a non-owner is the *correct* outcome.
- **Never** let a bad tool argument crash the app — surface it as `404` / `422`, not `500`.
- **Never** let generated text overrule DB facts (price, category) — the model
  *describes*, the DB *decides*.

---

## Error handling & tool results (cross-cutting)

Step 4 builds on TP1's `GlobalExceptionHandler` (409 / 422, etc.) and adds one
concern: **how a tool reports a business error to the LLM without a 500.**

| Situation | REST API (TP1) | MCP **tool result** (this TP) |
|---|---|---|
| Unknown listing id | **404** + JSON | tool result text **starts with `404`** + a human-readable message |
| Non-owner writes | **403** + JSON | tool result text **starts with `403`** + message |
| Invalid argument (`newPrice <= 0`) | **422** + `fields` | tool result text **starts with `422`** + field name |
| Gemini key missing / model down (bonus) | **503** + JSON | n/a (it is a REST endpoint, Step 5) |

> **The rule of thumb:** a tool **never throws a raw 500** for a *business*
> condition. It returns a **tool result** whose `text` carries the HTTP status
> (e.g. `(404) Listing not found: id=99999`). This is what keeps the LLM in a
> recoverable state and is the visible mark of "we know what we are doing."

---

## Step 1 — Bring up the MCP server   CORE

**Outcome** — Your TP1 API **is an MCP server over SSE**: `GET /sse` +
`POST /mcp/message` are exposed on port 8080, **keep HTTP Basic auth**
(anonymous still → 401), and the handshake reports
`serverInfo.name = listings-api`. Raw curl is enough; the AI client comes in
Step 3.

### Build it
1. **Add** the MCP server dependency to `pom.xml` —
   `org.springframework.ai:spring-ai-starter-mcp-server-webmvc` (the version
   block is in **README §0**).
2. **Add** the `spring.ai.mcp.server` block to `application.yml` —
   `name: listings-api`, `protocol: SSE` (README §0).
3. **Check** `SecurityConfig` — `/sse` and `/mcp/message` already fall under
   `anyRequest().authenticated()`, so they **keep Basic auth** (that is the
   point; do **not** permit them).
4. **Rebuild + restart** the app (`cd "$TP1_DIR" && docker compose up -d
   --build listings-api`), then run the Verify check.

### Expected behavior
- **Anonymous** `GET /sse` → **401** (the routes keep Basic auth; they are
  **not** opened to the world).
- `GET /sse` with a SELLER Basic credential → **200 `text/event-stream`**, and
  the **first event is `endpoint`** whose `data` is the message path
  (`/mcp/message?sessionId=…`).
- An `initialize` JSON-RPC call returns `serverInfo.name = listings-api`.
- `tools/list` is **empty** for now (Step 2 adds the two read tools).

### Verify
- [ ] `curl -fsS -o /dev/null -w '%{http_code}' http://localhost:8080/sse` (no Basic header) → **401**
- [ ] `curl -N -s -H "Authorization: Basic …" http://localhost:8080/sse` → `text/event-stream`; the first event is **`endpoint`** (its `data` is the message path)
- [ ] the `initialize` call returns `serverInfo.name = listings-api`
- [ ] `tools/list` → an **empty** list (no tools yet — Step 2)

### If you are stuck
- Paste the two dependencies + the `application.yml` block from **README → "0) Required base edit"** (the *what/why* is in **Appendix A**).
- The raw handshake + `tools/list` + `tools/call` are all in [`docs/verification.md` §1](docs/verification.md).
- `/sse` + `/mcp/message` are **defaults** of the Spring AI SSE auto-configuration — do **not** re-declare them as endpoints.
- If `GET /sse` returns **401**, that is **correct** — add the SELLER `Authorization: Basic …` header, do **not** open the route.

---

## Step 2 — Expose two read tools   CORE

**Outcome** — Two read tools (`get_listing`, `list_listings`) appear in
`tools/list`, each with a **non-trivial description** and an
**`inputSchema`**. They **delegate to TP1's `ListingService`** (no new DB
access) and their `tools/call` result **matches** the equivalent `curl GET`
field-for-field.

### Build it
1. **Create** `com.app.listings.ai.tools.ListingTools` — a `@Component` with
   **two public methods**, one per tool. Each method carries
   `@Tool(description="…")` and each parameter carries
   `@ToolParam(description="…")`. The body **delegates to TP1's
   `ListingService`** (no SQL, no duplicated rule).
2. **Create** `com.app.listings.ai.tools.AiToolsConfig` — a `@Configuration`
   declaring **one `ToolCallbackProvider`** bean:
   `MethodToolCallbackProvider.builder().toolObjects(listingTools).build()`.
3. **Rebuild + verify** — `tools/list` must show both tools (see Verify).
4. **Signatures flat & small** — `Long id`, `String category`,
   `BigDecimal maxPrice`; a tool is a *method*, not an endpoint (no
   `HttpServletRequest`, no HTTP DTO in the signature — you may **return** a
   DTO).

### Expected behavior
- `tools/list` returns **2 tools**: `get_listing` (arg: `id`) and
  `list_listings` (optional args: `category`, `maxPrice`).
- Each tool has a **`description`** an LLM can use to decide when to call it
  (e.g. *"Look up one listing by its id."*) and an `inputSchema` with each
  parameter described.
- `tools/call get_listing {id: 1}` returns **the same data** as
  `GET /listings/1` (field-for-field).
- `tools/call list_listings {category:"SPORTS", maxPrice:500}` returns **only**
  the matching rows (same set of ids as the REST endpoint).

### Verify
- [ ] `tools/list` shows **`get_listing`** and **`list_listings`**, each with a non-trivial `description` and an `inputSchema`
- [ ] `tools/call get_listing {id}` body **matches** `curl GET /listings/{id}` field-for-field
- [ ] `tools/call list_listings {category, maxPrice}` returns **the same set of ids** as the REST endpoint (side-by-side)
- [ ] `grep -rn "@Tool(" "$TP1_DIR/src/main/java"` — both tools are in `com.app.listings.ai.tools`
- [ ] `grep -n "listingService\.\|listingRepository\." "$TP1_DIR/src/main/java/com/app/listings/ai/tools/ListingTools.java"` — the tools **delegate to TP1's service** (no direct SQL bypass)

### If you are stuck
- `@Tool` and `@ToolParam` live in **`org.springframework.ai.tool.annotation`** (transitively pulled by the starter). Import directly — no extra dependency.
- The factory bean is `org.springframework.ai.tool.ToolCallbackProvider`; the builder is `org.springframework.ai.tool.method.MethodToolCallbackProvider.builder()`.
- **The description is the LLM's contract** — write it like a doc comment for a teammate, in the same tone as your Swagger tags.
- Full raw `curl`s for `tools/list` + `tools/call`: [`docs/verification.md` §2](docs/verification.md).

---

## Step 3 — Connect the AI client and "waouh" it   CORE

**Outcome** — An MCP client (Antigravity, Claude Desktop, or any SSE client) is
**registered against your server**, carries the **SELLER Basic header**, lists
**your two tools** (not a built-in), and answers plain-English read questions
with data that **matches a raw `curl GET /listings/{id}`**. This is the
**waouh** — the moment you see your Java run end-to-end from an AI client.

### Build it
1. **Grab** the client config from
   [`client/mcp-client.json`](client/mcp-client.json) (see
   [`client/README.md`](client/README.md) for the equivalent Claude Desktop /
   other-client layout). **Keep** the `Authorization: Basic …` header — the
   client authenticates as **one specific SELLER** (a TP1 account).
2. **Register** the server in the client's MCP settings (Antigravity: add the
   SSE server with the Basic header; Claude Desktop: the same JSON under
   `mcpServers`).
3. **Read-prompt only** for this step — writes are Step 4.
4. **Capture the transcript** (screenshot or markdown) next to a side-by-side
   `curl GET /listings/{id}` for the team README — that is the evidence.

### Expected behavior
- The client is **registered** against `http://localhost:8080/sse`, carries a
  SELLER Basic header, and the handshake succeeds.
- The client's tool list shows **your two tools** (not a built-in tool).
- A read prompt (e.g. *"what do you have in sports under 500?"*) results in
  **your Java running** and a natural-language answer.
- Every data value in the answer **matches** a side-by-side
  `curl GET /listings/{id}` — nothing is invented by the client.
- A client **without** the Basic header cannot even list tools (handshake 401).

### Verify (this step is the *waouh*)
- [ ] The client lists **both tools** by name **before** you ask any question
- [ ] A plain-English read question yields an answer whose values **equal** a raw `curl` (screenshot or transcript in the team README)
- [ ] A client configured **without** the Basic header → the client **cannot** list tools (handshake 401 / "unauthorized")
- [ ] The tool is **chosen** by the client, not hardcoded into the prompt (change the question; the client should still pick the right tool)

### If you are stuck
- Register your server in the client's MCP config — the JSON to paste is in [`client/mcp-client.json`](client/mcp-client.json).
- **Stay read-only** in this step — writes are Step 4.
- If the client shows **no tools**, go back to Step 1's handshake: the `endpoint` event must arrive before any `tools/list` is answered.
- To prove **your** Java runs (not a built-in), ask the client for a field you control (e.g. the exact price) and compare to a **side-by-side `curl GET /listings/{id}`** in a terminal.

---

## Step 4 — One write tool that respects ownership   CORE

**Outcome** — A **third tool**, `change_listing_price {id, newPrice}`, appears
in `tools/list`. It **delegates** to TP1's write path (owner check +
validation) and reports business failures **as a tool result string prefixed
with the HTTP status** — `404` for unknown, `403` for non-owner, `422 newPrice`
for a bad price — **never as a 500**, and the **DB is never mutated** on the
failure paths.

### Build it
1. **Add** a third method to `com.app.listings.ai.tools.ListingTools`:
   `change_listing_price(Long id, BigDecimal newPrice)`
   with `@Tool(description="…")` + `@ToolParam(...)`. The body:
   - resolves the listing via **TP1's `ListingService`** (so the TP1
     ownership check runs);
   - **validates** `newPrice > 0` (or delegates that to TP1);
   - wraps the call in `try { … } catch (IllegalStateException |
     AccessDeniedException ex) { return mapper.toToolResult(ex); }` —
     **never** re-throws as a 500;
   - delegates the **persistence** to TP1 (no direct SQL against `listing`).
2. **(optional)** Add a helper `com.app.listings.ai.tools.AiErrorMapper`
   with `String toToolResult(Throwable e)` — returns the status-prefixed
   string (`(404) …`, `(403) …`, `(422 newPrice)`) so the three cases
   live in **one place**.
3. **Rebuild + verify** — `tools/list` must now show **3** tools.
4. **Run** the four Verify cases (owner / non-owner / unknown / bad price).

> **Why the tool maps the exception, not the service** — the service keeps
> throwing the TP1 exception (`ResourceNotFoundException`, a `403`
> equivalent) — that is where TP1's **business rule** is enforced. The tool
> **catches** it and **converts** it to a **string the LLM can read**. The
> split keeps the *rule* in the *service* and the *AI-facing shape* in the
> *tool* (new in this TP).

### Expected behavior
The **full matrix** of this TP (one table, here, not duplicated per step):

| Caller (Basic cred) | `get_listing {id}` | `list_listings` | `change_listing_price` **own** | `change_listing_price` **other** | `change_listing_price` **unknown** | `change_listing_price newPrice<=0` |
|---|---|---|---|---|---|---|
| **Anonymous** | **401** | **401** | **401** | **401** | **401** | **401** |
| **SELLER (owner)** | 200 + body | 200 + list | **200/OK** + new price persisted in DB | — | **404** (as tool result) | **422 `newPrice`** (as tool result) |
| **SELLER (non-owner)** | 200 (public fields) | 200 | — | **403** (as tool result) | **404** (as tool result) | **422 `newPrice`** (as tool result) |

Additional rules:
- Owner write: the **new** price is persisted (verify with a fresh `curl
  GET /listings/{id}`).
- Non-owner write: the tool result **starts with** `403`; the DB is
  **unchanged** (TP1 ownership is **not** bypassed).
- Unknown `id`: tool result **starts with** `404` (e.g. `(404) Listing not
  found: id=99999`).
- `newPrice <= 0`: tool result **starts with** `422 newPrice`.

### Verify
- [ ] `tools/list` shows **3** tools now (`get_listing`, `list_listings`, `change_listing_price`)
- [ ] Owner write: **side-by-side** `curl GET /listings/{id}` before and after a `tools/call change_listing_price` shows the **new** price
- [ ] Non-owner write: the **tool result text** starts with `403 …`; the DB price is **unchanged**
- [ ] Unknown id: the **tool result text** starts with `404 …` (e.g. `(404) Listing not found: id=99999`)
- [ ] `newPrice = -5`: the **tool result text** contains `422` and `newPrice`
- [ ] `grep -n "AccessDeniedException\|ResourceNotFoundException" "$TP1_DIR/src/main/java/com/app/listings/ai/tools/ListingTools.java"` — both appear in a **catch clause**, not thrown as a raw 500

### If you are stuck
- In the tool body, wrap the call in `try { … } catch (IllegalStateException | AccessDeniedException ex) { return mapper.toToolResult(ex); }` — **do not rethrow** as a 500.
- TP1's likely exceptions: `AccessDeniedException` (Spring Security), a custom `ResourceNotFoundException`, or a `MethodArgumentNotValidException`-style `IllegalArgument`. Whatever TP1 throws — **reuse it**, do not invent a new one.
- For the `422 newPrice` case, throw a `jakarta.validation`-style exception **in the tool body** before delegating, and map it to `(422 newPrice)`.
- Full raw `curl`s for all four cases: [`docs/verification.md` §3](docs/verification.md).

---

## Step 5 — Flip roles: call Gemini from the API   BONUS

**Outcome** — A new REST endpoint `POST /listings/{id}/ai-description` (owner
only) returns a **200** with a 2-sentence description built from the listing's
**own DB fields**. **No key → 503** (clean JSON, no 500). The **key is read
only from the env** (`System.getenv("GEMINI_API_KEY")`), never in code/compose.

### Build it
1. **Add** the Gemini SDK to `pom.xml` — `com.google.genai:google-genai`
   (version block in **README §0**).
2. **Add** the `listings.ai` block to `application.yml` (`model:
   gemini-2.0-flash`, `timeout-ms: 10000`) — optional.
3. **Create** `com.app.listings.ai.GeminiClient` — a `@Component` that
   - reads the key **in the constructor** via `System.getenv("GEMINI_API_KEY")`;
     if the env is not set, **do not** build a `Client` (let it surface as
     `GeminiUnavailableException` on call);
   - exposes `String describe(String prompt)` calling
     `client.models().generateContent(model, prompt, null)`;
   - returns `candidates[0].content.parts[0].text`;
   - **throws** `GeminiUnavailableException` on **any** failure (missing key,
     timeout, non-2xx).
4. **Create** `com.app.listings.web.controller.AiListingController` — a
   `@RestController` for `POST /listings/{id}/ai-description`. Body:
   - **resolve** the listing (TP1) · **apply** the **ownership check** (TP1's
     `listingService` owner check — **do not** re-implement);
   - read `title` / `category` / `price`;
   - call `GeminiClient.describe(prompt)` where the **prompt embeds the exact
     DB values** as *facts* (see "If you are stuck");
   - return `{ "listingId": <n>, "description": "<text>" }`.
5. **Add** a `@ExceptionHandler(GeminiUnavailableException)` in TP1's
   `GlobalExceptionHandler` → **503** with `{ "error":
   "LLM_UNAVAILABLE", "message": "…" }`.
6. **Rebuild** (`cd "$TP1_DIR" && docker compose up -d --build`) **with the key
   in the env** (`export GEMINI_API_KEY=…` first) so the container sees it.
7. **Run** the Verify checks.

### Expected behavior
- Owner + key: **200** with a non-empty `description` that uses the listing's
  own `title` / `category` / `price` (no invented values).
- Non-owner → **403**; unknown id → **404** (both from **TP1**'s checks).
- Key **not in the env** → **503** + clean JSON — **never** a 500 stack trace.
- Model / network / timeout failure → **503** + JSON (same as above).
- **Bounded** timeout (≤ 10 s in the lab).
- **The key is nowhere** in `pom.xml`, `application.yml`, `docker-compose`,
  Java source, or the git history — it is **env only**.

### Verify
- [ ] `POST /listings/{id}/ai-description` (owner + key) → **200** + non-empty `description` using the listing's `title` / `category` / `price` (no invented values)
- [ ] `POST /listings/{id}/ai-description` (non-owner) → **403** · (unknown id) → **404**
- [ ] `POST /listings/{id}/ai-description` **without** `GEMINI_API_KEY` in the env → **503** + a JSON body (no 500 stack trace)
- [ ] `git -C "$TP1_DIR" grep -Rn -i "GEMINI_"` — only **`getenv("GEMINI_API_KEY")`** matches; **zero** literal keys in source / compose / `pom.xml`
- [ ] `docker inspect listings-api | grep -i GEMINI` returns **nothing** (the container is **not** baked with a key)

### If you are stuck
- The exact chain: `new com.google.genai.Client(apiKey)` → `client.models().generateContent("gemini-2.0-flash", prompt, null)` → `resp.getCandidates().get(0).getContent().getParts().get(0).getText()`.
- **The prompt keeps the model honest** — feed it the exact fields as *facts*: `"Write a 2-sentence, friendly, specific listing description. Use exactly the data: title=\"<title>\", category=<SPORTS|…>, price=<320> EUR. Do not invent a different price or category."`
- **Read the key only in the constructor** via `System.getenv("GEMINI_API_KEY")` — that is the **single** read site in the repo. If the env is **not** set, throw in the path that maps to 503.
- **Ownership:** reuse TP1's `listingService` owner check — do not re-implement it in the controller.
- Full raw `curl`s: [`docs/verification.md` §5](docs/verification.md).

---

## Step 6 — Persist the AI text and don't trust the prompt   BONUS

**Outcome** — The generated description is **persisted** on the `listing`
table (additive Liquibase file), **visible in the V2 payload** (any buyer can
read it — like `title`), **protected from prompt injection** (deny-list
sanitize, `WARN`-logged), and the **honesty guard** ensures the persisted text
never contradicts the DB. `POST /listings/{id}/ai-description/regenerate`
overwrites in place (owner only).

### Build it
1. **Add** a new Liquibase file `db/changelog/v1.1-ai.yaml` — **one** new
   changeSet (id `v1.1-ai`): `addColumn listing ai_description VARCHAR(2000)
   NULL`; **add one include** to `db.changelog-master.yaml`. Do **not** touch
   `v1.0-init.yaml` (TP1's rule).
2. **Extend** the `Listing` entity — **add** `String aiDescription` (same
   fluent-setter pattern as the other V2 fields).
3. **Extend** TP1's V2 `ListingResponseDto` mapping — **add** `aiDescription`
   to the **V2** payload, **absent** in V1 (TP1 visibility rule).
4. **Create** `com.app.listings.ai.AiDescriptionService` that, in order:
   - (a) **owns** the ownership check via TP1's `listingService`;
   - (b) **sanitizes** the `title` with a **deny-list** — a `Set<String>`
     of substrings (`"ignore previous"`, `"system:"`, `"you are now"`,
     case-insensitive) and a `lower(title).containsAnyOf(denyList)` check.
     On hit → **log WARN** and **do not** send the raw title to Gemini.
   - (c) calls `GeminiClient.describe(...)`;
   - (d) **asserts** the returned text still matches the **DB** price and
     category (`Category` enum name, normalized `BigDecimal` scale) — on
     mismatch, **do not** persist; throw a 422;
   - (e) **persists** via the TP1 `listingService`-side **update** path.
5. **Add** a `/ai-description/regenerate` route to `AiListingController`
   (same controller as Step 5) — non-owner → 403, unknown → 404 (TP1's
   rules).
6. **Rebuild + verify** — `GET /listings/{id}?version=V2` now returns
   `aiDescription`; `GET /listings/{id}` (no flag) does **not**.

### Expected behavior
- `GET /listings/{id}?version=V2` returns the stored text in **`aiDescription`**
  (public to any buyer — like `title`).
- `GET /listings/{id}` (no `?version=`) → the field is **absent** (TP1 V1
  rule).
- Prompt that tries `ignore previous instructions` in the title → the
  server's **sanitize** rejects it, logs `WARN`, and the Gemini call **does
  not** see the injected phrase (the model's response is unchanged from a
  clean title).
- `POST /listings/{id}/ai-description/regenerate` (owner only) regenerates +
  **persists the new text** in the same column; non-owner → 403, unknown →
  404.
- **Honesty guard:** if the generated text disagrees with the DB `price` or
  `category` (e.g. model says 999 € when the DB says 320 €), the call returns
  **422** and **nothing is written**.

### Verify
- [ ] `SELECT id FROM DATABASECHANGELOG` — **one** new row (the `v1.1-ai` changeSet)
- [ ] `curl GET /listings/{id}?version=V2` returns `aiDescription`; `curl GET /listings/{id}` (no flag) → the field is **absent**
- [ ] `POST /listings/{id}/ai-description/regenerate` (owner) → a fresh `GET /listings/{id}?version=V2` returns the **new** text, not the old one
- [ ] A listing whose `title` contains `"ignore previous instructions"`: `docker compose logs listings-api | grep WARN` shows the sanitize; the model's response is **not** influenced
- [ ] Two consecutive calls on the same listing return **two** coherent texts (same price/category); a **fresh** `regenerate` replaces the stored value in one place
- [ ] `grep -rn "containsAnyOf\|denyList\|SANITIZE" "$TP1_DIR/src"` — the sanitize code is **visible**, not just the call

### If you are stuck
- Liquibase additive: **one** `addColumn listing ai_description VARCHAR(2000) NULL`. New file `v1.1-ai.yaml`, new changeSet id `v1.1-ai`. Include in `db.changelog-master.yaml`. `v1.0-init.yaml` is **not** modified.
- **Sanitize:** a deny-list is enough for a lab. A `Set<String>` of substrings + `lower(title).containsAnyOf(denyList)` is the minimum. **log WARN** + **do not** send the raw title to Gemini.
- **Honesty guard:** compare *after* normalizing (`BigDecimal` scale, `Category` enum name) — on mismatch, **do not** persist, throw a 422.
- Full raw `curl`s (Steps 5 & 6 share §5): [`docs/verification.md` §5](docs/verification.md).

---

## Step 7 — Final Integration   DELIVERY

**Outcome** — The whole stack boots clean, the **3 tools** are discoverable
and respect ownership, the **waouh transcript** (read + write) is in the team
README side-by-side with the raw `curl`, and (if you did the bonuses) the
**key is env-only** and the **V2 payload** carries `aiDescription`. The
evaluator can `curl` a handshake, read the transcript, and **trust** it.

### Final checks
- [ ] `cd "$TP1_DIR" && docker compose up -d --build` → `listings-api` healthy; the **SSE handshake** works raw ([`docs/verification.md` §1](docs/verification.md))
- [ ] **3 tools** in `tools/list` (`get_listing`, `list_listings`, `change_listing_price`) — each with a **description** and an **`inputSchema`**
- [ ] The **AI client** is registered and the **waouh transcript** (read + write, side-by-side with the raw `curl`) is in the **team README**
- [ ] **Bonuses 5-6** (if done): `GEMINI_API_KEY` is **env var only** (never in code, compose, or repo); a raw `curl POST /listings/{id}/ai-description` works; `GET /listings/{id}?version=V2` carries `aiDescription`
- [ ] `cd "$TP1_DIR" && mvn clean verify` → **green** (TP1's existing smoke test still passes; the new code does **not** break existing tests)
- [ ] The **team README** carries a short *decision log*: why Basic auth on MCP · why Spring AI ≠ Gemini call · why the tool maps 403/404 into a **tool result** (not a 500) · why the key is env-only
- [ ] **Oral** (10 min + 5 Q&A): walk **one read + one write** from the client transcript → the raw `tools/call` → your Java → the DB (compare the `GET`)

### If you are stuck
- The **team README** is the **one place** the binôme writes **what** they
  decided and **why** — it is **not** a copy of the exercise statement. The
  *decision log* is your **oral-prep** (the evaluator's "why?" is answered by
  5-6 short paragraphs, not by `exercise.md`).
- The **smoke test to add in the TP1 repo** is the lowest bar: the **Spring
  context boots** and the `@Tool` beans are **registered**. It **must not**
  require the AI client (it is a boot test), and it **must not** call Gemini
  (no network). Add it to the existing TP1 smoke test, **do not** write a
  new suite from scratch.

---

## Deliverables

| Item | Detail |
|---|---|
| **Pair's TP1 repo** (`../springboot/`) | `+2 deps` (Appendix A); `+ spring.ai.mcp.server` block; `com.app.listings.ai` package (2-3 tools + a `ToolCallbackProvider` bean [+ `GeminiClient`, `AiDescriptionService`, `AiListingController` if Bonuses]); Liquibase `v1.1-ai.yaml` + include in the changelog master (if Bonus 6); `README.md` updated with the decision log |
| **This repo** (`IA/`) | `exercise.md` (this file), `README.md` (operational), `client/mcp-client.json` (client config), `docs/verification.md` (the copy-paste `curl`s). **No Java in this repo** — the pair's `../springboot` is the only place code is committed (single owner, one repo per artifact) |
| **AI-client transcript** (read + write) | **In the pair's README**, as a screenshot or markdown snippet — the *waouh* evidence, side-by-side with a raw `curl GET /listings/{id}` |
| **`mvn clean verify` in `../springboot`** | **Green** |
| **Oral** | 10 min + 5 min Q&A |

> **Do not commit** into the pair's TP1 repo (or this one): the AI key (env
> only), the raw model response body from a real Gemini call (privacy +
> rate-limit), any `.env` file — the anti-patterns in this file list them
> explicitly.

---

## Appendix A — the two dependencies + the application.yml block

The **code you paste is not in this file** — by design, it lives in the
**operational** [`README.md` § "0) Required base edit"](README.md), exactly
where the Keycloak and Monitoring TPs put their base edits. This appendix is
the **reference** for *what* the two additions are and *why*.

| Addition | Coordinates / key | Why |
|---|---|---|
| MCP server starter | `org.springframework.ai:spring-ai-starter-mcp-server-webmvc:2.0.1` | exposes the MCP **server** over SSE (`/sse`, `/mcp/message`). Matches TP1's **Boot 4.1.1** (Spring AI **2.0** is the Boot-4 line). |
| Gemini SDK | `com.google.genai:google-genai:1.78.0` | Steps 5-6 **call** the model. Spring AI does **not** ship a Gemini starter — this is a separate library. |
| `application.yml` | `spring.ai.mcp.server.name/protocol=SSE` (+ `listings.ai.model/timeout-ms`) | `name` shows in the handshake `serverInfo.name`; `protocol: SSE` enables the SSE transport. |

> **One owner per fact:** `pom.xml` + `application.yml` code → **README**; this
> appendix explains *which* artifact and *which* key and *why*. No new container,
> network, or port — unlike the Keycloak / Monitoring TPs, this TP does **not**
> add a `docker-compose.yml`.


## Appendix B — the AI client config

Already shipped as a file: [`client/mcp-client.json`](client/mcp-client.json).
If your MCP-over-SSE client uses a different JSON shape
(Claude Desktop / `claude_desktop_config.json`, for example), keep the **same
three facts**:

- `type: "sse"` (or the equivalent),
- `url: "http://localhost:8080/sse"`,
- `Authorization: Basic <base64(email:password)>` for a TP1 **SELLER** account
  (the client acts as that seller — **that is the identity inside the write
  tool**).

## Appendix C — what is NOT committed

- **The AI key** (Gemini or any other) — env only, never in the repo or the
  git history. (Anti-pattern, see *What NOT to do*.)
- **The AI-client transcript** (the "waouh") — in the **pair's TP1 README**
  (screenshot / markdown), **not** in this `IA/` repo.
- **No solution Java in this repo** — the pair's TP1 repo is the **only**
  place the code is committed (single owner). This repo contains **statement
  + client config + verification curls**.
- **No reference solution for Steps 5-6** shipped in this repo.

## Appendix D — Useful links (versions pinned)

- **Spring AI 2.0.1** (Boot-4 line): <https://github.com/spring-projects/spring-ai>
  — artifact `spring-ai-starter-mcp-server-webmvc:2.0.1`.
- **MCP (the protocol)**: <https://modelcontextprotocol.io/docs/learn/server-concepts>
- **Gemini (free tier via Google AI Studio)**: <https://aistudio.google.com>
- **`google-genai` 1.78 SDK**: <https://github.com/googleapis/java-genai>

> **TP rule (final repeat):** no `TODO`, no stub, no secret in code, no `500`
> where a `4xx` is correct, and the **AI client is a viewer of your API — the
> code is what you are graded on**.





