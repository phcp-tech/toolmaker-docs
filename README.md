# Toolmaker Agent

An AI-augmented requirements and system-design workbench: a Go backend with an embedded React UI for managing **Products, Features, Requirements, and UML/4+1-view system-design diagrams**, paired with an LLM-driven conversational agent and a Model Context Protocol (MCP) server so both humans and coding agents (e.g. Claude Code) can drive the same data model.

<!-- SCREENSHOT: main workbench overview, e.g. the Requirements page with the
     product/feature tree, detail panel, and chat panel all visible -->
![Toolmaker Agent overview](docs/images/overview.png)

## What this is

Toolmaker Agent is a single self-contained executable: a Go REST API with a layered architecture (adapter → service → infra/dao → domain/model), SQLite storage, and the compiled React frontend embedded directly into the binary via `go:embed`. Open one port and you get the full workbench — no separate frontend deployment, no external database to provision.

On top of the plain CRUD workbench, it adds two AI-native ways to manipulate the same data:

- **Conversational agent** — a chat panel backed by [`trpc-agent-go`](https://github.com/trpc-group/trpc-agent-go) that can create/update/delete Products, Features, Requirements, and UML diagrams through natural language, using a **propose → confirm** flow for any write (the model proposes the action, a human confirms it before it executes).
- **MCP Server** — a [Model Context Protocol](https://modelcontextprotocol.io/) endpoint (official [`modelcontextprotocol/go-sdk`](https://github.com/modelcontextprotocol/go-sdk)) exposing 20 tools for full CRUD over the same four entities, so a coding agent like Claude Code can manage requirements directly from the terminal. Unlike the chat's propose/confirm tools, MCP tools are **direct tools** — they execute immediately, with no confirmation step.

## Key features

- **Product / Feature / Requirement / UML CRUD**, each with optimistic concurrency (a `revision` field, checked on every update) and full-row responses on create/update (the backend `RETURNING`s the complete post-write row so the client never has to guess at server-computed fields like `updatedAt` or `revision`).

  <!-- SCREENSHOT: Product Management page, a product selected with its detail panel open -->
  ![Product management](docs/images/products.png)

  <!-- SCREENSHOT: Requirement Analysis page with a Feature selected, showing the Feature detail panel -->
  ![Feature management](docs/images/features.png)

  <!-- SCREENSHOT: Requirement Analysis page with a Requirement selected, showing the Requirement detail panel -->
  ![Requirement management](docs/images/requirements.png)

- **4+1 system-design views** rendered as [Mermaid](https://mermaid.js.org/) diagrams (flowcharts, sequence, C4/architecture, state, ER, and more), editable per Feature. Any diagram can be exported client-side as a **PNG or SVG** image (PNG rasterized at 3x scale for sharp output) directly from its detail panel — no server round trip.

  <!-- SCREENSHOT: system-design page showing a rendered Mermaid sequence diagram -->
  ![System design view](docs/images/system-design.png)

- **LLM chat agent** with:
  - A propose-confirm tool-calling flow for every write (`propose_create_*` / `propose_update_*` / `propose_delete_*`), so the model never mutates data without a human in the loop.
  - SSE-streamed responses.
  - Persisted, per-conversation history, automatically **summarized** once it grows past a threshold (rolling summary folds everything except the most recent turns, keeping long sessions within the model's context window).
  - Pluggable multi-provider configuration — OpenAI, Anthropic, Gemini, DeepSeek, Ollama, LM Studio, Hunyuan, Moonshot AI, Qwen, GLM, MiniMax. Managed either by hand-editing a local, git-ignored `config/settings.json` (never committed; a documented `.example` template is checked in instead), or entirely from the browser via the **Settings** page, which lists every configured provider and lets you add, edit, activate, or delete one without touching a file.

  <!-- SCREENSHOT: Settings page, LLM section, provider list with one entry's detail panel open -->
  ![LLM provider settings](docs/images/settings.png)

  <!-- SCREENSHOT: chat panel mid-conversation, ideally showing a
       propose/confirm dialog for a write action (e.g. "propose_update_requirement") -->
  ![LLM chat agent](docs/images/chat-agent.png)
- **MCP Server** — 20 tools (5 each for Product/Feature/Requirement/UML: create/query/query-list/update/delete), Streamable HTTP transport, so any MCP-aware client can query or edit the requirements model directly.
- **Embedded single-binary deployment** — the compiled frontend is staged into `web/dist` and embedded into the Go binary; the result is one executable that serves both the API and the UI.
- **Free-edition auth bypass** — out of the box, every request is authenticated as a fixed admin identity (no login flow), so the workbench is usable immediately in a local/demo environment. (Not intended as-is for a multi-tenant or internet-facing deployment.)

## Tech stack

| Layer | Technology |
|---|---|
| Backend language/runtime | Go 1.26 |
| HTTP framework | [Gin](https://github.com/gin-gonic/gin) |
| Database | SQLite via [`dbsqlx`](https://github.com/vinovest/sqlx) (raw SQL, no ORM); schema applied from `config/schema_sqlite.sql` |
| Agent/LLM orchestration | [`trpc-agent-go`](https://github.com/trpc-group/trpc-agent-go) |
| MCP | Official [`modelcontextprotocol/go-sdk`](https://github.com/modelcontextprotocol/go-sdk), Streamable HTTP transport |
| Auth/policy | Casbin (via `common-library-golang/auth`) |
| CLI | [Cobra](https://github.com/spf13/cobra) |
| Frontend | React 19, TypeScript, Vite |
| Frontend state | [TanStack Query](https://tanstack.com/query) v5 |
| Frontend routing | react-router-dom v7 |
| Styling | Tailwind CSS |
| Diagrams | [Mermaid](https://mermaid.js.org/) (+ Cytoscape, KaTeX for advanced diagram types) |

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

## License

Apache License 2.0 — see [LICENSE](./LICENSE).
