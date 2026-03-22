# CodeGraph Platform — Integrations

## 8. PR Context Injection, MCP Server & Integrations

### PR Context Injection

When a developer opens a PR, CodeGraph auto-comments with context from the entity graph.

**Trigger:** GitHub/GitLab/Bitbucket webhook for PR opened or updated.

**PR comment includes:**
- **Summary** — what changed in terms of entities, not just files
- **Blast radius** — services, endpoints, consumers affected
- **Cross-repo impact** — other repos depending on changed code. Flags breaking changes.
- **API contract changes** — old vs new schema if endpoint types changed
- **Related wiki pages** — links to wiki describing the changed code
- **Risks** — static analysis flags on changed code
- **curl/grpcurl** — updated commands for changed endpoints so the reviewer can test

**When NOT to comment:** documentation-only changes, test-only changes, dependency version bumps. Admin configures a minimum blast radius threshold.

### MCP Server

CodeGraph exposes an MCP server for AI IDEs (Claude Desktop, Cursor, Windsurf):

| MCP Tool | What It Does |
|---|---|
| `codegraph_search` | Search across code, wiki, entity graph (scoped) |
| `codegraph_entity_graph` | Query entity relationships |
| `codegraph_blast_radius` | Impact analysis for a file or entity |
| `codegraph_wiki_page` | Read a wiki page |
| `codegraph_api_catalog` | Get endpoints with curl/grpcurl |
| `codegraph_diagram` | Get or generate a diagram |
| `codegraph_code_search` | Trigram search across source code |

Authenticated via API key. Part of the Python backend (FastAPI endpoint following MCP protocol).

MCP tools are registered via the same tool registry as chat tools (Section 7). Each MCP tool maps to one or more chat agent tools under the hood. Adding a new MCP tool means registering the tool function — the MCP protocol layer exposes it automatically.

### Slack / Teams Bot

CodeGraph answers questions in messaging channels. Same chat agent (Quick Ask mode). Admin configures bot token + default scope per channel.

**Messaging platform adapter interface:** Each messaging platform (Slack, Teams, Discord, Mattermost) is implemented as an adapter behind a common interface: receive message → route to chat agent → format response → send reply. Adding a new messaging platform means implementing this adapter. The chat agent, scope resolution, and rate limiting work unchanged.

### CLI Tool

Command-line interface calling the REST API:

```
codegraph ask "how does auth work?" --scope payments-domain
codegraph search "payment error" --repo acme/payment-service
codegraph diagram architecture --group payments-domain
codegraph blast-radius src/main/java/RetryHandler.java
codegraph curl POST /api/payments/charge
codegraph status
```

### Embeddable Widget

A JavaScript snippet that adds CodeGraph chat to any internal web tool (Backstage, internal portal, Confluence). Floating chat button → expands to drawer → same chat agent, scoped to configured group.

### Public REST API

Every feature is accessible programmatically: wiki, entity graph, chat (streaming SSE), search (code/vector/wiki), diagrams, API catalog, blast radius, admin. Authenticated via API key or JWT. Rate limited.

---

**Prev:** [Chat & Research](05-chat-and-research.md) | **Next:** [Operations](07-operations.md) | [Index](00-index.md)
