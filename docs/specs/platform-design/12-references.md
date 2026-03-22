# CodeGraph Platform — External References

All external libraries, tools, standards, and academic references used in this spec. Organized by role in the system.

---

## Core Libraries

### Python Backend

| Library | Version | Link | Purpose |
|---|---|---|---|
| FastAPI | 0.128+ | https://github.com/fastapi/fastapi | Web framework, async HTTP, OpenAPI docs |
| LangGraph | 1.0+ | https://github.com/langchain-ai/langgraph | Agent orchestration, stateful graphs, checkpointing |
| LangChain | 1.0+ | https://github.com/langchain-ai/langchain | LLM abstraction, multi-provider, tool calling |
| SQLAlchemy | 2.0+ | https://github.com/sqlalchemy/sqlalchemy | ORM, async support, PostgreSQL-native |
| asyncpg | Latest | https://github.com/MagicStack/asyncpg | Async PostgreSQL driver |
| aiokafka | Latest | https://github.com/aio-libs/aiokafka | Async Kafka consumer/producer |
| APScheduler | 3.11+ | https://github.com/agronholm/apscheduler | Background scheduling, cron-style |
| PyDriller | 2.6+ | https://github.com/ishepard/pydriller | Git mining, bus factor, churn analysis |
| GitPython | 3.1+ | https://github.com/gitpython-developers/GitPython | Clone, pull, diff |
| openapi-spec-validator | Latest | https://github.com/python-openapi/openapi-spec-validator | OpenAPI spec validation |
| PyYAML | Latest | https://github.com/yaml/pyyaml | YAML parsing |
| ruamel.yaml | Latest | https://sourceforge.net/projects/ruamel-yaml/ | YAML parsing (round-trip) |
| PyJWT | Latest | https://github.com/jpadilla/pyjwt | JWT token validation |
| authlib | Latest | https://github.com/authlib/authlib | OIDC discovery, OAuth |
| PyGithub | 2.x | https://github.com/PyGithub/PyGithub | GitHub API |
| python-gitlab | 4.x | https://github.com/python-gitlab/python-gitlab | GitLab API |
| atlassian-python-api | 3.x | https://github.com/atlassian-api/atlassian-python-api | Bitbucket API |
| fsspec | 2026.2+ | https://github.com/fsspec/filesystem_spec | Unified S3/GCS/Azure/local storage |
| httpx | Latest | https://github.com/encode/httpx | Async HTTP client |
| pydantic | 2.x | https://github.com/pydantic/pydantic | Schema validation, API models |

### Java Extractor

| Library | Version | Link | Purpose |
|---|---|---|---|
| Spoon | 11.x | https://github.com/INRIA/spoon | Java AST analysis, JDT-based, NOCLASSPATH mode |
| PMD | 7.x | https://github.com/pmd/pmd | Static analysis, 400+ rules |
| Maven Resolver | Latest | https://github.com/apache/maven-resolver | Download JARs without compiling |
| Gradle Tooling API | Latest | https://docs.gradle.org/current/userguide/third_party_integration.html | Resolve Gradle dependencies |
| protobuf-java | Latest | https://github.com/protocolbuffers/protobuf | Parse .proto files |
| swagger-parser | 2.x | https://github.com/swagger-api/swagger-parser | Parse OpenAPI specs |
| JSqlParser | Latest | https://github.com/JSQLParser/JSqlParser | Parse SQL migrations |
| SnakeYAML | 2.x | https://github.com/snakeyaml/snakeyaml | Parse application.yml |
| JGit | 7.x | https://github.com/eclipse-jgit/jgit | Git diff within extractor |
| kafka-clients | Latest | https://github.com/apache/kafka/tree/trunk/clients | Event bus client (default Kafka impl behind EventBus interface) |
| Jackson | 3.x | https://github.com/FasterXML/jackson | JSON serialization |

### Frontend

| Library | Version | Link | Purpose |
|---|---|---|---|
| Next.js | 16 | https://github.com/vercel/next.js | SSR framework, App Router |
| React | 19 | https://github.com/facebook/react | UI framework |
| Vercel AI SDK | 5.x | https://github.com/vercel/ai | Chat UI, streaming, tool rendering |
| Mermaid | 11.4+ | https://github.com/mermaid-js/mermaid | Client-side diagram rendering |
| react-markdown | Latest | https://github.com/remarkjs/react-markdown | Wiki page rendering |
| Shiki | Latest | https://github.com/shikijs/shiki | Syntax highlighting |
| Tailwind CSS | 4 | https://github.com/tailwindlabs/tailwindcss | Utility-first styling |
| svg-pan-zoom | Latest | https://github.com/bumbu/svg-pan-zoom | Interactive diagram navigation |

