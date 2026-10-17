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
per-step `Acceptance checklist` **is** the "definition of done". Every `curl`
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
(Gemini Desktop / Antigravity or any MCP client) is the **client**.

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

## Before you start — the two tools

New to MCP or to Spring AI? This ~3-minute model makes the steps below click.

### The stack, in one picture

```
   AI client (Gemini Desktop / Antigravity)              [Step 3]
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

Work the core **in order**; each step's checklist is your "definition of done".
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

## Step 1 — Bring up the MCP server `CORE`

> **User story** — *As a **developer**, I want my TP1 API to **be an MCP server
> over SSE**, so that an **AI client can discover and call my code** in the
> following steps.*

### Behavior

| # | Requirement |
|---|---|
| 1.1 | Adding one dependency + a `spring.ai.mcp.server` block makes the app expose the **SSE endpoints** (`GET /sse`, `POST /mcp/message`) on port 8080. |
| 1.2 | **Anonymous** `GET /sse` → **401** (the routes keep Basic auth — they are **not** opened to the world). |
| 1.3 | `GET /sse` with a **SELLER** Basic credential → **200 `text/event-stream`**, and the **first SSE event is the `endpoint`** event whose `data` is the message path (`/mcp/message?sessionId=…`). |
| 1.4 | An `initialize` JSON-RPC call returns a `result` with `serverInfo.name = listings-api` (or your custom `spring.ai.mcp.server.name`). |

### Changes to make in TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| `pom.xml` | (no `spring-ai`) | **add** `org.springframework.ai:spring-ai-starter-mcp-server-webmvc:2.0.1` (Appendix A) |
| `application.yml` | (none) | **add** the `spring.ai.mcp.server` block — `name: listings-api`, `protocol: SSE` (Appendix A) |
| `SecurityConfig` | permits `/auth/register`, `/actuator/**`, `/v3/api-docs/**`, `/swagger-ui/**`; `anyRequest().authenticated()` | **no change** — `/sse` and `/mcp/message` fall under `anyRequest().authenticated()`, so they **keep Basic auth** (that is the point) |

### Hints

- Both the `pom.xml` dependency and the `application.yml` block are in
  **Appendix A** — this is the **only** prescribed config for the server.
- Verify without the AI client, raw `curl`: [`docs/verification.md` §1](docs/verification.md).
- SSE endpoints are **defaults** of the Spring AI SSE autoconfiguration
  (`/sse` + `/mcp/message`); you do not need to declare them.
- If `GET /sse` returns **401**, that is **correct** — add a SELLER
  `Authorization: Basic …` header (see `docs/verification.md`), do **not**
  permit the route.

### Acceptance checklist

- [ ] `GET http://localhost:8080/sse` **without** a Basic credential → **401**
- [ ] `GET http://localhost:8080/sse` **with** a SELLER Basic credential → `text/event-stream`, and the first emitted event is the **`endpoint`** (whose `data` is the message path)
- [ ] An `initialize` call returns `result.serverInfo.name = listings-api`
- [ ] `tools/list` currently returns an **empty** tool list (you have no tools yet — Step 2 adds them)

---

## Step 2 — Expose two read tools `CORE`

> **User stories** —
>
> - *As a **Seller / Visitor**, I want to query my listings **in natural
>   language** so that I can **find and inspect items** without writing `curl`.*
> - *As a **platform developer**, I want my existing service methods to be
>   **callable as MCP tools**, so that I **do not rewrite the business logic**
>   for the AI.*

### Behavior

| # | Requirement |
|---|---|
| 2.1 | Two tools exist and appear in `tools/list`: **`get_listing`** (one arg: `id`; returns the listing as JSON) and **`list_listings`** (optional args: `category`, `maxPrice`; returns the matching listings). |
| 2.2 | Each tool carries a **`description`** an LLM can use to decide **when to call it** (e.g. *"Look up one listing by its id."*), and an **`inputSchema`** that lists each parameter with a description. |
| 2.3 | The tools **delegate to TP1's `ListingService` / `ListingRepository`** — no new DB access, no duplicated business rules. |
| 2.4 | `tools/call get_listing {id: 1}` returns the **same data** as `GET /listings/1` (field-for-field). |
| 2.5 | `tools/call list_listings {category:"SPORTS", maxPrice:500}` returns **only** the matching rows (same set of ids as the REST endpoint). |

### Changes to make in TP1 (`../springboot`)

| What | From (TP1) | To (this step) |
|---|---|---|
| (new) `com.app.listings.ai.tools.ListingTools` | — | a `@Component` with **two public methods**: one per tool, each carrying **`@Tool(description="…")`** on the method and **`@ToolParam(description="…")`** on the parameters. The body **delegates to TP1's `ListingService`** |
| (new) `com.app.listings.ai.tools.AiToolsConfig` | — | a `@Configuration` that declares **one `ToolCallbackProvider` bean** built with `MethodToolCallbackProvider.builder().toolObjects(listingTools).build()` |

### Hints

- `@Tool` and `@ToolParam` live in **`org.springframework.ai.tool.annotation`**
  (the `spring-ai-model` jar, pulled transitively by the starter). Import
  directly — **no extra dependency**.
- The factory bean type is **`org.springframework.ai.tool.ToolCallbackProvider`**;
  the builder is **`org.springframework.ai.tool.method.MethodToolCallbackProvider.builder()`**.
- Keep tool signatures **flat and small** (`Long id`, `String category`,
  `BigDecimal maxPrice`) — a tool is a *method*, not an *endpoint*. No
  `HttpServletRequest`, no `HttpServletResponse`, no DTOs required by the tool
  signature (you can still **return** a DTO as the payload).
- The **description is the LLM's contract** — write it like a doc comment for a
  teammate, in the same tone as your Swagger tags.
- Verify: [`docs/verification.md` §2](docs/verification.md) — `tools/list` + two
  hand-written `tools/call`s you can paste into the SSE session.

### Acceptance checklist

- [ ] `tools/list` shows **`get_listing`** and **`list_listings`**, each with a non-trivial `description` and an `inputSchema`
- [ ] `tools/call get_listing {id}` body **matches** `curl GET /listings/{id}` field-for-field
- [ ] `tools/call list_listings {category, maxPrice}` returns **the same set of ids** as the REST endpoint (side-by-side)
- [ ] `grep -rn "@Tool(" ../springboot/src/main/java` — both tools are in `com.app.listings.ai.tools`
- [ ] grep for `listingService\.\|listingRepository\.` inside the tool class — the tools **delegate to TP1's service** (no direct SQL bypass)

---

## Step 3 — Connect the AI client and "waouh" it `CORE`

> **User story** — *As a **binôme**, I want an **AI client to call my tools in
> natural language** and answer in plain English, so that I **see the whole MCP
> round-trip working on my own code** before I trust it with writes.*

### Behavior

| # | Requirement |
|---|---|
| 3.1 | An MCP-capable client (Gemini Desktop / Antigravity, Claude Desktop, or any **SSE** MCP client) is **registered** against `http://localhost:8080/sse` and **carries the SELLER Basic header**. |
| 3.2 | Once registered, the client shows **your 2 tools** in its tool list (not a built-in). |
| 3.3 | A **read** prompt (e.g. *"what do you have in sports under 500?"*, *"what is listing 1?"*) results in **your Java running** and the client **rendering the data** in natural language. |
| 3.4 | The client's reported data **matches** `curl GET /listings/{id}` (no value is invented by the client). |
| 3.5 | A client registered **without** the Basic header **cannot** list or call tools — the handshake fails with a 401. |

### Client config to paste

Copy the snippet from [`client/mcp-client.json`](client/mcp-client.json). It
carries the **Basic header** (`alice@test` / `secret123`, one of the TP1
sellers). Do **not** remove that header — that would undo Step 1.2.

### Hints

- Register the server in the client's MCP settings (paste the JSON above).
- **Stay on read prompts for this step** — writes are Step 4.
- If the client shows **no tools**, go back to Step 1's handshake first: the
  `endpoint` event must arrive before any `tools/list` can be answered.
- To prove your Java runs (not a built-in), ask the client for a field you
  control (e.g. the exact price) and compare to a **side-by-side `curl GET
  /listings/{id}`** in a terminal.

### Acceptance checklist (this step is the *waouh*)

- [ ] The client lists **both tools** by name before you ask a question
- [ ] A plain-English read question yields an answer whose values **equal** a raw `curl` (screenshot or transcript in the team README)
- [ ] A client configured **without** the Basic header → the client **cannot** list tools (handshake fails with 401 / "unauthorized")
- [ ] The tool is *chosen* by the client, not hardcoded into the prompt (change the question and the client should still pick the right one)

---

## Step 4 — One write tool that respects ownership `CORE`

> **User story** — *As a **platform developer**, I want the LLM to be able to
> **call a write tool** but with **the same rules as the API**: `404` if
> unknown, `403` if not mine, `422` if the argument is invalid — so that an
> LLM **cannot do more than I allow**.*

### Behavior (the *one* full behavior matrix for this TP — put it here once, not per step)

| Caller (Basic cred) | `get_listing {id}` | `list_listings` | `change_listing_price` **own** | `change_listing_price` **other** | `change_listing_price` **unknown** | `change_listing_price newPrice<=0` |
|---|---|---|---|---|---|---|
| **Anonymous** | **401** | **401** | **401** | **401** | **401** | **401** |
| **SELLER (owner)** | 200 + body | 200 + list | **200/OK** + new price persisted in DB | — | **404** (as tool result) | **422 `newPrice`** (as tool result) |
| **SELLER (non-owner)** | 200 (public fields) | 200 | — | **403** (as tool result) | **404** (as tool result) | **422 `newPrice`** (as tool result) |

### Requirements

| # | Requirement |
|---|---|
| 4.1 | A **third tool** appears in `tools/list`: **`change_listing_price`**, args `{id, newPrice}`. |
| 4.2 | Owner: the tool succeeds and a fresh `GET /listings/{id}` returns the **new** price. |
| 4.3 | Non-owner: the tool returns a **result whose text starts with `403`**, and the DB is **unchanged**. TP1 ownership is **not** bypassed. |
| 4.4 | Unknown `id`: the tool returns a **result whose text starts with `404`**. |
| 4.5 | `newPrice <= 0`: the tool returns a **result whose text starts with `422 newPrice`**. |
| 4.6 | The tool **delegates** to TP1's `ListingService.update(...)` (or an equivalent method) — **not** to raw SQL against `listing`. |

### Changes to make in TP1 (`../springboot`)

| What | Before (TP1) | After (this step) |
|---|---|---|
| `ListingTools` (added in Step 2) | 2 read tools | **+ 1 write tool** `change_listing_price(Long id, BigDecimal newPrice)` that: resolves the listing via TP1, applies TP1's **ownership check** (via `listingService`), validates `newPrice > 0`, delegates the persistence, and maps thrown TP1 exceptions into a **tool result string** (`(404) …`, `(403) …`, `(422 newPrice)`) |
| (optional) `com.app.listings.ai.tools.AiErrorMapper` | — | a small helper: `String toToolResult(Throwable e)` that returns the status-prefixed string. Keeps the three cases in one place. |

> **Why the tool, not the service, maps the exception:** the **service** still
> throws the TP1 exception (`ResourceNotFoundException` / a `403` equivalent)
> — that is where TP1's business rule is enforced. The **tool** catches it and
> converts it into a **string the LLM can read**. That two-step split keeps the
> *business rule* in the *service* (same as TP1) and the *AI-facing shape* in
> the *tool* (new in this TP).

### Hints

- In the tool body, wrap the call in `try { … } catch (IllegalStateException | AccessDeniedException ex) { return mapper.toToolResult(ex); }` — **do not rethrow** as a 500.
- TP1's likely exceptions: `AccessDeniedException` (Spring Security), a custom `ResourceNotFoundException`, or a `MethodArgumentNotValidException`-style `IllegalArgument`. Whatever TP1 Step 3 already throws — **reuse it**, do not invent a new type.
- For a `422` response from a missing/invalid `newPrice`, throw a `jakarta.validation`-style exception **in the tool body** before delegating, and map it to `(422 newPrice)`.
- `docs/verification.md` §3 walks all three cases (valid / non-owner / unknown / bad price) with copy-paste `curl`s.

### Acceptance checklist

- [ ] `tools/list` shows **3** tools now (`get_listing`, `list_listings`, `change_listing_price`)
- [ ] Owner write: **side-by-side** `curl GET /listings/{id}` before and after a `tools/call change_listing_price` shows the **new** price
- [ ] Non-owner write: the **tool result text** starts with `403 …`; the DB price is **unchanged**
- [ ] Unknown id: the **tool result text** starts with `404 …` (e.g. `(404) Listing not found: id=99999`)
- [ ] `newPrice = -5`: the **tool result text** contains `422` and `newPrice`
- [ ] `grep -n "AccessDeniedException\|ResourceNotFoundException" ListingTools.java` — both appear in a **catch clause**, not thrown as a raw 500

---

## Step 5 — Flip roles: call Gemini from the API `BONUS`

> **User story** — *As a **Seller**, I want to click one button and get a
> **coherent 2-sentence description** for a listing based on its own `title`,
> `category` and `price`, so that **my listing looks professional** without
> me writing the copy.*

### Behavior

| # | Requirement |
|---|---|
| 5.1 | A new REST endpoint **`POST /listings/{id}/ai-description`** (auth = SELLER **owner**, same rules as TP1's PUT) returns `200` with body `{ "listingId": <n>, "description": "<text>" }`. |
| 5.2 | The **prompt** is built from the listing's **own DB fields** (`title`, `category`, `price`). The model may polish wording; it **must not invent** a different price or category. |
| 5.3 | **`GEMINI_API_KEY` not in the env** → the endpoint returns **503** with a JSON body: `{ "error": "NO_API_KEY", "message": "GEMINI_API_KEY is not set" }`. **The raw key/value never appears in a log line.** |
| 5.4 | Model/network failure (timeout, non-2xx) → the endpoint returns **503** with `{ "error": "LLM_UNAVAILABLE", "message": "…" }`, **not** a 500 stack trace. |
| 5.5 | The Gemini call has a **bounded** timeout (≤ 10 s lab is fine; use whatever the SDK gives you). |
| 5.6 | `GEMINI_API_KEY` is **nowhere** in `pom.xml`, `application.yml`, `docker-compose`, Java source, or the git history. It is read **from the env** at runtime (`System.getenv("GEMINI_API_KEY")`). |

### Changes to make in TP1 (`../springboot`)

| What | Before | After |
|---|---|---|
| `pom.xml` | — | **+ `com.google.genai:google-genai:1.78.0`** (Appendix A) |
| `application.yml` | — | **+** a `listings.ai` block (optional model name + timeout): `model: gemini-2.0-flash`, `timeout-ms: 10000` |
| (new) `com.app.listings.ai.GeminiClient` | — | a `@Component` that (a) reads `System.getenv("GEMINI_API_KEY")` **in the constructor**, (b) constructs `new com.google.genai.Client(apiKey)`, (c) exposes `String describe(String prompt)` calling `client.models().generateContent("<model>", prompt, null)` and returning `response.getCandidates().get(0).getContent().getParts().get(0).getText()`, (d) throws `GeminiUnavailableException` on any failure (missing key, timeout, non-2xx) |
| (new) `com.app.listings.web.controller.AiListingController` | — | a `@RestController` for **`POST /listings/{id}/ai-description`**; body: resolve the listing (TP1), apply the **ownership check** (TP1), read the `title`/`category`/`price`, call `GeminiClient.describe(...)`, return the JSON shape above |
| `GlobalExceptionHandler` (TP1) | (handles TP1 cases) | **+** a `@ExceptionHandler(GeminiUnavailableException)` returning **503** with the JSON shape above |

### Hints

- **The client (exact):** `new com.google.genai.Client(apiKey)` →
  `client.models().generateContent("gemini-2.0-flash", prompt, null)` →
  `resp.getCandidates().get(0).getContent().getParts().get(0).getText()`.
- **The prompt keeps the model honest** — feed it the **exact** fields as *facts*:
  `"Write a 2-sentence, friendly, specific listing description. Use exactly the
  data: title=\"<title>\", category=<SPORTS|…>, price=<320> EUR. Do not invent a
  different price or category."`
- **The key (anti-pattern):** read it *only* in the constructor via
  `System.getenv("GEMINI_API_KEY")`. If the env is **not** set, **do not** create
  the `Client` at all (throw `IllegalStateException` that maps to 503 on call).
  This is the *single* place the key is read in the repo.
- **Timeout** (5.5): the SDK honors the JDK's `HttpClient` timeout; for a lab,
  10 s is enough. The *behavior* (bounded call + 503 on timeout) is the
  acceptance item, not the exact API.
- **Ownership** (5.1): **reuse** TP1's `listingService` owner check — do not
  re-implement it in the controller.

### Acceptance checklist

- [ ] `POST /listings/{id}/ai-description` (owner + key) → **200** + non-empty `description` that uses the listing's `title` / `category` / `price` (no invented values)
- [ ] `POST /listings/{id}/ai-description` (non-owner) → **403** · (unknown id) → **404**
- [ ] `POST /listings/{id}/ai-description` **without** the key in the env → **503** + a JSON body (never a 500 stack trace)
- [ ] `git -C ../springboot grep -Rn -i "GEMINI_"` — only **`getenv("GEMINI_API_KEY")`** matches; zero literal keys in source / compose / `pom.xml`
- [ ] `docker inspect listings-api | grep -i GEMINI` returns **nothing** (the container is **not** baked with a key)

---

## Step 6 — Persist the AI text and don't trust the prompt `BONUS`

> **User story** — *As a **platform operator**, I want the generated text to be
> **stored with the listing**, **visible in `GET /listings/{id}`** (in V2,
> like `title`), **and protected from prompt injection / hallucinated price
> values**, so that **the buyer sees the same text the seller approved**.*

### Behavior

| # | Requirement |
|---|---|
| 6.1 | The generated description is **persisted** on the `listing` table via a **new additive** Liquibase changeSet (a **new file**, not an edit of `v1.0-init.yaml`). |
| 6.2 | `GET /listings/{id}?version=V2` returns the stored text in a **new field** — public to **any** buyer, like `title`. V1 (or no `?version=`) does **not** expose it (TP1's V1/V2 visibility rule). |
| 6.3 | **Prompt-injection hardening:** before sending a listing's user-supplied `title` to Gemini, the server **sanitizes / rejects** text that looks like an instruction (a short deny-list is enough for a lab — it does **not** have to be a detector). The rejection is **logged** (`WARN`). |
| 6.4 | **Regenerate:** `POST /listings/{id}/ai-description/regenerate` (owner only) re-generates and **persists the new text** in the same column. Non-owner → 403 · unknown → 404 (same rules as Step 5). |
| 6.5 | **Honesty guard:** the **persisted** text carries the **same price and category** as the DB. If it does not, the call returns **422** and **nothing is written**. |

### Changes to make in TP1 (`../springboot`)

| What | Before (TP1) | After (this step) |
|---|---|---|
| `db/changelog/` (TP1) | `v1.0-init.yaml` (+ `v1.0.1-identity.yaml`) | **+ `v1.1-ai.yaml`** — a single `addColumn` changeSet on `listing`: `ai_description`, `varchar(2000) NULL`. **Add one include** to `db.changelog-master.yaml`. |
| `Listing` entity | id + (TP1) V1/V2 fields | **+** `String aiDescription` — same fluent-setter pattern the skeleton uses |
| `ListingResponseDto` / TP1's V2 mapping | existing fields | **+** `aiDescription` in the **V2** payload, **absent** in V1 (TP1 rule — see `exercise.md` TP1 Step 6) |
| (new) `com.app.listings.ai.AiDescriptionService` | — | (a) ownership check via `listingService`, (b) **sanitize** the `title` (a **deny-list**, e.g. `ignore previous`, `system:`, `you are now`, case-insensitive), (c) call `GeminiClient.describe(...)`, (d) **assert** price/category equality with the DB, (e) `listingService`-side persist (additive `update`) |
| `AiListingController` (Step 5) | `/ai-description` | **+ `/ai-description/regenerate`** (same controller); the **main** `/ai-description` endpoint is **left** to only *return* the text (Step 5 is "return-only"), Step 6 adds the `regenerate` route |

### Hints

- Liquibase additive: one `addColumn listing ai_description VARCHAR(2000) NULL`. New file `v1.1-ai.yaml`, new changeSet `id: "v1.1-ai"`. Include it in `db.changelog-master.yaml`. **`v1.0-init.yaml` is not modified** (TP1's rule).
- **Sanitize (6.3):** a **deny-list** is enough for a lab. A `Set<String>` of
  substrings and a `lower(title).containsAnyOf(denyList)` check is the
  minimum. Log WARN if rejected and **do not send the raw title** to Gemini.
- **Honesty guard (6.5):** compare the model's returned text to the DB values
  *after* normalizing (`BigDecimal` scale, `Category` enum name) — if a mismatch
  is detected, **do not** persist, throw a 422 (or a custom `IllegalArgument`).
- `docs/verification.md` covers Step 5 but **not** Step 6 (bonus). Run Step 6 with
  the same *curl + side-by-side `GET`* pattern.

### Acceptance checklist

- [ ] `SELECT id FROM DATABASECHANGELOG` — **one** new row (the `v1.1-ai` changeSet)
- [ ] `curl GET /listings/{id}?version=V2` returns `aiDescription`; `curl GET /listings/{id}` (no `?version=`) **does not** (TP1 V2 rule)
- [ ] `POST /listings/{id}/ai-description/regenerate` (owner) — a fresh `GET /listings/{id}?version=V2` returns the **new** text, not the old one
- [ ] A listing whose `title` contains `"ignore previous instructions"`: the model response is **not** influenced by the injected phrase (side-by-side `GET /listings/{id}` + log `WARN` visible in `docker compose logs listings-api | grep WARN` shows the sanitize)
- [ ] Two calls in a row on the same listing return **two** coherent texts (same price/category), and a **fresh** `regenerate` replaces the stored value in one place
- [ ] `grep -rn "containsAnyOf\|denyList\|SANITIZE" ../springboot/src` — the sanitize code is visible, not just the call

---

## Step 7 — Delivery

| # | Requirement |
|---|---|
| 7.1 | `docker compose up -d` in `../springboot` → `listings-api` healthy; the **SSE handshake** works raw (`docs/verification.md` §1). |
| 7.2 | **3 tools** in `tools/list` (get, list, change) — each with a **description** and an **`inputSchema`**. |
| 7.3 | The **AI client** is registered (the `client/mcp-client.json` config, or the student's equivalent) and the **waouh transcript** (read + write, side-by-side with the raw `curl`) is in the **team README**. |
| 7.4 | If Bonuses 5-6 are done: `GEMINI_API_KEY` is an **env var only** (never in code, compose, or the repo); a raw `curl POST /listings/{id}/ai-description` works; `GET /listings/{id}?version=V2` carries `aiDescription`. |
| 7.5 | `mvn clean verify` in `../springboot` is **green** (TP1's starter test still passes; the new code does **not** break existing tests). |
| 7.6 | The **team README** (in `../springboot/README.md`, the TP1 repo): the **two** added dependencies (with exact versions), the `application.yml` MCP block, the **client config used**, and a short *decision log* (why Basic auth on MCP, why Spring AI ≠ Gemini call, why the tool maps 403/404 into a **tool result** rather than a 500, why the key is env-only). |
| 7.7 | Oral (10 min + 5 Q&A): **walk one read + one write** from the AI client transcript → the raw `tools/call` → your Java → the DB (compare the `GET`). |

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

**The `pom.xml` additions** (both *additive*; do not remove the TP1 deps):

```xml
<!-- MCP server over SSE (Spring AI 2.0.1, Boot-4 line — matches TP1's Boot 4.1.1) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
    <version>2.0.1</version>
</dependency>

<!-- Google Gemini SDK (Steps 5-6; the "free tier" is Google AI Studio) -->
<dependency>
    <groupId>com.google.genai</groupId>
    <artifactId>google-genai</artifactId>
    <version>1.78.0</version>
</dependency>
```

**The `application.yml` block** to add in `../springboot/src/main/resources/application.yml`:

```yaml
spring:
  ai:
    mcp:
      server:
        name: listings-api          # used by serverInfo.name in the handshake
        protocol: SSE               # enables the SSE autoconfig (/sse + /mcp/message)
# Bonus (Steps 5-6):
listings:
  ai:
    model: gemini-2.0-flash         # any gemini-*-flash on your free key
    timeout-ms: 10000
```

> That is the **entire** config surface for this TP — the **server** (Step 1)
> and the **client call** (Steps 5-6). No new container, no new network, no new
> port.

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





