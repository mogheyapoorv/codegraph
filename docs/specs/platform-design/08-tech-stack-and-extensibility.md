# CodeGraph Platform — Tech Stack & Extensibility

## 11. Tech Stack Summary

| Layer | Technology | Version |
|---|---|---|
| Frontend | Next.js, React, TypeScript, Vercel AI SDK, Tailwind CSS | Next.js 16, React 19, AI SDK 5.x, Tailwind 4 |
| Backend | Python, FastAPI, LangGraph, LangChain | Python 3.13+, FastAPI 0.128+, LangGraph 1.0+, LangChain 1.0+ |
| Java Extractor | Java, Spoon, PMD, JGit, Jackson | Java 21, Spoon 11.x, PMD 7.x, JGit 7.x, Jackson 3.x |
| Database | PostgreSQL, pgvector, Apache AGE, pg_trgm | PostgreSQL 18, pgvector 0.8+, AGE 1.5+ |
| Event Bus | Apache Kafka (KRaft-only) | Kafka 4.x |
| Cache | Redis (optional, cloud only) | Redis 8.x |
| Auth | Built-in JWT + optional Keycloak | Keycloak 26+ |
| LLM Providers | LangChain providers + optional LiteLLM proxy | LiteLLM 1.80+ |
| Object Storage | fsspec (local / S3 / GCS / Azure) | fsspec 2026.2+ |
| Diagrams | Mermaid (client-side rendering) | Mermaid 11.4+ |
| Git Operations | GitPython | GitPython 3.1+ |
| Deployment | Docker Compose (local), Kubernetes + Helm (cloud) | Compose v2, Helm 3 |

### What CodeGraph Builds vs Adopts

| Category | We Build (Core IP) | We Adopt (Battle-Tested) | Extensibility Point |
|---|---|---|---|
| Entity graph engine | Consensus engine, entity type registry | Spoon 11.x, PostgreSQL + AGE | New entity types and relation types via schema — no code changes |
| Language extraction | Extractor plugin interface, event bus contract, tiered extraction pattern | Spoon (Java), tree-sitter (multi-lang), Pyright (Python), TS Compiler API | New language = new extractor implementing `repo.indexed` → `entities.extracted` contract |
| Cross-repo resolution | Multi-source matcher, confidence scoring, source collector registry | MiSAR algorithm, Code2DFD patterns, MS Application Inspector-style pattern rules, Buf, Backstage catalog model | New dependency detection = new source collector implementing the collector interface |
| Wiki generation | Wiki agent, page planning, page type templates, content generation | LangGraph, LangChain, Mermaid | New page types = new template + entity graph query |
| Diagram engine | Diagram type registry, graph-to-Mermaid renderers | Mermaid | New diagram type = renderer function (graph query → Mermaid) registered by type name |
| Chat intelligence | Adaptive search, graph-grounded verification, scope-aware retrieval, tool registry | LangGraph, pgvector, pg_trgm | New chat tool = Python function registered in the tool registry |
| API Catalog | curl/grpcurl template generation, payload examples | swagger-parser, protobuf-java, Spoon type resolution | New protocol = new template generator (e.g., WebSocket, SSE) |
| Static analysis | Risk flag storage per entity, analyzer adapter interface | PMD 7.x (Java), Ruff (Python), RuboCop (Ruby), Biome (TS/JS) | New language = new analyzer adapter wrapping that language's linter |
| Anomaly detection | Anomaly type registry, detection scheduling | LangGraph | New anomaly type = detector function + anomaly_type enum value |
| Git intelligence | Metric aggregation per entity | PyDriller (git mining, bus factor, churn) | New metric = new PyDriller query + health score entry |
| Database layer | Repository interfaces (per capability) | PostgreSQL, pgvector, AGE, pg_trgm, tsvector | Swap any capability = new repository implementation behind the interface |
| Storage layer | Storage abstraction | fsspec (local/S3/GCS/Azure) | New backend = fsspec already supports it, or add a new fsspec filesystem |
| Integration layer | Integration adapter interface | MCP protocol, Slack SDK, Microsoft Bot Framework | New integration = adapter implementing the chat/notification interface |
| Auth | Setup wizard, role middleware, provider abstraction | Keycloak, authlib, PyJWT | New auth provider = any OIDC-compliant issuer, zero code changes |
| PR integration | Comment formatting, blast radius rendering, platform adapter | PyGithub (GitHub), python-gitlab (GitLab), atlassian-python-api (Bitbucket) | New git platform = new adapter implementing the PR comment interface |

