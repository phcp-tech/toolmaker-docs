# Toolmaker Agent

<p align="center"><strong><em>We don't care what you build. We care how you build it.</em></strong></p>

An AI-augmented requirements and system-design workbench: a Go backend with an embedded React UI for managing **Products, Features, Requirements, and UML/4+1-view system-design diagrams**, paired with an LLM-driven conversational agent and a Model Context Protocol (MCP) server so both humans and coding agents (e.g. Claude Code) can drive the same data model.

## What this is

Toolmaker Agent is a single self-contained executable: a Go REST API with a layered architecture, SQLite storage, and the compiled React frontend embedded directly into the binary. Open one port and you get the full workbench — no separate frontend deployment, no external database to provision.

On top of the plain CRUD workbench, it adds three AI-native layers on the same data:

- **Tool Calling** — a chat panel backed by [`trpc-agent-go`](https://github.com/trpc-group/trpc-agent-go) lets a user talk a Product/Feature/Requirement/UML diagram into existence instead of filling out a form. Writes go through a **propose → confirm** flow (the model proposes the action, a human confirms it before it executes); reads execute immediately, since they're side-effect-free.
- **MCP Server** — a [Model Context Protocol](https://modelcontextprotocol.io/) endpoint (official [`modelcontextprotocol/go-sdk`](https://github.com/modelcontextprotocol/go-sdk)) exposes that same CRUD capability as 20 direct tools, so a coding agent like Claude Code can manage requirements straight from the terminal — no confirmation step, since the caller is a developer already driving the agent.
- **RAG / semantic search** — every Product/Feature/Requirement/UML is embedded and indexed as it's written, so the chat agent, MCP clients, and a REST endpoint can all find entities by *meaning* ("find requirements similar to X"), not just exact keyword/OID matches.

## Key features

- **Product / Feature / Requirement / UML CRUD**, each with optimistic concurrency and full-row responses on create/update.

- **4+1 system-design views** rendered as [Mermaid](https://mermaid.js.org/) diagrams (flowcharts, sequence, C4/architecture, state, ER, and more), editable per Feature. Any diagram can be exported client-side as a **PNG or SVG** image directly from its detail panel — no server round trip.

- **LLM chat agent** with:
  - A propose-confirm tool-calling flow for every write, so the model never mutates data without a human in the loop.
  - SSE-streamed responses.
  - Persisted, per-conversation history, automatically **summarized** once it grows past a threshold (rolling summary folds everything except the most recent turns, keeping long sessions within the model's context window).
  - Pluggable multi-provider configuration — OpenAI, Anthropic, Gemini, DeepSeek, Ollama, LM Studio, Hunyuan, Moonshot AI.

- **MCP Server** — 20 tools, Streamable HTTP transport, so any MCP-aware client can query or edit the requirements model directly.

- **RAG / semantic search** — a global search box in the header (searches every product in the org by default) plus a `semantic_search` tool available from both the chat agent and MCP clients. Every Create/Update asynchronously (re-)embeds the entity's content and a SQLite-backed `embedding_cache` table persists every vector so the in-memory index rebuilds instantly on restart without re-calling the embedding API.

## Three Ways to Manage Your Data

Every entity in Toolmaker Agent — Product, Feature, Requirement, UML diagram — can be created, updated, and deleted through three independent front doors, all backed by the same service layer and the same database. Pick whichever fits the moment: fill out a form, describe what you want in plain language, or let a coding agent do it for you.

### 1. Manual — the web UI

Plain forms and detail panels: click "+ Create", type into a field, hit save. No AI in the loop at all — the baseline CRUD experience every other mode builds on.

The recording below creates a Product with content, a Feature, two Requirements (deleting one), a realistic UML sequence diagram under the Process view, and finishes with a live semantic-search lookup that jumps straight to a matching Feature.

![Manual CRUD demo](docs/images/manual-crud-demo.gif)

### 2. LLM Chat — propose, confirm, done

The header chat panel talks to the same entities in natural language. Every write goes through the **propose → confirm** flow described in [Tool Calling](#tool-calling) below: the model proposes an action, you see exactly what it's about to do, and nothing is written until you click Confirm.

The recording below runs the same scenario as the manual demo, entirely through chat — including the model asking a clarifying question when a required field (a Requirement's content) is missing, rather than guessing, and then proposing the write once you answer.

![LLM chat CRUD demo](docs/images/llmchat-crud-demo.gif)

### 3. MCP — a coding agent driving the same data

Register the server with an MCP-aware client such as Claude Code (see [MCP Server](#mcp-server) below) and it can create, query, update, and delete the exact same entities directly from the terminal. MCP tools execute immediately — there's no confirmation dialog, since the caller is a developer already driving the agent.

The recording below is a real Claude Code session issuing MCP tool calls end to end: create a Product, a Feature, and a Requirement (again pausing to ask for a missing required field instead of guessing — this time the agent asks *you*, in the terminal), query them back, delete a Requirement, and create a UML sequence diagram.

![MCP CRUD demo](docs/images/mcp-crud-demo.gif)

## Tool Calling

The chat agent's tools split into two categories by risk. **Propose tools** never touch the database directly — the model's call is handed to the UI as a confirmation dialog, and only an explicit human approval turns it into a real write. **Query tools** execute immediately, since a read has no side effects to confirm:

| Operation | Product | Feature | Requirement | UML |
|---|---|---|---|---|
| Create *(confirm)* | `propose_create_product` | `propose_create_feature` | `propose_create_requirement` | `propose_create_uml_diagram` |
| Update *(confirm)* | `propose_update_product` | `propose_update_feature` | `propose_update_requirement` | `propose_update_uml_diagram` |
| Delete *(confirm)* | `propose_delete_product` | `propose_delete_feature` | `propose_delete_requirement` | `propose_delete_uml_diagram` |
| Get one *(direct)* | `query_product` | `query_feature` | `query_requirement` | `query_uml` |
| List *(direct)* | `query_product_list` | `query_feature_list` | `query_requirement_list` | `query_uml_list` |

Plus `propose_generate_requirements` (drafts a whole Feature and its Requirements from the conversation for one combined confirmation) and `semantic_search` (direct, see below).

The same split shapes MCP and semantic search below: MCP tools skip the confirmation step entirely (the caller is a developer already driving the agent), while every entry point — chat, MCP, REST — ultimately runs through the same service layer.

## MCP Server

Register the server with an MCP-aware client (e.g. Claude Code):

```
claude mcp add --transport http toolmaker-agent http://127.0.0.1:8080/agtapi/v2/mcp
```

20 tools are exposed, 5 for each of Product / Feature / Requirement / UML:

| Operation | Product | Feature | Requirement | UML |
|---|---|---|---|---|
| Create | `create_product` | `create_feature` | `create_requirement` | `create_uml` |
| Get one | `query_product` | `query_feature` | `query_requirement` | `query_uml` |
| List | `query_product_list` | `query_feature_list` | `query_requirement_list` | `query_uml_list` |
| Update | `update_product` | `update_feature` | `update_requirement` | `update_uml` |
| Delete | `delete_product` | `delete_feature` | `delete_requirement` | `delete_uml` |

Entities are addressed by a stable, per-parent `OID` (not the internal database `id`), and Requirement/Feature/UML lookups take a `productOid` (Requirement additionally accepts an optional `featureOid`). MCP tools execute immediately against live data — there is no propose/confirm step here (that's specific to the web chat's tool-calling flow).

A 21st tool, `semantic_search`, is also exposed (requires `productOid`; searches that product's Features/Requirements/UML/itself by meaning) — see below.

## Semantic Search

Three ways to reach the same underlying vector index, each scoped differently:

| Surface | Scope | Notes |
|---|---|---|
| Chat agent tool (`semantic_search`) | The chat's current product only | Real-execution tool, like `query_requirement_list` — the model can't pick a different product itself |
| MCP tool (`semantic_search`) | One product, via required `productOid` | Same handler logic as the chat tool |
| `GET /agtapi/v2/search?q=...` | **Every product in the org** by default; optional `productOid` narrows to one | Backs the header search box; results include `productOid` since a hit can come from any product |

All three return each hit's kind (`product`/`feature`/`requirement`/`uml`), OID (or internal id for `uml`, which has none), name, and a relevance score — never the full content; follow up with the matching `query_*`/`get`/detail-panel lookup once you know which entity matched.

**Rebuilding the index**: `POST /agtapi/v2/admin/rag/reindex?productOid=<optional>&force=<optional>` walks every Product (or one, via `productOid`) and re-submits every entity under it for indexing. Use it to backfill data that existed before semantic search was configured, or — with `force=true` — to force a full re-embed after switching embedding models. Admin-only, trusted-network use — not intended for internet-facing deployments.

## Tech stack

| Layer | Technology |
|---|---|
| Backend language/runtime | Go 1.26 |
| HTTP framework | [Gin](https://github.com/gin-gonic/gin) |
| Database | SQLite via [`dbsqlx`](https://github.com/vinovest/sqlx) (raw SQL, no ORM); schema applied from `config/schema_sqlite.sql` |
| Agent/LLM orchestration | [`trpc-agent-go`](https://github.com/trpc-group/trpc-agent-go) |
| RAG / vector search | `trpc-agent-go`'s `knowledge/embedder` (OpenAI-compatible embeddings, incl. DashScope) + `knowledge/vectorstore/inmemory` |
| MCP | Official [`modelcontextprotocol/go-sdk`](https://github.com/modelcontextprotocol/go-sdk), Streamable HTTP transport |
| Auth/policy | Casbin (via `common-library-golang/auth`) |
| CLI | [Cobra](https://github.com/spf13/cobra) |
| Frontend | React 19, TypeScript, Vite |
| Frontend state | [TanStack Query](https://tanstack.com/query) v5 |
| Frontend routing | react-router-dom v7 |
| Styling | Tailwind CSS |
| Diagrams | [Mermaid](https://mermaid.js.org/) (+ Cytoscape, KaTeX for advanced diagram types) |

## License

Apache License 2.0 — see [LICENSE](./LICENSE).