### Observability

| Library | Version | Link | Purpose |
|---|---|---|---|
| opentelemetry-sdk (Python) | Latest | https://github.com/open-telemetry/opentelemetry-python | OTel core SDK |
| opentelemetry-javaagent | Latest | https://github.com/open-telemetry/opentelemetry-java-instrumentation | Java auto-instrumentation |
| @opentelemetry/sdk-trace-web | Latest | https://github.com/open-telemetry/opentelemetry-js | Browser-side performance traces |

---

## Infrastructure

### Databases

| Technology | Version | Link | Role |
|---|---|---|---|
| PostgreSQL | 18 | https://www.postgresql.org/ | Primary database (relational, default for all) |
| pgvector | 0.8+ | https://github.com/pgvector/pgvector | Vector search (embeddings) |
| Apache AGE | 1.5+ | https://github.com/apache/age | Graph queries (Cypher) |
| pg_trgm | Built-in | https://www.postgresql.org/docs/current/pgtrgm.html | Trigram code search |
| tsvector | Built-in | https://www.postgresql.org/docs/current/textsearch.html | Full-text wiki search |

### Event Bus

| Technology | Version | Link | Role |
|---|---|---|---|
| Apache Kafka | 4.x (KRaft) | https://github.com/apache/kafka | Default event bus |

### Cache

| Technology | Version | Link | Role |
|---|---|---|---|
| Redis | 8.x | https://github.com/redis/redis | Optional cache (cloud mode) |

### Container & Orchestration

| Technology | Version | Link | Role |
|---|---|---|---|
| Docker | Latest | https://www.docker.com/ | Container runtime |
| Docker Compose | v2 | https://docs.docker.com/compose/ | Local 5-container deployment |
| Kubernetes | Latest | https://kubernetes.io/ | Cloud orchestration |
| Helm | 3 | https://github.com/helm/helm | K8s package manager |

### Auth

| Technology | Version | Link | Role |
|---|---|---|---|
| Keycloak | 26+ | https://github.com/keycloak/keycloak | Optional OIDC/SAML provider |
| LiteLLM | 1.80+ | https://github.com/BerriAI/litellm | Optional LLM proxy |

---

## Swappable Alternatives (via abstraction layers)

### Database Layer

