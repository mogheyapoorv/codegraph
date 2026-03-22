# CodeGraph Platform — Goals & Architecture

---

## 1. Goals & Constraints

### Primary Use Cases
- **Internal developer tool** — a large org (50+ people, 500+ repos) uses CodeGraph to document, explore, and understand their codebase
- **Open-source platform** — we want this to be the most comprehensive open-source repo documentation and code intelligence tool out there
- **Compete with the best** — we're going after the same problems Greptile, Sourcegraph, and others are solving, but fully open-source and self-hostable

### Design Principles
- **Fully open-source** — no enterprise edition gate. Everything ships open.
- **Self-hostable** — `docker-compose up` gets you running. Five containers.
- **No vendor lock-in** — pluggable LLM providers, pluggable database components
- **Wiki is always public** — any authenticated user reads any wiki. No access gates.
- **Main/master branch only** — we generate wikis for the default branch. No branch-level wikis.
- **Auto-updating** — push to main, wiki regenerates. Zero manual intervention.
- **Deterministic extraction, intelligent generation** — code analysis is compiler-accurate. LLM only handles natural language and gap-filling.
- **CQRS architecture** — write path (async, event-driven) is separated from read path (sync, direct queries)
- **Extensible by default** — every component sits behind an interface. Language extractors, diagram renderers, chat tools, anomaly detectors, database backends, event bus, and integrations are all pluggable. Community contributors add new capabilities without touching core code.

### What We Think We Can Do Well
- **Entity graph from AST** — we extract code structure using compiler-level tools, so the graph is deterministic and verifiable rather than LLM-inferred
- **Cross-repo intelligence** — the entity graph spans repos, which lets us show how services actually connect and depend on each other
- **API Playground** — auto-generated curl/grpcurl for every discovered endpoint, so developers can test without leaving the wiki
- **Multi-host support** — GitHub, GitLab, and Bitbucket from day one
- **Self-hosted with full cost control** — teams run it on their own infrastructure and manage their own LLM spend

---

## 2. High-Level Architecture

### Pattern: CQRS + Event-Driven + Hybrid Pipeline/Agentic

```
WRITE PATH (async, event-driven):
  Repo push → Kafka → Extractors + Agents → PostgreSQL + Object Storage
  Optimized for: throughput, correctness, eventual consistency

READ PATH (sync, direct):
  User request → FastAPI → PostgreSQL (pgvector + AGE + tsvector + pg_trgm) → Response
  Optimized for: latency, availability
```

### Architecture Diagram

```mermaid
graph TB
    subgraph Frontend
        UI[Next.js 16 + Vercel AI SDK 5<br/>Wiki Viewer / Chat / API Playground<br/>Entity Graph Explorer / Admin Dashboard]
    end

    subgraph Python Backend
        API[FastAPI<br/>REST API / Chat SSE / Webhooks<br/>Admin API / MCP Server]
        AGENTS[Agents<br/>LangGraph: Chat / Cross-Repo / Wiki Gen / PR Context / Anomaly<br/>LangChain: Enrichment / API Catalog Builder]
        KAFKA_C[Kafka Consumers<br/>repo.pushed / repo.indexed<br/>entities.extracted / wiki.generated<br/>diagram.stale / api.discovered<br/>repo_group.membership_changed]
        JOBS[Background Jobs<br/>Poller / Diagram Consistency<br/>Anomaly Detection]
    end

    subgraph Java Extractor
        JDT[Spoon 11.x + Framework Parsers<br/>Tier 1 deterministic extraction<br/>Zero LLM]
    end

    subgraph PostgreSQL
        PG_REL[(Relational<br/>users / repos / groups / jobs)]
        PG_VEC[(pgvector<br/>embeddings)]
        PG_AGE[(Apache AGE<br/>entity graph)]
        PG_TRGM[(pg_trgm<br/>code search)]
        PG_TS[(tsvector<br/>wiki search)]
    end

    KAFKA[[Kafka - KRaft<br/>Event Bus]]

    subgraph Optional
        REDIS[(Redis<br/>cache)]
        S3[(Object Storage<br/>S3 / GCS)]
        KC[Keycloak<br/>OIDC / SAML]
    end

    UI -->|REST + SSE + WebSocket| API
    API --> AGENTS
    API --> KAFKA_C
    API --> JOBS
    AGENTS --> PG_REL
    AGENTS --> PG_VEC
    AGENTS --> PG_AGE
    KAFKA_C --> KAFKA
    KAFKA -->|repo.indexed| JDT
    JDT -->|entities.extracted| KAFKA
    JDT --> PG_REL
    KAFKA --> KAFKA_C
    API -->|optional| REDIS
    AGENTS -->|wiki content cloud mode| S3
    API -->|OIDC validate| KC
```

### Core Technology Decisions

