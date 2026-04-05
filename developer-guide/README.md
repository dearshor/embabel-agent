# Embabel Agent Framework — Developer Guide

This guide is intended for developers who want to understand the internals of the Embabel Agent Framework, contribute to it, or extend it for their own needs.

## Contents

| Document | Description |
|---|---|
| [Architecture Overview](architecture.md) | Core concepts, execution model, and how the pieces fit together |
| [embabel-agent-api](modules/embabel-agent-api.md) | Core framework: annotations, DSL, planning, blackboard, events |
| [embabel-agent-common](modules/embabel-agent-common.md) | Shared AI model abstractions and utilities |
| [embabel-agent-domain](modules/embabel-agent-domain.md) | Pre-built reusable domain types |
| [embabel-agent-autoconfigure](modules/embabel-agent-autoconfigure.md) | Spring Boot autoconfiguration for platform and all providers |
| [embabel-agent-starters](modules/embabel-agent-starters.md) | Opinionated Spring Boot starters for quick onboarding |
| [embabel-agent-test-support](modules/embabel-agent-test-support.md) | Unit and integration test helpers |
| [embabel-agent-shell](modules/embabel-agent-shell.md) | Interactive Spring Shell CLI |
| [embabel-agent-openai](modules/embabel-agent-openai.md) | OpenAI-compatible model utilities |
| [embabel-agent-rag](modules/embabel-agent-rag.md) | Retrieval-Augmented Generation pipeline |
| [embabel-agent-code](modules/embabel-agent-code.md) | Coding-agent tools (Git, Maven, Bash, symbol search) |
| [embabel-agent-mcp](modules/embabel-agent-mcpserver.md) | MCP server (SSE) integration + MCP security (`@SecureAgentTool`) |
| [embabel-agent-onnx](modules/embabel-agent-onnx.md) | ONNX Runtime local embedding service |
| [embabel-agent-a2a](modules/embabel-agent-a2a.md) | Google Agent-to-Agent (A2A) protocol |
| [embabel-agent-observability](modules/embabel-agent-observability.md) | OpenTelemetry tracing and Micrometer metrics |

## Quick orientation

```
embabel-agent/
├── embabel-agent-api/           # ← Start here: the programming model
├── embabel-agent-common/        # Shared AI model abstractions
├── embabel-agent-domain/        # Reusable domain objects
├── embabel-agent-autoconfigure/ # Spring Boot autoconfiguration
│   ├── embabel-agent-platform-autoconfigure/
│   ├── embabel-agent-shell-autoconfigure/
│   ├── embabel-agent-mcpserver-autoconfigure/
│   ├── embabel-agent-a2a-autoconfigure/
│   ├── embabel-agent-observability-autoconfigure/
│   └── models/                  # Per-provider autoconfiguration
├── embabel-agent-starters/      # One starter per use-case
├── embabel-agent-test-support/  # FakeOperationContext etc.
├── embabel-agent-shell/         # Spring Shell CLI
├── embabel-agent-openai/        # OpenAI-specific code
├── embabel-agent-rag/           # RAG subsystem
│   ├── embabel-agent-rag-core/
│   ├── embabel-agent-rag-pipeline/
│   ├── embabel-agent-rag-lucene/
│   ├── embabel-agent-rag-tika/
│   └── embabel-agent-rag-neo/
├── embabel-agent-code/          # Coding agent tools
├── embabel-agent-mcp/           # MCP server + security
│   ├── embabel-agent-mcpserver/ #   SSE server implementation
│   └── embabel-agent-mcp-security/ # @SecureAgentTool enforcement
├── embabel-agent-onnx/          # ONNX Runtime local embeddings
├── embabel-agent-a2a/           # A2A protocol
└── embabel-agent-observability/ # Tracing & metrics
```

## Build

```bash
# Build and test everything
mvn test

# Build a specific module
mvn test -pl embabel-agent-api

# Build including docs
mvn test -Pembabel-agent-docs
```

## Key external dependencies

- **Spring Boot 3.x / Spring AI** — the foundational framework
- **Kotlin 2.x** — primary language (Java is fully supported)
- **Jackson** — JSON serialization and JSON Schema generation
- **Micrometer / OpenTelemetry** — metrics and tracing
- **Apache Lucene** — optional embedded vector store for RAG
- **Apache Tika** — document parsing for RAG ingestion