| Capability | Default | Alternatives |
|---|---|---|
| Vector search | pgvector | [Qdrant](https://github.com/qdrant/qdrant), [Weaviate](https://github.com/weaviate/weaviate), [Pinecone](https://www.pinecone.io/), [Milvus](https://github.com/milvus-io/milvus) |
| Graph queries | Apache AGE | [Neo4j](https://github.com/neo4j/neo4j), [Memgraph](https://github.com/memgraph/memgraph) |
| Full-text search | tsvector | [Elasticsearch](https://github.com/elastic/elasticsearch), [Meilisearch](https://github.com/meilisearch/meilisearch) |
| Trigram search | pg_trgm | [Elasticsearch](https://github.com/elastic/elasticsearch), [Zoekt](https://github.com/sourcegraph/zoekt) |
| Relational | PostgreSQL | MySQL, [CockroachDB](https://github.com/cockroachdb/cockroach) |

### Event Bus Layer

| Default | Alternatives |
|---|---|
| Apache Kafka | [RabbitMQ](https://github.com/rabbitmq/rabbitmq-server), [NATS](https://github.com/nats-io/nats-server), Redis Streams, AWS SQS/SNS, Google Pub/Sub |

### Agent Framework Layer

| Default | Alternatives |
|---|---|
| LangGraph + LangChain | [CrewAI](https://github.com/crewAIInc/crewAI), [AutoGen](https://github.com/microsoft/autogen), [LlamaIndex Workflows](https://github.com/run-llama/llama_index) |

---

## Static Analysis Tools (per language)

| Language | Tool | Link |
|---|---|---|
| Java | PMD 7.x | https://github.com/pmd/pmd |
| Python | Ruff | https://github.com/astral-sh/ruff |
| Ruby | RuboCop | https://github.com/rubocop/rubocop |
| TypeScript/JS | Biome | https://github.com/biomejs/biome |

---

## Academic References

### MiSAR — Microservice Architecture Recovery

Model-driven approach for recovering microservice architectures from source code via annotation and configuration matching. We use MiSAR's patterns in our cross-repo source collectors (04, 08).

| Paper | Authors | Venue | Year | Link |
|---|---|---|---|---|
| Towards Micro Service Architecture Recovery: An Empirical Study | Alshuqayran, Ali, Evans | IEEE ICSA 2018 | 2018 | [DOI](https://doi.org/10.1109/ICSA.2018.00014) |
| MiSAR: The MicroService Architecture Recovery Toolset | Ali, Alshuqayran, Fakeeh, Rohman, Solis | ECSA 2023 (Springer LNCS 14590) | 2023 | [Springer](https://link.springer.com/chapter/10.1007/978-3-031-66326-0_20) |
| A Model-Driven Architecture Approach for Recovering Microservice Architectures: Defining and Evaluating MiSAR | Alshuqayran, Ali, Evans | Information and Software Technology, Vol. 186 | 2025 | [DOI](https://doi.org/10.1016/j.infsof.2025.107808) |

- **GitHub:** https://github.com/MicroServiceArchitectureRecovery/misar

### Code2DFD — Dataflow Diagram Extraction

Automatic extraction of security-enriched dataflow diagrams from microservice source code. F1=0.87 as single tool, F1=0.91 in ensemble with other recovery tools. We use Code2DFD's pattern detection approach in our cross-repo resolution (04, 08).

| Paper | Authors | Venue | Year | Link |
|---|---|---|---|---|
| Automatic Extraction of Security-Rich Dataflow Diagrams for Microservice Applications Written in Java | Schneider, Scandariato | Journal of Systems and Software, Vol. 202 | 2023 | [DOI](https://doi.org/10.1016/j.jss.2023.111722) |
| Comparison of Static Analysis Architecture Recovery Tools for Microservice Applications | Schneider, Bakhtin, Li, Soldani, Brogi, Cerny, Scandariato, Taibi | Empirical Software Engineering (Springer) | 2025 | [Springer](https://link.springer.com/article/10.1007/s10664-025-10686-2) / [arXiv](https://arxiv.org/abs/2412.08352) |

The comparison paper (2025) is the source of the F1 scores cited in this spec — 13 architecture recovery tools benchmarked across microservice applications.

- **GitHub:** https://github.com/tuhh-softsec/code2DFD

### MS Application Inspector

Cross-platform source code analysis tool from Microsoft using rule-based pattern matching. Identifies what code does (crypto usage, auth patterns, cloud services, HTTP calls) rather than finding bugs. 600+ rule patterns across many languages. We implement the pattern-matching approach for cross-language HTTP client detection in our source collectors — not the tool itself (04, 08).

- **GitHub:** https://github.com/microsoft/ApplicationInspector

---

## Standards & Protocols

| Standard | Link | Role in CodeGraph |
|---|---|---|
| OpenID Connect (OIDC) | https://openid.net/connect/ | Auth provider abstraction |
| SAML 2.0 | https://docs.oasis-open.org/security/saml/v2.0/ | Enterprise auth (bridged via Keycloak) |
| JWT (RFC 7519) | https://datatracker.ietf.org/doc/html/rfc7519 | Token format for built-in auth |
| OpenTelemetry | https://opentelemetry.io/ | Single observability stack |
| JSON Schema | https://json-schema.org/ | Event contract validation |
| OpenAPI 3.x | https://www.openapis.org/ | API specification format |
| gRPC / Protocol Buffers | https://grpc.io/ | Service-to-service RPC |
| MCP (Model Context Protocol) | https://modelcontextprotocol.io/ | AI IDE integration |
| Mermaid | https://mermaid.js.org/ | Diagram markup language |

---

## OTel Export Targets

| Platform | Link | Type |
|---|---|---|
| Jaeger | https://www.jaegertracing.io/ | Trace visualization (local) |
| Grafana Cloud | https://grafana.com/products/cloud/ | Traces, metrics, logs |
| Datadog | https://www.datadoghq.com/ | Full observability |
| Honeycomb | https://www.honeycomb.io/ | Distributed tracing |
| AWS X-Ray / CloudWatch | https://docs.aws.amazon.com/xray/ | AWS-native distributed tracing |
| Google Cloud Trace | https://cloud.google.com/trace | GCP-native tracing |

---

**[Index](00-index.md)**
