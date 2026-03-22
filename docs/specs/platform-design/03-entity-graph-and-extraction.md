# CodeGraph Platform — Entity Graph & Extraction

## 5. Entity Graph & Java Extraction

### Entity Graph Data Model

**CodeEntity (nodes):**
- id, repo_id, entity_type, name, fully_qualified_name
- file_path, line_start, line_end, language, signature
- entity_confidence (FLOAT — 1.0 = all analyzers agree, 0.9 = two agree, 0.8 = single analyzer, 0.7 = LLM gap-fill)
- discovered_by (ast_parse, framework_parser, llm_gap_fill)

**Entity types:**
- Code-level: function, class, method, interface, enum, module, file
- API surface: rest_endpoint, grpc_service, grpc_method, graphql_query
- Data layer: db_table, db_column, db_migration, message_topic, cache_key
- Infrastructure: docker_service, env_variable, config_key, cron_job, feature_flag

**EntityRelation (edges):**
- source_entity_id, target_entity_id, relation_type
- is_cross_repo, relation_confidence (FLOAT — see Section 6 multi-source confidence scoring), discovered_by, evidence_file, evidence_line

**Relation types:**
- Code: imports, calls, extends, implements, instantiates, depends_on
- API: exposes_api, consumes_api, shares_proto
- Data: reads_table, writes_table, publishes_to, subscribes_to, uses_config
- Implicit: co_changes_with (git-derived)

**Source storage:**
- `source_files` — full file content stored via relational interface (default: PostgreSQL, searchable via trigram search interface)
- `code_snippets` — per-entity source code linked to entities

**Graph queries via graph query interface (default: Apache AGE with Cypher):**
- Forward traversal: "What does PaymentService call?"
- Reverse traversal: "What depends on PaymentGateway?" (blast radius)
- Cross-repo edges: "Who consumes payment-service APIs?"
- Path finding: "What's the chain from user signup to first email?"

### Java Extraction — Tiered Approach

**Tier 1: Deterministic Multi-Analyzer (always runs, $0, 70-95% coverage)**

Three analyzers run in parallel, and a consensus engine merges the results:

**Analyzer 1 — Spoon 11.x (AST + Type Resolution):**
- Built on Eclipse JDT — same compiler-grade type resolution (95%+), but with a much cleaner metamodel API (`CtClass`, `CtMethod`, `CtInvocation` instead of raw JDT visitors)
- Parses all Java files in batch with full type resolution
- Classpath from downloaded Maven/Gradle JARs (dependency download only, no compilation)
- **NOCLASSPATH mode** — when JARs are unavailable (private Maven repos, internal libraries), Spoon degrades gracefully with partial results and clearly marked unresolved references, instead of just failing. This is critical for "works on any repo."
- Produces: classes, methods, fields, annotations, call graph, type hierarchy, import graph
- Annotation scanning via `CtAnnotation` — classifies every annotation as known-framework or custom, with full access to annotation values and meta-annotations
- Lombok plugin for desugaring (no compilation needed)

**Why Spoon?** We evaluated the alternatives:

| Tool | Type Resolution | No Build Required? | Why Not |
|---|---|---|---|
| **Spoon 11.x (INRIA)** | 95%+ (uses JDT internally) | Yes, with NOCLASSPATH graceful degradation | **Our choice.** Clean API, best handling of missing dependencies. |
| **Eclipse JDT (direct)** | 95%+ (compiler-grade) | Yes (source + JARs) | Same resolution quality but verbose IDE-oriented API. Spoon wraps it with a better developer experience. |
| **JavaParser + Symbol Solver** | 60-80% (known gaps with generics, lambdas, overloads) | Yes | Resolution accuracy too low for reliable call graphs on enterprise code. |
| **IntelliJ PSI** | Best-in-class | Impractical outside IDE | Can't be extracted from the IntelliJ platform. JetBrains themselves say it's not supported. |
| **tree-sitter** | None (syntax only) | Yes | No semantic analysis. Useful for other languages where no better parser exists, but not for Java. |
| **Soot/SootUp** | Full (bytecode only) | No — needs compilation | Best call graph construction, but requires compiled .class files. We can't reliably compile arbitrary repos. |