### Extensibility Architecture

#### Core vs Extensible — The Boundary

**Core (stable, not replaced — this IS CodeGraph):**
- Entity graph schema and query patterns — the backbone data model
- Event contracts — the 7 event types and their JSON Schemas in `contracts/`. These are the stable API between components.
- CQRS pipeline flow — the write path (push → index → extract → embed → generate) and read path (query → search → respond) are fixed patterns
- Confidence scoring model — entity_confidence and relation_confidence scales are universal
- Wiki page versioning model — pages, versions, annotations, diagrams
- Auth roles — three fixed roles (superadmin/admin/member). Custom RBAC is out of scope for v1.

**Extensible via interfaces (swap or add without touching core):**

| Extension Point | Interface | How to Add | Reference |
|---|---|---|---|
| Language extractor | `Extractor` — consume `repo.indexed`, produce `entities.extracted` | Register extractor manifest (languages, file patterns, capabilities) | Section 5 |
| Database capability | `Repository` — per capability (relational, vector, graph, full-text, trigram) | Implement repository interface for new backend | Section 2 |
| Event bus | `EventBus` — publish/subscribe with JSON Schema contracts | Implement EventBus interface for new transport | Section 2 |
| Agent framework | Task interfaces per agent — inputs, outputs, tools | Implement task interface with new orchestration framework | Section 2 |
| Diagram type | `DiagramRenderer` — entity graph query results → Mermaid markup | Register renderer function by type name | Section 6 |
| Wiki page type | `WikiTemplate` — entity graph queries + formatting rules + prompts | Register template in template registry | Section 6 |
| Chat tool | `ChatTool` — name, description, input schema, function | Register tool function in tool registry | Section 7 |
| MCP tool | Same as ChatTool — MCP layer auto-exposes registered tools | Register tool function (shared with chat tools) | Section 8 |
| Source collector | `SourceCollector` — repo entity graph → dependency signals | Register collector per dependency type | Section 6 |
| Static analyzer | `AnalyzerAdapter` — file path + language → risk flags | Implement adapter wrapping language linter | Section 5 |
| Anomaly detector | `AnomalyDetector` — entity graph + wiki state → anomalies | Register detector function by anomaly type | Section 14 |
| Health metric | `HealthMetric` — repo/group ID → score + details | Register metric function in metric registry | Section 9 |
| Git platform | `GitPlatformAdapter` — repo discovery, webhooks, PR comments, API | Implement adapter for new platform | Section 4 |
| Messaging platform | `MessagingAdapter` — receive message, route to agent, send reply | Implement adapter for new platform | Section 8 |
| LLM provider | LangChain provider interface (or direct SDK behind abstraction) | Add provider config in admin dashboard | Section 3 |
| Object storage | fsspec filesystem interface | fsspec supports it natively, or add new filesystem | Section 2 |
| Auth provider | OIDC standard | Point at any OIDC-compliant issuer | Section 3 |

#### Universal Pattern

Every extension point follows the same pattern: **interface → registry → discovery**.

- **Interfaces** define the contract (what a language extractor must implement, what a diagram renderer must produce)
- **Registries** map type names to implementations (entity types, diagram types, anomaly types, chat tools)
- **Discovery** happens at startup — the system scans plugin directories, Docker labels, and configuration to find registered implementations

This means adding a new language, diagram type, anomaly detector, or integration never requires modifying existing code — you implement the interface, register it, and existing consumers pick it up.

#### Entity and Relation Type Extension

Entity types and relation types are stored as string values in the database, not as hardcoded enums. Adding a new entity type (e.g., `graphql_mutation`, `websocket_channel`) or relation type (e.g., `rate_limits`, `circuit_breaks`) means:
1. Extractors produce entities/relations with the new type string
2. The entity graph stores them — no schema migration needed
3. Diagram renderers and wiki templates that don't know about the new type ignore it gracefully
4. New renderers/templates can be registered to handle the new type

#### What's Fixed by Design

- **Event contract structure** — the 7 event types are stable. New features plug into existing events rather than adding new ones. If a genuinely new event is needed, it goes through a contract review process.
- **Confidence scoring scales** — entity_confidence (0.0-1.0) and relation_confidence (0.0-1.0) use fixed scales. New extractors and collectors produce scores on the same scale.
- **Auth roles** — three roles (superadmin/admin/member). Custom RBAC is a potential future extension but not part of v1 scope.

