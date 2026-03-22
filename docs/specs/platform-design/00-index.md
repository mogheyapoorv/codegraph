# CodeGraph Platform — Design Specification

**Date:** 2026-03-22
**Status:** Draft
**Scope:** CodeGraph is a new open-source code intelligence platform. We auto-generate wikis, entity graphs, API catalogs, and cross-repo diagrams for enterprise codebases. The project draws inspiration from deepwiki-open as a starting point, but this is a different codebase with a different direction.

---

> This spec has been split into focused documents for easier navigation and review. Each file is self-contained but cross-references others where needed. The original section numbering is preserved.

## Table of Contents

| File | Sections | Description |
|---|---|---|
| [01-goals-and-architecture.md](01-goals-and-architecture.md) | 1--2 | Goals, constraints, architecture, abstractions, deployment |
| [02-auth-and-repo-management.md](02-auth-and-repo-management.md) | 3--4 | Authentication, admin bootstrap, repo management, repo groups |
| [03-entity-graph-and-extraction.md](03-entity-graph-and-extraction.md) | 5 | Entity graph, Java extraction, static analysis, plugin model |
| [04-wiki-generation-and-diagrams.md](04-wiki-generation-and-diagrams.md) | 6 | Wiki generator, 20 diagram types, cross-repo resolution |
| [05-chat-and-research.md](05-chat-and-research.md) | 7 | Chat modes, agent tools, adaptive search, deep research |
| [06-integrations.md](06-integrations.md) | 8 | PR context, MCP server, Slack/Teams, CLI, widget, REST API |
| [07-operations.md](07-operations.md) | 9, 14--16 | Analytics, anomaly detection, rate limiting, observability |
| [08-tech-stack-and-extensibility.md](08-tech-stack-and-extensibility.md) | 11 | Builds vs adopts, extensibility architecture, library inventory |
| [09-data-model.md](09-data-model.md) | 12 | All database tables |
| [10-event-contracts.md](10-event-contracts.md) | 13 | Event schemas and producer/consumer map |
| [11-scaling-and-phasing.md](11-scaling-and-phasing.md) | 10, 17 | Build phases, scaling limits, migration paths |

## Cross-Reference Map

| Concept | Files |
|---|---|
| Entity confidence scoring | 03, 09 |
| Event bus abstraction | 01, 10 |
| Database abstraction (pgvector, AGE, tsvector) | 01, 09 |
| LLM provider configuration | 01, 02, 08 |
| Spoon extraction pipeline | 03, 10 |
| Wiki generation agent (LangGraph) | 04, 05, 10 |
| Cross-repo resolution | 03, 04, 06 |
| Diagram types | 04, 08 |
| Chat tool registry | 05, 08 |
| Rate limiting and quotas | 02, 07 |
| OTel observability | 07, 08 |
| Build phases and priorities | 11, 01 |
| PR context integration | 06, 10 |
| Anomaly detection agent | 07, 05 |
