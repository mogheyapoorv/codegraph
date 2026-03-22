# CodeGraph Platform — Chat & Deep Research

## 7. Chat & Deep Research

### Three Modes, One Agent, Different Leash

| Mode | Max Steps | Timeout | Plan Approval | Checkpointing |
|---|---|---|---|---|
| Quick Ask | 5 | 30s | No | No |
| Conversational | 15 | 120s | No | No |
| Deep Research | 50 | 600s | Yes | Yes (PostgreSQL) |

Same LangGraph agent, same tools. Only the configuration differs.

### Ephemeral Sessions

- No cross-session memory. Messages live in the browser (AI SDK `useChat` hook with `sendMessage`).
- Tab closes = session gone.
- Wiki changes make old conversations stale anyway — no reason to persist.
- Deep Research reports are auto-saved to PostgreSQL (the report persists, not the conversation). Optionally also saved as a wiki page.

### Chat Agent Tools (In-Process Python Functions)

| Tool | Data Source (via repository interfaces) | What It Finds |
|---|---|---|
| `entity_graph_query` | Graph query interface (default: AGE/Cypher) | Named entities, relations, blast radius, call chains |
| `code_snippet_read` | Relational interface (code_snippets) | Actual source code for a specific entity |
| `code_search` | Trigram search interface (default: pg_trgm) | Exact strings — error messages, variable names, TODOs |
| `vector_search` | Vector search interface (default: pgvector) | Semantically similar code and wiki content |
| `wiki_search` | Full-text search interface (default: tsvector) | Wiki page content |
| `api_catalog_query` | Relational interface (api_endpoints) | Endpoints with curl/grpcurl examples |
| `git_metadata_query` | Relational interface (git_metadata) | Commit history, blame, recent changes |
| `generate_diagram` | Graph query interface → Mermaid | On-the-fly diagrams |

All tools are Python functions querying through the repository interfaces (Section 2). In-process, zero network hops.

**Tool registration:** Chat tools are registered via a tool registry. Each tool is a Python function with a name, description, input schema, and implementation. Adding a new chat tool (e.g., a Jira search, a Confluence lookup, a custom metric query) means implementing the tool interface and registering it — no changes to the agent or core code. Community-contributed tools follow the same pattern.

### Adaptive Search

The agent decides what to search per query — there's no fixed pipeline:
- Structural question about a named entity → entity graph first
- Exact string search → trigram (pg_trgm) first
- Conceptual question → vector search first
- Cross-repo question → entity graph traversal first
- Agent evaluates results, searches more if insufficient

### Graph-Grounded Anti-Hallucination

**Authority hierarchy for context assembly:**
1. Entity graph facts (highest — deterministic, from AST)
2. Code snippets (high — actual source code)
3. Source file trigram matches (high — exact text)
4. Wiki content (medium — LLM-generated from verified facts)
5. Vector search results (medium — semantic similarity)

**Post-generation verification:**
- Every entity mentioned → verify it exists in the entity graph
- Every file:line citation → verify it exists in source_files or code_snippets
- Unverifiable claim → flag with warning or remove
- No source for a statement → don't include it

### External Knowledge (Confluence / Google Docs)

Only activated when a user pastes a link in chat:
- No link → no external call. CodeGraph-owned data only.
- Confluence link → fetch via Atlassian MCP using user's connected OAuth token
- Google Docs link → fetch via Google API using user's connected OAuth token
- Content used for that one response only. Never stored.

### Deep Research

Long-running autonomous investigation:

1. **Plan** — Agent generates a research plan (5-10 steps) from the query
2. **Approve** — Plan sent to user via WebSocket. User approves, modifies, or cancels.
3. **Execute** — Agent executes each step, streams progress. LangGraph checkpoints after each step to PostgreSQL. Crash at step 7 → resume from step 7.
4. **Report** — Structured report: summary, findings, diagrams, sources, recommendations. All with file:line citations.
5. **Save** — Deep Research reports are **auto-saved** to prevent loss on tab close. Also optionally saved as a wiki page, making them searchable by future chat queries.

---

**Prev:** [Wiki Generation & Diagrams](04-wiki-generation-and-diagrams.md) | **Next:** [Integrations](06-integrations.md) | [Index](00-index.md)
