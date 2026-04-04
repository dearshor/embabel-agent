# embabel-agent-autoconfigure

**Maven artifact:** `com.embabel.agent:embabel-agent-autoconfigure` (parent POM)

This module contains all Spring Boot autoconfiguration classes. It is split into two groups:

1. **Platform/feature autoconfiguration** — wires up the AgentPlatform, Shell, MCP server, A2A, and observability.
2. **Model autoconfiguration** (`models/`) — registers `Llm` and `EmbeddingService` beans for each supported AI provider.

Users do not depend on this module directly; they use starters from `embabel-agent-starters`.

---

## Platform autoconfiguration sub-modules

| Module | Key configuration class | What it wires up |
|---|---|---|
| `embabel-agent-platform-autoconfigure` | `AgentPlatformConfiguration` | `DefaultAgentPlatform`, `Autonomy`, `ToolGroupResolver`, `Asyncer` |
| `embabel-agent-shell-autoconfigure` | `ShellConfiguration` | `ShellCommands`, personality providers, `TerminalServices` |
| `embabel-agent-mcpserver-autoconfigure` | `McpSyncServerConfiguration` / `McpAsyncServerConfiguration` | MCP SSE server, tool publishers |
| `embabel-agent-mcpserver-security-autoconfigure` | `SecureAgentToolConfiguration` | `SecureAgentToolAspect` + Spring method security for `@SecureAgentTool` |
| `embabel-agent-a2a-autoconfigure` | `A2AConfiguration` | A2A endpoint registration, `A2ARequestHandler` |
| `embabel-agent-observability-autoconfigure` | `EmbabelObservabilityAutoConfiguration` | Tracing observation handlers, metrics listener, MDC propagation |
| `embabel-agent-netty-client-autoconfigure` | `NettyClientConfiguration` | Netty-based HTTP client (for streaming) |

---

## Model autoconfiguration sub-modules

Located under `embabel-agent-autoconfigure/models/`.

| Module | Provider | Key class | Required environment variable |
|---|---|---|---|
| `embabel-agent-openai-autoconfigure` | OpenAI | `OpenAiModelsConfig` | `OPENAI_API_KEY` |
| `embabel-agent-openai-custom-autoconfigure` | Custom OpenAI-compatible | `OpenAiCustomModelsConfig` | (configurable) |
| `embabel-agent-anthropic-autoconfigure` | Anthropic | `AnthropicModelsConfig` | `ANTHROPIC_API_KEY` |
| `embabel-agent-bedrock-autoconfigure` | AWS Bedrock | `BedrockModelsConfig` | AWS credentials |
| `embabel-agent-gemini-autoconfigure` | Google Gemini (VertexAI) | `GeminiModelsConfig` | `GOOGLE_APPLICATION_CREDENTIALS` |
| `embabel-agent-google-genai-autoconfigure` | Google GenAI Studio | `GoogleGenAiModelsConfig` | `GOOGLE_STUDIO_API_KEY` |
| `embabel-agent-minimax-autoconfigure` | MiniMax | `MiniMaxModelsConfig` | `MINIMAX_API_KEY` |
| `embabel-agent-mistral-ai-autoconfigure` | Mistral AI | `MistralAiModelsConfig` | `MISTRAL_API_KEY` |
| `embabel-agent-ollama-autoconfigure` | Ollama (local) | `OllamaModelsConfig` | — (localhost) |
| `embabel-agent-deepseek-autoconfigure` | DeepSeek | `DeepSeekModelsConfig` | `DEEPSEEK_API_KEY` |
| `embabel-agent-dockermodels-autoconfigure` | Docker Model Runner | `DockerLocalModelsConfig` | — (Docker Desktop) |
| `embabel-agent-lmstudio-autoconfigure` | LM Studio | `LmStudioModelsConfig` | — (localhost) |
| `embabel-agent-onnx-autoconfigure` | ONNX Runtime (local embeddings) | `OnnxEmbeddingAutoConfiguration` | — (local model files) |

**MiniMax models**

