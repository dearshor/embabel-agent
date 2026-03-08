# AGENTS.md

This file provides guidance to AI coding agents (e.g. OpenAI Codex, GitHub Copilot Workspace, and others) when working with this repository.

## Important Rules

- ALL instructions in this document MUST be followed unless explicitly stated otherwise.
- DO NOT edit more code than necessary to complete the task.
- ALWAYS work test-first: write failing tests before implementing a feature or fix.
- NEVER execute `git commit` or `git push`. Use git only for history, diffing, or staging.
- Build the most relevant module (closest `pom.xml`) with Maven before presenting finished changes.
- Run the individual tests you have changed or that are most relevant.
- If any change impacts usage of the project, update relevant documentation files.

## Project Overview

Embabel Agent Framework is a JVM-based framework for authoring agentic flows that mix LLM-prompted interactions with code and domain models. It is written primarily in **Kotlin** with first-class **Java** support, built on **Spring Boot** and **Spring AI**.

Key concepts:
- **Actions**: Steps an agent takes (annotated with `@Action`)
- **Goals**: What an agent is trying to achieve (annotated with `@Goal` / `@AchievesGoal`)
- **Conditions**: Boolean conditions assessed before/after each action
- **Domain model**: Strongly-typed objects underpinning the flow
- **Plan**: A dynamically-formulated sequence of actions (uses GOAP or Utility AI)

Execution modes: *Focused*, *Closed*, and *Open*.

## Repository Structure

```
embabel-agent/                   # Root — parent Maven POM
├── embabel-agent-api/           # Core framework: annotations, DSL, events, models
├── embabel-agent-a2a/           # Google A2A protocol integration
├── embabel-agent-autoconfigure/ # Spring Boot autoconfiguration
├── embabel-agent-code/          # Coding-agent support
├── embabel-agent-common/        # Shared utilities
├── embabel-agent-dependencies/  # BOM / dependency management POM
├── embabel-agent-domain/        # Core domain abstractions
├── embabel-agent-mcpserver/     # MCP server (SSE) integration
├── embabel-agent-observability/ # OpenTelemetry tracing & metrics
├── embabel-agent-openai/        # OpenAI-specific configuration
├── embabel-agent-rag/           # RAG pipeline and Tika document processing
├── embabel-agent-shell/         # Interactive Spring Shell CLI
├── embabel-agent-starters/      # Spring Boot starters for easy onboarding
├── embabel-agent-test-support/  # Testing utilities (FakeOperationContext, etc.)
└── embabel-agent-docs/          # Asciidoctor documentation (build with -Pembabel-agent-docs)
```

## Build & Test

```bash
# Build and test the full project
mvn test

# Build a specific module
mvn test -pl embabel-agent-api

# Build including docs
mvn test -Pembabel-agent-docs
```

Tests run offline — no internet connection or external services are required.

## Coding Style

Follow `embabel-agent-api/.embabel/coding-style.md`. Key points:

- **Language**: Write new code in **Kotlin**. Files that are already Java stay in Java.
- **Immutability**: Favour immutable data classes and `val`.
- **No builders**: Use withers (as in `PromptRunner`), not builder patterns.
- **No licence headers**: The build adds them automatically.
- **Comments**: Only comment non-obvious things. Self-documenting names are preferred.
- **Logging**: Use SLF4J placeholders — `logger.info("{} {}", a, b)` — not string interpolation at the call site.
- **Spring idiom**: Use `@Configuration`, `@ConfigurationProperties`, constructor injection.
- **Visibility**: Make classes `internal` if possible; use `@ApiStatus.Internal` on public-but-internal classes.
- **Test structure**: Use `@Nested` test classes; name tests with backtick strings (`\`test complicated thing\``).
- **Kotlin/Java interop**: Add `@JvmOverloads`, `@JvmStatic` where needed for idiomatic Java use.
- **Mocking**: Use `mockk` in Kotlin tests; use `Mockito` in Java tests.
- **Blank lines in methods**: Only where they separate clearly distinct logical groups.

## Configuration

| Prefix | Scope | Owner |
|---|---|---|
| `embabel.agent.platform.*` | Internal framework behaviour | Library (rarely changed) |
| `embabel.agent.*` | Business / deployment choices | Developer (expected to be customised) |

Platform defaults are in `agent-platform.properties`. Developer overrides go in `application.yml`.

## Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | Yes | OpenAI API access |
| `ANTHROPIC_API_KEY` | No | Anthropic/Claude (needed for coding agent) |
| `GOOGLE_STUDIO_API_KEY` | No | Gemini, required when using A2A profile |

## Spring Profiles

| Profile | Purpose |
|---|---|
| `docker-desktop` | Use Docker Desktop MCP extension for web tools |
| `a2a` | Enable A2A server |
| `starwars` | Star Wars themed logging (default in examples) |
| `severance` | Severance themed logging |
| `colossus` | Colossus themed logging |
| `montypython` | Monty Python themed logging |
| `hh` | Hitchhiker's Guide themed logging |

## Testing Guidelines

- Every agent is unit-testable like any Spring bean. Construct with mock collaborators; call action methods directly.
- Use `FakeOperationContext` from `embabel-agent-test-support` to verify prompts and LLM invocations.
- Do not couple tests tightly to implementation details.
- Integration tests must not require network access or real LLM credentials.

Example unit test pattern:

```java
var context = new FakeOperationContext();
context.expectResponse(new MyResult("expected"));
myAgent.someAction(input, context);
var prompt = context.getLlmInvocations().getFirst().getPrompt();
assertTrue(prompt.contains("expected substring"));
```

## Documentation

- Documentation lives in `embabel-agent-docs/` and is written in **Asciidoctor**.
- When editing `foo.adoc`, check for a parallel `.foo.adoc` file that contains additional instructions.
- Never fabricate code examples — use code from this repository or the listed example repositories:
  - https://github.com/embabel/embabel-agent-examples
  - https://github.com/embabel/java-agent-template
  - https://github.com/embabel/impromptu

## MCP Server

Embabel can act as an MCP server over SSE. Configure Claude Desktop to connect:

```json
{
  "mcpServers": {
    "embabel": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:8080/sse"]
    }
  }
}
```

## A2A Integration

Enable the `a2a` Spring profile and set `GOOGLE_STUDIO_API_KEY`. Start the A2A web UI with:

```bash
docker compose --profile a2a up
```

Connect to your agent at `host.docker.internal:8080/a2a`.

## Adding New LLMs

Define Spring beans of type `Llm`. See `OpenAiConfiguration` for an example. Provide the knowledge cutoff date where known and make the configuration class conditional on the required API key.

---
(c) Embabel Software Inc 2024-2025.
