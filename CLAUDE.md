# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

CodeGraph is an open-source code intelligence platform. It auto-generates wikis, entity graphs, API catalogs, and cross-repo diagrams for enterprise codebases. Fully open-source, self-hostable via `docker-compose up`.

**Status:** Pre-implementation. The platform design spec is complete and under review at `docs/specs/platform-design.md`.

## Architecture

**Five-container local deployment:**
- **PostgreSQL** — pgvector + Apache AGE + pg_trgm + tsvector. Single DB for everything.
- **Kafka** — KRaft-only (4.x), event bus. JSON Schema contracts in `contracts/`.
- **Python Backend** — FastAPI + LangGraph 1.x + LangChain 1.x. All API, agents, consumers, background jobs.
- **Java Extractor** — Spoon 11.x (INRIA) for compiler-level Java analysis. The only JVM component.
- **Frontend** — Next.js 16 + Vercel AI SDK 5.x + Tailwind 4.

**Pattern:** CQRS + Event-Driven + Hybrid Pipeline/Agentic
- Write path: async, Kafka-driven (repo push → extraction → wiki generation)
- Read path: sync, direct PostgreSQL queries (chat, search, wiki viewing)

**Key design decisions:**
- Deterministic extraction (Spoon AST), LLM only for gap-fill and generation
- Entity graph is the authority source (not LLM inference)
- Ephemeral chat sessions (no cross-session memory)
- All LLM work in Python (LangGraph for complex agents, LangChain for simple ones)
- Admin-configured LLM providers, users never see config

## Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Backend | Python, FastAPI, LangGraph, LangChain | Python 3.13+, FastAPI 0.128+, LangGraph 1.0+, LangChain 1.0+ |
| Frontend | Next.js, React, AI SDK, Tailwind | Next.js 16, React 19, AI SDK 5.x, Tailwind 4 |
| Java Extractor | Java, Spoon, PMD | Java 21, Spoon 11.x, PMD 7.x, Jackson 3.x |
| Database | PostgreSQL | PostgreSQL 18, pgvector 0.8+, AGE 1.5+ |
| Event Bus | Apache Kafka (KRaft) | Kafka 4.x |
| Observability | OpenTelemetry | Single OTel stack for traces, metrics, logs |

## Project Structure (Planned)

```
codegraph/
├── docs/specs/          # Platform design specification
├── contracts/           # Kafka event JSON Schemas
├── backend/             # Python — FastAPI + LangGraph + LangChain
├── extractor/           # Java — Spoon + framework parsers
├── frontend/            # Next.js 16 + AI SDK 5.x
├── docker-compose.yml   # 5-container local deployment
├── docker-compose.otel.yml  # Optional OTel Collector overlay
└── helm/                # Kubernetes Helm chart
```

## Key Spec References

The platform design spec (`docs/specs/platform-design.md`) is the source of truth. Key sections:
- Section 2: Architecture + technology decisions
- Section 5: Entity graph + Java extraction (Spoon, tiered approach)
- Section 6: Wiki generation + 20 diagram types + cross-repo resolution
- Section 7: Chat + Deep Research (LangGraph agent)
- Section 11: Full tech stack with library inventory
- Section 12: Complete data model (all tables)
- Section 13: Kafka event contracts (7 events with producer/consumer map)

## Commands

No runnable code yet — project is in spec phase. When implementation begins:

```bash
# Full stack
docker-compose up

# Backend dev
cd backend && poetry install && python -m codegraph.main

# Frontend dev
cd frontend && yarn install && yarn dev

# Java extractor
cd extractor && ./gradlew build
```

## Guidelines

- **Battle-tested libraries over custom code** — always. Check the library inventory in Section 11 before adding dependencies.
- **Spoon for Java analysis** — not Eclipse JDT directly, not JavaParser, not tree-sitter.
- **LangGraph for complex agents** (chat, wiki gen, deep research, cross-repo, PR context, anomaly). **LangChain for simple agents** (enrichment, API catalog).
- **PostgreSQL for everything** — pgvector, AGE, tsvector, pg_trgm. Each behind an abstraction layer.
- **OTel for all observability** — no Prometheus client libs, no custom log shippers.
- **No enterprise edition gate** — everything ships open-source.
