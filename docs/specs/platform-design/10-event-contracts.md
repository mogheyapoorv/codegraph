# CodeGraph Platform — Event Contracts

## 13. Event Contracts

All events use JSON. Schemas live in the `contracts/` directory. These contracts are transport-agnostic — the same JSON Schemas apply whether the event bus is Kafka (default), RabbitMQ, NATS, or any other implementation behind the EventBus interface (Section 2).

**Event producer/consumer map:**

| Event | Producer | Consumer(s) |
|---|---|---|
| `repo.pushed` | Webhook handler / Poller / Manual trigger | Repo Indexer |
| `repo.indexed` | Repo Indexer | Java Extractor (Phase 1) / Language Extractors (Phase 2+) |
| `entities.extracted` | Java Extractor (Phase 1) / Language Extractors (Phase 2+) | Enrichment agent, Wiki Generator, Embedding Worker, API Catalog Builder |
| `wiki.generated` | Wiki Generator | Embedding Worker |
| `diagram.stale` | Wiki Generator (changed entities) / Admin API (membership changes) | Background consistency job |
| `api.discovered` | API Catalog Builder | (informational — no active consumer, used for analytics + incremental updates) |
| `repo_group.membership_changed` | Admin API (group membership endpoints) | Cross-repo resolution pipeline, Background consistency job |

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
  "trigger": "webhook|poll|manual|bulk_discovery",
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
  "repo_group_id": "uuid",
  "repo_id": "uuid",
  "action": "added|removed",
  "group_repo_ids": ["uuid1", "uuid2", "uuid3"],
  "timestamp": "ISO-8601"
}
```

---

**Prev:** [Data Model](09-data-model.md) | **Next:** [Scaling & Phasing](11-scaling-and-phasing.md) | [Index](00-index.md)