| Decision | Default Choice | Why We Chose It | Swappable Via |
|---|---|---|---|
| Backend | Python (FastAPI) | One language for all intelligence + API. In-process agent tools mean zero network hops. | Web framework is not abstracted (core) |
| Frontend | Next.js 16 + Vercel AI SDK 5 | Best AI chat rendering library available. SSR for wiki pages. | Frontend is not abstracted (core) |
| Agent Framework | LangGraph 1.x + LangChain 1.x | GA stable API, stateful graphs, checkpointing, multi-step reasoning. | Agent framework abstraction (task interfaces) |
| Java Extractor | Java (Spoon 11.x) | JVM-native compiler-level analysis via Spoon metamodel (JDT underneath). The only JVM component in the stack. | Extractor plugin interface |
| Event Bus | Kafka 4.x (KRaft-only) | Durable log, replay, partition by repo_id, solid plugin ecosystem. ZooKeeper removed in 4.0. | Event bus abstraction (EventBus interface) |
| Database | PostgreSQL (pgvector + AGE + tsvector + pg_trgm) | One DB for everything. Each component is swappable via an abstraction layer. | Database abstraction (5 repository interfaces) |
| Auth | Built-in JWT + optional Keycloak/OIDC | Simple by default, enterprise-ready when you need it. | Any OIDC-compliant provider |
| LLM Providers | LangChain providers + optional LiteLLM proxy | Admin-configured. Direct or proxy mode. | LangChain provider interface or LiteLLM |
| Chat Memory | LangGraph 1.x graph state + PostgreSQL checkpointing (Deep Research only) | Ephemeral sessions for Quick Ask / Conversational (messages live in browser). Deep Research checkpoints to PostgreSQL for crash recovery. No Zep/Mem0 needed. | Agent framework abstraction |
| Object Storage | fsspec (local/S3/GCS/Azure) — wiki content only (cloud mode) | Cloud-optional. Local mode stores everything in PostgreSQL. Diagrams always live in PostgreSQL. | fsspec filesystem interface |

### Database Abstraction

PostgreSQL is our default, but every capability sits behind an interface:

| Capability | Default | Swappable To |
|---|---|---|
| Relational | PostgreSQL | MySQL, CockroachDB |
| Vector search | pgvector | Qdrant, Weaviate, Pinecone, Milvus |
| Graph queries | Apache AGE | Neo4j, Memgraph |
| Full-text search | tsvector | Elasticsearch, Meilisearch |
| Trigram code search | pg_trgm | Elasticsearch, Zoekt |

Application code talks to repository interfaces. Never to PostgreSQL directly.

### Event Bus Abstraction

Kafka is our default, but the event bus sits behind an interface:

| Capability | Default | Swappable To |
|---|---|---|
| Event publishing | Kafka producer (aiokafka / kafka-clients) | RabbitMQ, Redis Streams, NATS, AWS SQS/SNS, Google Pub/Sub |
| Event consuming | Kafka consumer groups | RabbitMQ consumers, Redis Stream groups, NATS subscriptions |
| Event ordering | Kafka partitions (by repo_id) | RabbitMQ consistent hash exchange, Redis Stream keys |
| Event replay | Kafka log retention | RabbitMQ stream queues, NATS JetStream |

Application code publishes and consumes events through an `EventBus` interface. Never through Kafka clients directly. Event contracts (JSON Schema in `contracts/`) remain the same regardless of the underlying transport.

For simpler deployments (small teams, single instance), an in-process event bus (Python asyncio queues) is a valid implementation — no external broker needed.

### Agent Framework Abstraction

LangGraph and LangChain are our defaults for agent orchestration and LLM abstraction, but agent logic sits behind task-level interfaces:

| Capability | Default | Swappable To |
|---|---|---|
| Complex agents (chat, wiki gen, research, cross-repo, anomaly) | LangGraph 1.x | CrewAI, AutoGen, LlamaIndex Workflows, custom orchestration |
| Simple agents (enrichment, API catalog) | LangChain 1.x | Direct LLM API calls, LlamaIndex, Haystack |
| LLM provider routing | LangChain providers + LiteLLM | Direct provider SDKs, vLLM, Ollama |
| Embedding generation | LangChain embeddings | sentence-transformers (local), provider APIs directly |

Each agent is defined by its **task interface** — inputs, outputs, and tools. The orchestration framework is an implementation detail. A wiki generator agent takes entity graph data and produces wiki markdown — whether it uses LangGraph or a custom state machine is behind the interface.

### Pipeline vs Agentic Split

