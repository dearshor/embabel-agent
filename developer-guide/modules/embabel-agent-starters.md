# embabel-agent-starters

**Maven artifact:** `com.embabel.agent:embabel-agent-starters` (parent POM)

Starters are thin, opinionated dependency aggregators. They pull together the relevant autoconfiguration modules so application developers need only one dependency.

## Available starters

| Starter artifact | What it includes |
|---|---|
| `embabel-agent-starter` | Core platform + OpenAI (the most common starting point) |
| `embabel-agent-starter-platform` | Core agent platform only (no LLM provider) |
| `embabel-agent-starter-openai` | OpenAI model support |
| `embabel-agent-starter-anthropic` | Anthropic / Claude model support |
| `embabel-agent-starter-bedrock` | AWS Bedrock model support |
| `embabel-agent-starter-gemini` | Google Gemini (VertexAI) model support |
| `embabel-agent-starter-google-genai` | Google GenAI Studio model support |
| `embabel-agent-starter-mistral-ai` | Mistral AI model support |
| `embabel-agent-starter-ollama` | Ollama local model support |
| `embabel-agent-starter-deepseek` | DeepSeek model support |
| `embabel-agent-starter-dockermodels` | Docker Desktop model runner |
| `embabel-agent-starter-lmstudio` | LM Studio local model support |
| `embabel-agent-starter-openai-custom` | Custom OpenAI-compatible endpoint |
| `embabel-agent-starter-shell` | Spring Shell CLI |
| `embabel-agent-starter-mcpserver` | MCP SSE server |
| `embabel-agent-starter-a2a` | Google A2A protocol |
| `embabel-agent-starter-observability` | Tracing & metrics |
| `embabel-agent-starter-webmvc` | Spring MVC agent endpoint support |

## Typical dependencies

### Minimal application (OpenAI + Shell REPL)

```xml
<dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter</artifactId>
</dependency>
<dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter-shell</artifactId>
</dependency>
```

### Adding observability

```xml
<dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter-observability</artifactId>
</dependency>
```

### Adding a second LLM provider

```xml
<dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter-anthropic</artifactId>
</dependency>
```

## Creating a new starter

1. Create a directory under `embabel-agent-starters/`.
2. Add a `pom.xml` that depends only on the relevant autoconfigure modules — no code.
3. Add the starter to the parent `embabel-agent-starters/pom.xml` modules list.
