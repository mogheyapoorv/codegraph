# CodeGraph Platform — Design Specification

**Date:** 2026-03-22
**Status:** Draft
**Scope:** CodeGraph is a new open-source code intelligence platform. We auto-generate wikis, entity graphs, API catalogs, and cross-repo diagrams for enterprise codebases. The project draws inspiration from deepwiki-open as a starting point, but this is a different codebase with a different direction.

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
        UI[Next.js 15 + Vercel AI SDK<br/>Wiki Viewer / Chat / API Playground<br/>Entity Graph Explorer / Admin Dashboard]
    end

    subgraph Python Backend
        API[FastAPI<br/>REST API / Chat SSE / Webhooks<br/>Admin API / MCP Server]
        AGENTS[LangGraph Agents<br/>Chat / Cross-Repo Analysis / Wiki Generator<br/>PR Context / Anomaly Detector / Enrichment<br/>API Catalog Builder]
        KAFKA_C[Kafka Consumers<br/>repo.pushed / repo.indexed<br/>entities.extracted / wiki.generated]
        JOBS[Background Jobs<br/>Poller / Diagram Consistency<br/>Anomaly Detection]
    end

    subgraph Java Extractor
        JDT[Eclipse JDT + Framework Parsers<br/>Tier 1 deterministic extraction<br/>Zero LLM]
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
    JDT -->|consumes repo.indexed| KAFKA
    JDT -->|produces entities.extracted| KAFKA
    JDT --> PG_REL
    KAFKA --> KAFKA_C
```

### Core Technology Decisions

| Decision | Choice | Why We Chose It |
|---|---|---|
| Backend | Python (FastAPI + LangGraph + LangChain) | One language for all intelligence + API. In-process agent tools mean zero network hops. |
| Frontend | Next.js 15 + Vercel AI SDK | Best AI chat rendering library available. SSR for wiki pages. |
| Chat Agent | LangGraph | Best-in-class planning, evaluation, checkpointing, and multi-step reasoning. |
| Java Extractor | Java (Eclipse JDT) | JVM-native compiler-level analysis. The only JVM component in the stack. |
| Event Bus | Kafka (KRaft) | Durable log, replay, partition by repo_id, solid plugin ecosystem. |
| Database | PostgreSQL (pgvector + AGE + tsvector + pg_trgm) | One DB for everything. Each component is swappable via an abstraction layer. |
| Auth | Built-in JWT + optional Keycloak/OIDC | Simple by default, enterprise-ready when you need it. |
| LLM Providers | LangChain providers + optional LiteLLM proxy | Admin-configured. Direct or proxy mode. |
| Chat Memory | LangGraph graph state + PostgreSQL checkpointing | Ephemeral sessions. No Zep/Mem0 needed. |
| Object Storage | fsspec (local/S3/GCS/Azure) — wiki content only (cloud mode) | Cloud-optional. Local mode stores everything in PostgreSQL. Diagrams always live in PostgreSQL. |

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

### Pipeline vs Agentic Split

| Component | Mode | Why |
|---|---|---|
| Java Extractor (Tier 1) | **Pipeline** | Deterministic AST analysis. No LLM involved. |
| Enrichment (Tier 2) | **Agent** (LangChain) | LLM fills gaps for unknown annotations. |
| Embedding Worker | **Pipeline** | Straightforward: chunk, embed, store. |
| Diagram Generator | **Pipeline** | Deterministic: entity graph in, Mermaid out. |
| Wiki Generator | **Agent** (LangGraph) | Needs to plan structure and reason about content depth. |
| Chat (all modes) | **Agent** (LangGraph) | Adaptive search, multi-step reasoning. |
| Cross-Repo Analysis | **Agent** (LangGraph) | Structural + similarity + reasoning combined. |
| API Catalog Builder | **Agent** (LangChain) | Generates realistic example payloads. |
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
| `CODEGRAPH_STORAGE` | local / s3 / gcs / azure | Wiki content + diagram storage |
| `CODEGRAPH_CACHE` | postgres / redis | Caching layer |
| `CODEGRAPH_AUTH_MODE` | local / oidc | Authentication provider |
| `CODEGRAPH_LLM_MODE` | direct / proxy | LLM provider routing |

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
#   kafka (KRaft, single broker)
#   backend (Python — FastAPI + LangGraph)
#   java-extractor (JVM — Eclipse JDT)
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

## 3. Authentication & Admin Bootstrap

### Auth Modes

| Mode | When | How |
|---|---|---|
| **Built-in email/password** | Small teams, quick setup, air-gapped | bcrypt in PostgreSQL, CodeGraph issues JWTs |
| **Keycloak (bundled)** | Self-hosted orgs wanting SSO | Ships in docker-compose, pre-configured OIDC |
| **External OIDC** | Orgs with existing Okta/Auth0/Azure AD | Point at any OIDC discovery URL |
| **SAML (via Keycloak)** | Enterprise IdP requirement | Keycloak bridges SAML to OIDC |

Our auth middleware validates JWTs regardless of issuer. Same middleware, different token source.

### First-Time Admin Bootstrap

1. First boot detects empty user table
2. Setup wizard at `http://localhost:3000/setup`
3. Wizard asks:
   - Admin email + password (break-glass admin, always built-in)
   - Auth mode: "Email/password" or "Configure OIDC"
   - First repo credential (GitHub App / PAT)
   - First repo to index (optional — gets you to value fast)
4. Admin logs in, lands on Admin Dashboard

