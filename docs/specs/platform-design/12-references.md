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

| Reference | Relevance | Context |
|---|---|---|
| **MiSAR** (Microservice Architecture Recovery) | Cross-repo dependency detection via annotation and config matching | Used in source collector patterns (04, 08) |
| **Code2DFD** | Data flow diagram extraction from code, F1=0.87 single tool, F1=0.91 ensemble | Pattern detection for cross-repo resolution (04, 08) |
| **MS Application Inspector** | Cross-language static analysis rule patterns | We implement the pattern rules, not the tool itself (04, 08) |

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