| Component | Mode | Why |
|---|---|---|
| Repo Indexer | **Pipeline** | Consumes `repo.pushed`, clones/pulls repo to shared volume, computes git diff, stores source files, produces `repo.indexed`. This is the bridge between push events and extraction. |
| Java Extractor (Tier 1) | **Pipeline** | Deterministic AST analysis via Spoon. No LLM. Reads from shared clone volume. |
| Enrichment (Tier 2) | **Agent** (LangChain) | Triggered by `entities.extracted`. Runs in the Python backend. LLM fills gaps for unknown annotations. Simpler than LangGraph — no graph/checkpointing needed. |
| Embedding Worker | **Pipeline** | Chunks source code + wiki content, embeds via LLM provider, stores in pgvector. Subscribes to two topics: `entities.extracted` (code embeddings) and `wiki.generated` (wiki page embeddings). Separate consumer group from other consumers. |
| Wiki Generator | **Agent** (LangGraph) | Triggered by `entities.extracted`. Produces `wiki.generated` and `diagram.stale`. Generates wiki pages AND inline diagrams. Needs to plan structure and reason about content depth. |
| Background Diagram Regenerator | **Pipeline** | Deterministic: entity graph in, Mermaid out. Regenerates stale diagrams flagged by `diagram.stale` events. No LLM for most diagram types (some complex types use the wiki generator agent). |
| Chat (all modes) | **Agent** (LangGraph) | Adaptive search, multi-step reasoning. |
| Cross-Repo Analysis | **Agent** (LangGraph) | Structural + similarity + reasoning combined. |
| API Catalog Builder | **Agent** (LangChain) | Triggered by `entities.extracted`. Runs in parallel with Enrichment — works with Tier 1 framework-detected endpoints (Spring annotations, OpenAPI specs). Enrichment results for custom annotations are picked up on the next incremental update. Generates realistic example payloads. Produces `api.discovered`. Simpler than LangGraph — no graph/checkpointing needed. |
| PR Context Injector | **Agent** (LangGraph) | Reasons about blast radius severity. |
| Anomaly Detector | **Agent** (LangGraph) | Pattern detection, drift judgment. |

### Deployment Profiles

| Profile | Containers | Infrastructure |
|---|---|---|
| **Local** (`docker-compose up`) | 5: postgres, kafka, backend (Python), java-extractor, frontend (Next.js) | Single machine |
| **Local + OIDC** | 6: + keycloak | Single machine |
| **Cloud** | Backend + Frontend + Java Extractor on ECS/GKE | Managed: RDS, MSK, ElastiCache, S3 |

Everything is configuration-driven via environment variables:

| Setting | Options | Controls |
|---|---|---|
| `CODEGRAPH_MODE` | local / cloud | Storage, caching, infrastructure |
| `CODEGRAPH_STORAGE` | postgres / s3 / gcs / azure | Wiki markdown content storage (diagrams always in PostgreSQL) |
| `CODEGRAPH_CACHE` | postgres / redis | Caching layer |
| `CODEGRAPH_AUTH_MODE` | local / oidc | Authentication provider |
| `CODEGRAPH_LLM_MODE` | direct / proxy | LLM provider routing |
| `CODEGRAPH_OTEL_ENDPOINT` | URL or empty | OTel Collector endpoint. Empty = local file export only. |

All application settings (LLM config, repos, groups, routing) are configurable via the Admin Dashboard at runtime. No restarts needed.

### Local Setup

Getting CodeGraph running locally:

```bash
# 1. Clone the repo
git clone https://github.com/mogheyapoorv/codegraph.git
cd codegraph

# 2. Copy and edit environment file
cp .env.example .env
# Edit .env — at minimum set one LLM provider API key

# 3. Start everything
docker-compose up

# That's it. Five containers start:
#   postgres (pgvector + AGE + pg_trgm)
#   kafka (KRaft-only, single broker)
#   backend (Python — FastAPI + LangGraph)
#   java-extractor (JVM — Spoon 11.x)
#   frontend (Next.js)

# 4. Open http://localhost:3000/setup for first-time admin bootstrap
```

For OIDC/SSO, uncomment the keycloak service in docker-compose and set `CODEGRAPH_AUTH_MODE=oidc`.

### Kubernetes / Helm Chart

For cloud and production deployments, we provide a Helm chart:

```bash
# Add the CodeGraph Helm repo
helm repo add codegraph https://mogheyapoorv.github.io/codegraph/charts
helm repo update

# Install with default values (uses managed services)
helm install codegraph codegraph/codegraph \
  --namespace codegraph \
  --create-namespace \
  -f values.yaml
```

Key `values.yaml` settings:

```yaml
# Deployment mode
mode: cloud

# Managed PostgreSQL (RDS, Cloud SQL, etc.)
database:
  url: postgresql://user:pass@rds-instance:5432/codegraph

# Managed Kafka (MSK, Confluent Cloud, etc.)
kafka:
  bootstrapServers: msk-cluster:9092

# Object Storage for wiki content
storage:
  type: s3
  bucket: codegraph-data
  region: us-east-1

# Optional Redis for caching
cache:
  type: redis
  url: redis://elasticache:6379

# LLM configuration
llm:
  mode: direct
  provider: anthropic
  apiKeySecret: codegraph-llm-key  # K8s secret reference

# Scaling
backend:
  replicas: 2
javaExtractor:
  replicas: 2
frontend:
  replicas: 2

# Auth
auth:
  mode: oidc
  issuerUrl: https://keycloak.company.com/realms/codegraph
```

The Helm chart supports:
- Horizontal scaling per component (backend, java-extractor, frontend independently)
- Managed database, Kafka, Redis, and object storage
- K8s secrets for API keys and credentials
- Ingress configuration with TLS
- Resource limits and requests per container
- Health check and readiness probe configuration

---

**Next:** [Authentication & Repo Management](02-auth-and-repo-management.md) | [Index](00-index.md)