**Automated deployment override (Helm/Terraform/Ansible):**

```
CODEGRAPH_ADMIN_EMAIL=admin@company.com
CODEGRAPH_ADMIN_PASSWORD=<secure-password>
CODEGRAPH_AUTH_MODE=oidc
CODEGRAPH_OIDC_ISSUER_URL=https://keycloak.company.com/realms/codegraph
```

If these are set, first boot skips the wizard and auto-configures.

### Roles

| Role | Can Do |
|---|---|
| **Superadmin** | Everything. Created at bootstrap. Break-glass account. |
| **Admin** | Manage repos, groups, credentials, LLM config, view analytics. |
| **Member** | Browse wikis, chat, view diagrams, view API catalog. |

### LLM Configuration (Admin Only)

Admins configure this in the dashboard:

| Setting | Options |
|---|---|
| Mode | Direct (provider APIs) or Proxy (LiteLLM/OpenAI-compatible) |
| Providers | Anthropic, OpenAI, Google, Ollama, Bedrock, Azure |
| Operation routing | Different model per operation (chat, wiki gen, research, embedding, etc.) |
| Cost cap | Daily/monthly LLM spend cap. Pauses LLM operations when hit. |

Users never see any of this. Admins control cost, quality, and provider choices centrally.

### User Connected Accounts

For external doc access (Confluence/Google Docs) when a user pastes a link in chat:

- User connects their Google/Atlassian account via OAuth (Profile > Connected Accounts)
- We use their token ONLY when they paste a link. Never proactively.
- Token stored encrypted per-user. Doc content is never stored — fetched live, used for that single response.
- User hasn't connected? Graceful degradation. We answer with code and wiki only.

---

## 4. Repo Management & Repo Groups

### Private Repo Access

| Method | Platform | How |
|---|---|---|
| **GitHub App** (recommended) | GitHub Cloud / Enterprise Server | Install on org, auto-discovers repos, tokens auto-rotate |
| **Service Account PAT** | GitHub / GitLab / Bitbucket | Machine user with read-only scope |
| **Deploy Key** | Any (per-repo) | SSH key, most restrictive |

### Repo Discovery Flow

1. Admin adds a credential (GitHub App / PAT)
2. CodeGraph discovers repos via the platform API
3. All discovered repos appear in Admin Dashboard as "Unassigned"
4. Admin assigns repos to groups or leaves them ungrouped
5. All repos queue for indexing regardless

### Repo Change Detection — Three Equal Triggers

| Trigger | Use Case | How |
|---|---|---|
| **Webhook** | Orgs that allow inbound hooks | GitHub/GitLab/Bitbucket push events, signature validated |
| **Polling** | Enterprises blocking inbound webhooks (first-class, not a fallback) | `git ls-remote` on configurable interval (default 6h), lightweight |
| **Manual** | On-demand, force full re-index | UI button + API endpoint |

All three produce the same `[repo.pushed]` Kafka event. Downstream workers don't know or care how the trigger happened.

### Repo Groups

**Purpose:** Logical architecture grouping for exactly two things:
1. **Scoped Q&A** — chat searches within a group's repos
2. **Cross-repo diagrams** — architecture diagrams across repos in a group

**Key rules:**
- Repos work fully without any group (wiki, per-repo chat, API catalog, diagrams)
- A repo can belong to multiple groups
- Deleting a group doesn't delete repos
- Any authenticated user can browse any group
- Groups are created manually by admin — no HR/team system integration
- Grouping is optional. Can happen before, during, or after indexing.

### Bulk Indexing (1000 repos)

When admin adds many repos at once:

- **Priority:** P0 webhook (real-time) > P1 manual > P2 bulk discovery
- **Concurrency:** Admin-configurable max concurrent jobs (default: 5)
- **Scheduling:** Small repos first within queue (fast wins)
- **Cost control:** Daily LLM cost cap. Pipeline extraction (no LLM) keeps going regardless.
- **Failure handling:** Failed jobs don't block the queue. Retry separately.
- **Admin controls:** Pause, resume, cancel, reorder from the dashboard.
- **No dependency on groups:** Every repo indexes independently. Groups are labels, not scheduling units.

### Scoped Search

| Scope | What It Searches |
|---|---|
| **Repo** | Single repo |
| **Group** | All repos in one group |
| **Cross-Group** | User selects multiple groups |
| **Org** | All indexed repos |

All search tools (entity graph, vector, trigram, wiki, API catalog) respect the selected scope via SQL WHERE filters.

### Cross-Repo Entity Resolution

Runs after individual repos are indexed. Triggers when:
- A repo finishes indexing that belongs to a group
- Admin assigns repos to a group

This doesn't run as part of individual repo indexing — it's a group-level concern handled by the background consistency job.

---

## 5. Entity Graph & Java Extraction

### Entity Graph Data Model

**CodeEntity (nodes):**
- id, repo_id, entity_type, name, fully_qualified_name
- file_path, line_start, line_end, language, signature
- confidence (1.0 = AST, 0.7-0.9 = LLM gap-fill)
- discovered_by (ast_parse, framework_parser, llm_gap_fill)

**Entity types:**
- Code-level: function, class, method, interface, enum, module, file
- API surface: rest_endpoint, grpc_service, grpc_method, graphql_query
- Data layer: db_table, db_column, db_migration, message_topic, cache_key
- Infrastructure: docker_service, env_variable, config_key, cron_job, feature_flag

**EntityRelation (edges):**
- source_entity_id, target_entity_id, relation_type
- is_cross_repo, confidence, discovered_by, evidence_file, evidence_line

