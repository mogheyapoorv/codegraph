# CodeGraph Platform — Operations

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

| Metric | Source | Tool |
|---|---|---|
| Documentation coverage | % of entities with wiki sections | Entity graph query |
| Wiki staleness | % of pages marked stale | wiki_pages table |
| Orphaned code | Entities with zero incoming edges | Graph query interface |
| Test coverage gaps | Entities with no corresponding test file | Test-to-source file mapping |
| Complexity hotspots | High complexity + high blast radius | risk_flags from code_entities (pre-computed by PMD/Ruff/Biome during extraction) |
| Cross-repo coupling | Repos with too many cross-repo edges | Entity graph query |
| Bus factor | Modules with single-contributor knowledge | **PyDriller** git mining |
| Change frequency | Most changed files in last 90 days | **PyDriller** commit analysis |

**Git metric population:** PyDriller runs as a background job after each indexing cycle completes. It reads the git history for the repo and populates the `git_metadata` table with per-file stats (commit count, contributors, last author, churn). This runs independently of extraction — no LLM, no entity graph dependency. The data powers the health score metrics below and the `git_metadata_query` chat tool.

**Health metric interface:** Each metric is a registered function: input (repo_id or group_id), output (score + details + affected entities). Adding a new health metric (e.g., dependency freshness, API documentation completeness, circular dependency count) means implementing this interface and registering it. The dashboard and API automatically surface all registered metrics.

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

**Anomaly detector interface:** Each anomaly type is a registered detector function: input (entity graph + wiki state for a repo or group), output (list of anomalies with type, severity, affected entities, description, suggested action). Adding a new anomaly detector (e.g., circular dependency detection, API versioning inconsistency, license compliance) means implementing this interface and registering it. The scheduling, storage, and dashboard rendering work unchanged.

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

### Rate Limit Counter Storage

- **Local mode (no Redis):** In-memory sliding window counters in the Python backend. Resets on restart — acceptable for single-instance deployments.
- **Cloud mode (Redis available):** Redis sorted sets for distributed rate limiting across multiple backend instances. Standard pattern, battle-tested.

Cost caps and rate limit configuration stored in the `system_settings` table (Section 12).

---

## 16. Observability

OpenTelemetry is the single observability stack. Traces, metrics, and logs — all through OTel. No Prometheus client libraries, no custom log shippers, no separate tracing SDKs. One standard, one set of instrumentation.

### OpenTelemetry Setup

**Python backend:** `opentelemetry-sdk` + `opentelemetry-api` with auto-instrumentation:
- `opentelemetry-instrumentation-fastapi` — traces every HTTP request
- `opentelemetry-instrumentation-asyncpg` — traces every DB query
- `opentelemetry-instrumentation-aiokafka` — traces Kafka produce/consume (matches our aiokafka client)
- `opentelemetry-instrumentation-httpx` — traces outbound HTTP (webhooks, MCP, LLM calls)
- LangChain/LangGraph: `opentelemetry-instrumentation-langchain` or manual spans per agent step

**Java Extractor:** `opentelemetry-java-agent` (javaagent auto-instrumentation):
- Auto-instruments Kafka client, JDBC, HTTP clients with zero code changes
- Attach via `JAVA_TOOL_OPTIONS=-javaagent:opentelemetry-javaagent.jar`

**Frontend:** `@opentelemetry/sdk-trace-web` for browser-side performance traces (page loads, chat latency).

### Traces

Every operation gets a trace with span hierarchy:

```
trace: wiki-generation
  span: repo.pushed (Kafka consume)
    span: git-pull
    span: file-diff
  span: repo.indexed (Kafka produce)
  span: entities.extracted (Java extractor)
    span: spoon-parse (per file batch)
    span: framework-parser
    span: consensus-engine
  span: embedding (chunk + embed)
  span: wiki-generate (LangGraph agent)
    span: plan-structure
    span: generate-page (per page)
      span: llm-call (model, tokens, latency)
```

Kafka event chains are correlated via `traceparent` header propagated through Kafka message headers. One trace follows a repo push all the way through extraction → embedding → wiki generation → diagram update.

### Metrics

OTel metrics SDK exports to any OTel-compatible backend (Prometheus via OTel Collector, Grafana Cloud, Datadog, etc.):

- `codegraph.indexing.jobs.total` (counter, by status: completed/failed)
- `codegraph.indexing.duration_seconds` (histogram, by repo)
- `codegraph.kafka.consumer.lag` (gauge, by topic)
- `codegraph.llm.call.duration_seconds` (histogram, by provider, model)
- `codegraph.llm.tokens.total` (counter, by provider, model, direction: input/output)
- `codegraph.llm.cost.usd` (counter, by provider, model, operation)
- `codegraph.chat.response.duration_seconds` (histogram, by mode: quick/conversational/research)
- `codegraph.entity_graph.size` (gauge — total entities, total relations)
- `codegraph.active_users` (gauge)

### Logs

Structured JSON logs via OTel Logs SDK. Correlated with traces via trace_id/span_id:
- Every log line carries: timestamp, service, trace_id, span_id, level, message, attributes
- Log levels: DEBUG, INFO, WARNING, ERROR
- Key attributes: repo_id, operation, duration_ms, error

### Health Checks

- `GET /health` — backend liveness
- `GET /health/ready` — readiness (PostgreSQL connected, Kafka connected)
- Kafka consumer lag monitoring (via OTel metrics)
- Background job last-run tracking

### OTel Collector

We ship an OTel Collector config as an optional docker-compose override (`docker-compose.otel.yml`). Not part of the base 5-container deployment — you only add it if you want to ship telemetry to an external backend. Receives traces + metrics + logs from all components, exports to:
- **Local:** stdout/file (default), or Jaeger for trace visualization
- **Cloud:** any OTel-compatible backend — Grafana Cloud, Datadog, Honeycomb, AWS X-Ray, Google Cloud Trace

Admin configures the export target via `CODEGRAPH_OTEL_ENDPOINT` env var. If not set, observability data goes to local files only.

---

**Prev:** [Integrations](06-integrations.md) | **Next:** [Tech Stack & Extensibility](08-tech-stack-and-extensibility.md) | [Index](00-index.md)
