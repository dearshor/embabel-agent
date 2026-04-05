# embabel-agent-onnx

**Maven artifact:** `com.embabel.agent:embabel-agent-onnx`

Provides a local embedding service backed by [ONNX Runtime](https://onnxruntime.ai/) and the [DJL HuggingFace tokenizer](https://djl.ai/) library. No external API key or network call is required at inference time — the model runs entirely in-process.

Use the `embabel-agent-starter-onnx` starter to activate the Spring Boot autoconfiguration (`embabel-agent-onnx-autoconfigure`).

---

## Package map

```
com.embabel.agent.onnx/
├── OnnxModelLoader.kt          # Loads an ONNX model file into an OrtSession
└── embeddings/
    └── OnnxEmbeddingService.kt # EmbeddingService implementation using ONNX Runtime
```

---

## Key classes

### `OnnxEmbeddingService`

**Package:** `com.embabel.agent.onnx.embeddings`

Implements `com.embabel.common.ai.model.EmbeddingService`. Accepts a batch of strings and returns L2-normalised float vectors.

- Default model: `all-MiniLM-L6-v2` (384 dimensions)
- Implements `AutoCloseable` — close to release the ONNX `OrtSession` and environment.

### `OnnxModelLoader`

**Package:** `com.embabel.agent.onnx`

Utility that loads an ONNX model file from a `Path` and returns a ready `OrtSession`. Used internally by the autoconfiguration.

---

## Configuration

The `embabel-agent-onnx-autoconfigure` module wires up `OnnxEmbeddingService` as a Spring bean when the `embabel-agent-starter-onnx` starter is on the classpath.

Configuration prefix: `embabel.agent.platform.models.onnx`

| Property | Default | Description |
|---|---|---|
| `embabel.agent.platform.models.onnx.model-path` | (classpath default) | Path to the `.onnx` model file |
| `embabel.agent.platform.models.onnx.tokenizer-path` | (classpath default) | Path to the HuggingFace tokenizer directory |
| `embabel.agent.platform.models.onnx.dimensions` | `384` | Embedding vector dimensions |

---

## Usage

### Adding the dependency

```xml
<dependency>
    <groupId>com.embabel.agent</groupId>
    <artifactId>embabel-agent-starter-onnx</artifactId>
</dependency>
```

### Injecting the embedding service

```kotlin
@Agent(description = "Knowledge agent with local embeddings")
class KnowledgeAgent(
    private val embeddingService: EmbeddingService,
) {
    @Action
    fun embed(input: UserInput, ai: Ai): EmbeddedResult {
        val vector = embeddingService.embed(input.content)
        // use vector for similarity search …
    }
}
```

If multiple `EmbeddingService` beans are registered, qualify by name or use `@Primary` on the desired bean.

---

## When to use

Use `embabel-agent-onnx` when:

- You need fast, offline-capable embeddings without an API key.
- You want to avoid per-call costs for high-volume similarity searches.
- You are deploying in an air-gapped environment.

For cloud-hosted embeddings (OpenAI `text-embedding-*`, Bedrock Titan, etc.) use the corresponding provider autoconfiguration instead.