**Relation types:**
- Code: imports, calls, extends, implements, instantiates, depends_on
- API: exposes_api, consumes_api, shares_proto
- Data: reads_table, writes_table, publishes_to, subscribes_to, uses_config
- Implicit: co_changes_with (git-derived)

**Source storage:**
- `source_files` — full file content in PostgreSQL (searchable via pg_trgm)
- `code_snippets` — per-entity source code linked to entities

**Graph queries via Apache AGE (Cypher):**
- Forward traversal: "What does PaymentService call?"
- Reverse traversal: "What depends on PaymentGateway?" (blast radius)
- Cross-repo edges: "Who consumes payment-service APIs?"
- Path finding: "What's the chain from user signup to first email?"

### Java Extraction — Tiered Approach

**Tier 1: Deterministic Multi-Analyzer (always runs, $0, 70-95% coverage)**

Three analyzers run in parallel, and a consensus engine merges the results:

**Analyzer 1 — Eclipse JDT (AST + Type Resolution):**
- Parses all Java files in one batch with full type resolution
- Classpath from downloaded Maven/Gradle JARs (dependency download only, no compilation)
- Produces: classes, methods, fields, annotations, call graph, type hierarchy, import graph
- Annotation scanning: classifies every annotation as known-framework or custom
- Lombok plugin for desugaring (no compilation needed)

**Why Eclipse JDT?** We evaluated the alternatives:

| Tool | Type Resolution | No Build Required? | Why Not |
|---|---|---|---|
| **Eclipse JDT** | 95%+ (compiler-grade) | Yes (source + JARs) | **Our choice.** |
| **Spoon (INRIA)** | Same as JDT (built on top of it) | Yes, with NOCLASSPATH mode | Viable alternative. Cleaner API, but extra abstraction layer. We may switch to Spoon if JDT's API verbosity becomes a maintenance issue. |
| **JavaParser + Symbol Solver** | 60-80% (known gaps with generics, lambdas, overloads) | Yes | Resolution accuracy too low for reliable call graphs on enterprise code. |
| **IntelliJ PSI** | Best-in-class | Impractical outside IDE | Can't be extracted from the IntelliJ platform. JetBrains themselves say it's not supported. |
| **tree-sitter** | None (syntax only) | Yes | No semantic analysis. Useful for other languages where no better parser exists, but not for Java. |
| **Soot/SootUp** | Full (bytecode only) | No — needs compilation | Best call graph construction, but requires compiled .class files. We can't reliably compile arbitrary repos. |

JDT gives us compiler-grade type resolution from source without compilation. That's the hard requirement. The 20+ years of production use and active maintenance (1K+ commits/year) seal it. Spoon is our fallback if we want a cleaner API — it uses JDT underneath so the resolution quality is identical.

**Analyzer 2 — Framework-Specific Parsers:**
- Spring annotations → beans, endpoints, DI wiring
- JPA annotations → entities, tables, relations
- Proto files → gRPC services, methods (parsed directly, not compiled)
- OpenAPI spec (if checked in) → full API catalog
- SQL migrations (Flyway/Liquibase) → actual DB schema
- application.yml / application.properties → config keys with values
- pom.xml / build.gradle → dependency graph

**Analyzer 3 — File-Level Analyzers:**
- Dockerfile → docker_service entities
- docker-compose.yml → service topology
- K8s manifests → deployment topology
- .github/workflows → CI/CD pipeline
- CODEOWNERS → ownership
- README, ADRs → documentation references