Spoon gives us JDT's compiler-grade type resolution with a clean metamodel and NOCLASSPATH mode for graceful handling of missing dependencies. It's actively maintained (11.x requires JDK 17+, supports up to Java 25), and the INRIA research team behind it has been at this for 15+ years.

**Analyzer 2 — Framework-Specific Parsers:**
- Spring annotations → beans, endpoints, DI wiring (via Spoon `CtAnnotation` processors)
- JPA annotations → entities, tables, relations (via Spoon)
- Proto files → gRPC services, methods (via **protobuf-java**, parsed directly, not compiled)
- OpenAPI spec (if checked in) → full API catalog (via **swagger-parser 2.x**)
- SQL migrations (Flyway/Liquibase) → actual DB schema (via **JSqlParser**)
- application.yml / application.properties → config keys with values (via **SnakeYAML 2.x**)
- pom.xml → dependency graph (via **Maven Resolver**), build.gradle → (via **Gradle Tooling API**)

**Analyzer 3 — File-Level Analyzers:**
- Dockerfile → docker_service entities
- docker-compose.yml → service topology
- K8s manifests → deployment topology
- .github/workflows → CI/CD pipeline
- CODEOWNERS → ownership
- README, ADRs → documentation references

**Consensus Engine (entity_confidence — how certain we are about this entity's classification):**
- Merges all analyzer outputs, cross-references results
- All three analyzers agree → entity_confidence 1.0
- Two agree → entity_confidence 0.9
- Single source → entity_confidence 0.8
- Cross-reference: does the JPA entity match a SQL migration table? Does the endpoint match the OpenAPI spec?
- **Conflict resolution:** When analyzers disagree (e.g., Spoon says class, framework parser says endpoint), the framework parser wins for framework-specific types (it has richer context from annotations). For other conflicts, the analyzer with higher specificity wins. Unresolvable conflicts → store both with the lower entity_confidence (0.8) and flag for Tier 2 LLM gap-fill.

Note: entity_confidence (from extraction) and relation_confidence (from cross-repo resolution, Section 6) measure different things. entity_confidence = "how certain is this entity's type?" relation_confidence = "how certain is this cross-repo connection?"

**No compilation required.** Spoon resolves types from source + downloaded JARs (via JDT internally). NOCLASSPATH mode handles missing dependencies gracefully. Framework parsers read annotations and config files directly. Works on any repo.

**Tier 2: LLM Gap-Fill (automatic, fills the remaining 5-30%)**

For entities that Tier 1 couldn't classify (custom annotations, internal frameworks):

1. Heuristic classification first (meta-annotation analysis, naming conventions, package patterns) — catches ~80% of unknowns
2. Remaining truly unknown patterns → LLM reads the annotation/class definition source and reasons about its purpose
3. Tagged as `discovered_by: llm_gap_fill` with entity_confidence 0.7
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

We use **PMD 7.x** for Java static analysis — runs directly on source code, no compilation needed. PMD gives us 400+ built-in rules out of the box:
- Error prone: empty catch blocks, null checks, broken equals/hashCode
- Security: hardcoded credentials, insecure crypto
- Design: god classes, cyclomatic complexity, excessive method length
- Best practices: missing @Override, loose coupling

For Phase 2 languages:
- Python: **Ruff** (Rust-based, extremely fast, replaces flake8/pylint)
- Ruby: **RuboCop**
- TypeScript/JS: **Biome** (Rust-based, replaces ESLint for linting + formatting)

Risk flags are stored per entity in the entity graph. No custom rule engine — we use these tools' APIs programmatically.

**Analyzer adapter interface:** Each static analysis tool is wrapped in an adapter that implements a common interface: input (file path + language), output (list of risk flags with severity, rule ID, line number, message). Adding a new linter for a new language means implementing this adapter — the rest of the pipeline (risk flag storage, health score aggregation, wiki rendering) works unchanged.

### API Catalog & curl/grpcurl Generation

Every discovered API endpoint gets runnable request examples:

- REST endpoints → curl command with example payload
- gRPC services → grpcurl command with example payload
- GraphQL → example query

**Example payload generation:** The API Catalog Builder agent (LangChain) generates realistic values from field names and types (amount → 99.99, email → user@example.com, currency → "USD"). It uses actual type definitions from the codebase.

**Request/response schemas** come from the Java type system — the extractor captures full type information including nested objects, enums, and generics.

**Protocol template interface:** Each API protocol (REST, gRPC, GraphQL) has a template generator: input (endpoint entity + type definitions from entity graph), output (example request command + payload). Adding a new protocol (e.g., WebSocket, SSE, SOAP) means implementing this template generator and registering it. The API Catalog Builder agent discovers registered protocol templates and applies them to matching endpoints.

### Incremental Updates

Here's what actually happens on a push (via webhook, polling, or manual trigger):

1. `git diff old_sha..new_sha` → list of added/modified/deleted files
2. **Deleted files:** remove all entities, relations, snippets, source content, embeddings for that file (CASCADE)
3. **Modified files:** delete + re-insert everything for that file (idempotent)
4. **Added files:** insert new entities, snippets, source, embeddings
5. **Build file changed (pom.xml/build.gradle):** diff dependencies — if a framework changed, full re-extraction. Version bump only? No re-extraction.
6. Cross-repo resolution: affected entities flagged for re-resolution by the background consistency job (see Section 6)
7. Wiki pages referencing changed entities → mark stale → wiki generator regenerates
8. Diagrams including changed entities → mark stale → background job regenerates
9. API catalog entries for changed endpoints → regenerate curl/grpcurl

**Typical push (5-10 files):** seconds to update. **Framework upgrade:** 5-30 minutes for full re-extraction.

### Language Extractor Plugin Model

Future extractors (Ruby, Python, TypeScript) follow the same contract:

**Extractor interface — what every language extractor must implement:**
- **Input:** consumes `repo.indexed` from the event bus, receives repo path + changed files + language hints
- **File matching:** declares which file patterns it handles (*.rb, *.py, *.ts, etc.)
- **Analysis:** runs language-native analysis tools (tree-sitter, Pyright, TS Compiler API, etc.)
- **Tiered approach:** deterministic extraction first, LLM gap-fill for unknowns
- **Output:** produces `entities.extracted` matching the JSON Schema in `contracts/`
- Downstream workers process identically regardless of source language

**Registration and discovery:**
- Each extractor registers itself with a manifest: supported languages, file patterns, capabilities (AST resolution, type resolution, framework parsing), and maturity level
- The system discovers extractors at startup via a plugin directory (Python modules) or Docker service labels (containers)
- Admin enables/disables extractors per language from the dashboard
- Multiple extractors can handle the same language at different maturity levels — the admin chooses which one is active

**Deployment options:**
- **Python module** (preferred for most languages) — loaded in the main backend process, zero deployment overhead
- **Docker container** (for JVM or other runtimes) — separate container, same event bus contract
- **External service** (for contributed extractors that can't run in-process) — consumes/produces via event bus, fully decoupled

**Extractor maturity levels:**

| Level | Name | What It Provides |
|---|---|---|
| Level 1 | Basic | Syntax-level extraction (classes, functions, imports via tree-sitter). No type resolution. |
| Level 2 | Intermediate | Type resolution + framework detection (e.g., Pyright for Python). Partial call graphs. |
| Level 3 | Complete | Full type resolution, framework-specific parsers, call graphs, annotation scanning, NOCLASSPATH-equivalent graceful degradation. |

Java extractor is Level 3 (complete ecosystem). Future languages start at Level 1 (basic) and grow via community contribution.

Only Java needs a separate JVM container. All other language extractors are Python modules in the main backend by default, but can be deployed as separate containers if needed.

---

**Prev:** [Auth & Repo Management](02-auth-and-repo-management.md) | **Next:** [Wiki Generation & Diagrams](04-wiki-generation-and-diagrams.md) | [Index](00-index.md)
