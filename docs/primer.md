# TP IA primer — MCP, Spring AI, the Gemini SDK

Background for [`../exercise.md`](../exercise.md): what MCP is, the two
libraries of the TP, and what not to do. Read it once before Step 1.

## What is MCP?

**MCP is an open protocol that lets an AI application call your program through
standard "tools".** Your Spring Boot app is the **server**; an AI app
(**Antigravity**, or any other MCP client) is the **client**.

| Concept | What it is here |
|---|---|
| **Server** | your `listings-api` (TP1) exposing MCP over **Streamable HTTP** |
| **Tool** | one Java method (`get_listing`, `list_listings`, `change_listing_price`) an LLM can call |
| **Transport** | **Streamable HTTP**: one endpoint, `POST /mcp`, carrying JSON-RPC (`initialize`, `tools/list`, `tools/call`); the session id travels in the `Mcp-Session-Id` header |
| **Client** | an AI app that lists your tools and calls them in natural language |

You are **not** writing a chatbot and you are **not** hosting an LLM. You are
**exposing your existing business code as callable tools** (Steps 1-4) and, in
the bonuses, **calling an LLM from your code** (Steps 5-6). That split is the
whole idea of the day.

## The two libraries (versions pinned — do not guess)

| Library | Artifact · version | Used by |
|---|---|---|
| Spring AI (Boot 4 line) | `org.springframework.ai:spring-ai-starter-mcp-server-webmvc:**2.0.1**` | Steps 1-4 (the MCP server + tools) |
| Google Gen AI SDK | `com.google.genai:google-genai:**1.78.0**` | Steps 5-6 (calling Gemini) |

> **Why two different things:** Spring AI = **exposing tools** to an AI
> client. The **Gemini SDK** = **asking the model** a question from your code.
> Keep the two separate in your head. Spring AI **2.0.1** targets the Boot 4
> line and works with TP1's Boot **4.1.1**.

## What NOT to do

- **Never** put an API key (Gemini or any) in code, `application.yml`, the repo, or logs.
- **Never** open the MCP route to the world — it **keeps Basic auth**.
- **Never** bypass TP1 ownership from a tool — a **403** for a non-owner is the *correct* outcome.
- **Never** let generated text overrule DB facts (price, category) — the model
  *describes*, the DB *decides*.