MiniMax is an OpenAI-compatible provider with large context windows (up to 1M tokens). Available constants in `MiniMaxModels.kt`:

| Constant | Model ID |
|---|---|
| `MiniMaxModels.MINIMAX_M2_7` | `MiniMax-M2.7` |
| `MiniMaxModels.MINIMAX_M2_7_HIGHSPEED` | `MiniMax-M2.7-highspeed` |

Activated by setting `MINIMAX_API_KEY`. The configuration prefix is `embabel.agent.platform.models.minimax`.

**New Google GenAI / Gemini models**

| Constant | Model ID | Notes |
|---|---|---|
| `GEMINI_3_1_FLASH_LITE_PREVIEW` | `gemini-3.1-flash-lite-preview` | Lightweight preview model |
| `GEMINI_3_1_PRO_PREVIEW_CUSTOMTOOLS` | `gemini-3.1-pro-preview-customtools` | Pro preview with custom tool support |

Both constants are available in `GeminiModels.java` (Java) and `GoogleGenAiModels.kt` (Kotlin).

**Updated Anthropic models**

The following Claude 4.x model constants are available in `AnthropicModels.kt`:

| Constant | Model ID |
|---|---|
| `CLAUDE_37_SONNET` | `claude-3-7-sonnet-latest` |
| `CLAUDE_35_HAIKU` | `claude-3-5-haiku-latest` |
| `CLAUDE_40_OPUS` | `claude-opus-4-20250514` |
| `CLAUDE_41_OPUS` | `claude-opus-4-1` |
| `CLAUDE_SONNET_4_5` | `claude-sonnet-4-5` |
| `CLAUDE_HAIKU_4_5` | `claude-haiku-4-5` |
| `CLAUDE_OPUS_4_6` | `claude-opus-4-6` |
| `CLAUDE_SONNET_4_6` | `claude-sonnet-4-6` |

**ONNX local embeddings**

`embabel-agent-onnx-autoconfigure` wires up `OnnxEmbeddingService` as a local embedding provider using ONNX Runtime and DJL HuggingFace tokenizers. The default model is `all-MiniLM-L6-v2` (384 dimensions). Use the `embabel-agent-starter-onnx` starter to activate.

Each model config class follows the same pattern:

1. Extends `AbstractModelLoader`.
2. Is `@ConditionalOnProperty` or `@ConditionalOnEnvironmentVariable` — enabled only when the required key is present.
3. Fetches available models from the provider.
4. Registers a list of `Llm` Spring beans (one per model).

---

## How to add autoconfiguration for a new LLM provider

1. Create a sub-module under `models/` (copy `embabel-agent-openai-autoconfigure` as template).
2. Create a `@Configuration` class that:
   - `@ConditionalOnProperty` on the relevant API key env var.
   - Extends or delegates to `AbstractModelLoader`.
   - Registers `Llm` beans for each model.
3. Register your configuration in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
4. Implement `OptionsConverter<YourProviderChatOptions>` if the provider has non-standard options.
5. If the provider requires a specific tool response format (e.g. Google GenAI requires valid JSON objects), supply a `ToolResponseContentAdapter` when constructing `SpringAiLlmService`. See `GoogleGenAiModelsConfig` and `JsonWrappingToolResponseContentAdapter` for an example.

---

## Key SPI types used by platform autoconfigure

| Interface | Default implementation | Purpose |
|---|---|---|
| `ToolGroupResolver` | `SpringBeanToolGroupResolver` | Resolves tool groups by name to `ToolGroup` beans |
| `AgentProcessRepository` | `InMemoryAgentProcessRepository` | Stores active `AgentProcess` objects |
| `ContextRepository` | `InMemoryContextRepository` | Stores conversational context |
| `BlackboardProvider` | `InMemoryBlackboardProvider` | Creates `Blackboard` instances |
| `OperationScheduler` | `OperationScheduler.PRONTO` | Controls when actions are dispatched |
| `Asyncer` | `SpringContextAsyncExecutor` | Runs background tasks |

Override any of these by providing your own Spring bean.