**Consensus Engine (entity confidence — how certain we are about this entity's classification):**
- Merges all analyzer outputs, cross-references results
- Multiple analyzers agree → confidence 1.0
- Two agree → confidence 0.9
- Single source → confidence 0.8
- Cross-reference: does the JPA entity match a SQL migration table? Does the endpoint match the OpenAPI spec?

Note: entity confidence (from extraction) and relation confidence (from cross-repo resolution, Section 6) measure different things. Entity confidence = "how certain is this entity's type?" Relation confidence = "how certain is this cross-repo connection?"

**No compilation required.** JDT resolves types from source + downloaded JARs. Framework parsers read annotations and config files directly. Works on any repo.

**Tier 2: LLM Gap-Fill (automatic, fills the remaining 5-30%)**

For entities that Tier 1 couldn't classify (custom annotations, internal frameworks):

1. Heuristic classification first (meta-annotation analysis, naming conventions, package patterns) — catches ~80% of unknowns
2. Remaining truly unknown patterns → LLM reads the annotation/class definition source and reasons about its purpose
3. Tagged as `discovered_by: llm_gap_fill` with a confidence score
4. **Fully automatic** — no human review, no admin intervention
5. **No LLM key configured?** Tier 2 gets skipped. Tier 1 still gives you 70-95% coverage. Graceful degradation.

**Repo context understanding:**
- The extractor doesn't rely on generic pattern files
- It reads THIS repo's build files to know exact frameworks and versions
- Scans all annotations used, classifies known vs custom
- For custom: reads the annotation's source definition, meta-annotations, usage patterns
- Caches repo understanding for incremental runs — re-analyzes conventions only when build files or framework source changes

**No entity-level descriptions generated during extraction.** The wiki generator handles all natural language content downstream.

### Static Risk Analysis

Deterministic rules (SpotBugs/PMD-style patterns), no LLM:
- Empty catch blocks
- Unauthenticated public endpoints
- No null checks on external API returns
- High cyclomatic complexity
- Missing test files for entities
- Hardcoded strings matching credential patterns

### API Catalog & curl/grpcurl Generation

Every discovered API endpoint gets runnable request examples:

- REST endpoints → curl command with example payload
- gRPC services → grpcurl command with example payload
- GraphQL → example query

**Example payload generation:** The API Catalog Builder agent (LangChain) generates realistic values from field names and types (amount → 99.99, email → user@example.com, currency → "USD"). It uses actual type definitions from the codebase.

**Request/response schemas** come from the Java type system — the extractor captures full type information including nested objects, enums, and generics.

### Incremental Updates

Here's what actually happens on a push (via webhook, polling, or manual trigger):

1. `git diff old_sha..new_sha` → list of added/modified/deleted files
2. **Deleted files:** remove all entities, relations, snippets, source content, embeddings for that file (CASCADE)
3. **Modified files:** delete + re-insert everything for that file (idempotent)
4. **Added files:** insert new entities, snippets, source, embeddings
5. **Build file changed (pom.xml/build.gradle):** diff dependencies — if a framework changed, full re-extraction. Version bump only? No re-extraction.
6. Cross-repo resolution: re-run for affected entities only
7. Wiki pages referencing changed entities → mark stale → wiki generator regenerates
8. Diagrams including changed entities → mark stale → background job regenerates
9. API catalog entries for changed endpoints → regenerate curl/grpcurl

**Typical push (5-10 files):** seconds to update. **Framework upgrade:** 5-30 minutes for full re-extraction.

### Language Extractor Plugin Model

Future extractors (Ruby, Python, TypeScript) follow the same contract:

- Docker container (Java) or Python module (all others) that consumes `[repo.indexed]` from Kafka
- Checks for matching files (*.rb, *.py, *.ts)
- Runs language-native analysis tools (tree-sitter, Pyright, TS Compiler API)
- Same tiered approach: deterministic first, LLM gap-fill for unknowns
- Produces `[entities.extracted]` matching the same JSON Schema
- Downstream workers process identically regardless of source language

Java extractor is Level 3 (complete ecosystem). Future languages start at Level 1 (basic) and grow via community contribution.

Only Java needs a separate JVM container. All other language extractors are Python modules in the main backend.

---

## 6. Wiki Generation & Diagrams

### Wiki Generator Agent (LangGraph)

Consumes `[entities.extracted]` from Kafka. Plans and generates wiki pages from the entity graph + code snippets + source files.

**How it works:**

1. **Analyze** — reads entity graph structure for the repo (how many endpoints, entities, services, jobs, etc.)
2. **Plan** — decides wiki structure based on complexity (simple repo → single page, complex service → 5-8 pages, large monorepo → per-module sections)
3. **Generate** — for each page, queries the entity graph, reads code snippets, generates markdown with narrative, Mermaid diagrams, code examples, curl/grpcurl, and file:line citations

**Wiki page types:**
- Overview (architecture, key entry points)
- API Reference (REST + gRPC endpoints with curl/grpcurl)
- Data Model (JPA entities, ER diagram, DB schema from migrations)
- Event System (Kafka topics, producers, consumers, flow diagram)
- Scheduled Jobs
- Configuration (all config keys from application.yml)
- Dependencies (upstream/downstream services)

### 20 Diagram Types

Generated from entity graph data. Most are deterministic Mermaid generation from graph queries — no LLM needed. Some complex types (State Machine, Migration Path, Architecture Evolution, Technical Debt Map) require agent reasoning and are generated by the Wiki Generator agent.

**Structural:**

| # | Type | Shows |
|---|---|---|
| 1 | Architecture Diagram | Services, connections, DBs, queues |
| 2 | Dependency Graph | Repo-to-repo package dependencies |
| 3 | API Surface Map | Endpoints exposed and consumed |
| 4 | Data Flow Diagram | How data moves between services |
| 5 | Entity Relationship Diagram | DB tables, relations, foreign keys |
| 6 | Deployment Topology | How services are deployed |

**Behavioral:**

| # | Type | Shows |
|---|---|---|
| 7 | Sequence Diagram | Request flow across services |
| 8 | State Machine Diagram | Domain object lifecycles |
| 9 | Error/Failure Cascade | "If X goes down, what fails?" |
| 10 | Impact Blast Radius | Concentric rings from a changed entity |
| 11 | Migration/Refactor Path | Current → target state with steps |

**Git History:**

| # | Type | Shows |
|---|---|---|
| 12 | Change Heatmap | Architecture colored by churn intensity |
| 13 | Co-Change Coupling Map | Hidden dependencies between files/repos |
| 14 | Expertise/Ownership Map | Top contributors per module |
| 15 | Bus Factor Diagram | Single-person knowledge risk areas |
| 16 | Architecture Evolution Timeline | How the system grew/changed |

**Quality & Security:**

| # | Type | Shows |
|---|---|---|
| 17 | Security Boundary Diagram | Trust boundaries, attack surface |
| 18 | Test Coverage Overlay | Architecture x coverage = risk map |
| 19 | Technical Debt Map | Highest-impact refactoring targets |
| 20 | API Version Compatibility Matrix | Safe to deprecate? Check consumers. |

### Cross-Repo Diagrams (Group Scope)

Generated when repos are in a group. Built from cross-repo entity relations spanning multiple repos.

**Triggering:** We don't generate these on every push. Instead:
- Events (repo pushed, repo added/removed from group) → mark group diagrams as **stale**
- **Background consistency job (every 5 min)** checks for stale diagrams:
  1. Any repos in this group currently indexing? → skip, wait for next cycle
  2. All repos stable → run cross-repo resolution → regenerate stale diagrams
- Diagrams are **eventually consistent** within one job cycle
- Manual trigger available in Admin Dashboard

### Cross-Repo Resolution — Hybrid Approach

We combine three layers for maximum accuracy:

**Layer 1 — Structural Signals (deterministic, high confidence):**
- Contract matching: shared .proto files, OpenAPI specs, Kafka topic strings
- Annotation matching: @FeignClient naming, gRPC stub imports
- Config matching: service URLs in application.yml, shared datasource URLs
- Infrastructure matching: docker-compose links, K8s service definitions

**Layer 2 — Similarity Signals (vector-based, medium confidence):**
- Embedding similarity between API client code and endpoint handler code
- DTO field overlap across repos (similar data types = likely shared contract)
- Class/method name similarity across repos
- Unresolved import matching (JDT couldn't resolve → match to class in another repo)

**Layer 3 — Reasoning Agent (LangGraph, validates and discovers):**
- Takes Layer 1 + Layer 2 signals plus code snippets from both sides
- Validates: is this a real connection or a coincidence?
- Rejects false positives (same class name, different purpose)
- Discovers connections both layers missed
- Enriches: describes the nature of each connection (sync call, async event, shared data)

**Multi-signal confidence scoring:**
- All 3 layers agree → 1.0
- Structural + one other → 0.9-0.95
- Similarity + agent confirmed → 0.8-0.85
- Single weak signal → 0.5-0.6 (excluded from diagrams)
- Diagram inclusion threshold: >= 0.8 (solid line), 0.6-0.8 (dashed "inferred" line), < 0.6 (excluded)

### Sequence Diagram Generation

Entry points are auto-detected:
- REST endpoints on API gateway / BFF service
- Public endpoints (exposed in OpenAPI spec)
- Webhook receivers
- Scheduled jobs
- Message consumers with no internal producer

The agent traces the call chain from each entry point: internal call graph → hits API client (boundary entity) → matches to target repo endpoint → continues tracing in target repo → marks async boundaries (Kafka) → continues until no more cross-repo hops.

The agent picks 5-10 key flows per group (cross 3+ repos, involve data writes, show async boundaries) — not one per endpoint.

### Wiki Storage

- Wiki metadata (page structure, versions, generation status) → PostgreSQL
- Wiki content (markdown) → PostgreSQL (local mode) or Object Storage via fsspec (cloud mode)
- Diagrams (Mermaid source) → PostgreSQL (`wiki_diagrams` table)
- Versioning: `wiki_page_versions` table tracks every regeneration with source commit SHA

### Human-in-the-Loop (Wiki Annotations)

Users can annotate wiki pages. Annotations survive regeneration:

- Annotations are attached to entities (not wiki text) — stored in `wiki_annotations` table
- Types: note, warning, correction, context
- Displayed alongside auto-generated content
- When wiki regenerates, auto-generated content updates but annotations persist
- Example: team adds "Rate limited to 100 req/min in production" note to an endpoint's wiki section

### On-the-Fly Diagrams

The chat agent generates custom Mermaid diagrams during conversation — these aren't pre-stored. The agent queries the entity graph, formats results as Mermaid inline in the response. Unlike stored diagrams: these are custom to the specific question, not stored, zero additional LLM cost (Mermaid from graph data).

---

## 7. Chat & Deep Research

### Three Modes, One Agent, Different Leash

| Mode | Max Steps | Timeout | Plan Approval | Checkpointing |
|---|---|---|---|---|
| Quick Ask | 5 | 30s | No | No |
| Conversational | 15 | 120s | No | No |
| Deep Research | 50 | 600s | Yes | Yes (PostgreSQL) |

Same LangGraph agent, same tools. Only the configuration differs.

### Ephemeral Sessions

- No cross-session memory. Messages live in the browser (`useChat` hook).
- Tab closes = session gone.
- Wiki changes make old conversations stale anyway — no reason to persist.
- Deep Research results get saved as wiki pages (the output persists, not the conversation).

### Chat Agent Tools (In-Process Python Functions)

| Tool | Data Source | What It Finds |
|---|---|---|
| `entity_graph_query` | AGE (Cypher) | Named entities, relations, blast radius, call chains |
| `code_snippet_read` | code_snippets table | Actual source code for a specific entity |
| `code_search` | pg_trgm on source_files | Exact strings — error messages, variable names, TODOs |
| `vector_search` | pgvector | Semantically similar code and wiki content |
| `wiki_search` | tsvector | Wiki page content |
| `api_catalog_query` | api_endpoints table | Endpoints with curl/grpcurl examples |
| `git_metadata_query` | git_metadata table | Commit history, blame, recent changes |
| `generate_diagram` | Entity graph → Mermaid | On-the-fly diagrams |

All tools are Python functions querying PostgreSQL directly. Zero network hops.

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

### Slack / Teams Bot

CodeGraph answers questions in messaging channels. Same chat agent (Quick Ask mode). Admin configures bot token + default scope per channel.

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

## 9. Analytics & Cost Tracking

### Usage Analytics

Tracked in PostgreSQL:
- Wiki views (most viewed repos, pages, diagrams)
- Chat usage (queries/day, scope distribution, mode distribution)
- Search patterns (most common queries)
- API catalog usage (most viewed endpoints, most copied curl commands)
- MCP tool calls from IDEs
- PR context comments posted

### LLM Cost Tracking

Every LangChain/LangGraph LLM call automatically captures token usage via callback handler:
- Provider, model, prompt tokens, completion tokens → compute cost from admin-configured per-model pricing
- Aggregated per operation type, per group, per day/week/month
- Admin sets daily/monthly cost cap → system pauses LLM operations when hit. Pipeline extraction (no LLM) keeps going.
- If using LiteLLM proxy, cost data comes automatically.

### Codebase Health Scores

Per-repo and per-group metrics, all derived from existing data (no additional indexing or LLM):

| Metric | Source |
|---|---|
| Documentation coverage | % of entities with wiki sections |
| Wiki staleness | % of pages marked stale |
| Orphaned code | Entities with zero incoming edges |
| Test coverage gaps | Entities with no corresponding test file |
| Complexity hotspots | High complexity + high blast radius |
| Cross-repo coupling | Repos with too many cross-repo edges |
| Bus factor | Modules with single-contributor knowledge |
| Change frequency | Most changed files in last 90 days |

---

## 10. Phasing — Build Order

### Phase 1: Foundation

- PostgreSQL setup (pgvector + AGE + tsvector + pg_trgm)
- Python backend (FastAPI), auth (built-in JWT), admin API
- Kafka setup (KRaft, core event topics)
- Java Extractor (Eclipse JDT, framework parsers, Tier 1)
- LLM integration (LangChain providers, admin config)
- Tier 2 enrichment (LangChain gap-fill agent)
- Basic wiki generation (LangGraph wiki generator, single-repo)
- Frontend (Next.js, wiki viewer, admin dashboard, setup wizard)
- Docker Compose (4-container local deployment)

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

## 11. Tech Stack Summary

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, React 19, TypeScript, Vercel AI SDK, Tailwind CSS |
| Backend | Python 3.12+, FastAPI, LangGraph, LangChain |
| Java Extractor | Java 21, Eclipse JDT, Framework parsers |
| Database | PostgreSQL 16+ (pgvector, Apache AGE, tsvector, pg_trgm) |
| Event Bus | Apache Kafka (KRaft mode) |
| Cache | Redis (optional, cloud only) |
| Auth | Built-in JWT + optional Keycloak (OIDC/SAML) |
| LLM Providers | LangChain (Anthropic, OpenAI, Google, Ollama, Bedrock, Azure) + optional LiteLLM proxy |
| Object Storage | fsspec (local / S3 / GCS / Azure) — wiki content + diagrams only |
| Diagrams | Mermaid (client-side rendering) |
| Git Operations | gitpython / subprocess |
| Deployment | Docker Compose (local), Kubernetes + Helm (cloud) |

### What CodeGraph Builds vs Adopts

| Category | We Build (Core IP) | We Adopt (Battle-Tested) |
|---|---|---|
| Entity graph engine | Extraction pipeline, consensus engine, cross-repo resolution | PostgreSQL + AGE, Eclipse JDT |
| Wiki generation | Wiki agent, page planning, content generation | LangGraph, LangChain, Mermaid |
| Chat intelligence | Adaptive search, graph-grounded verification, scope-aware retrieval | LangGraph, pgvector, pg_trgm |
| API Catalog | Endpoint discovery, curl/grpcurl generation, payload examples | OpenAPI parser, protobuf parser |
| Plugin model | Event contracts, extractor interface | Kafka, Docker |
| Auth | JWT middleware, setup wizard | Keycloak, OIDC standard |

---

## 12. Data Model

### Core Tables

**users:**
- id (UUID PK), email, name, avatar_url, password_hash (nullable), role (superadmin/admin/member), auth_provider (local/oidc), oidc_issuer, oidc_subject, is_active, created_at, last_login

**user_connected_accounts:**
- id (UUID PK), user_id (FK), provider (google/atlassian/notion), encrypted_access_token, encrypted_refresh_token, token_expires_at, external_email, connected_at

**repo_credentials:**
- id (UUID PK), name, platform (github/gitlab/bitbucket), credential_type (github_app/pat/deploy_key), instance_url, encrypted_token, encrypted_key, app_id, installation_id, scope_type (org/group/repo), scope_value, created_by (FK), expires_at, last_used_at, is_active, created_at

**repositories:**
- id (UUID PK), platform, owner, name, url, default_branch, is_private, credential_id (FK), index_status (pending/indexing/indexed/failed), last_indexed_at, last_commit_sha, created_at
- UNIQUE(platform, owner, name)

**repo_groups:**
- id (UUID PK), name, slug (UNIQUE), description, icon, color, created_by (FK), created_at

**repo_group_memberships:**
- repo_group_id (FK), repo_id (FK), added_by (FK), added_at
- PK(repo_group_id, repo_id)

### Entity Graph Tables

**code_entities:**
- id (UUID PK), repo_id (FK), entity_type, name, fully_qualified_name, file_path, line_start, line_end, language, signature, confidence, discovered_by, last_commit_sha, created_at, updated_at
- Indexes: (repo_id, entity_type), (fully_qualified_name)

**entity_relations:**
- id (UUID PK), source_entity_id (FK CASCADE), target_entity_id (FK CASCADE), relation_type, is_cross_repo, confidence, discovered_by, evidence_file, evidence_line, created_at
- Indexes: (source_entity_id, relation_type), (target_entity_id, relation_type), (is_cross_repo) WHERE is_cross_repo = TRUE

**group_cross_repo_relations:**
- id (UUID PK), repo_group_id (FK CASCADE), source_entity_id (FK CASCADE), target_entity_id (FK CASCADE), relation_type, confidence, discovered_at

### Source Storage Tables

**source_files:**
- id (UUID PK), repo_id (FK), file_path, content (TEXT), language, size_bytes, commit_sha, updated_at
- Index: GIN (content gin_trgm_ops) for trigram search
- Index: (repo_id, file_path)

**code_snippets:**
- id (UUID PK), entity_id (FK CASCADE), repo_id (FK), file_path, line_start, line_end, content (TEXT), language, updated_at

### Embedding Tables

**knowledge_chunks:**
- id (UUID PK), repo_id (FK), source_id (file path), content (TEXT), embedding (vector), chunk_index, source_type (code/wiki), last_modified, created_at
- Index: IVFFlat (embedding vector_cosine_ops)
- Index: (repo_id)

### Wiki Tables

**wiki_pages:**
- id (UUID PK), repo_id (FK), title, slug, content (TEXT, local mode), content_path (cloud mode), parent_page_id (FK self), page_order, generation_status (generating/generated/stale/failed), source_entities (JSONB), source_commit_sha, version, created_at, updated_at

**wiki_page_versions:**
- id (UUID PK), wiki_page_id (FK), version, content (TEXT), content_path, source_commit_sha, generated_at

**wiki_annotations:**
- id (UUID PK), wiki_page_id (FK), entity_id (FK nullable), section_anchor, content (TEXT), annotation_type (note/warning/correction/context), created_by (FK), created_at, updated_at

**wiki_diagrams:**
- id (UUID PK), scope_type (repo/group), scope_id, diagram_type, mermaid_content (TEXT), generated_from (JSONB), is_stale, stale_reason, stale_since, generated_at, updated_at

### API Catalog Tables

**api_endpoints:**
- id (UUID PK), entity_id (FK), repo_id (FK), protocol (rest/grpc/graphql), http_method, path, service_name, method_name, request_type, response_type, request_schema (JSONB), response_schema (JSONB), auth_required, auth_type, headers (JSONB), curl_example (TEXT), grpcurl_example (TEXT), graphql_example (TEXT), created_at, updated_at

### Job Tables

**indexing_jobs:**
- id (UUID PK), repo_id (FK), priority (0=webhook/1=manual/2=bulk), status (queued/running/completed/failed/paused), trigger_type (webhook/poll/manual/bulk_discovery), batch_id, stage (cloning/extracting/embedding/generating_wiki), files_total, files_processed, commit_sha, error_message, retry_count, llm_tokens_used, llm_cost_usd, queued_at, started_at, completed_at

**webhook_configs:**
- id (UUID PK), repo_id (FK), platform, webhook_url, encrypted_secret, is_active, last_received_at

**poll_schedules:**
- id (UUID PK), repo_id (FK), interval_seconds (default 21600), last_polled_at, next_poll_at, is_active

**background_job_runs:**
- id (UUID PK), job_type (diagram_consistency/cross_repo_resolution/anomaly_scan), status, groups_processed, diagrams_regenerated, started_at, completed_at, error_message

### Git Metadata Tables

**git_metadata:**
- id (UUID PK), repo_id (FK), file_path, last_commit_sha, last_commit_date, last_author, commit_count_90d, unique_contributors, created_at, updated_at

### Config Tables

**llm_provider_configs:**
- id (UUID PK), mode (direct/proxy), provider_type, base_url, encrypted_api_key, model_pricing (JSONB), is_active, created_by (FK)

**model_routing_rules:**
- id (UUID PK), operation_type (chat_quick/chat_conversational/deep_research/wiki_generation/diagram/embedding/extraction_enrichment), provider_config_id (FK), model_name, priority, is_active

### Analytics Tables

**llm_usage_log:**
- id (UUID PK), operation_type, provider, model, prompt_tokens, completion_tokens, cost_usd, repo_id, repo_group_id, user_id, created_at

**wiki_view_events:**
- id (UUID PK), wiki_page_id (FK), repo_id, user_id, session_id, created_at

**chat_analytics:**
- id (UUID PK), session_id, mode, scope_type, scope_id, query_text, tool_calls_count, total_tokens, cost_usd, latency_ms, user_id, created_at

### Integration Tables

**pr_context_comments:**
- id (UUID PK), repo_id (FK), platform, pr_number, pr_url, comment_id, changed_files (JSONB), blast_radius (JSONB), cross_repo_impact (JSONB), comment_content (TEXT), posted_at

**api_keys:**
- id (UUID PK), user_id (FK), name, key_hash, key_prefix, scopes (JSONB), rate_limit_rpm, last_used_at, expires_at, is_active, created_at

**integration_configs:**
- id (UUID PK), integration_type (slack/teams/webhook), config (JSONB), created_by (FK), is_active, created_at

**deep_research_reports:**
- id (UUID PK), user_id (FK), title, scope_type, scope_id, query, plan (JSONB), findings (JSONB), report_content (TEXT), diagrams (JSONB), sources_used (JSONB), total_tokens, total_cost_usd, saved_as_wiki_page_id (FK nullable), created_at

---

## 13. Kafka Event Contracts

All events use JSON. Schemas live in the `contracts/` directory.

### repo.pushed

```json
{
  "event_type": "repo.pushed",
  "repo_id": "uuid",
  "platform": "github|gitlab|bitbucket",
  "owner": "acme",
  "name": "payment-service",
  "branch": "main",
  "old_sha": "abc123",
  "new_sha": "def456",
  "trigger": "webhook|poll|manual",
  "timestamp": "ISO-8601"
}
```

### repo.indexed

```json
{
  "event_type": "repo.indexed",
  "repo_id": "uuid",
  "commit_sha": "def456",
  "repo_path": "/tmp/repos/acme/payment-service",
  "changed_files": [
    {"path": "src/RetryHandler.java", "status": "modified"},
    {"path": "src/CircuitBreaker.java", "status": "added"},
    {"path": "src/OldRetry.java", "status": "deleted"}
  ],
  "is_full_reindex": false,
  "language_hints": ["java"],
  "timestamp": "ISO-8601"
}
```

### entities.extracted

```json
{
  "event_type": "entities.extracted",
  "repo_id": "uuid",
  "commit_sha": "def456",
  "extractor": "java|ruby|python|typescript",
  "entities_added": 5,
  "entities_updated": 12,
  "entities_removed": 2,
  "relations_added": 8,
  "relations_removed": 1,
  "api_endpoints_affected": ["uuid1", "uuid2"],
  "timestamp": "ISO-8601"
}
```

### wiki.generated

```json
{
  "event_type": "wiki.generated",
  "repo_id": "uuid",
  "pages_generated": 7,
  "pages_updated": 3,
  "commit_sha": "def456",
  "timestamp": "ISO-8601"
}
```

### diagram.stale

```json
{
  "event_type": "diagram.stale",
  "scope_type": "repo|group",
  "scope_id": "uuid",
  "diagram_ids": ["uuid1", "uuid2"],
  "reason": "repo_updated|membership_changed",
  "triggered_by_repo_id": "uuid",
  "timestamp": "ISO-8601"
}
```

### api.discovered

```json
{
  "event_type": "api.discovered",
  "repo_id": "uuid",
  "endpoints_added": 3,
  "endpoints_updated": 1,
  "endpoints_removed": 0,
  "timestamp": "ISO-8601"
}
```

### repo_group.membership_changed

```json
{
  "event_type": "repo_group.membership_changed",
  "group_id": "uuid",
  "repo_id": "uuid",
  "action": "added|removed",
  "group_repo_ids": ["uuid1", "uuid2", "uuid3"],
  "timestamp": "ISO-8601"
}
```

---

## 14. Anomaly Detection

### What It Detects

The Anomaly Detector agent (LangGraph) periodically scans the entity graph and wiki for inconsistencies:

| Anomaly Type | How We Detect It |
|---|---|
| **Documentation drift** | Wiki says "service uses PostgreSQL" but entity graph shows MongoDB driver imports |
| **Dead endpoints** | API endpoint exists in entity graph but no consumer in any group repo |
| **Shadow dependencies** | Service A calls Service B via hardcoded URL not in any config or annotation |
| **Convention violations** | All services use shared-protos except one that defines its own |
| **Orphaned code** | Entities with zero incoming edges (nothing calls them) |
| **Stale wikis** | Wiki pages marked stale for more than 7 days (regeneration failed or backlogged) |
| **Missing tests** | High blast-radius entities with no corresponding test file |
| **Config drift** | Same config key with different values across repos in a group |

### How It Differs From Static Risk Analysis

Static Risk Analysis (Section 5) runs during extraction — per-file, deterministic rules (empty catch blocks, etc.).

Anomaly Detection runs as a **background agent** on a schedule — cross-repo, cross-wiki, pattern-based. It reasons across the entire entity graph and wiki state, not individual files. This is the hard part.

### Trigger

Scheduled background job (configurable, default: daily). Can also be manually triggered from the Admin Dashboard.

### Output

Anomalies show up in the Admin Dashboard under the "Anomalies" tab. Each anomaly has: type, severity, affected entities, description, and suggested action.

---

## 15. Rate Limiting & Abuse Protection

### Per-User Limits

| Mode | Rate Limit | Concurrent |
|---|---|---|
| Quick Ask | 60 requests/min | 5 |
| Conversational | 30 requests/min | 3 |
| Deep Research | 5 requests/hour | 1 |

### Cost Protection

- **Per-user daily cost cap** (optional, admin-configurable): prevents one user's Deep Research binge from exhausting the org budget
- **Global daily/monthly cost cap** (required): pauses all LLM operations when hit
- **Deep Research cost estimate**: shown to user before approval ("This research will use approximately $X in LLM costs")

### API Rate Limiting

Per API key: configurable RPM (requests per minute) set when key is created. Default: 60 RPM.

---

## 16. Observability

### Structured Logging

All components emit structured JSON logs:
- Request ID propagated across Kafka events and agent steps
- Log levels: DEBUG, INFO, WARNING, ERROR
- Key fields: timestamp, service, request_id, repo_id, operation, duration_ms, error

### Health Checks

- `GET /health` — backend liveness
- `GET /health/ready` — readiness (PostgreSQL connected, Kafka connected)
- Kafka consumer lag monitoring
- Background job last-run tracking

### Metrics

Key operational metrics (exportable to Prometheus/Grafana):
- Indexing job success/failure rates
- Kafka consumer lag per topic
- LLM call latency per provider per model
- Chat response latency (P50, P95, P99)
- Entity graph size and growth trends
- Active users, concurrent chat sessions

### Distributed Tracing

OpenTelemetry integration for tracing across:
- Kafka event chains (repo.pushed → repo.indexed → entities.extracted → wiki.generated)
- Agent step execution (LangGraph states)
- Tool call latency within agents

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
