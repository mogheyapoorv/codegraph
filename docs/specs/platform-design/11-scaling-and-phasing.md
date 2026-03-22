# CodeGraph Platform — Scaling & Phasing

## 10. Phasing — Build Order

### Phase 1: Foundation

- PostgreSQL setup (pgvector + AGE + tsvector + pg_trgm)
- Python backend (FastAPI), auth (built-in JWT), admin API
- Kafka setup (KRaft, core event topics)
- Java Extractor (Spoon 11.x, framework parsers, Tier 1)
- LLM integration (LangChain providers, admin config)
- Tier 2 enrichment (LangChain gap-fill agent)
- Basic wiki generation (LangGraph wiki generator, single-repo)
- Frontend (Next.js, wiki viewer, admin dashboard, setup wizard)
- Embedding pipeline (chunking, pgvector population, vector search)
- Database abstraction interface definitions (repository interfaces for each capability)
- Docker Compose (5-container local deployment)

### Phase 2: Intelligence + Language Expansion

- Chat agent (LangGraph — Quick Ask + Conversational)
- Adaptive search (entity graph + vector + trigram + wiki tools)
- Graph-grounded verification (anti-hallucination)
- API Catalog (endpoint discovery, curl/grpcurl generation)
- Repo groups (group management, scoped search)
- Cross-repo resolution (hybrid structural + similarity + reasoning)
- Cross-repo diagrams (architecture, sequence, data flow, ER)
- Background consistency job
- Incremental updates (webhook + polling + diff-based re-extraction)
- **Language extractors: Ruby, Python, TypeScript, Node.js** (Level 1-2, Python modules in backend)

### Phase 3: Deep Features

- Deep Research mode (plan-approve-execute, checkpointing, reports as wiki)
- All 20 diagram types
- PR Context Injection (GitHub/GitLab/Bitbucket)
- Anomaly detection
- Codebase health scores
- Wiki annotations (human-in-the-loop)
- Analytics + cost tracking

### Phase 4: Platform & Ecosystem

- MCP server
- Slack/Teams bot
- CLI tool
- Embeddable widget
- Public REST API with API keys
- OIDC/Keycloak auth
- Cloud deployment (S3/GCS, Redis, managed Kafka, Helm charts)
- Language extractors grow to Level 3 via community contribution

---

## 17. Scaling Limits & Migration Paths

### Single PostgreSQL Limits

Estimated comfortable limits for one PostgreSQL instance:

| Metric | Limit | Notes |
|---|---|---|
| Repos | ~500-1000 | Depends on repo size |
| Source files in DB | ~500GB | pg_trgm index performance degrades beyond this |
| Embeddings | ~10M vectors | pgvector with IVFFlat, HNSW for better performance |
| Entity graph nodes | ~5M | AGE handles this well |
| Entity graph edges | ~20M | Index-dependent |
| Concurrent chat users | ~50-100 | Connection pool dependent |

### When to Scale Beyond Single PostgreSQL

The signals: pg_trgm queries > 500ms, vector search > 200ms, connection pool exhaustion.

### Migration Paths

| Bottleneck | Migration |
|---|---|
| Vector search too slow | Swap pgvector → Qdrant (via abstraction layer) |
| Graph queries too slow | Swap AGE → Neo4j (via abstraction layer) |
| Trigram search too slow | Swap pg_trgm → Elasticsearch/Zoekt (via abstraction layer) |
| Storage too large | PostgreSQL table partitioning by repo_id. Move cold repos to separate tablespace. |
| Too many concurrent users | Read replicas for PostgreSQL. Horizontal scale on the Python backend. |

### Data Retention

- Wiki page versions: kept for 90 days (configurable), then oldest versions pruned
- Source files: only current version stored. Updated on each push.
- LLM usage logs: aggregated after 30 days (raw logs pruned, daily summaries kept)
- Chat analytics: aggregated after 30 days
- Deep Research reports: kept indefinitely (auto-saved)

### Repo Removal

When a repo is removed from CodeGraph:
- Soft delete: repo marked inactive, data retained for 30 days
- After 30 days: hard delete cascades — entities, relations, snippets, source files, embeddings, wiki pages, diagrams all removed
- Cross-repo relations involving the removed repo: deleted immediately from all groups

---

**Prev:** [Event Contracts](10-event-contracts.md) | [Index](00-index.md)