### Full Library Inventory

Every external dependency we use, with purpose:

**Python Backend:**

| Purpose | Library | Why This One |
|---|---|---|
| Web framework | FastAPI 0.128+ | Async, OpenAPI auto-docs, WebSocket support |
| Agent framework | LangGraph 1.0+ | GA stable API, stateful graphs, checkpointing, multi-step reasoning |
| LLM abstraction | LangChain 1.0+ | Stable API (no breaking changes until 2.0), multi-provider, tool calling, embeddings |
| Database ORM | SQLAlchemy 2.0+ | Async support, mature, PostgreSQL-native |
| Async PostgreSQL | asyncpg | Fastest Python PostgreSQL driver |
| Kafka consumer/producer | aiokafka | Async-native, lightweight |
| Background scheduling | APScheduler 3.11+ | Async-compatible, cron-style scheduling (4.0 still in alpha) |
| Git mining | PyDriller 2.6+ | Commit history, blame, bus factor, churn analysis |
| Git operations | GitPython 3.1+ | Clone, pull, diff |
| OpenAPI parsing | openapi-spec-validator | Validate OpenAPI specs consumed from checked-in files (Java Extractor owns initial parsing via swagger-parser; Python validates for cross-repo matching) |
| YAML parsing | PyYAML / ruamel.yaml | Parse application.yml, config files |
| JWT auth | PyJWT + authlib | Token validation, OIDC discovery |
| GitHub API | PyGithub 2.x | PR comments, repo discovery, webhook validation |
| GitLab API | python-gitlab 4.x | MR comments, repo discovery |
| Bitbucket API | atlassian-python-api 3.x | PR comments, repo discovery for Bitbucket Cloud/Server |
| Object storage | fsspec 2026.2+ | Unified S3/GCS/Azure/local filesystem |
| HTTP client | httpx | Async HTTP for webhook delivery, MCP, external APIs |
| Schema validation | pydantic 2.x | Kafka event validation, API request/response models |
| Observability | opentelemetry-sdk, opentelemetry-instrumentation-fastapi, opentelemetry-instrumentation-asyncpg, opentelemetry-instrumentation-aiokafka, opentelemetry-instrumentation-httpx | Traces, metrics, logs — single OTel stack |

**Java Extractor:**

| Purpose | Library | Why This One |
|---|---|---|
| Java analysis | Spoon 11.x | Clean metamodel over JDT, NOCLASSPATH mode |
| Static analysis | PMD 7.x | 400+ rules, source-only, no compilation |
| Maven resolution | Maven Resolver (Apache) | Download dependency JARs without compiling |
| Gradle resolution | Gradle Tooling API | Resolve dependencies from Gradle projects |
| Protobuf parsing | protobuf-java (Google) | Parse .proto files for gRPC service extraction |
| OpenAPI parsing | swagger-parser 2.x | Parse checked-in OpenAPI specs |
| SQL parsing | JSqlParser | Parse Flyway/Liquibase migration SQL |
| YAML parsing | SnakeYAML 2.x | Parse application.yml, Spring config |
| Event bus client | kafka-clients (Apache) — default implementation | Consume/produce events via EventBus interface wrapper. Swappable if event bus changes (Section 2). |
| Git operations | JGit 7.x (Eclipse) | Diff within Java extractor (Python backend clones to shared volume; extractor reads from there, does NOT clone independently) |
| Observability | opentelemetry-javaagent | Auto-instrumentation via javaagent — zero code changes for Kafka, JDBC, HTTP tracing |
| JSON | Jackson 3.x (LTS) | Kafka event serialization, config parsing (3.x requires Java 17+, built-in java.time support) |

**Frontend:**

| Purpose | Library | Why This One |
|---|---|---|
| Framework | Next.js 16 | SSR for wiki pages, App Router |
| Chat UI | Vercel AI SDK 5.x | useChat hook (transport-based, sendMessage API), streaming, tool status rendering |
| Diagrams | Mermaid 11.4+ | Client-side diagram rendering |
| Markdown | react-markdown | Wiki page rendering |
| Syntax highlighting | react-syntax-highlighter or Shiki | Code block rendering in wiki and chat |
| Styling | Tailwind CSS 4 | Utility-first, fast iteration |
| Pan/zoom | svg-pan-zoom | Interactive diagram navigation |

---

**Prev:** [Operations](07-operations.md) | **Next:** [Data Model](09-data-model.md) | [Index](00-index.md)
