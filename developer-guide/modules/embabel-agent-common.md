# embabel-agent-common

**Maven artifact:** `com.embabel.agent:embabel-agent-common` (parent POM)

This module is a collection of sub-modules that provide shared AI model abstractions and web MVC utilities. It has no agent-specific logic; it can be depended on by any module that needs to work with LLMs or embedding models.

## Sub-modules

| Sub-module | Artifact | Purpose |
|---|---|---|
| `embabel-agent-ai` | `embabel-agent-ai` | LLM/embedding model abstractions, prompt contributors, output converters |
| `embabel-agent-webmvc` | `embabel-agent-webmvc` | Spring MVC helpers for agents exposed over HTTP |

---

## embabel-agent-ai

### Package map

```
com.embabel.common.ai.model/
├── AiModel.kt                # Base type for any AI model
├── EmbeddingService.kt       # Embedding operations abstraction
├── LlmOptions.kt             # Options for a single LLM call (model, temp, …)
├── LlmMetadata.kt            # Metadata about an LLM (knowledge cutoff, etc.)
├── ModelSelectionCriteria.kt # Criteria for choosing an LLM at runtime
├── OptionsConverter.kt       # Converts LlmOptions → provider-specific options
├── PricingModel.kt           # Cost estimation per token
└── SpringAiEmbeddingService.kt

com.embabel.common.ai.prompt/
├── PromptContributor.kt      # Interface to inject extra text into prompts
├── CurrentDate.kt            # Auto-injects current date into every prompt
└── KnowledgeCutoffDate.kt    # Injects knowledge cutoff date

com.embabel.common.ai.converters/
├── JacksonOutputConverter.kt           # Spring AI OutputConverter using Jackson
├── FilteringJacksonOutputConverter.kt  # Variant that strips ignored fields
└── streaming/
    └── StreamingJacksonOutputConverter.kt

com.embabel.common.ai.autoconfig/
├── AbstractModelLoader.kt              # Base class for all per-provider loaders
└── LlmAutoConfigMetadata.kt
```

### Key types

#### `LlmOptions`

Represents the configuration for a single LLM call. Immutable; use withers to produce variants.

Key fields:
- `model: String?` — explicit model name
- `criteria: ModelSelectionCriteria` — how to select a model (by name, role, default, auto)
- `temperature: Double?`
- `maxTokens: Int?`
- `toolGroups: Set<ToolGroupRequirement>`

Common factory methods:
- `LlmOptions.withModel("gpt-4o")` — select by name
- `LlmOptions.withAutoLlm()` — automatic selection
- `LlmOptions()` — use default model

#### `ModelSelectionCriteria`

Sealed hierarchy:

```kotlin
DefaultModelSelectionCriteria       // Use the configured default
AutoModelSelectionCriteria          // Auto-select based on task
ByNameModelSelectionCriteria        // Select by exact model name
ByRoleModelSelectionCriteria        // Select by role (e.g. "coding")
FallbackByNameModelSelectionCriteria // Try each name until one is found
```

#### `PromptContributor`

Implement this interface and register as a Spring bean to inject content into every prompt within an agent execution. Examples: `CurrentDate`, `KnowledgeCutoffDate`.

```kotlin
interface PromptContributor {
    fun promptContribution(): PromptContribution  // role + content
}
```

#### `OptionsConverter`

Each provider module registers a bean of type `OptionsConverter<T>` that translates `LlmOptions` into the provider-specific options type (e.g. `OpenAiChatOptions`). Implement this to add custom option mapping for a new provider.

---

## How to add a new LLM provider

1. Create a module under `embabel-agent-autoconfigure/models/`.
2. Implement `AbstractModelLoader` — it discovers models from the provider and registers them as `Llm` beans.
3. Implement `OptionsConverter<YourProviderOptions>` to translate `LlmOptions`.
4. Create a Spring Boot autoconfiguration class and add it to `spring.factories` / `AutoConfiguration.imports`.
5. Create a corresponding starter under `embabel-agent-starters/`.

Look at `embabel-agent-openai-autoconfigure` as a reference implementation.
