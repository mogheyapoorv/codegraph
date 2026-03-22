# CodeGraph Platform — Data Model

## 12. Data Model

### Core Tables

**users:**
- id (UUID PK), email, name, avatar_url, password_hash (nullable), role (superadmin/admin/member), auth_provider (local/oidc), oidc_issuer, oidc_subject, daily_cost_cap_usd (DECIMAL nullable — admin-set per-user cost cap, null = use global default), is_active, created_at, last_login

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
- id (UUID PK), repo_id (FK), entity_type, name, fully_qualified_name, file_path, line_start, line_end, language, signature, entity_confidence (FLOAT — how certain is this entity's type), discovered_by, risk_flags (JSONB — PMD/Ruff findings per entity), last_commit_sha, created_at, updated_at
- Indexes: (repo_id, entity_type), (fully_qualified_name)

**entity_relations:**
- id (UUID PK), source_entity_id (FK CASCADE), target_entity_id (FK CASCADE), relation_type, is_cross_repo, relation_confidence (FLOAT — 1.0 for AST-derived intra-repo relations, multi-source scoring for cross-repo — see Section 6), discovered_by, evidence_file, evidence_line, created_at
- Indexes: (source_entity_id, relation_type), (target_entity_id, relation_type), (is_cross_repo) WHERE is_cross_repo = TRUE

**group_cross_repo_relations:**
- id (UUID PK), repo_group_id (FK CASCADE), source_entity_id (FK CASCADE), target_entity_id (FK CASCADE), relation_type, relation_confidence (FLOAT), discovered_at

### Source Storage Tables

**source_files:**
- id (UUID PK), repo_id (FK), file_path, content (TEXT), language, size_bytes, commit_sha, updated_at
- Index: GIN (content gin_trgm_ops) for trigram search
- Index: (repo_id, file_path)

**code_snippets:**
- id (UUID PK), entity_id (FK CASCADE), repo_id (FK), file_path, line_start, line_end, content (TEXT), language, updated_at

### Embedding Tables

**knowledge_chunks:**
- id (UUID PK), repo_id (FK), source_id (VARCHAR — file_path for code chunks, wiki_page_id for wiki chunks), content (TEXT), embedding (vector), chunk_index, source_type (code/wiki), last_modified, created_at
- Index: IVFFlat (embedding vector_cosine_ops)
- Index: (repo_id)

### Wiki Tables

**wiki_pages:**
- id (UUID PK), repo_id (FK), title, slug, content (TEXT, local mode), content_path (cloud mode), search_tsv (tsvector — maintained in both modes for full-text search), parent_page_id (FK self), page_order, generation_status (generating/generated/stale/failed), source_entities (JSONB), source_commit_sha, version, created_at, updated_at
- Index: GIN (search_tsv) for wiki full-text search

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
- id (UUID PK), repo_id (FK), priority (0=webhook/1=manual/2=bulk), status (queued/running/completed/failed/paused), trigger_type (webhook/poll/manual/bulk_discovery), batch_id, stage (cloning/extracting/enriching/embedding/generating_wiki), files_total, files_processed, commit_sha, error_message, retry_count, llm_tokens_used, llm_cost_usd, queued_at, started_at, completed_at

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
- id (UUID PK), user_id (FK), title, scope_type, scope_id, query, status (planning/awaiting_approval/executing/completed/failed), plan (JSONB), findings (JSONB), report_content (TEXT), diagrams (JSONB), sources_used (JSONB), total_tokens, total_cost_usd, saved_as_wiki_page_id (FK nullable), auto_saved (BOOLEAN DEFAULT TRUE), created_at

### Anomaly Detection Tables

**anomalies:**
- id (UUID PK), repo_id (FK nullable), repo_group_id (FK nullable), anomaly_type (VARCHAR — open string, not a database enum. Built-in types: documentation_drift, dead_endpoint, shadow_dependency, convention_violation, orphaned_code, stale_wiki, missing_tests, config_drift. Community detectors register new types without schema migration.), severity (high/medium/low), affected_entity_ids (JSONB), description (TEXT), suggested_action (TEXT), scan_job_id (FK background_job_runs), resolved (BOOLEAN DEFAULT FALSE), resolved_by (FK nullable), created_at, resolved_at

### System Configuration Tables

**system_settings:**
- key (VARCHAR PK), value (JSONB), description, updated_by (FK), updated_at
- Used for: cost caps (daily/monthly), concurrency limits, polling defaults, rate limit defaults, blast radius thresholds

### Agent Checkpoint Tables

**langgraph_checkpoints:**
- Managed by LangGraph SDK. Used for Deep Research crash recovery (checkpoint after each step → resume on failure).
- Schema is owned by LangGraph, not CodeGraph. We run LangGraph's migration tool to create these tables.
- If the agent framework is swapped (Section 2 abstraction), the checkpoint storage mechanism changes accordingly.

### Extension Registry Tables

**extension_configs:**
- id (UUID PK), extension_type (extractor/diagram_renderer/analyzer/anomaly_detector/chat_tool/source_collector/health_metric/messaging_adapter/git_platform_adapter), extension_name (VARCHAR), config (JSONB — manifest, settings, capabilities), is_active (BOOLEAN DEFAULT TRUE), created_by (FK), created_at, updated_at
- UNIQUE(extension_type, extension_name)
- Stores admin-managed extension state: which extractors are enabled, which diagram types are active, etc. Populated at startup from plugin discovery, modifiable from the Admin Dashboard.

### Additional Analytics Tables

**search_analytics:**
- id (UUID PK), search_type (code/vector/wiki/entity_graph/api_catalog), scope_type, scope_id, query_text, results_count, latency_ms, user_id, source (web/api/cli/mcp), created_at

**api_catalog_view_events:**
- id (UUID PK), api_endpoint_id (FK), repo_id, user_id, action (view/copy_curl/copy_grpcurl), created_at

### Shared Repo Storage

**repo_clones:**
- id (UUID PK), repo_id (FK), clone_path (VARCHAR — local filesystem path), commit_sha (VARCHAR), status (cloning/ready/stale/deleted), created_at, updated_at
- Shared volume between Python backend and Java Extractor. Python backend clones, Java Extractor reads from the same path.

---

**Prev:** [Tech Stack & Extensibility](08-tech-stack-and-extensibility.md) | **Next:** [Event Contracts](10-event-contracts.md) | [Index](00-index.md)
