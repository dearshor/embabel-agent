# embabel-agent-openai

**Maven artifact:** `com.embabel.agent:embabel-agent-openai`

Provides OpenAI-specific utilities that are shared by the OpenAI autoconfiguration module and any other module that needs to construct OpenAI-compatible client objects programmatically.

---

## Package map

```
com.embabel.agent.openai/
├── OpenAiCompatibleModelFactory.kt  # Creates OpenAI-compatible chat/embedding models
└── converters.kt                    # LlmOptions → OpenAiChatOptions conversion
```

---

## Key classes

### `OpenAiCompatibleModelFactory`

Factory for creating Spring AI `OpenAiChatModel` and `OpenAiEmbeddingModel` instances pointing at any OpenAI-compatible endpoint. Used by:

- `OpenAiModelsConfig` (OpenAI cloud)
- `OpenAiCustomModelsConfig` (custom endpoints, e.g. Azure OpenAI, local proxies)
- `LmStudioModelsConfig`
- `DeepSeekModelsConfig`

### `converters.kt`

Contains the `LlmOptions → OpenAiChatOptions` conversion logic. This is an `OptionsConverter<OpenAiChatOptions>` implementation and is the reference for how to write a converter for other providers.

---

## Extension point

If you need to override how OpenAI options are constructed (e.g. to set provider-specific headers or custom fields), extend or replace the converter by providing a Spring bean of type `OptionsConverter<OpenAiChatOptions>`.
