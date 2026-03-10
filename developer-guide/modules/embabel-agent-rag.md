# embabel-agent-rag

**Maven artifact:** `com.embabel.agent:embabel-agent-rag` (parent POM)

Provides a complete Retrieval-Augmented Generation (RAG) subsystem. It is modular: you pick the storage backend and the pipeline features you need.

---

## Sub-modules

| Sub-module | Artifact | Purpose |
|---|---|---|
| `embabel-agent-rag-core` | `embabel-agent-rag-core` | Domain model, ingestion interfaces, storage interfaces, tools |
| `embabel-agent-rag-pipeline` | `embabel-agent-rag-pipeline` | `RagService`, pipeline enhancers (HyDE, reranking, compression) |
| `embabel-agent-rag-lucene` | `embabel-agent-rag-lucene` | Embedded Apache Lucene vector + keyword store |
| `embabel-agent-rag-tika` | `embabel-agent-rag-tika` | Apache Tika document parsers (HTML, Markdown, plain text) |
| `embabel-agent-rag-neo` | `embabel-agent-rag-neo` | Neo4j graph-based storage (experimental) |

---

## Core concepts

### Content model (rag-core)

```
NavigableDocument                # A document with sections (hierarchical)
├── HierarchicalContentElement   # A node in the document tree
│   └── Chunk                    # Atomic retrievable unit
└── Source                       # Origin URI + metadata

NamedEntity                      # A typed entity extracted from content
Retrievable                      # Anything that can be retrieved by search
```

### Ingestion pipeline

```
HierarchicalContentReader        # Reads a source → NavigableDocument
    (TikaHierarchicalContentReader for files/URLs)
ContentChunker                   # Splits into Chunks
ChunkTransformer                 # Mutates chunks (e.g. AddTitlesChunkTransformer)
Ingester                         # Orchestrates read → chunk → store
ContentElementRepository         # Stores Chunks + Retrievables
ContentRefreshPolicy             # Controls when to re-ingest (Always/Never/TTL)
```

### Search pipeline (rag-pipeline)

```
RagService                       # Top-level search entry point
RagRequest                       # Query + options
RagResponse                      # Retrieved chunks + metadata
RagServiceEnhancer               # Decorates RagService with extra steps
PipelinedRagServiceEnhancer      # Chains multiple enhancers
```

Built-in enhancers:

| Enhancer | Class | Purpose |
|---|---|---|
| HyDE | `HyDEQueryGenerator` | Generate a hypothetical answer, then search with it |
| Reranking | `RerankingEnhancer` | Re-score results using an LLM |
| Contextual compression | `PromptedContextualCompressionEnhancer` | Compress chunks to only the relevant parts |
| Chunk merging | `ChunkMergingEnhancer` | Merge adjacent chunks from the same document |
| Deduplication | `DeduplicatingEnhancer` | Remove duplicate chunks |
| Filter | `FilterEnhancer` | Apply property-based filters |

---

## Key classes

### `RagService`

**Package:** `com.embabel.agent.rag.service`

The main search interface:

```kotlin
interface RagService {
    fun search(request: RagRequest): RagResponse
}
```

### `ToolishRag`

**Package:** `com.embabel.agent.rag.tools`

Wraps a `RagService` and exposes its functionality as Spring AI `@Tool` methods that can be given to an LLM. Used by `@Action` methods that need RAG-backed search tools.

### `ContentElementRepository`

**Package:** `com.embabel.agent.rag.store`

Interface for storing and retrieving `ContentElement` objects (chunks + embeddings).

Implementations:
- `LuceneSearchOperations` (rag-lucene) — embedded, no external DB required

### `SearchOperations`

**Package:** `com.embabel.agent.rag.service`

Lower-level interface for vector similarity and keyword search. Use `SearchOperationsBuilder` to create instances.

---

## How to add RAG to an agent

```kotlin
@Agent(description = "Answers questions from the knowledge base")
class KnowledgeAgent(
    private val ragService: RagService,
) {
    @Action(toolGroups = [CoreToolGroups.RAG])
    fun answer(question: UserInput, ai: Ai): Answer {
        val context = ToolishRag(ragService)
        return ai.withDefaultLlm()
            .withToolObject(context)
            .createObject(question.content)
    }
}
```

Or expose RAG as a tool group directly by registering a `ToolGroup` bean backed by a `ToolishRag`.

---

## Lucene backend (rag-lucene)

No external services required. Add `embabel-agent-rag-lucene` to your dependencies. It uses an embedded Lucene index stored under `${user.home}/.embabel/lucene/` by default.

Key classes:

| Class | Purpose |
|---|---|
| `LuceneSearchOperations` | Vector + keyword search implementation |
| `LuceneSearchOperationsBuilder` | Fluent builder for `LuceneSearchOperations` |
| `LuceneDocumentMapper` | Converts between `Chunk` and Lucene `Document` |

---

## Tika document parsing (rag-tika)

Parses HTML, Markdown, and plain text files into `NavigableDocument` trees.

`TikaHierarchicalContentReader` (`rag-tika`) is the entry point:

```kotlin
val reader = TikaHierarchicalContentReader()
val doc = reader.read(url)  // returns NavigableDocument
```

To add a new format, implement `ContentFormatParser` and register it as a Spring bean.

---

## Extending the RAG pipeline

To add a new pipeline enhancer:

1. Implement `RagServiceEnhancer`:
   ```kotlin
   class MyEnhancer(private val delegate: RagService) : RagServiceEnhancer {
       override fun enhance(request: RagRequest, response: RagResponse): RagResponse { ... }
   }
   ```
2. Register as a Spring bean.
3. The `RagPipelineConfiguration` auto-discovers all `RagServiceEnhancer` beans and chains them.
